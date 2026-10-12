# Installation and operations

Curulia is one executable file. It needs a writable **data directory** and, in production,
**HTTPS**. HTTPS can come from a reverse proxy in front of the portal or from the portal itself.

- [Requirements](#requirements)
- [Linux with systemd](#linux-with-systemd)
- [macOS with launchd](#macos-with-launchd)
- [HTTPS and reverse proxies](#https-and-reverse-proxies)
- [First run](#first-run)
- [Email](#email)
- [Upgrading](#upgrading)
- [Monitoring and logs](#monitoring-and-logs)
- [Security checklist](#security-checklist)

## Requirements

- Linux (amd64 or arm64) or macOS 12 or later (Intel or Apple Silicon). For other platforms,
  contact the author.
- About 100 MB of RAM and a few hundred MB of disk, plus backups and plan images.
- A DNS name for the portal, such as `curulia.corp.example.com`. Kerberos single sign-on needs
  a real name, not an IP address.
- A TLS certificate for that name.
- Optional: an SMTP server for email, S3-compatible storage for off-site backups, and Active
  Directory.

Download the binaries from the [releases page](https://github.com/avysochin256/curulia-releases/releases): `curulia-linux-amd64`,
`curulia-linux-arm64`, `curulia-darwin-amd64` and `curulia-darwin-arm64`, plus `SHA256SUMS`
and the systemd unit `curulia.service`.

## Linux with systemd

```sh
sudo useradd --system --home /var/lib/curulia --shell /usr/sbin/nologin curulia
sudo install -m 755 curulia-linux-amd64 /usr/local/bin/curulia
sudo cp curulia.service /etc/systemd/system/curulia.service   # from the release files
sudo systemctl edit curulia        # override CURULIA_BASE_URL etc., see below
sudo systemctl daemon-reload
sudo install -d -o curulia -g curulia -m 700 /var/lib/curulia
sudo -u curulia CURULIA_DATA=/var/lib/curulia /usr/local/bin/curulia create-admin \
     --email admin@example.com --name "Admin"
sudo systemctl enable --now curulia
journalctl -u curulia -f
```

The unit file [`deploy/curulia.service`](../deploy/curulia.service) does the following:

- runs the portal as the unprivileged `curulia` user;
- keeps its data in `/var/lib/curulia` with mode 0700;
- listens on `127.0.0.1:8080` behind a reverse proxy on the same host;
- applies the usual systemd sandboxing options.

Example override (`systemctl edit curulia`):

```ini
[Service]
Environment=CURULIA_BASE_URL=https://curulia.corp.example.com
```

To serve HTTPS from the portal itself instead of through a proxy:

```ini
[Service]
Environment=CURULIA_ADDR=:443
Environment=CURULIA_TLS_CERT=/etc/curulia/tls.crt
Environment=CURULIA_TLS_KEY=/etc/curulia/tls.key
Environment=CURULIA_TRUST_PROXY=false
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

The certificate and key files must be readable by the `curulia` user. The portal reads them at
start-up, so restart it after you renew them.

## macOS with launchd

Create a standard user named `curulia` in *System Settings → Users & Groups* for the service
(it needs no login). Then, as an administrator:

```sh
sudo install -d -m 755 /usr/local/bin
sudo install -m 755 curulia-darwin-arm64 /usr/local/bin/curulia   # Intel Macs: curulia-darwin-amd64
sudo xattr -d com.apple.quarantine /usr/local/bin/curulia 2>/dev/null || true
sudo install -d -o curulia -m 700 /usr/local/var/curulia
sudo install -d -o curulia -m 755 /usr/local/var/log
sudo -u curulia /usr/local/bin/curulia create-admin --data /usr/local/var/curulia \
     --email admin@example.com --name "Admin"
```

A file downloaded with a browser is marked as coming from the internet, and macOS refuses to
start it because it isn't signed by an identified developer; the `xattr` line removes that mark.

Create `/Library/LaunchDaemons/io.github.avysochin256.curulia.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>io.github.avysochin256.curulia</string>
  <key>ProgramArguments</key>
  <array><string>/usr/local/bin/curulia</string><string>serve</string></array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>CURULIA_DATA</key><string>/usr/local/var/curulia</string>
    <key>CURULIA_ADDR</key><string>127.0.0.1:8080</string>
    <key>CURULIA_BASE_URL</key><string>https://curulia.corp.example.com</string>
    <key>CURULIA_TRUST_PROXY</key><string>true</string>
  </dict>
  <key>UserName</key><string>curulia</string>
  <key>Umask</key><integer>63</integer>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/usr/local/var/log/curulia.log</string>
  <key>StandardErrorPath</key><string>/usr/local/var/log/curulia.log</string>
</dict>
</plist>
```

Then start it, and restart it after changing the file:

```sh
sudo chown root:wheel /Library/LaunchDaemons/io.github.avysochin256.curulia.plist
sudo chmod 644 /Library/LaunchDaemons/io.github.avysochin256.curulia.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/io.github.avysochin256.curulia.plist
tail -f /usr/local/var/log/curulia.log

# after editing the .plist:
sudo launchctl bootout system/io.github.avysochin256.curulia
sudo launchctl bootstrap system /Library/LaunchDaemons/io.github.avysochin256.curulia.plist
```

`Umask` 63 (octal 077) keeps the files the portal creates private to the `curulia` user. For
HTTPS, put nginx in front (`brew install nginx`, see below) or set `CURULIA_TLS_CERT` and
`CURULIA_TLS_KEY`.

## HTTPS and reverse proxies

The portal refuses AD passwords over plain HTTP and marks its cookies `Secure`. Without HTTPS,
sign-in works only in `--dev` mode.

**Behind a proxy**, run the portal with `--trust-proxy` and have the proxy set
`X-Forwarded-Proto` and `X-Forwarded-For`. The portal uses the last `X-Forwarded-For` entry,
which is the one your proxy adds. Only enable `--trust-proxy` when the portal cannot be
reached except through the proxy; that is why it listens on `127.0.0.1` in the examples.

**nginx**: see [`deploy/nginx.conf`](../deploy/nginx.conf). Set `client_max_body_size` above
the plan upload limit. Kerberos needs large header buffers, because tickets of users in many
groups are big.

**Apache**:

```apache
<VirtualHost *:443>
    ServerName curulia.corp.example.com
    SSLEngine on
    SSLCertificateFile /etc/ssl/curulia/fullchain.pem
    SSLCertificateKeyFile /etc/ssl/curulia/privkey.pem
    LimitRequestFieldSize 65536
    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/
</VirtualHost>
```

Whatever proxy you use, it must not perform authentication itself. It must pass
`Authorization` and `WWW-Authenticate` through unchanged.

Automatic check-in from the office network (see the [administrator
guide](admin-guide.md#automatic-check-in-from-the-office-network)) depends on the portal seeing
the real client address. With a proxy, `--trust-proxy` and `X-Forwarded-For` are required.
Otherwise every request appears to come from the proxy.

## First run

1. Sign in as the admin you created.
2. Check **Administration → System status**. It shows the version, database, HTTPS status and
   background jobs.
3. Under **Settings**, choose the default language, then turn on the optional features:
   - **General**: seat attributes, upload limit.
   - **Booking**: default booking window; whether employees can book for guests.
   - **Who's in**: visibility of occupants and how far back employees can look.
   - **Fixed seats**: eligibility, maximum duration, auto-approval.
   - **Email**: SMTP and which events send mail.
   - **Check-in**: mode (off, track only, release no-shows), planned arrival times or a fixed
     deadline, office network ranges.
   - **No-shows**: the optional warning and restriction policy.
   - **Analytics**: who can open it, who can see names, hybrid target, data retention.
   - **Security**: password length, session lifetimes, login limits.
4. **Backups**: add a passphrase and S3 storage, then click **Back up now**
   (see [backups](backup.md)).
5. **Active Directory**: see [active-directory.md](active-directory.md).
6. Create locations, floors and seats (see the [administrator guide](admin-guide.md)).

Many settings (marked on the location's *Settings* page) can be overridden per location, for
example check-in times or who's-in visibility.

## Email

In **Settings → Email notifications**, enter the SMTP server:

- port 587 with STARTTLS, or port 465 with implicit TLS;
- `none` only for a relay on localhost;
- an optional username and password. The password is stored encrypted and is never sent
  without TLS.

Use **Send test email** to check the setup. Messages are queued and retried with back-off.
Failed messages appear on **System status**, where you can retry or delete them.

Emails use the recipient's language. Links in emails use `--base-url`.

## Database encryption

The database holds personal data. You can encrypt it at rest, so a copied data folder, database
file or backup reveals nothing without the key. SQLite pages are encrypted with Adiantum
(XChaCha12 + AES) through the `adiantum` VFS of the SQLite driver. Performance is almost
unchanged.

The key is 32 random bytes written as 64 hex digits. The portal looks for it in this order:

1. the `CURULIA_DB_KEY` environment variable;
2. a key file: `--db-key-file` or `CURULIA_DB_KEY_FILE`;
3. the systemd credential `dbkey` (`$CREDENTIALS_DIRECTORY/dbkey`);
4. a key built into the binary (see below).

**From the admin panel.** *System status → Database encryption* (super admins) does most of the
work:

1. **Generate key** shows a new random key once, with a download button and the configuration
   lines for systemd, macOS (launchd), Docker and the command line. The portal does not store
   the key.
2. Put the key where the service reads it, as shown, and restart the portal. On start, a portal
   that finds a key and an unencrypted database encrypts it: it checks the database, writes an
   encrypted copy, verifies it and replaces the old file. This takes a few seconds, and the
   card then shows *Encrypted* and where the key came from.
3. **Change key** works the same way. Configure the new key as the key and the old one as the
   *previous key*, then restart; the portal re-encrypts the database on start. Afterwards, remove
   the previous key: the card warns while it is still configured. The previous key comes from
   `CURULIA_DB_KEY_PREVIOUS`, `--db-key-previous-file` / `CURULIA_DB_KEY_PREVIOUS_FILE`, or the
   systemd credential `dbkey.previous`.
4. **Less protected copies** lists backup archives in `data/backups` that contain the database
   unencrypted or encrypted with another key, and folders left by a restore. You can delete
   them there. Copies on S3 are not listed; the card says from which date they are affected.

Every step is recorded in the audit log (the key itself never is). Turning encryption off is
deliberately not in the panel: use `curulia db decrypt` on the server.

**Recommended: a systemd credential.** The key stays outside the data directory and outside
the binary. systemd passes it only to the service.

```sh
sudo install -d -m 700 /etc/curulia
curulia db genkey | sudo tee /etc/curulia/dbkey > /dev/null
sudo chmod 600 /etc/curulia/dbkey
sudo systemctl stop curulia
# encrypt as the service user, with the key passed the same way the service gets it
sudo systemd-run --wait --pipe -p User=curulia -p LoadCredential=dbkey:/etc/curulia/dbkey \
     /usr/local/bin/curulia db encrypt --data /var/lib/curulia --yes
sudo systemctl edit curulia        # add the LoadCredential line below
sudo systemctl start curulia
```

```ini
[Service]
LoadCredential=dbkey:/etc/curulia/dbkey
```

The key file is on the same server. So the encryption protects copies of the data directory,
of the database file and of backups, but not a complete image of the disk. If the server has a
TPM, tie the key to it so that it also covers disk images:

```sh
sudo systemd-creds encrypt --name=dbkey /etc/curulia/dbkey /etc/curulia/dbkey.cred
sudo rm /etc/curulia/dbkey          # after saving a copy offline
```

Then use `LoadCredentialEncrypted=dbkey:/etc/curulia/dbkey.cred` instead of `LoadCredential=`.

On macOS, set `CURULIA_DB_KEY_FILE` to a file only the service account can read,
or set `CURULIA_DB_KEY`.

**Key built into the binary.** The released binaries have no built-in key. A binary with a key
compiled in is a custom build; contact the author. Note that anyone with a copy of such a binary
can read the key out of it.

**Commands.** Stop the portal first; the commands that rewrite the database refuse to run while
it is running. Each command checks the database, writes the new file next to it, and replaces
the old file only after checking the new one.

| Command | |
|---|---|
| `curulia db status` | shows whether the database is encrypted, where the key comes from, and whether it opens |
| `curulia db genkey` | prints a new key |
| `curulia db encrypt` | encrypts an unencrypted database with the configured key |
| `curulia db decrypt` | writes the database back unencrypted |
| `curulia db rekey --new-key-file FILE` | re-encrypts with a new key; configure the new key before starting the portal |

All commands take `--data` and `--db-key-file` like `serve`. `--yes` skips the confirmation
question.

**Keep a copy of the key outside the server**, for example in a password manager. Without it
the database and all backups made after encryption cannot be opened. *System status* shows
whether the database is encrypted and where the key came from.

Things to know:

- After `db encrypt`, older backup archives still contain the unencrypted database. The command
  lists them; delete them or keep them as safe as the key.
- Adiantum hides the contents but not which 4 KiB blocks changed between two copies, and it
  does not detect tampering. Backup archives keep their own authenticated encryption on top
  (see [backups](backup.md#encryption)).
- The encryption key and `secret.key` are different keys. `secret.key` protects the passwords
  the portal stores (SMTP, LDAP, S3) and the two-step sign-in secrets.

## Upgrading

1. Read the release notes.
2. Stop the portal.
3. Replace the binary (or pull the new image).
4. Start the portal.

At start-up, if the database needs schema changes, the portal first writes a safety copy to
`data/backups/curulia-premigrate-<schema>-<time>.tar.gz` and then applies the migrations.
These copies are not rotated; delete old ones when you no longer need them.

To roll back: stop the portal, put the old binary back, and restore the pre-migration copy:

```sh
curulia restore --data /var/lib/curulia /var/lib/curulia/backups/curulia-premigrate-….tar.gz
```

Do not run an older binary against a database that a newer one has already migrated.

## Monitoring and logs

- `GET /healthz` returns `ok` (HTTP 200) when the database is reachable. It needs no sign-in.
- Logs go to standard error: text by default, JSON with `--log-format json`. Every request is
  logged with a request ID, and the same ID is shown on error pages.
- **System status** shows background jobs (email, reminders, auto-release, fixed-seat request
  expiry, clean-up, retention, backup), the email queue and the last backup.
- The **Audit log** records admin actions and security events: sign-ins of admins, lock-outs,
  setting changes, backup downloads and more.

## Security checklist

- Serve the portal only over HTTPS, and never run production with `--dev`.
- The data directory must be readable only by the service account. It holds the database with
  personal data, `secret.key` and the Kerberos keytab. Keep the database key outside it.
- Keep a copy of `secret.key`, the database key and the backup passphrase outside the server,
  for example in a password manager. Without them, encrypted backups and stored SMTP, LDAP and
  S3 passwords cannot be recovered.
- Encrypt the database ([above](#database-encryption)), set the retention periods and the
  privacy notice link, and require two-step sign-in for admins with local accounts (see the
  [admin guide](admin-guide.md#privacy-and-personal-data)).
- Keep server clocks in sync with NTP. Kerberos, check-in times and schedules depend on it.
- Use a dedicated S3 key with access to the backup bucket only.
- With Active Directory, follow the recommendations in
  [active-directory.md](active-directory.md#security).
