# Pixagram Alphanet Docker Setup

This repository contains a `docker-compose.yml` file for running the **Pixagram** application using Docker.

## Requirements

- **Docker**: Install [Docker](https://www.docker.com/).
- **Docker Compose**: Ensure you have Docker Compose installed.

## Services

### Pixagram

- **Image**: `mkysel/pixagram:alpha`
- **Container Name**: `pixagram_container`
- **Ports**: Exposes port `7777` (container) mapped to port `7777` (host).
- **Volumes**: Mounts the local directory `./pixagram` to `/root/.pixagramd` in the container.
- **Command**: Starts the Pixagram service with `/usr/local/steemd`.

## Usage

1. Clone the repository and navigate to the directory:
   ```bash
   git clone <repository_url>
   cd <repository_directory>
   ```

2. Start the container:
   ```bash
   docker-compose up -d
   ```

3. Access the Pixagram service at `http://localhost:7777`.

4. To stop the container:
   ```bash
   docker-compose down
   ```

## Notes

- Modify the `docker-compose.yml` file to change configurations like ports or volumes if necessary.
- The `./pixagram` directory is mounted by default for persistent storage.
