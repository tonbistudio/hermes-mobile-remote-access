# hermes-mobile-remote-access

A [Hermes Agent](https://github.com/NousResearch/hermes-agent) skill for connecting the Hermes mobile app (or any browser) to a Hermes gateway on another machine, privately over [Tailscale](https://tailscale.com).

It covers two cases:

- **First-time setup**: serve the gateway's Webapp on your tailnet with `tailscale serve`, so the phone connects to `https://<machine>.<tailnet>.ts.net` with your dashboard username and password.
- **Migrating off `hermes dashboard` / `hermes serve`**: the old port-9119 backend is being deprecated. The skill moves you to the gateway Webapp without downtime, keeps your existing login, and retires the old process.

It also covers the problems people hit along the way: Tailscale stuck "starting" after a reboot, the gateway only starting at Windows logon, Windows Firewall silently blocking Hermes's bundled Python, the old backend dying when the desktop app closes, setting a dashboard password (there's no CLI command for it), "gateway error 4001" when reopening a session, and SSH phone apps on Windows (`sshd` startup and admin-account key placement).

## Install

Copy the skill folder into your Hermes skills directory:

```bash
git clone https://github.com/tonbistudio/hermes-mobile-remote-access
cp -r hermes-mobile-remote-access/skills/devops/hermes-mobile-remote-access ~/.hermes/skills/devops/
```

On Windows the skills directory is `%LOCALAPPDATA%\hermes\skills\`. Start a new session (or run `/reload-skills`) and Hermes loads it whenever you ask about connecting your phone.

## Use

Ask Hermes things like:

- "Set up the Hermes mobile app to connect to this PC over Tailscale."
- "Migrate my dashboard setup to the gateway webapp."
- "The mobile app can't connect. What's wrong?"
- "My SSH app wants a server, username and port."

## Requirements

- Hermes Agent with the gateway Webapp (`hermes webapp --help` should work).
- Tailscale on the host and the phone, on the same tailnet, with MagicDNS and HTTPS certificates enabled in the admin console.
- A dashboard sign-in provider (username/password or OAuth).

## Security

The setup is **tailnet-only** (`tailscale serve`). The skill tells Hermes never to use `tailscale funnel`, because username/password sign-in is meant for trusted networks, not the open internet. Hermes never asks for your password in chat; it gives you a script to set it yourself.

## License

MIT
