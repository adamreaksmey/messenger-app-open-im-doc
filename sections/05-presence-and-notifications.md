# 05 — Presence & Notifications (Current vs OpenIM)

This section maps **current** presence and push notifications to **OpenIM** options so we can reuse OpenIM where possible and keep a minimal custom layer only if needed.

---

## Current Presence

- **Where:** WebSocket service (NestJS). On Socket connect: add `socketId` to Redis `presence:user:${userId}` and `presence:socket:${socketId}` → userId. On disconnect: remove. TTL 1 hour.
- **Who uses it:** Chat service. Before publishing a “new message” to Redis for realtime, it checks **UserPresenceService** (Redis): if recipient is online → publish to Redis (websocket subscriber then emits to Socket.IO); else → enqueue to push queue.
- **Result:** “Online” = at least one Socket.IO connection for that user. No OpenIM today.

---

## OpenIM Presence / Connection Model

- **Connections:** Clients connect to **OpenIM WebSocket** with imToken. OpenIM knows which user is connected and can infer “online” from its own connection state.
- **Presence:** OpenIM may expose presence or “user online” via API or callbacks. If we need “is user X online” in our backend (e.g. for analytics or custom logic), we use OpenIM’s API or webhooks rather than our own Redis presence.
- **Realtime delivery:** OpenIM pushes messages over its WebSocket. We **no longer** need our “if online → Redis, else push” branch for **core** message delivery; OpenIM handles that.

---

## Best / Most Efficient Mapping

| Current | OpenIM migration |
|--------|-------------------|
| **Redis presence (presence:user:*, presence:socket:*)** | **Remove** for the purpose of “deliver message via WS vs push.” OpenIM’s connection state is the source of truth. Optional: keep a **minimal** presence only if we have non-OpenIM features that need “is user online” (e.g. custom dashboard). |
| **Chat service: IsUserOnline before Redis publish** | **Remove.** We no longer publish new messages to our Redis for delivery; OpenIM delivers. |
| **WebSocket service: presence add/remove on connect/disconnect** | **Remove** for core chat. If we keep Socket.IO at all (e.g. typing only), we could keep a small presence there for “who’s in this conversation and connected to our WS” — but typically typing can be best-effort without full presence. |
| **Typing indicators** | OpenIM may support typing. If yes, use OpenIM. If we need custom typing (e.g. our own channel), we can keep **one** Redis channel + Socket.IO for typing only (minimal surface). |

---

## Current Push Notifications

- **Flow:** Chat consumer, for each offline recipient, enqueues to RabbitMQ `notification.push`. A separate worker (or external service) would consume and send FCM/APNs. Payload includes user IDs and message payload.
- **No OpenIM today:** Our backend decides “offline” and enqueues push.

---

## OpenIM Push

- OpenIM can be configured for **APNs (iOS)** and **FCM (Android)**. Push credentials live in OpenIM config (e.g. `config/notification.yaml`). OpenIM sends push when the user is offline (no OpenIM WS connected) for messages it delivers.
- **Best approach:** Rely on **OpenIM push** for chat message notifications. Then we **remove** the “offline → enqueue to notification.push” path from our chat consumer (and optionally the whole consumer for message delivery).
- **Our own push:** If we still need **non-chat** push (e.g. OTP, account alerts, marketing), keep a small push pipeline: store device push tokens in our backend (e.g. at login), and a separate job/service that sends via FCM/APNs. That is independent of OpenIM.

---

## Summary

- **Presence:** No need to maintain our own Redis presence for message delivery. OpenIM’s connection state is enough. Optional: minimal presence only for non-OpenIM features.
- **Realtime message delivery:** Handled by OpenIM WebSocket; remove RabbitMQ → Redis → Socket.IO for messages.
- **Push for chat:** Use OpenIM’s push; remove our “offline → notification.push” for chat messages.
- **Push for other use cases:** Keep a thin “register push token” + “send push” in our backend, separate from OpenIM.

See [implementation-doc.md](../implementation-doc.md) §9 for OpenIM push configuration and [03-messaging-and-realtime.md](./03-messaging-and-realtime.md) for the full message pipeline removal.
