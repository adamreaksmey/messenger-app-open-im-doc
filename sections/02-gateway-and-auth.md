# 02 — Gateway & Authentication (Current vs OpenIM)

This section maps the **current** gateway and auth to the **OpenIM-based** design so we can implement the migration without rebuilding everything from scratch.

---

## Current Gateway Behavior

- **Routing:** Path-based. `/admin-service/*` → admin backend; `/user-service/*` → user backend; `/chat-service/*` → chat backend. Prefix is stripped before forwarding.
- **Admin:** JWT in `Authorization: Bearer <token>`. Gateway verifies JWT, sets `X-Gateway-Admin-*`; backends trust these headers.
- **User/Chat:** HMAC. Client sends `Authorization: Session <sessionId>`, `X-Signature`, `X-Timestamp`, `X-Nonce`. Gateway loads session from Redis/Postgres (session → userId, deviceId, serverHMACKey), verifies signature, forwards. Session TTL can be slid in Redis.

**No OpenIM or imToken today.** All requests are either unauthenticated, JWT (admin), or HMAC (user/chat).

---

## OpenIM Target: Two Paths

### REST (through gateway)

- Client still sends **same auth** (DH session for mobile, JWT for admin).
- Gateway must:
  1. Verify auth (unchanged).
  2. Resolve **user’s OpenIM token** from Redis (`im:token:<userID>`).
  3. **Inject** `token` (imToken) and `operationID` on the outgoing request.
  4. **Strip** our auth headers (Authorization, X-Signature, etc.) so OpenIM only sees its token.
  5. **Proxy** to OpenIM REST API (e.g. path `/im/*` → OpenIM, e.g. strip `/im` and forward to OpenIM’s `/msg/send_msg` etc.).

So we **extend** the gateway: add an OpenIM proxy middleware that runs **after** auth middleware and only on routes that target OpenIM (e.g. `/im/*`). Same route table concept; new backend = OpenIM API.

### WebSocket (not through gateway)

- **WebSocket must not go through the Go gateway.** OpenIM holds the connection and expects the client to connect with **imToken** (e.g. query param or header).
- **imToken lifecycle:** Minted at login (after OTP verification); stored in Redis for gateway REST use; **returned once to the client** for WebSocket. Client stores it securely and connects to `wss://your-domain.com/ws` (or similar). Nginx routes `/ws` **directly to OpenIM** (e.g. OpenIM WS port), not to our gateway.

So we **add**:
- At login: call OpenIM to register user (if new) and mint imToken; save in Redis; return imToken + wsURL in login response.
- Nginx: route `/ws` to OpenIM WS; keep `/api/*` and `/im/*` to our gateway (gateway then forwards `/im/*` to OpenIM REST).

---

## Implementation Notes (Efficient Reuse)

1. **Keep existing route table** in the gateway. Add a new “backend” for OpenIM REST (e.g. paths under `/im` or a dedicated prefix). Route selection: same as today (path prefix); for `/im/*`, backend = OpenIM, auth = same user HMAC (or admin JWT for admin OpenIM calls if we ever do that).
2. **Auth middleware:** Reuse as-is. It already sets `userID` (or admin context) in the request. The new OpenIM proxy middleware only runs when the backend is OpenIM; it reads `userID`, looks up `im:token:<userID>` in Redis, injects token + operationID, strips our headers, proxies.
3. **Session store:** Unchanged for HMAC. **New:** Redis key `im:token:<userID>` with imToken string. TTL e.g. 7 days; refresh when user re-authenticates. On ban: delete this key and call OpenIM `force_logout`.
4. **User service (login):** After successful OTP verification (and device/session creation), call OpenIM (from backend):
   - Register user if first time.
   - Mint imToken (platform-specific).
   - Store in Redis `im:token:<userID>`.
   - Return in response: sessionId (existing), **imToken**, **wsURL** (e.g. `wss://your-domain.com/ws`). Client uses sessionId for REST, imToken only for WebSocket.
5. **Nginx:** Two locations:
   - `/api/*`, `/im/*` → gateway (our Go app). Gateway proxies `/im/*` to OpenIM REST.
   - `/ws` → OpenIM WebSocket (e.g. `proxy_pass http://openim-server:10001`), with long timeouts.

No need to rebuild gateway from scratch: **add** imToken lookup + OpenIM proxy + Nginx WS passthrough, and **add** login-step to mint and return imToken.

---

## Summary Table

| Aspect | Current | OpenIM migration |
|--------|---------|-------------------|
| Gateway routing | Prefix → admin/user/chat backends | Same + `/im/*` → OpenIM REST |
| User auth (REST) | HMAC (Session + X-Signature, etc.) | Same; then inject imToken for `/im/*` |
| Admin auth | JWT | Same |
| Where imToken is used | N/A | REST: gateway injects from Redis. WS: client sends to OpenIM only |
| WebSocket | Our NestJS Socket.IO (/ws-app) | OpenIM WS; Nginx direct to OpenIM |
| Login response | sessionId, user, etc. | Add imToken, wsURL |

See [implementation-doc.md](../implementation-doc.md) §3 for full gateway/WebSocket and Nginx examples.
