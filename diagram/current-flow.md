# Current Architecture Flow

Diagrams for the **current** chat application (before OpenIM). See [sections/01-current-architecture.md](../sections/01-current-architecture.md) for the full description.

---

## 1. Component & Request Flow

```mermaid
flowchart TB
    subgraph clients["Client Layer"]
        Mobile["Mobile / Web Client"]
    end

    subgraph gateway["Gateway (Go/Gin)"]
        Auth["Verify Auth\n(JWT or HMAC)"]
        Route["Route by prefix\n/admin | /user | /chat"]
    end

    subgraph backends["Backend Services"]
        UserSvc["User Service\n(auth, profile, sessions)"]
        ChatSvc["Chat Service\n(messages, conversations, groups)"]
        AdminSvc["Admin Service"]
    end

    subgraph data["Data Layer"]
        Redis["Redis\n(sessions, presence, notify channel)"]
        Postgres["PostgreSQL\n(conversations, groups, users)"]
        Mongo["MongoDB\n(messages)"]
        RabbitMQ["RabbitMQ\n(chat.message.notify\nnotification.push)"]
    end

    subgraph ws["Realtime"]
        WS["WebSocket Service\n(NestJS / Socket.IO)"]
    end

    Mobile --> Auth
    Auth --> Route
    Route --> UserSvc
    Route --> ChatSvc
    Route --> AdminSvc

    UserSvc --> Redis
    UserSvc --> Postgres
    ChatSvc --> Mongo
    ChatSvc --> Postgres
    ChatSvc --> RabbitMQ
    ChatSvc --> Redis

    RabbitMQ -->|"consumer"| ChatSvc
    ChatSvc -->|"publish if online"| Redis
    Redis -->|"subscribe"| WS
    WS --> Mobile

    Mobile -->|"Socket.IO /ws-app\nuserId + sessionId"| WS
    WS --> Redis
```

---

## 2. Message Send & Realtime Delivery (Sequence)

```mermaid
sequenceDiagram
    participant C as Client
    participant G as Gateway
    participant Chat as Chat Service
    participant Mongo as MongoDB
    participant PG as PostgreSQL
    participant MQ as RabbitMQ
    participant Consumer as Notify Consumer
    participant Redis as Redis
    participant WS as WebSocket Service

    C->>G: POST /chat-service/.../messages/direct (HMAC)
    G->>G: Verify HMAC, resolve session
    G->>Chat: Forward request

    Chat->>PG: Get/create conversation, next sequence
    Chat->>Mongo: Insert message
    Chat->>PG: Update conversation last message
    Chat->>MQ: Publish MessageSentEvent
    Chat->>C: 200 OK

    MQ->>Consumer: Consume event
    Consumer->>PG: Get conversation & recipients
    loop For each recipient
        alt Online (Redis presence)
            Consumer->>Redis: PUBLISH CHAT_NOTIFICATION (userId, payload)
            Redis->>WS: Message
            WS->>C: Socket.IO emit to user:userId
        else Offline
            Consumer->>MQ: Enqueue notification.push
        end
    end
```

---

## 3. Presence & WebSocket

```mermaid
flowchart LR
    subgraph client_ws["Client"]
        App["App"]
    end

    subgraph ws_svc["WebSocket Service"]
        Socket["Socket.IO\n/ws-app"]
        Presence["PresenceService"]
    end

    subgraph redis_presence["Redis"]
        UserKey["presence:user:{userId}\n→ socketIds"]
        SocketKey["presence:socket:{socketId}\n→ userId"]
    end

    App -->|"connect auth: userId, sessionId"| Socket
    Socket -->|"add"| Presence
    Presence --> UserKey
    Presence --> SocketKey
    Socket -->|"join room: user:userId"| Socket
```

---

*Back to [README](../README.md) | [Section index](../sections/00-INDEX.md)*
