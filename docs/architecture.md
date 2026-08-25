# Architecture

This homelab runs on a single Raspberry Pi and provides two core functions: network-wide DNS filtering (Pi-hole) and secure remote access (Tailscale). The two services are linked so that ad-blocking DNS filtering applies not just to devices on the home LAN, but to any device on the tailnet, wherever it is.

## Components

| Component | Runs as | Purpose |
|---|---|---|
| Pi-hole | Docker container (`docker-compose.yaml`) | DNS resolver / sinkhole that blocks ads and trackers for all clients that use it |
| Tailscale | Native systemd service (installed directly on the Pi's OS, not containerized) | Creates a private mesh VPN (tailnet) so remote devices can securely reach the Pi and use it as their DNS server |

Tailscale runs natively rather than in Docker because it needs to manage host networking/routing directly (WireGuard interface, IP forwarding), which is simpler and more reliable outside a container.

## High-level diagram

![Network architecture](../diagrams/network-architecture.png)

Editable source: [diagrams/network-architecture.drawio](../diagrams/network-architecture.drawio) (open with [draw.io](https://app.diagrams.net/) / the [Draw.io Integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio) VS Code extension).

```mermaid
flowchart LR
    subgraph Home LAN
        Pi[Raspberry Pi]
        Pihole[(Pi-hole<br/>Docker container)]
        Router[Home Router]
        LANDevice[LAN Device]
    end

    subgraph Tailnet
        TSD[Tailscale daemon<br/>native service]
        Remote[Remote Device<br/>e.g. phone, laptop]
    end

    LANDevice -- DNS queries --> Pihole
    Pi --- Pihole
    Pi --- TSD
    Remote -- WireGuard tunnel --> TSD
    Remote -- DNS queries via tailnet --> Pihole
    Pihole -- upstream DNS --> Internet[(Upstream DNS / Internet)]
    Router --- Internet
```

## Request flow

1. **LAN client** sends a DNS query to the Pi's LAN IP (Pi-hole listens on port 53).
2. **Remote/tailnet client** connects to the tailnet over Tailscale's encrypted WireGuard tunnel, and is configured (via Tailscale's DNS settings) to use the Pi as its DNS server too.
3. Pi-hole checks the query against its blocklists:
   - If blocked, it returns a sinkholed response (ad/tracker blocked).
   - If allowed, it forwards the query to the configured upstream DNS resolver and returns the result.
4. All other traffic (actual web/app traffic) flows normally over the LAN or internet; only DNS resolution is routed through Pi-hole.

## Why this design

- **Single point of DNS control**: One Pi-hole instance filters ads/trackers for both local and remote devices, instead of maintaining separate configurations per network.
- **No exposed services**: Tailscale means the Pi-hole admin UI and the Pi itself don't need to be exposed to the public internet — access is only possible through the private tailnet or the local LAN.
- **Simple, low-power hardware**: A Raspberry Pi is enough to run both workloads since DNS filtering and VPN traffic are lightweight.

See [networking.md](./networking.md) for IP/port details and [security.md](./security.md) for the security model.
