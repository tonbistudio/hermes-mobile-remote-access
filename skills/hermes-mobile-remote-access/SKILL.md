---
name: hermes-mobile-remote-access
description: Use when connecting phone apps to a remote Hermes.
version: 2.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [tailscale, mobile, webapp, gateway, dashboard, serve, reverse-proxy, lan, ssh, windows, linux]
    related_skills: [hermes-agent]
---

# Hermes mobile remote access (gateway Webapp)

Tailscale is the recommended network and the default path below. Nothing in Hermes requires it: see "Without Tailscale" for LAN, other VPNs and reverse proxies/tunnels.

## When to Use
- **First-time setup:** a user wants the Hermes mobile app (or a phone/laptop browser) to reach Hermes on another machine.
- **Migration:** a user connects through `hermes dashboard` or `hermes serve` on port 9119. That path is being deprecated; move them to the gateway Webapp.
- The phone can't connect, the app asks for "server / username / password", or a session fails with "gateway error 4001".
- An SSH phone app (e.g. herdr-mobile) asks for server / username / port, or a phone key needs authorizing.

Answer the literal question first ("server is X, username Y"), then fix. Users doing this are usually mid-task on their phone and impatient.

## How it fits together
- **Gateway Webapp (current):** the always-on gateway (`hermes gateway`) serves the Webapp on its own listener, configured by `webapp.*` in `config.yaml` (default `127.0.0.1:9120`). `tailscale serve` puts tailnet-only HTTPS in front of it. Sign-in uses the dashboard auth provider (username/password or OAuth).
- **Old path (deprecated):** a separate `hermes dashboard` or `hermes serve` process on `--host <ip> --port 9119`, kept alive by a scheduled task / systemd unit. Extra process to supervise, plus firewall rules and port numbers in the phone app.
- Both use the **same sign-in**: `dashboard.basic_auth.*` in `config.yaml`, or the `HERMES_DASHBOARD_BASIC_AUTH_USERNAME` / `_PASSWORD` / `_PASSWORD_HASH` / `_SECRET` env vars in `.env`. Migrating keeps the user's existing username/password.

## Prerequisites (both flows)
1. **Tailscale** on the host and the phone, same tailnet, both connected. `tailscale status` should list the phone. `tailscale ip -4` gives the host's `100.x` address.
2. **MagicDNS and HTTPS certificates** turned on in the Tailscale admin console (DNS page). `tailscale serve` needs them for the `https://<machine>.<tailnet>.ts.net` name. Get the exact name from `tailscale status --json` (`Self.DNSName`, drop the trailing dot) or the output of `tailscale serve status`.
3. **Gateway installed as a service** so it survives reboots: `hermes gateway install` / `hermes gateway status`. Windows: a Startup-folder login item, so it starts at **user logon**, not at boot. Linux: user systemd unit; run `sudo loginctl enable-linger $USER` or it dies at logout.
4. **A sign-in provider.** Check with `hermes config get dashboard.basic_auth` (show `enabled`/`username` only; never print the hash or secret). If missing, set one up (see "Setting the password").

## First-time setup (new users)
1. Do the prerequisites. Back up `config.yaml`.
2. Configure the Webapp:
   ```bash
   hermes config set webapp.enabled true
   hermes config set webapp.public_url https://<machine>.<tailnet>.ts.net
   ```
   Leave `webapp.host` at `127.0.0.1` and `webapp.port` at `9120`. `public_url` is what turns sign-in on for a loopback bind and is the only outside hostname the Webapp accepts.
3. Expose it to the tailnet only: `tailscale serve --bg 9120`. Linux may need `sudo`, or once `sudo tailscale set --operator=$USER`. **Never `tailscale funnel`**: that publishes it to the internet, and username/password auth is for trusted networks only.
4. Restart the gateway: `hermes gateway restart`. If you are an agent running inside that gateway, see "Restarting from inside the gateway".
5. Verify (next section).
6. Phone: in the Hermes mobile app enter **server `https://<machine>.<tailnet>.ts.net`** (no port) and the dashboard **username/password**. The username is the dashboard one, not the OS account or the Tailscale login.

## Migrating from `hermes dashboard` / `hermes serve` (port 9119)
1. Find what runs the old backend, so it doesn't come back:
   - `hermes dashboard --status` / `hermes serve --status`
   - Windows: Task Scheduler entries (`Get-ScheduledTask | ? TaskName -match hermes`), Startup folder, wrapper scripts.
   - Linux: `systemctl --user list-units | grep -i hermes` (often `hermes-dashboard.service` running `... dashboard --host 0.0.0.0 --port 9119`).
2. Note the sign-in it uses (`dashboard.basic_auth` in config, or `HERMES_DASHBOARD_BASIC_AUTH_*` in `.env`). The gateway reads the same settings, so nothing changes for the user's login. If the creds were only passed to the old process's environment (e.g. a systemd `Environment=` line), move them into `.env` first.
3. Do the **First-time setup** steps 2-5 (enable the Webapp, `tailscale serve --bg 9120`, restart the gateway, verify) *while the old backend is still running*. They don't share a port, so the user is never cut off.
4. Retire the old backend: disable its scheduled task or `systemctl --user disable --now <unit>`, then `hermes dashboard --stop` (or kill the 9119 listener). Confirm nothing listens on 9119.
5. Clean up leftovers, after asking: firewall rules for 9119, old wrapper scripts, an old `tailscale serve`/`funnel` entry pointing at 9119.
6. Phone: **edit or re-add the saved connection**. The old one points at `http://100.x.x.x:9119`; the new server is `https://<machine>.<tailnet>.ts.net`, no port. Same username/password. Expect one sign-in.

## Verifying
- `hermes webapp --status` → `Serving at http://127.0.0.1:9120`. Anything else explains why (no auth provider, port taken, bad `public_url`).
- `tailscale serve status` → `https://<host>.ts.net (tailnet only)` with `/ proxy http://127.0.0.1:9120`.
- `curl -s https://<host>.ts.net/api/status` → JSON with `auth_required: true` and `auth_providers` containing `"basic"`; `gateway_state: running`.
- `curl -s -o /dev/null -w "%{http_code}" https://<host>.ts.net/` → `302` (redirect to sign-in).
- These checks prove the server side only. Have the user confirm on the phone and ask for the exact error text if it fails.

## Without Tailscale
The Webapp is plain HTTP(S) on `webapp.host:webapp.port`; any network that reaches it works. Pick by how exposed it is:

| Route | Config | Phone server | Sign-in |
|---|---|---|---|
| Home Wi-Fi / LAN | `webapp.host 0.0.0.0` (or `::`) | `http://<host-LAN-IP>:9120` | Username/password is acceptable on a trusted network |
| Other VPN (WireGuard, ZeroTier, Nebula…) | `webapp.host 0.0.0.0` | `http://<host-VPN-IP>:9120` | Same as LAN |
| Reverse proxy / tunnel (Caddy, nginx, Cloudflare Tunnel…) | keep `webapp.host 127.0.0.1`, proxy to `127.0.0.1:9120`, set `webapp.public_url https://<proxy-hostname>` | `https://<proxy-hostname>` | **OAuth** if internet-reachable; not username/password |

Gotchas for these routes:
- **Wildcard bind = every interface.** `0.0.0.0` listens on LAN, VPN and any other adapter. A wildcard bind turns sign-in on automatically; without a sign-in provider the gateway serves native clients only (`hermes webapp --status` says so).
- **Firewall:** a wildcard bind needs an inbound rule for 9120, scoped to the network you mean. Windows (admin): `New-NetFirewallRule -DisplayName "Hermes Webapp" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 9120 -RemoteAddress <LAN-or-VPN-CIDR>`; Linux: `ufw allow from <CIDR> to any port 9120 proto tcp`. Windows also blocks the network if its profile is Public; set the trusted network to Private.
- **Plain HTTP on LAN/VPN** sends the password and session cookie unencrypted on that network. That's fine for a WireGuard-style VPN (already encrypted), weaker on shared Wi-Fi. For HTTPS, use a reverse proxy.
- **LAN IPs change** (DHCP). Give the host a DHCP reservation, or use its mDNS/LAN hostname.
- **Reverse proxy:** the Webapp only accepts requests for the `webapp.public_url` hostname, and only trusts `X-Forwarded-*` headers from loopback plus the IPs/CIDRs in `dashboard.trusted_proxies`. Add the proxy's address there if it isn't on the same machine. The proxy must pass WebSocket upgrades (chat runs over a WebSocket): nginx needs `proxy_http_version 1.1` plus `Upgrade`/`Connection` headers; Caddy and Cloudflare Tunnel do this by default.
- **Internet exposure:** Hermes's docs reserve username/password for trusted networks. For anything reachable from the internet (Cloudflare Tunnel, port-forward, public proxy), configure an OAuth/OIDC sign-in provider (see the web-dashboard docs) or put an access layer (e.g. Cloudflare Access) in front. Never port-forward 9120 directly on the router.
- Changing `webapp.*` needs a gateway restart.

## Restarting from inside the gateway
An agent chatting through the gateway (desktop app, Webapp, Slack…) dies with it on `hermes gateway restart`. Write a small script that sleeps ~20 s, runs the restart, polls `http://127.0.0.1:9120/api/status`, runs `hermes webapp --status`, and appends results to a log under the Hermes `logs/` dir. Launch it **detached** (Windows: `Start-Process powershell -WindowStyle Hidden -ArgumentList '-File',...`; Linux: `setsid nohup ... &`). Tell the user the chat drops for about a minute and to message you afterwards; then read the log and re-verify.

## Setting the password
There is **no** `hermes dashboard setup` or password subcommand. Options:
- `.env`: `HERMES_DASHBOARD_BASIC_AUTH_USERNAME`, plus `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH` (preferred) or `_PASSWORD`, plus `HERMES_DASHBOARD_BASIC_AUTH_SECRET` (`openssl rand -base64 32`) so sessions survive restarts.
- Config: `hermes config set dashboard.basic_auth.password_hash <hash>` (plus `username`, `enabled true`).
- Make the hash with Hermes's own venv python, with the `hermes-agent` checkout on `sys.path`: `from plugins.dashboard_auth.basic import hash_password`, reading the password via `getpass`. Give the user a script to run themselves; **never take a password in chat** or type it with a tool.
- Restart the gateway after changing auth.

## Gotchas
- **Tailscale stuck "starting" after a reboot** (`tailscale status` → "Tailscale is starting", no `100.x` address): nothing on the host is reachable. The user restarts Tailscale (tray → quit/reopen, or `Restart-Service Tailscale` as admin), re-logging in if needed.
- **Tailscale restarts can drop serve/funnel config.** Re-check `tailscale serve status` after Tailscale problems and re-run `tailscale serve --bg 9120` if it's empty.
- **Windows starts the gateway at logon, not boot.** A rebooted PC sitting at the lock screen serves nothing until someone signs in.
- **No firewall rule needed for the Webapp path.** Tailscale serve forwards over loopback. If you bind `webapp.host 0.0.0.0` instead, you then need a firewall rule and the phone uses `http://<100.x>:9120`, which is worse; prefer serve.
- **`webapp.host` accepts only `127.0.0.1`, `::1`, `localhost`, `0.0.0.0` or `::`.** Any other value, including a specific Tailscale IP, silently keeps the Webapp off. `hermes webapp --status` says why.
- **`dashboard.public_url` does not apply to the Webapp.** Use `webapp.public_url`.
- **Profiles:** the Webapp settings come from the config of the gateway serving the profile (the default profile's for named profiles it serves). A standalone profile with its own gateway needs a different `webapp.port`. Sessions are per profile (`profiles/<name>/state.db`); a session "missing" on the phone may just be in another profile.
- **Port 9120 taken by another program:** the gateway serves native clients only and invalidates Webapp sign-ins; free the port, then sign in again.

### Old dashboard/serve path (when a user still runs it)
- A non-loopback `--host` requires an auth provider; it fails closed at startup. `--insecure` is a no-op.
- Bind to the Tailscale IP rather than `0.0.0.0` to keep it off the LAN.
- **Windows Firewall silently blocks** Hermes's bundled `tools\python-3.x\python.exe`; it has no allow rule. Symptom: works from the host itself, phone times out. Admin fix, tailnet only: `New-NetFirewallRule -DisplayName "Hermes 9119 (Tailscale)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 9119 -RemoteAddress 100.64.0.0/10`.
- **Dies when the desktop app closes** if launched from a PowerShell wrapper loop or an agent's background terminal (Task Scheduler last result `3221225786`, i.e. interrupted). Launch it detached (`Start-Process -WindowStyle Hidden`) from a one-shot check script on a repeating trigger. This fragility is a good reason to migrate.
- PowerShell 5.1 `*>>` logs come out as UTF-16 garbage; pipe through `Out-File -Encoding utf8` and set `PYTHONIOENCODING=utf-8`.
- A pending `hermes update` ("install out of sync", "update is still running") can delay or interrupt startup. Suggest `hermes update`.

## Gateway error 4001 when opening a session
`grep -a "session.resume refused" <hermes>/logs/gateway.log`. The text `invalid_params field=base_url ... must be an http(s) URL` means that session was saved with a `process://...` provider base_url (subscription-CLI providers), which the gateway's resume check rejects. The session's messages are intact, other sessions are unaffected, and new chats work. Options: `hermes update` + gateway restart; edit that session's `model_config` base_url in the profile's `state.db` after a backup; or start a new session and point it at the old one's output files.

## SSH phone apps (server / username / port)
- Server: the Tailscale IP or MagicDNS name. Port: `22`. Username: the **OS account** name (Windows: `whoami` without the domain), not the Tailscale or dashboard login.
- Windows needs OpenSSH Server; Tailscale SSH doesn't run on Windows hosts. It installs as Manual start, so it is dead after reboot until fixed: `Start-Service sshd; Set-Service sshd -StartupType Automatic` (admin). The default rule `OpenSSH-Server-In-TCP` allows the Private profile; Tailscale's adapter is Private.
- **Admin accounts ignore `~\.ssh\authorized_keys`.** Phone public keys go in `C:\ProgramData\ssh\administrators_authorized_keys`, then `icacls <file> /inheritance:r /grant Administrators:F /grant SYSTEM:F`; with looser permissions sshd silently ignores the file and falls back to password. Writing it needs elevation: run a script via `Start-Process -Verb RunAs` (the user clicks UAC) and have it log the result to a file you can read.
- Optional hardening once keys work: `PasswordAuthentication no` in `C:\ProgramData\ssh\sshd_config`, then restart sshd. `HKLM:\SOFTWARE\OpenSSH` `DefaultShell` switches the login shell from cmd to PowerShell.
- Connecting from the host to its own Tailscale IP can hang; don't read that as the phone failing.
