# 🔗 URL Shortener (Project Scaffold)

> **A learning-focused URL shortening service built in Go, designed to explore distributed systems concepts like caching, asynchronous processing, and database scaling.**

---

## 🎯 Project Vision

This project is a **hands-on learning exercise** to build a production-grade URL shortener from scratch. The goal is not just to make something that works, but to intentionally design it to handle **5,000+ requests per second** and explore the tradeoffs that real distributed systems face.

**Key Learning Objectives:**
- Build a RESTful API with Go.
- Implement caching (Redis) to reduce database load.
- Design asynchronous write processing using Go channels.
- Generate distributed IDs (Snowflake algorithm).
- Containerize the application with Docker.
- Deploy to AWS ECS Fargate.
- Load test and measure performance.

---

## 🧠 System Overview (High-Level)

At its core, url-shortner does two things:

1. **Shorten:** Take a long URL and generate a unique 7-character alias.
2. **Redirect:** Take a short alias and return the original long URL via an HTTP 302 redirect.

The system is designed to handle **heavy read traffic** (redirects) with low latency and **asynchronous writes** (shortening) to prevent database bottlenecks.

---

## 📐 High-Level Architecture

### Components

| Component | Responsibility | Tech |
| :--- | :--- | :--- |
| **API Gateway** | Rate limiting, request routing, authentication (future). | Go + `gorilla/mux` |
| **Shorty Service** | Core business logic (shorten, redirect, stats). | Go |
| **Redis** | Cache for hot URLs (90%+ cache hit ratio). | Redis 7.0 |
| **PostgreSQL** | Primary storage for all URL mappings. | PostgreSQL 15 |
| **Worker Pool** | Asynchronous write queue (in-memory). | Go channels + goroutines |
| **Snowflake ID** | Distributed ID generation. | Go implementation |

### Request Flows

**Shorten Flow (Async Write):**

Client -> POST /api/v1/shorten
-> API Gateway (rate limit)
-> url-shortner Service
-> Generate Snowflake ID & Base62 short code
-> Queue write in Worker Pool (async)
-> Return 202 Accepted with short URL
-> Worker Pool writes to PostgreSQL


**Redirect Flow (Sync Read):**

Client -> GET /{short_code}
-> API Gateway
-> url-shortner Service
-> Check Redis cache
-> If miss, query PostgreSQL
-> Cache result in Redis (TTL 24 hours)
-> Return 302 Redirect to long_url


---

## 🗺️ Project Roadmap (Phased Approach)

| Phase | Focus | What You'll Build | Key Learning |
| :--- | :--- | :--- | :--- |
| **Phase 1: Core** | Get it working locally. | REST API with `gorilla/mux`, in-memory storage (map), Base62 encoding, redirect logic. | Go HTTP server, routing, encoding. |
| **Phase 2: Persistence** | Add PostgreSQL. | Replace in-memory map with PostgreSQL. Implement `pgxpool` connection pooling. | SQL schema design, connection management. |
| **Phase 3: Caching** | Add Redis. | Implement Cache-Aside pattern. On redirect: check Redis first, fall back to PostgreSQL. Cache miss → query DB → store in Redis with TTL. | Caching strategies, TTLs, cache invalidation. |
| **Phase 4: Async Writes** | Scale writes. | Implement in-memory worker pool (Go channels). `POST /shorten` writes to channel, returns `202 Accepted`. Workers consume channel and write to PostgreSQL. | Concurrency, goroutines, channels, backpressure. |
| **Phase 5: ID Generation** | Remove DB auto-increment. | Implement Snowflake ID generator (Twitter's algorithm). Generate ID locally in Go. | Distributed ID generation, worker ID assignment, clock drift. |
| **Phase 6: Observability** | Add monitoring. | Add Prometheus metrics (RPS, latency, error rate, cache hit ratio). Add structured logging (JSON). | Metrics, logging, debugging. |
| **Phase 7: Containerization** | Package for deployment. | Write `Dockerfile` and `docker-compose.yml` for local dev. | Docker, multi-container orchestration. |
| **Phase 8: Cloud Deployment** | Deploy to AWS. | Deploy to ECS Fargate with ALB, RDS, and ElastiCache. | AWS ECS, IAM, networking, CI/CD. |
| **Phase 9: Load Testing** | Validate performance. | Write Locust load tests. Target: 5,000 RPS redirects at < 150ms p95. | Load testing, performance tuning. |

---

## 📂 Project Structure (Planned)

```text 
URL-SHORTNER/
├── cmd/
│ └── api/
│ └── main.go # Application entry point
├── internal/
│ ├── api/
│ │ ├── handlers.go # HTTP handlers (shorten, redirect, stats)
│ │ └── routes.go # Route registration
│ ├── cache/
│ │ └── redis.go # Redis client + Cache-Aside logic
│ ├── db/
│ │ └── postgres.go # PostgreSQL client + repository methods
│ ├── id/
│ │ └── snowflake.go # Snowflake ID generator
│ ├── queue/
│ │ └── worker.go # Worker pool (channels + goroutines)
│ └── config/
│ └── config.go # Configuration (env vars)
├── pkg/
│ └── utils/
│ └── base62.go # Base62 encoding/decoding
├── migrations/
│ └── 001_create_urls_table.sql # Database schema
├── docker-compose.yml # Local dev dependencies (Redis, PostgreSQL)
├── Dockerfile # Container image
├── Makefile # Build and test shortcuts
├── go.mod
├── go.sum
├── .env.example # Environment variables template
└── README.md # You are here
```


---

## 📊 Data Model (Planned)

**`urls` Table (PostgreSQL)**

| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | BIGINT | Primary key (Snowflake ID). |
| `short_code` | VARCHAR(10) | Unique 7-character alias (e.g., `abc123`). |
| `long_url` | TEXT | Original URL. |
| `created_at` | TIMESTAMP | When the URL was shortened. |
| `ttl` | INT | Time-to-live in seconds (default: 86400). |
| `click_count` | INT | Total redirects (updated asynchronously). |

**Indexes:** `short_code` (unique) | `created_at` (for archival).

---

## 🔌 API Endpoints (Planned)

| Method | Endpoint | Description | Status Code |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/shorten` | Shorten a URL (async) | `202 Accepted` |
| `GET` | `/{short_code}` | Redirect to long URL | `302 Found` |
| `GET` | `/api/v1/stats/{short_code}` | Get click count | `200 OK` |
| `GET` | `/health` | Health check for ALB | `200 OK` |

---

## 🧪 Learning Goals (What You'll Master)

By completing this project, I should be able to confidently explain:

- **Caching:** Why Cache-Aside > Read-Through for URL shorteners.
- **Async Processing:** When to use queues vs. synchronous writes.
- **ID Generation:** Why Snowflake > UUID for distributed systems.
- **Sharding:** How to horizontally scale the database (future phase).
- **Consistent Hashing:** How to distribute cache keys across multiple Redis nodes (future phase).
- **Observability:** Why metrics > logs for debugging in production.

---