# n8n Self-Hosted on Synology NAS

Self-hosted [n8n](https://n8n.io) workflow automation running on a Synology NAS via Docker,
exposed over HTTPS through Synology's built-in DDNS and reverse proxy.

---

## Architecture

```
Internet
   │
   │  https://n8n.er-buco-nas.familyds.net  (port 443)
   ▼
┌──────────────────────────────────────────┐
│  Synology DSM — Reverse Proxy            │
│  HTTPS → HTTP localhost:5678  (HSTS on)  │
│  WebSocket headers forwarded             │
└──────────────────────────────────────────┘
   │
   │  http://localhost:5678
   ▼
┌──────────────────────────────────────────┐
│  Docker container: n8nio/n8n             │
│  Volume: n8n_data → /volume1/docker/n8n/ │
└──────────────────────────────────────────┘
```

---

## 1. Synology DDNS (Dynamic DNS)

Synology provides a free DDNS hostname under `familyds.net` (and other domains).
It keeps the DNS record pointing to your home IP address even when the ISP changes it.

**Setup steps:**

1. Open **DSM → Control Panel → External Access → DDNS** and click **Add**.
2. Choose **Synology** as the service provider and pick a hostname —
   e.g. `er-buco-nas.familyds.net`.
3. Enable **Heartbeat** so DSM updates the record automatically whenever the IP changes.
4. Optionally, request a **Let's Encrypt certificate** for the hostname via
   **DSM → Control Panel → Security → Certificate**. DSM renews it automatically.

> The DDNS hostname used in this setup is `n8n.er-buco-nas.familyds.net` — a subdomain of the
> base DDNS name, covered by the same certificate (wildcard or SAN).

---

## 2. Reverse Proxy configuration

Configured in **DSM → Control Panel → Login Portal → Advanced → Reverse Proxy**.
It terminates TLS and forwards requests to the n8n container on `localhost:5678`.

### General tab

| Field | Value |
|---|---|
| Reverse Proxy Name | `n8n` |
| Source — Protocol | `HTTPS` |
| Source — Hostname | `n8n.er-buco-nas.familyds.net` |
| Source — Port | `443` |
| Enable HSTS | ✅ enabled |
| Access control profile | Not configured |
| Destination — Protocol | `HTTP` |
| Destination — Hostname | `localhost` |
| Destination — Port | `5678` |

### Custom Headers tab

These two headers are required for WebSocket support, which n8n uses for real-time UI updates
and its internal task runner.

| Header Name | Value |
|---|---|
| `Upgrade` | `$http_upgrade` |
| `Connection` | `$connection_upgrade` |

---

## 3. Docker startup script

Save as `docker-n8n.sh`, make it executable (`chmod +x docker-n8n.sh`),
and run it from an SSH session on the NAS.

```bash
#!/bin/bash

sudo docker run -it --rm \
    --name n8n \
    -p 5678:5678 \
    -e GENERIC_TIMEZONE="Europe/Brussels" \
    -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
    -e N8N_RUNNERS_ENABLED=true \
    -e WEBHOOK_URL=https://n8n.er-buco-nas.familyds.net \
    -e N8N_PROXY_HOPS=1 \
    -v n8n_data:/volume1/docker/n8n/data \
docker.n8n.io/n8nio/n8n
```

### Environment variables

| Variable | Purpose |
|---|---|
| `GENERIC_TIMEZONE` | Sets the timezone used for cron-based schedules inside n8n. |
| `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS` | Ensures sensitive settings files are not world-readable inside the container. |
| `N8N_RUNNERS_ENABLED` | Enables the task runner process required for executing workflow nodes. |
| `WEBHOOK_URL` | The public HTTPS URL n8n advertises for incoming webhooks. Must match the URL reachable from the internet. |
| `N8N_PROXY_HOPS` | Number of reverse-proxy hops in front of n8n (`1` = DSM reverse proxy). Allows n8n to read the real client IP from `X-Forwarded-For`. |

### Volume mount

`n8n_data:/volume1/docker/n8n/data`

The named Docker volume `n8n_data` is stored under `/volume1/docker/n8n/data` on the NAS.
This directory persists all workflows, credentials, and execution history across container restarts.

---

## 4. Notes & caveats

- `--rm` removes the container when it stops. Data is safe because it lives in the volume,
  not in the container layer.
- `-it` keeps the container attached to the terminal. For a background service, replace with
  `-d` (detached mode) and drop `--rm`.
- Port `5678` is only reachable on localhost; no need to open it on the external firewall.
- With HSTS enabled on the reverse proxy, browsers will refuse plain-HTTP connections to the
  hostname after the first visit.
