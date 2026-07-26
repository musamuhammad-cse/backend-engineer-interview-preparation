# JavaScript / Node.js — Tier 3: Senior (Architecture, Performance & Operations)

> You have 8+ years shipping backend systems in PHP, Go, and JavaScript. You've run multi-tenant SaaS platforms, trading systems, and distributed job schedulers in production. This tier assumes you know the language mechanics from Tier 1 and the framework patterns from Tier 2. Here, we cover what breaks at scale: event loop starvation, memory leaks, CPU-bound workloads, graceful shutdown in orchestrated environments, microservice communication, distributed tracing, and the operational runbooks you need to debug a production Node.js service at 3am with a pager on your nightstand.

---

## Table of Contents

1. [Advanced Async Patterns](#1-advanced-async-patterns)
2. [Event Loop Diagnosis](#2-event-loop-diagnosis)
3. [Worker Threads](#3-worker-threads)
4. [Child Processes](#4-child-processes)
5. [Memory Management & Leaks](#5-memory-management--leaks)
6. [Performance Profiling](#6-performance-profiling)
7. [Cluster Mode & PM2](#7-cluster-mode--pm2)
8. [Graceful Shutdown](#8-graceful-shutdown)
9. [Microservices with Node.js](#9-microservices-with-nodejs)
10. [Message Queues (Bull/BullMQ)](#10-message-queues-bullbullmq)
11. [Caching Strategies](#11-caching-strategies)
12. [Database Optimization](#12-database-optimization)
13. [Stream Processing](#13-stream-processing)
14. [Observability](#14-observability)
15. [OWASP in Node.js](#15-owasp-in-nodejs)
16. [TypeScript for Node.js](#16-typescript-for-nodejs)
17. [gRPC in Node.js](#17-grpc-in-nodejs)
18. [Tier 3 Q&A Drill](#18-tier-3-qa-drill)

---

## 1. Advanced Async Patterns

### Event Loop Starvation

The event loop is single-threaded. Any synchronous CPU-bound work blocks it. When blocked, no I/O, no timers, no incoming requests can be processed. At 8+ years, you should not be surprised by this — but you must be able to **diagnose and quantify** it.

```javascript
// Blocking the event loop — real scenario: processing a CSV of 500K rows
const fs = require('fs');
const data = fs.readFileSync('large-file.csv', 'utf8'); // blocks
const rows = data.split('\n').map(row => row.split(','));
// Event loop is frozen for seconds — all other requests queue up
```

```javascript
// Diagnose with setInterval heartbeat
const start = Date.now();
setInterval(() => {
  const elapsed = Date.now() - start;
  console.log(`Event loop delay: ${elapsed - 1000}ms`);
}, 1000);
// If the delay grows beyond ~50ms, you have starvation
```

> **Trap:** Engineers often underestimate blocking time. A `JSON.parse` on a 50MB payload blocks for hundreds of milliseconds. A single sync `crypto.createHash` on a large buffer does too. Measure before you dismiss.

```javascript
// NEVER do this in a request handler
app.post('/api/import', (req, res) => {
  const data = JSON.parse(req.body); // blocks event loop if body is large
  const hash = crypto.createHash('sha256').update(JSON.stringify(data)).digest('hex'); // blocks
  res.json({ hash });
});
```

```javascript
// Instead: offload parsing and hashing
app.post('/api/import', async (req, res) => {
  const data = req.body;
  const hash = await new Promise((resolve, reject) => {
    const hash = crypto.createHash('sha256');
    hash.update(JSON.stringify(data));
    hash.end();
    resolve(hash.digest('hex'));
  });
  res.json({ hash });
});
```

> **Follow-up:** *What specifically blocks the event loop?* Synchronous operations: `JSON.parse`/`JSON.stringify` on large objects, `crypto.hash` (sync), `fs.readFileSync`, `Array.sort` on huge arrays, `String.match` with catastrophic backtracking, `Math` heavy loops, template literal concatenation of giant strings, and any tight `for` loop without yielding.

### Offloading CPU Work to Worker Threads

```javascript
// worker.js
const { parentPort } = require('worker_threads');

parentPort.on('message', (data) => {
  const result = heavyComputation(data);
  parentPort.postMessage(result);
});

function heavyComputation(input) {
  let result = 0;
  for (let i = 0; i < 1_000_000_000; i++) {
    result += Math.sqrt(input * i);
  }
  return result;
}
```

```javascript
// main.js
const { Worker } = require('worker_threads');

function runWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js');
    worker.postMessage(data);
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}

app.post('/api/compute', async (req, res) => {
  const result = await runWorker(req.body);
  res.json({ result });
});
```

### Concurrency Control with `p-limit`, `p-map`, `p-queue`

```javascript
const pLimit = require('p-limit');

const limit = pLimit(10);

const inputs = [...Array(1000).keys()];
const results = await Promise.all(
  inputs.map(input => limit(() => processItem(input)))
);
```

```javascript
const pMap = require('p-map');

const results = await pMap(
  inputs,
  async (input) => {
    const result = await fetch(`/api/process/${input}`);
    return result.json();
  },
  { concurrency: 10 }
);
```

```javascript
const { default: PQueue } = await import('p-queue');

const queue = new PQueue({ concurrency: 5, interval: 1000, intervalCap: 10 });

queue.on('active', () => {
  console.log(`Working... Queue size: ${queue.size} | Pending: ${queue.pending}`);
});

const results = await Promise.all(
  jobs.map(job => queue.add(() => processJob(job)))
);
```

> **Trap:** `Promise.all` with a batch of 1000 concurrent API calls will hammer your target and likely cause socket exhaustion, connection timeouts, or rate limiting. Always cap concurrency.

```javascript
// WRONG — 1000 concurrent fetches
const results = await Promise.all(
  items.map(item => fetch(`/api/items/${item.id}`))
);

// RIGHT — limit to 10 at a time
const limit = pLimit(10);
const results = await Promise.all(
  items.map(item => limit(() => fetch(`/api/items/${item.id}`)))
);
```

### Async Iterators and Async Generators

```javascript
// Async generator — paginate through an API
async function* paginate(url, pageSize = 100) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${url}?page=${page}&limit=${pageSize}`);
    const data = await response.json();
    yield data.items;
    hasMore = data.hasMore;
    page++;
  }
}

// Usage
for await (const items of paginate('/api/items', 50)) {
  for (const item of items) {
    await processItem(item);
  }
}
```

```javascript
// Async iterator from a stream
const { createReadStream } = require('fs');
const { createInterface } = require('readline');

async function* readLines(filePath) {
  const rl = createInterface({
    input: createReadStream(filePath),
    crlfDelay: Infinity,
  });

  for await (const line of rl) {
    yield line;
  }
}

// Process line by line without loading entire file
for await (const line of readLines('huge-file.csv')) {
  await processLine(line);
}
```

### Sequential vs Parallel Execution — When Each Matters

```javascript
// SEQUENTIAL — required when each step depends on the previous
async function processDependent(items) {
  const results = [];
  for (const item of items) {
    const step1 = await transformA(item);
    const step2 = await transformB(step1);
    results.push(step2);
  }
  return results;
}
```

```javascript
// PARALLEL — independent operations, fan-out
async function processIndependent(items) {
  const results = await Promise.all(
    items.map(async (item) => {
      const [a, b] = await Promise.all([
        fetchExternalA(item),
        fetchExternalB(item),
      ]);
      return merge(a, b);
    })
  );
  return results;
}
```

```javascript
// SEQUENTIAL for rate-limited / resource-constrained operations
async function processRateLimited(jobs) {
  for (const job of jobs) {
    await submitToExternalAPI(job);
  }
}
```

```javascript
// HYBRID — sequential batches of parallel work
async function processBatched(items, batchSize = 10) {
  const results = [];
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await Promise.all(batch.map(item => processItem(item)));
    results.push(...batchResults);
  }
  return results;
}
```

> **Trap:** `Promise.all` with a single rejecter failing all. If you have 10 independent operations and one fails, `Promise.all` rejects immediately — the other 9 results are lost. Use `Promise.allSettled` when you need all results regardless of failure.

```javascript
const results = await Promise.allSettled([
  sendEmail(user.email, message),
  sendSMS(user.phone, message),
  sendPush(user.deviceToken, message),
]);

const failures = results.filter(r => r.status === 'rejected');
if (failures.length > 0) {
  console.error('Some notification channels failed:', failures);
}
```

### Avoid `util.promisify` in Favor of Native Promises

```javascript
// OLD — util.promisify
const { promisify } = require('util');
const readFile = promisify(require('fs').readFile);
const data = await readFile('/path/to/file', 'utf8');

// MODERN — native fs.promises
const fsp = require('fs').promises;
const data = await fsp.readFile('/path/to/file', 'utf8');
```

---

## 2. Event Loop Diagnosis

### Tools

**clinic.js** — The Node.js diagnostics toolkit.

```bash
npm install -g clinic

# Doctor — high-level health check
clinic doctor -- node server.js

# Bubbleprof — async flow visualization
clinic bubbleprof -- node server.js

# Flame — CPU flamegraph
clinic flame -- node server.js
```

```bash
clinic doctor -- autocannon -c 100 -d 30 http://localhost:3000/api/items
```

**0x** — Flamegraph generator.

```bash
npx 0x server.js
```

**`--inspect` + Chrome DevTools**

```bash
node --inspect server.js
# Open chrome://inspect → click "Open dedicated DevTools for Node"
```

### Identifying Long Tasks

```javascript
const { PerformanceObserver, performance } = require('perf_hooks');

const obs = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  for (const entry of entries) {
    if (entry.duration > 50) {
      console.warn(`Long task detected: ${entry.name} took ${entry.duration}ms`);
    }
  }
});

obs.observe({ entryTypes: ['function'] });
```

```javascript
let lastCheck = process.hrtime.bigint();
setInterval(() => {
  const now = process.hrtime.bigint();
  const elapsed = Number(now - lastCheck) / 1_000_000;
  if (elapsed > 100) {
    console.error(`Event loop blocked for ${elapsed.toFixed(2)}ms`);
  }
  lastCheck = now;
}, 100);
```

> **Trap:** Do not optimize without measuring first. Trusting intuition over data is the #1 performance mistake. Engineers often optimize database queries when the real bottleneck is JSON serialization or GC pressure.

### `process.hrtime.bigint()` for Precise Measurement

```javascript
const start = process.hrtime.bigint();
JSON.parse(largePayload);
const end = process.hrtime.bigint();
const durationMs = Number(end - start) / 1_000_000;
console.log(`JSON.parse took ${durationMs.toFixed(2)}ms`);
```

### Common Blocking Patterns

```javascript
// 1. JSON.parse on huge payloads
const payload = fs.readFileSync('large.json', 'utf8');
const parsed = JSON.parse(payload);

// Fix: stream-based JSON parser
const { createReadStream } = require('fs');
const { parse } = require('JSONStream');
createReadStream('large.json').pipe(parse('*')).on('data', (item) => {
  processItem(item);
});
```

```javascript
// 2. Sync crypto
const crypto = require('crypto');
const hash = crypto.createHash('sha256').update(data).digest('hex');

// Fix: use async crypto
const hash = await new Promise((resolve, reject) => {
  const hash = crypto.createHash('sha256');
  hash.on('readable', () => resolve(hash.read().toString('hex')));
  hash.write(data);
  hash.end();
});
```

```javascript
// 3. fs.readFileSync
app.get('/api/config', (req, res) => {
  const config = fs.readFileSync('/etc/app/config.json', 'utf8');
  res.json(JSON.parse(config));
});

// Fix: use async readFile
app.get('/api/config', async (req, res) => {
  const config = await fs.promises.readFile('/etc/app/config.json', 'utf8');
  res.json(JSON.parse(config));
});
```

```javascript
// 4. Large array operations
const huge = new Array(10_000_000).fill(0);
const sorted = huge.sort();
```

> **Follow-up:** *How do you measure event loop lag in production?* Use a high-resolution timer that compares elapsed wall clock time to the expected interval duration. Export event loop lag as a Prometheus metric via `event-loop-lag` or a custom gauge.

```javascript
const client = require('prom-client');
const eventLoopLag = new client.Gauge({
  name: 'nodejs_event_loop_lag_seconds',
  help: 'Event loop lag in seconds',
});

const start = process.hrtime.bigint();
setInterval(() => {
  const now = process.hrtime.bigint();
  const lag = Number(now - start) / 1_000_000_000 - 1;
  eventLoopLag.set(lag);
}, 1000);
```

---

## 3. Worker Threads

The `worker_threads` module enables running JavaScript in parallel on separate OS threads. Each worker has its own V8 isolate, event loop, and heap.

### Basic Worker Thread Pattern

```javascript
// worker.js
const { parentPort } = require('worker_threads');

parentPort.on('message', (task) => {
  try {
    const result = processTask(task);
    parentPort.postMessage({ type: 'result', id: task.id, data: result });
  } catch (error) {
    parentPort.postMessage({ type: 'error', id: task.id, error: error.message });
  }
});
```

```javascript
// main.js
const { Worker } = require('worker_threads');
const path = require('path');

function createWorker() {
  const worker = new Worker(path.join(__dirname, 'worker.js'));

  worker.on('message', (msg) => {
    if (msg.type === 'result') {
      console.log(`Task ${msg.id} completed:`, msg.data);
    } else if (msg.type === 'error') {
      console.error(`Task ${msg.id} failed:`, msg.error);
    }
  });

  worker.on('error', (err) => console.error('Worker error:', err));
  worker.on('exit', (code) => {
    if (code !== 0) console.error(`Worker exited with code ${code}`);
  });

  return worker;
}

const worker = createWorker();
worker.postMessage({ id: 1, input: 'data' });
```

### SharedArrayBuffer and Atomics

```javascript
const { Worker } = require('worker_threads');

const buffer = new SharedArrayBuffer(4 * 1024);
const view = new Int32Array(buffer);

const worker = new Worker('./shared-worker.js');
worker.postMessage(buffer);

Atomics.add(view, 0, 1);
Atomics.store(view, 1, Date.now());
```

```javascript
// shared-worker.js
const { parentPort } = require('worker_threads');

parentPort.on('message', (buffer) => {
  const view = new Int32Array(buffer);
  Atomics.wait(view, 2, 0);
  const counter = Atomics.load(view, 0);
  const timestamp = Atomics.load(view, 1);
  parentPort.postMessage({ counter, timestamp });
});
```

> **Trap:** Mutating shared state without `Atomics` leads to race conditions.

```javascript
// WRONG — race condition
view[0] = view[0] + 1; // not atomic

// RIGHT — atomic increment
Atomics.add(view, 0, 1);
```

### Worker Pool Pattern

```javascript
// worker-pool.js
const { Worker } = require('worker_threads');
const { EventEmitter } = require('events');

class WorkerPool extends EventEmitter {
  constructor(workerPath, numThreads = 4) {
    super();
    this.workers = [];
    this.active = new Map();
    this.queue = [];
    this.taskId = 0;

    for (let i = 0; i < numThreads; i++) {
      const worker = new Worker(workerPath);
      worker.on('message', (msg) => this._handleResult(worker, msg));
      worker.on('error', (err) => this._handleError(worker, err));
      worker.on('exit', (code) => {
        if (code !== 0 && !worker._terminated) {
          console.error(`Worker exited with code ${code}, replacing...`);
          this._replaceWorker(worker);
        }
      });
      this.workers.push(worker);
    }
  }

  runTask(task) {
    return new Promise((resolve, reject) => {
      const id = ++this.taskId;
      const entry = { id, task, resolve, reject };
      const worker = this._getIdleWorker();

      if (worker) {
        this.active.set(worker, entry);
        worker.postMessage({ id, ...task });
      } else {
        this.queue.push(entry);
      }
    });
  }

  _getIdleWorker() {
    return this.workers.find(w => !this.active.has(w)) || null;
  }

  _handleResult(worker, msg) {
    const entry = this.active.get(worker);
    if (!entry) return;
    this.active.delete(worker);
    if (msg.error) entry.reject(new Error(msg.error));
    else entry.resolve(msg.data);
    this._processQueue();
  }

  _handleError(worker, err) {
    const entry = this.active.get(worker);
    if (entry) { this.active.delete(worker); entry.reject(err); }
    this._replaceWorker(worker);
  }

  _processQueue() {
    while (this.queue.length > 0) {
      const worker = this._getIdleWorker();
      if (!worker) break;
      const entry = this.queue.shift();
      this.active.set(worker, entry);
      worker.postMessage({ id: entry.id, ...entry.task });
    }
  }

  async close() {
    for (const worker of this.workers) {
      worker._terminated = true;
      worker.terminate();
    }
    this.workers = [];
    this.active.clear();
    this.queue = [];
  }
}

module.exports = WorkerPool;
```

### When to Use Worker Threads vs Child Processes

| Criterion | Worker Threads | Child Processes |
|-----------|---------------|-----------------|
| Shared memory | `SharedArrayBuffer` + `Atomics` | None (IPC only) |
| Memory overhead | Lower (same process) | Higher (separate process) |
| Startup time | Faster | Slower |
| Isolation | Same process | Separate process |
| Use case | CPU-bound, shared memory | Non-JS tools, strong isolation |

> **Trap:** Worker Threads add overhead. For trivial work that takes <1ms, spawning a worker costs more than the work itself. Always measure first.

> **Follow-up:** *Can Worker Threads solve I/O problems?* No. I/O is already non-blocking in Node.js. Workers are for CPU-bound work only.

---

## 4. Child Processes

### `spawn` vs `exec` vs `fork` vs `execFile`

```javascript
const { spawn, exec, fork, execFile } = require('child_process');

// spawn — streams output, no shell by default
const ls = spawn('ls', ['-lh', '/tmp'], { stdio: ['pipe', 'pipe', 'pipe'] });
ls.stdout.on('data', (data) => console.log(`stdout: ${data}`));
ls.stderr.on('data', (data) => console.error(`stderr: ${data}`));
ls.on('close', (code) => console.log(`child process exited with code ${code}`));

// exec — shell command, buffers output (max 200KB default)
exec('ls -lh /tmp', { maxBuffer: 1024 * 1024 }, (error, stdout, stderr) => {
  if (error) { console.error(`exec error: ${error}`); return; }
  console.log(`stdout: ${stdout}`);
});

// execFile — executes directly without shell
execFile('convert', ['input.png', '-resize', '200x200', 'output.png'], (error, stdout, stderr) => {
  if (error) throw error;
});

// fork — spawns new Node.js process with IPC channel
const child = fork('./worker.js', ['arg1', 'arg2'], { stdio: ['pipe', 'pipe', 'pipe', 'ipc'] });
child.on('message', (msg) => console.log('Message from child:', msg));
child.send({ hello: 'world' });
```

### Error Handling: Exit Codes, Stderr, Timeouts

```javascript
const { spawn } = require('child_process');

function runCommand(command, args, options = {}) {
  return new Promise((resolve, reject) => {
    const child = spawn(command, args, {
      stdio: ['pipe', 'pipe', 'pipe'],
      timeout: options.timeout || 30000,
      ...options,
    });

    let stdout = '';
    let stderr = '';

    child.stdout.on('data', (data) => { stdout += data; });
    child.stderr.on('data', (data) => { stderr += data; });

    child.on('close', (code, signal) => {
      if (code === 0) resolve(stdout);
      else if (signal === 'SIGTERM') reject(new Error(`Command timed out after ${options.timeout}ms`));
      else {
        const error = new Error(`Command failed with exit code ${code}`);
        error.code = code;
        error.stderr = stderr;
        error.stdout = stdout;
        reject(error);
      }
    });

    child.on('error', (err) => reject(new Error(`Failed to spawn: ${err.message}`)));
  });
}
```

> **Trap:** Shell injection with `exec` — never pass user input to `exec`. Use `spawn` with array args.

```javascript
// WRONG — shell injection
exec(`cat ${req.params.filename}`, (err, stdout) => { /* ... */ });

// RIGHT — spawn with array args (no shell)
spawn('cat', [sanitizedFilename]);
```

> **Trap:** Buffer overflow with `exec`. Default `maxBuffer` is 200KB. Use `spawn` for large output.

> **Trap:** Zombie processes. Always listen for `'close'` event on child processes.

> **Follow-up:** *Difference between 'exit' and 'close'?* `'exit'` fires when the process exits. `'close'` fires when all stdio streams are closed. Always use `'close'` to ensure all output is consumed.

---

## 5. Memory Management & Leaks

### V8 Garbage Collection

V8's heap is divided into generations: Young Generation (Nursery, Intermediate) promoted to Old Generation via Mark-Sweep / Mark-Compact.

```javascript
const { PerformanceObserver } = require('perf_hooks');

const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.detail?.kind === 'gc') {
      const kind = ['', 'Scavenge', 'Mark-Sweep', '', 'Incremental', '', '', '', 'Weak'][entry.detail.kind] || 'Unknown';
      console.log(`GC ${kind} took ${entry.duration.toFixed(2)}ms`);
    }
  }
});
obs.observe({ entryTypes: ['gc'] });
```

### Memory Leak Patterns

```javascript
// 1. Event listeners not removed
class DataService {
  startPolling() {
    this.interval = setInterval(async () => {
      const data = await fetch('/api/data');
      this.cache.set(Date.now(), data);
    }, 1000);
  }
  // Missing stop() — never clears interval
}
```

```javascript
// 2. Closures capturing large objects
function createHandler(data) {
  const hugeData = data;
  return function handler() {
    console.log(hugeData.id); // entire hugeData retained
  };
}
// Fix: only close over what you need
function createHandler(data) {
  const { id } = data;
  return function handler() { console.log(id); };
}
```

```javascript
// 3. Caches without bounds
class InMemoryCache {
  constructor() { this.store = new Map(); }
  set(key, value) { this.store.set(key, value); }
  get(key) { return this.store.get(key); }
}
// Fix: use lru-cache with max size and TTL
```

```javascript
// 4. Accumulating EventEmitter listeners
class OrderProcessor extends EventEmitter {
  processOrder(order) {
    setImmediate(() => { this.emit('processed', order); });
  }
}
// Each call adds a listener — fix: use once() or removeListener
```

```javascript
// 5. Timers/intervals not cleared
class MetricsCollector {
  start() {
    this.timer = setInterval(() => { this.collect(); }, 1000);
  }
  // No stop() — interval keeps the collector alive
}
```

### Heap Snapshot Analysis

```bash
# Generate heap snapshot programmatically
node -e "
const v8 = require('v8');
const fs = require('fs');
const snapshot = v8.getHeapSnapshot();
const file = fs.createWriteStream('heap.heapsnapshot');
snapshot.pipe(file);
"
```

```javascript
// Compare heap usage
async function detectLeak(fn, iterations = 100) {
  const before = process.memoryUsage().heapUsed;
  for (let i = 0; i < iterations; i++) await fn();
  global.gc();
  const after = process.memoryUsage().heapUsed;
  const diff = after - before;
  console.log(`Memory delta: ${(diff / 1024 / 1024).toFixed(2)}MB`);
  if (diff > 10 * 1024 * 1024) console.warn('Potential leak detected');
}
```

> **Trap:** Relying on `--max-old-space-size` as a fix instead of finding the leak. It's a band-aid that delays the OOM.

```bash
# BAND-AID — delays the crash
node --max-old-space-size=8192 server.js

# Instead: profile the heap
node --inspect server.js
```

> **Follow-up:** *How do you identify what's retaining memory in a heap snapshot?* Load the `.heapsnapshot` in Chrome DevTools. Use the Comparison view to diff two snapshots. Look for objects with large Retained Size, closure scopes holding large objects, and event listeners that were never removed.

---

## 6. Performance Profiling

### CPU Profiling

```bash
# Chrome DevTools
node --inspect server.js
# Open chrome://inspect → Profiler → Start

# Clinic Flame
clinic flame -- node server.js

# 0x
npx 0x server.js
```

### Memory Profiling

```javascript
// Allocation sampling
const { SamplingHeapProfiler } = require('v8');
const profiler = new SamplingHeapProfiler();
profiler.start(1024 * 16);
// ... do work ...
const profile = profiler.stop();
for (const sample of profile.samples) {
  console.log(`${sample.size} bytes allocated by:`, sample.stack.join(' <- '));
}
```

### Latency Profiling

```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

const latency = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.name.startsWith('http.')) {
      console.log(`${entry.name}: ${entry.duration.toFixed(2)}ms`);
    }
  }
});
latency.observe({ entryTypes: ['measure'] });

app.use((req, res, next) => {
  performance.mark(`http.${req.path}.start`);
  res.on('finish', () => {
    performance.mark(`http.${req.path}.end`);
    performance.measure(`http.${req.method}:${req.path}`, `http.${req.path}.start`, `http.${req.path}.end`);
  });
  next();
});
```

### Flame Graph Interpretation

```
Wide bars = hot functions (high CPU time)
Tall stacks = deep call chains
Red/orange: JavaScript, Yellow: C++ bindings, Green: V8 runtime
```

### Load Testing Tools

```bash
npx autocannon -c 100 -d 30 -p 10 http://localhost:3000/api/items
wrk -t12 -c400 -d30s http://localhost:3000/api/items
npx artillery quick --count 100 --num 50 http://localhost:3000/api/items
```

> **Trap:** Profiling in isolation without realistic load data. Always profile under concurrent traffic patterns.

### Finding Bottlenecks

```
1. Event loop lag → blocking operations
2. CPU profiling → hot functions
3. Memory profiling → allocation pressure
4. Database profiling → slow queries, N+1
5. Network profiling → latency between services
6. GC profiling → if >10% CPU time, reduce allocations
```

> **Follow-up:** *What's the most common bottleneck?* Database queries (N+1, missing indexes) and JSON serialization of large payloads. Always profile the full request lifecycle before optimizing any single piece.

---

## 7. Cluster Mode & PM2

### Cluster Module

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  for (let i = 0; i < numCPUs; i++) cluster.fork();

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    cluster.fork();
  });
} else {
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end('Hello from worker ' + process.pid);
  }).listen(8000);
}
```

### Sticky Sessions (Needed for WebSockets)

Without sticky sessions, WebSocket connections route to different workers.

```javascript
const sticky = require('sticky-session');
const http = require('http');
const { WebSocketServer } = require('ws');

if (!sticky.listen(server, 8000)) {
  // Master
} else {
  // Worker
  const server = http.createServer((req, res) => { res.end('Worker: ' + process.pid); });
  const wss = new WebSocketServer({ server });
}
```

### PM2

```bash
npm install -g pm2
pm2 start server.js -i max
pm2 list
pm2 logs
pm2 monit
pm2 reload all          # zero-downtime reload
pm2 restart app         # hard restart (downtime)
```

### ecosystem.config.js

```javascript
module.exports = {
  apps: [{
    name: 'api-server',
    script: './server.js',
    instances: 'max',
    exec_mode: 'cluster',
    kill_timeout: 10000,
    listen_timeout: 3000,
    wait_ready: true,
    error_file: '/var/log/app/error.log',
    out_file: '/var/log/app/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,
    max_restarts: 10,
    restart_delay: 4000,
    min_uptime: 10000,
    max_memory_restart: '500M',
  }],
};
```

### Graceful Shutdown with PM2

```javascript
const server = http.createServer((req, res) => { res.end('Hello'); });

server.listen(3000, () => {
  if (process.send) process.send('ready');
});

process.on('SIGINT', () => {
  server.close(() => { process.exit(0); });
});

process.on('SIGTERM', () => {
  server.close(() => { process.exit(0); });
});
```

### PM2 in Docker

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["pm2-runtime", "start", "ecosystem.config.js", "--env", "production"]
```

### Docker Orchestration vs Cluster Mode

In K8s: run ONE process per container, let K8s scale. Do NOT use cluster mode inside a K8s pod.

> **Trap:** Not draining connections before restart. PM2 sends SIGKILL after `kill_timeout`. If your app doesn't handle SIGTERM, active users get disconnected.

> **Follow-up:** *Can you use cluster mode with WebSockets?* Yes, but you need sticky sessions so all messages from the same client go to the same worker.

---

## 8. Graceful Shutdown

### SIGTERM / SIGINT Handling

```javascript
const http = require('http');
const { Pool } = require('pg');

const pool = new Pool({ /* config */ });

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    if (server.closeCalled) {
      res.writeHead(503);
      res.end('Draining');
      return;
    }
    res.writeHead(200);
    res.end('OK');
    return;
  }
  setTimeout(() => { res.end('Done'); }, 5000);
});

function gracefulShutdown(signal) {
  console.log(`${signal} received — starting graceful shutdown`);
  server.closeCalled = true;

  server.close(async (err) => {
    if (err) { console.error('Error closing server:', err); process.exit(1); }
    try {
      await pool.end();
      console.log('Database connections closed');
    } catch (err) { console.error('Error closing database:', err); }
    console.log('Graceful shutdown complete');
    process.exit(0);
  });

  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
server.listen(3000);
```

### Draining Keep-Alive Connections

```javascript
const server = http.createServer((req, res) => { res.end('OK'); });
server.keepAliveTimeout = 5000;

function gracefulShutdown() {
  server.close(() => {
    console.log('All connections closed');
    process.exit(0);
  });
  server.closeIdleConnections();
  setTimeout(() => { console.error('Forcing shutdown'); process.exit(1); }, 10000).unref();
}
```

### Full Graceful Shutdown with Express and Terminus

```javascript
const express = require('express');
const { createTerminus } = require('@godaddy/terminus');

const app = express();
app.get('/health', (req, res) => { res.json({ status: 'ok' }); });

const server = http.createServer(app);

createTerminus(server, {
  signal: 'SIGTERM',
  healthChecks: { '/health': async () => { await db.query('SELECT 1'); } },
  onSignal: async () => {
    await db.end();
    await redis.quit();
    await queue.close();
  },
  timeout: 30000,
  logger: console.error,
});

server.listen(3000);
```

### K8s preStop Hook

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5 && kill -SIGTERM 1"]
terminationGracePeriodSeconds: 35
```

> **Trap:** Not handling `server.close` callback. If you don't wait for it, the process exits with active connections still being served.

```javascript
// WRONG
process.on('SIGTERM', () => { server.close(); process.exit(0); });

// RIGHT
process.on('SIGTERM', () => { server.close(() => { process.exit(0); }); });
```

> **Trap:** Killing workers before they finish in-flight requests. With PM2, set `kill_timeout` high enough. With K8s, set `terminationGracePeriodSeconds` to at least your longest request timeout.

> **Follow-up:** *Why use a preStop hook in K8s?* K8s removes the pod from the Service endpoint list asynchronously. The preStop hook gives a delay to ensure endpoint propagation before SIGTERM, preventing connection refused errors.

---

## 9. Microservices with Node.js

### Service Decomposition Patterns

```
By Domain: Inventory Service, Order Service, Payment Service, Notification Service
By Subdomain: Catalog, Fulfillment, Billing

Each service owns its data — no shared databases.
```

### Communication Patterns

```javascript
// HTTP/REST
const response = await fetch('http://inventory-service/api/items');

// gRPC — binary, strongly typed
const client = new InventoryService('inventory-service:50051', grpc.credentials.createInsecure());
const response = await client.listItems({ category: 'widgets' });

// Message Queues — async, decoupled
await queue.add('order.created', { orderId: '123' });
```

### API Gateway Pattern

```javascript
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();
app.use('/api/inventory', createProxyMiddleware({ target: 'http://inventory-service:3001', changeOrigin: true }));
app.use('/api/orders', createProxyMiddleware({ target: 'http://order-service:3002', changeOrigin: true }));
app.use('/api/users', createProxyMiddleware({ target: 'http://user-service:3003', changeOrigin: true }));
app.listen(3000);
```

### Service Discovery

```javascript
// Kubernetes DNS — built-in
const response = await fetch('http://inventory-service.default.svc.cluster.local:3001/api/items');

// Consul
const Consul = require('consul');
const consul = new Consul({ host: 'consul.service.consul' });

async function discoverService(serviceName) {
  const services = await consul.agent.service.list();
  const service = Object.values(services).find(s => s.Service === serviceName);
  if (!service) throw new Error(`Service ${serviceName} not found`);
  return `http://${service.Address}:${service.Port}`;
}
```

### Distributed Tracing (OpenTelemetry)

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { Resource } = require('@opentelemetry/resources');

const sdk = new NodeSDK({
  resource: new Resource({ 'service.name': 'inventory-service', 'service.version': '1.0.0' }),
  traceExporter: new OTLPTraceExporter({ url: 'http://jaeger-collector:4318/v1/traces' }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

```javascript
// Manual spans
const { trace } = require('@opentelemetry/api');
const tracer = trace.getTracer('inventory-service');

async function getItem(id) {
  return await tracer.startActiveSpan('getItem', async (span) => {
    span.setAttribute('item.id', id);
    try {
      const item = await db.query('SELECT * FROM items WHERE id = $1', [id]);
      return item;
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      throw err;
    } finally { span.end(); }
  });
}
```

### Circuit Breaker (Opossum)

```javascript
const CircuitBreaker = require('opossum');

const breaker = new CircuitBreaker(async (itemId) => {
  const response = await fetch(`http://inventory-service/check/${itemId}`);
  if (!response.ok) throw new Error('Inventory service unavailable');
  return response.json();
}, {
  timeout: 3000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000,
  volumeThreshold: 5,
});

breaker.on('open', () => console.log('Circuit breaker opened'));
breaker.on('halfOpen', () => console.log('Circuit breaker half-open'));
breaker.on('close', () => console.log('Circuit breaker closed'));

async function checkAvailability(itemId) {
  try {
    return await breaker.fire(itemId);
  } catch (err) {
    return { available: false, reason: 'service_unavailable' };
  }
}
```

### Retry with Backoff

```javascript
async function fetchWithRetry(url, options = {}) {
  const maxRetries = options.retries || 3;
  const baseDelay = options.baseDelay || 1000;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url);
      if (!response.ok && response.status >= 500) throw new Error(`HTTP ${response.status}`);
      return await response.json();
    } catch (err) {
      if (attempt === maxRetries) throw err;
      const delay = baseDelay * Math.pow(2, attempt - 1) + Math.random() * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### Saga Pattern

```javascript
// Orchestration-based saga
class CreateOrderSaga {
  async execute(orderData) {
    try {
      const order = await orderService.createOrder(orderData);
      await inventoryService.reserveStock(order.items);
      await paymentService.processPayment(order.total);
      await notificationService.sendConfirmation(order);
      await orderService.markComplete(order.id);
    } catch (err) {
      await orderService.markFailed(orderData.id);
      await inventoryService.releaseStock(orderData.items);
      await paymentService.refund(orderData.total);
      throw err;
    }
  }
}
```

> **Trap:** Synchronous calls between services create a distributed monolith. Use async communication for non-critical paths, set aggressive timeouts, and always have fallbacks.

```javascript
// WRONG — synchronous chain
const order = await orderService.getOrder(id);
const user = await userService.getUser(order.userId); // if down, request fails
const payment = await paymentService.getPayment(order.paymentId);

// BETTER — parallel with fallbacks
const [user, payment] = await Promise.allSettled([
  userService.getUser(order.userId).catch(() => null),
  paymentService.getPayment(order.paymentId).catch(() => null),
]);
```

> **Follow-up:** *How do you handle partial failure?* Circuit breakers, timeouts, retries with backoff, fallbacks, graceful degradation. Return partial results rather than failing entirely.

---

## 10. Message Queues (Bull/BullMQ)

Bull and BullMQ are Redis-backed job/task queues. BullMQ is the modern version.

### Job Structure

```javascript
const { Queue } = require('bullmq');

const emailQueue = new Queue('email', {
  connection: { host: 'localhost', port: 6379 },
});

await emailQueue.add('send-welcome', {
  userId: 123,
  email: 'user@example.com',
  template: 'welcome',
}, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 2000 },
  removeOnComplete: { age: 3600, count: 1000 },
  removeOnFail: { age: 86400 },
  priority: 10,
  delay: 5000,
});
```

### Queue Lifecycle

```
Job added → Wait → Active → Completed / Failed
                ↻ stalled (re-processed if worker crashes)
```

### Producer and Consumer

```javascript
// Producer
const reportQueue = new Queue('report-generation', {
  connection: { host: process.env.REDIS_HOST, port: 6379 },
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    removeOnComplete: { age: 3600 },
    removeOnFail: { age: 86400 * 7 },
  },
});

app.post('/api/reports/generate', async (req, res) => {
  const job = await reportQueue.add('generate', {
    reportId: uuid(),
    userId: req.user.id,
    filters: req.body.filters,
  });
  res.json({ jobId: job.id });
});

// Consumer
const { Worker } = require('bullmq');

const worker = new Worker('report-generation', async (job) => {
  const { reportId, userId, filters } = job.data;
  job.updateProgress(10);
  const data = await fetchData(filters);
  job.updateProgress(50);
  const report = await generateReport(data);
  job.updateProgress(100);
  return { reportId, url: `/reports/${reportId}` };
}, {
  connection: { host: process.env.REDIS_HOST, port: 6379 },
  concurrency: 5,
  lockDuration: 30000,
});

worker.on('completed', (job, returnvalue) => {
  console.log(`Job ${job.id} completed:`, returnvalue);
});
worker.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err.message);
});
```

### Concurrency and Rate Limiting

```javascript
// Per-queue concurrency
const worker = new Worker('email', processor, { concurrency: 20 });

// Rate limiting
const queue = new Queue('api-calls', {
  limiter: { max: 100, duration: 1000 },
});
```

### Repeatable Jobs

```javascript
const { QueueScheduler } = require('bullmq');
const scheduler = new QueueScheduler('reports', { connection: { host: 'localhost', port: 6379 } });

await reportQueue.add('daily-report', { type: 'daily' }, {
  repeat: { pattern: '0 6 * * *', tz: 'America/New_York' },
});
```

### Job Dependencies

```javascript
const jobA = await queue.add('step-a', { data: 'a' });
const jobB = await queue.add('step-b', { data: 'b' }, { dependencies: [jobA.id] });
```

### Graceful Shutdown

```javascript
async function shutdown(queue, worker, scheduler) {
  if (scheduler) await scheduler.close();
  if (worker) await worker.close();
  if (queue) await queue.close();
}

process.on('SIGTERM', async () => {
  await shutdown(reportQueue, reportWorker, reportScheduler);
  process.exit(0);
});
```

### Error Handling

```javascript
const worker = new Worker('orders', async (job) => {
  try {
    await processOrder(job.data.orderId);
  } catch (err) {
    if (err.code === 'ETIMEDOUT' || err.code === 'ECONNRESET') throw err; // retry
    if (err.code === 'VALIDATION_ERROR') {
      await job.discard();
      await deadLetterQueue.add('order-failed', job.data, { attempts: 1 });
      return;
    }
    throw err;
  }
}, { stalledInterval: 30000, maxStalledCount: 2 });
```

```javascript
// Observability
const { QueueEvents } = require('bullmq');
const queueEvents = new QueueEvents('orders', { connection: { host: 'localhost', port: 6379 } });

queueEvents.on('completed', ({ jobId, returnvalue }) => { metrics.recordJobSuccess(jobId); });
queueEvents.on('failed', ({ jobId, failedReason }) => { metrics.recordJobFailure(jobId); });
queueEvents.on('progress', ({ jobId, data }) => { console.log(`Job ${jobId}: ${data}%`); });
```

> **Trap:** Jobs that never fail but never complete (stalled). Set `stalledInterval` appropriately and ensure workers update progress for long jobs to extend the lock.

> **Trap:** `removeOnComplete` not set causes Redis memory bloat. Always set it with a reasonable age/count.

> **Trap:** Not handling job data corruption. Always validate job data with Zod at the start of the processor.

```javascript
const EmailJobSchema = z.object({
  to: z.string().email(),
  subject: z.string().min(1).max(200),
  body: z.string(),
});

const worker = new Worker('email', async (job) => {
  const parsed = EmailJobSchema.safeParse(job.data);
  if (!parsed.success) { await job.discard(); return; }
  await sendEmail(parsed.data);
});
```

---

## 11. Caching Strategies

### Redis Caching Patterns

```javascript
// Cache-Aside (Lazy Loading)
async function getItem(id) {
  const cacheKey = `item:${id}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const item = await db.query('SELECT * FROM items WHERE id = $1', [id]);
  if (item) await redis.set(cacheKey, JSON.stringify(item), 'EX', 3600);
  return item;
}
```

```javascript
// Write-Through
async function updateItem(id, data) {
  const item = await db.query('UPDATE items SET name = $1, price = $2 WHERE id = $3 RETURNING *', [data.name, data.price, id]);
  await redis.set(`item:${id}`, JSON.stringify(item), 'EX', 3600);
  return item;
}
```

```javascript
// Write-Behind
class WriteBehindCache {
  constructor(redis, db) {
    this.redis = redis;
    this.db = db;
    this.queue = [];
    this.flushInterval = setInterval(() => this.flush(), 5000);
  }

  async set(key, value) {
    await this.redis.set(key, JSON.stringify(value), 'EX', 3600);
    this.queue.push({ key, value });
  }

  async flush() {
    const batch = this.queue.splice(0);
    if (batch.length === 0) return;
    // Batch write to database
  }

  async close() { clearInterval(this.flushInterval); await this.flush(); }
}
```

### Multi-Level Caching (L1 + L2)

```javascript
const L1 = new Map(); // or lru-cache

async function getItem(id) {
  const key = `item:${id}`;
  const l1 = L1.get(key);
  if (l1 && l1.expiry > Date.now()) return l1.data;

  const l2 = await redis.get(key);
  if (l2) {
    const parsed = JSON.parse(l2);
    L1.set(key, { data: parsed, expiry: Date.now() + 60000 });
    return parsed;
  }

  const item = await db.query('SELECT * FROM items WHERE id = $1', [id]);
  if (item) {
    await redis.set(key, JSON.stringify(item), 'EX', 3600);
    L1.set(key, { data: item, expiry: Date.now() + 60000 });
  }
  return item;
}
```

### Cache Invalidation

```javascript
// TTL
await redis.set('item:123', JSON.stringify(item), 'EX', 3600);

// Explicit invalidation on write
async function updateItem(id, data) {
  await db.query('UPDATE items SET ... WHERE id = $1', [data, id]);
  await redis.del(`item:${id}`);
  await redis.del(`item:${id}:details`);
  await redis.del(`category:${data.categoryId}`);
}

// Event-driven invalidation (pub/sub)
await redis.publish('cache:invalidate', JSON.stringify({ key: `item:${id}` }));
```

### Cache Stampede (Dog-Pile)

```javascript
// Lock-and-rebuild
async function getItem(id) {
  const cacheKey = `item:${id}`;
  const lockKey = `lock:item:${id}`;

  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const lock = await redis.set(lockKey, 'locked', 'NX', 'EX', 10);
  if (lock) {
    try {
      const item = await expensiveDbQuery(id);
      await redis.set(cacheKey, JSON.stringify(item), 'EX', 3600);
      return item;
    } finally { await redis.del(lockKey); }
  }

  await new Promise(resolve => setTimeout(resolve, 100));
  return getItem(id);
}
```

```javascript
// Stale-while-revalidate
async function getItem(id) {
  const cacheKey = `item:${id}`;
  const cached = await redis.get(cacheKey);
  if (cached) {
    const { item, expiresAt } = JSON.parse(cached);
    if (expiresAt > Date.now()) return item;
    refreshItem(id);
    return item;
  }
  return await refreshItem(id);
}

async function refreshItem(id) {
  const item = await expensiveDbQuery(id);
  await redis.set(`item:${id}`, JSON.stringify({
    item, expiresAt: Date.now() + 3600000,
  }), 'EX', 7200);
  return item;
}
```

### Key Naming Conventions

```javascript
const tenantPrefix = `tenant:${orgId}`;
await redis.set(`${tenantPrefix}:item:${itemId}`, JSON.stringify(item));
```

> **Trap:** Caching authenticated user-specific responses. Never cache `/api/profile` without including the user ID in the cache key.

> **Trap:** Inconsistent cache invalidation. Always invalidate ALL related keys atomically using `multi()`.

---

## 12. Database Optimization

### Connection Pooling

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
  queueTimeoutMillis: 10000,
});

async function query(text, params) {
  const client = await pool.connect();
  try { return await client.query(text, params); }
  finally { client.release(); }
}
```

> **Trap:** Too many connections. PostgreSQL defaults to `max_connections = 100`. 10 Node processes × 20 pool = 200 connections — the 201st is rejected. Use PgBouncer or reduce pool size.

### Query Optimization

```javascript
// Composite index: (tenant_id, category_id, created_at DESC)
// WHERE tenant_id = $1 AND category_id = $2 ORDER BY created_at DESC

// EXPLAIN ANALYZE
async function explainQuery(text, params) {
  const result = await pool.query(`EXPLAIN ANALYZE ${text}`, params);
  console.log(result.rows.map(r => r['QUERY PLAN']).join('\n'));
}
```

### N+1 Detection and Prevention

```javascript
// WRONG — lazy loading
const orders = await Order.findAll({ where: { userId: 1 } });
for (const order of orders) {       // 1 query
  const items = await order.getItems(); // N queries
}

// RIGHT — eager loading
const orders = await Order.findAll({
  where: { userId: 1 },
  include: [Item], // JOIN + 1 query
});
```

```javascript
// DataLoader
const DataLoader = require('dataloader');

const itemLoader = new DataLoader(async (ids) => {
  const items = await db.query('SELECT * FROM items WHERE id = ANY($1)', [ids]);
  const byId = new Map(items.rows.map(i => [i.id, i]));
  return ids.map(id => byId.get(id) || null);
});

// Automatically batches individual loads
const items = await Promise.all(order.itemIds.map(id => itemLoader.load(id)));
// Only 2 queries total instead of N+1
```

### Batch Operations

```javascript
// Bulk insert
async function bulkInsertItems(items) {
  const values = items.map((item, i) => `($${i * 3 + 1}, $${i * 3 + 2}, $${i * 3 + 3})`).join(', ');
  const params = items.flatMap(item => [item.name, item.price, item.tenantId]);
  await pool.query(`INSERT INTO items (name, price, tenant_id) VALUES ${values}`, params);
}
```

### Read Replicas

```javascript
const writer = new Pool({ host: process.env.DB_WRITER_HOST, max: 20 });
const reader = new Pool({ host: process.env.DB_READER_HOST, max: 50 });

const db = {
  async query(text, params) {
    const isWrite = /^\s*(INSERT|UPDATE|DELETE)/i.test(text);
    return (isWrite ? writer : reader).query(text, params);
  },
};
```

### Pagination Strategies

```javascript
// CURSOR (keyset) pagination — fast, stable
async function getItems(cursor, limit = 20) {
  const result = await pool.query(
    'SELECT * FROM items WHERE id > $1 ORDER BY id LIMIT $2',
    [cursor || 0, limit]
  );
  const items = result.rows;
  const nextCursor = items.length === limit ? items[items.length - 1].id : null;
  return { items, nextCursor };
}

// Composite cursor
async function getItemsOrdered(cursor, limit = 20) {
  const [createdAt, id] = cursor ? cursor.split('_') : [new Date().toISOString(), 0];
  const result = await pool.query(
    `SELECT * FROM items WHERE (created_at, id) < ($1::timestamp, $2::int)
     ORDER BY created_at DESC, id DESC LIMIT $3`,
    [createdAt, id, limit]
  );
  const last = result.rows[result.rows.length - 1];
  const nextCursor = last ? `${last.created_at.toISOString()}_${last.id}` : null;
  return { items, nextCursor };
}
```

> **Trap:** Not using `FOR UPDATE SKIP LOCKED` for concurrent access. Multiple workers will grab the same job row.

```javascript
async function pickJob() {
  const job = await pool.query(
    `UPDATE jobs SET status = 'processing', picked_at = NOW()
     WHERE id = (SELECT id FROM jobs WHERE status = 'pending' ORDER BY created_at LIMIT 1 FOR UPDATE SKIP LOCKED)
     RETURNING *`
  );
  return job.rows[0] || null;
}
```

---

## 13. Stream Processing

### Transform Streams

```javascript
const { Transform, pipeline } = require('stream');
const { createReadStream, createWriteStream } = require('fs');

const upperCaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  },
});

pipeline(createReadStream('input.txt'), upperCaseTransform, createWriteStream('output.txt'), (err) => {
  if (err) console.error('Pipeline failed:', err);
  else console.log('Pipeline completed');
});
```

### Object Mode Streams

```javascript
const filterTransform = new Transform({
  objectMode: true,
  transform(item, encoding, callback) {
    if (item.quantity > 0) this.push(item);
    callback();
  },
});

readableStream.pipe(filterTransform).on('data', (item) => {
  console.log('In-stock item:', item.sku);
});
```

### `pipeline` for Backpressure

```javascript
const { pipeline } = require('stream/promises');
const { createReadStream, createWriteStream } = require('fs');
const { createGzip } = require('zlib');

async function compressFile(input, output) {
  try {
    await pipeline(createReadStream(input), createGzip(), createWriteStream(output));
    console.log('Compression complete');
  } catch (err) {
    console.error('Pipeline failed:', err);
  }
}
```

### Parsing Large JSON/CSV Files

```javascript
// Large JSON with JSONStream
const { createReadStream } = require('fs');
const JSONStream = require('JSONStream');

await pipeline(
  createReadStream('huge-orders.json'),
  JSONStream.parse('items.*'),
  new Transform({
    objectMode: true,
    transform(order, encoding, callback) {
      processOrder(order);
      callback();
    },
  })
);
```

```javascript
// Large CSV with csv-parse
const { parse } = require('csv-parse');
const { stringify } = require('csv-stringify');

await pipeline(
  createReadStream('huge-dataset.csv'),
  parse({ columns: true, skip_empty_lines: true }),
  new Transform({
    objectMode: true,
    transform(record, encoding, callback) {
      callback(null, transformRecord(record));
    },
  }),
  stringify({ header: true }),
  createWriteStream('processed.csv')
);
```

### Streaming HTTP Responses

```javascript
app.get('/api/download/:fileId', async (req, res) => {
  const filePath = `/data/uploads/${req.params.fileId}`;
  const { size } = await stat(filePath);
  res.writeHead(200, { 'Content-Type': 'application/octet-stream', 'Content-Length': size });
  await pipeline(createReadStream(filePath), res);
});
```

```javascript
app.get('/api/items/export', async (req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json', 'Transfer-Encoding': 'chunked' });
  res.write('[');
  let first = true;
  const cursor = db.query(cursorQuery('SELECT * FROM items'));
  for await (const item of cursor) {
    if (!first) res.write(',');
    res.write(JSON.stringify(item));
    first = false;
  }
  res.write(']');
  res.end();
});
```

> **Trap:** Not handling backpressure. `pipe` and `pipeline` handle it automatically. Manual `data` events do NOT handle backpressure — everything buffers in memory.

```javascript
// WRONG — no backpressure
readable.on('data', (chunk) => { slowWrite.write(chunk); });

// RIGHT — pipeline handles backpressure
await pipeline(readable, slowWrite);
```

> **Trap:** Object mode `highWaterMark` counts objects (default 16). For 50MB objects, 16 × 50MB = 800MB buffered. Set `highWaterMark` appropriately.

> **Follow-up:** *How does error propagation differ between .pipe() and pipeline()?* (Node v15+): `pipeline` forwards errors between all streams and destroys them properly. `.pipe()` does not forward errors — causing hanging resources and memory leaks.

---

## 14. Observability

### OpenTelemetry Tracing

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { Resource } = require('@opentelemetry/resources');

const sdk = new NodeSDK({
  resource: new Resource({
    'service.name': 'order-service',
    'deployment.environment': process.env.NODE_ENV,
  }),
  traceExporter: new OTLPTraceExporter({ url: 'http://otel-collector:4317' }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();

process.on('SIGTERM', () => { sdk.shutdown().then(() => process.exit(0)); });
```

```javascript
// Manual spans
const { trace, context, SpanStatusCode } = require('@opentelemetry/api');
const tracer = trace.getTracer('order-service');

async function processOrder(orderId) {
  const span = tracer.startSpan('processOrder', { attributes: { 'order.id': orderId } });
  return await context.with(trace.setSpan(context.active(), span), async () => {
    try {
      await validateOrder(orderId);
      await processPayment(orderId);
      span.setStatus({ code: SpanStatusCode.OK });
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      throw err;
    } finally { span.end(); }
  });
}
```

### OpenTelemetry Metrics

```javascript
const { MeterProvider } = require('@opentelemetry/sdk-metrics');

const meter = new MeterProvider({ resource: new Resource({ 'service.name': 'order-service' }) }).getMeter('order-service');

const orderCounter = meter.createCounter('orders.created.total', { description: 'Total orders created' });
const orderLatency = meter.createHistogram('orders.processing.duration', {
  description: 'Processing duration', unit: 'ms',
  boundaries: [10, 50, 100, 200, 500, 1000, 2000, 5000],
});

orderCounter.add(1, { tenant: 'acme-corp' });
orderLatency.record(Date.now() - start, { status: 'success' });
```

### Structured Logging with Trace Context

```javascript
const pino = require('pino');
const { trace } = require('@opentelemetry/api');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    log(object) {
      const span = trace.getActiveSpan();
      if (span) {
        object.trace_id = span.spanContext().traceId;
        object.span_id = span.spanContext().spanId;
      }
      return object;
    },
  },
  redact: ['req.headers.authorization', 'password', 'token'],
});

app.use((req, res, next) => {
  req.log = logger.child({ requestId: req.id, userId: req.user?.id });
  next();
});
```

### Correlation IDs

```javascript
const { v4: uuidv4 } = require('uuid');

app.use((req, res, next) => {
  const correlationId = req.headers['x-correlation-id'] || uuidv4();
  req.correlationId = correlationId;
  res.setHeader('x-correlation-id', correlationId);
  next();
});

// Propagate to downstream services
async function callInventoryService(req, endpoint) {
  return fetch(`http://inventory-service${endpoint}`, {
    headers: { 'x-correlation-id': req.correlationId },
  });
}
```

### Health Check Endpoints

```javascript
// Readiness
app.get('/health/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');
    await redis.ping();
    res.json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not ready', reason: err.message });
  }
});

// Liveness
app.get('/health/live', (req, res) => { res.json({ status: 'alive' }); });
```

### Metrics Endpoints for Prometheus

```javascript
const promClient = require('prom-client');
promClient.collectDefaultMetrics({ prefix: 'order_service_' });

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
});

app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    const labels = { method: req.method, route: req.route?.path || req.path, status: res.statusCode };
    httpRequestTotal.inc(labels);
    end(labels);
  });
  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

### Key Metrics

```
Request (RED): Rate, Errors, Duration (p50/p95/p99)
System (USE): CPU, Memory, GC pause, Event loop lag
Business: Queue depth, Cache hit ratio, Connection pool usage
```

> **Trap:** High-cardinality labels. Never tag metrics with user IDs — each unique label creates a new time series. 10K users = 10K time series per metric.

> **Trap:** Tracing every request. Use sampling (10% is typical for high-traffic services). Sample all errors + a percentage of successes.

> **Trap:** Blocking the health check with dependency checks. Keep liveness probes lightweight. Only readiness probes should check dependencies, with timeouts.

---

## 15. OWASP in Node.js

### SQL/NoSQL Injection

```javascript
// WRONG — SQL injection
const query = `SELECT * FROM items WHERE name = '${req.query.name}'`;

// RIGHT — parameterized queries
const result = await pool.query('SELECT * FROM items WHERE name = $1', [req.query.name]);
```

```javascript
// NoSQL injection (MongoDB)
// WRONG
await User.find({ username: req.body.username, password: req.body.password });
// Input: { "username": "admin", "password": { "$gt": "" } }

// RIGHT — validate with Zod first
const loginSchema = z.object({ username: z.string(), password: z.string() });
const { username, password } = loginSchema.parse(req.body);
await User.find({ username, password });
```

```javascript
const createItemSchema = z.object({
  name: z.string().min(1).max(200),
  price: z.number().positive(),
  sku: z.string().regex(/^[A-Z0-9-]+$/),
  description: z.string().max(2000).optional(),
});

app.post('/api/items', async (req, res) => {
  const parsed = createItemSchema.safeParse(req.body);
  if (!parsed.success) return res.status(422).json({ errors: parsed.error.issues });
  const item = await db.query('INSERT INTO items ... RETURNING *', [parsed.data]);
  res.json(item.rows[0]);
});
```

### XSS

```javascript
const helmet = require('helmet');
app.use(helmet()); // CSP, X-Content-Type-Options, etc.

// Sanitize user input if rendering HTML
const createDOMPurify = require('dompurify');
const { JSDOM } = require('jsdom');
const purify = createDOMPurify(new JSDOM('').window);
const safeHtml = purify.sanitize(userInput);
```

### CSRF

```javascript
const csrf = require('csurf');
const cookieParser = require('cookie-parser');

app.use(cookieParser());
app.use(csrf({ cookie: true }));

app.get('/form', (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});
```

### SSRF

```javascript
const { URL } = require('url');
const { check } = require('ssrfcheck');

app.get('/api/proxy', async (req, res) => {
  try {
    const parsed = new URL(req.query.url);
    const allowedDomains = ['api.external-service.com'];
    if (!allowedDomains.includes(parsed.hostname)) return res.status(403).json({ error: 'Domain not allowed' });
    await check(req.query.url); // blocks private IPs
    const response = await fetch(req.query.url, { redirect: 'manual' });
    if (response.status >= 300 && response.status < 400) return res.status(403).json({ error: 'Redirects not allowed' });
    res.json(await response.json());
  } catch (err) { res.status(400).json({ error: 'Invalid URL' }); }
});
```

### Command Injection

```javascript
// WRONG — shell injection
exec(`convert ${req.body.fileName} -resize 200x200 /output/${req.body.fileName}`);

// RIGHT — spawn with array args (no shell)
const filename = path.basename(req.body.fileName);
if (!/^[a-zA-Z0-9._-]+$/.test(filename)) throw new Error('Invalid filename');
spawn('convert', [filename, '-resize', '200x200', `/output/${filename}`]);
```

### Insecure Deserialization

```javascript
// NEVER use eval, new Function, or vm.runInThisContext
eval(userInput); // arbitrary code execution

// Always validate parsed JSON
const parsed = itemSchema.parse(req.body);
```

### Dependency Vulnerabilities

```bash
npm audit
npm audit --audit-level=high
npx snyk test
```

### Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 100 });
app.use('/api/', limiter);

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true,
});
app.use('/api/auth/login', authLimiter);
```

> **Trap:** Relying on frontend validation — an attacker bypasses it trivially. Always validate server-side.

> **Trap:** Not escaping user input in error messages that appear in logs (log injection).

> **Trap:** Using `eval` or `new Function` even indirectly. Double-check no library uses them with user input.

---

## 16. TypeScript for Node.js

### Why TypeScript

```
1. Type safety — catch errors at compile time
2. Better DX — autocomplete, refactoring
3. Self-documenting types
4. Catches null checks, type mismatches early
5. Scales for large codebases
```

### Key Features

```typescript
interface Item {
  id: number;
  sku: string;
  name: string;
  price: number;
  quantity: number;
  categoryId?: number;
  readonly createdAt: Date;
}

type OrderStatus = 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';

interface ApiResponse<T> {
  data: T;
  meta: { total: number; page: number; pageSize: number };
}

type Result<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; error: string; code: number }
  | { status: 'loading' };

function handleResult<T>(result: Result<T>) {
  switch (result.status) {
    case 'success': console.log(result.data); break;
    case 'error': console.error(result.error, result.code); break;
    case 'loading': console.log('Loading...'); break;
  }
}

// Utility types
type PartialItem = Partial<Item>;
type ItemName = Pick<Item, 'id' | 'name'>;
type ItemWithoutTimestamps = Omit<Item, 'createdAt' | 'updatedAt'>;
type ItemRecord = Record<string, Item>;
```

### Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### NestJS Framework

```typescript
@Controller('api/orders')
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  @Post()
  @UseGuards(AuthGuard)
  async create(@Body() dto: CreateOrderDto) { return this.orderService.create(dto); }

  @Get(':id')
  async findOne(@Param('id') id: string) { return this.orderService.findOne(id); }
}

@Injectable()
export class OrderService {
  constructor(@InjectRepository(Order) private readonly orderRepo: Repository<Order>) {}

  async create(dto: CreateOrderDto): Promise<Order> {
    const saved = await this.orderRepo.save(this.orderRepo.create(dto));
    return saved;
  }
}
```

> **Trap:** Using `any` defeats the purpose of TypeScript. Use `unknown` instead — it forces type checking before use.

> **Trap:** Not enabling `strict: true`. Without it, TypeScript allows implicit `any` and unsafe null checks.

> **Trap:** Over-engineering types with complex generics. Write types that communicate intent clearly.

> **Trap:** TypeScript does NOT provide runtime validation. Always use Zod for runtime validation of external data.

```typescript
const ItemSchema = z.object({
  id: z.number(),
  sku: z.string().min(1).max(50),
  name: z.string().min(1).max(200),
  price: z.number().positive(),
});

const item = ItemSchema.parse(JSON.parse(jsonString));
// item is typed AND validated at runtime
```

---

## 17. gRPC in Node.js

### Protocol Buffers Definition

```protobuf
syntax = "proto3";
package inventory;

service InventoryService {
  rpc GetItem (GetItemRequest) returns (Item);
  rpc ListItems (ListItemsRequest) returns (stream Item);
  rpc UpdateStock (stream StockUpdate) returns (stream StockResult);
}

message GetItemRequest { string id = 1; }

message Item {
  string id = 1;
  string sku = 2;
  string name = 3;
  double price = 4;
  int32 quantity = 5;
  int64 created_at = 6;
  map<string, string> attributes = 7;
}

message ListItemsRequest {
  int32 page_size = 1;
  string page_token = 2;
  string category_id = 3;
}

message StockUpdate { string item_id = 1; int32 delta = 2; }
message StockResult { string item_id = 1; int32 new_quantity = 2; bool success = 3; }
```

### Code Generation

```bash
npm install @grpc/proto-loader @grpc/grpc-js
npm install --save-dev grpc-tools

npx grpc_tools_node_protoc \
  --js_out=import_style=commonjs,binary:./src/proto \
  --grpc_out=grpc_js:./src/proto \
  --ts_out=./src/proto \
  -I ./protos ./protos/inventory.proto
```

### Unary RPC

```javascript
// Server
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true, longs: String, enums: String, defaults: true, oneofs: true,
});
const inventoryProto = grpc.loadPackageDefinition(packageDefinition).inventory;

const server = new grpc.Server();
server.addService(inventoryProto.InventoryService.service, {
  GetItem: async (call, callback) => {
    try {
      const item = await db.query('SELECT * FROM items WHERE id = $1', [call.request.id]);
      if (!item.rows[0]) return callback({ code: grpc.status.NOT_FOUND, message: 'Not found' });
      callback(null, item.rows[0]);
    } catch (err) { callback({ code: grpc.status.INTERNAL, message: err.message }); }
  },
});

server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  server.start();
});

// Client
const client = new inventoryProto.InventoryService('inventory-service:50051', grpc.credentials.createInsecure());
client.GetItem({ id: '123' }, (error, item) => {
  if (error) console.error('gRPC error:', error.code, error.details);
  else console.log('Item:', item);
});
```

### Server Streaming

```javascript
// Server
ListItems: async (call) => {
  const items = await db.query('SELECT * FROM items WHERE category_id = $1 ORDER BY id LIMIT $2', [
    call.request.category_id, call.request.page_size,
  ]);
  for (const item of items.rows) call.write(item);
  call.end();
},

// Client
const call = client.ListItems({ page_size: 10, category_id: 'electronics' });
call.on('data', (item) => console.log('Received:', item.sku));
call.on('end', () => console.log('All items received'));
call.on('error', (err) => console.error('Stream error:', err));
```

### Bidirectional Streaming

```javascript
// Server
CheckAvailability: (call) => {
  call.on('data', async (request) => {
    const item = await db.query('SELECT quantity FROM items WHERE id = $1', [request.item_id]);
    const available = item.rows[0] && item.rows[0].quantity >= request.quantity;
    call.write({ item_id: request.item_id, available: !!available });
  });
  call.on('end', () => call.end());
},

// Client
const call = client.CheckAvailability();
call.on('data', (response) => console.log(`Item ${response.item_id}: ${response.available ? 'OK' : 'Low'}`));
call.write({ item_id: '1', quantity: 5 });
call.write({ item_id: '2', quantity: 100 });
call.end();
```

### Error Handling

```javascript
// Server
GetItem: (call, callback) => {
  if (!call.request.id) {
    return callback({ code: grpc.status.INVALID_ARGUMENT, message: 'ID required' });
  }
  // ...
}

// Client
client.GetItem({ id: '123' }, (error, item) => {
  if (error) {
    switch (error.code) {
      case grpc.status.INVALID_ARGUMENT: // bad request
      case grpc.status.NOT_FOUND: // not found
      case grpc.status.UNAVAILABLE: // service down — retry
      case grpc.status.DEADLINE_EXCEEDED: // timeout
    }
  }
});

// With deadline
client.GetItem({ id: '123' }, { deadline: Date.now() + 5000 }, (error, item) => { /* ... */ });
```

### gRPC vs REST

| Feature | gRPC | REST |
|---------|------|------|
| Protocol | HTTP/2 (binary) | HTTP/1.1 (text) |
| Serialization | Protobuf (binary) | JSON (text) |
| Schema | Required (.proto) | Optional (OpenAPI) |
| Streaming | Native (unary, server, client, bidirectional) | Chunked transfer |
| Browser support | Requires gRPC-Web | Native |
| Load balancing | Needs connection awareness | Standard HTTP LB |

> **Trap:** Browser support requires gRPC-Web (additional proxy). Debugging binary protocols is harder than JSON. Load balancing gRPC needs L7 awareness (HTTP/2 connection reuse).

---

## 18. Tier 3 Q&A Drill

1. **Q:** How do you diagnose event loop starvation in production?
   **A:** Use `process.hrtime.bigint()` or an interval-based lag detector. Export event loop lag as a Prometheus gauge. Use clinic doctor/flame to visualize. Look for synchronous operations (JSON.parse, crypto, fs.readFileSync) in hot paths.

2. **Q:** When would you use Worker Threads vs Child Processes?
   **A:** Worker Threads for CPU-bound JavaScript work (image processing, data transformation, crypto) where you want shared memory and lower overhead. Child Processes for running non-JS tools (Python scripts, ffmpeg), shell commands, or when you need strong process isolation.

3. **Q:** What causes memory leaks in Node.js?
   **A:** Event listeners not removed, closures capturing large objects, unbounded caches, global variables, timers/intervals not cleared, accumulating EventEmitter listeners, and detached DOM nodes (if applicable).

4. **Q:** How do you debug a memory leak in production?
   **A:** Take heap snapshots via `node --inspect` or `v8.getHeapSnapshot()`. Diff two snapshots in Chrome DevTools. Look for objects with unexpectedly large retained size. Use `why-is-node-running` for active handles. Track GC event frequency and duration.

5. **Q:** Describe graceful shutdown. Why is it critical in K8s?
   **A:** Handle SIGTERM by closing the HTTP server (stop accepting connections), draining in-flight requests, closing database/Redis/queue connections, and exiting cleanly. Critical in K8s because pods are killed regularly during rolling updates, scaling down, and node maintenance. Without graceful shutdown, active users get connection resets.

6. **Q:** What's the difference between PM2 reload and restart?
   **A:** Restart kills all workers and starts new ones (downtime). Reload restarts workers one by one, waiting for each new worker to become ready before killing the old one (zero-downtime). Use `--wait-ready` and `process.send('ready')` for safe reloads.

7. **Q:** How do you prevent cache stampede?
   **A:** Lock-and-rebuild: only one process rebuilds the cache, others wait. Stale-while-revalidate: serve stale data while refreshing in the background. Early expiration: refresh the cache before TTL expires.

8. **Q:** What is the N+1 problem and how do you prevent it?
   **A:** When an ORM lazy-loads related entities one at a time. For N parent records, you execute 1 + N queries. Prevent with eager loading (JOIN), DataLoader for batching, or manually batching queries.

9. **Q:** How do you handle concurrent job processing from a database-backed queue?
   **A:** Use `FOR UPDATE SKIP LOCKED` (PostgreSQL) to atomically claim a job. This ensures each job is processed exactly once, even with multiple workers.

10. **Q:** What's the difference between `Promise.all` and `Promise.allSettled`?
    **A:** `Promise.all` short-circuits on first rejection — other results are lost. `Promise.allSettled` waits for all promises to settle, returning both fulfilled and rejected results. Use `allSettled` when you need all results regardless of failure (e.g., batch notifications).

11. **Q:** How do you propagate a correlation ID across microservices?
    **A:** Generate a UUID at the API gateway or first service. Pass it via HTTP header (`x-correlation-id`) or gRPC metadata. Include it in structured logs (Pino), spans (OpenTelemetry), and downstream requests.

12. **Q:** What metrics do you monitor for a Node.js service?
    **A:** RED: Request rate, error rate, duration (p50/p95/p99). USE: CPU utilization, memory (heap used, RSS), event loop lag, GC pause time. Business: queue depth, cache hit ratio, connection pool utilization.

13. **Q:** When should you use gRPC over REST?
    **A:** Internal service-to-service communication where performance matters, you need strong typing via Protobuf, or you need streaming (bidirectional, server push). Not ideal for browser clients (needs gRPC-Web) or public APIs where debuggability matters.

14. **Q:** How do you handle partial failure in microservices?
    **A:** Circuit breakers (opossum), timeouts, retries with exponential backoff and jitter, fallbacks (stale cache or defaults), graceful degradation (return partial data), and async communication for non-critical paths.

15. **Q:** What's the overhead of spawning a Worker Thread?
    **A:** Creating a V8 isolate, loading modules, allocating memory. For trivial work taking <1ms, the overhead exceeds the benefit. Batch work into chunks and send to a worker pool.

16. **Q:** How do you measure and optimize GC in Node.js?
    **A:** Use PerformanceObserver to track GC events. If GC is >10% of CPU time, reduce allocation frequency (object pooling, reuse buffers, avoid creating objects in hot paths). Increase `--max-old-space-size` only as a temporary measure.

17. **Q:** What's the difference between `spawn` and `exec`?
    **A:** `spawn` streams output via events, default no shell, handles large data well. `exec` buffers output (max 200KB default), uses a shell (risk of injection). Use `spawn` for most cases, `exec` only for simple shell commands with controlled input.

18. **Q:** How do you implement the Saga pattern in a Node.js microservice architecture?
    **A:** Choreography: each service publishes events and listens for compensating events. Orchestration: a central Saga coordinator calls services and executes compensating actions in reverse order on failure. Store saga state in a durable store for recovery.

19. **Q:** What security headers should you set for a Node.js API?
    **A:** Use `helmet` which sets: Content-Security-Policy, X-Content-Type-Options (nosniff), X-Frame-Options (DENY), Strict-Transport-Security, X-XSS-Protection, and more. Add CORS headers if needed.

20. **Q:** How do you validate that a retry strategy is working?
    **A:** Log retry attempts with attempt number, delay, and error. Export a counter metric for retries by operation and outcome. Monitor success rate after retries vs total failures. Use distributed tracing to see retry chains across services.
