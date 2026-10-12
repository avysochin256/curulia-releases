# Quick start with only the binary

Curulia is one self-contained file. The pages, translations and database set-up are built
into it, so a new server needs nothing else: no database server, no runtime, no Docker. This
guide takes you from the downloaded binary to a working portal, then to a production set-up.

Every command below was tested on a clean directory containing only the binary.

## 1. Get the binary

Download it from the [latest release](https://github.com/avysochin256/curulia-releases/releases/latest):

| System | File |
|---|---|
| Linux, Intel/AMD 64-bit | `curulia-linux-amd64` |
| Linux, ARM 64-bit | `curulia-linux-arm64` |
| macOS, Apple Silicon | `curulia-darwin-arm64` |
| macOS, Intel | `curulia-darwin-amd64` |

The binaries need no libraries on the server. Download `SHA256SUMS` too and check the file:

```sh
sha256sum -c SHA256SUMS --ignore-missing
chmod +x curulia-linux-amd64
./curulia-linux-amd64 version
```

On macOS, use `shasum -a 256 -c SHA256SUMS --ignore-missing`. A binary downloaded with a browser
must also be cleared to run, because it isn't signed:
`xattr -d com.apple.quarantine curulia-darwin-arm64`.

Below, the binary is called `curulia`.

## 2. Try it in five minutes

```sh
mkdir curulia && cd curulia
cp /path/to/curulia-linux-amd64 ./curulia
./curulia create-admin --data ./data --email admin@example.com --name "Admin"
./curulia serve --data ./data --dev
```

- `create-admin` asks for the password twice. It must have at least 10 characters. For
  scripts, pipe it in with `--password-stdin`.
- Open <http://localhost:8080>, or `http://<server-address>:8080` from another computer, and
  sign in.
- `--dev` allows sign-in over plain HTTP from other computers, and the portal shows a
  "Development mode" bar. Use it only for trials; see [step 4](#4-run-it-for-real).
- Stop the portal with Ctrl+C.

On the first start the portal creates the data directory:

```
data/
  curulia.db      database
  secret.key      key for the passwords the portal stores (SMTP, LDAP, S3). Keep a copy.
  uploads/        floor-plan images
  backups/        backups; the first one is made right after the first start
  curulia.lock    exists while the portal runs
```

## 3. First steps in the portal

1. **Administration → Locations → New location**: a name, the time zone (for example
   `Europe/Moscow`) and the working days.
2. **Add a floor** on the location's page. Upload the floor plan image (PNG, JPEG or WebP).
3. On the floor page choose **Edit seats**, then **Add seats**, click on the plan where the
   seats are, and **Save**.
4. **Book a seat**: **Book**, pick the date, click a green seat, confirm. A default rule allows
   one seat per person per day; change it under **Rules**.
5. **Add people**:
   - one by one: **Users → New user**;
   - in bulk: **Users → Import users**, from a CSV file;
   - or connect Active Directory in **Settings → Active Directory**
     (see [active-directory.md](active-directory.md)).
6. **Settings** has everything else: check-in, email, booking limits, backups, security. Each
   field has a "?" button that explains it.
7. Optional: **QR codes** on the floor page gives a printable sheet for check-in at the desks.

## 4. Run it for real

A production set-up needs four things: a public address, HTTPS, a service that starts the
portal, and protected data.

### Public address

Set `--base-url` to the address people type, for example `https://curulia.company.com`. The
portal uses it in emails and QR codes.

### HTTPS

Without HTTPS, and without `--dev`, browsers on other computers drop the sign-in cookie, so
sign-in just returns to the login page. Choose one of two ways:

- **Built into the portal.** Pass the certificate and key (PEM files):

  ```sh
  ./curulia serve --data /var/lib/curulia --addr :443 \
      --base-url https://curulia.company.com \
      --tls-cert /etc/curulia/tls.crt --tls-key /etc/curulia/tls.key
  ```

  Port 443 needs privileges. Under systemd use the unit below; otherwise run
  `sudo setcap cap_net_bind_service=+ep ./curulia` once.
- **Behind a reverse proxy** (nginx or Apache) on the same machine:

  ```sh
  ./curulia serve --data /var/lib/curulia --addr 127.0.0.1:8080 \
      --base-url https://curulia.company.com --trust-proxy
  ```

  The proxy terminates HTTPS and forwards to `127.0.0.1:8080`. See
  [install.md](install.md#https-and-reverse-proxies) for an nginx example.

Every flag can also be set with an environment variable: `CURULIA_DATA`, `CURULIA_ADDR`,
`CURULIA_BASE_URL`, `CURULIA_TLS_CERT`, `CURULIA_TLS_KEY`, `CURULIA_TRUST_PROXY`,
`CURULIA_DB_KEY_FILE` and so on.

### As a Linux service (systemd)

```sh
sudo useradd --system --home /var/lib/curulia --shell /usr/sbin/nologin curulia
sudo install -m 755 curulia /usr/local/bin/curulia
sudo install -d -o curulia -g curulia -m 700 /var/lib/curulia
sudo -u curulia /usr/local/bin/curulia create-admin --data /var/lib/curulia \
     --email admin@company.com --name "Admin"
```

Create `/etc/systemd/system/curulia.service`:

```ini
[Unit]
Description=Curulia seat booking portal
After=network-online.target
Wants=network-online.target

[Service]
User=curulia
Group=curulia
Environment=CURULIA_DATA=/var/lib/curulia
Environment=CURULIA_ADDR=127.0.0.1:8080
Environment=CURULIA_BASE_URL=https://curulia.company.com
Environment=CURULIA_TRUST_PROXY=true
ExecStart=/usr/local/bin/curulia serve
Restart=on-failure
UMask=0077
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/curulia

[Install]
WantedBy=multi-user.target
```

Then:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now curulia
journalctl -u curulia -f
```

For built-in HTTPS on port 443, set `CURULIA_ADDR=:443`, add the two `CURULIA_TLS_*` lines and
`AmbientCapabilities=CAP_NET_BIND_SERVICE`, and set `CURULIA_TRUST_PROXY=false`. The release has a
fuller unit with more sandboxing: `curulia.service`, also in
[`deploy/curulia.service`](../deploy/curulia.service).

For macOS (a launchd service), see [install.md](install.md).

## 5. Protect the data on day one

1. **Copy `secret.key`** and keep it outside the server, for example in a password manager.
2. **Encrypt the database.** Create a key outside the data directory and start with it. The
   portal encrypts the database on start, which takes a few seconds:

   ```sh
   sudo install -d -m 700 /etc/curulia
   curulia db genkey | sudo tee /etc/curulia/dbkey > /dev/null
   sudo chmod 600 /etc/curulia/dbkey
   ```

   With systemd, add `LoadCredential=dbkey:/etc/curulia/dbkey` to the unit and restart;
   otherwise pass `--db-key-file /etc/curulia/dbkey`. Keep a copy of the key outside the server:
   without it the database and later backups can't be read.
   **System status → Database encryption** shows the state and can generate a key for you;
   [install.md](install.md#database-encryption) covers key changes and other options.
3. **Backups** run daily at 02:00 (server time) and keep 7 archives in `data/backups`. In
   **Settings → Backups**, set an encryption passphrase and, if you have one, S3-compatible
   storage, so copies leave the server.
4. **Two-step sign-in for admins**: **Settings → Security → Two-step sign-in** → *required for
   administrators*.

## 6. Check that it works

```sh
curl -s https://curulia.company.com/healthz   # prints "ok"
```

In the portal, **Administration → System status** shows whether the database is encrypted and
where the key comes from, the background jobs, the last backup and the email queue.

On the command line, `curulia db status --data <data directory> --db-key-file <key file>` shows
the same for the database. Run it as the service user, with the same key the service gets; with
systemd:

```sh
sudo systemd-run --wait --pipe -p User=curulia -p LoadCredential=dbkey:/etc/curulia/dbkey \
     /usr/local/bin/curulia db status --data /var/lib/curulia
```

Don't run commands on the data directory as root: files the database creates would then belong
to root, and the service couldn't open them.

## 7. Upgrading later

Build or download the new binary, stop the portal, replace the file and start it again. If the
database needs changes, the portal first writes a safety copy to `data/backups` and then
upgrades. See [install.md](install.md#upgrading) for rolling back.

## Troubleshooting

| What you see | Cause and fix |
|---|---|
| Sign-in returns to the login page | Plain HTTP without `--dev`: the browser drops the secure cookie. Use HTTPS (step 4), or `--dev` for a trial. |
| `data directory in use` | Another `curulia serve` (or a restore) is running on the same data directory. Stop it first. |
| `the database is encrypted` at start | No key configured. Pass `--db-key-file` or set up the systemd credential. |
| `the database key is wrong` | The configured key doesn't match. Use the key the database was encrypted with. |
| `permission denied` on port 443 | Run on 8080 behind a proxy, or grant `CAP_NET_BIND_SERVICE` (step 4). |
| AD sign-in refused | Active Directory sign-in needs HTTPS; it is refused over plain HTTP. |
| Links in emails or QR codes point to the wrong address | Set `--base-url` to the public address. |

More: [install.md](install.md) (operations, reverse proxy, macOS),
[backup.md](backup.md), [admin-guide.md](admin-guide.md).
