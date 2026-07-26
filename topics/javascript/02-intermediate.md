# JavaScript / Node.js — Tier 2: Intermediate (Backend Development)

> **Target Audience**: Senior Backend Engineers (8+ years experience, PHP/Laravel, Go, JS) transitioning to or deepening Node.js backend skills. This tier assumes you're comfortable with JavaScript fundamentals (event loop, async/await, closures, prototypes) and focuses on production backend patterns.

Building backend APIs in Node.js goes far beyond "hello world" Express apps. At this level, you're designing middleware pipelines, choosing the right framework trade-offs, securing endpoints, managing database access patterns, structuring logs, clustering for multi-core, and testing with confidence. This guide covers the patterns that separate production-grade Node.js services from prototypes.

---

## Table of Contents

1. [Express.js Deep Dive](#1-expressjs-deep-dive)
2. [Fastify](#2-fastify)
3. [Middleware Patterns](#3-middleware-patterns)
4. [Authentication & Authorization](#4-authentication--authorization)
5. [Input Validation](#5-input-validation)
6. [Database Access Patterns](#6-database-access-patterns)
7. [Streams & File Handling](#7-streams--file-handling)
8. [Process Management & Clustering](#8-process-management--clustering)
9. [Testing with Jest](#9-testing-with-jest)
10. [Testing with Vitest](#10-testing-with-vitest)
11. [Logging](#11-logging)
12. [Error Handling Patterns](#12-error-handling-patterns)
13. [API Design](#13-api-design)
14. [CORS & Security Headers](#14-cors--security-headers)
15. [Caching & Rate Limiting](#15-caching--rate-limiting)
16. [Tier 2 Q&A Drill](#16-tier-2-qa-drill)

---

## 1. Express.js Deep Dive

Express is the most widely adopted Node.js HTTP framework. Its middleware-based architecture is simple but powerful — and easy to misuse.

### Application vs Router

```javascript
// app.js — Application-level
const express = require('express');
const app = express();

app.use(express.json());
app.get('/health', (req, res) => res.send('OK'));

// Mount a Router
const usersRouter = require('./routes/users');
app.use('/api/users', usersRouter);
```

```javascript
// routes/users.js — Router-level
const { Router } = require('express');
const router = Router();

router.get('/', listUsers);
router.post('/', createUser);
router.get('/:id', getUserById);

module.exports = router;
```

> **Note**: `app` is a singleton. `Router` creates isolated middleware and route stacks. Routers can be nested and mounted at different path prefixes, enabling modular code organization — critical for larger codebases.

### Middleware Signature

```javascript
function myMiddleware(req, res, next) {
  // req — incoming request (IncomingMessage + extensions)
  // res — outgoing response (ServerResponse + extensions)
  // next — fn that passes control to next middleware in the stack
}
```

**What Express adds to `req` and `res`**:

| Extension | Description |
|-----------|-------------|
| `req.params` | Route parameters (`:id ` → `req.params.id`) |
| `req.query` | Parsed query string |
| `req.body` | Parsed request body (after `express.json()` or similar) |
| `req.path` | Pathname of the request |
| `req.ip` | Remote IP address |
| `res.json()` | Sends JSON response |
| `res.status()` | Sets HTTP status code (chainable) |
| `res.send()` | Sends response of various types |

### Request Lifecycle

```
Incoming HTTP Request
  → express.json() / express.urlencoded()   // built-in body parsers
  → cors()                                   // CORS headers
  → morgan()                                 // request logging
  → authMiddleware()                         // authentication
  → validationMiddleware()                   // input validation
  → route handler                            // business logic
  → error handler (if next(err) called)      // centralized error handler
  → HTTP Response
```

### Middleware Ordering

Express executes middleware **top to bottom**. The order you call `app.use()` or `router.use()` is the order they run.

```javascript
// WRONG — CORS after routes
app.use('/api', routes);
app.use(cors()); // cors() never runs for /api routes

// RIGHT — critical middleware first
app.use(cors());
app.use(express.json());
app.use('/api', routes);
app.use(errorHandler);
```

> **Trap**: Middleware order matters for correctness. CORS must be registered before routes that need it. Auth middleware must be applied before protected handlers. Body parsers must come before route handlers that read `req.body`. If you mount routes before a middleware that the routes depend on, that middleware never fires for those routes.

### Serving Static Files

```javascript
app.use('/static', express.static('public'));
// Serves ./public/logo.png at /static/logo.png
```

For API-only services, you rarely need this. It's useful for admin panels, docs, or file upload serving.

### Template Engines

```javascript
app.set('view engine', 'pug');
// res.render('index', { title: 'Home' });
```

> **Context**: For backend API work, template engines are irrelevant — your API returns JSON. Template engines matter if you're doing SSR or building monoliths.

---

## 2. Fastify

Fastify is a high-performance Node.js web framework focused on speed, schema-based serialization, and a robust plugin system.

### Schema-Based Serialization

```javascript
const fastify = require('fastify')({ logger: true });

fastify.get('/user/:id', {
  schema: {
    params: {
      type: 'object',
      properties: { id: { type: 'integer' } }
    },
    response: {
      200: {
        type: 'object',
        properties: {
          id: { type: 'integer' },
          name: { type: 'string' },
          email: { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => {
  const user = await getUser(request.params.id);
  return user;
});
```

> **Trap**: Fastify's schema serialization is strict — properties not defined in the response schema are **silently dropped**. This is by design for performance, but can surprise you if you expect `reply.send(user)` to include all fields. Always declare all fields you need in the response schema.

### Plugins (Encapsulated Context)

```javascript
// plugins/authenticate.js
const fp = require('fastify-plugin');

async function authPlugin(fastify, opts) {
  fastify.decorate('authenticate', async (request, reply) => {
    const token = request.headers.authorization?.split(' ')[1];
    if (!token) throw new Error('Missing token');
    request.user = await verifyToken(token);
  });
}

module.exports = fp(authPlugin); // fp breaks encapsulation
```

Plugins have their own context — decorators, hooks, and routes declared inside a plugin are not visible outside unless wrapped with `fastify-plugin`.

### Hooks

```javascript
fastify.addHook('onRequest', async (request, reply) => {
  // Runs on every request, before schema validation
});

fastify.addHook('preHandler', async (request, reply) => {
  // Runs after schema validation, before route handler
});

fastify.addHook('onSend', async (request, reply, payload) => {
  // Runs before the response is sent — transform payload here
});

fastify.addHook('onError', async (request, reply, error) => {
  // Global error hook
});
```

### Comparison to Express

| Aspect | Express | Fastify |
|--------|---------|---------|
| Performance | ~30k req/s | ~60k+ req/s |
| Serialization | `JSON.stringify()` via `res.json()` | Compiled JSON schema serializers |
| Plugin system | `require()` + manual `app.use()` | Formal encapsulation + decorators |
| Logging | Third-party (morgan, pino-http) | Built-in Pino |
| Validation | Third-party (express-validator, Joi) | Built-in JSON Schema (Ajv) |
| TypeScript | Manual | Native support + TypeBox |
| Community | Massive | Growing, production proven |

> **Trap**: Fastify is *not* a drop-in Express replacement. The request/reply API is different (e.g., `reply.send()` instead of `res.json()`, async handlers are native, error handling via `throw`). You can't just copy-paste Express middleware into Fastify — you need Fastify-compatible plugins.

---

## 3. Middleware Patterns

Middleware is the backbone of Express (and similar frameworks). Understanding the patterns solidifies your architecture.

### Application-Level Middleware

```javascript
app.use(express.json());              // built-in body parser
app.use(cors({ origin: '*' }));       // CORS headers
app.use(morgan('combined'));          // HTTP request logging
app.use(compression());               // gzip response
```

### Router-Level Middleware

```javascript
const router = express.Router();

// Only applies to routes on this router
router.use(authenticate);
router.use(validateTenant);

router.get('/orders', listOrders);
```

### Error-Handling Middleware (4 args)

```javascript
function errorHandler(err, req, res, next) {
  // Express detects error middleware by the 4-parameter signature
  console.error(err);
  res.status(err.status || 500).json({
    error: {
      message: err.message || 'Internal Server Error',
      code: err.code || 'INTERNAL_ERROR'
    }
  });
}
```

> **Trap**: Error-handling middleware must have **exactly 4 parameters**. If you write `function errorHandler(err, req, res)` (missing `next`), Express treats it as regular middleware and never calls it for errors.

### Third-Party Middleware

```javascript
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');
const morgan = require('morgan');

app.use(helmet());                          // security headers
app.use(cors({ origin: process.env.CORS_ORIGIN }));
app.use(compression());                     // gzip/brotli
app.use(morgan('combined'));                // HTTP logging
```

### Custom Middleware Patterns

**Request ID generation**:
```javascript
const { v4: uuidv4 } = require('uuid');

function requestId(req, res, next) {
  req.requestId = req.headers['x-request-id'] || uuidv4();
  res.setHeader('X-Request-Id', req.requestId);
  next();
}
```

**Tenant context**:
```javascript
function tenantContext(req, res, next) {
  const tenant = req.headers['x-tenant-id'];
  if (!tenant) return res.status(400).json({ error: 'Missing tenant' });
  req.tenant = tenant;
  next();
}
```

**Idempotency**:
```javascript
async function idempotency(req, res, next) {
  if (req.method !== 'POST') return next();
  const key = req.headers['idempotency-key'];
  if (!key) return next();

  const existing = await cache.get(`idempotent:${key}`);
  if (existing) return res.status(200).json(existing);

  req.idempotencyKey = key;
  res.on('finish', async () => {
    if (res.statusCode < 500) {
      await cache.set(`idempotent:${key}`, res.body, { ttl: 86400 });
    }
  });
  next();
}
```

**Request logging**:
```javascript
function requestLogger(req, res, next) {
  const start = Date.now();
  res.on('finish', () => {
    logger.info({
      method: req.method,
      url: req.originalUrl,
      status: res.statusCode,
      duration: Date.now() - start,
      requestId: req.requestId
    });
  });
  next();
}
```

**Auth verification**:
```javascript
async function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  try {
    const token = authHeader.split(' ')[1];
    req.user = await verifyToken(token);
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### Async Middleware — The Catch Problem

```javascript
// DANGEROUS — async middleware without catch
app.use(async (req, res, next) => {
  const user = await getUser(req.userId); // if this throws, Express catches nothing
  req.user = user;                         // Unhandled promise rejection crashes the process
  next();
});

// SAFE — explicit try/catch
app.use(async (req, res, next) => {
  try {
    const user = await getUser(req.userId);
    req.user = user;
    next();
  } catch (err) {
    next(err);
  }
});

// CLEANER — wrapper utility
const catchAsync = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

app.use(catchAsync(async (req, res, next) => {
  const user = await getUser(req.userId);
  req.user = user;
  next();
}));
```

> **Trap**: Express 4.x does **not** catch promise rejections in middleware. If your async middleware throws without calling `next(err)`, the error is swallowed and the request hangs until timeout. Always wrap async middleware with a `catchAsync` utility or use Express 5.x (which handles async errors).

> **Trap**: Not calling `next()` at all causes the request to hang indefinitely. Not calling `next(err)` when catching an error silently swallows it.

---

## 4. Authentication & Authorization

### JWT Structure

A JSON Web Token has three parts separated by dots:

```
header.payload.signature
```

**Header** — algorithm and token type:
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload** — claims:
```json
{
  "sub": "user_123",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622,
  "roles": ["admin"]
}
```

**Signature** — prevents tampering. Created by combining encoded header + payload + secret.

### Signing

```javascript
const jwt = require('jsonwebtoken');

// HMAC (symmetric) — same secret signs and verifies
const token = jwt.sign({ sub: userId, roles: ['admin'] }, process.env.JWT_SECRET, {
  expiresIn: '15m',
  algorithm: 'HS256'
});

// RS256 (asymmetric) — private key signs, public key verifies
const token = jwt.sign({ sub: userId }, privateKey, {
  algorithm: 'RS256',
  expiresIn: '15m'
});
```

### Verification

```javascript
try {
  const decoded = jwt.verify(token, process.env.JWT_SECRET, {
    algorithms: ['HS256'] // Always specify — prevents algorithm confusion
  });
  req.user = decoded;
} catch (err) {
  // Token expired, invalid signature, malformed
  return res.status(401).json({ error: 'Invalid token' });
}
```

> **Trap**: Not specifying the `algorithms` option in `jwt.verify()` is a vulnerability. An attacker can change the algorithm in the header to `none` or switch from RS256 to HS256 if they have the public key. Always explicitly set `algorithms`.

### JWT Expiry & Rotation

```javascript
// Short-lived access token (15 min)
const accessToken = jwt.sign({ sub }, secret, { expiresIn: '15m' });

// Long-lived refresh token (7 days)
const refreshToken = jwt.sign({ sub, type: 'refresh' }, refreshSecret, { expiresIn: '7d' });

// Store refresh tokens in DB for revocation
await db.refreshTokens.insert({ token: hashedRefreshToken, userId, expiresAt });
```

> **Trap**: Too-long JWT expiry without revocation capability means compromised tokens remain valid. Use short expiry (15-60 minutes) + refresh tokens. If you must revoke, maintain a blocklist in Redis checked on every request.

### Session-Based Auth

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: true,    // HTTPS only
    sameSite: 'strict',
    maxAge: 24 * 60 * 60 * 1000 // 24h
  }
}));
// Sessions stored server-side, only session ID cookie sent to client
```

### OAuth 2.0 Flows

**Authorization Code + PKCE** (for third-party apps / SPAs):
```
1. Client → Authorization Server: ?response_type=code&code_challenge=SHA256(verifier)
2. Authorization Server → Client: authorization code (redirect)
3. Client → Token Endpoint: code + code_verifier
4. Token Endpoint → Client: access_token + refresh_token
```

**Client Credentials** (service-to-service):
```javascript
// Server requests a token using its client_id + client_secret
const response = await fetch('https://auth.example.com/oauth/token', {
  method: 'POST',
  body: JSON.stringify({
    grant_type: 'client_credentials',
    client_id: process.env.CLIENT_ID,
    client_secret: process.env.CLIENT_SECRET,
    scope: 'api:read api:write'
  })
});
const { access_token } = await response.json();
```

### Scopes vs Roles vs Permissions

| Concept | Description | Example |
|---------|-------------|---------|
| **Scope** | What a token can do (OAuth concept) | `api:read`, `api:write` |
| **Role** | Job function grouping permissions | `admin`, `editor`, `viewer` |
| **Permission** | Granular action access | `post.create`, `post.delete` |

```javascript
// Middleware for protected routes
function authorize(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) return res.status(401).json({ error: 'Unauthenticated' });
    if (!allowedRoles.some(role => req.user.roles?.includes(role))) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

// Usage
app.post('/admin/users', authenticate, authorize('admin'), createUser);
```

> **Trap**: JWT stored in `localStorage` is XSS-vulnerable — any injected script can read it. Use `httpOnly`, `secure`, `sameSite` cookies for browsers. For SPAs, use the BFF (Backend for Frontend) pattern to keep tokens server-side.

---

## 5. Input Validation

### Why Validate Server-Side

- **Injection prevention**: SQLi, NoSQLi, XSS mitigated by type-checking and sanitization
- **Type safety**: Ensures API contracts are honored
- **User feedback**: Meaningful error messages for malformed input

### Zod (Runtime Validation + TypeScript Inference)

```javascript
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email().transform(v => v.toLowerCase()),
  password: z.string().min(8).max(128),
  name: z.string().trim().min(1).max(100),
  role: z.enum(['admin', 'user']).default('user')
});

// TypeScript type inferred from schema — zero duplication
type CreateUserDTO = z.infer<typeof CreateUserSchema>;

// In route handler
app.post('/users', (req, res) => {
  const result = CreateUserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({
      error: 'Validation failed',
      details: result.error.flatten().fieldErrors
    });
  }
  // result.data is fully typed
  await createUser(result.data);
});
```

### Joi

```javascript
const Joi = require('joi');

const schema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).max(128).required(),
  name: Joi.string().trim().min(1).max(100).required()
});

const { error, value } = schema.validate(req.body, { abortEarly: false });
if (error) {
  return res.status(400).json({
    error: error.details.map(d => d.message)
  });
}
```

### express-validator

```javascript
const { body, validationResult } = require('express-validator');

app.post('/users',
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }),
  body('name').trim().isLength({ min: 1 }),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // Proceed
  }
);
```

### Validation Middleware Pattern

```javascript
function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        status: 'error',
        code: 'VALIDATION_ERROR',
        message: 'Invalid request body',
        errors: result.error.flatten().fieldErrors
      });
    }
    req.validatedBody = result.data; // Replace raw body with validated + transformed
    next();
  };
}

app.post('/users', validate(CreateUserSchema), createUser);
```

> **Trap**: Client-side validation is for UX, not security. Always validate server-side. A curl request bypasses your frontend entirely.

> **Trap**: Not normalizing input leads to duplicate users (e.g., "John@Example.com" vs "john@example.com"). Always trim strings and lowercase emails.

> **Trap**: Error message leakage — returning the full Zod/Joi error object may expose schema internals. Map errors to user-safe messages.

---

## 6. Database Access Patterns

### Prisma (Type-Safe ORM)

```javascript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Type-safe query — result is fully typed
const user = await prisma.user.findUnique({
  where: { email },
  include: { posts: true } // Eager loading — prevents N+1
});

// Migration
// $ npx prisma migrate dev --name add_user_table

// Interactive transaction
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({ data: { email } });
  const profile = await tx.profile.create({ data: { userId: user.id } });
  return { user, profile };
});
```

### Knex (Query Builder)

```javascript
const knex = require('knex')({
  client: 'pg',
  connection: process.env.DATABASE_URL,
  pool: { min: 2, max: 10 }
});

// Query builder
const users = await knex('users')
  .join('profiles', 'users.id', 'profiles.user_id')
  .where('users.active', true)
  .select('users.*', 'profiles.bio');

// Transaction
const result = await knex.transaction(async (trx) => {
  const [user] = await trx('users').insert({ email }).returning('*');
  await trx('profiles').insert({ user_id: user.id });
});

// Raw SQL when needed
const [result] = await knex.raw('SELECT * FROM users WHERE email = ?', [email]);
```

### Sequelize (ORM)

```javascript
const { Sequelize, DataTypes } = require('sequelize');
const sequelize = new Sequelize(process.env.DATABASE_URL, {
  pool: { max: 10, min: 2, acquire: 30000, idle: 10000 }
});

const User = sequelize.define('User', {
  email: { type: DataTypes.STRING, allowNull: false, unique: true },
  name: { type: DataTypes.STRING }
});

// Eager loading prevents N+1
const users = await User.findAll({
  include: [{ model: Post, as: 'posts' }]
});
```

### Connection Pooling

```javascript
// pg-pool directly
const { Pool } = require('pg');
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20, // Max connections in pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

// Prisma connection pool
const prisma = new PrismaClient({
  datasources: {
    db: { url: process.env.DATABASE_URL }
  },
  // Prisma manages its own pool internally
  connectionLimit: 10
});
```

> **Trap**: Not managing connection pool size. Each Node.js instance creates its own pool — with clustering, `max: 20` x 4 workers = 80 connections. Set pool size intentionally based on your database's `max_connections` and expected concurrency.

### N+1 Prevention

```javascript
// BAD — N+1 queries
const users = await User.findAll();
for (const user of users) {          // 1 query for users
  const posts = await Post.findAll({ where: { userId: user.id } }); // N queries
}

// GOOD — eager loading (Prisma)
const users = await prisma.user.findMany({
  include: { posts: true }           // 1 query with JOIN or batch
});

// GOOD — batch loading (DataLoader for GraphQL)
const DataLoader = require('dataloader');
const userLoader = new DataLoader(async (ids) => {
  const users = await db('users').whereIn('id', ids);
  return ids.map(id => users.find(u => u.id === id));
});
```

> **Trap**: N+1 is especially dangerous in GraphQL resolvers — each resolver can fire a separate DB query. Always use DataLoader or batched queries.

> **Trap**: Long-running transactions hold database connections and locks. Keep transactions short. Never await external HTTP calls inside a transaction.

---

## 7. Streams & File Handling

### Reading Large Files

```javascript
// BAD — loads entire file into memory
const data = fs.readFileSync('/path/to/large-file.csv'); // 500MB = 500MB RAM

// GOOD — streaming
const rs = fs.createReadStream('/path/to/large-file.csv', {
  highWaterMark: 64 * 1024 // 64KB chunks
});

rs.on('data', (chunk) => {
  console.log('Received', chunk.length, 'bytes');
});

rs.on('end', () => console.log('Done'));
rs.on('error', (err) => console.error('Stream error:', err));
```

### Pipeline (Backpressure-Aware)

```javascript
const { pipeline } = require('stream/promises');
const fs = require('fs');
const zlib = require('zlib');

// Automatically handles backpressure — pauses reads when writes can't keep up
await pipeline(
  fs.createReadStream('input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('input.txt.gz')
);
```

> `pipeline` is preferred over `.pipe()` because it properly cleans up streams on error and handles backpressure.

### Transform Streams

```javascript
const { Transform } = require('stream');

const upperCaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});

await pipeline(
  fs.createReadStream('input.txt'),
  upperCaseTransform,
  fs.createWriteStream('output.txt')
);
```

### File Uploads with Multer

```javascript
const multer = require('multer');

const upload = multer({
  dest: 'uploads/',
  limits: { fileSize: 10 * 1024 * 1024 }, // 10MB
  fileFilter: (req, file, cb) => {
    if (!file.mimetype.startsWith('image/')) {
      return cb(new Error('Only images allowed'), false);
    }
    cb(null, true);
  }
});

app.post('/upload', upload.single('avatar'), (req, res) => {
  // req.file contains file info
  // req.body contains other fields
  res.json({ id: req.file.filename });
});
```

### CSV Parsing (Stream-Based)

```javascript
const { parse } = require('csv-parse');
const fs = require('fs');

const results = [];
await pipeline(
  fs.createReadStream('data.csv'),
  parse({ columns: true, skip_empty_lines: true }),
  async function* (source) {
    for await (const record of source) {
      // Process each row without loading all into memory
      results.push(record);
    }
  }
);
```

> **Trap**: Using `fs.readFile` for large files blocks the event loop and exhausts memory. Always use streams for files > 10MB or when processing unknown sizes.

> **Trap**: Not handling stream `error` events causes unhandled errors that crash the process. Use `pipeline` (which handles errors) or attach `.on('error')` handlers.

> **Trap**: Backpressure ignored causes memory spikes. `pipeline` handles this automatically; `.pipe()` does not in all cases.

---

## 8. Process Management & Clustering

### Cluster Module

```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  const numWorkers = os.cpus().length;
  console.log(`Master ${process.pid} forking ${numWorkers} workers`);

  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died, restarting`);
    cluster.fork(); // Auto-restart
  });
} else {
  // Worker — runs your Express/Fastify app
  const app = require('./app');
  app.listen(3000);
}
```

> **Trap**: The master process should **not** handle requests — it only manages workers. You must call `cluster.fork()` in master and set up your server only in workers.

> **Trap**: Round-robin scheduling (default on most platforms) distributes connections evenly. On Windows, the default is platform-based scheduling. Be aware of the difference when testing.

### PM2

```bash
# Start in cluster mode
pm2 start app.js -i max        # Fork one worker per CPU
pm2 start app.js -i 4          # Exactly 4 workers

# Zero-downtime reload
pm2 reload app                 # Restart workers one by one

# Monitoring
pm2 monit                      # Real-time CPU/memory per worker
pm2 logs                       # Consolidated log stream

# Ecosystem file
```
```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'api',
    script: 'app.js',
    instances: 'max',
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    env_production: {
      NODE_ENV: 'production'
    },
    log_file: '/var/log/app/combined.log',
    error_file: '/var/log/app/error.log',
    max_memory_restart: '500M'
  }]
};
```

### Environment-Based Configuration

```javascript
// dotenv — loads .env file into process.env
require('dotenv').config();

// config module — hierarchical config
const config = require('config');
// config/default.json, config/production.json, config/custom-environment-variables.json
const dbUrl = config.get('database.url');
```

> **Trap**: Inaccurate worker count — `os.cpus().length` may undercount in containerized environments (cgroups limit). Consider using `PM2` with `instances: 'max'` which reads cgroup-aware CPU count, or explicitly set worker count via env variable.

> **Trap**: Sticky sessions with WebSockets — cluster mode distributes connections across workers. WebSocket connections need to stick to the same worker (same process). Solutions: use Redis pub/sub or an external adapter (socket.io-redis), or run WebSocket server separately.

---

## 9. Testing with Jest

Jest is the most popular testing framework in the Node.js ecosystem, offering everything from unit testing to coverage reporting.

### Test Structure

```javascript
describe('User Service', () => {
  beforeAll(async () => {
    await setupTestDatabase();
  });

  afterAll(async () => {
    await teardownTestDatabase();
  });

  beforeEach(() => {
    jest.clearAllMocks(); // Reset mocks between tests
  });

  describe('createUser', () => {
    it('should create a user with valid input', async () => {
      const user = await createUser({ email: 'test@test.com', name: 'Test' });
      expect(user).toHaveProperty('id');
      expect(user.email).toBe('test@test.com');
    });

    it('should reject duplicate emails', async () => {
      await createUser({ email: 'dup@test.com', name: 'A' });
      await expect(createUser({ email: 'dup@test.com', name: 'B' }))
        .rejects.toThrow('Email already exists');
    });
  });
});
```

### Matchers

```javascript
expect(value).toBe(42);            // ===
expect(value).toEqual({ a: 1 });   // deep equality
expect(value).toMatchObject({ a: 1 }); // subset match
expect(array).toContain('item');
expect(fn).toHaveBeenCalledTimes(1);
expect(fn).toHaveBeenCalledWith('arg1', 'arg2');
expect(promise).resolves.toEqual(value);
expect(promise).rejects.toThrow('Error');
```

### Mocking

```javascript
// jest.fn() — simple function mock
const mockFn = jest.fn().mockReturnValue('default');
mockFn('arg');
expect(mockFn).toHaveBeenCalledWith('arg');

// jest.spyOn() — spy on existing methods
const getUserSpy = jest.spyOn(userService, 'getUser');
getUserSpy.mockResolvedValue({ id: 1, name: 'Test' });

// jest.mock() — module-level mocking
jest.mock('ioredis', () => {
  return jest.fn().mockImplementation(() => ({
    get: jest.fn().mockResolvedValue(null),
    set: jest.fn().mockResolvedValue('OK')
  }));
});
```

### Integration Testing with Supertest

```javascript
const request = require('supertest');
const app = require('../app');

describe('POST /api/users', () => {
  it('should return 201 for valid user creation', async () => {
    const res = await request(app)
      .post('/api/users')
      .send({ email: 'test@test.com', password: 'securePass123', name: 'Test' })
      .expect(201);

    expect(res.body).toMatchObject({
      id: expect.any(Number),
      email: 'test@test.com'
    });
  });

  it('should return 400 for invalid email', async () => {
    const res = await request(app)
      .post('/api/users')
      .send({ email: 'not-an-email', password: 'short' })
      .expect(400);

    expect(res.body).toHaveProperty('error');
  });
});
```

### Coverage Thresholds

```javascript
// jest.config.js
module.exports = {
  collectCoverage: true,
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/index.js' // Entry point — hard to unit test
  ]
};
```

> **Trap**: Mock hoisting — `jest.mock()` calls are hoisted to the top of the file by Jest's transpiler. You cannot use variables that are defined after the mock call. If you need dynamic mocks, use `jest.mock(path, factory)` and reference modules, not local variables.

> **Trap**: Not resetting mocks between tests causes test pollution. Use `beforeEach(() => { jest.clearAllMocks(); })` or set `clearMocks: true` in Jest config.

> **Trap**: Testing implementation details (e.g., checking that a private method was called) makes tests brittle. Test behavior and outputs, not internal calls.

---

## 10. Testing with Vitest

Vitest is a blazing-fast test runner powered by Vite. It's Jest-compatible with important differences.

```javascript
import { describe, it, expect, vi, beforeEach } from 'vitest';

// Nearly identical to Jest
vi.mock('ioredis');                        // Like jest.mock()
const spy = vi.spyOn(service, 'method');   // Like jest.spyOn()
const fn = vi.fn();                        // Like jest.fn()
```

### Key Differences vs Jest

| Aspect | Jest | Vitest |
|--------|------|--------|
| ESM support | Experimental, requires `--experimental-vm-modules` | Native ESM out of the box |
| Speed | Slower (Jest CLI overhead) | ~2-10x faster (Vite transforms) |
| Config | `jest.config.js` | `vitest.config.js` (or in `vite.config.js`) |
| Watch mode | `jest --watch` | Built-in, instant HMR-style |
| Mock API | `jest.mock()`, `jest.fn()` | `vi.mock()`, `vi.fn()` |
| TypeScript | Requires `ts-jest` | Native via Vite/esbuild |
| Hanging tests | `--detectOpenHandles` | Built-in `--dangerouslyIgnoreUnhandledErrors` |

```javascript
// vitest.config.js
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,              // Like Jest's globals
    environment: 'node',
    coverage: {
      provider: 'v8',           // or 'istanbul'
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80
      }
    }
  }
});
```

### When to Choose Vitest

- **New projects**: No legacy Jest config migration
- **Vite projects**: Shared transform pipeline, faster dev feedback
- **ESM-only codebases**: Native ESM without flags
- **Speed-sensitive CI**: Faster test runs save money

> **Note**: If your project is deeply integrated with Jest-specific plugins or custom Jest transforms, migrating may not be worth it. For greenfield Node.js API services, Vitest is now the recommended default.

---

## 11. Logging

### Pino (Structured JSON Logging)

```javascript
const pino = require('pino');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV !== 'production'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  redact: ['req.headers.authorization', 'req.body.password', 'user.password']
});

logger.info({ userId: 123, action: 'login' }, 'User logged in');
logger.error({ err, userId: 123 }, 'Failed to process payment');

// Output: {"level":30,"time":...,"pid":123,"hostname":"...","userId":123,"action":"login","msg":"User logged in"}
```

### Winston

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: { service: 'user-api' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});
```

### Morgan (HTTP Request Logging)

```javascript
const morgan = require('morgan');

// Predefined formats: 'combined', 'common', 'dev', 'short', 'tiny'
app.use(morgan('combined'));

// Custom format with Pino
app.use(morgan((tokens, req, res) => {
  logger.info({
    method: tokens.method(req, res),
    url: tokens.url(req, res),
    status: Number(tokens.status(req, res)),
    duration: tokens['response-time'](req, res),
    contentLength: tokens.res(req, res, 'content-length')
  });
}));
```

### Correlation IDs

```javascript
const { v4: uuidv4 } = require('uuid');

// Middleware
function correlationId(req, res, next) {
  req.correlationId = req.headers['x-correlation-id'] || uuidv4();
  res.setHeader('X-Correlation-Id', req.correlationId);
  next();
}

// Child logger with correlation ID
app.use((req, res, next) => {
  req.logger = logger.child({ correlationId: req.correlationId });
  next();
});

// Usage in route handler
app.get('/users/:id', async (req, res) => {
  req.logger.info({ userId: req.params.id }, 'Fetching user');
  // ...
});
```

### Log Levels

```
fatal (60) — service is unusable
error (50) — request failed, needs investigation
warn  (40) — something unexpected but handled
info  (30) — normal operational messages
debug (20) — detailed debugging
trace (10) — finest-grained events
```

> **Trap**: Logging sensitive data (passwords, tokens, credit cards) in plain text is a compliance violation (GDPR, PCI-DSS). Always redact sensitive fields. Pino supports built-in `redact` paths. Never log full request bodies.

> **Trap**: Sync logging (`console.log`, Winston's default console transport) blocks the event loop. Use async logging (Pino is async, Winston file transport is async with streams).

> **Trap**: Excessive logging at `info` level in high-throughput routes degrades performance. Keep hot-path logging at `warn` or higher. Use `debug` for verbose output.

---

## 12. Error Handling Patterns

### Centralized Error Handler Middleware

```javascript
// error-handler.js
function errorHandler(err, req, res, next) {
  const status = err.status || err.statusCode || 500;
  const code = err.code || 'INTERNAL_ERROR';

  // Log with full context
  req.logger?.error({ err, status, code }, 'Request failed');

  res.status(status).json({
    status: 'error',
    code,
    message: err.message || 'An unexpected error occurred',
    timestamp: new Date().toISOString(),
    requestId: req.requestId
  });
}

// Register LAST
app.use(errorHandler);
```

### Custom Error Classes

```javascript
class AppError extends Error {
  constructor(message, status = 500, code = 'INTERNAL_ERROR') {
    super(message);
    this.name = this.constructor.name;
    this.status = status;
    this.code = code;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

class ValidationError extends AppError {
  constructor(message = 'Validation failed') {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED');
  }
}
```

### Async Handler Wrapper

```javascript
const catchAsync = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage
app.get('/users/:id', catchAsync(async (req, res) => {
  const user = await findUserOrFail(req.params.id);
  res.json(user);
}));
```

### Consistent Error Response Format

```javascript
// Always return
{
  "status": "error",
  "code": "VALIDATION_ERROR",
  "message": "Invalid email format",
  "timestamp": "2026-07-26T12:00:00.000Z",
  "requestId": "abc-123"
}

// Success
{
  "status": "success",
  "data": { ... }
}
```

### Unhandled Rejection Handler

```javascript
process.on('unhandledRejection', (reason, promise) => {
  logger.fatal({ err: reason }, 'Unhandled Rejection — shutting down');
  process.exit(1); // Exit and let PM2/docker restart
});

process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught Exception — shutting down');
  process.exit(1);
});
```

> **Trap**: Not differentiating between **operational errors** (expected: validation errors, 404s) and **programmer errors** (bugs: `TypeError` of undefined, `ReferenceError`). Operational errors should be handled gracefully. Programmer errors should crash the process (then restart by PM2/docker).

> **Trap**: Throwing non-Error objects (`throw "string"` or `throw { code: 500 }`) breaks error handling because they lack a stack trace and proper prototype. Always `throw new Error(...)` or `throw new AppError(...)`.

> **Trap**: Catching everything and swallowing (`try { ... } catch {}` with no action) makes debugging impossible. Always at minimum log the error or pass it to `next(err)`.

---

## 13. API Design

### RESTful Resource Naming

```
GET    /users              — list users
POST   /users              — create user
GET    /users/:id          — get user by id
PATCH  /users/:id          — partial update
DELETE /users/:id          — delete user
GET    /users/:id/orders   — nested resource (user's orders)
POST   /users/:id/orders   — create order for user
```

**Conventions:**
- Plural nouns (`/users`, not `/user` or `/getUsers`)
- Hyphens for multi-word (`/line-items`, not `/lineItems` or `/line_items`)
- Lowercase
- No file extensions (`/users`, not `/users.json`)

### HTTP Methods & Status Codes

| Method | Action | Success | Validation Error | Auth Error | Not Found |
|--------|--------|---------|-----------------|------------|-----------|
| GET | Read | 200 | — | 401 | 404 |
| POST | Create | 201 | 400 | 401 | — |
| PUT | Full replace | 200 | 400 | 401 | 404 |
| PATCH | Partial update | 200 | 400 | 401 | 404 |
| DELETE | Remove | 204 | — | 401 | 404 |

### Versioning

```javascript
// URL path versioning (most common)
app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);

// Header versioning
app.use((req, res, next) => {
  const version = req.headers['accept-version'];
  req.apiVersion = version || '1';
  next();
});
```

> **Trap**: Not versioning from day 1. Once you have external consumers, changing API contracts breaks clients. Add `/v1/` to the URL from the start, even if it's the only version. You can always add a redirect from `/api` if needed for internal convenience.

### Pagination

```javascript
// Offset pagination (simple, but inconsistent for real-time data)
// GET /users?offset=0&limit=20
app.get('/users', async (req, res) => {
  const offset = parseInt(req.query.offset) || 0;
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  const [users, total] = await Promise.all([
    prisma.user.findMany({ skip: offset, take: limit }),
    prisma.user.count()
  ]);
  res.json({
    data: users,
    pagination: { offset, limit, total, hasMore: offset + limit < total }
  });
});

// Cursor pagination (preferred for real-time data)
// GET /users?cursor=eyJpZCI6MX0&limit=20
app.get('/users', async (req, res) => {
  const cursor = req.query.cursor ? Buffer.from(req.query.cursor, 'base64').toString() : null;
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);

  const users = await prisma.user.findMany({
    take: limit + 1,
    ...(cursor ? { cursor: { id: parseInt(cursor) }, skip: 1 } : {}),
    orderBy: { id: 'asc' }
  });

  const hasMore = users.length > limit;
  if (hasMore) users.pop();

  res.json({
    data: users,
    pagination: {
      nextCursor: hasMore ? Buffer.from(String(users[users.length - 1].id)).toString('base64') : null,
      hasMore
    }
  });
});
```

### Filtering & Field Selection

```
GET /users?role=admin&status=active         — equality filter
GET /users?createdAfter=2026-01-01          — range filter
GET /users?fields=id,name,email             — field selection
GET /users?sort=-createdAt,name             — sort (desc: -prefix)
GET /users?search=john                      — full-text search
```

### Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,                    // limit per window
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: 'Too many requests, please try again later' }
});

app.use('/api/', limiter);
```

### HATEOAS Basics

```javascript
app.get('/users/:id', async (req, res) => {
  const user = await findUser(req.params.id);
  res.json({
    ...user,
    _links: {
      self: { href: `/api/v1/users/${user.id}` },
      orders: { href: `/api/v1/users/${user.id}/orders` },
      update: { href: `/api/v1/users/${user.id}`, method: 'PATCH' },
      delete: { href: `/api/v1/users/${user.id}`, method: 'DELETE' }
    }
  });
});
```

> **Trap**: Using 200 for everything — even creation (should be 201), validation errors (should be 400), and unauthorized (should be 401). Proper status codes make API consumption easier for clients. Inconsistency in error formats across endpoints makes client error-handling logic brittle.

---

## 14. CORS & Security Headers

### CORS Configuration

```javascript
const cors = require('cors');

// Permissive — development only
app.use(cors());

// Specific — production
app.use(cors({
  origin: [
    'https://myapp.com',
    'https://admin.myapp.com',
    /\.myapp\.com$/    // regex for subdomains
  ],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-Id'],
  exposedHeaders: ['X-Request-Id', 'X-RateLimit-Remaining'],
  credentials: true,    // Allow cookies/auth headers
  maxAge: 86400         // Cache preflight for 24h
}));
```

### Preflight Requests

For "complex" requests (non-simple methods, custom headers, or non-simple content types like `application/json`), browsers send an OPTIONS preflight request:

```
OPTIONS /api/users
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type
```

Your server must respond with appropriate CORS headers. The `cors` middleware handles this automatically.

> **Trap**: Reflecting origin without validation — `origin: true` or `cors({ origin: req.headers.origin })` allows any site to make requests. Always validate against a whitelist in production.

### Helmet (Security Headers)

```javascript
const helmet = require('helmet');

// Sensible defaults — 15 security headers
app.use(helmet());

// Custom CSP (Content Security Policy)
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "'unsafe-inline'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", 'data:', 'https:'],
    connectSrc: ["'self'", 'https://api.myapp.com'],
    fontSrc: ["'self'"],
    objectSrc: ["'none'"],
    upgradeInsecureRequests: []
  }
}));
```

> **Trap**: Wide-open CSP (`default-src *` or `default-src 'none'` without overriding what you need) defeats the purpose of CSP. Start restrictive and loosen only as needed. Test with `Content-Security-Policy-Report-Only` first.

> **Trap**: Not handling preflight for complex requests — if your API supports `Authorization` header or `application/json` content type, ensure OPTIONS requests return proper CORS headers. The `cors` middleware handles this, but custom middleware before `cors()` may break it.

---

## 15. Caching & Rate Limiting

### In-Memory Caching

```javascript
const LRU = require('lru-cache');

const cache = new LRU({
  max: 500,                // max items
  ttl: 1000 * 60 * 5      // 5 minute TTL
});

// Cache-aside pattern
async function getUser(id) {
  const key = `user:${id}`;
  const cached = cache.get(key);
  if (cached) return cached;

  const user = await db.findUser(id);
  cache.set(key, user);
  return user;
}
```

### Redis Caching

```javascript
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

async function getUser(id) {
  const key = `user:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const user = await db.findUser(id);
  await redis.setex(key, 300, JSON.stringify(user)); // 5 min TTL
  return user;
}
```

### Cache Invalidation Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **TTL** | Expire after time | Simple, stale data acceptable |
| **Write-through** | Update cache when writing to DB | Strong consistency needed |
| **Cache-aside** | App manages cache, checks cache first | Most common for APIs |
| **Stale-while-revalidate** | Serve stale, refresh in background | High-read, stale-tolerant |

**Write-through pattern**:
```javascript
async function updateUser(id, data) {
  const user = await db.updateUser(id, data);
  await redis.setex(`user:${id}`, 300, JSON.stringify(user));
  return user;
}
```

### Stale-While-Revalidate

```javascript
async function getCachedOrFetch(key, fetchFn, ttl = 300, swr = 3600) {
  const cached = await redis.get(key);
  if (cached) {
    const parsed = JSON.parse(cached);
    // Serve stale data, refresh async
    if (parsed._expiresAt && Date.now() > parsed._expiresAt) {
      fetchFn().then(data => {
        redis.setex(key, ttl, JSON.stringify({ ...data, _expiresAt: Date.now() + ttl * 1000 }));
      }).catch(() => {}); // Silently fail — stale data is better than no data
    }
    return parsed;
  }

  const data = await fetchFn();
  await redis.setex(key, ttl, JSON.stringify({ ...data, _expiresAt: Date.now() + ttl * 1000 }));
  return data;
}
```

### Rate Limiting Strategies

```javascript
// Fixed window — simple but bursts at window boundary
const rateLimit = require('express-rate-limit');
const limiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100
});

// Sliding window — more accurate, prevents boundary bursts
// Implemented with Redis sorted sets or sliding window algorithm
const { slidingWindow } = require('rate-limit-redis');
```

### Distributed Rate Limiting with Redis

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

const limiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
    prefix: 'ratelimit:'
  }),
  windowMs: 60 * 1000,
  max: 100,
  keyGenerator: (req) => req.user?.id || req.ip // Per-user or per-IP
});
```

> **Trap**: Cache stampede (dog-pile effect) — when a cached item expires and multiple requests simultaneously hit the database. Mitigate with: (1) mutex/locking, (2) stale-while-revalidate, (3) early recompute before TTL expiry.

> **Trap**: Rate limiting by IP behind a proxy — without `trust proxy` or reading `X-Forwarded-For`, Express sees the proxy's IP, not the client's. Configure `app.set('trust proxy', 1)` to read the correct IP.

> **Trap**: Not namespacing cache keys — `user:123` in dev and production share the same Redis keys. Always namespace (e.g., `appname:env:user:123`).

---

## 16. Tier 2 Q&A Drill

### Questions

1. **Q**: What's the difference between `app` and `Router` in Express? When would you use each?

   **A**: `app` is the application singleton — you call `app.listen()`, `app.set()`, and top-level middleware on it. `Router` creates isolated middleware + route stacks that you mount on `app` (or other routers) at a path prefix. Use `Router` for modular route organization (separate files per resource), `app` for global concerns (body parsers, CORS, error handler). `Router` is also useful for versioning (`/api/v1/`, `/api/v2/`).

2. **Q**: Why does Express middleware order matter? What happens if you put routes before CORS?

   **A**: Express executes middleware in registration order. If CORS middleware is registered after routes, requests to those routes won't get CORS headers, and browsers will reject the response. Order should be: security headers → body parsers → CORS → request logging → auth → validation → routes → error handler.

3. **Q**: How does Express detect error-handling middleware vs regular middleware?

   **A**: By the number of parameters — error-handling middleware has exactly 4 params (`err, req, res, next`). Express checks `fn.length === 4`. If you omit `next`, Express treats it as regular middleware and won't call it for errors.

4. **Q**: What's the main performance advantage of Fastify over Express?

   **A**: Schema-based serialization — Fastify compiles JSON Schema definitions into optimized serialization functions using `fast-json-stringify`. This is significantly faster than Express's `JSON.stringify()`. Combined with Ajv for fast validation, Fastify can handle roughly 2x the throughput of Express for JSON APIs.

5. **Q**: What's a Fastify plugin's encapsulation, and how do you break it?

   **A**: Plugins create an isolated context — decorators, hooks, and routes defined inside are invisible outside. To break encapsulation (share across plugins), wrap with `fastify-plugin` which bypasses the encapsulation boundary. This is necessary for shared decorators like `authenticate`.

6. **Q**: What's the difference between `onRequest` and `preHandler` hooks in Fastify?

   **A**: `onRequest` runs before schema validation/body parsing — use for raw request inspection. `preHandler` runs after validation — use for auth, data loading, etc., where you need parsed body and validated params.

7. **Q**: Explain the JWT structure and why algorithm verification matters.

   **A**: JWT = `header.payload.signature` (base64url encoded). Header specifies the algorithm. If you don't explicitly set `algorithms: ['HS256']` in `jwt.verify()`, an attacker can craft a token with `alg: 'none'` or switch from RS256 to HS256 using the (public) key as the HMAC secret, bypassing verification.

8. **Q**: What are the trade-offs between JWT and session-based auth for a backend API?

   **A**: JWT: stateless (no server-side storage), works across services, but hard to revoke (must wait for expiry), payload size affects every request. Sessions: easy to revoke (delete from store), smaller cookie, but require a shared session store (Redis) for multi-instance deployments, more server round-trips.

9. **Q**: Why use Zod over TypeScript interfaces for validation?

   **A**: Zod validates at **runtime**, not just compile-time. TypeScript interfaces vanish after compilation — they don't prevent malformed API input. Zod also provides transformations (trim, lowercase, defaults), parse-time defaults, and inferred TypeScript types (`z.infer<typeof schema>`) to eliminate type duplication.

10. **Q**: What is the N+1 problem and how do you prevent it in Prisma?

    **A**: N+1 occurs when you fetch a list of N parent records, then issue N separate queries for each child. In Prisma, prevent it using `include` (eager loading via JOINs or batched queries) or custom batch-loading. Example: `prisma.user.findMany({ include: { posts: true } })` issues 2 queries max, not N+1.

11. **Q**: Why is `pipeline()` preferred over `.pipe()` for streams?

    **A**: `pipeline()` (especially `stream/promises` pipeline) automatically handles backpressure, properly destroys all streams on error, and returns a Promise. `.pipe()` does not destroy streams on error and can cause memory leaks in error scenarios.

12. **Q**: How does Node.js `cluster` module work? What pitfalls exist?

    **A**: The master process forks workers (child processes), each running the app independently. Workers share the same port via the OS (TCP connection passing). Pitfalls: master shouldn't handle requests; sticky sessions needed for WebSocket/Session affinity; `os.cpus().length` may be wrong in containers; shared state needs external store (Redis).

13. **Q**: What's the difference between operational and programmer errors? Why does it matter?

    **A**: Operational errors are expected runtime issues (validation errors, 404, DB connection failure) — handle gracefully with user-friendly responses. Programmer errors are bugs (TypeError on undefined, misconfiguration) — crash the process and let PM2/Docker restart it. Mixing them up leads to either brittle services (crashing on bad input) or silent failures (catching bugs and masking them).

14. **Q**: How would you implement idempotency for POST endpoints?

    **A**: Accept an `Idempotency-Key` header. Before processing, check if the key exists in cache/DB — if yes, return cached response. If no, process the request, store the result under the key (with TTL), and return the response. Use a unique constraint on the key column and `INSERT ... ON CONFLICT` or a Redis `SET NX` to prevent duplicate processing from concurrent requests.

15. **Q**: How do you choose between offset and cursor pagination?

    **A**: Offset pagination (`?offset=0&limit=20`) is simpler but has problems: (1) results drift if items are inserted/deleted between pages, (2) large offsets are slow (DB must scan/skip rows). Cursor pagination (`?cursor=base64(id)`) is stable (no drift), fast with indexed cursors, and preferred for real-time data, feeds, and high-scale APIs. Use offset for simple, admin-style list views.

16. **Q**: How do you handle proxy IPs for rate limiting?

    **A**: Express must know to trust the proxy. Call `app.set('trust proxy', 1)` — for a single proxy, or `true` for all proxies. This makes `req.ip` return the client IP from `X-Forwarded-For`. Without it, rate limiting sees the proxy's IP, affecting all users behind that proxy as one.

17. **Q**: What's the cache stampede problem and how do you prevent it?

    **A**: When a cached item expires and many concurrent requests all miss the cache, they all hit the database simultaneously, overwhelming it. Prevention: (1) stale-while-revalidate (serve stale, refresh async), (2) mutex locking (one request refills cache, others wait), (3) early recomputation (refresh before TTL expiry, e.g., with probabilistic early expiration).

18. **Q**: What's the difference between Zod's `safeParse` and `parse`?

    **A**: `parse()` throws a ZodError on invalid input — use in try/catch or async middleware. `safeParse()` returns `{ success: true, data }` or `{ success: false, error }` — use for validation middleware that returns error responses without throwing. `safeParse` is preferred in route handlers to avoid throwing for predictable validation failures.

19. **Q**: How would you handle file uploads for large files (100MB+) in Node.js?

    **A**: Use `multer` with streaming to disk, not memory. Set `limits.fileSize`. For very large files: (1) stream directly to cloud storage (S3 multipart upload), (2) use a separate upload service, (3) implement resumable uploads (chunked with tus protocol). Never use `req.on('data')` with buffering — it'll overwhelm memory.

20. **Q**: What's the difference between `jest.fn()`, `jest.spyOn()`, and `jest.mock()`?

    **A**: `jest.fn()` creates a new mock function. `jest.spyOn()` wraps an existing method, tracking calls but keeping the original implementation unless you call `.mockImplementation()` — useful for observing. `jest.mock()` replaces an entire module at the module system level (hoisted to top of file) — used for external dependencies (DB clients, HTTP clients, Redis). Use `jest.mock()` for modules, `jest.spyOn()` when you need to restore the original implementation, `jest.fn()` for simple callbacks.
