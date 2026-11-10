---
name: hermes-mobile-remote-access
description: Use when connecting phone apps to a remote Hermes.
version: 3.0.0
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
- **Migration:** a user connects through `hermes dashboard` or `hermes serve` on port 9119, either directly or through an SSH tunnel. That path is being deprecated; move them to the gateway Webapp.
- The phone can't connect, the app asks for "server / username / password", or a session fails with "gateway error 4001".
- An SSH phone app (e.g. herdr-mobile) asks for server / username / port, or a phone key needs authorizing.

Answer the literal question first ("server is X, username Y"), then fix. Users doing this are usually mid-task on their phone and impatient.

## How it fits together
- **Gateway Webapp (current):** the always-on gateway (`hermes gateway`) serves the Webapp on its own listener, configured by `webapp.*` in `config.yaml` (default `127.0.0.1:9120`). `tailscale serve` puts tailnet-only HTTPS in front of it. Sign-in uses the dashboard auth provider (username/password or OAuth).
- **Old path (deprecated):** a separate `hermes dashboard` or `hermes serve` process on port 9119, kept alive by a scheduled task or systemd unit. The mobile app reached it in one of two ways:
  1. **URL mode:** `http://<ip>:9119` directly (dashboard bound to a reachable address).
  2. **SSH-tunnel mode:** the app's "Add Instance" form SSHes to the host and tunnels to a loopback-only dashboard.
  Retiring 9119 breaks **both**; the user must re-add the connection with the new HTTPS URL.
- Both paths use the **same sign-in**: `dashboard.basic_auth.*` in `config.yaml`, or the `HERMES_DASHBOARD_BASIC_AUTH_USERNAME` / `_PASSWORD` / `_PASSWORD_HASH` / `_SECRET` env vars in `.env`. Migrating keeps the user's existing username/password.

## Safety rules (agent)
- **Back up `config.yaml` and `.env` first** (dated copies, same permissions; `.env` stays mode 600).
- **Never print secret values** (password, hash, secret, tokens). Report key *names* and booleans only.
- **Never take a password in chat**, and don't reveal one if the user asks: point them to where it lives (`.env` / the file you generated) and let them read it locally.
- **Ask before replacing anything that already exists**, especially `tailscale serve` mappings and running services.
- **Keep a rollback** (below) and tell the user what it is.

## Prerequisites (both flows)
1. **Tailscale** on the host and the phone, same tailnet. In `tailscale status`, a connected phone shows `active` or `-`; `offline, last seen …` means the phone isn't connected (open the Tailscale app on it). `tailscale ip -4` gives the host's `100.x` address.
2. **MagicDNS and HTTPS certificates** turned on in the Tailscale admin console (DNS page). `tailscale serve` needs them for `https://<machine>.<tailnet>.ts.net`. Get the exact name from `tailscale status --json` (`Self.DNSName`, drop the trailing dot).
3. **A recent Hermes** with the gateway Webapp: `hermes webapp --help` must work (verified on v0.23.0). Otherwise `hermes update` first.
4. **Gateway installed as a service** so it survives reboots: `hermes gateway install` / `hermes gateway status`.
   - Windows: a Startup-folder login item, so it starts at **user logon**, not at boot.
   - Linux: a user systemd unit. Check `loginctl show-user $USER -p Linger`; if `Linger=no`, run `sudo loginctl enable-linger $USER` or the gateway dies at logout and doesn't start at boot.
5. **A sign-in provider.** Check with `hermes config get dashboard.basic_auth` and `grep -o '^HERMES_DASHBOARD_BASIC_AUTH_[A-Z_]*' ~/.hermes/.env` (names only). If none exists, see "Setting the password".

## First-time setup (new users)
1. Do the prerequisites. Back up `config.yaml` and `.env`.
2. Configure the Webapp:
   ```bash
   hermes config set webapp.enabled true
   hermes config set webapp.public_url https://<machine>.<tailnet>.ts.net
   ```
   Leave `webapp.host` at `127.0.0.1` and `webapp.port` at `9120`. `public_url` is what turns sign-in on for a loopback bind, and it's the only outside hostname the Webapp accepts.
3. **Build the Webapp bundle:** `hermes webapp --build-only`. No restart is needed for this: a running gateway serves the bundle as soon as it exists. Skipping it leaves `--status` at `Unavailable: The Webapp is not built`.
4. **Check existing Tailscale serve config first:** `tailscale serve status`. If HTTPS 443 already proxies to another app, `tailscale serve --bg 9120` **silently replaces it**. Ask the user; alternatives are a different HTTPS port (`tailscale serve --bg --https=8443 9120`, then `public_url` must include `:8443`) or a path mount.
5. Expose it to the tailnet only: `tailscale serve --bg 9120`. Linux may need `sudo`, or once `sudo tailscale set --operator=$USER`. **Never `tailscale funnel`**: that publishes it to the internet, and username/password auth is for trusted networks only.
6. **Restart the gateway** so it reads the new `webapp.*` settings. See "Restarting the gateway"; normally the user runs it.
7. Verify (see "Verifying").
8. Phone: in the Hermes mobile app enter **server `https://<machine>.<tailnet>.ts.net`** (no port) and the dashboard **username/password**. The username is the dashboard one, not the OS account or the Tailscale login.

## Migrating from `hermes dashboard` / `hermes serve` (port 9119)
1. **Find what runs the old backend**, so it doesn't come back:
   - `hermes dashboard --status` / `hermes serve --status`
   - Windows: `Get-ScheduledTask | ? TaskName -match hermes`, the Startup folder, wrapper scripts.
   - Linux: `systemctl --user list-units --all | grep -i hermes`. Expect units like `hermes-dashboard.service`, and on multi-profile hosts **several `hermes-gateway-*.service` units**. Find the one serving the default profile; the Webapp settings come from it.
2. **Find where its sign-in lives.** `systemctl --user cat <unit>` shows `Environment=` lines and `EnvironmentFile=` paths. The gateway reads `dashboard.basic_auth` in `config.yaml` or `HERMES_DASHBOARD_BASIC_AUTH_*` in `~/.hermes/.env`. If the creds live only in the unit (an `Environment=` line or a separate `EnvironmentFile`), copy **only the `HERMES_DASHBOARD_BASIC_AUTH_*` lines** into `.env` (e.g. `grep '^HERMES_DASHBOARD_BASIC_AUTH_' <file> >> ~/.hermes/.env`), keep `.env` at mode 600, and never print the values.
3. Back up `config.yaml` and `.env`, then do **First-time setup steps 2-7** *while the old backend is still running*. They don't share a port, so the user isn't cut off.
4. **Retire the old backend safely:**
   - Linux: `systemctl --user disable --now <unit>`. **Keep the unit file** for rollback.
   - Windows: `Disable-ScheduledTask`, then stop the 9119 process.
   - Then `hermes dashboard --stop` if anything remains, and confirm nothing listens on 9119 (`ss -ltnp | grep 9119` / `Get-NetTCPConnection -LocalPort 9119 -State Listen`).
5. Clean up leftovers, after asking: firewall rules for 9119, wrapper scripts, an old `tailscale serve`/`funnel` entry pointing at 9119.
6. Phone: **re-add the connection.** URL-mode entries (`http://100.x:9119`) and SSH-tunnel entries ("Add Instance" over SSH) both stop working. New server: `https://<machine>.<tailnet>.ts.net`, no port, same username/password. Expect one sign-in.

## Rollback
1. `hermes config set webapp.enabled false` (or restore the `config.yaml` backup), and restore `.env` from its backup if you changed it.
2. `tailscale serve --https=443 off`, then re-create any mapping it replaced (e.g. `tailscale serve --bg <old-port>`).
3. Re-enable the old backend: `systemctl --user enable --now <unit>` / `Enable-ScheduledTask` + start.
4. Restart the gateway, and tell the user to switch the phone back to the old connection.

## `hermes webapp --status` messages and fixes
| Message | Meaning | Fix |
|---|---|---|
| `Serving at http://127.0.0.1:9120` | Working | Done. Verify from outside. |
| `The Webapp is off. Set webapp.enabled: true in …` | Not enabled in config | `hermes config set webapp.enabled true`, then restart the gateway |
| `The Webapp is off: <reason>. Fix it, then …` | A `webapp.*` value is invalid (e.g. a specific IP in `host`, a bad `public_url`) | Fix that value, then restart |
| `The gateway started before the Webapp was turned on…` | Config is right; gateway hasn't re-read it | Restart the gateway |
| `Unavailable: The Webapp is not built…` | No bundle (or stale after an update) | `hermes webapp --build-only` (or `hermes update`); no restart needed |
| `Unavailable: <other reason>` (port taken, no sign-in provider…) | As stated | Fix the cause, then restart |
| `The gateway is still starting the Webapp…` | Warm-up | Run `--status` again shortly |
| `The gateway is not running…` | Gateway down | Start it (`hermes gateway start`) |
| `The running gateway is older than the Webapp…` | Updated Hermes, old gateway process | Restart the gateway |

## Restarting the gateway
- **Default: the user runs it.** Give them `hermes gateway restart` to run from a separate terminal (not a chat that goes through that gateway), then verify afterwards.
- Hermes **refuses `gateway restart`/`stop` from inside the gateway process** ("Refusing to restart the gateway from inside the gateway process"), and **refuses while sessions are live** unless you pass `--yes`, which interrupts them (including the agent's own chat).
- If you must launch it yourself, use a fully detached process that runs outside the gateway's process tree, and **call Hermes by absolute path**: transient units and fresh shells may have no `PATH`. On Linux, `systemd-run --user` units fail with `hermes: command not found`; use `~/.local/bin/hermes` (check with `command -v hermes`). On Windows, `Start-Process powershell -WindowStyle Hidden -File <script>`. Have the script log results under the Hermes `logs/` dir, warn the user the chat drops for about a minute, then read the log. If the restart is refused, fall back to handing the user the command.

## Verifying
- `hermes webapp --status` → `Serving at http://127.0.0.1:9120` (see the table for anything else).
- `tailscale serve status` → `https://<host>.ts.net (tailnet only)` with `/ proxy http://127.0.0.1:9120`, and nothing you didn't intend to replace.
- `curl -s https://<host>.ts.net/api/status` → JSON with `auth_required: true`, `auth_providers` containing `"basic"`, `gateway_state: running`.
- Healthy sign-in responses: `/` → `302`; `/login?next=%2F` → `200`. `/auth/login` returning `422` to a bare request is normal (no credentials sent), not an error. A `503` right after a restart is warm-up.
- Multi-profile gateways log `Plugin 'basic' … from profile scope; ignored` warnings. They're harmless.
- **These checks only prove the server side.** Have the user try the phone, and ask for the exact error text if it fails.

## Without Tailscale
The Webapp is plain HTTP(S) on `webapp.host:webapp.port`; any network that reaches it works. Pick by how exposed it is:

| Route | Config | Phone server | Sign-in |
|---|---|---|---|
| Home Wi-Fi / LAN | `webapp.host 0.0.0.0` (or `::`) | `http://<host-LAN-IP>:9120` | Username/password is acceptable on a trusted network |
| Other VPN (WireGuard, ZeroTier, Nebula…) | `webapp.host 0.0.0.0` | `http://<host-VPN-IP>:9120` | Same as LAN |
| Reverse proxy / tunnel (Caddy, nginx, Cloudflare Tunnel…) | keep `webapp.host 127.0.0.1`, proxy to `127.0.0.1:9120`, set `webapp.public_url https://<proxy-hostname>` | `https://<proxy-hostname>` | **OAuth** if internet-reachable; not username/password |

Gotchas for these routes:
- **A wildcard bind listens on every interface** (LAN, VPN, any other adapter). It turns sign-in on automatically; without a sign-in provider the gateway serves native clients only (`hermes webapp --status` says so).
- **Firewall:** a wildcard bind needs an inbound rule for 9120, scoped to the network you mean. Windows (admin): `New-NetFirewallRule -DisplayName "Hermes Webapp" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 9120 -RemoteAddress <LAN-or-VPN-CIDR>`. Linux: `ufw allow from <CIDR> to any port 9120 proto tcp`. Windows also blocks a network whose profile is Public; set the trusted network to Private.
- **Plain HTTP:**
  - The password and session cookie travel unencrypted on that network. That's fine inside an encrypted VPN, weaker on shared Wi-Fi.
  - Browsers also switch off secure-context features: microphone/dictation/voice, clipboard paste, notifications and the bot screen.
  - Use HTTPS (reverse proxy or `tailscale serve`) to keep them.
- **LAN IPs change** (DHCP). Give the host a DHCP reservation, or use its mDNS/LAN hostname.
- **Reverse proxy:**
  - The Webapp only accepts requests for the `webapp.public_url` hostname.
  - It only trusts `X-Forwarded-*` headers from loopback plus the IPs/CIDRs in `dashboard.trusted_proxies`. Add the proxy's address there if the proxy runs on another machine.
  - The proxy must pass WebSocket upgrades. nginx needs `proxy_http_version 1.1` plus `Upgrade`/`Connection` headers; Caddy and Cloudflare Tunnel do this by default.
- **Internet exposure:**
  - Hermes's docs reserve username/password for trusted networks.
  - For anything reachable from the internet, use an OAuth/OIDC sign-in provider (see the web-dashboard docs) or an access layer such as Cloudflare Access in front.
  - Never port-forward 9120 on the router.
- Changing `webapp.*` needs a gateway restart.

## Setting the password
There is **no** `hermes dashboard setup` or password subcommand.
- `.env` (mode 600):
  - `HERMES_DASHBOARD_BASIC_AUTH_USERNAME`
  - `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH` (preferred) or `_PASSWORD`
  - `HERMES_DASHBOARD_BASIC_AUTH_SECRET` (`openssl rand -base64 32`), so sessions survive restarts
- Config: `hermes config set dashboard.basic_auth.password_hash <hash>`, plus `username` and `enabled true`.
- **To hash a password:** use Hermes's own venv python with the `hermes-agent` checkout on `sys.path`, `from plugins.dashboard_auth.basic import hash_password`, and read the password via `getpass`. Give the user a script to run themselves.
- **If no provider exists and the user wants one generated:** create a random password and write it to a mode-600 file the user reads locally. Put the hash and secret into `.env`, and tell them only the file's path.
- Restart the gateway after changing auth.

## Gotchas
- **Tailscale stuck "starting" after a reboot** (`tailscale status` → "Tailscale is starting", no `100.x` address): nothing on the host is reachable. The user restarts Tailscale (tray → quit/reopen, `Restart-Service Tailscale` as admin, or `sudo systemctl restart tailscaled`), signing in again if needed.
- **Tailscale restarts can drop serve config.** Re-check `tailscale serve status` after Tailscale problems.
- **Windows starts the gateway at logon, not boot.** A rebooted PC sitting at the lock screen serves nothing until someone signs in. On Linux the equivalent problem is lingering being off.
- **No firewall rule is needed for `tailscale serve`.** It forwards over loopback.
- **`webapp.host` accepts only `127.0.0.1`, `::1`, `localhost`, `0.0.0.0` or `::`.** Any other value, including a specific Tailscale IP, keeps the Webapp off; `--status` says why.
- **`dashboard.public_url` does not apply to the Webapp.** Use `webapp.public_url`.
- **Profiles:**
  - Webapp settings come from the config of the gateway serving the profile: the default profile's for named profiles it serves.
  - A standalone profile's own gateway needs a different `webapp.port`.
  - Sessions are per profile (`profiles/<name>/state.db`), so a session "missing" on the phone may be in another profile.
- **Port 9120 taken by another program:** the gateway serves native clients only and invalidates Webapp sign-ins. Free the port, then sign in again.

### Old dashboard/serve path (when a user still runs it)
- A non-loopback `--host` requires an auth provider; without one it fails closed at startup. `--insecure` is a no-op.
- Bind to the Tailscale IP rather than `0.0.0.0` to keep it off the LAN.
- **Windows Firewall silently blocks** Hermes's bundled `tools\python-3.x\python.exe`, which has no allow rule. The symptom: it works from the host, but the phone times out. Admin fix, tailnet only: `New-NetFirewallRule -DisplayName "Hermes 9119 (Tailscale)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 9119 -RemoteAddress 100.64.0.0/10`.
- **It dies when the desktop app closes** if launched from a PowerShell wrapper loop or an agent's background terminal (Task Scheduler last result `3221225786`, i.e. interrupted).
  - Launch it detached (`Start-Process -WindowStyle Hidden`) from a one-shot check script on a repeating trigger.
  - This fragility is a good reason to migrate.
- PowerShell 5.1 `*>>` logs come out as UTF-16 garbage. Pipe through `Out-File -Encoding utf8` and set `PYTHONIOENCODING=utf-8`.
- A pending `hermes update` ("install out of sync", "update is still running") can delay or interrupt startup. Suggest `hermes update`.

## Gateway error 4001 when opening a session
Run `grep -a "session.resume refused" <hermes>/logs/gateway.log`. The text `invalid_params field=base_url ... must be an http(s) URL` means:
- That session was saved with a `process://...` provider base_url (subscription-CLI providers), which the gateway's resume check rejects.
- The session's messages are intact, other sessions are unaffected, and new chats work.

Options:
- `hermes update`, then restart the gateway.
- Back up the profile's `state.db`, then edit that session's `model_config` base_url.
- Start a new session and point it at the old one's output files.

## SSH phone apps (server / username / port)
- **What to enter:**
  - Server: the Tailscale IP or MagicDNS name.
  - Port: `22`.
  - Username: the **OS account** name (Windows: `whoami` without the domain), not the Tailscale or dashboard login.
- Connecting from the host to its own Tailscale IP can hang; don't read that as the phone failing.

### Windows host
- **OpenSSH Server is required.** Tailscale SSH doesn't run on Windows hosts.
- **It installs as Manual start**, so it's dead after every reboot. Fix as admin: `Start-Service sshd; Set-Service sshd -StartupType Automatic`.
- **Firewall:** the default rule `OpenSSH-Server-In-TCP` allows the Private profile, and Tailscale's adapter is Private.
- **Admin accounts ignore `~\.ssh\authorized_keys`.**
  - Phone public keys go in `C:\ProgramData\ssh\administrators_authorized_keys`.
  - Then run `icacls <file> /inheritance:r /grant Administrators:F /grant SYSTEM:F`. With looser permissions, sshd silently ignores the file and falls back to password.
  - Writing the file needs elevation: run a script via `Start-Process -Verb RunAs` (the user clicks UAC), and have it log the result to a file you can read.
- **Optional, once keys work:**
  - Set `PasswordAuthentication no` in `C:\ProgramData\ssh\sshd_config`, then restart sshd.
  - The registry value `HKLM:\SOFTWARE\OpenSSH` `DefaultShell` switches the login shell from cmd to PowerShell.

### Linux host
1. Back up the file: `cp ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak-$(date +%F)`. Create `~/.ssh` with mode 700 if it's missing.
2. If the file doesn't end in a newline, add one first (`[ -n "$(tail -c1 f)" ] && echo >> f`). Otherwise the new key gets glued onto the last line.
3. Append the key only if it isn't already there (`grep -qxF "<key>" f || echo "<key>" >> f`).
4. `chmod 600 ~/.ssh/authorized_keys`.
5. Print the fingerprint (`ssh-keygen -lf <(echo "<key>")`) so the user can compare it with the one the phone shows.
6. Alternative: Tailscale SSH (`tailscale set --ssh`) works on Linux hosts and needs no keys. Mention it, but don't switch to it unasked.
