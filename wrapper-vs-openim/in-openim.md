# What to Implement Inside OpenIM (or Leave to OpenIM)

This document lists what is **best implemented inside OpenIM** or left to OpenIM’s native behavior. We use OpenIM as the messaging engine and **do not fork it** for our default path; “inside OpenIM” here means “rely on OpenIM’s built-in features” rather than reimplementing them in our wrapper.

If we later add custom logic that truly must live in OpenIM’s codebase (e.g. a missing webhook or API), we would do that via a minimal patch or upstream contribution, not by moving our gateway or auth into OpenIM.

---

## 1. Core Messaging

| Feature | In OpenIM | Notes |
|--------|-----------|--------|
| **Send / receive messages** | ✅ | OpenIM REST + SDK; we proxy REST through our gateway with imToken injection. |
| **Message storage** | ✅ | OpenIM owns message persistence (e.g. MongoDB or configured storage). |
| **Message history / sync** | ✅ | OpenIM APIs for history, sync; we don’t store a duplicate message store. |
| **Conversations** | ✅ | OpenIM conversation model; we migrate off our Postgres conversation table for OpenIM-backed chats. |
| **Read receipts, delivery state** | ✅ | Use OpenIM’s mechanisms where available. |

We do **not** reimplement message pipeline (Mongo + RabbitMQ + Redis) on our side; OpenIM is the single source of truth for messages and conversations it manages.

---

## 2. Groups & Super-Groups

| Feature | In OpenIM | Notes |
|--------|-----------|--------|
| **Create / join / leave group** | ✅ | OpenIM group APIs. |
| **Group membership, roles** | ✅ | OpenIM. |
| **Group metadata** | ✅ | OpenIM. |
| **Channels / super-groups** | ✅ | OpenIM model; we use OpenIM APIs. |

Our backend may **call** OpenIM admin API (e.g. get_groups, dismiss_group) but does not store or replicate group state for OpenIM groups.

---

## 3. Realtime (WebSocket)

| Feature | In OpenIM | Notes |
|--------|-----------|--------|
| **WebSocket connection** | ✅ | Clients connect **directly to OpenIM WS** with imToken (no gateway in WS path). |
| **Online / connection state** | ✅ | OpenIM knows who is connected; we don’t maintain a separate presence layer for message delivery. |
| **Push of new messages** | ✅ | OpenIM pushes over its WS; we don’t run RabbitMQ → Redis → Socket.IO for OpenIM messages. |

We do **not** put our gateway in front of OpenIM’s WebSocket; Nginx routes `/ws` to OpenIM.

---

## 4. Push Notifications (Chat)

| Feature | In OpenIM | Notes |
|--------|-----------|--------|
| **FCM / APNs for chat messages** | ✅ | OpenIM config (e.g. `config/notification.yaml`); OpenIM sends push when user is offline. |
| **Offline message delivery** | ✅ | OpenIM decides “offline” and triggers push; we don’t enqueue chat messages to our own push queue. |

Non-chat push (e.g. OTP, account alerts) stays in our wrapper (see [as-wrapper.md](./as-wrapper.md)).

---

## 5. User Registration, User Info & IM Profile (OpenIM)

OpenIM **does** have user management: it’s not just “an ID.” We use it for IM-facing identity and display.

| Feature | In OpenIM | Notes |
|--------|-----------|--------|
| **User registration** | ✅ | `user_register`: we call at login with userID, nickname, faceURL; OpenIM stores the user. |
| **User info (get_users_info)** | ✅ | OpenIM stores and returns userID, nickname, faceURL, ex, createTime, globalRecvMsgOpt, etc. Clients can get other users’ display info via OpenIM (proxied through our gateway with imToken). |
| **IM-facing profile** | ✅ | Nickname and faceURL (and optional ex) are the profile fields OpenIM uses for chat UI. We sync these to OpenIM at register and when the user updates profile (our backend calls OpenIM update if available). |
| **imToken minting** | ✅ | OpenIM mints imToken; we store it in Redis and pass it to clients for WS; gateway injects it for REST. |

We do **not** duplicate OpenIM’s user info in our DB for IM display; we use OpenIM as the source of truth for nickname/faceURL in chat. Our wrapper keeps **identity and auth** (OTP, session, devices) and optional **extended profile** (see [as-wrapper.md](./as-wrapper.md)).

---

## 6. When We Might Add Code Inside OpenIM (Rare)

Consider **contributing or small in-tree changes** only when:

- OpenIM has **no API or webhook** for something we need (e.g. a new callback, or a new admin operation).
- The change is **local and small** (e.g. one new webhook type) so upstream merge or minimal fork is feasible.

We do **not** move gateway, auth, or business logic into OpenIM by default; see [as-wrapper.md](./as-wrapper.md).

---

*Back to [README](./README.md) | [As wrapper](./as-wrapper.md) | [Section index](../sections/00-INDEX.md)*
