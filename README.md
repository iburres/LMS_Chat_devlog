# LMS Chat — Dev Log

Public development journal for **LMS Chat**, a student-instructor chat service built for Canvas and other LTI-compatible learning management systems.

> The app itself is private. This devlog is public — building in the open.

---

## What is LMS Chat?

A channel-based real-time chat tool that lives inside Canvas as an LTI 1.3 app. Students can talk to each other, ask instructors questions, request office hours, and search past answers — all scoped to their specific course.

**Key properties:**
- LTI v1.3 + OAuth2 — launches directly from within Canvas
- FERPA compliant — PII encrypted at rest, full audit trail
- Course-isolated — students in PROG101 cannot see PROG102 chats
- Real-time — Socket.io with Redis for live presence and messaging

---

## Stack

| Layer | Tech |
|---|---|
| Backend | Node.js + Express |
| LTI | ltijs (LTI v1.3) |
| Real-time | Socket.io + Redis adapter |
| Database | PostgreSQL + Redis |
| LTI internal | MongoDB (ltijs) |
| Frontend | React + Vite |

---

## Dev Log

---

### 2026-02-22 — Entry 001: Project scaffold

Full project scaffold from scratch. 41 files across server and client.

**Server highlights:**
- `ltijs` wired up for LTI v1.3 — OIDC login flow, JWT validation, JWKS, platform registration
- PostgreSQL schema with course isolation baked in at the FK level (`context_id` on every table)
- PII (student name + email) encrypted with AES-256-GCM before storage — key never touches the DB
- `audit_logs` table records every read/write of student data per FERPA requirements
- Role-based middleware: `instructor` / `ta` / `student` — enforced on every route
- REST API: session bootstrap, channels, paginated messages, soft-delete, office hours, full-text search
- Full-text search via PostgreSQL `tsvector` + GIN index — no external search engine needed
- Socket.io with Redis adapter for horizontal scaling
- Typing indicators + presence tracking via Redis sets
- docker-compose: Postgres 16, Redis 7, MongoDB 7

**Client highlights:**
- Channel-based UI in React + Vite
- Channel sidebar, live chat window, infinite scroll message history
- Search overlay with ranked results scoped to the current course
- Office hours request + scheduling UI (role-aware — instructors see all requests)
- `useSocket` / `useMessages` / `usePresence` hooks
- Vite proxy — `/api` and `/socket.io` forward to Express in dev

---

### 2026-02-22 — Entry 002: First successful boot + dev login

Got the scaffolded app running end-to-end for the first time.

**Startup bugs fixed:**
- `ioredis` ESM import — changed to `import Redis from 'ioredis'` + `new Redis(url)`; removed manual `.connect()` (ioredis auto-connects)
- `ltijs` ESM import — changed to `import { Provider as Lti } from 'ltijs'`
- `Lti.server` is private — deploy with `serverless: true`, wrap `Lti.app` in `http.createServer()`, pass to Socket.io
- ltijs blocks all routes by default — `Lti.whitelist(new RegExp('^/api'))` lets our own auth handle API routes
- Missing `.env` — created `server/.env` from `.env.example` with generated secrets

**Dev login flow:**
- `POST /api/dev-login` seeds a fake platform, context, user, and membership in Postgres and creates a real session
- React form (name + role picker) calls the API then does `window.location.href = '/app'` on success
- Solved a cookie-through-proxy-redirect bug by returning JSON and redirecting client-side

---

### 2026-02-22 — Entry 003: Light theme + message edit/delete

**Light theme** — Canvas-compatible palette:
- Accent: Canvas Orange `#E66000`
- Backgrounds: white sidebar, `#F5F7FA` main area
- Text: `#1F2D3D` (primary), `#6B7780` (secondary), `#9EA7AD` (muted)

**Message edit:**
- Author-only inline edit; textarea replaces bubble on click
- Enter saves, Escape cancels; `(edited)` label shown after save
- FERPA: original content stored in `audit_logs.metadata` before overwrite

**Message delete:**
- Author or instructor/TA; soft-delete (`deleted_at`) preserves audit trail
- Both edit and delete broadcast in real time via Socket.io

---

### 2026-02-23 — Entry 004: Channel creation UI

- `+` button in sidebar for instructors/TAs only
- Modal: channel name (slugified with live preview), description, "who can post" checkboxes
- Escape or click-outside closes; React Query cache invalidated on success
- `requireRole('instructor', 'ta')` enforced on both UI and backend

---

### 2026-02-23 — Entry 005: Private Office Hours chat

Each student gets a private message thread with their instructor/TA — no other students can see it (enforced at API + socket layer).

- New `office_hour_messages` table; index on `(office_hour_request_id, created_at ASC)`
- `GET/POST /api/office-hours/:id/messages` — access checked: student owns request OR user is instructor/TA
- Socket room `oh:{requestId}`; `join_oh_session` / `leave_oh_session` events with server-side access re-verification
- `useOHMessages` hook + `OHChatPanel` inline chat panel (auto-scroll, Enter-to-send)
- Office Hours page fully rewritten: light theme, color-coded status badges, "Open Chat" toggle per card

---

### 2026-02-23 — Entry 006: Unread counts + Office Hours badge

**Channel unread counts:**
- `channel_reads (user_id, channel_id, last_read_at)` table
- `GET /api/channels/unread` + `POST /api/channels/:id/read`
- `useUnreadCounts` hook: loads server counts, increments live via socket, clears when channel opens
- Orange badge on channel items in sidebar

**Office Hours pending badge:**
- `GET /api/office-hours/pending-count` for instructors/TAs
- `oh_request_created` socket event triggers instant badge update
- Demo seed: "Alex Chen" student + pending OH request always present on dev login

---

### 2026-02-23 — Entry 007: Right-click context menus + channel leave/delete

**Reusable `ContextMenu` component:**
- Portaled into `document.body` — never clipped by scroll containers
- Auto-clamps to viewport edges; dismisses on outside click or Escape

**Message right-click:** Edit / Delete (respects author/role permissions)

**Channel right-click:**
- Leave Channel (all roles) — inserts `channel_leaves (user_id, channel_id)`; channel hidden permanently for that user
- Delete Channel (instructors/TAs, shown in red) — hard-deletes + cascades; emits `channel_deleted` socket event so it disappears from everyone's sidebar in real time

---

### 2026-02-23 — Entry 008: Mobile-responsive layout

- `useMediaQuery` hook — reactive wrapper around `window.matchMedia`
- Breakpoint: 640px — sidebar collapses off-screen, slides in as fixed overlay with backdrop
- Selecting a channel auto-closes the sidebar
- `ChatWindow` mobile header: hamburger (☰) + `#channel-name`
- `MessageInput` padding uses `env(safe-area-inset-bottom)` for iOS home indicator
- `viewport-fit=cover` for notched devices

---

### 2026-02-23 — Entry 009: Dark mode toggle

- CSS custom properties on `<html>` via `data-theme` attribute — no CSS-in-JS library
- `ThemeContext` reads `localStorage`, falls back to `prefers-color-scheme`; persists across reloads
- Toggle button in sidebar footer: `"Dark"` / `"Light"`
- All 13 component files migrated from hardcoded hex to `var(--color-xxx)` references

---

### 2026-02-24 — Entry 010: Typing indicators + online presence

**Typing indicators:**
- `typing_start` event now carries `name`; server forwards it in `user_typing` broadcast
- `useTypingIndicator(channelId)` hook — auto-clears any user after 3s; returns `string[]` of names
- `MessageInput` fires `typing_start/stop` on keystrokes (stops after 2s idle or on send)
- `ChatWindow` shows animated triple-dot + "Alex is typing…" / "Several people are typing…" in a fixed 22px bar (no layout shift)

**Online presence:**
- `usePresence()` in `Chat.jsx`; `onlineIds: Set` passed to `ChannelList` + `ChatWindow`
- Green dot on own avatar in sidebar footer
- `• N online` count next to channel list header + in channel header
- Green dot on sender avatar in message bubbles when that user is online

---

### 2026-02-24 — Entry 011: Emoji reactions, reply, forward, threads

**Emoji reactions:**
- Six reactions: 👍 👎 ❤️ 😂 ✅ ❓
- Emoji picker on hover; reaction pills below bubble with emoji + count
- `POST /api/messages/:id/reactions` — upserts/deletes from `message_reactions`; emits `reaction_updated` via socket

**Unified hover action bar:**
- Replaced right-click context menu with a hover action bar above the bubble
- Buttons with tooltips: ☺ Reactions · ↩ Reply · ↗ Forward · ✏ Edit · ✕ Delete
- Each button only renders if the user has the relevant permission

**Reply (Slack/iPhone style):**
- ↩ Reply sets `replyTo` in `ChatWindow`; reply banner shows in `MessageInput`
- Reply is sent as a new main-feed message with a quote block above the bubble (accent bar + sender name + 100-char snippet)

**Forward:**
- ↗ Forward opens `ForwardModal` — channel picker; posts content to selected channel via REST

**Thread backend:**
- Migration `006_threads_replies.sql` adds `reply_to_id` + `thread_root_id` to `messages`
- `thread_root_id IS NULL` keeps thread replies out of the main feed
- `GET/POST /api/messages/:id/thread`; emits `new_thread_reply` + `thread_reply_count_updated`

---

### 2026-02-24 — Entry 012: Real-time bug fixes

**Root bug — `req.io` always undefined:**
- `req.io` middleware was registered *after* the API routes in `server/index.js`. Express runs middleware in order, so every route handler ran before `req.io` was set. Optional-chaining (`req.io?.emit`) silently swallowed all socket emissions.
- Fix: declared `let _io = null` before route registration. Middleware captures it by closure reference; `_io = io` is assigned after Socket.io initialises.

**Reactions now update live:**
- `addReaction` / `removeReaction` now update local state immediately from the REST response — no longer reliant solely on the socket event. Socket event still fires for other clients.

**Sent messages appear immediately:**
- `sendMessage` now optimistically adds the message from the socket ack payload. `onNew` deduplicates by ID to prevent double-add from the broadcast.

---

### 2026-02-24 — Entry 013: Emoji reaction redesign — accumulate counts, names, right-click remove

Previous behavior toggled a reaction off on second click. The new model matches Slack:

**Behavior:**
- **Left-click emoji picker** — always *adds*. Picker highlights emojis you've already added. Clicking an already-reacted emoji again is a no-op.
- **Left-click reaction pill** — adds the reaction if you haven't yet; no-op if you have.
- **Right-click reaction pill** — confirm prompt to remove *only your own* reaction. Count decrements by 1; other users' reactions are unaffected.
- **Hover a pill** — tooltip lists every user who reacted (e.g. "Ian, Alex").

**Backend:**
- `POST /messages/:id/reactions` — add-only; skips duplicate insert
- `DELETE /messages/:id/reactions/:emoji` — new endpoint; removes requesting user's reaction only; emits `reaction_updated`
- `getReactions()` now JOINs `users` to return decrypted `names[]` alongside counts
- `formatMessage()` decrypts `nameEncrypteds` from the SQL reactions subquery

**Frontend:**
- `addReaction` + `removeReaction` replace `toggleReaction` in `api/index.js`
- Both update local state from REST response immediately
- `MessageBubble` pill/picker behavior as above; `onRemoveReact` threaded through `MessageList` + `ChatWindow`

---

## Testing

This prototype is planned for testing with students and instructors at the **University of Texas San Antonio (UTSA)**.

---

*Built with [Claude Code](https://claude.ai/claude-code)*
