# 🔗 URL Shortener (Project Scaffold)

> **A learning-focused URL shortening service built in Spring Boot, designed to explore distributed systems concepts like caching, asynchronous processing, and database scaling.**

---

## 🎯 Project Vision

This project is a **hands-on learning exercise** to build a production-grade URL shortener from scratch. The goal is not just to make something that works, but to intentionally design it to handle **5,000+ requests per second** and explore the tradeoffs that real distributed systems face.

**Key Learning Objectives:**
- Build a RESTful API with Spring Boot.
- Implement caching (Redis) to reduce database load using Spring Cache.
- Design asynchronous write processing using Spring's `@Async`.
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
| **API Gateway** | Rate limiting, request routing, authentication (future). | Spring Boot + Spring Security |
| **Shorty Service** | Core business logic (shorten, redirect, stats). | Spring Boot |
| **Redis** | Cache for hot URLs (90%+ cache hit ratio). | Redis 7.0 + Spring Cache |
| **PostgreSQL** | Primary storage for all URL mappings. | PostgreSQL 15 + Spring Data JPA |
| **Worker Pool** | Asynchronous write queue. | Spring `@Async` / TaskExecutor |
| **Snowflake ID** | Distributed ID generation. | Java implementation |

### Request Flows

**Shorten Flow (Async Write):**
Client -> POST /api/v1/shorten
-> API Gateway (rate limit)
-> URL Shortener Service
-> Generate Snowflake ID & Base62 short code
-> Queue write asynchronously (@Async)
-> Return 202 Accepted with short URL
-> Async task writes to PostgreSQL


**Redirect Flow (Sync Read):**
Client -> GET /{short_code}
-> API Gateway
-> URL Shortener Service
-> Check Redis cache (@Cacheable)
-> If miss, query PostgreSQL
-> Cache result in Redis (TTL 24 hours)
-> Return 302 Redirect to long_url


---

## 🗺️ Project Roadmap (Phased Approach)

| Phase | Focus | What You'll Build | Key Learning |
| :--- | :--- | :--- | :--- |
| **Phase 1: Core** | Get it working locally. | REST API with Spring Boot, in-memory storage (ConcurrentHashMap), Base62 encoding, redirect logic. | Spring Boot basics, HTTP controllers, encoding. |
| **Phase 2: Persistence** | Add PostgreSQL. | Replace in-memory map with JPA/Hibernate. Implement Spring Data JPA repositories. | ORM, JPA, connection pooling with HikariCP. |
| **Phase 3: Caching** | Add Redis. | Implement Spring Cache abstraction with Redis. Use `@Cacheable` and `@CacheEvict`. | Caching strategies, Spring Cache abstraction, TTLs. |
| **Phase 4: Async Writes** | Scale writes. | Implement `@Async` with Spring's TaskExecutor. `POST /shorten` returns `202 Accepted`. | Concurrency, `@Async`, thread pools, backpressure. |
| **Phase 5: ID Generation** | Remove DB auto-increment. | Implement Snowflake ID generator (Twitter's algorithm). Generate ID locally in Java. | Distributed ID generation, worker ID assignment, clock drift. |
| **Phase 6: Observability** | Add monitoring. | Add Micrometer + Prometheus metrics. Add structured logging with Logback (JSON). | Metrics, logging, debugging, Actuator. |
| **Phase 7: Containerization** | Package for deployment. | Write `Dockerfile` and `docker-compose.yml` for local dev. | Docker, multi-container orchestration. |
| **Phase 8: Cloud Deployment** | Deploy to AWS. | Deploy to ECS Fargate with ALB, RDS, and ElastiCache. | AWS ECS, IAM, networking, CI/CD. |
| **Phase 9: Load Testing** | Validate performance. | Write JMeter or Gatling load tests. Target: 5,000 RPS redirects at < 150ms p95. | Load testing, performance tuning. |

---

## 📂 Project Structure (Planned)
URL-SHORTENER/
├── src/
│ ├── main/
│ │ ├── java/
│ │ │ └── com/urlshortener/
│ │ │ ├── UrlShortenerApplication.java
│ │ │ ├── controller/
│ │ │ │ ├── UrlController.java
│ │ │ │ └── HealthController.java
│ │ │ ├── service/
│ │ │ │ ├── UrlService.java
│ │ │ │ ├── AsyncUrlService.java
│ │ │ │ └── StatsService.java
│ │ │ ├── repository/
│ │ │ │ └── UrlRepository.java
│ │ │ ├── cache/
│ │ │ │ └── CacheConfig.java
│ │ │ ├── model/
│ │ │ │ ├── Url.java
│ │ │ │ └── UrlDTO.java
│ │ │ ├── util/
│ │ │ │ ├── Base62Encoder.java
│ │ │ │ └── SnowflakeIdGenerator.java
│ │ │ └── config/
│ │ │ ├── AsyncConfig.java
│ │ │ └── RedisConfig.java
│ │ └── resources/
│ │ ├── application.yml
│ │ ├── application-dev.yml
│ │ ├── application-prod.yml
│ │ ├── db/
│ │ │ └── migration/
│ │ │ └── V1__create_urls_table.sql
│ │ └── logback-spring.xml
│ └── test/
│ └── java/
│ └── com/urlshortener/
│ └── UrlControllerTest.java
├── docker-compose.yml
├── Dockerfile
├── pom.xml (or build.gradle)
├── .env.example
└── README.md


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

**JPA Entity Example:**
```java
@Entity
@Table(name = "urls")
public class Url {
    @Id
    private Long id;
    
    @Column(unique = true, length = 10)
    private String shortCode;
    
    @Column(columnDefinition = "TEXT")
    private String longUrl;
    
    private LocalDateTime createdAt;
    private Integer ttl;
    private Integer clickCount;
    
    // getters, setters, constructors
}


🔌 API Endpoints (Planned)
Method	Endpoint	Description	Status Code
POST	/api/v1/shorten	Shorten a URL (async)	202 Accepted
GET	/{short_code}	Redirect to long URL	302 Found
GET	/api/v1/stats/{short_code}	Get click count	200 OK
GET	/health	Health check for ALB	200 OK

Sample Request/Response:
// POST /api/v1/shorten
{
    "longUrl": "https://example.com/very/long/url/that/needs/shortening"
}

// Response (202 Accepted)
{
    "shortUrl": "http://localhost:8080/abc123",
    "shortCode": "abc123",
    "message": "URL shortening in progress"
}

// GET /api/v1/stats/abc123
{
    "shortCode": "abc123",
    "longUrl": "https://example.com/very/long/url/that/needs/shortening",
    "clickCount": 42,
    "createdAt": "2024-01-15T10:30:00Z"
}

