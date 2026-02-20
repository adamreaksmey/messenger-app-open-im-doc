# Current Architecture Flow

Diagrams for the **current** chat application (before OpenIM). See [sections/01-current-architecture.md](../sections/01-current-architecture.md) for the full description.

---

## 1. Component & Request Flow

```mermaid
sequenceDiagram
    autonumber
    %% Participants
    participant Mobile as Mobile / Web Client
    participant Auth as Verify Auth (JWT or HMAC)
    participant Route as Route by prefix (/admin | /user | /chat)
    participant UserSvc as User Service (auth, profile, sessions)
    participant ChatSvc as Chat Service (messages, conversations, groups)
    participant AdminSvc as Admin Service
    participant Redis as Redis (sessions, presence, notify channel)
    participant Postgres as PostgreSQL (conversations, groups, users)
    participant Mongo as MongoDB (messages)
    participant RabbitMQ as RabbitMQ (chat.message.notify, notification.push)
    participant WS as WebSocket Service (NestJS / Socket.IO)

    %% Auth & routing
    Mobile->>Auth: Request
    Auth->>Route: Route by prefix
    Route->>UserSvc: /user requests
    Route->>ChatSvc: /chat requests
    Route->>AdminSvc: /admin requests

    %% User service data access
    UserSvc->>Redis: sessions, presence
    UserSvc->>Postgres: users, profiles

    %% Chat service data access
    ChatSvc->>Mongo: messages
    ChatSvc->>Postgres: conversations, groups
    ChatSvc->>RabbitMQ: chat.message.notify
    ChatSvc->>Redis: publish if online

    %% RabbitMQ consumption
    RabbitMQ->>ChatSvc: consumer

    %% Real-time notifications
    Redis->>WS: subscribe
    WS->>Mobile: notify
    Mobile->>WS: Socket.IO /ws-app (userId + sessionId)
    WS->>Redis: presence updates
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
sequenceDiagram
    autonumber
    %% Participants
    participant App as App (Client)
    participant Socket as Socket.IO Service (/ws-app)
    participant Presence as PresenceService
    participant UserKey as Redis Key: presence:user:{userId} → socketIds
    participant SocketKey as Redis Key: presence:socket:{socketId} → userId

    %% Connection flow
    App->>Socket: connect (auth: userId, sessionId)
    Socket->>Presence: add connection
    Presence->>UserKey: map userId → socketIds
    Presence->>SocketKey: map socketId → userId
    Socket->>Socket: join room: user:userId
```

---

*Back to [README](../README.md) | [Section index](../sections/00-INDEX.md)*
