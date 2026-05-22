# UniClub

UniClub is a full-stack university club management platform that consolidates club operations — clubs, members, advisors, board members, events, registrations, budgets, sponsorships, and messaging — into a single role-aware web app.

# Team Project
Developed collaboratively with my teammates.

# My Contribution
- Frontend/UI Eng.

**Live demo**

- Frontend: https://uni-club-bay.vercel.app
- Backend: https://uniclub-production.up.railway.app
- API docs (Swagger): https://uniclub-production.up.railway.app/docs

## What problem this solves

University club operations are fragmented across spreadsheets, group chats, and ad-hoc tracking. That fragmentation produces inconsistent records, weak access control, and poor visibility into events, registrations, budgets, and sponsorships.

UniClub puts every workflow behind a single role-based UI: students join clubs, board members run them, advisors approve, and admins govern through a permission matrix that can be edited live without redeploying.

## Highlights

- **5 user roles** (admin, advisor, board member, member, guest) with per-permission grants stored in the database and editable from an admin dashboard.
- **3 second-factor methods** that can be combined per account: TOTP authenticator app (RFC 6238), Email OTP, and WebAuthn / passkey.
- **2 social login providers** wired up via Authlib OpenID Connect: Google and GitHub.
- **Forgot password / reset flow** with single-use, hashed, 1-hour TTL tokens and an enumeration-safe API response.
- **Transactional email via the Resend HTTP API**, with a transport priority chain that falls back to SMTP and then to a console logger so local dev and CI never need a provider account.
- **CI on every push**: backend `pytest --cov` against an in-memory SQLite (90% threshold), frontend `vitest`, frontend production build, and Playwright e2e against a real Postgres + FastAPI + Vite preview stack.

## Architecture

```
Browser ── HTTPS ──> Vercel (React SPA, static + edge cache)
                         │
                         └─ XHR ──> Railway (FastAPI + Uvicorn)
                                       │
                                       ├─ SQL ─> PostgreSQL (Railway managed)
                                       ├─ HTTPS ─> Resend (transactional email)
                                       └─ OAuth ─> Google / GitHub
```

CI runs in GitHub Actions on every push and pull request.

## Tech stack

**Backend** — FastAPI · SQLModel + SQLAlchemy · Pydantic v2 · Uvicorn · python-jose (JWT) · passlib + bcrypt · pyotp (TOTP) · py-webauthn · authlib (OAuth) · httpx (Resend client)

**Frontend** — React 19 + TypeScript · Vite · React Router · TanStack React Query · Axios · react-hook-form + Zod · Tailwind CSS · react-hot-toast

**Infrastructure** — Railway (API + Postgres) · Vercel (web) · Resend (email) · Google + GitHub OAuth · GitHub Actions (CI) · Playwright (e2e)

## Repository layout

| Path | Description |
| --- | --- |
| `uniclub-api/` | FastAPI backend, models, routers, tests |
| `uniclub-web/` | React + Vite frontend |
| `backend prompts/` | Iterative prompt history for backend work |
| `frontend prompts/` | Iterative prompt history for frontend work |
| `database prompts/` | Schema and data-modeling prompts |
| `api integration & validation prompts/` | Cross-stack integration prompts |
| `screenshots/` | UI captures referenced from the README |
| `responsibilities/` | Team responsibility notes |

## Authentication and authorization

| Feature | Endpoint(s) | Notes |
| --- | --- | --- |
| Email + password register / login | `POST /auth/register`, `POST /auth/login` | bcrypt-hashed; JWT issued on success, or a short-lived challenge token if 2FA is enabled. |
| Google / GitHub OAuth | `GET /auth/oauth/{provider}/login` and `/callback` | OpenID Connect via Authlib; auto-provisions or links users. |
| TOTP 2FA | `/2fa/totp/setup`, `/confirm`, `/disable`, `/login/totp/verify` | RFC 6238 codes via `pyotp`, ±1 step window. |
| Email OTP 2FA | `/2fa/email/enable`, `/disable`, `/login/email/send`, `/verify` | 6-digit code, hashed at rest, 10-minute TTL. |
| WebAuthn 2FA | `/2fa/webauthn/register/*`, `/login/webauthn/*` | Hardware key or platform passkey; sign-count enforced. |
| Forgot password | `POST /auth/password/forgot`, `POST /auth/password/reset` | 32-byte urlsafe token, SHA-256 hash stored, single-use, 1-hour TTL; sibling tokens are invalidated on use. Returns `200` regardless of email validity to prevent account enumeration. |
| Permission matrix | `GET/PUT /admin/permissions` | Admins edit role-to-permission grants live; changes apply immediately without redeploy. |

The forgot endpoint deliberately responds with a generic 200 so an attacker cannot enumerate registered emails. Reset tokens are stored only as SHA-256 hashes — the raw token only ever appears in the email link.

## Email delivery: SMTP → Resend pivot

Initial implementation used Gmail SMTP. Railway, like most managed PaaS platforms, blocks outbound SMTP ports 587 / 465 to discourage spam abuse from free-tier projects. The first send attempt failed with `[Errno 101] Network is unreachable` and no env-var change could fix it.

Solution: switched delivery to the Resend HTTP API and added a transport priority chain in `email_utils.py`:

1. **Resend** — used when `RESEND_API_KEY` is set (production).
2. **SMTP** — used when `SMTP_HOST` is set (kept for environments where SMTP is unblocked).
3. **Console fallback** — used otherwise (local dev and CI).

The fallback prints the OTP code to stdout so local development and CI never require a real mail provider account.

## Running locally

### Backend

```powershell
cd uniclub-api
python -m venv ..\.venv
..\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env   # then edit with your values
python -m uvicorn main:app --reload
```

Backend defaults to http://127.0.0.1:8000 — Swagger at `/docs`, health at `/health/db`.

### Frontend

```powershell
cd uniclub-web
npm install
npm run dev
```

Frontend defaults to http://localhost:5173.

### Tests

```powershell
# Backend tests + coverage gate
cd uniclub-api
pytest --cov

# Frontend unit tests
cd uniclub-web
npm test

# End-to-end (requires Postgres + a built frontend)
npx playwright test
```

## Environment variables

### Backend (`uniclub-api/.env`)

| Variable | Required | Description |
| --- | --- | --- |
| `DATABASE_URL` | yes | PostgreSQL DSN (e.g. `postgresql+psycopg://user:pass@host:5432/uniclub_db`). Railway-style `postgres://` URLs are auto-normalized. |
| `SECRET_KEY` | yes | JWT signing key. |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | no | Default `1440`. |
| `SEED_*_EMAIL` / `SEED_*_PASSWORD` | no | Seed credentials for the demo accounts; deterministic fallback if unset. |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | no | Enables Google OAuth. |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | no | Enables GitHub OAuth. |
| `MICROSOFT_CLIENT_ID` / `MICROSOFT_CLIENT_SECRET` | no | Enables Microsoft OAuth. |
| `FACEBOOK_CLIENT_ID` / `FACEBOOK_CLIENT_SECRET` | no | Enables Facebook OAuth. |
| `OAUTH_REDIRECT_BASE` | no | Base URL the providers redirect back to (e.g. `https://uniclub-production.up.railway.app`). |
| `OAUTH_FRONTEND_REDIRECT` | no | URL the backend bounces the user to after a successful OAuth exchange. |
| `RESEND_API_KEY` | no | Enables the Resend HTTP transport for OTP and reset emails. |
| `RESEND_FROM` | no | `From:` value for Resend (e.g. `onboarding@resend.dev` until a domain is verified). |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASSWORD` / `SMTP_FROM` / `SMTP_USE_TLS` | no | Optional SMTP transport (only used if `RESEND_API_KEY` is unset). |
| `FRONTEND_BASE_URL` | no | Used to build links in transactional emails (default `http://localhost:5173`). |
| `WEBAUTHN_RP_ID` / `WEBAUTHN_ORIGIN` / `WEBAUTHN_RP_NAME` | no | Required for WebAuthn registration to work outside `localhost`. |

### Frontend (`uniclub-web/.env`)

| Variable | Required | Description |
| --- | --- | --- |
| `VITE_API_BASE_URL` | yes (in non-default deployments) | Backend base URL, e.g. `https://uniclub-production.up.railway.app`. |

## CI

`.github/workflows/ci.yml` defines three jobs that run on every push and pull request to `main`:

1. **backend** — installs Python deps and runs `pytest --cov` with `fail_under = 90` on the security-critical modules (`auth.py`, `permissions_catalog.py`, `routers/admin.py`, `routers/twofa.py`, `email_utils.py`).
2. **frontend-unit** — installs npm deps, runs the vitest suite, then `npm run build` to fail on TypeScript errors.
3. **e2e** — depends on the two above, spins up a Postgres service, starts the API on `:8000` and the Vite preview on `:5173`, then runs the Playwright suite against the real stack.

## Screenshots

The `screenshots/` directory holds UI captures used in the demo and report. Notable captures:

- **Auth flow** — login screen, register screen, Google OAuth consent, 2FA challenge tabs.
- **Email OTP delivery** — Resend dashboard showing successful 200 responses.
- **Forgot password flow** — request screen, generic success message, reset email, new-password screen.
- **Admin permission matrix** — live editing of role-to-permission grants.
- **Dashboard** — health indicator, club overview, event capacity pulse.

See `uniclub-api/README.md` for backend SQL diagnostics screenshots and `uniclub-web/README.md` for the page-by-page route map.

## Sub-project documentation

- Backend setup, seed accounts, and SQL diagnostics: [`uniclub-api/README.md`](uniclub-api/README.md)
- Frontend routes, services, and authorization rules: [`uniclub-web/README.md`](uniclub-web/README.md)

## License

This project is for educational and demonstration purposes.
