# Docker Media Stack

A self-hosted media automation stack built on Docker Compose. Combines Jellyfin, Radarr, Sonarr, Prowlarr, qBittorrent, and ProtonVPN (WireGuard) into a single setup;

---

## Stack Overview

| Container | Purpose | Port |
|---|---|---|
| Jellyfin | Media server | 8096 |
| Radarr | Movie management | 7878 |
| Sonarr | TV show management | 8989 |
| Prowlarr | Indexer management | 9696 |
| qBittorrent | Torrent client (routed through VPN) | 8080 |
| Gluetun | WireGuard VPN gateway (ProtonVPN) | — |
| FlareSolverr | Cloudflare bypass for indexers | 8191 |
| Unpackerr | Auto-extraction of downloaded archives | — |
| Glances | System/container monitoring | 61208 |
| Watchtower | Automatic container image updates | — |

---

## Prerequisites

- Docker and Docker Compose v2 installed
- A ProtonVPN account (any paid plan supports WireGuard + port forwarding)
- Basic familiarity with a terminal

---

## Directory Structure

The stack expects a specific directory layout for media and configuration. Adjust `DATA_DIR` and `CONFIG_DIR` in your `.env` to match your system.

```
DATA_DIR/
├── torrents/
│   ├── movies/
│   └── tv/
├── movies/
└── tv/

CONFIG_DIR/
├── jellyfin_config/
├── radarr_config/
├── sonarr_config/
├── prowlarr_config/
└── qbittorrent_config/

gluetun/
└── config.toml        ← Gluetun API auth config (see below)
```

Create the data directories before starting the stack:

```bash
mkdir -p /mnt/disk/data/{torrents/movies,torrents/tv,movies,tv}
```

The config directories will be created automatically by Docker on first run.

---

## Environment Setup

Copy the template and fill in the values:

```bash
cp .env.template .env
```

### `.env` Reference

```dotenv
PUID=1000
PGID=1000
TZ=America/New_York
CONFIG_DIR=./config
DATA_DIR=/mnt/disk/data/
RADARR_API_KEY=~
SONARR_API_KEY=~
WIREGUARD_PRIVATE_KEY=~
SERVER_COUNTRIES="United States"
GSP_GTN_API_KEY=some_long_random_string
```

**`PUID` / `PGID`** — The user and group ID that containers will run as. Using `0` (root) works but is not recommended. To use your current user:

```bash
id -u   # outputs PUID
id -g   # outputs PGID
```

**`TZ`** — Your timezone in [tz database format](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). Example: `America/New_York`, `Europe/London`.

**`CONFIG_DIR`** — Where container config files are stored. `./config` (relative to the compose file) is fine for most setups.

**`DATA_DIR`** — The root of your media storage. All containers mount this directory so downloads, movies, and TV shares are accessible across the stack without copying files.

**`RADARR_API_KEY` / `SONARR_API_KEY`** — Obtained from within each container after first run. See the [Post-Start Configuration](#post-start-configuration) section below.

**`WIREGUARD_PRIVATE_KEY`** — Your ProtonVPN WireGuard private key. See the [ProtonVPN Setup](#protonvpn-wireguard-setup) section below.

**`SERVER_COUNTRIES`** — Which ProtonVPN country to connect to. Must be a valid ProtonVPN country name, e.g. `"United States"`, `"Netherlands"`.

**`GSP_GTN_API_KEY`** — A shared secret used by the GSP mod inside qBittorrent to authenticate with the Gluetun API. This value must match the `apikey` field in `gluetun/config.toml`. Generate a secure random string:

```bash
openssl rand -hex 32
```

Paste the same value into both `.env` (`GSP_GTN_API_KEY`) and `gluetun/config.toml` (`apikey`).

---

## ProtonVPN WireGuard Setup

1. Log in to [account.proton.me](https://account.proton.me) and navigate to **VPN > Downloads**.
2. Select **WireGuard configuration** as the protocol.
3. Create a new configuration. Select a server that supports **P2P** (required for port forwarding). Download the `.conf` file.
4. Open the file and copy the value after `PrivateKey =`. This is your `WIREGUARD_PRIVATE_KEY`.

> Port forwarding is required for good upload ratios and seeding performance. Only P2P-enabled ProtonVPN servers support it. In Gluetun, this is handled automatically when `VPN_PORT_FORWARDING=on` and `VPN_PORT_FORWARDING_PROVIDER=protonvpn` are set.

---

## Gluetun API Auth (`gluetun/config.toml`)

Create the file at `gluetun/config.toml` with the following content. The `apikey` must match `GSP_GTN_API_KEY` in your `.env`:

```toml
[[roles]]
name = "t-anc/GSP-Qbittorent-Gluetun-sync-port-mod"
routes = ["GET /v1/portforward"]
auth = "apikey"
apikey = "your_generated_key_here"
```

This restricts the Gluetun HTTP API so only the GSP port-sync mod can query it.

---

## Starting the Stack

```bash
docker compose up -d
```

Gluetun will start first and establish the VPN tunnel. qBittorrent will not start until Gluetun reports healthy. Allow 30–60 seconds on first run.

Check that the VPN is up before proceeding:

```bash
docker logs gluetun | grep "VPN is up"
```

---

## Post-Start Configuration

Complete the following steps in order. Most only need to be done once.

### 1. qBittorrent — Enable WebUI Bypass for Local Access

qBittorrent runs inside the Gluetun network namespace and its WebUI is exposed at `http://<host>:8080`.

1. Open the WebUI. Default credentials are `admin` / `adminadmin`.
2. Go to **Tools > Options > Web UI**.
3. Under **Authentication**, enable **Bypass authentication for clients on localhost** (also called **Bypass for whitelisted IPs**) and add your local subnet, e.g. `192.168.1.0/24`. Without this, Sonarr and Radarr cannot connect to qBittorrent from inside Docker.
4. Change the default password.
5. Under **Connection**, verify the listening port matches what the GSP mod has synced. You can check the synced port with:

```bash
docker logs qbittorrent | grep "port"
```

6. Set the **Default Save Path** to `/data/torrents/`. Set the category save paths to `/data/torrents/movies` and `/data/torrents/tv` respectively.

### 2. Radarr — Initial Setup

1. Open `http://<host>:7878`.
2. Complete the setup wizard. Set the media path to `/data/movies`.
3. Navigate to **Settings > General** and copy your API key. Add it to `.env` as `RADARR_API_KEY`.
4. Navigate to **Settings > Download Clients**, add qBittorrent:
   - Host: `localhost` (qBittorrent shares Gluetun's network)
   - Port: `8080`
   - Category: `movies`
5. Navigate to **Settings > Media Management** and enable **Rename Movies**. Enable **Hardlinks** if your downloads and library share the same filesystem (they do if both are under `DATA_DIR`).

### 3. Sonarr — Initial Setup

1. Open `http://<host>:8989`.
2. Set the media path to `/data/tv`.
3. Navigate to **Settings > General** and copy the API key. Add it to `.env` as `SONARR_API_KEY`.
4. Add qBittorrent as a download client (same as Radarr above, category: `tv`).
5. Enable **Hardlinks** under **Settings > Media Management**.

After updating both API keys, restart the stack to apply them to Unpackerr:

```bash
docker compose up -d
```

### 4. Prowlarr — Indexer Setup

1. Open `http://<host>:9696`.
2. Add your indexers under **Indexers > Add Indexer**.
3. If any indexers are behind Cloudflare, add FlareSolverr as a proxy:
   - Go to **Settings > Indexers > Add Proxy**
   - Type: `FlareSolverr`
   - URL: `http://flaresolverr:8191`
4. Connect Prowlarr to Radarr and Sonarr under **Settings > Apps**:
   - Radarr: `http://radarr:7878`, API key from `.env`
   - Sonarr: `http://sonarr:8989`, API key from `.env`

Prowlarr will automatically sync indexers to both apps.

### 5. Jellyfin — Add Media Libraries

1. Open `http://<host>:8096` and complete the setup wizard.
2. Add two libraries:
   - **Movies** — folder path `/data/movies`
   - **TV Shows** — folder path `/data/tv`
3. Run a library scan. Jellyfin will pick up anything already in those directories.

---

## Verifying the VPN

To confirm qBittorrent traffic is routed through the VPN:

```bash
docker exec -it qbittorrent curl -s https://ipinfo.io
```

The returned IP should belong to ProtonVPN, not your ISP.

---

## Updating Containers

Watchtower is configured with `--label-enable`, meaning it will only auto-update containers that have the label `com.centurylinklabs.watchtower.enable=true`. All relevant containers in this stack have that label. Watchtower checks for updates once per day by default and cleans up old images automatically.

To trigger an immediate update manually:

```bash
docker compose pull && docker compose up -d
```

---

## Troubleshooting

**qBittorrent won't start** — Gluetun must be healthy first. Check `docker logs gluetun` for VPN errors. Common causes are an invalid `WIREGUARD_PRIVATE_KEY` or an unsupported server.

**Radarr/Sonarr can't reach qBittorrent** — Ensure the WebUI authentication bypass is configured as described above. The containers communicate over Docker's internal network.

**Indexers aren't working** — Check Prowlarr's indexer status page. If a Cloudflare-protected indexer is failing, confirm FlareSolverr is running (`docker ps`) and the proxy is configured in Prowlarr.

**Downloads not moving to the library** — Confirm hardlinks are enabled in Radarr/Sonarr and that both `torrents/` and the destination library folder are under the same `DATA_DIR` mount. Hardlinks do not work across different filesystems.

**Port forwarding not updating in qBittorrent** — Check `docker logs qbittorrent` for GSP mod output. Verify `GSP_GTN_API_KEY` in `.env` matches `apikey` in `gluetun/config.toml` exactly.