# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Config-only repo for [glanceapp/glance](https://github.com/glanceapp/glance) — a self-hosted dashboard. The single file `config/glance.yml` renders the homepage for `unserver23.net`, a home lab. There is no application code, build step, linter, or test suite; the "build" is Glance parsing this YAML at container start.

## Running it

Glance is run as a Docker container on the Unraid host (`10.23.23.100`), mounting `config/` in as `/app/config`:

```
docker run -d --name glance -p 8085:8080 \
  -e UNRAID_IP=10.23.23.X -e HA_IP=10.23.23.X \
  -e ADGUARD_USER=... -e ADGUARD_PASSWORD=... \
  -v /mnt/user/appdata/glance:/app/config \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  glanceapp/glance
```

To validate a config edit, restart the container and check its logs — Glance fails fast on invalid YAML/schema.

## Structure of `config/glance.yml`

- `server` / `theme` / `branding`: global Glance settings (port 8080 behind the reverse proxy, dark green theme, branded as "unserver23").
- `define: &check`: a YAML anchor (`timeout: 5s`, `allow-insecure: true`) merged into every `monitor` site via `<<: *check`. Add new monitored sites the same way instead of repeating those two keys.
- `pages`: two pages, **Home** and **Containers**, each with column groups of widgets (`clock`, `weather`, `dns-stats`, `server-stats`, `search`, `monitor`, `bookmarks`, `releases`, `reddit`, `docker-containers`).
- The **Home** page's `monitor` widgets are grouped by purpose (Media, Home, AI & Files, Infra) — each site pairs a public `url` (the `*.unserver23.net` Cloudflare-tunneled hostname) with an internal `check-url` that Glance polls directly (LAN IP + service port) to determine up/down status.

## Editing conventions

- Services live behind Cloudflare Tunnel records; every public `url` should resolve as `https://<subdomain>.unserver23.net` even though `check-url` hits the LAN IP directly.
- `check-url` ports are best-guess defaults per app — verify against the actual container's port mapping when adding a service, since a wrong port shows the site as down even when it's reachable. Also check for collisions: two `check-url`s hitting the same host:port (e.g. both on `10.23.23.100:8080`) means at least one is wrong, since only one process can own that port.
- Icons use `si:<slug>` (Simple Icons) when the slug exists there; niche self-hosted apps (Lidarr, Prowlarr, Overseerr, etc.) often aren't in Simple Icons and silently render blank — use `sh:<slug>` (selfh.st icons) or `di:<slug>` (Dashboard Icons) instead. Verify the slug exists in the target library before committing.
- Secrets (`ADGUARD_USER`, `ADGUARD_PASSWORD`) and host IPs (`UNRAID_IP`, `HA_IP`) are injected via env vars referenced as `${VAR}` — never hardcode credentials or IPs for these into the YAML.
- Keep new monitor entries inside the correct group (Media / Home / AI & Files / Infra) rather than creating new groups, unless the entry doesn't fit any existing category.
