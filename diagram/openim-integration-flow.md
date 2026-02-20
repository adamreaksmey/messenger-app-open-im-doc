# OpenIM Integration Flow

Diagrams for the **target** architecture with OpenIM. See [implementation-doc.md](../implementation-doc.md) and the [sections/](../sections/) for full description.

---

## 1. Component & Two Paths (REST vs WebSocket)

```mermaid
sequenceDiagram
    autonumber
    %% Participants
    participant Mobile as Mobile / Web (sessionId for REST, imToken for WS only)
    participant Admin as Admin Dashboard (JWT)
    participant Gateway as Your Gateway (Go/Gin)
    participant Auth as Verify Auth (HMAC or JWT)
    participant Resolve as Resolve imToken (Redis im:token:userID)
    participant Inject as Inject token + operationID (Strip our headers)
    participant Proxy as Proxy to OpenIM REST
    participant UserSvc as User Service (device, OTP, profile)
    participant Login as Login / OTP verify → register user in OpenIM → mint imToken → Redis
    participant Webhook as Webhook receiver (/moderation, analytics)
    participant OpenIMREST as OpenIM REST API (msg, group, user, auth)
    participant OpenIMWS as OpenIM WebSocket (port 10001)
    participant OpenIMAdmin as OpenIM Admin API (force_logout, get_msgs, etc.)
    participant Redis as Redis (sessions, im:token:userID)
    participant PG as PostgreSQL (your users, devices)

    %% REST flow
    Mobile->>Auth: REST: Session + HMAC
    Auth->>Resolve: Verify token
    Resolve->>Redis: Fetch imToken
    Resolve->>Inject: Pass token
    Inject->>Proxy: Forward with operationID
    Proxy->>OpenIMREST: Proxy REST requests

    %% WS flow
    Mobile->>OpenIMWS: WS: imToken only (direct, no gateway)

    %% Admin flow
    Admin->>Auth: JWT auth
    Auth->>UserSvc: Validate admin
    UserSvc->>PG: Query user/device info

    %% Login flow
    Login->>OpenIMREST: Register user / Verify OTP
    Login->>Redis: Store imToken

    %% Webhook flow
    OpenIMREST->>Webhook: Webhooks (moderation, analytics)

    %% Admin actions
    UserSvc->>OpenIMAdmin: Perform admin actions
```

---

## 2. Login & imToken Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant G as Gateway
    participant Backend as Your Backend
    participant OpenIM as OpenIM API
    participant Redis as Redis

    C->>G: POST /api/v1/auth/otp/verify (HMAC + OTP)
    G->>Backend: Forward
    Backend->>Backend: Verify OTP, create/find user, create session
    Backend->>OpenIM: Register user (if new)
    Backend->>OpenIM: Mint user token (userID, platformID)
    OpenIM-->>Backend: imToken
    Backend->>Redis: SET im:token:userID, imToken (TTL 7d)
    Backend->>C: 200 { sessionId, imToken, wsURL, user, ... }

    Note over C,Redis: Client stores imToken for WebSocket only. REST uses sessionId; gateway injects imToken from Redis.
```

---

## 3. Message Send & Realtime (OpenIM)

```mermaid
sequenceDiagram
    participant C as Client
    participant G as Gateway
    participant OpenIM as OpenIM Server
    participant WS as OpenIM WebSocket

    Note over C,WS: REST path (send / history)
    C->>G: POST /im/msg/send_msg (Session + HMAC)
    G->>G: Verify HMAC → userID
    G->>G: Redis GET im:token:userID
    G->>OpenIM: POST /msg/send_msg (token, operationID)
    OpenIM->>OpenIM: Store message, route to recipients
    OpenIM-->>G: 200
    G-->>C: 200

    Note over C,WS: Realtime: client connected to OpenIM WS with imToken (no gateway)
    OpenIM->>WS: Push new message
    WS->>C: Deliver to recipient's connection
```

---

## 4. Admin & Webhooks

```mermaid
sequenceDiagram
    autonumber
    %% Participants
    participant AdminUI as Admin Dashboard
    participant YourAPI as Your Backend API
    participant OpenIMAdmin as OpenIM Admin API
    participant OpenIMCore as OpenIM (message/group events)
    participant Webhook as Your /webhooks/openim

    %% Admin flow
    AdminUI->>YourAPI: JWT
    YourAPI->>OpenIMAdmin: Admin token
    OpenIMAdmin->>OpenIMAdmin: get_groups, del_msgs, force_logout, dismiss_group

    %% Webhook flow
    OpenIMCore->>Webhook: afterSendMsg, afterCreateGroup, ...
    Webhook->>YourAPI: moderation, analytics
```

---

*Back to [README](../README.md) | [Section index](../sections/00-INDEX.md)*
