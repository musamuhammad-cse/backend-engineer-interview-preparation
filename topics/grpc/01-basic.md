# gRPC — Tier 1: Basic (Fundamentals & Protocol Buffers)

gRPC is Google's high-performance RPC framework and the de facto standard for internal microservices communication. For a Senior Backend Engineer with a distributed systems project like Chronos (a Go-based distributed job scheduler using Raft), gRPC is the natural choice for node-to-node communication, client-worker coordination, and management APIs. This document covers the fundamentals you must own before moving to advanced gRPC topics like interceptors, deadlines, load balancing, and streaming patterns.

---

## Table of Contents

1. [What is gRPC & Why It Exists](#1-what-is-grpc--why-it-exists)
2. [Protocol Buffers — The Schema Language](#2-protocol-buffers--the-schema-language)
3. [Field Numbers & Compatibility](#3-field-numbers--compatibility)
4. [Service Definition](#4-service-definition)
5. [Unary RPC in Go](#5-unary-rpc-in-go)
6. [HTTP/2 Foundations](#6-http2-foundations)
7. [gRPC vs REST vs GraphQL](#7-grpc-vs-rest-vs-graphql)
8. [Protocol Buffers Best Practices](#8-protocol-buffers-best-practices)
9. [Tier 1 Q&A Drill](#9-tier-1-qa-drill)

---

## 1. What is gRPC & Why It Exists

gRPC is a language-agnostic, high-performance Remote Procedure Call (RPC) framework originally developed by Google. It uses **HTTP/2** for transport, **Protocol Buffers** (protobuf) for serialization, and generates strongly-typed client and server stubs from a `.proto` service definition.

### Problems gRPC Solves

| Problem | gRPC Solution |
|---|---|
| **Slow serialization** with JSON/XML | Binary protobuf serialization — orders of magnitude faster and smaller payloads |
| **No schema contract** in REST | `.proto` files define the exact shape of every request/response — no guessing |
| **No native streaming** in HTTP/1.1 REST | HTTP/2 multiplexed streams enable client, server, and bidirectional streaming |
| **Language coupling** — REST clients need manual wiring | Code generation from `.proto` files produces type-safe stubs in 11+ languages |
| **No built-in context propagation** | gRPC propagates deadlines, cancellations, and metadata through the context chain |

### gRPC vs REST

| Aspect | gRPC | REST |
|---|---|---|
| Protocol | HTTP/2 (binary) | HTTP/1.1 or HTTP/2 (text) |
| Serialization | Protobuf (binary) | JSON/XML (text) |
| Schema | Required (.proto) | Optional (OpenAPI) |
| Streaming | Native (client, server, bidi) | SSE, WebSocket hacks |
| Code gen | Mandatory | Optional (OpenAPI tools) |
| Browser support | Requires gRPC-web proxy | Native |
| Caching | No HTTP caching | HTTP caching |

### gRPC vs Message Queues (e.g., Kafka, RabbitMQ)

| Aspect | gRPC | Message Queue |
|---|---|---|
| Communication | Synchronous (blocking call) | Asynchronous (eventual) |
| Coupling | Tight (caller knows callee) | Loose (publisher → topic → consumer) |
| Latency | Sub-millisecond possible | Millisecond+ (persistence, broker hops) |
| Ordering | Per-stream ordering | Configurable partitioning |
| Use case | Request-response, real-time | Event-driven, decoupled, durable |

### Where gRPC Excels

- **Internal service-to-service**: high throughput, low latency, schema-enforced contracts
- **Microservices**: polyglot teams each own a service but share proto contracts
- **Streaming use cases**: real-time telemetry, log tailing, event subscriptions
- **Mobile clients**: binary is smaller than JSON; codegen reduces client bugs
- **Distributed systems**: Chronos nodes communicate scheduler state, job assignments, and Raft consensus via gRPC

> **Trap:** Thinking gRPC replaces REST entirely. It doesn't. REST is still the standard for public/external APIs where tooling diversity, browser compatibility, and HTTP caching matter. gRPC-web exists but adds operational complexity (eproxy/envoy). For Chronos, use gRPC for internal cluster communication and REST for a management HTTP API or webhook delivery.

> **Trap:** Using vanilla gRPC for browser clients without gRPC-web proxy. Browsers cannot send raw HTTP/2 frames. You need a proxy (Envoy, gRPC-web, or a gateway) to translate between browser HTTP/1.1 requests and gRPC HTTP/2 frames.

> **Follow-up:** *"In your Chronos project, which parts would you expose via gRPC vs REST?"*
> Answer: Internal node communication (heartbeats, job assignments, Raft replication) via gRPC. Admin dashboard, webhook delivery, and third-party integrations via REST.

---

## 2. Protocol Buffers — The Schema Language

Protocol Buffers is the Interface Definition Language (IDL) for gRPC. Every gRPC service starts with a `.proto` file.

### .proto File Structure

```protobuf
syntax = "proto3";

package scheduler.v1;

import "google/protobuf/timestamp.proto";
import "common/shared.proto";

option go_package = "github.com/chronos/api/schedulerv1";

message Job {
  string id = 1;
  string name = 2;
  google.protobuf.Timestamp created_at = 3;
  JobStatus status = 4;
  map<string, string> labels = 5;
  oneof schedule {
    string cron_expr = 6;
    fixed64 epoch_ms = 7;
  }
  repeated string tags = 8;
}

enum JobStatus {
  JOB_STATUS_UNSPECIFIED = 0;
  JOB_STATUS_PENDING = 1;
  JOB_STATUS_RUNNING = 2;
  JOB_STATUS_COMPLETED = 3;
  JOB_STATUS_FAILED = 4;
}

message JobConfig {
  int32 max_retries = 1;
  int64 timeout_ms = 2;
  string queue = 3;
}
```

### Scalar Types

| .proto Type | Go Type | Notes |
|---|---|---|
| `int32` / `int64` | `int32` / `int64` | Variable-length; negative values use more bytes |
| `uint32` / `uint64` | `uint32` / `uint64` | Variable-length unsigned |
| `sint32` / `sint64` | `int32` / `int64` | Zig-zag encoding; efficient for signed values |
| `fixed32` / `fixed64` | `uint32` / `uint64` | Always 4/8 bytes; fast for values > 2^28 |
| `sfixed32` / `sfixed64` | `int32` / `int64` | Always 4/8 bytes; signed variant |
| `float` / `double` | `float32` / `float64` | IEEE 754 |
| `bool` | `bool` | 1 byte |
| `string` | `string` | Must be UTF-8 or ASCII; length-delimited |
| `bytes` | `[]byte` | Any binary data |

### Message Structure

Each field in a message has:
- **Type** (scalar, enum, another message, map, repeated)
- **Name** (snake_case)
- **Field number** (unique within message, 1 to 2^29-1)

### `repeated` (Arrays/Lists)

```protobuf
message BatchCreateResponse {
  repeated Job jobs = 1;
  repeated string errors = 2;
}
```

### `map`

```protobuf
message JobResult {
  map<string, ExecutionMetric> metrics = 1;
}
```

Map keys must be scalar types (string, int, bool — no float, no enum). The map is unordered.

### `oneof`

Only one field in the `oneof` can be set at a time. Protobuf clears all others when a new one is set.

```protobuf
message ScheduleTrigger {
  oneof trigger {
    string cron_expr = 1;
    fixed64 unix_ms = 2;
    EventTrigger event = 3;
  }
}
```

### `enum`

Enums in proto3 must have a zero value. The first enum value is the default.

```protobuf
enum NodeRole {
  NODE_ROLE_UNSPECIFIED = 0;
  NODE_ROLE_LEADER = 1;
  NODE_ROLE_FOLLOWER = 2;
  NODE_ROLE_CANDIDATE = 3;
}
```

### Nested Messages

```protobuf
message ClusterStatus {
  message Node {
    string id = 1;
    string address = 2;
    NodeRole role = 3;
  }
  repeated Node nodes = 1;
  int32 term = 2;
}
```

### Importing Other Protos

```protobuf
import "google/protobuf/timestamp.proto";
import "chronos/common.proto";
```

### Well-Known Types

| Type | Package | Use |
|---|---|---|
| `Timestamp` | `google.protobuf.Timestamp` | Absolute point in time (seconds + nanos from epoch) |
| `Duration` | `google.protobuf.Duration` | Span of time |
| `Any` | `google.protobuf.Any` | Arbitrary serialized message (use sparingly) |
| `Empty` | `google.protobuf.Empty` | No fields — for void responses |
| `Struct` | `google.protobuf.Struct` | Dynamic JSON-like structure |

### JSON Mapping for Protobuf

Protobuf has a canonical JSON representation. This matters for debugging, logging, and REST-gRPC transcoding.

```protobuf
message Job {
  string id = 1;
  int64 retry_count = 2;
  JobStatus status = 3;
  google.protobuf.Timestamp created_at = 4;
}
```

Serializes to JSON as:

```json
{
  "id": "job-abc123",
  "retryCount": "3",
  "status": "JOB_STATUS_COMPLETED",
  "createdAt": "2026-07-26T12:00:00Z"
}
```

Key details:
- Field names become `camelCase` by default
- `int64`/`int64` values are strings (JavaScript number precision loss)
- Enums use the enum value name string
- `Timestamp` becomes RFC 3339 string
- `bytes` becomes base64-encoded string

> **Trap:** Reusing field numbers when deleting fields. This causes **silent data corruption** — old serialized data with the old field number gets deserialized into the new field with the same number. Always use `reserved`:

```protobuf
message Job {
  reserved 2, 4, 8 to 12;
  reserved "deprecated_field", "old_config";
  // ...
}
```

> **Trap:** Using `int32` for enum values. Proto3 enums are 32-bit signed integers internally, but you should never hardcode them. Use the enum name symbol — code generation ensures type safety.

> **Trap:** Changing a field from `string` to `bytes` is **not** wire-compatible (different wire type: string = wire type 2 LEN, bytes = wire type 2 LEN — actually they are compatible since both are length-delimited wire type 2). Changing from `int32` to `uint32` is incompatible (wire type 0 varint, but encoding differs for negative values). Changing `int32` to `fixed32` breaks because wire type changes (0 → 5). Always think in terms of **wire type**.

```protobuf
// Change from fixed32 to fixed64 — INCOMPATIBLE (wire type 5 vs wire type 1)
message Old { fixed32 id = 1; }
message New { fixed64 id = 1; } // BAD — breaks wire format
```

> **Follow-up:** *"Why would you use `sint32` vs `int32`?"*
> Answer: `int32` uses variable-length encoding. Small positive values (0–127) use 1 byte. But negative values use the maximum 10 bytes. `sint32` uses zig-zag encoding, mapping negative numbers to positive for more compact encoding. Use `sint32/sint64` when your values are frequently negative.

> **Follow-up:** *"Can you use `float` for field numbers?"*
> Answer: No. Field numbers must be integers in the range 1 to 536,870,911 (2^29-1). Float types are message field types, not field identifiers.

---

## 3. Field Numbers & Compatibility

### Why Field Numbers Matter

Field numbers are **not** just indices — they are the identity of a field on the wire. Each field's tag is: `(field_number << 3) | wire_type`. This 1-byte or multi-byte tag is what makes protobuf self-describing and allows forward/backward compatibility.

### Wire Types

| Type | Meaning | Used For |
|---|---|---|
| 0 | Varint | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1 | 64-bit | fixed64, sfixed64, double |
| 2 | Length-delimited | string, bytes, embedded messages, repeated fields |
| 3 | Start group | Deprecated (proto2 only) |
| 4 | End group | Deprecated (proto2 only) |
| 5 | 32-bit | fixed32, sfixed32, float |

### Forward/Backward Compatibility Rules

| Rule | Explanation |
|---|---|
| **Never reuse a field number** | Old data decodes into the wrong field — silent corruption |
| **Never change a field type** | Wire format changes — parse failures or garbled data |
| **Renaming a field is safe on-wire, but breaks code** | Wire uses numbers, not names. Codegen renames the Go struct field |
| **Adding new fields is safe** | Old code ignores unknown field numbers; new code gets default values |
| **Removing a field** | Use `reserved` to prevent re-use; old code gets default value |
| **Changing `repeated` to singular** | Compatible (read side treats repeated as singular — takes last)|
| **Changing singular to `repeated`** | Compatible (singular becomes a list of 1, but the wire format is different — actually this is compatible if you use packed encoding since repeated scalar becomes a length-delimited blob) |

### Field Number Ranges

| Range | Tag Size | Use |
|---|---|---|
| 1 – 15 | 1 byte (tag) | Most frequently used fields — optimize for small messages |
| 16 – 2047 | 2 bytes | Most other fields |
| 2048 – 16,383 | 3 bytes | Reserved for extensions or rarely set fields |
| 16,384 – 1,048,575 | 4 bytes | Rare |
| 1,048,576 – 536,870,911 | 5 bytes | Very rare |

> **Trap:** Thinking renaming a field is backwards compatible. It IS on the wire (field numbers don't change). But it breaks code compilation — every consumer must regenerate protos. Renaming is a build-time breaking change.

```protobuf
// v1
message Job { string name = 1; }

// v2 — BAD: code that reads "name" breaks
message Job { string job_name = 1; }
```

> **Trap:** Reusing a field number — worst bug in protobuf.

```protobuf
// v1
message Job { 
  string status = 5; // "pending", "running" 
}

// v1 data serialized with field 5 = "running"

// v2 — WRONG! Do not reuse field 5
message Job {
  string status = 5; // REMOVED THIS — field 5 was freed
  int32  retry_count = 5; // REUSED! Old data "running" decodes as bytes for retry_count
  // Result: retry_count = 0 (since "running" is not a valid varint, it gets skipped or errors)
}
```

Always use:

```protobuf
message Job {
  reserved 5;
  reserved "status";
  int32 retry_count = 6; // new number
}
```

> **Follow-up:** *"You're adding a new field to a proto used by 50 services. What do you do?"*
> Answer: Add the field with a new number. Deploy the **consumers** first (they ignore the unknown field — graceful). Then deploy the **producers**. For breaking changes (rename, delete, change type), you must coordinate with a migration: add the new field with a new number, deploy everyone, then remove the old field in a future version.

---

## 4. Service Definition

### The `service` Keyword

A gRPC service is defined in `.proto` with the `service` keyword. Each method is a unary or streaming RPC.

```protobuf
syntax = "proto3";

package chronos.scheduler.v1;

option go_package = "github.com/chronos/api/schedulerv1";

import "google/protobuf/empty.proto";
import "chronos/common.proto";

service SchedulerService {
  // Unary RPCs
  rpc CreateJob(CreateJobRequest) returns (Job);
  rpc GetJob(GetJobRequest) returns (Job);
  rpc DeleteJob(DeleteJobRequest) returns (google.protobuf.Empty);
  
  // Server-streaming RPC
  rpc WatchJobs(WatchJobsRequest) returns (stream JobEvent);
  
  // Client-streaming RPC
  rpc ReportBatchResults(stream JobResult) returns (BatchSummary);
  
  // Bidirectional streaming RPC
  rpc Heartbeat(stream HeartbeatRequest) returns (stream HeartbeatResponse);
}
```

### Package-Qualified Service Names

The fully-qualified service name becomes: `chronos.scheduler.v1.SchedulerService`

This is used in gRPC URLs: `/chronos.scheduler.v1.SchedulerService/CreateJob`

### Multiple Services Per Proto File

```protobuf
service SchedulerService { ... }
service AdminService { ... }
service HealthService { ... }
```

### Code Generation with `protoc`

```bash
protoc --proto_path=. \
  --go_out=. \
  --go_opt=paths=source_relative \
  --go-grpc_out=. \
  --go-grpc_opt=paths=source_relative \
  chronos/scheduler/v1/service.proto
```

This generates:
- `service.pb.go` — message types, marshalling/unmarshalling, registration
- `service_grpc.pb.go` — client interface, server interface, client struct, server registration function

### Generated Code Structure

```go
// Client interface (generated)
type SchedulerServiceClient interface {
  CreateJob(ctx context.Context, in *CreateJobRequest, opts ...grpc.CallOption) (*Job, error)
  GetJob(ctx context.Context, in *GetJobRequest, opts ...grpc.CallOption) (*Job, error)
  WatchJobs(ctx context.Context, in *WatchJobsRequest, opts ...grpc.CallOption) (SchedulerService_WatchJobsClient, error)
}

// Server interface (generated)
type SchedulerServiceServer interface {
  CreateJob(context.Context, *CreateJobRequest) (*Job, error)
  GetJob(context.Context, *GetJobRequest) (*Job, error)
  WatchJobs(*WatchJobsRequest, SchedulerService_WatchJobsServer) error
  mustEmbedUnimplementedSchedulerServiceServer()
}
```

### Buf CLI

Modern gRPC projects increasingly use **Buf** instead of raw `protoc`:

```bash
# buf.gen.yaml
version: v1
managed:
  enabled: true
plugins:
  - plugin: go
    out: gen
    opt: paths=source_relative
  - plugin: go-grpc
    out: gen
    opt: paths=source_relative
```

```bash
buf generate
```

Buf provides:
- Breaking change detection (`buf breaking`)
- Linting (`buf lint`)
- Dependency management (`buf mod`)
- BSR (Buf Schema Registry) for publishing and sharing protos

> **Trap:** Not generating code in CI/CD. Hand-written stubs drift from the `.proto` definition. Always generate code in CI and fail the build if `git diff` is non-empty (protos are the source of truth).

```yaml
# CI step
- run: buf generate
- run: test -z "$(git diff)" || (echo "Generated code out of sync" && exit 1)
```

> **Trap:** Poor proto file organization. Common antipatterns:
> - One giant `everything.proto` (million imports, slow compilation, merge conflicts)
> - One file per message (too many imports)
> - Circular imports (not allowed — organize in layers: common types → domain messages → service definitions)

Best practice:

```
proto/
  chronos/
    scheduler/
      v1/
        scheduler.proto    # service definition + RPC-specific messages
    common/
      v1/
        common.proto       # shared enums, messages (TimedEntity, Error, etc.)
    health/
      v1/
        health.proto       # health check service
```

> **Follow-up:** *"Can you share protos across repositories?"*
> Answer: Yes, but it adds complexity. Options:
> 1. Mono-repo for proto definitions (simplest)
> 2. Buf Schema Registry (BSR) — publish protos as modules, consume as dependencies
> 3. Git submodule — fragile, easy to desync
> 4. Copy protos with CI validation — works but violates DRY
> For Chronos, a single proto repository or BSR is recommended since multiple services (scheduler, worker nodes, admin API) share the same contracts.

---

## 5. Unary RPC in Go

A complete, real-world example from concept to running server and client.

### Step 1: Proto Definition

```protobuf
syntax = "proto3";

package chronos.scheduler.v1;

option go_package = "github.com/chronos/api/schedulerv1";

service JobService {
  rpc CreateJob(CreateJobRequest) returns (Job);
  rpc GetJob(GetJobRequest) returns (Job);
}

message CreateJobRequest {
  string name = 1;
  string command = 2;
  string cron_expr = 3;
  int32 max_retries = 4;
  map<string, string> labels = 5;
}

message GetJobRequest {
  string job_id = 1;
}

message Job {
  string id = 1;
  string name = 2;
  string command = 3;
  string cron_expr = 4;
  int32 max_retries = 5;
  JobStatus status = 6;
  map<string, string> labels = 7;
  google.protobuf.Timestamp created_at = 8;
}

enum JobStatus {
  JOB_STATUS_UNSPECIFIED = 0;
  JOB_STATUS_PENDING = 1;
  JOB_STATUS_RUNNING = 2;
  JOB_STATUS_COMPLETED = 3;
  JOB_STATUS_FAILED = 4;
}
```

### Step 2: Generate Code

```bash
protoc --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  proto/chronos/scheduler/v1/job.proto
```

### Step 3: Server Implementation

```go
package server

import (
  "context"
  "fmt"
  "log"
  "net"

  "github.com/chronos/api/schedulerv1"
  "google.golang.org/grpc"
  "google.golang.org/grpc/reflection"
)

type jobServer struct {
  schedulerv1.UnimplementedJobServiceServer
  // your storage — could be a raft-backed store for Chronos
  store map[string]*schedulerv1.Job
}

func (s *jobServer) CreateJob(ctx context.Context, req *schedulerv1.CreateJobRequest) (*schedulerv1.Job, error) {
  job := &schedulerv1.Job{
    Id:        generateID(),
    Name:      req.Name,
    Command:   req.Command,
    CronExpr:  req.CronExpr,
    MaxRetries: req.MaxRetries,
    Labels:    req.Labels,
    Status:    schedulerv1.JobStatus_JOB_STATUS_PENDING,
    CreatedAt: timestamppb.Now(),
  }
  s.store[job.Id] = job
  log.Printf("job created: %s", job.Id)
  return job, nil
}

func (s *jobServer) GetJob(ctx context.Context, req *schedulerv1.GetJobRequest) (*schedulerv1.Job, error) {
  job, ok := s.store[req.JobId]
  if !ok {
    return nil, status.Errorf(codes.NotFound, "job %s not found", req.JobId)
  }
  return job, nil
}

func RunGRPCServer(addr string) error {
  lis, err := net.Listen("tcp", addr)
  if err != nil {
    return fmt.Errorf("failed to listen on %s: %w", addr, err)
  }

  srv := grpc.NewServer(
    grpc.MaxRecvMsgSize(10*1024*1024), // 10 MB
    grpc.UnaryInterceptor(unaryLoggingInterceptor),
  )

  schedulerv1.RegisterJobServiceServer(srv, &jobServer{
    store: make(map[string]*schedulerv1.Job),
  })

  // Enable server reflection (grpcurl, Postman, debugging)
  reflection.Register(srv)

  log.Printf("gRPC server listening on %s", addr)
  return srv.Serve(lis)
}

func unaryLoggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
  log.Printf("--> %s", info.FullMethod)
  resp, err := handler(ctx, req)
  if err != nil {
    log.Printf("<-- %s ERROR: %v", info.FullMethod, err)
  } else {
    log.Printf("<-- %s OK", info.FullMethod)
  }
  return resp, err
}
```

### Step 4: Client

```go
package client

import (
  "context"
  "log"
  "time"

  "github.com/chronos/api/schedulerv1"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials/insecure"
)

func CreateJobExample(addr string) {
  // Dial — use insecure for development only
  conn, err := grpc.Dial(
    addr,
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithBlock(),
    grpc.WithTimeout(5*time.Second),
  )
  if err != nil {
    log.Fatalf("did not connect: %v", err)
  }
  defer conn.Close()

  client := schedulerv1.NewJobServiceClient(conn)

  ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
  defer cancel()

  job, err := client.CreateJob(ctx, &schedulerv1.CreateJobRequest{
    Name:       "nightly-backup",
    Command:    "/usr/bin/backup.sh",
    CronExpr:   "0 3 * * *",
    MaxRetries: 3,
    Labels:     map[string]string{"env": "production"},
  })
  if err != nil {
    log.Fatalf("CreateJob failed: %v", err)
  }

  log.Printf("Job created: id=%s, status=%s", job.Id, job.Status)
}
```

### Context Propagation

gRPC clients propagate context through the call chain. Context carries:
- **Deadline**: `context.WithTimeout` or `context.WithDeadline` — server sees `ctx.Deadline()` and can preempt
- **Cancellation**: `context.WithCancel` — server sees `ctx.Done()` and stops work
- **Metadata**: key-value pairs (headers) via `metadata.NewOutgoingContext` / `metadata.FromIncomingContext`
- **Values**: arbitrary values via `context.WithValue` (use sparingly — prefer metadata)

```go
// Client: attach metadata
md := metadata.Pairs("authorization", "Bearer "+token)
ctx := metadata.NewOutgoingContext(context.Background(), md)
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

resp, err := client.SomeRPC(ctx, req)
```

```go
// Server: read metadata
func (s *server) SomeRPC(ctx context.Context, req *pb.Request) (*pb.Response, error) {
  md, ok := metadata.FromIncomingContext(ctx)
  if !ok {
    return nil, status.Errorf(codes.Unauthenticated, "no metadata")
  }
  tokens := md.Get("authorization")
  // ...
}
```

### Key Server/Client Options

**Server:**
- `grpc.UnaryInterceptor` / `grpc.StreamInterceptor` — middleware chain
- `grpc.MaxRecvMsgSize` — limit incoming message size (default 4 MB)
- `grpc.MaxSendMsgSize` — limit outgoing message size
- `grpc.Creds` — TLS credentials
- `grpc.KeepaliveParams` — keepalive ping settings
- `grpc.KeepaliveEnforcementPolicy` — enforce min ping interval, reject abusive clients
- `grpc.StatsHandler` — connection/method stats
- `grpc.UnknownServiceHandler` — custom handler for unknown methods

**Client:**
- `grpc.WithTransportCredentials` — TLS (use `insecure.NewCredentials()` for dev only)
- `grpc.WithBlock` — block until connection established
- `grpc.WithTimeout` — dial timeout (deprecated in favor of context)
- `grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(...))` — per-call limits
- `grpc.WithUnaryInterceptor` / `grpc.WithChainUnaryInterceptor` — client-side interceptors
- `grpc.WithStreamInterceptor` / `grpc.WithChainStreamInterceptor`
- `grpc.WithDefaultServiceConfig` — set load balancing policy (round_robin, pick_first)
- `grpc.WithConnectParams` — configure min connect timeout and backoff

### Connection Management and Reconnection

gRPC clients maintain connection pools and handle reconnection automatically. The default behavior uses exponential backoff:

```go
// Default backoff configuration (can be overridden via grpc.WithConnectParams)
connectParams := grpc.ConnectParams{
  MinConnectTimeout: 5 * time.Second, // initial timeout
  Backoff:           backoff.DefaultConfig, // exponential backoff
}
conn, err := grpc.Dial(addr, grpc.WithConnectParams(connectParams))
```

The backoff sequence starts at ~1 second and doubles with jitter up to ~120 seconds. If a connection drops, the client retries transparently. This is critical for Chronos nodes that may temporarily lose connectivity during leader elections or network partitions.

### Graceful Shutdown

```go
func ServeGracefully(addr string, srv *grpc.Server) {
  lis, err := net.Listen("tcp", addr)
  if err != nil {
    log.Fatalf("failed to listen: %v", err)
  }

  // Channel to listen for signals
  quit := make(chan os.Signal, 1)
  signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

  // Serve in background
  go func() {
    log.Printf("server listening on %s", addr)
    if err := srv.Serve(lis); err != nil {
      log.Fatalf("serve error: %v", err)
    }
  }()

  <-quit
  log.Println("shutting down gracefully...")

  // Give active RPCs a deadline to finish
  done := make(chan struct{})
  go func() {
    srv.GracefulStop() // wait for active RPCs to complete
    close(done)
  }()

  select {
  case <-done:
    log.Println("graceful shutdown complete")
  case <-time.After(30 * time.Second):
    log.Println("graceful shutdown timed out, forcing stop")
    srv.Stop() // force-stop remaining RPCs
  }
}
```

The two-phase shutdown pattern is essential in production: first `GracefulStop()` (drains in-flight requests, rejects new ones), then if it takes too long, `Stop()` (hard close). Chronos nodes must coordinate shutdown with Raft leader transfer before stopping the gRPC server.

### Error Handling and Status Codes

gRPC defines a canonical set of status codes. Every RPC returns a status code and an optional message.

```go
import (
  "google.golang.org/grpc/codes"
  "google.golang.org/grpc/status"
)

// Server: return a proper status error
func (s *jobServer) GetJob(ctx context.Context, req *schedulerv1.GetJobRequest) (*schedulerv1.Job, error) {
  if req.JobId == "" {
    return nil, status.Errorf(codes.InvalidArgument, "job_id is required")
  }
  job, ok := s.store[req.JobId]
  if !ok {
    return nil, status.Errorf(codes.NotFound, "job %s not found", req.JobId)
  }
  return job, nil
}

// Client: check status codes
job, err := client.GetJob(ctx, &schedulerv1.GetJobRequest{JobId: "unknown"})
if err != nil {
  st, ok := status.FromError(err)
  if !ok {
    log.Fatalf("non-grpc error: %v", err)
  }
  switch st.Code() {
  case codes.NotFound:
    log.Println("job not found, that's ok")
  case codes.Unavailable:
    log.Println("server is down, retry later")
  default:
    log.Printf("unexpected error: %v", st.Message())
  }
}
```

Common status codes you'll use daily:

| Code | Number | When |
|---|---|---|
| `OK` | 0 | Success |
| `Canceled` | 1 | Context cancelled |
| `InvalidArgument` | 3 | Bad request data |
| `DeadlineExceeded` | 4 | Timeout |
| `NotFound` | 5 | Resource missing |
| `AlreadyExists` | 6 | Duplicate |
| `PermissionDenied` | 7 | Auth failure |
| `Unavailable` | 14 | Server down, transient |
| `Internal` | 13 | Server bug |
| `Unauthenticated` | 16 | Bad credentials |

> **Trap:** Forgetting to close the listener `defer lis.Close()` does not shut down in-progress requests. Use `srv.GracefulStop()` in a signal handler to wait for active RPCs, and a secondary `srv.Stop()` with a timeout to force-stop.

```go
func main() {
  lis, _ := net.Listen("tcp", ":50051")
  srv := grpc.NewServer()
  // register services...

  go func() {
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, os.Interrupt, syscall.SIGTERM)
    <-sig
    log.Println("shutting down gracefully...")
    srv.GracefulStop() // wait for active RPCs
  }()

  log.Fatal(srv.Serve(lis))
}
```

> **Trap:** Using `grpc.WithInsecure()` in production — this sends data unencrypted. Never use this outside local dev. Always use TLS certificates in production.

```go
// PRODUCTION — ALWAYS use TLS
creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
srv := grpc.NewServer(grpc.Creds(creds))

creds, err := credentials.NewClientTLSFromFile("ca.crt", "server.chronos.internal")
conn, err := grpc.Dial(addr, grpc.WithTransportCredentials(creds))
```

> **Trap:** Not handling server startup errors. `srv.Serve(lis)` can return errors (port already bound, TLS handshake failures, etc.). Always check the return value.

> **Follow-up:** *"How does gRPC handle connection management when a server restarts?"*
> Answer: Clients keep pools of connections. When a server disconnects, the client's health checking can detect it. With `grpc.WithDefaultServiceConfig`, clients can use round_robin or pick_first load balancing. In Kubernetes, use a headless service with a gRPC-aware load balancer like Envoy or Linkerd. For Chronos, nodes would re-establish connections on leader election events triggered by Raft.

---

## 6. HTTP/2 Foundations

gRPC is built on HTTP/2. Understanding HTTP/2 is essential for debugging performance issues.

### HTTP/2 vs HTTP/1.1

| Feature | HTTP/1.1 | HTTP/2 |
|---|---|---|
| Framing | Text-based | Binary |
| Multiplexing | No (6 parallel connections max) | Yes (multiple streams on one connection) |
| Head-of-line blocking | Application layer (request waits for previous) | Streams are independent (at application layer) |
| Header compression | None | HPACK (compresses headers, often 90%+ reduction) |
| Server push | No | Yes (server sends resources proactively) |
| Stream priority | No | Yes (weighted priority tree) |
| Protocol overhead | High (text parsing) | Low (binary framing) |

### How gRPC Uses HTTP/2

```
[gRPC Client] ── TCP Connection ──> [gRPC Server]
               ├── Stream 1: CreateJob (HEADERS + DATA + TRAILERS)
               ├── Stream 2: GetJob (HEADERS + DATA + TRAILERS)
               ├── Stream 3: Heartbeat (HEADERS + DATA ... DATA ... TRAILERS)
               └── Stream 4: WatchJobs (HEADERS + DATA + DATA + ...)
```

- One long-lived TCP connection per client (typically)
- Multiple "streams" multiplexed over that connection
- Each gRPC RPC is one HTTP/2 stream
- Headers contain method path, content-type (`application/grpc`), metadata
- Data frames carry protobuf payloads (with a 5-byte gRPC frame prefix: 1 byte compressed flag + 4 byte length)
- Trailers carry gRPC status code and status message

### gRPC Frame Format (on top of HTTP/2 DATA frames)

```
┌──────────┬───────────────┬──────────────────────────────────┐
│ 1 byte   │ 4 bytes       │ N bytes                          │
│ Compressed?│ Message length│ Protobuf message                 │
│ (0/1)    │ (big-endian)  │                                  │
└──────────┴───────────────┴──────────────────────────────────┘
```

### Stream Lifecycle

1. **Client sends HEADERS frame** — `:method POST`, `:path /package.Service/Method`, `content-type: application/grpc`, `te: trailers`, metadata
2. **Client sends DATA frames** — protobuf payload (for unary: one DATA frame; for streaming: multiple)
3. **Client sends empty DATA with END_STREAM** — signals client is done sending
4. **Server sends HEADERS** — response metadata
5. **Server sends DATA frames** — protobuf response payload(s)
6. **Server sends HEADERS with END_STREAM** — trailers containing `grpc-status` and `grpc-message`

### Pseudocode Flow

```
CLIENT                              SERVER
  │                                  │
  │--- HEADERS (method, path, te) -->│
  │--- DATA (protobuf request) ----->│
  │--- HEADERS (END_STREAM) -------->│
  │                                  │--- process request
  │<-- HEADERS (response metadata) --│
  │<-- DATA (protobuf response) -----│
  │<-- HEADERS (grpc-status: 0, -----│
  │          grpc-message: "OK")     │
```

> **Trap:** HTTP/2 has head-of-line blocking at the TCP layer, not the application layer. Since HTTP/2 streams are multiplexed over a single TCP connection, **one dropped packet blocks ALL streams** (TCP's in-order delivery requirement). This is why HTTP/3 exists — it runs over QUIC (UDP), where each stream is independent. For gRPC, this matters when packet loss is high; use HTTP/3 for lossy networks (mobile, edge).

> **Trap:** Not tuning HTTP/2 settings for gRPC workloads. Default HTTP/2 settings in many proxies and servers are tuned for web pages, not long-lived gRPC streams.

```go
// Tuning HTTP/2 for gRPC (Go net/http2)
import "golang.org/x/net/http2"

server := &http.Server{
  // ...
}
http2.ConfigureServer(server, &http2.Server{
  MaxConcurrentStreams: 1000,
  InitialWindowSize:    1048576, // 1 MB — increase from 64KB default
  InitialConnWindowSize: 6291456, // 6 MB — increase from 64KB default
})
```

Key tunables:
- `MaxConcurrentStreams` — limit concurrent RPCs on one connection (default: unlimited in Go, 100 in most proxies)
- `InitialWindowSize` — per-stream flow control window (default 64KB — increase for streaming)
- `InitialConnWindowSize` — per-connection flow control window

> **Trap:** Assuming HTTP/2 is always faster than HTTP/1.1. For a single small request (like a simple gRPC unary call with no streaming), HTTP/2 adds framing overhead. The real wins come with:
> 1. Many concurrent requests (multiplexing)
> 2. Streaming (no head-of-line blocking)
> 3. Repeated connections to the same server (connection reuse)
> 4. Small messages + many requests (HPACK header compression saves bytes)

> **Follow-up:** *"How would you debug gRPC performance issues at the HTTP/2 level?"*
> Answer: Use `grpcurl` with verbose flags, `tcpdump`/Wireshark with HTTP/2 decoding, gRPC built-in channelz (`grpcz`), or Envoy's gRPC stats. Look for: high connection count (should reuse connections), flow control window starvation (small windows kill streaming throughput), stream resets (indicates server overload or client cancellations).

---

## 7. gRPC vs REST vs GraphQL

### Detailed Comparison

| Dimension | gRPC | REST | GraphQL |
|---|---|---|---|
| **Serialization** | Protobuf (binary) — fast, compact | JSON (text) — slower, larger | JSON (text) — same as REST |
| **Schema** | Required (.proto) — strict contract | Optional (OpenAPI) — often loose | Required (SDL) — strict contract |
| **Streaming** | Native: client, server, bidi | Hacky: SSE, WebSocket, polling | Subscriptions via WebSocket |
| **Code generation** | Required, robust | Optional, tooling varies | Optional, Apollo/Relay exist |
| **Browser support** | gRPC-web only (needs proxy) | Native (fetch, axios) | Native (fetch, Apollo Client) |
| **HTTP caching** | Not available (no HTTP semantics) | Full HTTP caching (ETag, Cache-Control) | Limited (Apollo cache, no HTTP CDN caching) |
| **Debuggability** | Hard: binary on wire, need grpcurl, protobuf decoder | Easy: curl, browser dev tools, Postman | Medium: GraphiQL, Apollo Studio |
| **Payload size** | Very small | Medium (text JSON) | Medium (text JSON, over-fetching can be worse) |
| **Performance** | Very high | Moderate | Moderate |
| **Tooling maturity** | Mature for server, growing for debugging | Extremely mature | Mature |
| **Error handling** | Status codes (canonical 1-16) | HTTP status codes + body | `errors` array in response |
| **Versioning** | Via package/field deprecation (no /v1/ in URL) | URL path or header versioning | Deprecation in schema |

### Decision Framework

| Use Case | Best Choice | Why |
|---|---|---|
| **Internal microservices** | gRPC | Performance, schema contracts, streaming |
| **Public/external APIT** | REST | HTTP caching, curl, browser native, wide ecosystem |
| **Mobile clients** | gRPC (gRPC-web proxy) | Binary smaller, type safety, fewer client bugs |
| **Complex data fetching (dashboard)** | GraphQL | Multiple data sources, client-specific shapes |
| **Event-driven/streaming** | gRPC (bidi streaming) | Native streaming, backpressure, cancellation |
| **IoT/edge devices** | gRPC | Binary, low bandwidth, connection reuse |
| **CDN-served APIs** | REST | HTTP caching at edge (CloudFront, Cloudflare) |
| **File uploads** | REST (multipart) | gRPC requires byte streaming — doable but REST is simpler |

> **Trap:** Exposing gRPC as a public API without a proxy. Browsers can't call gRPC directly. You need Envoy/gRPC-web/gRPC-gateway to translate HTTP/1.1 to HTTP/2. Debugging is harder (binary wire format). If your users expect `curl`, they'll be frustrated.

> **Trap:** Using REST for high-throughput internal communication (e.g., Chronos nodes talking to each other). REST's JSON serialization is 3-10x slower than protobuf for large payloads. No schema contract means field name typos are runtime errors. No native streaming means you need SSE or polling for real-time updates. This is exactly the use case gRPC was designed for.

> **Follow-up:** *"When would you combine gRPC and REST in the same system?"*
> Answer: Common architecture: **gRPC internally, REST externally**. Each microservice exposes a gRPC endpoint for inter-service communication. The API gateway (e.g., Envoy, Kong, or a custom Go gateway) translates external REST/JSON into internal gRPC calls. This gives you performance internally and compatibility externally. For Chronos: gRPC between scheduler nodes and workers. REST API for the admin dashboard and CI/CD webhook integrations.

> **Follow-up:** *"Can you use gRPC for real-time notifications to web clients?"*
> Answer: Yes, but with caveats. Use gRPC-web (client streaming) through a proxy like Envoy. Or better: use gRPC internally and expose Server-Sent Events (SSE) from the API gateway. The gateway subscribes to a gRPC server-streaming endpoint and converts each event to SSE for the browser. This is a common pattern at scale.

### Real-World Architecture: gRPC + REST Gateway

```
                         ┌──────────────┐
Browser / curl ──HTTP──> │   Gateway    │
Mobile ──HTTP──>          │ (Envoy/Kong) │
                         └──────┬───────┘
                                │ gRPC (internal)
                    ┌───────────┼───────────┐
                    │           │           │
              ┌─────▼──┐  ┌────▼───┐  ┌───▼────┐
              │Service A│  │Service B│  │Service C│
              │ (gRPC)  │  │ (gRPC)  │  │ (gRPC)  │
              └─────────┘  └────────┘  └────────┘
```

The gateway:
1. Receives HTTP/1.1 REST/JSON requests
2. Transcodes to protobuf (based on `google.api.http` annotations or manual mapping)
3. Makes gRPC calls to internal services
4. Transcodes response back to JSON

For Chronos, this means:
- **gRPC**: scheduler ↔ worker nodes (Heartbeat, AssignJob, ReportResult, Raft consensus messages)
- **gRPC**: scheduler ↔ scheduler (Raft replication via gRPC streaming)
- **REST**: admin UI, webhook delivery, third-party CI/CD integrations
- **REST**: health check endpoints (Kubernetes probes don't speak gRPC by default — use `grpc-health-probe`)

### Performance Benchmark (Approximate)

| Operation | REST (JSON) | gRPC (protobuf) | Ratio |
|---|---|---|---|
| Serialization (1KB message) | ~2 µs | ~0.2 µs | 10x |
| Deserialization (1KB message) | ~3 µs | ~0.3 µs | 10x |
| Payload size (1KB object) | ~1.2 KB | ~200 B | 6x |
| Throughput (concurrent) | ~10K req/s | ~50K req/s | 5x |
| Connection memory | ~10 MB / 100 connections | ~1 MB / 100 connections (multiplexed) | 10x |

Numbers are approximate and vary by hardware, message structure, and implementation. The key insight: gRPC's advantages compound at scale. For Chronos with potentially thousands of job executions per second, the savings in CPU and bandwidth are significant.

---

## 8. Protocol Buffers Best Practices

### File Organization

```
proto/
  chronos/
    scheduler/
      v1/
        service.proto         # service definition
        scheduler.proto       # scheduler-specific messages
        job.proto             # job-related types
    common/
      v1/
        common.proto          # shared types across all services
        errors.proto          # error details
  google/
    type/
      datetime.proto          # well-known types (usually vendored)
```

### Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Package | lowercase, dotted | `chronos.scheduler.v1` |
| Message | PascalCase | `CreateJobRequest` |
| Field | snake_case | `max_retries` |
| Enum | PascalCase | `JobStatus` |
| Enum value | UPPER_SNAKE_CASE | `JOB_STATUS_COMPLETED` |
| Service | PascalCase | `SchedulerService` |
| RPC | PascalCase | `CreateJob` |
| File name | snake_case | `scheduler_service.proto` |

### Avoid Breaking Changes

**Safe changes (backward compatible):**
- Add a new field with a new, unused field number
- Add a new enum value (consumers ignore unknown values)
- Add a new oneof variant
- Rename a field name (wire uses field numbers, but breaks codegen)
- Add a new service method

**Breaking changes (not backward compatible):**
- Change a field number
- Change a field type (different wire format)
- Remove a field (use `reserved` instead)
- Rename a package or service (changes wire path)
- Change a oneof to a regular field or vice versa

### Enums and Default Values

In proto3, the first enum value (index 0) is the **default**. Unset enum fields serialize to the zero value.

```protobuf
// BAD: first value is a real state
enum JobStatus {
  JOB_STATUS_PENDING = 0;      // 0 is default — pending is a real state
  JOB_STATUS_RUNNING = 1;
}
// Unset defaults to PENDING — dangerous! Did someone set pending, or was it unset?

// GOOD: first value is UNSPECIFIED
enum JobStatus {
  JOB_STATUS_UNSPECIFIED = 0;  // default — explicitly not set
  JOB_STATUS_PENDING = 1;
  JOB_STATUS_RUNNING = 2;
}
```

### `google.protobuf.Empty` vs No Fields

```protobuf
// Clear intent — DeleteJob intentionally returns nothing
rpc DeleteJob(DeleteJobRequest) returns (google.protobuf.Empty);

// Future-proof — if you later need to return something, you can't
rpc DeleteJob(DeleteJobRequest) returns (DeleteJobResponse); // empty message
```

Prefer `google.protobuf.Empty` for void returns. If you might add response fields later, define an empty response message.

### Nullable Fields with `wrappers.proto`

Proto3 does not have native null — unset fields use their zero value. To distinguish "not set" from "set to zero/false/empty", use wrapper types:

```protobuf
import "google/protobuf/wrappers.proto";

message JobConfig {
  google.protobuf.Int32Value max_retries = 1;  // nil = not set, 0 = explicitly 0
  google.protobuf.StringValue description = 2; // nil = not set, "" = explicitly empty
  google.protobuf.BoolValue enabled = 3;       // nil = not set, false = explicitly false
}
```

In Go:

```go
config.MaxRetries == nil           // not set
config.MaxRetries.GetValue() == 0  // explicitly zero
```

### Documentation with Proto Comments

Proto files are the source of truth — document them rigorously. Comments in `.proto` files flow into generated code and can be exported to documentation.

```protobuf
// Job represents a scheduled task in the Chronos system.
// Each job is tracked by the scheduler and assigned to workers
// based on the configured routing strategy.
message Job {
  // Unique identifier for the job (UUID v4, 16 bytes).
  bytes id = 1;

  // Human-readable name. Must match regex: ^[a-zA-Z0-9_-]{1,64}$
  string name = 2;

  // cron_expr in standard 5-field cron format (min hour day mon weekday).
  // Example: "*/5 * * * *" = every 5 minutes.
  optional string cron_expr = 3;

  // Current lifecycle status of the job.
  JobStatus status = 4;

  // Arbitrary key-value pairs for routing, filtering, and metadata.
  // Keys must be lowercase alphanumeric + underscore.
  map<string, string> labels = 5;
}
```

### Avoiding Common Pitfalls with `oneof`

`oneof` fields reset all other fields in the oneof when set. This can cause subtle bugs:

```protobuf
message Alert {
  oneof channel {
    string email = 1;
    string webhook_url = 2;
    PagerDutyConfig pagerduty = 3;
  }
  string email = 4; // DIFFERENT field name — this is fine
}

// WRONG — this causes confusion
message AlertBad {
  oneof channel {
    string email = 1;
    string webhook = 2;
  }
  bool email_enabled = 3; // This is fine — different field number
}
```

`oneof` fields get special generated code — you check `GetChannel()` which returns the `isAlert_Channel` interface, then type-switch or use the typed getters.

### Deprecation Strategy

```protobuf
message JobConfig {
  option deprecated = true;

  int32 max_retries = 1;
  int64 timeout_ms = 2 [deprecated = true]; // use duration field instead
  google.protobuf.Duration max_execution_time = 3; // replacement
}
```

Mark deprecated fields and messages with `deprecated = true`. This triggers compiler warnings in consumers and documents the migration path. Remove deprecated fields only after verifying no consumers reference them (use Buf's breaking change detection).

### Performance Considerations

| Field Type | Wire Size | Use Case |
|---|---|---|
| `int32` (value 0-127) | 1 byte | Small counters |
| `sint32` (zigzag) | 1 byte for 0-63 | Any signed value |
| `fixed32` | 4 bytes | Values > 2^28 consistently |
| `string` (short) | 2 + len bytes | IDs, names |
| `bytes` (UUID) | 1 + 16 bytes ~ 17 bytes | Binary IDs |
| `enum` | 1-2 bytes | State machine |

```protobuf
// Good: efficient
message Job {
  bytes id = 1;          // UUID as 16 bytes
  sint32 priority = 2;   // small signed value
  fixed64 unix_ms = 3;   // timestamps are large, fixed size
}

// Not as efficient
message Job {
  string id = 1;          // UUID as 36-byte string
  int32 priority = 2;     // not zigzag — negative values waste bytes
  int64 unix_ms = 3;      // varint fine, but fixed64 is faster for large numbers
}
```

> **Trap:** Using `int64` for timestamps instead of `google.protobuf.Timestamp`. The well-known type provides standardized semantics, JSON conversion (RFC 3339), and interoperability with other protobuf consumers. Same for `Duration` vs manual `int64` nanosecond fields.

> **Trap:** Using `string` for UUIDs instead of `bytes`. A UUID as a string is 36 bytes (e.g., `"550e8400-e29b-41d4-a716-446655440000"`). As bytes, it's 16 bytes + wire overhead. That's more than 2x savings, which matters in high-throughput systems like Chronos's Raft log replication.

```protobuf
// Prefer
message Job { bytes id = 1; [(google.api.field_behavior) = REQUIRED]; }

// Over
message Job { string id = 1; }
```

> **Trap:** Using enum value 0 as a real semantic state. As discussed above — always reserve 0 for `UNSPECIFIED`.

> **Follow-up:** *"You have a message with 30 fields. How do you decide field number assignments?"*
> Answer: Assign 1-15 to the most frequently serialized/deserialized fields. Use 16+ for rarely-set or optional fields. Leave gaps between logical groups (1-10 for identity, 11-20 for config, 21+ for metadata) so you can add frequently-accessed fields in the 1-15 range later. Document field number ranges in a comment or spec.

---

## 9. Tier 1 Q&A Drill

1. **Q: What are the three fundamental building blocks of gRPC?**
   **A:** HTTP/2 (transport), Protocol Buffers (serialization), and code generation (type-safe stubs).

2. **Q: Why can't you just use `int32` for all numeric proto fields?**
   **A:** `int32` uses variable-length varint encoding — small values save bytes but negative values waste 10 bytes. For signed values that can be negative, use `sint32` (zig-zag). For large values consistently above 2^28, use `fixed32/fixed64` for faster encoding at a fixed size.

3. **Q: What happens if two services read the same proto but one adds a field?**
   **A:** The service with the old proto ignores the unknown field number during deserialization. The field is preserved in the binary but not accessible in the generated struct (unless you use `proto.Unmarshal` with `protojson` or deserialize into the new type). This is forward compatibility.

4. **Q: Can I rename a proto field?**
   **A:** Yes, on the wire (field numbers don't change). But renaming breaks all consumers' code — they reference the old field name. Renaming is a build-time breaking change. Fix: add a new field with a new number, deprecate the old one, migrate all consumers, then remove.

5. **Q: What's the difference between `repeated` fields in proto3 and arrays in JSON?**
   **A:** In proto3, `repeated` scalar fields use **packed encoding** by default (a single length-delimited blob of all values). This is more compact. Non-packed encoding (proto2 default) repeats the tag+value for each element. For messages, `repeated` is always a list of length-delimited messages.

6. **Q: How does gRPC handle timeouts?**
   **A:** Clients set a deadline via `context.WithTimeout` or `context.WithDeadline`. The deadline is sent to the server as part of the `grpc-timeout` header. The server checks `ctx.Deadline()` and should stop processing if the deadline has passed. If the server doesn't respect the context, the client disconnects.

7. **Q: What is the purpose of the `grpc.WithBlock()` dial option?**
   **A:** It makes `grpc.Dial` block until the connection is established (or timeout/error). Without it, `grpc.Dial` returns immediately and connects lazily — the first RPC might fail or be delayed. Use `WithBlock()` in scenarios where you want fail-fast behavior on startup.

8. **Q: How do you handle authentication in gRPC?**
   **A:** Three approaches: (1) TLS mutual auth (mTLS) at the transport level — `grpc.WithTransportCredentials`. (2) Metadata-based tokens — attach `authorization` header via `metadata.NewOutgoingContext`, validate in a server interceptor. (3) Google Cloud IAM or similar for cloud deployments. Per-RPC credentials via `grpc.PerRPCCredentials` interface.

9. **Q: What's the difference between `google.protobuf.Empty` and a message with no fields?**
   **A:** They're semantically the same wire format. Use `Empty` for clarity when an RPC intentionally returns nothing. If you might add return fields later, define a named empty message type so you can add fields without changing the RPC signature.

10. **Q: How would you implement a health check for a gRPC service?**
    **A:** Use the standard `grpc.health.v1.Health` service from `grpc-health-probing`. Implement `Check` (unary) and `Watch` (server-streaming) RPCs. gRPC clients can use health checking to automatically reconnect. Kubernetes can use `grpc_health_probe` as a liveness/readiness probe.

11. **Q: Can two `.proto` files import each other?**
    **A:** No — circular imports are not allowed. Organize proto dependencies as a DAG: common types → domain messages → service definitions. If A depends on B and B depends on A, extract the common types into a shared package.

12. **Q: What encoding does a proto field with type `bytes` use on the wire?**
    **A:** Wire type 2 (length-delimited). On the wire, it's: tag (field_number << 3 \| 0x02) + varint length + raw bytes. Same wire type as `string` — the difference is only in semantics (UTF-8 validation for strings) and generated code (`[]byte` vs `string`).
