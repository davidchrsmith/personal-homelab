# Security

## Threat model summary

This is a home DNS filter + remote access setup, not an internet-facing production service. The security goals are:

1. Never expose Pi-hole's admin UI or DNS service directly to the public internet.
2. Keep remote access to the Pi limited to authenticated tailnet members.
3. Keep secrets (passwords, keys) out of version control.

## Secrets management

- Real configuration values live in a local `.env` file, which is **git-ignored** (see [`.gitignore`](../.gitignore)).
- [`.env.example`](../.env.example) documents the required variables (`YourTimeZone`, `FTLCONF_webserver_api_password`) without real values, so the repo can be shared/published safely.
- The Pi-hole admin password is set via `FTLCONF_webserver_api_password` and should be a strong, unique value — never reuse a password from another service.

## Network exposure

- Pi-hole's DNS (53) and admin UI (8080→80) ports are only reachable from the LAN and the tailnet (`tailscale0` interface) — no port forwarding is configured on the home router for these.
- Because Tailscale provides the only remote access path, there is no publicly routable way to reach the Pi's services; an attacker would need to compromise a device already authenticated into the tailnet.

## Tailscale-specific considerations

- Each device must individually authenticate to join the tailnet (via the identity provider configured on the Tailscale account — e.g. Google/GitHub/Microsoft SSO).
- Review the [Tailscale admin console](https://login.tailscale.com/admin/machines) periodically and remove devices that are no longer in use.
- Consider enabling [key expiry](https://tailscale.com/kb/1028/key-expiry) for devices so lost/stolen devices don't retain indefinite access, and use [ACLs](https://tailscale.com/kb/1018/acls/) if the tailnet grows beyond a fully-trusted set of personal devices.
- Since Pi-hole is used as the tailnet's global DNS nameserver, any device on the tailnet can send it DNS queries — keep the tailnet membership limited to trusted personal devices.

## Pi-hole-specific considerations

- Keep the Pi-hole image up to date (`docker compose pull && docker compose up -d`) to receive security and blocklist-engine fixes.
- The admin UI password is the only thing standing between a LAN/tailnet user and Pi-hole's settings — treat it like any other admin credential.
- Blocklists are just DNS-level filtering; they are not a substitute for endpoint security or a firewall.

## Host-level hardening (recommended, not yet automated here)

- Keep the Raspberry Pi OS patched (`apt update && apt upgrade`).
- Disable password SSH login in favor of key-based auth, or restrict SSH to the tailnet only.
- Take periodic backups of the Pi-hole config volumes (`etc-pihole/`, `etc-dnsmasq.d/`) so filtering rules survive a re-flash of the SD card.
