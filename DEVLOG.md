# LMS Chat — Dev Log

---

## 2026-02-22 — First successful boot + dev login

### What we did
Got the scaffolded app running end-to-end for the first time and added a dev login flow so the UI can be worked on without a real Canvas instance.

### Startup bugs fixed

**Redis client library** — The Redis client library we were using requires a different initialization pattern in modern JavaScript modules than what was in the scaffold. Fixed the import and removed a manual connection call that the library does not expect (it connects automatically).

**LTI library import** — The LTI library's default export does not expose the setup method. Fixed by importing the named provider class directly.

**LTI + Socket.io conflict** — The LTI library normally controls the underlying HTTP server, making it inaccessible for Socket.io to attach to. Fixed by running the LTI library in serverless mode and manually constructing the HTTP server so Socket.io can share it.

**LTI route interception** — The LTI library was intercepting all incoming requests — including our own API routes — and rejecting them with an authentication error. Fixed by explicitly whitelisting our API path prefix so the LTI layer only guards the LTI launch flow.

**Missing environment file** — The server crashed immediately because no environment file existed. Created one from the example template with freshly generated secrets.

### Dev login
Added a dev-only login flow so the UI can be developed and tested without a live Canvas installation. A simple form lets a developer enter a name and choose a role (student, TA, instructor). On submit, the server creates a realistic fake session backed by the same database tables a real LTI launch would populate. The browser is then redirected into the app as that user.

A subtle bug with session cookies being lost through the development proxy was resolved by having the server return a success response and letting the browser handle the redirect itself, rather than issuing a server-side redirect.

### Docker
Started all required backing services (PostgreSQL, Redis, MongoDB) via Docker Compose. Cleaned up a deprecation warning in the Compose file. Database migrations ran cleanly against a fresh database.

---

## 2026-02-22 — Light theme + message edit/delete

### Light theme
Replaced the default dark theme with a light theme matching Canvas brand guidelines:
- White sidebar, light gray main area
- Accent color: Canvas Orange (#E66000) — used on buttons, active channel highlight, own message bubbles, and avatars
- Dark navy-gray text on light backgrounds for accessibility

### Message edit
- Authors can edit their own messages by clicking a pencil icon that appears on hover
- The message bubble is replaced with an editable text area; Save and Cancel controls appear below
- Enter saves, Escape cancels
- Edited messages are marked with an "(edited)" indicator
- FERPA compliance: the original message content is stored in the audit log before the edit is saved

### Message delete
- Authors can delete their own messages; instructors and TAs can delete any message
- A delete icon appears on hover alongside the edit icon
- Messages are soft-deleted in the database (the record is preserved for the audit trail)
- Edit and delete actions update all connected users in real time via the live messaging layer

---

## 2026-02-23 — Channel creation UI

- A create button appears next to the channels heading in the sidebar, visible to instructors and TAs only
- Clicking it opens a modal where the channel name, optional description, and posting permissions (who can post: TA, students) can be configured. The channel name is automatically formatted into a URL-safe slug with a live preview.
- The modal closes on Escape or clicking outside
- The new channel appears in the sidebar immediately on creation, without a page refresh
- The restriction is enforced on both the client (button hidden from students) and the server (role check on the creation endpoint)

---

## 2026-02-23 — Private Office Hours chat

Each student gets a private message thread with their instructor or TA tied to their office hours request. No other students can see the conversation — access is enforced at both the API and real-time messaging layers, in compliance with FERPA.

- Message history is persisted in the database and loaded when a student or instructor opens the chat panel
- New messages from either party appear in real time without refreshing
- The student's view shows an "Open Chat" toggle on their office hours card; instructors see the same panel when reviewing a request
- Cancelled requests disable the chat panel
- The Office Hours page was redesigned with a light theme, color-coded status badges (pending, scheduled, completed, cancelled), and role-appropriate action controls (Schedule, Decline, Mark Complete for instructors)

---

## 2026-02-23 — Unread counts + Office Hours badge

### Channel unread counts
- Each channel in the sidebar now shows an orange badge with the count of unread messages
- The count updates in real time as new messages arrive in channels the user is not currently viewing
- Opening a channel clears its badge and marks the channel as read, persisted to the database so it survives a page refresh

### Office Hours pending badge
- Instructors and TAs see a badge on the Office Hours navigation button showing the number of pending requests
- The badge updates instantly when a new request is submitted by a student, without requiring a page refresh
- Falls back to a periodic refresh as a safety net

### Demo data
- The dev login seeder now always ensures a sample student account ("Alex Chen") exists with one pending office hours request, so instructors always see the badge and pending request flow on first login

---

## 2026-02-23 — Right-click context menus + channel leave/delete

### Context menu component
A reusable right-click context menu was built that renders outside the normal page flow so it is never clipped by scrolling containers. It clamps to the viewport edges, dismisses on outside click or Escape, and supports a danger (red) style for destructive actions.

### Message right-click
Right-clicking a message shows Edit and Delete options (where the user has permission), as an alternative to the hover buttons.

### Channel right-click
Right-clicking a channel in the sidebar shows two options:
- **Leave Channel** — available to all roles. The channel is hidden from that user's sidebar permanently and is excluded from their channel list on every subsequent load.
- **Delete Channel** — available to instructors and TAs only, shown in red. Permanently deletes the channel and all its messages. The channel disappears from everyone's sidebar in real time via a socket event.

### Sidebar UX
- Office Hours was removed from the default channel list — it is a dedicated page, not a chat channel
- The sidebar footer was reorganized: user info on top, a full-width Office Hours button below with the pending badge for instructors/TAs

---

## 2026-02-23 — Bug fixes: server crash + stale UI on channel delete

### Server crash on channel delete
- **Root cause:** When a user opened a channel and it was simultaneously deleted by an instructor, the next "mark as read" request from the client tried to write a database record referencing the now-deleted channel. This caused a foreign key violation that crashed the server process.
- **Fix:** The mark-as-read operation now silently does nothing if the referenced channel no longer exists, rather than writing unconditionally.

### Channel not removed from sidebar after delete
- **Root cause:** The client waited for confirmation from the server's socket event before removing the channel from the sidebar. If the server had already crashed in the window above, that event never arrived and the UI stayed stale.
- **Fix:** The channel is now removed from the local UI state immediately when the delete action is triggered, before the server responds. The UI is correct regardless of server timing.

---

## 2026-02-23 — Mobile-responsive layout

- Below 640px, the sidebar collapses off-screen and slides in as an overlay when opened
- A semi-transparent backdrop behind the open sidebar dismisses it on tap
- Selecting a channel automatically closes the sidebar
- An empty-state screen on mobile shows a "Browse channels" prompt
- The chat window gains a mobile top bar showing the current channel name and a hamburger button to open the sidebar
- Bottom padding on the message input accounts for the home indicator on phones without a physical home button

---

## 2026-02-23 — Dark mode toggle

- A toggle button in the sidebar footer switches between light and dark themes
- The preference is saved in the browser and restored on next load; if no preference has been set, the user's operating system setting is respected
- All colors across every component use theme variables so the entire interface switches cleanly — no hardcoded color values remain (except semantic status badge colors such as pending/completed, which are self-contained and readable in both modes)

---

## 2026-02-24 — Typing indicators + online presence

### Typing indicators
- When a user is typing in a channel, other users in that channel see an animated three-dot indicator and the text "[Name] is typing…"
- Handles multiple typers gracefully: two names are listed individually; three or more collapse to "Several people are typing…"
- The indicator clears automatically if the user stops typing, even without sending a message
- A fixed-height bar is always reserved for the indicator so the message list does not jump when it appears or disappears

### Online presence
- A green dot appears on the user's own avatar in the sidebar footer while they are connected
- A live online count appears next to the channels heading and in the channel header
- Message bubbles show a green dot on the sender's avatar if that user is currently online

---

## 2026-02-24 — Emoji reactions, reply, forward, threads

### Emoji reactions
- Six reactions available: 👍 👎 ❤️ 😂 ✅ ❓
- Hovering a message reveals an action bar; clicking the reaction button opens the emoji picker
- Selected reactions appear as pills below the message bubble showing the emoji and count
- A pill is highlighted when the current user has reacted with that emoji
- Reactions update in real time for all users in the channel

### Message action bar
- The previous right-click-only context menu for messages was replaced with a persistent hover action bar that appears above the message bubble
- The bar shows: Reactions, Reply, Forward, Edit (author only), Delete (author or instructor/TA)
- Each button shows a tooltip label on hover
- Buttons that the user lacks permission to use are not shown at all

### Reply — Slack/iPhone style
- Clicking Reply opens the reply composer at the bottom of the screen
- The reply is sent as a new message in the main feed — it is not nested inline under the original
- A compact quote block above the reply bubble shows the original sender's name and a snippet of their message

### Forward
- Clicking Forward opens a channel picker
- Confirming copies the message content to the selected channel

### Thread replies (backend infrastructure)
- The database was extended to support thread replies as a distinct message type, separate from the main channel feed
- Thread reply counts are tracked and broadcast in real time so the main feed can eventually show reply counts per message

---

## 2026-02-24 — Real-time bug fixes

### Root bug — socket events never broadcast from REST handlers
- **Root cause:** The middleware responsible for giving route handlers access to the real-time messaging layer was registered in the wrong order in the server startup sequence. Because the web framework processes middleware in the order it is registered, every route handler that tried to send a socket event was working with an uninitialized reference. The failure was silent — the code appeared to try to emit but nothing was sent.
- **Fix:** Restructured the server startup sequence so the messaging middleware is registered before the route handlers. A deferred reference pattern ensures the middleware always resolves to the live Socket.io instance once it is initialized, regardless of when a request arrives.
- **Impact:** This was the root cause of reactions, edits, and deletes not broadcasting to other users in real time.

### Reactions now update live
- Previously, a reaction click updated the database but no socket event was ever broadcast (due to the bug above), so other users had to refresh to see changes.
- Fix applied as part of the root bug resolution above. As an additional safeguard, the reacting user's own UI now updates immediately from the server's response rather than waiting for the socket bounce-back.

### Sent messages now appear immediately for the sender
- Previously, a sent message did not appear in the sender's own feed until the server broadcast it back via the socket — adding a perceptible delay.
- Fix: the server now acknowledges the message immediately in its response, and the client adds it to the feed from that acknowledgement. Duplicate-prevention logic ensures the message is not added a second time when the broadcast also arrives.

---

## 2026-02-24 — Emoji reaction redesign: accumulate counts, names, right-click remove

### New behavior
The previous model toggled a reaction off if you clicked it again. The new model matches how reactions work in Slack and most social platforms:

- **Left-click the emoji picker** — always adds your reaction. The picker highlights emojis you have already used on this message. Clicking an already-reacted emoji again does nothing.
- **Left-click a reaction pill** — adds that reaction if you have not yet reacted with it; does nothing if you already have.
- **Right-click a reaction pill** — prompts you to remove your own reaction. If confirmed, your reaction is removed and the count decreases by one. Other users' reactions are not affected.
- **Hover a reaction pill** — a tooltip lists the names of everyone who reacted with that emoji (e.g. "Ian, Alex").

### What changed on the server
- The add-reaction endpoint now only adds; it will not create a duplicate if the same user reacts twice with the same emoji.
- A new remove-reaction endpoint was added that deletes only the requesting user's reaction for a given emoji and broadcasts the updated reaction state to all users in real time.
- The reaction data returned to clients was extended to include the display names of everyone who reacted, so the tooltip can be populated without a second request.

### What changed on the client
- The reaction toggle function was replaced with separate add and remove actions
- Both actions update the local message state immediately from the server response, so the UI feels instant
- The message bubble, reaction list, and channel components were all updated to wire the remove action through correctly

---

## 2026-02-24 — Action bar improvements + quick dev login

### Larger action icons
All icons in the hover action bar were increased in size for easier clicking and better visibility on all screen sizes.

### Direct reaction buttons
Three reactions are now available as one-click buttons directly in the action bar, without opening a picker:
- 🙌 Job well done
- 👍 Thumbs up
- 👎 Thumbs down

A thin visual divider separates the reaction buttons from the other action buttons. Buttons already reacted to are highlighted in the accent color.

### Additional reactions picker — opens left
The ☺ button opens a secondary picker for less common reactions (❤️ 😂 😮 😢 ✅ ❓). Previously this picker opened upward and was obscured by the channel header when hovering messages near the top of the window. It now opens to the left of the action bar, staying fully visible regardless of where the message sits in the scroll window.

### Updated icons
The edit button now shows a pencil emoji and the delete button shows a trashcan emoji, replacing the previous text symbols.

### Quick dev login presets
The developer login screen now shows two one-click buttons at the top: "Ian Burres — Instructor" and "Alex Chen — Student". Clicking either logs in immediately without filling out the form. The manual name/role form remains available below for other test users. This makes switching between accounts straightforward when testing features that require two users — such as verifying that reaction counts and names appear correctly from a second user's perspective.

---

## Testing

This prototype is planned for testing with students and instructors at the **University of Texas San Antonio (UTSA)**.
