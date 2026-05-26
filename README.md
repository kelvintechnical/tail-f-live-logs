# Lab: Monitoring Live Log Files — `tail -f`

**Series:** linux-ops-mastery — RHCSA Text File Management
**Subjects covered:** `tail`, `tail -f` (follow), `tail -F` (follow with re-open across rotation), `tail -n N` / `-c N`, `head`, multi-file follow, combining with `grep --line-buffered`, the difference between `tail -f` and `less +F`, log-rotation gotchas (`logrotate`, `copytruncate`), file-descriptor vs name-based following, `journalctl -f` as the systemd equivalent, `inotifywait` for arbitrary file watches
**Career arcs covered:** RHCSA (interpret `/var/log/*` during practical tasks), RHCE (watch `ansible-playbook` log files), SRE (live troubleshooting, incident triage), DevOps (CI/CD pipeline log streaming), AI/MLOps (training run console logs, loss curves)
**Prerequisite:** Lab 20 (less / more)
**Time Estimate:** 25 to 30 minutes
**Difficulty arc:** Task 1 foundation (`tail FILE`, `tail -n`) · 2 follow mode (`-f`) · 3 follow across rotation (`-F`) · 4 multi-file follow + grep filtering · 5 `journalctl -f` and `inotifywait` · 6 RHCSA exam-realistic capstone

---

## Objective

Watch a log file change in real time so you can correlate user actions to system behavior **as it happens**. By the end of this lab you can follow any log (including rotated ones), filter for the messages you care about, and know when to use `tail -f` versus `journalctl -f`.

The capstone is an exam-realistic prompt: *"In one terminal follow `/var/log/secure` filtered to lines containing `Failed password`. In a second terminal, attempt an SSH login as a non-existent user. Confirm the failure appears live and copy the matching line to `/root/secure-failures.log`."*

> **Lab safety note:** All operations are read-only on system logs. Writing the captured line to `/root` requires `sudo`.

---

## Concept: `-f` Means "Don't Close — Keep Reading As It Grows"

`tail FILE` prints the last N lines (10 by default) and exits. `tail -f FILE` prints the last N lines **and then stays open, polling for new data**. Every time something appends to the file, `tail -f` prints it.

```
   FILE on disk ───► tail -f keeps the file open ───► your terminal
        ▲                                              (lines as they appear)
        │
        └─ another process appends → tail prints the new bytes
```

The crucial distinction:

```
   tail -f  follows the FILE DESCRIPTOR   (file is "the same disk inode I opened")
   tail -F  follows the FILE NAME         (re-open on rotation; survives logrotate)
```

> **Why this matters:** When `logrotate` renames `/var/log/messages` to `/var/log/messages-20260526` and creates a fresh `/var/log/messages`, `tail -f` keeps reading the *old* (now-renamed) file and never sees new entries. `tail -F` notices the new file and resumes following it. **Use `-F` on production logs.**

---

## 📜 Why `tail -f` Exists — The Story

`tail` originates from Unix V6 (1975); the `-f` "follow" option was added by BSD in the early 1980s when sysadmins started managing larger systems with continually-growing logs. Before `-f`, watching a log meant `tail FILE`, waiting, `tail FILE` again, comparing — painful.

By the 1990s, `tail -f /var/log/messages` was the canonical "what is my server doing right now" command. `-F` (capital, GNU extension) followed in the 2000s to survive `logrotate`.

`journalctl -f` (systemd, 2010s) is the modern equivalent for binary journal logs — same idea (follow as it grows) but reads structured journal records instead of text files.

> **The point of the story:** `tail -f` is the original "live tail." Every newer observability tool (Splunk live tail, Datadog Live Tail, `kubectl logs -f`, `journalctl -f`) is a network-aware reimplementation of this single Unix flag.

---

## 👪 The Tail Family — Who Lives There

| Tool | Notes |
|---|---|
| `tail FILE` | Last 10 lines and exit |
| `tail -n 50 FILE` | Last 50 lines |
| `tail -n +50 FILE` | From line 50 to end |
| `tail -c 100 FILE` | Last 100 bytes |
| `tail -f FILE` | Follow by descriptor (Ctrl-C to stop) |
| `tail -F FILE` | Follow by name; survives rotation |
| `tail --pid=PID -f FILE` | Stop following when PID exits |
| `tail -f FILE1 FILE2` | Multi-file follow (banner separates each) |
| `head FILE` | First 10 lines |
| `less +F FILE` | Pager in follow mode (Ctrl-C to step out, `/` to search) |
| `journalctl -f` | Follow systemd journal |
| `journalctl -fu SERVICE` | Follow only one unit |
| `kubectl logs -f POD` | Follow pod log (same idea, over network) |
| `docker logs -f CONTAINER` | Follow container log |
| `inotifywait -m FILE` | Notify on any filesystem event, not just append |
| `multitail FILE1 FILE2` | Side-by-side follow (requires install) |

### `tail` flag cheat sheet

| Flag | Meaning |
|---|---|
| `-n N` | Last N lines |
| `-n +N` | From line N to end |
| `-c N` | Last N bytes |
| `-f` | Follow descriptor |
| `-F` | Follow name (re-open on rotate) |
| `-q` | Quiet — no header banner in multi-file mode |
| `-v` | Verbose — always print header |
| `--pid=PID` | Exit when PID terminates |
| `-s SECONDS` | Polling interval (default 1.0) |
| `--retry` | If file missing, retry (combine with `-F`) |

> **The point of the family tree:** `tail -F` is the version you'll use 95% of the time on real systems. Memorize the capital `F`.

---

## 🔬 The Anatomy of `tail -F` — In One Diagram

```
$ sudo tail -F /var/log/secure
==> /var/log/secure <==                           ← header banner (only on first or after switch)
May 26 14:01:23 host sshd[1234]: Accepted password for kelvin
May 26 14:01:30 host sudo[2345]:    kelvin : TTY=pts/0 ; PWD=/home/kelvin ; ...
(...waits for more lines...)                      ← cursor sits here; new lines appear in real time
tail: '/var/log/secure' has become inaccessible: No such file or directory     ← logrotate moved it
tail: '/var/log/secure' has appeared;  following new file                       ← -F re-opened it
==> /var/log/secure <==
May 26 14:05:01 host sshd[3456]: Failed password for invalid user nobody
```

> **Reading rule:** "Has appeared; following new file" is the magic line that tells you `-F` saved your live tail across a `logrotate` run.

---

## 📚 tail Reference Table

| Task | Command | Notes |
|---|---|---|
| Last 10 lines | `tail FILE` | |
| Last N lines | `tail -n 50 FILE` | |
| From line N | `tail -n +50 FILE` | |
| Last N bytes | `tail -c 1024 FILE` | |
| Follow descriptor | `tail -f FILE` | Stops on rotation |
| Follow name | `tail -F FILE` | Survives rotation — **default for prod** |
| Quiet multi-file | `tail -qF FILE1 FILE2` | No banners |
| Stop when PID exits | `tail --pid=PID -f FILE` | Good in scripts |
| Filter while following | `tail -F FILE \| grep --line-buffered PAT` | Without `--line-buffered`, grep buffers and you wait forever |
| Follow systemd journal | `journalctl -f` | |
| Follow one service | `journalctl -fu sshd` | |
| Last 100 lines + follow | `tail -n 100 -F FILE` | Common combo |
| Follow on remote host | `ssh host 'tail -F /var/log/messages'` | |
| Quit | `Ctrl-C` | |

> **Rule one of tail:** Use `tail -F` on real logs. Use `--line-buffered` when piping into `grep`. Use `journalctl -f` for systemd-managed services.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Exam tasks often say "check that SSH is working" — `tail -F /var/log/secure` + a login attempt is the quickest proof. |
| **RHCE candidate** | Tail playbook log files in one pane; run `ansible-playbook -vv` in the other. |
| **SRE / Platform** | Live tail during incident war rooms is the difference between "we see the error" and "we'll get back to you." |
| **DevOps** | `kubectl logs -f` and `docker logs -f` are the same UX. |
| **AI / MLOps** | Long training runs: `tail -F training.log \| grep -E 'loss\|epoch'`. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **last N → follow → multi-file → rotate-safe** muscle memory.

---

### Task 1 — `head` and basic `tail`

```bash
head /etc/services
tail /etc/services
tail -n 20 /etc/services
tail -n +500 /etc/services | head    # from line 500 onward, first 10
tail -c 200 /etc/services
```

---

### Task 2 — `tail -f` for live following

```bash
# Terminal A: follow
sudo tail -f /var/log/messages

# Terminal B: generate a log line
logger "test: hello from $(whoami) at $(date -Is)"

# Back in Terminal A you'll see the new line appear; Ctrl-C to exit.
```

---

### Task 3 — `tail -F` survives rotation

```bash
sudo -i

# Make a fake rotating log
mkdir -p /root/rotlab
cd /root/rotlab
echo "line 1" > app.log

# Terminal A:
tail -F /root/rotlab/app.log

# Terminal B (simulate logrotate):
mv /root/rotlab/app.log /root/rotlab/app.log.1
echo "line 2 in new file" > /root/rotlab/app.log
echo "line 3" >> /root/rotlab/app.log

# Terminal A should report "has appeared; following new file" and print lines 2 & 3.
# Try the same with -f instead of -F to see -f miss the rotated file.

# Cleanup
rm -rf /root/rotlab
exit
```

---

### Task 4 — Multi-file follow + grep filter

```bash
sudo tail -F /var/log/messages /var/log/secure
# Header banners separate the two files.

# Filter for sshd messages live
sudo tail -F /var/log/secure | grep --line-buffered sshd

# Filter for any "error" / "fail" / "denied" (case-insensitive)
sudo tail -F /var/log/messages | grep --line-buffered -Ei 'error|fail|denied'
```

**Reading it left to right:** `--line-buffered` tells `grep` to flush its output after each line instead of waiting for a full block buffer; without it, the live tail appears to hang.

---

### Task 5 — `journalctl -f` and `inotifywait`

```bash
# systemd journal live follow
sudo journalctl -f
# Trigger an event in another terminal:
sudo systemctl restart chronyd 2>/dev/null || sudo systemctl restart sshd

# One service only
sudo journalctl -fu sshd

# Watch any filesystem event, not just appends
which inotifywait || sudo dnf -y install inotify-tools >/dev/null
inotifywait -m /etc/passwd
# In another terminal: sudo touch /etc/passwd
# inotifywait will print MODIFY / ACCESS / CLOSE events.
# Ctrl-C to exit.
```

---

### Task 6 — Capstone: live-tail failed SSH and capture matches

**Task statement:** Follow `/var/log/secure` for "Failed password" lines, trigger one by attempting `ssh nosuchuser@localhost`, and append the matching line to `/root/secure-failures.log`.

```bash
sudo -i

# Pane A (foreground in one terminal): live tail + grep, also tee into capture file
tail -F /var/log/secure \
  | grep --line-buffered 'Failed password' \
  | tee -a /root/secure-failures.log

# Pane B (in a second terminal or split): generate the failure
ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 nosuchuser@localhost true 2>/dev/null
# Type any password and press Enter; ssh will fail.

# Pane A should print the matching line and append it to /root/secure-failures.log.
# Ctrl-C in Pane A to stop.

# Verify the captured file
tail -n 5 /root/secure-failures.log
wc -l /root/secure-failures.log
```

**Expected verification output (approx.):**

```text
May 26 14:30:11 host sshd[12345]: Failed password for invalid user nosuchuser from ::1 port 60022 ssh2
1 /root/secure-failures.log
```

**Cleanup**

```bash
rm -f /root/secure-failures.log
exit
```

---

## 🔍 Live-Tail Decision Guide

```
What are you watching?
  ├── "A plain text log on disk"               → tail -F FILE
  ├── "A log that gets rotated"                → tail -F FILE   (NOT -f)
  ├── "A systemd-managed service"              → journalctl -fu SERVICE
  ├── "Several logs side by side"              → tail -F FILE1 FILE2 …
  ├── "I want to also search interactively"    → less +F FILE   (Ctrl-C, /PAT, then `F` again to resume)
  ├── "I want filter-while-following"          → tail -F FILE | grep --line-buffered PAT
  ├── "I want any filesystem event"            → inotifywait -m FILE
  ├── "A container log"                        → kubectl logs -f POD   (or docker logs -f)
  └── "Stop when a build finishes"             → tail --pid=$(pgrep build) -f build.log
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 `head` / `tail` / `tail -n` / `tail -c`
- [ ] 02 `tail -f` follows a live log
- [ ] 03 `tail -F` survives rotation; `-f` does not
- [ ] 04 Multi-file follow + `grep --line-buffered`
- [ ] 05 `journalctl -f`, `journalctl -fu UNIT`, `inotifywait -m`
- [ ] 06 Capstone — capture live "Failed password" lines into a file

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Used `-f` on a log that rotates | Tail goes silent after midnight | Use `-F` |
| Piped to `grep` with no `--line-buffered` | Output appears hung | Add `--line-buffered` |
| Permission denied on `/var/log/secure` | Need root | `sudo tail -F ...` |
| Used `tail` on binary journal | Garbled output | `journalctl -f` instead |
| `tail -f` shows old content forever after `cp --truncate` | logrotate `copytruncate` mode keeps descriptor but resets size | `-F` re-opens the file |
| Closed terminal — follow died | Standard; that's intentional | Use `tmux` / `screen` / `nohup` for persistent follow |
| Followed two files but cannot tell them apart | Header banner only on switch | Add `-v` to force per-line headers, or use `multitail` |
| Filter matches but does not print live | Buffering | `grep --line-buffered` or `awk` with `system("")` flush |
| Stop key did not work | Caught in pager | If using `less +F`, hit Ctrl-C then `q` |
| Mixed up `head` and `tail` | Wrong end of file | Mnemonic: `head` reads from the top |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Whenever an exam task says "verify the change took effect," default to `tail -F` on the relevant log while you trigger the action.

**RHCE candidate**
- `tail -F` `ansible.log` plus `-vv` verbosity uncovers idempotency bugs fast.

**SRE / Platform interview**
- "Walk me through how you'd debug a 5xx spike." → `tail -F` the access log; `grep --line-buffered ' 5[0-9][0-9] '`; correlate with `journalctl -f` from the app service.

**DevOps**
- `kubectl logs -f deploy/api --tail=200` is the cloud-native version of this lab.

**AI / MLOps**
- `tail -F train.log | grep --line-buffered -E 'loss|val_acc'` keeps a clean training dashboard in one pane.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 20 — Scrolling Through Large Files | `less +F` is the interactive cousin |
| Lab 22 — Filtering Text with grep | `grep --line-buffered` glues to `tail -F` |
| Lab 23 — Diff Comparing Files | After capture, `diff` snapshots |
| Lab — `journalctl` *(later)* | systemd-native live tail |
| Lab 41 — File Ownership | Logs you must `sudo` to read |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
