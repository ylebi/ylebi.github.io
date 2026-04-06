+++
title = "SQLite3 Backup Strategies That Actually Work in Production"
date = "2026-04-06"
image = "/images/sqlite3-backup-strategies.webp"
tags = [
  "sqlite",
  "devops",
  "backup",
  "shell",
  "databases",
]
categories = [
  "Development",
]
+++

SQLite gets dismissed as "the toy database." Then you look at where it actually runs: every iOS and Android app, every Electron desktop tool, every edge function that can't afford a Postgres round-trip, every CI pipeline that needs a local store without a server. [OpenCode](https://opencode.ai), the AI coding assistant I use daily, keeps its entire session and conversation history in a single SQLite file at `~/.local/share/opencode/opencode.db`. Mine currently sits at 286 MB and represents months of agent conversations, project context, and tool outputs I'd rather not lose.

The problem with SQLite backups is they look trivially easy right up until they silently corrupt your data. A `cp` against a live database is not a backup — it's a gamble. WAL mode (Write-Ahead Logging), which SQLite uses by default in many modern applications, makes the situation messier: you might have outstanding transactions in a `-wal` sidecar file that your `cp` will cheerfully ignore.

This post walks through a production-grade backup and restore setup. The scripts and Makefile I'm describing are real, running in my `~/.config/opencode/` config repo right now.

## The Project Structure

```
~/.config/opencode/
├── Makefile
├── scripts/
│   ├── backup-db.sh
│   └── restore-db.sh
└── backups/
    └── opencode.db.gz    ← tracked via Git LFS
```

This layout reflects a deliberate philosophy: treat your tooling configuration like application code. Shell scripts are versioned, the Makefile is the public interface, and the backup artifact itself is stored in the repo under Git LFS — so `git pull` on a new machine gets you the last known-good database state in a single command.

If that structure feels like overkill for a local tool, consider that your OpenCode session history, your AGENTS.md context, your custom slash commands — all of that lives and dies with this database. A clean backup strategy takes twenty minutes to set up and has saved me at least twice from accidental `rm`.

## Why `sqlite3 .backup` and Not `cp`

Before looking at the scripts, this decision needs an explanation because it comes up in every discussion of SQLite backups.

When you `cp` a SQLite database file:

- If WAL mode is active, the `-wal` file may contain transactions not yet checkpointed into the main database file. Your copy of the `.db` is inconsistent without it.
- If any write is mid-flight, you can copy a file in a partially written state.
- You get no guarantee of atomicity across the main file, the `-wal`, and the `-shm` (shared memory) sidecar.

The `.backup` command, exposed via `sqlite3`'s command-line shell, uses the [SQLite Online Backup API](https://www.sqlite.org/backup.html) internally. It acquires a shared lock, copies pages incrementally, handles any concurrent writes by re-reading modified pages, and produces a fully consistent, self-contained snapshot. WAL checkpoints are handled for you. The result is a valid database file you can immediately open, inspect, and restore from.

The rule is simple: **never `cp` a SQLite file that could be open by another process**. Use `.backup`.

## The Backup Script

```bash
#!/usr/bin/env bash
# Uses sqlite3 `.backup` for a consistent, WAL-safe snapshot instead of plain cp.
set -euo pipefail

DB_SOURCE="$HOME/.local/share/opencode/opencode.db"
REPO_ROOT="$(git -C "$(dirname "$0")" rev-parse --show-toplevel)"
BACKUP_FILE="$REPO_ROOT/backups/opencode.db.gz"
TMP_DB="/tmp/opencode-backup-$$.db"

if [ ! -f "$DB_SOURCE" ]; then
  echo "Error: $DB_SOURCE not found." >&2
  exit 1
fi

mkdir -p "$REPO_ROOT/backups"

echo "→ Snapshotting $DB_SOURCE ..."
sqlite3 "$DB_SOURCE" ".backup '$TMP_DB'"

echo "→ Compressing to backups/opencode.db.gz ..."
gzip -c "$TMP_DB" > "$BACKUP_FILE"
rm -f "$TMP_DB"

SIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
echo "✓ Backup complete: backups/opencode.db.gz ($SIZE)"
echo ""
echo "Stage and commit:"
echo "  git add backups/opencode.db.gz && git commit -m 'chore: update db backup'"
```

### Step-by-Step Breakdown

**`set -euo pipefail`**

This is non-negotiable in any production shell script. `-e` aborts on any non-zero exit code, `-u` treats unset variables as errors, and `-o pipefail` propagates failures through pipelines instead of masking them. Without these three, your script will silently continue past a failed `.backup` call and happily compress an empty temp file.

**`REPO_ROOT="$(git -C "$(dirname "$0")" rev-parse --show-toplevel)"`**

The script finds the repository root dynamically using the location of the script itself, not `$PWD`. This means it works correctly whether you run it as `bash scripts/backup-db.sh`, `make backup-db`, or from a cron job with an unrelated working directory. Anchoring to `$0` instead of `$PWD` is what separates a script that works everywhere from one that only works at your desk.

**`TMP_DB="/tmp/opencode-backup-$$.db"`**

`$$` expands to the current shell's PID. The temp file is namespaced to this process, which prevents collisions if two backup jobs somehow run simultaneously, and ensures cleanup is targeting exactly your temp file. The final `rm -f "$TMP_DB"` always fires because it comes after the `gzip` step that runs under `set -e` — if `gzip` fails, the script aborts before the `rm`, but that's fine because the next run will use a different PID-based name anyway.

**`sqlite3 "$DB_SOURCE" ".backup '$TMP_DB'"`**

This is the critical line. The `.backup` command writes a consistent snapshot to `$TMP_DB` using the Online Backup API. Note the single quotes around `$TMP_DB` inside the double-quoted outer string — this is SQLite's shell syntax, not Bash quoting. The snapshot is written to `/tmp`, not directly to the backups directory, to keep the intermediate uncompressed file off the repo filesystem.

**`gzip -c "$TMP_DB" > "$BACKUP_FILE"`**

The `-c` flag writes to stdout, which is redirected to the target file. This avoids `gzip` creating a `.db.gz` alongside the original in `/tmp` and immediately deleting the input. Using stdout redirection gives you explicit control over the output path and makes the compression step transparent in the pipeline.

For a 286 MB database, gzip typically brings this down to around 40–60 MB depending on data entropy. For databases with lots of repeated text (conversation history, code), compression ratios are excellent.

## The Restore Script

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO_ROOT="$(git -C "$(dirname "$0")" rev-parse --show-toplevel)"
BACKUP_FILE="$REPO_ROOT/backups/opencode.db.gz"
RESTORE_TARGET="$HOME/.local/share/opencode/opencode.db"
SAFETY_COPY="${RESTORE_TARGET}.pre-restore-$(date +%Y%m%d%H%M%S)"

if [ ! -f "$BACKUP_FILE" ]; then
  echo "Error: no backup found at backups/opencode.db.gz" >&2
  exit 1
fi

if [ -f "$RESTORE_TARGET" ]; then
  echo "→ Saving existing DB to $(basename "$SAFETY_COPY") ..."
  cp "$RESTORE_TARGET" "$SAFETY_COPY"
fi

echo "→ Decompressing and restoring ..."
gunzip -c "$BACKUP_FILE" > "$RESTORE_TARGET"
echo "✓ Restored to $RESTORE_TARGET"
```

### What the Restore Script Gets Right

**Safety copy before overwrite.** Before anything is overwritten, the current live database is copied to a timestamped sidecar: `opencode.db.pre-restore-20260406143022`. This is a raw `cp` — which is fine here because the safety copy is a snapshot of the currently closed file, not a backup being taken under load. If the restore fails partway through or turns out to be the wrong backup, your last known state is sitting right there.

**`gunzip -c "$BACKUP_FILE" > "$RESTORE_TARGET"`** streams decompression to stdout and writes directly to the target path. The existing file is not truncated until `gunzip` successfully opens the archive. If the `.gz` file is corrupted, `gunzip` exits non-zero, `set -e` aborts the script, and the `>` redirect never fires — the original file survives. This is the correct way to restore: decompress to stdout, redirect to destination, never modify the destination until decompression succeeds.

**WAL cleanup.** One thing to be aware of after a restore: if the application left behind a stale `-wal` or `-shm` file from before the restore, SQLite will attempt to apply it on next open. The restore script writes a clean database file but does not automatically remove the WAL sidecars. In practice, for OpenCode this is a non-issue because the app creates fresh WAL files on startup. But if you're restoring a database that will be opened by a long-running process with WAL mode active, manually removing the `-wal` and `-shm` files before starting the application is the correct precaution.

## Using Makefile to Simplify SQLite Backups

```makefile
.PHONY: backup-db restore-db

backup-db:
	@bash scripts/backup-db.sh
	@git add backups/opencode.db.gz

restore-db:
	@bash scripts/restore-db.sh
```

Eight lines. This is the right size for a Makefile that wraps shell scripts.

The `backup-db` target does two things: runs the backup, then immediately stages the compressed file with `git add`. This is intentional. The workflow is:

```bash
make backup-db
git commit -m "chore: update db backup"
git push
```

Staging is automated; committing and pushing are explicit human decisions. You don't want a script auto-committing on your behalf — you want to control when backup state is recorded in history. The Makefile gives you the convenience of a one-command snapshot without removing your agency over the Git timeline.

The `@` prefix on each command suppresses the echoed command line. You see output from the scripts themselves (the `→` and `✓` lines), not the Makefile boilerplate.

Why Makefile and not just an alias or a shell function? A few reasons:

- **Discoverability.** Anyone who clones this repo sees `make backup-db` listed as a target. There's nothing to find in `~/.bashrc`.
- **CI/CD integration.** `make backup-db` works identically on your laptop and in a GitHub Actions job without any environment setup.
- **Composability.** You can add pre/post steps (lint, test, notify) to the Makefile target without changing the scripts. `make backup-db` can become `make backup-db push-lfs notify-slack` with three lines.

## SQLite Backup Methods Compared

| Method | WAL-safe | Works on live DB | Atomic | Notes |
|---|---|---|---|---|
| `sqlite3 .backup` | Yes | Yes | Yes | Recommended for all production use |
| `cp` | No | No | No | Only safe if DB is closed and no WAL |
| `VACUUM INTO` | Yes | Yes | Yes | Compacts while copying; slower |
| SQLite Backup API (C/Go) | Yes | Yes | Yes | Same as `.backup`, programmatic |
| `rsync` | No | No | No | Same failure modes as `cp` |

`VACUUM INTO 'file.db'` is worth knowing about — it's an SQL command that creates a defragmented copy. Useful if your database has grown large due to deletions and you want to reclaim space. It's slower than `.backup` on very large files, and it's a full rewrite, not an incremental copy. For routine backups, `.backup` is faster and simpler.

## Best Practices for SQLite Backups

**WAL mode is your friend but requires awareness.** WAL mode is generally better for concurrent reads, and it's what enables `.backup` to work without blocking writers. But if you shut down a WAL-mode database without a checkpoint, the data lives in the `-wal` file, not the main `.db`. The `.backup` command handles this correctly by calling a checkpoint internally. Raw `cp` does not.

**Verify your backups.** After each backup, validate the file is a real SQLite database:

```bash
# Test decompression and integrity in one shot
gunzip -c backups/opencode.db.gz | sqlite3 :memory: ".dump" > /dev/null && echo "OK"
```

The Makefile doesn't include this step currently because the `.backup` command itself guarantees a consistent file. But for unattended cron backups, adding an integrity check to the script is worth the extra seconds.

**Don't store backups only on the same disk as the live database.** The setup above commits backups to a Git repo pushed to a remote — that's your offsite copy. If you're not using Git for this, replicate to S3 or a different host:

```bash
aws s3 cp backups/opencode.db.gz \
  s3://your-bucket/opencode/opencode-$(date +%Y%m%d).db.gz
```

**Test restores.** The restore script has been run. I know it works because I've tested it, not because I assume it works. A backup you've never restored is a hypothesis, not a guarantee. Run `make restore-db` on a staging copy of your setup at least once.

**Frequency depends on write rate.** OpenCode's database gets written on every agent interaction. For active development sessions, a daily backup is probably insufficient. A pre-push hook that warns if the live database is newer than the last committed backup is the right forcing function. You get reminded every time you push code.

**Use Git LFS for large binary backups.** The `.gitattributes` in this setup tracks the compressed backup as:

```
backups/opencode.db.gz filter=lfs diff=lfs merge=lfs -text
```

Without LFS, a 40–60 MB compressed database in your config repo would bloat the Git object store with every commit. Set up LFS before your first commit if the backup artifact is large.

## Automating SQLite Backups

### Cron

For local development machines:

```cron
# Backup OpenCode DB every 4 hours
0 */4 * * * cd ~/.config/opencode && make backup-db >> ~/.local/share/opencode/backup.log 2>&1
```

Note `cd ~/.config/opencode` before `make` — the Makefile's `git add` step needs to run from within the repo. Alternatively, use the `-C` flag: `make -C /Users/yourname/.config/opencode backup-db`.

### Docker

If you're running an application with a SQLite database inside a container, mount the backup destination as a volume and exec the backup via a sidecar:

```yaml
services:
  app:
    image: myapp
    volumes:
      - db_data:/data

  db-backup:
    image: alpine
    command: >
      sh -c "apk add --no-cache sqlite && while true; do
        sqlite3 /data/app.db \".backup '/backups/app-\$\$(date +%Y%m%d%H%M%S).db'\";
        sleep 14400;
      done"
    volumes:
      - db_data:/data:ro
      - ./backups:/backups
```

### Kubernetes CronJob

For edge workloads or applications running in Kubernetes with a SQLite database on a PersistentVolumeClaim:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sqlite-backup
  namespace: myapp
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: alpine:3.21
              command:
                - /bin/sh
                - -c
                - |
                  set -euo pipefail
                  apk add --no-cache sqlite aws-cli
                  TIMESTAMP=$(date +%Y%m%d%H%M%S)
                  TMP_DB="/tmp/backup-$$.db"
                  sqlite3 /data/app.db ".backup '$TMP_DB'"
                  gzip -c "$TMP_DB" | aws s3 cp - "s3://$BACKUP_BUCKET/app-${TIMESTAMP}.db.gz"
                  rm -f "$TMP_DB"
              env:
                - name: BACKUP_BUCKET
                  valueFrom:
                    secretKeyRef:
                      name: backup-config
                      key: s3-bucket
              volumeMounts:
                - name: db-storage
                  mountPath: /data
                  readOnly: true
          volumes:
            - name: db-storage
              persistentVolumeClaim:
                claimName: app-db-pvc
```

Mounting the PVC as `readOnly: true` in the backup job is important — the `.backup` command only needs read access, and removing write permission from the backup job prevents an accidentally misconfigured script from touching your live data.

## Common Mistakes

**Copying the file while the app is running.** This is the most common mistake and the hardest to notice because it often works fine until it doesn't. SQLite is resilient enough that a partial copy will sometimes open without error but with missing or corrupted rows. Use `.backup`.

**Forgetting the `-wal` file.** If you insist on using `cp`, you must also copy the `.db-wal` and `.db-shm` files atomically. This is nearly impossible to do correctly without quiescing the database first. The path of least resistance is to use `.backup` and let SQLite handle it.

**Never testing a restore.** I've inherited backup systems where the restore script had a bug that made it produce empty databases. Months of backups, all useless. Test your restore. Automate the test if you want to be thorough.

**Storing backups on the same disk.** If the disk fails, you lose the database and all backups simultaneously. The Git remote or S3 copy is your actual backup. The local `.gz` is just a fast-access cache.

**Not monitoring backup freshness.** A backup job that silently fails is worse than no backup job, because it creates false confidence. If you're running cron or a Kubernetes CronJob, alert on job failure and monitor the age of the last successful backup.

**Ignoring WAL files after restore.** After restoring a `.db` file, leftover `-wal` or `-shm` files from the previous database instance will confuse SQLite. Clean them up:

```bash
rm -f "${RESTORE_TARGET}-wal" "${RESTORE_TARGET}-shm"
```

## FAQ

**How do I safely backup a SQLite3 database?**

Use `sqlite3 your.db ".backup '/path/to/backup.db'"`. This uses the SQLite Online Backup API, which is WAL-aware, handles concurrent writes, and produces a consistent snapshot regardless of whether the database is actively in use.

**Is copying the SQLite file enough?**

Only if the database is completely closed and no WAL file exists. For any database that might be open by another process — which includes most production and developer tooling scenarios — a raw `cp` is unsafe. It can silently produce a corrupted or incomplete copy.

**What is WAL mode and how does it affect backups?**

WAL (Write-Ahead Logging) is a SQLite journaling mode where writes go to a separate `-wal` file before being checkpointed into the main database. It improves read concurrency but means the main `.db` file alone doesn't represent the current database state. The `.backup` command handles WAL correctly; raw file copies do not.

**How often should I backup SQLite?**

It depends on how much data you can afford to lose. For a database that gets written continuously (like a session store or AI tool history), every few hours is reasonable. For configuration databases that change rarely, daily or on-change is fine. The right answer is always: more frequently than you'd regret.

**Can I backup SQLite to S3?**

Yes. After running `.backup` to a local temp file, compress it with `gzip` and push with the AWS CLI: `gzip -c backup.db | aws s3 cp - s3://bucket/backup-$(date +%Y%m%d).db.gz`. The Kubernetes CronJob example above shows this exact pattern.

**Do I need Git LFS for database backups in a Git repo?**

If the compressed backup is small (under a few MB), plain Git handles it fine. Above that, Git LFS is strongly recommended. Committing large binaries without LFS bloats the repository's object store permanently — even if you later delete the file, the objects remain in history.

## Try This in OpenCode

Want OpenCode to set this up for you? Here's a prompt you can use:

```
Create a SQLite backup system for my OpenCode database at ~/.config/opencode/ with:
1. backup-db.sh using sqlite3 .backup with gzip compression to backups/opencode.db.gz
2. restore-db.sh with timestamped safety copy before restore
3. Makefile with backup-db and restore-db targets
4. .gitattributes for Git LFS tracking of backups/opencode.db.gz
```

---

The scripts in `~/.config/opencode/scripts/` are twenty-eight total lines of meaningful Bash. The Makefile is eight. The entire backup strategy fits in your terminal history. That's the point: SQLite backup is not a complex problem, but it does have specific failure modes that aren't obvious until you've hit them. Get the fundamentals right — `.backup` not `cp`, WAL awareness, safety copy on restore, offsite storage — and you have a durable setup that will run unattended for years.
