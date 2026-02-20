# OpenIM: Gotchas, Common Issues & Things to Watch For

First-time OpenIM teams often hit the same classes of problems. This doc lists them so you can avoid or fix them quickly. It’s not a full troubleshooting guide — see [§7](#7-where-to-look-when-things-break) for where to look when things break.

---

## 1. Auth & Tokens & Gateway

### 1.1 Missing `operationID` or `token` header (1001 ArgsError)

**Symptom:** OpenIM returns `errCode: 1001`, message like `header must have operationID` or `header must have token`.

**Cause:** Every OpenIM REST request **must** include:
- `operationID` — unique per request (e.g. UUID or timestamp).
- `token` — user’s imToken (or admin token for admin APIs).

**Fix:** In gateway proxy: always set both headers before forwarding. For direct backend calls to OpenIM, use a helper that injects `operationID` and the correct token. Never forward client requests to OpenIM without adding these.

---

### 1.2 “im session expired, please re-login”

**Symptom:** Gateway returns this when proxying to OpenIM.

**Cause:** Redis key `im:token:<userID>` is missing or expired. User needs to log in again so your backend mints a new imToken and stores it in Redis.

**Fix:** Ensure login flow (OTP verify) mints imToken and does `SET im:token:<userID> <token>` with a sensible TTL (e.g. 7 days). Refresh this on every re-auth. If you use a short TTL, consider sliding it on each successful proxied request, or have the client re-login when they get this error.

---

### 1.3 Gateway forwards our auth headers to OpenIM

**Symptom:** OpenIM rejects requests or behaves oddly.

**Cause:** OpenIM doesn’t understand `Authorization: Session ...`, `X-Signature`, `X-Timestamp`, `X-Nonce`. If the gateway forwards them, they can confuse OpenIM or leak into logs.

**Fix:** In the OpenIM proxy middleware, **strip** your auth headers before proxying and send only OpenIM’s `token` and `operationID`. See implementation-doc §3.

---

### 1.4 Ban / force logout doesn’t fully work

**Symptom:** After “ban user,” the user can still send messages or stay connected.

**Cause:** Ban must do **all** of: (1) mark user banned in your DB, (2) delete `im:token:<userID>` from Redis (so gateway denies REST), (3) call OpenIM **force_logout** (so OpenIM drops WebSocket and invalidates token).

**Fix:** Implement all three in your admin “ban user” handler. If you only delete Redis, the existing WebSocket connection can keep working until it dies or reconnects.

---

### 1.5 User not registered in OpenIM before minting token

**Symptom:** Token mint or send message fails with “user not found” or similar.

**Cause:** OpenIM needs the user to exist (via `user_register`) before you can mint a user token or send messages as that user.

**Fix:** On first successful login (or when you create the user), call OpenIM `user_register` with userID, nickname, faceURL, then mint the token. Don’t mint token for users that were never registered.

---

## 2. WebSocket & Realtime

### 2.1 Routing WebSocket through the gateway

**Symptom:** WS connections fail, or you try to proxy WS in Go and it’s complex and fragile.

**Cause:** OpenIM WebSocket is long-lived and stateful. Proxying every frame through your gateway is the wrong pattern (see implementation-doc §3).

**Fix:** Client connects **directly** to OpenIM WS (e.g. Nginx `location /ws` → OpenIM WS port). Client uses imToken (from login) in query or header as OpenIM expects. Gateway is **not** in the WebSocket path.

---

### 2.2 Nginx / LB cutting WebSocket connections

**Symptom:** WS connects then drops after 60s or so.

**Cause:** Default proxy read/send timeouts are often 60s. OpenIM connections are long-lived (hours).

**Fix:** For the OpenIM WS location, set long timeouts, e.g. `proxy_read_timeout 3600s; proxy_send_timeout 3600s;` (or higher). See implementation-doc §3 Nginx example.

---

### 2.3 Client uses wrong URL or token for WebSocket

**Symptom:** WS handshake fails or OpenIM returns auth error.

**Cause:** Client must use the **wsURL** returned at login and the **imToken** (not sessionId). Some clients mistakenly use the REST base URL or sessionId for WS.

**Fix:** Document clearly: REST = sessionId (via gateway); WS = imToken + wsURL, direct to OpenIM. Store imToken securely and use it only for the WebSocket connection.

---

## 3. OpenIM API Quirks

### 3.1 Admin token expiry and caching

**Symptom:** Admin API calls start failing with auth errors after some hours.

**Cause:** Admin token expires. If you cache it forever, it will eventually be invalid.

**Fix:** Cache admin token with an expiry (e.g. 6 hours). Before each admin call, check expiry and refresh (call OpenIM with SECRET to get a new token) if needed. See implementation-doc §4 OpenIM Client example.

---

### 3.2 Platform ID mismatch

**Symptom:** Token works for one client type but not another, or push/behavior is wrong.

**Cause:** OpenIM uses platform IDs (e.g. iOS=1, Android=2, Web=5). Minting token with wrong platformID can affect multi-device behavior or push routing.

**Fix:** When minting user token, pass the correct platformID from the device/platform field in your login request. Keep a consistent mapping (e.g. in a small table or enum) so all services agree.

---

### 3.3 Path prefix when proxying

**Symptom:** OpenIM returns 404 or “unknown path.”

**Cause:** Your gateway might expose `/im/msg/send_msg` but OpenIM expects `/msg/send_msg`. Or the reverse.

**Fix:** In the proxy director, strip your prefix (e.g. `/im`) so the path sent to OpenIM matches its API. Document the mapping so the team knows which paths are proxied and how.

---

## 4. Webhooks

### 4.1 OpenIM rejects webhook response

**Symptom:** OpenIM retries webhook or marks it failed; message delivery or group creation might be affected.

**Cause:** OpenIM expects a **specific** response shape. Many docs and examples use `actionCode: 0, errCode: 0, errMsg: ""` to mean “success.” Wrong shape or non-2xx can be treated as failure.

**Fix:** Return 200 with the exact JSON OpenIM expects (see implementation-doc §5 and OpenIM webhook docs). Handle errors inside your handler but still return the success shape so OpenIM doesn’t retry unnecessarily. Validate webhook secret (e.g. header) and return 401 for invalid calls, but use the correct body shape for accepted requests.

---

### 4.2 Webhook URL unreachable from OpenIM

**Symptom:** OpenIM never calls your webhook, or you see timeouts in OpenIM logs.

**Cause:** Webhook URL is internal/local, or firewall/security group blocks OpenIM (e.g. in Docker) from reaching your backend.

**Fix:** Use a URL reachable from the OpenIM deployment (e.g. same Docker network, or a public hostname with correct routing). Test with a simple POST from the OpenIM host to your webhook URL. Check OpenIM webhook config (URL, timeouts).

---

## 5. SDK & Client

### 5.1 SDK init called more than once

**Symptom:** Unpredictable behavior, crashes, or “already initialized” errors.

**Cause:** OpenIM SDK is often designed to be initialized **once** per process/app lifecycle. Calling init again can leave global state inconsistent.

**Fix:** Init once (e.g. after login or on app start when you have imToken and wsURL). Reuse the same SDK instance. For re-login, follow SDK docs (some support “logout” then “login” without full re-init).

---

### 5.2 Token expired / kicked offline not handled

**Symptom:** App stays in a broken state after token expiry or force_logout; user doesn’t get a clear “please log in again” flow.

**Cause:** SDK exposes callbacks like `onUserTokenExpired` and `onKickedOffline`. If the app ignores them, the user sees stale or failed behavior.

**Fix:** Register listeners for token expired and kicked offline. On these events, clear local state, optionally clear stored imToken, and redirect to login. Don’t keep trying to send with an invalid token.

---

### 5.3 Multiple accounts / tabs (Web)

**Symptom:** Second login overwrites first, or messages go to wrong window; “multi-account” issues.

**Cause:** Single-page apps or single SDK instance often assume one user. Storing one global imToken and one WS connection doesn’t work for multiple accounts or tabs.

**Fix:** Either one account per tab (separate SDK instance per tab) or use OpenIM’s multi-account support if the SDK and server support it. Document the chosen model so frontend and backend stay in sync.

---

## 6. Infrastructure & Ops

### 6.1 Redis key collision with OpenIM

**Symptom:** Odd auth or session behavior; OpenIM or your app overwrites each other’s keys.

**Cause:** If your app and OpenIM share the same Redis, key names might collide (e.g. both use `session:*` or `token:*`).

**Fix:** Use a dedicated prefix for your keys (e.g. `im:token:`, `session:`, `presence:user:`). Check OpenIM’s Redis key layout and avoid overlapping namespaces. If possible, use separate Redis DBs or instances for your app vs OpenIM.

---

### 6.2 OpenIM’s MongoDB vs your MongoDB

**Symptom:** Confusion about who owns which collections; migrations or backups affect the wrong data.

**Cause:** OpenIM uses MongoDB for messages (and possibly other data). You might still have your own MongoDB for legacy or non-OpenIM data. Same product, different instances or DBs.

**Fix:** Clearly separate: “OpenIM MongoDB” (managed by OpenIM, don’t write app messages there) vs “our MongoDB” (if you keep any). Document which services connect to which. Don’t point your chat service at OpenIM’s MongoDB for your own message writes after migration.

---

### 6.3 Wrong OpenIM port (REST vs WS vs Admin)

**Symptom:** Connection refused or wrong API version; admin calls hit user API.

**Cause:** OpenIM uses different ports for REST (e.g. 10002), WebSocket (e.g. 10001), and Admin API (e.g. 10008). Mixing them causes wrong behavior.

**Fix:** Document and configure: REST base URL (for gateway proxy and backend user-token calls), WS URL (for client), Admin API base URL (for backend admin token and admin operations). Double-check Nginx and env vars against OpenIM’s actual ports.

---

### 6.4 OpenIM not healthy at startup

**Symptom:** Your backend or gateway starts but fails when calling OpenIM (connection refused, timeouts).

**Cause:** OpenIM (or its dependencies: ETCD, Kafka, MongoDB, Redis) aren’t ready yet. Docker Compose order or health checks might not wait long enough.

**Fix:** Wait for OpenIM health (e.g. `GET /healthz`) before marking your app “ready” or before running integration tests. Use depends_on and healthchecks in Compose; optionally retry OpenIM client creation in your backend until OpenIM responds.

---

## 7. Where to Look When Things Break

### 7.1 OpenIM error codes

OpenIM REST and SDK return `errCode`. Check the official API docs for the error code list (e.g. [OpenIM REST API / errcode](https://docs.openim.io/restapi/errcode)). Common ones:

- **1001** — ArgsError (missing/invalid header or param, e.g. operationID, token).
- **10100–10102** — User (e.g. user not found, login issues).
- **10200–10207** — Message (unsupported type, decompression, etc.).
- **10301–10303** — Conversation.
- **10400–10401** — Group (e.g. group not found).
- **14** / “error reading from server: EOF” — Connection closed (server crash, network, or timeout).

Log `errCode` and `errMsg` in your gateway and backend so you can match them to the docs.

---

### 7.2 Logs

- **Your gateway/backend:** Log every proxied request (path, userID, and whether imToken was found). Log OpenIM response status and errCode.
- **OpenIM:** Depends on deployment. Docker: `docker logs <openim-api-container>`. Linux/systemd: `journalctl -u openim-api` or check log path in OpenIM config (`log.storageLocation`). Look for WARN, ERROR, PANIC.
- **Client/SDK:** Many SDKs write to a log directory or console; enable in debug builds and check for connection, token, and listener errors.

---

### 7.3 Checklist when “nothing works”

1. **OpenIM up?** `curl http://<openim-host>:10002/healthz` (or your REST port).
2. **Headers?** Every OpenIM request has `operationID` and `token`.
3. **User registered?** Call `user_register` before minting token and sending messages.
4. **Redis:** Can your gateway read `im:token:<userID>`? TTL not expired?
5. **WebSocket:** Client using imToken + wsURL? Nginx routing `/ws` to OpenIM WS port with long timeouts?
6. **Webhook:** If enabled, can OpenIM reach your URL? Response shape correct?

---

### 7.4 External references

- [OpenIM REST API](https://docs.openim.io/restapi) — endpoints, params, error codes.
- [OpenIM Docker](https://github.com/openimsdk/openim-docker) — run OpenIM and see required env (SECRET, ports, etc.).
- [Troubleshooting methodology (OpenIM)](https://nsddd.top/posts/troubleshooting-guide-for-openim/) — general approach: identify → locate (logs, Delve) → fix. Use `make check` or `systemctl status openim-api` to verify services.

---

*Back to [common-issues README](./README.md) | [Doc index](../README.md)*
