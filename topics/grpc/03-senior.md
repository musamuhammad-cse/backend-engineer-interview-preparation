# gRPC — Tier 3: Senior (Production Operations & Architecture)

> **Target:** Senior Backend Engineer (8+ years, Go, PHP, JS)  
> **Focus:** Load balancing, connection management, retry/hedging, reflection, health checking, K8s deployment, security, streaming performance, message queues, production Go patterns  
> **Prerequisites:** gRPC Tier 1 (basic) and Tier 2 (intermediate) knowledge assumed  
> **Your anchors:** Chronos (Go distributed job scheduler using Raft) — gRPC is the communication layer for scheduling commands, job status streaming, and cluster management

At the senior level, you don't just build gRPC services — you run them in production at scale. You design for load balancing without connection affinity killing throughput, you handle zombie connections with keepalive, you decide between retry and hedging, and you integrate gRPC deeply into K8s, service meshes, and observability stacks. Every trap here is a real incident that has happened at companies running gRPC in production.

Every section includes **Traps** (what interviewers watch for) and **Follow-ups** (the second and third questions they ask). Code examples are in Go using `google.golang.org/grpc`.

---

## Table of Contents

1. [Load Balancing Strategies](#1-load-balancing-strategies)
2. [Connection Management & Keepalive](#2-connection-management--keepalive)
3. [Retry & Hedging](#3-retry--hedging)
4. [Reflection & gRPCurl](#4-reflection--grpcurl)
5. [Health Checking & Readiness](#5-health-checking--readiness)
6. [Production Deployment (K8s & Service Mesh)](#6-production-deployment-k8s--service-mesh)
7. [Security — mTLS & Beyond](#7-security--mtls--beyond)
8. [Streaming Performance Tuning](#8-streaming-performance-tuning)
9. [gRPC vs Message Queues](#9-grpc-vs-message-queues)
10. [gRPC in Go — Production Patterns](#10-grpc-in-go--production-patterns)
11. [Tier 3 Q&A Drill](#11-tier-3-qa-drill)

---

## 1. Load Balancing Strategies

### Why gRPC Load Balancing Is Different

gRPC uses **long-lived HTTP/2 connections** with multiplexed streams. A single TCP connection carries many concurrent RPCs. This breaks traditional L4 load balancing:

- **Round-robin TCP** — an L4 balancer distributes TCP connections. Each client opens one gRPC channel = one TCP connection = one backend. All RPCs go to that backend. No balancing happens at the RPC level.
- **Sticky sessions** — the balancer pins a client to one backend. If that backend is overloaded, the client keeps sending RPCs to it.
- **No per-RPC distribution** — L4 can't see gRPC requests (they're binary on a single stream). L7 can, but most simple load balancers don't speak HTTP/2.

```
// Problem: L4 round-robin
Client → TCP Conn → [LB] → Backend A (all RPCs pinned here)
                            Backend B (idle)
                            Backend C (idle)

// Solution: Client-side or L7 load balancing
Client → [pick_first/round_robin resolver]
         ├── Backend A (subchannel 1)
         ├── Backend B (subchannel 2)
         └── Backend C (subchannel 3)
```

### Strategy 1: Client-Side Load Balancing

gRPC has built-in client-side load balancing via the `resolver` and `balancer` architecture:

- **Resolver** — resolves a target string (e.g., `dns:///service.example.com`) to a list of addresses
- **Balancer** — creates subchannels (one per address) and picks one per RPC

**Built-in policies:**

| Policy | Behavior |
|--------|----------|
| `pick_first` (default) | Connect to first resolved address. If it fails, try the next. One subchannel at a time. |
| `round_robin` | Connect to every resolved address. Distribute RPCs across subchannels in rotation. |

**Using round_robin:**

```go
import "google.golang.org/grpc/balancer/roundrobin"

conn, err := grpc.Dial(
    "dns:///service.example.com:8080",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin": {}}]}`),
    grpc.WithInsecure(), // for dev only
)
```

**Custom resolvers** — DNS is built-in, but you can write resolvers for Consul, etcd, or custom service discovery:

```go
import "google.golang.org/grpc/resolver"

type consulResolver struct {
    target resolver.Target
    cc      resolver.ClientConn
    stopCh  chan struct{}
}

func (r *consulResolver) ResolveNow(o resolver.ResolveNowOptions) {
    // Trigger re-resolution (called on subchannel failure)
    go r.resolve()
}

func (r *consulResolver) resolve() {
    addrs := consulLookup(r.target.Endpoint)
    var state resolver.State
    for _, addr := range addrs {
        state.Addresses = append(state.Addresses, resolver.Address{Addr: addr})
    }
    r.cc.UpdateState(state)
}

// Register the custom resolver scheme
func init() {
    resolver.Register(&consulResolverBuilder{})
}
```

**gRPC-LB protocol** — for environments where you can't do client-side LB (e.g., mobile clients that shouldn't know about backends), use an external load balancer service. The gRPC-LB protocol lets a client ask an LB for backend addresses:

```
Client → [gRPC-LB Service] → returns backend list
       → connects directly to backend A
```

This is rarely used directly; most teams use Envoy or a service mesh instead.

### Strategy 2: Proxy-Based (Envoy, NGINX, HAProxy)

An L7 proxy terminates the client's HTTP/2 connection and opens new connections to backends. Each RPC can go to a different backend.

**Envoy sidecar config (gRPC):**

```yaml
static_resources:
  listeners:
  - name: grpc_listener
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: grpc_json
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route:
                  cluster: grpc_backend
                  timeout: 0s
          http_filters:
          - name: envoy.filters.http.router
  clusters:
  - name: grpc_backend
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: grpc_backend
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: backend-a, port_value: 9090 }
        - endpoint:
            address:
              socket_address: { address: backend-b, port_value: 9090 }
    health_checks:
    - timeout: 1s
      interval: 10s
      grpc_health_check: {}
```

Key: `type: STRICT_DNS` (not STATIC) — Envoy re-resolves DNS periodically. `lb_policy: ROUND_ROBIN` for per-RPC balancing. `grpc_health_check: {}` to use gRPC native health checking.

### Strategy 3: DNS-Based (Headless Services in K8s)

A **headless Service** in K8s (`.spec.clusterIP: "None"`) returns pod IPs directly via DNS A/AAAA records instead of a single ClusterIP. gRPC's DNS resolver picks up all pod IPs.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: chronos-scheduler
spec:
  clusterIP: None  # headless
  selector:
    app: chronos-scheduler
  ports:
  - port: 9090
    targetPort: grpc
```

Client connects with:

```go
conn, err := grpc.Dial(
    "dns:///chronos-scheduler.default.svc.cluster.local:9090",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin": {}}]}`),
)
```

The DNS resolver periodically re-resolves to pick up new or removed pods. Use `grpc.WithConnectParams()` to tune re-resolution frequency.

### Architecture Comparison

```
┌─────────────────────────────┬──────────────────┬──────────────────┬───────────────────────┐
│ Strategy                    │ Per-RPC balance? │ Client change?   │ Best for              │
├─────────────────────────────┼──────────────────┼──────────────────┼───────────────────────┤
│ Client-side (round_robin)   │ Yes              │ Yes (grpc.Dial)  │ Go services, internal  │
│ Proxy-based (Envoy)         │ Yes              │ No               │ Polyglot, external     │
│ DNS-based (headless)        │ Yes              │ Yes (DNS target) │ K8s-native, simple     │
│ gRPC-LB protocol            │ Yes              │ Yes              │ Mobile, external LB    │
│ L4 TCP (ClusterIP)          │ No               │ No               │ Will NOT work          │
└─────────────────────────────┴──────────────────┴──────────────────┴───────────────────────┘
```

> **Trap:** Using a regular K8s Service (ClusterIP) for gRPC — the ClusterIP does L4 TCP load balancing, which pins each gRPC client to one backend via the long-lived HTTP/2 connection. All RPCs from that client go to the same pod. The balancer appears to work (connections distribute) but RPCs don't. Always use a headless Service or an L7 proxy.

> **Follow-up:** "How do you handle the case where your gRPC client can't do client-side load balancing (e.g., a PHP or JS client)?" — Use an L7 proxy. Envoy as a sidecar or gateway terminates the HTTP/2 connection and balances per-RPC. The client connects to Envoy, Envoy handles backend selection. NGINX ≥1.13.10 with `grpc_pass` also works.

> **Trap:** Assuming client-side load balancing works without a proper resolver — `grpc.Dial("host:port")` with a fixed address always uses `pick_first` (one backend). You must use a scheme like `dns:///` or a custom resolver for `round_robin` to have multiple backends to balance across.

> **Trap:** Not configuring keepalive alongside client-side balancing — if a backend goes down, the subchannel stays connected (TCP doesn't detect the failure). RPCs fail with `Unavailable` but the subchannel is never removed. Keepalive detects dead connections and triggers re-resolution.

---

## 2. Connection Management & Keepalive

### HTTP/2 Connection Pooling

A `grpc.ClientConn` maintains one HTTP/2 TCP connection per backend address (subchannel). All RPCs on the same channel are multiplexed over that single connection.

```
ClientConn (channel)
├── Subchannel → TCP conn → Backend A (port 9090)
├── Subchannel → TCP conn → Backend B (port 9090)
└── Subchannel → TCP conn → Backend C (port 9090)
```

**Channel configuration options:**

```go
conn, err := grpc.Dial(
    "dns:///chronos-scheduler:9090",
    grpc.WithDefaultServiceConfig(`{
        "loadBalancingConfig": [{"round_robin": {}}],
        "methodConfig": [{
            "name": [{"service": "chronos.Scheduler"}],
            "retryPolicy": {
                "maxAttempts": 3,
                "initialBackoff": "0.1s",
                "maxBackoff": "1s",
                "backoffMultiplier": 2,
                "retryableStatusCodes": ["UNAVAILABLE"]
            }
        }]
    }`),
    grpc.WithConnectParams(grpc.ConnectParams{
        MinConnectTimeout: 5 * time.Second,
    }),
    grpc.WithUserAgent("chronos-client/1.0"),
)
```

### Keepalive — Client Side

```go
import "google.golang.org/grpc/keepalive"

conn, err := grpc.Dial(
    "dns:///chronos-scheduler:9090",
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                30 * time.Second, // ping every 30s of inactivity
        Timeout:             10 * time.Second, // wait 10s for ping ack
        PermitWithoutStream: true,             // ping even without active RPCs
    }),
)
```

### Keepalive — Server Side

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/keepalive"
)

var kaep = keepalive.EnforcementPolicy{
    MinTime:             5 * time.Second, // client can ping at most every 5s
    PermitWithoutStream: false,            // don't allow pings without RPCs
}

var kasp = keepalive.ServerParameters{
    MaxConnectionIdle:     15 * time.Minute, // close idle connections after 15m
    MaxConnectionAge:      30 * time.Minute, // force reconnect every 30m
    MaxConnectionAgeGrace: 5 * time.Second,  // grace period for in-flight RPCs
    Time:                  30 * time.Second, // server ping interval
    Timeout:               5 * time.Second,  // ping timeout
}

s := grpc.NewServer(grpc.KeepaliveParams(kasp), grpc.KeepaliveEnforcementPolicy(kaep))
```

### Why Keepalive Matters

1. **Detect dead connections** — TCP doesn't always detect a crashed backend. Keepalive pings detect unresponsive peers (process killed, network partition).
2. **Rebalance connections** — `MaxConnectionAge` forces periodic reconnection. After 30 minutes, the client disconnects and reconnects (potentially to a different backend). This distributes load evenly across pods over time.
3. **Prevent firewall timeouts** — AWS NLB, GCP LB, and corporate firewalls may kill idle TCP connections after 5-15 minutes. Keepalive pings keep the connection alive.
4. **Resource cleanup** — `MaxConnectionIdle` closes connections with no active RPCs, freeing server memory.

### Channel Pooling for High Throughput

A single gRPC channel can handle thousands of RPCs per second (HTTP/2 multiplexing). But in extreme throughput scenarios, or when using blocking I/O, multiple channels can improve throughput:

```go
type ChannelPool struct {
    channels []*grpc.ClientConn
    next     uint64
    mu       sync.Mutex
}

func NewChannelPool(n int, target string, opts ...grpc.DialOption) *ChannelPool {
    pool := &ChannelPool{}
    for i := 0; i < n; i++ {
        conn, err := grpc.Dial(target, opts...)
        if err != nil {
            log.Fatalf("failed to dial: %v", err)
        }
        pool.channels = append(pool.channels, conn)
    }
    return pool
}

func (p *ChannelPool) Get() *grpc.ClientConn {
    p.mu.Lock()
    idx := atomic.AddUint64(&p.next, 1) % uint64(len(p.channels))
    p.mu.Unlock()
    return p.channels[idx]
}

// Usage: pick channel per goroutine
pool := NewChannelPool(4, "dns:///chronos-scheduler:9090", opts)
pool.Get().Invoke(ctx, "/chronos.Scheduler/ScheduleJob", req, resp)
```

> **Trap:** Not setting any keepalive — zombie connections accumulate. A backend goes down, the client doesn't know for hours. RPCs fail with `context deadline exceeded` instead of fast `Unavailable`. All retries target the same dead backend.

> **Trap:** Setting keepalive too aggressive — a client sending pings every 1s with `PermitWithoutStream: true` while idle can overwhelm the server. The server sends `GOAWAY` with `ENHANCE_YOUR_CALM` and may close the connection. Respect server `MinTime`.

> **Trap:** Not setting `PermitWithoutStream: false` on the server — if a client establishes a connection and sends no RPCs and no pings, the server can't tell if it's alive. With `PermitWithoutStream: true`, the server allows pings even without RPCs, which prevents idle timeout from cleaning up.

> **Trap:** Re-creating channels for every RPC — `grpc.Dial` is expensive: TCP handshake, TLS handshake, HTTP/2 preface, resolver setup. Always reuse the connection. Create a channel once at application startup and reuse it.

> **Follow-up:** "How do you handle connection backoff after repeated failures?" — gRPC has built-in exponential backoff: `grpc.WithBackoffMaxDelay(5 * time.Second)`. Default: initial 1s, max 120s, multiplier 1.6, jitter ±20%. On connection failure, the client backs off before retrying. Configure with `ConnectParams`.

### Connection Backoff

```go
conn, err := grpc.Dial(
    "dns:///chronos-scheduler:9090",
    grpc.WithConnectParams(grpc.ConnectParams{
        MinConnectTimeout: 5 * time.Second,
    }),
    grpc.WithBackoffMaxDelay(30 * time.Second),
)
```

Default backoff sequence: 1s, 1.6s, 2.56s, 4.1s, 6.55s, ... 120s (capped). After reaching the cap, it retries every 120s indefinitely. This prevents a thundering herd when services restart.

---

## 3. Retry & Hedging

### Built-in Retry Policy

gRPC supports a built-in retry mechanism configured via **service config** (JSON). No interceptor needed.

```json
{
  "methodConfig": [{
    "name": [{"service": "chronos.Scheduler"}],
    "retryPolicy": {
      "maxAttempts": 4,
      "initialBackoff": "0.1s",
      "maxBackoff": "1s",
      "backoffMultiplier": 2,
      "retryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED", "ABORTED"]
    }
  }]
}
```

**Fields:**

| Field | Description | Default |
|-------|-------------|---------|
| `maxAttempts` | Total attempts (including original) | 1 (no retry) |
| `initialBackoff` | Delay before first retry | Random [0, initialBackoff) |
| `maxBackoff` | Caps the exponential backoff | |
| `backoffMultiplier` | Multiplier for exponential backoff | 2 |
| `retryableStatusCodes` | Which codes trigger a retry | |

**Enabling retry at Dial:**

```go
import "google.golang.org/grpc"

conn, err := grpc.Dial(
    "dns:///chronos-scheduler:9090",
    grpc.WithDefaultServiceConfig(`{
        "loadBalancingConfig": [{"round_robin": {}}],
        "methodConfig": [{
            "name": [{"service": "chronos.Scheduler"}],
            "retryPolicy": {
                "maxAttempts": 3,
                "initialBackoff": "0.1s",
                "maxBackoff": "1s",
                "backoffMultiplier": 2,
                "retryableStatusCodes": ["UNAVAILABLE"]
            }
        }]
    }`),
)
```

### Hedging

Hedging sends **multiple identical requests simultaneously** and uses the first successful response, cancelling the rest.

```json
{
  "methodConfig": [{
    "name": [{"service": "chronos.Scheduler"}],
    "hedgingPolicy": {
      "maxAttempts": 3,
      "hedgingDelay": "0.05s",
      "nonFatalStatusCodes": ["UNAVAILABLE"]
    }
  }]
}
```

**Fields:**

| Field | Description |
|-------|-------------|
| `maxAttempts` | Max concurrent requests (including original) |
| `hedgingDelay` | Delay before sending each subsequent hedge (staggered) |
| `nonFatalStatusCodes` | Codes that don't stop hedging (others are returned immediately) |

**Tracking hedged requests server-side:**

```go
import "google.golang.org/grpc/peer"

func (s *SchedulerServer) ScheduleJob(ctx context.Context, req *pb.ScheduleJobRequest) (*pb.Job, error) {
    p, ok := peer.FromContext(ctx)
    if ok {
        log.Printf("hedged request from %s", p.Addr.String())
    }
    // Process normally — the server can't tell if it's a hedge
    // (unless you add a custom header in client interceptor)
}
```

### Retry vs Hedging — Trade-offs

```
┌───────────────┬──────────────────────┬──────────────────────────┐
│               │ Retry                │ Hedging                  │
├───────────────┼──────────────────────┼──────────────────────────┤
│ Latency       │ Adds backoff delay   │ Lowest latency (first    │
│               │                      │ response wins)           │
│ Server load   │ Lower (one at a time)│ Higher (N concurrent     │
│               │                      │ requests)                │
│ Best for      │ Idempotent, transient│ Latency-sensitive reads, │
│               │ failures             │ tail latency reduction   │
│ Worst for     │ Latency-sensitive    │ Write operations         │
│               │ operations           │ (duplicate mutations)    │
│ Implementation│ Built-in service     │ Built-in service config  │
│               │ config               │                          │
└───────────────┴──────────────────────┴──────────────────────────┘
```

### Custom Retry Interceptor

When you need conditional retry logic (e.g., retry only if the request is idempotent, log each attempt):

```go
func UnaryClientRetryInterceptor(maxRetries int) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{},
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {

        var err error
        for attempt := 0; attempt <= maxRetries; attempt++ {
            if attempt > 0 {
                // Exponential backoff
                delay := time.Duration(math.Pow(2, float64(attempt))) * 100 * time.Millisecond
                select {
                case <-time.After(delay):
                case <-ctx.Done():
                    return ctx.Err()
                }
            }

            err = invoker(ctx, method, req, reply, cc, opts...)
            if err == nil {
                return nil
            }

            st, ok := status.FromError(err)
            if !ok || !isRetryableCode(st.Code()) {
                return err // non-retryable
            }

            log.Printf("retry attempt %d/%d for %s: %v", attempt+1, maxRetries, method, err)
        }
        return err
    }
}
```

> **Trap:** Retrying non-idempotent operations — if `ScheduleJob` creates a job and the request is retried after a timeout (but the server actually processed it), you create duplicate jobs. Use idempotency keys or only retry on idempotent methods.

> **Follow-up:** "How do you make a gRPC mutation idempotent?" — Add an `idempotency_key` field to the proto request. The server checks if it has seen this key (Redis or DB), and if so, returns the previous response instead of creating a duplicate. This allows safe retry of any operation.

> **Trap:** Retrying on non-retryable codes — `INVALID_ARGUMENT`, `PERMISSION_DENIED`, `NOT_FOUND` will never succeed on retry. Only retry `UNAVAILABLE`, `DEADLINE_EXCEEDED` (if deadline is reset), `ABORTED` (concurrent transaction conflict), and `RESOURCE_EXHAUSTED` (if backoff).

> **Trap:** Too many retry attempts — cascading failures. If the backend is overloaded, every client retrying 5 times multiplies the load 5x. Use `maxAttempts: 3` with exponential backoff. Add jitter. Monitor retry rate.

> **Trap:** Hedging non-idempotent writes — two `ScheduleJob` requests both create a job. Even if one is cancelled, the server may have already committed. Never hedge writes without idempotency keys.

---

## 4. Reflection & gRPCurl

### gRPC Reflection Protocol

The reflection protocol allows a client to discover services, methods, and message types at runtime — without a compiled proto file or generated client.

**Enabling reflection on the server:**

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
)

func main() {
    s := grpc.NewServer()
    pb.RegisterSchedulerServer(s, &schedulerServer{})
    reflection.Register(s)

    lis, _ := net.Listen("tcp", ":9090")
    s.Serve(lis)
}
```

The server now exposes `grpc.reflection.v1alpha.ServerReflection` service. Any gRPC client can call `ListServices`, `FileContainingSymbol`, etc.

### gRPCurl

gRPCurl is the curl equivalent for gRPC:

```bash
# List all services
grpcurl -plaintext localhost:9090 list

# List methods on a service
grpcurl -plaintext localhost:9090 list chronos.Scheduler

# Describe a message type
grpcurl -plaintext localhost:9090 describe chronos.ScheduleJobRequest

# Invoke a unary RPC
grpcurl -plaintext -d '{"job_name": "nightly-cleanup", "cron": "0 2 * * *"}' \
    localhost:9090 chronos.Scheduler/ScheduleJob

# Invoke with TLS
grpcurl -cacert ca.pem -cert client.pem -key client-key.pem \
    localhost:9090 chronos.Scheduler/ScheduleJob

# Stream RPC (watch jobs)
grpcurl -plaintext -d '{}' localhost:9090 chronos.Scheduler/WatchJobs
```

**Common use cases:**

| Use | Command |
|------|---------|
| List all services | `grpcurl localhost:9090 list` |
| Show proto file | `grpcurl localhost:9090 describe .chronos.Scheduler` |
| Call with JSON file | `grpcurl -d @ localhost:9090 svc/method < input.json` |
| Add metadata header | `grpcurl -H "authorization: Bearer token" -H "x-request-id: abc"` |
| Output format | `-format json` (default), `-format text` (protobuf text) |

### Other Tools

**gRPC-ui** — web UI that serves a GraphQL-like explorer using reflection. Run alongside your server to provide a GUI for debugging.

```bash
grpcui -plaintext localhost:9090
# Opens browser at http://localhost:8080
```

**Evans CLI** — interactive gRPC REPL with auto-completion:

```bash
evans -r --host localhost --port 9090

> package chronos
> service Scheduler
> call ScheduleJob
job_name (string): nightly-cleanup
cron (string): 0 2 * * *
{
  "job_id": "job_abc123"
}
```

### Debugging with Reflection

1. **Verify the server is serving the correct proto** — after a deployment, run `grpcurl list` to confirm the right services are registered.
2. **Test a specific RPC during incident** — if the client reports errors, call the same RPC via grpcurl to isolate client vs server.
3. **Inspect message shapes** — `grpcurl describe` shows field numbers, types, and annotations. Use during code review of proto changes.
4. **Health check** — invoke `grpc.health.v1.Health/Check` directly.

> **Trap:** Enabling reflection in production without access control — anyone with network access can discover all your services, methods, and message schemas. This is a security risk (information disclosure). Solutions: bind reflection to a separate admin port, or wrap it in an auth interceptor.

> **Follow-up:** "How do you safely expose reflection in production?" — Run a separate admin server on an internal-only port (not exposed via ingress). Or add an interceptor that checks for an admin token before allowing reflection calls.

```go
// Admin server on internal port — reflection and health only
func startAdminServer() {
    lis, _ := net.Listen("tcp", ":9091")
    s := grpc.NewServer(
        grpc.ChainUnaryInterceptor(adminAuthInterceptor),
    )
    reflection.Register(s)
    grpc_health_v1.RegisterHealthServer(s, healthServer)
    s.Serve(lis)
}

func adminAuthInterceptor(ctx context.Context, req interface{},
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    // Only allow internal IPs or require admin token
    if md, ok := metadata.FromIncomingContext(ctx); ok {
        if md["x-admin-token"][0] == os.Getenv("ADMIN_TOKEN") {
            return handler(ctx, req)
        }
    }
    return nil, status.Error(codes.PermissionDenied, "admin access required")
}
```

> **Trap:** Not including reflection for development — debugging gRPC without reflection means you need the exact `.proto` file and protoc to generate a client. Reflection makes development and debugging dramatically faster. Always enable it in staging and dev.

> **Trap:** grpcurl with large messages — grpcurl loads the entire request/response into memory. For large streaming payloads, pipe from a file or use a client that supports streaming. Use `-d @` to read from stdin.

---

## 5. Health Checking & Readiness

### gRPC Health Checking Protocol

`grpc.health.v1.Health` is the standard gRPC health checking protocol. It defines two RPCs:

```protobuf
service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}

message HealthCheckRequest {
  string service = 0;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;
  }
  ServingStatus status = 1;
}
```

### Server Implementation

```go
import (
    "google.golang.org/grpc/health"
    "google.golang.org/grpc/health/grpc_health_v1"
)

// Create health server
healthServer := health.NewServer()

// Register with gRPC server
grpc_health_v1.RegisterHealthServer(grpcServer, healthServer)

// Set initial status (NOT_SERVING until initialization is complete)
healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_NOT_SERVING)

// After initialization (DB connected, Raft cluster joined)
healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_SERVING)

// Set overall server status (affects Check("") — the empty string query)
healthServer.SetServingStatus("", grpc_health_v1.HealthCheckResponse_SERVING)
```

**Per-service health** — set status for each service individually:

```go
// Dependency-aware: mark Scheduler as NOT_SERVING if the Raft cluster is unhealthy
func (s *SchedulerServer) healthCheckLoop() {
    ticker := time.NewTicker(10 * time.Second)
    for range ticker.C {
        if s.raftCluster.IsHealthy() {
            healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_SERVING)
        } else {
            healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_NOT_SERVING)
        }
    }
}
```

### Watch Method for Streaming Health

Clients can subscribe to health changes:

```go
client := grpc_health_v1.NewHealthClient(conn)
stream, err := client.Watch(ctx, &grpc_health_v1.HealthCheckRequest{Service: "chronos.Scheduler"})
for {
    resp, err := stream.Recv()
    if err != nil {
        break
    }
    if resp.Status == grpc_health_v1.HealthCheckResponse_SERVING {
        log.Println("Scheduler is healthy — start sending requests")
    } else {
        log.Println("Scheduler is unhealthy — stop sending requests")
    }
}
```

### K8s Integration

K8s 1.24+ supports **gRPC readiness probes** natively:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-scheduler
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chronos-scheduler
  template:
    metadata:
      labels:
        app: chronos-scheduler
    spec:
      containers:
      - name: scheduler
        image: chronos-scheduler:latest
        ports:
        - containerPort: 9090
          name: grpc
        - containerPort: 9091
          name: admin
        readinessProbe:
          grpc:
            port: 9090
            service: chronos.Scheduler
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          grpc:
            port: 9090
            service: chronos.Scheduler
          initialDelaySeconds: 30
          periodSeconds: 30
```

**Before K8s 1.24** — use an exec probe with `grpc_health_probe`:

```yaml
readinessProbe:
  exec:
    command:
    - /grpc-health-probe
    - -addr=:9090
    - -service=chronos.Scheduler
  initialDelaySeconds: 5
```

### Dependency-Aware Health

A service should report `NOT_SERVING` when its dependencies are down:

```go
type SchedulerServer struct {
    healthServer *health.Server
    db           *sql.DB
    redis        *redis.Client
    raft         *raft.Raft
}

func (s *SchedulerServer) checkDependencies() {
    if err := s.db.Ping(); err != nil {
        s.healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_NOT_SERVING)
        return
    }
    if err := s.redis.Ping(); err != nil {
        s.healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_NOT_SERVING)
        return
    }
    if s.raft.State() != raft.Leader && s.raft.State() != raft.Follower {
        s.healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_NOT_SERVING)
        return
    }
    s.healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_SERVING)
}
```

### Admin Server for Health and Reflection

Separate internal port for health checking and debugging, not exposed to external traffic:

```go
func main() {
    // Main gRPC server (external traffic)
    grpcServer := grpc.NewServer()
    pb.RegisterSchedulerServer(grpcServer, schedulerServer)

    // Admin server (internal: health + reflection)
    adminServer := grpc.NewServer(
        grpc.ChainUnaryInterceptor(adminIPRestrictionInterceptor),
    )
    healthServer := health.NewServer()
    grpc_health_v1.RegisterHealthServer(adminServer, healthServer)
    reflection.Register(adminServer)

    // Start both
    go grpcServer.Serve(mustListen(":9090"))
    go adminServer.Serve(mustListen(":9091"))

    // Set serving status after init
    healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_SERVING)
}
```

> **Trap:** Returning `SERVING` before the service is ready — the K8s readiness probe marks the pod as ready, traffic starts flowing, but the service hasn't finished initialization (DB migration, warm-up cache, Raft cluster join). Race condition causes request failures. Set `NOT_SERVING` on startup, then switch to `SERVING` after initialization.

> **Follow-up:** "How long should the initial delay be?" — Depends on initialization time. For Chronos: Raft cluster join can take 1-5s. Use `initialDelaySeconds: 5`, but also make the health check loop wait for actual readiness rather than a fixed delay. Return `NOT_SERVING` until all deps are confirmed ready.

> **Trap:** Not checking dependencies in the health check — the server returns `SERVING` even though Redis is down and all RPCs that hit the cache layer fail. K8s keeps the pod in service. Always check critical dependencies.

> **Trap:** Using HTTP health check for gRPC service — an HTTP health probe hitting `/healthz` on a gRPC port returns nothing (gRPC doesn't speak HTTP/1.1). The probe fails or passes for the wrong reasons. Always use gRPC-specific health probes.

> **Trap:** Not implementing per-service health — `health.Check("")` (empty service) checks the overall server status. But a multi-service server might have one broken service and three healthy ones. The broken service should report `NOT_SERVING` while the others remain `SERVING`. This allows partial traffic routing.

---

## 6. Production Deployment (K8s & Service Mesh)

### K8s for gRPC — Deployment + Headless Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-scheduler
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chronos-scheduler
  template:
    metadata:
      labels:
        app: chronos-scheduler
    spec:
      containers:
      - name: scheduler
        image: chronos-scheduler:latest
        ports:
        - containerPort: 9090
          name: grpc
        - containerPort: 9091
          name: admin
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"     # protobuf parsing is memory-intensive
            cpu: "1"
        readinessProbe:
          grpc:
            port: 9090
            service: chronos.Scheduler
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          grpc:
            port: 9090
            service: chronos.Scheduler
          initialDelaySeconds: 30
          periodSeconds: 30
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "15"]  # wait for K8s to remove endpoints
---
apiVersion: v1
kind: Service
metadata:
  name: chronos-scheduler
spec:
  clusterIP: None         # headless — DNS returns pod IPs directly
  selector:
    app: chronos-scheduler
  ports:
  - port: 9090
    name: grpc
    targetPort: 9090
```

**Client uses DNS resolver:**

```go
conn, err := grpc.Dial(
    "dns:///chronos-scheduler.default.svc.cluster.local:9090",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin": {}}]}`),
)
```

### Graceful Shutdown in K8s

K8s sends `SIGTERM` to the pod. The gRPC server must stop gracefully:

```go
func main() {
    lis, _ := net.Listen("tcp", ":9090")
    s := grpc.NewServer()
    pb.RegisterSchedulerServer(s, &schedulerServer{})

    // Mark as NOT_SERVING before shutdown
    healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_NOT_SERVING)

    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
        <-sigCh

        // Stop accepting new connections — finish in-flight RPCs within grace period
        stopped := make(chan struct{})
        go func() {
            s.GracefulStop()
            close(stopped)
        }()

        select {
        case <-stopped:
        case <-time.After(30 * time.Second):
            s.Stop() // force stop after grace period
        }
    }()

    s.Serve(lis)
}
```

### Envoy Sidecar

Envoy as a sidecar container in the same pod:

```yaml
spec:
  containers:
  - name: envoy
    image: envoyproxy/envoy:v1.30-latest
    ports:
    - containerPort: 8080
      name: grpc-inbound
    - containerPort: 9090
      name: admin
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
  - name: scheduler
    ports:
    - containerPort: 9090
      name: grpc
  volumes:
  - name: envoy-config
    configMap:
      name: envoy-config
```

Envoy config for gRPC (permissive mTLS, observability):

```yaml
static_resources:
  listeners:
  - name: inbound
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_grpc
          route_config:
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route:
                  cluster: local_grpc
                  timeout: 0s
          http_filters:
          - name: envoy.filters.http.router
          access_log:
          - name: envoy.access_loggers.file
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
              path: /dev/stdout
  clusters:
  - name: local_grpc
    type: STATIC
    lb_policy: ROUND_ROBIN
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: local_grpc
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: 127.0.0.1, port_value: 9090 }
```

### Service Mesh (Istio, Linkerd)

A service mesh provides sidecar proxies with zero-config mTLS, traffic splitting, retries, circuit breaking, and observability.

**Istio for gRPC:**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: chronos-scheduler
spec:
  host: chronos-scheduler
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    connectionPool:
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: chronos-scheduler
spec:
  hosts:
  - chronos-scheduler
  http:
  - match:
    - headers:
        x-canary:
          exact: "v2"
    route:
    - destination:
        host: chronos-scheduler
        subset: v2
      weight: 100
  - route:
    - destination:
        host: chronos-scheduler
        subset: v1
      weight: 100
```

### gRPC Ingress

K8s ingress controller must support HTTP/2:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chronos-grpc
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - chronos.example.com
    secretName: chronos-tls
  rules:
  - host: chronos.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: chronos-scheduler
            port:
              number: 9090
```

### gRPC + API Gateways

| Gateway | Support | Notes |
|---------|---------|-------|
| Kong | gRPC, gRPC-stream | `protocols: ["grpc", "grpcs"]` |
| AWS API Gateway | gRPC (HTTP/2 + TLS) | Requires HTTP/2 and TLS termination |
| NGINX | `grpc_pass` | `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"` |
| Envoy | Native | Best gRPC support |
| Azure API Mgmt | gRPC (preview) | Limited |

### Docker Multi-Stage Build for Proto Generation

```dockerfile
FROM golang:1.22 AS protobuf
RUN go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
RUN go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
COPY proto/ /build/proto/
WORKDIR /build
RUN protoc --go_out=. --go-grpc_out=. proto/chronos/v1/*.proto

FROM golang:1.22 AS app
WORKDIR /app
COPY --from=protobuf /build/ .
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /app/server ./cmd/server

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=app /app/server /usr/local/bin/
EXPOSE 9090
CMD ["server"]
```

> **Trap:** Not setting `protocol: GRPC` in K8s ingress — NGINX forwards traffic as HTTP/1.1 by default. gRPC over HTTP/1.1 doesn't work. The annotation `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"` or `"GRPCS"` is required.

> **Trap:** Using ClusterIP Service instead of headless — as explained in section 1, the ClusterIP's L4 load balancer pins the gRPC connection to one backend. RPCs don't balance across pods. Use headless (`clusterIP: None`) for client-side balancing, or use an L7 proxy.

> **Trap:** Not allocating enough memory — protobuf deserialization allocates memory for every message. A large proto message (multi-MB) can spike memory. Set resource limits. Monitor with `container_memory_working_set_bytes`.

> **Trap:** Not handling rolling updates with connection draining — when a pod is terminated during rolling update, in-flight RPCs are interrupted. Use `preStop` lifecycle hook with `sleep 15` to give K8s time to remove the pod from endpoints. Set `MaxConnectionAge` so clients naturally reconnect.

---

## 7. Security — mTLS & Beyond

### TLS Basics

```
Client                     Server
  │                          │
  │ 1. ClientHello           │
  │ 2. ServerHello + Cert    │
  │ 3. ClientKeyExchange     │
  │ 4. ChangeCipherSpec      │
  │ 5. Encrypted data        │
  │                          │
  │ mTLS adds:               │
  │ 3b. CertificateRequest   │
  │ 3c. Client Certificate   │
  │ 3d. ClientCertVerify     │
```

### mTLS Setup in Go

**Server loads CA cert to verify client:**

```go
func loadMTLSCredentials() (credentials.TransportCredentials, error) {
    serverCert, err := tls.LoadX509KeyPair("certs/server.crt", "certs/server.key")
    if err != nil {
        return nil, err
    }

    caCert, err := os.ReadFile("certs/ca.crt")
    if err != nil {
        return nil, err
    }
    caPool := x509.NewCertPool()
    caPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{serverCert},
        ClientCAs:    caPool,
        ClientAuth:   tls.RequireAndVerifyClientCert,
        MinVersion:   tls.VersionTLS13,
    }

    return credentials.NewTLS(tlsConfig), nil
}

func main() {
    creds, _ := loadMTLSCredentials()
    s := grpc.NewServer(grpc.Creds(creds))
    pb.RegisterSchedulerServer(s, &schedulerServer{})
    // ...
}
```

**Client loads client cert:**

```go
func loadClientTLSCredentials() (credentials.TransportCredentials, error) {
    clientCert, err := tls.LoadX509KeyPair("certs/client.crt", "certs/client.key")
    if err != nil {
        return nil, err
    }

    caCert, err := os.ReadFile("certs/ca.crt")
    if err != nil {
        return nil, err
    }
    caPool := x509.NewCertPool()
    caPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{clientCert},
        RootCAs:      caPool,
        MinVersion:   tls.VersionTLS13,
    }

    return credentials.NewTLS(tlsConfig), nil
}

func main() {
    creds, _ := loadClientTLSCredentials()
    conn, _ := grpc.Dial("chronos-scheduler:9090", grpc.WithTransportCredentials(creds))
}
```

### Certificate Rotation

**Static files (watch with fsnotify):**

```go
func watchCertificates(certPath, keyPath string, creds *credentials.TransportCredentials) {
    watcher, _ := fsnotify.NewWatcher()
    watcher.Add(certPath)
    watcher.Add(keyPath)

    for {
        select {
        case event := <-watcher.Events:
            if event.Op&(fsnotify.Write|fsnotify.Create) != 0 {
                newCreds, err := loadMTLSCredentials()
                if err == nil {
                    creds = &newCreds
                    log.Println("certificates reloaded")
                }
            }
        }
    }
}
```

**Or use `tls.GetConfigForClient` for per-connection cert loading:**

```go
tlsConfig := &tls.Config{
    GetConfigForClient: func(info *tls.ClientHelloInfo) (*tls.Config, error) {
        cert, err := tls.LoadX509KeyPair("certs/server.crt", "certs/server.key")
        if err != nil {
            return nil, err
        }
        return &tls.Config{
            Certificates: []tls.Certificate{cert},
            ClientCAs:    caPool,
            ClientAuth:   tls.RequireAndVerifyClientCert,
        }, nil
    },
}
```

### Token-Based Auth (JWT) on Top of TLS

mTLS authenticates the **connection** (this is a valid service). JWT authenticates the **caller** (this user/service has permission):

```go
// Client attaches JWT to metadata
func (s *SchedulerServer) ScheduleJob(ctx context.Context, req *pb.ScheduleJobRequest) (*pb.Job, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "no metadata")
    }
    tokens := md["authorization"]
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "no token")
    }

    claims, err := validateJWT(tokens[0])
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }

    // Use claims for authorization
    if !claims.HasPermission("scheduler:schedule") {
        return nil, status.Error(codes.PermissionDenied, "insufficient permissions")
    }

    // Process request
    return s.createJob(ctx, req, claims.UserID)
}
```

**Or use an interceptor:**

```go
func AuthInterceptor(ctx context.Context, req interface{},
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {

    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }

    token := extractBearerToken(md["authorization"])
    if token == "" {
        return nil, status.Error(codes.Unauthenticated, "missing token")
    }

    claims, err := validateJWT(token)
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }

    // Inject claims into context for downstream use
    ctx = context.WithValue(ctx, "claims", claims)
    return handler(ctx, req)
}
```

### Centralized Auth with Envoy External Auth

```yaml
http_filters:
- name: envoy.filters.http.ext_authz
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
    grpc_service:
      envoy_grpc:
        cluster_name: auth_service
      timeout: 0.5s
    with_request_body:
      max_request_bytes: 1024
```

> **Trap:** Relying solely on TLS for authentication — mTLS verifies the **client certificate** is signed by a trusted CA. It does not verify that the caller has permission to call this specific method. After mTLS, you still need per-RPC authorization. mTLS is authentication (who), not authorization (can they).

> **Follow-up:** "So you need both mTLS and JWT?" — It depends. Internal microservices in a service mesh: mTLS (service identity) + JWT/OAuth (user identity). Machine-to-machine in a trusted network: mTLS-only may be sufficient. External-facing: always JWT on top of TLS. Defense in depth: both.

> **Trap:** Not rotating certificates — certificates expire. Expired certs cause connection drops. Automate rotation with cert-manager (K8s), Vault PKI, or Let's Encrypt. Monitor certificate expiry with alerts at 30, 14, and 7 days before expiry.

> **Trap:** Embedding secrets in code — hardcoded API keys, JWT secrets, or cert passwords in source. Use environment variables, K8s Secrets, Vault, or AWS Secrets Manager. Scan for secrets in git pre-commit hooks.

> **Trap:** Wide-open authorization — checking at the connection level ("this service can call any method") instead of per-method ("this service can call ScheduleJob but not DeleteJob"). Use method-level authorization in interceptors for fine-grained control.

---

## 8. Streaming Performance Tuning

### Large Message Handling

Default max message size is **4MB** (server) / **512MB** (for internal servers, the limit is high). In production, tune explicitly:

```go
// Server — accept messages up to 100MB
s := grpc.NewServer(
    grpc.MaxRecvMsgSize(100 * 1024 * 1024),
    grpc.MaxSendMsgSize(100 * 1024 * 1024),
)

// Client — send/receive up to 100MB
conn, err := grpc.Dial(
    "dns:///chronos-scheduler:9090",
    grpc.WithDefaultCallOptions(
        grpc.MaxCallRecvMsgSize(100 * 1024 * 1024),
        grpc.MaxCallSendMsgSize(100 * 1024 * 1024),
    ),
)
```

**Never send large payloads in a single message.** Use streaming:

```protobuf
// BAD — single large message
message BatchProcessRequest {
  repeated Job jobs = 1; // could be 10K jobs → multi-MB
}

// GOOD — streaming
service Scheduler {
  rpc BatchProcess(stream Job) returns (BatchSummary);
}
```

### Message Chunking

For very large data (files, snapshots), chunk and reassemble:

```protobuf
message Chunk {
  string upload_id = 1;
  int32  sequence = 2;
  bytes  data = 3; // 1MB per chunk
  bool   final = 4;
}

service Scheduler {
  rpc UploadSnapshot(stream Chunk) returns (UploadResult);
}
```

### HTTP/2 Flow Control

HTTP/2 has per-stream and per-connection flow control using WINDOW_UPDATE frames:

```
Default initial window size: 64KB per stream, 64KB per connection
```

Tuning for high-throughput links (especially high latency × high bandwidth):

```go
s := grpc.NewServer(
    grpc.InitialWindowSize(1024 * 1024),        // 1MB per stream
    grpc.InitialConnWindowSize(4 * 1024 * 1024), // 4MB per connection
)
```

Larger windows allow the sender to fill the pipe without waiting for WINDOW_UPDATE. For high-latency links (cross-region), larger windows significantly improve throughput.

```
Window sizing guidelines:
- Same datacenter (0.5ms latency): 64KB–256KB
- Same region (5ms latency): 256KB–1MB
- Cross-region (50ms latency): 1MB–4MB
- Cross-continent (150ms+): 4MB–16MB
```

### Buffering and Backpressure

```
Client send buffer → HTTP/2 stream → Server recv buffer → Handler
                                                    ↓
                                          If handler is slow → buffer fills
                                          → WINDOW_UPDATE stops → sender blocks
```

gRPC client-side streaming has backpressure built in: if the server stops reading, the flow control window fills up, and the client's `Send` blocks. This is automatic — no special configuration needed.

But understand where buffering happens:

1. **Client send buffer** — `grpc.MaxCallSendMsgSize` limits individual messages. The `Send` method blocks when the window is full.
2. **Server recv buffer** — `grpc.MaxRecvMsgSize` limits individual messages. The `Recv` method blocks when no data is available.
3. **HTTP/2 stream buffers** — the operating system's TCP buffer. Tune `SO_SNDBUF` / `SO_RCVBUF` for large streams:

```go
lis, err := net.Listen("tcp", ":9090")
if tc, ok := lis.(*net.TCPListener); ok {
    if f, ok := tc.File(); ok {
        f.SetReadBuffer(4 * 1024 * 1024)  // 4MB recv
        f.SetWriteBuffer(4 * 1024 * 1024) // 4MB send
    }
}
```

### Compression

| Algorithm | Speed | Ratio | Best for |
|-----------|-------|-------|----------|
| `gzip` | Moderate | 4-6x | Large messages, infrequent calls |
| `snappy` | Fast | 2-3x | Streaming, high-throughput, low CPU cost |
| None | — | 1x | Very small messages (<1KB, compression overhead > savings) |

```go
// Client enables compression
conn, err := grpc.Dial(
    "dns:///chronos-scheduler:9090",
    grpc.WithDefaultCallOptions(grpc.UseCompressor("gzip")),
)

// Or per-call
client.ScheduleJob(ctx, req, grpc.UseCompressor("snappy"))
```

**Server must register the compressor:**

```go
import "google.golang.org/grpc/encoding/gzip"
// gzip is registered by default in gRPC Go.
// For snappy, register a custom compressor:
import "github.com/golang/snappy"
import "google.golang.org/grpc/encoding"

type snappyCompressor struct{}

func (c snappyCompressor) Name() string { return "snappy" }
func (c snappyCompressor) Compress(dst, src []byte) ([]byte, error) {
    return snappy.Encode(dst, src), nil
}
func (c snappyCompressor) Decompress(dst, src []byte) ([]byte, error) {
    return snappy.Decode(dst, src), nil
}

func init() {
    encoding.RegisterCompressor(snappyCompressor{})
}
```

### Benchmarking with ghz

[ghz](https://ghz.sh) is a gRPC load testing tool:

```bash
# Basic unary benchmark
ghz --insecure \
    --proto ./proto/chronos/v1/scheduler.proto \
    --call chronos.Scheduler/ScheduleJob \
    -d '{"job_name": "bench", "cron": "* * * * *"}' \
    -n 10000 \
    -c 50 \
    localhost:9090

# With keepalive
ghz --insecure \
    --proto ./proto/chronos/v1/scheduler.proto \
    --call chronos.Scheduler/WatchJobs \
    -d '{}' \
    --stream-interval 5s \
    --stream-call-timeout 30s \
    -n 10000 \
    -c 50 \
    localhost:9090

# Output format
ghz --insecure \
    --call chronos.Scheduler/ScheduleJob \
    -d '{}' \
    -n 10000 \
    -c 50 \
    --format html \
    --output report.html \
    localhost:9090
```

**Key metrics:**

| Metric | What it measures |
|--------|-----------------|
| Latency p50/p95/p99 | Response time distribution |
| Requests/sec | Throughput |
| Error % | Failure rate |
| Concurrency | Active connections |
| Bytes in/out | Bandwidth |

### Flame Graphs for Bottlenecks

```bash
# Profile the server during ghz load test
go tool pprof -seconds=30 http://localhost:9090/debug/pprof/profile

# Generate flame graph
go tool pprof -svg -output profile.svg /path/to/binary /path/to/profile
```

Look for:
- **Protobuf marshaling/unmarshaling** — large messages dominate CPU. Optimize: smaller messages, streaming, or proto optimization (avoid `repeated` of complex types).
- **GC pressure** — high allocation rate from protobuf parsing. Pool message objects with `sync.Pool`.
- **Mutex contention** — `grpc.Server` has per-stream locks. High concurrency + small messages can cause contention.

> **Trap:** Disabling flow control — setting window sizes too large or disabling flow control entirely can OOM the server. The sender can fill server memory faster than the handler processes. Always keep flow control enabled; tune window sizes conservatively.

> **Follow-up:** "What's the downside of increasing initial window size?" — Memory. The sender can have up to `window_size` bytes in flight per stream without receiving a WINDOW_UPDATE. For 1000 concurrent streams × 4MB window = 4GB potential buffer. Balance throughput against memory budget.

> **Trap:** Using gzip compression for streaming — gzip is CPU-intensive and adds latency per message. For streaming, the CPU overhead of compressing each message often outweighs bandwidth savings. Use snappy (faster, lower CPU) or no compression. Gzip is better for large unary requests.

> **Trap:** Forgetting to set message size limits — default 4MB on server (older versions) or effective unlimited (512MB). Without explicit limits, a client can send a 1GB message and OOM the server. Always set `MaxRecvMsgSize` and `MaxSendMsgSize` based on your expected message sizes.

> **Trap:** Not tuning window sizes for high-latency links — default 64KB means on a transatlantic link (150ms RTT), the sender sends 64KB then waits 150ms for the WINDOW_UPDATE. Max throughput = 64KB / 0.15s ≈ 430KB/s. Tune to match your bandwidth-delay product.

---

## 9. gRPC vs Message Queues

### Decision Matrix

```
┌──────────────────────┬──────────────────────────────┬──────────────────────────────┐
│ Property             │ gRPC                         │ Message Queue (Kafka, RMQ)   │
├──────────────────────┼──────────────────────────────┼──────────────────────────────┤
│ Communication model  │ Synchronous / RPC            │ Async / Event-driven         │
│ Coupling             │ Tight (both must be up)      │ Loose (producer doesn't know │
│                      │                              │ consumer)                    │
│ Latency              │ Low (milliseconds)            │ Moderate (milliseconds+)     │
│ Persistence          │ None (in-memory)             │ Durable (disk)               │
│ Replay               │ No                           │ Yes (offset reset)           │
│ Fan-out              │ No (one-to-one)              │ Yes (consumer groups)        │
│ FIFO ordering        │ Per-stream (within conn)     │ Per-partition (Kafka)        │
│ Streaming            │ Bidirectional, real-time     │ Poll-based (consumer pull)   │
│ Backpressure         │ Built-in (flow control)      │ Consumer lag (configurable)  │
│ Message size         │ Limited (typically <100MB)   │ Up to 1GB+                   │
│ Schema               │ Strict (protobuf)            │ Schema registry (Avro, JSON) │
│ Reliability          │ Best-effort (retries help)   │ At-least-once, exactly-once  │
└──────────────────────┴──────────────────────────────┴──────────────────────────────┘
```

### When to Use gRPC

- **Synchronous request-response** — client needs an answer: "Schedule this job, tell me the job ID"
- **Low latency required** — single-digit milliseconds
- **Streaming data** — real-time job output, status updates, log tailing
- **Tight coupling is acceptable** — both services are in the same deployment (same team, same SLA)
- **Control plane operations** — commands, queries, configuration

### When to Use Message Queues

- **Async / event-driven** — "When a job completes, send a notification." Producer doesn't wait for consumer.
- **Loose coupling** — producer and consumer can be deployed independently, different teams, different SLAs
- **Buffering / load leveling** — handle traffic spikes by queueing messages
- **Fan-out** — one event triggers multiple downstream processors (billing, analytics, notifications)
- **Replay / reprocessing** — consumer fails, can replay from last committed offset
- **FIFO ordering** — process events in strict order
- **Long-term persistence** — retain events for days/weeks for audit, analytics, or reprocessing

### Hybrid Patterns — Chronos Example

Chronos uses **both gRPC and a message queue**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Chronos                                      │
│                                                                     │
│  Control Plane (gRPC)              Event Plane (MQ)                 │
│  ┌──────────────────────┐          ┌──────────────────────┐        │
│  │ CreateJob            │          │ JobCompleted         │        │
│  │ CancelJob            │          │ JobFailed             │        │
│  │ PauseJob             │          │ JobProgressUpdate     │        │
│  │ GetJobStatus         │          │ SchedulerScaled       │        │
│  │ ListJobs             │          │ NodeJoined/Left       │        │
│  │ WatchJobOutput       │          │                      │        │
│  └──────────────────────┘          └──────────────────────┘        │
│           │                                  │                      │
│           ▼                                  ▼                      │
│    gRPC service                     Kafka topic                     │
│    (synchronous)                     (async, durable)               │
└─────────────────────────────────────────────────────────────────────┘
```

**Control plane (gRPC):** clients call `ScheduleJob`, `CancelJob`, `GetJobStatus` synchronously. These need immediate responses. The scheduler acknowledges the command, validates it, and returns a job ID.

**Event plane (MQ):** when a job transitions state (running → completed), the scheduler publishes `JobCompleted` to a Kafka topic. Downstream services (billing, monitoring, webhooks) consume independently. The scheduler doesn't wait for them.

**Transactional outbox pattern:**

```
Client → gRPC: ScheduleJob
  Server:
    1. BEGIN transaction
    2. INSERT job
    3. INSERT outbox event (JobScheduled)
    4. COMMIT
    5. Publish event to MQ (or outbox relay picks it up)

  gRPC response: { job_id, status: "scheduled" }
```

This ensures the job is persisted and the event is reliably delivered. The outbox prevents dual-write problems (DB write succeeds, MQ publish fails).

### gRPC Streaming Is NOT a Message Queue Replacement

```protobuf
// This looks like a message queue — but it isn't
service Scheduler {
  rpc JobEvents(JobEventsRequest) returns (stream JobEvent);
}
```

**Why gRPC streaming can't replace an MQ:**

| MQ Feature | gRPC Streaming |
|------------|----------------|
| Durable persistence | No — events are gone if server restarts |
| Message replay | No — you can't re-read past events |
| Consumer disconnect tolerance | No — if consumer disconnects, unread events are lost |
| Fan-out to multiple consumers | No — each stream goes to one consumer |
| Independent consumer offset | No — server tracks send state per-stream |
| Backlog management | No — consumer must keep up or events are dropped |

> **Trap:** Trying to use gRPC streaming as a message queue — gRPC streams are in-memory, not persistent. If the consumer disconnects or is slow, events are lost. There is no replay, no fan-out, no durability. Use gRPC streaming for real-time notification (consumer is always connected, can tolerate loss). Use a message queue for durable, replayable event delivery.

> **Follow-up:** "What if I need both real-time streaming and durability?" — Hybrid: publish events to Kafka, and have a gRPC streaming endpoint that reads from Kafka. The server maintains a Kafka consumer and forwards events to gRPC clients. The Kafka log provides durability and replay; the gRPC stream provides real-time delivery.

> **Trap:** Using message queues for low-latency request-response — MQs add latency (serialization, network, persistence, dequeue). Kafka produce → consumer latency is typically 5-50ms. gRPC unary is 0.5-5ms. Don't use a queue where synchronous RPC is the right pattern.

> **Trap:** Thinking gRPC and MQ are mutually exclusive — they complement each other. Use gRPC for the control plane (commands, queries, real-time streams). Use MQ for the event plane (notifications, event-driven processing, buffering, fan-out). This is the standard architecture for distributed systems like Chronos.

---

## 10. gRPC in Go — Production Patterns

### Server Setup — Graceful Shutdown

```go
func serve() error {
    lis, err := net.Listen("tcp", ":9090")
    if err != nil {
        return err
    }

    healthServer := health.NewServer()
    schedulerServer := newSchedulerServer()

    s := grpc.NewServer(
        grpc.MaxRecvMsgSize(100 * 1024 * 1024),
        grpc.MaxSendMsgSize(100 * 1024 * 1024),
        grpc.ChainUnaryInterceptor(
            loggingInterceptor,
            authInterceptor,
            metricsInterceptor,
        ),
        grpc.ChainStreamInterceptor(
            streamLoggingInterceptor,
            streamMetricsInterceptor,
        ),
        grpc.KeepaliveParams(keepalive.ServerParameters{
            MaxConnectionIdle:     15 * time.Minute,
            MaxConnectionAge:      30 * time.Minute,
            MaxConnectionAgeGrace: 5 * time.Second,
            Time:                  30 * time.Second,
            Timeout:               5 * time.Second,
        }),
        grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
            MinTime:             5 * time.Second,
            PermitWithoutStream: false,
        }),
    )

    pb.RegisterSchedulerServer(s, schedulerServer)
    grpc_health_v1.RegisterHealthServer(s, healthServer)
    reflection.Register(s)

    // Mark as not serving until initialized
    healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_NOT_SERVING)

    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
        <-sigCh

        log.Println("shutting down...")
        healthServer.SetServingStatus("chronos.Scheduler", grpc_health_v1.HealthCheckResponse_NOT_SERVING)

        stopped := make(chan struct{})
        go func() {
            s.GracefulStop()
            close(stopped)
        }()

        select {
        case <-stopped:
            log.Println("graceful stop complete")
        case <-time.After(30 * time.Second):
            log.Println("graceful stop timeout, forcing stop")
            s.Stop()
        }
    }()

    return s.Serve(lis)
}
```

**`GracefulStop` vs `Stop`:**

| Method | Behavior |
|--------|----------|
| `GracefulStop()` | Stops accepting new connections. Waits for all in-flight RPCs to complete. Blocks indefinitely. |
| `Stop()` | Immediately terminates all connections. In-flight RPCs are cancelled. Non-graceful. |

Always use `GracefulStop` with a timeout fallback to `Stop`.

### Interceptor Chaining

Order matters — outer interceptor runs first:

```go
grpc.ChainUnaryInterceptor(
    outerInterceptor,   // runs first (request → middleware)
    middleInterceptor,  // runs second
    innerInterceptor,   // runs third (closest to handler)
)
```

Execution order when an RPC arrives:

```
Request → outerInterceptor → middleInterceptor → innerInterceptor → handler
Response ← outerInterceptor ← middleInterceptor ← innerInterceptor ← handler
```

This means:
- **Logging** should be outermost (wraps everything)
- **Auth** should be second (fail fast before metrics)
- **Metrics** should be inner (measure actual handler latency)
- **Panic recovery** should be outermost (catches panics from all downstream interceptors)

```go
func recoveryInterceptor(ctx context.Context, req interface{},
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("panic in %s: %v", info.FullMethod, r)
            err = status.Errorf(codes.Internal, "internal error")
        }
    }()
    return handler(ctx, req)
}
```

### Client Setup — Dial, Connection Reuse, Service Config

```go
func newSchedulerClient() (pb.SchedulerClient, func(), error) {
    conn, err := grpc.Dial(
        "dns:///chronos-scheduler.default.svc.cluster.local:9090",
        grpc.WithTransportCredentials(creds),
        grpc.WithDefaultServiceConfig(`{
            "loadBalancingConfig": [{"round_robin": {}}],
            "methodConfig": [{
                "name": [{"service": "chronos.Scheduler"}],
                "retryPolicy": {
                    "maxAttempts": 3,
                    "initialBackoff": "0.1s",
                    "maxBackoff": "1s",
                    "backoffMultiplier": 2,
                    "retryableStatusCodes": ["UNAVAILABLE"]
                }
            }]
        }`),
        grpc.WithKeepaliveParams(keepalive.ClientParameters{
            Time:                30 * time.Second,
            Timeout:             10 * time.Second,
            PermitWithoutStream: true,
        }),
        grpc.WithConnectParams(grpc.ConnectParams{
            MinConnectTimeout: 5 * time.Second,
        }),
        grpc.WithDefaultCallOptions(
            grpc.MaxCallRecvMsgSize(100 * 1024 * 1024),
            grpc.MaxCallSendMsgSize(100 * 1024 * 1024),
            grpc.WaitForReady(true),
        ),
        grpc.WithUserAgent("chronos-client/1.0"),
    )
    if err != nil {
        return nil, nil, err
    }

    cleanup := func() {
        conn.Close()
    }

    return pb.NewSchedulerClient(conn), cleanup, nil
}
```

**`WaitForReady`:** if set to `true`, the client waits until the connection is ready before sending the RPC. Without it, RPCs fail immediately if the channel is in `TRANSIENT_FAILURE` state.

### Error Wrapping

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *SchedulerServer) ScheduleJob(ctx context.Context, req *pb.ScheduleJobRequest) (*pb.Job, error) {
    if req.JobName == "" {
        return nil, status.Error(codes.InvalidArgument, "job_name is required")
    }

    job, err := s.store.CreateJob(ctx, req)
    if err != nil {
        if errors.Is(err, ErrJobAlreadyExists) {
            return nil, status.Error(codes.AlreadyExists, "job already exists")
        }
        if errors.Is(err, ErrQuotaExceeded) {
            return nil, status.Error(codes.ResourceExhausted, "quota exceeded")
        }
        // Log the internal error but don't expose details
        log.Printf("failed to create job: %v", err)
        return nil, status.Error(codes.Internal, "internal error")
    }

    return job, nil
}
```

**On the client side:**

```go
job, err := client.ScheduleJob(ctx, req)
if err != nil {
    st, ok := status.FromError(err)
    if ok {
        switch st.Code() {
        case codes.InvalidArgument:
            // Retry won't help
            return err
        case codes.Unavailable:
            // Retry with backoff
            case codes.DeadlineExceeded:
                // Increase timeout or retry
        }
    }
}
```

### Context Propagation

Pass deadline, cancellation, and metadata through the service layer:

```go
func (s *SchedulerServer) ScheduleJob(ctx context.Context, req *pb.ScheduleJobRequest) (*pb.Job, error) {
    // Extract tracing info
    md, _ := metadata.FromIncomingContext(ctx)
    traceID := md["x-trace-id"][0]

    // Create a derived context for the DB operation
    dbCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    job, err := s.store.CreateJob(dbCtx, req)
    // ...

    // Pass trace ID to downstream calls
    outCtx := metadata.AppendToOutgoingContext(ctx, "x-trace-id", traceID)
    notificationClient.NotifyJobScheduled(outCtx, job)
}
```

### Testing — In-Memory with bufconn

```go
import (
    "google.golang.org/grpc/test/bufconn"
)

const bufSize = 1024 * 1024

func TestSchedulerServer(t *testing.T) {
    lis := bufconn.Listen(bufSize)
    s := grpc.NewServer()
    pb.RegisterSchedulerServer(s, &SchedulerServer{store: newTestStore()})

    go s.Serve(lis)
    defer s.Stop()

    conn, err := grpc.DialContext(context.Background(), "bufnet",
        grpc.WithContextDialer(func(ctx context.Context, s string) (net.Conn, error) {
            return lis.Dial()
        }),
        grpc.WithInsecure(),
    )
    require.NoError(t, err)
    defer conn.Close()

    client := pb.NewSchedulerClient(conn)
    job, err := client.ScheduleJob(context.Background(), &pb.ScheduleJobRequest{
        JobName: "test-job",
        Cron:    "* * * * *",
    })
    require.NoError(t, err)
    require.Equal(t, "test-job", job.JobName)
}
```

### Testing — Mock Server

```go
//go:generate mockgen -source=api/chronos/v1/scheduler_grpc.pb.go -package=mocks -destination=mocks/scheduler.go

func TestWorkflow(t *testing.T) {
    ctrl := gomock.NewController(t)
    mockScheduler := mocks.NewMockSchedulerClient(ctrl)

    mockScheduler.EXPECT().
        ScheduleJob(gomock.Any(), gomock.Any()).
        Return(&pb.Job{JobId: "job_123", JobName: "test"}, nil)

    // Test code that uses the client
}
```

### OpenTelemetry Integration

```go
import (
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

// Server with automatic instrumentation
s := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)

// Client with automatic instrumentation
conn, err := grpc.Dial(
    "dns:///chronos-scheduler:9090",
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

This captures:
- **Span per RPC** (method name, status code, duration)
- **Context propagation** (trace ID flows through gRPC metadata)
- **Metrics** (request count, latency histogram, error rate)

### Prometheus Metrics

```go
import "github.com/grpc-ecosystem/go-grpc-prometheus"

// Register metrics
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        grpc_prometheus.UnaryServerInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        grpc_prometheus.StreamServerInterceptor,
    ),
)

// Register with Prometheus
grpc_prometheus.Register(s)

// Expose metrics endpoint
http.Handle("/metrics", promhttp.Handler())
go http.ListenAndServe(":9092", nil)
```

**Key metrics:**

| Metric | Type | Description |
|--------|------|-------------|
| `grpc_server_handled_total` | Counter | Total RPCs by method and status |
| `grpc_server_handling_seconds` | Histogram | Latency by method |
| `grpc_server_msg_received_total` | Counter | Messages received by method |
| `grpc_server_msg_sent_total` | Counter | Messages sent by method |
| `grpc_client_started_total` | Counter | Client RPCs started |

### Migrating from REST to gRPC — grpc-gateway

[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) generates a REST JSON API from your proto definitions:

```protobuf
import "google/api/annotations.proto";

service Scheduler {
  rpc ScheduleJob(ScheduleJobRequest) returns (Job) {
    option (google.api.http) = {
      post: "/v1/jobs"
      body: "*"
    };
  }
  rpc GetJob(GetJobRequest) returns (Job) {
    option (google.api.http) = {
      get: "/v1/jobs/{job_id}"
    };
  }
  rpc ListJobs(ListJobsRequest) returns (ListJobsResponse) {
    option (google.api.http) = {
      get: "/v1/jobs"
    };
  }
}
```

Generate:

```bash
protoc -I. --grpc-gateway_out=. --grpc-gateway_opt=logtostderr=true \
    proto/chronos/v1/scheduler.proto
```

Server that serves both gRPC and REST:

```go
func main() {
    // gRPC server
    grpcServer := grpc.NewServer()
    schedulerServer := &SchedulerServer{}
    pb.RegisterSchedulerServer(grpcServer, schedulerServer)

    // HTTP server with grpc-gateway mux
    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithInsecure()}
    pb.RegisterSchedulerHandlerFromEndpoint(context.Background(), mux, "localhost:9090", opts)

    httpServer := &http.Server{
        Handler: mux,
        Addr:    ":8080",
    }

    go grpcServer.Serve(mustListen(":9090"))
    httpServer.ListenAndServe()
}
```

> **Trap:** Not handling `GracefulStop` timeout — `GracefulStop` blocks indefinitely if an RPC hangs. Always wrap it with a timeout and fall back to `Stop`. Otherwise, a stuck RPC prevents the pod from shutting down and K8s eventually kills it with `SIGKILL`.

> **Trap:** Creating clients inside handlers — `grpc.Dial` creates a real TCP connection (even with lazy connection). Creating a new client per request leaks goroutines and connections. Create once, reuse for the lifetime of the application.

> **Trap:** Not setting max message size in production — default max receive size is 4MB (server) / 512MB (client). Without explicit limits, clients can send messages that take excessive time to parse or OOM the server. Set based on actual message sizes with headroom.

> **Trap:** Mixing gRPC versions — `google.golang.org/grpc` v1.x uses `protobuf` v2 (`google.golang.org/protobuf`). Old code using `github.com/golang/protobuf` v1 can cause subtle incompatibilities. Use `go mod tidy` and check imports. Proto imports should use `google.golang.org/protobuf` packages.

---

## 11. Tier 3 Q&A Drill

Close the file, answer out loud, then check your answer for vagueness.

### Load Balancing

**Q1:** Why can't you use a regular K8s ClusterIP Service for gRPC load balancing?

<details>
<summary>Answer</summary>

ClusterIP does L4 TCP load balancing. When a gRPC client connects, it opens one long-lived HTTP/2 TCP connection. The ClusterIP proxy picks one backend for that connection and pins it. All subsequent RPCs on that channel go to the same backend. There's no per-RPC balancing. Use a headless Service (DNS-based, each pod gets its own IP) with client-side `round_robin`, or use an L7 proxy like Envoy.
</details>

**Q2:** What's the difference between `pick_first` and `round_robin` in gRPC client-side load balancing?

<details>
<summary>Answer</summary>

`pick_first` (default): the resolver returns multiple addresses, but the channel only connects to the first one. If that connection fails, it tries the next. At any time, only one backend is used. `round_robin`: the channel connects to every resolved address and distributes RPCs across them in rotation. Use `pick_first` for failover (active-passive). Use `round_robin` for load distribution (active-active).
</details>

**Q3:** How does the gRPC resolver/balancer architecture work?

<details>
<summary>Answer</summary>

The resolver takes a target URI (e.g., `dns:///service:9090`) and returns a list of addresses. The balancer creates one subchannel per address and implements the load balancing policy (`pick_first`, `round_robin`, or custom). The balancer picks a subchannel for each RPC. Resolvers can be custom (Consul, etcd, K8s API). Balancers can be custom (weighted, least-loaded, etc.).
</details>

### Connection Management

**Q4:** What happens if you don't configure keepalive on either the client or server?

<details>
<summary>Answer</summary>

Without keepalive, neither side detects dead connections. If a backend crashes, the server's TCP connection stays half-open. The client sends RPCs that eventually time out (after the application-level deadline), but the subchannel is never cleaned up. Zombie connections accumulate. Firewalls may kill idle connections after 5-15 minutes without the client knowing. Always configure keepalive on both sides.
</details>

**Q5:** What is `ENHANCE_YOUR_CALM` and when does it appear?

<details>
<summary>Answer</summary>

`ENHANCE_YOUR_CALM` is a gRPC error code (`codes.Unavailable`) with a special message. The server sends it when a client violates the keepalive enforcement policy — typically sending pings more frequently than `MinTime` (default 5s). The server may follow with a `GOAWAY` frame. Fix by adjusting client keepalive `Time` to be >= server `MinTime`.
</details>

**Q6:** Why would you set `MaxConnectionAge` on the server?

<details>
<summary>Answer</summary>

To force periodic reconnection. After MaxConnectionAge, the server sends GOAWAY and closes the connection. The client reconnects and may land on a different backend. This rebalances connections over time, especially useful with client-side round_robin where long-lived connections might otherwise become unbalanced during rolling updates or scale events.
</details>

### Retry & Hedging

**Q7:** What status codes should you retry on? Why not retry `INVALID_ARGUMENT`?

<details>
<summary>Answer</summary>

Retry: `UNAVAILABLE` (transient server failure), `DEADLINE_EXCEEDED` (if deadline is extended), `ABORTED` (transaction conflict), `RESOURCE_EXHAUSTED` (with backoff). Do NOT retry: `INVALID_ARGUMENT`, `NOT_FOUND`, `PERMISSION_DENIED`, `UNIMPLEMENTED` — these are deterministic client errors that will never succeed. Only retry codes that indicate transient conditions.
</details>

**Q8:** When would you use hedging instead of retry?

<details>
<summary>Answer</summary>

Hedging is for latency-sensitive operations where you can't afford the backoff delay of retry. Send multiple identical requests simultaneously, use the first response. Best for idempotent reads. Trade-off: extra server load (N concurrent requests instead of sequential). Never hedge non-idempotent writes.
</details>

**Q9:** How do you make a gRPC mutation safe for retry?

<details>
<summary>Answer</summary>

Add an `idempotency_key` field to the request proto. The server deduplicates: if the key was seen before, return the previous response instead of processing again. Store the key → response mapping in Redis with a TTL matching the maximum retry window (e.g., 24h). Additionally, use a DB unique constraint on the business identifier as a safety net.
</details>

### Reflection & gRPCurl

**Q10:** How do you debug a gRPC service that's returning unexpected errors without a client?

<details>
<summary>Answer</summary>

Use grpcurl against the server if reflection is enabled. List services, describe messages, invoke RPCs directly. Check health via `grpc.health.v1.Health/Check`. If reflection is disabled, use the compiled proto file with `-proto` flag. For streaming RPCs, pipe input from a file. For TLS, provide certs with `-cacert`, `-cert`, `-key`.
</details>

**Q11:** What's the security concern with reflection in production?

<details>
<summary>Answer</summary>

Reflection exposes all service names, method signatures, and message schemas. An attacker can discover the full API surface. Mitigations: run reflection on a separate admin port (internal-only), require authentication via interceptor, or only enable in staging/dev. Never expose reflection to external clients.
</details>

### Health Checking

**Q12:** Why should you implement per-service health rather than a single server-level health check?

<details>
<summary>Answer</summary>

A single gRPC server can register multiple services. One service might be healthy while another is broken (e.g., Scheduler service depends on Raft, but JobExecutor service is fine). Per-service health allows clients and load balancers to route traffic to only the healthy services. Envoy and K8s probes can target specific services. Without per-service health, a partial failure takes down the entire server.
</details>

**Q13:** How do you implement dependency-aware health checks?

<details>
<summary>Answer</summary>

In a background loop, check each critical dependency (DB ping, Redis ping, Raft cluster health). If any dependency fails, set the service status to `NOT_SERVING`. If all dependencies are healthy, set to `SERVING`. Don't just check at startup — check continuously. A dependency that goes down later should cause the health status to change.
</details>

### Production Deployment

**Q14:** Your gRPC service is deployed on K8s. During a rolling update, in-flight RPCs fail. What's wrong?

<details>
<summary>Answer</summary>

Multiple issues likely: (1) No `preStop` hook — the pod is terminated before K8s removes it from endpoints, so new RPCs are routed to a terminating pod. Add `lifecycle.preStop.exec.command: ["sleep", "15"]`. (2) No graceful shutdown — the server calls `Stop` instead of `GracefulStop`, killing in-flight RPCs. (3) No `MaxConnectionAge` — clients don't reconnect, so they stay on old pods until the connection drops. (4) No readiness probe change on SIGTERM — the server should set `NOT_SERVING` on shutdown so the load balancer stops routing.
</details>

**Q15:** What's the correct way to set up gRPC in K8s for client-side load balancing?

<details>
<summary>Answer</summary>

Create a headless Service (`.spec.clusterIP: None`) so that DNS returns pod IPs directly. Client uses `dns:///service.namespace.svc.cluster.local:9090` with `round_robin` load balancing config. Set gRPC health checks for readiness and liveness. Use `MaxConnectionAge` on the server for periodic rebalancing. Configure keepalive so dead connections are detected. Use `preStop` hook for connection draining.
</details>

### Security

**Q16:** What's the difference between TLS and mTLS in gRPC?

<details>
<summary>Answer</summary>

TLS: server presents a certificate, client verifies the server. Client is unauthenticated (any client with the server CA can connect). mTLS: both sides present certificates. The server also verifies the client certificate. Both sides authenticate each other. mTLS is the standard for service-to-service communication in a service mesh.
</details>

**Q17:** Why isn't mTLS sufficient for authorization?

<details>
<summary>Answer</summary>

mTLS authenticates the *client service* (this is the Scheduler service making a call), but not the *user or operation*. It doesn't tell you whether this caller is allowed to call this specific method. You still need per-RPC authorization (e.g., JWT claims, RBAC checks in an interceptor). mTLS is authentication (who), not authorization (may they).
</details>

### Streaming Performance

**Q18:** You have a cross-continent gRPC streaming connection. Throughput is much lower than expected. What do you check?

<details>
<summary>Answer</summary>

HTTP/2 flow control windows are the likely culprit. The default initial window size (64KB) means the sender fills 64KB then waits for a RTT before the WINDOW_UPDATE arrives. On a 150ms RTT link, max throughput ≈ 64KB / 0.15s ≈ 430KB/s. Increase `InitialWindowSize` (per-stream) and `InitialConnWindowSize` (per-connection) to match the bandwidth-delay product. Also check TCP buffer sizes and consider snappy compression.
</details>

**Q19:** When should you use gzip compression vs snappy vs no compression for gRPC?

<details>
<summary>Answer</summary>

No compression: for messages <1KB where compression overhead outweighs savings. gzip: for large unary messages (multi-KB to MB) where bandwidth is expensive and CPU is available. Snappy: for streaming or high-throughput scenarios where CPU is limited and moderate compression is acceptable. For streaming, prefer snappy or no compression — gzip per-message overhead adds latency.
</details>

### gRPC vs Message Queues

**Q20:** When would you choose gRPC streaming over Kafka for delivering job status updates?

<details>
<summary>Answer</summary>

gRPC streaming: when the consumer is always connected, real-time delivery is required, durability and replay are not needed, and the consumer can tolerate missed events during disconnection. Kafka: when durability, replay, fan-out to multiple consumers, independent consumer offsets, or long-term persistence are required. In Chronos, both are used: gRPC streaming for real-time job output (consumer connected, low latency), Kafka for job completion events (multiple downstream consumers, durability, replay).
</details>

**Q21:** You need to notify 10 downstream services when a job completes. Should you use gRPC streaming or a message queue?

<details>
<summary>Answer</summary>

Message queue. gRPC streaming is one-to-one (one stream per consumer). You'd need 10 separate server-side streams, each consuming resources. The server would need to track which consumers have received which events. If a consumer restarts, it missed events. A message queue handles fan-out natively: one topic, multiple consumer groups, each consumer reads independently, with offset management and replay.
</details>

### Go Production Patterns

**Q22:** What's the difference between `GracefulStop` and `Stop`? When would you use each?

<details>
<summary>Answer</summary>

`GracefulStop`: stops accepting new connections, waits for all in-flight RPCs to complete. Blocks indefinitely if an RPC hangs. `Stop`: immediately terminates all connections, cancelling in-flight RPCs. Use `GracefulStop` in production for zero-downtime shutdown, but wrap it with a timeout (e.g., 30s) and fall back to `Stop`. Use `Stop` only in tests or when you don't care about in-flight RPCs.
</details>

**Q23:** How do you test a gRPC server without opening a real port?

<details>
<summary>Answer</summary>

Use `bufconn` — an in-memory listener that implements `net.Listener`. Create a `bufconn.Listen`, pass it to `grpc.Server.Serve()`, and connect with a `grpc.WithContextDialer` that calls `lis.Dial()`. No port needed, no firewall issues, fast tests. This is the recommended testing pattern in the gRPC-Go documentation.
</details>

**Q24:** What are the gotchas when using grpc-gateway to dual-expose a service?

<details>
<summary>Answer</summary>

(1) The gateway makes a gRPC call to the backend — you're adding a network hop. For low-latency requirements, bypass the gateway. (2) Error mapping: gRPC status codes must be mapped to HTTP status codes (default mapping may not fit your API design). (3) Streaming: grpc-gateway supports server-side streaming (SSE), but not client-streaming or bidi streaming. (4) Latency: the gateway adds serialization (protobuf → JSON) and deserialization (JSON → protobuf). Profile before committing to dual-exposure for all endpoints.
</details>

---

> **End of Tier 3.** You should now be able to design, deploy, and operate gRPC services in production at scale. Move on to the question bank ([04-question-bank.md](./04-question-bank.md)) for extensive drill practice.
