# Admin Dashboard Implementation: Route Through Your Service

**Position:** Admin dashboard requests must not talk to OpenIM directly. They must go through **your admin service**, which then calls OpenIM’s API Gateway (and, where needed, OpenIM’s admin APIs). OpenIM’s core must not be modified to support your admin flows; your service is the single place that orchestrates admin logic and talks to OpenIM.

This document states why this flow is required and how it fits the wrapper approach.

---

## 1. Centralized Admin Logic and Security

Your admin dashboard typically does more than OpenIM alone (billing, app-wide user permissions, reporting, etc.). **All administrative business logic and authorization** must live in your system, not inside OpenIM.

- Route admin requests through your admin service so that **one place** owns admin actions and security policy.
- OpenIM provides utilities such as [CheckAdmin](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/pkg/authverify/token.go#L34) for its own internal checks. You must not patch OpenIM to implement your admin rules—your service enforces them and then calls OpenIM with the appropriate credentials.

---

## 2. Consistent Authentication and Authorization

Admins authenticate with **JWT via your auth gateway**. Your admin service consumes that validated identity and then:

- **Calls OpenIM’s API Gateway** with the credentials OpenIM expects (e.g. admin token or equivalent), or  
- Uses your **token issuance** so that requests to OpenIM carry a token your wrapper has validated and OpenIM can accept.

OpenIM’s API Gateway already has mechanisms you use as-is: [GinParseToken](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L347) in [internal/api/router.go](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go), and admin checks such as [CheckAdmin](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/pkg/authverify/token.go#L34) in [pkg/authverify/token.go](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/pkg/authverify/token.go). Your JWT-based admin auth is implemented in **your gateway and admin service**; you do not modify OpenIM’s auth layer to accept your JWTs directly.

---

## 3. Encapsulation and Abstraction

Your admin service acts as a **wrapper** around OpenIM’s administrative APIs:

- The dashboard depends on **your** admin API, not on OpenIM’s URL shapes or request formats.
- If OpenIM’s APIs change, only your admin service is updated; the dashboard and other callers stay unchanged.
- This keeps OpenIM as an internal implementation detail and avoids coupling the rest of your system to OpenIM’s structure.

---

## 4. Custom Admin Features and Webhooks

Custom admin features that touch both your system and OpenIM must be **orchestrated in your admin service**, not by changing OpenIM’s code.

- For events that OpenIM must push to you (e.g. message flags, user actions), use **OpenIM’s webhook configuration** so that callbacks target your admin service. See [webhooks.yml](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/config/README.md?plain=1#L19) in [config/README.md](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/config/README.md).
- Your admin service can then react to OpenIM events asynchronously without the dashboard or OpenIM’s core being modified.

---

## 5. Performance and Scalability

Adding your admin service in front of OpenIM adds one hop. For a well-designed service, this overhead is small compared to the gains in maintainability, security, and upgradeability. Both your admin service and OpenIM’s API Gateway are expected to be performant; optimization should focus on your service and configuration, not on patching OpenIM.

---

## Required Request Flow

Admin requests must follow this path:

**Admin Dashboard → Your auth gateway (JWT validation) → Your admin service → OpenIM’s API Gateway → OpenIM’s internal RPC services.**

No direct connection from the dashboard to OpenIM. No modifications to OpenIM’s code to support your admin flows.

---

## Summary

- **Centralize** admin logic and security in your admin service.  
- **Authenticate** admins with JWT at your gateway; your service then calls OpenIM with the right credentials.  
- **Keep** OpenIM behind your wrapper so the dashboard and other systems are not coupled to OpenIM.  
- **Use** OpenIM’s webhooks to receive events in your admin service; do not modify OpenIM’s core.

This flow is the correct way to implement the admin dashboard while keeping OpenIM unmodified and maintainable.
