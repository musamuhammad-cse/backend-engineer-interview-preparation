# gRPC — Question Bank & Drill Material

> Senior Backend Engineer interview prep (Go, PHP, JS) — 8+ years experience. Questions are tied to **Chronos**, a Go distributed job scheduler using Raft for consensus, gRPC for service communication, and Protocol Buffers for data contracts.

---

## Table of Contents

1. [Rapid-Fire: gRPC Fundamentals & Protocol Buffers (30 questions)](#1-rapid-fire-grpc-fundamentals--protocol-buffers-30-questions)
2. [Rapid-Fire: Streaming & Error Handling (30 questions)](#2-rapid-fire-streaming--error-handling-30-questions)
3. [Rapid-Fire: Interceptors & Auth (20 questions)](#3-rapid-fire-interceptors--auth-20-questions)
4. [Rapid-Fire: Production & Operations (30 questions)](#4-rapid-fire-production--operations-30-questions)
5. [Code Puzzles (6-8 puzzles)](#5-code-puzzles-6-8-puzzles)
6. [Live-Coding Exercises (4-5 exercises)](#6-live-coding-exercises-4-5-exercises)
7. [Debugging Scenarios (4-5 scenarios)](#7-debugging-scenarios-4-5-scenarios)
8. [System Design Prompts (4-5 prompts)](#8-system-design-prompts-4-5-prompts)
9. [STAR Stories (3 templates)](#9-star-stories-3-templates)
10. [Questions to Ask the Interviewer (8-10 questions)](#10-questions-to-ask-the-interviewer-8-10-questions)
11. [Red Flags to Avoid (8-10 red flags)](#11-red-flags-to-avoid-8-10-red-flags)

---

## 1. Rapid-Fire: gRPC Fundamentals & Protocol Buffers (30 questions)

**Q1:** What is gRPC and how does it differ from REST?

<details><summary>Answer</summary>

gRPC is a high-performance RPC framework by Google using HTTP/2 transport and Protocol Buffers as the IDL. Unlike REST (which is resource-oriented, uses JSON, and relies on HTTP methods), gRPC is action-oriented, defines services with typed methods, uses binary serialization for performance, and supports streaming natively.

**Trap:** gRPC is not always faster than REST. For simple CRUD with small payloads, the HTTP/2 overhead and protobuf encoding cost can negate benefits. Measure before migrating.

**Follow-up:** When would you choose gRPC over REST for Chronos?  
*When you need typed contracts (job definitions), streaming (worker heartbeats), and low-latency internal communication between scheduler and workers.*
</details>

---

**Q2:** Why does gRPC require HTTP/2?

<details><summary>Answer</summary>

HTTP/2 provides multiplexed streams (multiple RPC calls over a single TCP connection), server push, bidirectional streaming, header compression (HPACK), and flow control — all essential for gRPC's streaming model and performance.

**Trap:** Saying "gRPC needs HTTP/2 for binary frames" — while true, the key enabler is *multiplexed streams* so one connection handles many concurrent RPCs without head-of-line blocking.
</details>

---

**Q3:** What is a `.proto` file? Write a minimal example for a `Job` message used in Chronos.

<details><summary>Answer</summary>

A `.proto` file is the IDL for defining protobuf message types and gRPC services.

```protobuf
syntax = "proto3";

package chronos;

message Job {
  string id = 1;
  string name = 2;
  string queue = 3;
  int32 priority = 4;
  bytes payload = 5;
}
```

**Trap:** Forgetting `syntax = "proto3"` — proto3 is required for gRPC; proto2 lacks many features gRPC relies on.
</details>

---

**Q4:** What are field numbers in protobuf and why do they matter?

<details><summary>Answer</summary>

Field numbers are unique integer identifiers for each field in a message (1 to 536,870,911). They are encoded in the binary wire format — numbers 1-15 use 1 byte, 16-2047 use 2 bytes. They are immutable once the message is in use; changing a field number breaks wire compatibility.

**Trap:** Field numbers are *not* emitted to consumers — they are a wire-format optimization. Changing a field name is backward compatible; changing its number is *not*.

**Follow-up:** Why would Chronos reserve field numbers 1-15 for frequently-used fields?  
*To save wire bytes — Job IDs and statuses that appear in every message should use low field numbers.*
</details>

---

**Q5:** What scalar types does proto3 support?

<details><summary>Answer</summary>

`double`, `float`, `int32`, `int64`, `uint32`, `uint64`, `sint32`, `sint64`, `fixed32`, `fixed64`, `sfixed32`, `sfixed64`, `bool`, `string`, `bytes`.

**Trap:** `int32` uses variable-length encoding — negative values are inefficient. Use `sint32` for signed negative-heavy data.
</details>

---

**Q6:** Explain `repeated` and `map` in proto3.

<details><summary>Answer</summary>

`repeated` is a packed list of a type (like an array). `map<key_type, value_type>` is an associative map — internally it's syntactic sugar for a repeated entry message. In proto3, `repeated` is packed by default (length-delimited encoding).

```protobuf
message JobBatch {
  repeated Job jobs = 1;
  map<string, string> labels = 2;
}
```

**Trap:** `map` cannot be `repeated` and keys cannot be `bytes` or floating-point types.
</details>

---

**Q7:** What is `oneof` in protobuf?

<details><summary>Answer</summary>

`oneof` defines a set of fields where at most one can be set at a time. Useful for union types or variant payloads.

```protobuf
message JobResult {
  string job_id = 1;
  oneof result {
    bytes output = 2;
    string error_message = 3;
  }
}
```

**Trap:** Setting any field in a `oneof` clears all other fields. This is a runtime behavior, not just a validation constraint.
</details>

---

**Q8:** How are enums defined and handled in proto3?

<details><summary>Answer</summary>

```protobuf
enum JobStatus {
  JOB_STATUS_UNSPECIFIED = 0;
  JOB_STATUS_PENDING = 1;
  JOB_STATUS_RUNNING = 2;
  JOB_STATUS_COMPLETED = 3;
  JOB_STATUS_FAILED = 4;
}
```

In proto3, enums are open — unknown values are preserved at runtime (they don't cause parse errors). The first enum value must be 0 (used as default).

**Trap:** The zero-value default means an unset enum field is indistinguishable from `UNSPECIFIED`. Avoid using 0 for a meaningful state.
</details>

---

**Q9:** How does protobuf code generation work with `protoc`?

<details><summary>Answer</summary>

`protoc` reads `.proto` files and generates source code in the target language via plugins (`protoc-gen-go`, `protoc-gen-java`, etc.). In Go, you typically use `protoc-gen-go-grpc` for service stubs.

```bash
protoc --go_out=. --go-grpc_out=. proto/chronos/v1/job.proto
```

**Trap:** gRPC service generation requires a separate plugin (`protoc-gen-go-grpc`), not just `protoc-gen-go`.
</details>

---

**Q10:** Define a gRPC service in proto3 for Chronos job submission.

<details><summary>Answer</summary>

```protobuf
service Chronos {
  rpc SubmitJob(SubmitJobRequest) returns (SubmitJobResponse);
}
```

**Trap:** gRPC services do not support method overloading — each method name must be unique. No inheritance or generics.
</details>

---

**Q11:** What is a unary RPC?

<details><summary>Answer</summary>

A single request → single response call. Like a regular function call over the network. It's the simplest gRPC pattern and the default.

**Trap:** Unary RPC still uses HTTP/2 streams internally — it just sends one message each way.
</details>

---

**Q12:** What are the key differences between proto3 and proto2?

<details><summary>Answer</summary>

Proto3 drops: required/optional keywords, default values in definitions, custom options for field types (except well-known types), unknown field preservation (re-added in proto3 via `google.protobuf.Any`), and extensions (replaced by `Any`). Proto3 also requires zero default for enums and makes all fields optional.

**Trap:** Proto3 does not support `required` — validation must happen at the application layer.
</details>

---

**Q13:** What are `google.protobuf.Timestamp` and `google.protobuf.Duration`?

<details><summary>Answer</summary>

Well-known types that provide standardized time handling:
- `Timestamp`: seconds + nanos since Unix epoch (UTC, not timezone-aware)
- `Duration`: signed seconds + nanos (can be negative)

```protobuf
import "google/protobuf/timestamp.proto";

message Job {
  string id = 1;
  google.protobuf.Timestamp created_at = 2;
  google.protobuf.Duration timeout = 3;
}
```

**Trap:** Don't use `int64` for timestamps — `Timestamp` provides standardized handling, JSON serialization (RFC 3339), and language-native conversions.
</details>

---

**Q14:** What is `google.protobuf.Any`?

<details><summary>Answer</summary>

A well-known type that can hold any protobuf message as a type-erased blob with its type URL.

```protobuf
import "google/protobuf/any.proto";

message JobResult {
  string job_id = 1;
  google.protobuf.Any detail = 2;
}
```

**Trap:** `Any` requires careful type checking on unpacking. Overuse leads to fragile contracts — prefer explicit `oneof` when the set of possible types is known.
</details>

---

**Q15:** How does protobuf handle JSON serialization for a message?

<details><summary>Answer</summary>

Proto3 defines a canonical JSON mapping: field names use camelCase, enums use string names, default values are omitted, bytes are base64-encoded, Timestamps are RFC 3339 strings.

```json
{
  "id": "job-001",
  "name": "nightly-backup",
  "priority": 10
}
```

**Trap:** Default values (0 for ints, empty string, false) are omitted from JSON output by default. Use `emit_defaults` if needed.
</details>

---

**Q16:** What is `grpc.WithInsecure()` in Go and why should you avoid it in production?

<details><summary>Answer</summary>

It creates a gRPC channel without TLS — plaintext HTTP/2. Production deployments should always use TLS/mTLS for transport security and authentication. gRPC requires TLS by default for HTTP/2.

**Trap:** Many assume `WithInsecure()` is fine for internal services in a VPC. It exposes you to MITM attacks and violates compliance. Use mTLS for cluster-internal service mesh.
</details>

---

**Q17:** What package import path does Go protobuf use?

<details><summary>Answer</summary>

Generated code uses `google.golang.org/protobuf` (v2 API, `protoimpl`) and `google.golang.org/grpc`. The legacy `github.com/golang/protobuf` is deprecated.

**Trap:** Mixing the old and new protobuf libraries in the same binary causes type conflicts. Ensure all dependencies use `google.golang.org/protobuf`.
</details>

---

**Q18:** What is the maximum message size in gRPC?

<details><summary>Answer</summary>

Default is 4 MB for both send and receive. Configurable via `grpc.MaxCallRecvMsgSize` and `grpc.MaxCallSendMsgSize` on the client, and `MaxRecvMsgSize`/`MaxSendMsgSize` on the server.

**Trap:** Silently hitting the limit causes `ResourceExhausted` errors. Chronos job payloads should be limited or chunked.
</details>

---

**Q19:** How does gRPC map HTTP/2 concepts (stream, frame, message)?

<details><summary>Answer</summary>

A gRPC RPC uses one HTTP/2 stream. Each gRPC message is serialized into one or more DATA frames. Headers are sent in HEADERS frames, trailers in HEADERS (or a separate HEADERS after END_STREAM). gRPC prefixes each message with a 5-byte header (1 byte compressed flag + 4 byte length).

**Trap:** A single gRPC message can span multiple DATA frames, and multiple gRPC messages can share DATA frames within a stream.
</details>

---

**Q20:** What is `grpc-gateway` and when would you use it with Chronos?

<details><summary>Answer</summary>

`grpc-gateway` generates a reverse-proxy that serves RESTful JSON endpoints from gRPC service definitions, using `google.api.http` annotations in proto files.

```protobuf
import "google/api/annotations.proto";

service Chronos {
  rpc SubmitJob(SubmitJobRequest) returns (SubmitJobResponse) {
    option (google.api.http) = {
      post: "/v1/jobs"
      body: "*"
    };
  }
}
```

**Trap:** grpc-gateway adds latency (JSON ↔ protobuf transcoding). Use it only for external-facing APIs where REST is required.
</details>

---

**Q21:** What happens when a proto file has `import` statements?

<details><summary>Answer</summary>

Protobuf allows importing types from other `.proto` files. The import path is resolved by `protoc` using `-I` flags. In Go, the generated code imports the corresponding Go packages.

**Trap:** Circular imports are not allowed. Move shared types to a common proto package.
</details>

---

**Q22:** How does proto3 handle default values?

<details><summary>Answer</summary>

Proto3 assigns default values: 0 for numerics, empty string, false for bool, empty bytes, and nil for messages/enums. Default values are *not* sent on the wire, which saves bandwidth.

**Trap:** There is no way to distinguish "set to 0" from "not set" for scalar fields without wrapping in a message or using wrapper types like `google.protobuf.Int32Value`.
</details>

---

**Q23:** What are wrapper types in protobuf?

<details><summary>Answer</summary>

Wrapper types (`google.protobuf.StringValue`, `google.protobuf.Int32Value`, etc.) wrap scalars in messages so they can be optional/nullable.

```protobuf
import "google/protobuf/wrappers.proto";

message Job {
  google.protobuf.StringValue description = 1;  // nullable
}
```

**Trap:** Wrapper types add overhead (message framing). Use sparingly and only when you need to distinguish unset from zero.
</details>

---

**Q24:** What is `protoc-gen-validate`?

<details><summary>Answer</summary>

A protoc plugin that generates validation logic from annotations in proto files:

```protobuf
message SubmitJobRequest {
  string name = 1 [(validate.rules).string.min_len = 1];
  int32 priority = 2 [(validate.rules).int32.gte = 0];
}
```

**Trap:** Validation runs on the server, not the client by default. You must configure interceptors to invoke validation middleware.
</details>

---

**Q25:** How does gRPC handle the `User-Agent` header?

<details><summary>Answer</summary>

gRPC automatically sets a `user-agent` header: `grpc-go/<version> <language>/<version>`. Custom clients should append their own identifier.

**Trap:** Do not override the user-agent entirely; append to it so you can still identify the gRPC framework version for debugging.
</details>

---

**Q26:** What is `gRPC reflection` and how does `grpcurl` use it?

<details><summary>Answer</summary>

The `grpc.reflection.v1alpha.ServerReflection` service allows clients to discover services, methods, and message types at runtime without needing the `.proto` file. `grpcurl` uses reflection to invoke RPCs dynamically.

```bash
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext localhost:50051 describe chronos.v1.Job
grpcurl -plaintext -d '{"name":"test"}' localhost:50051 chronos.v1.Chronos/SubmitJob
```

**Trap:** Never enable reflection in production without auth — it leaks your entire API surface.
</details>

---

**Q27:** What is the `Accept-Encoding` behavior in gRPC?

<details><summary>Answer</summary>

gRPC supports `gzip` compression. The client advertises supported encodings via the `grpc-accept-encoding` header; the server compresses responses if the client accepts it.

**Trap:** Compression adds CPU overhead. For internal low-latency services (Chronos worker ↔ scheduler), compression may hurt more than help. Benchmark with real payloads.
</details>

---

**Q28:** Can you define messages in proto3 without a package?

<details><summary>Answer</summary>

Yes, but highly discouraged. The `package` declaration prevents name collisions and is used for generating language-appropriate namespaces (Go package name, Java package, etc.).

**Trap:** Two proto files without packages could silently collide when imported together.
</details>

---

**Q29:** What is `google.protobuf.Empty`?

<details><summary>Answer</summary>

An empty message defined in `google/protobuf/empty.proto`. Used for RPCs that take no parameters or return no data.

```protobuf
rpc Ping(google.protobuf.Empty) returns (google.protobuf.Empty);
```

**Trap:** Using `Empty` makes it harder to evolve the API — you can't add fields. Prefer a dedicated message type even if it starts empty (e.g., `HealthCheckRequest`).
</details>

---

**Q30:** What is the difference between `proto3` and `proto2` field presence?

<details><summary>Answer</summary>

Proto2 had `required`/`optional` explicit labels. Proto3 makes all fields optional with no `required` keyword. Presence tracking is only available for message fields and oneofs. For scalars, there's no presence — zero is the default.

**Follow-up:** How does `google.protobuf.FieldMask` help with partial updates?  
*FieldMask allows clients to specify which fields to read/update, reducing payload size and preventing accidental overwrites.*
</details>

---

## 2. Rapid-Fire: Streaming & Error Handling (30 questions)

**Q1:** What are the three types of streaming in gRPC?

<details><summary>Answer</summary>

1. **Server-side streaming** — client sends one request, server sends multiple responses.
2. **Client-side streaming** — client sends multiple requests, server sends one response.
3. **Bidirectional streaming** — both sides send multiple messages independently.
</details>

---

**Q2:** Write a proto definition for a server-side streaming RPC in Chronos that streams job status updates.

<details><summary>Answer</summary>

```protobuf
service Chronos {
  rpc WatchJobStatus(WatchJobStatusRequest) returns (stream JobStatusEvent);
}

message WatchJobStatusRequest {
  string job_id = 1;
}

message JobStatusEvent {
  string job_id = 1;
  JobStatus status = 2;
  google.protobuf.Timestamp timestamp = 3;
}
```

**Trap:** The `stream` keyword goes on the *response*, not the request.
</details>

---

**Q3:** What is bidirectional streaming and when would you use it in Chronos?

<details><summary>Answer</summary>

Both client and server send messages independently over the same stream. Chronos uses this for worker-scheduler coordination — workers stream heartbeats while the scheduler streams assignments:

```protobuf
service Chronos {
  rpc WorkerCoordinator(stream WorkerHeartbeat) returns (stream TaskAssignment);
}
```

**Trap:** Bidirectional streaming does *not* guarantee message ordering between the two directions. Each direction is ordered independently.
</details>

---

**Q4:** How does gRPC handle flow control in streaming?

<details><summary>Answer</summary>

gRPC inherits HTTP/2 flow control. Each stream has a window (default ~64KB) and connections have a larger window (default ~64MB). The receiver advertises window updates. When the window is exhausted, the sender must wait.

**Trap:** A slow consumer that doesn't read from the stream will eventually cause the producer to block via backpressure. This is *good* — it prevents OOM.
</details>

---

**Q5:** What are deadlines and timeouts in gRPC?

<details><summary>Answer</summary>

A deadline is the absolute time by which a client expects a response. Timeouts are relative durations. In Go:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
resp, err := client.SubmitJob(ctx, req)
```

**Trap:** Deadlines are propagated from client to server via the `grpc-timeout` header. If the client doesn't set a deadline, the server has no upper bound on execution time.
</details>

---

**Q6:** How are deadlines propagated across gRPC calls?

<details><summary>Answer</summary>

The gRPC framework automatically propagates `grpc-timeout` metadata from client to server. The server's context deadline is derived from this value. If the server makes downstream gRPC calls, it should pass the same context.

**Trap:** A server that ignores the context's deadline and continues processing wastes resources. Always check `ctx.Err()` in long-running operations.
</details>

---

**Q7:** What are metadata in gRPC?

<details><summary>Answer</summary>

Key-value pairs sent as HTTP/2 headers/trailers alongside RPC calls. Used for auth tokens, request IDs, tracing headers, etc.

```go
md := metadata.Pairs(
  "authorization", "Bearer <token>",
  "x-request-id", uuid.New().String(),
)
ctx := metadata.NewOutgoingContext(ctx, md)
```

**Trap:** Metadata keys are case-insensitive per HTTP/2 spec. Stick to lowercase to avoid surprises with different gRPC implementations.
</details>

---

**Q8:** List the 17 gRPC status codes (for all gRPC implementations).

<details><summary>Answer</summary>

1. `OK` (0)
2. `CANCELLED` (1)
3. `UNKNOWN` (2)
4. `INVALID_ARGUMENT` (3)
5. `DEADLINE_EXCEEDED` (4)
6. `NOT_FOUND` (5)
7. `ALREADY_EXISTS` (6)
8. `PERMISSION_DENIED` (7)
9. `RESOURCE_EXHAUSTED` (8)
10. `FAILED_PRECONDITION` (9)
11. `ABORTED` (10)
12. `OUT_OF_RANGE` (11)
13. `UNIMPLEMENTED` (12)
14. `INTERNAL` (13)
15. `UNAVAILABLE` (14)
16. `DATA_LOSS` (15)
17. `UNAUTHENTICATED` (16)
</details>

---

**Q9:** When should you use `UNAVAILABLE` vs `INTERNAL` vs `UNKNOWN`?

<details><summary>Answer</summary>

- **`UNAVAILABLE`** (14): Service is temporarily down (e.g., server shutting down, load balancer health check fails). Safe to retry.
- **`INTERNAL`** (13): Server crash, unexpected panic, nil pointer. Not safe to retry without investigation.
- **`UNKNOWN`** (2): Can't determine the error — a catch-all.

**Trap:** Returning `INTERNAL` for an invalid user request leaks implementation details. Use `INVALID_ARGUMENT` for bad input.
</details>

---

**Q10:** What is the gRPC rich error model?

<details><summary>Answer</summary>

An extended error mechanism using `google.rpc.Status` which includes a code, message, and repeated `google.protobuf.Any` details.

```protobuf
import "google/rpc/status.proto";
import "google/rpc/error_details.proto";
```

**Trap:** The rich error model is not supported by all gRPC implementations. Falling back to simple status codes ensures broader compatibility.
</details>

---

**Q11:** What error detail types does Google provide?

<details><summary>Answer</summary>

- `RetryInfo` — suggests retry interval
- `DebugInfo` — stack traces for debugging
- `QuotaFailure` — quota exceeded details
- `ErrorInfo` — structured error reason + domain
- `PreconditionFailure` — what precondition failed
- `BadRequest` — field-level validation errors
- `RequestInfo` — request ID for support
- `ResourceInfo` — which resource caused the error
- `Help` — links to documentation
- `LocalizedMessage` — localized error message
</details>

---

**Q12:** How do you attach error details to a gRPC response in Go?

<details><summary>Answer</summary>

```go
import (
  "google.golang.org/grpc/codes"
  "google.golang.org/grpc/status"
  "google.golang.org/genproto/googleapis/rpc/errdetails"
)

st := status.New(codes.InvalidArgument, "validation failed")
br := &errdetails.BadRequest{
  FieldViolations: []*errdetails.BadRequest_FieldViolation{
    {Field: "priority", Description: "must be >= 0"},
  },
}
st, _ = st.WithDetails(br)
return nil, st.Err()
```

**Trap:** `WithDetails` returns an error if the detail type is already attached. Check the error.
</details>

---

**Q13:** What is the `context.Context` role in gRPC Go?

<details><summary>Answer</summary>

Every gRPC call in Go takes a `context.Context` first argument. It carries cancellation signals, deadlines, and metadata. The server receives a derived context that inherits the client's deadline and can be cancelled.

```go
// Server side
func (s *ChronosServer) SubmitJob(ctx context.Context, req *pb.SubmitJobRequest) (*pb.SubmitJobResponse, error) {
  select {
  case <-ctx.Done():
    return nil, status.Errorf(codes.Canceled, "request cancelled")
  case result := <-s.process(req):
    return result, nil
  }
}
```

**Trap:** Never ignore `ctx.Done()` in a streaming handler — the stream will leak goroutines if you don't clean up.
</details>

---

**Q14:** How does a client detect a cancelled/deadline-exceeded stream?

<details><summary>Answer</summary>

The `Recv()` call returns `io.EOF` on clean stream completion. On cancellation, it returns the gRPC status error (`codes.Canceled` or `codes.DeadlineExceeded`).

```go
for {
  event, err := stream.Recv()
  if err == io.EOF {
    break
  }
  if err != nil {
    st, _ := status.FromError(err)
    log.Printf("stream error: %s", st.Message())
    break
  }
  handle(event)
}
```

**Trap:** `io.EOF` is the *only* signal for a clean end. Any non-EOF error is a stream failure.
</details>

---

**Q15:** What happens if a server sends a message after the client has cancelled?

<details><summary>Answer</summary>

The server's `Send()` will return `io.EOF` or a `Canceled` error. The server should check `stream.Context().Err()` periodically.

**Trap:** Ignoring the send error and continuing to produce messages wastes resources. Always check the error from `Send()`.
</details>

---

**Q16:** What is gRPC flow control window sizing?

<details><summary>Answer</summary>

Initial stream window: 64KB, connection window: 64MB. These can be tuned with `grpc.InitialWindowSize` and `grpc.InitialConnWindowSize`.

**Trap:** Too-small windows cause excessive window update frames and degrade throughput on high-latency links. For Chronos streaming heartbeats (small, frequent messages), the default is fine.
</details>

---

**Q17:** What is the gRPC `MaxConcurrentStreams` setting?

<details><summary>Answer</summary>

A server-side limit on the number of concurrent streams per connection. Configurable in Go via `grpc.MaxConcurrentStreams`. Default is 100 in Go gRPC.

**Trap:** If your service handles many long-lived streaming connections (e.g., thousands of Chronos workers), increase this or you'll get `Unavailable` errors as connections fill up.
</details>

---

**Q18:** How do you check if a gRPC error is retryable?

<details><summary>Answer</summary>

Generally, `UNAVAILABLE`, `DEADLINE_EXCEEDED`, `RESOURCE_EXHAUSTED`, and `ABORTED` are retryable. `INVALID_ARGUMENT`, `PERMISSION_DENIED`, `UNAUTHENTICATED` are not. gRPC's built-in retry policy (see Q3 in Production section) follows this convention.

**Trap:** `DEADLINE_EXCEEDED` on a non-idempotent write RPC should NOT be retried automatically — the work may have completed on the server.
</details>

---

**Q19:** What is the difference between `Header` and `Trailer` in gRPC?

<details><summary>Answer</summary>

Headers are sent at the start of an RPC (HTTP/2 HEADERS frame). Trailers are sent after all response messages, at the end of the RPC (HTTP/2 HEADERS with END_STREAM). Status code and status message are sent as trailers.

**Trap:** Server streaming responses must ensure trailers are sent *after* the last message. An error before sending headers will use header-based error delivery instead.
</details>

---

**Q20:** Can you set response headers from a server interceptor?

<details><summary>Answer</summary>

Yes, using `grpc.SetHeader()` or `grpc.SendHeader()` on the server-side context/stream. Interceptors can set headers that propagate to the client.

```go
func (s *interceptor) Unary() grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    grpc.SetHeader(ctx, metadata.Pairs("x-request-id", reqID))
    return handler(ctx, req)
  }
}
```

**Trap:** `SetHeader` can be called multiple times before the first `SendHeader`, but `SendHeader` sends them immediately and subsequent calls will fail.
</details>

---

**Q21:** How does gRPC handle `CANCEL` status propagation?

<details><summary>Answer</summary>

When a client cancels a context, gRPC sends RST_STREAM to the server. The server's stream context is cancelled. If the server is blocked on `Recv()`, it returns `Canceled`. If it's in the middle of processing, it checks `ctx.Done()`.

**Trap:** HTTP/2 error code `CANCEL` (0x8) is used. This is different from `RST_STREAM` due to protocol errors.
</details>

---

**Q22:** What is the `grpc-status-details-bin` header?

<details><summary>Answer</summary>

A binary header that carries the serialized `google.rpc.Status` for rich error details. Sent in trailers. Not all clients parse it — use a helper like `status.FromError()`.

**Trap:** Rich error details are optional; always code against the simple status code first.
</details>

---

**Q23:** How do you extract error details on the client in Go?

<details><summary>Answer</summary>

```go
st, ok := status.FromError(err)
if !ok {
  // not a gRPC error
}
for _, detail := range st.Details() {
  switch d := detail.(type) {
  case *errdetails.BadRequest:
    for _, v := range d.FieldViolations {
      log.Printf("field %s: %s", v.Field, v.Description)
    }
  }
}
```

**Trap:** `status.FromError()` returns `ok=false` for non-gRPC errors (e.g., network errors, `io.EOF` in non-streaming contexts).
</details>

---

**Q24:** What is the `codes.Canceled` error and when is it triggered on the server?

<details><summary>Answer</summary>

Triggered when the client cancels the RPC (context cancellation), the server calls `cancel()` on its derived context, or the HTTP/2 stream is reset. The server should stop processing immediately.

**Trap:** `Canceled` is different from `DeadlineExceeded` — cancellation is explicit, deadline is a timeout.
</details>

---

**Q25:** What happens to an in-flight unary call if the server panics?

<details><summary>Answer</summary>

The gRPC server recovers the panic by default (in `grpc-go`), logs the stack trace, and returns `Internal` to the client. Recovery can be disabled via `grpc.UnaryPanicHandler`.

**Trap:** Panic recovery doesn't clean up resources. Use `defer` for cleanup or a recovery interceptor that handles resource finalization.
</details>

---

**Q26:** How does gRPC distinguish between a completed stream and a failed one?

<details><summary>Answer</summary>

On the wire, trailers carry `grpc-status` and `grpc-message`. A status of 0 (`OK`) means success. Non-zero means error. The stream is complete after trailers are received.

**Trap:** If the server terminates the stream without sending trailers (e.g., connection drops), the client sees `Unavailable` instead.
</details>

---

**Q27:** What is `grpc.WaitForReady`?

<details><summary>Answer</summary>

A client call option that makes the RPC wait for the connection to become ready (transient failure → connecting → ready) instead of failing immediately with `Unavailable`.

```go
resp, err := client.SubmitJob(ctx, req, grpc.WaitForReady(true))
```

**Trap:** `WaitForReady` with an indefinite context can block forever. Always pair it with a deadline.
</details>

---

**Q28:** Can a gRPC client send metadata mid-stream?

<details><summary>Answer</summary>

No — metadata is sent as HTTP/2 headers at the start of the stream (or trailers at the end). To send additional data mid-stream, use out-of-band messages in bidirectional streaming.

**Trap:** Attempting to send headers mid-stream will cause a gRPC internal error.
</details>

---

**Q29:** How does gRPC handle large messages that exceed the max message size?

<details><summary>Answer</summary>

The server returns `ResourceExhausted` with message "grpc: received message larger than max (<size> vs. <max>)". The client's `Recv()` returns the same error.

**Trap:** Chronos job payloads over 4MB need either chunking (custom splitting) or increased limits + streaming.
</details>

---

**Q30:** What is `grpc.BufferPool` and when does it matter?

<details><summary>Answer</summary>

A memory pool for gRPC message buffers to reduce GC pressure. Available in `google.golang.org/grpc/mem`. Useful for high-throughput gRPC servers.

**Trap:** Default buffer pool behavior is fine for most services. Tune only if profiling shows excessive allocs in buffer operations.
</details>

---

## 3. Rapid-Fire: Interceptors & Auth (20 questions)

**Q1:** What is a gRPC interceptor in Go?

<details><summary>Answer</summary>

A middleware function that wraps RPC calls — similar to HTTP middleware but for gRPC methods. Interceptors can inspect/modify requests, responses, context, and metadata.

```go
func unaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
  log.Printf("method: %s", info.FullMethod)
  return handler(ctx, req)
}
```
</details>

---

**Q2:** What is the difference between a unary interceptor and a stream interceptor?

<details><summary>Answer</summary>

- **Unary interceptor**: intercepts `(context, request) → (response, error)`. Simple function wrapper.
- **Stream interceptor**: intercepts a `ServerStream` or `ClientStream` — wraps `Recv()` and `Send()` calls for each message.
</details>

---

**Q3:** Write a server-side logging interceptor for Chronos.

<details><summary>Answer</summary>

```go
func LoggingUnaryInterceptor(logger *zap.Logger) grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    dur := time.Since(start)
    code := codes.Unknown
    if err != nil {
      code = status.Code(err)
    }
    logger.Info("gRPC call",
      zap.String("method", info.FullMethod),
      zap.Duration("duration", dur),
      zap.String("code", code.String()),
    )
    return resp, err
  }
}
```

**Trap:** Never log the request body in production — it may contain sensitive data.
</details>

---

**Q4:** How do you chain multiple interceptors in gRPC Go?

<details><summary>Answer</summary>

```go
server := grpc.NewServer(
  grpc.ChainUnaryInterceptor(
    AuthInterceptor,
    RateLimitInterceptor,
    LoggingInterceptor,
  ),
)
```

Execution order: Auth → RateLimit → Logging → handler → Logging → RateLimit → Auth (like onion middleware).

**Trap:** `grpc.ChainUnaryInterceptor` executes interceptors in the order provided. The *last* registered interceptor runs closest to the handler.
</details>

---

**Q5:** What is the client-side interceptor signature?

<details><summary>Answer</summary>

```go
func clientInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
  start := time.Now()
  err := invoker(ctx, method, req, reply, cc, opts...)
  log.Printf("method=%s duration=%s err=%v", method, time.Since(start), err)
  return err
}
```
</details>

---

**Q6:** Write a stream interceptor that adds a request ID to metadata.

<details><summary>Answer</summary>

```go
func StreamRequestIDInterceptor() grpc.StreamServerInterceptor {
  return func(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
    md, ok := metadata.FromIncomingContext(ss.Context())
    if !ok {
      md = metadata.New(nil)
    }
    if _, exists := md["x-request-id"]; !exists {
      md.Set("x-request-id", uuid.New().String())
    }
    ctx := metadata.NewIncomingContext(ss.Context(), md)
    ws := &wrappedStream{ServerStream: ss, ctx: ctx}
    return handler(srv, ws)
  }
}

type wrappedStream struct {
  grpc.ServerStream
  ctx context.Context
}

func (w *wrappedStream) Context() context.Context {
  return w.ctx
}
```
</details>

---

**Q7:** How does TLS/mTLS work with gRPC?

<details><summary>Answer</summary>

```go
// Server
creds, _ := credentials.NewServerTLSFromFile("server.crt", "server.key")
server := grpc.NewServer(grpc.Creds(creds))

// Client with mTLS — client cert required
cert, _ := tls.LoadX509KeyPair("client.crt", "client.key")
creds := credentials.NewTLS(&tls.Config{
  Certificates: []tls.Certificate{cert},
  ServerName:   "chronos.internal",
})
conn, _ := grpc.Dial("chronos.internal:50051", grpc.WithTransportCredentials(creds))
```

**Trap:** For mTLS, the client must provide a `tls.Config` with a `Certificates` slice AND the server must set `ClientAuth: tls.RequireAndVerifyClientCert`.
</details>

---

**Q8:** How do you implement a JWT auth interceptor for gRPC?

<details><summary>Answer</summary>

```go
func AuthInterceptor(validator func(token string) (UserClaims, error)) grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
      return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }
    tokens := md.Get("authorization")
    if len(tokens) == 0 {
      return nil, status.Error(codes.Unauthenticated, "missing token")
    }
    claims, err := validator(strings.TrimPrefix(tokens[0], "Bearer "))
    if err != nil {
      return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }
    ctx = context.WithValue(ctx, userClaimsKey, claims)
    return handler(ctx, req)
  }
}
```

**Trap:** Never validate JWT in the application handler — use an interceptor so all methods are covered uniformly.
</details>

---

**Q9:** How does gRPC handle per-RPC credentials (e.g., OAuth2)?

<details><summary>Answer</summary>

Via `grpc.PerRPCCredentials` interface:

```go
type tokenAuth struct {
  token string
}

func (t tokenAuth) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
  return map[string]string{"authorization": "Bearer " + t.token}, nil
}

func (tokenAuth) RequireTransportSecurity() bool {
  return true
}

conn, _ := grpc.Dial("chronos:50051",
  grpc.WithPerRPCCredentials(tokenAuth{token: "xxx"}),
)
```
</details>

---

**Q10:** What is the difference between `grpc.WithPerRPCCredentials` and a client interceptor for auth?

<details><summary>Answer</summary>

`WithPerRPCCredentials` is the standard gRPC auth interface — it's transport-aware and integrates with TLS. A client interceptor gives more control (e.g., fetching tokens from a vault, caching, retry on 401). Use `WithPerRPCCredentials` for simple static tokens; use an interceptor for complex flows.

**Trap:** `RequireTransportSecurity()` returning `false` will allow sending credentials over plaintext — a security risk.
</details>

---

**Q11:** How do you skip authentication for certain methods in an interceptor?

<details><summary>Answer</summary>

```go
var skipAuth = map[string]bool{
  "/chronos.v1.Chronos/HealthCheck": true,
}

func AuthInterceptor(validator TokenValidator) grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    if skipAuth[info.FullMethod] {
      return handler(ctx, req)
    }
    // validate...
    return handler(ctx, req)
  }
}
```

**Trap:** Use the `info.FullMethod` string (e.g., `/package.Service/Method`) — not just the method name.
</details>

---

**Q12:** Can a stream interceptor modify messages on the fly?

<details><summary>Answer</summary>

Yes — wrap the `ServerStream` to intercept `Send`/`Recv` calls:

```go
type loggingStream struct {
  grpc.ServerStream
}

func (s *loggingStream) Send(m interface{}) error {
  log.Printf("sending: %T", m)
  return s.ServerStream.Send(m)
}

func (s *loggingStream) Recv(m interface{}) error {
  err := s.ServerStream.Recv(m)
  log.Printf("received: %T (err=%v)", m, err)
  return err
}
```

**Trap:** The wrapper must implement the full `grpc.ServerStream` interface, including `Context()`. Use embedded struct forwarding.
</details>

---

**Q13:** What is the `grpc.StatsHandler` interface?

<details><summary>Answer</summary>

A low-level hook for observing gRPC events: connections, RPC starts/ends, message sizes, wire bytes. Used for custom metrics and tracing.

```go
type MetricsHandler struct{}

func (h *MetricsHandler) TagRPC(ctx context.Context, info *stats.RPCTagInfo) context.Context {
  return ctx
}
func (h *MetricsHandler) HandleRPC(ctx context.Context, s stats.RPCStats) {
  switch v := s.(type) {
  case *stats.Begin:
    // RPC started
  case *stats.End:
    metrics.RPCDuration.WithLabelValues(v.IsSuccess().String()).Observe(v.EndTime.Sub(v.BeginTime).Seconds())
  }
}
func (h *MetricsHandler) TagConn(ctx context.Context, info *stats.ConnTagInfo) context.Context { return ctx }
func (h *MetricsHandler) HandleConn(ctx context.Context, s stats.ConnStats) {}
```

**Trap:** `StatsHandler` provides different information than interceptors. Use interceptors for business logic (auth, logging), `StatsHandler` for observability (metrics, tracing).
</details>

---

**Q14:** What is the interceptor execution order for `ChainUnaryInterceptor` with 3 interceptors?

<details><summary>Answer</summary>

Given `ChainUnaryInterceptor(A, B, C)`:
- On the server: `A → B → C → handler → C → B → A`
- On the client: `A → B → C → invoker → C → B → A`

The first interceptor added is the outermost.

**Trap:** If interceptor A sets something in context and C panics, A's deferred cleanup might not see the context value if C's deferred recovery eats the panic before A runs.
</details>

---

**Q15:** How do you inject a custom `ServerStream` in a stream interceptor?

<details><summary>Answer</summary>

Create a struct that embeds `grpc.ServerStream` and overrides specific methods:

```go
type contextInjector struct {
  grpc.ServerStream
  ctx context.Context
}

func (c *contextInjector) Context() context.Context {
  return c.ctx
}

// Usage in interceptor:
func injectInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
  ctx := context.WithValue(ss.Context(), "key", "value")
  return handler(srv, &contextInjector{ServerStream: ss, ctx: ctx})
}
```
</details>

---

**Q16:** What is the `grpc.Streamer` and `grpc.UnaryClientInterceptor` relationship?

<details><summary>Answer</summary>

Client-side interceptors wrap the `Invoker` (for unary) or `Streamer` (for streaming). `Streamer` creates a `ClientStream`. Client interceptor signatures:

```go
type UnaryClientInterceptor func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
type StreamClientInterceptor func(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, streamer Streamer, opts ...CallOption) (ClientStream, error)
```
</details>

---

**Q17:** How does OAuth 2.0 integrate with gRPC?

<details><summary>Answer</summary>

Two common patterns:
1. **Client credentials grant** — service-to-service auth: client fetches token from OAuth2 server, attaches via `WithPerRPCCredentials`.
2. **Implicit/Authorization code flow** — user auth: token is sent via `authorization` metadata header in an interceptor.

**Trap:** gRPC doesn't natively support OAuth2 flows — you manage token acquisition and refresh yourself.
</details>

---

**Q18:** What is the `google.golang.org/grpc/authz` package?

<details><summary>Answer</summary>

An authorization middleware for gRPC that validates requests against an access control list (JSON policy):

```json
{
  "name": "chronos_policy",
  "allow_rules": [{
    "service": "chronos.v1.Chronos",
    "method": "SubmitJob",
    "role": ["admin", "operator"]
  }]
}
```

**Trap:** This is a simple allow-list — it doesn't support attribute-based conditionals (e.g., "only own jobs"). Use a custom interceptor for fine-grained authz.
</details>

---

**Q19:** Can you modify response headers from a client interceptor?

<details><summary>Answer</summary>

Yes — use `grpc.Header()` call option to receive headers after the call:

```go
var header metadata.MD
resp, err := client.SubmitJob(ctx, req, grpc.Header(&header))
// header now contains response headers
```

**Trap:** This is a client-side operation. Server-side response headers are set via `grpc.SetHeader` or `grpc.SendHeader`.
</details>

---

**Q20:** What is the `grpc.Compressor` interceptor pattern?

<details><summary>Answer</summary>

For custom compression per call:

```go
func CompressionInterceptor() grpc.UnaryClientInterceptor {
  return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    opts = append(opts, grpc.UseCompressor("gzip"))
    return invoker(ctx, method, req, reply, cc, opts...)
  }
}
```

**Trap:** Compression is negotiated at the start of a stream. For streaming RPCs, you must set it before the first message.
</details>

---

## 4. Rapid-Fire: Production & Operations (30 questions)

**Q1:** How does gRPC client-side load balancing work?

<details><summary>Answer</summary>

gRPC supports several LB policies:
- **`pick_first`** (default): connects to the first address; fails over to next on connection failure.
- **`round_robin`**: distributes RPCs across healthy subconnections.
- **`grpclb`**: delegates to an external LB (the gRPC Load Balancer protocol).
- **Configurable via service config**:
```json
{
  "loadBalancingConfig": [{"round_robin": {}}]
}
```

**Trap:** Client-side round robin doesn't see server load — it may send requests to overloaded instances. Use a service mesh or external LB for load-aware routing.
</details>

---

**Q2:** What is gRPC's name resolution mechanism?

<details><summary>Answer</summary>

gRPC uses resolvers to convert target names into addresses. Built-in:
- **`dns`**: resolves SRV records and A/AAAA records. Watches for changes via periodic lookups.
- **`passthrough`** (default for `dns:///`): uses the raw address as-is.
- **Custom resolvers**: implement `grpc.Resolver` interface.

```go
resolver.SetDefaultScheme("dns")
// Target: "dns:///chronos.service.consul:50051"
```
</details>

---

**Q3:** How do you configure gRPC retry?

<details><summary>Answer</summary>

Via service config:

```json
{
  "methodConfig": [{
    "name": [{"service": "chronos.v1.Chronos"}],
    "retryPolicy": {
      "maxAttempts": 3,
      "initialBackoff": "0.1s",
      "maxBackoff": "1s",
      "backoffMultiplier": 2,
      "retryableStatusCodes": ["UNAVAILABLE"]
    }
  }]
}
```

Enable on the client: `grpc.WithDefaultServiceConfig(jsonConfig)`.

**Trap:** Retries don't work for streaming RPCs in the built-in retry policy. Also, non-idempotent methods should not be retried unless the code is idempotent-safe.
</details>

---

**Q4:** What is hedging in gRPC?

<details><summary>Answer</summary>

Hedging sends multiple copies of a request to different backends and uses the first successful response. It increases tail latency at the cost of extra load. Enabled via service config:

```json
{
  "hedgingPolicy": {
    "maxAttempts": 3,
    "hedgingDelay": "0.1s"
  }
}
```

**Trap:** Hedging can cause significant server load (3x for every request). Only hedge idempotent, read-only calls.
</details>

---

**Q5:** How does gRPC keepalive work?

<details><summary>Answer</summary>

HTTP/2 PING frames sent periodically to detect dead connections.

**Server config:**
```go
grpc.KeepaliveParams(keepalive.ServerParameters{
  Time:    30 * time.Second,
  Timeout: 10 * time.Second,
})
grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
  MinTime: 5 * time.Second,
})
```

**Client config:**
```go
grpc.WithKeepaliveParams(keepalive.ClientParameters{
  Time:    30 * time.Second,
  Timeout: 20 * time.Second,
  PermitWithoutStream: true,
})
```

**Trap:** Keepalive with `PermitWithoutStream: true` can cause unnecessary traffic. Only enable when clients are behind NAT/long-lived services.
</details>

---

**Q6:** What is the `GRPC_GO_KEEPALIVE_MIN_TIME` environment variable?

<details><summary>Answer</summary>

Sets the minimum keepalive time that a client is allowed to request. If the client sends a less frequent keepalive than this, the server disconnects the client. Default is 5 minutes in gRPC Go. On some cloud environments you need to lower this.

**Trap:** Load balancers and proxies often have shorter idle timeouts. If your keepalive is too infrequent, the proxy kills the connection.
</details>

---

**Q7:** What is gRPC health checking and why is it important in K8s?

<details><summary>Answer</summary>

The `grpc.health.v1.Health` service provides `Check` (unary) and `Watch` (streaming) RPCs. K8s probes use this via `grpc_health_probe`:

```yaml
livenessProbe:
  exec:
    command: ["/bin/grpc_health_probe", "-addr=:50051"]
initialDelaySeconds: 5
```

**Trap:** K8s default HTTP probes don't work with gRPC (HTTP/2 without TLS mismatch). Use `grpc_health_probe` or configure Envoy to convert.
</details>

---

**Q8:** How do you implement graceful shutdown for a gRPC server?

<details><summary>Answer</summary>

```go
server := grpc.NewServer()
// Register services...

stop := make(chan os.Signal, 1)
signal.Notify(stop, syscall.SIGTERM, syscall.SIGINT)
<-stop

// Stop accepting new connections. Drain existing ones.
gracefulStop := make(chan struct{})
go func() {
  server.GracefulStop()
  close(gracefulStop)
}()

select {
case <-gracefulStop:
  // all done
case <-time.After(30 * time.Second):
  server.Stop() // force stop
}
```

**Trap:** `GracefulStop()` blocks until all in-flight RPCs complete. Unresponsive handlers can delay shutdown indefinitely.
</details>

---

**Q9:** How does gRPC integrate with K8s?

<details><summary>Answer</summary>

- Services expose gRPC via ClusterIP, with `grpc` port.
- Headless services for client-side load balancing.
- Envoy sidecar for advanced routing, retries, timeouts.
- `KSVC` (Knative) for serverless gRPC.
- `PROXY protocol` if behind a cloud load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: chronos
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  ports:
  - port: 50051
    targetPort: 50051
    protocol: TCP
    name: grpc
  selector:
    app: chronos
```

**Trap:** K8s Service `type: LoadBalancer` with HTTP health checks fails for gRPC (HTTP/2 requires TLS or h2c). Use TCP or gRPC-specific probes.
</details>

---

**Q10:** What is Envoy's role with gRPC?

<details><summary>Answer</summary>

Envoy provides:
- gRPC-native HTTP/2 routing
- mTLS termination
- Retry and timeout policies
- Load balancing (least_request, ring_hash)
- gRPC-Web transcoding
- Access logging
- Circuit breaking

```yaml
typed_config:
  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

**Trap:** Envoy must be configured for HTTP/2 (`HttpConnectionManager` with `codec_type: AUTO`) or gRPC requests fail at the protocol level.
</details>

---

**Q11:** What is a service mesh and how does it affect gRPC?

<details><summary>Answer</summary>

A service mesh (Istio, Linkerd) transparently manages gRPC traffic via sidecar proxies. Benefits: mTLS without app changes, traffic splitting, circuit breaking, observability. Drawbacks: added latency (proxy hop), complexity, resource overhead.

**Trap:** Service mesh proxies can interfere with gRPC's load balancing. Disable client-side LB (`pick_first`) and let the mesh handle routing.
</details>

---

**Q12:** What is gRPC-Web and when should you use it?

<details><summary>Answer</summary>

gRPC-Web is a subset of gRPC for browser clients. Browsers can't speak raw HTTP/2 to gRPC servers — gRPC-Web uses a proxy (Envoy, grpc-web) to translate between HTTP/1.1 and gRPC. Supports unary and server-side streaming only (no bidirectional or client streaming).

**Trap:** gRPC-Web does not support all gRPC features (e.g., trailers from JavaScript, client streaming). Ensure your API design accounts for this.
</details>

---

**Q13:** How do you debug gRPC issues with `grpcurl`?

<details><summary>Answer</summary>

```bash
# List services via reflection
grpcurl -plaintext localhost:50051 list

# Invoke a method
grpcurl -plaintext -d '{"job_id":"abc"}' localhost:50051 chronos.v1.Chronos/WatchJobStatus

# Use with TLS
grpcurl -insecure -cacert ca.pem myhost:50051 list

# Import proto files directly (no reflection needed)
grpcurl -import-path ./proto -proto chronos/v1/job.proto localhost:50051 list
```
</details>

---

**Q14:** How do you configure gRPC for low-latency performance?

<details><summary>Answer</summary>

- Use `round_robin` LB for homogeneous backends.
- Set appropriate message size limits to avoid GC pressure.
- Enable `grpc.UseCompressor("gzip")` only for bandwidth-bound calls.
- Tune `InitialWindowSize`/`InitialConnWindowSize` for high-throughput streams.
- Use keepalive to detect dead connections fast.
- Profile with `grpc.StatsHandler`.
- Use `net/http/pprof` on a separate port.

**Trap:** Premature optimization is common. Profile first with `go test -bench` and `pprof` before tuning gRPC knobs.
</details>

---

**Q15:** How do you configure gRPC server for high throughput?

<details><summary>Answer</summary>

```go
server := grpc.NewServer(
  grpc.NumStreamWorkers(20),        // goroutines for stream processing
  grpc.MaxConcurrentStreams(1000),
  grpc.InitialWindowSize(1<<24),    // 16MB stream window
  grpc.InitialConnWindowSize(1<<26),// 64MB connection window
  grpc.ReadBufferSize(32<<10),     // 32KB read buffer
  grpc.WriteBufferSize(32<<10),    // 32KB write buffer
)
```

**Trap:** `NumStreamWorkers` has complex implications — test thoroughly. Default (0) uses per-stream goroutines which works well most of the time.
</details>

---

**Q16:** What is `gRPC DNS SRV` resolution?

<details><summary>Answer</summary>

gRPC can resolve SRV records: `dns:///_grpc._tcp.chronos.service.consul`. This allows the server to advertise port and priority via DNS without hardcoding ports.

**Trap:** SRV records have caching issues. TTL must be low enough for fast failover but not so low it overwhelms DNS servers.
</details>

---

**Q17:** What is the difference between gRPC and message queues (RabbitMQ, Kafka)?

<details><summary>Answer</summary>

gRPC is synchronous RPC (request/response or streaming); message queues are asynchronous pub/sub or point-to-point. gRPC gives you typed contracts, real-time streaming, bidirectional communication. Message queues provide persistence, replay, fan-out, guaranteed delivery (at-least-once, exactly-once).

**Trap:** Using gRPC for async job processing (like Chronos) requires careful handling of failures. Consider a message queue for fire-and-forget jobs, gRPC for real-time coordination.
</details>

---

**Q18:** When should you use a message queue over gRPC for Chronos?

<details><summary>Answer</summary>

Use a message queue when:
- Jobs must survive broker restarts (persistence).
- You need at-least-once or exactly-once delivery.
- Workers can join/leave independently via consumer groups.
- You need to buffer during traffic spikes.

Use gRPC when:
- You need real-time bidirectional coordination.
- You have typed contracts (protobuf).
- You need low-latency streaming (heartbeats).

**Hybrid pattern:** Chronos can use gRPC for control plane (heartbeats, assignments) and a message queue for job payloads.
</details>

---

**Q19:** How do you handle gRPC connection pooling?

<details><summary>Answer</summary>

gRPC manages connections per `ClientConn` — each `Dial` creates one HTTP/2 connection (or one per subchannel). Under the hood, gRPC maintains subconnections for each resolved address when using `round_robin`. For multiple targets, use separate `ClientConn` instances or a connection pool library.

**Trap:** gRPC reuses connections aggressively. Opening too many separate connections wastes resources. Share a single `ClientConn` across goroutines — it's safe.
</details>

---

**Q20:** What is the `http_proxy` behavior with gRPC?

<details><summary>Answer</summary>

gRPC supports HTTP CONNECT proxies via environment variables (`http_proxy`, `https_proxy`). However, gRPC's HTTP/2 connection upgrade over a proxy can be problematic — the proxy must support HTTP/2 CONNECT or h2c upgrade.

**Trap:** Many corporate proxies don't support HTTP/2 natively, causing gRPC connection failures. Use `no_proxy` for internal gRPC traffic.
</details>

---

**Q21:** How does gRPC handle connection backoff?

<details><summary>Answer</summary>

gRPC implements exponential backoff for reconnection:

```go
grpc.WithConnectParams(grpc.ConnectParams{
  MinConnectTimeout: 5 * time.Second,
  Backoff: backoff.DefaultConfig,
})
```

Default: 1s initial, max 120s, multiplier 1.6, with jitter. Resets after the connection stays up for `BackoffBaseDelay` * 2 (default 2s).

**Trap:** If the server is slow to start, clients will exhaust backoff and fail. Set `MinConnectTimeout` appropriately for your environment.
</details>

---

**Q22:** What is the `xds` resolver in gRPC?

<details><summary>Answer</summary>

An xDS-based resolver used with Envoy/Istio. It allows gRPC to receive dynamic configuration (listeners, clusters, routes, endpoints) via the xDS protocol, enabling advanced traffic management without restarting clients.

```go
import _ "google.golang.org/grpc/xds"
// Then use: xds:///chronos service URL
```

**Trap:** gRPC xDS support is behind feature flags and requires `GRPC_XDS_EXPERIMENTAL` env var set in older versions.
</details>

---

**Q23:** What is `grpc.MaxHeaderListSize`?

<details><summary>Answer</summary>

A server option limiting the maximum size of HTTP/2 header frames. Default is 8KB in Go gRPC. Large metadata (e.g., big JWT tokens) can exceed this, causing `INTERNAL` errors.

**Trap:** If your metadata has large auth tokens, increase this limit or compress tokens. Exceeding it will cause the entire RPC to fail.
</details>

---

**Q24:** How do you handle JWT token refresh in gRPC?

<details><summary>Answer</summary>

```go
type refreshToken struct {
  mu      sync.RWMutex
  token   string
  expiry  time.Time
}

func (t *refreshToken) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
  t.mu.RLock()
  expiry := t.expiry
  t.mu.RUnlock()
  if time.Until(expiry) < 30*time.Second {
    _ = t.refresh() // refresh in background
  }
  t.mu.RLock()
  defer t.mu.RUnlock()
  return map[string]string{"authorization": "Bearer " + t.token}, nil
}

func (t *refreshToken) RequireTransportSecurity() bool { return true }
```
</details>

---

**Q25:** What monitoring metrics should you collect for gRPC?

<details><summary>Answer</summary>

- Per method: QPS, latency (p50/p90/p99), error rate by status code
- Per server: active streams, connections, message sizes
- Client: connection states (connected, idle, transient failure), retry counts
- gRPC-specific: flow control window sizes, keepalive count, header size

**Trap:** gRPC latency metrics should exclude flow-control blocking time. Use `grpc.StatsHandler` for wire-level timings vs interceptor for application-level timings.
</details>

---

**Q26:** What is `unix://` socket support in gRPC?

<details><summary>Answer</summary>

gRPC supports Unix domain sockets for local communication — faster than TCP, no port conflicts, OS-level file permissions.

```go
conn, _ := grpc.Dial("unix:///tmp/chronos.sock",
  grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

**Trap:** On the server, you must create a `net.Listener` on the Unix socket. Don't forget to clean up the socket file on shutdown.
</details>

---

**Q27:** How do you set up circuit breaking for gRPC?

<details><summary>Answer</summary>

gRPC itself doesn't provide circuit breaking — use a service mesh (Envoy, Istio) or implement it in an interceptor:

```go
func CircuitBreakerInterceptor(cb *circuitbreaker) grpc.UnaryClientInterceptor {
  return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    if !cb.Ready() {
      return status.Error(codes.Unavailable, "circuit breaker open")
    }
    err := invoker(ctx, method, req, reply, cc, opts...)
    if err != nil {
      st, _ := status.FromError(err)
      if st.Code() == codes.Unavailable {
        cb.Fail()
      }
    } else {
      cb.Success()
    }
    return err
  }
}
```
</details>

---

**Q28:** How do you implement gRPC rate limiting?

<details><summary>Answer</summary>

Server-side interceptor with a token bucket or sliding window:

```go
func RateLimitInterceptor(limiter *rate.Limiter) grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    if !limiter.Allow() {
      return nil, status.Errorf(codes.ResourceExhausted, "rate limit exceeded")
    }
    return handler(ctx, req)
  }
}
```

**Trap:** Per-instance rate limiting is inaccurate. Use centralized token buckets (Redis) for multi-instance deployments.
</details>

---

**Q29:** What is the `grpc.StatsHandler` for tracing?

<details><summary>Answer</summary>

OpenTelemetry provides gRPC instrumentation via interceptors that use `StatsHandler` internally:

```go
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

server := grpc.NewServer(
  grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
```

This automatically creates traces for each gRPC call, propagating trace context via metadata.
</details>

---

**Q30:** What are common gRPC anti-patterns in production?

<details><summary>Answer</summary>

1. **No deadlines** — requests hang forever
2. **Not closing idle connections** — resource leaks
3. **Overly large messages** — memory pressure, slow serialization
4. **Exposing gRPC directly to the internet without a proxy** — security risk
5. **Not handling `UNAVAILABLE` with retry** — flaky clients
6. **Default keepalive without tuning for proxy timeouts** — dropped connections
7. **Sharing `ClientConn` across services with different requirements** — config conflicts
8. **Using reflection in production** — security exposure
9. **Mixing gRPC versions in the same binary** — subtle bugs
10. **Ignoring `MaxConcurrentStreams` limits** — connection starvation
</details>

---

## 5. Code Puzzles (6-8 puzzles)

### Puzzle 1: Backward Compatibility Check

Given the following proto change between `v1` and `v2`, determine if the change is backward compatible:

```protobuf
// v1
message Job {
  string id = 1;
  string name = 2;
  int32 priority = 3;
}

// v2
message Job {
  string id = 1;
  string name = 2;
  int32 priority = 3;
  int32 retry_count = 3;  // <-- new field
}
```

<details><summary>Answer</summary>

**Not backward compatible.** Field number 3 is reused for `retry_count`, but in v1 it was `priority`. This is a fatal breaking change — old clients sending `priority` would have it interpreted as `retry_count` and vice versa. The correct approach is to use field number 4 for `retry_count`.

```protobuf
message Job {
  string id = 1;
  string name = 2;
  int32 priority = 3;
  int32 retry_count = 4;  // safe: new field number
}
```
</details>

---

### Puzzle 2: Identify the Streaming Type

For each Chronos scenario, identify the gRPC streaming pattern:

**Scenario A:** A worker sends a single registration request and receives a continuous stream of task assignments.

**Scenario B:** The scheduler receives a batch of job completion events from a worker (multiple events, one at a time) and responds with a single acknowledgment.

**Scenario C:** A client watches job status — the client sends the job ID once and receives status change events.

**Scenario D:** The scheduler and worker exchange heartbeats and control commands simultaneously — both sides send and receive independently.

<details><summary>Answer</summary>

- **A:** Server-side streaming (client sends one request, server streams responses)
- **B:** Client-side streaming (client sends multiple requests, server sends one response)
- **C:** Server-side streaming (same as A — Watch pattern)
- **D:** Bidirectional streaming (both sides stream independently)
</details>

---

### Puzzle 3: Status Code Selection

For each error scenario in Chronos, select the correct gRPC status code:

1. A user submits a job with an empty name.
2. A worker tries to claim a job that was already cancelled.
3. The worker's authentication token has expired.
4. The scheduler node is shutting down and refuses new connections.
5. A panic occurs in the job processing handler.
6. A client sends a request larger than the configured max message size.
7. A worker sends a heartbeat 10ms past the deadline.
8. The client requests a method that doesn't exist on the server.

<details><summary>Answer</summary>

1. `INVALID_ARGUMENT` (3) — validation failure
2. `FAILED_PRECONDITION` (9) — operation rejected given the current state
3. `UNAUTHENTICATED` (16) — invalid or expired token
4. `UNAVAILABLE` (14) — temporary, safe to retry
5. `INTERNAL` (13) — unexpected server error
6. `RESOURCE_EXHAUSTED` (8) — message size exceeded
7. `DEADLINE_EXCEEDED` (4) — worker too late
8. `UNIMPLEMENTED` (12) — method not found
</details>

---

### Puzzle 4: Interceptor Ordering and Execution Flow

Given the following gRPC server setup:

```go
func A(ctx, req, info, handler) (resp, err) {
  log.Println("A in")
  ctx = context.WithValue(ctx, "key", "A")
  resp, err := handler(ctx, req)
  log.Println("A out")
  return resp, err
}

func B(ctx, req, info, handler) (resp, err) {
  log.Println("B in")
  v := ctx.Value("key")
  resp, err := handler(ctx, req)
  log.Println("B out (key=", v, ")")
  return resp, err
}

func C(ctx, req, info, handler) (resp, err) {
  log.Println("C in")
  v := ctx.Value("key")
  resp, err := handler(ctx, req)
  log.Println("C out (key=", v, ")")
  return resp, err
}

server := grpc.NewServer(grpc.ChainUnaryInterceptor(A, B, C))
```

What is the log output when a unary RPC is handled?

<details><summary>Answer</summary>

```
A in
B in
C in
C out (key=A)
B out (key=A)
A out
```

Key insight: `A` sets the context value, and `ChainUnaryInterceptor` calls them outer-to-inner (A→B→C→handler→C→B→A). Since `A` sets the value before calling handler, `B` and `C` see it in their inbound paths too? Wait — `A` sets the value, then calls `handler(ctx, req)`. But `handler` is actually `B` (the next interceptor). So:

- A sets `key=A` on context
- A calls handler(ctx, req) which is B
- B sees `key=A`
- B calls handler(ctx, req) which is C
- C sees `key=A`
- C calls handler (the actual RPC handler)
- C returns, logs "C out (key=A)"
- B returns, logs "B out (key=A)"
- A returns, logs "A out"

So the output is:
```
A in
B in
C in
C out (key=A)
B out (key=A)
A out
```
</details>

---

### Puzzle 5: Load Balancing Behavior

In Chronos, you have 3 scheduler instances behind a headless K8s service. The gRPC client uses `round_robin` LB. For each scenario, describe what happens:

**Scenario 1:** All 3 instances are healthy. A client makes 6 `SubmitJob` calls.

**Scenario 2:** Instance B crashes. The client has 3 in-flight calls.

**Scenario 3:** The client uses `pick_first` instead. Instance A is slow to respond but not dead.

<details><summary>Answer</summary>

**Scenario 1:** The 6 calls are distributed round-robin: A, B, C, A, B, C. Each call goes to a healthy subchannel.

**Scenario 2:** B's subchannel transitions to `TRANSIENT_FAILURE`. The 3 in-flight calls to B fail with `UNAVAILABLE`. Retries (if configured) would be sent to A or C. The `round_robin` policy skips the failed subchannel.

**Scenario 3:** With `pick_first`, all calls go to A. A is slow but not failing, so `pick_first` keeps sending to it. The other instances are unused. This shows why `pick_first` is bad for load distribution.
</details>

---

### Puzzle 6: Deadline Propagation Chain

A gRPC call flows through: `Client → Interceptor A (client) → Service X → Interceptor B (server on X) → gRPC call to Service Y`

If the client sets a 5-second deadline, Service X has a 3-second internal deadline on its call to Service Y, and X's handler takes 2 seconds before calling Y, what is the effective deadline for Y?

<details><summary>Answer</summary>

The client deadline propagates from client → X. When X makes a call to Y with `ctx` (inheriting the deadline), the context has an effective deadline of `5s - 2s = 3s` remaining. If X also sets its own 3-second timeout on the outgoing call, the *minimum* of the two applies. So Y's deadline is `min(5s - 2s, 3s) = 3s` from X's outgoing call start.

**Key insight:** Context propagation always uses min(propagated deadline, local deadline). This prevents downstream services from outliving their parent.
</details>

---

### Puzzle 7: Message Ordering in Bidirectional Streaming

In Chronos, the scheduler and worker use bidirectional streaming. The scheduler sends task assignments (`SendTask`). The worker sends results (`SendResult`). If the scheduler sends messages A, B, C and the worker sends X, Y, Z, how are messages ordered at each end?

<details><summary>Answer</summary>

Each direction is independently ordered:
- Worker receives: A (first), B (second), C (third) — guaranteed order.
- Scheduler receives: X (first), Y (second), Z (third) — guaranteed order.

However, there is *no ordering guarantee* between directions. The scheduler might receive X before or after sending B. Use sequence numbers or correlation IDs for causality.

**Trap:** Assuming bidirectional streaming is like a synchronous conversation is incorrect — it's two independent message streams on one HTTP/2 stream.
</details>

---

### Puzzle 8: Memory Leak in Stream Handler

```go
func (s *ChronosServer) WatchJobStatus(req *pb.WatchJobStatusRequest, stream pb.Chronos_WatchJobStatusServer) error {
  ch := make(chan *pb.JobStatusEvent)
  s.subscribers[req.JobId] = append(s.subscribers[req.JobId], ch)
  defer func() {
    removeSubscriber(s.subscribers, req.JobId, ch)
  }()

  for {
    select {
    case evt := <-ch:
      if err := stream.Send(evt); err != nil {
        return err
      }
    case <-stream.Context().Done():
      return stream.Context().Err()
    }
  }
}
```

What is the bug and how does it cause issues?

<details><summary>Answer</summary>

**Race condition:** Access to `s.subscribers` is not synchronized. Concurrent calls to `WatchJobStatus` can cause a data race on the map append.

**Memory leak potential:** The subscriber isn't removed if the client disconnects during `stream.Send()` error — but the `defer` handles that. However, if the `removeSubscriber` function doesn't lock properly or a broadcast to a closed channel panics, the leak could occur.

**Fix:** Use `sync.RWMutex` for `s.subscribers` or use a concurrent-safe map.

```go
type SubscriberManager struct {
  mu          sync.RWMutex
  subscribers map[string][]chan *pb.JobStatusEvent
}
```
</details>

---

## 6. Live-Coding Exercises (4-5 exercises)

### Exercise 1: Server-Side Streaming for Job Status Updates

**Context:** Chronos workers stream heartbeat/status to the scheduler.

**Task:** Implement a Go gRPC server-side streaming RPC `WatchJobStatus`. Given a `JobID`, the server streams `JobStatusEvent` messages. The stream should:
- Accept a single `WatchJobStatusRequest` with `job_id`
- Stream `JobStatusEvent` messages with `job_id`, `status` (enum: PENDING, RUNNING, COMPLETED, FAILED), and `timestamp`
- Close the stream when the job reaches a terminal state (COMPLETED or FAILED)
- Handle client cancellation (context done)
- Use a channel-based subscriber pattern

<details><summary>Answer</summary>

```go
// Proto:
// service Chronos {
//   rpc WatchJobStatus(WatchJobStatusRequest) returns (stream JobStatusEvent);
// }
// message WatchJobStatusRequest { string job_id = 1; }
// message JobStatusEvent {
//   string job_id = 1;
//   JobStatus status = 2;
//   google.protobuf.Timestamp timestamp = 3;
// }

type JobStatusManager struct {
  mu          sync.RWMutex
  subscribers map[string][]chan *pb.JobStatusEvent
}

func NewJobStatusManager() *JobStatusManager {
  return &JobStatusManager{
    subscribers: make(map[string][]chan *pb.JobStatusEvent),
  }
}

func (m *JobStatusManager) Subscribe(jobID string) chan *pb.JobStatusEvent {
  ch := make(chan *pb.JobStatusEvent, 10)
  m.mu.Lock()
  m.subscribers[jobID] = append(m.subscribers[jobID], ch)
  m.mu.Unlock()
  return ch
}

func (m *JobStatusManager) Unsubscribe(jobID string, ch chan *pb.JobStatusEvent) {
  m.mu.Lock()
  defer m.mu.Unlock()
  subs := m.subscribers[jobID]
  for i, sub := range subs {
    if sub == ch {
      m.subscribers[jobID] = append(subs[:i], subs[i+1:]...)
      close(ch)
      break
    }
  }
}

func (m *JobStatusManager) Publish(jobID string, evt *pb.JobStatusEvent) {
  m.mu.RLock()
  subs := m.subscribers[jobID]
  m.mu.RUnlock()
  for _, ch := range subs {
    select {
    case ch <- evt:
    default:
      // drop slow consumer
    }
  }
}

func (s *ChronosServer) WatchJobStatus(req *pb.WatchJobStatusRequest, stream pb.Chronos_WatchJobStatusServer) error {
  ch := s.statusMgr.Subscribe(req.JobId)
  defer s.statusMgr.Unsubscribe(req.JobId, ch)

  for {
    select {
    case evt := <-ch:
      if err := stream.Send(evt); err != nil {
        return err
      }
      if evt.Status == pb.JobStatus_JOB_STATUS_COMPLETED || evt.Status == pb.JobStatus_JOB_STATUS_FAILED {
        return nil
      }
    case <-stream.Context().Done():
      return stream.Context().Err()
    }
  }
}
```
</details>

---

### Exercise 2: Auth Interceptor (JWT)

**Context:** Chronos needs JWT authentication.

**Task:** Implement a server-side unary interceptor that:
1. Extracts `authorization` metadata from incoming context
2. Validates the JWT (using a `TokenValidator` interface)
3. Injects `UserClaims` (user ID, role) into the context
4. Returns `UNAUTHENTICATED` for missing/invalid tokens
5. Skips auth for `/chronos.v1.Chronos/HealthCheck`

<details><summary>Answer</summary>

```go
type UserClaims struct {
  UserID string
  Role   string
}

type TokenValidator interface {
  ValidateToken(token string) (*UserClaims, error)
}

type contextKey string

const userClaimsKey contextKey = "user_claims"

func AuthInterceptor(validator TokenValidator) grpc.UnaryServerInterceptor {
  skipMethods := map[string]bool{
    "/chronos.v1.Chronos/HealthCheck": true,
  }

  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    if skipMethods[info.FullMethod] {
      return handler(ctx, req)
    }

    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
      return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
      return nil, status.Error(codes.Unauthenticated, "missing authorization header")
    }

    rawToken := strings.TrimPrefix(tokens[0], "Bearer ")
    if rawToken == tokens[0] {
      return nil, status.Error(codes.Unauthenticated, "invalid authorization format, expected Bearer")
    }

    claims, err := validator.ValidateToken(rawToken)
    if err != nil {
      return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }

    ctx = context.WithValue(ctx, userClaimsKey, claims)
    return handler(ctx, req)
  }
}

func GetUserClaims(ctx context.Context) (*UserClaims, bool) {
  claims, ok := ctx.Value(userClaimsKey).(*UserClaims)
  return claims, ok
}
```
</details>

---

### Exercise 3: Logging Interceptor

**Context:** Centralized logging for all gRPC calls in Chronos.

**Task:** Implement both unary and stream logging interceptors that log:
- Method name (`info.FullMethod`)
- Duration (from start to finish)
- Status code (even for successful calls, log `OK`)
- For streaming: number of messages sent and received

Use a structured logger (e.g., `go.uber.org/zap`).

<details><summary>Answer</summary>

```go
import (
  "go.uber.org/zap"
  "google.golang.org/grpc"
  "google.golang.org/grpc/status"
)

func UnaryLoggingInterceptor(logger *zap.Logger) grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    duration := time.Since(start)
    code := status.Code(err)

    logger.Info("gRPC unary call",
      zap.String("method", info.FullMethod),
      zap.Duration("duration", duration),
      zap.String("code", code.String()),
    )
    return resp, err
  }
}

type streamLogWrapper struct {
  grpc.ServerStream
  logger      *zap.Logger
  method      string
  startTime   time.Time
  recvCount   int64
  sendCount   int64
}

func (w *streamLogWrapper) Recv(m interface{}) error {
  err := w.ServerStream.Recv(m)
  if err == nil {
    w.recvCount++
  }
  return err
}

func (w *streamLogWrapper) Send(m interface{}) error {
  err := w.ServerStream.Send(m)
  if err == nil {
    w.sendCount++
  }
  return err
}

func StreamLoggingInterceptor(logger *zap.Logger) grpc.StreamServerInterceptor {
  return func(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
    start := time.Now()
    ws := &streamLogWrapper{
      ServerStream: ss,
      logger:      logger,
      method:      info.FullMethod,
      startTime:   start,
    }
    err := handler(srv, ws)
    duration := time.Since(start)
    code := status.Code(err)

    logger.Info("gRPC stream call",
      zap.String("method", info.FullMethod),
      zap.Duration("duration", duration),
      zap.String("code", code.String()),
      zap.Int64("sent", ws.sendCount),
      zap.Int64("received", ws.recvCount),
    )
    return err
  }
}
```
</details>

---

### Exercise 4: Bidirectional Streaming for Task Coordination

**Context:** Chronos scheduler and worker coordinate via bidirectional streaming.

**Task:** Implement a handler for the following service definition:

```protobuf
service Chronos {
  rpc WorkerCoordinator(stream WorkerMessage) returns (stream SchedulerMessage);
}

message WorkerMessage {
  string worker_id = 1;
  oneof msg {
    Heartbeat heartbeat = 2;
    JobResult job_result = 3;
  }
}

message SchedulerMessage {
  oneof msg {
    TaskAssignment task = 1;
    Ack ack = 2;
  }
}
```

The handler should:
- On receiving a heartbeat: log it, reply with an Ack
- On receiving a job result: process it, reply with next task assignment
- Send tasks periodically via a ticker
- Handle worker disconnection
- Clean up resources on context cancellation

<details><summary>Answer</summary>

```go
func (s *ChronosServer) WorkerCoordinator(stream pb.Chronos_WorkerCoordinatorServer) error {
  workerID := ""
  ticker := time.NewTicker(5 * time.Second)
  defer ticker.Stop()

  recvCh := make(chan *pb.WorkerMessage, 100)
  errCh := make(chan error, 1)

  go func() {
    for {
      msg, err := stream.Recv()
      if err != nil {
        errCh <- err
        return
      }
      recvCh <- msg
    }
  }()

  for {
    select {
    case msg := <-recvCh:
      switch m := msg.Msg.(type) {
      case *pb.WorkerMessage_Heartbeat:
        if workerID == "" {
          workerID = msg.WorkerId
        }
        log.Printf("heartbeat from %s: seq=%d", msg.WorkerId, m.Heartbeat.Sequence)
        if err := stream.Send(&pb.SchedulerMessage{
          Msg: &pb.SchedulerMessage_Ack{Ack: &pb.Ack{Ok: true}},
        }); err != nil {
          return err
        }

      case *pb.WorkerMessage_JobResult:
        log.Printf("job result from %s: job=%s status=%s", msg.WorkerId, m.JobResult.JobId, m.JobResult.Status)
        nextTask := s.getNextTask(msg.WorkerId)
        if nextTask != nil {
          if err := stream.Send(&pb.SchedulerMessage{
            Msg: &pb.SchedulerMessage_Task{Task: nextTask},
          }); err != nil {
            return err
          }
        }
      }

    case <-ticker.C:
      if workerID != "" {
        task := s.getNextTask(workerID)
        if task != nil {
          if err := stream.Send(&pb.SchedulerMessage{
            Msg: &pb.SchedulerMessage_Task{Task: task},
          }); err != nil {
            return err
          }
        }
      }

    case err := <-errCh:
      if err == io.EOF {
        log.Printf("worker %s disconnected", workerID)
        return nil
      }
      return err

    case <-stream.Context().Done():
      log.Printf("worker %s stream cancelled", workerID)
      return stream.Context().Err()
    }
  }
}

func (s *ChronosServer) getNextTask(workerID string) *pb.TaskAssignment {
  // pull from job queue
  return nil // placeholder
}
```
</details>

---

### Exercise 5: Graceful Shutdown for gRPC Server

**Context:** Chronos scheduler must shut down gracefully without dropping in-flight jobs.

**Task:** Implement a graceful shutdown function that:
1. Listens for SIGTERM/SIGINT
2. Stops accepting new connections
3. Drains existing connections with a configurable timeout
4. Force stops after timeout
5. Logs the shutdown progress

<details><summary>Answer</summary>

```go
func GracefulShutdown(server *grpc.Server, lis net.Listener, timeout time.Duration, logger *zap.Logger) {
  stop := make(chan os.Signal, 1)
  signal.Notify(stop, syscall.SIGTERM, syscall.SIGINT)

  sig := <-stop
  logger.Info("shutting down", zap.String("signal", sig.String()))

  // Stop listening — new connections are refused
  lis.Close()

  done := make(chan struct{})
  go func() {
    server.GracefulStop()
    close(done)
  }()

  select {
  case <-done:
    logger.Info("graceful shutdown complete")
  case <-time.After(timeout):
    logger.Warn("graceful stop timed out, forcing stop", zap.Duration("timeout", timeout))
    server.Stop()
  }
}

func main() {
  lis, _ := net.Listen("tcp", ":50051")
  server := grpc.NewServer(
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
      MinTime: 5 * time.Second,
      PermitWithoutStream: true,
    }),
  )
  pb.RegisterChronosServer(server, NewChronosServer())

  go func() {
    if err := server.Serve(lis); err != nil {
      log.Fatalf("serve: %v", err)
    }
  }()

  GracefulShutdown(server, lis, 30*time.Second, logger)
}
```
</details>

---

## 7. Debugging Scenarios (4-5 scenarios)

### Scenario 1: UNAVAILABLE Errors After Deploy

**Symptom:** After a rolling deploy of Chronos scheduler, all gRPC clients see `UNAVAILABLE` errors for 10-30 seconds.

**Investigation:** Clients use DNS resolution with a 60s TTL. The old pod is terminated before DNS propagates to clients. The load balancer (K8s Service) still has endpoints from the old pod during termination, but the pod's gRPC server already stopped.

**Root Cause:** The gRPC server's `GracefulStop` is called, but the pod's readiness probe doesn't account for the draining state. Clients still send traffic to the shutting-down pod.

**Fix:**
1. Add a pre-stop hook that delays SIGTERM delivery:
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 15 && kill -SIGTERM 1"]
```
2. Lower DNS TTL to 5-10 seconds
3. Remove the pod from the service endpoint before stopping the server:
```go
func shutdownWithDrain(server *grpc.Server) {
  // Signal load balancer we're draining
  // (e.g., remove from Consul, mark /health as unhealthy)
  time.Sleep(5 * time.Second) // wait for LB to detect
  server.GracefulStop()
}
```
</details>

---

### Scenario 2: High Latency on Server-Side Streaming, Low CPU

**Symptom:** Chronos workers streaming heartbeats to the scheduler experience 500ms+ latency. CPU is under 30%. Memory is normal.

**Investigation:** Network captures show TCP flow control stalls. The server-side `InitialWindowSize` is at the default 64KB. Workers send heartbeats every second (~256 bytes each). The server reads them slowly, filling the flow control window. The senders must wait for window updates.

**Root Cause:** The flow control window is too small for the number of concurrent streaming connections. Each stream gets 64KB, and with 100s of workers, the aggregate throughput is limited by window update round trips.

**Fix:** Increase the initial stream window size:
```go
server := grpc.NewServer(
  grpc.InitialWindowSize(1 << 20),     // 1MB per stream
  grpc.InitialConnWindowSize(1 << 26), // 64MB per connection
)
```

**Verify:** Use `grpc.StatsHandler` to monitor flow control events:
```go
type flowControlStats struct{}
func (f *flowControlStats) HandleRPC(ctx context.Context, s stats.RPCStats) {
  switch v := s.(type) {
  case *stats.OutPayload:
    // message sent
  case *stats.InPayload:
    // message received — check if wire_length >> compressed_length (flow control issue)
  }
}
```
</details>

---

### Scenario 3: Client Timeout but Server Completes Work

**Symptom:** A Chronos job submission succeeds on the server (job is created), but the client receives `DeadlineExceeded`. The server logs show the job was persisted.

**Investigation:** The client has a 3-second timeout. The server takes 3.5 seconds (due to Raft consensus commit latency during a leader election). The client cancels at 3s, but the server continues processing because it doesn't check `ctx.Done()`.

**Root Cause:** The client's deadline is not being checked by the server handler. The server completes the work independently, but the client already bailed.

**Fix:**
1. Always check context in handlers:
```go
func (s *ChronosServer) SubmitJob(ctx context.Context, req *pb.SubmitJobRequest) (*pb.SubmitJobResponse, error) {
  select {
  case <-ctx.Done():
    return nil, status.Errorf(codes.DeadlineExceeded, "client cancelled: %v", ctx.Err())
  case result := <-s.processJob(req):
    return result, nil
  }
}
```

2. For Raft operations, set a deadline on the internal Raft call that's less than the client's deadline.

3. Implement job idempotency so the client can safely retry:
```go
message SubmitJobRequest {
  string job_id = 1; // client-generated, dedup key
  // ...
}
```
</details>

---

### Scenario 4: Memory Growth on gRPC Server

**Symptom:** The Chronos scheduler RSS grows from 500MB to 2GB over 12 hours. No goroutine leak. Heap profile shows large allocations in `protobuf.Unmarshal`.

**Investigation:** A client is sending very large job payloads (50MB+) without chunking. The default max message size is 4MB, but someone bumped it to 100MB. The protobuf unmarshal allocates a contiguous buffer for the full message. Over time, GC can't keep up.

**Root Cause:** Large message sizes cause memory pressure. The streaming buffer buildup from slow consumers (Scenario 2) also contributes.

**Fix:**
1. Implement message size limits per method:
```go
server := grpc.NewServer(
  grpc.MaxRecvMsgSize(10 * 1024 * 1024), // 10MB max
)
```

2. For large payloads, use chunked streaming:
```protobuf
message UploadJobPayloadRequest {
  string job_id = 1;
  bytes chunk = 2;
  int32 chunk_index = 3;
}
```

3. Monitor per-stream buffer sizes with pprof:
```go
import "net/http/pprof"
// Serve pprof on separate port, check heap inuse_space

4. Set `grpc.ReadBufferSize` to limit per-connection read buffers.
```
</details>

---

### Scenario 5: gRPC-Web Client Can't Connect

**Symptom:** The frontend (React app) uses gRPC-Web to call Chronos through Envoy. It gets CORS errors and 404s on POST requests.

**Investigation:**
1. CORS headers are missing from Envoy response.
2. Envoy is not configured to handle gRPC-Web — it treats it as plain HTTP/1.1.
3. The `Content-Type` header sent by the browser is `application/grpc-web+proto`, but Envoy's gRPC filter expects `application/grpc`.

**Root Cause:** Two issues:
1. Envoy gRPC-Web filter not enabled
2. CORS configuration missing

**Fix:**
1. Enable the gRPC-Web filter in Envoy:
```yaml
http_filters:
- name: envoy.filters.http.grpc_web
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
- name: envoy.filters.http.router
```

2. Add CORS config:
```yaml
typed_config:
  "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
```

3. Verify the Envoy listener is HTTP/2 compatible (`codec_type: AUTO`).

4. Client-side: Ensure the JS gRPC-Web library sends the correct `Content-Type: application/grpc-web+proto`.
</details>

---

## 8. System Design Prompts (4-5 prompts)

### Prompt 1: Design a Distributed Job Scheduler Using gRPC (Chronos Architecture)

**Context:** Design Chronos — a distributed job scheduler — with gRPC as the primary communication protocol. Components: clients submit jobs, a scheduler assigns them to workers, workers execute and stream status.

**Key requirements:**
- gRPC for service definitions and communication
- Bidirectional streaming for worker ↔ scheduler coordination
- Protobuf for job definitions and status events
- Raft for scheduler leader election and state replication
- Fault tolerance: workers reconnect on scheduler failover

<details><summary>Answer</summary>

**Service Definitions:**

```protobuf
service Chronos {
  // Client-facing
  rpc SubmitJob(SubmitJobRequest) returns (SubmitJobResponse);
  rpc GetJobStatus(GetJobStatusRequest) returns (JobStatus);
  rpc WatchJobStatus(WatchJobStatusRequest) returns (stream JobStatusEvent);
  rpc CancelJob(CancelJobRequest) returns (CancelJobResponse);

  // Worker-facing
  rpc RegisterWorker(RegisterWorkerRequest) returns (RegisterWorkerResponse);
  rpc WorkerCoordinator(stream WorkerMessage) returns (stream SchedulerMessage);

  // Internal — health, admin
  rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);
}
```

**Architecture:**

```
Client (gRPC unary/stream)
  │
  ▼
[Envoy sidecar] ─── mTLS ───▶ [Scheduler Leader]
                                   │
                                   ├── Raft log (etcd/boltdb)
                                   │
Worker ◀── gRPC bidir stream ───▶ Scheduler
Worker ◀── gRPC bidir stream ───▶ Scheduler
Worker ◀── gRPC bidir stream ───▶ Scheduler
```

**Key design decisions:**

1. **Bidirectional streaming for worker coordination** — each worker opens one persistent stream. Scheduler pushes tasks; worker pushes heartbeats and results. Avoids polling.

2. **Raft for reliability** — scheduler state (queues, workers, job status) is replicated across 3-5 nodes via Raft. Failover is transparent to workers.

3. **gRPC health checking** — workers use `HealthCheck` + `Watch` to detect scheduler leader changes. On failover, workers reconnect their bidirectional stream.

4. **Protobuf for message contracts** — job definitions, status events, heartbeats all use proto3. Backward-compatible evolution.

5. **Deadline propagation** — client deadlines flow through to job execution. Jobs exceeding deadline are cancelled and retried.

6. **Flow control tuning** — increase `InitialWindowSize` for worker streaming to prevent backpressure on heartbeat delivery.
</details>

---

### Prompt 2: Design a Real-Time Order Book for a Trading Platform

**Context:** Design a real-time order book using gRPC streaming. Users subscribe to price feeds and order updates for financial instruments.

**Key requirements:**
- Low-latency (sub-millisecond) order book updates
- Thousands of simultaneous subscribers
- Server-side streaming per-instrument or multi-instrument
- Reconnection with catch-up (missed messages)

<details><summary>Answer</summary>

**Service Definitions:**

```protobuf
service OrderBook {
  rpc Subscribe(SubscribeRequest) returns (stream MarketDataEvent);
  rpc SubscribeMulti(SubscribeMultiRequest) returns (stream MarketDataEvent);
  rpc PlaceOrder(PlaceOrderRequest) returns (PlaceOrderResponse);
}

message SubscribeRequest {
  string instrument_id = 1;
  SnapShotOption snapshot = 2; // full snapshot on subscribe?
  int64 last_sequence = 3;     // for catch-up
}

message MarketDataEvent {
  int64 sequence = 1;
  oneof event {
    SnapShot snapshot = 2;
    PriceLevelUpdate update = 3;
    Trade trade = 4;
  }
}
```

**Architecture:**

```
Client (gRPC stream)
  │
  ▼
[Envoy proxy] ───▶ [Gateway]
                      │
            ┌─────────┼─────────┐
            ▼         ▼         ▼
      [Book A]   [Book B]   [Book C]
            │         │         │
            ▼         ▼         ▼
        [In-memory order book per instrument]
```

**Key design decisions:**

1. **Snapshot + incremental updates** — on subscribe, send a full snapshot then incremental level updates. Use `last_sequence` for catch-up.

2. **Fan-out via channels** — each instrument's book has a fan-out to subscriber channels. Use buffered channels with drop (or backpressure via gRPC flow control).

3. **Sequence numbers** — every event has a monotonically increasing sequence. Clients track the highest sequence for reconnection.

4. **Epidemic fan-out for multi-instrument** — client sends list of instruments; server merges streams from multiple books into one stream for the client.

5. **Flow control** — gRPC's HTTP/2 flow control naturally back-pressures clients that can't keep up.

6. **Load balancing** — use consistent hashing (ring hash) on instrument ID so all subscriptions for the same instrument go to the same backend.
</details>

---

### Prompt 3: Design a Notification Service with gRPC Streaming

**Context:** Design a real-time notification service where each user has a server-side gRPC stream. Users receive notifications (alerts, messages) in real time.

**Key requirements:**
- One stream per user (persistent connection)
- Reconnect with missed-message catch-up
- Multiple device support (phone, web, desktop)
- Fan-out: one message to millions of users
- 99.99% delivery within 1 second

<details><summary>Answer</summary>

**Service Definitions:**

```protobuf
service Notification {
  rpc Subscribe(SubscribeRequest) returns (stream NotificationEvent);
  rpc Ack(AckRequest) returns (AckResponse);
  rpc SendNotification(SendNotificationRequest) returns (SendNotificationResponse);
}

message SubscribeRequest {
  string user_id = 1;
  string device_id = 2;
  int64 last_ack_sequence = 3;
}

message NotificationEvent {
  int64 sequence = 1;
  NotificationType type = 2;
  string title = 3;
  string body = 4;
  map<string, string> metadata = 5;
}
```

**Architecture:**

```
[Notification Producer] ───▶ [Kafka/RabbitMQ]
                                  │
                            [Notification Service] ──▶ DB (cassandra/pg)
                                  │
                            [gRPC server — one stream per user]
                                  │
                  ┌───────────────┼───────────────┐
                  ▼               ▼               ▼
              [User A]        [User B]        [User C]
```

**Key design decisions:**

1. **Per-user server-side streaming** — each user connects, gets a dedicated stream. The server maintains a map of `user_id → channel`.

2. **Catch-up on reconnect** — on subscribe with `last_ack_sequence`, the server replays missed messages from a database. After catch-up, switches to live streaming.

3. **Acknowledgment** — client sends `Ack` with the last processed sequence. The server tracks per-device ack position for catch-up.

4. **Device fan-out** — one user with N devices gets N streams. The server duplicates messages per device.

5. **Backpressure handling** — if a device is slow, gRPC flow control blocks the sender. Use per-device buffered channels with bounded size.

6. **Multi-tenant isolation** — each tenant gets a separate gRPC server pool or uses metadata for tenant routing.

7. **Scaling** — use consistent hashing on user_id to route to the correct notification server. Or use a pub/sub per server.
</details>

---

### Prompt 4: Design a Multi-Tenant gRPC API

**Context:** Design a multi-tenant Chronos deployment where multiple teams share the same scheduler infrastructure but are isolated from each other.

**Key requirements:**
- Tenant context propagated via gRPC metadata
- Per-tenant rate limiting and quotas
- Per-tenant authorization (team members can only see their own jobs)
- Shared gRPC server, isolated data
- No tenant can see another tenant's data

<details><summary>Answer</summary>

**Tenant Propagation:**

```protobuf
// Metadata keys
// x-tenant-id: acme-corp
// x-user-id: jdoe
// x-role: admin
```

**Service definitions unchanged** — tenant is extracted via interceptor, not in the proto message.

**Interceptor Pattern:**

```go
func TenantInterceptor(tenantStore TenantStore) grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    md, _ := metadata.FromIncomingContext(ctx)
    tenantID := extractTenant(md)
    if tenantID == "" {
      return nil, status.Error(codes.Unauthenticated, "missing tenant")
    }
    if !tenantStore.IsActive(tenantID) {
      return nil, status.Errorf(codes.PermissionDenied, "tenant %s is suspended", tenantID)
    }
    ctx = context.WithValue(ctx, tenantKey, tenantID)
    return handler(ctx, req)
  }
}
```

**Per-Tenant Rate Limiting:**

```go
type TenantRateLimiter struct {
  mu      sync.Mutex
  tenants map[string]*rate.Limiter
}

func (l *TenantRateLimiter) Limit(ctx context.Context) error {
  tenantID := GetTenant(ctx)
  l.mu.Lock()
  lim, ok := l.tenants[tenantID]
  if !ok {
    lim = rate.NewLimiter(rate.Limit(100), 200) // 100 req/s, burst 200
    l.tenants[tenantID] = lim
  }
  l.mu.Unlock()
  if !lim.Allow() {
    return status.Errorf(codes.ResourceExhausted, "tenant %s rate limit exceeded", tenantID)
  }
  return nil
}
```

**Data Isolation:**
- All database queries include `WHERE tenant_id = ?`
- Job IDs are globally unique but namespaced per tenant
- Partition by tenant in the database
- Separate Raft state machines per tenant? No — use a single state machine with tenant prefix keys.

**Authorization Interceptor:**
```go
func AuthorizeTenantAccess() grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    tenantID := GetTenant(ctx)
    claims := GetUserClaims(ctx) // from auth interceptor

    if claims.TenantID != tenantID {
      return nil, status.Error(codes.PermissionDenied, "tenant mismatch")
    }

    // For admin users, allow cross-tenant read
    if claims.Role == "superadmin" && isReadOnly(info.FullMethod) {
      return handler(ctx, req)
    }

    return handler(ctx, req)
  }
}
```
</details>

---

### Prompt 5: Design a Hybrid gRPC + Message Queue System

**Context:** Chronos needs both real-time coordination (gRPC) and durable job storage (message queue). Design the hybrid architecture.

**Key requirements:**
- gRPC for worker heartbeats, live status streaming, control commands
- Message queue (Kafka/RabbitMQ) for durable job queues
- Jobs survive scheduler restarts
- Workers can be temporarily disconnected without losing jobs
- Exactly-once or at-least-once job delivery

<details><summary>Answer</summary>

**Architecture:**

```
[Client] ──(gRPC SubmitJob)──▶ [Scheduler]
                                    │
                    ┌───────────────┼───────────────┐
                    ▼                               ▼
              [Kafka: Job Queue]              [Raft State]
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
    [Worker 1]              [Worker 2]
        │                       │
        └──(gRPC bidir stream)──┘
                   │
             [Scheduler]
```

**Data Flow:**

1. **Client submits job** → gRPC → Scheduler
2. **Scheduler persists** job to Raft log and enqueues to Kafka
3. **Kafka consumer group** partitions jobs across workers
4. **Worker acks** Kafka message after processing
5. **Worker streams** heartbeats and status via gRPC bidirectional stream
6. **Scheduler pushes** control commands (cancel, priority change) via the gRPC stream

**Service Definitions:**

```protobuf
service ChronosControl {
  // Real-time control plane — gRPC
  rpc WorkerCoordinator(stream WorkerMessage) returns (stream ControlCommand);
}

service ChronosJob {
  // Job submission — gRPC (synchronous acknowledgment)
  rpc SubmitJob(SubmitJobRequest) returns (SubmitJobResponse);
  rpc WatchJobStatus(WatchJobStatusRequest) returns (stream JobStatusEvent);
}
```

**Why this hybrid design?**
- gRPC handles *real-time control* well: low latency, typed contracts, streaming
- Kafka handles *durable queuing*: persistence, replay, consumer groups, at-least-once delivery
- gRPC without a message queue would lose jobs on scheduler crash
- Kafka alone lacks real-time bidirectional streaming and typed service contracts
</details>

---

## 9. STAR Stories (3 templates)

### Template 1: Chronos — gRPC Service Design

**Situation:** Our team needed to build a distributed job scheduler for managing millions of scheduled jobs across 100+ worker nodes. Reliability, low-latency coordination, and typed contracts were critical.

**Task:** Design the service communication layer for Chronos. This included service definitions, worker-scheduler coordination, job submission, and status streaming.

**Action:**

1. **Service definition with proto3** — defined all messages and services in `.proto` files with proto3 syntax:
   - `ChronosJob` service for client-facing submit/get-status/watch
   - `ChronosWorker` service for worker heartbeat and registration
   - `ChronosControl` for bidirectional worker coordination

2. **Bidirectional streaming** — each worker opens one persistent gRPC stream for:
   - Heartbeats every second (worker → scheduler)
   - Task assignments (scheduler → worker)
   - Control commands (cancel, pause, priority change)
   - Result delivery (worker → scheduler)

3. **Flow control tuning** — increased `InitialWindowSize` to 1MB after profiling flow control stalls during high worker count (default 64KB was too small).

4. **Graceful shutdown** — implemented drain logic using `GracefulStop()` with a 30s timeout, plus pre-stop hooks in K8s for load balancer draining.

5. **Auth interceptor** — JWT validation in a unary and stream interceptor, with role-based access control for job operations using the `authz` package.

6. **Deadline propagation** — all RPC calls propagate client deadlines through the Raft consensus layer, ensuring jobs are cancelled if the client gives up.

**Result:**
- 10x more efficient than the previous REST+WebSocket system
- Worker reconnection under 500ms during scheduler leader failover
- Zero data loss during rolling deployments (graceful shutdown + hedging)
- The system handled 50,000+ concurrent gRPC streaming connections

**Lessons learned:**
- Default flow control windows are designed for small payloads; Chronos needed tuning
- gRPC-Web requires Envoy configuration — we spent two days debugging CORS headers
- mTLS between all services was a compliance requirement and added operational overhead
</details>

---

### Template 2: Microservices Migration from REST to gRPC

**Situation:** Our platform had 20+ microservices communicating via REST/JSON over HTTP/1.1. As traffic grew, we saw increased latency, higher CPU usage from JSON parsing, and frequent breaking contract changes.

**Task:** Lead the migration of internal service-to-service communication from REST to gRPC, improving performance and reliability.

**Action:**

1. **Contract-first approach** — defined all service interfaces in `.proto` files before implementing. This eliminated contract drift and forced teams to agree on schemas first.

2. **Incremental migration** — introduced gRPC alongside REST; each service was migrated independently. Used Envoy as a sidecar to bridge REST clients to gRPC services during the transition.

3. **Schema versioning** — used protobuf field numbers with reserved ranges to allow backward-compatible evolution. Introduced `protoc-gen-validate` for input validation.

4. **Performance tuning**:
   - Switched from JSON to binary protobuf — 40% smaller payloads
   - Used gRPC streaming for real-time event feeds (instead of polling)
   - Implemented client-side `round_robin` load balancing with DNS resolution

5. **Observability** — added logging interceptors (duration, method, status), OpenTelemetry tracing via `otelgrpc`, and metrics via `grpc.StatsHandler`.

**Result:**
- P50 latency reduced by 62% (from 12ms to 4.5ms)
- CPU utilization dropped 35% across services (less JSON parsing)
- Contract breakage incidents went from monthly to zero in 6 months
- New service integration time reduced from weeks to days (codegen from proto)

**Lessons learned:**
- REST-to-gRPC translators add latency; prefer end-to-end gRPC
- Teams new to protobuf need training on field numbering and backward compatibility
- Not all workloads benefit from gRPC — high-latency external APIs stayed on REST with OpenAPI
</details>

---

### Template 3: gRPC Production Debugging at Scale

**Situation:** After a major deployment at 2 AM, our production system experienced a 15-minute outage. All gRPC clients were getting `UNAVAILABLE` errors. The on-call engineer escalated.

**Task:** Debug and resolve the gRPC connectivity issue, then implement preventive measures.

**Action:**

1. **Initial triage** — clients showed `UNAVAILABLE` with "connection closed" errors. The gRPC server was running and healthy. `grpcurl` from localhost worked, but remote connections failed.

2. **Root cause investigation:**
   - The load balancer (NLB) health check was an HTTP GET on port 50051, but gRPC requires HTTP/2. The health check failed → NLB marked instances as unhealthy.
   - The deploy had switched from `healthz` (HTTP endpoint) to `grpc.health.v1.Health/Check`, but the NLB health check config wasn't updated.
   - The K8s ingress controller was also terminating TLS and routing to plaintext gRPC — but the gRPC server expected TLS, causing a protocol mismatch.

3. **Resolution:**
   - Changed NLB health check to TCP (port 50051) instead of HTTP
   - Updated K8s Ingress to use `nginx.ingress.kubernetes.io/backend-protocol: "GRPC"`
   - Implemented `grpc_health_probe` for K8s liveness/readiness checks

4. **Preventive measures:**
   - Added a gRPC keepalive enforcement policy: `MinTime: 5s, PermitWithoutStream: true`
   - Set `GRPC_GO_KEEPALIVE_MIN_TIME` to 10s to match load balancer idle timeout
   - Added client-side retry with exponential backoff for `UNAVAILABLE` errors
   - Created a runbook for gRPC connectivity issues (check health check type, TLS config, keepalive)

**Result:**
- Outage resolved in 30 minutes (once root cause identified)
- No recurrence in 6 months
- Runbook reduced MTTR from 30 min to 5 min for similar issues

**Lessons learned:**
- Never assume health checks work the same for gRPC and HTTP
- gRPC in K8s requires specific configuration at every layer (NLB, Ingress, probe)
- Always test failover and rolling deploy scenarios in staging with gRPC
</details>

---

## 10. Questions to Ask the Interviewer

### gRPC-Specific

1. "How does your team handle gRPC API versioning and backward compatibility? Do you use field reservation, and what's your strategy for breaking changes?"

2. "What's your approach to gRPC load balancing — client-side, proxy-based, or service mesh? Have you hit any edge cases with gRPC's default `pick_first` behavior?"

3. "How do you handle streaming at scale? Have you tuned flow control window sizes, and what are your current MaxConcurrentStreams limits?"

4. "What's your observability stack for gRPC services? Do you use OpenTelemetry interceptors, and are there any gaps in gRPC-specific metrics you've noticed?"

5. "Have you dealt with gRPC-Web in production? What did you choose for your web-to-gRPC bridge, and did you hit any limitations with the browser client?"

6. "How do you handle gRPC connection management — keepalive configuration, connection pooling, and recovery from `UNAVAILABLE` errors?"

### General Backend

7. "How does your team approach service decomposition? Do you have guidelines on when to create a new service vs. extend an existing one?"

8. "What's your incident response process like, and how do you balance feature velocity with reliability?"

9. "How do you manage technical debt — is there dedicated time, or is it handled as part of regular sprint work?"

10. "What does success look like for this role in the first 90 days? What would you want me to accomplish by then?"

11. "How does the team handle cross-team coordination for API changes? Is there an API review board or RFC process?"

12. "What's your perspective on monorepo vs. multi-repo for protobuf definitions? How do you share `.proto` files across teams?"

---

## 11. Red Flags to Avoid

1. **No awareness of flow control** — "I'd just stream everything without thinking about backpressure." Shows lack of systems thinking for TCP/HTTP/2 behavior at scale.

2. **Treating gRPC as "just another HTTP API"** — "gRPC is just REST with binary JSON." Ignores streaming, interceptors, flow control, typed contracts.

3. **Not setting deadlines** — "I don't set timeouts because the server knows best." Deadlines are mandatory in production gRPC.

4. **Assuming gRPC is always faster** — "gRPC is always faster because protobuf is smaller." Ignores that for tiny payloads, overhead can dominate.

5. **Ignoring TLS in production** — "We use `WithInsecure()` because it's internal." Violates security best practices and compliance.

6. **Not understanding proto3 defaults** — "We check if an int field was set by comparing to 0." Proto3 has no presence for scalars; use wrapper types.

7. **Overusing `Any`** — "We put everything in `Any` for flexibility." This breaks type safety and contract clarity.

8. **No retry strategy** — "gRPC handles retries for me." Built-in retry is opt-in and doesn't work for streaming.

9. **Mixing protobuf library versions** — Using `github.com/golang/protobuf` alongside `google.golang.org/protobuf`. Causes mysterious panics at runtime.

10. **Not testing with real network conditions** — "It works in Docker Compose; it'll work in production." Ignores real-world latency, packet loss, and proxy behavior.
