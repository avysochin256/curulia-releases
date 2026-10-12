# Curulia

A self-hosted portal for booking office desks. Employees pick a seat on the floor plan for one
day, a range of days or every chosen weekday. Admins can book for colleagues and visitors, and
control who may sit where.

- **Floor plans**: upload an image of each floor and place numbered seats on it by clicking.
  Group seats into colour-coded zones and tag them with attributes such as *monitor* or
  *standing desk*.
- **Booking**: book one day, a date range, or recurring weekdays. Book for yourself, a colleague
  or a visitor. Free seats are shown on the map, and the portal explains why a seat can't be
  booked.
- **Rules**: set access, quota, booking-window and closure rules for everyone, for employees or
  guests, or for a group or person. Each rule applies to all locations, one location, a floor,
  a zone or a single seat. A rule tester shows which rules decide a booking.
- **Fixed seats**: people who come in every day can request a permanent seat. The request is
  approved by an admin, or automatically for chosen groups. Owners can release single days
  for others to book.
- **Who's in**: see who sits where today, on a past day ("who broke my chair?") or on a future
  day (to sit next to a colleague). You can also search for a person.
- **Check-in** (optional): each booking has a planned arrival time, so late starters are not
  penalised. People check in with a button, by scanning the QR code on the desk, or
  automatically when they open the portal from the office network. "Running late" and "Not
  coming" buttons are available, and seats nobody checks in to are released after a grace
  period. An optional no-show policy warns and then restricts people who repeatedly book and
  don't come.
- **Analytics**: occupancy, how often people come in (with an optional hybrid-work target),
  no-shows, a seat-usage heatmap, rarely used fixed seats, and which teams are in on which
  days. CSV export throughout. Per-person figures are restricted and audited.
- **Email notifications** and an **audit log**.
- **Privacy**: optional encryption of the database at rest, retention periods for bookings,
  visitors and the audit log, "Download my data" for everyone, anonymisation of leavers, a
  privacy-notice link, and optional two-step sign-in (authenticator app) for local accounts.
- **Accounts**: local accounts and Active Directory. AD users sign in with their AD password
  (LDAP) or automatically on domain computers (Kerberos). Guests are records without a login.
- **Backups**: scheduled, optionally encrypted archives stored locally and on S3-compatible
  storage, and a one-command restore.
- **English and Russian** interface. Every optional feature can be switched on or off, and many
  settings can be changed per location.

It is a single Go binary with an embedded SQLite database and no external services. It is meant
for one server and up to about 1000 users.

## Download

Get the binaries from the [releases page](https://github.com/avysochin256/curulia-releases/releases): Linux and macOS, Intel/AMD and ARM, with
SHA256 checksums. This repository holds the documentation and license; the source code is not
public.

## Quick start

```sh
chmod +x curulia-linux-amd64 && mv curulia-linux-amd64 curulia   # macOS: curulia-darwin-arm64
./curulia create-admin --data ./data --email admin@example.com --name "Admin"
./curulia serve --data ./data --dev
```

Open http://localhost:8080 and sign in. `--dev` allows sign-in over plain HTTP for a trial;
production needs HTTPS. The [quick start guide](docs/quickstart.md) goes from the download to a
production set-up.

## Documentation

- [Quick start with only the binary](docs/quickstart.md)
- [Installation and operations](docs/install.md): Linux (systemd), macOS (launchd), HTTPS,
  database encryption, upgrades
- [Active Directory](docs/active-directory.md): AD password login and Kerberos single sign-on
- [Backups and restore](docs/backup.md)
- [Administrator guide](docs/admin-guide.md)

## Questions and problems

Open an issue in this repository. Report security problems privately, as described in
[SECURITY.md](SECURITY.md).

## License

Curulia is free for internal use under the
[PolyForm Internal Use License 1.0.0](LICENSE): any organisation may install it and use it for its
own internal operations, at no cost. The license does not allow redistributing it: download it
from this repository.

Selling it, redistributing it, hosting it for clients or otherwise making money with it needs a
commercial license. Contact [Alexander Vysochin](https://github.com/avysochin256).

The binaries contain open-source software under its own licenses; see
[THIRD_PARTY_LICENSES.txt](THIRD_PARTY_LICENSES.txt), or run `curulia licenses`.

Copyright 2026 Alexander Vysochin.
