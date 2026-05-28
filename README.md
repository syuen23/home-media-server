# home-media-server

Self-hosted media stack on Docker Compose.

## Data model

Configs and media are kept on separate roots:

- `${CONFIG_DIR}` (default `/opt/appdata`) — per-app config, small, should be backed up
- `${DATA_ROOT}` (default `/srv/media-server/data`) — media and downloads, typically on a large storage drive

Every container that touches media or downloads mounts `${DATA_ROOT}` as `/data` — same path everywhere so imports are hardlinks instead of copies.

```
/opt/appdata/             # CONFIG_DIR — on OS drive
├── prowlarr/
├── sonarr/
├── radarr/
├── nzbget/
└── plex/

/srv/media-server/data/   # DATA_ROOT — mounted as /data in every container
├── downloads/
│   └── usenet/           # nzbget completes here, bucketed by category
│       ├── tv/
│       └── movies/
└── media/                # final library — Plex reads from here
    ├── tv/
    └── movies/
```

In-app paths to set:

- nzbget MainDir: `/data/downloads/usenet`
- Sonarr root folder: `/data/media/tv` · nzbget category: `tv`
- Radarr root folder: `/data/media/movies` · nzbget category: `movies`
- Plex libraries: `/data/media/tv`, `/data/media/movies`

## First-time setup

### 1. Prerequisites

- [Docker Engine](https://docs.docker.com/engine/install/ubuntu/) installed on Ubuntu
- [Docker Compose plugin](https://docs.docker.com/compose/install/linux/) (`docker compose`, not the older `docker-compose`)
- A Tailscale account and `tailscale` installed on the host (`tailscale ip -4` to get your IP)
- A Usenet provider (e.g. Newshosting) and a Usenet indexer account (e.g. NZBGeek)

### 2. Clone the repo

```bash
git clone <repo-url> ~/home-media-server
cd ~/home-media-server
```

### 3. Configure your environment

Find your user ID and group ID:

```bash
id
# uid=1000(simon) gid=1000(simon) ...
```

Copy the example env file and fill in your values:

```bash
cp .env.example .env
```

Key values to set in `.env`:

| Variable                      | How to get it                                                             |
| ----------------------------- | ------------------------------------------------------------------------- |
| `PUID` / `PGID`               | Output of `id` above                                                      |
| `TZ`                          | Your timezone, e.g. `America/New_York`                                    |
| `TAILSCALE_IP`                | Output of `tailscale ip -4`                                               |
| `PLEX_CLAIM`                  | https://plex.tv/claim (expires in 4 min, grab it right before first boot) |
| `NZBGET_USER` / `NZBGET_PASS` | Choose your own credentials for the NZBGet web UI                         |

### 4. Create directories

Pre-create the data directories with the correct ownership. If Docker auto-creates
them they'll be owned by root and the containers (running as `PUID:PGID`) won't
be able to write to them. The linuxserver.io images handle their own config dirs,
so only the data tree and config root need to be created manually.

```bash
# Config root (linuxserver images create their own subdirs on first boot)
sudo mkdir -p /opt/appdata
sudo chown -R 1000:1000 /opt/appdata

# Data tree
sudo mkdir -p /srv/media-server/data/downloads/usenet/{tv,movies}
sudo mkdir -p /srv/media-server/data/media/{tv,movies}
sudo chown -R 1000:1000 /srv/media-server
```

Replace `1000:1000` with your actual `PUID:PGID` if different.

### 5. Start the stack

```bash
docker compose up -d
```

Check that all containers started cleanly:

```bash
docker compose ps
docker compose logs -f   # Ctrl+C to exit
```

### 6. In-app configuration

Configure services in this order so each one can find the service it depends on:

1. **NZBGet** — set MainDir to `/data/downloads/usenet`. Create categories `tv` and `movies`
   with DestDir `${MainDir}/${Category}` (gives `/data/downloads/usenet/tv` etc).

2. **Prowlarr** — add your Usenet indexers, then connect them to Sonarr and Radarr
   via _Settings → Apps_.

3. **Sonarr** — add NZBGet as a download client (category: `tv`). Set root folder
   to `/data/media/tv`.

4. **Radarr** — same as Sonarr but category `movies`, root folder `/data/media/movies`.

5. **Plex** — claim the server using your `PLEX_CLAIM` token, then add libraries:
    - TV Shows → `/data/media/tv`
    - Movies → `/data/media/movies`

6. **Seerr** — connect to your Plex server and your Sonarr/Radarr instances so
   users can request content.

After this, \*arr imports are instant hardlinks — the file appears in
`/data/media/...` without duplicating disk space.

## Things to add

- buildarr for automating configuration/setup

- tailscale for backend service access
- proxy for remote server access

- tautulli for monitoring media server

- jellyfin server?

- separate sonarr/radarr instances for 4k searches
