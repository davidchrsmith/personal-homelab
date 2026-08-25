# Invidious

Self-hosted, ad-free front-end for YouTube, running as its own Docker Compose stack, reachable only over the tailnet.

## Why this exists

Pi-hole can't block YouTube ads — ads and video are served from the same domains, and increasingly stitched directly into the stream server-side, so there's nothing DNS-level to sinkhole without also breaking playback. Invidious sidesteps the problem entirely: it fetches video data itself and serves a clean page with no ads and no tracking.

## Why it's a separate compose stack from Pi-hole

Pi-hole is critical infrastructure — it's the DNS resolver for the whole LAN and tailnet. Invidious is not. Keeping them in separate folders/compose projects means updating, restarting, or troubleshooting Invidious can never accidentally affect DNS for the household. See [`services/pihole/`](../pihole/) for the equivalent, independent Pi-hole stack.

## Architecture

Three containers, all defined in this folder's [`docker-compose.yaml`](./docker-compose.yaml):

| Container | Purpose |
|---|---|
| `invidious` | The web front-end itself |
| `invidious-companion` | Fetches actual video/audio streams from YouTube on Invidious's behalf (required for playback to work at all) |
| `invidious-db` | Postgres — stores accounts/subscriptions plus a disposable video/channel metadata cache |

## Configuration

Environment variables (set in a local `.env` file, based on [`.env.example`](./.env.example)):

| Variable | Purpose |
|---|---|
| `INVIDIOUS_HMAC_KEY` | Signs CSRF tokens/cookies. Any random string — generate with `openssl rand -hex 20` |
| `INVIDIOUS_COMPANION_KEY` | Shared secret between Invidious and its companion. **Must be exactly 16 characters** — generate with `openssl rand -hex 8` |
| `INVIDIOUS_TAILNET_DOMAIN` | The tailnet hostname this instance is served at (see [Exposing it on the tailnet](#exposing-it-on-the-tailnet) below) |

## One-time setup: pulling Postgres init files

The `invidious-db` container needs Invidious's SQL migration files and init script, which aren't part of this repo (they come from upstream):

```bash
cd services/invidious
git clone --depth 1 https://github.com/iv-org/invidious.git /tmp/invidious-src
cp -r /tmp/invidious-src/config/sql config/sql
cp /tmp/invidious-src/docker/init-invidious-db.sh init-invidious-db.sh
rm -rf /tmp/invidious-src
```

## Usage

```bash
cd services/invidious
cp .env.example .env   # fill in real secrets — see Configuration above

docker compose up -d
docker compose logs -f invidious

# update
docker compose pull && docker compose up -d
```

`pgdata/` and `companioncache/` are created automatically on first run and are git-ignored.

![Invidious homepage](../../screenshots/invidious.png)

## Exposing it on the tailnet

Tailscale runs natively on the Pi (see [`services/tailscale/`](../tailscale/)), so Invidious is published using `tailscale serve` rather than any port forwarding or public exposure:

```bash
sudo tailscale serve --bg https / http://127.0.0.1:3000
tailscale serve status   # prints the https://<hostname>.<tailnet>.ts.net URL
```

Set `INVIDIOUS_TAILNET_DOMAIN` in `.env` to that hostname (no `https://` prefix), then `docker compose up -d invidious` to pick it up. It is **not** reachable from the LAN or the public internet — tailnet membership only.

## Disk/cache maintenance

Invidious's `videos` table is a disposable cache (metadata re-fetched from YouTube on demand, not authoritative data) that grows unbounded with regular use. Given limited disk headroom on the Pi, back up Invidious's built-in auto-cleanup with a weekly cron job:

```bash
crontab -e
```
```
@weekly docker exec invidious-db psql -U kemal -d invidious -c "DELETE FROM nonces * WHERE expire < current_timestamp" > /dev/null
@weekly docker exec invidious-db psql -U kemal -d invidious -c "TRUNCATE TABLE videos" > /dev/null
0 4 * * * cd ~/invidious && docker compose restart invidious
```

Truncating `videos` never loses accounts, subscriptions, or watch history (those live in the `users` table) — it just means the next view of a given video re-fetches its metadata from YouTube.

Check disk usage periodically, especially in the first few weeks:
```bash
df -h /
```

## Resource requirements

- ~2GB free RAM (mostly Postgres + companion), restarted daily for stability
- At least 20GB disk headroom is recommended upstream — tight on a stock Pi SD card. Monitor `df -h /` and rely on the cache-truncation cron above rather than adding small/slow USB flash storage (see [docs/security.md](../../docs/security.md) for why).

## Limitations

- Browser-only (or apps that speak the Invidious API, like Clipious on Android) — it does not remove ads from the native YouTube app on phones or Smart TVs.
- Google periodically changes anti-bot measures; if playback breaks, check [docs.invidious.io](https://docs.invidious.io/) for companion/config updates.
- Self-hosting Invidious fetches YouTube content outside its official API, which sits in a legal gray area under YouTube's Terms of Service — see [docs/security.md](../../docs/security.md#invidious-specific-considerations) for how that's handled here (private, tailnet-only, never publicized as a public instance).
