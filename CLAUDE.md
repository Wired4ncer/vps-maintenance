# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single-file monthly maintenance script for Ubuntu VPS servers. The entire tool is `vps_maintenance.sh` — there are no other source files, dependencies, or build steps.

## Running the script

```bash
# Basic run (prompts for sudo once)
bash vps_maintenance.sh

# Already root
sudo bash vps_maintenance.sh

# Force plain ASCII output (for cron, broken terminals)
bash vps_maintenance.sh --no-color
```

## Viewing logs from a run

```bash
# List all past runs
ls -lt /var/log/vps_maintenance/

# Full output log
cat /var/log/vps_maintenance/maintenance_<timestamp>.log

# Summary report only
cat /var/log/vps_maintenance/report_<timestamp>.txt

# Verify which packages were actually upgraded
cat /var/log/apt/history.log | tail -50
```

## Deploying to a server

```bash
# Single server
sudo curl -fsSL https://raw.githubusercontent.com/Wired4ncer/vps-maintenance/main/vps_maintenance.sh \
  -o /usr/local/sbin/vps_maintenance.sh && \
  sudo chmod +x /usr/local/sbin/vps_maintenance.sh && \
  ln -sf /usr/local/sbin/vps_maintenance.sh ~/vps_maintenance.sh
```

Re-running the deploy command on an existing server updates the script in place.

## Architecture

### Error handling model

The script uses `set -uo pipefail` but intentionally omits `set -e`. Individual check failures (grep returning no matches, optional binaries missing) must not abort the whole run. Instead, every operation calls `err()` or `warn()` which increment global `ERRORS` and `WARNINGS` counters. The script always runs to completion and exits 0 for warnings, 1 for errors.

### Result accumulation pattern

Every completed task calls `result "Task Name" "OK/!!/XX  detail"`. The `result()` function writes into an associative array `RESULTS[]` and appends to `RESULT_ORDER[]` to preserve insertion order. The final report iterates `RESULT_ORDER` to print a consistent summary table regardless of which stages ran.

### Output routing

Almost every output line goes through `log()` which calls `tee -a "$LOG_FILE"`, keeping terminal and log file in sync. The final report renders twice via `print_report()`: once in `color` mode (ANSI codes, to terminal) and once in `plain` mode (no ANSI, to log and report files) — because ANSI codes in log files break grep/cat.

### Sudo keepalive

After the initial `sudo -v`, a background subshell runs `sleep 50; sudo -n true` in a loop so the sudo timestamp never expires mid-run. It is spawned with `</dev/null >/dev/null` so it cannot interact with stdin. The `trap` on EXIT/INT/TERM/HUP kills it and restores terminal state.

### UFW orphaned rule detection

Stage 4 cross-references `ufw status verbose` allowed ports against `ss -tlnu` active listeners. Any UFW-allowed port with no matching listener is flagged as an orphaned rule (a firewall hole left by a removed service).

### SSL certificate check

If certbot is present: checks `certbot.timer` is active, runs `certbot renew --dry-run`, and parses `certbot certificates` for expiry dates. Falls back to scanning `/etc/nginx/ssl/`, `/etc/apache2/ssl/`, and `/etc/ssl/private/` with `openssl x509 -enddate` if certbot is not installed.

## Configurable thresholds

All tunable values are variables at the top of the script:

| Variable | Default | Meaning |
|---|---|---|
| `DISK_WARN_PERCENT` | `80` | Warn if any partition exceeds this % |
| `MEM_WARN_PERCENT` | `90` | Warn if memory usage exceeds this % |
| `LOAD_WARN_MULTIPLIER` | `2` | Warn if load avg > cores × this |
| `SSH_FAIL_WARN` | `50` | Warn if failed SSH logins in 24h exceed this |
| `CERT_WARN_DAYS` | `30` | Warn if any cert expires within this many days |

## Compatibility

- Ubuntu 20.04, 22.04, 24.04 only
- bash 4.0+ required (uses associative arrays)
- `dist-upgrade` is intentionally excluded — it can remove packages and is not safe unattended
- Old kernel removal keeps current kernel + one previous as a safety net
