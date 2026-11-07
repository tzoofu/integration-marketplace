# Nightly SQLite Snapshot Backup with rclone to S3-Compatible Storage

- **category**: storage
- **provider**: rclone + `sqlite3` CLI on the host; any S3-compatible object store as the off-site copy (example: Oracle Cloud Infrastructure Object Storage via its S3 compatibility endpoint)
- **reusable**: yes — a single bash script plus a cron line; only the list of database files and paths change per app.
- **docs**: https://rclone.org/s3/ · https://www.sqlite.org/backup.html · https://www.sqlite.org/lang_vacuum.html#vacuuminto · https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/s3compatibleapi.htm

## Overview
A cron-driven shell script takes **consistent online snapshots** of one or more SQLite databases while the app keeps running. It works in WAL mode, with no downtime and no torn copies. Each snapshot is integrity-checked and bundled with the app's encryption key file into a timestamped `tar.gz`. The script keeps N days locally and copies each archive to a private S3-compatible bucket with rclone. Reach for it for a single-VM app whose state is SQLite files on local disk.

## Playbook

### Prerequisites
- On the host: `sqlite3` (`apt-get install -y sqlite3`) and rclone (`curl -fsSL https://rclone.org/install.sh | sudo bash`).
- A **private** bucket on an S3-compatible store, plus an access key/secret pair scoped to it. On OCI that means a bucket, your tenancy **namespace** and **region**, and a *Customer Secret Key* from your user profile.
- Root (or the data-dir owner) to run the job. The data directory is usually owned by the container's non-root UID.

### Setup steps
1. **Configure an rclone remote** (non-interactive). For OCI Object Storage:
   ```bash
   rclone config create offsite s3 provider Other \
     access_key_id "$ACCESS_KEY" secret_access_key "$SECRET_KEY" \
     endpoint "https://<namespace>.compat.objectstorage.<region>.oraclecloud.com" \
     region "<region>" force_path_style true no_check_bucket true
   ```
   For AWS S3 use `provider AWS`. For Cloudflare R2 use `provider Cloudflare` with the account endpoint. For Backblaze B2 use its S3 endpoint.
2. Install the script below (e.g. `/opt/app/backup.sh`, `chmod 700`).
3. Test it by hand: `APP_BACKUP_REMOTE=offsite:app-backups sudo -E /opt/app/backup.sh`, then `rclone ls offsite:app-backups`.
4. Make the remote visible to the account cron runs as. For root cron, either run `sudo rclone config` or copy `~/.config/rclone/rclone.conf` to `/root/.config/rclone/` with mode 600.
5. Schedule it daily in root's crontab (`sudo crontab -e`), at an odd minute so it doesn't collide with other on-the-hour jobs:
   ```
   17 3 * * * APP_BACKUP_REMOTE=offsite:app-backups /opt/app/backup.sh >> /var/log/app-backup.log 2>&1
   ```
6. Add a bucket **lifecycle rule** to expire remote objects after N days. The script only prunes the local copies.
7. Run the script by hand before every app upgrade. Schema migrations usually only move forward, so the pre-upgrade backup is your rollback.
8. Rehearse a restore (below) once, on a scratch machine.

### Core pattern
```bash
#!/usr/bin/env bash
# Online backup: consistent SQLite snapshots (safe while the app runs, WAL-aware) + the key file.
set -euo pipefail

DATA_DIR="${APP_DATA_DIR:-/srv/app/data}"
BACKUP_DIR="${APP_BACKUP_DIR:-/srv/app/backups}"
KEEP_DAYS="${APP_BACKUP_KEEP_DAYS:-14}"
REMOTE="${APP_BACKUP_REMOTE:-}"           # e.g. offsite:app-backups ; empty = local only
DBS=(app.db)                              # list every SQLite file the app owns

stamp="$(date -u +%Y%m%dT%H%M%SZ)"
umask 077                                  # archives contain keys: owner-only
mkdir -p "$BACKUP_DIR"
tmp="$(mktemp -d "$BACKUP_DIR/.app-$stamp.XXXXXX")"
trap 'rm -rf "$tmp"' EXIT
mkdir -p "$tmp/app-$stamp"

for db in "${DBS[@]}"; do
  src="$DATA_DIR/$db"; dst="$tmp/app-$stamp/$db"
  if [[ -f "$src" ]]; then
    # Online backup API: consistent snapshot including committed WAL content.
    sqlite3 "$src" ".timeout 10000" ".backup '$dst'"
    # Alternative: sqlite3 "$src" "VACUUM INTO '$dst'"   (compacted copy, same consistency)
    sqlite3 "$dst" "PRAGMA integrity_check;" | grep -qx ok \
      || { echo "integrity check failed: $db" >&2; exit 1; }
  else
    echo "warning: $src not found, skipped" >&2
  fi
done

# Without the encryption key, sealed credentials in the DB are unreadable after restore.
[[ -f "$DATA_DIR/secret.key" ]] && cp -p "$DATA_DIR/secret.key" "$tmp/app-$stamp/"

archive="$BACKUP_DIR/app-$stamp.tar.gz"
tar -C "$tmp" -czf "$archive" "app-$stamp"
echo "backup written: $archive ($(du -h "$archive" | cut -f1))"

if [[ -n "$REMOTE" ]]; then
  rclone copy "$archive" "$REMOTE"
  echo "uploaded to $REMOTE"
fi

# Local retention only; use a bucket lifecycle rule for the remote.
find "$BACKUP_DIR" -maxdepth 1 -name 'app-*.tar.gz' -type f -mtime +"$KEEP_DAYS" -delete
```

**Restore**
```bash
docker compose stop app                                  # never restore under a running writer
rclone copy offsite:app-backups/app-<stamp>.tar.gz /tmp/
tar -C /tmp -xzf /tmp/app-<stamp>.tar.gz
sudo rm -f /srv/app/data/app.db-wal /srv/app/data/app.db-shm   # stale WAL would be replayed onto the old file
sudo cp /tmp/app-<stamp>/{app.db,secret.key} /srv/app/data/
sudo chown -R <app-uid>:<app-gid> /srv/app/data
docker compose start app
```

**Optional client-side encryption** (the archive holds keys, so an encrypted remote is cheap insurance):
```bash
rclone config create offsite-crypt crypt remote offsite:app-backups password "$(rclone obscure "$PASSPHRASE")"
APP_BACKUP_REMOTE=offsite-crypt: /opt/app/backup.sh
```

### Env vars
- `APP_DATA_DIR`, `APP_BACKUP_DIR`, `APP_BACKUP_KEEP_DAYS`, `APP_BACKUP_REMOTE`
- rclone credentials live in `rclone.conf` (`access_key_id`, `secret_access_key`), not in env. Alternatively use `RCLONE_CONFIG_<REMOTE>_ACCESS_KEY_ID` / `RCLONE_CONFIG_<REMOTE>_SECRET_ACCESS_KEY`.

### Gotchas
- **Never `cp` a live SQLite file.** In WAL mode recent commits live in `-wal`, and a plain copy can be torn mid-write. `.backup` (the online backup API) and `VACUUM INTO` both produce a consistent snapshot while the app keeps writing. Give `.backup` a `.timeout` so it waits out a busy writer instead of failing.
- **Integrity-check the snapshot, not the source.** `PRAGMA integrity_check` on the copy proves the archive is usable, and it doesn't lock the live database.
- **Back up the key file with the database** if the app encrypts stored credentials. A database restored without its key decrypts nothing. The archive is then highly sensitive: keep it `umask 077`, put it in a private bucket, and consider an `rclone crypt` remote.
- **OCI S3 compatibility needs `force_path_style true`.** Virtual-host style URLs don't resolve on the compat endpoint. The endpoint embeds your tenancy *namespace* (not the tenancy name), and Customer Secret Keys are per user (there is a small per-user limit).
- **`no_check_bucket true`** avoids rclone trying to create or HEAD the bucket, which fails with least-privilege keys.
- **Cron user vs rclone config.** Cron runs with a minimal env and root's home. A remote configured as a normal user doesn't exist for root's cron. Reading a data dir owned by a container UID usually needs root, so configure rclone for root.
- **Delete `-wal`/`-shm` before restoring** a `.db`. A leftover WAL from the old database can be replayed onto the restored file and corrupt it.
- **Stop the app before restoring**, and run exactly one writer afterwards.
- **Remote retention is separate.** `find -mtime` only prunes local copies. Set a lifecycle or retention rule on the bucket, or the remote grows forever.
- `set -euo pipefail` plus a `trap` cleanup means a failed snapshot exits non-zero without leaving temp dirs. Point the log at a file and watch it (or ping a heartbeat URL on success) so silent failures surface.
- Large, re-downloadable media is usually left out. Add an `rclone sync "$DATA_DIR/media" "$REMOTE/media"` step if you need it.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
