# Security

## Threat model summary

This is a home DNS filter + remote access setup, not an internet-facing production service. The security goals are:

1. Never expose Pi-hole's admin UI or DNS service directly to the public internet.
2. Keep remote access to the Pi limited to authenticated tailnet members.
3. Keep secrets (passwords, keys) out of version control.

## Secrets management

- Real configuration values live in a local `.env` file per service folder, which is **git-ignored** (see [`.gitignore`](../.gitignore)).
- [`services/pihole/.env.example`](../services/pihole/.env.example) documents the required Pi-hole variables (`YourTimeZone`, `FTLCONF_webserver_api_password`) without real values, so the repo can be shared/published safely.
- [`services/invidious/.env.example`](../services/invidious/.env.example) documents the required Invidious variables (`INVIDIOUS_HMAC_KEY`, `INVIDIOUS_COMPANION_KEY`, `INVIDIOUS_TAILNET_DOMAIN`), same rationale.
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

## Invidious-specific considerations

- **Network exposure**: Invidious is published only through `tailscale serve` (tailnet-only, HTTPS with a Tailscale-issued cert). It is never bound to the LAN interface or forwarded through the router — the only way to reach it is tailnet membership, same trust boundary as the Pi-hole admin UI.
- **Secrets**: `INVIDIOUS_HMAC_KEY` and `INVIDIOUS_COMPANION_KEY` are random values generated with `openssl rand`, kept only in the git-ignored `.env` — never commit real values.
- **Container hardening**: `invidious-companion` runs with `cap_drop: [ALL]`, a read-only root filesystem, and `no-new-privileges` — it only gets a single writable cache directory.
- **Separate stack from Pi-hole**: kept in its own Compose project specifically so Invidious maintenance/failures can't affect DNS — see [architecture.md](./architecture.md).
- **Disk hygiene**: Invidious's `videos` cache table grows unbounded with use, and this Pi's disk headroom is limited. A weekly cron truncates it (see [services/invidious/README.md](../services/invidious/README.md#disk-cache-maintenance)) — this is disposable cache data, not user accounts/history.
- **Legal/ToS note**: Invidious fetches YouTube content outside YouTube's official API, which is a documented gray area under YouTube's Terms of Service. This instance is kept private and tailnet-only — never exposed publicly or listed as a public Invidious instance — to keep the footprint as low-risk as reasonably possible.

## Host-level hardening (recommended, not yet automated here)

- Keep the Raspberry Pi OS patched (`apt update && apt upgrade`).
- Disable password SSH login in favor of key-based auth, or restrict SSH to the tailnet only.
- Take periodic backups of the Pi-hole config volumes (`etc-pihole/`, `etc-dnsmasq.d/`) so filtering rules survive a re-flash of the SD card.
