# hermes-mobile-remote-access

A [Hermes Agent](https://github.com/NousResearch/hermes-agent) skill for connecting the Hermes mobile app (or any browser) to a Hermes gateway on another machine.

[Tailscale](https://tailscale.com) is the recommended route, but it isn't required: the skill also covers your home network, other VPNs and reverse proxies/tunnels.

It covers two cases:

- **First-time setup**: turn on the gateway's Webapp and reach it from your phone. With Tailscale, `tailscale serve` gives you `https://<machine>.<tailnet>.ts.net`, with no firewall rules and no ports open to the internet.
- **Migrating off `hermes dashboard` / `hermes serve`**: the old port-9119 backend is being deprecated. The skill moves you to the gateway Webapp without downtime, keeps your existing login, and retires the old process.

## Network options

| Route | Phone connects to | Good for |
|---|---|---|
| Tailscale (`tailscale serve`) | `https://<machine>.<tailnet>.ts.net` | Recommended: HTTPS and a stable name, works anywhere, nothing exposed publicly |
| Home Wi-Fi / LAN | `http://<LAN-IP>:9120` | Home only; needs a firewall rule; unencrypted |
| Other VPN (WireGuard, ZeroTier…) | `http://<VPN-IP>:9120` | Same as LAN, over your own VPN |
| Reverse proxy / tunnel (Caddy, nginx, Cloudflare Tunnel…) | `https://<your-hostname>` | Your own domain; use OAuth sign-in if it's reachable from the internet |

## Gotchas it covers

- Tailscale stuck "starting" after a reboot, and Tailscale restarts dropping the `serve` config
- The gateway only starting at Windows logon, not at boot
- `webapp.host` rejecting specific IPs; `dashboard.public_url` not applying to the Webapp
- Firewall rules, Public vs Private networks on Windows, and DHCP address changes for LAN setups
- Reverse proxies: `public_url` host matching, `dashboard.trusted_proxies`, and WebSocket upgrades
- The old backend: Windows Firewall silently blocking Hermes's bundled Python, and dying when the desktop app closes
- Setting a dashboard password (there's no CLI command for it)
- "gateway error 4001" when reopening a session
- SSH phone apps on Windows: `sshd` startup type and where admin accounts' keys go

## Install

Copy the skill folder into your Hermes skills directory:

```bash
git clone https://github.com/tonbistudio/hermes-mobile-remote-access
cp -r hermes-mobile-remote-access/skills/hermes-mobile-remote-access ~/.hermes/skills/
```

On Windows the skills directory is `%LOCALAPPDATA%\hermes\skills\`. Start a new session (or run `/reload-skills`) and Hermes loads it whenever you ask about connecting your phone.

## Use

Ask Hermes things like:

- "Set up the Hermes mobile app to connect to this PC over Tailscale."
- "Let my phone reach Hermes on my home Wi-Fi."
- "Migrate my dashboard setup to the gateway webapp."
- "The mobile app can't connect. What's wrong?"
- "My SSH app wants a server, username and port."

## Requirements

- Hermes Agent with the gateway Webapp (`hermes webapp --help` should work).
- A dashboard sign-in provider (username/password, or OAuth for anything internet-facing).
- For the Tailscale route: Tailscale on the host and the phone, on the same tailnet, with MagicDNS and HTTPS certificates enabled in the admin console.

## Security

- The recommended setup is **tailnet-only** (`tailscale serve`). The skill tells Hermes never to use `tailscale funnel` or port-forward the Webapp on your router.
- Username/password sign-in is meant for trusted networks. Anything reachable from the internet should use OAuth or an access layer in front.
- Hermes never asks for your password in chat; it gives you a script to set it yourself.

## License

MIT
