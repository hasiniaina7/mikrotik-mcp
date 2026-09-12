# Renewing the `login.techzone.lat` certificate on MikroTik hotspots

The MikroTik RouterOS hotspot profile `hsprof1` on every TECHZONE router
terminates HTTPS for the captive portal using a certificate issued for
`login.techzone.lat`. That certificate is a normal Let's Encrypt cert with a
90-day lifetime, obtained on a shared VPS and pushed out to each router as a
PKCS#12 (`.p12`) bundle. It is **not** auto-renewed on the routers — it must
be regenerated and redeployed manually (or via a cron'd script) before each
expiry.

## Why a VPS is involved at all

MikroTik RouterOS has no ACME/Let's Encrypt client, so the certificate is
obtained on an external host and imported. The DNS name `login.techzone.lat`
is pointed at the shared VPS's public IP (`13.247.123.11`, same EC2 host used
for `hotspot.techzone.lat` / `info.techzone.lat` / `techzone.lat`) purely so
Let's Encrypt's HTTP-01 challenge can be served from there — the VPS does not
otherwise serve the hotspot login page; that's the routers' job.

## 1. Obtain / renew the certificate on the VPS

```bash
ssh -i key-not-for-faneva.pem ubuntu@ec2-13-247-123-11.af-south-1.compute.amazonaws.com

sudo certbot certificates --cert-name login.techzone.lat   # check current status/expiry

# renew (webroot authenticator, already configured in
# /etc/letsencrypt/renewal/login.techzone.lat.conf, serves from /var/www/html
# via the default nginx vhost's `server_name _`):
sudo certbot renew --cert-name login.techzone.lat --force-renewal \
  --non-interactive --no-random-sleep-on-renew
```

Notes:
- `--force-renewal` is needed because plain `renew` refuses to touch a cert
  that isn't within its renewal window (default: last 30 days of validity).
- Always pass `--no-random-sleep-on-renew` when running this interactively —
  without it, `--non-interactive` mode sleeps for a random 0-~10 minutes
  before doing anything (anti-thundering-herd jitter meant for unattended
  cron jobs), which looks like a hang.
- If a previous run was interrupted, `Another instance of Certbot is already
  running` means a stale lock: check `ps aux | grep certbot`, kill any
  leftover process, then `sudo rm -f /var/lib/letsencrypt/.certbot.lock`.
- If Let's Encrypt reports `DNS problem: NXDOMAIN` for `login.techzone.lat`
  even though public resolvers (`1.1.1.1`, `8.8.8.8`) resolve it fine, it's
  usually transient DNS propagation lag across Let's Encrypt's
  multi-perspective validation — wait a few minutes and retry rather than
  assuming the DNS record is broken.

## 2. Export to PKCS#12

MikroTik's `/certificate import` only accepts PEM (unencrypted, one cert at a
time) or PKCS#12. A `.p12` bundling the leaf cert + private key in one file
is the simplest artifact to ship to routers:

```bash
sudo openssl pkcs12 -export \
  -in  /etc/letsencrypt/live/login.techzone.lat/fullchain.pem \
  -inkey /etc/letsencrypt/live/login.techzone.lat/privkey.pem \
  -out /tmp/login.techzone.lat.p12 \
  -name login.techzone.lat \
  -passout pass:'<PASSPHRASE>'
```

Pick a fresh random passphrase per renewal (`openssl rand -base64 24`) and
keep it only alongside the `.p12` file — it's needed again at import time on
every router. Pull the file down and delete the copy on the VPS's `/tmp`
(`shred -u`) once retrieved, since it embeds the private key.

## 3. Deploy to each router

Every router is reachable over SSH with the same key
(`~/ssh/mikrotik_admin`, user `admin-ssh`). Per router:

```bash
scp -i ~/ssh/mikrotik_admin login.techzone.lat.p12 admin-ssh@<router-ip>:login-2026.p12

ssh -i ~/ssh/mikrotik_admin admin-ssh@<router-ip> \
  '/certificate import file-name=login-2026.p12 passphrase="<PASSPHRASE>"'
```

RouterOS auto-derives the certificate name from the uploaded file name:
importing `login-2026.p12` creates `login-2026.p12_0` (the leaf, with its
private key) and, if not already trusted, `_1`/`_2`/`_3` for the
intermediate/root chain. The uploaded file is consumed (deleted) by the
import automatically — no manual cleanup needed there.

Then point the hotspot profile at the new certificate:

```bash
ssh -i ~/ssh/mikrotik_admin admin-ssh@<router-ip> \
  '/ip hotspot profile set hsprof1 ssl-certificate=login-2026.p12_0'
```

Verify before moving to the next router:

```bash
ssh -i ~/ssh/mikrotik_admin admin-ssh@<router-ip> \
  '/ip hotspot profile print detail where name=hsprof1; /certificate print detail where name=login-2026.p12_0'
```

Confirm `ssl-certificate=login-2026.p12_0` (not `*<hex-id>`, which means the
referenced certificate was deleted/missing) and that `invalid-after` is the
new expiry date.

### Removing the old expired certificate

Do this **only after** confirming the profile now points at the new
certificate, and remove old certs **by exact name**, one at a time — do not
use a `/certificate remove [find ...]` filter combining conditions with
`and`/`!`, RouterOS's console query syntax does not parse that the way you'd
expect and can silently match (and delete) the certificate you just
imported too. Concretely, this is wrong:

```
# DO NOT DO THIS - deleted the newly-imported cert during this renewal:
/certificate remove [find common-name="login.techzone.lat" and !name="login-2026.p12_0"]
```

Instead, list what's there and remove the specific stale name(s):

```
/certificate print brief where common-name="login.techzone.lat"
/certificate remove [find name="cert.p12_0"]
```

Existing naming varies by router depending on what filename was used the
last time someone imported a cert there (`cert.p12_0`,
`login.techzone.lat.p12_0`, `fullchain.pem_0` have all been used historically
— `/certificate print brief` on each router shows what's actually present).
Leave the shared chain certificates (the CN=E8/YE1/Root YE/ISRG entries)
alone; they're reused across renewals and other routers' certs may still
reference them.

## Router inventory (as of 2026-09-12)

All reachable via `ssh -i ~/ssh/mikrotik_admin admin-ssh@<ip>`, hotspot
profile name is `hsprof1` on all of them:

| Address    | Site                    | Status                                    |
|------------|-------------------------|--------------------------------------------|
| 10.5.1.2   | 2-Imandry               | unreachable (connection timed out) — not renewed, needs on-site/VPN check |
| 10.5.1.3   | 11-Anjoma-Nord          | renewed |
| 10.5.1.4   | Ankidona                | renewed |
| 10.5.1.5   | 5-Antsororokavo         | renewed |
| 10.5.1.6   | 6-Ankofafa              | renewed |
| 10.5.1.10  | ankofafa-remplacement   | renewed |
| 10.5.1.11  | 11-Anjoma-Ouest         | renewed |
| 10.5.1.12  | 12-Soatsiadino          | renewed |
| 10.5.1.14  | 2-Imandry-remplacement  | renewed |
| 10.5.1.15  | 15-Talatamaty-ax2       | renewed |
| 10.5.1.16  | 16-Imandry-ax2          | renewed |

`10.5.1.2` needs a manual retry once it's reachable — its certificate is
still the one issued 2026-09-12 expiring today and is not yet renewed.

## Automated renewal (since 2026-09-12)

Because the DNS record for `login.techzone.lat` is only pointed at the VPS
temporarily (added/removed by hand on Namecheap around each renewal, not
kept permanently), the renewal can't be fully unattended end to end — but
the checking, the actual renewal, and the deployment to all routers are
automated on the VPS itself, gated on DNS being present.

**Location:** `ubuntu@ec2-13-247-123-11.af-south-1.compute.amazonaws.com`,
chosen over the local machine because the VPS already has WireGuard routes
(`wg0`/`wg1`) reaching every router's LAN directly (it's the RadiusDesk
hub), so it can both renew the cert and deploy to all 11 routers without
depending on the local machine being on or VPN-connected.

**Files on the VPS:**
- `/etc/techzone-certs/renew-login-techzone.sh` — the automation script.
- `/etc/techzone-certs/mikrotik_admin` — copy of the routers' SSH private
  key (root:root, mode 600). This is the security tradeoff of running
  deployment from a shared VPS: router credentials now also live there, not
  just on the operator's own machine.
- `/etc/msmtprc` — SMTP config (Spacemail/Namecheap Private Email,
  `mail.spacemail.com:465`, account `contact@techzone.lat`) used to send
  alert/status emails to `hhasiniainachristian@gmail.com`.
- `/etc/systemd/system/techzone-cert-renew.{service,timer}` — runs the
  script daily at 07:30 UTC (±10 min jitter).
- `/var/log/techzone-cert-renew.log` — deployment output per run.
- `/etc/techzone-certs/last-notice-date`, `last-renewed-serial` — small
  state files so the daily run doesn't spam a reminder email more than once
  a day and can tell whether a renewal actually changed the certificate.

**Behavior of `renew-login-techzone.sh`, run once a day:**

1. Read the current certificate's expiry; if more than 7 days remain, exit
   immediately (no-op, no email).
2. Inside the J-7..J-3 window (and beyond, if nothing has happened yet):
   check whether `login.techzone.lat` currently resolves (via a public
   DoH resolver, `1.1.1.1`, not the VPS's local cache — see the DNS
   propagation note above) to the VPS's own IP.
   - If not: send one reminder email per day telling the operator to add
     the DNS A record on Namecheap, then exit. No changes are made.
   - If yes: run `certbot renew --cert-name login.techzone.lat
     --force-renewal --non-interactive --no-random-sleep-on-renew`, export
     the result to a freshly-named `.p12` (`login-YYYYMMDD.p12`, random
     passphrase per run), and deploy it to every router in the inventory
     table above (import + `/ip hotspot profile set hsprof1
     ssl-certificate=...`). Failures per router are collected into the
     summary rather than aborting the whole run.
   - On success, sends one email with the per-router deploy report and a
     reminder to remove the DNS record now (it's not needed again until the
     next ~90-day cycle). On certbot failure, sends a failure email with
     the certbot output instead.
3. The script deliberately does **not** delete the old, now-expired
   certificate objects on each router (the `cert.p12_0` /
   `login.techzone.lat.p12_0` / `fullchain.pem_0` from the previous round) —
   see the warning above about `/certificate remove [find ...]` filters
   deleting more than intended. Stale expired certs are cosmetically
   untidy but harmless (RouterOS just keeps them unreferenced); clean them
   up by hand occasionally using the exact-name procedure above.

**One-time setup still needed:** the SMTP password for `contact@techzone.lat`
must be set by hand in `/etc/msmtprc` on the VPS (deliberately never typed
into an AI conversation) — replace the `CHANGEME_SET_REAL_PASSWORD_HERE`
placeholder, then verify with:

```bash
echo "test" | msmtp -a techzone hhasiniainachristian@gmail.com
```

**Manually triggering a check/renewal outside the daily schedule:**

```bash
ssh -i key-not-for-faneva.pem ubuntu@ec2-13-247-123-11.af-south-1.compute.amazonaws.com
sudo /etc/techzone-certs/renew-login-techzone.sh
```

## 2026-09-12 renewal record

- New cert: CN `login.techzone.lat`, valid 2026-09-12 → 2026-12-11.
- Bundle kept at `login.techzone.lat/` in this repo's working tree
  (fullchain.pem, privkey.pem, the .p12, and its passphrase) — those files
  are git-ignored (see `.gitignore`) and must never actually be committed.
- Deployed as `login-2026.p12_0` on all routers above except `10.5.1.2`.
- Old expired certs (`cert.p12_0`, `login.techzone.lat.p12_0` /
  `fullchain.pem_0` depending on router) removed after confirming the
  hotspot profile no longer referenced them.
