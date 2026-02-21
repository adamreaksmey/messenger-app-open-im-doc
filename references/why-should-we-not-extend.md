# Why You Should Not Extend OpenIM's Core

Avoid editing OpenIM's core code as much as possible. Implement customized features in **your wrapper** instead. Below are the main reasons and how OpenIM is set up to support this approach.

---

## 1. Maintainability and Upgradability

OpenIM is an actively developed open-source project. **Direct modifications to its core codebase** make it much harder to upgrade. As upstream adds features and fixes bugs, you would have to keep reapplying your custom changes, leading to:

- Complex merge conflicts  
- High ongoing maintenance  
- Difficulty following official release paths  

The [OpenIM Branch Management and Versioning](https://codewiki.google/github.com/openimsdk/open-im-server#openim-branch-management-and-versioning-a-blueprint-for-high-grade-software-development) documentation (especially the "Release Process" section) describes a clear versioning and release strategy: [`MAJOR.MINOR.PATCH`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/docs/contrib/version.md?plain=1#L37) protocol, [`main`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/cmd/main.go#L50) and [`release`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/.env#L4) branches, with new work on [`main`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/cmd/main.go#L50) and backports to supported release branches. Modifying the core makes it nearly impossible to follow these upgrade paths.

---

## 2. Community Support and Documentation

Community help and docs assume the **official, unmodified** OpenIM. If you run a heavily modified core:

- Debugging is harder  
- Your environment is no longer standard  
- Many answers and examples won’t apply  

The project’s [`README.md`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/README.md?plain=1#L22) encourages joining the community (e.g. Slack, Twitter) and contributing via the [Contributor Documentation](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/CONTRIBUTING.md?plain=1#L94). These are most useful when you stay close to a standard setup.

---

## 3. Stability and Reliability

OpenIM’s core is built to be robust and scalable (see [OpenIM Server Architecture and Core Services](https://codewiki.google/github.com/openimsdk/open-im-server#openim-server-architecture-and-core-services)). **Changing the core** increases the risk of:

- New bugs  
- Performance regressions  
- Weakening the whole messaging platform  

OpenIM enforces quality and style with [`golangci-lint`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/.golangci.yml#L67) and strict rules: e.g. no [`panic`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/cmd/main.go#L363), and proper error wrapping and propagation, as in the [OpenIM development specification](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/docs/contrib/go-code.md?plain=1#L309) in [`docs/contrib/go-code.md`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/docs/contrib/go-code.md). Deviating from these in a fork can compromise stability.

---

## 4. Leveraging OpenIM's Extensibility

OpenIM is built for integration: microservices, clear APIs, and **extensibility points** you can use instead of forking. It already provides:

- An **API Gateway** with middleware ([API Gateway and Client Interaction](https://codewiki.google/github.com/openimsdk/open-im-server#openim-server-architecture-and-core-services-api-gateway-and-client-interaction))  
- A **webhook** mechanism for event processing  

Use these rather than editing core code.

### API Gateway

The [`internal/api`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api) directory hosts the HTTP API gateway: it handles requests, routes them to gRPC microservices, and applies cross-cutting concerns via middleware.

You can plug in your own auth by using or extending this middleware, for example:

- **[`GinParseToken`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L347)** in [`internal/api/router.go`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go) for user identification  
- **[`CheckAdmin`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/pkg/authverify/token.go#L34)** in [`internal/api/config_manager.go`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/config_manager.go) for admin privileges  

### Webhooks

Webhooks let you run custom logic **before or after** specific events, without changing OpenIM’s core:

| Area   | Location | Examples |
|--------|----------|----------|
| Messages | [`internal/rpc/msg/callback.go`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/msg/callback.go) ([`msgServer`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/msg/server.go#L59)) | [`webhookBeforeSendSingleMsg`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/msg/callback.go#L71), [`webhookAfterSendSingleMsg`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/msg/callback.go#L93) |
| Friends  | [`internal/rpc/relation/callback.go`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/relation/callback.go) ([`friendServer`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/relation/friend.go#L45)) | [`webhookBeforeAddFriend`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/relation/callback.go#L36), [`webhookAfterAddFriend`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/relation/callback.go#L54) |
| User reg | [`internal/rpc/user/callback.go`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/user/callback.go) ([`userServer`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/user/user.go#L56)) | [`webhookBeforeUserRegister`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/user/callback.go#L88), [`webhookAfterUserRegister`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc/user/callback.go#L108) |

These let your external services react to OpenIM events without altering core code.

---

## 5. Clear Separation of Concerns

A **wrapper** keeps a clear boundary between:

- **OpenIM’s core** — messaging, users, groups, etc.  
- **Your app** — custom auth, business rules, integrations  

That separation makes the system easier to understand, test, and maintain, and follows good practice: keep your requirements in your own layer and treat OpenIM as a third-party component you integrate, not fork.

---

## Risks of Modifying the Core

### 1. Hidden Couplings and Fragility

- OpenIM’s code is **highly interconnected**: a small change in one microservice can ripple into unexpected behavior elsewhere.
- Even with infinite patience, you **cannot predict every side effect**, especially under load or in production.
- Bugs introduced here might be subtle and extremely hard to trace—they won’t necessarily show up in dev or test environments.

### 2. Upgrade and Security Risks

- OpenIM actively fixes bugs and security vulnerabilities. If you’ve patched the core, **upgrading means either discarding your changes or merging them again**, which could:
  - Accidentally undo security fixes.
  - Reintroduce old bugs.
  - Require re-auditing every security patch.
- Time may be “irrelevant” now, but **security and compliance risk isn’t**, and that risk grows every month your fork drifts from upstream.

### 3. Lost Standardization

- Community support, examples, and documentation **assume a standard OpenIM deployment**.
- Once you heavily modify the code, **you are on your own** for debugging, scaling, or performance tuning.
- Even if you have all the time in the world, you **lose the benefit of collective knowledge**—your team becomes the sole experts on a fragile fork.

### 4. Testing Explosion

- Every change you make in the core multiplies testing complexity: functional, integration, load, concurrency, etc.
- Even one minor feature might require **redoing tests for dozens of other services** because they may implicitly depend on the part you changed.
- “Time is irrelevant” doesn’t scale when the **number of edge cases is astronomical**.

### 5. Extensibility Exists for a Reason

- OpenIM was built to allow **wrappers, webhooks, and middleware**. You can **achieve the same feature goals without touching the core**, preserving stability and control.
- Modifying the core bypasses the safe extension points—so you’re building in a **fragile shortcut** that will eventually bite you when behavior changes, upstream fixes arrive, or new features are added.

> **Bottom line:** Even if your team is okay with spending months reading and patching code, modifying OpenIM core **guarantees a long-term maintenance burden and increases risk exponentially**. Wrapping it gives you **all the flexibility you need** without touching the fragile parts, keeping your system predictable, auditable, and secure.

---

## Summary

| Do | Don’t |
|----|--------|
| Implement custom behavior in **your wrapper** | Edit OpenIM’s core |
| Use the **API Gateway** and **middleware** for auth | Fork and patch OpenIM |
| Use **webhooks** for event-driven logic | Reimplement or replace internal services |

This keeps upgrades manageable, aligns with community and docs, preserves stability, and uses OpenIM’s built-in extensibility.
