# 01 — Current Architecture Summary

This document summarizes the **current** chat application so we have a clear baseline for the OpenIM migration. It does not describe OpenIM; see [implementation-doc.md](../implementation-doc.md) and the other section docs for target design and mapping.

---

## High-Level Components

| Component | Tech | Role |
|-----------|------|------|
| **messenger-gateway** | Go/Gin | Single entry point. Route by path prefix; enforce auth (Admin JWT or User HMAC); proxy to backend services. |
| **messenger-user-service** | Go | User auth, device registration, sessions (Redis/Postgres), HMAC session issuance, profile, friends, link requests. |
| **messenger-chat-service** | Go/Gin | Messages (MongoDB), conversations (PostgreSQL), chat groups (PostgreSQL). REST API for messages/conversations/groups. Publishes to RabbitMQ for realtime fan-out. |
| **messenger-websocket** | NestJS + Socket.IO | Real-time delivery. Clients connect with `userId` + `sessionId`. Subscribes to Redis; forwards published payloads to Socket.IO rooms per user. |

Clients call the **gateway** only (e.g. `http://localhost:8000/user-service/...`, `.../chat-service/...`, `.../admin-service/...`). The gateway validates auth and forwards to the appropriate backend.

---

## Gateway (messenger-gateway)

- **Route prefixes:** `/admin-service`, `/user-service`, `/chat-service`.
- **Auth:**
  - **Admin:** JWT in `Authorization: Bearer <token>`. Verified by gateway; backend receives `X-Gateway-Admin-*` headers.
  - **User/Chat:** HMAC. Headers: `Authorization: Session <sessionId>`, `X-Signature`, `X-Timestamp`, `X-Nonce`. Gateway resolves session from Redis (or Postgres fallback), gets `serverHMACKey`, verifies signature on path/body.
- **Session store:** Redis primary; Postgres fallback. Session data includes `userId`, `deviceId`, `serverHMACKey`. TTL slid on each request.

See [02-gateway-and-auth.md](./02-gateway-and-auth.md) for how this maps to OpenIM (imToken, WebSocket exception).

---

## User Service (messenger-user-service)

- Device registration, OTP/auth flows, session creation.
- Stores sessions and device HMAC keys; gateway uses same Redis/Postgres to validate HMAC.
- User profile, friends, settings, link request/approve/cancel.
- Admin JWT issued here (or by admin service); gateway verifies with same secret.

---

## Chat Service (messenger-chat-service)

### Data

- **Messages:** MongoDB. Fields include `conversation_id`, `sender_id`, `message_type`, `content`, `media_ids`, `mentions`, `sequence_number`, `reply_to_message_id`, `edited_at`, `is_deleted`, etc.
- **Conversations:** PostgreSQL. `conversations` table: type (`direct` | `group`), `user1_id`/`user2_id` (direct) or `group_id` (group), last message metadata, unread counts, pinned/archived per user.
- **Chat groups:** PostgreSQL. `chat_groups`, `group_members`, `group_invites`, reports, etc. Rich group lifecycle: create, invite, invite links, join by code, leave, promote, mute, pin, favorite, archive.

### Message Flow

1. Client sends message via gateway → chat service (e.g. `POST /api/v1/messages/direct` or `.../messages/group`).
2. Chat service: generate sequence number, write message to MongoDB, update conversation last-message in Postgres.
3. Publish **MessageSentEvent** to RabbitMQ queue `chat.message.notify` (optional sharding by `conversation_id`). Ordered publisher keeps per-conversation order.
4. Consumer (in chat service) processes event: resolve conversation type and recipients (direct: one recipient; group: batch member IDs). For each recipient:
   - If **online** (Redis presence): publish to Redis channel `REDIS_CHANNEL_PREFIX:CHAT_NOTIFICATION` with `userId`, `channel`, `payload` (full message).
   - If **offline**: enqueue to `notification.push` for push (FCM/APNs) later.

### Presence

- Chat service uses **UserPresenceService** (Redis): track which user IDs are online. Used to decide “publish to Redis” vs “enqueue push”.
- Actual “who is online” is determined by the **websocket** service (see below).

### REST API (summary)

- Messages: create (direct/group), read, get by conversation (with cursor), get by sender, update content, soft/hard delete.
- Conversations: get by ID, get messages, count, latest message.
- Chat groups: full CRUD, invite, invite-link, join by code, members, leave, mute, pin, favorite, archive, reports.
- Admin: conversations/groups/members, reports, broadcast groups, import users, etc.

---

## WebSocket Service (messenger-websocket)

- **Socket.IO** namespace: `/ws-app`. Transports: websocket.
- **Auth:** Client sends `userId` and `sessionId` in handshake `auth`. No server-side session verification against user-service in the snippet; in production this should validate session.
- **Rooms:** Each client joins `user:${userId}`. Notifications are emitted to `user:${userId}` (or multiple user rooms).
- **Presence:** **PresenceService** (Redis): on connect, add `socketId` to `presence:user:${userId}` and `presence:socket:${socketId}` → `userId`. On disconnect, remove. TTL 1 hour. So “online” = at least one socket in `presence:user:${userId}`.
- **Redis subscriber:** Subscribes to channels (e.g. `REDIS_CHANNEL_PREFIX:CHAT_NOTIFICATION`, `TYPING_MESSAGE`). On message: payload includes `userId` (and optional `channel`, `payload`). Gateway calls `notifyUserId(userId, channel, payload)` or `notifyAllUser(channel, payload)` to emit to Socket.IO clients.

So **realtime path** today: Chat service → RabbitMQ → chat consumer → Redis PUBLISH → websocket service (Redis SUB) → Socket.IO → client.

---

## Data Flow Summary

```
Client
  → Gateway (HMAC or JWT)
    → User service: auth, profile, friends
    → Chat service: messages, conversations, chat groups (REST)
  → Chat service writes message to MongoDB + Postgres
  → Chat service publishes to RabbitMQ (chat.message.notify)
  → Chat consumer: for each recipient, if online → Redis PUBLISH; else → push queue
  → WebSocket service: Redis SUB → Socket.IO rooms (user:userId)
  → Client receives realtime event
```

---

## What OpenIM Replaces (Preview)

- **Message storage and routing:** OpenIM stores and routes messages; we stop using MongoDB for core chat messages and the RabbitMQ → Redis → Socket.IO pipeline for delivery.
- **Realtime:** OpenIM provides WebSocket; clients connect directly to OpenIM with imToken. We can decommission or repurpose the NestJS Socket.IO service for non-OpenIM features only (e.g. typing, or custom channels).
- **Conversations / groups:** OpenIM has its own conversation and group model; we either use OpenIM as source of truth or keep a thin sync layer for UI-specific data (see sections 03, 04, 07).

What we **keep** (and possibly extend): Gateway (with imToken resolution and OpenIM proxy), user service (auth, sessions, profiles), admin flows (calling OpenIM admin API from our backend), and optional features (stickers, GIFs, media catalog) as in implementation-doc.
