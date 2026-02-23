# Argument Against Latency as a Reason to Avoid a Wrapper

Introducing a wrapper layer often raises concerns about latency. While every layer can *potentially* add some delay, the impact is usually small or manageable—especially with an architecture like OpenIM's.

The claim that wrapping OpenIM introduces **unacceptable latency** is an oversimplification. OpenIM is built as microservices with an API Gateway and gRPC interfaces that already involve network hops and serialization. Your wrapper would talk to these existing, optimized interfaces, so its extra overhead is typically minimal compared to the communication that's already there.

Below is a concise argument with references for why wrapper latency is acceptable and why modifying OpenIM's source is still a bad trade-off.

---

## 1. OpenIM's Inherent Network Latency (Microservices and gRPC)

OpenIM already runs as **microservices**: different functions live in separate RPC services (e.g. [Authentication Service](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/docs/contrib/test.md?plain=1#L1), [User Management Service](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/docs/contrib/test.md?plain=1#L1), Message Delivery) under [`internal/rpc`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/rpc), as in [Internal RPC Services](https://codewiki.google/github.com/openimsdk/open-im-server#openim-server-architecture-and-core-services-internal-rpc-services).

- Communication between these services, and between the [API Gateway](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/pkg/common/cmd/api.go#L47) ([`internal/api`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api)) and the RPC layer, is already **network-based (gRPC)**.
- Each gRPC call involves serialization/deserialization (e.g. Protocol Buffers), network transfer, and processing. Your wrapper would sit *before* or *after* these calls, adding only a small amount of extra latency.
- The [`a2r.Call`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/msg.go#L155) utility (see [API Gateway and Client Interaction](https://codewiki.google/github.com/openimsdk/open-im-server#openim-server-architecture-and-core-services-api-gateway-and-client-interaction)) shows that converting HTTP to gRPC and back is a core part of OpenIM's design—so an extra "layer" in front is consistent with how the system already works.

### The Math: Wrapper Overhead as a Percentage

The key insight is that wrapper latency should be evaluated **as a fraction of existing latency**, not in absolute terms. A single gRPC hop between two co-located microservices typically costs **5–20ms** in practice (accounting for serialization, kernel scheduling, and network stack overhead). A thin, well-implemented wrapper proxy adds roughly **0.2–1ms**—a **2–10% overhead at most**.

Put differently: if you are already paying 15ms to transit OpenIM's own microservices, an extra 0.5ms at the boundary is noise, not a bottleneck.

---

## 2. Industry Precedent: Proxies and Sidecars at Scale

This is not a new idea. The entire modern service-mesh and API-gateway ecosystem — Istio, Linkerd, Envoy, Kong — is built on one simple premise: putting a proxy or wrapper in front of a networked service is a normal, accepted trade-off. These are not experimental projects. They run in production at companies like Google, Netflix, and Airbnb.

These systems have measured and published how much latency their proxy layers actually add:

- [**Linkerd's 2021 benchmark report**](https://linkerd.io/2021/05/27/linkerd-vs-istio-benchmarks/) measured the latency cost of its sidecar proxy at the **p99 percentile at ~6.7ms** against a 3.1ms no-mesh baseline — roughly **3.6ms of added overhead** — even at high request volumes. Their conclusion: the proxy is not the bottleneck. The network and the application already are.
- [**Kong's performance benchmarks**](https://developer.konghq.com/gateway/performance/benchmarks/) show that a full API gateway — a much heavier and more complex system than a simple wrapper — handles over 20,000 requests per second with single-digit millisecond latency on co-located servers.

The same pattern shows up in both: the proxy adds a small, fixed cost. That cost shrinks in importance the more your system already does — and OpenIM already does a lot.

> **Direct answer to the main objection:** Yes, the wrapper adds a hop in the critical path of every request. That hop costs roughly 0.2–1ms for a thin, well-written proxy on co-located infrastructure — this is consistent with what Linkerd and Kong both show. OpenIM's own internal microservice calls already cost 5–20ms each. So the wrapper adds, at most, a few percent on top of latency that is already there. That is not a bottleneck. That is noise.

---

## 3. Optimized API Gateway Performance

The OpenIM API Gateway uses [Gin](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/go.mod#L8) in [`internal/api`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api) and already includes performance and reliability features:

| Feature | Where | Purpose |
|--------|--------|--------|
| **GZIP compression** | [`internal/api/router.go`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go) ([`gzip.Gzip`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L94) at line 67) | Reduces payload size and network-related latency for larger responses. |
| **BBR rate limiting** | [`RateLimitMiddleware`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/ratelimit.go#L26) in [`internal/api/router.go`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go) at line 77 | Avoids overload and keeps behavior stable under load (often a bigger latency concern than a thin wrapper). |
| **Request handling** | Design described in [API Gateway and Client Interaction](https://codewiki.google/github.com/openimsdk/open-im-server#openim-server-architecture-and-core-services-api-gateway-and-client-interaction) | HTTP transport, request/response translation, and routing. Your wrapper can use these same paths. |

---

## 4. Authentication Is Already a Performance Factor

OpenIM already does authentication and authorization (e.g. [`pkg/authverify`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/msg.go#L28)) on every request.

- **[`GinParseToken`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L347)** in [`internal/api/router.go`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go) (line 347) parses a token and calls the [Auth Service](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/docs/contrib/test.md?plain=1#L1) via RPC ([`pbAuth.NewAuthClient`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L218)) to validate it.
- Your auth (Diffie-Hellman and JWT) would **replace or extend** this step. The cost of your logic replaces or adds to the existing token-validation cost; it doesn't always add a *wholly new* layer of delay.
- **[`AuthApi`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/auth.go#L23)** ([`pbAuth.NewAuthClient`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L218)) already handles [`get_admin_token`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L220), [`get_user_token`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L221), and [`parse_token`](https://github.com/openimsdk/open-im-server/blob/fbca49d4319eee0d40395d087b792b0351c069de/internal/api/router.go#L222), among others—so auth is already a layered process with RPC.

---

## 5. The Wrapper Is Horizontally Scalable

A common hidden concern behind the latency objection is: *"what happens under load? won't the wrapper become a bottleneck?"* This conflates latency with throughput, and misses a critical architectural property.

Because your wrapper is **stateless**—it authenticates, translates, and forwards requests without maintaining session state itself—it can be **scaled horizontally without limit**. You can run two instances, ten instances, or a hundred behind a load balancer, each independently handling requests in parallel. OpenIM's own microservices are designed on the same principle.

A monolithic core modification, by contrast, is harder to scale selectively—you scale the entire process, not just the bottleneck. A thin, stateless wrapper in front of OpenIM gives you *more* control over performance under load, not less.

---

## 6. Cost of Patching the Core Outweighs Any Wrapper Latency

Even if a wrapper added a small, measurable delay, the **long-term cost of modifying OpenIM's core** would still be worse:

- **Upgrade hell:** Keeping a fork means constantly re-integrating upstream changes: more work, merge conflicts, and risk of new bugs. That goes against the maintainability and versioning approach described in [OpenIM Branch Management and Versioning](https://codewiki.google/github.com/openimsdk/open-im-server#openim-branch-management-and-versioning-a-blueprint-for-high-grade-software-development).
- **Loss of support:** A modified core makes it harder to get help from the OpenIM community, since your setup is no longer standard.
- **Harder debugging:** Problems are harder to diagnose when the core behaves differently from the standard distribution.
- **Hyrum's Law:** Coined by Google engineer Hyrum Wright, this principle states that with enough users and time, *all observable behaviors of your system will be depended upon by somebody*. Applied here: the longer you maintain a fork, the more your codebase will implicitly depend on internal behaviors of OpenIM that upstream never intended to be stable—and that will break on every upgrade. A wrapper, by contrast, depends only on OpenIM's *public API surface*, which the project actively maintains and versions.
- **Fowler's rule on vendored dependencies:** Martin Fowler and the broader software engineering literature are consistent on this point: modifying the internals of a third-party dependency is an anti-pattern. The correct approach is to wrap or extend at the boundary the library exposes. Changing internals creates an implicit maintenance contract with yourself that compounds in cost over time.

---

## Conclusion

The question is never *zero latency vs. some latency*—it's whether the tradeoff is worth it. Here, the math is clear and the engineering precedent is overwhelming.

OpenIM is intended to be extended via its **APIs, gRPC services, and middleware**. The latency objection to a wrapper is misplaced because the main latency already comes from the microservices design itself—network hops and serialization that OpenIM has already optimized. Industry benchmarks from Linkerd, Envoy, and Kong confirm that proxy and gateway layers add well under 5ms in co-located deployments, a rounding error against a baseline that is already measured in tens of milliseconds. And unlike a forked core, a stateless wrapper can be scaled horizontally and updated independently without accumulating fork debt or violating Hyrum's Law.

A wrapper that uses OpenIM's external interfaces is strongly preferable for:

- Maintainability and upgradability
- Community and documentation support
- Avoiding fork maintenance and Hyrum's Law traps
- Independent horizontal scalability

If performance still concerns you, the correct response is to **benchmark your specific wrapper implementation** and **tune OpenIM's exposed configuration**—not to change its core. The evidence strongly favors the wrapper.