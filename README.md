# Backend Engineer — Interview Preparation

Deep-dive preparation material for **Senior Backend Engineer** roles at product-based / FAANG-type companies.

Each topic is broken into tiers (Basic → Intermediate → Senior), written with working code examples, common interview traps, and the follow-up questions a senior interviewer actually asks. Examples are tied to real production experience wherever possible rather than to toy problems.

---

## Skill tracker

| # | Topic | Status | Material |
|---|-------|--------|----------|
| 1 | **PHP / Laravel** | ✅ Complete | [`topics/php-laravel/`](./topics/php-laravel/) |
| 2 | **Go (Golang) / Gin** | ✅ Complete | [`topics/golang-gin/`](./topics/golang-gin/) |
| 3 | _next topic_ | ⬜ Not started | see `.cursorrules` |

Remaining skills from the CV, roughly in the order they're likely to matter:
databases (PostgreSQL/MySQL/Redis/Elasticsearch/DynamoDB) · system design & architecture (microservices, event-driven, DDD, CQRS) · AWS · Kubernetes & Docker · messaging (Kafka, RabbitMQ, SQS/SNS) · auth & security (OAuth 2.0, JWT, OWASP) · observability · CI/CD & Terraform.

---

## Topics

### 1. PHP / Laravel — [`topics/php-laravel/`](./topics/php-laravel/)

| File | Focus |
|------|-------|
| [`01-basic.md`](./topics/php-laravel/01-basic.md) | PHP language fundamentals and Laravel basics |
| [`02-oop-php.md`](./topics/php-laravel/02-oop-php.md) | **Canonical source for OOP, SOLID and design patterns** — also covariance, traits, enums, value objects, property hooks |
| [`03-intermediate.md`](./topics/php-laravel/03-intermediate.md) | Container, providers, middleware, queues, events, caching, auth |
| [`04-senior.md`](./topics/php-laravel/04-senior.md) | Multi-tenancy, concurrency, migrations, performance, scaling, security |
| [`05-question-bank.md`](./topics/php-laravel/05-question-bank.md) | Question bank and drill material |

### 2. Go (Golang) / Gin — [`topics/golang-gin/`](./topics/golang-gin/)

| File | Focus |
|------|-------|
| [`01-basic.md`](./topics/golang-gin/01-basic.md) | Type system, **slice and map internals**, method sets, interfaces, the **nil interface trap**, errors, generics |
| [`02-concurrency.md`](./topics/golang-gin/02-concurrency.md) | **The make-or-break tier** — goroutines, channels, `select`, `sync`, `context`, the memory model, patterns, **goroutine leaks** |
| [`03-gin-web.md`](./topics/golang-gin/03-gin-web.md) | `net/http`, Gin router and context, middleware, binding, `database/sql` pool tuning, gRPC, testing |
| [`04-senior.md`](./topics/golang-gin/04-senior.md) | GMP scheduler, GC, escape analysis, pprof, **Raft and Chronos**, distributed locks, resilience, production |
| [`05-question-bank.md`](./topics/golang-gin/05-question-bank.md) | 25 verified "what does this print" puzzles, 180+ rapid-fire Q&A, live-coding, system design, STAR stories |

> All Go examples are compiled and verified against **Go 1.26**. Quoted program output, allocation counts, and compiler thresholds are real measurements, not recalled figures.

---

## How to use this

Active recall, not passive reading. Passive re-reading feels productive and does almost nothing for retrieval under pressure.

1. **Read** a section, then close the file and explain it out loud as if to an interviewer.
2. **Type** the code examples from memory — do not copy/paste.
3. **Answer** the section's Q&A cold, then diff against the written answer.
4. **Note** where your answer was vague. Vagueness is what fails senior loops, not ignorance.

Grade yourself on precision rather than on whether you were "basically right." "Go has a fast GC" and "two sub-millisecond stop-the-world phases, and mark assists mean allocation pressure surfaces as latency" are the same fact at two very different scores.

---

## Real-world anchors

These come up repeatedly across topics because concrete numbers and real incidents are what make senior answers credible:

- **88% query reduction** — profiling method, N+1 elimination, regression prevention
- **Zero-downtime migration of 15M+ records** — expand/migrate/contract, batched backfill, rollback plan
- **20K+ DAU trading platform** — scale, latency, concurrency and consistency
- **Multi-tenant inventory SaaS** — row-level tenancy via `organization_id`, Passport, permissions in teams mode, isolation testing
- **Chronos** — Go distributed job scheduler with Raft leader election, idempotent execution, exactly-once *effects*

---

## Repository layout

```
README.md                      ← you are here (master index + skill tracker)
.cursorrules                   ← coaching brief and skill order
topics/
  php-laravel/                 ← topic index + 5 tiered files
  golang-gin/                  ← topic index + 5 tiered files
```
