# Known Issues

Findings from a code review of `vps_maintenance.sh`. Listed by severity.

---

## Critical

### 1. Script duplicated — fails to parse
**Lines:** 652–1259  
The file contains two complete copies of the script concatenated together. Line 652 reads `fi#!/usr/bin/env bash` (the closing `fi` of the last block and the second copy's shebang fused onto one line), so bash cannot parse the file at all:

```
$ bash -n vps_maintenance.sh
vps_maintenance.sh: line 1260: syntax error: unexpected end of file
```

The two copies are different versions — lines 1–651 are newer (includes UFW orphaned-rules check, improved failed-services parser). Lines 653–1259 are an older revision.

**Fix:** Delete lines 652–1259 (keep only the first copy). Ensure the file ends with the closing `fi` on its own line.

**Root cause:** Updates appear to be pasted/appended in the GitHub web editor instead of replacing file contents. This has happened multiple times (commit `60624f2` had four copies, `9353291` had three). Adding `bash -n vps_maintenance.sh` as a pre-commit check would catch this.

---

## High

### 2. Old-kernel purge can delete the wrong kernel
**Lines:** 228–232  
`grep -v "$CURRENT_KERNEL" | head -n -1` relies on `dpkg`'s alphabetical sort order, which is not version order. Example: `linux-image-5.15.0-100` sorts before `linux-image-5.15.0-99`, so after an upgrade (before reboot) the script can purge the newly installed kernel and keep an older one.

**Fix:** Add `sort -V` before `head -n -1`:
```bash
OLD_KERNELS=$(echo "$ALL_KERNELS" | grep -v "$CURRENT_KERNEL" | sort -V | head -n -1 || true)
```

### 3. `apt-get upgrade` failures silently reported as success
**Lines:** 182–194  
The exit status of the upgrade command is never checked. A dpkg error, held packages, or a full disk will still print `OK  Package upgrade complete`. The same applies to the `autoremove` block (lines 202–208).

Additionally, because output is captured into a variable, the terminal shows nothing during the longest step — contradicting the README's "Full terminal output during every stage" claim.

**Fix:** Check `${PIPESTATUS[0]}` after the command, or scan the captured output for `^E:` lines, and call `err` on failure. Consider using `tee` to the log while also streaming to the terminal.

### 4. journalctl SSH-count fallback produces a corrupt value
**Lines:** 272–273  
```bash
FAILED_SSH=$($SUDO journalctl ... | grep -c "Failed password" || echo "0")
```
When there are zero matches, `grep -c` already prints `0` and exits 1, so the `|| echo "0"` appends a second `0`. `FAILED_SSH` becomes `"0\n0"`, and the later `(( FAILED_SSH > SSH_FAIL_WARN ))` throws an arithmetic syntax error.

**Fix:** Drop the `|| echo "0"` — `grep -c` already handles the zero case:
```bash
FAILED_SSH=$($SUDO journalctl _COMM=sshd --since "24 hours ago" 2>/dev/null \
    | grep -c "Failed password" || true)
FAILED_SSH=${FAILED_SSH:-0}
```

### 5. SSH failed-login time filter is unreliable
**Lines:** 267–269  
```bash
awk -v d="$(date --date='24 hours ago' '+%b %e')" '$0 >= d'
```
This compares full log lines lexicographically against a short date string like `"Jun  9"`. Month names don't sort chronologically (`Sep > Jun`), so entries from months ago can be included in January, and recent entries can be missed depending on the month. Ubuntu 24.04 also uses ISO timestamps in auth.log, where this comparison matches nothing useful.

**Fix:** Prefer the `journalctl --since "24 hours ago"` branch whenever `journalctl` is available, and only fall back to raw auth.log without attempting to filter by time.

---

## Medium

### 6. UFW orphaned-port check has false positives and silent gaps
**Lines:** 340–359  
- Multi-port rules like `80,443/tcp` only yield the first port; the rest are silently ignored.
- `ALLOW OUT` rules match the `/ALLOW/` filter and get flagged as orphaned inbound rules.
- App-profile rules (`OpenSSH`, `Nginx Full`) are skipped without any mention.

The "consider closing unused rules" advice can fire on ports that are legitimately open. This is a heuristic, but it should note its own limitations or be scoped to `ALLOW IN` rules with plain port numbers only.

---

## Low

### 7. `reboot-required.pkgs` list ends with a trailing comma
**Line:** 404  
```bash
REBOOT_PKGS=$(cat /var/run/reboot-required.pkgs 2>/dev/null | tr '\n' ', ' | sed 's/, $//')
```
`tr '\n' ', '` maps newlines to `,` (the space is a second character tr maps to, not a separator — all spaces in the file are also replaced). The `sed` strip then never matches because the trailing character is `,` not `, `.

**Fix:**
```bash
REBOOT_PKGS=$(paste -sd ', ' /var/run/reboot-required.pkgs 2>/dev/null)
```

### 8. `--no-color` only works as the first argument
**Line:** 19  
`[[ "${1:-}" == "--no-color" ]]` only checks `$1`. Passing it in any other position (e.g. `bash vps_maintenance.sh --no-color`) silently ignores the flag.

**Fix:** Parse arguments in a loop before the config block, or check all positional parameters.

### 9. `log()` uses `echo -e`, expanding backslashes in dynamic content
**Line:** 79  
```bash
log() { echo -e "$*" | tee -a "$LOG_FILE"; }
```
Any backslash sequences in package names, service names, or file paths passed through `log` would be interpreted. For example, a package with `\n` in its name would insert a newline.

**Fix:** Use `printf '%s\n'` instead of `echo -e`, or ensure all callers quote their input safely.

### 10. Fallback cert scan paths don't match the README
**Lines:** 495, 497  
The README states the fallback cert check scans `/etc/ssl/certs`, but the code scans `/etc/nginx/ssl`, `/etc/apache2/ssl`, and `/etc/ssl/private/*.crt`. `/etc/ssl/private` typically holds private keys, not certificates, so this glob is unlikely to match anything useful and the README is misleading.

**Fix:** Update either the code or the README to be consistent. Consider adding `/etc/ssl/certs/*.pem` to the scan paths.
