# Wrapper vs OpenIM — Where to Implement

This folder documents our **architecture decision**: what we implement **inside OpenIM** (or rely on OpenIM to do natively) vs what we keep **as a wrapper** (our gateway, backend, and surrounding services).

**Starter preference:** We keep our **gateway separated and as a wrapper**. We do not fork OpenIM to put gateway logic inside it; we proxy and inject tokens from our own gateway.

For the full decision argument and implementation guardrails, see [../build-on-top-strategy.md](../build-on-top-strategy.md).

---

## Documents

| Document | Purpose |
|----------|--------|
| **[in-openim.md](./in-openim.md)** | What is best implemented **inside OpenIM** or left to OpenIM natively (messaging, groups, WS, push, etc.). |
| **[as-wrapper.md](./as-wrapper.md)** | What we keep **as a wrapper** — gateway, auth, profiles, webhooks, admin orchestration. |

---

## Quick Reference

| Concern | Where | Reason |
|--------|--------|--------|
| **Gateway** | Wrapper | We keep our gateway; it does auth, imToken resolution, proxy. |
| **Auth (session, HMAC, JWT)** | Wrapper | Our identity and session model; gateway and user service own it. |
| **Messages, groups, conversations** | OpenIM | Core IM; OpenIM owns storage and delivery. |
| **Realtime (WebSocket)** | OpenIM | OpenIM WS; we don’t replace it with our own. |
| **Push (FCM/APNs for chat)** | OpenIM | OpenIM config; we use it for chat message push. |
| **Moderation / webhooks** | Wrapper | Our backend receives webhooks; we call OpenIM admin API. |
| **Admin UI & orchestration** | Wrapper | Our backend and admin dashboard; they call OpenIM admin API. |
| **IM profile (nickname, faceURL)** | OpenIM | OpenIM `user_register` / `get_users_info`; we sync on register/update. |
| **Auth, devices, admin user list, non-IM push** | Wrapper | Identity, OTP, session, devices; our user service and DB. |

---

*See [sections/00-INDEX.md](../sections/00-INDEX.md) for the full migration index and [implementation-doc.md](../implementation-doc.md) for the target design.*
