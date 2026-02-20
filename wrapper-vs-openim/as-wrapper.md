# What to Keep as a Wrapper

This document lists what we **keep as a wrapper** around OpenIM: our gateway, backend services, and any logic we intentionally do **not** put inside OpenIM’s codebase.

**We prefer to keep our gateway separated and as a wrapper.** The gateway remains our single entry point for REST (auth, routing, imToken injection, proxy); we do not move gateway logic into OpenIM.

---

## 1. Gateway

| Responsibility | Wrapper | Notes |
|----------------|--------|--------|
| **Single entry point for REST** | ✅ | All REST (including `/im/*`) goes through our gateway. |
| **Auth verification** | ✅ | HMAC (session) for user, JWT for admin; gateway verifies and sets user/admin context. |
| **imToken resolution** | ✅ | Gateway looks up `im:token:<userID>` in Redis and injects imToken (and operationID) when proxying to OpenIM. |
| **Strip our headers** | ✅ | Gateway removes Authorization, X-Signature, etc. before forwarding to OpenIM so OpenIM only sees its token. |
| **Route by prefix** | ✅ | `/admin`, `/user`, `/im`, etc.; OpenIM is one backend among others. |
| **WebSocket** | ❌ not in gateway | WS goes **direct** to OpenIM (Nginx); gateway is not in the WS path. |

We do **not** fork OpenIM to implement gateway or auth inside it; the gateway stays in our codebase and our control.

---

## 2. Authentication & Sessions

| Responsibility | Wrapper | Notes |
|----------------|--------|--------|
| **Session model** | ✅ | Our session (sessionId, deviceId, HMAC key); stored in Redis/Postgres. |
| **HMAC verification** | ✅ | Gateway and our auth middleware. |
| **Admin JWT** | ✅ | Gateway validates JWT; backends trust gateway headers. |
| **OTP / login flow** | ✅ | Our user service; after verify we call OpenIM to register user and mint imToken, then store imToken in Redis. |
| **imToken storage** | ✅ | Redis `im:token:<userID>` with TTL; refreshed on re-auth; deleted on ban/force_logout. |

Identity and session are **ours**; OpenIM only gets an imToken we obtain from it and pass through.

---

## 3. User Service (Identity, Auth, Devices, Admin) — Not “All User” in OpenIM

OpenIM **does** have user APIs: `user_register`, `get_users_info`, and it stores **IM-facing profile** (nickname, faceURL, ex). We use that for chat display and don’t duplicate it. What we keep as a **wrapper** is everything OpenIM doesn’t do:

| Responsibility | Wrapper | Notes |
|----------------|--------|--------|
| **Identity & login** | ✅ | OTP, phone/email verification, “our” user record (e.g. phone → userID). OpenIM has no OTP or session; we own that. |
| **Session (HMAC)** | ✅ | Session storage, HMAC verification; gateway uses this to resolve imToken. |
| **Devices** | ✅ | Device list, push tokens for **non-IM** push (OTP, alerts). OpenIM handles chat push; we handle the rest. |
| **Admin user list / search** | ✅ | Our DB (e.g. search by phone, name); we don’t rely only on OpenIM’s user list for admin. Optional: sync or call OpenIM `get_users_info` when needed. |
| **Ban / force logout** | ✅ | We update our DB, delete imToken from Redis, call OpenIM `force_logout`. |
| **Extended profile (optional)** | ✅ | If we need more than nickname/faceURL (e.g. bio, settings), we store in our DB and sync nickname/faceURL to OpenIM for IM display. |

So “user service as wrapper” means: **auth, sessions, devices, admin semantics, and any profile data beyond what OpenIM stores.** IM display profile (nickname, faceURL) lives in OpenIM; we sync it there at register and on update.

---

## 4. Admin & Moderation

| Responsibility | Wrapper | Notes |
|----------------|--------|--------|
| **Admin dashboard** | ✅ | Our UI and APIs. |
| **Admin auth** | ✅ | JWT; gateway and our backend. |
| **Orchestration** | ✅ | Our backend **calls** OpenIM admin API (get_msgs, del_msgs, force_logout, dismiss_group, etc.). |
| **Reports** | ✅ | Our DB; report resolution may trigger OpenIM admin API calls. |
| **Webhook receiver** | ✅ | Our endpoint (e.g. `/webhooks/openim`); we handle moderation, analytics, logging. |
| **Media catalog (stickers, etc.)** | ✅ | Our backend CRUD; clients get catalog from us and send OpenIM messages with refs. |

We do **not** put admin UI or report storage inside OpenIM; we keep them in our backend and use OpenIM only for IM operations.

---

## 5. Data We Own

| Data | Wrapper | Notes |
|------|--------|--------|
| **Sessions** | ✅ | Redis/Postgres. |
| **imToken cache** | ✅ | Redis. |
| **Users (identity), devices, extended profile** | ✅ | Postgres. IM display (nickname, faceURL) is in OpenIM. |
| **Reports, audit logs** | ✅ | Our DB. |
| **Non-IM push tokens** | ✅ | Our backend (e.g. for OTP, alerts). |
| **Messages / conversations / groups** | ❌ | OpenIM owns these for OpenIM-backed chats. |

See [../sections/07-data-and-storage.md](../sections/07-data-and-storage.md) for full data ownership.

---

## 6. Non-Chat Push & Other Notifications

| Responsibility | Wrapper | Notes |
|----------------|--------|--------|
| **OTP, account alerts, marketing push** | ✅ | Our backend; we store device push tokens and send via FCM/APNs ourselves. |
| **Chat message push** | ❌ | OpenIM (see [in-openim.md](./in-openim.md)). |

---

## Summary

- **Gateway:** Always wrapper — auth, imToken resolution, proxy, routing; WebSocket stays direct to OpenIM.
- **Auth, sessions, devices, admin user list, webhooks, reports, media catalog, non-IM push:** All wrapper. IM profile (nickname, faceURL) is in OpenIM; we sync on register/update.
- **Messaging, groups, OpenIM WS, chat push:** OpenIM.

This keeps upgrades to OpenIM simple and keeps our business logic in our repo.

---

*Back to [README](./README.md) | [In OpenIM](./in-openim.md) | [Section index](../sections/00-INDEX.md)*
