# 09 — Admin Auth & OpenIM Admin Integration Plan

This section defines the secure execution plan for:

1. **Admin Dashboard -> Our Admin Service** authentication/authorization.
2. **Our Admin Service -> OpenIM Admin APIs** service-to-service verification.

It is written as an implementation plan, not just guidance.

---

## 1) Goal and Scope

### Goal
Ship a secure admin architecture where:
- browser clients never call OpenIM admin APIs directly,
- admin identity and permissions are enforced by our own service,
- OpenIM admin credentials stay backend-only.

### Scope
- Admin login/session model
- Endpoint middleware and verification chain
- RBAC policy and action mapping
- OpenIM admin token handling
- Network and secrets controls
- Audit and incident response requirements

---

## 2) Key Decision: Admin Auth Is Different from End-User Chat Auth

Admin auth is a separate trust domain from end-user chat auth.

- **End user chat path:** DH/OTP/session model + OpenIM user token for IM operations.
- **Admin path:** our admin login/JWT/RBAC/MFA model, then backend calls OpenIM admin APIs with backend-held credentials.

Do not reuse end-user auth assumptions for admin actions.

---

## 3) Trust Boundaries and Allowed Paths

### Allowed
- Dashboard -> our Admin Service (`/api/v1/admin/*`)
- Admin Service -> OpenIM admin endpoints (private network only)

### Not allowed
- Dashboard/browser -> OpenIM admin HTTP endpoints
- Dashboard/browser -> OpenIM secret or admin token
- Any public internet access to OpenIM admin ports

---

## 4) End-to-End Request Flow

1. Admin logs in to our Admin Service.
2. Admin Service verifies credentials and issues:
   - short-lived access JWT (10-15 min),
   - rotating refresh token (up to 7 days).
3. Dashboard calls admin endpoints with access JWT.
4. Middleware chain validates identity, session, MFA state, and role.
5. Handler calls OpenIM via Admin Client.
6. Admin Client injects `operationID` and valid OpenIM admin token.
7. Result is returned to dashboard; audit event is written.

---

## 5) Verification Chain (Middleware Order)

For all protected admin routes (except `/login`, `/refresh`, `/logout`):

1. **RequestID middleware**  
   Attach `requestID` to context and logs.

2. **Rate limiting**  
   Per admin + IP limits (stronger limits for login endpoints).

3. **JWT verification**  
   Enforce algorithm, issuer, audience, exp/nbf.

4. **Session revocation check**  
   Validate `jti`/session ID exists and is not revoked (Redis-backed).

5. **MFA gate**  
   Enforce MFA for privileged roles/actions.

6. **RBAC authorization**  
   Check permission for requested action.

7. **Input validation**  
   Validate payload and IDs before side effects.

8. **Handler execution**

9. **Audit write (post-handler)**  
   Record actor, action, target, result, requestID, operationID.

---

## 6) RBAC Baseline

Use explicit action permissions, not broad route-based checks only.

| Action | moderator | superadmin |
|---|---:|---:|
| View users / messages / groups | yes | yes |
| Delete message | yes | yes |
| Temporary user ban | yes | yes |
| Permanent ban / unban | no | yes |
| Dismiss group / kick member | yes (policy dependent) | yes |
| Manage admins / roles | no | yes |
| Security/config operations | no | yes |

For destructive/bulk actions, require step-up MFA.

---

## 7) OpenIM Admin Token Strategy

Treat OpenIM admin token as a **service credential**.

### Requirements
- Kept only in backend memory/cache; never exposed to clients.
- Minted using secure backend secret handling.
- Cached with safety margin (`expiresAt - skew`).
- Auto-refresh on expiry; one retry on `unauthorized` due to stale token.
- Emit metrics for refresh success/failure and token age.

### Version note
Endpoint naming can vary by OpenIM version. Use the installed version's documented admin token endpoint and keep this encapsulated inside `OpenIMAdminClient`.

---

## 8) Network and Secrets Hardening

1. Keep OpenIM admin APIs private (VPC/internal subnet only).
2. Firewall/security groups: only Admin Service can reach OpenIM admin ports.
3. Store OpenIM secret in secret manager (or restricted env injection); never in frontend.
4. Rotate admin secrets by runbook; validate token renewal after rotation.
5. Enforce TLS in all non-local environments.

---

## 9) Logging, Audit, and Traceability

Every admin action must emit:
- actor admin ID and role,
- action name and target resource IDs,
- requestID and OpenIM operationID,
- outcome (success/failure) and error code.

Audit log should be append-only from application perspective.

Minimum sensitive events:
- login success/failure, refresh, logout,
- role change,
- ban/unban,
- message delete,
- group dismiss/kick,
- OpenIM admin token refresh failures.

---

## 10) Admin Endpoint Patterns

All dashboard actions call our service first.

Examples:
- `GET /api/v1/admin/users`
- `GET /api/v1/admin/messages?conversationID=...`
- `DELETE /api/v1/admin/messages/:msgID`
- `POST /api/v1/admin/users/:userID/ban`
- `POST /api/v1/admin/groups/:groupID/dismiss`

Handlers should:
1. validate permission,
2. call OpenIM API and/or our DB as needed,
3. write audit event,
4. return normalized response.

---

## 11) Rollout Plan

### Phase A — Identity and Session
- Implement admin login, refresh, logout.
- Add JWT verification + revocation checks.
- Add login rate limits and account lock policy.

### Phase B — Authorization
- Implement permission map and middleware enforcement.
- Add MFA for privileged actions.

### Phase C — OpenIM Integration
- Build `OpenIMAdminClient` with token manager.
- Migrate admin handlers to call OpenIM endpoints through client.
- Keep OpenIM admin ports private and restricted.

### Phase D — Audit and Hardening
- Ship structured audit events and dashboards.
- Add integration tests for forbidden/allowed actions.
- Run failure drills: token expiry, OpenIM outage, permission misconfiguration.

---

## 12) Acceptance Criteria

This plan is complete when all are true:

1. Browser cannot reach OpenIM admin API directly.
2. Admin dashboard never receives OpenIM admin token/secret.
3. Every protected admin route enforces JWT + revocation + RBAC (and MFA where required).
4. Admin actions to OpenIM run only through `OpenIMAdminClient`.
5. Audit logs exist for all sensitive admin operations.
6. OpenIM admin token refresh is automatic and observable.

---

## 13) Implementation Checklist (Short)

- [ ] Add admin auth endpoints (`login`, `refresh`, `logout`)
- [ ] Add middleware chain in correct order
- [ ] Define RBAC permission map and tests
- [ ] Add MFA policy for destructive actions
- [ ] Implement `OpenIMAdminClient` token caching + refresh
- [ ] Lock network access to OpenIM admin ports
- [ ] Add audit logging and alerting
- [ ] Add integration tests for authz and OpenIM call path

---

See also:
- [06-admin-and-moderation.md](./06-admin-and-moderation.md)
- [../implementation-doc.md](../implementation-doc.md)
- [../build-on-top-strategy.md](../build-on-top-strategy.md)
