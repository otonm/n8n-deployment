# n8n Self-Hosted Stack

A self-contained Docker Compose deployment of [n8n](https://n8n.io) with external task runners for JavaScript and Python code nodes, and a Cloudflare Tunnel for zero-exposed-port public access.

---

## Architecture overview

The stack consists of four services, all sharing a single internal bridge network (`n8n-network`). No ports are published to the Docker host — the only inbound path from the internet is through the Cloudflare Tunnel, which terminates TLS at Cloudflare's edge and forwards plain HTTP to `n8n-main` over the internal network.

```
Internet
   │  HTTPS (n8n.mahnic.org)
   ▼
Cloudflare Edge
   │  Encrypted tunnel (outbound-initiated)
   ▼
cloudflared ──────────────────────────────────────────┐
                                                      │  http://n8n-main:5678
n8n-network (bridge)                                  │       |
   ├── n8n-main          (ports 5678, 5679 — internal only) ◄─┘
   ├── n8n-runners       (connects to n8n-main:5679 via WebSocket)
   └── cloudflared       (outbound to Cloudflare edge only)

(n8n-config-patcher runs once at startup — not on the network)
```

**`n8n-main`** is built from a custom Dockerfile that extends `n8nio/n8n:latest` by copying `apk` from Alpine and installing `bash`, `curl`, and `ffmpeg`. The resulting image is tagged locally as `n8n-ffmpeg`. `curl` is required for the healthcheck; `ffmpeg` is available to workflows that process audio or video. `pull_policy: never` prevents Docker Compose from trying to pull this locally-built image from a registry.

**`n8n-config-patcher`** is a one-shot container (`restart: no`) that runs at startup before the runners service starts. It uses the Python interpreter shipped inside `n8nio/runners` to copy the image's own `/etc/n8n-task-runners.json` to a host-mounted path (`/root/n8n/config/`) and patches it to unlock all modules. The config file is then bind-mounted read-only into `n8n-runners`. Because this container only performs local file I/O, it is intentionally not attached to any network.

**`n8n-runners`** runs the `n8nio/runners` image as a sidecar, handling execution of JavaScript and Python code nodes in isolation from the main n8n process. It connects to the task broker in `n8n-main` over WebSocket on port 5679. It depends on `n8n-config-patcher` completing successfully (ensuring the patched config is in place before the launcher reads it) and on `n8n-main` passing its healthcheck (ensuring the broker is listening before the runner attempts to connect).

**`cloudflared`** maintains a persistent outbound tunnel to Cloudflare's edge network. The routing rule — `n8n.mahnic.org → http://n8n-main:5678` — is configured in the Cloudflare Zero Trust dashboard and encoded into the tunnel token. It depends on `n8n-main` being healthy so the public endpoint isn't exposed before n8n is ready to serve requests.

---

## Why the config file must be patched

n8n's task runner launcher controls which modules JavaScript and Python code nodes can import through a config file at `/etc/n8n-task-runners.json`, not through environment variables. The relevant keys (`NODE_FUNCTION_ALLOW_BUILTIN`, `NODE_FUNCTION_ALLOW_EXTERNAL`, `N8N_RUNNERS_STDLIB_ALLOW`, `N8N_RUNNERS_EXTERNAL_ALLOW`) live in an `env-overrides` section of that file, which the launcher injects into runner child processes unconditionally — overriding any values set as container environment variables. Setting these on the `n8n-runners` service in your compose file has zero effect.

The patcher solves this by modifying the config file directly using Python's `json` module, which is robust to any whitespace or formatting changes in future image releases (unlike `sed`-based approaches). The patched file is mounted read-only into the runners container, making it the single authoritative source for module permissions.

---

## Prerequisites

- Docker Engine with the Compose plugin
- A domain managed by Cloudflare
- A Cloudflare Zero Trust account

---

## Setup

### 1. Create directory structure

```bash
mkdir -p /root/n8n/{data,work,config}
```

### 2. Create the tunnel in Cloudflare Zero Trust

1. Navigate to **Networks → Tunnels → Create a tunnel**.
2. Name the tunnel (e.g., `n8n`), save, and copy the token shown on screen.
3. Under **Public Hostnames**, add a hostname:
   - **Subdomain / Domain:** `n8n.mahnic.org`
   - **Service type:** HTTP
   - **URL:** `n8n-main:5678`

### 3. Configure environment variables

Create `/root/n8n/.env`:

```env
N8N_ENCRYPTION_KEY=<generate with: openssl rand -hex 32>
N8N_RUNNERS_AUTH_TOKEN=<generate with: openssl rand -hex 32>
CLOUDFLARE_TUNNEL_TOKEN=<token from Cloudflare Zero Trust dashboard>
```

### 4. Place the compose file

Copy `compose.yml` to `/root/n8n/compose.yml`.

### 5. Build and start

```bash
cd /root/n8n
docker compose build
docker compose up -d
```

On first start, `n8n-config-patcher` will run and write the patched config to `/root/n8n/config/n8n-task-runners.json`. You can inspect it to verify the `env-overrides` values were set to `*`.

---

## Automated weekly updates (systemd)

The stack ships with a systemd service and timer that update all containers every Saturday at 02:00 local time (Europe/Zurich). The update sequence is:

1. `docker compose down` — bring the stack down cleanly
2. `docker compose pull` — pull new versions of all directly-referenced images
3. `docker compose build --pull` — rebuild `n8n-ffmpeg` with a freshly pulled `n8nio/n8n:latest` base layer
4. `docker compose up -d` — bring everything back up

If any step fails the chain halts, preventing a broken image from being started.

### Installing the timer

```bash
# Copy both unit files
cp n8n-update.service n8n-update.timer /etc/systemd/system/

# Reload systemd and enable the timer
systemctl daemon-reload
systemctl enable --now n8n-update.timer

# Confirm the timer is loaded and check its next trigger time
systemctl list-timers n8n-update.timer
```

To trigger an update manually at any time:

```bash
systemctl start n8n-update.service
```

To inspect the log of a past update run:

```bash
journalctl -u n8n-update.service
```

If the host was powered off at the scheduled time, the update will run automatically on the next boot (`Persistent=true` in the timer unit).

---

## Healthchecks

All four long-running services have healthchecks. The `depends_on: condition: service_healthy` chain means the startup order is enforced not just by process start time, but by actual readiness.

| Service | Endpoint checked | Tool used | Notes |
|---|---|---|---|
| `n8n-main` | `localhost:5678/healthz` and `localhost:5679/healthz` | `curl` | Both n8n web service and task broker must pass |
| `n8n-runners` | `localhost:5680/healthz` | `node` inline HTTP client | Port 5680 is the launcher's own health endpoint |
| `cloudflared` | `localhost:2000` (metrics server) | `cloudflared ... ready` | Confirms tunnel connection to Cloudflare edge |
| `n8n-config-patcher` | N/A — exits 0 on success | — | `service_completed_successfully` condition used |

`n8n-main` uses `curl` (installed in the custom Dockerfile) rather than `wget`, since the base n8n image does not ship `wget`. `n8n-runners` uses a Node.js one-liner since the runners image ships neither `curl` nor `wget`.

---

## File layout

```
/root/n8n/
├── docker-compose.yml
├── .env                          # secrets — do not commit
├── data/                         # n8n persistent data (workflows, credentials)
├── work/                         # shared working directory for n8n nodes
└── config/
    └── n8n-task-runners.json     # written by n8n-config-patcher at startup
```

---

## Notable design decisions

**No reverse proxy.** Cloudflared acts as the sole ingress path, eliminating the need for an nginx/Caddy/Traefik container. TLS is handled at Cloudflare's edge; internal traffic is plain HTTP on the bridge network.

**`expose` instead of `ports`.** Port 5678 and 5679 are declared via `expose`, making them reachable by containers on the same network but not published to the Docker host. The only externally reachable surface is the Cloudflare tunnel.

**Config patcher over image extension.** Mounting the patched config file is simpler than building a derived `n8nio/runners` image. The patcher always starts from the image's own shipped config, so new fields or runner types added in future releases are inherited automatically.

**`N8N_RUNNERS_ENABLED` is absent.** The variable is not needed in `n8n-main`'s environment because `N8N_RUNNERS_MODE=external` implies runners are enabled. Adding it would be redundant.

---

## Upgrading n8n manually

The weekly timer handles routine updates. For an out-of-schedule upgrade:

```bash
cd /root/n8n
docker compose down
docker compose pull
docker compose build --pull
docker compose up -d
```

Note that `n8nio/n8n` and `n8nio/runners` image versions must match. Both are pinned to `latest`, so pulling at the same time keeps them in sync. If you want to pin to a specific version, update the image tags in `docker-compose.yml` for both services simultaneously.
