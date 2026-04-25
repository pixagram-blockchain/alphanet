# Pixagram Alphanet

Docker Compose stack for running the Pixagram pre-mainnet: blockchain node, HAF indexer, Hivemind social layer, API proxy, price feed, and SSL.

## Architecture

```
Internet → Caddy (SSL) → Jussi (API proxy + field-name translation) → hived / Hivemind
```

Jussi routes JSON-RPC requests to the correct backend (hived for chain queries, Hivemind for social queries) and translates between chain-internal field names (`reward_hive`, `hbd_balance`, …) and Pixagram field names (`reward_pixa`, `pxs_balance`, …) bidirectionally. Asset symbols (`PIXA`, `PXS`) are emitted natively by hived and not translated.

## Services

| Service | Container | Image | Ports | Description |
|---|---|---|---|---|
| **pixagram** | `pixagram_container` | `pixadock/pixagram:pre-mainnet` | 7777 (HTTP), 2001 (P2P) | Main blockchain node (hived) |
| **pixagram_haf** | `pixagram_haf_container` | `pixadock/pixagram-haf:pre-mainnet` | 7779 (HTTP), 8092 (WS), 2002 (P2P) | HAF node (hived + PostgreSQL indexer) |
| **hivemind_sync** | `hivemind_sync_container` | `mkysel/hivemind:x86-testnet` | — | Block processor, indexes HAF data for social queries |
| **hivemind** | `hivemind_container` | `mkysel/hivemind:x86-testnet` | 7778 (HTTP) | PostgREST API server for social queries (bridge, follow, tags) |
| **jussi** | `jussi_container` | `openresty/openresty:alpine` | 8080 (internal) | API proxy: routes requests + field-name translation |
| **bigmac-feed** | `bigmac_feed_container` | `pixadock/bigmac-feed:latest` | — | Witness price feed (1 PXS = 1 Big Mac) |
| **ssl-proxy** | `ssl_proxy_container` | `caddy:alpine` | 80, 443 | TLS termination with auto-cert |

### Init services (run once)

| Service | Description |
|---|---|
| **init_permissions** | Sets file ownership for bind mounts |
| **hivemind_setup** | Creates Hivemind DB schemas and roles |

## Setup

```bash
git clone https://github.com/pixagram-blockchain/alphanet.git
cd alphanet

# Set the witness key for the price feed
echo "WITNESS_WIF=<active-private-key>" > .env

# Start everything
docker compose up -d
```

## API Endpoints

| Endpoint | URL |
|---|---|
| Public API (via Caddy) | `https://api.pixagram.com` |
| Pixagram node (direct) | `http://localhost:7777` |
| Hivemind (direct) | `http://localhost:7778` |
| HAF node (direct) | `http://localhost:7779` |

### Jussi Routing

Requests are routed to Hivemind for social queries (`bridge.*`, `follow_api.*`, `tags_api.*`, and specific `condenser_api` methods like `get_content`, `get_followers`, `get_trending_tags`, etc.). Everything else goes to hived. See `jussi/nginx.conf` for the full routing table.

### Field Name Translation

Hived's source still uses Hive-derived field names (`hbd_balance`, `reward_hive`, `total_vesting_fund_hive`, …). Jussi rewrites these on the wire so external clients see Pixagram-flavored names:

| Chain (internal) | API (external) |
|---|---|
| `reward_hive` | `reward_pixa` |
| `reward_hbd` | `reward_pxs` |
| `hbd_balance` | `pxs_balance` |
| `hbd_exchange_rate` | `pxs_exchange_rate` |
| `total_vesting_fund_hive` | `total_vesting_fund_pixa` |
| `current_hbd_supply` | `current_pxs_supply` |
| `dhf_interval_ledger` | `dpf_interval_ledger` |
| ... | (see `jussi/nginx.conf` for full list) |

Translation runs in both directions, including broadcast transactions (signatures are over the binary NAI representation, not JSON, so renames are safe).

**Defensive response-side symbol renames** also run — pre-mainnet hived emits PIXA/PXS natively for new ops, but legacy code paths can still leak old symbol names; jussi rewrites those before the client sees them:

| Chain (legacy leak) | API (external) |
|---|---|
| `TESTS` | `PIXA` |
| `TBD` | `PXS` |
| `HBD` | `PXS` |

These are response-only — request-side symbol renames are intentionally omitted because the chain natively expects PIXA/PXS in broadcasts.

## Price Feed (Big Mac Index)

The `bigmac-feed` service publishes a witness price feed where **1 PXS = 1 Big Mac**. It fetches the latest US Big Mac price from [The Economist](https://www.economist.com/big-mac-index) and calculates:

```
1 PXS = Big Mac USD price / PIXA USD price
```

Currently: `1 PXS = $6.12 / $0.06 = 102 PIXA`

Configuration via `docker-compose.yml` command args:
- `--witness` — witness account (default: `initminer`)
- `--token-price` — USD price of 1 PIXA (default: `0.06`)
- `--interval` — publish frequency (default: `1h`)

## Volumes

- `./pixagram/` — blockchain data for the main node
- `./pixagram-haf/` — blockchain data + PostgreSQL for the HAF node
- `./jussi/nginx.conf` — API proxy configuration
- `./ssl-proxy/Caddyfile` — Caddy TLS configuration

## Useful Commands

```bash
# Check all container statuses
docker compose ps

# View logs
docker compose logs -f pixagram_haf    # HAF node
docker compose logs -f hivemind_sync   # Hivemind block processor
docker compose logs -f bigmac_feed     # Price feed

# Restart Jussi after config changes
docker compose restart jussi

# Test the API
curl -s -X POST https://api.pixagram.com \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"condenser_api.get_dynamic_global_properties","params":[],"id":1}'
```

## Notes

- The `pixagram_haf` service has `restart: unless-stopped` to auto-recover from crashes
- P2P ports (2001, 2002) are exposed directly for peer connections
- The `.env` file contains the witness private key — do not commit it
- If you see `Permission denied`, fix ownership: `sudo chown -R 1000:1000 ./pixagram ./pixagram-haf`
