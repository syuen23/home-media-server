# home-media-server

Self-hosted media stack on Docker Compose.

## Data model

Single root at `${DATA_ROOT}` (default `/mnt/data`). Every container that touches
media or downloads mounts `${DATA_ROOT}/data` as `/data` — same path everywhere
so imports are hardlinks instead of copies.

```
${DATA_ROOT}/
├── config/           # per-app configs (sqlite, logs, plex library)
│   ├── prowlarr/
│   ├── sonarr/
│   ├── radarr/
│   ├── nzbget/
│   └── plex/
└── data/             # mounted as /data in every container
    ├── usenet/       # nzbget completes here, bucketed by category
    │   ├── tv/
    │   └── movies/
    ├── torrents/     # reserved for future qBittorrent
    │   ├── tv/
    │   └── movies/
    └── media/        # final library — Plex reads from here
        ├── tv/
        └── movies/
```

In-app paths to set:

- nzbget MainDir: `/data/usenet`
- Sonarr root folder: `/data/media/tv` · nzbget category: `tv`
- Radarr root folder: `/data/media/movies` · nzbget category: `movies`
- Plex libraries: `/data/media/tv`, `/data/media/movies`

## First-time setup

### Before `docker compose up`

Pre-create the host directories so the bind mounts land with the right
ownership. If Docker auto-creates them, they'll be owned by root and the
containers (running as `PUID:PGID`) won't be able to write to them.

```bash
sudo mkdir -p /mnt/data/{config/{prowlarr,sonarr,radarr,nzbget,plex},data/{usenet/{tv,movies},torrents/{tv,movies},media/{tv,movies}}}
sudo chown -R 1000:300 /mnt/data    # match PUID:PGID from .env
```

Then copy `.env.example` to `.env` and fill in the secrets (`PLEX_CLAIM`,
`NZBGET_USER`/`PASS`, `TAILSCALE_IP`).

### In-app configuration order

Bring the stack up (`docker compose up -d`) and configure in this order so each
service can find the one it depends on:

1. **nzbget** — set MainDir to `/data/usenet`. Create categories `tv` and `movies`
   (DestDir = `${MainDir}/${Category}`, which gives `/data/usenet/tv` and
   `/data/usenet/movies`).
2. **Prowlarr** — add indexers, then push them to Sonarr/Radarr via _Settings →
   Apps_.
3. **Sonarr** — add nzbget as a download client with category `tv`. Add root
   folder `/data/media/tv`.
4. **Radarr** — same, category `movies`, root folder `/data/media/movies`.
5. **Plex** — claim the server (one-time `PLEX_CLAIM` token from
   https://plex.tv/claim, valid 4 min), then add libraries pointed at
   `/data/media/tv` and `/data/media/movies`.

After this, \*arr imports are instant hardlinks — the file appears in
`/data/media/...` without duplicating disk space.

## Areas for improvement

- wrap download layer within Gluetun container -> vpn requirement for downloads

- implement logging

- build in non-root containerization? extra security layer
