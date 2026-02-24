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
Replaced the default dark theme with a Canvas-compatible light theme:
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

---

## 2026-02-23 — Bug fixes: server crash + channel delete

### Server crash on channel delete
- Root cause: `POST /channels/:id/read` inserted into `channel_reads` with FK on `channel_id`; if the channel was deleted between client sending the request and server processing it, a FK violation crashed the server process
- Fix: changed INSERT to use `SELECT ... WHERE EXISTS (SELECT 1 FROM channels WHERE id = $2)` — silently no-ops if channel is gone

### Channel not removed from sidebar after delete
- Root cause: `handleDelete` relied on the socket round-trip to call `qc.invalidateQueries`; if server crashed during that window the UI stayed stale
- Fix: `removeFromCache(channelId)` optimistically removes the channel from the React Query cache before the API call — instant UI update regardless of server state

---

## 2026-02-23 — Mobile-responsive layout

- `useMediaQuery(query)` hook — reactive wrapper around `window.matchMedia`
- **Breakpoint: 640px**
- Sidebar collapses off-screen on mobile; slides in as a fixed overlay (`transform: translateX`) with 0.25s transition
- Semi-transparent backdrop dismisses sidebar on tap
- Selecting a channel auto-closes the sidebar
- Empty state shows a "Browse channels" button on mobile
- `ChatWindow` gains a mobile top bar: hamburger (☰) + `#channel-name`
- `MessageInput` padding uses `env(safe-area-inset-bottom)` for iOS home indicator
- `viewport-fit=cover` added to meta viewport for notched devices
- `ChannelList.onSelect` now passes full `{id, name}` channel object so Chat.jsx can display the name in the mobile header without an extra query

---

## 2026-02-23 — Dark mode toggle

### Approach
CSS custom properties (`var()`) on the `<html>` element via a `data-theme` attribute — no CSS-in-JS library needed.

### CSS variables (`client/index.html`)
Defined on `:root` (light) and `[data-theme="dark"]` (override):
- `--color-bg` / `--color-surface` / `--color-border`
- `--color-text` / `--color-text-secondary` / `--color-text-muted`
- `--color-accent` (Canvas Orange `#E66000` — unchanged in both themes)
- `--color-accent-subtle` / `--color-accent-active`
- `--color-bubble` (message bubble background)
- `--color-danger`

### ThemeContext (`client/src/context/ThemeContext.jsx`)
- `ThemeProvider` reads `localStorage('lms-theme')`, falls back to `prefers-color-scheme`
- Sets `document.documentElement.dataset.theme` on every change
- Exposes `{ theme, toggle }` via `useTheme()` hook
- Wrapped around `<AppProvider>` in `App.jsx`

### Toggle button
- Added to Chat.jsx sidebar footer (right side of user row)
- Small text button: shows `"Dark"` in light mode, `"Light"` in dark mode
- Preference persists across page reloads via localStorage

### Components updated
All inline `style` objects across 13 files migrated from hardcoded hex colors to `var()` references:
`Chat`, `ChannelList`, `ChatWindow`, `MessageBubble`, `MessageInput`, `CreateChannelModal`, `SearchBar`, `ContextMenu`, `OHChatPanel`, `OfficeHours`, `DevLogin`, `App`

### Notes
- Status badge colors in `OfficeHours` (`pending`/`scheduled`/`completed`/`cancelled`) kept as hardcoded semantic colors — they are self-contained with their own bg+text pair and read well in both modes
- White text on accent/danger buttons (`color: '#fff'`) kept as-is — white on orange is correct in both themes

---

## 2026-02-24 — Typing indicators + online presence dots

### Typing indicators
- **Server:** `typing_start` event now accepts `{ channelId, name }` and forwards `name` in the `user_typing` broadcast
- **`useTypingIndicator(channelId)`** (new hook) — listens for `user_typing`/`user_stop_typing` for the active channel; auto-clears any user after 3s of no update; returns `string[]` of names currently typing
- **`MessageInput`** — now accepts `channelId` prop; fires `typing_start { channelId, name }` on every keystroke (auto-stops after 2s idle or on send); imports `useApp` to read own display name
- **`ChatWindow`** — shows a 22px typing bar below the message list: animated triple-dot + "Alex is typing…" / "Alex and Bob are typing…" / "Several people are typing…"; bar is always rendered (no layout shift)
- **CSS animation** — `@keyframes typing-blink` added to `index.html`; `.typing-dot` class drives it (mix of class + inline styles, necessary for keyframes)

### Online presence
- **`usePresence()`** moved to `Chat.jsx` (was in `ChannelList`); `onlineIds: Set<userId>` passed as a prop to both `ChannelList` and `ChatWindow`
- **Sidebar footer** — green dot (9px) overlaid on own avatar; confirms the user is connected
- **Channel list** — small `• N` online count next to "CHANNELS" label
- **Channel header** — `ChatWindow` now has a persistent header (desktop + mobile): `#channel-name` + `• N online` count; replaces the mobile-only header that was there before
- **Message bubble avatars** — green dot (10px, white border) at bottom-right of sender avatar when that user is online; `isOnline` prop passed from `MessageList` → `MessageBubble`

---

## 2026-02-24 — Emoji reactions, message action bar, reply, forward, threads

### Emoji reactions
- Six supported reactions: 👍 👎 ❤️ 😂 ✅ ❓
- Emoji picker opens from the ☺ action button on hover; clicking an emoji toggles the reaction for the current user
- Reaction pills appear below the bubble showing emoji + count; clicking a pill toggles the same reaction
- A pill highlights in accent color when the current user has reacted; otherwise neutral border style
- Server: `POST /api/messages/:messageId/reactions` — upserts or deletes from `message_reactions`; returns full recalculated reaction array; emits `reaction_updated` via socket to the channel room
- Client: `reaction_updated` socket event updates the affected message in state without a full refetch

### Message action bar redesign
- Removed right-click context menu for messages; replaced with a unified hover action bar
- Action bar appears at top-right of the bubble on hover (opacity transition, no layout shift)
- Buttons with tooltip-on-hover: ☺ Reactions · ↩ Reply · ↗ Forward · ✏ Edit · ✕ Delete
- Each button only renders if the user has the relevant permission (edit/delete gated by author or instructor/TA role)
- `ActionBtn` wrapper component handles tooltip state independently per button

### Reply
- Clicking ↩ sets `replyTo` state in `ChatWindow`
- `MessageInput` shows a reply banner above the text field: accent-colored left bar + sender name + 60-char snippet; ✕ dismisses
- On send, `replyToId` is passed to the `send_message` socket event
- Socket handler validates the referenced message belongs to the same channel; populates full `replyTo` shape (`id`, `content`, `senderName`) in the emitted payload
- Reply appears as a new message in the main feed with a quote block above the bubble: accent vertical bar + **sender name** + up to 100-char content snippet

### Forward
- ↗ Forward opens `ForwardModal` — a channel picker listing all channels the user can post to
- Selecting a channel and confirming posts the message content to that channel via REST (`POST /api/channels/:id/messages`)
- Modal closes on success or on clicking outside / pressing Escape

### Reply in thread (backend)
- Migration `006_threads_replies.sql` adds `reply_to_id` and `thread_root_id` columns to `messages`
- `thread_root_id IS NULL` filter keeps thread replies out of the main channel feed
- `GET /api/messages/:id/thread` — returns thread replies in ascending order
- `POST /api/messages/:id/thread` — inserts a thread reply; emits `new_thread_reply` (for thread panel viewers) and `thread_reply_count_updated` (for main feed counter) via socket

---

## 2026-02-24 — Reply system redesign + real-time bug fixes

### Reply system — Slack/iPhone style
Changed reply UX to match familiar patterns from Slack and iPhone Messages:
- Clicking ↩ Reply on any message opens the reply composer at the bottom of the screen (not inline)
- The sent reply appears as a **new message in the main feed**, not in a nested thread
- Above the reply's own bubble, a compact quote block shows who was being replied to and a snippet of their original message — the original message is not reposted in full
- Removed the inline "Reply in thread" sub-feed from bubbles; thread backend remains intact

### Action bar repositioned (above bubble)
- Action bar now floats **above** the bubble (`position: absolute; top: -30px; right: 0`) rather than below it
- Removed `paddingBottom: 34` that was previously reserved to prevent layout shift — no longer needed with the above-bubble placement
- Reaction pills now sit directly below the bubble with only a 2px gap (previously appeared 1–2 lines too low due to the reserved padding)

### Real-time bug: `req.io` always undefined
- **Root cause:** `req.io` middleware was registered at line 98 of `server/index.js`, after the API routes were mounted at line 75. Express executes middleware in registration order, so every route handler ran before `req.io` was ever set — it was always `undefined`. The optional-chaining `req.io?.emit(...)` silently swallowed the call, meaning no reaction or edit/delete socket events were ever broadcast from REST handlers.
- **Fix:** Declared `let _io = null` before the routes. The middleware (`req.io = _io`) is now registered before the routes. After Socket.io initialises, `_io = io` is assigned. The closure captures `_io` by reference, so all subsequent requests resolve to the real `io` instance.
- **Files:** `server/index.js`

### Real-time: reactions now update live
- `toggleReaction` in `useMessages.js` previously only emitted a REST call and relied entirely on the `reaction_updated` socket event for the state update (which never fired due to the `req.io` bug above)
- **Fix (belt-and-suspenders):** `toggleReaction` now also updates local state immediately from the REST response — the returned `reactions` array (fully recalculated server-side) is applied via `setMessages` without waiting for a socket event. The socket event still fires for other users in the channel.

### Real-time: sent messages now appear immediately
- `sendMessage` previously emitted the `send_message` socket event and waited for the `new_message` broadcast to bounce back from the server before the message appeared in the sender's feed
- **Fix:** `sendMessage` now optimistically adds the message to state from the socket ack payload the moment the server acknowledges it. The `onNew` socket handler deduplicates by message ID so the message is not added twice if the broadcast also arrives.
- **Files:** `client/src/hooks/useMessages.js`

---

## 2026-02-24 — Emoji reaction redesign: accumulate counts, names, right-click remove

### Reaction behavior (left-click adds, right-click removes)
Previous behavior toggled a reaction on/off with a single click. The new model matches how reactions work in Slack and social apps:

- **Left-click on the emoji picker** — always *adds* a reaction. If the user has already reacted with that emoji the button is visually highlighted in the picker, but clicking it again is a no-op (server skips duplicate insert).
- **Left-click on a reaction pill** — adds the same emoji if the user has not yet reacted; does nothing if they already have.
- **Right-click on a reaction pill** — shows a browser confirm dialog ("Remove your 👍 reaction?"). If confirmed, removes only the current user's own reaction and decrements the count by 1. Other users' reactions are unaffected.

### Cumulative count + who reacted
- Each reaction pill shows `{emoji} {count}`, where count reflects every user who has clicked that emoji on the message.
- Hovering a pill shows a tooltip listing the names of everyone who reacted (e.g. "Ian, Alex").
- The emoji picker highlights emojis the current user has already added (accent background).

### Backend changes
- **`POST /messages/:id/reactions`** — changed from toggle to add-only; skips the insert if the `(user_id, message_id, emoji)` row already exists.
- **`DELETE /messages/:id/reactions/:emoji`** — new endpoint; removes only the requesting user's reaction for that emoji; emits `reaction_updated` via Socket.io so all clients update in real time.
- **`getReactions()`** — now JOINs the `users` table to aggregate encrypted name values alongside `user_id` and `count`. Names are decrypted server-side before being returned.
- **`formatMessage()`** — updated to decrypt `nameEncrypteds` returned from the inline SQL reactions subquery, producing a `names[]` array on every message's reactions.

### Frontend changes
- `api/index.js`: `toggleReaction` replaced by `addReaction` (POST) and `removeReaction` (DELETE with `encodeURIComponent` on the emoji).
- `useMessages.js`: both functions update local state immediately from the REST response (belt-and-suspenders alongside the socket event).
- `MessageBubble.jsx`: pill click/right-click behavior as described above; emoji picker highlights already-reacted emojis; reaction pill tooltip shows comma-joined names.
- `MessageList.jsx` + `ChatWindow.jsx`: `onRemoveReact` prop threaded through to bubbles.

---

## Testing

This prototype is planned for testing with students and instructors at the **University of Texas San Antonio (UTSA)**.
