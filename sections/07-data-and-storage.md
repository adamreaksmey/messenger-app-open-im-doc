# 07 — Data & Storage (Current vs OpenIM)

This section clarifies **what data lives where** after migrating to OpenIM so we don’t duplicate storage or leave orphaned systems.

---

## Current Data Stores

| Store | Used for |
|-------|----------|
| **PostgreSQL (user-service)** | Users, devices, sessions, profile, friends, link requests, admin users. |
| **PostgreSQL (chat-service)** | Conversations, conversation_sequences, chat_groups, group_members, group_invites, message_reads, mentions, media_files, reports, etc. |
| **MongoDB (chat-service)** | Messages (full document per message). |
| **Redis (shared)** | Session cache (gateway + user-service), presence (websocket + chat-service), chat notification channel (chat-service publish, websocket subscribe), nonce for HMAC. |
| **RabbitMQ** | chat.message.notify (message fan-out), notification.push (offline push). |

---

## OpenIM-Owned Data (After Migration)

- **Messages:** OpenIM stores and indexes messages (its own MongoDB or configured storage). We **do not** store core chat messages in our MongoDB for OpenIM conversations.
- **Conversations:** OpenIM maintains conversation list and ordering per user. We don’t need our `conversations` and `conversation_sequences` tables for OpenIM-driven chats **unless** we keep a thin layer for UI-only flags (see 04).
- **Groups:** OpenIM stores groups and members. We don’t duplicate full group state; optionally a small “group extension” or “report” table keyed by OpenIM group_id.
- **User registry in OpenIM:** We register users in OpenIM at first login (user_register). OpenIM stores userID, nickname, faceURL, etc. We remain source of truth for “our” user profile (phone, device, etc.) in our Postgres.

---

## What We Keep (Our Backend)

| Store | Keep for |
|-------|----------|
| **PostgreSQL (user-service)** | Users, devices, sessions, profile, friends, link requests, admin users. **Unchanged.** Add nothing for OpenIM message storage. |
| **PostgreSQL (chat-service)** | **Remove or deprecate:** conversations, conversation_sequences (for OpenIM chats). **Keep (or add):** reports (group/message/user), optional conversation-side flags (pinned/archived per user), optional group extension (e.g. custom type, report status), invite-link/code table if we don’t use OpenIM join link only. **Keep:** media_files if we use it as catalog for admin/analytics (media URLs can still be referenced in OpenIM custom messages). |
| **MongoDB (chat-service)** | **Stop writing** new messages for OpenIM conversations. Optionally keep for legacy migration or non-OpenIM use; otherwise decommission for chat. |
| **Redis** | **Keep:** Session cache (gateway + user-service), HMAC nonce. **Add:** `im:token:<userID>` for gateway injection. **Remove:** presence keys and CHAT_NOTIFICATION publish/subscribe for core message delivery (see 03, 05). |
| **RabbitMQ** | **Remove** (or repurpose): chat.message.notify and consumer for message delivery. **Optional:** keep notification.push for **non-chat** push only (e.g. OTP, alerts). |

---

## Summary Table

| Data | Before | After (OpenIM) |
|------|--------|-----------------|
| Chat messages | Our MongoDB | OpenIM |
| Conversation list / seq | Our Postgres | OpenIM |
| Group list / members | Our Postgres | OpenIM (optional small extension in Postgres) |
| User profile (our fields) | Our Postgres | Our Postgres |
| Sessions / devices | Our Postgres + Redis | Unchanged |
| imToken | — | Redis only |
| Reports / audit | Our Postgres | Our Postgres |
| Media catalog | Our Postgres | Our Postgres (optional) |
| Realtime delivery state | Redis + RabbitMQ | OpenIM WS + OpenIM push |

This keeps a **single source of truth** for messaging (OpenIM) while we keep identity, sessions, admin, and optional extensions in our stack. See [08-migration-phases.md](./08-migration-phases.md) for the order to change or decommission each piece.
