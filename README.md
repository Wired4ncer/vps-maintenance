# vps_maintenance.sh

A single-script monthly maintenance tool for Ubuntu VPS servers. Run it once a month, confirm your sudo password once at the start, and get a full health + maintenance report at the end.

Tested on Ubuntu 20.04, 22.04, and 24.04.

---

## Features

- Single sudo prompt at the start — no interruptions during the run
- Full terminal output during every stage so you can see what's happening
- Saves a timestamped log and summary report to `/var/log/vps_maintenance/`
- Plain ASCII output — works correctly on any terminal or SSH client
- Auto-detects color support — degrades gracefully on dumb terminals
- Clean terminal restore after the run — no broken stdin/cursor issues

---

## What it does

The script runs 8 stages in sequence:

### 1. Health Snapshot (before)
- Uptime
- Load average — warns if above `cores x 2`
- Memory usage — warns above 90%
- Disk usage per partition — warns above 80%

### 2. Package Updates
- Refreshes the apt package index
- Shows how many packages are available to upgrade
- Runs `apt upgrade` and reports exactly how many were upgraded
- Uses non-interactive mode with safe config conflict handling (`--force-confdef`)

### 3. Cleanup
- `apt autoremove` — removes unneeded dependency packages
- `apt autoclean` + `apt clean` — clears the local package cache
- Old kernel removal — purges outdated kernels, keeping current + one backup
- Journal vacuum — deletes systemd logs older than 30 days
- `/tmp` cleanup — removes files not accessed in 7+ days

### 4. Security Checks
- Failed SSH login attempts in the last 24 hours — warns if over 50
- Accounts with empty passwords
- All currently listening TCP/UDP ports
- UFW firewall status — warns if not active
- UFW orphaned rule detection — warns about allowed ports that have no active listener (rules left from removed services)
- Whether `unattended-upgrades` is installed

### 5. Services Check
- Lists any systemd services in a failed state
- Detects whether a reboot is required after kernel/lib updates

### 6. SSL Certificate Check
- Verifies `certbot.timer` is active (auto-renewal enabled)
- Runs `certbot renew --dry-run` to confirm renewals would succeed
- Shows days remaining for every managed certificate — warns under 30 days, errors if expired
- Falls back to scanning `/etc/nginx/ssl/`, `/etc/apache2/ssl/` with `openssl` if certbot is not installed

### 7. Health Snapshot (after)
- Disk usage across all partitions post-cleanup
- Memory usage after cleanup
- Lets you see how much space was recovered

### 8. Final Report
A summary table printed to the terminal and saved to `/var/log/vps_maintenance/report_<timestamp>.txt` showing pass/warn/fail for every task and an overall status line.

---

## Usage

```bash
# Basic run — prompts for sudo once
bash vps_maintenance.sh

# If already root
sudo bash vps_maintenance.sh

# Force plain output on terminals with broken color/encoding
bash vps_maintenance.sh --no-color
```

---

## Deploying to a server

The easiest way is a single curl command run directly on the VPS -- no cloning required. It downloads the script, makes it executable, and creates a symlink in your home folder so you can run it with `./vps_maintenance.sh` from anywhere:

```bash
sudo curl -fsSL https://raw.githubusercontent.com/Wired4ncer/vps-maintenance/main/vps_maintenance.sh -o /usr/local/sbin/vps_maintenance.sh && sudo chmod +x /usr/local/sbin/vps_maintenance.sh && ln -sf /usr/local/sbin/vps_maintenance.sh ~/vps_maintenance.sh
```

To deploy or update all your servers at once from your local machine:

```bash
for VPS in user@vps1.example.com user@vps2.example.com user@vps3.example.com; do
    echo ">>> Deploying to $VPS"
    ssh $VPS "sudo curl -fsSL https://raw.githubusercontent.com/Wired4ncer/vps-maintenance/main/vps_maintenance.sh -o /usr/local/sbin/vps_maintenance.sh && sudo chmod +x /usr/local/sbin/vps_maintenance.sh && ln -sf /usr/local/sbin/vps_maintenance.sh ~/vps_maintenance.sh"
done
```

Re-running the deploy command on an existing server will update the script and refresh the symlink cleanly.

To pin to a specific commit instead of always pulling the latest from main:

```bash
sudo curl -fsSL https://raw.githubusercontent.com/Wired4ncer/vps-maintenance/COMMIT_HASH/vps_maintenance.sh -o /usr/local/sbin/vps_maintenance.sh && sudo chmod +x /usr/local/sbin/vps_maintenance.sh && ln -sf /usr/local/sbin/vps_maintenance.sh ~/vps_maintenance.sh
```

---

## Scheduling with cron

To run automatically on the 1st of each month at 3am:

```bash
sudo crontab -e
```

Add:
```
0 3 1 * * /usr/local/sbin/vps_maintenance.sh --no-color >> /var/log/vps_maintenance/cron.log 2>&1
```

> **Note:** When running via cron, always use `sudo crontab -e` (root's crontab) rather than a user crontab, so no sudo password is needed.

---

## Configuration

All thresholds are set at the top of the script:

| Variable | Default | Description |
|---|---|---|
| `DISK_WARN_PERCENT` | `80` | Warn if any partition exceeds this % |
| `MEM_WARN_PERCENT` | `90` | Warn if memory usage exceeds this % |
| `LOAD_WARN_MULTIPLIER` | `2` | Warn if load avg exceeds cores x this |
| `SSH_FAIL_WARN` | `50` | Warn if failed SSH logins in 24h exceeds this |
| `CERT_WARN_DAYS` | `30` | Warn if any certificate expires within this many days |

---

## Logs and reports

Every run saves two files to `/var/log/vps_maintenance/`:

| File | Contents |
|---|---|
| `maintenance_<timestamp>.log` | Full output of every stage |
| `report_<timestamp>.txt` | Summary table with pass/warn/fail per task |

The exact `cat` command with the full timestamped filename is printed at the bottom of every report, so you can copy-paste it directly after each run.

To list all past runs:
```bash
ls -lt /var/log/vps_maintenance/
```

To view the full log of a specific run:
```bash
cat /var/log/vps_maintenance/maintenance_<timestamp>.log
```

To view just the summary report:
```bash
cat /var/log/vps_maintenance/report_<timestamp>.txt
```

To verify what packages were actually upgraded:
```bash
cat /var/log/apt/history.log | tail -50
```

---

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Completed successfully (warnings are OK) |
| `1` | Completed with errors — review the log |

---

## Requirements

- Ubuntu 20.04 / 22.04 / 24.04
- `bash` 4.0+
- `sudo`
- Standard Ubuntu tools: `apt`, `df`, `free`, `ss`, `systemctl`, `journalctl`
- Optional: `certbot` (for SSL checks), `ufw` (for firewall checks)

---

## Notes

- `dist-upgrade` is intentionally excluded. It can remove packages to resolve dependencies and is not safe to run unattended on production servers. Stick to `apt upgrade` for monthly maintenance.
- The script keeps the current kernel plus one previous kernel as a safety backup when cleaning old kernels.
- The sudo keepalive runs fully detached from the terminal (`</dev/null >/dev/null`) so it cannot interfere with stdin.
