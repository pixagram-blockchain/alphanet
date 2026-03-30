# Pixagram Alphanet

Docker Compose stack for running the Pixagram testnet: blockchain node, HAF indexer, Hivemind social layer, API proxy, price feed, and SSL.

## Architecture

```
Internet → Caddy (SSL) → Jussi (API proxy + token translation) → hived / Hivemind
```

Jussi routes JSON-RPC requests to the correct backend and translates between chain-native names (`reward_hive`, `TESTS`, `TBD`) and Pixagram names (`reward_pixa`, `PIXA`, `PXS`) bidirectionally.

## Services

| Service | Container | Image | Ports | Description |
|---|---|---|---|---|
| **pixagram** | `pixagram_container` | `mkysel/pixagram:testnet-x86` | 7777 (HTTP), 2001 (P2P) | Main blockchain node (hived) |
| **pixagram_haf** | `pixagram_haf_container` | `mkysel/pixagram-haf:testnet-x86` | 7779 (HTTP), 8092 (WS), 2002 (P2P) | HAF node (hived + PostgreSQL indexer) |
| **hivemind_sync** | `hivemind_sync_container` | `mkysel/hivemind:x86-testnet` | — | Block processor, indexes HAF data for social queries |
| **hivemind** | `hivemind_container` | `mkysel/hivemind:x86-testnet` | 7778 (HTTP) | PostgREST API server for social queries (bridge, follow, tags) |
| **jussi** | `jussi_container` | `openresty/openresty:alpine` | 8080 (internal) | API proxy: routes requests + PIXA/PXS token translation |
| **bigmac-feed** | `bigmac_feed_container` | `mkysel/bigmac-feed:latest` | — | Witness price feed (1 PXS = 1 Big Mac) |
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
| Public API (via Caddy) | `https://pixagram.dev` |
| Pixagram node (direct) | `http://localhost:7777` |
| Hivemind (direct) | `http://localhost:7778` |
| HAF node (direct) | `http://localhost:7779` |

### Token Translation (Jussi)

The API proxy automatically translates between internal chain names and Pixagram names:

| Chain (internal) | API (external) |
|---|---|
| `TESTS` | `PIXA` |
| `TBD` | `PXS` |
| `HBD` | `PXS` |
| `reward_hive` | `reward_pixa` |
| `reward_hbd` | `reward_pxs` |
| `hbd_balance` | `pxs_balance` |
| `total_vesting_fund_hive` | `total_vesting_fund_pixa` |
| ... | (see `jussi/nginx.conf` for full list) |

This applies to both requests and responses. Broadcast transactions are also translated so clients can sign and send using PIXA/PXS.

### Jussi Routing

Requests are routed to Hivemind for social queries (`bridge.*`, `follow_api.*`, `tags_api.*`, and specific `condenser_api` methods like `get_content`, `get_followers`, `get_trending_tags`, etc.). Everything else goes to hived. See `jussi/nginx.conf` for the full routing table.

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
curl -s -X POST https://pixagram.dev \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"condenser_api.get_dynamic_global_properties","params":[],"id":1}'
```

## Notes

- The `pixagram_haf` service has `restart: unless-stopped` to auto-recover from crashes
- P2P ports (2001, 2002) are exposed directly for peer connections
- The `.env` file contains the witness private key — do not commit it
- If you see `Permission denied`, fix ownership: `sudo chown -R 1000:1000 ./pixagram ./pixagram-haf`
