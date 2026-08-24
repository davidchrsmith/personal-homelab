# personal-homelab

A Raspberry Pi-based personal homelab providing network-wide DNS filtering ([Pi-hole](https://pi-hole.net/)) and secure remote access ([Tailscale](https://tailscale.com/)). Pi-hole runs as a Docker container; Tailscale runs natively on the host and is configured to route the whole tailnet's DNS through Pi-hole, so ad/tracker blocking follows your devices wherever they are.

![Network architecture](./diagrams/network-architecture.png)

## Services

| Service | How it runs | Docs |
|---|---|---|
| [Pi-hole](./services/pihole/README.md) | Docker container ([`docker-compose.yaml`](./docker-compose.yaml)) | Network-wide DNS filtering + admin UI |
| [Tailscale](./services/tailscale/README.md) | Native systemd service | Private mesh VPN, tailnet-wide DNS |

## Documentation

- [Architecture](./docs/architecture.md) — components, diagram, and request flow
- [Networking](./docs/networking.md) — ports, IPs, and DNS resolution paths
- [Security](./docs/security.md) — secrets, exposure, and hardening notes

## Getting started

```bash
git clone <this-repo>
cd personal-homelab
cp .env.example .env   # fill in your timezone and Pi-hole admin password
docker compose up -d   # start Pi-hole

# install Tailscale natively and join the tailnet
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Once Pi-hole is running, view the admin dashboard at `http://<raspberry-pi-ip>:8080/admin` (plain HTTP, no TLS is configured — reachable from the LAN or the tailnet only).

See the individual service docs above for configuration details, ports, and how DNS is wired between the LAN and the tailnet.
