# journald-remote-follower-rsyslog

Forwards the journals that `systemd-journal-remote` receives to rsyslog, in
the order the entries arrived, over RELP, so that no entry is lost.

## Why

rsyslog's `imjournal` with `Remote="on"` reads the local journal and the
remote journals as one stream, merged by time. An entry that is uploaded late
(a backlog after a host was offline, a batch after `systemd-journal-upload`
reconnected) is older than the entries imjournal has already read from the
local journal. When imjournal moves on, the journal library seeks to the
first entry newer than the last one it returned, and the late entries are
skipped. They stay in the remote journal, but never reach rsyslog's files or
anything else rsyslog feeds.

A test that shows it: stop `systemd-journal-upload` on a host, log 30
numbered messages, start it again. In one run on a Debian 13 server
(rsyslog 8.2504, systemd 257), messages 1–17 and 29–30 reached rsyslog, and
18–28 did not. A host's backlog of three days was missing completely, and on
a normal day 3–20 % of the remote entries.

This program reads only the remote journals and follows them in the order
the entries were written. rsyslog acknowledges every entry (RELP), and the
position is saved only after that: delivery is at least once.

## How it works

- It reads `/var/log/journal/remote` with the systemd journal library
  (python3-systemd) and waits for new entries.
- Each entry becomes an RFC 5424 message:
  - **timestamp:** `__REALTIME_TIMESTAMP`, in local time with microseconds,
    as imjournal uses it
  - **host:** `_HOSTNAME` without the domain
  - **program:** `SYSLOG_IDENTIFIER`, else `_COMM`
  - **PID:** `SYSLOG_PID`, else `_PID`
  - **priority:** `SYSLOG_FACILITY` (default user) and `PRIORITY` (default
    notice)
- It sends the messages in batches of 500 to rsyslog's `imrelp` (default
  `127.0.0.1:2514`).
- It saves the cursor of the last entry in the state file once rsyslog
  acknowledged the batch. If sending fails, it goes back to the saved cursor
  and sends again, retrying after 1 s and up to 5 min.
- Without a state file, it starts with the entries that arrive from then on
  (`--start tail`) or at the oldest one (`--start head`).

### Requirement: one writer in the directory

Within one journal file, and across its rotated predecessors, which keep the
same sequence number ID, the entries are read in the order they were
written. Across files of different sources, the journal library compares
timestamps again, and the skipping comes back. So the directory it follows
must contain the files of a single source only:

- **all hosts upload through one address**, e.g. a TLS proxy (stunnel) in
  front of systemd-journal-remote, so `SplitMode=host` writes everything to
  `remote-127.0.0.1.journal`; or
- systemd-journal-remote writes a single file (`--split-mode=none`).

Move the files of other sources elsewhere, e.g. to
`/var/log/journal/remote-archive`. `journalctl -D` still reads them there,
and so does the backfill.

## Backfill

```sh
journald-remote-follower-rsyslog \
    -D /var/log/journal/remote -D /var/log/journal/remote-archive \
    --since 2026-10-09T00:00 --until 2026-10-12T10:00 \
    --target 127.0.0.1:2515 --dry-run
```

This sends the entries of a time range that are missing from rsyslog's
per-host files (`--hosts-dir`, default `/var/log/hosts`,
`<dir>/<host>/messages*`). An entry counts as present if a line with the
same timestamp (to the microsecond) and program is there. Without
`--dry-run` it sends the missing ones.

- It reads each journal file separately and completely, because merging by
  time would skip entries here too. Over gigabytes of journals this takes a
  while.
- Send the backfill to an input whose ruleset only writes the files (port
  2515 in the example). Otherwise old warnings and errors would be notified
  again.
- The lines are appended to the files, after newer ones.

## Installation

Requirements:

- **Python 3.7 or later** (`datetime.fromisoformat`); tested with Python
  3.13 on Debian 13. openSUSE Leap 15's default `python3` is 3.6 and too old;
  Leap 16 has 3.13.
- **python3-systemd** (the systemd journal bindings; tested with 235)
- **rsyslog with imrelp** (rsyslog-relp on Debian, rsyslog-module-relp on
  openSUSE)
- **systemd-journal-remote** writing the journals to follow

```sh
apt install python3-systemd rsyslog-relp      # Debian
zypper install python3-systemd rsyslog-module-relp   # openSUSE

install -m 755 journald-remote-follower-rsyslog /usr/local/sbin/
install -m 644 journald-remote-follower-rsyslog.service /etc/systemd/system/
install -Dm 644 README.md /usr/local/share/doc/journald-remote-follower-rsyslog/README.md
```

Then:

1. Set up rsyslog as in `examples/rsyslog-journald-remote-follower.conf`:
   - `imjournal` with `Remote="off"`
   - the `imrelp` inputs
   - rulesets that write the same lines as imjournal did
2. Move other sources' journal files out of the followed directory (see
   above), restart rsyslog, then start the service:
   ```sh
   systemctl enable --now journald-remote-follower-rsyslog.service
   ```

The service runs with `DynamicUser=yes` and the supplementary group
`systemd-journal-remote`, which can read the remote journals. Its cursor is
in `/var/lib/journald-remote-follower-rsyslog/cursor`.

## Options

```
-D, --directory DIR   journal directory (default /var/log/journal/remote;
                      backfill may take several)
--target HOST:PORT    rsyslog's imrelp input (default 127.0.0.1:2514)
--state FILE          cursor file (default /var/lib/journald-remote-follower-rsyslog/cursor)
--start tail|head     where to start without a cursor (default tail)
--batch N             entries per RELP batch (default 500)
--since TIME          backfill from TIME (ISO 8601)
--until TIME          backfill up to TIME (default now)
--hosts-dir DIR       backfill: rsyslog's per-host files (default /var/log/hosts)
-n, --dry-run         backfill: only count
```

## Checks

```sh
systemctl status journald-remote-follower-rsyslog
journalctl -u journald-remote-follower-rsyslog    # start position, send errors
```

Compare a host's entries in the journal and in its file for an hour:

```sh
journalctl -D /var/log/journal/remote --since 09:00 --until 10:00 _HOSTNAME=<host> -q -o json | wc -l
grep -c '^2026-10-12T09:' /var/log/hosts/<host>/messages
```

## Limitations

- **Duplicates:** at-least-once delivery. If rsyslog restarts between
  receiving a batch and acknowledging it, the batch is sent again.
- **Only the fields above:** the other journal fields
  are not forwarded.
- **One directory:** the follow mode reads one directory, with the files of a
  single source (see above).

## License

GPL-2.0-or-later; see `COPYING`.
