# 08 — Migration Phases

This document outlines a **phased** implementation plan so we can move to OpenIM without a big-bang rewrite. Each phase has clear inputs and outcomes and builds on the previous one.

---

## Phase 0 — Prep (No App Change)

- **Run OpenIM** locally (e.g. openim-docker). Verify health (e.g. `GET /healthz`).
- **Document** OpenIM REST endpoints we will use (user_register, auth/user_token, msg/send_msg, get_conversation_msgs_by_seq, group/*, auth/force_logout, etc.).
- **Decide** Nginx/routing: which host/port for gateway, which for OpenIM REST, which for OpenIM WS.

**Outcome:** OpenIM running; team aligned on endpoints and routing.

---

## Phase 1 — Gateway + imToken (No Client Change Yet)

- **Backend (user-service or gateway):** After successful login (OTP verify), call OpenIM to register user (if new) and mint imToken; store in Redis `im:token:<userID>`; **do not** return imToken to client yet if we want to test backend-only first.
- **Gateway:** Add OpenIM proxy middleware for a test path (e.g. `/im/user/get_self_user_info`): verify HMAC → get userID → get imToken from Redis → inject token + operationID → proxy to OpenIM REST. Strip our auth headers.
- **Nginx:** Route `/im/*` to gateway; gateway forwards to OpenIM API.

**Outcome:** Gateway can proxy OpenIM REST with imToken injection; imToken is stored at login. Optional: simple test client that calls gateway with HMAC and gets OpenIM user info.

---

## Phase 2 — Client Gets imToken & Connects to OpenIM WS

- **Login response:** Include `imToken` and `wsURL` (e.g. `wss://your-domain.com/ws`) in addition to sessionId.
- **Nginx:** Add location `/ws` → OpenIM WebSocket (proxy_pass to OpenIM WS port); long read/send timeouts.
- **Client:** After login, store imToken securely; connect to `wsURL` with imToken (query or header as OpenIM expects). Verify connection and that OpenIM pushes are received (e.g. send a test message via OpenIM API from backend).

**Outcome:** Client has single path for realtime: OpenIM WebSocket. No change yet to our message send flow (can still use old API for a short time).

---

## Phase 3 — Send / Receive Messages via OpenIM

- **Gateway:** Expose proxy for OpenIM message APIs (send_msg, get history, conversation list, etc.) under `/im/*`. Client uses existing HMAC; gateway injects imToken.
- **Client:** Switch to OpenIM SDK (or direct calls through gateway) for sending and for receiving (OpenIM WS). Stop calling our chat service message endpoints for **new** conversations.
- **Backend:** Stop writing new messages to MongoDB for OpenIM conversations. Optionally: keep old endpoints read-only for legacy conversation IDs during transition.

**Outcome:** New traffic uses OpenIM for messaging; our MongoDB and RabbitMQ/Redis/Socket.IO path are no longer used for new OpenIM chats.

---

## Phase 4 — Conversations & Groups via OpenIM

- Use OpenIM for **creating** single/group chats and for **conversation list** and **group list**. Gateway proxies OpenIM group/conversation APIs.
- **Client:** Use OpenIM conversation/group APIs (via gateway) instead of our chat service conversation/group endpoints.
- **Backend:** Deprecate or remove chat service routes that return conversation/message list from our DB for OpenIM-backed data. Keep admin routes that call OpenIM admin API (see Phase 5).
- **Optional:** Thin tables in Postgres for UI-only flags (pinned, archived) or reports; keyed by OpenIM conversation_id / group_id.

**Outcome:** Conversation and group lifecycle are OpenIM-driven; our Postgres no longer source of truth for OpenIM conversations/groups.

---

## Phase 5 — Admin & Webhooks

- **Backend:** Implement OpenIM admin token usage (get and cache). Add handlers that call OpenIM admin API: list groups, get messages, delete message, dismiss group, kick member, force_logout.
- **Admin UI:** Keep same routes; backend implementation switches to “call OpenIM + our DB” (reports, audit). Ban user: our DB + delete imToken + force_logout.
- **Webhook:** Add `/webhooks/openim`; validate secret; handle afterSendMessage, afterCreateGroup, etc. Use for moderation/analytics only.

**Outcome:** Admin and moderation work via OpenIM and our DB; no dependency on our MongoDB/Postgres for OpenIM message/group data.

---

## Phase 6 — Decommission Old Pipeline

- **Chat service:** Remove (or disable) RabbitMQ publish on send message; remove consumer that publishes to Redis and enqueues push. Remove or repurpose UserPresenceService usage for message delivery.
- **WebSocket service:** Remove Redis subscription for CHAT_NOTIFICATION (and presence if unused). Optionally keep service for typing-only or remove entirely.
- **MongoDB:** No new message writes for OpenIM; optionally migrate or archive old data; eventually drop message collection or keep for legacy read-only.
- **Postgres (chat-service):** Drop or archive conversations/conversation_sequences for OpenIM; keep reports, optional group extension, media catalog, invite links as needed.
- **RabbitMQ:** Remove chat.message.notify and notification.push for chat (or repurpose push for non-chat only).

**Outcome:** No duplicate message path; single source of truth for messaging is OpenIM.

---

## Phase 7 — Polish & Optional Features

- **Stickers / GIFs:** Catalog in our DB; client sends OpenIM custom messages with sticker/GIF refs; optional proxy for GIF search (Tenor) to hide API key.
- **Push:** Rely on OpenIM for chat push; keep our push token registration and sender for non-chat notifications.
- **Media:** OpenIM + MinIO for chat media; our media_files for catalog/analytics if needed.
- **Monitoring:** Health checks for OpenIM, gateway, and webhook; alerting on admin token failure or webhook errors.

**Outcome:** Feature-complete OpenIM-based app with optional extras and clear ownership of data and delivery.

---

## Dependencies Between Phases

```
Phase 0 → Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7
          (gateway   (client    (send/     (conv/     (admin)   (tear     (optional
           + token)   WS)       receive)   groups)              down)     features)
```

Phases 3 and 4 can overlap (e.g. proxy message APIs and conversation APIs in the same sprint). Phase 6 should only start after Phase 5 is in place and clients are fully on OpenIM for messaging and conversations.

---

## Reference

- **Target design:** [implementation-doc.md](../implementation-doc.md)
- **Section index and critical path:** [00-INDEX.md](./00-INDEX.md)
- **Per-concern mapping:** [01-current-architecture.md](./01-current-architecture.md) through [07-data-and-storage.md](./07-data-and-storage.md)
