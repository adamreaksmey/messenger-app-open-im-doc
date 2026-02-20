# 03 — Messaging & Realtime (Current vs OpenIM)

This section maps the **current** message and realtime pipeline to **OpenIM** so we can reuse what fits and remove or repurpose the rest without rebuilding everything from scratch.

---

## Current Messaging Pipeline

1. **Send message (REST):** Client → Gateway (HMAC) → Chat service.  
   - `POST /api/v1/messages/direct` or `POST /api/v1/messages/group`.
2. **Chat service:**  
   - Resolve or create conversation (Postgres).  
   - Generate sequence number (Postgres `conversation_sequences`).  
   - Write message to **MongoDB** (`messages`).  
   - Update conversation last-message (Postgres).  
   - Publish **MessageSentEvent** to **RabbitMQ** (`chat.message.notify`, optionally sharded by conversation_id).
3. **Consumer (chat service):**  
   - For each recipient: if **online** (Redis presence) → **Redis PUBLISH** to `REDIS_CHANNEL_PREFIX:CHAT_NOTIFICATION` with `userId`, channel, full message payload; else → enqueue to **notification.push**.
4. **WebSocket service (NestJS):**  
   - Subscribes to Redis (CHAT_NOTIFICATION, TYPING_MESSAGE).  
   - On message: emit to Socket.IO room `user:${userId}` (or broadcast).  
5. **Client:** Receives realtime event on Socket.IO.

So: **MongoDB (storage) + Postgres (conversation meta, sequence) + RabbitMQ (fan-out) + Redis (online delivery) + Socket.IO (realtime)**.

---

## OpenIM Model

- **Message storage and history:** OpenIM stores messages (in its own store, e.g. MongoDB used by OpenIM). We do **not** persist core chat messages in our MongoDB.
- **Sending:** Client uses OpenIM SDK; sends via OpenIM API. From our backend we can either:
  - **Option A (recommended):** Client calls our gateway with HMAC; gateway injects imToken and proxies to OpenIM REST (e.g. `/msg/send_msg`). So “send message” becomes a proxy to OpenIM, no custom message write in our chat service.
  - **Option B:** Backend sends on behalf of user (e.g. server-side bot) using OpenIM admin/API with user identity.
- **Realtime:** Client connects to **OpenIM WebSocket** with imToken (direct to OpenIM, not our Socket.IO). OpenIM pushes new messages, read receipts, etc. over that connection. **We do not use RabbitMQ → Redis → Socket.IO for core message delivery.**

---

## Best / Most Efficient Mapping

| Current piece | Action |
|---------------|--------|
| **Chat service: message write to MongoDB** | **Remove** for core single/group chat. OpenIM is the source of truth for messages. |
| **Chat service: conversation_sequences, UpdateConversationMetaDataAfterMessage** | **Remove** for OpenIM-managed conversations. OpenIM has its own conversation/seq model. |
| **Chat service: RabbitMQ publish (MessageSentEvent)** | **Remove** for core message delivery. OpenIM delivers via its WS. |
| **Chat service: chat.message.notify consumer** | **Remove** (or repurpose for non-OpenIM side effects only, e.g. analytics, custom logging). |
| **Chat service: Redis publish for “new message”** | **Remove** for core messages. OpenIM pushes to client over its WS. |
| **WebSocket service: Redis SUB → Socket.IO for CHAT_NOTIFICATION** | **Remove** for core messages. Clients use OpenIM WS. |
| **Chat service REST: create message, get by conversation, etc.** | **Replace** with gateway proxy to OpenIM REST (or SDK on client). Our chat service no longer implements message CRUD for OpenIM conversations. |
| **Read receipts, typing** | OpenIM supports some of this; optional: keep a **minimal** Socket.IO or Redis channel for typing if we need custom behavior (see 05). |

We **keep** (or add):

- **Gateway:** Proxying send/listen to OpenIM REST with imToken injection (see 02).
- **Client:** OpenIM SDK for send/receive and WebSocket connection to OpenIM.
- **Optional:** Our backend still handles **webhooks** from OpenIM (e.g. after message sent) for moderation, analytics, or push — but we don’t use webhooks to “deliver” messages to our own Socket.IO.

---

## What Stays in Our Backend (If Anything)

- **Stickers / GIFs / custom content:** Sent as OpenIM custom messages (content type + JSON body). No need for our MongoDB message storage; optional: we keep **catalog** of sticker packs / GIF config in our DB and serve that via our API (see implementation-doc §6, §8).
- **Media files:** OpenIM can use MinIO (or S3). We can keep a **media catalog** in Postgres for admin/analytics (e.g. `media_files`) without storing message bodies.
- **Mentions, reply_to:** Map to OpenIM message format (custom fields or standard fields as per OpenIM API). No duplicate storage in our Mongo.

---

## Implementation Order (High Level)

1. Introduce OpenIM; clients get imToken and connect to OpenIM WS for realtime.
2. Add gateway proxy for OpenIM message APIs (send_msg, get history, etc.); clients call gateway with HMAC, gateway injects imToken and forwards.
3. Migrate clients to use OpenIM SDK for send/receive; stop calling our chat service message endpoints for core messaging.
4. Turn off or remove: message write to MongoDB for OpenIM chats, RabbitMQ notify flow for message delivery, Redis publish for new_message, Socket.IO CHAT_NOTIFICATION handler for messages.
5. Keep or repurpose: conversation list if we need it from our DB (see 04); otherwise use OpenIM conversation list.

See [04-conversations-and-groups.md](./04-conversations-and-groups.md) for conversation/group mapping and [07-data-and-storage.md](./07-data-and-storage.md) for what remains in our data stores.
