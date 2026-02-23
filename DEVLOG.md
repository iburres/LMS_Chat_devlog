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

---

## 2026-02-23 — Channel creation UI

- `+` button appears next to "CHANNELS" label in sidebar for instructors and TAs only
- Modal with channel name (slugified with live preview), optional description, and "who can post" checkboxes (instructor always enabled, TA and students toggleable)
- Escape or clicking outside closes the modal
- On success, channels list updates immediately via React Query cache invalidation
- Both UI and backend (`requireRole('instructor', 'ta')`) enforce the restriction — students cannot create channels

---

### Architecture note
- `io` (Socket.io instance) now shared with Express routes via `req.io` middleware, enabling REST handlers to emit socket events directly.

---

## 2026-02-23 — Private Office Hours chat

Each student gets a private message thread with their instructor/TA.
No other students can see the conversation (FERPA compliant — enforced at API + socket layer).

### DB
- New `office_hour_messages` table: `id`, `office_hour_request_id`, `user_id`, `content`, `created_at`
- Index on `(office_hour_request_id, created_at ASC)` for efficient history fetch
- Migration: `server/migrations/002_office_hour_messages.sql`

### Server
- `GET /api/office-hours/:id/messages` — returns message history; `assertAccess` verifies student owns the request OR user is instructor/TA in the context
- `POST /api/office-hours/:id/messages` — inserts message, broadcasts `new_oh_message` via Socket.io to `oh:{requestId}` room
- `server/sockets/chat.js` — new `join_oh_session` / `leave_oh_session` handlers; access re-verified in socket layer before joining room

### Client
- `client/src/api/index.js` — added `fetchOHMessages`, `sendOHMessage`
- `client/src/hooks/useOHMessages.js` — loads history on mount, joins socket room, listens for `new_oh_message`, exposes `send()`
- `client/src/components/officeHours/OHChatPanel.jsx` — 300px inline chat panel with auto-scroll and Enter-to-send
- `client/src/pages/OfficeHours.jsx` — full light-theme rewrite; status badges (color-coded), "Open Chat" toggle per card (disabled for cancelled requests), instructor Schedule/Decline/Mark Complete controls

---

## 2026-02-23 — Unread counts + Office Hours badge

### Channel unread counts
- DB: `channel_reads (user_id, channel_id, last_read_at)` — PK on both columns
- `GET /api/channels/unread` — counts messages after `last_read_at` per channel (aggregate query)
- `POST /api/channels/:id/read` — upserts `last_read_at = GREATEST(existing, NOW())`
- `useUnreadCounts(activeChannelId)` hook: loads server counts, increments in real-time via `new_message` socket events for non-active channels, clears + persists when active channel changes
- Orange badge on channel items in sidebar; hidden when channel is active

### Office Hours pending badge
- `GET /api/office-hours/pending-count` — lightweight count query for instructors/TAs
- `POST /api/office-hours` now emits `oh_request_created` to `context:{contextId}` socket room
- `Chat.jsx` listens for `oh_request_created` and invalidates the count query → instant badge update
- Badge appears next to "Office Hours" link in sidebar (instructors/TAs only), refetches every 30s as fallback

### Demo data
- `devLogin.js` seeder now always ensures "Alex Chen" (student) exists with one pending OH request
- Guarantees instructors always see the badge on first login

---

## 2026-02-23 — Right-click context menus + channel leave/delete

### Context menu component
- `client/src/components/ContextMenu.jsx` — reusable portaled menu (renders into `document.body` via `createPortal` so it's never clipped by scroll containers)
- Dismisses on outside click or Escape; auto-clamps position to viewport edges
- Supports `{ label, action, danger? }` items and `'divider'` separators

### Message right-click
- Right-clicking any message you have permissions on shows Edit / Delete
- Complements existing hover buttons — same actions, alternate trigger
- No menu shown if the user has neither edit nor delete permission

### Channel right-click
- Right-click any channel in the sidebar to get a context menu
- **Leave Channel** (all roles) — inserts into `channel_leaves (user_id, channel_id)` table; channel is hidden from that user's list permanently (survives refresh); deselects channel if it was active
- **Delete Channel** (instructors/TAs only, shown in red) — hard-deletes the channel and cascades to messages; emits `channel_deleted` socket event to all context members so it disappears from everyone's sidebar in real time
- DB: `channel_leaves` table — migration 004
- `GET /api/channels` updated to exclude channels the user has left via `NOT EXISTS` subquery
- `POST /api/channels/:id/leave` — inserts leave record
- `DELETE /api/channels/:id` — instructor/TA only, emits socket event after delete

### Sidebar UX fix
- Removed `office-hours` from `DEFAULT_CHANNELS` seed — Office Hours is a dedicated page, not a chat channel
- Sidebar footer redesigned: user avatar/name/role in one row, full-width "Office Hours" / "Request Office Hours" button below with orange pending-count badge for instructors/TAs
