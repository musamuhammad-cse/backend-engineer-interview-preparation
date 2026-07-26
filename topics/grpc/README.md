# gRPC — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Chronos (Go distributed job scheduler using Raft) — gRPC is the natural communication layer for this project. Multi-tenant SaaS, trading platform, microservices architecture.

---

## How to use this material

This is not a skim-read. Each tier builds on the previous one and is designed for **active recall**, not passive reading.

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as to an interviewer | 20 min/section |
| 2 | Type out the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what fails senior loops | 5 min |

**The senior signal is not knowing definitions.** It's knowing trade-offs, failure modes, and what you'd do at 3am when it breaks. Every section below flags **Traps** (what interviewers use to catch you) and **Follow-ups** (the second and third question they will ask).

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | gRPC overview & motivation, Protocol Buffers (schema, types, compilation), HTTP/2 foundations, server/client implementation in Go, unary RPC, gRPC vs REST/graphQL, simple service patterns | 6–8 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Server streaming, client streaming, bidirectional streaming, gRPC error handling & status codes, deadlines & timeouts, metadata (headers), interceptor/middleware patterns (client & server), authentication (TLS, JWT), gRPC-web, advanced protobuf features | 10–12 hours |
| [`03-senior.md`](./03-senior.md) | Load balancing strategies (client-side, proxy, look-aside), connection management & keepalive, channel pooling & retry, reflection & gRPCurl, gRPC health checking & readiness, production deployment (K8s, Envoy, Linkerd), security (TLS mTLS, ALTS), streaming performance tuning, error budgets & circuit breaking, comparison to message queues, gRPC in Go with Chronos integration | 12–14 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 140+ interview questions, code puzzles, live-coding exercises, debugging scenarios, system design prompts, STAR stories | Ongoing drill |

---

## Coverage map

### gRPC fundamentals
- What is gRPC: RPC framework by Google, HTTP/2 transport, Protocol Buffers serialization, code generation
- Protocol Buffers: `.proto` files, scalar types, message structure, `repeated`, `map`, `oneof`, enums
- Service definition: `service` keyword, rpc methods, request/response messages
- Code generation: protoc, protoc-gen-go, protoc-gen-go-grpc, Buf CLI
- HTTP/2: multiplexing, binary framing, streams, server push, header compression (HPACK)
- Unary RPC: simple request-response, implementation in Go
- gRPC vs REST vs GraphQL: performance, schema strictness, streaming, ecosystem
- Protocol Buffers best practices: field numbers, backward/forward compatibility, naming conventions

### gRPC patterns
- Server-side streaming: multiple responses, when to use (monitoring, log tailing)
- Client-side streaming: multiple requests, when to use (file upload, batch)
- Bidirectional streaming: real-time, when to use (chat, collaborative editing)
- Deadlines & timeouts: per-RPC deadlines, context propagation, cancellation
- Metadata: request/response headers, authentication tokens, correlation IDs
- Error handling: gRPC status codes (OK, Canceled, Unknown, InvalidArgument, DeadlineExceeded, NotFound, etc.), rich error model with details
- Interceptors: server-side unary & stream interceptors, client-side unary & stream interceptors, logging, auth, metrics, retry
- Authentication: TLS (server cert, mutual TLS), JWT (metadata), OAuth 2.0 (Google's gRPC OAuth)
- gRPC-web: enabling gRPC from browsers, Envoy proxy, performance impact

### Advanced gRPC
- Load balancing: client-side balancing (pick_first, round_robin), proxy-based (Envoy, NGINX), DNS-based (headless services in K8s), look-aside (external LB)
- Connection management: keepalive (ping), max connection age, connection limits
- Retry & hedging: built-in retry policy (maxAttempts, initialBackoff, maxBackoff, retryableStatusCodes), hedging for latency-sensitive services
- Channel management: connection pooling, channel reuse, subchannel failing
- Reflection & debugging: gRPC reflection protocol, gRPCurl, gRPC UI, Evans CLI
- Health checking: gRPC Health Checking Protocol (grpc.health.v1), readiness probes in K8s
- Production deployment: K8s with gRPC (headless services, session persistence), Envoy sidecar, service mesh (Linkerd/Istio)
- Security: TLS, mutual TLS (mTLS), certificate rotation, ALTS (Google cloud), channel encryption vs payload encryption
- Flow control: HTTP/2 flow control, initial windows size, dynamic flow control, performance tuning for large messages
- Benchmarking: ghz for gRPC load testing, latency/throughput profiling
- gRPC in Go: `google.golang.org/grpc` package, `Server`, `ClientConn`, dial options, interceptors, reflection
- gRPC vs message queues: when to use each, event sourcing with gRPC, async vs sync communication

---

## Study order recommendation

gRPC is a core protocol for microservices communication. If Chronos is a Go distributed scheduler running on gRPC, treat this as a portfolio-strength topic.

```
Week 1:  01-basic.md     + Basic Q&A drill
Week 2:  02-intermediate.md + Intermediate Q&A drill
Week 3:  03-senior.md    + Senior Q&A drill
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** Databases (PostgreSQL, MySQL, indexing, transactions, isolation levels).
