# Runbook for the deployer agent on the clinic's Windows host

This file is for the Claude Code session that runs **on the clinic's host PC** (Windows 11 Pro) of the Windows
edition. It helps the owner install, connect to the internet, update and repair the app. The PC holds real patient
records, so read the rules first. **Read `docs/DEPLOY-AGENT-WINDOWS-READ-FIRST.md` before this file.** Design
background: `docs/specs/windows-edition.md`.

Status: written with the spec on 2026-10-28, before the edition is built. Commands that call `KlerkTool.exe` exist
only from slice W6 (host page) and W8 (internet) on; check `C:\Program Files\Klerk\version.txt` and
`KlerkTool.exe --help` before using them.

## How the host is laid out

| Folder | Holds | Who writes to it |
|---|---|---|
| `C:\Program Files\Klerk\` | Program files: `node\`, `app\`, `caddy\`, `winsw\`, `window\`, `tool\`, `version.txt` | Only the installer |
| `C:\Program Files\Klerk.previous\` | The version before the last update | Only the installer |
| `C:\ProgramData\Klerk\` | `data\` (database, patient files, backups, trash, `sms.key`), `caddy\` (certificates, local root key, DNS-host token if dynamic DNS is used), `logs\`, `status.json` | The services and `KlerkTool.exe` |

Services (`Get-Service Klerk*`):

| Service | What it is | Listens on |
|---|---|---|
| `KlerkServer` | Node app | `127.0.0.1:3000` (TCP), UDP 41730 (discovery, clinic network) |
| `KlerkCaddy` | HTTPS for the clinic network and the internet | `:443` and `:80` |

Addresses: `https://klerk.internal` inside the clinic (Caddy's own certificate authority; answers private addresses
only), `https://<domain>` from the internet (Let's Encrypt). The host is the only server: there is no VPS or relay
(D62). The internet reaches the host through the clinic router's port forward of TCP 443 and 80.

## Rules

1. **Never touch patient data.** Do not read, print, copy, move or delete anything under
   `C:\ProgramData\Klerk\data\patients\`, `data\trash\`, or inside `data\clinic.db`. Never open the database with a
   query tool. Never print or replace `data\sms.key`. Backups and restores move whole files only, through the
   installer or `KlerkTool.exe`.
2. **Never print secrets:** the DNS-host token, Caddy's root key (`caddy\pki\authorities\local\root.key`), passwords, the
   setup token. Show a config file's keys without values if you must.
3. **No code changes here.** There is no git repository on the host. Never commit or push from it. If the app
   misbehaves, report the exact error and the log lines; the developer fixes the app.
4. **Back up before every change** to services or Caddy: `KlerkTool.exe backup` (or stop
   `KlerkServer` and copy `data\clinic.db` with its `-wal` file closed to `data\backups\` under a dated name).
5. **No destructive commands without the owner's yes:** deleting anything under `C:\ProgramData\Klerk`,
   `icacls ... /reset`, removing services, uninstalling, formatting a backup drive.
6. **Do not weaken security to make a check pass:** no firewall rule on the Public profile, no port 3000 rule, no
   Remote Desktop to the internet, no Defender exclusion, no `TRUST_PROXY` change.
7. When a step needs the owner, write `needs input:` and say exactly what they must do.

## What only the owner can supply

| Item | Why |
|---|---|
| The domain name and access to its DNS settings | It lives at the registrar |
| A fixed public IPv4 from the clinic's ISP, or a DNS host with an API token for dynamic DNS | The ISP contract and the DNS account are the owner's |
| Router admin access | The router is the clinic's; it must forward TCP 443 and 80 to the host |
| The owner account's username and password | The owner types them on the first-run page; you never see them |
| Approval to turn on BitLocker | It needs the recovery key stored somewhere safe by the owner |

## First install

The owner runs the installer; you watch and check. Stop at the first failure and report it.

1. The owner downloads `klerk-setup-<tag>.exe` and `SHA256SUMS.txt` from the newest release of
   `https://github.com/MMACABRE/MirKazemi-s.Public-/releases`. Check: `Get-FileHash .\klerk-setup-<tag>.exe` matches
   the line in `SHA256SUMS.txt`.
2. The owner runs it ("More info", "Run anyway" on the SmartScreen warning; the installer is unsigned), picks
   «سرور مطب», and finishes. The window app opens on «راه‌اندازی سرور مطب», where the owner creates the owner account.
3. Check:
   - `Get-Service Klerk*`: `KlerkServer` and `KlerkCaddy` Running, start type Automatic.
   - `Invoke-WebRequest http://127.0.0.1:3000/health` returns `ok`.
   - `icacls C:\ProgramData\Klerk\data` lists SYSTEM, Administrators and `NT SERVICE\KlerkServer` only.
   - `Get-NetConnectionProfile`: the clinic network is `Private`. If `Public`, the owner presses
     «تبدیل به شبکه‌ی خصوصی» on «وضعیت سرور مطب».
   - `powercfg /query SCHEME_CURRENT SUB_SLEEP STANDBYIDLE`: AC value 0.
   - BitLocker: `manage-bde -status C:`. If off: `needs input:` the owner turns it on (Settings, Privacy & security,
     Device encryption or BitLocker) and stores the recovery key away from the PC.
4. Restart the PC. Without signing in to Windows, from another device on the clinic network the window app or
   `https://klerk.internal` (after joining) answers.

## The outside test (can this clinic be reached from the internet?)

Run with the owner at the clinic, before buying the domain.

1. Public address: compare the router's WAN address (owner reads it from the router page) with
   `Invoke-RestMethod https://api.ipify.org`. If they differ, or the WAN address is in `100.64.0.0/10`, the line has no
   reachable public IP. Stop: `needs input:` the owner asks the ISP for a public (ideally fixed) IPv4. Until then the
   app works on the clinic network only.
2. If they match: the owner forwards TCP 443 and 80 on the router to the host's LAN address (give the host a fixed
   LAN address or a DHCP reservation on the router first, so the forward keeps pointing at it).
3. From a phone on mobile data (inside Iran), open `http://<public IP>/` while Caddy runs: Caddy answers (a redirect or
   an error page is fine; no answer is a failure).
4. From abroad (the owner asks someone outside Iran, or uses any online port-check site that tests from outside Iran):
   ports 80 and 443 on the public IP answer. Let's Encrypt validates from outside Iran, so this decides the
   certificate method:
   - Both pass: HTTP-01 (default).
   - Inside passes, abroad fails: DNS-01 is needed (`needs input:` a DNS host with an API token).
   - Inside fails: inbound traffic is blocked on this line; remove the forward and report `needs input:`.

## Connecting to the internet

1. At the registrar: an A record for the domain pointing at the clinic's public IPv4. Wait until
   `Resolve-DnsName <domain>` shows it.
2. On the host: `KlerkTool.exe internet --domain <domain>` adds the public site to Caddy and restarts `KlerkCaddy`.
   With a changing public IP add `--ddns <provider> --ddns-token-file <file>` (the tool updates the A record every 5
   minutes); with DNS-01 add `--dns01`. Delete the token file afterwards; the tool keeps its own protected copy.
3. Check `logs\caddy` for a certificate obtained for the domain, then from a phone on mobile data open
   `https://<domain>`: the sign-in page, no certificate warning.
4. Check the lockout sees real addresses: two failed sign-ins from a phone and one from another network; the owner
   confirms in «گزارش رویدادها» that they show different addresses, not the router's address.
5. From the phone: `https://<domain>/setup` and `https://<domain>/backup-feed/` answer 404, and
   `curl.exe -k --resolve klerk.internal:443:<public IP> https://klerk.internal/` from outside gets no page.
6. The router forwards only 443 and 80. Check its port-forward list has nothing else pointing at the host (no 3000,
   no 41730, no 3389).

## Workstations

The owner runs the same installer on each workstation, picks «اتصال به سرور», selects the host, and types the
«کد اتصال» shown on «وضعیت سرور مطب». You only check, on the host: the workstation appears under joined computers.
If no host is found, check the host's network is Private and `Get-NetFirewallRule -DisplayName "Klerk*"` shows the
UDP 41730 rule enabled.

## Update

1. The owner downloads the new release; check its hash as in First install.
2. The owner runs it on the host. It stops the services, writes `data\backups\clinic-…-before-update.db`, keeps the
   old program in `Klerk.previous`, starts the new one and checks `/health`. On failure it rolls back by itself and
   says so; then collect the last 50 lines of `logs\server` and report.
3. Workstations: only when their window app shows «نسخه‌ی تازه‌ی برنامه را روی این رایانه نصب کنید».

## Repair checklist

Go in order; stop at the first thing that is wrong and report it.

1. `Get-Service Klerk*` all expected services Running. If one keeps stopping: last 50 lines of its log folder.
2. `Invoke-WebRequest http://127.0.0.1:3000/health`.
3. Caddy: `logs\caddy` for certificate errors; `Test-NetConnection 127.0.0.1 -Port 443`.
4. Internet: `Invoke-RestMethod https://api.ipify.org` still equals the domain's A record (`Resolve-DnsName <domain>`);
   the router still forwards 443 and 80 to the host's current LAN address.
5. Disk: `Get-PSDrive C` free space; under 5 GB is a problem to report.
6. Time and zone: `Get-TimeZone` is Iran Standard Time; clock correct.
7. Backups: newest file in `data\backups\` is from today or last night; «وضعیت سرور مطب» shows the last off-host copy.
8. `status.json` warnings (BitLocker, network, Remote Desktop).

Restart a service with `Restart-Service <name>`. Never delete
`caddy\` to "fix" a certificate: that makes a new local root and every workstation must join again.

## Restore

Only on the owner's request. Use «بازگردانی از پشتیبان» on «وضعیت سرور مطب» (host window) or the installer's host
mode on a new PC. It moves the current data aside (`data-before-restore-<time>\`) and deletes nothing. After a restore
onto a new PC: workstations join again (new code), and the clinic types its SMS key again (`sms.key` is never in
backups).

## Report back

After each session tell the owner, in short sentences: what you did, what you checked and its result, and every
`needs input:` item still open.
