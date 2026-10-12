# Active Directory

Curulia supports two ways of signing in with Windows domain accounts. You can enable either,
both or neither:

| Method | Who | How it works |
|---|---|---|
| **Kerberos single sign-on** | domain-joined Windows, Linux or macOS computers | The browser presents the user's Kerberos ticket, so there is no password prompt. The portal never sees a password. |
| **AD password login (LDAP)** | any device, such as phones or home laptops | The user types their AD login and password into the portal's form. The portal checks them against a domain controller. |

Both methods lead to the same portal account for a person. Accounts are matched by the AD
**objectSid**, which does not change when the person is renamed or moved to another OU. Users
are created on their first sign-in. Local accounts keep working next to AD accounts, which
gives you a break-glass admin if AD is unavailable.

All settings are under **Administration → Active Directory**, in three blocks: *Active
Directory* (shared), *LDAP* and *Single sign-on (Kerberos)*.

- [Recommended setup](#recommended-setup)
- [1. Shared settings](#1-shared-settings)
- [2. Kerberos single sign-on](#2-kerberos-single-sign-on)
- [3. AD password login (LDAP)](#3-ad-password-login-ldap)
- [Moving local accounts to AD](#moving-local-accounts-to-ad)
- [Security](#security)
- [Troubleshooting](#troubleshooting)

## Recommended setup

- **Kerberos SSO** for domain computers, with **no LDAP service account**. Set the email source
  to `upn` or `ask`. The portal then stores no AD credentials and never handles passwords.
  The only secret to protect is the keytab.
- Turn on **AD password login** in `direct_bind` mode only if people must sign in with AD
  credentials from devices that are not in the domain.
- Keep one **local super admin** as a break-glass account, and set **Security → Local accounts**
  to `admins_only`.
- Serve the portal over HTTPS. The portal refuses AD sign-in over plain HTTP.

## 1. Shared settings

| Setting | Example | Notes |
|---|---|---|
| Kerberos realm | `CORP.EXAMPLE.COM` | the AD DNS domain in capitals |
| UPN suffix | `corp.example.com` | lets users type `jdoe` instead of `jdoe@corp.example.com` |
| NetBIOS domain | `CORP` | optional; used for `CORP\jdoe` binds when there is no UPN suffix |
| Allowed group | `S-1-5-21-…-1234` | optional; only members of this group (nested membership counts) can sign in |
| Group mappings | `S-1-5-21-…-2001 = Finance` | one per line; portal groups are created as needed. Use them in rules such as "zone Finance: allow group Finance". |
| Role mappings | `S-1-5-21-…-3001 = super_admin` | one per line; `super_admin` or `location_admin`. When at least one mapping exists, roles of AD users come only from AD and cannot be edited in the portal. Location admins still need their locations assigned when you edit a location. |
| Email of AD users | `upn` | `upn`: use the UPN as the email address. `ldap`: read the `mail` attribute (needs LDAP). `ask`: ask the user once after the first sign-in. |

Groups are mapped by **SID**, not by name, so renaming a group cannot grant access, and a
deleted group fails closed. To find a group's SID, use **Tools → Find AD groups**. This needs
either the LDAP service account or your own credentials, which are typed in and not stored.
The *Copy* button copies a ready-made mapping line. In PowerShell:
`(Get-ADGroup "Finance").SID.Value`.

Group membership is refreshed at every sign-in. **Security → AD session lifetime** (default
12 hours) forces a new sign-in, so group and role changes take effect at least that often. With
automatic SSO this re-login is silent.

## 2. Kerberos single sign-on

The portal validates Kerberos service tickets with a **keytab**. A keytab is a file holding
the long-term key of the portal's service account, which is what domain controllers encrypt
tickets for the portal with. The keytab contains no user secrets. However, anyone who has it
can forge tickets to the portal and sign in as any user, admins included. Protect it like a
private key.

### 2.1 Create the service account and SPN

On a domain controller or any machine with the AD PowerShell module (as a domain admin):

```powershell
$pw = Read-Host -AsSecureString "Password for svc-curulia"
New-ADUser -Name "svc-curulia" -SamAccountName svc-curulia `
    -UserPrincipalName "svc-curulia@corp.example.com" `
    -AccountPassword $pw -Enabled $true -PasswordNeverExpires $true `
    -CannotChangePassword $true -KerberosEncryptionType AES256 `
    -Description "Curulia portal (Kerberos service)"

# Register the SPN for the exact name users type in the browser
setspn -S HTTP/curulia.corp.example.com CORP\svc-curulia
```

- The SPN host must be the **exact DNS name** in the portal URL. Use an **A record**. If you use
  a CNAME, browsers request a ticket for the canonical name, so register the SPN for that name
  instead.
- An SPN must be registered only once in the forest. `setspn -S` checks for duplicates. Do not
  register it on the computer account too.
- Deny the account interactive and RDP logon (GPO *User Rights Assignment*). It is used for
  nothing else.

### 2.2 Export the keytab

**On Windows** (as a domain admin):

```powershell
ktpass /princ HTTP/curulia.corp.example.com@CORP.EXAMPLE.COM /mapuser CORP\svc-curulia `
       /crypto AES256-SHA1 /ptype KRB5_NT_PRINCIPAL /pass * /out curulia.keytab
```

`/pass *` asks for a password and **sets it on the account**. Choose a long random one; you
will not need it again. Always use `/crypto AES256-SHA1`, because RC4 is weak and the portal
warns about it.

**On Linux** (with `msktutil` 1.1 or later, run as a user that may reset the account's
password; like `ktpass`, this sets a new random password):

```sh
kinit admin@CORP.EXAMPLE.COM
msktutil --update --use-service-account --account-name svc-curulia \
         --service HTTP/curulia.corp.example.com --enctypes 0x10 \
         --keytab ./curulia.keytab --server dc1.corp.example.com
```

Whichever tool you use, confirm the result with **Check keytab** (§2.4). It must list
`HTTP/curulia.corp.example.com@CORP.EXAMPLE.COM` with an AES-256 key.

### 2.3 Install the keytab

Copy the file to the server over a secure channel and make it readable **only** by the
portal's service account:

```sh
sudo install -o curulia -g curulia -m 600 curulia.keytab /var/lib/curulia/curulia.keytab
shred -u curulia.keytab      # remove the copy you exported
```

On Windows, put it in the data directory, whose ACL allows only the service account and
Administrators.

The default path is `curulia.keytab` in the data directory. You can change it with **Keytab
file**. On Linux and macOS the portal refuses a keytab that other users can read. It also
reloads the file automatically when it changes.

### 2.4 Configure the portal

1. Fill in the shared settings (realm etc.).
2. In *Single sign-on*, set **Service principal name** to `HTTP/curulia.corp.example.com`, then
   click **Check keytab**. It lists the principals, key versions and encryption types and warns
   about problems. It never shows keys.
3. Turn on **Enable single sign-on** and choose the **Login page** mode:
   - `button`: the login page shows *Sign in with your Windows account* above the password
     form.
   - `auto`: opening the portal signs domain users in straight away. Other devices fall back to
     the password form after one round trip. After signing out, the user is not signed in
     again automatically for 10 minutes.
4. On a domain computer, click **Test single sign-on**. It shows the identity from the ticket
   (SID, login, UPN, group SIDs) and the groups and role the user would get. Your own session
   is not changed.

The server clock must be within **Maximum clock difference** (default 5 minutes) of the
domain controllers, so use NTP.

### 2.5 Browser setup on client computers

Browsers only send Kerberos tickets to sites you trust. Distribute the setting by policy.

**Windows: Edge, Chrome and Internet Explorer settings.** Add the site to the *Local intranet*
zone with GPO: *Computer Configuration → Administrative Templates → Windows Components →
Internet Explorer → Internet Control Panel → Security Page → Site to Zone Assignment List*.
Enter `https://curulia.corp.example.com` with value `1`. Edge and Chrome on Windows use this
zone.

Alternatively, set the Edge and Chrome policy `AuthServerAllowlist` = `curulia.corp.example.com`.

**Firefox (all platforms).** Use the policy `Authentication → SPNEGO`, or the preference
`network.negotiate-auth.trusted-uris`:

```json
{ "policies": { "Authentication": { "SPNEGO": ["curulia.corp.example.com"] } } }
```

Put this in `policies.json`: `/etc/firefox/policies/` on Linux, the `distribution` folder of
the Firefox installation on Windows, or use the Firefox ADMX templates.

**Linux: Chrome and Chromium.** Clients need a Kerberos ticket, obtained by signing in through
SSSD/realmd or with `kinit`. Add a managed policy:

```sh
# Chrome: /etc/opt/chrome/policies/managed/curulia.json
# Chromium: /etc/chromium/policies/managed/curulia.json
{ "AuthServerAllowlist": "curulia.corp.example.com" }
```

**macOS.** Domain-joined Macs, or Macs using the Kerberos SSO extension, work with Safari out of
the box. For Chrome, set `AuthServerAllowlist` in a configuration profile.

### 2.6 Keytab rotation

Rotate the key once a year, and immediately if the keytab may have leaked:

```powershell
ktpass /princ HTTP/curulia.corp.example.com@CORP.EXAMPLE.COM /mapuser CORP\svc-curulia `
       /crypto AES256-SHA1 /ptype KRB5_NT_PRINCIPAL /pass * /in curulia.keytab /out curulia-new.keytab
```

Resetting the password creates a new key version. `/in` keeps the old key in the new file, so
users whose browsers still hold tickets for the old key (for up to 10 hours) are not rejected.
Replace the file on the server; the portal picks it up without a restart. After a day, export
again without `/in` to drop the old key. **After a leak, do not keep the old key.** Export
without `/in`. Affected users can clear their tickets with `klist purge`, or they will get new
ones within hours.

## 3. AD password login (LDAP)

| Setting | Notes |
|---|---|
| Allow signing in with AD password | the switch for this method |
| Domain controllers | one URL per line, tried in order, e.g. `ldaps://dc1.corp.example.com` and `ldaps://dc2.corp.example.com` |
| Connection security | `ldaps` (port 636) or `starttls` (port 389). `insecure` sends passwords in clear text and is for labs only. |
| CA certificate | PEM of your enterprise CA, if the server does not already trust it |
| Skip certificate verification | never in production |
| Mode | `direct_bind` (recommended): the portal binds as the user (`login@upn_suffix`) and reads their own entry, so no AD credentials are stored. `service_account`: the portal searches for the user with a service account first. Use this when logins do not map to a UPN. |
| Base DN, user filter, attributes | used in `service_account` mode and for lookups |
| Service account DN and password | used only in `service_account` mode, and then also for the `ldap` email source and the group search. In `direct_bind` mode they are ignored: the `ldap` email source finds nothing, and the group search asks for your own credentials. The password is stored encrypted. |

Test with **Tools → Test connection** and **Test a login**. *Test a login* runs the full sign-in
flow with credentials you type in, which are not stored. It shows the SID, groups and role the
user would get.

On the login form, a person types either their email (for local accounts) or their AD login
(`jdoe`, `jdoe@corp.example.com` or `CORP\jdoe`). A local account with the same email as the
typed login always takes priority.

**Service account (if you use one):** a normal domain user with no extra rights (reading users
and groups needs none), a long random password, and interactive/RDP logon denied. You may
reuse `svc-curulia`.

**Lockouts:** the portal blocks a login name after **Security → Failed logins per user**
attempts (default 5 per 15 minutes) and an address after *Failed logins per address*. Keep the
per-user limit **below your AD lockout threshold**, so that the portal cannot be used to lock
people out of Windows.

## Moving local accounts to AD

If people already use local accounts, open **Users → (user) → Convert to AD account** and enter
their AD login. The account keeps its bookings, fixed seat and history. It loses its password,
and the next AD sign-in (SSO or LDAP) links it to the AD user.

New AD users need no preparation: they are created at their first sign-in. Accounts are never
linked by email address.

## Security

What the portal keeps from AD:

| Data | Stored? |
|---|---|
| User passwords | **Never.** SSO never sees them. LDAP login uses the password once in memory for the bind and never writes or logs it. |
| Service-account password | only if configured; encrypted with the key in `secret.key` |
| Keytab | a file in the data directory (mode 0600); not in the database, the UI or backups |
| User profile | objectSid, login, UPN, display name, email |
| Groups | only mapped groups, as portal group memberships |

Measures in place:

- HTTPS is required for AD sign-in, and LDAP must use LDAPS or StartTLS with certificate checks.
- Only Kerberos is accepted. NTLM is refused, tickets without a signed PAC are refused, and
  replays and clock skew are checked.
- Empty passwords are refused before any LDAP bind, which prevents AD's "unauthenticated bind"
  from counting as a success.
- Login names are escaped in LDAP filters.
- Failed logins are rate-limited per user and per address. The error message and response time
  are the same for unknown users and wrong passwords.
- Roles and groups are mapped by SID. AD users are never merged with local accounts.
- A user disabled in the portal cannot sign in even if AD accepts them. A user disabled in AD
  can no longer get tickets or bind, and portal sessions end after the AD session lifetime.
- The post-login redirect only accepts relative paths on the same site.

Your part:

- Protect the keytab (0600, service account only) and rotate it yearly.
- Keep the portal on the internal network or VPN if possible.
- Keep a local break-glass admin, and set **Local accounts** to `admins_only`.
- Encrypt backups. They contain names, emails and booking history, but no keytab and no
  plaintext secrets.
- If the server is compromised, rotate the keytab and the service-account password. If LDAP
  password login was on, passwords typed during the compromise may have been captured.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| "Single sign-on isn't available on this device" | The browser sent no ticket: the device is not in the domain or has no ticket (`klist`), or the site is not trusted by browser policy (§2.5). |
| "Your browser offered NTLM instead of Kerberos" | Windows could not get a ticket for the SPN. Check that the URL uses the exact name in the SPN (not an IP address or short name), that the SPN exists (`setspn -Q HTTP/curulia.corp.example.com`) and that it is not duplicated. |
| "Single sign-on failed" | Look at the portal log. Common causes: key version mismatch after a password reset (export the keytab again and check it with **Check keytab**), clocks out of sync, or a wrong SPN or realm. |
| "The Kerberos ticket has no group information (PAC)" | The service account has the `NO_AUTH_DATA_REQUIRED` flag (userAccountControl 0x2000000), which stops domain controllers adding the PAC. Clear it. |
| Keytab check: "no AES key" | Enable AES 256 on the account and export again with `/crypto AES256-SHA1`. |
| Keytab check: "readable by other users" | `chmod 600` and `chown` the file to the service account. |
| "Your account may not use this portal" | The user is not in the *Allowed group*. (A user disabled in the portal gets the usual "invalid login" message.) |
| "The directory is unavailable" | No domain controller answered. Check the URLs, firewall (636/389) and certificates with **Test connection**. |
| AD users must sign in again often | This is the *AD session lifetime*. In `auto` SSO mode the re-login is silent. |
| Groups not updated | Groups are read at sign-in and in Kerberos tickets. Users must sign in again, and Windows users may need to lock and unlock their session or run `klist purge` to get a ticket with the new groups. |
