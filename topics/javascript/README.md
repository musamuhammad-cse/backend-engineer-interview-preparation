# JavaScript / Node.js — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant inventory SaaS, 88% query reduction, zero-downtime migration, 20K+ DAU trading platform, Chronos (distributed job scheduler)

---

## How to use this material

This is not a skim-read. Each tier builds on the previous one and is designed for **active recall**, not passive reading.

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud as if to an interviewer | 20 min/section |
| 2 | Type out the code examples from memory — do not copy/paste | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff your answer | 20 min/section |
| 4 | Write down where your answer was vague — vagueness is what fails senior loops | 5 min |

**The senior signal is not knowing definitions.** It's knowing trade-offs, failure modes, and what you'd do at 3am when it breaks. Every section below flags **Traps** (what interviewers use to catch you) and **Follow-ups** (the second and third question they will ask).

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | JavaScript engine & execution model, types & coercion, scope & hoisting, closures, `this` binding, prototypes & inheritance, promises & async/await, error handling, ES6+ features, Node.js process model, modules | 6–8 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Express/Fastify frameworks, middleware patterns, routing, auth & JWT, database access (Knex/Prisma/Sequelize), streams & buffers, process management & clustering, testing with Jest/Vitest, logging & debugging, error handling patterns, API design & best practices, rate limiting, pagination, CORS & security | 10–12 hours |
| [`03-senior.md`](./03-senior.md) | Advanced async patterns, event loop diagnosis, worker threads & child processes, memory management & leak detection, performance profiling, cluster mode & PM2, microservices & message queues (Bull/BullMQ), caching strategies, database optimization, graceful shutdown, stream processing, OpenTelemetry observability, OWASP in Node.js, TypeScript for Node.js, gRPC | 14–18 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 200+ interview questions with answers, code puzzles, live-coding exercises, debugging scenarios, system design prompts, STAR stories | Ongoing drill |

---

## Coverage map

### JavaScript language
- Execution model (V8, event loop, call stack, callback queue, microtask queue)
- Types: primitives, objects, type coercion, `===` vs `==`, `typeof`, `instanceof`
- Scope (global, function, block), hoisting, TDZ
- Closures: lexical scope, practical use, memory implications
- `this` binding rules: default, implicit, explicit (call/apply/bind), arrow, `new`
- Prototypes: `__proto__`, `prototype`, prototype chain, `Object.create`, classes (syntax sugar)
- Promises: states, chaining, static methods (`all`, `allSettled`, `race`, `any`)
- Async/await: error handling, concurrency (`Promise.all`), sequential vs parallel
- Error handling: `try/catch/finally`, error types, custom errors, async error propagation
- ES6+ features: arrow functions, destructuring, spread/rest, template literals, optional chaining, nullish coalescing, modules
- Data structures: Map, Set, WeakMap, WeakSet, typed arrays
- Generators & iterators, `Symbol.iterator`, `for...of`

### Node.js runtime
- Process model: single-threaded event loop, libuv, thread pool
- Event loop phases: timers, pending callbacks, idle/prepare, poll, check, close
- `process.nextTick`, microtasks vs macrotasks
- `process` object: env, argv, exit codes, signals
- CommonJS vs ESM, `require` resolution algorithm
- `Buffer`, `Stream`, path, fs, crypto modules
- Child processes: `spawn`, `exec`, `fork`, `execFile`
- Cluster module and load balancing

### Express/Fastify frameworks
- Middleware: application-level, router-level, error-handling, third-party
- Request lifecycle, middleware ordering, async middleware
- Route parameters, query strings, body parsing
- Authentication: JWT, session-based, OAuth 2.0
- Input validation (Joi, Zod, express-validator)
- Error handling patterns: centralized error handler, async wrapper
- Rate limiting, CORS, helmet, compression
- File upload handling (multer)
- Testing with Supertest

### Databases
- Connection pooling
- ORM/query builder selection: Prisma, Knex, Sequelize, TypeORM
- Migrations and seeding
- N+1 prevention, eager loading
- Transaction management
- Caching strategies (Redis)

### Performance & operations
- Memory leak patterns: closures, event emitters, global variables, retained references
- Diagnosis: heap snapshots, CPU profiles, Clinic.js, 0x
- Cluster mode and PM2 for process management
- Graceful shutdown: `SIGTERM` handling, draining connections, finishing requests
- Stream processing for large datasets
- Worker threads for CPU-intensive work
- Bull/BullMQ for job queues

### Architecture & patterns
- Microservices with Node.js
- Event-driven architecture with EventEmitter
- Observer, Pub/Sub patterns
- Circuit breaker, retry, timeout patterns
- OpenTelemetry for distributed tracing
- GraphQL with Apollo Server
- gRPC in Node.js
- TypeScript for Node.js: types, interfaces, generics, decorators

---

## Study order recommendation

Even if you use Node.js daily, run the tiers in order. The Basic tier covers the event loop, closure mechanics, and `this` binding — topics that trip up senior candidates who rely on intuition rather than understanding.

```
Week 1:  01-basic.md              + Basic Q&A drill
Week 2:  02-intermediate.md       + Intermediate Q&A drill
Week 3:  03-senior.md (first half: architecture & patterns)
Week 4:  03-senior.md (second half: performance & operations)
Week 5+: 04-question-bank.md daily drill + STAR story rehearsal
```

**Next topic in skill order:** Databases (PostgreSQL, MySQL, indexing, transactions, isolation levels).
