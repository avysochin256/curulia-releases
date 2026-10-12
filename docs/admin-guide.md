# Administrator guide

- [Roles](#roles)
- [Places: locations, floors, zones, seats](#places)
- [People: users, groups, visitors](#people)
- [Booking](#booking)
- [Rules](#rules)
- [Closures](#closures)
- [Fixed seats](#fixed-seats)
- [Who's in](#whos-in)
- [Check-in and auto-release](#check-in-and-auto-release)
- [No-show policy](#no-show-policy)
- [Analytics and data retention](#analytics-and-data-retention)
- [Settings per location](#settings-per-location)

## Roles

| | Employee | Location admin | Super admin |
|---|---|---|---|
| Book for themselves (within the rules) | ✓ | ✓ | ✓ |
| Book for visitors | if allowed in *Settings → Booking* | ✓ | ✓ |
| Book for colleagues, ignore rules (not closures or conflicts) | | ✓ | ✓ |
| Edit floors, seats, zones, rules, closures, fixed seats; approve requests | | own locations | ✓ |
| Analytics | | if allowed in *Settings → Analytics* | ✓ |
| Users, groups, locations, settings, AD, backups, audit log | | | ✓ |

Location admins are assigned to locations on the location's edit page.

## Places

**Location** → **floors** → **zones** (optional) → **seats**.

- A **location** has a time zone and working days. All dates and times (check-in deadlines,
  reminders, "today") are in the location's time zone. Bookings are only possible on working
  days.
- A **floor** has a plan image (PNG, JPEG or WebP, up to *Settings → General → Maximum upload
  size*). Uploading a new image keeps the seats, whose positions are stored relative to the
  image.
- **Zones** group seats, for example a team area or a quiet room. Rules can target them, and
  they give seats a colour. *Fixed seats allowed* marks zones for fixed-seat requests when
  those are limited to marked zones.
- **Seats** have a number (unique on the floor), an optional note, attributes and a
  bookable/unbookable state.

**Removing a location.** To hide a location for a while, untick *Active* on its edit page. To
remove it for good, use *Delete location* at the bottom of that page (super admins). A window
first shows what will change, and you confirm by typing the location's name:

- the location disappears with its floors and seats;
- bookings for today or later are cancelled, and the people are notified if email is on;
- fixed seats end, and pending fixed-seat requests are rejected;
- rules and current or future closures for the location, its floors, zones or seats are
  deleted, as are its location admins and per-location settings.

Past bookings stay in the history and statistics under the location's name. Deleting cannot be
undone and is recorded in the audit log.

### Plan editor

Open a floor and click **Edit seats**.

- **Add seats**: click the plan. Numbers continue from *Next seat number*.
- **Select**: click a seat or drag a box around several; Shift adds to the selection. Drag
  selected seats to move them. The arrow keys nudge them (Shift for bigger steps). Ctrl+Z
  (⌘Z) undoes and Shift+Ctrl+Z redoes.
- With a selection you can set the zone, add or remove attributes, make seats
  bookable/unbookable, **renumber** them (left to right or top to bottom, from a start number),
  or delete them.
- **Move view**: pan the plan. The mouse wheel zooms.
- Changes are saved automatically. If deleting or disabling seats would cancel existing
  bookings or fixed seats, the editor lists them and asks first. If another admin changed the
  floor meanwhile, you are asked to reload.

**QR codes** (on the floor page) prints one card per seat for check-in.

**Seat attributes** (super admin: *Seat attributes*) are tags such as *monitor*, *docking
station* or *standing desk*. People can filter the booking map by them. You can turn them off
in *Settings → General*.

## People

- **Users**: create them one at a time, import them from CSV (`email,name,groups,role,locale`;
  groups separated by `;`; you see a preview before anything is written), or let AD create them
  at first sign-in. New local users get a temporary password and must change it. Deactivating a
  user cancels their future bookings and ends their fixed seat.
- **Groups** are used in rules, fixed-seat eligibility and auto-approval. Groups can be filled
  by hand or mapped from AD groups (see [active-directory.md](active-directory.md)).
- **Visitors** have no login. When booking, choose *A new visitor* (name, optional company and
  email) or *A known visitor*. The person who books is the visitor's host. Visitors count
  against their own quotas, not the host's. Rules can target *guests* as a whole.

## Booking

The **Book** page shows the floor map for the chosen date: free, taken, yours, fixed, closed.
Pick a seat, then a booking type:

- **One day**;
- **Several days**: every working day in a range;
- **Every week**: chosen weekdays between two dates.

Before anything is saved, the confirmation shows every date. Dates that cannot be booked are
listed with the reason, such as a taken seat, a closure, or the name of the rule that stopped
the booking. You can book the remaining dates or go back.

Admins can also book *for a colleague* or *for a visitor*, and can override rules. Closures and
double bookings are never overridden. **All bookings** lists and filters bookings, exports them
to CSV, and lets admins cancel or move them to another seat. The owner is notified when email
is on.

Employees cancel their own bookings from **My bookings**, until the end of the day or only
before the day starts (*Settings → Booking*).

## Rules

Rules decide who may book what. Each rule has:

- a **name**, shown to people when it stops a booking;
- a **place**: everywhere, a location, a floor, a zone or a seat;
- a **subject**: everyone, employees, guests, a group or a person.

A default quota rule, *One seat per person per day*, is created on first run.

### Access rules (allow / deny)

For a seat, the portal looks for access rules from the most specific place to the least:
seat → zone → floor → location → everywhere. **The first level that has a rule matching the
person decides.** At that level, the most specific subject wins (person > group >
employees/guests > everyone). If an allow and a deny rule are equally specific, deny wins.
When no rule matches at all, booking is allowed.

Common patterns (the **Reserve for…** button creates the first kind in one step):

| Goal | Rules |
|---|---|
| Finance zone for Finance only | zone Finance: *deny everyone*; *allow group Finance* |
| Visitors only in the guest area | location HQ: *deny guests*; zone Guest area: *allow guests* |
| Contractors not at HQ, except Alice | location HQ: *deny group Contractors*; *allow person Alice* |
| Seat 101 only for John | seat 101: *deny everyone*; *allow person John* |
| No visitors anywhere | everywhere: *deny guests* |

### Quota rules

A quota rule gives a maximum number of booked days per **day**, **week** (Monday to Sunday) or
**calendar month**, counted **within the rule's place**. An example: "max 3 per week at
location HQ".

- All quota rules that match must pass.
- If several rules for the same place and period match a person, the most specific subject
  wins. Between two group rules, the more generous limit applies.
- A fixed seat counts as one booking per day for daily quotas only.

### Booking-window rules

These rules set how far ahead people can book (*Days ahead*), the longest range, whether
recurring bookings are allowed, and for how many weeks. Each limit is resolved separately:
the most specific place, then the most specific subject, then the most generous value. The
defaults are in *Settings → Booking*. A weekly booking only needs its **start** within *Days
ahead*; that is the point of recurring bookings.

### Rule tester

**Rule tester** (under Booking) shows, for a person or visitor, a seat and a date, every check
the portal makes and which rules matched at each level. Use it when someone asks "why can't I
book this?".

Changing rules never cancels existing bookings. Rules apply to new bookings.

## Closures

**Closures** block a place (everywhere, a location, a floor, a zone or a seat) for a date range,
for example a public holiday or a renovation. **Check affected bookings** lists the bookings
that will be cancelled. After you confirm, they are cancelled and their owners notified.
Fixed seats are not ended, but their days inside the closure are blocked. Deleting a closure
does not restore cancelled bookings.

## Fixed seats

For people who are in the office every day (*Settings → Fixed seats*, per location):

1. An employee opens a seat on the booking page and chooses **Request as my fixed seat**, with a
   start date, an optional end date and a comment.
2. The request goes to the location's admins, or to super admins if the location has none,
   under **Requests**. A badge shows the number waiting. They approve (optionally changing the
   dates) or reject with a comment. Requests can be **approved automatically** for everyone or
   for chosen groups when there are no conflicts. Requests waiting longer than *auto-reject
   after* days expire.
3. If other people have bookings on that seat in the period, the approver chooses to cancel
   them (their owners are notified) or to start the fixed seat after the last one.

The owner can **free the seat on chosen days**, so others can book it, and **take a day back**
if nobody has booked it yet. They can also **give up** the seat. Admins can create, edit and
end fixed seats directly under **Fixed seats**.

Settings: who may request (groups), which seats (all, or zones marked *Fixed seats allowed*),
the longest duration in months (0 = no limit and no end date needed), one per person, and
auto-approval.

## Who's in

**Who's in** shows the floor for any date:

- **today**: who is booked and who has checked in, refreshed every minute;
- **past days**: click a seat to see who sat there ("who broke my chair?");
- **future days**: see where colleagues will sit, to book next to them.

**Sit near a colleague.** In the list view of the booking page, *Sit near* takes a name, AD
login or email and suggests matching colleagues as you type. The list updates while you type:
the colleague's seat on the first chosen day comes first, then the free seats nearest to them
(the three closest are marked), then the rest. If they sit on another floor, a link opens it. It is available
where names are visible to the viewer (see the settings below), and it finds only people who
have a seat on that day or later.

**Find a colleague** searches by name or email and shows where the person sits on the chosen
date and the next days.

Settings (per location): whether the page is on; who sees names (everyone, or only the
location's admins, in which case everyone else sees only free or occupied); whether visitor
names are shown; and how many days back employees can look. Admins can always look further
back.

## Check-in and auto-release

Check-in is set per location (*Settings → Check-in*, or the location's own settings).

**Mode**

- **Off**: no check-in.
- **Track only**: people check in, which gives attendance data in Analytics, but seats are
  never released.
- **Release no-shows**: seats of people who don't check in in time are released, so others
  can book them for the rest of the day.

**Timing**

- **Planned arrival** (the default). Each booking has its own arrival time, so people who start
  at different times are treated fairly.
  - The admin sets the range people can choose from (default 07:00–12:00), the step (15, 30 or
    60 minutes) and a default time.
  - Each person can set a **usual start time** in *My account*. It is suggested when they book,
    and they can change it on the booking's confirm step or later in *My bookings*.
  - Recurring bookings use one arrival time for the whole series. It can be changed for one
    day or for all following days.
  - Fixed-seat owners set an arrival time on their seat and can change it for a single day.
  - Check-in opens *N* minutes before the arrival time (default 60). The seat is released
    after the **grace period** (default 60 minutes). A booking made for today after its arrival
    time still gets the full grace period from the moment it was made.
- **Fixed deadline**: one opening time and one deadline for everyone.

**On the day**, the home page shows a *Check in today* card with the arrival time and the
time the seat is kept until:

- **I'm here** checks in. People can also scan the seat's **QR code**, and admins can mark
  people as arrived.
- **Running late** keeps the seat longer (+60 minutes by default; how often per day is
  configurable, and 0 hides the button).
- **Not coming** frees the seat at once. It works even when cancelling on the day is otherwise
  not allowed. It is counted as a *voluntary release*, not a no-show.
- **Change arrival time** for today.

With email on, a **reminder** is sent at the arrival time, or when check-in opens with fixed
timing, to people who haven't checked in. It says when the seat will be released and links to
the card.

**Walk-in:** scanning the QR code of a free seat offers "Book it for today and check in".

**Fixed seats** are released only if *Also release fixed seats* is on.

**Visitor** bookings are never released. The host checks them in.

### Automatic check-in from the office network

With *Check in automatically from the office network* on, opening any portal page from one of
the listed addresses checks the person in, for bookings and fixed seats whose check-in window
is open. No QR scan is needed.

- List the office LAN and Wi-Fi ranges, one per line, such as `10.20.0.0/16` or
  `2001:db8::/48`. **Never list VPN or guest networks**, or people would be checked in from
  home.
- The portal must see the real client address: direct connections, or a reverse proxy with
  `--trust-proxy` (see [installation](install.md#https-and-reverse-proxies)).
- A person is checked at most once every 5 minutes, so page views stay fast.

## No-show policy

*Settings → No-shows*, off by default. It needs check-in with **Release no-shows**. A no-show
is a booking, or a fixed-seat day, released because nobody checked in. *Not coming* and
cancellations never count.

- No-shows are counted over a window (default 30 days).
- At **warn after** (default 2), the person gets an email reminding them to cancel or press
  *Not coming*.
- At **restrict after** (default 3), for **restrict days** (default 14) they can book at most
  **N days ahead** (default 1) and cannot make recurring bookings. Their existing bookings are
  kept.
- Restricted people see a banner on the Home and Book pages. The booking page, the rule tester
  and the user page show the restriction.
- Super admins can **lift** a restriction early, on the user page or on *Analytics → No-shows*.
  This is recorded in the audit log.
- Visitors are never counted. Admins can still book for restricted people with *Override
  rules*.

## Analytics and data retention

**Analytics** (for super admins, or for all admins if *Settings → Analytics* allows) covers
one location or all of yours, for the last 4 or 13 weeks or custom dates. Every figure is
compared with the previous period of the same length.

- **Overview**: average and peak occupancy, people in the office, office days per person per
  week, no-show rate, voluntary releases, use of fixed seats and visitor days; occupancy by day
  and by weekday; planned vs. actual arrival times.
- **Attendance**: how many people come in 0–1, 1–2, … days per week, and a table per person.
  With a hybrid-work **target** (*Settings → Analytics*, e.g. 3 days per week), who is at or
  below it.
- **No-shows**: weekly no-shows and voluntary releases, a table per person, and the
  restrictions in force.
- **Space**: a **heatmap** of the floor plan showing how often each seat was used, the least
  used seats, **fixed seats their owners rarely use** (candidates to give back), zone use and
  nearly full days.
- **Teams**: for each group, how many members are in on each weekday. Useful for planning team
  days.
- **Tables**: the detailed report tables (occupancy, by weekday, by person).

Every table can be downloaded as CSV (UTF-8, opens in Excel). The *How numbers are counted*
section on the page gives the exact definitions.

**Privacy.** Per-person attendance is personal data, and employees usually have to be told it
is collected.

- *Who can see names in analytics* is **super admins only** by default. It can be set to
  *nobody* (totals only) or to *all admins* (for their locations).
- Every view or export of a per-person table is recorded in the audit log.
- Viewers without name access don't see groups with fewer than 3 members.

*Settings → Analytics → Data retention* (months; 0 = keep forever) deletes bookings and
related history older than that. It runs automatically and is recorded in the audit log.

## Privacy and personal data

The portal holds names, email addresses, AD logins, where and when people sat, arrival times,
and the IP addresses and browsers of sign-ins. These features help you meet data-protection
rules such as the GDPR or Russian Federal Law 152-FZ.

**Retention.** *Settings → Analytics*:

- *Data retention* deletes bookings and their history after the given number of months.
  Visitor records are deleted once none of their bookings are left.
- *Keep the audit log, months* (default **24**) deletes older audit entries.
- For both, 0 means keep forever. Both run every 6 hours and are recorded in the audit log.

**Privacy notice.** Put the address of your privacy notice in *Settings → General → Privacy
notice link*. The link then appears on the sign-in page, in the footer and on the account page.

**Download my data.** On **My account**, everyone can download what the portal holds about
them as a JSON file:

- profile, groups and managed locations;
- bookings, fixed seats, check-ins and arrival times;
- requests and restrictions;
- sign-in sessions;
- their own audit-log actions.

Super admins can download the same file for anyone on the user's page. Every download is
recorded in the audit log.

**Anonymise user.** Use it when someone leaves or asks for their data to be deleted. On the
user's page, *Personal data → Anonymise user* (super admins; you have to type the person's
email to confirm) does the following:

- deactivates the account: future bookings are cancelled and fixed seats ended;
- replaces the name with "Former user #ID";
- removes the email, password, AD identifiers, sessions, groups, managed locations, second
  factor and queued emails;
- replaces the email and name inside audit-log details with `[anonymised]`.

Past bookings stay under the pseudonym, so statistics don't change. This can't be undone. For
AD users, also remove the person from the AD groups that give access, or they will be created
again at their next sign-in.

**Two-step sign-in** (local accounts). *Settings → Security → Two-step sign-in*:

| Value | Effect |
|---|---|
| off | no second factor |
| optional (default) | everyone with a local account can turn it on |
| required for administrators | local location admins and super admins must set it up before they can do anything else |

To set it up, go to **My account → Two-step sign-in**:

1. Scan the QR code with an authenticator app, for example Google or Microsoft Authenticator,
   FreeOTP or a password manager.
2. Enter a code from the app to confirm.
3. Save the 8 **recovery codes** that are shown once. Each works once instead of an app code.

After that, signing in asks for a code after the password. A code can't be used twice. After
5 wrong codes the sign-in is cancelled for 15 minutes.

If someone loses their phone and their recovery codes, a super admin can use *Reset two-step
sign-in* on the user's page. The person then signs in with the password only and can set it up
again.

AD accounts are not affected. Their second factor is the directory's job, for example
through your VPN or ADFS.

**Encryption at rest.** The database can be encrypted with a key kept outside the data
directory. Start on *System status → Database encryption*, which generates the key and shows
how to configure it; see [installation](install.md#database-encryption).

## Settings per location

Settings marked as per-location can be overridden on **Locations → (location) → Settings**.
Examples are check-in times, who's-in visibility, whether employees may book for visitors, and
fixed-seat rules. The
global value applies wherever no override is set, and the page shows it next to each field.

## Help buttons

Every field and setting has a "?" button. Hover over it or click it for an explanation of the
field; for settings, the default value and whether it can be set per location are added.
*Settings → General → Help buttons next to fields* turns the buttons off for everyone.
