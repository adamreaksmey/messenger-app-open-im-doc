# 04 — Conversations & Groups (Current vs OpenIM)

This section maps the **current** conversation and chat-group model to **OpenIM** so we can reuse OpenIM’s concepts and only keep or add what’s needed on our side.

---

## Current Model

### Conversations (PostgreSQL)

- **Table:** `conversations`. Types: `direct`, `group`.
- **Direct:** `user1_id`, `user2_id`, last message id/at/preview, unread counts and pinned/archived **per user** (user1/user2).
- **Group:** `group_id`; same last-message and unread/pin/archive semantics can be extended per member elsewhere (e.g. in UI or a side table).
- **Usage:** Conversation is the scope for message list and sequence numbers (`conversation_sequences`). Message API is conversation-scoped (e.g. get messages by `conversation_id`).

### Chat Groups (PostgreSQL)

- **Tables:** `chat_groups`, `group_members`, `group_invites`, reports, etc.
- **Features:** Create group, invite members, invite links, join by code, leave, promote, mute, pin, favorite, archive, report. Admin: list groups, members, dismiss group, update status, handle reports, import users, etc.

---

## OpenIM Model

- **Conversations:** OpenIM has a conversation list per user (single chat = conversation between two users; group chat = conversation for a group). Conversation ID format and semantics are defined by OpenIM. Message history and sequencing are per conversation in OpenIM.
- **Groups:** OpenIM has group create, invite, kick, dismiss, roles, etc. Group ID and member list are managed by OpenIM. We don’t need to duplicate full group state in Postgres for core messaging.

---

## Best / Most Efficient Mapping

| Current | OpenIM migration |
|--------|-------------------|
| **Conversation as scope for messages** | Use **OpenIM conversation** as scope. Client and backend call OpenIM APIs for conversation list and message history. No need to keep our `conversations` table for OpenIM-driven chats **unless** we need extra UI-only state (see below). |
| **Sequence numbers** | OpenIM provides message ordering/seq. Drop our `conversation_sequences` for OpenIM conversations. |
| **Last message / preview in our DB** | Use OpenIM conversation list / last message. Optional: cache or mirror in our DB for admin or heavy queries only. |
| **Unread / pinned / archived per user** | Prefer OpenIM APIs if available. If we need custom semantics, we can keep a **thin** table keyed by (user_id, conversation_id) for our flags only, updated via our backend or webhooks. |
| **Chat groups (members, roles, invite links)** | Use **OpenIM groups** for creation, members, invite, kick, dismiss. Map our “invite link” to OpenIM join mechanism (e.g. group join by link or code if OpenIM supports it). |
| **Group metadata we own (e.g. name, avatar, settings)** | Either store in OpenIM group profile/extension or keep a **small** table in Postgres keyed by OpenIM `group_id` for fields OpenIM doesn’t hold. Avoid duplicating member list and message history. |
| **Reports, broadcast groups, admin actions** | Keep in our backend: admin calls OpenIM admin API (dismiss group, etc.) and our own DB for reports, audit, broadcast list (see 06). |

---

## What to Keep in Our DB (Minimal)

- **Optional conversation-side table:** Only if we need **our** flags (e.g. custom pinned/archived/favorite) or admin analytics. Key: (user_id, openim_conversation_id) or (user_id, openim_group_id for groups). No message storage.
- **Group extension (optional):** If OpenIM doesn’t store everything we need (e.g. custom group type, report state), one table keyed by OpenIM `group_id`: our metadata and status only.
- **Invite links:** If OpenIM has “join by link,” use it. If we need our own link/code table, keep a small table: code/link → OpenIM group_id, expiry, use count. Backend resolves code to group_id and calls OpenIM join or returns join info to client.

---

## Implementation Order

1. Use OpenIM for **creating** single and group conversations; use OpenIM for **conversation list** and **message history**.
2. Expose these via gateway proxy (imToken injection) so clients call our gateway, which forwards to OpenIM.
3. Migrate group lifecycle (create, invite, leave, dismiss) to OpenIM APIs; keep our admin and reporting layers calling OpenIM + our DB as above.
4. Deprecate or drop: our `conversations` and `conversation_sequences` usage for OpenIM chats; our full group member/list storage for OpenIM groups. Remove or simplify chat service routes that return conversation/message data from our Mongo/Postgres in favor of OpenIM-backed responses (or proxy responses).

See [06-admin-and-moderation.md](./06-admin-and-moderation.md) for admin and [07-data-and-storage.md](./07-data-and-storage.md) for a full picture of what stays in our stores.
