# 06 — Admin & Moderation (Current vs OpenIM)

This section maps **current** admin and moderation behavior to the **OpenIM-based** design so we can keep admin capabilities without rebuilding everything from scratch.

---

## Current Admin (Summary)

- **Auth:** Admin JWT. Gateway validates JWT and forwards to backends with `X-Gateway-Admin-*` headers. Chat service has `/api/v1/admin/*` routes (admin JWT or `INTERNAL_SERVICE_TOKEN`).
- **Chat service admin routes (examples):**  
  - Conversations: get messages by conversation.  
  - Chat groups: list privates/groups/channels, get members, dismiss/status, reports, remove member, import users, get non-members, get user’s groups, update settings.  
  - Messages: migrate, admin get messages.  
  - Media: register/update/delete catalog.
- **User service (admin):** User management, ban, etc. (separate from chat).
- **No OpenIM today:** All data comes from our MongoDB/PostgreSQL and our APIs.

---

## OpenIM Admin API

- OpenIM exposes an **admin API** (e.g. port 10008 or same API with admin token). With an **admin token** (minted with OpenIM secret), our backend can:
  - List/get users, groups, conversations.
  - Get/delete messages (e.g. by conversation, by seq).
  - Dismiss groups, kick users, force logout.
  - Other moderation and config operations (see OpenIM REST docs).

So **admin actions that affect messages/groups/users in OpenIM** should be implemented by our backend **calling OpenIM admin API**, not by our DB alone.

---

## Best / Most Efficient Mapping

| Current admin need | OpenIM migration |
|--------------------|------------------|
| **List / search users** | Keep in our backend (user service DB). Optional: sync or query OpenIM user list if we register all users there. |
| **Ban user** | Our backend: update our DB (banned flag), delete `im:token:<userID>` from Redis, call OpenIM **force_logout** for that user. Gateway will then deny new REST calls (no imToken). OpenIM WS will drop connections. |
| **View messages in a conversation** | Backend calls OpenIM admin API (e.g. get messages by conversation ID / seq). No need to read from our MongoDB for OpenIM conversations. |
| **Delete message** | Backend calls OpenIM admin API (e.g. del_msgs). No need to soft-delete in our Mongo for that message. |
| **List groups / channels** | Backend calls OpenIM admin API (e.g. get_groups). Optionally cache or mirror in our DB for reporting only. |
| **Dismiss group / remove member** | Backend calls OpenIM admin API (dismiss_group, kick_group). Keep our own “reports” or “audit” table if we need history of who dismissed what. |
| **Reports (group / message / user)** | **Keep in our DB.** Reports are our product/moderation data. Admin UI lists reports; “resolve” action can call OpenIM admin API (e.g. delete message, dismiss group) and then update report status in our DB. |
| **Media catalog (stickers, etc.)** | Keep in our backend (admin CRUD). Not stored in OpenIM; clients get catalog from us and send OpenIM custom messages with sticker ID/URL. |
| **Broadcast / import users to group** | “Import users” can be implemented by backend: resolve users in our DB, then call OpenIM API to add members to group. |

---

## Webhooks (OpenIM → Our Backend)

- OpenIM can send **webhooks** on events (e.g. after send message, after create group). Our backend exposes a webhook endpoint (e.g. `/webhooks/openim`), validates secret, and then:
  - **Moderation:** Check content, log, auto-delete or flag.
  - **Analytics:** Count messages, new groups.
  - **Custom push or side effects:** If we need to trigger something else (e.g. our own notification) on message send, do it here — but **don’t** use webhooks to “deliver” messages to clients; OpenIM already does that.

So we **add** a webhook handler; we don’t replace our admin APIs with webhooks. Admin UI still talks to **our** backend; our backend talks to OpenIM admin API and our DB.

---

## Implementation Order

1. Obtain and cache OpenIM **admin token** in our backend (same pattern as implementation-doc: get admin token with secret, cache, use for admin calls).
2. Implement admin handlers that **proxy or translate** to OpenIM admin API: list groups, get messages, delete message, dismiss group, kick member, force_logout.
3. Keep admin UI unchanged from the user’s perspective: same routes (e.g. `/admin-service/...`, `/chat-service/api/v1/admin/...`), same JWT. Only the backend implementation changes from “read/write our DB” to “call OpenIM + our DB where needed.”
4. Add webhook endpoint; configure OpenIM with our URL and secret; use for moderation/analytics only.
5. Keep reports and audit data in our DB; link to OpenIM IDs (user_id, group_id, conversation_id, message id) for reference.

See [implementation-doc.md](../implementation-doc.md) §5 (webhooks), §7 (admin), and [07-data-and-storage.md](./07-data-and-storage.md) for what stays in our stores.

---

## Admin UI: Native/Built-in vs Wrapper (Decision Note)

B hongly raised: *use OpenIM’s native/built-in admin where possible; a wrapper can add complexity and latency.*

**Latency:** For an admin UI, one extra hop (Dashboard → our backend → OpenIM) is rarely the bottleneck. It only matters if responses are very slow (e.g. >2–3s). So latency alone is usually **not** a strong reason to avoid a wrapper.

**Complexity:** A wrapper means we maintain endpoints that proxy or translate to the OpenIM admin API. The flip side: **we already need our own backend** for things OpenIM doesn’t own — ban user (our DB + Redis + `force_logout`), reports (our DB), media catalog, audit. So we can’t remove the backend; the choice is how much of “admin” goes through it.

**Recommendation:**

| Approach | When it fits |
|----------|------------------|
| **Use OpenIM’s built-in admin** (e.g. openim-admin at port 11002, or community UIs like opim_admin that talk to OpenIM) for **OpenIM-centric** tasks: browse messages, list groups, dismiss group, kick member, view OpenIM users. Then add a **small custom dashboard** (or a few pages in your app) only for **your** workflows: reports list, “ban user” (DB + Redis + force_logout), media catalog, audit. | You’re okay with two entry points (OpenIM admin + our dashboard). Less code to maintain; no wrapper for OpenIM-only operations. |
| **Single custom dashboard (wrapper)** — your React (or other) UI talks only to your backend; your backend calls OpenIM admin API + your DB. One login (your JWT), one nav, one UX. | You want one unified, customized experience (branding, combined data, custom workflows). The extra hop is acceptable; complexity lives in one place and you have full control. |

**Hybrid:** Use OpenIM’s native admin for quick/raw OpenIM operations (and for ops/monitoring if it includes Grafana etc.), and keep a **custom dashboard** for product/moderation (reports, ban, catalog, audit). That way you still avoid wrapping every OpenIM admin call in your backend, while having a tailored UI for “our” concerns.

**Security note:** Don’t send the OpenIM **admin token** to the browser. If the custom UI needs to perform admin actions, those should go through your backend (which holds and caches the admin token and calls OpenIM server-side). So even with a “native” OpenIM UI for viewing, any custom actions that need the admin token should still go via your backend.
