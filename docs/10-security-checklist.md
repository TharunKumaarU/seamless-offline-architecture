# 10 — Security Checklist

*The self-inflicted wounds of internal business platforms, and the boring disciplines that prevent them.*

Field platforms rarely fall to exotic attacks. They fall to a `.env` committed in week one, a default admin password that outlived the demo, and an approval endpoint that trusted an ID from the request body. This checklist is ordered by how often each item actually bites.

## 1. Secrets hygiene (where the bodies are buried)

- **Nothing secret in the repo — including history.** A secret committed once is compromised forever unless history is rewritten *and* the secret rotated; "we deleted it in a later commit" protects nothing, and "the repo is private" is one org-membership change away from false. The moment a leak is found: **rotate first**, tidy git second.
- Ship `.env.example` with every key and a fake value; real `.env` files are gitignored *from the first commit* (pre-commit secret scanners — gitleaks-class — make this structural, not disciplinary).
- Parameterize compose/CI files (`${DB_PASSWORD}`) — hardcoded values in `docker-compose.yml` are the second-most-common leak site after `.env`.
- Service-account key files (push-notification credentials and friends) are deploy-time artifacts from a secret store, never repo contents — and never sitting in the working tree where a future "add ." sweeps them in.
- Mobile signing keystores + passwords: secret store + offline backup only (doc 08).
- **Backups and data exports are secrets too.** Database dumps and production extracts must not live in project folders that sync, zip, or ship anywhere — they are the single densest leak object you own. Dedicated encrypted location, retention policy, done.

## 2. Default and bootstrap credentials

- Bootstrap admin accounts get a **generated** password printed once at provision time — never a literal `changeme` in an init script that "everyone knows gets changed" (it doesn't).
- Audit for re-seeding surprises: an idempotent startup routine that "ensures admin exists" must never *reset* an existing admin's password on redeploy — that's a self-inflicted account takeover.
- Password-reset utility scripts with hardcoded values are radioactive; parameterize or delete.

## 3. Token lifecycle for offline fleets

The tension: short-lived tokens are safer; field devices are offline for days. Resolve it deliberately (this pairs with doc 02's auth section):

- Access tokens short (hours–days), **refresh tokens** long, rotated on use, revocable server-side per device.
- Refresh **proactively with a wide buffer** — a fleet whose tokens all expire during a long weekend produces a Monday-morning 401 storm that looks exactly like an outage (the 401 panel in doc 09 is how you tell them apart).
- A refresh endpoint that requires a *still-valid* token is a design bug for field fleets: pair it with a device-bound refresh credential that outlives the access token, or accept that every deep-offline period ends in re-login (and make offline re-login not lose the outbox — doc 02).
- On password change / device loss: revoke that device's refresh credential; don't rotate global signing keys as an ops habit — that logs out the entire fleet at once (same blast radius as the long-weekend storm, but self-inflicted).

## 4. Authorization (the IDOR farm)

Covered in depth in doc 04; the security-review shortlist:

- Tenant filter applied by construction (single chokepoint), including on **aggregates and dashboards**.
- Write paths validate the tenant of every **referenced** entity, not just the new row.
- Read scopes (own/team/tenant) enforced server-side on every list.
- **All gates fail closed** on error or missing config — an approval gate that fails open under malformed data is the exploit *and* the incident.
- Object storage / document downloads: signed URLs or authenticated proxy — never public-but-unlisted links to business documents; "the URL has a UUID in it" is not access control.

## 5. Web/API basics that get skipped under deadline

- CORS: explicit origin allow-list per environment. `allow_origins=["*"]` (with credentials) is a to-do item that ships to production and stays there — put an environment-gated assertion in startup checks so prod literally refuses to boot wide-open.
- Rate limiting on auth endpoints; uniform "invalid credentials" errors (no user-enumeration oracle).
- Passwords: bcrypt/argon2 with per-user salts (a library, not homegrown); never log payloads containing credentials or tokens (structured logging with an explicit redact-list, doc 09).
- Uploads: validate type/size server-side, strip EXIF GPS from field photos at ingest (photo metadata is a location trail of your workforce), store outside the web root, serve with content-type discipline.
- HTTPS everywhere including internal staging (free certs removed the excuse); HSTS on the portals. Mixed-content bugs — an https page assembling http asset URLs from a stale config — are found by the browser console, not by users, so check it after every base-URL change.

## 6. Startup checks: turn policy into code

The cheapest security control in this document — the app refuses to boot (or loudly warns) when:

```python
STARTUP_CHECKS = [
    ("SECRET_KEY set and not a known default", check_secret_key),
    ("DB password not 'postgres'/'password'",  check_db_password),
    ("CORS not wildcard in production",        check_cors),
    ("Bootstrap admin password not literal default", check_admin_seed),
    ("Debug mode off in production",           check_debug_flag),
]
```

Every item above graduated from a real class of production finding to a boot-time assertion. That's the pattern: **each incident retires into an automatic check** — the checklist executes itself.

## 7. Review cadence

- Secret scan (gitleaks-class) in CI on every push; full-history scan quarterly.
- Dependency audit (pip-audit / npm audit / pub audit) monthly and before releases.
- An "auth walk" before each major release: one engineer, one hour, trying the five IDOR shapes in section 4 against the new endpoints.
- Post-incident: the mandatory question is "which startup check or CI gate would have caught this?" — and then adding it.
