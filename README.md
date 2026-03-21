# Dell Wyse 3040 — Optimized High-Performance Home Server

Complete guide to turning a resource-constrained $20 thin client into a professional home server environment with DNS adblocking, unified monitoring, automated testing, and secure remote access.

### **Project Overview**
This project demonstrates a production-grade, containerized architecture deployed on a **Dell Wyse 3040**. The core challenge was to transform a low-power Intel Atom device (4 cores, 2GB RAM / 8GB eMMC) into a reliable, multi-service node handling network-wide ad-blocking (Pi-hole), service monitoring, and automated performance testing.

#### **Key Features**
*   **Resource Optimization:** Designed a lean environment using **Alpine Linux** and **Podman** (daemonless) to run 6+ concurrent services while maintaining a memory footprint under **30% (approx. 600MB/2GB)**.
*   **Storage Management:** Implemented custom **log rotation logic** and **automated disk-space guards** (Python/Shell) to ensure stability on the hardware-limited 8GB eMMC storage.
*   **Network Performance:** Engineered a custom, multi-threaded **Speedtest-CLI** integration to maximize Gigabit Ethernet throughput, overcoming the typical single-thread bottlenecks of low-power Atom CPUs.
*   **Self-Healing Architecture:** Persistence via OpenRC and Watchtower, ensuring zero-touch recovery and automated image updates.

#### **Tech Stack**
*   **Infrastructure:** Alpine Linux, Podman, Tailscale VPN.
*   **Ad-blocking & DNS:** Pi-hole (with optimized monthly query database limits).
*   **Monitoring:** Beszel (Lightweight Agency), Uptime Kuma (Uptime/Alerts), Glance (Unified UI dashboard).
*   **Automation:** Python/Bash (Custom metrics & API triggers), Watchtower (Auto-updates).

---

## Hardware

| | |
|---|---|
| **Model** | Dell Wyse 3040 |
| **CPU** | Intel Atom x5-Z8350 (4 cores, 1.44–1.92 GHz) |
| **RAM** | 2 GB DDR3L |
| **Storage** | 8 GB eMMC (~5 GB usable) |
| **Power** | ~5W idle |

## What You Get

| Service | Purpose | Port |
|---|---|---|
| **Pi-hole** | DNS server + network-wide ad blocking | 53 (DNS), 80/443 (web admin) |
| **Uptime Kuma** | Monitors service uptime, sends alerts when something goes down | 3001 |
| **Beszel** | System monitoring dashboard — CPU, RAM, disk, network, containers | 8090 |
| **Beszel-agent** | Local agent that feeds metrics to Beszel hub | 45876 (internal) |
| **Glance** | Unified dashboard — shows all services, DNS stats, speedtest results | 8080 |
| **Speedtest** | Automated internet speed tests every 30 min + manual trigger button | 8902 (trigger) |
| **Watchtower** | Automatically updates containers every Sunday at 3:00 AM | — |
| **Tailscale** | VPN mesh — access everything remotely without port forwarding | — |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Dell Wyse 3040  ·  Alpine Linux  ·  Podman                 │
│                                                             │
│  ┌──────────┐  ┌─────────────┐  ┌───────────────────────┐   │
│  │ Pi-hole  │  │ Uptime Kuma │  │  Beszel + Beszel-agent│   │
│  │ :53 :80  │  │   :3001     │  │  :8090                │   │
│  └──────────┘  └─────────────┘  └───────────────────────┘   │
│                                                             │
│  ┌───────────────────────┐  ┌──────────────────┐            │
│  │  Glance Dashboard     │  │ Speedtest Trigger│            │
│  │  :8080                │  │ :8902            │            │
│  │  DNS stats, monitors, │  │ HTTP → run.sh    │            │
│  │  speedtest table      │  └──────────────────┘            │
│  └───────────────────────┘                                  │
│                                                             │
│      Tailscale VPN  ·  Watchtower (Sun 3AM)  ·  Cron        │
└─────────────────────────────────────────────────────────────┘
```

## File Structure

```
/server/
├── pihole/
│   ├── etc-pihole/          # Pi-hole config + databases
│   └── etc-dnsmasq.d/      # DNS config
├── uptime-kuma/
│   └── data/                # Uptime Kuma SQLite database
├── glance/
│   ├── glance.yml           # Dashboard config
│   └── assets/
│       └── speedtest.json   # Speedtest results (auto-generated)
└── speedtest/
    ├── run.sh               # Speedtest script (cron + manual)
    ├── trigger.py           # HTTP trigger server
    └── logs/
        └── speedtest.log    # Raw results (max 48 entries)
```

> **Note:** Beszel data is stored in a Podman named volume (`beszel_data`), not in `/server/`.

---

# Installation

## 1. Install Alpine Linux

1. Download Alpine Linux Extended from https://alpinelinux.org/downloads/ (x86_64)
2. Flash ISO to USB (e.g. with Rufus or `dd`)
3. Boot Wyse 3040 from USB (F12 for boot menu)
4. Run setup:

```sh
setup-alpine
```

Key settings during setup:
- **Keyboard:** `us` (or your layout)
- **Hostname:** `wyse`
- **Network:** configure `eth0` or `mlan0` (WiFi) with static IP
- **DNS:** `8.8.8.8` (temporary, Pi-hole will replace this)
- **Timezone:** `Europe/Warsaw`
- **Disk:** `mmcblk0`, use `sys` mode

Example static IP config:
```
Interface: eth0 (or mlan0 for WiFi)
IP: <SERVER_IP>/24
Gateway: <GATEWAY_IP>
```

Reboot after install, remove USB.

## 2. Verify Network

```sh
ping -c 3 8.8.8.8
ping -c 3 google.com
```

## 3. System Update

```sh
apk update && apk upgrade
```

## 4. Add Required Repositories

```sh
cat >> /etc/apk/repositories << 'EOF'
http://dl-cdn.alpinelinux.org/alpine/edge/main
http://dl-cdn.alpinelinux.org/alpine/edge/community
EOF
apk update
```

## 5. Install Packages

```sh
apk add podman fuse-overlayfs slirp4netns python3 bash curl tailscale
```

## 6. Enable Core Services

```sh
# cgroups — required for containers
rc-update add cgroups boot
rc-service cgroups start

# Podman — container runtime
rc-update add podman default

# Tailscale — VPN
rc-update add tailscale default
rc-service tailscale start

# SSH
rc-update add sshd default

# Cron — for scheduled speedtests
rc-update add crond default
```

## 7. Allow DNS Port

Pi-hole needs port 53, which normally requires root privileges:

```sh
echo "net.ipv4.ip_unprivileged_port_start=53" >> /etc/sysctl.conf
sysctl -p
```

## 8. Connect Tailscale

```sh
tailscale up
```

Open the printed URL in a browser to authenticate. Verify:

```sh
tailscale status
```

---

# Container Setup

## 9. Prepare Directories

```sh
mkdir -p /server/pihole/etc-pihole
mkdir -p /server/pihole/etc-dnsmasq.d
mkdir -p /server/uptime-kuma/data
mkdir -p /server/glance/assets
mkdir -p /server/speedtest/logs
```

## 10. Pull Container Images

```sh
podman pull docker.io/pihole/pihole:latest
podman pull docker.io/louislam/uptime-kuma:latest
podman pull docker.io/henrygd/beszel:latest
podman pull docker.io/henrygd/beszel-agent:latest
podman pull docker.io/glanceapp/glance:latest
podman pull docker.io/containrrr/watchtower:latest
```

## 11. Create Containers

All containers use `--network host` (share the host's network stack) and `--log-opt max-size=1m` (prevent log files from filling the tiny 8 GB eMMC).

### Pi-hole — DNS + Ad Blocking

Blocks ads and trackers at DNS level for every device on your network.

```sh
podman run -d --name pihole --network host --restart unless-stopped \
  --log-opt max-size=1m \
  -e TZ=Europe/Warsaw \
  -e WEBPASSWORD=pihole \
  -e DNSMASQ_USER=pihole \
  -e FT_DNS_PERMIT_ALL=true \
  -v /server/pihole/etc-pihole:/etc/pihole \
  -v /server/pihole/etc-dnsmasq.d:/etc/dnsmasq.d \
  docker.io/pihole/pihole:latest
```

After startup, go to `https://<IP>/admin` and log in with password `pihole`.

Then point your router's DNS to this server's IP so all devices use Pi-hole.

### Pi-hole Blocklists

Add [Hagezi](https://github.com/hagezi/dns-blocklists) blocklists for comprehensive ad/tracker/threat blocking. Go to Pi-hole Admin → Adlists and add:

| List | URL | Domains |
|---|---|---|
| **Hagezi Multi Pro** | `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/hosts/pro.txt` | ~383K |
| **Hagezi TIF (threats)** | `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/hosts/tif.txt` | ~782K |

Then update gravity:

```sh
podman exec pihole pihole -g
```

This brings total blocked domains from ~82K (default StevenBlack list) to ~1.25M.

### Uptime Kuma — Service Monitoring

Web-based uptime monitor. Set up checks for any URL/port and get notified when something goes down.

```sh
podman run -d --name uptime-kuma --network host --restart unless-stopped \
  --log-opt max-size=1m \
  --cap-add=NET_RAW \
  -e TZ=Europe/Warsaw \
  -v /server/uptime-kuma/data:/app/data \
  docker.io/louislam/uptime-kuma:latest
```

`--cap-add=NET_RAW` is required for Uptime Kuma's ping monitors to work.

First visit to `:3001` will prompt you to create an admin account.

### Beszel — System Monitoring

Lightweight server monitoring with a web UI showing CPU, RAM, disk, network, and container stats. Consists of a **hub** (web UI) and a **local agent**.

#### Beszel Hub

```sh
podman run -d --name beszel --network host --restart unless-stopped \
  --log-opt max-size=1m \
  -e TZ=Europe/Warsaw \
  -e APP_URL=http://localhost:8090 \
  -v beszel_data:/beszel_data \
  docker.io/henrygd/beszel:latest \
  serve --http=0.0.0.0:8090
```

First visit to `:8090` will prompt you to create an admin account. After logging in, add a system and copy the SSH key shown — you'll need it for the agent.

#### Beszel Agent

```sh
podman run -d --name beszel-agent --network host --restart unless-stopped \
  --log-opt max-size=1m \
  -e TZ=Europe/Warsaw \
  -e HUB_URL=http://localhost:8090 \
  -e LISTEN=45876 \
  -e KEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEZBMRHRB+Xqqy06dNXM2zO/4CAFpV28O2bc2RZTDznO" \
  -v /var/run/podman/podman.sock:/var/run/docker.sock \
  docker.io/henrygd/beszel-agent:latest
```

Replace the `KEY` value with the SSH public key shown in your Beszel hub when adding a system.

### Glance — Unified Dashboard

Single-page dashboard that ties everything together: clock, service status monitors, DNS stats from Pi-hole, server stats, and speedtest results table.

```sh
podman run -d --name glance --network host --restart unless-stopped \
  --log-opt max-size=1m \
  -v /server/glance/assets:/app/assets:ro \
  -v /server/glance/glance.yml:/app/config/glance.yml:ro \
  docker.io/glanceapp/glance:latest
```

### Watchtower — Automatic Container Updates

Automatically checks for and applies container image updates every **Sunday at 3:00 AM**. Cleans up old images after updating.

```sh
podman run -d --name watchtower --network host --restart unless-stopped \
  --log-opt max-size=1m \
  -e TZ=Europe/Warsaw \
  -e WATCHTOWER_SCHEDULE="0 0 3 * * 0" \
  -e WATCHTOWER_CLEANUP=true \
  -v /var/run/podman/podman.sock:/var/run/docker.sock \
  docker.io/containrrr/watchtower:latest \
  uptime-kuma glance beszel beszel-agent
```

> **Note:** Only the listed containers are watched. Pi-hole and Glance dashboard config container are excluded to avoid unexpected config resets. Speedtest-trigger runs as a host service (not a container).

## 12. Verify Containers

```sh
podman ps
```

You should see all containers running:

```
pihole        Up ...
uptime-kuma   Up ...
beszel        Up ...
beszel-agent  Up ...
glance        Up ...
watchtower    Up ...
```

---

# Speedtest Setup

Automated internet speed tests (download, upload, ping) every 30 minutes, with a manual trigger button on the dashboard.

## 13. Speedtest Script

Create `/server/speedtest/run.sh`:

```sh
cat > /server/speedtest/run.sh << 'SCRIPT'
#!/bin/sh
# Speedtest using official multi-threaded Ookla CLI
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export HOME=/root
LOG_DIR="/server/speedtest/logs"
JSON_FILE="/server/glance/assets/speedtest.json"
TIMESTAMP=$(date +"%Y-%m-%d_%H:%M")
LOG_FILE="$LOG_DIR/speedtest.log"

mkdir -p "$LOG_DIR"
# Skip if disk is >95% full
DISK_PCT=$(df / | awk 'NR==2 {gsub(/%/,""); print $5}')
DISK_FREE_PCT=$(( 100 - ${DISK_PCT:-0} ))
DISK_FREE=$(df -h / | awk 'NR==2 {print $4}')
if [ "$DISK_PCT" -gt 95 ] 2>/dev/null; then
  echo "$(date +"%Y-%m-%d_%H:%M") SKIPPED disk ${DISK_PCT}% full" >> "$LOG_FILE"
  exit 0
fi

# Run official speedtest CLI and output JSON
JSON_DATA=$(/usr/local/bin/speedtest-ookla --accept-license --accept-gdpr -f json 2>/dev/null)

if [ -n "$JSON_DATA" ]; then
  # Parse JSON and extract mbps, server, and url
  python3 -c "
import json
try:
    data = json.loads('''$JSON_DATA''')
    dl_mbps = data.get('download', {}).get('bandwidth', 0) * 8 / 1000000
    ul_mbps = data.get('upload', {}).get('bandwidth', 0) * 8 / 1000000
    ping = data.get('ping', {}).get('latency', 0)
    srv = f\"{data.get('server', {}).get('name', 'Unknown')} {data.get('server', {}).get('location', '')}\".strip()
    url = data.get('result', {}).get('url', '#')
    print(f'{dl_mbps:.2f}|{ul_mbps:.2f}|{ping:.2f}|{srv}|{url}')
except:
    print('ERROR')
" > /tmp/speed_res

  RES=$(cat /tmp/speed_res)
  if [ "$RES" != "ERROR" ]; then
    DL=$(echo "$RES" | cut -d'|' -f1)
    UL=$(echo "$RES" | cut -d'|' -f2)
    PING=$(echo "$RES" | cut -d'|' -f3)
    SRV=$(echo "$RES" | cut -d'|' -f4)
    URL=$(echo "$RES" | cut -d'|' -f5)
    echo "${TIMESTAMP} | ${DL} | ${UL} | ${PING} | ${SRV} | ${URL}" >> "$LOG_FILE"
  else
    echo "${TIMESTAMP} ERROR" >> "$LOG_FILE"
  fi
else
  echo "${TIMESTAMP} ERROR" >> "$LOG_FILE"
fi

# Keep 30 days of entries (every 30m = 48/day = 1440)
tail -1440 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"

# Generate JSON for Glance dashboard
python3 -c "
import json
lines = []
with open('$LOG_FILE') as f:
    for line in f:
        line = line.strip()
        if line: lines.append(line)

dl_sum = ul_sum = ping_sum = valid_count = 0
for line in lines:
    if '|' in line and 'ERROR' not in line:
        parts = [p.strip() for p in line.split('|')]
        try:
            dl_sum += float(parts[1])
            ul_sum += float(parts[2])
            ping_sum += float(parts[3])
            valid_count += 1
        except: pass

avg_dl = f'{dl_sum/valid_count:.1f}' if valid_count > 0 else '-'
avg_ul = f'{ul_sum/valid_count:.1f}' if valid_count > 0 else '-'
avg_ping = f'{ping_sum/valid_count:.0f}' if valid_count > 0 else '-'

entries = []
for line in lines[-60:]:
    if '|' in line:
        parts = [p.strip() for p in line.split('|')]
        t = parts[0]
        if 'ERROR' in line:
            entries.append({'time': t, 'download': '', 'upload': '', 'ping': '', 'server': '', 'url': '#'})
        else:
            entries.append({'time': t, 'download': parts[1], 'upload': parts[2], 'ping': parts[3], 'server': parts[4], 'url': parts[5]})
    else:
        # backward compatibility
        parts = line.split()
        t = parts[0]
        if 'ERROR' in line:
            entries.append({'time': t, 'download': '', 'upload': '', 'ping': '', 'server': '', 'url': '#'})
        else:
            dl = ul = ping = ''
            for i, p in enumerate(parts):
                if p == 'Download:' and i+1 < len(parts): dl = parts[i+1]
                if p == 'Upload:' and i+1 < len(parts): ul = parts[i+1]
                if p == 'Ping:' and i+1 < len(parts): ping = parts[i+1]
            entries.append({'time': t, 'download': dl, 'upload': ul, 'ping': ping, 'server': '-', 'url': '#'})
with open('$JSON_FILE', 'w') as f:
    json.dump({
        'entries': list(reversed(entries)), 
        'disk_pct': '$DISK_PCT', 
        'disk_free_pct': '$DISK_FREE_PCT',
        'avg_dl': avg_dl,
        'avg_ul': avg_ul,
        'avg_ping': avg_ping
    }, f)
"
SCRIPT
chmod +x /server/speedtest/run.sh
```

## 14. Manual Trigger Server

A tiny HTTP server that lets you run a speedtest from the dashboard with one click.

Create `/server/speedtest/trigger.py`:

```sh
cat > /server/speedtest/trigger.py << 'PYTHON'
import http.server, subprocess, threading

class ReuseServer(http.server.HTTPServer):
    allow_reuse_address = True
    allow_reuse_port = True

class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        host = self.headers.get("Host", "192.168.0.30:8902").split(":")[0]
        self.send_response(302)
        self.send_header("Location", "http://%s:8080/" % host)
        self.end_headers()
        threading.Thread(target=lambda: subprocess.run(
            ["sh", "/server/speedtest/run.sh"], capture_output=True
        ), daemon=True).start()
    def do_OPTIONS(self):
        self.send_response(204)
        self.send_header("Access-Control-Allow-Origin", "*")
        self.end_headers()
    def log_message(self, *a): pass

ReuseServer(("0.0.0.0", 8902), H).serve_forever()
PYTHON
```

How it works: clicking "▶ run test" on the dashboard navigates to `:8902`, the server launches the speedtest in a background thread, then immediately redirects (HTTP 302) back to the dashboard. Results appear after ~2 minutes.

## 15. Trigger OpenRC Service

Create `/etc/init.d/speedtest-trigger`:

```sh
cat > /etc/init.d/speedtest-trigger << 'EOF'
#!/sbin/openrc-run

description="Speedtest manual trigger HTTP server"
command="/usr/bin/python3"
command_args="/server/speedtest/trigger.py"
command_background="yes"
pidfile="/run/speedtest-trigger.pid"
start_stop_daemon_args="--stdout /dev/null --stderr /dev/null"

depend() {
    need net
    after podman
}
EOF
chmod +x /etc/init.d/speedtest-trigger
rc-update add speedtest-trigger default
rc-service speedtest-trigger start
```

## 16. Speedtest Cron Job

Run speedtest every 30 minutes:

```sh
crontab -e
```

Add:

```
*/30 * * * * /server/speedtest/run.sh
```

---

# Dashboard Configuration

## 17. Glance Dashboard Config

Create `/server/glance/glance.yml`:

```yaml
theme:
  background-color: 240 5 9
  primary-color: 217 80 58
  positive-color: 158 60 45
  negative-color: 0 70 55

server:
  assets-path: /app/assets

pages:
  - name: Wyse 3040
    columns:
      - size: small
        widgets:
          - type: clock
            hour-format: 24h
            timezones:
              - timezone: Europe/Warsaw
                label: Warszawa

          - type: server-stats
            servers:
              - type: local
                name: Wyse 3040

          - type: monitor
            cache: 1m
            title: Services
            sites:
              - title: Pi-hole
                url: https://<SERVER_IP>/admin
                icon: si:pihole
                allow-insecure: true
              - title: Uptime Kuma
                url: http://<SERVER_IP>:3001
                icon: si:uptimekuma
              - title: Beszel
                url: http://<SERVER_IP>:8090/system/< Beszel Agent Token >
                icon: si:statuspage
              - title: Google
                url: https://google.com
                icon: si:google

      - size: full
        widgets:
          - type: dns-stats
            service: pihole-v6
            url: https://127.0.0.1
            allow-insecure: true
            password: <YOUR_PIHOLE_PASSWORD>

          - type: custom-api
            title: Speedtest
            cache: 30s
            url: http://localhost:8080/assets/speedtest.json
            template: |
              <div style="text-align: right; margin-bottom: 4px; font-size: 0.85em; opacity: 0.9;">
                <a href="http://<SERVER_IP>:8902/" style="color: hsl(217,50%,55%); text-decoration:none; font-weight:bold;">&#9654; Run test</a>
              </div>
              <div style="font-size: 0.8em; opacity: 0.8; margin-bottom: 12px; border-bottom: 1px solid rgba(255,255,255,0.1); padding-bottom: 8px;">
                M-AVG <small style="opacity:0.6">(30 days)</small>: 
                <strong class="color-positive" style="margin-left: 6px;">{{ if ne (.JSON.String "avg_dl") "" }}{{ .JSON.String "avg_dl" }}{{ else }}-{{ end }}</strong> DL | 
                <strong style="color: hsl(280, 65%, 70%);">{{ if ne (.JSON.String "avg_ul") "" }}{{ .JSON.String "avg_ul" }}{{ else }}-{{ end }}</strong> UL | 
                <strong>{{ if ne (.JSON.String "avg_ping") "" }}{{ .JSON.String "avg_ping" }}{{ else }}-{{ end }}</strong> ms
              </div>
              {{ if .JSON.Exists "entries" }}
                {{ $entries := .JSON.Array "entries" }}
                {{ $total := len $entries }}
                {{ if eq $total 0 }}
                  <p class="color-subtext">No speedtest data yet</p>
                {{ else }}
                  <table style="width:100%; border-collapse:collapse; table-layout:fixed; font-size:0.95em;">
                    <colgroup>
                      <col style="width:16%;">
                      <col style="width:16%;">
                      <col style="width:16%;">
                      <col style="width:16%;">
                      <col style="width:36%;">
                    </colgroup>
                    <thead>
                      <tr style="border-bottom: 1px solid rgba(255,255,255,0.15);">
                        <th style="text-align:left; padding:8px 6px; font-weight:600;">Time</th>
                        <th style="text-align:right; padding:8px 6px; font-weight:600;">Download</th>
                        <th style="text-align:right; padding:8px 6px; font-weight:600;">Upload</th>
                        <th style="text-align:right; padding:8px 6px; font-weight:600;">Ping</th>
                        <th style="text-align:left; padding:8px 6px; font-weight:600; padding-left:14px;">Server</th>
                      </tr>
                    </thead>
                    <tbody>
                      {{ range $i, $e := $entries }}
                        {{ if lt $i 10 }}
                          <tr style="border-bottom: 1px solid rgba(255,255,255,0.05);">
                            <td style="padding:5px 6px; overflow:hidden; white-space:nowrap; text-overflow:ellipsis;" class="color-subtext">{{ $e.String "time" }}</td>
                            <td style="text-align:right; padding:5px 6px; overflow:hidden;">
                              {{ if eq ($e.String "download") "" }}
                                <span class="color-negative">ERROR</span>
                              {{ else }}
                                <span class="color-positive">{{ $e.String "download" }} <small style="opacity:0.6">Mb</small></span>
                              {{ end }}
                            </td>
                            <td style="text-align:right; padding:5px 6px; overflow:hidden;">
                              {{ if eq ($e.String "upload") "" }}
                                <span class="color-negative">ERROR</span>
                              {{ else }}
                                <span style="color: hsl(280, 65%, 70%);">{{ $e.String "upload" }} <small style="opacity:0.6">Mb</small></span>
                              {{ end }}
                            </td>
                            <td style="text-align:right; padding:5px 6px; overflow:hidden;">
                              {{ if eq ($e.String "ping") "" }}
                                <span class="color-negative">ERROR</span>
                              {{ else }}
                                {{ $e.String "ping" }} <small style="opacity:0.6">ms</small>
                              {{ end }}
                            </td>
                            <td style="padding:5px 6px; padding-left:14px; overflow:hidden; white-space:nowrap; text-overflow:ellipsis;">
                              {{ if ne ($e.String "server") "" }}
                                <a href="{{ $e.String "url" }}" target="_blank" style="color: inherit; text-decoration: underline; opacity: 0.8;">{{ $e.String "server" }}</a>
                              {{ end }}
                            </td>
                          </tr>
                        {{ end }}
                      {{ end }}
                    </tbody>
                  </table>
                  {{ if gt $total 10 }}
                    <details style="margin-top:6px;">
                      <summary style="cursor:pointer; color:hsl(217,50%,55%); font-size:0.85em; list-style:none; padding:4px 6px;">&#9660; show all</summary>
                      <table style="width:100%; border-collapse:collapse; table-layout:fixed; margin-top:4px; font-size:0.95em;">
                        <colgroup>
                          <col style="width:16%;">
                          <col style="width:16%;">
                          <col style="width:16%;">
                          <col style="width:16%;">
                          <col style="width:36%;">
                        </colgroup>
                        <tbody>
                          {{ range $i, $e := $entries }}
                            {{ if ge $i 10 }}
                              <tr style="border-bottom: 1px solid rgba(255,255,255,0.05);">
                                <td style="padding:5px 6px; overflow:hidden; white-space:nowrap; text-overflow:ellipsis;" class="color-subtext">{{ $e.String "time" }}</td>
                                <td style="text-align:right; padding:5px 6px; overflow:hidden;">
                                  {{ if eq ($e.String "download") "" }}
                                    <span class="color-negative">ERROR</span>
                                  {{ else }}
                                    <span class="color-positive">{{ $e.String "download" }} <small style="opacity:0.6">Mb</small></span>
                                  {{ end }}
                                </td>
                                <td style="text-align:right; padding:5px 6px; overflow:hidden;">
                                  {{ if eq ($e.String "upload") "" }}
                                    <span class="color-negative">ERROR</span>
                                  {{ else }}
                                    <span style="color: hsl(280, 65%, 70%);">{{ $e.String "upload" }} <small style="opacity:0.6">Mb</small></span>
                                  {{ end }}
                                </td>
                                <td style="text-align:right; padding:5px 6px; overflow:hidden;">
                                  {{ if eq ($e.String "ping") "" }}
                                    <span class="color-negative">ERROR</span>
                                  {{ else }}
                                    {{ $e.String "ping" }} <small style="opacity:0.6">ms</small>
                                  {{ end }}
                                </td>
                                <td style="padding:5px 6px; padding-left:14px; overflow:hidden; white-space:nowrap; text-overflow:ellipsis;">
                                  {{ if ne ($e.String "server") "" }}
                                    <a href="{{ $e.String "url" }}" target="_blank" style="color: inherit; text-decoration: underline; opacity: 0.8;">{{ $e.String "server" }}</a>
                                  {{ end }}
                                </td>
                              </tr>
                            {{ end }}
                          {{ end }}
                        </tbody>
                      </table>
                    </details>
                  {{ end }}
                {{ end }}
              {{ else }}
                <p class="color-subtext">Waiting for speedtest data...</p>
              {{ end }}
```

> **Note:** Replace all occurrences of `192.168.0.30` with your server's IP address.

After creating/editing `glance.yml`, restart the container:

```sh
podman restart glance
```

## 18. Pi-hole Log Management

To prevent disk exhaustion on the Dell Wyse 3040's limited eMMC storage, Pi-hole's long-term query database is limited to 30 days.

### Apply or verify the limit:
```sh
podman exec -it pihole sh -c "echo 'MAXDBDAYS=30' >> /etc/pihole/pihole-FTL.conf"
podman restart pihole
```

### Check currently occupied space:
```sh
podman exec -it pihole ls -lh /etc/pihole/pihole-FTL.db
```


---

# Boot & Autostart

## 18. Container Boot Script

Create `/etc/local.d/containers.start`:

```sh
cat > /etc/local.d/containers.start << 'EOF'
#!/bin/sh
# Wyse 3040 – autostart containers on boot

sysctl -w net.ipv4.ip_unprivileged_port_start=53 >/dev/null 2>&1

# Start containers (restart policy handles recovery)
podman start pihole uptime-kuma glance beszel beszel-agent watchtower 2>/dev/null

# Note: do NOT add speedtest-trigger here — it runs as an OpenRC service, not a container

exit 0
EOF
chmod +x /etc/local.d/containers.start
rc-update add local default
```

Containers are created once (step 11) with `--restart unless-stopped`. On boot, the script just starts them — no need to recreate.

## 19. Verify Boot Services

```sh
rc-status
```

Expected services in `default` runlevel:

| Service | Purpose |
|---|---|
| `sshd` | SSH access |
| `podman` | Container runtime |
| `tailscale` | VPN mesh |
| `crond` | Cron scheduler |
| `speedtest-trigger` | Manual speedtest HTTP trigger |
| `local` | Runs `/etc/local.d/containers.start` |
| `acpid` | ACPI events (power button etc.) |
| `chronyd` | NTP time sync |

---

# Storage Protection

The 8 GB eMMC has only ~5 GB usable, so storage management is critical.

## 20. Container Log Limits

All containers are created with `--log-opt max-size=1m` (each log capped at 1 MB).

This is also set globally in `/etc/containers/containers.conf`:

```
log_size_max = 1000000
```

So any future containers will also have the 1 MB limit.

## 21. Speedtest Data Retention

`run.sh` keeps only the last 48 entries (~4 KB). It also skips the test entirely if disk usage exceeds 95%.

## 22. Weekly Cleanup

A weekly cron job cleans up dangling container images and build cache:

Create `/etc/periodic/weekly/podman-prune`:

```sh
cat > /etc/periodic/weekly/podman-prune << 'EOF'
#!/bin/sh
# Remove unused images and build cache to save disk space
podman image prune -af 2>/dev/null
podman system prune -f 2>/dev/null
EOF
chmod +x /etc/periodic/weekly/podman-prune
```

Runs automatically every Saturday at 3:00 AM (via Alpine's default weekly cron).

> **Note:** Watchtower also cleans up old images (`WATCHTOWER_CLEANUP=true`) after each Sunday update run.

---

# SSH Login Banner

Replace the default Alpine welcome message with a dynamic status dashboard that shows system info, containers, and IPs on every SSH login.

## 23. Create Dynamic MOTD

Create `/etc/profile.d/motd.sh`:

```sh
cat > /etc/profile.d/motd.sh << 'SCRIPT'
#!/bin/sh
# Dynamic MOTD — Wyse 3040 server status

BOLD="\033[1m"
DIM="\033[2m"
GREEN="\033[32m"
YELLOW="\033[33m"
CYAN="\033[36m"
RED="\033[31m"
RESET="\033[0m"

# System info
UPTIME=$(uptime | sed "s/.*up //" | sed "s/,  *[0-9]* user.*//" | sed "s/^ *//")
LOAD=$(cat /proc/loadavg | awk '{print $1, $2, $3}')

# Memory
MEM_TOTAL=$(free -m | awk '/Mem:/ {print $2}')
MEM_USED=$(free -m | awk '/Mem:/ {print $3}')
MEM_PCT=$(echo "$MEM_USED $MEM_TOTAL" | awk '{printf "%d", $1 * 100 / $2}')

# Disk
DISK_USED=$(df -h / | awk 'NR==2 {print $3}')
DISK_TOTAL=$(df -h / | awk 'NR==2 {print $2}')
DISK_PCT=$(df / | awk 'NR==2 {printf "%d", $5}')

# Network
ETH_IP=$(ifconfig eth0 2>/dev/null | awk '/inet / {print $2}')
WLAN_IP=$(ifconfig mlan0 2>/dev/null | awk '/inet / {print $2}')
TS_IP=$(ifconfig tailscale0 2>/dev/null | awk '/inet / {print $2}')

# Containers
CONTAINERS=$(podman ps --format "{{.Names}}:{{.Status}}" 2>/dev/null)

# Color for percentages
color_pct() {
  if [ "$1" -gt 90 ]; then printf "${RED}";
  elif [ "$1" -gt 70 ]; then printf "${YELLOW}";
  else printf "${GREEN}"; fi
}

printf "\n"
printf "${BOLD}${CYAN}  ╔══════════════════════════════════════════╗${RESET}\n"
printf "${BOLD}${CYAN}  ║        WYSE 3040  ·  HOME SERVER         ║${RESET}\n"
printf "${BOLD}${CYAN}  ╚══════════════════════════════════════════╝${RESET}\n"
printf "\n"
printf "  ${DIM}Uptime${RESET}     %s\n" "$UPTIME"
printf "  ${DIM}Load${RESET}       %s\n" "$LOAD"
printf "\n"

MEM_COLOR=$(color_pct $MEM_PCT)
DISK_COLOR=$(color_pct $DISK_PCT)
printf "  ${DIM}Memory${RESET}     ${MEM_COLOR}%s MB / %s MB (%s%%)${RESET}\n" "$MEM_USED" "$MEM_TOTAL" "$MEM_PCT"
printf "  ${DIM}Disk${RESET}       ${DISK_COLOR}%s / %s (%s%%)${RESET}\n" "$DISK_USED" "$DISK_TOTAL" "$DISK_PCT"
printf "\n"
printf "  ${DIM}eth0${RESET}       %s\n" "${ETH_IP:-down}"
printf "  ${DIM}mlan0${RESET}      %s\n" "${WLAN_IP:-down}"
printf "  ${DIM}tailscale${RESET}  %s\n" "${TS_IP:-down}"
printf "\n"

printf "  ${DIM}CONTAINERS${RESET}\n"
if [ -n "$CONTAINERS" ]; then
  echo "$CONTAINERS" | while IFS=: read -r name status; do
    case "$status" in
      Up*) printf "  ${GREEN}●${RESET} %-15s %s\n" "$name" "$status" ;;
      *)   printf "  ${RED}●${RESET} %-15s %s\n" "$name" "$status" ;;
    esac
  done
else
  printf "  ${RED}● no containers running${RESET}\n"
fi
printf "\n"
printf "  ${DIM}Dashboard${RESET}  http://%s:8080\n" "${WLAN_IP:-$ETH_IP}"
printf "\n"
SCRIPT
```

## 24. Disable Default MOTD

```sh
echo '' > /etc/motd
```

On every SSH login you'll now see:

```
  ╔══════════════════════════════════════════╗
  ║        WYSE 3040  ·  HOME SERVER         ║
  ╚══════════════════════════════════════════╝

  Uptime     3 min
  Load       0.52 0.31 0.12

  Memory     503 MB / 1885 MB (26%)     ← green/yellow/red
  Disk       4.2G / 5.0G (88%)          ← green/yellow/red

  eth0       192.168.0.20
  mlan0      192.168.0.30
  tailscale  100.106.42.29

  CONTAINERS
  ● pihole          Up 19 hours
  ● uptime-kuma     Up 15 hours
  ● beszel          Up 13 hours
  ● beszel-agent    Up 13 hours
  ● glance          Up 12 hours
  ● watchtower      Up 12 hours

  Dashboard  http://192.168.0.30:8080
```

---

# Access

## Network Addresses

| Interface | IP | Purpose |
|---|---|---|
| `eth0` | 192.168.0.20 | Wired LAN |
| `mlan0` | 192.168.0.30 | WiFi (primary) |
| `tailscale0` | 100.106.42.29 | VPN (remote access) |

## Web Interfaces

| Service | Local URL | Tailscale URL |
|---|---|---|
| **Glance Dashboard** | http://192.168.0.30:8080 | http://100.106.42.29:8080 |
| **Pi-hole Admin** | http://192.168.0.30/admin | http://100.106.42.29/admin |
| **Uptime Kuma** | http://192.168.0.30:3001 | http://100.106.42.29:3001 |
| **Beszel** | http://192.168.0.30:8090 | http://100.106.42.29:8090 |

## SSH

```sh
ssh root@192.168.0.30       # local
ssh root@100.106.42.29      # via Tailscale
```

---

# Maintenance

## Update System

```sh
apk update && apk upgrade
```

## Update Containers

Watchtower handles this automatically every Sunday at 3:00 AM. To update manually:

```sh
podman pull docker.io/pihole/pihole:latest
podman pull docker.io/louislam/uptime-kuma:latest
podman pull docker.io/henrygd/beszel:latest
podman pull docker.io/henrygd/beszel-agent:latest
podman pull docker.io/glanceapp/glance:latest
podman pull docker.io/containrrr/watchtower:latest

# Recreate with same settings (example for beszel):
podman stop beszel && podman rm beszel
podman run -d --name beszel --network host --restart unless-stopped \
  --log-opt max-size=1m \
  -e TZ=Europe/Warsaw \
  -e APP_URL=http://localhost:8090 \
  -v beszel_data:/beszel_data \
  docker.io/henrygd/beszel:latest \
  serve --http=0.0.0.0:8090
# Repeat for other containers...
```

## Check Disk Usage

```sh
df -h /
```

## Troubleshooting

```sh
podman ps -a                  # list all containers (including stopped)
podman logs pihole            # check container logs
podman restart glance         # restart a container
rc-status                     # check OpenRC services
tailscale status              # check VPN
```

---

# Notes

- Port **53 must be free** for Pi-hole — stop any existing DNS resolvers first
- All persistent data lives in `/server/` — back up this directory to save your config
- Beszel data is stored in a Podman named volume `beszel_data` — include it in backups (`podman volume export beszel_data`)
- Tailscale provides remote access without port forwarding or dynamic DNS
- Watchtower uses a **6-field cron** format (with seconds as first field): `0 0 3 * * 0` = Sunday at 03:00:00
- With 2 GB RAM, keep services minimal — these 6 containers use ~30% of available memory
- The 8 GB eMMC is the main bottleneck — log limits and weekly pruning prevent filling it up
- Container images take ~900 MB total (beszel adds ~180 MB); keep this in mind when adding more services

## Dual Network Fallback (WiFi & Ethernet)

If you use both ethernet (`eth0`) and WiFi (`mlan0`), you can make the server automatically prioritize the wired connection while keeping WiFi as a fallback. To do this, add a `gateway` to both interfaces in `/etc/network/interfaces`, but use the `metric` parameter to set priorities (lower metric = higher priority).

```sh
auto eth0
iface eth0 inet static
        address <IP_ADDRESS_1>
        netmask 255.255.255.0
        gateway <GATEWAY_IP>
        metric 10

auto mlan0
iface mlan0 inet static
        address <IP_ADDRESS_2>
        netmask 255.255.255.0
        gateway <GATEWAY_IP>
        metric 20
```
