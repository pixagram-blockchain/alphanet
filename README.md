# Pixagram Alphanet

**Internal deployment repo.** This is what actually runs on our own nodes, keys
and bootstrap settings included. It is not the thing to hand to an operator.

Public equivalents:

| Repo | For |
|---|---|
| [pixagram-blockchain/pixagram-node](https://github.com/pixagram-blockchain/pixagram-node) | Running a public API node |
| [pixagram-blockchain/witness](https://github.com/pixagram-blockchain/witness) | Running a witness (block producer) |

Changes made here do not reach those repos automatically.

Docker Compose stack for running the Pixagram pre-mainnet: blockchain node, HAF indexer, Hivemind social layer, API proxy, price feed, and SSL.

## Architecture

```
Internet → Caddy (SSL) → Jussi (API proxy + field-name translation) → hived / Hivemind
```

Jussi routes JSON-RPC requests to the correct backend (hived for chain queries, Hivemind for social queries) and translates between chain-internal field names (`reward_hive`, `hbd_balance`, …) and Pixagram field names (`reward_pixa`, `pxs_balance`, …) bidirectionally. Asset symbols (`PIXA`, `PXS`) are emitted natively by hived and not translated.

## Services

| Service | Container | Image | Ports | Description |
|---|---|---|---|---|
| **pixagram** | `pixagram_container` | `pixadock/pixagram:mainnet` | 7777 (HTTP), 2001 (P2P) | Main blockchain node (hived) |
| **pixagram_haf** | `pixagram_haf_container` | `pixadock/pixagram-haf:mainnet` | 7779 (HTTP), 8092 (WS), 2002 (P2P) | HAF node (hived + PostgreSQL indexer) |
| **hivemind_sync** | `hivemind_sync_container` | `pixadock/hivemind:mainnet` | — | Block processor, indexes HAF data for social queries |
| **hivemind** | `hivemind_container` | `pixadock/hivemind:mainnet` | 7778 (HTTP) | PostgREST API server for social queries (bridge, follow, tags) |
| **jussi** | `jussi_container` | `openresty/openresty:alpine` | 8080 (internal) | API proxy: routes requests + field-name translation |
| **bigmac-feed** | `bigmac_feed_container` | `pixadock/bigmac-feed:latest` | — | Witness price feed (1 PXS = 1 Big Mac) |
| **ssl-proxy** | `ssl_proxy_container` | `caddy:alpine` | 80, 443 | TLS termination with auto-cert |

### Init services (run once)

| Service | Description |
|---|---|
| **init_permissions** | Sets file ownership for bind mounts |
| **hivemind_setup** | Creates Hivemind DB schemas and roles |

## Required: a synchronised clock

Every machine running this stack must have an NTP daemon. Check it before
`docker compose up`, not after — these nodes produce blocks, and a drifting
clock on a block producer silently costs **other** witnesses their blocks.

Block slots are 3 seconds and hived decides when to produce from the local
system clock. A clock that is a second or two slow emits its block that late in
real time, so it reaches peers at the boundary of the *next* witness's slot.
That witness has not seen it, builds on the previous head, produces a competing
block at the same height and loses the fork race — and the miss is recorded
against **them**, not against the node with the bad clock. Your own
`total_missed` stays clean the whole time, so nothing in your own numbers will
tell you.

On 2026-09-06 one of our own witnesses was found running 2.672 s slow. It
accounted for 121 of the 123 same-height block collisions on the network and
109 missed blocks for the witness furthest from it. The box was a Debian 12
image with no time daemon installed at all. After adding one the offset fell to
3 ms and the misses stopped outright: 4163 blocks from 4163 slots over the next
3.5 hours, none empty. No hived restart was needed — the clock step is applied
underneath a running node, and stepping *forward* is safe for a producer since
it can only skip a slot, never double-sign.

**Check:**

```bash
timedatectl | grep -E 'synchronized|NTP service'
```

You want `System clock synchronized: yes` and an active service. Our Ubuntu
boxes use `systemd-timesyncd` and the GCP images use `chrony`, both enabled out
of the box. **The Debian 12 images ship with nothing** — both IBM boxes were
affected, one by 2.7 s and one by 12.4 s.

**If it is missing:**

```bash
sudo apt-get update && sudo apt-get install -y chrony
sudo systemctl enable --now chrony
```

`enable`, not just `start`, or it will not survive a reboot.

**Verify the real offset from the machine itself.** Comparing `date` over SSH
from a laptop measures your connection latency, not the clock — it produces
readings that grow in the order you polled the hosts.

```bash
python3 - <<'PY'
import socket, struct, time, statistics
def probe(host):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM); s.settimeout(4)
    try:
        t1 = time.time(); s.sendto(b'\x1b' + 47 * b'\0', (host, 123))
        d, _ = s.recvfrom(1024); t4 = time.time()
    finally:
        s.close()
    u = struct.unpack('!12I', d[:48])
    t2 = u[8] + u[9] / 2**32 - 2208988800
    t3 = u[10] + u[11] / 2**32 - 2208988800
    return ((t2 - t1) + (t3 - t4)) / 2
offsets = []
for host in ('pool.ntp.org', 'time.google.com', 'time.cloudflare.com'):
    for _ in range(3):
        try: offsets.append(probe(host))
        except Exception: pass
print(f'clock offset: {statistics.median(offsets):+.4f}s  ({len(offsets)} samples)'
      if offsets else 'could not reach any NTP server')
PY
```

Under ~50 ms is fine. Past half a second you are costing other witnesses blocks.

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

## Upgrading an existing deployment to v1.0.0

v1.0.0 moves to hived/HAF 1.28.7 and hivemind 1.28.6. Two things change that an
existing datadir does not survive on its own.

**1. The `metadata` plugin is now required.** Up to 1.28.5 `account_metadata_object`
was a core chain object, so account `json_metadata` / `posting_json_metadata`
(display names, avatars) were always stored. 1.28.7 moved that index into a
plugin. Without it every account profile reads back empty from the API.

Enabling it adds a new chainbase index, so you need **both** steps — either one
alone fails:

```bash
docker compose down

# 1. drop the state file (keeping it -> "Inconsistency occurs. A new index is
#    created, but other indexes are found in shared_memory_file")
rm -f pixagram/blockchain/shared_memory.bin
rm -rf pixagram/blockchain/account-history-rocksdb-storage \
       pixagram/blockchain/comments-rocksdb-storage

# 2. replay once (without it -> "Headblock and statefile are inconsistent,
#    need to start hived with --replay-blockchain")
HIVED_EXTRA_ARGS=--replay-blockchain docker compose up -d pixagram

# once it logs "entering live mode", start the rest normally
docker compose up -d
```

`block_log*` is the chain itself — never delete it. Everything else under
`blockchain/` is derived state and is rebuilt by the replay.

**2. `post_id` is gone from bridge API responses.** Upstream removed internal
post and vote IDs in 1.28.6, so `bridge.get_ranked_posts`, `get_account_posts`
and `get_post` now return `post_id: null`. Check any frontend that reads it.

## Upgrading to 1.29.0 (hardfork 29)

Hardfork 29 activates on **2026-09-18 12:00:00 UTC**. Every hived on the network -
witnesses and API nodes alike - must run 1.29.0 before then; after activation the network
rejects blocks and state produced by older versions.

hived stamps its build configuration into `shared_memory.bin` and refuses a state file
written by another version ("Blockchain config from shared memory file mismatch current
version of app"), so a plain `restart` is not enough. HAF's serializer likewise cannot
replay into a database that already holds blocks. The upgrade is therefore: replay the
consensus node from its block log, and rebuild HAF and Hivemind from scratch - the whole
chain resyncs in minutes.

```bash
git pull                                            # 1.29.0 tags in docker-compose.yml
docker compose pull pixagram pixagram_haf

# 1. consensus node: rebuild the state file from block_log (~20 s), then start normally
docker compose stop pixagram
HIVED_EXTRA_ARGS="--force-replay --exit-before-sync" docker compose run --rm --no-deps pixagram
docker compose up -d pixagram

# 2. HAF + Hivemind: drop derived state and let them resync together
docker compose stop pixagram_haf hivemind_sync hivemind hivemind_setup
docker compose rm -f pixagram_haf hivemind_sync hivemind hivemind_setup
sudo rm -rf pixagram-haf/haf_db_store pixagram-haf/blockchain pixagram-haf/logs pixagram-haf/p2p
docker compose up -d

# 3. Jussi keeps the old Hivemind address and answers 502 on bridge.* until restarted
docker compose restart jussi
```

HAF is healthy within a few minutes; `hivemind_sync` logs live `blocks N-N` lines after
10-15 minutes. Start Hivemind together with HAF as above - started against an
already-synced HAF, its massive-to-live hand-over has tripped on
`hive_notification_cache_pkey`. On a witness host the consensus node is offline for about
a minute: at most one or two missed slots.

## Notes

- The `pixagram_haf` service has `restart: unless-stopped` to auto-recover from crashes
- P2P ports (2001, 2002) are exposed directly for peer connections
- The `.env` file contains the witness private key — do not commit it
- If you see `Permission denied`, fix ownership: `sudo chown -R 1000:1000 ./pixagram ./pixagram-haf`
