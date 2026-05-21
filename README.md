# Lab 21: Monitoring Live Log Files — `tail -f`, `tail -F`, `head`

**Series:** File Operations & Shell Fundamentals · **Lab 21 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (live service verification), RHCE EX294 (Ansible run monitoring), CKA (`kubectl logs -f`), RHCA building blocks (RH342 live troubleshooting, RH358 service log streaming, RH236 brick log monitoring)  
**Prerequisite:** Labs 05–20 (navigation, listing, `cat`, `less`)  
**Time Estimate:** 30–45 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–12 practical · 13–17 advanced · 18–20 exam-realistic

---

## 🎯 Objective

By the end of this lab you will **watch log files change in real time** with the calm precision of someone who's done it a thousand times. You'll know the difference between `tail -f` and `tail -F`, how to follow rotating logs without missing a beat, how to follow multiple files at once, and how to combine `tail -f` with `grep` and `awk` for live filtering — the exact skillset that turns 2 a.m. pages into 10-minute incidents.

---

## 🧠 Concept: Logs Are Append-Only Streams

Almost every Linux log file is **append-only**: services write new lines at the bottom; old lines are never edited. That property makes "live follow" trivial — keep the file open, watch the byte position, print anything new.

```
service writes a line → kernel appends to /var/log/messages
                            │
                            └── tail -f watches the file's "end-of-file marker"
                                  └── prints the new bytes as they arrive
```

### `tail -f` vs. `tail -F` — the single most important distinction

| Variant | Watches by | Survives log rotation? | When to use |
|---|---|---|---|
| `tail -f` | **File descriptor** (inode) | ❌ No — keeps reading the rotated-away file | Short investigations on stable files |
| `tail -F` | **File name** | ✅ Yes — reopens when the name reappears | Always preferred in production |

> **Senior-engineer rule:** Default to `tail -F`. Use `-f` only when you specifically need to follow an inode through a rename.

### Companion commands

| Command | Purpose |
|---|---|
| `head FILE` | Show the **first** N lines (default 10) |
| `tail FILE` | Show the **last** N lines (default 10) |
| `tail -f FILE` | Follow, watching the file descriptor |
| `tail -F FILE` | Follow, watching the filename (reopens on rotate) |
| `journalctl -f` | Systemd's live log follower (next-gen replacement) |
| `kubectl logs -f` | Kubernetes live container logs |

---

## 📚 Command Reference

### `tail`

| Flag | Long form | Purpose |
|---|---|---|
| `-n N` | `--lines=N` | Last N lines (default 10) |
| `-n +N` | — | Skip first N-1 lines, print from line N onward |
| `-c N` | `--bytes=N` | Last N bytes |
| `-f` | `--follow=descriptor` | Follow the open FD — won't reopen on rotate |
| `-F` | `--follow=name --retry` | Follow filename — reopen on rotate, wait if missing |
| `--retry` | — | Keep retrying if file doesn't exist yet |
| `-s SEC` | `--sleep-interval=SEC` | Polling interval (default 1.0 s) |
| `--pid=PID` | — | Stop following when process PID exits |
| `-q` | `--quiet` | Suppress filename headers when following many |
| `-v` | `--verbose` | Always show headers |

### `head`

| Flag | Purpose |
|---|---|
| `-n N` | First N lines (default 10) |
| `-n -N` | All lines **except** the last N |
| `-c N` | First N bytes |
| `-q` | Suppress headers across multiple files |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **Foundation** | Watching a log while restarting a service is Day-1 muscle memory |
| **RHCSA EX200** | "Configure X, then verify it works" → `tail -F /var/log/messages` in second pane |
| **RHCE EX294** | Tail `/var/log/ansible.log` while a playbook runs against many hosts |
| **CKA** | `kubectl logs -f` mirrors `tail -F` semantics for containers |
| **RHCA — RH342 (Troubleshooting)** | Live-tail multiple logs side-by-side during war-room incidents |
| **RHCA — RH358 (Services)** | `journalctl -fu nginx` is the modern `tail -F /var/log/nginx/error.log` |
| **RHCA — RH236 (Storage)** | Brick logs roll fast — `tail -F` survives daily logrotate |

---

## 🔧 The 20 Tasks

> Each task ends with three short callouts: **Switches**, **Output decoded**, and **Troubleshoot**.

---

### Task 1 — See the last 10 lines with plain `tail`

**Purpose:** The simplest invocation. Default is the last 10 lines.

```bash
tail /etc/passwd
```

**Expected output:**

```
sshd:x:74:74:Privilege-separated SSH:/usr/share/empty.sshd:/sbin/nologin
chrony:x:993:991::/var/lib/chrony:/sbin/nologin
ec2-user:x:1000:1000:EC2 Default User:/home/ec2-user:/bin/bash
```

**Switches**

| Token | Meaning |
|---|---|
| `tail FILE` | Print the last 10 lines |

**Output decoded**

| Line | Meaning |
|---|---|
| Each `name:x:UID:…` | One user record from `/etc/passwd` |
| Last visible line | The most recently-added account (typically `ec2-user` on AWS RHEL) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Output is short (fewer than 10) | File has fewer than 10 lines — `tail` just prints what exists |
| `Permission denied` | Need `sudo`, e.g., `sudo tail /var/log/secure` |

---

### Task 2 — Print the last N lines with `-n`

**Purpose:** Tune how much history to show. `-n 50` is the everyday log "first peek."

```bash
sudo tail -n 50 /var/log/messages
```

**Switches**

| Flag | Meaning |
|---|---|
| `-n 50` | Show last 50 lines |
| `-n50` | Same (no space required) |
| `--lines=50` | Long form |

**Output decoded**

| Element | Meaning |
|---|---|
| 50 newest log entries | In file order — oldest of those 50 at top, newest at bottom |
| Timestamps | Help you tell if log is "fresh" or stale |

**Why a sysadmin needs this on RHCA RH342:** "What's the system saying right now?" → `sudo tail -n 50 /var/log/messages` is the universal sanity check.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wanted everything since boot | Use `journalctl -b` instead |

---

### Task 3 — Show the first lines with `head`

**Purpose:** `head` is `tail`'s mirror. Use it to confirm a file's header / banner / first few lines.

```bash
head /etc/passwd
head -n 5 /etc/services
```

**Expected output (truncated):**

```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
…
# /etc/services:
# $Id$
#
# Network services, Internet style
#
```

**Switches**

| Flag | Meaning |
|---|---|
| `head FILE` | First 10 lines |
| `head -n 5 FILE` | First 5 lines |
| `head -c 200 FILE` | First 200 **bytes** |

**Output decoded**

| Block | Meaning |
|---|---|
| First block | First 10 user records (`/etc/passwd` is always sorted with root first) |
| Second block | First 5 comment-prefixed lines of `/etc/services` |

**Why a sysadmin needs this:** Quickly confirm a script's shebang (`head -n 1`), or check that a file starts as expected.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Need everything **except** the last N | `head -n -10 FILE` |

---

### Task 4 — Follow a file live with `tail -f`

**Purpose:** Open the live event stream of a log. New lines appear automatically.

```bash
sudo tail -f /var/log/messages
```

In a second terminal (or `Ctrl-Z + bg` in this one), generate an event:

```bash
logger "Lab 21 — testing tail -f"
```

Back in the `tail -f` terminal, you'll see the new line appear. Press `Ctrl-C` to exit.

**Switches**

| Flag | Meaning |
|---|---|
| `-f` | **F**ollow — keep the file open and print new appends |
| `Ctrl-C` | Exits `tail -f` cleanly |

**Output decoded**

| Element | Meaning |
|---|---|
| Initial last-10 lines | History context |
| Pause | Waiting for new content |
| New `…Lab 21 — testing tail -f` | `logger` sent it to the syslog daemon, which wrote to `/var/log/messages` |

**Why a sysadmin needs this:** Confirms what your changes are actually producing — restart a service, watch the log.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Nothing appears after `logger` | Distribution uses `journalctl` only — try `journalctl -f` instead |
| Stuck — can't quit | Press `Ctrl-C`; if terminal frozen, `Ctrl-\` (SIGQUIT) |

---

### Task 5 — `tail -F` survives log rotation

**Purpose:** Production logs rotate (renamed to `messages-20260101.gz`, new empty `messages` created). `-f` keeps reading the **rotated** file — useless. `-F` reopens the new file by name.

```bash
sudo tail -F /var/log/messages
```

To simulate a rotation in a *non-production* file:

```bash
# In another terminal:
echo "before rotate" | sudo tee -a /var/log/messages
sudo mv /var/log/messages /var/log/messages.old
sudo touch /var/log/messages
echo "after rotate" | sudo tee -a /var/log/messages
```

The `tail -F` session catches both lines. With `-f`, the second line would be missed.

**Switches**

| Flag | Meaning |
|---|---|
| `-F` | Equivalent to `--follow=name --retry` |
| `--follow=name` | Watch the filename; reopen if inode changes |
| `--retry` | Keep trying if file is missing (e.g., during the brief rotate gap) |

**Output decoded**

| Element | Meaning |
|---|---|
| `tail: '/var/log/messages' has been replaced; following new file` | Status message from `tail -F` confirming reopen |

**Why a sysadmin needs this on RHCA RH358:** `logrotate` runs nightly on RHEL. Long-running `tail -f` sessions silently lose data without `-F`.

> ⚠️ **Don't experiment on real production log files.** Use a test file: `tail -F /tmp/test.log` and `echo … > /tmp/test.log` instead.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `tail -f` stops showing output after rotate | You needed `-F` |
| `tail -F` floods with retries | File path is wrong — fix the path |

---

### Task 6 — Follow multiple files at once

**Purpose:** `tail -F FILE1 FILE2 …` shows all updates in one stream, with headers separating files.

```bash
sudo tail -F /var/log/messages /var/log/secure
```

(Press `Ctrl-C` to exit.)

**Expected output (excerpt):**

```
==> /var/log/messages <==
May 21 14:39:01 host systemd[1]: Started Daily Cleanup of Temporary Directories.

==> /var/log/secure <==
May 21 14:40:12 host sshd[2401]: Accepted publickey for ec2-user from 10.0.0.5 …
```

**Switches**

| Element | Meaning |
|---|---|
| Multiple file args | `tail` follows them all, multiplexed |
| `==> FILE <==` header | Printed whenever the active file switches |
| `-q` | Suppress headers (e.g., when piping) |

**Output decoded**

| Block | Meaning |
|---|---|
| Header `==> /var/log/messages <==` | The next chunk comes from this file |
| Subsequent line(s) | Newly-appended content |

**Why a sysadmin needs this on RHCA RH342:** Correlating events across services (e.g., NetworkManager and sshd) in one pane.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Files mix without headers | You ran with `-q` or piped — switches printed when `tail` sees them as redundant |
| Want unified time-ordered view | Use `journalctl -f -u svc1 -u svc2` instead |

---

### Task 7 — Combine `tail -f` with `grep` for live filtering

**Purpose:** Watch only the events you care about. `grep --line-buffered` is the critical flag — without it, output is block-buffered and you'll see nothing until the buffer fills.

```bash
sudo tail -F /var/log/messages | grep --line-buffered -i error
```

In another terminal:

```bash
logger -p user.err "Lab 21 simulated error"
```

You'll see the error line. Other events are silently filtered.

**Switches**

| Token | Meaning |
|---|---|
| `\|` | Pipe `tail -F`'s STDOUT into `grep` |
| `grep -i` | Case-insensitive |
| `--line-buffered` | Flush after each newline — critical for live output |
| `error` | Pattern |

**Output decoded**

| Element | Meaning |
|---|---|
| Only matching lines | Everything else hidden |
| Pause | Empty stream — nothing to display |

**Why a sysadmin needs this on RHCA RH342:** "Watch only the OOM killer events" — `tail -F messages \| grep --line-buffered -i 'killed process'`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No output even when events fire | You forgot `--line-buffered` — output buffer is holding lines |
| Want multiple patterns | `grep -E --line-buffered 'error\|fail\|denied'` |

---

### Task 8 — Live-filter with `awk` and `sed`

**Purpose:** When `grep` isn't enough — e.g., you want to **reformat** each matching line.

```bash
sudo tail -F /var/log/messages | awk '/sshd/ { print strftime("%H:%M:%S"), $0; fflush() }'
```

(In another terminal:)

```bash
sudo systemctl restart sshd
```

**Expected output (excerpt):**

```
14:42:01 May 21 14:42:01 host sshd[2510]: Server listening on 0.0.0.0 port 22.
```

**Switches**

| Token | Meaning |
|---|---|
| `awk '/sshd/ { … }'` | Apply the action only to lines matching `/sshd/` |
| `strftime("%H:%M:%S")` | Prepend the current wall-clock time |
| `print $0` | The whole original line |
| `fflush()` | Flush after each line — `awk`'s analogue to `--line-buffered` |

**Output decoded**

| Column | Meaning |
|---|---|
| First field | Reception time (when **you** saw it) |
| Rest | The original log line |

**Why a sysadmin needs this on RHCA RH358:** Real services often emit events out of sequence; recording reception time helps the timeline.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Output buffered | Add `fflush()` in `awk`, `-u` in `sed`, or `stdbuf -oL` for arbitrary commands |
| Want only a single field | `awk '/sshd/ { print $5, $6 }'` |

---

### Task 9 — Stop following when a process exits with `--pid`

**Purpose:** Tail while a script runs; quit automatically when the script finishes. Indispensable for build/deploy loops.

```bash
(sleep 5 && logger "Lab 21 deploy done") &
PID=$!
sudo tail -F --pid="$PID" /var/log/messages
```

After 5 seconds, the `logger` event appears and `tail` exits on its own.

**Switches**

| Token | Meaning |
|---|---|
| `( … ) &` | Run subshell in background |
| `$!` | PID of the most recently backgrounded process |
| `--pid=PID` | Exit `tail` when PID terminates |

**Output decoded**

| Behavior | Meaning |
|---|---|
| Live log lines | Until the watched PID is gone |
| Clean shell return | No need to press `Ctrl-C` |

**Why a sysadmin needs this on RHCE EX294:** Tail Ansible's log while a playbook runs; exit automatically when the play completes.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `tail` hangs after PID exits | Some older GNU coreutils had bugs — update or use `timeout 60 tail -f …` |

---

### Task 10 — Skip the first N lines with `tail -n +N`

**Purpose:** Skip a file header. Iconic example: skip the first line of a CSV.

```bash
printf 'name,age\nalice,30\nbob,25\ncarol,40\n' > ~/people.csv
tail -n +2 ~/people.csv
```

**Expected output:**

```
alice,30
bob,25
carol,40
```

**Switches**

| Token | Meaning |
|---|---|
| `-n +2` | Start at line 2 (i.e., skip 1 line) |
| `-n +N` | Start at line N |

**Output decoded**

| Element | Meaning |
|---|---|
| Header removed | Line 1 (the CSV header) is gone |
| Data lines preserved | Original order |

**Why a sysadmin needs this on RHCE EX294:** Reading CSV inventories in scripts: `tail -n +2 hosts.csv \| while IFS=, read host group; do …; done`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Confusion with `head -n -1` | `tail -n +2` removes the **top**; `head -n -1` removes the **bottom** |

---

### Task 11 — Follow output of a long-running command

**Purpose:** Pipe a long command's STDOUT through `tail -f`-style behavior. `stdbuf -oL` keeps the pipe flushed line-by-line.

```bash
ping -c 60 127.0.0.1 | stdbuf -oL grep --line-buffered ttl | tail -n 0 -f
```

(Press `Ctrl-C` after a few seconds.)

**Switches**

| Token | Meaning |
|---|---|
| `ping -c 60 127.0.0.1` | Send 60 pings, one per second |
| `stdbuf -oL grep …` | Line-buffer the grep |
| `tail -n 0 -f` | Skip pre-existing content (none here) and follow |

**Output decoded**

| Element | Meaning |
|---|---|
| One line per ping reply | Live stream |
| `Ctrl-C` | Stops `ping`, then `grep`, then `tail` |

**Why a sysadmin needs this:** Composing live monitors from arbitrary tools.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Lines arrive in chunks | Missing `stdbuf -oL` or `--line-buffered` somewhere in the pipeline |

---

### Task 12 — Highlight matches inside a live tail

**Purpose:** `grep --color=always` makes errors jump out. Combine with `tail -F` for a colorized monitor.

```bash
sudo tail -F /var/log/messages | grep --color=always --line-buffered -iE 'error|fail|denied|warn'
```

In another terminal:

```bash
logger "this is an ERROR test"
logger "this is informational"
```

The error line appears in red; the informational one is hidden (no match).

**Switches**

| Token | Meaning |
|---|---|
| `--color=always` | Force color even when piped (default is `auto` which disables in pipes) |
| `-iE 'error\|fail\|denied\|warn'` | Case-insensitive ERE alternation |
| `--line-buffered` | Real-time output |

**Output decoded**

| Element | Meaning |
|---|---|
| Colored matched substrings | Visual alarm |
| Non-matching lines | Hidden — narrowing focus |

**Why a sysadmin needs this on RHCA RH342:** Visual triage during incidents — color is faster than reading.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Colors look broken downstream | Don't pipe further; if you must, use `less -R` |

---

### Task 13 — Tail compressed (rotated) logs with `zcat` + `tail`

**Purpose:** Yesterday's evidence is in `messages-20260120.gz`. Read just the last N lines without decompressing the whole thing to disk.

```bash
sudo zcat /var/log/messages-*.gz 2>/dev/null | tail -n 100
```

**Switches**

| Token | Meaning |
|---|---|
| `zcat` | Like `cat`, but for gzipped files |
| `*.gz` | All rotated archives |
| `2>/dev/null` | Suppress permission noise if any |
| `tail -n 100` | Last 100 lines of the combined stream |

**Output decoded**

| Element | Meaning |
|---|---|
| Last 100 log lines across archives | A peek at the most recent history that's been rotated out |

**Why a sysadmin needs this on RHCA RH342:** "When did the issue **start**?" requires reaching into rotated logs.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Order wrong (alphabetical, not chronological) | Use `sort` or specify files explicitly: `zcat $(ls -1tr /var/log/messages-*.gz)` |
| `.xz` files | Use `xzcat` instead; `.bz2` → `bzcat` |

---

### Task 14 — Modern alternative: `journalctl -f` for systemd

**Purpose:** On systemd-based distros, structured logs go to the journal. `journalctl -f` is the systemd-native live tail.

```bash
sudo journalctl -f
sudo journalctl -fu sshd
sudo journalctl -f --since '5 min ago'
```

(`Ctrl-C` to exit.)

**Switches**

| Token | Meaning |
|---|---|
| `journalctl -f` | Follow all units |
| `-u UNIT` | Filter to a specific unit (e.g., `sshd`) |
| `--since 'TIME'` | Start from a relative time |
| `--no-pager` | Useful when piping |

**Output decoded**

| Element | Meaning |
|---|---|
| Each line | One journal entry with timestamp, host, unit, message |
| Live updates | New entries appear automatically |

**Why a sysadmin needs this on RHCA RH358:** On modern RHEL, `journalctl` is often **more complete** than `/var/log/messages` because some daemons skip syslog.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Want plaintext output (no color, no escape codes) | `journalctl -f --output=cat` |
| Want JSON for machine parsing | `journalctl -f --output=json` |

---

### Task 15 — Tail container logs with `kubectl logs -f`

**Purpose:** The CKA equivalent of `tail -F`. Works identically.

```bash
kubectl logs -f deploy/nginx -n web --tail=50
kubectl logs -f pod/api-7f -c sidecar --since=10m
```

(`Ctrl-C` to stop.)

**Switches**

| Token | Meaning |
|---|---|
| `-f` | Follow |
| `--tail=N` | Last N lines from history |
| `-c CONTAINER` | Pick a sidecar inside a multi-container pod |
| `--since=10m` | Only entries from the last 10 minutes |
| `-l app=nginx` | Multi-pod follow by label |
| `--prefix` | Add pod/container prefix to each line (multi-pod) |

**Output decoded**

| Element | Meaning |
|---|---|
| Container STDOUT/STDERR | Streamed as the container writes |
| `Error from server (BadRequest): previous terminated container "nginx" in pod …` | Container restarted — use `--previous` to see the last instance |

**Why a sysadmin needs this on CKA:** Live debugging is identical to `tail -F` muscle memory.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Logs end abruptly | Container is restarting — try `--previous` |
| Multi-pod chaos | Combine `-l app=foo --prefix --max-log-requests=10` |

---

### Task 16 — Combine `head` and `tail` to extract a line range

**Purpose:** Print lines 50–60 of a file. `sed -n '50,60p'` is the canonical way; this is the `head`/`tail` equivalent.

```bash
seq 1 100 > ~/range.txt
head -n 60 ~/range.txt | tail -n 11
```

**Expected output:**

```
50
51
52
53
54
55
56
57
58
59
60
```

**Switches**

| Token | Meaning |
|---|---|
| `head -n 60` | First 60 lines |
| `\| tail -n 11` | Last 11 of those — lines 50 through 60 inclusive |

**Output decoded**

| Element | Meaning |
|---|---|
| Lines 50–60 | The target range |

**Why a sysadmin needs this:** Quick "show me lines around the error" — `head -n $((LINE+5)) /var/log/X \| tail -n 11`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Off-by-one issues | Remember: `head -n 60 \| tail -n 11` is lines 50–60; `head -n 60 \| tail -n 10` is 51–60 |

---

### Task 17 — Make `tail` wait for a file to appear with `--retry`

**Purpose:** Sometimes the log file doesn't exist yet (e.g., a service hasn't started). `tail -F` already implies `--retry`; here we make it explicit on `-f`.

```bash
sudo tail -f --retry /var/log/new-service.log
```

In another terminal:

```bash
sudo touch /var/log/new-service.log
echo "first event" | sudo tee -a /var/log/new-service.log
```

`tail` quietly retries until the file exists, then begins following.

**Switches**

| Token | Meaning |
|---|---|
| `--retry` | Keep trying when file is missing or inaccessible |
| `-f --retry` | Equivalent to `-F` when paired with `--follow=name` |

**Output decoded**

| Element | Meaning |
|---|---|
| `tail: cannot open '/var/log/new-service.log' for reading: No such file or directory` followed by retry messages | Expected — tail is polling |
| `first event` | Appears as soon as the file is written |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Retry never succeeds | Path typo or permissions — check with `ls -l` |

---

### Task 18 — Suppress headers with `-q` when piping multi-file follows

**Purpose:** Headers help humans but break scripts. `-q` strips them.

```bash
sudo tail -q -F /var/log/messages /var/log/secure | grep --line-buffered sshd
```

**Switches**

| Token | Meaning |
|---|---|
| `-q` | **Q**uiet — no `==> FILE <==` headers |
| Piping through `grep` | Filter without the noise |

**Output decoded**

| Element | Meaning |
|---|---|
| Mixed-source lines | sshd events from either file |
| No headers | Clean pipeline |

**Why a sysadmin needs this:** Building tools that emit one type of event regardless of source file.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Lost track of which file gave which line | Trade-off — drop `-q` if you need provenance |

---

### Task 19 — Build a "live counter" of matching events

**Purpose:** Show a running count of how many times a pattern hits the log per second — a poor-man's real-time metric.

```bash
sudo tail -F /var/log/messages | awk '
  /sshd/ { count++ }
  {
    now = systime()
    if (now != prev) {
      printf "[%s] sshd events this second: %d\n", strftime("%H:%M:%S", now), count
      count = 0
      prev = now
      fflush()
    }
  }
'
```

(Press `Ctrl-C` after a few minutes; trigger events with `sudo systemctl restart sshd`.)

**Switches**

| Token | Meaning |
|---|---|
| `/sshd/ { count++ }` | Increment counter for lines containing `sshd` |
| `systime()` | Current epoch seconds |
| `strftime("%H:%M:%S", now)` | Pretty time |
| `fflush()` | Flush after each printed line |

**Output decoded**

| Line | Meaning |
|---|---|
| `[14:42:01] sshd events this second: 3` | 3 sshd events appeared in that second |

**Why a sysadmin needs this on RHCA RH342:** Quick triage of "is this storm constant or bursty?"

**Troubleshoot**

| Symptom | Fix |
|---|---|
| All counts zero | Awk's regex didn't match — adjust the pattern |
| Bursty / batched output | Add `fflush()` (already in this script) and confirm `stdbuf -oL` is not needed upstream |

---

### Task 20 — Exam-style scenario: live-verify a service restart

**Task statement (RHCSA / RH358-style):** *"Restart `chronyd`. Confirm the restart is clean by live-tailing `/var/log/messages` filtered to chronyd, capturing any warnings or errors, and **automatically stopping** the tail when the restart completes."*

```bash
# Step 1: start the live filtered tail in the background, scoped to the restart.
(
  sudo tail -F /var/log/messages \
    | grep --line-buffered -E 'chronyd|chrony' \
    | grep --line-buffered -iE 'warn|err|fail|denied|start|stop|listen' &
  TAIL_PID=$!
  sleep 10
  kill "$TAIL_PID" 2>/dev/null
) &
WATCHER=$!

# Step 2: restart the service.
sudo systemctl restart chronyd
sudo systemctl status chronyd --no-pager | head -n 5

# Step 3: wait for the watcher to finish.
wait "$WATCHER"
echo "Restart verification complete."
```

**Expected output (excerpt):**

```
May 21 14:50:01 host chronyd[2733]: chronyd exiting
May 21 14:50:01 host chronyd[2740]: chronyd version 4.3 starting (…)
May 21 14:50:01 host chronyd[2740]: Loaded seccomp filter (level 2)
● chronyd.service - NTP client/server
     Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-05-21 14:50:01 UTC; 1s ago
Restart verification complete.
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `sudo tail -F` | Survive any rotation during the restart window |
| `grep -E 'chronyd\|chrony'` | Narrow the firehose to the relevant service |
| Second `grep -iE 'warn\|err\|…'` | Highlight noteworthy lines |
| Background subshell + `sleep 10 + kill` | Auto-stop the tail after the restart should be done |
| `systemctl restart chronyd` | The action under verification |
| `systemctl status … \| head -n 5` | Quick health snapshot |
| `wait "$WATCHER"` | Block until the verification window finishes |

**Output decoded**

| Line | Meaning |
|---|---|
| `chronyd exiting` | Old process shutting down cleanly |
| `chronyd version 4.3 starting` | New process up |
| `Loaded seccomp filter` | Healthy startup |
| `Active: active (running)` | systemd agrees the service is healthy |
| `Restart verification complete.` | Our watcher confirms the window closed without errors |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Watcher never returns | The `sleep 10 && kill` didn't fire — check that the parens grouping is intact |
| Need longer window | Increase `sleep 10` |
| Want to fail loudly on errors | Replace second `grep` with: `awk '/error\|fail/ { print; exit 1 }'` |
| Prefer `journalctl` | Replace `tail -F /var/log/messages \| grep chrony` with `journalctl -fu chronyd --since '1 sec ago'` |

---

## 🔍 Live-Log Decision Guide

```
"Show me the last 10 lines"              → tail FILE
"Show me the last N lines"               → tail -n N FILE
"Show me everything after line N"        → tail -n +N FILE
"Watch a file change"                    → tail -F FILE             (always -F)
"Survive log rotation"                   → tail -F                  (-f does NOT)
"Watch many files"                       → tail -F F1 F2 …
"Filter live"                            → tail -F F | grep --line-buffered PATTERN
"Stop when a script finishes"            → tail -F --pid=PID F
"Live container logs"                    → kubectl logs -f …
"Live systemd unit logs"                 → journalctl -fu UNIT
"Read rotated archives"                  → zcat *.gz | tail -n N
"Get a specific line range"              → head -n END F | tail -n RANGE
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 `tail FILE` shows last 10 lines
- [ ] 02 `tail -n 50 FILE` tunes count
- [ ] 03 `head FILE` and `head -n N FILE` for top lines
- [ ] 04 `tail -f FILE` follows live
- [ ] 05 `tail -F FILE` survives rotation
- [ ] 06 Multi-file follow with `tail -F F1 F2`
- [ ] 07 Pipe through `grep --line-buffered` for live filter
- [ ] 08 Pipe through `awk` with `fflush()`
- [ ] 09 `--pid=PID` auto-stops when a process ends
- [ ] 10 `tail -n +N` skips header lines
- [ ] 11 Build a live monitor from another long-running command
- [ ] 12 `grep --color=always --line-buffered` for highlighted live triage
- [ ] 13 `zcat` rotated archives into `tail`
- [ ] 14 `journalctl -f` as the systemd-native equivalent
- [ ] 15 `kubectl logs -f` is the Kubernetes equivalent
- [ ] 16 Combine `head` and `tail` to extract a line range
- [ ] 17 `--retry` waits for a file that doesn't exist yet
- [ ] 18 `-q` strips multi-file headers for cleaner piping
- [ ] 19 Build a per-second event counter with `awk`
- [ ] 20 Exam combo: live-verify a service restart with auto-stop

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Using `-f` instead of `-F` in production | Stops seeing events after nightly `logrotate` | Always `-F` for long-running follows |
| Forgetting `--line-buffered` on `grep` | Live filter shows nothing | Add `--line-buffered` (or `stdbuf -oL`) |
| `tail`ing a binary file | Garbage on screen | Check with `file FILE` first |
| Pressing `Ctrl-Z` instead of `Ctrl-C` | Process suspended, terminal hung | Use `fg` then `Ctrl-C` |
| Tailing `/var/log/messages` while it's been deleted | `tail -f` keeps reading nothing | Use `-F`; or restart with the correct filename |
| Pipes losing color | Plain output downstream | `--color=always` on source |
| Forgetting `sudo` | `Permission denied` on `/var/log/secure` | `sudo tail …` |
| Following on a network filesystem | Lag, false negatives | Use `journalctl` or the service's structured logs |

---

## 📌 Exam Strategy

**RHCSA EX200**
- Keep a second SSH session open with `sudo tail -F /var/log/messages` during every config change.
- For services: `sudo tail -F /var/log/messages /var/log/secure` covers most of the surface.

**RHCE EX294 (Ansible)**
- `sudo tail -F /var/log/ansible.log` on the control node while playbooks run.
- For per-host debug: `tail -F /var/log/messages` on the target via `ansible all -a 'tail -F /var/log/messages'` is risky — prefer SSH sessions.

**CKA**
- `kubectl logs -f deploy/X --tail=100 --since=10m` is the workhorse.
- `kubectl logs -f -l app=foo --max-log-requests=10` for multi-pod live follows.

**RHCA**
- RH342: `journalctl -fu UNIT --since '5 min ago' \| grep --line-buffered -iE 'warn\|err'` is the universal triage move.
- RH358: combine `tail -F /var/log/$SERVICE/access.log /var/log/$SERVICE/error.log` for web tier work.
- RH236: GlusterFS brick logs rotate fast — always `-F`.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 19 — `cat` | Static counterpart to live `tail -f` |
| Lab 20 — `less` | `less +F` is the interactive cousin of `tail -F` |
| Lab 22 — `grep` | The standard filter to pair with `tail -F` |
| Lab 24 — `sed` | Live transformations on a follow stream |
| Lab 25 — `awk` | Live counters and per-field transformations |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
