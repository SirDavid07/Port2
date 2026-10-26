# Neural OS portfolio

A responsive futuristic portfolio (desktop console and mobile layouts), with a SQLite database,
a hardened Node API and an admin panel to edit everything without touching code.

No npm dependencies. Needs Node 22.13 or newer (uses the built-in `node:sqlite`).

## Run it

```bash
cp .env.example .env          # then edit it
npm run create-admin          # prompts for a username and a 12+ character password
npm start                     # http://127.0.0.1:3000
```

- Site: `/`
- Admin: `/admin/` (projects, labs, profile, inbox, password, activity log)
- Tests: `npm test`

For local development set `NODE_ENV=development` in `.env` (production mode sets `Secure` cookies,
which browsers only accept over HTTPS).

## Layout

```
public/            static site (index.html, css/, js/, admin/)
src/app.js         HTTP server, routing, global limits, error handling
src/routes.js      public and admin API
src/db.js          SQLite schema, seed data, audit log
src/auth.js        scrypt hashing, sessions, cookies
src/security.js    CSP and security headers, origin check
src/validate.js    allowlist validation and sanitising
src/ratelimit.js   in-memory rate limiter
scripts/           create-admin.js
test/              API and security tests
```

## Database

SQLite file at `DB_PATH` (default `data/portfolio.db`, mode 0600, outside `public/`).
Tables: `profile`, `projects`, `messages`, `admins`, `sessions`, `login_failures`, `audit_log`, `settings`.
All queries are prepared statements, and CHECK constraints enforce lengths and enums at the database level.
Back it up by copying the file while the app is stopped, or with `sqlite3 data/portfolio.db ".backup backup.db"`.

## Security measures

| Area | What is done |
| --- | --- |
| Injection | Prepared statements only. Strict allowlist validation per field. Control and bidi characters stripped. |
| XSS | CSP with no `unsafe-inline` (no inline script or style anywhere). The front end renders data with `textContent`, never `innerHTML`. Links must be `https://`. |
| Auth | scrypt (N=32768) with per-password salt. Constant-time compare. Dummy hash for unknown users. Same error for wrong user or password. |
| Brute force | Per-IP login rate limit plus per-IP+username lockout (5 failures, 15 minutes). |
| Sessions | 256-bit random token, stored only as a SHA-256 hash. `HttpOnly`, `SameSite=Strict`, `Secure` and `__Host-` prefix in production. 30 minute idle and 8 hour absolute expiry. Max 5 sessions. Password change signs out other devices. |
| CSRF | Per-session token required on every admin write, plus Origin / Sec-Fetch-Site checks. |
| Abuse | Global, contact, login and admin rate limits. Honeypot field. Daily cap on stored messages. 16 KB body limit. Request and header timeouts. |
| Privacy | Visitor IPs are never stored, only a salted hash. Audit log purged after 180 days. |
| Headers | CSP, HSTS (production), `X-Frame-Options: DENY`, `nosniff`, `Referrer-Policy: no-referrer`, COOP/CORP, Permissions-Policy. |
| Static files | Realpath check (no traversal or symlink escape), dotfiles and unknown extensions never served. |

## Deploying

1. Put it behind HTTPS (Caddy, nginx, or your host's TLS). The app itself speaks plain HTTP on `127.0.0.1`.
2. Set `NODE_ENV=production` and `TRUST_PROXY=1` (number of proxies in front). Without it every visitor shares one rate limit.
3. Run `npm run create-admin`, then make sure `ADMIN_PASSWORD` is not left in `.env`.
4. Keep `data/` and `.env` out of version control (already in `.gitignore`) and back up `data/`.
5. Run it under a process manager (systemd, pm2, Docker) as a non-root user.

## Known limits

- Rate limits and sessions' idle tracking are per process. Run a single instance, or move limits to a shared store if you scale out.
- Contact messages are stored for you to read in the admin panel. There is no outgoing email.
- No two-factor login yet. Use a long unique password; a TOTP step is the natural next addition.
