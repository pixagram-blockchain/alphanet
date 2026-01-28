# Pixagram Alphanet Docker Setup

This repository contains a `docker-compose.yml` file for running **Pixagram**, **Pixagram HAF**, and **Hivemind** directly (no nginx proxy).

## Requirements

- **Docker**: Install [Docker](https://www.docker.com/).
- **Docker Compose**: Ensure you have Docker Compose installed.

## Services

### Pixagram

- **Image**: `mkysel/pixagram:testnet-x86`
- **Container Name**: `pixagram_container`
- **Ports**: HTTP is published directly on `7777`. P2P is published directly on `2001`.
- **Volumes**: Mounts the local directory `./pixagram` to `/home/hived/datadir` in the container.

### Pixagram HAF

- **Image**: `mkysel/pixagram-haf:testnet-x86`
- **Container Name**: `pixagram_haf_container`
- **Ports**: HTTP is published directly on `7779`. WS is published directly on `8092`. P2P is published directly on `2002`.
- **Volumes**: Mounts the local directory `./pixagram-haf` to `/home/hived/datadir` in the container.

### Hivemind

- **Image**: `mkysel/hivemind:x86-testnet`
- **Container Name**: `hivemind_container`
- **Ports**: HTTP is published directly on `7778` (container port `8080`).
- **Notes**: Requires a one-time setup step (`setup --with-apps`) to create DB roles and schemas.

## Usage

1. Clone the repository and navigate to the directory:
   ```bash
   git clone <repository_url>
   cd <repository_directory>
   ```

2. Start the containers (the init services run automatically on first start):
   ```bash
   docker compose up -d
   ```

   If you need to re-run the Hivemind setup later:
   ```bash
   docker compose run --rm hivemind_setup
   ```

3. Access the services:
   - Pixagram HTTP: `http://localhost:7777`
   - Pixagram P2P: `tcp://localhost:2001`
   - Hivemind HTTP: `http://localhost:7778`
   - Pixagram HAF WS: `ws://localhost:8092`
   - Pixagram HAF HTTP: `http://localhost:7779`
   - Pixagram HAF P2P: `tcp://localhost:2002`

4. To stop the containers:
   ```bash
   docker compose down
   ```

## Notes

- There is no CORS proxy; configure CORS in the application(s) if needed.
- P2P is exposed directly on `2001` and `2002`.
- The `./pixagram` and `./pixagram-haf` directories are mounted for persistent storage.
- If you see `Permission denied` in `pixagram` or `pixagram_haf`, fix ownership on the host:
  ```bash
  sudo chown -R 1000:1000 ./pixagram ./pixagram-haf
  ```
  These containers run as a non-root user (UID 1000), so the bind-mounted folders must be writable.
- The `init_permissions` and `hivemind_setup` services are one-shot init steps; they exit after completing.
