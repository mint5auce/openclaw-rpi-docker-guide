# OpenClaw on Raspberry Pi via Docker

An opinionated, Raspberry Pi–focused setup for running OpenClaw securely. 
It avoids LAN exposure by binding services to 127.0.0.1, with access provided 
via SSH tunnelling (optionally over Tailscale or similar). The CLI runs in an 
ephemeral container that shares the gateway network namespace for local-only connectivity.

This setup runs the OpenClaw gateway as a WebSocket service bound to
localhost on the Pi. Use `oc health` and `oc dashboard` to interact with it.

## Setup

1) Create data directories and lock permissions:
   - `mkdir -p ~/agents/openclaw-data/config ~/agents/openclaw-data/workspace`
   - `chmod 700 ~/agents/openclaw-data ~/agents/openclaw-data/config ~/agents/openclaw-data/workspace`

2) Clone the upstream OpenClaw repo:
   - `git clone https://github.com/openclaw/openclaw ~/agents/openclaw`

3) Copy this repo's files into `~/agents/openclaw`:
   - `cp docker-compose.secure.yml .env.example ~/agents/openclaw/`
   - `cp -r scripts systemd ~/agents/openclaw/`
   - Create `~/agents/openclaw/.env` from `.env.example`, set a real token,
     then `chmod 600 ~/agents/openclaw/.env`
   - Do not commit `.env`

### Auth environment variables note

OpenClaw's authentication/env var names can vary depending on version and whether you're using browser-session cookies, Claude sessions, or API keys. Treat the `CLAUDE_*` variables in the compose file and `.env.example` as examples—check the upstream OpenClaw docs for the correct variables for your chosen auth method.

4) Build the image:
   - `cd ~/agents/openclaw`
   - `docker-compose -f docker-compose.secure.yml build`

5) Start the gateway service only:
   - `docker-compose -f docker-compose.secure.yml up -d openclaw-gateway`

6) Install the helper CLI:
   - `install -d ~/bin`
   - `cp scripts/openclaw-cli ~/bin/openclaw-cli`
   - `cp scripts/oc ~/bin/oc`
   - `chmod +x ~/bin/openclaw-cli ~/bin/oc`
   - Ensure your PATH includes `~/bin` in `~/.profile`:
     `export PATH="$HOME/bin:$PATH"`

7) Run basic commands:
   - `oc setup`
   - `oc health`
   - `oc dashboard`

8) From your laptop, tunnel the gateway:
   - `ssh -N -L 18789:127.0.0.1:18789 pi@your-pi`

## Notes

### Localhost-only exposure vs OpenClaw bind mode

This setup prevents LAN access by publishing the gateway ports on **127.0.0.1 only** in Docker Compose:

- `127.0.0.1:${OPENCLAW_GATEWAY_PORT:-18789}:18789`
- `127.0.0.1:${OPENCLAW_BRIDGE_PORT:-18790}:18790`

That means the gateway is reachable only from the Pi itself (or via an SSH tunnel), regardless of what OpenClaw uses for its internal `--bind` mode.

`OPENCLAW_GATEWAY_BIND` controls OpenClaw's internal bind mode **inside the container network namespace** (e.g. `lan` or `loopback`). The default in this guide is `lan` because it tends to behave more predictably with the "CLI shares gateway network" pattern. If you encounter connectivity issues, try switching `OPENCLAW_GATEWAY_BIND` to `loopback`.

## Troubleshooting

- `docker-compose exec` only works when a container is running; this guide
  uses short-lived `docker run` containers instead.
- Do not expect `curl http://127.0.0.1:18789/` to show a website; the gateway
  speaks WebSocket.
- The `chmod 700` permissions are why the container runs as your UID/GID.
- Never commit `.env` or share its token.
