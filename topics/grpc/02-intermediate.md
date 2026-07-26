# gRPC — Tier 2: Intermediate (Streaming, Error Handling & Interceptors)

> **Target:** Senior Backend Engineer (8+ years, Go, PHP, JS)  
> **Your anchors:** Chronos (Go distributed job scheduler using Raft) — gRPC is the communication layer for scheduler workers, clients, and control plane.  
> **Prerequisites:** Tier 1 (gRPC overview, Protocol Buffers, HTTP/2, unary RPC, basic server/client setup)

This tier covers everything beyond unary RPC: the three streaming patterns, rich error handling, metadata, interceptors, authentication, and gRPC-web. As a senior engineer, you will be asked to design streaming APIs, reason about error semantics, and build middleware that cuts across every RPC in the system. Know these cold.

---

## Table of Contents

1. [Server-Side Streaming](#1-server-side-streaming)
2. [Client-Side Streaming](#2-client-side-streaming)
3. [Bidirectional Streaming](#3-bidirectional-streaming)
4. [Deadlines & Timeouts](#4-deadlines--timeouts)
5. [Metadata](#5-metadata)
6. [Error Handling & Status Codes](#6-error-handling--status-codes)
7. [Interceptors (Middleware)](#7-interceptors-middleware)
8. [Authentication (TLS & JWT)](#8-authentication-tls--jwt)
9. [gRPC-web](#9-grpc-web)
10. [Tier 2 Q&A Drill](#10-tier-2-qa-drill)

---

## 1. Server-Side Streaming

Server-side streaming is when the client sends a single request and the server sends back a stream of messages. The client reads until the stream is closed.

### Proto Definition

```protobuf
service JobService {
  // Server-side streaming: one request, multiple responses
  rpc ListJobs(ListJobsRequest) returns (stream Job);
}

message ListJobsRequest {
  string status = 1;      // filter by status: "running", "queued", "failed"
  int32  limit = 2;       // max results
  string namespace = 3;   // Chronos namespace filter
}

message Job {
  string id = 1;
  string name = 2;
  string status = 3;
  int64  created_at = 4;
  int64  finished_at = 5;
  map<string, string> labels = 6;
}
```

### Server Implementation (Go)

```go
func (s *jobServer) ListJobs(ctx context.Context, req *pb.ListJobsRequest, stream pb.JobService_ListJobsServer) error {
    // Fetch jobs from the scheduler state (Chronos store)
    jobs, err := s.store.QueryJobs(req.Namespace, req.Status, int(req.Limit))
    if err != nil {
        return status.Errorf(codes.Internal, "failed to query jobs: %v", err)
    }

    for _, job := range jobs {
        // Check for client cancellation before each send
        select {
        case <-ctx.Done():
            log.Printf("ListJobs cancelled by client: %v", ctx.Err())
            return ctx.Err()
        default:
        }

        err := stream.Send(&pb.Job{
            Id:         job.ID,
            Name:       job.Name,
            Status:     job.Status,
            CreatedAt:  job.CreatedAt.Unix(),
            FinishedAt: job.FinishedAt.Unix(),
            Labels:     job.Labels,
        })
        if err != nil {
            log.Printf("ListJobs send error: %v", err)
            return err
        }
    }

    // Returning nil signals end of stream
    return nil
}
```

### Client Implementation (Go)

```go
func listJobs(ctx context.Context, client pb.JobServiceClient, namespace, status string) error {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    stream, err := client.ListJobs(ctx, &pb.ListJobsRequest{
        Namespace: namespace,
        Status:    status,
        Limit:     100,
    })
    if err != nil {
        return fmt.Errorf("ListJobs stream open failed: %w", err)
    }

    for {
        job, err := stream.Recv()
        if err == io.EOF {
            break // stream ended cleanly
        }
        if err != nil {
            return fmt.Errorf("ListJobs recv error: %w", err)
        }

        log.Printf("Received job: %s (status=%s)", job.Name, job.Status)
        // process job...
    }

    return nil
}
```

### Use Cases

| Use Case | Why Server Streaming | Chronos Example |
|----------|---------------------|-----------------|
| Large result sets | Avoid loading everything into memory | `ListJobs` with thousands of results |
| Real-time updates | Client gets notified as events happen | `WatchJobStatus` — monitor job state transitions |
| Log streaming | Tail logs from a running job | `StreamJobLogs` — follow container output |
| Batch processing | Pipe results through a processing chain | `ExportJobs` — stream to sink |

### Backpressure

gRPC uses HTTP/2 flow control. The client advertises a window size (default 64 KB). The server cannot send more data than the window allows. If the client reads slowly, the server's `Send()` will block. This is backpressure built into the transport.

> **Senior note:** You don't need to implement your own backpressure mechanism in most cases. HTTP/2 flow control handles it. However, if you send messages much faster than the client can consume, you can exhaust server memory with queued frames. Monitor the `grpc_server_msg_sent` metrics.

### Error Handling During Streaming

```go
// Client: handle stream errors properly
for {
    job, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        // Check if it's a gRPC status error
        st, ok := status.FromError(err)
        if ok {
            switch st.Code() {
            case codes.DeadlineExceeded:
                log.Printf("ListJobs timed out after 30s")
            case codes.Canceled:
                log.Printf("ListJobs was cancelled")
            case codes.Unavailable:
                log.Printf("ListJobs server unavailable, retry later")
            default:
                log.Printf("ListJobs error: %s: %s", st.Code(), st.Message())
            }
        } else {
            log.Printf("ListJobs non-gRPC error: %v", err)
        }
        return err
    }
    // process job...
}
```

> **🪤 Trap:** Not handling client cancellation. If the client cancels the context, the server must stop sending. Always check `ctx.Done()` in streaming loops. Failure to do so wastes resources and leaves the server writing to a closed stream.
>
> **🪤 Trap:** Sending responses too fast without backpressure awareness. Even though HTTP/2 flow control exists, you can still buffer many frames in memory if the server goroutine keeps calling `Send()` faster than the transport can flush. If the client is slow, this is a memory bomb.
>
> **🪤 Trap:** Forgetting to check `stream.Recv()` error against `io.EOF`. Newcomers write `for { job, err := stream.Recv(); if err != nil { break } }` and miss the distinction between EOF (success) and a real error (failure).
>
> **🔎 Follow-up:** "How would you handle a client that disconnects mid-stream on the server side?" — Check `ctx.Done()` in the send loop. The context is cancelled when the client drops the connection. Your send will also return an error once the transport detects the disconnection.
>
> **🔎 Follow-up:** "What happens if the server sends data after the client has cancelled?" — The `Send()` will return an error (typically `codes.Canceled`). The server should stop sending.

---

## 2. Client-Side Streaming

Client-side streaming is when the client sends a stream of messages and the server sends back a single response after all messages are received.

### Proto Definition

```protobuf
service JobService {
  // Client-side streaming: multiple requests, single response
  rpc UploadLogs(stream LogEntry) returns (UploadSummary);
}

message LogEntry {
  string job_id      = 1;
  int64  timestamp   = 2;
  string severity    = 3;  // "info", "warn", "error"
  string message     = 4;
  bytes  raw_data    = 5;
}

message UploadSummary {
  int32  entries_received = 1;
  int64  first_timestamp  = 2;
  int64  last_timestamp   = 3;
  int32  error_count      = 4;
  string upload_id        = 5;
}
```

### Server Implementation (Go)

```go
func (s *jobServer) UploadLogs(stream pb.JobService_UploadLogsServer) error {
    var (
        entries []*pb.LogEntry
        errCount int32
    )

    for {
        entry, err := stream.Recv()
        if err == io.EOF {
            break // client finished sending
        }
        if err != nil {
            log.Printf("UploadLogs recv error: %v", err)
            return err
        }

        // Process each entry — validate and store
        if entry.Severity == "error" {
            errCount++
        }

        // Simulate validation
        if entry.JobId == "" {
            return status.Errorf(codes.InvalidArgument, "log entry missing job_id")
        }

        entries = append(entries, entry)

        // Optional: periodic progress (send headers mid-stream)
        // grpc.SendHeader(ctx, metadata.Pairs("x-entries-received", strconv.Itoa(len(entries))))
    }

    // After stream ends, compute summary and send response
    if len(entries) == 0 {
        return status.Errorf(codes.InvalidArgument, "no log entries received")
    }

    summary := &pb.UploadSummary{
        EntriesReceived: int32(len(entries)),
        FirstTimestamp:  entries[0].Timestamp,
        LastTimestamp:   entries[len(entries)-1].Timestamp,
        ErrorCount:      errCount,
        UploadId:        uuid.New().String(),
    }

    return stream.SendAndClose(summary)
}
```

### Client Implementation (Go)

```go
func uploadLogs(ctx context.Context, client pb.JobServiceClient, jobID string, messages []string) error {
    ctx, cancel := context.WithTimeout(ctx, 60*time.Second)
    defer cancel()

    stream, err := client.UploadLogs(ctx)
    if err != nil {
        return fmt.Errorf("UploadLogs stream open failed: %w", err)
    }

    for i, msg := range messages {
        entry := &pb.LogEntry{
            JobId:    jobID,
            Timestamp: time.Now().Unix(),
            Severity: "info",
            Message:  msg,
        }

        if err := stream.Send(entry); err != nil {
            // Server may have rejected the stream — check the error
            return fmt.Errorf("UploadLogs send error at entry %d: %w", i, err)
        }
    }

    // Close the send side and wait for the server's response
    summary, err := stream.CloseAndRecv()
    if err != nil {
        return fmt.Errorf("UploadLogs close/recv error: %w", err)
    }

    log.Printf("Upload complete: %s (%d entries, %d errors)",
        summary.UploadId, summary.EntriesReceived, summary.ErrorCount)
    return nil
}
```

### Use Cases

| Use Case | Why Client Streaming | Chronos Example |
|----------|---------------------|-----------------|
| File uploads | Break large files into chunks | `UploadJobArtifact` — upload binary artifacts |
| Batch data ingestion | Send many items without waiting per item | `SubmitBatchJobs` — submit 10K jobs |
| Telemetry collection | Continuous metrics push | `ReportWorkerHealth` — worker heartbeat with metrics |
| Large payload splitting | Avoid 4MB gRPC message limit | `UploadLargePayload` — chunk and reassemble |

### Error Handling on Client Stream

```go
// Server can return an error mid-stream — client must handle it
func uploadLogsRobust(ctx context.Context, client pb.JobServiceClient, entries []*pb.LogEntry) error {
    stream, err := client.UploadLogs(ctx)
    if err != nil {
        return err
    }

    // Buffered channel to collect result
    resultCh := make(chan error, 1)

    go func() {
        summary, err := stream.CloseAndRecv()
        if err != nil {
            resultCh <- err
            return
        }
        log.Printf("Upload successful: %s", summary.UploadId)
        resultCh <- nil
    }()

    for i, entry := range entries {
        // If the server returned an error, send will fail
        if err := stream.Send(entry); err != nil {
            // Drain the resultCh to avoid goroutine leak
            <-resultCh
            return fmt.Errorf("send failed at entry %d: %w", i, err)
        }
    }

    // Signal we're done sending
    if err := stream.CloseSend(); err != nil {
        return fmt.Errorf("CloseSend failed: %w", err)
    }

    return <-resultCh
}
```

> **🪤 Trap:** Not calling `CloseAndRecv()` on the client. If the client calls `stream.Send()` in a loop and exits without closing, the server will never receive EOF, and the client will never get the response. The client blocks forever waiting for the response.
>
> **🪤 Trap:** Assuming the server processes messages before `Recv` returns. gRPC may buffer received messages on the server side. The server handler may not have called `Recv()` yet when the client thinks it has sent the data. Do not assume synchronous processing order between client sends and server receives.
>
> **🪤 Trap:** Sending huge individual messages. The default gRPC max message size is 4 MB. Messages larger than this are rejected. If you need to send large payloads, chunk them on the client side into smaller messages and reassemble on the server.
>
> **🔎 Follow-up:** "How does the server reject messages mid-stream?" — The server returns a status error from the Recv loop. When the client calls `Send()` next, the error propagates. Alternatively, the server can return an error from any point, and the next client `Send()` or `CloseAndRecv()` will get it.
>
> **🔎 Follow-up:** "Can the server send progress updates during client streaming?" — Yes, via streaming response headers. The server can call `grpc.SendHeader()` during processing. The client reads these with `stream.Header()`.

---

## 3. Bidirectional Streaming

Bidirectional streaming is when both client and server send a stream of messages to each other, independently and asynchronously. The streams are independent — reads and writes happen concurrently.

### Proto Definition

```protobuf
service JobService {
  // Bidirectional streaming: both sides send multiple messages
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);

  // Chronos-specific: worker heartbeat exchange
  rpc WorkerHeartbeat(stream HeartbeatRequest) returns (stream HeartbeatResponse);
}

message ChatMessage {
  string user_id = 1;
  string text    = 2;
  int64  timestamp = 3;
}

message HeartbeatRequest {
  string worker_id    = 1;
  string job_id       = 2;
  string status       = 3;  // "running", "healthy", "stalled"
  int32  cpu_percent  = 4;
  int64  memory_bytes = 5;
  int32  active_tasks = 6;
}

message HeartbeatResponse {
  string worker_id          = 1;
  bool   acknowledged       = 2;
  string action             = 3;  // "none", "cancel_job", "reschedule", "drain"
  int32  next_heartbeat_sec = 4;  // hint for next heartbeat interval
}
```

### Server Implementation (Bidirectional — Go)

```go
func (s *jobServer) WorkerHeartbeat(stream pb.JobService_WorkerHeartbeatServer) error {
    // Channel to serialize outgoing messages
    sendCh := make(chan *pb.HeartbeatResponse, 10)

    // errCh captures send goroutine errors
    errCh := make(chan error, 1)

    // Send goroutine: reads from sendCh, writes to stream
    go func() {
        for resp := range sendCh {
            if err := stream.Send(resp); err != nil {
                errCh <- fmt.Errorf("send heartbeat failed: %w", err)
                return
            }
        }
    }()

    // Recv goroutine (main goroutine): reads from stream, processes
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // Client closed send — we're done
            close(sendCh)
            break
        }
        if err != nil {
            close(sendCh)
            return err
        }

        // Process heartbeat from worker
        log.Printf("Heartbeat from worker %s: job=%s status=%s CPU=%d%%",
            req.WorkerId, req.JobId, req.Status, req.CpuPercent)

        // Determine action based on worker state
        action := "none"
        if req.Status == "stalled" {
            action = "reschedule"
        }

        // Send response back through the channel
        select {
        case sendCh <- &pb.HeartbeatResponse{
            WorkerId:          req.WorkerId,
            Acknowledged:      true,
            Action:            action,
            NextHeartbeatSec:  15,
        }:
        case <-stream.Context().Done():
            close(sendCh)
            return stream.Context().Err()
        }
    }

    // Wait for send goroutine to drain
    select {
    case err := <-errCh:
        return err
    default:
        return nil
    }
}
```

### Client Implementation (Bidirectional — Go)

```go
func workerHeartbeat(ctx context.Context, client pb.JobServiceClient, workerID string) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Minute)
    defer cancel()

    stream, err := client.WorkerHeartbeat(ctx)
    if err != nil {
        return fmt.Errorf("heartbeat stream open failed: %w", err)
    }

    // errCh captures errors from either goroutine
    errCh := make(chan error, 2)

    // Goroutine 1: Send heartbeats periodically
    go func() {
        ticker := time.NewTicker(15 * time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                req := &pb.HeartbeatRequest{
                    WorkerId:    workerID,
                    Status:      "healthy",
                    CpuPercent:  42,
                    MemoryBytes: 2_147_483_648, // 2GB
                    ActiveTasks: 3,
                }
                if err := stream.Send(req); err != nil {
                    errCh <- fmt.Errorf("send heartbeat failed: %w", err)
                    return
                }
            case <-ctx.Done():
                errCh <- ctx.Err()
                return
            }
        }
    }()

    // Goroutine 2 (main goroutine): Receive responses
    for {
        resp, err := stream.Recv()
        if err == io.EOF {
            return nil // stream closed cleanly
        }
        if err != nil {
            return fmt.Errorf("recv heartbeat response failed: %w", err)
        }

        log.Printf("Heartbeat ack for %s: action=%s interval=%ds",
            resp.WorkerId, resp.Action, resp.NextHeartbeatSec)

        // If server says to drain, stop sending
        if resp.Action == "drain" {
            log.Printf("Worker %s instructed to drain", workerID)
            stream.CloseSend()
            return nil
        }
    }
}
```

### Use Cases

| Use Case | Why Bidirectional | Chronos Example |
|----------|-------------------|-----------------|
| Real-time chat | Both sides send independently | Coordination between scheduler nodes |
| Collaborative editing | Ops sent in both directions | Job configuration live sync |
| Real-time trading feeds | Subscribe and receive | Job execution price feeds |
| Worker coordination | Heartbeat with dynamic control | `WorkerHeartbeat` — worker ↔ scheduler |
| Job status streaming | Report status, receive commands | `JobExecutionStream` — report progress, receive cancel/resume |

### Concurrency Architecture

```
Client                          Server
  │                               │
  │  ┌─ Send goroutine ──┐       │  ┌─ Recv (main goroutine) ─┐
  │  │ stream.Send(msg)  │───────┼─>│ stream.Recv() → process │
  │  └───────────────────┘       │  └─────────────────────────┘
  │                               │
  │  ┌─ Recv (main goroutine) ─┐ │  ┌─ Send goroutine ────────┐
  │  │ stream.Recv() → process │<───┼──── stream.Send(resp)   │
  │  └──────────────────────────┘ │  └─────────────────────────┘
```

Key points:
- `Recv` and `Send` on the same stream can happen concurrently — they're independent operations on different HTTP/2 streams.
- Use channels to coordinate between the send and recv goroutines.
- Either side can close their send direction with `CloseSend()`.

> **🪤 Trap:** Assuming `Recv` and `Send` are synchronized. They are **independent** operations. You cannot assume that a `Send` on the client is received by a matching `Recv` on the server in lockstep. If you need request-response pairing, use a correlation ID in the message or stick with unary RPC.
>
> **🪤 Trap:** Deadlocks from blocked sends. If the send channel is unbuffered and the send goroutine is not consuming responses, both sides can deadlock. Always use a buffered channel (size 10-100) or a separate goroutine for sending.
>
> **🪤 Trap:** Not handling half-close properly. When one side calls `CloseSend()`, the other side's `Recv()` returns `io.EOF`. But the side that called `CloseSend()` can still receive. Make sure your loop handles this correctly — don't exit the receive loop just because you finished sending.
>
> **🔎 Follow-up:** "How would you implement a bidirectional stream that must maintain order?" — Use a sequence number in each message. The receiver buffers out-of-order messages and reorders them before processing. This adds complexity — consider whether unary RPC with batching would suffice.
>
> **🔎 Follow-up:** "How do you handle worker disconnection in Chronos's heartbeat RPC?" — Set a deadline on the context. If no heartbeat is received within the deadline, the server marks the worker as lost and reschedules its jobs. The worker reconnects and re-registers.

---

## 4. Deadlines & Timeouts

Deadlines and timeouts prevent RPCs from hanging forever. In gRPC, deadlines are propagated from client to server automatically via HTTP/2 headers.

### Setting Deadlines on the Client

```go
// WithTimeout: relative duration
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// WithDeadline: absolute time
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// Use the context in the RPC call
resp, err := client.ListJobs(ctx, &pb.ListJobsRequest{Status: "running"})
```

### Server Detecting Deadlines

```go
func (s *jobServer) ListJobs(ctx context.Context, req *pb.ListJobsRequest, stream pb.JobService_ListJobsServer) error {
    // Check if a deadline was set
    deadline, ok := ctx.Deadline()
    if ok {
        remaining := time.Until(deadline)
        log.Printf("ListJobs called with deadline in %v", remaining)

        if remaining < 1*time.Second {
            return status.Errorf(codes.DeadlineExceeded,
                "deadline too short: %v remaining", remaining)
        }
    }

    // Expensive operation — check deadline before proceeding
    if !ok || time.Until(deadline) > 10*time.Second {
        // Only do the expensive prefetch if we have time
        s.prefetchCache(req.Namespace)
    }

    // Watch for deadline in the streaming loop
    for _, job := range jobs {
        select {
        case <-ctx.Done():
            // Deadline exceeded or cancelled
            switch ctx.Err() {
            case context.DeadlineExceeded:
                return status.Errorf(codes.DeadlineExceeded, "deadline exceeded")
            case context.Canceled:
                return status.Errorf(codes.Canceled, "client cancelled")
            default:
                return ctx.Err()
            }
        default:
        }

        stream.Send(&pb.Job{...})
    }

    return nil
}
```

### Default Behavior

```go
// BAD: No deadline — RPC can hang forever
resp, err := client.ListJobs(context.Background(), req)

// GOOD: Always set a deadline
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
resp, err := client.ListJobs(ctx, req)

// GOOD: Use the default timeout from configuration
ctx, cancel := context.WithTimeout(context.Background(), s.cfg.RPCTimeout)
defer cancel()
```

### Deadline Propagation

```
Client                       Server
  │                            │
  │ --- gRPC Call --->         │
  │ Deadline: T+5s (header)    │
  │                            │
  │                    context.WithDeadline(ctx, deadline)
  │                    ctx.Done() fired at T+5s
  │                            │
  │ <--- DeadlineExceeded ---- │
```

The client's deadline is automatically propagated to the server via the `grpc-timeout` header. The server's context is derived from this deadline. Both sides see the timeout simultaneously.

### Graceful Handling on the Server

```go
// Check deadlines before expensive operations
func (s *jobServer) ProcessJobBatch(ctx context.Context, req *pb.BatchRequest) (*pb.BatchResponse, error) {
    deadline, ok := ctx.Deadline()
    if !ok {
        // No deadline set — use a server-enforced maximum
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, 60*time.Second)
        defer cancel()
    }

    for i, job := range req.Jobs {
        // Check time remaining before each expensive operation
        if ok && time.Until(deadline) < 5*time.Second {
            // Save what we have and return partial results
            return &pb.BatchResponse{
                ProcessedCount: int32(i),
                Errors:         nil,
                PartialResult:  true,
            }, nil
        }

        if err := s.processJob(ctx, job); err != nil {
            return nil, err
        }
    }

    return &pb.BatchResponse{ProcessedCount: int32(len(req.Jobs))}, nil
}
```

### Client Handling Deadline Errors

```go
func callWithRetry(ctx context.Context, client pb.JobServiceClient) error {
    const maxRetries = 3

    for attempt := 0; attempt < maxRetries; attempt++ {
        ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
        defer cancel()

        _, err := client.ListJobs(ctx, &pb.ListJobsRequest{Status: "running"})
        if err == nil {
            return nil
        }

        st, ok := status.FromError(err)
        if ok && st.Code() == codes.DeadlineExceeded {
            // Deadline exceeded — retry with longer timeout
            log.Printf("Deadline exceeded (attempt %d), retrying with longer timeout", attempt)
            time.Sleep(time.Duration(attempt+1) * time.Second)
            continue
        }

        return err // non-retryable error
    }

    return fmt.Errorf("all retries exhausted")
}
```

| Scenario | Deadline Setting | Why |
|----------|-----------------|-----|
| Unary get (e.g., `GetJob`) | 5s | Fast lookup, no streaming |
| Server stream (e.g., `ListJobs`) | 30-60s | Unknown number of results |
| Client stream (e.g., `UploadLogs`) | Based on data size | Estimate 1s per MB + buffer |
| Bidirectional (e.g., `WorkerHeartbeat`) | Per-call context | Heartbeat interval × 3 |
| Batch processing | 60-300s | Depends on batch size |

> **🪤 Trap:** Not setting any deadlines. Without a deadline, an RPC can hang forever if the server is unreachable or hangs. The connection will eventually be killed by keepalive, but that could take minutes. **Always set a deadline.**
>
> **🪤 Trap:** Setting too-short deadlines for streaming calls. A 5-second deadline on `ListJobs` that returns 10K results will always fail. Deadlines must account for the total expected duration of the stream.
>
> **🪤 Trap:** Ignoring `ctx.Done()` in server handlers. Even if the deadline is set, the server must check the context. A server that ignores context cancellation will continue processing after the deadline has passed.
>
> **🪤 Trap:** Not handling `DeadlineExceeded` errors on the client. The client must handle this error gracefully — retry with backoff, log, or fail with a meaningful message.
>
> **🔎 Follow-up:** "Can the server set its own deadline tighter than the client's?" — Yes. The server can create a derived context with a shorter deadline than the client's propagated one. This is useful for internal SLA enforcement.
>
> **🔎 Follow-up:** "What happens if the server deadline expires before the client deadline?" — The server returns `DeadlineExceeded`. The client receives this as a gRPC error. The client's context is NOT affected — it can still make other calls.
>
> **🔎 Follow-up:** "How do deadlines interact with retries?" — Each retry attempt should have its own deadline. The overall operation should use a parent context with a longer timeout. If the parent context expires, stop retrying.

---

## 5. Metadata

Metadata is key-value pairs transmitted as HTTP/2 headers. gRPC uses metadata for things like authentication tokens, correlation IDs, tracing headers, and client version information.

### Metadata in the HTTP/2 Frame

```
HEADERS frame (gRPC):
  :method = POST
  :path = /package.Service/Method
  :scheme = http
  content-type = application/grpc
  te = trailers
  authorization = Bearer <jwt_token>
  x-correlation-id = abc-123
  x-client-version = 1.2.3
  grpc-timeout = 5s

DATA frame (proto payload)

HEADERS frame (trailers):
  grpc-status = 0
  grpc-message = OK
```

### Working with Metadata in Go

```go
import "google.golang.org/grpc/metadata"
```

#### Client Sending Metadata

```go
// Method 1: metadata.New
md := metadata.New(map[string]string{
    "authorization":    "Bearer " + token,
    "x-correlation-id": correlationID,
    "x-client-version": "1.2.3",
})
ctx := metadata.NewOutgoingContext(context.Background(), md)

// Method 2: metadata.Pairs (duplicate keys allowed)
md := metadata.Pairs(
    "authorization", "Bearer " + token,
    "x-correlation-id", correlationID,
    "x-retry-attempt", "3",
)
ctx := metadata.NewOutgoingContext(ctx, md)

// Method 3: Append to existing metadata
ctx = metadata.AppendToOutgoingContext(ctx, "x-forwarded-for", "10.0.0.1")

// Use the context
resp, err := client.ListJobs(ctx, req)
```

#### Server Reading Metadata

```go
func (s *jobServer) ListJobs(ctx context.Context, req *pb.ListJobsRequest) (*pb.ListJobsResponse, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "no metadata found")
    }

    // Get authorization token
    authHeaders := md.Get("authorization")
    if len(authHeaders) == 0 {
        return nil, status.Errorf(codes.Unauthenticated, "no authorization header")
    }
    token := authHeaders[0] // "Bearer <jwt>"

    // Get correlation ID for logging
    corrIDs := md.Get("x-correlation-id")
    correlationID := ""
    if len(corrIDs) > 0 {
        correlationID = corrIDs[0]
    }

    log.Printf("[%s] ListJobs called", correlationID)
    return s.listJobs(ctx, req)
}
```

#### Server Sending Response Headers

```go
func (s *jobServer) ListJobs(ctx context.Context, req *pb.ListJobsRequest, stream pb.JobService_ListJobsServer) error {
    // Method 1: Set response headers before or during streaming
    header := metadata.Pairs(
        "x-response-version", "1.0",
        "x-request-id", uuid.New().String(),
    )
    if err := grpc.SetHeader(ctx, header); err != nil {
        log.Printf("failed to set header: %v", err)
    }

    // Method 2: Send headers immediately (before any response)
    // grpc.SendHeader(ctx, header)

    // ... stream data ...

    // Set trailing metadata (sent after the final response)
    trailer := metadata.Pairs(
        "x-entries-sent", strconv.Itoa(count),
        "x-processing-time-ms", strconv.Itoa(int(time.Since(start).Milliseconds())),
    )
    grpc.SetTrailer(ctx, trailer)

    return nil
}
```

#### Client Reading Response Headers & Trailers

```go
func listJobsWithMetadata(ctx context.Context, client pb.JobServiceClient) error {
    stream, err := client.ListJobs(ctx, &pb.ListJobsRequest{Status: "running"})
    if err != nil {
        return err
    }

    // Read response headers (blocking until server sends them)
    header, err := stream.Header()
    if err != nil {
        return fmt.Errorf("read header failed: %w", err)
    }
    log.Printf("Response version: %s", header.Get("x-response-version"))
    log.Printf("Request ID: %s", header.Get("x-request-id"))

    // Read the stream
    for {
        job, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        log.Printf("Job: %s", job.Name)
    }

    // Read trailing metadata (after stream completes)
    trailer := stream.Trailer()
    log.Printf("Entries sent: %s", trailer.Get("x-entries-sent"))
    log.Printf("Processing time: %sms", trailer.Get("x-processing-time-ms"))

    return nil
}
```

### Common Use Cases

| Use Case | Key | Value Example |
|----------|-----|---------------|
| Authentication | `authorization` | `Bearer eyJhbGciOi...` |
| API Key | `x-api-key` | `sk-abc123def456` |
| Correlation ID | `x-correlation-id` | `uuid-abc-123-456` |
| Request ID | `x-request-id` | `req-uuid-789` |
| Client Version | `x-client-version` | `2.4.1` |
| User Agent | `x-user-agent` | `chronos-worker/1.0` |
| Tracing (OpenTelemetry) | `traceparent` | `00-0af765...` |
| Forwarded For | `x-forwarded-for` | `10.0.0.1` |

### Binary Metadata

```go
// Binary values use the "-bin" suffix and are base64-encoded
md := metadata.Pairs(
    "trace-id-bin", base64.StdEncoding.EncodeToString(traceID),
    "auth-token-bin", base64.StdEncoding.EncodeToString(authBytes),
)
ctx := metadata.NewOutgoingContext(context.Background(), md)
```

### Metadata Size Limits

gRPC has a default metadata size limit of 8 KB. If you need larger, configure it:

```go
// Server: increase metadata size limit
s := grpc.NewServer(
    grpc.MaxRecvMsgSize(16 * 1024),     // 16 KB max receive
    grpc.MaxHeaderListSize(16 * 1024),  // 16 KB header list
)

// Client: increase metadata size limit
conn, err := grpc.Dial("localhost:50051",
    grpc.WithDefaultCallOptions(
        grpc.MaxCallRecvMsgSize(16 * 1024),   // 16 KB
        grpc.MaxCallHeaderListSize(16 * 1024), // 16 KB
    ),
    grpc.WithTransportCredentials(insecureCredentials),
)
```

> **🪤 Trap:** Not using lowercase keys. gRPC normalizes all metadata keys to lowercase. If you set `X-Correlation-Id`, it becomes `x-correlation-id`. Always use lowercase in your code for consistency.
>
> **🪤 Trap:** Storing large payloads in metadata. Metadata is sent as HTTP/2 headers. Headers are a single HPACK block — large metadata blocks increase latency and can hit the 8 KB default limit. Keep values small (< 1 KB). Use the message body for large payloads.
>
> **🪤 Trap:** Mixing metadata for auth and business logic. Auth metadata (tokens, API keys) should be handled in interceptors, not in business logic handlers. Keep your handlers clean — they should receive parsed, validated values via context.
>
> **🔎 Follow-up:** "How do you propagate metadata across microservices?" — Use a metadata propagator. Extract headers from the incoming context, copy them to the outgoing context for downstream calls. Libraries like `go.opentelemetry.io/otel/propagation` handle this for tracing.
>
> **🔎 Follow-up:** "What's the difference between response headers and trailers?" — Response headers are sent with the first HTTP/2 DATA frame (or before). Trailers are sent after all data frames. Trailers are typically used for post-processing metadata like entry counts. Trailers are guaranteed to be delivered even if the stream fails mid-way.

---

## 6. Error Handling & Status Codes

gRPC defines a canonical set of status codes. Every RPC returns a status code + an optional message + optional error details. Consistent error handling across your services is a hallmark of a senior engineer.

### gRPC Status Codes

| Code | Number | gRPC Name | When to Use |
|------|--------|-----------|-------------|
| OK | 0 | `OK` | Success |
| CANCELLED | 1 | `Canceled` | Operation cancelled, typically by the caller |
| UNKNOWN | 2 | `Unknown` | Default error when no specific code fits |
| INVALID_ARGUMENT | 3 | `InvalidArgument` | Client specified an invalid argument |
| DEADLINE_EXCEEDED | 4 | `DeadlineExceeded` | Deadline expired before operation completed |
| NOT_FOUND | 5 | `NotFound` | Requested entity was not found |
| ALREADY_EXISTS | 6 | `AlreadyExists` | Entity already exists (create conflict) |
| PERMISSION_DENIED | 7 | `PermissionDenied` | Caller doesn't have permission |
| RESOURCE_EXHAUSTED | 8 | `ResourceExhausted` | Resource quota exhausted, rate limit hit |
| FAILED_PRECONDITION | 9 | `FailedPrecondition` | System not in required state (e.g., job not idle) |
| ABORTED | 10 | `Aborted` | Operation aborted due to concurrency conflict (e.g., optimistic lock) |
| OUT_OF_RANGE | 11 | `OutOfRange` | Operation was attempted past valid range |
| UNIMPLEMENTED | 12 | `Unimplemented` | Method not implemented |
| INTERNAL | 13 | `Internal` | Internal server error (bugs, invariants) |
| UNAVAILABLE | 14 | `Unavailable` | Service temporarily unavailable (load, maintenance) |
| DATA_LOSS | 15 | `DataLoss` | Unrecoverable data loss or corruption |
| UNAUTHENTICATED | 16 | `Unauthenticated` | Request not authenticated or auth expired |

### Returning Errors from the Server

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Simple error with message
func (s *jobServer) GetJob(ctx context.Context, req *pb.GetJobRequest) (*pb.Job, error) {
    if req.JobId == "" {
        return nil, status.Error(codes.InvalidArgument, "job_id is required")
    }

    job, err := s.store.GetJob(req.JobId)
    if err == store.ErrNotFound {
        return nil, status.Errorf(codes.NotFound, "job %s not found", req.JobId)
    }
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to get job: %v", err)
    }

    return job, nil
}

// Rich error model with details
func (s *jobServer) CreateJob(ctx context.Context, req *pb.CreateJobRequest) (*pb.Job, error) {
    var fieldErrors []*errdetails.BadRequest_FieldViolation

    if req.Name == "" {
        fieldErrors = append(fieldErrors, &errdetails.BadRequest_FieldViolation{
            Field:       "name",
            Description: "Job name must not be empty",
        })
    }
    if req.Owner == "" {
        fieldErrors = append(fieldErrors, &errdetails.BadRequest_FieldViolation{
            Field:       "owner",
            Description: "Job owner must be specified",
        })
    }
    if len(req.Command) == 0 {
        fieldErrors = append(fieldErrors, &errdetails.BadRequest_FieldViolation{
            Field:       "command",
            Description: "At least one command is required",
        })
    }
    if req.TimeoutSec < 1 || req.TimeoutSec > 86400 {
        fieldErrors = append(fieldErrors, &errdetails.BadRequest_FieldViolation{
            Field:       "timeout_sec",
            Description: "Timeout must be between 1 and 86400 seconds",
        })
    }

    if len(fieldErrors) > 0 {
        st := status.New(codes.InvalidArgument, "validation failed")
        st, err := st.WithDetails(&errdetails.BadRequest{
            FieldViolations: fieldErrors,
        })
        if err != nil {
            return nil, status.Errorf(codes.Internal, "failed to build error details: %v", err)
        }
        return nil, st.Err()
    }

    // ... create job ...
    return s.store.CreateJob(req)
}
```

### Standard Error Details

```go
import "google.golang.org/genproto/googleapis/rpc/errdetails"

// BadRequest: field-level validation errors
st.WithDetails(&errdetails.BadRequest{
    FieldViolations: []*errdetails.BadRequest_FieldViolation{
        {Field: "email", Description: "invalid email format"},
    },
})

// RetryInfo: how long to wait before retrying
st.WithDetails(&errdetails.RetryInfo{
    RetryDelay: durationpb.New(5 * time.Second),
})

// DebugInfo: stack trace or debug info (not for production clients)
st.WithDetails(&errdetails.DebugInfo{
    StackEntries: []string{"file.go:42"},
    Detail:       "internal state inconsistent",
})

// QuotaFailure: rate limit exceeded
st.WithDetails(&errdetails.QuotaFailure{
    Violations: []*errdetails.QuotaFailure_Violation{
        {Subject: "namespace:prod", Description: "rate limit: 100 req/s"},
    },
})

// ErrorInfo: structured error with domain and reason
st.WithDetails(&errdetails.ErrorInfo{
    Domain:  "scheduler.chronos.internal",
    Reason:  "JOB_ALREADY_RUNNING",
    Metadata: map[string]string{"job_id": req.JobId},
})

// PreconditionFailure: system not in correct state
st.WithDetails(&errdetails.PreconditionFailure{
    Violations: []*errdetails.PreconditionFailure_Violation{
        {Type: "JOB", Subject: req.JobId, Description: "job is not in IDLE state"},
    },
})

// RequestInfo: help clients debug requests
st.WithDetails(&errdetails.RequestInfo{
    RequestId:   correlationID,
    ServingData: "us-east-1-az3",
})

// ResourceInfo: which resource had the issue
st.WithDetails(&errdetails.ResourceInfo{
    ResourceType: "worker",
    ResourceName: "worker-42",
    Owner:        "scheduler",
    Description:  "worker is draining",
})
```

### Client Error Handling

```go
func getJob(client pb.JobServiceClient, jobID string) (*pb.Job, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    job, err := client.GetJob(ctx, &pb.GetJobRequest{JobId: jobID})
    if err != nil {
        // Parse the gRPC status from the error
        st, ok := status.FromError(err)
        if !ok {
            // Not a gRPC error (transport error, etc.)
            return nil, fmt.Errorf("non-gRPC error: %w", err)
        }

        switch st.Code() {
        case codes.NotFound:
            // Graceful: return nil, not an error
            log.Printf("Job %s not found", jobID)
            return nil, nil

        case codes.InvalidArgument:
            // Parse rich error details
            for _, detail := range st.Details() {
                switch d := detail.(type) {
                case *errdetails.BadRequest:
                    for _, v := range d.FieldViolations {
                        log.Printf("Validation error: %s = %s", v.Field, v.Description)
                    }
                }
            }
            return nil, fmt.Errorf("invalid request: %s", st.Message())

        case codes.DeadlineExceeded:
            return nil, fmt.Errorf("request timed out: %s", st.Message())

        case codes.Unavailable:
            // Retryable
            for _, detail := range st.Details() {
                if retryInfo, ok := detail.(*errdetails.RetryInfo); ok {
                    wait := retryInfo.RetryDelay.AsDuration()
                    log.Printf("Retry suggested after %v", wait)
                }
            }
            return nil, NewRetryableError(err)

        case codes.PermissionDenied:
            return nil, fmt.Errorf("permission denied: %s", st.Message())

        default:
            return nil, fmt.Errorf("unexpected error [%s]: %s", st.Code(), st.Message())
        }
    }

    return job, nil
}
```

### Error Handling Decision Tree

```
                    ┌─ Is it a transport error (connection)?
                    │     → codes.Unavailable → retry
                    │
Error received ─────┼─ Is it a clear client mistake?
                    │     → codes.InvalidArgument | NotFound | AlreadyExists
                    │     → Return to client with rich details
                    │
                    ├─ Is it an auth failure?
                    │     → codes.Unauthenticated | PermissionDenied
                    │     → Return to client (may need re-auth)
                    │
                    ├─ Is it a transient state issue?
                    │     → codes.Unavailable | ResourceExhausted | Aborted
                    │     → Retry with backoff
                    │
                    ├─ Is it a timeout?
                    │     → codes.DeadlineExceeded
                    │     → Retry with longer deadline or abort
                    │
                    └─ Is it a server bug?
                          → codes.Internal | DataLoss
                          → Log alert, return generic message to client
```

> **🪤 Trap:** Returning `Internal` for client errors. A bad request is not an internal error. Using `Internal` for validation failures confuses clients and makes monitoring useless — you can't distinguish "we broke something" from "client sent garbage." Use `InvalidArgument`, `NotFound`, `AlreadyExists`, etc.
>
> **🪤 Trap:** Not using rich error details for validation errors. Returning `status.Error(codes.InvalidArgument, "bad request")` gives the client no information about which field was wrong. Use `errdetails.BadRequest` with `FieldViolations`. Senior engineers design APIs that are easy to debug.
>
> **🪤 Trap:** Exposing internal error messages to clients. `status.Errorf(codes.Internal, "mysql connection pool exhausted: %v", err)` leaks internals. Return a generic message like "internal server error" and log the detail server-side. Only expose details for client-facing errors.
>
> **🪤 Trap:** Not checking `status.Code(err) == codes.DeadlineExceeded` specifically. A deadline exceeded error needs different handling than a generic error — you may want to retry with a longer timeout rather than fail permanently.
>
> **🔎 Follow-up:** "How do you handle partial failures in batch operations with gRPC?" — Return a response with per-item errors rather than failing the entire batch. Use `errdetails.BadRequest` or a `repeated ErrorDetail` field in the response message.
>
> **🔎 Follow-up:** "When would you use `Aborted` vs `Unavailable`?" — `Aborted` = operation conflict (optimistic locking, CAS failure), retry with backoff. `Unavailable` = service temporarily down (deploy, overload), retry on a different server or after a delay.
>
> **🔎 Follow-up:** "How do you propagate errors across service boundaries in a distributed system?" — Map domain errors to gRPC status codes at each boundary. Do not blindly forward internal errors. Use `errdetails.ErrorInfo` with a `Domain` field to indicate which service produced the error.

---

## 7. Interceptors (Middleware)

Interceptors are gRPC's middleware. They wrap RPC handlers to add cross-cutting concerns: logging, authentication, metrics, rate limiting, tracing, panic recovery. Every senior engineer should know how to write and chain interceptors.

### Server-Side Unary Interceptor

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Logging interceptor
func loggingUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    start := time.Now()
    log.Printf("--> %s", info.FullMethod)

    resp, err := handler(ctx, req)

    duration := time.Since(start)
    code := status.Code(err)
    log.Printf("<-- %s [%s] %v", info.FullMethod, code, duration)

    return resp, err
}

// Auth interceptor
func authUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "no metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Errorf(codes.Unauthenticated, "no authorization token")
    }

    token := tokens[0]
    claims, err := validateJWT(token)
    if err != nil {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }

    // Inject user info into context
    ctx = context.WithValue(ctx, userContextKey, claims)

    return handler(ctx, req)
}

// Recovery interceptor — catches panics and returns Internal error
func recoveryUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
    defer func() {
        if r := recover(); r != nil {
            // Log the stack trace
            log.Printf("PANIC in %s: %v\n%s", info.FullMethod, r, debug.Stack())
            err = status.Errorf(codes.Internal, "internal server error")
        }
    }()

    return handler(ctx, req)
}

// Rate limiter interceptor
func rateLimitUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    // Token bucket rate limiter
    if !s.rateLimiter.Allow() {
        st := status.New(codes.ResourceExhausted, "rate limit exceeded")
        st, _ = st.WithDetails(&errdetails.RetryInfo{
            RetryDelay: durationpb.New(time.Second),
        })
        return nil, st.Err()
    }

    return handler(ctx, req)
}
```

### Server-Side Stream Interceptor

```go
func loggingStreamInterceptor(srv interface{}, stream grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
    start := time.Now()
    log.Printf("--> [stream] %s (isClientStream=%v isServerStream=%v)",
        info.FullMethod, info.IsClientStream, info.IsServerStream)

    err := handler(srv, stream)

    duration := time.Since(start)
    code := status.Code(err)
    log.Printf("<-- [stream] %s [%s] %v", info.FullMethod, code, duration)

    return err
}
```

### Chaining Interceptors

```go
// gRPC provides ChainUnaryInterceptor and ChainStreamInterceptor
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        recoveryUnaryInterceptor,   // outermost (runs first)
        loggingUnaryInterceptor,     // second
        rateLimitUnaryInterceptor,   // third
        authUnaryInterceptor,        // innermost (runs last, closest to handler)
    ),
    grpc.ChainStreamInterceptor(
        loggingStreamInterceptor,
    ),
)
```

### Interceptor Execution Order

```
Client                    Server
  │                         │
  │ ---- gRPC Call ---->   │
  │                         │
  │                    ┌─ recoveryUnaryInterceptor (outermost)
  │                    │   │
  │                    │   └─ loggingUnaryInterceptor
  │                    │       │
  │                    │       └─ rateLimitUnaryInterceptor
  │                    │           │
  │                    │           └─ authUnaryInterceptor
  │                    │               │
  │                    │               └─ handler (actual method)
  │                    │                   │
  │                    │               ┌─ response
  │                    │           ┌───┘
  │                    │       ┌───┘
  │                    │   ┌───┘
  │                    │   └─ recovery (if panic)
  │                    │
  │ <--- response ----│
```

### Client-Side Unary Interceptor

```go
// Client-side logging interceptor
func clientLoggingInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    start := time.Now()
    log.Printf("--> %s", method)

    err := invoker(ctx, method, req, reply, cc, opts...)

    duration := time.Since(start)
    code := status.Code(err)
    log.Printf("<-- %s [%s] %v", method, code, duration)

    return err
}

// Client-side retry interceptor
func clientRetryInterceptor(maxRetries int) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        var err error
        for attempt := 0; attempt < maxRetries; attempt++ {
            err = invoker(ctx, method, req, reply, cc, opts...)
            if err == nil {
                return nil
            }

            st, ok := status.FromError(err)
            if !ok || st.Code() != codes.Unavailable {
                // Non-retryable error
                return err
            }

            wait := time.Duration(1<<uint(attempt)) * 100 * time.Millisecond
            log.Printf("Retry %d/%d for %s after %v", attempt+1, maxRetries, method, wait)

            select {
            case <-time.After(wait):
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return err
    }
}

// Client-side auth interceptor
func clientAuthInterceptor(token string) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        md := metadata.Pairs("authorization", "Bearer "+token)
        ctx = metadata.NewOutgoingContext(ctx, md)
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}

// Client-side stream interceptor (for auth)
func clientAuthStreamInterceptor(token string) grpc.StreamClientInterceptor {
    return func(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
        md := metadata.Pairs("authorization", "Bearer "+token)
        ctx = metadata.NewOutgoingContext(ctx, md)
        return streamer(ctx, desc, cc, method, opts...)
    }
}
```

### Using Client Interceptors

```go
conn, err := grpc.Dial(
    "localhost:50051",
    grpc.WithTransportCredentials(creds),
    grpc.WithUnaryInterceptor(
        grpc.ChainUnaryInterceptor(
            clientLoggingInterceptor,
            clientAuthInterceptor(token),
            clientRetryInterceptor(3),
        ),
    ),
    grpc.WithStreamInterceptor(
        clientAuthStreamInterceptor(token),
    ),
)
```

### Common Interceptor Patterns

| Pattern | What It Does | Server/Client |
|---------|-------------|---------------|
| Logging | Log method, duration, status code, request size | Both |
| Metrics | Increment counters, record histograms (Prometheus) | Both |
| Auth | Extract and validate JWT from metadata | Server |
| Auth injection | Add JWT/auth token to metadata | Client |
| Rate limiting | Token bucket or leaky bucket | Server |
| Request validation | Validate incoming proto messages | Server |
| Tracing | Create/manage spans (OpenTelemetry) | Both |
| Panic recovery | recover() from panics, return Internal | Server |
| Retry | Retry failed calls on retryable codes | Client |
| Circuit breaker | Open circuit after N failures | Client |
| Request ID | Inject request ID into context | Server |
| Compression | Compress/decompress payloads | Both |
| Timeout | Enforce per-method timeouts | Server |

> **🪤 Trap:** Not handling context deadlines in interceptors. If your auth interceptor makes a database call to validate a token (e.g., session lookup), and no deadline is set, that call can hang. Always check `ctx.Done()` or use `context.WithTimeout` for interceptor-side calls.
>
> **🪤 Trap:** Modifying the request in an interceptor. Proto messages are immutable by convention. If you need to add fields, create a new context value or use a wrapper. Use `ctx = context.WithValue(ctx, key, val)` to pass data to handlers.
>
> **🪤 Trap:** Panic recovery interceptor not being the outermost chain. If recovery is not the outermost, a panic in a higher-ordered interceptor will not be caught. Always put `recoveryUnaryInterceptor` first in the chain.
>
> **🪤 Trap:** Blocking in interceptors. Interceptors run synchronously before the handler. A blocking auth call (e.g., HTTP request to an auth service) delays every RPC. Use async validation, caching, or a fast path.
>
> **🔎 Follow-up:** "How do you pass data from an interceptor to a handler?" — Use `context.WithValue` with typed context keys. The handler can then extract values with `ctx.Value(userContextKey)`. Do not use string keys — define a package-level type.
>
> **🔎 Follow-up:** "How do you skip auth for certain methods (e.g., health check)?" — Check `info.FullMethod` in the interceptor. If the method is `/grpc.health.v1.Health/Check`, call the handler directly without auth validation. Maintain a map of public methods.
>
> **🔎 Follow-up:** "How do interceptors impact performance?" — Each interceptor adds function call overhead and potential latency (auth, rate limiting). Measure with benchmarks. For high-throughput services, consider which interceptors are truly necessary per-method, or use separate gRPC servers for internal vs external traffic.

---

## 8. Authentication (TLS & JWT)

Authentication in gRPC operates at two levels: transport-level (TLS/mTLS) and application-level (JWT/OAuth). For Chronos, you typically use TLS for inter-service communication and JWT for client authentication.

### TLS (Transport Layer Security)

#### Server-Side TLS Configuration

```go
import (
    "google.golang.org/grpc/credentials"
    "google.golang.org/grpc"
)

// Method 1: Load from file
func newTLSServer() (*grpc.Server, error) {
    creds, err := credentials.NewServerTLSFromFile(
        "certs/server.crt",  // certificate
        "certs/server.key",  // private key
    )
    if err != nil {
        return nil, fmt.Errorf("failed to load TLS credentials: %w", err)
    }

    s := grpc.NewServer(grpc.Creds(creds))
    return s, nil
}

// Method 2: Configure manually
func newTLSManualServer() (*grpc.Server, error) {
    certPool := x509.NewCertPool()
    caCert, err := os.ReadFile("certs/ca.crt")
    if err != nil {
        return nil, err
    }
    if !certPool.AppendCertsFromPEM(caCert) {
        return nil, fmt.Errorf("failed to append CA cert")
    }

    cert, err := tls.LoadX509KeyPair("certs/server.crt", "certs/server.key")
    if err != nil {
        return nil, err
    }

    creds := credentials.NewTLS(&tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientAuth:   tls.RequireAndVerifyClientCert, // for mTLS
        ClientCAs:    certPool,
        MinVersion:   tls.VersionTLS12,
    })

    return grpc.NewServer(grpc.Creds(creds)), nil
}
```

#### Client-Side TLS Configuration

```go
// Method 1: Load CA cert from file
func newTLSClientConn(target string) (*grpc.ClientConn, error) {
    creds, err := credentials.NewClientTLSFromFile(
        "certs/ca.crt",  // CA certificate to verify the server
        "server.chronos.internal",  // expected server name (SNI)
    )
    if err != nil {
        return nil, fmt.Errorf("failed to load TLS credentials: %w", err)
    }

    conn, err := grpc.Dial(target, grpc.WithTransportCredentials(creds))
    if err != nil {
        return nil, fmt.Errorf("failed to dial: %w", err)
    }

    return conn, nil
}

// Method 2: System cert pool
func newTLSClientSystemPool(target string) (*grpc.ClientConn, error) {
    creds := credentials.NewClientTLSFromCert(nil, "") // uses system cert pool
    conn, err := grpc.Dial(target, grpc.WithTransportCredentials(creds))
    return conn, err
}

// Method 3: mTLS client (sends client certificate)
func newMTLSClient(target string) (*grpc.ClientConn, error) {
    cert, err := tls.LoadX509KeyPair("certs/client.crt", "certs/client.key")
    if err != nil {
        return nil, err
    }

    creds := credentials.NewTLS(&tls.Config{
        Certificates: []tls.Certificate{cert},
        ServerName:   "server.chronos.internal",
        MinVersion:   tls.VersionTLS12,
    })

    return grpc.Dial(target, grpc.WithTransportCredentials(creds))
}
```

### gRPC Insecure (Never in Production)

```go
// NEVER do this in production
conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())

// Production — always use TLS
conn, _ := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(creds))
```

### JWT (JSON Web Token) Authentication

JWT tokens are sent via metadata and validated in an interceptor.

#### Client: Inject JWT into Metadata

```go
func newAuthenticatedClient(target, jwtToken string) (*grpc.ClientConn, error) {
    creds := credentials.NewClientTLSFromFile("certs/ca.crt", "server.chronos.internal")

    conn, err := grpc.Dial(target,
        grpc.WithTransportCredentials(creds),
        grpc.WithUnaryInterceptor(clientAuthInterceptor(jwtToken)),
        grpc.WithStreamInterceptor(clientAuthStreamInterceptor(jwtToken)),
    )
    if err != nil {
        return nil, err
    }

    return conn, nil
}

// Per-call JWT (alternative to interceptor)
func getJobWithJWT(ctx context.Context, client pb.JobServiceClient, token, jobID string) (*pb.Job, error) {
    md := metadata.Pairs("authorization", "Bearer "+token)
    ctx = metadata.NewOutgoingContext(ctx, md)

    return client.GetJob(ctx, &pb.GetJobRequest{JobId: jobID})
}
```

#### Server: Validate JWT in Auth Interceptor

```go
type UserClaims struct {
    UserID    string   `json:"user_id"`
    Roles     []string `json:"roles"`
    Namespace string   `json:"namespace"`
    jwt.RegisteredClaims
}

type contextKey string

const userContextKey contextKey = "user"

func authUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    // Skip auth for health check
    if info.FullMethod == "/grpc.health.v1.Health/Check" {
        return handler(ctx, req)
    }

    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "missing metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Errorf(codes.Unauthenticated, "missing authorization token")
    }

    tokenStr := tokens[0]
    // Strip "Bearer " prefix
    if len(tokenStr) > 7 && tokenStr[:7] == "Bearer " {
        tokenStr = tokenStr[7:]
    }

    claims := &UserClaims{}
    token, err := jwt.ParseWithClaims(tokenStr, claims, func(token *jwt.Token) (interface{}, error) {
        // Validate signing method
        if _, ok := token.Method.(*jwt.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        // Return the public key (loaded at startup, cached)
        return s.publicKey, nil
    })
    if err != nil || !token.Valid {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }

    // Inject user info into context for downstream handlers
    ctx = context.WithValue(ctx, userContextKey, claims)

    // Optionally inject namespace from claims into metadata
    ctx = metadata.AppendToOutgoingContext(ctx, "x-namespace", claims.Namespace)

    return handler(ctx, req)
}
```

#### Using Claims in Handlers

```go
func (s *jobServer) ListJobs(ctx context.Context, req *pb.ListJobsRequest) (*pb.ListJobsResponse, error) {
    claims, ok := ctx.Value(userContextKey).(*UserClaims)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "user not authenticated")
    }

    log.Printf("User %s listing jobs in namespace %s", claims.UserID, claims.Namespace)

    // Optionally enforce namespace scoping
    if req.Namespace == "" {
        req.Namespace = claims.Namespace
    } else if req.Namespace != claims.Namespace && !hasRole(claims.Roles, "admin") {
        return nil, status.Errorf(codes.PermissionDenied,
            "cannot access namespace %s", req.Namespace)
    }

    return s.store.ListJobs(ctx, req.Namespace, req.Status)
}
```

### mTLS (Mutual TLS)

In mutual TLS, both sides present certificates. The server verifies the client's cert, and the client verifies the server's cert.

```
Client                          Server
  │                               │
  │ Client Hello                  │
  │ Server Hello + Cert --------> │
  │ <-------- Client Cert Request│
  │ Client Cert + Cert Verify -> │
  │ (server verifies client)      │
  │ <------- Finished ----------- │
  │                               │
  │ Both sides trust each other   │
```

```go
// Server: require client cert
serverCreds := credentials.NewTLS(&tls.Config{
    Certificates: []tls.Certificate{serverCert},
    ClientAuth:   tls.RequireAndVerifyClientCert,
    ClientCAs:    clientCertPool,
    MinVersion:   tls.VersionTLS12,
})
s := grpc.NewServer(grpc.Creds(serverCreds))

// Client: send client cert
clientCreds := credentials.NewTLS(&tls.Config{
    Certificates: []tls.Certificate{clientCert},
    ServerName:   "server.chronos.internal",
    MinVersion:   tls.VersionTLS12,
})
conn, _ := grpc.Dial("server:50051", grpc.WithTransportCredentials(clientCreds))
```

### Authentication Strategy for Chronos

| Component | Authentication | Rationale |
|-----------|---------------|-----------|
| Scheduler ↔ Worker | mTLS | Both sides authenticate; no JWTs needed for internal communication |
| Client (CLI) ↔ Scheduler | TLS + JWT | Identify users and enforce namespace scoping |
| Web UI → Envoy → Scheduler | TLS + JWT | Envoy terminates TLS, forwards JWT to scheduler |
| Cross-datacenter replication | mTLS | Strong mutual authentication between clusters |

> **🪤 Trap:** Using `grpc.WithInsecure()` in production. This disables all TLS. Traffic is sent in plaintext. Credentials can be stolen. **Never use `WithInsecure` outside local development.**
>
> **🪤 Trap:** Putting JWT in the URL or query parameters. gRPC doesn't use URLs in the traditional REST sense, but the principle applies: tokens in URLs can be logged, leaked in referer headers, and cached. Always use the `authorization` metadata header.
>
> **🪤 Trap:** Not caching token validation results. If every RPC triggers a JWT verification (which involves asymmetric crypto), it adds significant CPU overhead. Cache the decoded claims for the token's validity period (with appropriate revocation checks).
>
> **🪤 Trap:** Passing authentication context as Go values with untyped keys. `context.WithValue(ctx, "user", claims)` is fragile. Use a typed, package-level key: `type contextKey string; const userContextKey contextKey = "user"`.
>
> **🔎 Follow-up:** "How do you handle token revocation?" — Maintain a server-side blocklist (Redis) of revoked token IDs (`jti`). Check the blocklist in your auth interceptor. For high-security environments, use short-lived tokens (15 min) with refresh tokens.
>
> **🔎 Follow-up:** "Should you use TLS or JWT for internal service-to-service auth?" — Both. TLS provides transport security (encryption, server identity). JWT provides application-level identity (which service, which user, which roles). For Chronos scheduler ↔ worker, mTLS alone may suffice because identity is the certificate CN. For user-facing APIs, add JWT.
>
> **🔎 Follow-up:** "How do you rotate TLS certificates in production without downtime?" — Use a certificate manager (cert-manager on K8s, or internal CA). Reload certificates on SIGHUP or use a file watcher. gRPC supports `tls.GetConfigForClient` callback for dynamic certificate loading.

---

## 9. gRPC-web

gRPC-web enables browser-based clients to communicate with gRPC services. Browsers cannot speak raw gRPC because they lack fine-grained control over HTTP/2 (no trailers, no bidirectional streaming from JS). gRPC-web solves this with a proxy layer.

### Why gRPC-web Exists

```
Traditional REST:
  Browser → REST API → JSON → Works natively

gRPC Direct:
  Browser → gRPC → ❌ Browsers can't control HTTP/2 frames or read trailers

gRPC-web:
  Browser → gRPC-web → Envoy proxy → gRPC → ✅ Works
```

### Architecture

```
┌──────────┐   gRPC-web    ┌──────────┐   gRPC    ┌──────────┐
│  Browser  │ ───────────> │  Envoy   │ ────────> │  gRPC    │
│ (JS/TS)   │ <─────────── │  Proxy   │ <──────── │  Server  │
└──────────┘               └──────────┘           └──────────┘
  text/grpc-web              HTTP/2                  HTTP/2
```

### Envoy Configuration

```yaml
# envoy.yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: grpc_web
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route:
                  cluster: grpc_backend
                  timeout: 30s
          http_filters:
          - name: envoy.filters.http.grpc_web     # gRPC-web filter
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
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
              socket_address:
                address: grpc-server
                port_value: 50051
```

### gRPC-web Client (JavaScript)

```javascript
// Using @grpc/grpc-js or protobuf-generated clients with gRPC-web transport
const { GrpcWebClient } = require('./grpc-web-client');
const { JobServiceClient } = require('./proto/job_grpc_web_pb');

const client = new JobServiceClient(
    'http://localhost:8080',   // Envoy proxy address
    null,                       // credentials (handled by proxy)
    { transport: GrpcWebClient }
);

const request = new ListJobsRequest();
request.setStatus('running');

// Server-side streaming works
const stream = client.listJobs(request, {});
stream.on('data', (job) => {
    console.log(`Job: ${job.getName()}`);
});
stream.on('end', () => {
    console.log('Stream ended');
});
stream.on('error', (err) => {
    console.error(`Error: ${err}`);
});
```

### gRPC-web Limitations

| Feature | gRPC-native | gRPC-web |
|---------|-------------|----------|
| Unary RPC | ✅ Full support | ✅ Full support |
| Server-side streaming | ✅ Full support | ✅ Supported (via chunked transfer) |
| Client-side streaming | ✅ Full support | ❌ Not supported |
| Bidirectional streaming | ✅ Full support | ❌ Not supported |
| Trailers | ✅ Native HTTP/2 trailers | ✅ Emulated in response body |
| Binary payloads | ✅ Protobuf binary | ✅ Base64-encoded (text format) |
| TLS | ✅ Native | ✅ Via HTTPS (Envoy terminates) |

### grpc-gateway (RESTful JSON Alternative)

grpc-gateway generates a RESTful JSON API from your proto file annotations, serving alongside gRPC.

```protobuf
import "google/api/annotations.proto";

service JobService {
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

  rpc CreateJob(CreateJobRequest) returns (Job) {
    option (google.api.http) = {
      post: "/v1/jobs"
      body: "*"
    };
  }
}
```

```bash
# Generate both gRPC and gateway code
protoc -I . \
  --go_out . --go-grpc_out . \
  --grpc-gateway_out . \
  proto/job.proto
```

```go
// Start both gRPC and HTTP gateway servers
func main() {
    // gRPC server
    grpcServer := grpc.NewServer()
    jobServer := &jobServer{}
    pb.RegisterJobServiceServer(grpcServer, jobServer)

    // HTTP gateway
    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithTransportCredentials(insecureCreds)}
    err := pb.RegisterJobServiceHandlerFromEndpoint(ctx, mux, "localhost:50051", opts)
    if err != nil {
        log.Fatalf("failed to register gateway: %v", err)
    }

    // Serve both
    go grpcServer.Serve(grpcLis)
    http.ListenAndServe(":8080", mux)
}
```

### When to Use What

| Scenario | Solution |
|----------|----------|
| Browser needs gRPC | gRPC-web via Envoy |
| Mobile apps | gRPC-native (no proxy needed) |
| REST clients (legacy) | grpc-gateway |
| Both gRPC and REST | grpc-gateway + gRPC on same server |
| Server-side streaming in browser | gRPC-web ✅ |
| Client streaming in browser | ❌ gRPC-web doesn't support — use grpc-gateway with chunked upload or WebSocket |

> **🪤 Trap:** Assuming gRPC-web has full gRPC feature parity. Client-side streaming and bidirectional streaming are NOT supported in gRPC-web. If your API design requires bidirectional streaming, you need a WebSocket bridge or a separate REST endpoint for browser clients.
>
> **🪤 Trap:** Forgetting to configure Envoy's gRPC-web filter. Envoy does not automatically translate gRPC-web to gRPC. You must add the `envoy.filters.http.grpc_web` filter to your HTTP connection manager configuration.
>
> **🪤 Trap:** gRPC-web performance overhead. gRPC-web uses base64 encoding for binary protobuf data, which adds ~33% overhead. For large payloads, this is significant. Consider using JSON transcoding (grpc-gateway) for browsers if payload size is a concern.
>
> **🪤 Trap:** Not using grpc-gateway for REST clients. gRPC-web requires a JS gRPC-web client library. If you have REST clients (curl, Postman, non-JS), provide a REST API via grpc-gateway. Do not force REST clients to use gRPC-web.
>
> **🔎 Follow-up:** "How would you support client-side streaming in the browser?" — You can't with gRPC-web alone. Options: (1) Use grpc-gateway with a POST endpoint that accepts the entire payload. (2) Use WebSockets to implement a custom streaming protocol. (3) Upload via multipart HTTP and process server-side.
>
> **🔎 Follow-up:** "Does gRPC-web work with any proxy, or only Envoy?" — Envoy is the most common, but other proxies support it: Cloudflare gRPC, NGINX (with gRPC-web module), and gRPC-web Go proxy (improbable-eng/grpc-web). Envoy is the reference implementation.
>
> **🔎 Follow-up:** "How do you handle authentication with gRPC-web?" — The browser sends JWT tokens in the `Authorization` header. Envoy forwards this header to the gRPC server as metadata. The server's auth interceptor validates it. CORS must be configured on Envoy for browser requests.

---

## 10. Tier 2 Q&A Drill

Drill these questions until you can answer them fluently. Cover the answer, then say it out loud as if to an interviewer. Record which questions you stumble on.

---

**Q1:** What are the three streaming patterns in gRPC, and when would you use each?

<details>
<summary>Answer</summary>

- **Server-side streaming** (`rpc ListJobs(Request) returns (stream Job)`): One request, multiple responses. Use for large result sets (avoid pagination overhead), real-time notifications (job status updates), log streaming, and monitoring feeds.
- **Client-side streaming** (`rpc UploadLogs(stream LogEntry) returns (UploadSummary)`): Multiple requests, single response. Use for file uploads, batch data ingestion, telemetry collection, and large payload splitting.
- **Bidirectional streaming** (`rpc Chat(stream ChatMsg) returns (stream ChatMsg)`): Independent streams in both directions. Use for real-time chat, collaborative editing, worker heartbeats with dynamic control, and trading feeds.

Chronos context: `WorkerHeartbeat` is bidirectional — workers report status and scheduler responds with actions (none, cancel, drain). `ListJobs` is server-streaming. `UploadJobArtifact` is client-streaming.
</details>

---

**Q2:** How does backpressure work in gRPC streaming? Do you need to implement your own flow control?

<details>
<summary>Answer</summary>

gRPC uses HTTP/2 flow control at the transport layer. Each side advertises an initial window size (default 64 KB per stream). The sender cannot send more data than the receiver's window allows. When the receiver reads and processes data, it sends WINDOW_UPDATE frames to replenish the window. If the receiver is slow, `Send()` blocks on the sender side. This is built-in backpressure — you generally don't need application-level flow control.

However, be aware: if your server goroutine calls `Send()` in a tight loop faster than the transport can flush, frames buffer in memory. This can exhaust server RAM under high concurrency. Monitor `grpc_server_msg_sent` metrics and consider using `grpc.WithInitialWindowSize()` to tune flow control for your workload.
</details>

---

**Q3:** What happens if a client disconnects mid-stream? How should the server handle it?

<details>
<summary>Answer</summary>

When the client disconnects, the server's context is cancelled (`ctx.Done()` is closed). The server's `stream.Send()` will return an error (typically `codes.Canceled`). The server must:

1. Check `ctx.Done()` in a select statement before each `Send()`.
2. Return from the handler when the context is cancelled or when `Send()` fails.
3. Not continue processing — the client isn't listening.

A common trap is writing a streaming loop that only checks `Send()` errors but doesn't check `ctx.Done()`. If `Send()` hasn't been called recently (e.g., between sends in a slow stream), the server can waste resources processing data that nobody will receive.
</details>

---

**Q4:** When should you use `CloseAndRecv()` vs `CloseSend()` on a client-streaming call?

<details>
<summary>Answer</summary>

- `CloseAndRecv()`: Closes the send side of the stream and waits for the server's single response. Use for client-streaming RPCs (`rpc Upload(stream Req) returns (Resp)`). This is the standard way to finish a client stream.
- `CloseSend()`: Closes only the send side, indicating no more messages will be sent. The client can still receive responses. Use in bidirectional streaming when you're done sending but expect more responses.

Trap: Using `CloseSend()` on a client-streaming RPC and forgetting to call `Recv()` — you'll never get the server's response. Use `CloseAndRecv()` for client-streaming RPCs.
</details>

---

**Q5:** How do deadlines propagate from client to server in gRPC?

<details>
<summary>Answer</summary>

The client sets a deadline using `context.WithTimeout()` or `context.WithDeadline()`. The gRPC framework serializes this deadline into the `grpc-timeout` HTTP/2 header (e.g., `grpc-timeout: 5s`). The server reads this header and derives its own context from the deadline. If the server makes downstream gRPC calls, it passes its context, which includes the remaining deadline.

Both sides observe the same timeout. When it expires, the server's `ctx.Done()` fires, and the server should return `codes.DeadlineExceeded`. The client also sees its context expire and can act accordingly.

The server can enforce a stricter deadline than the client's by creating a derived context with a shorter timeout. It should NOT extend the client's deadline — that defeats the purpose.
</details>

---

**Q6:** What are the common gRPC status codes and when do you use each?

<details>
<summary>Answer</summary>

| Code | When |
|------|------|
| `OK` | Everything worked |
| `Canceled` | Operation cancelled by caller |
| `InvalidArgument` | Client specified wrong input |
| `NotFound` | Requested resource doesn't exist |
| `AlreadyExists` | Resource creation conflict |
| `PermissionDenied` | Not authorized for this action |
| `Unauthenticated` | Missing/invalid authentication |
| `ResourceExhausted` | Rate limited, quota exceeded |
| `FailedPrecondition` | System not in correct state |
| `Aborted` | Concurrency conflict (optimistic lock) |
| `OutOfRange` | Pagination beyond valid range |
| `Unimplemented` | Method not implemented |
| `Internal` | Server bug, invariant violated |
| `Unavailable` | Service temporarily down |
| `DataLoss` | Unrecoverable data loss |
| `DeadlineExceeded` | Timeout before completion |
| `Unknown` | Catch-all for unexpected errors |

Senior engineer practice: Use `InvalidArgument` with `errdetails.BadRequest` for validation errors. Never expose `Internal` details to clients. Use `Unavailable` for retryable failures. Use `Aborted` for optimistic locking failures.
</details>

---

**Q7:** What is the order of interceptor execution when using `grpc.ChainUnaryInterceptor`?

<details>
<summary>Answer</summary>

Interceptors are executed from outermost to innermost before the handler, and then unwound from innermost to outermost after the handler.

```go
grpc.ChainUnaryInterceptor(
    recovery,   // 1st to run before handler, last after handler
    logging,    // 2nd before, 2nd after
    auth,       // 3rd before, 1st after (closest to handler)
)
```

Execution: `recovery → logging → auth → handler → auth → logging → recovery`

The recovery interceptor must be outermost so it can catch panics in any interceptor or the handler. Think of it like middleware in HTTP — each layer wraps the next.
</details>

---

**Q8:** How would you implement authentication for a gRPC service that supports both JWT and mTLS?

<details>
<summary>Answer</summary>

mTLS and JWT operate at different layers and can be used together:

1. **Transport layer (TLS/mTLS)**: Configure the gRPC server with `tls.Config` requiring client certificates. This identifies the calling service (machine identity). Use for service-to-service communication.

2. **Application layer (JWT)**: Create an auth interceptor that:
   - Extracts the `authorization` metadata header.
   - Parses and validates the JWT (check signature with public key, verify expiry, check claims).
   - Injects user info into the context via `context.WithValue`.
   - Returns `codes.Unauthenticated` on failure.

For Chronos: Workers use mTLS for scheduler communication (machine identity). CLI clients use JWT (user identity). Both can be validated simultaneously — the interceptor checks JWT if present, but mTLS-based calls (from known workers) can skip JWT validation by checking the TLS peer certificate.
</details>

---

**Q9:** What are the limitations of gRPC-web, and how do you work around them?

<details>
<summary>Answer</summary>

Limitations:
1. **No client-side streaming** — browser cannot send a stream of messages.
2. **No bidirectional streaming** — both sides cannot stream simultaneously.
3. **Base64 overhead** — binary protobuf is base64-encoded, ~33% larger payloads.
4. **Requires a proxy** — Envoy or similar must translate gRPC-web to gRPC.

Workarounds:
- For client streaming: Use grpc-gateway (REST endpoint that accepts the full payload) or WebSocket.
- For bidirectional streaming: Use WebSocket or fall back to polling.
- For overhead: Use JSON transcoding via grpc-gateway if payload size matters more than schema strictness.
- Always configure Envoy's `grpc_web` filter and handle CORS for browser clients.
</details>

---

**Q10:** How would you design a bidirectional streaming API for Chronos worker heartbeats? What error handling, deadlines, and concurrency considerations apply?

<details>
<summary>Answer</summary>

Design:

```protobuf
rpc WorkerHeartbeat(stream HeartbeatRequest) returns (stream HeartbeatResponse);
```

- **Concurrency**: Use separate goroutines for send and receive on both sides. Use a buffered channel (size 10-100) to decouple send from recv, preventing deadlocks.
- **Deadlines**: Set a per-heartbeat deadline (e.g., 30s). If no heartbeat received within that window, the server should close the stream and mark the worker as lost.
- **Error handling**: On `ctx.Done()` (client disconnect), stop processing and return. On send errors, log and return. Distinguish between `io.EOF` (clean close) and real errors.
- **Backpressure**: The server sends a `next_heartbeat_sec` hint in each response. The worker adjusts its heartbeat interval based on server load (longer interval = less load).
- **Actions**: Server responds with `action`: "none" (keep going), "cancel_job" (current job cancelled), "reschedule" (job moved to another worker), "drain" (worker should stop accepting new jobs).
- **Reconnection**: If the stream breaks, the worker reconnects with backoff. The scheduler detects stale workers by missing heartbeats and reschedules their jobs.

This is a departure from simple polling — bidirectional streaming gives the scheduler real-time control over workers without the worker needing to poll for commands.
</details>

---

> **End of Tier 2.** You should now be able to design streaming APIs, implement authentication and authorization, build custom interceptors, handle errors with rich details, and reason about gRPC-web trade-offs. Move to [03-senior.md](./03-senior.md) for load balancing, production deployment, performance tuning, and Chronos-specific architecture.
