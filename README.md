# healthmon — Installation & Operations Guide

A lightweight bash-based health monitor that watches CPU, memory, disk, and systemd services, and pushes real-time alerts (and recovery notices) to Slack via webhook.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites — Install Required Packages](#2-prerequisites--install-required-packages)
3. [Create Directories](#3-create-directories)
4. [Configuration File](#4-configuration-file)
5. [The Bash Script — Explained in Detail](#5-the-bash-script--explained-in-detail)
6. [The systemd Service](#6-the-systemd-service)
7. [What is a Daemon and Why We Use One](#7-what-is-a-daemon-and-why-we-use-one)
8. [Enable and Start the Service](#8-enable-and-start-the-service)
9. [Checking Logs and Status](#9-checking-logs-and-status)
10. [Testing the Monitor](#10-testing-the-monitor)
11. [Troubleshooting](#11-troubleshooting)
12. [Tuning Recommendations](#12-tuning-recommendations)

---

## 1. Overview

`healthmon` is a small monitoring daemon written in pure Bash. It runs continuously in the background via systemd and periodically checks:

- **CPU usage** — percentage over a 1-second sample
- **Memory usage** — percentage of RAM used (excluding cache)
- **Disk usage** — per-mount percentage
- **Services** — whether configured systemd services are active

When a metric crosses its threshold, it fires an alert to a Slack webhook and suppresses repeat alerts for a configurable cooldown period. When the metric returns to normal, it sends a recovery notification.

Everything is written to a log file (`/var/log/healthmon/healthmon.log`) and to the systemd journal.

---

## 2. Prerequisites — Install Required Packages

You need three things installed:

- **`curl`** — to send the Slack webhook
- **`stress`** — only used for *testing* the alerts, not required for the daemon itself
- **`nginx`** — used as an example service to monitor and to test service-down alerts

### On Debian / Ubuntu

```bash
sudo apt update
sudo apt install -y curl stress nginx
```

### On RHEL / Rocky / AlmaLinux / Fedora

```bash
sudo dnf install -y curl stress-ng nginx
sudo systemctl enable --now nginx
```

*(On RHEL family, `stress` is called `stress-ng`. Use `stress-ng` in the test commands instead.)*

### On Arch

```bash
sudo pacman -S --noconfirm curl stress nginx
sudo systemctl enable --now nginx
```

### Verify

```bash
curl --version | head -1
stress --version 2>/dev/null || stress-ng --version
nginx -v
```

You should also have a **Slack Incoming Webhook URL**. If you don't have one yet:

1. Go to https://api.slack.com/apps → **Create New App** → **From scratch**
2. Name it `healthmon`, choose your workspace
3. **Incoming Webhooks** → toggle **On** → **Add New Webhook to Workspace**
4. Pick a channel → copy the URL (looks like `https://hooks.slack.com/services/T000/B000/XXXX`)

---

## 3. Create Directories

The daemon writes logs and state files in two separate locations:

| Path | Purpose |
|---|---|
| `/etc/healthmon/` | Configuration file |
| `/var/log/healthmon/` | Log file |
| `/var/lib/healthmon/` | State files (alert cooldowns + recovery tracking) |

Create them:

```bash
sudo mkdir -p /etc/healthmon
sudo mkdir -p /var/log/healthmon
sudo mkdir -p /var/lib/healthmon
```

The script itself will also `mkdir -p` these on startup, so this step is belt-and-braces — but doing it explicitly avoids any first-run race conditions.

You'll also want the script itself at `/usr/local/bin/`:

```bash
sudo mkdir -p /usr/local/bin
```

---

## 4. Configuration File

Create `/etc/healthmon/healthmon.conf`:

```bash
sudo nano /etc/healthmon/healthmon.conf
```

Paste:

```bash
# ---------- Thresholds (percent) ----------
CPU_THRESHOLD=85
MEM_THRESHOLD=90
DISK_THRESHOLD=90

# ---------- Timing ----------
# Seconds between each full check cycle
CHECK_INTERVAL=10

# Seconds before the same alert can fire again for the same issue
COOLDOWN_SEC=300

# ---------- Monitored services (space-separated) ----------
SERVICES="nginx sshd cron"

# ---------- Slack ----------
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/REPLACE/ME/HERE"
```

Lock it down so only root can read it (the webhook URL is a secret):

```bash
sudo chmod 600 /etc/healthmon/healthmon.conf
sudo chown root:root /etc/healthmon/healthmon.conf
```

### What each setting does

| Setting | Meaning |
|---|---|
| `CPU_THRESHOLD` | Alert if CPU ≥ this percentage |
| `MEM_THRESHOLD` | Alert if RAM used ≥ this percentage |
| `DISK_THRESHOLD` | Alert if any mount ≥ this percentage |
| `CHECK_INTERVAL` | Sleep seconds between loop iterations |
| `COOLDOWN_SEC` | Minimum gap between repeated alerts for the *same* key |
| `SERVICES` | Which systemd services to watch |
| `SLACK_WEBHOOK_URL` | Where alerts get POSTed |

---

## 5. The Bash Script — Explained in Detail

Save as `/usr/local/bin/healthmon.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

source /etc/healthmon/healthmon.conf

LOG=/var/log/healthmon/healthmon.log
STATE=/var/lib/healthmon

mkdir -p "$(dirname "$LOG")" "$STATE"
touch "$LOG"

log() { echo "$(date '+%F %T') [$1] $2" >> "$LOG"; }

slack_post() {
  local emoji=$1 msg=$2
  [[ -z ${SLACK_WEBHOOK_URL:-} ]] && return 0
  curl -s -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"${emoji} $(hostname): ${msg}\"}" \
    "$SLACK_WEBHOOK_URL" >/dev/null || log WARN "slack post failed"
}

alert() {
  local key=$1 msg=$2
  local cooldown_f="$STATE/$key.last"
  local state_f="$STATE/$key.state"

  if [[ -f $state_f ]]; then
    if [[ -f $cooldown_f ]] && (( $(date +%s) - $(cat "$cooldown_f") < COOLDOWN_SEC )); then
      return 0
    fi
  fi

  date +%s > "$cooldown_f"
  touch "$state_f"
  log CRIT "$msg"
  slack_post "🚨" "$msg"
}

mark_ok() {
  local key=$1 msg=$2
  local state_f="$STATE/$key.state"
  if [[ -f $state_f ]]; then
    log INFO "recovered: $msg"
    slack_post "✅" "$msg"
  fi
  rm -f "$state_f" "$STATE/$key.last"
}

cpu_usage() {
  local -a a b
  read -r -a a < /proc/stat
  sleep 1
  read -r -a b < /proc/stat
  local idle=$(( (b[4] + b[5]) - (a[4] + a[5]) ))
  local total=0
  local i
  for i in 1 2 3 4 5 6 7 8; do
    total=$(( total + b[i] - a[i] ))
  done
  (( total == 0 )) && { echo 0; return; }
  echo $(( 100 * (total - idle) / total ))
}

mem_usage() {
  free | awk '/Mem:/ {printf "%d", ($2-$7)/$2*100}'
}

check_cpu() {
  local v; v=$(cpu_usage)
  log INFO "cpu=${v}%"
  if (( v >= CPU_THRESHOLD )); then
    alert cpu "CPU at ${v}% (limit ${CPU_THRESHOLD}%)"
  else
    mark_ok cpu "CPU back to normal (${v}%)"
  fi
}

check_mem() {
  local v; v=$(mem_usage)
  log INFO "mem=${v}%"
  if (( v >= MEM_THRESHOLD )); then
    alert mem "Memory at ${v}% (limit ${MEM_THRESHOLD}%)"
  else
    mark_ok mem "Memory back to normal (${v}%)"
  fi
}

check_disk() {
  df -P --output=pcent,target -x tmpfs -x devtmpfs | tail -n +2 |
  while read -r pct mount; do
    pct=${pct%\%}
    local key="disk$(echo "$mount" | tr '/' '_')"
    log INFO "disk ${mount}=${pct}%"
    if (( pct >= DISK_THRESHOLD )); then
      alert "$key" "Disk ${mount} at ${pct}% (limit ${DISK_THRESHOLD}%)"
    else
      mark_ok "$key" "Disk ${mount} back to normal (${pct}%)"
    fi
  done
}

check_services() {
  local svc
  for svc in $SERVICES; do
    local key="svc_$svc"
    if systemctl is-active --quiet "$svc"; then
      log INFO "service ${svc}=active"
      mark_ok "$key" "Service ${svc} is back UP"
    else
      alert "$key" "Service ${svc} is DOWN"
    fi
  done
}

trap 'log INFO "shutting down"; exit 0' TERM INT

log INFO "healthmon daemon starting (interval=${CHECK_INTERVAL}s)"

SECONDS=0
LAST_DISK=0

while true; do
  check_cpu
  check_mem
  check_services

  if (( SECONDS - LAST_DISK >= 60 )); then
    check_disk
    LAST_DISK=$SECONDS
  fi

  sleep "$CHECK_INTERVAL"
done
```

Make it executable:

```bash
sudo chmod 755 /usr/local/bin/healthmon.sh
```

### Script walkthrough

#### Header & setup

```bash
#!/usr/bin/env bash
set -euo pipefail
```
- `-e` — exit on any error
- `-u` — error if an unset variable is referenced
- `-o pipefail` — a pipeline fails if any stage fails

```bash
source /etc/healthmon/healthmon.conf
```
Loads thresholds, timing, and the webhook URL as shell variables.

```bash
LOG=/var/log/healthmon/healthmon.log
STATE=/var/lib/healthmon
mkdir -p "$(dirname "$LOG")" "$STATE"
touch "$LOG"
```
Defines the log/state paths and creates them if missing. `touch` guarantees the log file exists so `>>` always works.

#### `log()`

```bash
log() { echo "$(date '+%F %T') [$1] $2" >> "$LOG"; }
```
Appends a timestamped line: `2026-10-10 14:00:01 [INFO] cpu=12%`.

#### `slack_post()`

```bash
slack_post() {
  local emoji=$1 msg=$2
  [[ -z ${SLACK_WEBHOOK_URL:-} ]] && return 0
  curl -s -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"${emoji} $(hostname): ${msg}\"}" \
    "$SLACK_WEBHOOK_URL" >/dev/null || log WARN "slack post failed"
}
```
Posts a JSON payload to Slack. If the webhook URL is empty, it silently returns. If the POST fails, it logs a warning but doesn't crash the daemon.

#### `alert()`

```bash
alert() {
  local key=$1 msg=$2
  local cooldown_f="$STATE/$key.last"
  local state_f="$STATE/$key.state"

  if [[ -f $state_f ]]; then
    if [[ -f $cooldown_f ]] && (( $(date +%s) - $(cat "$cooldown_f") < COOLDOWN_SEC )); then
      return 0
    fi
  fi

  date +%s > "$cooldown_f"
  touch "$state_f"
  log CRIT "$msg"
  slack_post "🚨" "$msg"
}
```

Two state files per alert key (e.g. `cpu`):

- **`cpu.state`** — exists ⇔ we are currently in an alerting state
- **`cpu.last`** — unix timestamp of the last time an alert was *sent*

The logic:
1. If we're already alerting (`cpu.state` exists) **and** the cooldown hasn't elapsed → skip (avoid spam).
2. Otherwise write the current timestamp, mark state, log `CRIT`, and POST to Slack.

#### `mark_ok()`

```bash
mark_ok() {
  local key=$1 msg=$2
  local state_f="$STATE/$key.state"
  if [[ -f $state_f ]]; then
    log INFO "recovered: $msg"
    slack_post "✅" "$msg"
  fi
  rm -f "$state_f" "$STATE/$key.last"
}
```
Called when a check returns to normal. If we were previously alerting, send a recovery message. Either way, clear state so the next alert fires fresh.

#### `cpu_usage()`

```bash
cpu_usage() {
  local -a a b
  read -r -a a < /proc/stat
  sleep 1
  read -r -a b < /proc/stat
  local idle=$(( (b[4] + b[5]) - (a[4] + a[5]) ))
  local total=0
  for i in 1 2 3 4 5 6 7 8; do
    total=$(( total + b[i] - a[i] ))
  done
  echo $(( 100 * (total - idle) / total ))
}
```
Reads `/proc/stat` twice, 1 second apart, and computes the CPU delta. Fields:
- `[4]` = idle
- `[5]` = iowait
- `[1..8]` = user, nice, system, idle, iowait, irq, softirq, steal

Percentage = `(total − idle) / total × 100`.

#### `mem_usage()`

```bash
mem_usage() {
  free | awk '/Mem:/ {printf "%d", ($2-$7)/$2*100}'
}
```
Uses `free`, computes `(total − available) / total × 100`. Field `$7` is "available" on modern `free` — this excludes reclaimable cache so the number matches what you'd call "real" memory pressure.

#### `check_cpu`, `check_mem`, `check_disk`, `check_services`

Each one:
1. Samples the metric
2. Logs the value at INFO level
3. Calls `alert` if above threshold, otherwise calls `mark_ok`

`check_disk` iterates all mounts and uses a sanitized key per mount (`disk__` → `disk___`). `check_services` uses `systemctl is-active --quiet`.

#### Main loop

```bash
trap 'log INFO "shutting down"; exit 0' TERM INT

log INFO "healthmon daemon starting (interval=${CHECK_INTERVAL}s)"

SECONDS=0
LAST_DISK=0

while true; do
  check_cpu
  check_mem
  check_services

  if (( SECONDS - LAST_DISK >= 60 )); then
    check_disk
    LAST_DISK=$SECONDS
  fi

  sleep "$CHECK_INTERVAL"
done
```

- `trap` catches `SIGTERM`/`SIGINT` (sent by `systemctl stop`) and exits cleanly with a log entry.
- The `while true` loop is the daemon body.
- CPU/mem/services run every iteration (~every 10s).
- Disk is throttled to once per 60s — disks fill slowly, no need for per-second `df`.

---

## 6. The systemd Service

Create `/etc/systemd/system/healthmon.service`:

```bash
sudo nano /etc/systemd/system/healthmon.service
```

Paste:

```ini
[Unit]
Description=Health Monitor Daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/healthmon.sh
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Line-by-line

| Directive | Purpose |
|---|---|
| `After=network-online.target` | Don't start until networking is up (Slack needs DNS/internet) |
| `Wants=network-online.target` | Pull in the network-online target if not already active |
| `Type=simple` | The process stays in the foreground; systemd tracks it directly |
| `ExecStart=...` | The command to run |
| `Restart=always` | If the script ever exits (crash or otherwise), restart it |
| `RestartSec=5` | Wait 5 seconds before restarting |
| `StandardOutput=journal` | Send stdout to the journal |
| `StandardError=journal` | Same for stderr |

There is **no `.timer` file** for this service — that was the older polling design. The daemon's own `while` loop handles scheduling now.

---

## 7. What is a Daemon and Why We Use One

A **daemon** is a long-running background process that stays alive and performs work on a schedule or in response to events.

### Why not just a timer?

Your original design used a systemd `.timer` that fired a `.service` every N seconds. That's called **polling on a schedule** and has two weaknesses:

1. **Latency** — if the timer fires every 60 seconds and something spikes at second 5, you wait up to 55 seconds for the next check.
2. **Cost per run** — each timer activation spawns a new process, sources config, sets up state, runs the check, and exits. On a 10-second interval, that's ~8600 process spawns per day, each one more expensive than a single iteration of a loop.

### What the daemon gives you

- **Lower latency** — checks every `CHECK_INTERVAL` seconds (default 10).
- **Persistent in-memory state** — counters like `LAST_DISK`, `SECONDS` live across iterations without needing files.
- **One process, many checks** — much lower overhead than repeated timer spawns.
- **Natural fit for continuous monitoring** — which is what health monitoring is.

### Trade-offs

- If the script crashes, you need `Restart=always` (which we set) to bring it back.
- A bug in the loop can stall monitoring — so `set -e` and clean trap handling matter.
- You can't "fire it once" easily; for one-off checks, run the checks manually (see Testing below).

---

## 8. Enable and Start the Service

```bash
# Reload systemd so it sees the new unit file
sudo systemctl daemon-reload

# Start now and enable at boot
sudo systemctl enable --now healthmon.service
```

Check it's running:

```bash
systemctl status healthmon.service
```

Expected output:

```
● healthmon.service - Health Monitor Daemon
     Loaded: loaded (/etc/systemd/system/healthmon.service; enabled; preset: enabled)
     Active: active (running) since ...
   Main PID: 12345 (healthmon.sh)
      Tasks: 2 (limit: ...)
     Memory: 1.2M
        CPU: 40ms
```

---

## 9. Checking Logs and Status

### Service status

```bash
systemctl status healthmon.service
```

Shows: active/inactive, main PID, memory, CPU, and last few journal lines.

### Live journal (stream)

```bash
journalctl -u healthmon.service -f
```

Follows logs in real time. Ctrl-C to exit.

### Last N journal lines

```bash
journalctl -u healthmon.service -n 50 --no-pager
```

### Journal for a time window

```bash
journalctl -u healthmon.service --since "10 min ago"
journalctl -u healthmon.service --since "2026-10-10 14:00" --until "2026-10-10 15:00"
```

### The daemon's own log file

```bash
tail -f /var/log/healthmon/healthmon.log
```

This is usually more useful than the journal because it includes every check with its measured value, not just startup/shutdown events.

### Filter for alerts only

```bash
grep '\[CRIT\]' /var/log/healthmon/healthmon.log
grep 'recovered' /var/log/healthmon/healthmon.log
```

### Current alert state

Look at which state files exist:

```bash
ls -la /var/lib/healthmon/
```

If `cpu.state` exists, CPU is currently in alert. If not, it's fine.

### Reboot persistence

```bash
systemctl is-enabled healthmon.service
```

Should print `enabled`.

---

## 10. Testing the Monitor

### Test 1 — CPU alert

```bash
# Stress all cores for 60 seconds
stress --cpu "$(nproc)" --timeout 60s
# (On RHEL: stress-ng --cpu "$(nproc)" --timeout 60s)
```

Wait ~15 seconds. You should see in the log:

```
[CRIT] CPU at 92% (limit 85%)
```

And a 🚨 message in Slack. When stress ends, you'll get:

```
[INFO] recovered: CPU back to normal (12%)
```

And a ✅ message in Slack.

### Test 2 — Memory alert

```bash
stress --vm 1 --vm-bytes 95% --timeout 30s
```

Expected: `[CRIT] Memory at 96% (limit 90%)`, then a recovery.

### Test 3 — Service down alert

```bash
sudo systemctl stop nginx
```

Within ~10 seconds:

```
[CRIT] Service nginx is DOWN
```

Bring it back:

```bash
sudo systemctl start nginx
```

You'll get:

```
[INFO] recovered: Service nginx is back UP
```

### Test 4 — Disk alert (hard to trigger safely)

Don't actually fill a real disk. Instead, temporarily lower the threshold to confirm the code path:

```bash
sudo sed -i 's/DISK_THRESHOLD=.*/DISK_THRESHOLD=1/' /etc/healthmon/healthmon.conf
sudo systemctl restart healthmon.service
sleep 65   # disk check runs once per minute
grep disk /var/log/healthmon/healthmon.log | tail
```

You should see `[CRIT] Disk / at X% (limit 1%)`. Revert:

```bash
sudo sed -i 's/DISK_THRESHOLD=.*/DISK_THRESHOLD=90/' /etc/healthmon/healthmon.conf
sudo systemctl restart healthmon.service
```

### Test 5 — Webhook sanity check

Run this directly to confirm Slack is reachable:

```bash
source /etc/healthmon/healthmon.conf
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"Manual healthmon test"}' \
  "$SLACK_WEBHOOK_URL"
```

You should see `ok` as the response and a Slack message in your channel.

### Test 6 — Restart resilience

Kill the daemon and confirm systemd restarts it:

```bash
sudo pkill -f healthmon.sh
sleep 10
systemctl status healthmon.service
```

Should show `Active: active (running)` with a new `Main PID`.

### Clearing stuck state during testing

If an alert is suppressed by the cooldown and you want to force a fresh test:

```bash
sudo rm -f /var/lib/healthmon/cpu.state /var/lib/healthmon/cpu.last
sudo systemctl restart healthmon.service
```

---

## 11. Troubleshooting

### No Slack alerts at all

1. Test the webhook manually (Test 5).
2. Check the daemon is running: `systemctl status healthmon.service`.
3. Check the log for `slack post failed`: `grep WARN /var/log/healthmon/healthmon.log`.
4. Verify DNS from the daemon's perspective: `journalctl -u healthmon.service | grep -i dns`.

### Alert fires but no recovery message

Check that `mark_ok` is being reached — look for `recovered:` in the log. If you see neither, the `.state` file may not exist (meaning the alert never fired to begin with). Confirm with:

```bash
ls /var/lib/healthmon/
```

### CPU never alerts even during stress

- Verify threshold: `grep CPU_THRESHOLD /etc/healthmon/healthmon.conf`.
- Watch the logged values: `grep 'cpu=' /var/log/healthmon/healthmon.log | tail -20`.
- A single `stress --cpu 1` on an 8-core box gives ~12% system-wide — use `--cpu $(nproc)`.

### Disk check never appears in log

`check_disk` only runs once per 60 seconds. If you restarted the daemon recently, wait a minute. Confirm with:

```bash
grep 'disk ' /var/log/healthmon/healthmon.log | tail
```

### Service flaps (up/down in quick succession)

Increase `COOLDOWN_SEC` to reduce noise. Or use `systemctl is-active` + retry logic (not included here).

### `df` complains about `--output`

Your `df` is likely aliased with `-P` in your shell (this is the exact `-P` / `--output` mutual-exclusion issue). The daemon runs without shell aliases, so it isn't affected — but if you test the `df` command manually in your shell and see an error, use `\df` or `command df` to bypass the alias.

---

## 12. Tuning Recommendations

| Setting | Low-traffic box | Busy server |
|---|---|---|
| `CHECK_INTERVAL` | `15` (fewer wake-ups) | `5` (faster detection) |
| `COOLDOWN_SEC` | `600` (10min, less noise) | `180` (3min, tighter feedback) |
| `CPU_THRESHOLD` | `85` | `75` (if baseline is high) |
| `MEM_THRESHOLD` | `90` | `85` |
| `DISK_THRESHOLD` | `85` | `85` |

To tune CPU realistically, look at the 99th percentile of logged values over a day:

```bash
grep 'cpu=' /var/log/healthmon/healthmon.log | awk -F'cpu=' '{print $2}' | tr -d '%' | sort -n | tail -20
```

Set your threshold a bit above your typical peaks — you want alerts on anomalies, not on normal load.

### Log rotation

The log grows slowly (~a few KB/day at INFO level), but over time you should rotate it. Add a logrotate config at `/etc/logrotate.d/healthmon`:

```
/var/log/healthmon/healthmon.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
    copytruncate
}
```

`copytruncate` is important — the daemon keeps the file open, so logrotate must copy then truncate in place rather than moving the file.

---

## Summary

| Component | Location |
|---|---|
| Config | `/etc/healthmon/healthmon.conf` |
| Script | `/usr/local/bin/healthmon.sh` |
| Service unit | `/etc/systemd/system/healthmon.service` |
| Log file | `/var/log/healthmon/healthmon.log` |
| State files | `/var/lib/healthmon/*.state`, `*.last` |
| Journal | `journalctl -u healthmon.service` |

Once installed, managing the daemon is standard systemd:

```bash
sudo systemctl start healthmon.service
sudo systemctl stop healthmon.service
sudo systemctl restart healthmon.service
sudo systemctl status healthmon.service
journalctl -u healthmon.service -f
```

That's the whole system. If it's working, you'll get a 🚨 the moment something goes wrong and a ✅ the moment it recovers — with no polling latency and no external monitoring infrastructure.