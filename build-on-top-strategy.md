# Build on Top of OpenIM (Do Not Modify Core) - Strategy and Implementation

This document is the single-source position for our architecture decision:

**Do not modify OpenIM's core code.**  
**Build our product capabilities on top of OpenIM through a wrapper layer we own.**

---

## 1) Decision Statement

We will treat OpenIM as an upstream dependency and messaging engine, not as an internal codebase to customize.

- OpenIM owns: message transport, conversation state, group mechanics, realtime WS, chat push.
- Our wrapper owns: auth, policy, gateway routing, data enrichment, admin orchestration, and product-specific behavior.

This gives us full product flexibility without taking fork maintenance risk.

---

## 2) Why We Should Not Modify OpenIM Core

### A. Upgrade and security risk compounds over time
- Any core patch creates a long-lived fork burden.
- Every upstream release requires merge, conflict resolution, and revalidation.
- Security fixes become slower to adopt when we diverge from upstream.

### B. Hidden coupling and regression risk
- OpenIM services are interconnected; "small" internal changes can have broad side effects.
- Bugs introduced in forked internals are harder to isolate than wrapper-layer bugs.

### C. Loss of community leverage
- Community docs, examples, and support assume standard OpenIM behavior.
- A forked deployment reduces reproducibility and makes troubleshooting slower.

### D. Wrong ownership boundary
- Our auth model (DH/JWT), policy, moderation logic, and domain rules are ours.
- Putting those rules inside OpenIM blends responsibilities and increases blast radius.

### E. No real latency win for data enrichment use cases
- If we need our DB data (for example media metadata), some component still must fetch it.
- Moving that fetch into OpenIM does not remove the fetch; it only relocates dependency.
- Wrapper-side enrichment keeps control while preserving near-identical data access cost.

### F. Better failure handling in wrapper
- Wrapper can degrade gracefully (return partial data, cached fields, retry policy).
- Core patches often force OpenIM request paths to depend on our systems without clear fallback.

### G. Clearer compliance and audit surface
- Security review and audit are easier when custom auth/policy lives in our codebase.
- Forking OpenIM expands audit scope to internals we do not want to own.

### H. Faster team velocity over the full lifecycle
- Wrapper changes are local and reversible.
- Core patches create long-term drag across releases, testing, and incident response.

---

## 3) What "Build on Top" Means in Practice

### Integration boundary
- Clients call **our gateway/backend** for REST APIs.
- Our gateway validates app auth (DH session for users, JWT for admins).
- Gateway resolves OpenIM token from Redis and injects OpenIM-required headers.
- Gateway forwards to OpenIM endpoints through supported APIs.
- WebSocket is the bounded exception: client connects directly to OpenIM WS using imToken.

### Data boundary
- OpenIM data: messages, conversation state, group state, realtime delivery.
- Our data: identity, OTP/session, devices, admin users/roles, moderation reports, enrichment tables.

### Ownership boundary
- OpenIM is product infrastructure.
- Wrapper is product logic.

---

## 4) Concrete Implementation Blueprint

### 4.1 Gateway and authentication

1. Keep one public API entrypoint in our gateway.
2. Verify user/admin auth in gateway middleware.
3. Resolve `im:token:<userID>` from Redis.
4. Inject OpenIM `token` + `operationID`.
5. Strip app-specific auth headers before forwarding to OpenIM.

Result: clients keep our auth model while OpenIM receives what it expects.

### 4.2 Login/token lifecycle

1. User completes OTP + device flow in our backend.
2. Backend registers/syncs user in OpenIM (if needed).
3. Backend mints OpenIM user token.
4. Backend stores token in Redis for gateway use.
5. Backend returns `imToken` + `wsURL` to client for direct WS connection.

### 4.3 Realtime and websocket

- Do not proxy websocket frames through app gateway.
- Route `/ws` directly to OpenIM via Nginx.
- On ban/logout:
  - delete Redis token cache
  - call OpenIM force logout to terminate active sessions

### 4.4 Business extensions (wrapper-only)

For features not natively in OpenIM:
- Call OpenIM APIs from wrapper
- Enrich with our own DB data
- Apply product policy and return merged response

Examples:
- media metadata hydration
- admin search/reporting views
- moderation workflows
- custom sticker/GIF catalogs

### 4.5 Webhooks and orchestration

- Use OpenIM webhooks for event-driven hooks (after send, group events, etc.).
- Keep side effects in wrapper (analytics, moderation pipeline, custom notifications).

---

## 5) Counterarguments and Rebuttals

### "Patching core is faster for this one feature."
Short-term yes, long-term no. It shifts cost into every future upgrade and incident.

### "Wrapper adds latency."
A thin, co-located wrapper adds small overhead compared to existing service hops.  
If latency matters, benchmark and optimize wrapper path (cache/index/payload size) before considering core changes.

### "We need deeper control."
Use wrapper + webhooks + admin APIs first.  
Only consider upstream contribution for genuinely missing primitives.

---

## 6) Guardrails: When Core Modification Is Allowed

Core change is allowed only if all checks pass:

1. No supported API/webhook path exists.
2. Requirement cannot be met in wrapper without unacceptable product impact.
3. Change is minimal, isolated, and upstream-friendly.
4. We have a rebase/upgrade plan for next 3 upstream releases.
5. We open an upstream PR or maintain a clearly scoped patch with owner + tests.

Default remains: **wrapper first**.

---

## 7) Rollout Plan

Phase 1: Foundation
- Deploy OpenIM baseline.
- Implement gateway auth + token injection.
- Complete login/token lifecycle and direct WS connection.

Phase 2: Core messaging migration
- Move DM/group messaging flows to OpenIM.
- Decommission duplicated realtime/message pipeline components.

Phase 3: Product extensions on wrapper
- Implement enrichment APIs, moderation, admin orchestration, catalog features.
- Add webhook consumers and operational observability.

Phase 4: Hardening
- Load test gateway/OpenIM path.
- Validate failover, token expiry behavior, forced logout, and rollback playbooks.

---

## 8) Success Criteria

- Upstream OpenIM upgrades can be applied without rebasing custom core logic.
- All product-specific behavior is implemented in wrapper services we own.
- No client direct HTTP access to OpenIM APIs (except controlled WS path).
- Messaging reliability and latency meet targets at expected scale.
- Security/audit review passes with clear boundary ownership.

---

## 9) Final Position

Building on top of OpenIM gives us the correct engineering tradeoff:

- open-source leverage without fork debt
- product flexibility without coupling business rules to third-party internals
- maintainable operations and faster long-term delivery

For our use case, this is the durable architecture decision.
