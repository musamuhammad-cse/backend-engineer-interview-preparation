# JavaScript / Node.js — Tier 1: Basic (Deep Foundations)

> Do not skip this tier. You have 8+ years of backend experience, but JavaScript is not PHP or Go. Its single-threaded event loop, prototypal inheritance, and async model are fundamentally different. Interviewers plant "easy" questions about `this`, closures in loops, and the event loop to test whether your knowledge is deep or just enough to ship CRUD. If you cannot explain why `[] == ![]` is `true` or what happens between `setTimeout(fn, 0)` and `process.nextTick`, you are not ready for a senior Node.js role.

---

## Table of Contents

**Part A — JavaScript Language**
1. [Execution Model](#1-execution-model)
2. [Types & Coercion](#2-types--coercion)
3. [Scope & Hoisting](#3-scope--hoisting)
4. [Closures](#4-closures)
5. [`this` Binding](#5-this-binding)
6. [Prototypes & Inheritance](#6-prototypes--inheritance)
7. [Promises](#7-promises)
8. [Async/Await](#8-asyncawait)
9. [Error Handling](#9-error-handling)
10. [ES6+ Features](#10-es6-features)
11. [Data Structures](#11-data-structures)
12. [Generators & Iterators](#12-generators--iterators)

**Part B — Node.js Fundamentals**
13. [Node.js Process Model](#13-nodejs-process-model)
14. [Event Loop Phases](#14-event-loop-phases)
15. [Modules](#15-modules)
16. [Buffer & Stream](#16-buffer--stream)
17. [Tier 1 Q&A Drill](#17-tier-1-qa-drill)

---

# Part A — JavaScript Language

## 1. Execution Model

JavaScript runs inside a **host environment** (browser, Node.js, Deno). The engine (V8 in Chrome/Node, SpiderMonkey in Firefox, JavaScriptCore in Safari) executes the code. The engine itself has a call stack, a heap, and interacts with the event loop and callback queues provided by the host.

### The V8 Pipeline

```
Source Code
    ↓
Parser (tokenize → AST)
    ↓
Ignition (interpreter) → Bytecode
    ↓
TurboFan (JIT compiler) → Optimized Machine Code
    ↓
    ↺ Deoptimization if assumptions break
```

V8 compiles JavaScript to **bytecode** via the Ignition interpreter. Hot functions are compiled to **optimized machine code** by TurboFan. If runtime behavior invalidates the optimizer's assumptions, the code is **deoptimized** back to bytecode. This is why monomorphic code (same-shaped objects) runs faster than polymorphic code.

### The Call Stack

A LIFO stack of **execution contexts** (frames). Each function call pushes a frame; each return pops it.

```javascript
function multiply(a, b) {
  return a * b;          // frame 3
}

function square(n) {
  return multiply(n, n); // frame 2
}

console.log(square(5));  // frame 1 (global)
```

Call stack at `return a * b`:
```
square(5)       ← frame 2
<main>          ← frame 1
```

Stack overflow happens when frames exceed the limit (~10K in V8). Infinite recursion, deeply nested synchronous loops, or **synchronous recursion through event emitters** can cause it.

```javascript
function recurse() { recurse(); }
recurse();  // RangeError: Maximum call stack size exceeded
```

### The Heap

Objects, closures, arrays, and functions are allocated on the **heap**. V8's garbage collector is generational:

- **Young generation** (semi-space, scavenge collector) — fast allocation, frequent minor GC.
- **Old generation** (mark-sweep, mark-compact) — promoted after surviving a GC cycle, infrequent major GC.

Memory leaks happen when the GC cannot reclaim memory because references are retained (closures, event listeners, global caches, detached DOM nodes in browsers).

### The Event Loop — Text Diagram

```
                  ┌─────────────────────────────────┐
                  │         Call Stack              │
                  │  (synchronous execution)        │
                  └──────────┬──────────────────────┘
                             │ empty?
                             ▼
           ┌─────────────────────────────────────┐
           │         Microtask Queue             │
           │  (Promise.then/catch/finally,       │
           │   queueMicrotask, process.nextTick)  │
           └────────────────┬────────────────────┘
                            │ entire queue drained
                            ▼
           ┌─────────────────────────────────────┐
           │         Macrotask Queue             │
           │  (setTimeout, setInterval, I/O,     │
           │   setImmediate, UI events)          │
           └────────────────┬────────────────────┘
                            │ one task per tick
                            ▼
                  ┌─────────────────┐
                  │   Render step   │  (browser only)
                  └─────────────────┘
```

**The algorithm:**

1. Execute all synchronous code on the call stack.
2. When the stack is empty, drain the **entire microtask queue** (one by one — new microtasks added during this step are also drained).
3. Pick **one** macrotask from the front of the macrotask queue, execute it.
4. Repeat.

### How async actually works — without threads

JavaScript never has two pieces of code running at exactly the same time in the same thread. Instead:

- **I/O is non-blocking**: `fs.readFile`, network requests, timers are handed off to **libuv** (in Node.js) or the browser runtime.
- **libuv** uses the OS's **epoll** (Linux), **kqueue** (macOS), or **IOCP** (Windows) to watch for completion.
- When the operation completes, the callback is placed into the appropriate queue.
- The event loop picks it up when the call stack is empty.

```javascript
console.log('1: start');

setTimeout(() => console.log('2: timeout'), 0);

Promise.resolve().then(() => console.log('3: microtask'));

console.log('4: end');

// Output:
// 1: start
// 4: end
// 3: microtask
// 2: timeout
```

The promise `.then` runs before the `setTimeout` because microtasks are drained before the next macrotask.

> **Trap:** "JavaScript is single-threaded" is a useful simplification but imprecise. The **JavaScript language** has no threading model — it defines an execution model with a single thread of execution per **agent** (realm). However:
> - **Web Workers** (browser) and **Worker Threads** (Node.js) run JavaScript in separate OS threads with their own event loops, call stacks, and heaps. They communicate via message passing (`postMessage`), not shared memory (unless `SharedArrayBuffer` is used).
> - **libuv's thread pool** (4 threads by default, configurable via `UV_THREADPOOL_SIZE`) handles CPU-bound operations like `crypto.pbkdf2`, `fs` operations, and `zlib`. These run on separate threads but the **callbacks** come back to the main event loop.
> - Saying "JavaScript is single-threaded" in a senior interview without qualifying with Worker Threads and libuv's thread pool signals shallow understanding.

> **Follow-up:** *What happens when a synchronous CPU-heavy task runs?* It blocks the entire event loop. No I/O, no timers, no event handling can proceed. This is why you should offload CPU work to Worker Threads or split it into chunks with `setImmediate` / `setTimeout` to yield to the event loop.

> **Follow-up:** *How does await actually suspend execution?* `await` is syntactic sugar for a generator-based state machine. The engine saves the current function's state (local variables, `this`, execution position), yields control back to the event loop, and registers a microtask to resume when the awaited promise settles. This is why `await` never blocks the thread — it only suspends the current function.

---

## 2. Types & Coercion

### Primitives

JavaScript has 7 primitive types (and one for BigInt):

```javascript
// Primitives — immutable, compared by value
const str = 'hello';           // string
const num = 42;                // number
const big = 9007199254740991n; // bigint
const bool = true;             // boolean
const undef = undefined;       // undefined
const nil = null;              // null
const sym = Symbol('id');      // symbol

// Everything else is an object (including functions and arrays)
const obj = { a: 1 };          // object
const arr = [1, 2];            // object
const fn = () => {};           // function (callable object)
```

### `typeof` quirks

```javascript
typeof 'hello'           // 'string'
typeof 42                // 'number'
typeof 42n               // 'bigint'
typeof true              // 'boolean'
typeof undefined         // 'undefined'
typeof Symbol('id')      // 'symbol'
typeof null              // 'object'     ← THE FAMOUS BUG
typeof []                // 'object'
typeof {}                // 'object'
typeof (() => {})        // 'function'

Array.isArray([])        // true — use this instead of typeof
```

> **Trap:** `typeof null === 'object'` is a historical bug in JavaScript that cannot be fixed without breaking the web. To check for null: `value === null`.

```javascript
// Safe null check
function isReallyObject(v) {
  return v !== null && typeof v === 'object';
}
```

### Loose vs Strict Equality

| Operator | Behavior |
|----------|----------|
| `===` (strict) | No coercion. Returns `false` if types differ. |
| `==` (loose) | Coerces operands to the same type before comparing. |
| `Object.is` | Same as `===` except `NaN`, `-0`, `+0` |

```javascript
'1' == 1    // true  — string coerced to number
'1' === 1   // false — no coercion, types differ

0 == false  // true
0 === false // false

'' == false // true
'' === false// false

null == undefined  // true  — special rule
null === undefined // false
```

### The coercion algorithm for `==`

For `x == y`:

1. Same type → use `===`.
2. `null == undefined` → `true`.
3. String vs Number → coerce string to number.
4. Boolean vs any → coerce boolean to number (`true → 1`, `false → 0`), then re-evaluate.
5. Object vs String/Number → call `.valueOf()` or `.toString()` on object, then re-evaluate.

### The infamous `[] == ![]`

```javascript
[] == ![]  // true
```

Step by step:

1. `![]` → `!true` → `false` (any object is truthy, `!` negates).
2. `[] == false` → boolean vs object → coerce boolean: `[] == 0`.
3. `[] == 0` → object vs number → `[].valueOf()` returns `[]` (not primitive).
4. `[].toString()` returns `''` → `'' == 0`.
5. `'' == 0` → string vs number → `Number('')` → `0`.
6. `0 == 0` → `true`.

> **Trap:** The expression `[] == ![]` is `true` because of the chain: `![]` flips truthy to `false`, then coercion turns both sides into `0`. Interviewers love this because it forces you to walk the coercion algorithm step by step.

```javascript
// Other gotchas
[1, 2] == '1,2'      // true — toString gives "1,2"
[1] == 1              // true — toString "1" → Number("1") → 1
''
 == false             // true — Number('') = 0, false → 0
'0' == false          // true — Number('0') = 0
'\n' == 0             // true — Number('\n') = 0
```

### Falsy values

Exactly 7 falsy values in JavaScript:

```javascript
false
0               // and -0 (distinct from +0 for Object.is, but falsy)
0n              // bigint zero
''              // empty string
null
undefined
NaN
```

Everything else is truthy, including:
```javascript
'0'              // truthy! (non-empty string)
'false'          // truthy!
[]               // truthy!
{}               // truthy!
Infinity         // truthy!
```

### Number() coercion surprises

```javascript
Number('')           // 0
Number('  ')         // 0
Number('\t\n')       // 0
Number('0')          // 0
Number(' 1 ')        // 1
Number('1.5')        // 1.5
Number('0x1F')       // 31
Number('abc')        // NaN
Number(undefined)    // NaN
Number(null)         // 0
Number([])           // 0
Number([1])          // 1
Number([1, 2])       // NaN
Number({})           // NaN
```

### Boolean() coercion

```javascript
Boolean('')           // false
Boolean('false')      // true — non-empty string
Boolean(0)            // false
Boolean(-1)           // true — non-zero number
Boolean([])           // true
Boolean({})           // true
Boolean(null)         // false
Boolean(undefined)    // false
Boolean(NaN)          // false
```

> **Follow-up:** *You're building an API that receives JSON. A field `count` can be `0` or `null`. Why does `if (!body.count)` break?* Because `0` is falsy. The check treats a valid `count: 0` as if it were missing. Use `body.count === null` or `body.count !== undefined && body.count !== null`.

> **Follow-up:** *How do you compare floating point numbers safely in JavaScript?* Never use `===`. Use an epsilon: `Math.abs(a - b) < Number.EPSILON`. Better, use integer cent-values for money or a library like decimal.js.

---

## 3. Scope & Hoisting

### Lexical Scopes

JavaScript has three main types of scope:

1. **Global scope** — variables declared outside any function or block.
2. **Function scope** — variables declared with `var` inside a function.
3. **Block scope** — variables declared with `let` and `const` inside `{ }`.

```javascript
const globalVar = 'global';

function outer() {
  const outerVar = 'outer';

  function inner() {
    const innerVar = 'inner';
    console.log(globalVar, outerVar, innerVar); // all accessible
  }

  inner();
  console.log(innerVar); // ReferenceError — not in scope
}

outer();
```

### `var` vs `let` vs `const`

| Declaration | Scope | Hoisting | Reassignable | TDZ |
|-------------|-------|----------|--------------|-----|
| `var` | Function | Yes (initialized `undefined`) | Yes | No |
| `let` | Block | Yes (uninitialized) | Yes | Yes |
| `const` | Block | Yes (uninitialized) | No | Yes |

### Hoisting Mechanics

**`var`** — declaration is hoisted to the top of its function scope, initialized with `undefined`.

```javascript
console.log(a); // undefined — not ReferenceError
var a = 5;

// The compiler sees this as:
var a;
console.log(a); // undefined
a = 5;
```

**`let` and `const`** — declaration is hoisted but **not initialized**. Accessing them before the declaration throws `ReferenceError` because they are in the **Temporal Dead Zone (TDZ)**.

```javascript
console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 10;
```

```javascript
// TDZ zone
let x = 'outer';

{
  console.log(x); // ReferenceError — TDZ for x in this block
  let x = 'inner';
}
```

> **Trap:** `var` hoisting with `undefined` vs `let` TDZ throwing `ReferenceError`. The variable exists in both cases (hoisted), but `var` initializes to `undefined` while `let`/`const` remain uninitialized until the declaration is reached.

```javascript
// var: silent undefined — hides bugs
function fn() {
  console.log(value); // undefined — no error, might mask missing initialization
  var value = 10;
}

// let: loud ReferenceError — fails fast
function fn() {
  console.log(value); // ReferenceError
  let value = 10;
}
```

### Function Hoisting

Function **declarations** are fully hoisted (both name and body):

```javascript
sayHi(); // 'Hello!'

function sayHi() {
  console.log('Hello!');
}
```

Function **expressions** are not:

```javascript
sayHi(); // TypeError: sayHi is not a function

var sayHi = function() {
  console.log('Hello!');
};

// With let:
sayHi2(); // ReferenceError
let sayHi2 = () => console.log('Hello!');
```

### Block Scope and `var` in loops — the classic leak

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // 3, 3, 3
}

// i is hoisted to the enclosing function/global scope
// By the time callbacks run, i is 3
```

```javascript
// Fix 1: let (block scope)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // 0, 1, 2
}

// Fix 2: IIFE (pre-ES6)
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100); // 0, 1, 2
  })(i);
}
```

> **Follow-up:** *Why does `let` fix the loop closure problem?* Each iteration creates a new lexical binding for `i`. The `let` declaration creates a new variable for each iteration because of the per-iteration binding semantics defined in the spec. The closure captures that specific iteration's binding.

---

## 4. Closures

A **closure** is a function that retains access to its lexical scope even when executed outside that scope.

```javascript
function makeCounter() {
  let count = 0;           // closed-over variable

  return function() {
    count += 1;
    return count;
  };
}

const counter = makeCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

The inner function "closes over" the `count` variable. Even after `makeCounter` returns, the returned function keeps a reference to `count` in its closure scope.

### How closures work under the hood

Each function carries a hidden `[[Environment]]` reference that points to the scope where it was created. When the function executes, this scope chain is used to resolve variables. If a variable is referenced by a closure, it survives GC even after the outer function returns.

### Practical Use — Module Pattern

```javascript
// Module pattern — data hiding before ES6 modules
const InventoryStore = (function() {
  const items = new Map();  // private state

  return {
    add(sku, qty) {
      items.set(sku, (items.get(sku) || 0) + qty);
    },
    get(sku) {
      return items.get(sku) ?? 0;
    },
    total() {
      let sum = 0;
      for (const qty of items.values()) sum += qty;
      return sum;
    },
  };
})();

InventoryStore.add('WIDGET-A', 10);
InventoryStore.add('WIDGET-A', 5);
console.log(InventoryStore.total()); // 15
console.log(InventoryStore.items);   // undefined — private
```

### Practical Use — Partial Application / Currying

```javascript
function createLogger(level) {
  return function(message) {
    console.log(`[${level.toUpperCase()}] ${message}`);
  };
}

const info = createLogger('info');
const warn = createLogger('warn');

info('Server started');   // [INFO] Server started
warn('Low disk space');   // [WARN] Low disk space
```

### Practical Use — Memoization

```javascript
function memoize(fn) {
  const cache = new Map();

  return function(arg) {
    if (cache.has(arg)) {
      return cache.get(arg);
    }
    const result = fn(arg);
    cache.set(arg, result);
    return result;
  };
}

const expensiveFib = memoize(function fib(n) {
  return n <= 1 ? n : fib(n - 1) + fib(n - 2);
});
```

### Memory Implications

Closures keep references to their closed-over variables. This is the #1 source of memory leaks in Node.js.

```javascript
function createLeak() {
  const hugeData = new Array(1_000_000).fill('x');  // ~8 MB
  const handler = function() {
    // even if this callback doesn't use hugeData,
    // hugeData is in the closure scope → retained!
    console.log('click');
  };

  document.addEventListener('click', handler);
  return handler;
}

// If we never removeEventListener, hugeData stays in memory forever
```

```javascript
// Safer — only close over what you need
function createNoLeak() {
  const hugeData = new Array(1_000_000).fill('x');
  const result = process(hugeData); // extract needed data
  const handler = function() {
    console.log(result); // only result is closed over
  };

  document.addEventListener('click', handler);
  return handler;
}
```

> **Trap:** Closure in a loop with `var` is the classic interview question. What does this print?

```javascript
for (var i = 0; i < 5; i++) {
  setTimeout(function() {
    console.log(i);
  }, i * 1000);
}
// Prints: 5, 5, 5, 5, 5 (one per second)
```

All five closures share the same `i` variable. By the time the timeouts fire, the loop has completed and `i` is `5`.

Fixes:
1. Use `let` (block-scoped per iteration):
   ```javascript
   for (let i = 0; i < 5; i++) {
     setTimeout(() => console.log(i), i * 1000);
   }
   ```

2. IIFE (creates a new scope per iteration):
   ```javascript
   for (var i = 0; i < 5; i++) {
     (function(j) {
       setTimeout(() => console.log(j), j * 1000);
     })(i);
   }
   ```

> **Follow-up:** *When should you NOT use a closure?* In hot paths where thousands of closures are created per second (e.g., per-row processing in a 100K-row CSV). Each closure allocates heap memory. For batch operations, prefer a static function with explicit arguments passed each call, or use class methods.

> **Follow-up:** *How do you debug a closure memory leak in Node.js?* Take a heap snapshot (`node --inspect`), look for unexpected retained size in closures, identify hidden `[[Scopes]]` references in DevTools Memory tab, or use `clinic` / `why-is-node-running` to find retained handles.

---

## 5. `this` Binding

The value of `this` is determined by **how a function is called** (invocation context), not where it is defined. There are exactly 5 rules.

### Rule 1: Default Binding

```javascript
function showThis() {
  console.log(this);
}

showThis(); // global (window in browser, global in Node < ESM)
```

In **strict mode**, default binding gives `undefined`:

```javascript
'use strict';
function showThis() {
  console.log(this); // undefined
}
showThis();
```

Node.js ES modules (`"type": "module"`) are strict by default, so `this` at module top level is `undefined`.

### Rule 2: Implicit Binding

```javascript
const obj = {
  name: 'inventory',
  show() {
    console.log(this.name);
  },
};

obj.show(); // 'inventory' — this = obj

// But detach the method:
const detached = obj.show;
detached(); // undefined (or global) — default binding applies
```

### Rule 3: Explicit Binding (call, apply, bind)

```javascript
function introduce(role) {
  console.log(`${this.name} is a ${role}`);
}

const user = { name: 'Alice' };

introduce.call(user, 'engineer');   // 'Alice is a engineer'
introduce.apply(user, ['engineer']);// 'Alice is a engineer'

const bound = introduce.bind(user, 'engineer');
bound();                            // 'Alice is a engineer'
```

`bind` creates a new function with permanent `this` — subsequent `call`/`apply` cannot override it.

### Rule 4: `new` Binding

```javascript
function Item(sku) {
  this.sku = sku;
  this.createdAt = new Date();
}

const item = new Item('WIDGET-01');
console.log(item.sku); // 'WIDGET-01'

// What new does:
// 1. Create a new empty object (prototype linked to Item.prototype)
// 2. Call Item with this = the new object
// 3. Return the new object (unless constructor returns a non-primitive)
```

### Rule 5: Arrow Function (Lexical `this`)

Arrow functions do **not** have their own `this`. They capture `this` from the enclosing lexical scope (like a variable).

```javascript
const obj = {
  name: 'inventory',
  regular: function() {
    console.log(this.name);
  },
  arrow: () => {
    console.log(this.name);
  },
};

obj.regular(); // 'inventory'   — implicit binding
obj.arrow();   // undefined     — arrow captured global/outer this
```

```javascript
// Practical use: preserving this in callbacks
class InventoryService {
  constructor(items) {
    this.items = items;
  }

  // Before arrow functions — common pattern
  filterLowStockOld(threshold) {
    const self = this; // capture this
    return this.items.filter(function(item) {
      return item.quantity < threshold && item.organizationId === self.orgId;
    });
  }

  // With arrow — lexical this
  filterLowStock(threshold) {
    return this.items.filter((item) => {
      return item.quantity < threshold && item.organizationId === this.orgId;
    });
  }
}
```

### Priority

The priority (highest to lowest):
1. `new` binding
2. Explicit (`call`/`apply`/`bind`)
3. Implicit (object method)
4. Default (global or undefined)

Arrow functions ignore rules 1–4 entirely and use lexical scoping.

### Common Traps

**Trap: Losing `this` in a callback**

```javascript
class ApiClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
  }

  fetch(path) {
    return fetch(`${this.baseUrl}${path}`).then(function(response) {
      // this is undefined (strict) or global — NOT ApiClient instance
      console.log(this.baseUrl); // TypeError: Cannot read properties of undefined
      return response.json();
    });
  }
}
```

Fixes:
```javascript
// 1. Arrow function
fetch(path) {
  return fetch(`${this.baseUrl}${path}`).then((response) => {
    return response.json();
  });
}

// 2. .bind(this)
fetch(path) {
  return fetch(`${this.baseUrl}${path}`).then(function(response) {
    return response.json();
  }.bind(this));
}

// 3. Capture this
fetch(path) {
  const self = this;
  return fetch(`${this.baseUrl}${path}`).then(function(response) {
    return response.json();
  });
}
```

**Trap: Arrow function in object method**

```javascript
const counter = {
  count: 0,
  increment: () => {
    this.count++; // this is NOT counter — it's the outer scope (global/undefined)
  },
};

counter.increment();
console.log(counter.count); // 0 — didn't work
```

**Trap: `this` in event handlers**

```javascript
class Button {
  constructor(label) {
    this.label = label;
    this.element = document.createElement('button');
    this.element.addEventListener('click', this.onClick); // loses this
  }

  onClick() {
    console.log(this.label); // undefined — this = button DOM element
  }
}
```

Fixes:
```javascript
// Arrow function in constructor
this.element.addEventListener('click', (e) => this.onClick(e));

// bind
this.element.addEventListener('click', this.onClick.bind(this));
```

> **Follow-up:** *Can you override the `this` of an arrow function with `call`?* No. Arrow functions ignore `call`, `apply`, and `bind` for `this` binding. `arrowFn.call(otherThis, arg)` still uses the captured lexical `this`.

> **Follow-up:** *What is the `this` value inside a class field arrow function?*

```javascript
class Foo {
  bar = () => this;  // class field — captured at construction
}

const f = new Foo();
console.log(f.bar() === f); // true
```

> Class field arrow functions create a new function per instance (not on the prototype). They capture the instance `this`. This is convenient but uses more memory than method definitions on the prototype.

---

## 6. Prototypes & Inheritance

### `__proto__` vs `prototype`

These are two different things that sit on two different objects:

| Property | Lives on | Purpose |
|----------|----------|---------|
| `.__proto__` | Every object | Points to the object's prototype (the object it inherits from) |
| `.prototype` | Constructor functions (classes) | The object that will become `.__proto__` of instances created with `new` |

```javascript
function Item(sku) {
  this.sku = sku;
}

Item.prototype.getSku = function() {
  return this.sku;
};

const item = new Item('WIDGET-01');

// item.__proto__ === Item.prototype           // true
// Item.prototype.__proto__ === Object.prototype // true
// Object.prototype.__proto__ === null           // true — end of chain
```

### The Prototype Chain

When you access `item.getSku()`:

1. JS checks if `item` has own property `getSku` → no.
2. Follow `item.__proto__` → `Item.prototype` → found `getSku`.
3. Execute it with `this = item`.

```javascript
// Visual chain:
item
  → Item.prototype  (getSku, constructor)
    → Object.prototype (toString, hasOwnProperty, etc.)
      → null

// Method lookup never fails silently — it returns undefined or throws
item.nonExistent     // undefined
item.nonExistent()   // TypeError: item.nonExistent is not a function
```

### Object.create

```javascript
const baseInventory = {
  init(sku, qty) {
    this.sku = sku;
    this.qty = qty;
    return this;
  },
  adjust(amount) {
    this.qty += amount;
  },
};

const item = Object.create(baseInventory);
item.init('WIDGET-01', 10);
item.adjust(5);
console.log(item.qty); // 15
console.log(item.sku); // 'WIDGET-01'
console.log(item.__proto__ === baseInventory); // true
```

`Object.create(null)` creates an object with **no prototype** — no `toString`, no `hasOwnProperty`, no `constructor`. Useful for pure dictionaries:

```javascript
const dict = Object.create(null);
dict['__proto__'] = 'hack';  // no prototype pollution
console.log(dict.toString);  // undefined — no inherited methods
```

### Constructor Functions

```javascript
function InventoryItem(sku, quantity) {
  // this = new object (implicit with new)
  this.sku = sku;
  this.quantity = quantity;

  // return this (implicit with new)
}

// Static property — on the constructor itself
InventoryItem.MIN_QUANTITY = 0;

// Static method
InventoryItem.createDefault = function(sku) {
  return new InventoryItem(sku, 0);
};

// Instance method — on prototype
InventoryItem.prototype.adjust = function(amount) {
  this.quantity += amount;
};
```

### ES6 Classes are Syntactic Sugar

Under the hood, classes still use prototypal inheritance:

```javascript
class InventoryItem {
  static MIN_QUANTITY = 0;

  constructor(sku, quantity = 0) {
    this.sku = sku;
    this.quantity = quantity;
  }

  adjust(amount) {
    this.quantity += amount;
  }

  static createDefault(sku) {
    return new InventoryItem(sku, 0);
  }
}

console.log(typeof InventoryItem); // 'function'
console.log(InventoryItem.prototype.adjust); // [Function: adjust]
```

### Inheritance

```javascript
class PerishableItem extends InventoryItem {
  constructor(sku, quantity, expiryDate) {
    super(sku, quantity); // must call super before using this
    this.expiryDate = expiryDate;
  }

  isExpired() {
    return new Date() > this.expiryDate;
  }

  // Override
  adjust(amount) {
    if (this.isExpired()) {
      throw new Error('Cannot adjust expired item');
    }
    super.adjust(amount); // call parent method
  }
}
```

What `extends` does (simplified):

```javascript
// Object.setPrototypeOf(PerishableItem, InventoryItem); // static inheritance
// Object.setPrototypeOf(PerishableItem.prototype, InventoryItem.prototype); // instance inheritance
```

### `instanceof` Operator

```javascript
item instanceof InventoryItem          // true
item instanceof PerishableItem         // false (if item is InventoryItem)
item instanceof Object                 // true — all objects are instanceof Object
item instanceof Array                  // false
[] instanceof Array                    // true
```

`instanceof` checks the prototype chain. It walks `left.__proto__` and checks if `right.prototype` is found.

> **Trap: `instanceof` across realms (e.g., iframes, different Node.js vm contexts)**

```javascript
// In a browser with an iframe:
const iframeArray = iframe.contentWindow.Array;
const arr = new iframeArray(1, 2, 3);

arr instanceof Array           // false — Array.prototype is different across realms
arr instanceof iframeArray     // true
```

The same issue occurs with Node.js `vm` module. Prefer `Array.isArray()` (works across realms) or `Object.prototype.toString.call(value)`.

> **Trap: Extending Array**

```javascript
class MyArray extends Array {
  first() { return this[0]; }
}

const arr = new MyArray(1, 2, 3);
console.log(arr.first()); // 1
console.log(arr.length);  // 3

// The problem: Array methods return Arrays, not MyArray
const doubled = arr.map(x => x * 2);
console.log(doubled instanceof MyArray); // true — thanks to Symbol.species

// But:
const sliced = arr.slice(1);
console.log(sliced instanceof MyArray); // true (also uses Symbol.species)

// But methods that create plain arrays internally:
const filtered = arr.filter(x => x > 1);
console.log(filtered instanceof MyArray); // true — Symbol.species handles it
```

Actually, built-in methods on subclasses of Array use `Symbol.species` to construct the returned array, but this has edge cases. The general advice: prefer composition over inheritance for arrays — wrap them in a utility class instead.

### Property descriptor and `hasOwnProperty`

```javascript
const item = new InventoryItem('WIDGET', 10);

item.hasOwnProperty('sku');      // true — own property
item.hasOwnProperty('adjust');   // false — on prototype

Object.getOwnPropertyDescriptor(item, 'sku');
// { value: 'WIDGET', writable: true, enumerable: true, configurable: true }
```

### Private fields (ES2022+)

```javascript
class BankAccount {
  #balance = 0; // hard private — not accessible outside class

  deposit(amount) {
    this.#balance += amount;
  }

  getBalance() {
    return this.#balance;
  }
}

const acct = new BankAccount();
acct.deposit(100);
console.log(acct.getBalance()); // 100
console.log(acct.#balance);     // SyntaxError — truly private
```

`#` private fields are *hard* private (enforced by the engine), unlike TypeScript's `private` which is a compile-time check.

> **Follow-up:** *How do you create truly private state without `#`?* Use WeakMap closures (the module pattern):

```javascript
const balances = new WeakMap();

class Account {
  constructor() {
    balances.set(this, 0);
  }

  deposit(amount) {
    balances.set(this, balances.get(this) + amount);
  }

  getBalance() {
    return balances.get(this);
  }
}
```

> **Follow-up:** *What is `Symbol.species`?* A well-known symbol that lets a subclass control what type is returned by methods like `map`, `filter`, `slice` on built-ins like Array, Map, Set, Promise. If you extend Array and don't override `Symbol.species`, your subclass type is preserved.

---

## 7. Promises

### States

A Promise has exactly one of three states:

```
Pending ──► Fulfilled (resolved with a value)
Pending ──► Rejected  (rejected with a reason)
Fulfilled ──► (cannot transition)
Rejected  ──► (cannot transition)
```

Once settled, a promise is immutable — calling `resolve` or `reject` again has no effect.

```javascript
const promise = new Promise((resolve, reject) => {
  // executor runs synchronously
  if (Math.random() > 0.5) {
    resolve('success');
  } else {
    reject(new Error('failed'));
  }
  resolve('ignored'); // no-op — already settled
});
```

### Chaining

```javascript
fetch('/api/inventory')
  .then(response => response.json())
  .then(data => process(data))
  .catch(error => console.error(error))
  .finally(() => hideSpinner());
```

Each `.then` returns a new promise, enabling chaining. If a `.then` callback returns a value, the next `.then` receives it. If it returns a promise, the chain waits for it to settle.

```javascript
Promise.resolve(1)
  .then(x => x + 1)
  .then(x => Promise.resolve(x + 1))
  .then(x => console.log(x)); // 3
```

### Error Propagation

If any `.then` throws or returns a rejected promise, execution skips to the nearest `.catch` downstream:

```javascript
Promise.resolve(1)
  .then(x => { throw new Error('boom'); })
  .then(x => console.log('never runs'))
  .catch(err => console.log(err.message)) // 'boom'
  .then(x => console.log('after catch')); // runs — catch recovered
```

If a `.catch` itself throws, the rejection propagates further:

```javascript
Promise.reject('err1')
  .catch(err => { throw new Error('err2'); })
  .catch(err => console.log(err.message)); // 'err2'
```

> **Trap: Forgetting to return a promise in a chain**

```javascript
fetch('/api/items')
  .then(response => {
    response.json(); // ← missing return!
  })
  .then(data => {
    console.log(data); // undefined — response.json() returned undefined
    // The .then() above returned undefined, not a promise
    // data is the resolved value of the previous .then, which is undefined
  });
```

```javascript
// Correct:
fetch('/api/items')
  .then(response => response.json()) // implicit return
  .then(data => console.log(data));
```

> **Trap: Nested promises instead of flat chains**

```javascript
// BAD — nested pyramid
fetch('/a')
  .then(a => {
    fetch('/b')
      .then(b => {
        console.log(a, b);
      });
  });

// GOOD — flat chain
fetch('/a')
  .then(a => fetch('/b').then(b => [a, b]))
  .then(([a, b]) => console.log(a, b));

// BETTER — Promise.all for independent requests
const [a, b] = await Promise.all([fetch('/a'), fetch('/b')]);
```

### Static Methods

```javascript
// Promise.all — short-circuits on first rejection
const [users, items] = await Promise.all([
  fetch('/api/users').then(r => r.json()),
  fetch('/api/items').then(r => r.json()),
]);
// If any fails, the whole thing rejects immediately

// Promise.allSettled — waits for all, never short-circuits
const results = await Promise.allSettled([
  fetch('/api/users'),
  fetch('/api/items'),
]);
for (const result of results) {
  if (result.status === 'fulfilled') {
    console.log(result.value);
  } else {
    console.error(result.reason);
  }
}

// Promise.race — settles on the first settled promise (resolve or reject)
const timeout = new Promise((_, reject) =>
  setTimeout(() => reject(new Error('timeout')), 5000)
);
const data = await Promise.race([fetch('/api/items'), timeout]);
// If fetch takes >5s, timeout wins

// Promise.any — settles on the first fulfilled, rejects only if ALL reject
const promises = [
  fetch('/api/mirror-1'),
  fetch('/api/mirror-2'),
  fetch('/api/mirror-3'),
];
const fastest = await Promise.any(promises);
// If all reject → AggregateError with all rejection reasons
```

### Unhandled Rejection

If a promise rejects and there is no `.catch` to handle it, Node.js emits `unhandledRejection`. In Node 15+, this terminates the process by default.

```javascript
// BAD — no handler
Promise.reject(new Error('unhandled'));

// In Node.js:
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // Clean up and exit gracefully
  process.exit(1);
});
```

> **Follow-up:** *Why would you prefer `Promise.allSettled` over `Promise.all`?* When you need to wait for all operations to complete regardless of individual failures — e.g., sending a batch of notifications. With `Promise.all`, one failure loses all results. With `allSettled`, you collect both successes and failures.

---

## 8. Async/Await

`async`/`await` is syntactic sugar over promises. Every `async` function returns a promise.

```javascript
async function fetchItems() {
  const response = await fetch('/api/items');
  const data = await response.json();
  return data;
}

// Equivalent to:
function fetchItems() {
  return fetch('/api/items')
    .then(response => response.json());
}
```

### `await` suspends execution

When the engine hits `await`, it suspends the current function, yielding control back to the event loop. A microtask is queued to resume the function when the awaited promise settles.

```javascript
async function demo() {
  console.log('1: inside async — before await');
  await Promise.resolve();
  console.log('3: inside async — after await (resumed as microtask)');
}

console.log('0: before call');
demo();
console.log('2: after call — demo() returned a pending promise');

// Output:
// 0: before call
// 1: inside async — before await
// 2: after call — demo() returned a pending promise
// 3: inside async — after await (resumed as microtask)
```

### Error Handling

Use `try/catch` to handle rejected promises:

```javascript
async function processItem(id) {
  try {
    const response = await fetch(`/api/items/${id}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    console.error('Failed to process item:', error);
    throw error; // re-throw if caller should handle
  }
}
```

If an `async` function throws without catching, the returned promise rejects:

```javascript
async function fail() {
  throw new Error('boom');
}

fail().catch(err => console.log(err.message)); // 'boom'
```

### Sequential vs Parallel

```javascript
// SEQUENTIAL — slower, each waits for previous
async function processSequential(items) {
  for (const item of items) {
    await processItem(item); // one at a time
  }
}

// PARALLEL — faster, all run concurrently
async function processParallel(items) {
  await Promise.all(items.map(item => processItem(item)));
}

// Controlled concurrency — limit to 5 at a time
async function processBatched(items, concurrency = 5) {
  const results = [];
  for (let i = 0; i < items.length; i += concurrency) {
    const batch = items.slice(i, i + concurrency);
    const batchResults = await Promise.all(batch.map(processItem));
    results.push(...batchResults);
  }
  return results;
}
```

> **Trap: Using `await` in `forEach`** — `forEach` does not wait for async callbacks.

```javascript
// BAD — does NOT work as expected
async function process(items) {
  items.forEach(async (item) => {
    await processItem(item); // fire and forget — forEach ignores the returned promise
  });
  console.log('done'); // runs before any item is processed
}

// GOOD — use for...of
async function process(items) {
  for (const item of items) {
    await processItem(item);
  }
  console.log('done'); // runs after all items processed
}

// GOOD — use Promise.all
async function process(items) {
  await Promise.all(items.map(item => processItem(item)));
  console.log('done'); // runs after all items processed
}
```

> **Trap: Forgetting `try/catch` swallows rejected promises silently**

```javascript
// BAD — if fetch fails, the error is an unhandled rejection
async function load(id) {
  const response = await fetch(`/api/items/${id}`); // throws on network error
  // If fetch throws, the function's promise rejects silently
  // No one catches it → unhandledRejection → process.exit in Node 15+
}

// GOOD
async function load(id) {
  try {
    const response = await fetch(`/api/items/${id}`);
    return await response.json();
  } catch (err) {
    console.error('Load failed:', err);
    throw err; // or return fallback
  }
}
```

### Top-level await (ES2022)

In modules (ESM), you can use `await` outside an async function:

```javascript
// items.mjs (Node with "type": "module" or .mjs)
const response = await fetch('/api/items');
export const items = await response.json();

// This module waits for the fetch to complete before its imports resolve
```

### Async/Await and the Event Loop

```javascript
async function foo() {
  console.log('2');
  await null; // await of a non-thenable — still creates a microtask
  console.log('4');
}

console.log('1');
foo();
console.log('3');

// Output: 1, 2, 3, 4
```

Even `await null` yields the thread — the rest of the function runs as a microtask.

> **Follow-up:** *What happens when you await a non-promise value?* JavaScript wraps it in `Promise.resolve()` and still yields the thread. The rest of the function executes as a microtask. There is no synchronous path past `await`.

> **Follow-up:** *Can you cancel a running async function?* Not natively. There is no built-in cancellation for async functions. You must use an **AbortController** with fetch/streams, or implement a cancellation token pattern:

```javascript
function cancellable(asyncFn) {
  let cancelled = false;

  const promise = new Promise((resolve, reject) => {
    asyncFn({
      onCancelled: () => cancelled = true,
      resolve,
      reject,
    });
  });

  return {
    promise,
    cancel: () => { cancelled = true; },
  };
}
```

---

## 9. Error Handling

### The Error Class and Custom Errors

```javascript
// Built-in error types
new Error('generic');
new SyntaxError('parse error');
new TypeError('wrong type');
new ReferenceError('not defined');
new RangeError('out of range');
new URIError('bad URI');

// Custom error classes
class AppError extends Error {
  constructor(message, code = 'INTERNAL_ERROR', status = 500) {
    super(message);
    this.name = 'AppError';
    this.code = code;
    this.status = status;

    // Restore prototype chain (needed in some V8 versions with transpilers)
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

class NotFoundError extends AppError {
  constructor(resource, id) {
    super(`${resource} with id ${id} not found`, 'NOT_FOUND', 404);
    this.name = 'NotFoundError';
    this.resource = resource;
    this.id = id;
  }
}

class ValidationError extends AppError {
  constructor(errors) {
    super('Validation failed', 'VALIDATION_ERROR', 422);
    this.name = 'ValidationError';
    this.errors = errors;
  }
}
```

### throw anything

You can throw any value, but `throw`ing non-Error objects loses stack traces:

```javascript
throw 'string error';            // works — but no stack trace
throw { code: 500, msg: 'oops' };// works — but no stack
throw new Error('proper');       // correct — has stack trace
```

### `try/catch/finally`

```javascript
try {
  processItem(inventoryItem);
} catch (error) {
  console.error('Failed:', error);
  // Could re-throw, return default, log, etc.
} finally {
  // Always runs — even after return, throw, or break
  closeConnection();
}
```

The `finally` block runs even if the `try` block returns:

```javascript
function example() {
  try {
    return 'from try';
  } finally {
    console.log('finally runs'); // runs before the return value is returned
  }
}

console.log(example());
// Output:
// finally runs
// from try
```

### Async Error Handling

Errors in async functions and promises need special attention:

```javascript
// 1. try/catch in async function
async function safeFetch(url) {
  try {
    const res = await fetch(url);
    return await res.json();
  } catch (err) {
    console.error('Fetch failed:', url, err.message);
    return null;
  }
}

// 2. .catch on promise
async function loadItems() {
  return fetch('/api/items')
    .then(r => r.json())
    .catch(err => {
      console.error('Failed to load items:', err);
      return [];
    });
}

// 3. Global handlers
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
  // Clean up and exit — app is in unknown state
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled rejection:', reason);
  // In Node 15+, the process terminates by default
  // You should still handle this — it indicates a bug
});
```

> **Trap: Catching and not re-throwing**

```javascript
async function getItemCount() {
  try {
    const items = await fetchItems();
    return items.length;
  } catch (error) {
    // BUG: caught and swallowed — caller gets undefined instead of rejecting
    console.error(error);
  }
}

// The function never rejects — it resolves to undefined
const count = await getItemCount(); // count is undefined if fetchItems fails
// Never know something went wrong
```

Correct pattern: always re-throw or return a sentinel value:

```javascript
async function getItemCount() {
  try {
    const items = await fetchItems();
    return items.length;
  } catch (error) {
    console.error(error);
    throw error; // re-throw so caller can handle
    // Or: return 0; // if a default is safe
  }
}
```

> **Trap: `process.on('uncaughtException')` should never resume normal operation**

```javascript
// WRONG — dangerous
process.on('uncaughtException', (err) => {
  console.error('Recovered from:', err);
  // Continuing is unsafe — app state is corrupted
});

// RIGHT
process.on('uncaughtException', (err) => {
  console.error('Fatal:', err);
  // Log, flush, send alert
  process.exit(1); // exit uncleanly
});
```

The difference: `uncaughtException` means something broke in a way that corrupted state (e.g., `this` references may be stale, handles may be half-closed, stack may be unwound incorrectly). The only safe thing is to exit. For graceful recovery, use process managers (PM2, systemd) that restart.

### Error handling patterns for Express/Fastify

```javascript
// Express async error wrapper
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage
app.get('/api/items/:id', asyncHandler(async (req, res) => {
  const item = await findItem(req.params.id);
  if (!item) {
    throw new NotFoundError('Item', req.params.id);
  }
  res.json(item);
}));

// Centralized error handler (Express)
app.use((err, req, res, next) => {
  const status = err.status || 500;
  const code = err.code || 'INTERNAL_ERROR';
  const message = status === 500 ? 'Internal server error' : err.message;

  console.error(`[${err.name}] ${err.message}`, err.stack);

  res.status(status).json({
    error: { code, message },
  });
});
```

> **Follow-up:** *In a production API, should you send stack traces to the client?* Never in production. Always log the full error server-side and return a sanitized, user-safe message. Stack traces leak implementation details (file paths, dependency versions, code structure) that help attackers.

> **Follow-up:** *Difference between operational and programmer errors?* Operational errors: failed network request, file not found, validation failure — expected, recoverable, handle with retries/fallbacks. Programmer errors: `TypeError: Cannot read property of undefined`, calling a non-function — these are bugs, not runtime conditions. Distinguish them in your error classes and handle differently.

---

## 10. ES6+ Features

### Arrow Functions

```javascript
// Lexical this (covered in §5)
// Implicit return for expression bodies
const double = x => x * 2;
const add = (a, b) => a + b;

// Block body needs explicit return
const process = (items) => {
  return items.filter(i => i.quantity > 0);
};

// Object literal needs wrapping parens
const makeItem = (sku, qty) => ({ sku, qty });

// No arguments object — use rest params
const log = (...args) => console.log(...args);
```

### Destructuring

```javascript
// Object destructuring
const item = { sku: 'WIDGET', quantity: 10, price: 19.99 };
const { sku, quantity } = item;
const { price: unitPrice } = item; // rename
const { tags = [] } = item; // default value

// Nested destructuring
const user = {
  name: 'Alice',
  address: { city: 'NYC', zip: '10001' },
};
const { address: { city, zip } } = user;

// Array destructuring
const [first, second, ...rest] = [1, 2, 3, 4, 5];
// first = 1, second = 2, rest = [3, 4, 5]

// Swapping
let a = 1, b = 2;
[a, b] = [b, a];

// Function parameters
function render({ sku, quantity = 0, price = 0 } = {}) {
  return `${sku}: ${quantity} @ ${price}`;
}

render({ sku: 'ABC', quantity: 5 }); // 'ABC: 5 @ 0'
```

### Spread/Rest Operator

```javascript
// Rest — collects remaining elements into array
function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4); // 10

// Rest in destructuring
const [first, ...others] = [1, 2, 3, 4];

// Spread — expands iterable into elements
Math.max(...[1, 5, 3]); // 5

// Array concatenation
const combined = [...arr1, ...arr2];

// Object spread (ES2018)
const base = { sku: 'ABC', price: 10 };
const updated = { ...base, price: 15 }; // { sku: 'ABC', price: 15 }
const withMeta = { ...base, createdAt: new Date() };

// Shallow clone
const clone = { ...item };

// Remove property with rest
const { secret, ...safeItem } = userItem; // safeItem has everything except secret
```

### Template Literals

```javascript
const name = 'Alice';
const greeting = `Hello, ${name}!`; // 'Hello, Alice!'

// Multi-line strings
const sql = `
  SELECT sku, quantity
  FROM inventory_items
  WHERE organization_id = ${orgId}
`;

// Tagged templates
function sql(strings, ...values) {
  // strings[0] = "\n  SELECT sku, quantity\n  FROM inventory_items\n  WHERE organization_id = "
  // strings[1] = "\n"
  // values[0] = orgId
  // Sanitize values, build query
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] !== undefined ? sanitize(values[i]) : '');
  }, '');
}

const query = sql`SELECT * FROM items WHERE id = ${userInput}`;
// Prevents SQL injection by escaping values
```

### Optional Chaining (`?.`)

```javascript
// Before
const city = user && user.address && user.address.city;

// With optional chaining
const city = user?.address?.city;

// Dynamic property access
const value = obj?.[key];

// Optional function call
const result = callback?.();

// Works on arrays
const firstItem = items?.[0];
```

Short-circuits to `undefined` if any part of the chain is `null` or `undefined`.

### Nullish Coalescing (`??`)

```javascript
const config = {
  retries: 0,
  timeout: null,
};

const retries = config.retries ?? 3;    // 0 — because 0 is not nullish
const timeout = config.timeout ?? 5000; // 5000 — null falls through
```

> **Trap: `??` vs `||`**

`||` returns the right side for **any falsy** value (0, '', false, NaN).
`??` returns the right side only for `null` or `undefined`.

```javascript
const count = 0;
const a = count || 10;  // 10 — because 0 is falsy
const b = count ?? 10;  // 0 — because 0 is not null or undefined

const name = '';
const c = name || 'default'; // 'default' — empty string is falsy
const d = name ?? 'default'; // '' — empty string is not null/undefined

const flag = false;
const e = flag || true;  // true
const f = flag ?? true;  // false — exactly what you wanted
```

This is critical in config handling: `||` would default a valid `0` or `''` to the fallback value. `??` is almost always safer.

### Modules

```javascript
// moduleA.mjs — named exports
export const API_URL = 'https://api.example.com';
export function fetchItems() { /* ... */ }
export class InventoryService { /* ... */ }

// Default export
export default class Logger { /* ... */ }

// moduleB.mjs — imports
import Logger, { API_URL, fetchItems } from './moduleA.mjs';
import * as Inventory from './moduleA.mjs';
// Inventory.API_URL, Inventory.fetchItems, Inventory.default

// Re-export
export { fetchItems } from './moduleA.mjs';
export * from './moduleA.mjs';

// Dynamic import (ES2020)
const module = await import('./moduleA.mjs');
```

#### CommonJS (Node.js, `.cjs` or default without `"type": "module"`)

```javascript
// moduleA.cjs
const API_URL = 'https://api.example.com';
function fetchItems() { /* ... */ }
class InventoryService { /* ... */ }

module.exports = { API_URL, fetchItems, InventoryService };
exports.fetchItems = fetchItems; // module.exports shortcut

// moduleB.cjs
const { API_URL, fetchItems } = require('./moduleA.cjs');
```

#### Key differences

| Feature | ESM | CommonJS |
|---------|-----|----------|
| Syntax | `import`/`export` | `require`/`module.exports` |
| Loading | Static (parse-time), async | Dynamic (runtime), sync |
| Top-level `this` | `undefined` | `exports` |
| Live bindings | Yes — exported values update | No — copies at require time |
| File extensions | `.mjs`, or `.js` with `"type": "module"` | `.cjs`, or `.js` without `"type": "module"` |

> **Trap:** Destructuring `require` can miss live bindings:

```javascript
// counter.cjs
let count = 0;
module.exports = { count, increment: () => count++ };

// consumer.cjs
const { count, increment } = require('./counter.cjs');
increment();
console.log(count); // 0 — count was copied by value at require time

// FIX: access through the exports object
const counter = require('./counter.cjs');
counter.increment();
console.log(counter.count); // 1 — live access through the module object
```

---

## 11. Data Structures

### Map vs Object

| Feature | `Map` | `Object` |
|---------|-------|----------|
| Key types | Any (objects, functions, primitives) | String or Symbol |
| Order | Insertion order | Integer keys first, then insertion order (ES2015+) |
| Size | `map.size` property | Manual tracking |
| Iteration | Direct (`forEach`, `for...of`, `keys()`, `values()`, `entries()`) | `Object.keys()`, `Object.values()`, `Object.entries()` |
| Performance | Optimized for frequent additions/removals | Optimized for static property access |
| Prototype | No prototype chain (safe) | Has `Object.prototype` (can be polluted) |
| JSON | Not serializable | `JSON.stringify` built-in |

```javascript
// Object — good for static shapes, JSON
const item = { sku: 'ABC', quantity: 10 };

// Map — good for dynamic keys, frequent iteration
const inventory = new Map();
inventory.set('WIDGET-A', 100);
inventory.set('WIDGET-B', 50);

console.log(inventory.get('WIDGET-A')); // 100
console.log(inventory.has('WIDGET-C')); // false
console.log(inventory.size);            // 2

inventory.delete('WIDGET-B');
inventory.clear();

// Iteration
for (const [sku, qty] of inventory) {
  console.log(sku, qty);
}

inventory.forEach((qty, sku) => console.log(sku, qty));

// Object keys
const obj = { a: 1, b: 2 };
Object.keys(obj);   // ['a', 'b']
Object.values(obj); // [1, 2]
Object.entries(obj);// [['a', 1], ['b', 2]]
```

### Set

```javascript
const skus = new Set();
skus.add('WIDGET-A');
skus.add('WIDGET-B');
skus.add('WIDGET-A'); // ignored — already present

console.log(skus.size); // 2
console.log(skus.has('WIDGET-A')); // true

skus.delete('WIDGET-C');
skus.clear();

// Array deduplication
const unique = [...new Set([1, 2, 2, 3, 1])]; // [1, 2, 3]

// Set operations (no native union/intersection/difference — must implement)
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);

// Union
const union = new Set([...a, ...b]);                  // {1, 2, 3, 4}

// Intersection
const intersection = new Set([...a].filter(x => b.has(x))); // {2, 3}

// Difference
const difference = new Set([...a].filter(x => !b.has(x))); // {1}
```

### WeakMap and WeakSet

These hold **weak references** to their keys. If no other reference to the key exists, the entry is eligible for garbage collection.

```javascript
// Use case: associating metadata with objects without preventing GC
const metadata = new WeakMap();

class Item {
  constructor(sku) {
    this.sku = sku;
    metadata.set(this, { createdAt: new Date() });
  }

  getMeta() {
    return metadata.get(this);
  }
}

let item = new Item('WIDGET-A');
console.log(item.getMeta()); // { createdAt: ... }
item = null; // item object is GC-able — WeakMap entry is cleaned up automatically
```

Key properties:

| Feature | `Map` | `WeakMap` |
|---------|-------|-----------|
| Key types | Any | Objects only |
| Iterable | Yes | **No** — no iteration, no `size`, no `keys()`, no `values()`, no `entries()` |
| GC impact | Holds references (prevents GC) | Weak references (does not prevent GC) |

> **Trap:** WeakMap keys **must be objects**:

```javascript
const wm = new WeakMap();
wm.set('string', 1); // TypeError: Invalid value used as weak map key
wm.set(42, 1);       // TypeError
wm.set(null, 1);     // TypeError
wm.set({}, 1);       // OK
wm.set([], 1);       // OK
```

> **Trap:** WeakMap is **not iterable** — no `forEach`, no `for...of`, no `size`:

```javascript
const wm = new WeakMap();
wm.set(obj, 'value');
wm.size;             // undefined
wm.keys();           // TypeError: wm.keys is not a function
for (const [k, v] of wm) {} // TypeError: wm is not iterable
```

WeakMap/WeakSet are not enumerable by design — the contents could be partially collected at any time, making iteration meaningless.

Use cases:
- **Private data** (see §6 — WeakMap-based private state)
- **DOM node metadata** (browser): store data on nodes without modifying them
- **Object tagging**: track which objects have been processed

### Typed Arrays

For binary data — working with files, networking, crypto, WebGL:

```javascript
// Create an 8-byte buffer
const buffer = new ArrayBuffer(8);

// Views interpret the buffer
const int32 = new Int32Array(buffer);
int32[0] = 42;
int32[1] = 100;

const uint8 = new Uint8Array(buffer);
console.log(uint8); // Uint8Array(8) [42, 0, 0, 0, 100, 0, 0, 0]
// (little-endian by default on most systems)

// Direct typed arrays (allocates buffer automatically)
const bytes = new Uint8Array(1024);
const floats = new Float64Array(100);

// DataView for mixed types
const view = new DataView(buffer);
view.setInt32(0, 12345, true); // true = little-endian
view.setUint8(4, 255);
console.log(view.getInt32(0, true)); // 12345
```

Typed arrays are used by:
- `fs.read` / `fs.write` (buffer-based)
- `crypto` operations
- Network protocols (WebSocket binary frames)
- File format parsing (images, audio)

---

## 12. Generators & Iterators

### Generator Functions

Generator functions can pause and resume execution. They return a **generator object** that conforms to both the iterable and iterator protocols.

```javascript
function* idGenerator() {
  let id = 1;
  while (true) {
    yield id++;
  }
}

const gen = idGenerator();
console.log(gen.next().value); // 1
console.log(gen.next().value); // 2
console.log(gen.next().value); // 3
// Infinite — never done
```

### The Iterable Protocol

An object is **iterable** if it has a `Symbol.iterator` method that returns an iterator.

```javascript
// Built-in iterables: Array, Set, Map, String, arguments, NodeList, TypedArrays, generators
for (const x of [1, 2, 3]) {}
for (const x of 'hello') {}
for (const [k, v] of new Map()) {}

// Create a custom iterable
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;

    return {
      next() {
        if (current <= end) {
          return { value: current++, done: false };
        }
        return { value: undefined, done: true };
      },
    };
  }
}

for (const n of new Range(1, 5)) {
  console.log(n); // 1, 2, 3, 4, 5
}
```

### Practical Uses

**Lazy evaluation** — processing large datasets without loading everything:

```javascript
function* readLines(chunks) {
  let buffer = '';
  for (const chunk of chunks) {
    buffer += chunk;
    const lines = buffer.split('\n');
    buffer = lines.pop(); // keep incomplete line
    for (const line of lines) {
      yield line;
    }
  }
  if (buffer) yield buffer; // last line if no trailing newline
}

// Usage: process a huge file line by line
const stream = fs.createReadStream('inventory_export.csv', 'utf8');
const lines = readLines(stream);

for (const line of lines) {
  if (line.trim()) processLine(line);
}
```

**Infinite sequences:**

```javascript
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const fib = fibonacci();
console.log(fib.next().value); // 0
console.log(fib.next().value); // 1
console.log(fib.next().value); // 1
console.log(fib.next().value); // 2
console.log(fib.next().value); // 3
// Take first 10
const first10 = [...new Array(10)].map(() => fib.next().value);
```

**Two-way communication — `.next(value)` sends a value back:**

```javascript
function* question() {
  const answer = yield 'What is your name?';
  yield `Hello, ${answer}!`;
}

const gen = question();
console.log(gen.next().value);       // 'What is your name?'
console.log(gen.next('Alice').value);// 'Hello, Alice!'
```

### Async Generators

Combine async with generators for streaming data:

```javascript
async function* fetchPages(url, maxPages = 10) {
  let page = 1;
  while (page <= maxPages) {
    const response = await fetch(`${url}?page=${page}`);
    const data = await response.json();
    yield data;
    if (data.length === 0) break;
    page++;
  }
}

// Usage: for await...of
for await (const page of fetchPages('/api/items')) {
  for (const item of page) {
    process(item);
  }
}
```

### `for...of` vs `for...in`

```javascript
const arr = [10, 20, 30];
arr.customProp = 'hello';

for (const i in arr) {
  console.log(i); // '0', '1', '2', 'customProp' — enumerable properties, including inherited
}

for (const v of arr) {
  console.log(v); // 10, 20, 30 — values from the iterator
}
```

`for...in` iterates **keys** (including inherited). `for...of` iterates **values** via `Symbol.iterator`.

> **Trap:** Generators are **single-use** — once exhausted, you cannot iterate again:

```javascript
function* oneTwo() { yield 1; yield 2; }
const g = oneTwo();
console.log([...g]); // [1, 2]
console.log([...g]); // [] — generator exhausted
```

> **Follow-up:** *What happens when a generator yields a promise?* It's just a yielded value. For consuming async generators, use `for await...of`, which handles each yielded promise automatically (awaits it before continuing).

> **Follow-up:** *How do generators relate to async/await?* The first implementations of async/await were compiled to generator-based state machines (Babel, TypeScript `asyncToGenerator`). `await` is essentially `yield` with automatic promise unwrapping and error propagation.

---

# Part B — Node.js Fundamentals

## 13. Node.js Process Model

### Architecture

```
JavaScript (V8)
    ↓
Node.js Bindings (JS → C++ bridge)
    ↓
libuv (C library)
    ├── Event Loop (main thread)
    ├── Thread Pool (default 4 threads)
    └── OS Async I/O (epoll/kqueue/IOCP)
```

Node.js runs JavaScript on a single thread via the V8 engine. All I/O operations are delegated to **libuv**, which uses the operating system's asynchronous I/O facilities (epoll on Linux, kqueue on macOS, IOCP on Windows). For operations that don't have native async OS support (e.g., `fs` operations, DNS lookups, crypto), libuv uses a **thread pool**.

### The Thread Pool

```javascript
const crypto = require('crypto');
const fs = require('fs');

// These use the thread pool:
crypto.pbkdf2('secret', 'salt', 100_000, 64, 'sha512', (err, key) => {});
fs.readFile('/etc/hosts', (err, data) => {});

// These do NOT (they use OS async I/O):
http.request()    // network — epoll/kqueue/IOCP
process.nextTick  // never blocks
setTimeout        // timer — event loop phase
```

Default thread pool size: `4`. Increase with `UV_THREADPOOL_SIZE`:

```bash
UV_THREADPOOL_SIZE=8 node server.js
```

Setting this higher than the number of CPU cores typically hurts performance due to context switching. But for I/O-heavy operations, you might benefit from a slightly larger pool.

```javascript
// Verify thread pool size
const os = require('os');
const poolSize = parseInt(process.env.UV_THREADPOOL_SIZE || '4', 10);
console.log(`Thread pool: ${poolSize}, CPUs: ${os.cpus().length}`);
```

### The `process` Object

```javascript
// Command-line arguments
// $ node server.js --port 3000 --env production
// process.argv = ['/usr/bin/node', '/app/server.js', '--port', '3000', '--env', 'production']
const args = process.argv.slice(2);

// Using process.argv for simple parsing
function parseArgs(argv) {
  const args = {};
  for (let i = 0; i < argv.length; i += 2) {
    if (argv[i].startsWith('--')) {
      args[argv[i].slice(2)] = argv[i + 1];
    }
  }
  return args;
}

// Better: use util.parseArgs (Node 18.3+)
const { values, positionals } = require('util').parseArgs({
  args: process.argv.slice(2),
  options: {
    port: { type: 'string', default: '3000' },
    env: { type: 'string', default: 'development' },
  },
});

// Environment variables
const env = process.env.NODE_ENV || 'development';
const port = parseInt(process.env.PORT || '3000', 10);

// Exit codes
process.exit(0);   // success
process.exit(1);   // general error
// Standard exit codes:
// 0  — success
// 1  — uncaught fatal exception
// 2  — unused by Node (bash reserved)
// 3  — internal JavaScript parse error
// 4  — internal JavaScript evaluation failure
// 5  — fatal error (should not happen)
// 6  — non-function internal exception handler
// 7  — internal exception handler run-time failure
// 8  — uncaught exception (before 0.8)
// 9  — invalid argument
// 10 — internal JavaScript run-time failure
// 12 — invalid debug argument
// 13 — unfinished top-level await (≥14)
// 128+SIGTERM — signal exit

// Platform
console.log(process.platform); // 'linux', 'darwin', 'win32'
console.log(process.arch);     // 'x64', 'arm64'

// Memory usage
console.log(process.memoryUsage());
// {
//   rss: 30_000_000,      // Resident Set Size — total memory allocated
//   heapTotal: 12_000_000, // V8 heap total
//   heapUsed: 8_000_000,   // V8 heap used
//   external: 1_000_000,   // C++ objects (Buffers, etc.)
//   arrayBuffers: 500_000, // ArrayBuffer memory
// }

// CPU usage
const startUsage = process.cpuUsage();
// ... do work ...
const diff = process.cpuUsage(startUsage);
// { user: 123000, system: 45000 } — microseconds

// Process signals
process.on('SIGTERM', () => {
  console.log('Shutting down gracefully...');
  server.close(() => process.exit(0));
});

process.on('SIGINT', () => {
  console.log('Interrupted — exiting');
  process.exit(0);
});
```

### Platform-specific behavior

```javascript
if (process.platform === 'win32') {
  // Windows: different path separator, different signal handling
  // Use path.win32 for Windows paths
} else {
  // Unix: proper SIGTERM, SIGUSR1, etc.
}

// EOL (end of line)
const os = require('os');
console.log(os.EOL); // '\n' on Linux/macOS, '\r\n' on Windows
```

### process.cwd() vs __dirname

```javascript
console.log(process.cwd());  // working directory where node was started
console.log(__dirname);      // directory of the current module file
console.log(__filename);     // full path of the current module file
```

> **Trap: Blocking the event loop**

```javascript
// BAD — JSON.parse on a huge payload blocks the event loop
app.post('/api/import', (req, res) => {
  // If body is 1GB, this blocks the loop for seconds
  const data = JSON.parse(req.body);

  res.json({ processed: true });
});

// BETTER — stream parse or limit body size
const express = require('express');
app.use(express.json({ limit: '10mb' }));

// For huge payloads, use streaming JSON parser
const { createReadStream } = require('fs');
// or: use streaming body parser + JSONStream

// BAD — sync crypto blocks
function hashPassword(password) {
  const crypto = require('crypto');
  return crypto.createHash('sha256').update(password).digest('hex');
  // For a single call this is fine. For 10K concurrent requests, it blocks
}

// BETTER — async version uses thread pool
const { promisify } = require('util');
const pbkdf2 = promisify(crypto.pbkdf2);

async function hashPassword(password) {
  const key = await pbkdf2(password, 'salt', 100_000, 64, 'sha512');
  return key.toString('hex');
}
```

> **Trap: CPU-bound operations block the event loop**

```javascript
// This blocks the event loop for the entire duration
function processLargeArray(items) {
  for (const item of items) {
    // Heavy computation
    const result = expensiveCalculation(item);
    results.push(result);
  }
}

// Solution 1: chunk and yield with setImmediate
function processChunks(items, chunkSize = 100) {
  return new Promise((resolve) => {
    const results = [];
    let index = 0;

    function processChunk() {
      const end = Math.min(index + chunkSize, items.length);
      for (; index < end; index++) {
        results.push(expensiveCalculation(items[index]));
      }
      if (index < items.length) {
        setImmediate(processChunk); // yield to event loop
      } else {
        resolve(results);
      }
    }

    processChunk();
  });
}

// Solution 2: Worker Threads (for CPU-bound parallel work)
const { Worker } = require('worker_threads');
```

> **Trap: `crypto` synchronous methods**

```javascript
// AVOID — blocks event loop
crypto.randomBytes(65536); // sync — blocks!
crypto.pbkdf2Sync('secret', 'salt', 1_000_000, 64, 'sha512'); // blocks!

// PREFER — async versions
crypto.randomBytes(65536, (err, buf) => {});
crypto.pbkdf2('secret', 'salt', 1_000_000, 64, 'sha512', (err, key) => {});
```

---

## 14. Event Loop Phases

Node.js's event loop is implemented by **libuv** and has distinct phases. Each phase has a **FIFO queue** of callbacks to execute.

### The Six Phases

```
   ┌───────────────────────────┐
┌─>│         timers            │ ← setTimeout/setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │ ← I/O callbacks deferred to next iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │ ← internal use
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │ ← NEW I/O events, run callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │ ← setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │ ← close event callbacks (socket.on('close'))
│  └───────────────────────────┘
└────────────────────────────────
```

### Phase Details

**1. Timers** — Executes callbacks scheduled by `setTimeout` and `setInterval`.

```javascript
setTimeout(() => console.log('timeout'), 0);
// Actually fires after the poll phase — minimum delay is 1ms even with 0
```

**2. Pending Callbacks** — I/O callbacks deferred from the previous iteration (e.g., `TCP error` callbacks).

**3. Idle, Prepare** — Used internally by libuv. Not accessible from JavaScript.

**4. Poll** — The most important phase. It:
- Watches for new I/O events (file reads, network data).
- Executes I/O callbacks if any are ready.
- If no I/O callbacks and the queue is empty:
  - If there are `setImmediate` timers in the check queue → skip to check phase.
  - If there are timers expiring → skip to timers phase.
  - Otherwise, block and wait for I/O events.

**5. Check** — Executes `setImmediate` callbacks.

**6. Close Callbacks** — Executes `close` event callbacks (e.g., `socket.on('close')`).

### `setImmediate` vs `setTimeout(fn, 0)`

```javascript
// Order depends on which phase the event loop is in when these are called:

// Case 1: In the main module (poll phase not yet started)
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// Output: ambiguous — order varies

// Case 2: Inside an I/O callback (poll phase)
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// Output: 'immediate' always before 'timeout'
// Because: the I/O callback runs in poll. After poll, check phase runs (setImmediate).
// Then timers phase runs on the next iteration.
```

### `process.nextTick` — The Microtask Extension

`process.nextTick` is **not** part of the event loop. It runs between each phase, immediately after the current operation completes.

```
Event Loop Phase
    ↓ (after each phase)
process.nextTick queue — drained entirely
    ↓
Microtask queue (Promise callbacks) — drained entirely
    ↓
Next Event Loop Phase
```

```javascript
console.log('1: start');

setTimeout(() => console.log('2: timeout'), 0);
setImmediate(() => console.log('3: immediate'));

Promise.resolve().then(() => console.log('4: promise'));

process.nextTick(() => console.log('5: nextTick'));

console.log('6: end');

// Output:
// 1: start
// 6: end
// 5: nextTick     ← runs between the current phase and the next
// 4: promise      ← microtask queue (same priority as nextTick in practice, though spec differs)
// 2: timeout
// 3: immediate
```

Note: In Node.js, `process.nextTick` callbacks run **before** promise microtasks within the same "next tick queue" step, though the exact interleaving with microtasks can be implementation-specific. The key point: both run between event loop phases.

> **Trap: `process.nextTick` recursion starves I/O**

```javascript
function recursiveTick() {
  process.nextTick(recursiveTick);
  // This prevents the event loop from ever reaching the poll phase
  // No I/O, no timers, no network — event loop is stuck in nextTick loop
}
recursiveTick();

setTimeout(() => console.log('never runs'), 1000);
// The timer callback is queued but never delivered
```

```javascript
// Same problem with promise microtask recursion
function recursivePromise() {
  Promise.resolve().then(recursivePromise);
}
recursivePromise();
// Same effect — I/O starved
```

> **Trap: `setImmediate` vs `nextTick` vs `setTimeout(fn, 0)`**

| API | When it runs | Use case |
|-----|-------------|----------|
| `process.nextTick` | After current phase, before next phase | "I need to run this callback after the current operation, before any I/O" — e.g., handling errors, cleaning up |
| `setTimeout(fn, 0)` | Next timer phase (minimum 1ms delay) | Delaying work to the next iteration |
| `setImmediate` | Check phase (after poll) | "I need to run this callback after current I/O is done" — typical use: yielding to event loop |

```javascript
// When to use what:
// process.nextTick — for scheduling a callback after the current operation
//   (API: "run this after the current handler but before any I/O")
//   Risk: can starve I/O if recursive

// setImmediate — for yielding the event loop during CPU-heavy work
//   (API: "run this after current I/O events are processed")
//   Safer than nextTick for breaking up work

// setTimeout(fn, 0) — for deferring to the next timer cycle
//   (minimum 1ms delay, imprecise timing)
```

> **Follow-up:** *What is the actual resolution of `setTimeout(fn, 0)`?* Minimum 1ms delay. If the timer phase arrives and <1ms has elapsed, the timer callback is not yet ready and the loop continues to the next phase. The callback fires on the next timer phase. `setImmediate` has no such minimum — it fires on the very next check phase.

> **Follow-up:** *How would you implement a `wait(ms)` that doesn't block?*

```javascript
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Not precise — but non-blocking
await sleep(100);
```

---

## 15. Modules

### CommonJS (`require`)

```javascript
// math.js
const PI = 3.14159;
function add(a, b) { return a + b; }

// Attach to exports
exports.PI = PI;
exports.add = add;

// OR: replace module.exports entirely
module.exports = { PI, add };

// OR: assign to a single value
module.exports = class Calculator { /* ... */ };
```

```javascript
// app.js
const math = require('./math');
console.log(math.PI);     // 3.14159
console.log(math.add(2, 3)); // 5

// Destructuring — note: copies values, not live bindings
const { add } = require('./math');
```

### Module Resolution Algorithm

When you call `require('x')`:

1. **Core modules** — check `http`, `fs`, `path`, `crypto`, etc. (highest priority)
2. **Relative path** — `'./foo'` or `'../foo'` → check `foo.js`, `foo.json`, `foo/index.js`
3. **`node_modules` walk** — look in `node_modules/x` in current directory, then parent, etc.
4. **Not found** → throw `MODULE_NOT_FOUND`

```javascript
// File extension resolution order (without extension):
// 1. .js
// 2. .json (JSON.parse)
// 3. .node (compiled addon)
// 4. .mjs (with --experimental-require-module or Node 22+)
```

### Module Caching

Modules are cached after the first `require`. Subsequent `require` calls return the **same** module instance (singleton by default):

```javascript
// counter.js
let count = 0;
module.exports = {
  increment: () => ++count,
  getCount: () => count,
};

// a.js
const c = require('./counter');
c.increment(); // count = 1

// b.js
const c = require('./counter');
console.log(c.getCount()); // 1 — same module, cached

// To invalidate the cache:
delete require.cache[require.resolve('./counter')];
```

### Circular Dependencies

Node.js handles circular `require` by returning the **partially initialized** module.exports:

```javascript
// a.js
console.log('a.js: start');
const b = require('./b');
console.log('a.js: after require b, b.name =', b.name);
module.exports = { name: 'module-a' };
console.log('a.js: end');

// b.js
console.log('b.js: start');
const a = require('./a');
console.log('b.js: after require a, a.name =', a.name);
module.exports = { name: 'module-b' };
console.log('b.js: end');

// main.js
require('./a');
// Output:
// a.js: start
// b.js: start
// b.js: after require a, a.name = undefined   ← a's exports not yet ready!
// b.js: end
// a.js: after require b, b.name = module-b
// a.js: end
```

> **Trap:** Circular dependencies produce partially-initialized modules. In `b.js`, `a` exports are `undefined` because `a.js` hasn't finished its `module.exports` assignment yet. This is a design smell — refactor to extract the shared dependency into a third module.

### ESM (`import`/`export`)

```javascript
// Named exports
export const API_URL = 'https://api.example.com';
export function fetchItems() {}
export class Service {}

// Default export
export default class Logger {}

// Re-export
export { fetchItems } from './items.mjs';

// Import all
import * as Items from './items.mjs';

// Import with alias
import { fetchItems as getItems } from './items.mjs';
```

### ESM vs CJS Differences

```javascript
// ESM has no __dirname, __filename, require, module, or exports
// Alternatives:
import { fileURLToPath } from 'url';
import { dirname } from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// Dynamic imports in ESM
const module = await import('./dynamic-module.mjs');

// JSON imports (ESM)
import data from './data.json' assert { type: 'json' };
// Node 22+: import data from './data.json' with { type: 'json' };
```

### Package.json Fields

```json
{
  "name": "inventory-service",
  "version": "1.0.0",
  "main": "dist/index.js",          // CJS entry point (require)
  "module": "dist/index.mjs",       // ESM entry point (bundlers)
  "exports": {                      // Conditional exports (modern, overrides main)
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./utils": {
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.js"
    }
  },
  "type": "module",                 // "module" = ESM by default, "commonjs" = CJS
  "types": "dist/index.d.ts",       // TypeScript types
  "engines": { "node": ">=18.0.0" }
}
```

> **Trap:** `type: "module"` in package.json makes `.js` files use ESM. `.cjs` always forces CommonJS, `.mjs` always forces ESM. Mixing CJS `require()` in an ESM file throws `ReferenceError: require is not defined`.

> **Trap:** Destructuring `require` can miss live bindings:

```javascript
// counter.cjs
let count = 0;

module.exports = {
  get count() { return count; }, // getter — live access
  increment() { count++; },
};

// app.cjs
const { count, increment } = require('./counter.cjs');
increment();
console.log(count); // 0 — count is a copy, even though module.exports uses getter
// The destructuring calls the getter ONCE at require time and stores the resulting value

// FIX: use the exports object
const counter = require('./counter.cjs');
counter.increment();
console.log(counter.count); // 1 — now the getter is called live each time
```

---

## 16. Buffer & Stream

### Buffer

`Buffer` is a Node.js global for handling **raw binary data**. It was created before JavaScript had `Uint8Array` (ES2015) and is still widely used.

```javascript
// Create buffers
const buf1 = Buffer.alloc(10);          // 10 bytes, zero-filled
const buf2 = Buffer.alloc(10, 0xff);    // 10 bytes, all set to 0xFF
const buf3 = Buffer.from('hello', 'utf8'); // <Buffer 68 65 6c 6c 6f>
const buf4 = Buffer.from([0x48, 0x65, 0x6c, 0x6c, 0x6f]); // from byte array

// Read/write
buf1.writeUInt32BE(0x12345678, 0);  // write big-endian 32-bit int at offset 0
buf1.readUInt32BE(0);              // 0x12345678

// Encoding
const str = buf3.toString('utf8');    // 'hello'
const hex = buf3.toString('hex');     // '68656c6c6f'
const base64 = buf3.toString('base64'); // 'aGVsbG8='

// Slice vs subarray
const original = Buffer.from('hello world');
const slice = original.slice(0, 5);  // new Buffer referencing same memory (no copy!)
slice[0] = 0x4A;                     // modifies original[0] too!
// .subarray() — same behavior (Uint8Array style)

// .copy() — true copy
const copy = Buffer.from(original);
// or
const copy2 = Buffer.alloc(original.length);
original.copy(copy2);
```

> **Trap: `buffer.toString()` without encoding**

```javascript
// Defaults to 'utf8' — usually what you want, but be explicit:
buf.toString('utf8');

// Binary data converted to string and back can lose data
const buf = Buffer.from([0x48, 0x65]);
buf.toString();               // 'He' — works for text

// For non-text data, keep as Buffer or use hex/base64 encoding
```

> **Trap: `slice` shares memory — mutating it mutates the original**

```javascript
const original = Buffer.alloc(10, 0x41);
const slice = original.slice(2, 5);
slice[0] = 0x42;
console.log(original[2]); // 0x42 — original changed!

// If you need an independent copy:
const copy = Buffer.alloc(3);
original.copy(copy, 0, 2, 5);
```

### Buffer pooling

Node.js uses an internal buffer pool for small `Buffer.alloc()` and `Buffer.from()` calls (anything under 4KB). This means `slice` may reference the pool rather than a standalone allocation:

```javascript
// Buffer pooling can cause subtle issues when converting to other types
const buf = Buffer.alloc(10);
const view = new Uint8Array(buf.buffer, buf.byteOffset, buf.byteLength);
// Always use buf.buffer carefully — the underlying ArrayBuffer may be shared
```

### Streams

Streams process data in chunks rather than loading everything into memory. Four types:

| Type | Purpose | Example |
|------|---------|---------|
| `Readable` | Source of data | `fs.createReadStream`, `http.IncomingMessage` |
| `Writable` | Destination for data | `fs.createWriteStream`, `http.ServerResponse` |
| `Transform` | Readable + Writable (modifies data) | `zlib.createGzip`, `crypto.createCipheriv` |
| `Duplex` | Readable + Writable (separate I/O) | `net.Socket`, `child_process.stdout` |

```javascript
// Readable — reading a large file
const fs = require('fs');
const readStream = fs.createReadStream('inventory_export.csv', {
  highWaterMark: 64 * 1024, // 64KB chunks
  encoding: 'utf8',
});

readStream.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes`);
});

readStream.on('end', () => {
  console.log('File fully read');
});

readStream.on('error', (err) => {
  console.error('Stream error:', err);
});
```

### Backpressure

When a readable is faster than a writable, backpressure manages the flow:

```javascript
const readStream = fs.createReadStream('huge-file.csv');
const writeStream = fs.createWriteStream('output.csv');

readStream.on('data', (chunk) => {
  const canContinue = writeStream.write(chunk);

  if (!canContinue) {
    readStream.pause();             // stop reading
    writeStream.once('drain', () => {
      readStream.resume();          // resume when write buffer drains
    });
  }
});

readStream.on('end', () => {
  writeStream.end();
});
```

### `pipe` — Simplified piping

```javascript
readStream.pipe(writeStream);

// `pipe` handles backpressure automatically
// But it does NOT destroy the streams on error — that's a leak risk
```

### `pipeline` — The Safe Way (Node 10+)

```javascript
const { pipeline } = require('stream');
const zlib = require('zlib');

pipeline(
  fs.createReadStream('input.csv'),
  zlib.createGzip(),
  fs.createWriteStream('input.csv.gz'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Pipeline completed');
    }
  }
);

// Promisified (Node 15+)
const { pipeline } = require('stream/promises');
await pipeline(
  fs.createReadStream('input.csv'),
  zlib.createGzip(),
  fs.createWriteStream('input.csv.gz')
);
```

> **Trap: `pipeline` vs `pipe`** — `pipe` does not handle errors properly. If the readable errors, the writable is left open (potential memory leak). If the writable errors, the readable is left streaming (wasted data). `pipeline` cleans up both streams on any error.

### Transform Stream Example

```javascript
const { Transform } = require('stream');

class CsvToJsonTransform extends Transform {
  constructor() {
    super({ readableObjectMode: true, writableObjectMode: true });
  }

  _transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n').filter(Boolean);
    const headers = lines[0].split(',');
    const rows = lines.slice(1).map(line => {
      const values = line.split(',');
      return headers.reduce((obj, header, i) => {
        obj[header.trim()] = values[i]?.trim();
        return obj;
      }, {});
    });
    rows.forEach(row => this.push(row));
    callback();
  }
}

// Usage
const csvStream = fs.createReadStream('items.csv');
const csvToJson = new CsvToJsonTransform();

pipeline(csvStream, csvToJson, async function* (source) {
  for await (const row of source) {
    console.log('Parsed row:', row);
  }
}, (err) => {
  if (err) console.error('Failed:', err);
});
```

### `fs.createReadStream` vs `fs.readFile`

```javascript
// BAD for large files — loads entire file into memory
fs.readFile('huge-file.csv', (err, data) => {
  // data is entire file in memory — OOM risk for files > available RAM
});

// GOOD for large files — streams in chunks
fs.createReadStream('huge-file.csv')
  .pipe(process.stdout);
```

| | `fs.readFile` | `fs.createReadStream` |
|--|---------------|-----------------------|
| Memory | Entire file in RAM | One chunk at a time (default 64KB) |
| Start time | After full read | Immediate (first chunk) |
| Use when | Small files, need whole content in memory | Large files, processing incrementally |

> **Trap: Not handling `error` and `finish` events on streams**

```javascript
// INCOMPLETE — missing error handling
const stream = fs.createReadStream('missing-file.csv');
stream.pipe(writeStream);
// If 'missing-file.csv' doesn't exist, an error is emitted but not caught
// Process may crash, writeStream left open

// COMPLETE
const stream = fs.createReadStream('missing-file.csv');
stream.on('error', (err) => {
  console.error('Read failed:', err);
  writeStream.destroy(err); // clean up the downstream
});

stream.pipe(writeStream);
writeStream.on('finish', () => console.log('Done'));
writeStream.on('error', (err) => console.error('Write failed:', err));
```

> **Follow-up:** *When would you use a Transform stream vs just mapping an array?* For infinite or very large datasets where you can't fit the entire collection in memory. Transform streams let you process data as it arrives — useful for HTTP request bodies, file uploads, CSV processing, and compression pipelines.

> **Follow-up:** *What is `highWaterMark`?* The buffer threshold for streams. For readable streams, it's the maximum number of bytes to buffer internally before `read()` stops reading. For writable streams, it's the high-water mark for the internal buffer — `write()` returns `false` when the buffer exceeds this size (triggering backpressure). Default: 16KB for streams, 64KB for files.

---

## 17. Tier 1 Q&A Drill

Answer each without looking at the section above. Then diff your answer.

**Q1:** Walk through every step that produces `[] == ![] === true`.

**Q2:** What is the difference between `var`, `let`, and `const` in terms of scope, hoisting, and the Temporal Dead Zone? Write a code example that demonstrates a ReferenceError from TDZ.

**Q3:** Explain closures. Then predict the output of:
```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
```
Why does this happen and what are two fixes?

**Q4:** List all 5 rules for `this` binding in JavaScript. For each rule, give a code example. Then explain why arrow functions are different.

**Q5:** What is the prototype chain? How does property lookup work when you write `obj.property`? What is the difference between `obj.__proto__`, `Function.prototype`, and `Object.getPrototypeOf(obj)`?

**Q6:** Trace the output of this code and explain each line:
```javascript
console.log('A');
setTimeout(() => console.log('B'), 0);
Promise.resolve().then(() => console.log('C'));
process.nextTick(() => console.log('D'));
setImmediate(() => console.log('E'));
console.log('F');
```

**Q7:** Why does `await` in a `forEach` callback not behave as expected? Write the correct way to process an array of items sequentially with async/await, and the correct way to process them in parallel.

**Q8:** Explain the difference between `??` and `||`. Give three examples where using `||` instead of `??` would cause a bug.

**Q9:** What is backpressure in Node.js streams? How does `pipe` handle it, and why is `pipeline` from `stream/promises` preferred?

**Q10:** What happens when you require a module that has a circular dependency? Trace through an example with two files that require each other, and explain what each gets at require time.
