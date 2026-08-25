# Tailscale

Provides secure remote access into the homelab via a private WireGuard-based mesh network (tailnet). Installed as a **native systemd service** on the Raspberry Pi's OS — not run in Docker — so it can manage host networking directly.

## Why native instead of Docker

Tailscale needs to create and manage a network interface (`tailscale0`) and handle routing on the host. Running it natively avoids the extra privileges/capabilities a container would need for the same behavior, and keeps it running independently of the Docker Compose stack.

## Installation

```bash
# install (official script, adds Tailscale's apt repo)
curl -fsSL https://tailscale.com/install.sh | sh

# authenticate and bring the interface up
sudo tailscale up
```

Follow the printed login link to authenticate the Pi against the tailnet.

## Role in this homelab

Tailscale's main job here is to let the Pi's Pi-hole instance act as the **DNS server for the whole tailnet**, not just the LAN:

1. In the [Tailscale admin console](https://login.tailscale.com/admin/dns), the Pi's tailnet IP (`100.x.x.x`) is added as a **global nameserver**.
2. Every device that joins the tailnet then automatically uses Pi-hole for DNS resolution, wherever it physically is — getting the same ad/tracker blocking as devices on the home LAN.

See [docs/networking.md](../../docs/networking.md) for the full DNS request flow and [docs/architecture.md](../../docs/architecture.md) for how this fits into the overall design.

![Tailscale admin console](../../screenshots/tailscale-dash.png)

## Useful commands

```bash
# check status / connected devices
tailscale status

# check the Pi's tailnet IP
tailscale ip -4

# check connectivity/DNS config
tailscale netcheck

# log out / disconnect
sudo tailscale down
```

## Managing devices

Add/remove/authorize devices from the [Tailscale admin console](https://login.tailscale.com/admin/machines). See [docs/security.md](../../docs/security.md) for recommended hardening (key expiry, ACLs, periodic device review).
