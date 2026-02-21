# OpenIM Migration — Implementation Documentation

This directory contains the **implementation plan** for moving the current chat application to run on top of [OpenIM](https://github.com/openimsdk/openim-server). It is **documentation only** — each section describes how current behavior maps to OpenIM and what to build or change, not the code itself.

---

## Document Map

| Document | Purpose |
|----------|---------|
| **[build-on-top-strategy.md](./build-on-top-strategy.md)** | Decision paper: why we should **not modify OpenIM core** and how to implement a **wrapper-first** architecture on top of OpenIM. |
| **[implementation-doc.md](./implementation-doc.md)** | Full OpenIM-based design: architecture, gateway, auth, WebSocket, admin, media, push, env, team responsibilities. **Reference for the target state.** |
| **[sections/00-INDEX.md](./sections/00-INDEX.md)** | Section index, reading order, and critical path for migration. |
| **[sections/01-current-architecture.md](./sections/01-current-architecture.md)** | Summary of the **current** app (gateway, chat service, websocket, user service) so we know what we're migrating. |
| **[sections/02-gateway-and-auth.md](./sections/02-gateway-and-auth.md)** | Current gateway + auth vs OpenIM: imToken injection, WebSocket exception, Nginx. |
| **[sections/03-messaging-and-realtime.md](./sections/03-messaging-and-realtime.md)** | Current messaging (MongoDB, RabbitMQ → Redis → Socket.IO) vs OpenIM messages and realtime. |
| **[sections/04-conversations-and-groups.md](./sections/04-conversations-and-groups.md)** | Current conversations/groups (PostgreSQL) vs OpenIM conversations and groups. |
| **[sections/05-presence-and-notifications.md](./sections/05-presence-and-notifications.md)** | Current presence (Redis) and push (RabbitMQ) vs OpenIM options. |
| **[sections/06-admin-and-moderation.md](./sections/06-admin-and-moderation.md)** | Current admin routes vs OpenIM admin API and webhooks. |
| **[sections/07-data-and-storage.md](./sections/07-data-and-storage.md)** | Current data stores (Mongo, Postgres, Redis) vs what OpenIM owns and what we keep. |
| **[sections/08-migration-phases.md](./sections/08-migration-phases.md)** | Phased migration plan: order of work and dependencies. |
| **[diagram/current-flow.md](./diagram/current-flow.md)** | Mermaid diagrams: current component flow, message send sequence, presence & WebSocket. |
| **[diagram/openim-integration-flow.md](./diagram/openim-integration-flow.md)** | Mermaid diagrams: OpenIM component flow, login/imToken, message send, admin & webhooks. |
| **[wrapper-vs-openim/README.md](./wrapper-vs-openim/README.md)** | What to implement **inside OpenIM** vs what to keep **as a wrapper** (gateway, auth, profiles, webhooks). Gateway stays separate. |
| **[common-issues/README.md](./common-issues/README.md)** | First-time OpenIM: list of problems, gotchas, and things to watch for (with [gotchas-and-issues.md](./common-issues/gotchas-and-issues.md)). |

---

## How to Use

1. **Architecture decision first:** Read [build-on-top-strategy.md](./build-on-top-strategy.md) for the argument and implementation stance: build on OpenIM, do not modify OpenIM core.
2. **Target design:** Read [implementation-doc.md](./implementation-doc.md) to understand the desired OpenIM-based architecture.
3. **Current state:** Read [sections/01-current-architecture.md](./sections/01-current-architecture.md) to see how the app works today.
4. **Per concern:** Use the section files under `sections/` to see current → OpenIM mapping and implementation notes.
5. **Execution order:** Follow [sections/00-INDEX.md](./sections/00-INDEX.md) and [sections/08-migration-phases.md](./sections/08-migration-phases.md) for the recommended migration path.

---

## Stack Summary

- **Current:** Go/Gin gateway + Go chat service (MongoDB messages, PostgreSQL conversations/groups) + NestJS/Socket.IO websocket + Go user service. Auth: Admin JWT, User HMAC (session). Real-time: RabbitMQ → Redis → Socket.IO.
- **Target:** Same gateway (extended with imToken resolution and OpenIM proxy), OpenIM as messaging engine (messages, groups, WebSocket), your backend for auth, profiles, webhooks, admin, and optional extras (stickers, GIFs, media catalog).
