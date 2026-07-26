# JavaScript / Node.js — Question Bank & Drill Material

Daily practice: Pick one section each day. Start with Rapid-Fire (flashcard style), attempt 2-3 code puzzles, then one live-coding exercise or design prompt. Review STAR stories before interview. Drill until each answer comes without hesitation.

## Table of Contents

1. [Rapid-Fire: JavaScript Language (60 questions)](#1-rapid-fire-javascript-language)
2. [Rapid-Fire: Node.js Runtime (40 questions)](#2-rapid-fire-nodejs-runtime)
3. [Rapid-Fire: Express/Fastify & Backend Patterns (40 questions)](#3-rapid-fire-expressfastify--backend-patterns)
4. [Rapid-Fire: Architecture & Systems (40 questions)](#4-rapid-fire-architecture--systems)
5. [Code Puzzles — "What Does This Print?" (10 puzzles)](#5-code-puzzles)
6. [Live-Coding Exercises](#6-live-coding-exercises)
7. [Debugging Scenarios](#7-debugging-scenarios)
8. [System Design Prompts](#8-system-design-prompts)
9. [STAR Stories](#9-star-stories)
10. [Questions to Ask the Interviewer](#10-questions-to-ask-the-interviewer)
11. [Red Flags to Avoid](#11-red-flags-to-avoid)

---

## 1. Rapid-Fire: JavaScript Language (60 questions)

**Q1:** What is the event loop?
**A:** Mechanism that handles async callbacks — call stack empty → microtasks → macrotask (one) → render.

**Q2:** What are the phases of Node.js event loop?
**A:** timers → pending callbacks → idle/prepare → poll → check → close callbacks. `process.nextTick` runs between each phase.

**Q3:** What is the difference between `null` and `undefined`?
**A:** `undefined` = variable declared but not assigned. `null` = intentional absence of value (typeof null → "object", historical bug).

**Q4:** What does `typeof NaN` return?
**A:** `"number"`. NaN is the only JS value not equal to itself (`NaN !== NaN` → true).

**Q5:** What is hoisting?
**A:** `var` declarations and `function` declarations are hoisted (moved to top of scope). `let`/`const` are hoisted but in TDZ (Temporal Dead Zone).

<details><summary>Q6: What does this print?</summary>

```js
console.log(0.1 + 0.2 === 0.3);
```
`false`. Floating point IEEE 754 precision. Use `Math.abs(a - b) < Number.EPSILON`.

</details>

**Q7:** What is closure?
**A:** Function retains access to its lexical scope even when executed outside that scope. Used for data privacy, currying, factory functions.

<details><summary>Q8: Closure trap — what prints?</summary>

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```
`3` printed three times. `var` is function-scoped, `i` is shared. Fix: use `let` (block-scoped) or IIFE.

</details>

**Q9:** How is `this` determined?
**A:** By invocation site: 1) default → global/window (strict: `undefined`), 2) implicit → object before `.`, 3) explicit → `call`/`apply`/`bind`, 4) `new` → new instance, 5) arrow → lexical scope.

**Q10:** What is the prototype chain?
**A:** Objects have `[[Prototype]]` link (accessed via `Object.getPrototypeOf` or `__proto__`). Property lookup walks the chain until found or `null`.

**Q11:** What is the difference between `__proto__` and `prototype`?
**A:** `__proto__` is the actual prototype link on instances; `prototype` is a property on constructor functions set on instances created with `new`.

<details><summary>Q12: Prototype puzzle — what prints?</summary>

```js
class A { say() { return 'A'; } }
class B extends A { say() { return 'B'; } }
const b = new B();
console.log(b.__proto__.say.call(new A()));
```
`'B'`. `b.__proto__` is `B.prototype`; `B.prototype.say.call(new A())` runs `B`s method with `A` instance as `this`.

</details>

**Q13:** What is the difference between `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`?
**A:** `.all` → rejects fast on first rejection. `.allSettled` → waits all, returns statuses. `.race` → settles on first (resolve or reject). `.any` → resolves on first resolve, rejects only if all reject (AggregateError).

**Q14:** What is the event loop priority between microtasks and macrotasks?
**A:** After each macrotask, drain entire microtask queue before next macrotask. Microtasks: `process.nextTick` > `Promise.then`/`queueMicrotask`.

<details><summary>Q15: Print order puzzle</summary>

```js
console.log(1);
setTimeout(() => console.log(2), 0);
Promise.resolve().then(() => console.log(3));
process.nextTick(() => console.log(4));
console.log(5);
```
`1, 5, 4, 3, 2`. Sync first → nextTick → promise microtask → setTimeout macrotask.

</details>

**Q16:** What is the difference between `==` and `===`?
**A:** `==` allows coercion (`null == undefined`, `'1' == 1`); `===` checks type and value (no coercion).

<details><summary>Q17: Coercion puzzle</summary>

```js
console.log([] == ![]);
```
`true`. `![]` → `false` → `0`. `[] == 0` → `'' == 0` → `0 == 0` → true.

</details>

<details><summary>Q18: More coercion</summary>

```js
console.log(1 < 2 < 3);
console.log(3 > 2 > 1);
```
`true` (1 < 2 → true → 1 < 3 → true), `false` (3 > 2 → true → 1 > 1 → false).

</details>

**Q19:** What is IIFE?
**A:** Immediately Invoked Function Expression: `(function(){ ... })()`. Creates isolated scope before `let`/`const` existed.

**Q20:** What is a generator?
**A:** Function with `function*` that yields values via `yield`. Returns iterator. Pauses execution, resumes later. Used for lazy evaluation, async flows, infinite sequences.

<details><summary>Q21: Generator puzzle</summary>

```js
function* gen() { yield 1; yield 2; }
const g = gen();
console.log(g.next());
console.log(g.return(42));
```
`{ value: 1, done: false }`, `{ value: 42, done: true }`. `return()` terminates the generator with given value.

</details>

**Q22:** What is a Symbol?
**A:** Unique, immutable primitive. Used for object property keys (avoid name collisions). `Symbol.for` creates global registry symbols.

**Q23:** What is the difference between `Map` and `WeakMap`?
**A:** `WeakMap` keys must be objects, held weakly (no prevent GC), not iterable, no `.size`. Prevents memory leaks in caches/metadata.

**Q24:** What is the spread operator doing?
**A:** Iterates and expands iterable into elements. `[...arr]`, `{...obj}`, `fn(...args)`. Shallow copy.

**Q25:** What is destructuring?
**A:** Unpack values from arrays or properties from objects into variables. `const { a, b } = obj`, `const [x, y] = arr`.

**Q26:** What is optional chaining?
**A:** `obj?.prop?.nested`. Short-circuits to `undefined` if intermediate is `null`/`undefined`. Avoids `Cannot read property of null`.

**Q27:** What is nullish coalescing?
**A:** `a ?? b` returns `b` only if `a` is `null` or `undefined`. Unlike `||`, doesn't treat `0`, `''`, `false` as falsy.

**Q28:** What are default parameters?
**A:** `function f(x = 1) {}`. Evaluated at call time. Each call gets a fresh default (important for mutable defaults like `[]`).

**Q29:** What is the difference between `var`, `let`, `const`?
**A:** `var`: function-scoped, hoisted, can redeclare. `let`: block-scoped, TDZ, can reassign. `const`: block-scoped, TDZ, cannot reassign (but object properties can mutate).

**Q30:** What is a Promise?
**A:** Object representing eventual completion (or failure) of async operation. States: pending → fulfilled / rejected. `.then()`, `.catch()`, `.finally()`.

**Q31:** What is `async/await` syntactic sugar over?
**A:** Promises. `async` function returns a promise. `await` pauses execution until promise settles (non-blocking, microtask-based).

<details><summary>Q32: Async/await trap</summary>

```js
async function foo() {
  return 'bar';
}
console.log(foo());
```
`Promise { 'bar' }`. Async functions always return promises, not values directly.

</details>

**Q33:** What is the difference between `throw` and `throw new Error()`?
**A:** `throw` can throw any value (string, number, object). `throw new Error()` creates stack trace — always use Error for debugging.

**Q34:** What is a `try/catch/finally`?
**A:** `try` runs code; `catch` handles thrown errors; `finally` runs regardless of success/failure (for cleanup). `finally`'s return overrides try/catch return.

**Q35:** What is `JSON.stringify` behavior with `undefined`, `function`, `Symbol`?
**A:** They are omitted (or replaced with `null` in arrays). Use replacer function for customization.

**Q36:** What are template literals?
**A:** Backtick strings with interpolation `${}` and multi-line support. Tagged templates allow custom string processing.

**Q37:** What is `Object.freeze` vs `Object.seal`?
**A:** `freeze`: immutable — cannot add/delete/change properties or prototype. `seal`: cannot add/delete, but existing properties can be changed. Both shallow.

**Q38:** What is `Array.prototype.reduce`?
**A:** Accumulates values via callback `(acc, val, idx, arr) => newAcc`. With initial value; without, uses first element as acc.

<details><summary>Q39: Reduce trap</summary>

```js
const r = [1, 2, 3].reduce((acc, v) => acc + v);
console.log(r);
```
`6`. No initial value → acc starts as `1`, iterates `2`, `3`.

</details>

**Q40:** What is `Array.prototype.flatMap`?
**A:** Maps each element then flattens result one level. Like `map().flat()`. More memory-efficient as it does both in one pass.

**Q41:** What is the difference between `splice` and `slice`?
**A:** `splice` mutates array (add/remove). `slice` returns new array (non-mutating). `splice(idx, count, ...items)`, `slice(start, end)`.

**Q42:** What is `RegExp` `exec` vs `match`?
**A:** `exec` (on regex) returns match object with index, input, groups, and updates `lastIndex` for global regex. `match` (on string) returns array of matches or null.

**Q43:** What is `Intl` used for?
**A:** Internationalization: `Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.Collator`. Respects locale-sensitive formatting.

**Q44:** What is `BigInt`?
**A:** Arbitrary precision integers. `123n` or `BigInt(123)`. Cannot mix with regular Number in arithmetic. `typeof BigInt(1)` → `"bigint"`.

**Q45:** What is `globalThis`?
**A:** Standard global object across environments. `window` in browser, `global` in Node, `self` in workers. Unified access.

**Q46:** What is `Proxy`?
**A:** Wraps object to intercept operations via traps (get, set, has, deleteProperty, apply, construct). Used for validation, logging, reactivity.

<details><summary>Q47: Proxy puzzle</summary>

```js
const target = { a: 1 };
const handler = { get: (obj, prop) => prop in obj ? obj[prop] : 42 };
const p = new Proxy(target, handler);
console.log(p.a, p.b);
```
`1`, `42`. Proxy defines custom getter — `a` exists, `b` doesn't so returns fallback 42.

</details>

**Q48:** What is `Reflect`?
**A:** Object with methods that correspond to Proxy traps. `Reflect.get`, `Reflect.set`, etc. Enables default forwarding behavior in proxies.

**Q49:** What is `ArrayBuffer` and `TypedArray`?
**A:** `ArrayBuffer` is a fixed-length raw binary data buffer. `TypedArray` (Uint8Array, Float64Array, etc.) provides views into the buffer.

**Q50:** What is `SharedArrayBuffer` and `Atomics`?
**A:** `SharedArrayBuffer` is shared memory between workers. `Atomics` provides atomic operations (add, compareExchange, wait, notify) for synchronization.

**Q51:** What are ES modules vs CommonJS?
**A:** ESM: `import`/`export`, static analysis, top-level `await`, strict by default, live bindings. CJS: `require`/`module.exports`, dynamic loading, `__dirname`/`__filename` available.

**Q52:** What is tree-shaking?
**A:** Dead code elimination by bundlers. Works with ESM static structure — unused exports are dropped. CJS is dynamic, harder to tree-shake.

**Q53:** What are dynamic imports?
**A:** `import('module')` returns promise. Enables code splitting, lazy loading. Works in both ESM and CJS.

**Q54:** What is the `import.meta` object?
**A:** Contains metadata about the current module. `import.meta.url` gives module file URL. `import.meta.resolve` resolves specifiers.

**Q55:** What is `eval`?
**A:** Executes arbitrary string as code. Security risk, disables optimizations (V8 cannot optimize). Never use — use `Function` constructor as safer alternative.

**Q56:** What is `with` statement?
**A:** Extends scope chain with given object. Disallowed in strict mode. Leads to ambiguity and performance issues. Never use.

**Q57:** What is tail-call optimization?
**A:** If function's last action is calling another function (and not retaining stack frame), engine can reuse current stack frame. Only ES6 strict mode; V8 implements but not for all cases.

<details><summary>Q58: Tail call example</summary>

```js
function factorial(n, acc = 1) {
  if (n <= 1) return acc;
  return factorial(n - 1, n * acc); // tail call
}
```
Optimizable by engine (but V8 often doesn't apply TCO outside strict mode).

</details>

**Q59:** What is the difference between `for...in` and `for...of`?
**A:** `for...in` iterates enumerable property keys (including prototype chain). `for...of` iterates iterable values (requires Symbol.iterator).

**Q60:** What is `Symbol.iterator`?
**A:** Well-known symbol defining default iterator for an object. Objects with `[Symbol.iterator]` are iterable. Arrays, Strings, Maps, Sets have it built-in.

**Q61:** What is a decorator (proposal stage)?
**A:** Function that modifies class/method/property. `@decorator` syntax. In JS still TC39 stage 3. TypeScript has experimental support. Used in NestJS extensively.

**Q62:** What is the pipeline operator proposal?
**A:** `|>` proposal. Pipes value through functions: `value |> fn1 |> fn2`. Stage 2. Not yet in standard. Alternative: method chaining or composition.

**Q63:** What is `Array.prototype.toSorted` vs `Array.prototype.sort`?
**A:** `.toSorted()` returns new sorted array (non-mutating). `.sort()` sorts in place (mutating). Both use stable sort since V8 7.0 / ES2019.

**Q64:** What is `structuredClone`?
**A:** Global function for deep cloning objects. Handles `Date`, `RegExp`, `Map`, `Set`, `ArrayBuffer`, circular references. Better than `JSON.parse(JSON.stringify())`.

**Q65:** What is `Array.prototype.groupBy`?
**A:** Groups array elements by key. Returns `{ [key]: [values] }`. `Object.groupBy(items, callback)`. Stage 3 → ES2024.

---

## 2. Rapid-Fire: Node.js Runtime (40 questions)

> Trap: Many senior engineers confuse `nextTick` with microtasks. Know the queue order cold.

**Q1:** What is `process.nextTick`?
**A:** Queues callback to run **after current operation**, before next macrotask. Runs between each phase of event loop. Higher priority than promise microtasks.

**Q2:** What is the difference between `process.nextTick` and `setImmediate`?
**A:** `nextTick` runs before next macrotask (between phases). `setImmediate` runs in check phase (after poll). `nextTick` can starve I/O if called recursively.

**Q3:** Timer ordering — `setTimeout(fn, 0)` vs `setImmediate`?
**A:** In main module: order is non-deterministic (depends on phase at time of call). In I/O callback: `setImmediate` always before `setTimeout` (check phase runs before timers).

<details><summary>Q4: Timer trap</summary>

```js
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'));
  setImmediate(() => console.log('immediate'));
});
```
`'immediate'` then `'timeout'`. In I/O callback, poll phase completes → check phase (setImmediate) → timers phase (setTimeout).

</details>

**Q5:** What is libuv?
**A:** C library that provides the event loop, thread pool, async I/O (files, DNS, sockets). Node.js delegates non-blocking I/O to libuv.

**Q6:** What is the libuv thread pool? Default size?
**A:** Thread pool for async I/O that kernel doesn't support (file system, DNS, some crypto). Default size: 4 threads. Controllable via `UV_THREADPOOL_SIZE`.

**Q7:** What runs on the thread pool vs native async?
**A:** Network I/O (sockets) uses OS async I/O (epoll/kqueue/IOCP), not thread pool. File system, `crypto.pbkdf2`, `crypto.randomBytes`, `zlib` use thread pool.

**Q8:** What is `Buffer`?
**A:** Raw memory allocation outside V8 heap. `Buffer.from()`, `Buffer.alloc()`. Fixed size. Represents binary data. `toString()`, `slice()` (returns new Buffer referencing same memory).

**Q9:** `Buffer.alloc` vs `Buffer.allocUnsafe`?
**A:** `alloc` zero-fills. `allocUnsafe` may contain old data (security risk if not overwritten) but faster. Always zero-fill or immediately write.

**Q10:** What is a Stream?
**A:** Abstract interface for reading/writing data in chunks. Four types: Readable, Writable, Duplex, Transform. Extends EventEmitter.

<details><summary>Q11: Stream backpressure</summary>

```js
readable.pipe(writable);
```
`pipe` handles backpressure automatically — pauses readable when writable's internal buffer exceeds `highWaterMark`. Without pipe, need manual `drain` event handling.

</details>

**Q12:** `pipeline` vs `pipe`?
**A:** `pipeline` (streams v2+) accepts multiple streams, calls callback on completion/error, and properly cleans up. `pipe` returns destination stream and doesn't handle errors between streams.

**Q13:** `spawn` vs `exec` vs `fork`?
**A:** `spawn`: streams stdout/stderr, no shell. `exec`: buffers output, uses shell. `fork`: spawns Node process with IPC channel, used for worker processes.

<details><summary>Q14: exec trap</summary>

```js
const { exec } = require('child_process');
exec('ls -la', (err, stdout, stderr) => {
  console.log(stdout);
});
```
`exec` buffers all output in memory. Max buffer (default 1MB) → `Error: stdout maxBuffer exceeded` for large output. Use `spawn` for large data.

</details>

**Q15:** What is `child_process.fork` IPC?
**A:** Creates bidirectional IPC channel between parent and child. Uses `send()` and `message` event. Messages are serialized with `JSON.stringify`/`parse`.

**Q16:** What is the cluster module?
**A:** Spawns multiple worker processes sharing same port. Master process manages workers. Round-robin (default on Linux) or OS scheduling. Workers crash independently — master restarts.

<details><summary>Q17: Cluster sticky sessions</summary>

```js
const cluster = require('cluster');
if (cluster.isMaster) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
} else {
  // worker: create server
}
```
Workers share server port. No sticky session by default — socket.io requires sticky via `sticky-session` or Redis adapter.

</details>

**Q18:** Module resolution — how does `require('./foo')` work?
**A:** 1) check cache, 2) check if built-in module, 3) resolve file: `foo`, `foo.js`, `foo.json`, `foo.node`, 4) resolve directory: `package.json` main, `index.js`, 5) walk `node_modules` up, 6) throw MODULE_NOT_FOUND.

**Q19:** `require` vs `import` — can they coexist?
**A:** In `.mjs` files: only `import`. In `.cjs` files: only `require`. In `.js` files: depends on `package.json` type field (`"type": "module"` for ESM). ESM can `import` CJS; CJS can `require` ESM only with dynamic import.

**Q20:** What is `ERR_REQUIRE_ESM`?
**A:** Thrown when `require()` is used on an ESM module. Must use dynamic `import()` or rename to `.cjs`.

**Q21:** What is `NODE_ENV` used for?
**A:** Convention to indicate environment (development/production). Express uses it for error handling. Libraries may optimize based on it. Not built into Node runtime itself.

**Q22:** What is the `path` module?
**A:** Utilities for file paths: `path.join`, `path.resolve`, `path.basename`, `path.dirname`, `path.extname`, `path.parse`. Cross-platform: `path.sep` (`/` vs `\`).

**Q23:** `path.join` vs `path.resolve`?
**A:** `join` concatenates with platform separator. `resolve` produces absolute path (resolves `..` and prepends cwd if not absolute).

**Q24:** What is `fs.promises`?
**A:** Promise-based file system API. `fs.promises.readFile`, `fs.promises.writeFile`, etc. Available since Node 10. Prefer over callback fs or `util.promisify`.

**Q25:** `readFile` vs `createReadStream`?
**A:** `readFile` loads entire file into memory (buffer). `createReadStream` streams chunks — use for large files to avoid OOM.

**Q26:** What is `os.cpus()`?
**A:** Returns array of objects with CPU info: model, speed, times (user, nice, sys, idle, irq). Used for cluster fork count.

**Q27:** What is `process.env`?
**A:** Object containing user environment variables. Values are strings. `process.env.NODE_ENV`. Can modify but don't — affects child processes.

**Q28:** What is `process.argv`?
**A:** Array: first element is Node path, second is script path, rest are CLI arguments. Use `yargs` or `commander` for parsing.

**Q29:** What is `process.exit` vs `process.exitCode`?
**A:** `process.exit(code)` forces immediate termination (can skip cleanup, truncate output). Set `process.exitCode = code` and let process exit naturally.

**Q30:** What is `process.on('uncaughtException')`?
**A:** Catch-all for unhandled errors. Should cleanup and exit (process is unstable after). Use for logging only — better to let process crash and restart via supervisor.

**Q31:** What is `process.on('unhandledRejection')`?
**A:** Catches unhandled promise rejections. In Node 15+, unhandled rejections terminate process. Always handle or attach a handler.

**Q32:** What is `process.on('warning')`?
**A:** Emitted for process warnings (deprecations, memory, maxListeners). `--no-warnings` flag suppresses. Listen to detect issues.

**Q33:** What is the `util.promisify`?
**A:** Converts callback-based function (error-first callback) to promise-based. Works with functions following `(err, val) => {}` convention.

**Q34:** What is `EventEmitter`?
**A:** Core pattern for observability. `emitter.on('event', handler)`, `emitter.emit('event', data)`. Manages listener list. Max 10 listeners by default (warning at > 10).

**Q35:** What is `once` vs `on`?
**A:** `emitter.once('event', fn)` — listener fires at most once and auto-removes. `emitter.on('event', fn)` — fires every time.

**Q36:** Memory leak — EventEmitter listeners?
**A:** Forgetting to remove listeners causes memory leaks (callback retains references). Use `emitter.removeListener`, `emitter.off`, or `once`.

**Q37:** What is `process.memoryUsage()`?
**A:** Returns `{ rss, heapTotal, heapUsed, external, arrayBuffers }`. `rss` = Resident Set Size (total memory). `heapUsed` = V8 heap. Monitor for leaks.

**Q38:** What is `process.hrtime.bigint()`?
**A:** High-resolution time in nanoseconds. Use for precise performance measurement. `process.hrtime()` is deprecated in favor of `process.hrtime.bigint()`.

**Q39:** What is the `util.types` module?
**A:** Type checking utilities: `util.types.isPromise`, `util.types.isRegExp`, `util.types.isDate`, `util.types.isNativeError`. More reliable than `typeof`.

**Q40:** What is `--inspect` flag?
**A:** Enables Chrome DevTools Protocol debugger. `node --inspect server.js`. Connect via Chrome `chrome://inspect`. Also `--inspect-brk` for break on first line.

---

## 3. Rapid-Fire: Express/Fastify & Backend Patterns (40 questions)

**Q1:** What is middleware ordering in Express?
**A:** Middleware executes in the order they are `app.use()`'d. Place global middleware (cors, helmet, parser) first, routes after, error middleware last.

<details><summary>Q2: Error middleware trap</summary>

```js
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```
Error middleware **must** have 4 parameters. If omitted `next`, Express treats it as regular middleware. Always call `next(err)` from route handlers.

</details>

**Q3:** How to handle async errors in Express 4?
**A:** Express 4 doesn't catch promise rejections. Must wrap async routes: `fn => (req, res, next) => fn(req, res, next).catch(next)`. Express 5 catches automatically.

**Q4:** JWT — access vs refresh token?
**A:** Access token (short-lived, 15min) in Authorization header. Refresh token (long-lived, 7d) used to get new access tokens. Refresh token stored in httpOnly cookie or DB.

**Q5:** JWT payload — what to include?
**A:** Minimal: `sub` (user ID), `iat`, `exp`. Optional: `roles`, `scope`. Never store sensitive data (JWT is base64url, not encrypted).

<details><summary>Q6: JWT verification trap</summary>

```js
const decoded = jwt.verify(token, secret);
```
Always verify signature and expiration. Never use `jwt.decode` alone — it decodes without verification. Always catch `JsonWebTokenError`, `TokenExpiredError`.

</details>

**Q7:** Zod vs Joi vs Yup?
**A:** Zod: TypeScript-first, inferred types, composable, small bundle. Joi: powerful, legacy, large. Yup: similar to Zod but runtime validation. Zod is modern standard.

**Q8:** Zod — how to validate request body?
**A:** `z.object({ name: z.string().min(1) }).parse(req.body)`. Use `safeParse` for error handling without try/catch.

**Q9:** Prisma vs Knex vs Sequelize?
**A:** Prisma: type-safe client, auto-generated, declarative migrations. Knex: query builder (unopinionated), raw SQL power. Sequelize: heavy ORM, auto-magic, known for gotchas.

**Q10:** What is N+1 query problem?
**A:** Fetching a list + one query per item. E.g., `SELECT * FROM users` then for each user `SELECT * FROM orders WHERE user_id = ?`. Solution: eager loading (JOIN, `IN` clause), dataloader.

<details><summary>Q11: How Prisma solves N+1</summary>

```js
const users = await prisma.user.findMany({
  include: { orders: true }
});
```
Single query with JOIN or batched `findMany`. Use `include` or `select` for relational data.

</details>

**Q12:** Connection pooling — what is it?
**A:** Reuses database connections instead of opening/closing per request. Pool maintains min/max connections. Improves latency and resource usage.

**Q13:** What is `pg-pool` default max?
**A:** Default `max` connections in `pg-pool` is 10. Should match DB `max_connections`. Calculate as `(max_connections - reserved) / instances`.

**Q14:** Connection pool exhaustion symptoms?
**A:** API timeouts, `TimeoutError: waiting for connection`, DB CPU low but app unresponsive. Check: leaked connections, pool too small, slow queries.

**Q15:** Redis — common use cases?
**A:** Caching, session store, rate limiting, pub/sub, distributed locks (Redlock), queues (Bull/BullMQ), leaderboards (sorted sets).

**Q16:** Redis — what data types?
**A:** String, List (LPUSH/BRPOP), Set, Sorted Set (ZADD), Hash (HSET), Stream (XADD, XREADGROUP — for message queues).

<details><summary>Q17: Redis atomicity trap</summary>

```js
const val = await redis.get('key');
// RACE: another client may change between get and set
await redis.set('key', Number(val) + 1);
```
Not atomic. Use `INCR` for counters, `WATCH`/`MULTI`/`EXEC` for transactions, or Lua scripting for custom atomic ops.

</details>

**Q18:** Rate limiting — strategies?
**A:** Token bucket (burst allowance), Sliding window (Redis sorted set), Fixed window (simple, inaccurate at edges). Libraries: `express-rate-limit`, `ratelimiter` (Redis-backed).

**Q19:** CORS — what headers?
**A:** `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`. Preflight `OPTIONS` request for non-simple requests.

<details><summary>Q20: CORS with credentials trap</summary>

```js
res.setHeader('Access-Control-Allow-Origin', '*');
res.setHeader('Access-Control-Allow-Credentials', 'true');
```
Cannot use `*` with credentials. Must specify explicit origin like `https://example.com`.

</details>

**Q21:** Helmet — what does it do?
**A:** Sets security HTTP headers: `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Strict-Transport-Security`, `Content-Security-Policy`, etc.

**Q22:** File upload — multer vs busboy?
**A:** Multer (Express middleware, uses busboy internally) handles `multipart/form-data`. Busboy (lower-level, streaming, no req.body mixing). Use multer for simple cases, busboy for streaming to S3.

**Q23:** Error handling patterns?
**A:** Custom error classes extending Error. Centralized error middleware. Structured response: `{ error: { code, message, details } }`. Distinguish operational vs programmer errors.

<details><summary>Q24: Custom error class</summary>

```js
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
  }
}
```

</details>

**Q25:** Graceful shutdown — what steps?
**A:** 1) Listen for `SIGTERM`/`SIGINT`, 2) stop accepting new requests (`server.close()`), 3) drain in-flight requests (timeout), 4) close DB/Redis connections, 5) exit.

<details><summary>Q26: Graceful shutdown code</summary>

```js
process.on('SIGTERM', async () => {
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 30000).unref();
  await db.destroy();
});
```

</details>

**Q27:** PM2 — what is it?
**A:** Production process manager for Node. Features: cluster mode, auto-restart, zero-downtime reload, log management, monitoring, startup scripts.

**Q28:** PM2 cluster mode vs Node cluster module?
**A:** PM2 wraps cluster module. `pm2 start app.js -i max` forks workers. PM2 adds health checks, graceful reload, log aggregation, memory limit restart.

**Q29:** Node — how to measure event loop lag?
**A:** `process.hrtime()` diffs in timer callbacks. Or `require('perf_hooks').monitorEventLoopDelay({ resolution: 20 })` (Node 10+).

**Q30:** What is `server.timeout`?
**A:** Sets server-level timeout (in milliseconds) for incoming requests. Default 0 (no timeout). Set to avoid hanging connections.

**Q31:** `app.set('trust proxy')` — why?
**A:** Express behind proxy (nginx, LB) sees proxy IP, not client. Setting `trust proxy` makes `req.ip` use `X-Forwarded-For`. Value: number of proxy hops or boolean.

**Q32:** What are conditional requests?
**A:** `If-None-Match` (ETag) and `If-Modified-Since` (Last-Modified). Server returns `304 Not Modified` if resource unchanged. Saves bandwidth.

**Q33:** ETag generation?
**A:** Express generates ETag automatically (hash of response). For performance, use `app.set('etag', false)` and generate custom strong ETags.

**Q34:** Compress middleware?
**A:** Compresses responses (gzip/brotli). `compression` npm package. Add early in middleware chain. Trade-off: high CPU, less bandwidth.

**Q35:** Session middleware — express-session vs cookie-session?
**A:** `express-session`: stores session data on server (MemoryStore, RedisStore). `cookie-session`: stores session data in signed cookie (size limit 4KB). RedisStore for production.

**Q36:** CSRF protection?
**A:** `csurf` middleware or `sameSite: 'strict'` cookie attribute. CSRF token in forms/AJAX headers. Largely mitigated by SameSite cookies in modern browsers.

**Q37:** How to version an API?
**A:** URL path (`/v1/users`), header (`Accept: application/vnd.api+json;version=1`), or query param. URL is simplest for production. Deprecate with `Sunset` header.

**Q38:** Pagination best practices?
**A:** Cursor-based (keyset pagination) for large datasets (stable under writes). Offset-based easier but can miss/duplicate rows. Always include `total`, `next_cursor`, `prev_cursor`.

<details><summary>Q39: Cursor pagination SQL</summary>

```js
SELECT * FROM users WHERE id > ? ORDER BY id ASC LIMIT 20
```
More efficient than `OFFSET` — uses index, no scan. Cursor is last `id` from previous page.

</details>

**Q40:** Fastify vs Express — key differences?
**A:** Fastify: faster (JSON serialization optimization), schema-based validation, built-in serialization, plugin system, logging. Express: larger ecosystem, more familiar, middleware-first.

---

## 4. Rapid-Fire: Architecture & Systems (40 questions)

**Q1:** Microservices — when to use?
**A:** Independent deployability, team autonomy, polyglot, scaling independently. Avoid for small teams or simple domains — start monolith, extract when boundaries stabilize.

**Q2:** Message queues — RabbitMQ vs Redis Streams vs Bull/BullMQ?
**A:** RabbitMQ: AMQP, complex routing, persistent, mature. Redis Streams: Redis-native, consumer groups, simpler. Bull/BullMQ: Redis-based, priority, delayed jobs, UI.

**Q3:** Bull/BullMQ — key concepts?
**A:** Queue, Job, Worker. Jobs: add with data, options (delay, attempts, backoff). Workers: process jobs. Concurrency control. Events: completed, failed, stalled.

<details><summary>Q4: Bull queue retry with backoff</summary>

```js
const job = await queue.add(data, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 2000 }
});
```
Exponential backoff: 2s, 4s, 8s, 16s. Retries on failure. Manual ack only on success.

</details>

**Q5:** gRPC vs REST?
**A:** gRPC: binary (protobuf), HTTP/2, streaming (unary, server, client, bidirectional), strict schema, code gen. REST: text-based, HTTP/1.1, caching, browser-friendly, loose schema.

**Q6:** gRPC error handling — what is a status code?
**A:** gRPC has built-in status codes: `OK`, `CANCELED`, `INVALID_ARGUMENT`, `DEADLINE_EXCEEDED`, `NOT_FOUND`, `ALREADY_EXISTS`, `PERMISSION_DENIED`, `UNAUTHENTICATED`, `INTERNAL`, `UNAVAILABLE`.

**Q7:** OpenTelemetry — what signals?
**A:** Traces (distributed tracing), Metrics (counters, histograms, gauges), Logs. Context propagation via W3C TraceContext headers.

**Q8:** What is sampling in tracing?
**A:** Not all traces are recorded. Head-based (decision at root, pass `sampled` flag) or tail-based (decision after complete). Default: probabilistic sampling (e.g., 5%).

<details><summary>Q9: OpenTelemetry Node setup</summary>

```js
const { NodeTracerProvider } = require('@opentelemetry/node');
const { BatchSpanProcessor } = require('@opentelemetry/tracing');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const provider = new NodeTracerProvider();
provider.addSpanProcessor(new BatchSpanProcessor(new OTLPTraceExporter()));
provider.register();
```

</details>

**Q10:** Circuit breaker — states?
**A:** Closed (normal), Open (failures threshold exceeded — requests fail fast), Half-Open (allow test request after timeout). Libraries: `opossum`, `cockatiel`.

<details><summary>Q11: Circuit breaker with opossum</summary>

```js
const circuit = new CircuitBreaker(asyncCall, {
  timeout: 3000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000
});
circuit.fallback(() => cache.get('key'));
```
50% failure rate opens circuit. 30s reset timeout. Fallback to cache.

</details>

**Q12:** Saga pattern — choreography vs orchestration?
**A:** Choreography: each service emits events, others react. Orchestration: central orchestrator (Saga manager) commands each service and handles compensation. Orchestration is easier to monitor.

**Q13:** Compensating transaction?
**A:** If a step in saga fails, run compensation actions (undo previous steps). E.g., order created → payment failed → cancel order, release inventory.

**Q14:** Worker threads vs child processes?
**A:** Worker threads (`worker_threads`): shared memory (SharedArrayBuffer), same process, lighter. Child processes: separate memory, isolated, communicate via IPC. Use workers for CPU-intensive JS, child processes for running non-Node programs.

<details><summary>Q15: Worker thread example</summary>

```js
const { Worker, isMainThread, parentPort } = require('worker_threads');
if (isMainThread) {
  const worker = new Worker(__filename);
  worker.postMessage('work');
} else {
  parentPort.on('message', (msg) => {
    parentPort.postMessage(heavyComputation(msg));
  });
}
```

</details>

**Q16:** Event loop blocking — how to detect?
**A:** Monitor event loop lag (`perf_hooks.monitorEventLoopDelay`). Watchdogs set threshold (e.g., > 50ms lag = blocked). Tools: `clinic.js doctor`, `0x` flamegraphs.

**Q17:** Common memory leak patterns in Node?
**A:** 1) Retained references (global caches, closures), 2) EventEmitter listeners not removed, 3) Timer/callback holding large closure, 4) Accumulating data in arrays, 5) Promises never resolved leaving dangling chains, 6) Large `Buffer` references.

**Q18:** How to debug memory leaks?
**A:** Heap snapshot comparison (Chrome DevTools). `--inspect`, take snapshot before/after suspected leak, diff. Look for detached DOMs (unlikely in Node) or unexpected retained objects.

**Q19:** TypeScript vs JavaScript for Node?
**A:** TS: type safety, refactoring, IDE support, catch bugs at compile time, better DX. JS: simpler, faster iteration, less build step. For backend > 10k LOC, TS wins.

**Q20:** NestJS — what is it?
**A:** Opinionated Node framework. Modules, decorators, DI, guards, interceptors, pipes. Uses Express or Fastify under the hood. Java/Spring-like structure.

<details><summary>Q21: NestJS guard example</summary>

```ts
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest();
    return validateToken(req.headers.authorization);
  }
}
```

</details>

**Q22:** Docker for Node — best practices?
**A:** Multi-stage builds (dev → build → prod). Use `node:18-alpine` (small, secure). `.dockerignore` (avoid copying node_modules). Use non-root user. Enable `NODE_ENV=production`.

**Q23:** Node in Kubernetes — readiness vs liveness probes?
**A:** Readiness: traffic should reach pod (check `/health`). Liveness: pod is healthy — restart if fails (check deeper `/health/ready`). Don't use same endpoint for both.

<details><summary>Q24: Health check endpoint</summary>

```js
app.get('/health', (req, res) => {
  const dbOk = await db.raw('SELECT 1');
  res.status(dbOk ? 200 : 503).json({ status: dbOk ? 'ok' : 'degraded' });
});
```

</details>

**Q25:** Graceful shutdown in K8s?
**A:** K8s sends SIGTERM, then SIGKILL after `terminationGracePeriodSeconds` (default 30s). Pod removed from endpoints immediately. Node needs to handle SIGTERM and drain within the period.

**Q26:** What is Prometheus?
**A:** Time-series metrics database. Pull model (scrapes targets). Metrics types: Counter, Gauge, Histogram, Summary. Node libraries: `prom-client` for instrumenting.

**Q27:** What are Grafana dashboards?
**A:** Visualization layer for Prometheus (and other data sources). Create dashboards with panels. Alerting rules. Senior engineers should know RED metrics: Rate, Errors, Duration.

**Q28:** RED method for monitoring?
**A:** Rate (requests per second), Errors (failed requests), Duration (latency p50/p95/p99). Covers user-facing service health.

**Q29:** USE method for monitoring?
**A:** Utilization (time resource busy), Saturation (extra work queued), Errors (failure count). For infrastructure (CPU, memory, disk, network).

**Q30:** What is the Node.js `async_hooks` module?
**A:** Tracks async resources (promises, timeouts, TCP). Can create async context propagation (like cls-hooked/AsyncLocalStorage). Use `AsyncLocalStorage` for request context.

<details><summary>Q31: AsyncLocalStorage usage</summary>

```js
const { AsyncLocalStorage } = require('async_hooks');
const storage = new AsyncLocalStorage();
app.use((req, res, next) => {
  storage.run({ requestId: uuid() }, () => next());
});
// Anywhere: storage.getStore().requestId
```

</details>

**Q32:** What is the `vm` module?
**A:** Runs JavaScript in isolated V8 context. Use for sandboxing (user code execution). Limited security — not a real sandbox (can escape via prototype poisoning).

**Q33:** What is WASM in Node?
**A:** WebAssembly runs in V8. Use for CPU-heavy computation where native perf needed. Load `.wasm` file, instantiate, call exported functions.

**Q34:** HTTP/2 in Node?
**A:** Built-in `http2` module. Server push, multiplexing, header compression. Use `http2.createSecureServer` with TLS. Express doesn't natively support — use `spdy` or Fastify.

**Q35:** Server-Sent Events (SSE) vs WebSocket?
**A:** SSE: one-directional (server→client), HTTP, auto-reconnect, text-only. WebSocket: bidirectional, binary, lower overhead. Use SSE for real-time updates (dashboards), WebSocket for chat/collaboration.

<details><summary>Q36: SSE implementation</summary>

```js
res.writeHead(200, {
  'Content-Type': 'text/event-stream',
  'Cache-Control': 'no-cache',
  Connection: 'keep-alive'
});
setInterval(() => res.write(`data: ${JSON.stringify(data)}\n\n`), 1000);
```

</details>

**Q37:** What is DI (Dependency Injection)?
**A:** Passing dependencies externally rather than creating internally. Improves testability (mock), flexibility. NestJS has built-in DI. For plain Node: `awilix`, `tsyringe`, or manual.

**Q38:** What is the Repository pattern?
**A:** Abstraction layer between business logic and data access. `UserRepository.find(id)` vs raw `User.findById(id)`. Makes testing easier, swaps DB without changing business code.

**Q39:** What is CQRS?
**A:** Command Query Responsibility Segregation. Separate write (commands) and read (queries) models. Used in complex domains. Often paired with Event Sourcing.

**Q40:** Twelve-Factor App — relevant for Node?
**A:** 1) Codebase (single repo), 2) Dependencies (lockfile, no global), 3) Config (env vars), 4) Backing services (disposable), 5) Build/release/run, 6) Stateless processes, 7) Port binding, 8) Concurrency (process model), 9) Disposability (fast start/graceful shutdown), 10) Dev/prod parity, 11) Logs (stdout), 12) Admin processes.

---

## 5. Code Puzzles — "What Does This Print?" (10 puzzles)

<details><summary>Puzzle 1: setTimeout + Promise ordering</summary>

```js
setTimeout(() => console.log('A'), 0);
Promise.resolve().then(() => console.log('B'));
setTimeout(() => console.log('C'), 0);
Promise.resolve().then(() => console.log('D'));
console.log('E');
```

**Answer:** `E, B, D, A, C`

Explanation: Sync `E` first → microtasks `B`, `D` (Promise.then) → macrotask `A` (timer0) → macrotask `C` (timer0). Macrotask queue processes one at a time between microtask drainage.

</details>

<details><summary>Puzzle 2: `this` binding in nested callbacks</summary>

```js
const obj = {
  name: 'Alice',
  greet: function() {
    setTimeout(function() {
      console.log(this.name);
    }, 100);
  }
};
obj.greet();
```

**Answer:** `undefined` (or `undefined` in strict mode, global in non-strict).

Explanation: `function(){}` inside `setTimeout` loses `this` — no implicit binding (not called as method). Fix: arrow function, `.bind(this)`, or closure `const self = this`.

</details>

<details><summary>Puzzle 3: Classic closure loop with `var`</summary>

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), i * 100);
}
```

**Answer:** `3, 3, 3` (each after 0ms, 100ms, 200ms).

Explanation: `var i` is function-scoped (one `i` shared). By the time callbacks run, loop finished and `i = 3`. Each callback references same `i`. Fix: `let` (block-scoped) or IIFE.

</details>

<details><summary>Puzzle 4: async/await and error handling</summary>

```js
async function test() {
  try {
    return await Promise.reject('Error!');
  } finally {
    console.log('Finally');
  }
}
test().catch(console.log);
```

**Answer:** `Finally` then `Error!`

Explanation: `finally` runs before the rejection propagates. The `return await` ensures rejection is caught by `try`, but `finally` runs even without explicit `catch`. The `catch(console.log)` on `test()` catches it because the `finally` doesn't return a value to suppress the rejection.

</details>

<details><summary>Puzzle 5: `==` coercion chain</summary>

```js
const a = [1, 2, 3];
const b = [1, 2, 3];
const c = '1,2,3';
console.log(a == b);
console.log(a == c);
```

**Answer:** `false`, `true`

Explanation: Objects compared by reference — `a` and `b` are different arrays. `a == c`: `a.toString()` → `'1,2,3'` → `'1,2,3' == '1,2,3'` → `true`.

</details>

<details><summary>Puzzle 6: Prototype chain + instanceof</summary>

```js
function Animal(name) { this.name = name; }
Animal.prototype.speak = function() { return '...'; };

function Dog(name) { this.name = name; }
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

const d = new Dog('Rex');
console.log(d instanceof Dog);
console.log(d instanceof Animal);
console.log(d.constructor.name);
```

**Answer:** `true`, `true`, `Dog`

Explanation: `d.__proto__` → `Dog.prototype` (which is an object with `__proto__` → `Animal.prototype`). `instanceof` walks the chain. Explicit constructor fix ensures `d.constructor.name` is `Dog`, not `Animal`.

</details>

<details><summary>Puzzle 7: Microtask + macrotask deep nesting</summary>

```js
process.nextTick(() => console.log('tick1'));
setTimeout(() => console.log('timeout1'), 0);
Promise.resolve().then(() => {
  console.log('promise1');
  process.nextTick(() => console.log('tick2'));
  Promise.resolve().then(() => console.log('promise2'));
});
setTimeout(() => console.log('timeout2'), 0);
```

**Answer:** `tick1`, `promise1`, `tick2`, `promise2`, `timeout1`, `timeout2`

Explanation: Start → nextTick (tick1) → microtask queue (promise1) → inside microtask: adds tick2 (nextTick) and promise2 (microtask). NextTick runs before microtask? No — actually: nextTick queue is drained between each event loop phase and after each microtask? Let me be precise:

Correct order: `tick1` (nextTick runs before promise microtasks in Node), then `promise1` (microtask), then within promise1 handler: `tick2` added to nextTick queue, `promise2` added to microtask queue. After microtask completes, Node processes nextTick queue → `tick2`. Then more microtasks → `promise2`. Then `timeout1`, `timeout2`.

</details>

<details><summary>Puzzle 8: Spread + mutation</summary>

```js
const original = { a: 1, b: { c: 2 } };
const copy = { ...original };
copy.a = 10;
copy.b.c = 20;
console.log(original.a, original.b.c);
```

**Answer:** `1`, `20`

Explanation: Spread creates shallow copy. `copy.a` is independent (primitive). `copy.b` references same `{c: 2}` object as `original.b`. Mutating `copy.b.c` also changes `original.b.c`. Deep clone: `structuredClone()` or `JSON.parse(JSON.stringify())`.

</details>

<details><summary>Puzzle 9: typeof and instanceof gotchas</summary>

```js
console.log(typeof null);
console.log(typeof []);
console.log(typeof function(){});
console.log([] instanceof Array);
console.log([] instanceof Object);
console.log(null instanceof Object);
```

**Answer:** `'object'`, `'object'`, `'function'`, `true`, `true`, `false`

Explanation: `typeof null` → historical bug (000 in type tag). `[]` is object (instanceof Array and Object via prototype chain). `null instanceof Object` → `null` has no prototype → `false`.

</details>

<details><summary>Puzzle 10: Async generator + for-await-of</summary>

```js
async function* gen() {
  yield 1;
  yield await Promise.resolve(2);
  yield 3;
}
(async () => {
  for await (const val of gen()) {
    console.log(val);
  }
})();
```

**Answer:** `1`, `2`, `3`

Explanation: `async generator` yields promises (or values that get wrapped). `for-await-of` awaits each yielded value. Each `yield` suspends until the next `next()` call from the loop.

</details>

---

## 6. Live-Coding Exercises

<details><summary>Puzzle 11: Promise constructor + resolve timing</summary>

```js
new Promise((resolve) => {
  console.log('A');
  resolve('B');
  console.log('C');
}).then(console.log);
console.log('D');
```

**Answer:** `A`, `C`, `D`, `B`

Explanation: Promise executor runs synchronously. `A` logged, `resolve('B')` queues `.then` callback as microtask, `C` logged. Sync continues with `D`. Then microtask `B` runs.

</details>

<details><summary>Puzzle 12: `delete` operator on array</summary>

```js
const arr = [1, 2, 3];
delete arr[1];
console.log(arr.length);
console.log(arr);
```

**Answer:** `3`, `[1, empty, 3]`

Explanation: `delete` removes element but does not update length. Array becomes sparse. `arr[1]` is `undefined`. Use `splice()` to actually remove and re-index.

</details>

---

## 6. Live-Coding Exercises

Implement an in-memory sliding window rate limiter for Express. Allow `maxRequests` per `windowMs`. Return 429 with `Retry-After` header when exceeded.

<details><summary>Solution</summary>

```js
function rateLimiter({ windowMs = 60000, maxRequests = 100 } = {}) {
  const hits = new Map();

  return (req, res, next) => {
    const key = req.ip;
    const now = Date.now();
    if (!hits.has(key)) hits.set(key, []);

    const timestamps = hits.get(key).filter(t => now - t < windowMs);

    if (timestamps.length >= maxRequests) {
      const oldest = timestamps[0];
      const retryAfter = Math.ceil((oldest + windowMs - now) / 1000);
      res.set('Retry-After', retryAfter);
      return res.status(429).json({ error: 'Too many requests' });
    }

    timestamps.push(now);
    hits.set(key, timestamps);
    next();
  };
}
```

For production: use Redis sorted sets, sliding window with ZREMRANGEBYSCORE, ZCOUNT.

</details>

### Exercise 2: Idempotent API Endpoint with Redis

Implement an idempotent POST `/payments` endpoint. Use an `Idempotency-Key` header. If key already processed, return cached response. Ensure exactly-once execution.

<details><summary>Solution</summary>

```js
app.post('/payments', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  if (!idempotencyKey) return res.status(400).json({ error: 'Missing Idempotency-Key' });

  const cached = await redis.get(`idempotent:${idempotencyKey}`);
  if (cached) return res.status(200).json(JSON.parse(cached));

  // SET with NX to prevent concurrent duplicate processing
  const acquired = await redis.set(`lock:${idempotencyKey}`, '1', 'NX', 'EX', 30);
  if (!acquired) return res.status(409).json({ error: 'Request in progress' });

  try {
    const result = await processPayment(req.body);
    await redis.setex(`idempotent:${idempotencyKey}`, 86400, JSON.stringify(result));
    res.json(result);
  } finally {
    await redis.del(`lock:${idempotencyKey}`);
  }
});
```

</details>

### Exercise 3: Promise Pool / Concurrency Limiter

Implement a function `promisePool(tasks, concurrency)` that runs at most `concurrency` promises in parallel, returns results in order.

<details><summary>Solution</summary>

```js
async function promisePool(tasks, concurrency) {
  const results = [];
  const executing = new Set();
  let idx = 0;

  async function enqueue() {
    while (idx < tasks.length) {
      const currentIdx = idx++;
      const p = tasks[currentIdx]().then(r => {
        results[currentIdx] = r;
        executing.delete(p);
      });
      executing.add(p);

      if (executing.size >= concurrency) {
        await Promise.race(executing);
      }
    }
  }

  await enqueue();
  await Promise.all(executing);
  return results;
}

// Usage:
// const results = await promisePool(urls.map(u => () => fetch(u)), 5);
```

Alternative: Use a queue-based approach with `n` runners pulling from a shared task queue.

</details>

### Exercise 4: Event Emitter-Based Pub/Sub System

Implement a simple in-process pub/sub with topics, wildcard support, and at-least-once delivery. No external deps.

<details><summary>Solution</summary>

```js
class PubSub {
  constructor() {
    this.subscribers = new Map();
  }

  subscribe(topic, handler) {
    if (!this.subscribers.has(topic)) this.subscribers.set(topic, new Set());
    this.subscribers.get(topic).add(handler);
    return () => this.subscribers.get(topic)?.delete(handler);
  }

  publish(topic, message) {
    const parts = topic.split('.');
    for (const [key, handlers] of this.subscribers) {
      if (this.match(key, parts)) {
        for (const handler of handlers) {
          try { handler(message, topic); } catch (e) { console.error('handler error:', e); }
        }
      }
    }
  }

  match(pattern, topicParts) {
    const patParts = pattern.split('.');
    if (patParts.length > topicParts.length) return false;
    for (let i = 0; i < patParts.length; i++) {
      if (patParts[i] === '*') continue;
      if (patParts[i] !== topicParts[i]) return false;
    }
    return true;
  }
}

const bus = new PubSub();
const unsub = bus.subscribe('order.*', (msg) => console.log(msg));
bus.publish('order.created', { id: 1 });
unsub();
```

</details>

### Exercise 5: File Processing Pipeline with Streams

Process a CSV file: read a large CSV, transform rows (validate, enrich), write to JSONL output. Use streams to avoid loading everything in memory.

<details><summary>Solution</summary>

```js
const { createReadStream, createWriteStream } = require('fs');
const { Transform, pipeline } = require('stream');
const split2 = require('split2');

const input = createReadStream('input.csv');
const output = createWriteStream('output.jsonl');

const transform = new Transform({
  objectMode: true,
  transform(chunk, enc, done) {
    const line = chunk.toString();
    const [id, name, email] = line.split(',');
    if (!email || !email.includes('@')) return done();
    const obj = JSON.stringify({ id, name, email, processedAt: new Date().toISOString() });
    this.push(obj + '\n');
    done();
  }
});

pipeline(
  input,
  split2(),           // Split into lines
  transform,          // Validate and enrich
  output,
  (err) => {
    if (err) console.error('Pipeline failed:', err);
    else console.log('Done');
  }
);
```

For large CSVs: use `csv-parse` (parser stream) instead of manual split. Add backpressure handling with `drain` events.

</details>

---

## 7. Debugging Scenarios

### Scenario 1: API Latency Spike After Deploy

**Symptoms:** p95 latency went from 50ms to 2s after deploy. CPU normal, DB CPU normal.

**Procedure:**
1. Deploy diff — check what changed (new middleware? new DB call? synchronous file read?).
2. Check event loop lag (`monitorEventLoopDelay`). If > 100ms, event loop is blocked.
3. Check if new `JSON.parse`/`JSON.stringify` on large payloads, or `crypto` sync calls.
4. Profile with `clinic doctor` or `--inspect` + CPU profile (Chrome DevTools).
5. Common culprit: `JSON.stringify(res)` on large response, or forgotten `await` causing sync waterfall.

**Trap:** Senior engineer says "it's the DB" — but DB CPU is normal. Always check app-level metrics first.

### Scenario 2: Memory Leak in Queue Worker (OOM After Hours)

**Symptoms:** Bull worker processes email/sms jobs. After ~6 hours, OOM killed. Restart fixes, then repeats.

**Procedure:**
1. Check `process.memoryUsage()` trend — `heapUsed` growing linearly.
2. Take heap snapshot on startup + after 1000 jobs. Compare in Chrome DevTools.
3. Look for: accumulations in arrays, retained objects by closed-over variables, EventEmitter listeners not removed.
4. Common: job data stored in module-level cache without eviction, or `Map` storing job results indefinitely.
5. Fix: `WeakMap` or `Map` with TTL + size limit, ensure `emitter.off()` in cleanup.

**Trap:** Devs blame "GC not running" — but V8 GC runs fine. The real issue is retained references preventing GC.

### Scenario 3: Event Loop Blocked — High CPU, No Throughput

**Symptoms:** Server responding slowly. CPU 100%, but active connections are low. DB fine.

**Procedure:**
1. Use `top -H` or `clinic flame` to find CPU hotspot.
2. Common: `JSON.parse`/`JSON.stringify` on massive payloads (buffer bloat), crypto operations (`crypto.createHash` sync), `for` loops over million-item arrays, `util.inspect` on deep objects, `zlib` sync methods.
3. Check if `UV_THREADPOOL_SIZE` is too small for crypto heavy loads.
4. Fix: move CPU work to worker threads, offload to child process, increase thread pool size.

**Trap:** "Add more instances" — may not help if single request blocks the event loop. Need to fix the blocking call first.

### Scenario 4: Database Connection Pool Exhaustion

**Symptoms:** API intermittently returns 503 / timeout. DB CPU at 20%, but app logs show `TimeoutError: waiting for connection`.

**Procedure:**
1. Check `pg-pool` stats: `pool.waitingCount`, `pool.totalCount`, `pool.idleCount`.
2. Look for leaked connections: `app.use()` middleware opening connections without calling `release()`/`end()`, `pg.Client()` created outside pool.
3. Check if pool size (default 10) matches `max_connections` on Postgres.
4. Check for slow queries — if query takes 30s, it holds connection for 30s.
5. Apply statement timeout (`statement_timeout`) and connection timeout.

**Trap:** Dev says "increase pool size to 100" — but Postgres has max connections (100-300). Pool per instance × instance count must not exceed DB limit. Formula: `pool_max = (db_max_connections - superuser_reserved) / app_instances`.

### Scenario 5: Stale Cache Serving Wrong Data

**Symptoms:** User updates profile, but other services/users see old data for minutes. Cache TTL is 60s.

**Procedure:**
1. Check if cache is invalidated on write — `SET` + `DEL` on update?
2. Check cache key strategy: `user:{id}:profile` — deleting correctly?
3. Read-after-write consistency: If you write to DB and then delete cache, a concurrent read might repopulate stale data before DB write is committed.
4. Check for cache stampede: multiple requests simultaneously recreating cache.
5. Fix: write-through cache (update cache on write, not Lazy-loading), or set short TTL and accept eventual consistency.

**Trap:** "Just lower TTL to 5s" — works temporarily but increases DB load and doesn't solve race condition.

### Scenario 6: gRPC Service Returning INTERNAL Error Intermittently

**Symptoms:** gRPC service fails 1-5% of requests with `INTERNAL` error. Logs show no exception. Works on retry.

**Procedure:**
1. Enable gRPC interceptor to log full request/response metadata.
2. Check if error is from deadline exceeded (client timeout too short).
3. Check if it's a serialization error (protobuf encoding/decoding failure on large payloads).
4. Check if client is using HTTP/2 connection that was reset — gRPC requires HTTP/2. Check `GOAWAY` frames.
5. Check if any middle proxy (envoy, nginx) has max request size limit.
6. Add retry with backoff in gRPC client. Check if server's concurrency limit is hit.

**Trap:** Junior says "it's a bug in gRPC library" — 99% of the time it's configuration, timeout, or proxy issue.

### Scenario 7: File Descriptor Leak in Production

**Symptoms:** `EMFILE: too many open files` error after running for 24h. Restart fixes, then returns. Connections drop, health check fails.

**Procedure:**
1. Check `lsof -p <pid> | wc -l` — count open FDs. Compare with `ulimit -n` (default 1024).
2. Look for streams, file handles, sockets not properly closed.
3. Common: `fs.createReadStream()` without `close`/`destroy` on error, HTTP agents not reusing connections, database connections not returned to pool, temp files not cleaned.
4. Check `readable.on('end')` → missing `readable.destroy()`. Check error paths — `stream.on('error')` handlers that don't clean up.
5. Fix: Always use `pipeline` (auto-cleanup on error), add FD monitoring alerts, set `UV_THREADPOOL_SIZE` appropriately.

**Trap:** "Increase ulimit to 65535" — masks the leak. Fix the leaked file handles, not the limit. Also applies to sockets (`EMFILE` on server.listen).

---

## 8. System Design Prompts

### Prompt 1: Design a URL Shortener

**Requirements:** Shorten long URLs, redirect `https://short.ly/abc123` to original. Analytics (clicks, referrer, geo). High read volume (100:1 read:write). 100M URLs.

**Key decisions:**
- Hash function: Base62 (62^7 = 3.5T unique URLs) or Snowflake ID encoded.
- Storage: PostgreSQL (URLs, users), Redis cache (hot URLs, TTL).
- Redirection: 301 (permanent) for most, 302 (temporary) for analytics tracking.
- Analytics: async via message queue (Kafka/Redis Streams) — don't block redirect.
- Rate limiting per user for URL creation.
- Health check: ping DB + Redis.

**Trap:** Don't use "hash of URL" for short code — collisions and user can't have same URL twice with different custom slugs. Use auto-increment/snowflake + encode.

### Prompt 2: Design a Rate Limiter

**Requirements:** Rate limit per user/IP/API key. Burst allowance. Distributed. Low latency (< 5ms overhead).

**Key decisions:**
- Algorithm: Sliding window (Redis sorted sets) or Token bucket (Redis key with counter + timestamp).
- Storage: Redis cluster. Lua script for atomicity.
- Response: `429 Too Many Requests` with `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
- Config: per API key overriding default limits.
- Avoid hot keys: shard by user + route pattern.

**Trap:** Using fixed window (e.g., 1000 req/hour) — bursts at window boundary allow 2x traffic. Sliding window prevents this.

### Prompt 3: Design a Notification Service

**Requirements:** Send email, SMS, push notifications. Supports templates, priority queues, retry with backoff, delivery status tracking. Scale: 10M notifications/day.

**Key decisions:**
- Queue: Bull/BullMQ with priority queues (critical > transactional > marketing).
- Workers: Separate worker processes per channel (email worker, SMS worker, push worker).
- Templates: Stored in DB, rendered with Handlebars/MJML.
- Deduplication: Idempotency key per notification (prevent double send on retry).
- Provider abstraction: multiple email providers (SendGrid, SES) with circuit breaker.
- Status tracking: `notification.status` in DB (pending → sent → delivered/bounced).
- Analytics: Prometheus metrics (sent count, delivery rate, latency).

### Prompt 4: Design a Real-Time Collaborative Editing Backend

**Requirements:** Multiple users edit same document simultaneously. Conflicts resolved via CRDT (or OT). Persist document. 100K concurrent documents.

**Key decisions:**
- CRDT (Yjs/Automerge) over OT — CRDT is simpler, no central server needed for conflict resolution.
- WebSocket server (ws/uWebSockets.js) for real-time sync.
- Transport: binary protobuf/MessagePack for efficiency.
- Persistence: PostgreSQL document store (snapshot + append-only operations log).
- Horizontal scaling: Redis pub/sub to broadcast ops across WebSocket server instances. Or use Kafka for ordered log.
- Awareness: cursor positions, presence.
- Rate limiting per user (avoid spam).

**Trap:** "Use OT like Google Docs" — OT requires operational transformation, complex to implement correctly. CRDT (Yjs) is the modern standard.

### Prompt 5: Design a Distributed Job Scheduler

**Requirements:** Schedule one-time and recurring jobs. CRON-like but distributed. Guarantee at-least-once execution. Jobs: HTTP call, message queue publish, code execution. Failure handling with retries. 1M scheduled jobs.

**Key decisions:**
- Storage: PostgreSQL for job definitions (id, type, payload, schedule, next_run_at). Index on `next_run_at` for efficient polling.
- Poller: Application polls `next_run_at <= NOW()` in batches. Use `SELECT ... FOR UPDATE SKIP LOCKED` for distributed polling.
- Dispatch: Instead of direct execution, push to Bull/BullMQ for actual processing (decouple scheduling from execution).
- Recurring: `cron_next_schedule()` to compute next run. Store cron expression and compute next.
- Missed jobs: "catch up" mode or skip? Configurable per job.
- Leader election: For single scheduler writer, use Redis lock or PostgreSQL advisory lock.

**Trap:** Running cron on each app instance — all instances fire the same job. Use distributed lock or skip-locked queries to ensure single execution.

### Prompt 6: Design an Inventory Management System

**Requirements:** Track stock levels across warehouses. Reserve inventory during checkout. Release on timeout. Prevent oversell. Real-time stock updates. 10K SKUs, 100 warehouses.

**Key decisions:**
- SKU + warehouse granularity: `inventory(sku_id, warehouse_id, quantity, reserved)`.
- Reservation: `UPDATE inventory SET reserved = reserved + ? WHERE sku_id = ? AND warehouse_id = ? AND quantity - reserved >= ?`. Atomic with `WHERE` condition.
- Timeout: Redis key `reservation:{id}` TTL. On expiry, release reserved quantity via delayed job (Bull).
- Oversell prevention: DB constraint `CHECK (quantity >= 0)` or `CHECK (quantity - reserved >= 0)`.
- Caching: Redis hash for hot SKUs. Write-through: update DB + invalidate cache.
- Real-time: SSE or WebSocket push for dashboard.

**Trap:** Reading stock then decrementing without atomicity → race condition allows oversell. Always use `UPDATE ... SET quantity = quantity - ? WHERE quantity >= ?` in one statement.

---

## 9. STAR Stories

Use these frameworks to prepare specific examples from your experience. Fill in actual details.

### STAR 1: Database Query Reduction (Performance)

- **Situation:** API serving dashboard had p95 latency of 8s. N+1 queries everywhere. Each page load triggered 200+ SQL queries.
- **Task:** Reduce to < 20 queries per page, latency < 200ms.
- **Action:** Applied eager loading (Prisma `include`/Knex `join`). Implemented dataloader pattern for batched queries. Added Redis cache for reference data. Used SQL `WHERE id IN (...)`. Added pagination with cursor-based approach.
- **Result:** p95 latency 150ms (98% reduction). DB CPU dropped from 80% to 15%. Freed capacity for 3x traffic growth.

**Trap:** The interviewer might ask "Why not just throw more DB replicas?" — answer: "That masks the problem. Fixing the application first saves infrastructure costs and addresses root cause."

### STAR 2: Zero-Downtime Database Migration

- **Situation:** Migrating from MySQL to PostgreSQL for a multi-tenant SaaS with 24/7 SLA. 500GB data, 2000 tenants.
- **Task:** Zero downtime, no data loss, rollback possible.
- **Action:** Dual-write phase: write to both DBs (with feature flag). Backfill historical data in batches. Validation phase (compare checksums). Cutover: flip read to new DB. Monitor for a week then decommission old DB.
- **Result:** Zero incidents during migration. Rolled back twice during validation (data inconsistencies caught by comparison tool). Cutover complete in 2 hours.

**Trap:** "Just pg_dump and restore" — causes downtime. Dual-write is more work but safe for zero-downtime.

### STAR 3: Trading Platform Concurrency

- **Situation:** Crypto trading platform — concurrent order execution causing race conditions. Users charged twice (double spend bug).
- **Task:** Ensure exactly-once order execution at high throughput (1000 orders/sec).
- **Action:** Implemented optimistic locking on order rows (`UPDATE ... WHERE version = ?`). Idempotency key on order submission. Redis distributed lock for account balance updates. Dead letter queue for failed retries.
- **Result:** Zero double-spend incidents post-fix. Order throughput 2000/s. Reduced customer support tickets by 90%.

**Trap:** "Use mutex/lock everywhere" — creates bottlenecks. Optimistic locking and idempotency keys work better at scale.

### STAR 4: Multi-Tenant Architecture

- **Situation:** Single-tenant SaaS needed to become multi-tenant. Customers had data isolation requirements (HIPAA).
- **Task:** Design isolation strategy — shared DB + tenant filtering vs dedicated DB per tenant.
- **Action:** Chose hybrid: dedicated DB for enterprise (largest customers), shared DB with row-level security for SMBs. Tenant context via middleware (JWT → tenant_id in `AsyncLocalStorage`). Migrated data in phases, starting with new customers.
- **Result:** Saved $50K/month on infrastructure (SMBs on shared). Enterprise customers got full isolation. Scaling: 50 → 2000 tenants.

**Trap:** "Just use schema per tenant" — hard to backup/restore individual tenants. Row-level security + tenant_id column is simpler and proven.

### STAR 5: Microservices Migration

- **Situation:** Monolithic PHP/Laravel application with 500K LOC. Deployments took 2 hours, high regression risk.
- **Task:** Extract billing and notification services independently.
- **Action:** Strangler fig pattern: new endpoints route to new services; old endpoints still served by monolith. Shared DB first, then extract data ownership. Feature flags for gradual rollout. Async communication via RabbitMQ for events (invoice.paid, user.registered).
- **Result:** Deploy time reduced to 15 mins per service. Regression bugs down 70%. Team autonomous (3 squads). Handled 5x traffic growth.

---

## 10. Questions to Ask the Interviewer

1. "What does the team's on-call rotation look like, and what's the typical incident response process?"
2. "How do you currently handle observability — tracing, metrics, and logging?"
3. "What's the biggest technical challenge the team is facing right now?"
4. "How do you approach database migrations and zero-downtime deployments?"
5. "What's the decision process for choosing between TypeScript and JavaScript for new services?"
6. "How do you manage inter-service communication — message queues, gRPC, REST?"
7. "What does the CI/CD pipeline look like, and how long from PR to production?"
8. "How is error handling and retry logic standardized across services?"
9. "What's the most significant outage or incident in the past year, and what was the root cause?"
10. "How does the team balance shipping speed vs code quality / testing?"

**Trap:** Don't ask about perks, remote policy, or vacation in first interview. Keep the focus on engineering challenges and team dynamics.

---

## 11. Red Flags to Avoid

1. **"We don't write tests"** — no unit/integration tests means regression hell. Ask about testing strategy.
2. **"We don't have any monitoring"** — flying blind in production. No Prometheus/Datadog, no alerting.
3. **"We use a shared DB for everything"** — microservices but all services talk to same DB. Distributed monolith.
4. **"No code reviews"** — or rubber-stamp reviews. Indicates low engineering standards.
5. **"We need you to fix our legacy code that nobody understands"** — without documentation, tests, or team support. Ask how they plan to support you.
6. **"We rewrite everything every 2 years"** — no long-term thinking, chasing trends.
7. **"Deployments are manual and take all day"** — no CI/CD, fear of deployment, no rollback strategy.
8. **"We're looking for a 'rockstar' / '10x engineer'"** — expects heroics, usually means understaffed with unrealistic deadlines.
9. **"We don't use version control properly"** — directly committing to main, no branching strategy.
10. **"We don't have time for refactoring"** — accumulating technical debt indefinitely. No culture of continuous improvement.

---

*Last updated: July 2026. Drill every section until answers are immediate. Good luck.*
