# CEIR Architecture Case Study (Focus: Real-Time CheckIMEI Ingestion)
**National-scale telecom compliance & analytics platform**  
**Focus subsystem throughput:** ~500,000 requests per second  
**Role:** Lead / Senior Backend Engineer (Ruby on Rails)

> **Note on scope:** CEIR is a broader platform with multiple subsystems (National List management, user/role management, GSMA and other reference-data pipelines, reporting and admin tooling).  
> This case study intentionally focuses on the **most technically demanding subsystem**: **real-time high-throughput ingestion and processing of MNO CheckIMEI events**.  
> The choices below (OLTP/OLAP separation, event-driven ingestion, and performance engineering) were driven by this core scalability and reliability challenge.

---

## 1. System Context

CEIR enforces regulatory device compliance and fraud controls across telecom operators. The platform must support:

- High-volume transactional API traffic (operator integrations + internal workflows)
- Continuous device event ingestion (CheckIMEI and related events)
- Real-time compliance validation (allow/deny, status classification, enrichment)
- Large-scale analytical querying workloads (dashboards, investigations, reporting)

Key engineering requirements:

- **Low-latency and correctness** for transactional decisions
- **Horizontal scalability** for ingestion under extreme bursts
- **Replayability** for audit/reprocessing
- **Isolation of analytics** so OLAP queries do not degrade OLTP latency
- **Operational reliability** under sustained throughput

---

## 2. Focus Subsystem: CheckIMEI Real-Time Ingestion & Processing

### Problem
Mobile Network Operators (MNOs) generate high-frequency **CheckIMEI** events that must be ingested and processed in near real-time to:

- Validate device status against compliance rules and national lists
- Enrich events (operator, device metadata, policy context)
- Persist authoritative transaction outcomes (audit-grade)
- Stream normalized data into an analytics layer for investigations and reporting

### Throughput / Constraints
- Sustained extreme load (on the order of **~500k RPS** for the integration surface)
- Strict reliability requirements (regulatory environment)
- Mixed read/write database patterns under concurrency
- Need for **asynchronous analytics** without impacting decision latency

---

## 3. High-Level Architecture (Focused View)

```
MNO / Clients
   │
   ▼
Rails Integration API (stateless)
   │
   ├──► PostgreSQL (authoritative OLTP: decisions, state, audit)
   │
   └──► Kafka (durable event stream)
           │
           ▼
     Spark Streaming (normalize/enrich/transform)
           │
           ▼
   ClickHouse (OLAP analytics + reporting)
```

### Architectural Principles
- **Stateless app tier** behind load balancing for horizontal scale
- **Idempotent request handling** for safe retries and duplicate events
- **OLTP/OLAP separation** to protect transactional latency
- **Event-driven ingestion** via Kafka for buffering and replay
- **Deterministic transformations** in Spark to preserve compliance correctness

---

## 4. Application Layer (Ruby on Rails)

### Responsibilities
- Ingress endpoints for MNO integrations (CheckIMEI and related operations)
- Compliance decision orchestration and transactional persistence
- Background processing for non-blocking tasks (when applicable)
- Observability: structured logs, request tracing, operational metrics

### Design Characteristics
- Stateless services behind a load balancer
- Idempotency keys / dedup strategy to tolerate retries
- Carefully scoped DB transactions to reduce lock contention
- Defensive timeouts and backpressure-aware behavior

---

## 5. Transactional Data Layer (PostgreSQL - OLTP)

PostgreSQL remains the **system of record** for:

- Compliance decisions and state transitions
- Audit-relevant transactional history
- Reference pointers to national list statuses (where applicable)

### Performance Engineering
- Index strategy for high-cardinality lookups and hot paths
- Query plan analysis (`EXPLAIN` / `EXPLAIN ANALYZE`)
- Reduction of sequential scans and lock contention
- Write-path tuning for sustained concurrency
- Mitigation of timeout risks under peak throughput

---

## 6. Streaming & Analytics Pipeline

### Kafka (Durable Event Bus)
- Partitioning for horizontal scale
- Ordering guarantees per key (where relevant)
- Replay support for recovery and reprocessing
- Decouples OLTP decisions from downstream analytics

### Spark (Stream Processing)
- Normalization, enrichment, and transformation of ingestion events
- Backpressure management from Kafka
- Deterministic logic to preserve correctness

### ClickHouse (OLAP Store)
- Columnar design optimized for analytics queries
- Fast aggregations and scans at high data volumes
- Keeps OLAP workloads isolated from OLTP latency

---

## 7. Infrastructure & Deployment

- Dockerized services deployed across multiple servers
- CI/CD enabling repeatable, low-risk releases
- Environment isolation (staging vs production)
- Monitoring + centralized logging for incident response

Operational goals:

- Zero/near-zero downtime deploys
- Fast rollback capability
- Predictable behavior under extreme load
- Strong observability during incidents

---

## 8. Core Engineering Challenges (Focused)

- Scaling integration APIs to **~500k RPS**
- Minimizing OLTP contention while sustaining reads/writes
- Protecting decision latency while streaming to analytics
- Ensuring correctness across async pipelines
- Operating distributed infrastructure reliably in production

---

## 9. Outcomes & Impact

- Reduced query latency and production timeout frequency via targeted DB optimization
- Stable high-throughput ingestion and near real-time processing pipeline
- Improved release safety and operational visibility
- Architecture that supports broader CEIR capabilities (national list workflows, GSMA pipelines, admin tooling) while keeping ingestion fast

---

## 10. Technology Stack

**Backend:** Ruby, Ruby on Rails, REST APIs, Redis  
**Databases:** PostgreSQL (OLTP), ClickHouse (OLAP)  
**Streaming & Processing:** Kafka, Apache Spark  
**Infrastructure:** Docker, Linux, CI/CD, multi-server deployments  
**Observability:** Structured logging, metrics, monitoring

---

## 11. Diagram

![CEIR - Focused CheckIMEI Ingestion Architecture](ceir_checkimei_architecture.svg)

---

## Author

**Treasure Kabareebe**  
Lead / Senior Backend Engineer (Ruby on Rails)  
Focus: high-scale backend systems, data-intensive platforms, production reliability  
Open to remote Lead/Senior Backend roles.
