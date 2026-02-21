# Best Wrapper Practices: Communication Flow with OpenIM

**Position:** OpenIM must be used as a black box. Your wrapper is the only place where your authentication, routing, and business logic run. No custom code belongs inside OpenIM's codebase; integration is done via its public API Gateway and gRPC services.

This document describes the correct request/response flow and confirms that this design aligns with best practices for integrating with OpenIM without modifying its core.

---

## Inbound Request Flow (Client → OpenIM)

1. **Client → Your auth gateway**  
   The client (user or admin) sends the request to **your** authentication gateway first.
   - Your gateway performs **Diffie-Hellman authentication** (users) or **JWT verification** (admins).
   - It validates the client and produces the credentials needed for the next hop (e.g. an OpenIM-compatible token or a validated user/admin ID).

2. **Your auth gateway → Your backend service**  
   The gateway forwards the **already-validated** request to one of your backend services (your wrapper).

3. **Your service → OpenIM's API Gateway**  
   Your service, acting as the wrapper, calls **OpenIM's API Gateway** over HTTP (or the appropriate transport). This is the only point where your system talks to OpenIM.
   - The request carries the authentication OpenIM expects (e.g. a token your wrapper has obtained or minted for OpenIM).
   - OpenIM's API Gateway ([`internal/api`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api)), as described in [API Gateway and Client Interaction](https://codewiki.google/github.com/openimsdk/open-im-server#openim-server-architecture-and-core-services-api-gateway-and-client-interaction), does parsing, middleware, and routing to internal gRPC services. The [`a2r.Call`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/msg.go#L155) utility is used to turn HTTP ([`gin.Context`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/msg.go#L97)) requests into gRPC calls. **You do not modify this gateway;** your wrapper is the layer that enforces your auth and then calls it.

4. **OpenIM's API Gateway → OpenIM's internal gRPC services**  
   The API Gateway translates the HTTP request into a gRPC call and sends it to the relevant OpenIM RPC service (User, Group, Message, Auth, etc., under [`internal/rpc`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc)). This step is entirely inside OpenIM; your code is not involved.

---

## Outbound Response Flow (OpenIM → Client)

When a response is returned, the path is reversed:

1. **OpenIM internal gRPC service → OpenIM's API Gateway** — The RPC service returns a gRPC response to the API Gateway.
2. **OpenIM's API Gateway → Your service** — The API Gateway converts the gRPC response to HTTP and returns it to your wrapper.
3. **Your service → Your auth gateway** — Your wrapper may apply business logic or reshape the response, then forwards it to your auth gateway.
4. **Your auth gateway → Client** — Your gateway sends the final response to the client.

---

## Why Clients Must Never Connect Directly to OpenIM

Before examining why the wrapper flow is required, it is worth being explicit about a constraint the flow diagram implies but does not state: **clients must never be permitted to send HTTP API requests directly to OpenIM's API Gateway.** The only direct client-to-OpenIM connection that is acceptable is the **WebSocket connection** for real-time messaging, and even that should be understood as a deliberate, bounded exception—not a precedent.

### The WebSocket Exception

OpenIM's real-time messaging relies on a persistent WebSocket connection between the client and OpenIM's message gateway. This connection is the foundation of push delivery, presence, and low-latency message receipt. It is inherently stateful and long-lived in a way that HTTP request/response flows are not, which makes proxying it through your wrapper a non-trivial operation with genuine latency and complexity costs.

For this reason, the WebSocket connection is the one place where a client connects to OpenIM directly. This is an architectural concession to the realities of real-time messaging—not a general license for direct client access. The WebSocket surface is narrow: it handles message delivery and presence, not user management, group operations, auth token issuance, or any of the administrative API endpoints. Everything outside that narrow surface must go through your wrapper.

Even for the WebSocket connection, your gateway should control *how* the client obtains the credentials needed to establish it. The client should not be able to connect to OpenIM's WebSocket endpoint with credentials it has obtained independently; it should receive a short-lived token from your auth gateway, scoped specifically to establishing that connection, with your gateway retaining the ability to revoke access.

### Why Direct HTTP API Access Is Prohibited

Allowing clients to call OpenIM's HTTP API directly would undermine the entire architecture. The consequences are concrete:

**1. Your authentication is bypassed entirely.**
Your DH and JWT auth logic lives in your gateway. If a client can reach OpenIM's API Gateway directly, it can interact with OpenIM using only OpenIM's own token model—completely circumventing the security layer you have built. Any client that can discover or guess OpenIM's endpoint can authenticate with OpenIM directly, using whatever credentials OpenIM accepts, without going through your validation. This is not a hardening concern; it is a complete auth bypass.

**2. Your business logic and access control do not apply.**
Your wrapper is where you enforce which users can do what—which groups they can join, which messages they can send, which users they can look up. If the client talks directly to OpenIM, none of that logic executes. A client can call any OpenIM API endpoint it knows about, with any parameters, and OpenIM will process the request according to its own rules—which know nothing about your application's policies.

**3. OpenIM's admin and internal APIs are exposed.**
OpenIM's API Gateway exposes endpoints beyond what a normal user client should ever reach—administrative operations, token management, user creation, and more. These are intended to be called by your backend service acting as a trusted intermediary, not by untrusted clients. Direct client access collapses the distinction between "client-facing API" and "backend-facing API" because there is only one API surface, and it is now reachable by anyone who can connect to it.

**4. Rate limiting, logging, and abuse prevention become impossible.**
Your wrapper is the natural place to enforce per-user rate limits, log all API calls for auditing, detect and block abusive patterns, and apply any other cross-cutting controls. None of this is possible if clients bypass it. OpenIM has its own rate limiting (the BBR middleware described in the latency document), but it operates on a different model—it does not know your users, your tiers, or your policies.

**5. You lose the ability to version or evolve the API surface clients see.**
When clients talk directly to OpenIM, they are coupled to OpenIM's API schema. If OpenIM changes an endpoint, renames a field, or deprecates a parameter, your clients break—and you have no opportunity to absorb or translate that change. Your wrapper is a versioning boundary: you can update OpenIM underneath it and keep the client-facing API stable, or roll out a new client API while keeping the OpenIM integration unchanged. Direct client access eliminates that flexibility permanently.

### Network-Level Enforcement

This constraint must be enforced at the network level, not just as a convention. It is not sufficient to tell clients not to connect directly to OpenIM—the connection must be made impossible. In practice this means:

- OpenIM's API Gateway and WebSocket endpoint should be on an **internal network or VPC** that is not reachable from the public internet.
- The only public-facing entry points are your auth gateway and, if necessary, a controlled proxy for the WebSocket connection.
- Firewall rules or security groups should explicitly deny inbound traffic to OpenIM's ports from anything other than your wrapper services.

If OpenIM is reachable from the internet, the architecture is not secure regardless of how well your wrapper is implemented—because the wrapper can simply be bypassed.

### Summary of What Clients Are Allowed to Connect To

| Connection Type | Allowed Direct Client Access? | Notes |
|---|---|---|
| OpenIM HTTP API Gateway | **No** | Must go through your wrapper |
| OpenIM internal gRPC services | **No** | Internal only, never client-facing |
| OpenIM WebSocket message gateway | **Yes (bounded exception)** | Credentials must be issued by your gateway; scope is limited to real-time messaging only |
| Your auth gateway | Yes | Primary entry point for all client requests |
| Your wrapper/backend service | Via your gateway only | Clients do not address your backend directly |

---

## Why This Flow Is Required

### 1. It Is the Pattern OpenIM Explicitly Intends

OpenIM's own architecture exposes a deliberate public surface—the HTTP API Gateway and gRPC service definitions—for exactly this purpose. The project separates [`internal/api`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api) (the externally-facing gateway) from [`internal/rpc`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc) (internal service logic) deliberately. Making the gateway the external integration point and keeping RPC services internal is not an accident—it is OpenIM's stated extension model. Bypassing it by patching internals means building against the grain of the project's own architecture decisions.

This mirrors a general principle in software design: libraries and platforms expose stable public APIs precisely because they expect those to be the integration point. Everything behind that boundary is subject to change without notice. OpenIM's gRPC service definitions and HTTP API are the contract; everything inside `internal/` is not.

### 2. Separation of Concerns Is a Load-Bearing Design Principle

The layered architecture here is not bureaucratic—it is doing real structural work:

- **Your gateway owns authentication.** This means auth logic has exactly one home. When you need to rotate a key, change your DH parameters, or swap JWT libraries, there is one place to change it, one place to test it, and one failure mode to reason about.
- **OpenIM owns messaging.** It handles user management, group state, message routing, and delivery guarantees. It does this well. Adding your auth logic inside OpenIM blurs this boundary and forces you to reason about both problem spaces simultaneously whenever something breaks.
- **Your wrapper owns translation.** It is the adapter between your world and OpenIM's. This is a well-understood design pattern—the Adapter pattern from the Gang of Four—applied at the service boundary level.

Martin Fowler's formulation of this principle is direct: a system where every component knows about every other component's internal details is a system where no component can be changed safely. Separation of concerns is what makes individual components replaceable, testable, and understandable in isolation.

### 3. Black-Box Integration Is How the Industry Treats Upstream Dependencies

The principle that you should not modify the internals of a dependency you do not own is foundational to how large-scale software systems are maintained. Google's internal engineering documentation (which shaped practices now widely adopted across the industry) treats modification of vendored third-party code as a last resort, not a design strategy. The reasoning is straightforward: every change you make to a third-party codebase is a change you must carry forward through every future upgrade. The cost is not paid once—it compounds.

Concretely, OpenIM releases updates that fix security vulnerabilities, add features, and improve performance. If you have modified `internal/api` or any RPC service, every one of those upgrades requires you to:

- Diff upstream changes against your fork
- Identify conflicts with your patches
- Re-apply, re-test, and re-validate your changes
- Do this again for the next release, and every release after that

This is not a hypothetical concern. It is the routine experience of every team that has maintained a fork of an active open-source project. The wrapper approach eliminates this entirely: OpenIM is upgraded like any other dependency, your integration layer is unchanged, and you validate the upgrade at the API boundary.

### 4. Security Updates Must Flow Freely

Security is perhaps the most concrete argument for the black-box approach. When OpenIM ships a security patch—for a vulnerability in its auth service, message routing, or any internal component—you need to be able to apply that patch immediately, with confidence that it changes only what it says it changes.

If you have patched OpenIM's internals, you cannot do this. Every security update from upstream is now a merge operation against your fork, where you must verify that the upstream fix does not conflict with your modifications. In practice, teams in this situation delay security updates, apply them incompletely, or skip them under time pressure. This is a real security risk, not a theoretical one.

The wrapper approach makes security updates trivially safe: pull the new OpenIM version, run your integration tests at the API boundary, deploy. Your code is untouched.

### 5. Community Support Depends on a Standard Deployment

The OpenIM community—its maintainers, issue tracker, documentation, and other users—operates on the assumption that your deployment matches the standard distribution. When you open an issue or ask a question, the people helping you are reasoning from a known baseline.

A modified OpenIM core invalidates that baseline. Problems that appear to be OpenIM bugs may be caused by your patches. OpenIM maintainers cannot reproduce your issues. Other users' experiences become irrelevant comparisons. Effectively, you have forked the project and taken on sole responsibility for supporting your version—without the resources of a dedicated team.

This is a soft cost that accumulates invisibly: slower debugging, less helpful community responses, documentation that doesn't quite match your system. Over a long enough timeline it is significant.

### 6. Testability and Auditability Are Concentrated Where They Belong

With a clean wrapper architecture, there is a precise contract to test: your wrapper must correctly translate your auth model into valid OpenIM API calls, and correctly translate OpenIM responses back to your clients. This contract can be validated with integration tests at the API boundary, without needing to instrument OpenIM's internals.

This also matters for audits. If you need to demonstrate to a security auditor, compliance officer, or external reviewer what your system does with user credentials and data, you can point to your gateway and your wrapper. The boundary is clean, the surface area is small, and the logic is entirely yours. OpenIM is a verified upstream dependency that you have not modified.

If your auth and business logic are scattered across a patched version of OpenIM's internals, that audit becomes an examination of the entire OpenIM codebase plus your modifications—a significantly larger and harder task.

---

## Conclusion

This flow is not the convenient choice—it is the correct one. The pattern of wrapping a black-box dependency at its public API boundary has decades of support in software engineering practice, from the Gang of Four Adapter pattern to modern API gateway architecture to Google's own internal dependency management principles.

Every alternative—patching OpenIM's internals, injecting custom middleware into its routing layer, modifying its RPC service definitions, or allowing clients to reach OpenIM directly—trades a short-term convenience for a long-term liability: auth bypass, policy enforcement gaps, upgrade friction, security update delays, and an ever-growing maintenance surface.

Two rules govern the client boundary:

1. **Clients never connect directly to OpenIM's HTTP API.** All requests go through your auth gateway and wrapper, where your authentication, access control, rate limiting, and logging live.
2. **The WebSocket connection is the only bounded exception.** It exists because real-time messaging requires it. It is not a precedent for general direct access, and even it must be gated by credentials your gateway issues.

Both rules must be enforced at the network level—not as convention, but as a hard infrastructure constraint.

The wrapper approach gives you:

- **A clean, auditable auth boundary** with a single place to reason about security
- **Freely flowing upstream updates**, including security patches, applied with confidence
- **Full community support** on a standard OpenIM deployment
- **Independent scalability** of your auth layer and OpenIM separately
- **A testable contract** at the API surface, not scattered through internal implementation details
- **Access control and rate limiting that cannot be bypassed**, because the bypass path does not exist at the network level

Deviating from this flow—either by putting custom logic inside OpenIM's codebase, or by allowing clients to reach OpenIM directly—is not a shortcut. It is the beginning of a maintenance and security debt that will be paid repeatedly, on every upgrade and in every incident, for as long as the system runs.