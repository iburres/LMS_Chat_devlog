# LMS Chat — Dev Log

Public development journal for **LMS Chat**, a student-instructor chat service built for Canvas and other LTI-compatible learning management systems.

> The app itself is private. This devlog is public — building in the open.

---

## What is LMS Chat?

A real-time chat tool that lives inside Canvas as an LTI 1.3 app. Students can talk to each other, ask instructors questions, request office hours, and search past answers — all scoped to their specific course.

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

### 2026-02-22 — Entry 001: Project scaffold

**What was built:**

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
- Discord-inspired dark UI in React + Vite
- Channel sidebar, live chat window, infinite scroll message history
- Search overlay with ranked results scoped to the current course
- Office hours request + scheduling UI (role-aware — instructors see all requests)
- `useSocket` / `useMessages` / `usePresence` hooks
- Vite proxy — `/api` and `/socket.io` forward to Express in dev

**Next up:**
- `npm install` both packages and get the dev environment running
- Register a Canvas developer key and test the LTI launch flow end-to-end
- UI polish pass

---

*Built with [Claude Code](https://claude.ai/claude-code)*
