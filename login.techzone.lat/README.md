# login.techzone.lat - certificate bundle (2026-09-12 renewal)

**Never commit the key material in this folder.** `fullchain.pem`,
`privkey.pem`, `login.techzone.lat.p12`, and `p12-password.txt` contain a
private key and its passphrase in cleartext; they are excluded via the
repo's `.gitignore` (`/login.techzone.lat/*.pem`, `*.p12`,
`p12-password.txt`). Only this `README.md` is meant to be tracked. Double
check `git status` before any commit that touches this directory.

## Contents

- `fullchain.pem` / `privkey.pem` — raw Let's Encrypt output (leaf + intermediate, EC/prime256v1).
- `login.techzone.lat.p12` — PKCS#12 bundle built from the two files above, for MikroTik import.
- `p12-password.txt` — passphrase used to encrypt the `.p12` (also required at import time on each router).

## Validity

- Issued: 2026-09-12
- Expires: 2026-12-11 (89 days, standard Let's Encrypt lifetime)
- CN / SAN: `login.techzone.lat`

See `docs/mikrotik-login-techzone-lat-certificate-renewal.md` in the
`mikrotik-mcp` repo for the full renewal procedure and the list of routers
this bundle was deployed to.
