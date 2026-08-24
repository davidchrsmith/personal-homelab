# Pi-hole

Network-wide DNS-based ad/tracker blocker, running as a Docker container defined in the repo's root [`docker-compose.yaml`](../../docker-compose.yaml).

## What it does

Pi-hole acts as the DNS server for the home LAN (and, via Tailscale, the tailnet — see [networking.md](../../docs/networking.md)). It answers DNS queries directly, sinkholing any domain that matches its blocklists and forwarding everything else to an upstream resolver.

## Configuration

Environment variables (set in a local `.env` file, based on [`.env.example`](../../.env.example)):

| Variable | Purpose |
|---|---|
| `YourTimeZone` | Container timezone (e.g. `America/New_York`), used for accurate logs/stats |
| `FTLCONF_webserver_api_password` | Password for the Pi-hole admin web UI |

`FTLCONF_dns_listeningMode` is set to `all` in `docker-compose.yaml` so Pi-hole accepts queries from every network interface on the Pi (LAN + Tailscale), not just localhost.

## Ports

| Port | Purpose |
|---|---|
| `53/tcp`, `53/udp` | DNS queries |
| `8080/tcp` → `80` | Admin web UI |

## Volumes

| Host path | Container path | Purpose |
|---|---|---|
| `./etc-pihole` | `/etc/pihole` | Pi-hole config, blocklists, query logs |
| `./etc-dnsmasq.d` | `/etc/dnsmasq.d` | Custom DNS/dnsmasq configuration |

These are bind-mounted so Pi-hole's configuration and stats persist across container restarts/updates.

## Usage

```bash
# copy the example env file and fill in real values
cp .env.example .env

# start Pi-hole
docker compose up -d

# view logs
docker compose logs -f pihole

# update to the latest image
docker compose pull pihole && docker compose up -d pihole
```

Admin UI: `http://<pi-lan-or-tailnet-ip>:8080/admin`

## Setting devices to use it

- **LAN devices**: point their DNS (or your router's DHCP DNS setting) at the Pi's LAN IP.
- **Tailnet devices**: handled globally via the Tailscale DNS setting — see [tailscale/README.md](../tailscale/README.md) and [docs/networking.md](../../docs/networking.md).
