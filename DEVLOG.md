# LMS Chat — Dev Log

---

## 2026-02-22 — First successful boot + dev login

### What we did
Got the scaffolded app running end-to-end for the first time and added a dev login flow so the UI can be worked on without a real Canvas instance.

### Bugs fixed (startup)

**`ioredis` ESM import** — `ioredis` is a CommonJS module; named imports don't work in ESM.
- `import { createClient } from 'ioredis'` → `import Redis from 'ioredis'` + `new Redis(url)`
- Also removed the manual `.connect()` call — ioredis auto-connects on instantiation.

**`ltijs` ESM import** — default import doesn't expose `.setup()`.
- `import Lti from 'ltijs'` → `import { Provider as Lti } from 'ltijs'`

**`Lti.server` is a private field** — can't be accessed after `deploy()`.
- Solution: deploy with `serverless: true`, then wrap `Lti.app` in `http.createServer()` manually and pass that to Socket.io.

**ltijs blocks all routes by default** — including our `/api/*` routes, returning `NO_LTIK_OR_IDTOKEN_FOUND` for everything.
- Solution: `Lti.whitelist(new RegExp('^/api'))` — ltijs only guards the LTI launch flow; our own `authenticate` middleware handles API auth.

**Missing `.env`** — server crashed immediately on startup.
- Created `server/.env` from `.env.example` with `openssl rand -hex 32` generated secrets.

### Dev login

Added a dev-only login flow so we can work on the UI without Canvas:

- **`server/routes/devLogin.js`** — POST `/api/dev-login` seeds a fake platform, context, user, and membership in Postgres and creates a real session. Whitelisted from ltijs. Must be mounted before `apiRouter` so the `authenticate` middleware doesn't block it.
- **`client/src/pages/DevLogin.jsx`** — React form (name + role picker). Calls the API via axios (`withCredentials: true`), then does `window.location.href = '/app'` on success. Shown automatically in dev mode when there's no session.
- Vite proxy redirect + `Set-Cookie` was unreliable (cookie lost through proxy redirect) — solved by returning JSON and redirecting client-side instead.

### Docker
- Pulled and started postgres:16-alpine, redis:7-alpine, mongo:7 images.
- `docker-compose.yml`: removed deprecated `version: '3.8'` field (Docker Compose v2).
- Migrations ran cleanly against fresh Postgres.

---

## 2026-02-22 — Light theme + message edit/delete

### Light theme
Replaced the Discord dark theme with a Canvas-compatible light theme:
- Backgrounds: white sidebar, `#F5F7FA` main area
- Text: `#1F2D3D` (dark navy-gray), `#6B7780` secondary, `#9EA7AD` muted
- Accent: Canvas Orange `#E66000` — buttons, active channel highlight, own message bubbles, avatars
- Borders: `#DCE0E5`
- Active channel: `#FFF0E8` background, `#C4520A` text

### Message edit
- Author-only inline edit (pencil icon appears on hover)
- Bubble turns into a textarea with Save/Cancel controls
- Enter saves, Escape cancels
- Edited messages show `(edited)` label
- **FERPA**: original content stored in `audit_logs.metadata` before overwrite

### Message delete
- Author or instructor/TA can delete (trash icon on hover)
- Soft-delete preserved in DB (`deleted_at`), FERPA compliant
- Both edit and delete broadcast real-time via Socket.io to all channel members

### Architecture note
- `io` (Socket.io instance) now shared with Express routes via `req.io` middleware, enabling REST handlers to emit socket events directly.
