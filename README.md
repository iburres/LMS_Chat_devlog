# LMS Chat — Dev Log

Public development journal for **LMS Chat**, a student-instructor chat service built for Canvas and other LTI-compatible learning management systems.

> The app itself is private. This devlog is public — building in the open.

---

## What is LMS Chat?

A channel-based real-time chat tool that lives inside Canvas as an LTI 1.3 app. Students can talk to each other, ask instructors questions, request office hours, and search past answers — all scoped to their specific course.

**Key properties:**
- LTI v1.3 + OAuth2 — launches directly from within Canvas
- FERPA compliant — student data encrypted at rest, full audit trail
- Course-isolated — students in one course cannot see another course's chats
- Real-time — live messaging, presence, and typing indicators via Socket.io and Redis

---

## Stack

| Layer | Tech |
|---|---|
| Backend | Node.js + Express |
| LTI | ltijs (LTI v1.3) |
| Real-time | Socket.io + Redis |
| Database | PostgreSQL + Redis |
| Frontend | React + Vite |

---

## Dev Log

---

### 2026-02-22 — Entry 001: Project scaffold

Built the full project scaffold from scratch — 41 files across server and client.

**Server:** LTI v1.3 launch flow, PostgreSQL schema with course isolation at the data layer, AES-256-GCM encryption of all student PII before it touches the database, FERPA audit logging on every read and write, role-based access control (instructor / TA / student), REST API covering channels, paginated message history, office hours, and full-text search. Full-text search is handled entirely by PostgreSQL with no external search engine. Socket.io with a Redis adapter for horizontal scaling. Typing indicators and presence tracking via Redis.

**Client:** Channel sidebar, live chat window with infinite scroll, search overlay scoped to the current course, office hours request and scheduling UI, development proxy routing API and socket traffic to the local server.

---

### 2026-02-22 — Entry 002: First successful boot + dev login

Got the app running end-to-end for the first time.

**Startup bugs fixed:**
- The Redis client library required a different initialization pattern in ES modules than what was scaffolded — the import style and an unnecessary manual connection call were both corrected.
- The LTI library's main export does not expose the setup method; the named class had to be imported directly.
- The LTI library normally controls the HTTP server, making it inaccessible for Socket.io to attach to. Running it in serverless mode and constructing the server manually resolved the conflict.
- The LTI library was intercepting all requests — including our own API routes — and rejecting them with an auth error. The API path prefix was whitelisted so the LTI layer only guards the LTI launch flow.
- The server crashed immediately on startup because no environment file existed. Created one from the template with freshly generated secrets.

**Dev login:** A developer form lets you enter a name and pick a role; the server creates a fully realistic session backed by the real database so the UI can be worked on without a live Canvas installation.

---

### 2026-02-22 — Entry 003: Light theme + message edit/delete

**Light theme** designed to match Canvas brand guidelines — white backgrounds, Canvas Orange (#E66000) as the accent color for buttons, active states, and own-message bubbles.

**Message edit:** Authors can edit their own messages inline. The original content is stored in the audit log before the change is saved (FERPA requirement). Edits broadcast to all connected users in real time and are marked with an "(edited)" indicator.

**Message delete:** Authors can delete their own messages; instructors and TAs can delete any message. Records are soft-deleted so the audit trail is preserved. Deletes broadcast in real time.

---

### 2026-02-23 — Entry 004: Channel creation UI

Instructors and TAs see a create button next to the channels heading. A modal lets them name the channel (with a live slug preview), add a description, and choose who can post. The restriction is enforced on both the client and the server — students cannot create channels. The new channel appears in the sidebar immediately without a page refresh.

---

### 2026-02-23 — Entry 005: Private Office Hours chat

Each student gets a private message thread with their instructor or TA tied to their specific office hours request. No other students can see the conversation — access is enforced at both the API and real-time layers, in compliance with FERPA.

Messages persist across sessions, load on open, and arrive in real time. Students see an "Open Chat" toggle on their request card; instructors see the same panel when reviewing requests. The Office Hours page was redesigned with a light theme, color-coded status badges, and role-appropriate action controls.

---

### 2026-02-23 — Entry 006: Unread counts + Office Hours badge

**Channel unread counts:** Each channel in the sidebar shows an orange badge with the number of unread messages. Counts update live as messages arrive in channels the user is not currently viewing. Opening a channel clears the badge and persists the read position to the database so it survives a page refresh.

**Office Hours badge:** Instructors and TAs see a live badge on the Office Hours button showing the number of pending requests. The badge updates instantly when a student submits a new request, with a periodic fallback refresh.

**Demo seed:** The dev login flow always ensures a sample student and pending request exist so the badge and pending-request workflow are immediately visible on first login.

---

### 2026-02-23 — Entry 007: Right-click context menus + channel leave/delete

A reusable context menu component was built — it renders outside scroll containers so it is never clipped, auto-clamps to the viewport, and dismisses on outside click or Escape.

**Messages:** Right-clicking shows Edit / Delete based on the user's permissions.

**Channels:** Right-clicking a channel in the sidebar shows:
- **Leave Channel** (all roles) — hides the channel from the user's sidebar permanently, persisted across refreshes.
- **Delete Channel** (instructors/TAs, shown in red) — permanently deletes the channel and all its messages; disappears from everyone's sidebar in real time.

The sidebar footer was also reorganized: user info on top, a full-width Office Hours button below with the pending badge.

---

### 2026-02-23 — Entry 008: Bug fixes — server crash + stale sidebar

**Server crash on channel delete:** If a channel was deleted while a user still had it open, the next "mark as read" request tried to write a database record referencing the now-deleted channel. The resulting constraint violation crashed the server. Fixed so the operation silently does nothing if the channel no longer exists.

**Stale sidebar after delete:** The client was waiting for a socket confirmation before removing a deleted channel from the sidebar. If the server had already crashed in the window above, that event never came and the UI stayed stale. Fixed by removing the channel from the local UI immediately when the delete is triggered, regardless of server timing.

---

### 2026-02-23 — Entry 009: Mobile-responsive layout

Below 640px the sidebar collapses off-screen and slides in as a full-height overlay. A backdrop dismisses it on tap. Selecting a channel closes the sidebar automatically. The chat window gains a top bar on mobile showing the channel name and a button to open the sidebar. The message input accounts for the home indicator on phones without a physical home button.

---

### 2026-02-23 — Entry 010: Dark mode toggle

A toggle in the sidebar footer switches between light and dark themes. The preference is saved in the browser and respected on next load; if no preference exists, the operating system setting is used. All colors across every component use theme variables — the entire interface switches cleanly with no leftover hardcoded values.

---

### 2026-02-24 — Entry 011: Typing indicators + online presence

**Typing indicators:** When a user types in a channel, other members see an animated three-dot indicator and the name of who is typing. Multiple typers are listed by name up to two; three or more collapse to "Several people are typing…". The indicator clears automatically when typing stops. A reserved fixed-height bar prevents the message list from jumping when the indicator appears.

**Online presence:** A green dot on the user's own avatar confirms they are connected. A live count of online users appears in the channel header and next to the channels list. Message bubbles show a green dot on the sender's avatar if they are currently online.

---

### 2026-02-24 — Entry 012: Emoji reactions, reply, forward, threads

**Emoji reactions:** Six reactions are available (👍 👎 ❤️ 😂 ✅ ❓). A hover action bar on each message opens the emoji picker. Reactions appear as pills below the bubble with a count. The current user's reactions are highlighted. Reactions update live for everyone in the channel.

**Hover action bar:** The previous right-click context menu for messages was replaced with a clean action bar that appears above the bubble on hover: Reactions, Reply, Forward, Edit, Delete. Buttons the user lacks permission to use are hidden entirely. Each button shows a tooltip label on hover.

**Reply — Slack/iPhone style:** Clicking Reply opens the composer at the bottom of the screen. The reply appears as a new message in the main feed with a compact quote block above it showing the original sender's name and a snippet of their message.

**Forward:** Opens a channel picker; confirming copies the message to the chosen channel.

**Thread infrastructure:** The database was extended to support threaded replies as a distinct type separate from the main channel feed, with reply counts tracked and broadcast in real time.

---

### 2026-02-24 — Entry 013: Real-time bug fixes

**Root bug — socket events never broadcast from REST handlers:**
The middleware responsible for giving route handlers access to the real-time layer was registered in the wrong order during server startup. The web framework processes middleware in registration order, so every route handler was working with an uninitialized reference when it tried to send a socket event. The calls failed silently — the code appeared to run but nothing was ever sent. This was the root cause of reactions, edits, and deletes not appearing in real time for other users. Fixed by restructuring startup so the middleware is registered before the routes.

**Reactions update live:** Now resolved by the root fix above. As a belt-and-suspenders measure, the reacting user's own view also updates immediately from the server's response rather than waiting for the socket broadcast.

**Messages appear immediately for the sender:** Previously a sent message only appeared in the sender's own feed after the server broadcast it back — adding a noticeable delay. Fixed so the message is added from the server's immediate acknowledgement. Duplicate-prevention logic handles the case where the broadcast also arrives shortly after.

---

### 2026-02-24 — Entry 014: Emoji reaction redesign — accumulate counts, names, right-click remove

The previous model toggled a reaction off on a second click. The new model matches Slack and most social platforms:

- **Left-click the emoji picker** — always adds your reaction. The picker highlights emojis you have already used. Clicking one you have already added does nothing.
- **Left-click a reaction pill** — adds that reaction if you have not yet reacted; does nothing if you have.
- **Right-click a reaction pill** — prompts to remove your own reaction. If confirmed, your reaction is removed and the count drops by one. Other users' reactions are not affected.
- **Hover a reaction pill** — a tooltip shows the names of everyone who reacted with that emoji.

The server was updated to support add-only behavior, a new remove-my-reaction action, and to include reactor names in the data returned to clients. The client was updated to use separate add and remove actions that both update the local state immediately so the UI feels instant.

---

### 2026-02-24 — Entry 015: Action bar improvements + quick dev login

**Larger action icons** — all icons in the hover action bar were increased in size for easier clicking and better visibility.

**Direct reaction buttons in the bar** — three reactions are now one click away directly in the action bar without opening a picker:
- 🙌 Job well done
- 👍 Thumbs up
- 👎 Thumbs down

A thin divider separates the reaction buttons from the action buttons (Reply, Forward, Edit, Delete).

**Additional reactions picker repositioned** — the ☺ button now opens a secondary picker for less common reactions (❤️ 😂 😮 😢 ✅ ❓). This picker opens to the **left** of the action bar rather than above it, so it is never blocked by the channel header when hovering messages near the top of the screen.

**Updated icons** — the edit button now shows a pencil emoji and the delete button shows a trashcan emoji.

**Quick dev login** — the developer login screen now has one-click preset buttons to log in immediately as "Ian Burres — Instructor" or "Alex Chen — Student", making it easy to switch between accounts for testing features like reactions that require two different users.

---

## Testing

This prototype is planned for testing with students and instructors at the **University of Texas San Antonio (UTSA)**.

---

*Built with [Claude Code](https://claude.ai/claude-code)*
