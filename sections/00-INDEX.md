# Section Index & Reading Order

This file is the entry point for the **migration plan** sections. It lists documents in a logical order and summarizes the critical path.

---

## Reading Order

1. **[01-current-architecture.md](./01-current-architecture.md)** — What we have today (gateway, chat, websocket, user service, data flow).
2. **[02-gateway-and-auth.md](./02-gateway-and-auth.md)** — How gateway and auth change with OpenIM (imToken, WebSocket direct to OpenIM).
3. **[03-messaging-and-realtime.md](./03-messaging-and-realtime.md)** — Messages and realtime: what to remove (custom message pipeline) vs what OpenIM provides.
4. **[04-conversations-and-groups.md](./04-conversations-and-groups.md)** — Conversations and chat groups: OpenIM model vs current Postgres model.
5. **[05-presence-and-notifications.md](./05-presence-and-notifications.md)** — Presence and push: current Redis/RabbitMQ vs OpenIM options.
6. **[06-admin-and-moderation.md](./06-admin-and-moderation.md)** — Admin and moderation: current routes vs OpenIM admin API and webhooks.
7. **[07-data-and-storage.md](./07-data-and-storage.md)** — Data ownership: what OpenIM stores vs what stays in our DB/Redis.
8. **[08-migration-phases.md](./08-migration-phases.md)** — Phased plan and dependencies for executing the migration.

The root **[../implementation-doc.md](../implementation-doc.md)** is the **target design** (how to build the app on OpenIM); the section docs above map **current → target** and call out migration steps.

---

## Critical Path (High Level)

| Step | Focus | Doc(s) |
|------|--------|--------|
| 1 | Run OpenIM locally; verify health | implementation-doc §2 |
| 2 | Keep gateway; add imToken resolution + OpenIM REST proxy; keep DH/JWT auth | 02-gateway-and-auth |
| 3 | At login: register user in OpenIM, mint imToken, store in Redis, return imToken + wsURL to client | implementation-doc §4, 02-gateway-and-auth |
| 4 | Nginx: REST via gateway; WebSocket direct to OpenIM (no gateway in WS path) | 02-gateway-and-auth |
| 5 | Clients use OpenIM SDK; send/receive messages via OpenIM (no custom message API for core messaging) | 03-messaging-and-realtime |
| 6 | Replace custom conversation/group storage with OpenIM conversations/groups where applicable | 04-conversations-and-groups |
| 7 | Decommission or repurpose: messenger-websocket (realtime), messenger-chat-service message pipeline (RabbitMQ → Redis → WS) | 03, 05, 08 |
| 8 | Admin: call OpenIM admin API from your backend; optional webhooks for moderation | 06-admin-and-moderation |
| 9 | Presence/push: use OpenIM config or keep minimal custom layer | 05-presence-and-notifications |

---

## Cross-References

- **Gateway & imToken:** implementation-doc §3; sections 02, 07.
- **WebSocket:** implementation-doc §3 (WebSocket exception); sections 02, 03.
- **Messages:** implementation-doc §6; sections 03, 07.
- **Groups:** implementation-doc §6, §7; sections 04, 06.
- **Admin:** implementation-doc §7; section 06.
- **Data stores:** implementation-doc §2, §8; section 07.
