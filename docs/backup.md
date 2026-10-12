# Backups and restore

A backup is one archive file, `curulia-YYYYMMDD-HHMMSS.tar.gz`, containing:

- `curulia.db`: a consistent snapshot of the database, taken with SQLite `VACUUM INTO` while
  the portal keeps running;
- `uploads/`: the floor-plan images (optional, on by default);
- `manifest.json`: the portal version, the schema version and the creation time.

The archive does **not** contain `secret.key` or the Kerberos keytab.

## Settings

**Administration → Backups**:

| Setting | Default | |
|---|---|---|
| Schedule | daily | `daily` at a set time (server time), every N hours, or off |
| Local copies to keep | 7 | archives in `data/backups`; older ones are deleted after each backup |
| Include floor-plan images | on | |
| Encryption passphrase | (none) | when set, archives are encrypted and end in `.tar.gz.enc` |
| Upload to S3 | off | endpoint, TLS, region, bucket, key prefix, access key, secret key, path-style URLs |
| Copies to keep in S3 | 30 | older archives under the prefix are deleted after each upload |
| Delete S3 copies older than | 0 days | 0 = keep by count only; the newest copy is never deleted |

Use **Back up now** to test the setup. **Test S3 connection** checks the saved S3 settings.
The page lists recent backups with their size and status, and local copies you can download.
Downloads are recorded in the audit log.

If a backup fails, super admins get an email (if email is on and the *backup failed* event is
enabled), and the failure appears on the Backups and System status pages. The local copy is
still made when only the S3 upload fails.

## Encryption

With a passphrase, the archive is encrypted with AES-256-GCM. The key is derived from the
passphrase with Argon2id and a random salt. The file is encrypted in 1 MiB chunks with a
marker on the last chunk, so a truncated or modified file is detected when it is restored.

**Keep the passphrase outside the server**, for example in a password manager. Without it an
encrypted backup cannot be restored. The portal stores it encrypted with `secret.key`, which
is not in the backup either.

**Encrypted database.** If the database is encrypted at rest (see
[installation](install.md#database-encryption)), the snapshot in the archive stays encrypted with
the database key. It is never written to disk unencrypted. The archive's `manifest.json` records
this (`db_encrypted`). Such a backup can only be restored with **both** the passphrase and the
database key.

## S3-compatible storage

This works with AWS S3, MinIO, Ceph RGW, Wasabi, Backblaze B2 (S3 API), Yandex Object Storage
and others.

- **Endpoint**: host and optional port, without `https://`. For example `s3.amazonaws.com`,
  `s3.eu-central-1.amazonaws.com`, `minio.corp.local:9000` or `storage.yandexcloud.net`.
- **Path-style URLs**: needed by MinIO and some others.
- **Key prefix** (default `curulia/`): retention only deletes objects under this prefix whose
  names match `curulia-YYYYMMDD-HHMMSS.tar.gz[.enc]`. Other objects are never touched.

Use a private bucket, ideally with server-side encryption and versioning or object lock.
Create a key that can only access this bucket. AWS IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::my-curulia-backups" },
    { "Effect": "Allow", "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::my-curulia-backups/curulia/*" }
  ]
}
```

If you use object lock or want the bucket's lifecycle rules to handle retention, set *Copies to
keep in S3* high. The portal then does not try to delete locked objects.

## Command line

```sh
# Same as "Back up now": local archive (rotated) + S3 upload when enabled
curulia backup --data /var/lib/curulia

# One archive to a file of your choice (encrypted if a passphrase is set; --plain to skip)
curulia backup --data /var/lib/curulia --out /mnt/usb/curulia.tar.gz.enc
```

`curulia backup` can run while the portal is running. It refuses to work on a database whose
schema is from a different version of the portal.

## Restore

Stop the portal first; restore refuses to run while the portal holds `data/curulia.lock`.
Then:

```sh
sudo systemctl stop curulia
sudo -u curulia curulia restore --data /var/lib/curulia /var/lib/curulia/backups/curulia-20260925-020000.tar.gz.enc
sudo systemctl start curulia
```

Restore:

1. reads the archive (local file or `s3://bucket/key`);
2. decrypts it if it is encrypted, verifies it completely and runs an SQLite integrity check,
   before changing anything;
3. moves the current `curulia.db` (with `-wal`/`-shm`) and `uploads/` to
   `data/restore-previous-<time>/`;
4. puts the restored database and images in place.

You can undo a restore by moving the files back from `restore-previous-<time>`.

Options:

| | |
|---|---|
| `--yes` | do not ask for confirmation (required when not run on a terminal) |
| `--passphrase-stdin` | read the passphrase from standard input; alternatively set `CURULIA_BACKUP_PASSPHRASE`, or type it when asked |
| `--db-key-file` | the database key, for backups of an encrypted database; alternatively `CURULIA_DB_KEY`, `CURULIA_DB_KEY_FILE` or a key built into the binary |
| `--s3-endpoint`, `--s3-region`, `--s3-access-key`, `--s3-path-style`, `--s3-no-tls` | S3 connection for `s3://` sources; the secret key comes from `CURULIA_S3_SECRET_KEY`. Without `--s3-endpoint`, the S3 settings of the existing database in `--data` are used. |

A backup from an older version is upgraded automatically when the portal starts.

With a database key configured, restoring an unencrypted backup encrypts the restored database
with that key. Restoring an encrypted backup without the right key stops with an error before
anything is changed.

### Restoring onto a new server

1. Install the same or a newer version of the portal (see [installation](install.md)).
2. Copy `secret.key` from the old server into the new data directory, if you have it. If the
   database was encrypted, configure the same database key (see
   [installation](install.md#database-encryption)).
3. Run `curulia restore` with the archive and passphrase:

   ```sh
   CURULIA_S3_SECRET_KEY=… curulia restore --data /var/lib/curulia --yes \
       --s3-endpoint s3.amazonaws.com --s3-access-key AKIA… --passphrase-stdin \
       s3://my-curulia-backups/curulia/curulia-20260925-020000.tar.gz.enc < passphrase.txt
   ```

4. Without the old `secret.key`, the restore prints which stored secrets could not be
   decrypted: the SMTP password, the LDAP service-account password, the S3 secret key and the
   backup passphrase. Enter them again in the admin pages.
5. Copy the Kerberos keytab back, or export a new one (see
   [active-directory.md](active-directory.md#2-kerberos-single-sign-on)).

## What to keep off the server

- the backup passphrase;
- `secret.key` (or the `CURULIA_SECRET_KEY` value);
- the database key, if the database is encrypted;
- the S3 credentials, so you can fetch backups when the server is gone.
