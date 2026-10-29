# Read first: deploying the Windows edition in the clinic

For the Claude Code deployer agent on the clinic's host PC. Read this page before `docs/DEPLOY-AGENT-WINDOWS.md`
(the full runbook). Decision D62 in `Plan.md` and section 7 of `docs/specs/windows-edition.md` hold the reasoning.

## The setup in one paragraph

The host is a physical Windows 11 Pro PC inside the doctor's clinic, and it is the only server. There is **no VPS
and no relay**. Clinic workstations reach it over the clinic network at `https://klerk.internal`. Internet users
reach it at the owner's domain, through the clinic router's port forward of TCP 443 and 80 to this PC. Two Windows
services run it: `KlerkServer` (the app, on `127.0.0.1:3000` only) and `KlerkCaddy` (HTTPS on `:443` and `:80`).
There is no `KlerkTunnel`, no frp and no `tunnel\` folder; if an older document mentions them, ignore it.

## Pay attention to these

1. **The router is the internet path.** The owner must forward TCP 443 and 80 on the clinic router to this PC.
   Nothing else: never forward 3000, 41730 (discovery), 3389 (Remote Desktop) or any other port. Check the router's
   port-forward list after every change.
2. **Give the host a fixed LAN address first.** Ask the owner to set a DHCP reservation on the router (or a static
   address on the PC) before creating the port forward, so the forward keeps pointing at the host after a restart.
3. **The domain points at the clinic line's public IPv4.** A fixed public IPv4 from the ISP is the simple case. If
   the ISP gives a changing address, the host keeps the domain's A record up to date itself
   (`KlerkTool.exe internet --ddns ...`), which needs a DNS host with an API token. Never print that token.
4. **Run the outside test before buying the domain** (steps in the runbook, "The outside test"). It answers three
   questions: does the line have a public IPv4 (not a `100.64.0.0/10` carrier-grade NAT address), do ports 443 and 80
   answer from a phone on mobile data inside Iran, and do they answer from outside Iran.
5. **Let's Encrypt checks from outside Iran.** If the ports answer from inside Iran but not from abroad, the normal
   certificate check (HTTP-01) fails. Then the certificate needs DNS-01, which needs a DNS host with an API token:
   write `needs input:` and ask the owner for one. Do not retry the normal check in a loop: Let's Encrypt allows only
   5 failed checks per name per hour.
6. **If the line cannot receive incoming connections at all** (no public IPv4, carrier-grade NAT, or inbound
   blocked), internet access stays off for this clinic. The app still works fully inside the clinic, and the host
   status page «وضعیت سرور مطب» says that internet access is off. Do not set up a relay, a tunnel service, Cloudflare
   Tunnel or ngrok to work around it: the owner ruled that out (D62). Report `needs input:` and stop.
7. **`klerk.internal` must answer only clinic-network addresses.** Internet visitors arrive on the same ports as
   clinic workstations, so Caddy serves `klerk.internal` only to private and loopback addresses. After connecting to
   the internet, check from outside that asking the public address for `klerk.internal` returns no page, and that
   `https://<domain>/setup` and `https://<domain>/backup-feed/` answer 404.
8. **Check the lockout sees real addresses.** After connecting to the internet, make two failed sign-ins from a phone
   on mobile data and one from another network. The owner confirms in «گزارش رویدادها» that they show different
   addresses, not the router's address. If they all show one address, stop and report it: the sign-in lockout would
   lock out every internet user at once.
9. **Firewall stays on the Private profile.** Internet visitors come in through the router on the clinic network, so
   the installer's Private-profile rules for 443 and 80 already cover them. Never add rules on the Public profile,
   and if Windows shows the clinic network as Public, the owner switches it to Private on «وضعیت سرور مطب».
10. **The clinic network keeps working without the internet.** `klerk.internal` uses Caddy's own certificate
    authority, so it does not depend on Let's Encrypt. Never delete `C:\ProgramData\Klerk\caddy\` to "fix" a
    certificate: that makes a new local root, and every workstation must join again with a new «کد اتصال».

## What only the owner can do (write `needs input:` for each)

- Give router admin access, set the host's fixed LAN address, and create the 443 and 80 port forward.
- Ask the ISP for a public IPv4, fixed if possible.
- Buy the domain and set its A record (or give a DNS-host API token for dynamic DNS or DNS-01).
- Arrange the "from outside Iran" part of the outside test.
- Turn on BitLocker and keep the recovery key away from the PC.
