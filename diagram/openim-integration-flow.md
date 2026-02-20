# OpenIM Integration Flow

Diagrams for the **target** architecture with OpenIM. See [implementation-doc.md](../implementation-doc.md) and the [sections/](../sections/) for full description.

---

## 1. Component & Two Paths (REST vs WebSocket)

```mermaid
flowchart TB
    subgraph clients["Client Layer"]
        Mobile["Mobile / Web\n(sessionId for REST)\n(imToken for WS only)"]
        Admin["Admin Dashboard\n(JWT)"]
    end

    subgraph gateway["Your Gateway (Go/Gin)"]
        Auth["Verify Auth\nHMAC or JWT"]
        Resolve["Resolve imToken\nRedis im:token:userID"]
        Inject["Inject token + operationID\nStrip our headers"]
        Proxy["Proxy to OpenIM REST"]
    end

    subgraph your_backend["Your Backend"]
        UserSvc["User Service\n(device, OTP, profile)"]
        Login["Login / OTP verify\n→ register user in OpenIM\n→ mint imToken → Redis"]
        Webhook["Webhook receiver\n/moderation, analytics"]
    end

    subgraph openim["OpenIM Server"]
        OpenIMREST["REST API\n(msg, group, user, auth)"]
        OpenIMWS["WebSocket\n(port 10001)"]
        OpenIMAdmin["Admin API\n(force_logout, get_msgs, etc.)"]
    end

    subgraph data["Data"]
        Redis["Redis\n(sessions, im:token:userID)"]
        PG["PostgreSQL\n(your users, devices)"]
    end

    Mobile -->|"REST: Session + HMAC"| Auth
    Auth --> Resolve
    Resolve --> Redis
    Resolve --> Inject
    Inject --> Proxy
    Proxy --> OpenIMREST

    Mobile -->|"WS: imToken only\n(direct, no gateway)"| OpenIMWS
    Admin --> Auth
    Auth --> UserSvc
    UserSvc --> PG
    Login --> OpenIMREST
    Login --> Redis
    OpenIMREST -->|"webhooks"| Webhook
    UserSvc -->|"admin actions"| OpenIMAdmin
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
flowchart LR
    subgraph admin_flow["Admin Flow"]
        AdminUI["Admin Dashboard"]
        YourAPI["Your Backend API"]
        OpenIMAdmin["OpenIM Admin API"]
    end

    subgraph webhook_flow["Webhook Flow"]
        OpenIMCore["OpenIM (message/group events)"]
        Webhook["Your /webhooks/openim"]
    end

    AdminUI -->|"JWT"| YourAPI
    YourAPI -->|"admin token"| OpenIMAdmin
    OpenIMAdmin -->|"get_groups, del_msgs,\nforce_logout, dismiss_group"| OpenIMAdmin

    OpenIMCore -->|"afterSendMsg,\nafterCreateGroup, ..."| Webhook
    Webhook -->|"moderation, analytics"| YourAPI
```

---

*Back to [README](../README.md) | [Section index](../sections/00-INDEX.md)*
