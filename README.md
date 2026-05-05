# dune-lab · platform

Local development environment for the dune-lab adaptive learning platform. Runs all services, databases, Kafka, and the full observability stack with a single command.

---

## Quick Start

```bash
docker compose up
```

That's it. All services start in dependency order. Migrations run automatically.

---

## Services

| Service | Port | Description |
|---------|------|-------------|
| **arrakis** | 5173 | Frontend SPA (React + Vite → nginx) |
| **imperium** | 3004 | BFF — single entry point for the client |
| **janus** | 3003 | Auth — issues JWT tokens |
| **atreides** | 3002 | User identity & credential validation |
| **persona** | 3000 | Student profiles |
| **odyssey** | 3001 | Learning journey state machine + DLQ |

## Infrastructure

| Service | Port | Description |
|---------|------|-------------|
| **odyssey-db** | 5434 | PostgreSQL — journeys + DLQ messages |
| **atreides-db** | 5435 | PostgreSQL — users |
| **persona-db** | 5433 | PostgreSQL — students |
| **kafka** | 9092 / 29092 | Apache Kafka 3.7 |
| **kafka-ui** | 8081 | Kafka UI — browse topics and messages |
| **grafana** | 4001 | Dashboards — HTTP, Kafka, DLQ, alerts |
| **loki** | internal | Log aggregation |
| **promtail** | internal | Log shipping (Docker socket) |
| **dashboard** | 9000 | Platform status dashboard |

---

## Architecture

```
Browser
  └─► arrakis:5173
         └─► imperium:3004
               ├─► janus:3003
               │     └─► atreides:3002
               ├─► atreides:3002
               ├─► persona:3000
               └─► odyssey:3001
                     ├─► odyssey-db:5434
                     └─► kafka:9092
                           └─► odyssey (internal saga consumers)
```

### Communication Model

- **HTTP** — synchronous commands (do something, get data)
- **Kafka** — asynchronous events (something happened)
- **SSE** — real-time push to browser (journey progress updates)

### Service Isolation

Each service owns its database. No cross-service joins. No shared schema. Inter-service data access is always via HTTP.

---

## Authentication Flow

```
1. arrakis  POST /auth/login { email, password }
                    ↓
2. imperium → janus POST /auth/login
                    ↓
3. janus → atreides POST /users/authenticate
                    ↓
4. atreides validates → returns { id, email, role }
                    ↓
5. janus signs JWT { userId, role } → returns { token }
                    ↓
6. arrakis stores token, sends Authorization: Bearer <token>
                    ↓
7. Each service validates the JWT independently via @enxoval/auth
```

---

## Event-Driven Architecture (Odyssey Saga)

Odyssey runs a fully autonomous Kafka saga. Once a journey is started, it advances itself through all steps without any external trigger:

```
POST /journeys → publish journeyInitiated
  └── journeyInitiated          → publish diagnosticTriggered
  └── diagnosticTriggered       → publish diagnosticCompleted
  └── diagnosticCompleted       → publish analysisStarted
  └── analysisStarted           → publish analysisFinished
  └── analysisFinished          → publish curriculumGenerated
  └── curriculumGenerated       → publish contentDispatched
  └── contentDispatched         → publish studentEngagementReceived
  └── studentEngagementReceived → publish progressMilestoneReached
  └── progressMilestoneReached  → journey complete
```

Atreides also publishes `userCreated` and `mailConfirmed` events — available for future consumers.

---

## Resilience — Harkonnen DLQ

When any odyssey saga consumer fails 3 consecutive times, `@enxoval/messaging` routes the message to the `student-journey-dlq` Kafka topic.

A dedicated consumer (`odyssey-harkonnen-dlq` group) reads this topic and persists every failure in the `harkonnen_messages` table.

Operators can inspect, edit, and reprocess failed messages via:
- **UI**: arrakis `/admin/dlq`
- **API**: `GET/POST /harkonnen` on odyssey or imperium

```
saga consumer fails × 3
  └─► student-journey-dlq
        └─► harkonnen consumer → harkonnen_messages (status: pending)
              └─► operator reprocesses → republish to original topic
                    └─► saga retries
```

---

## Observability Stack

### Logs

All services emit structured JSON via Pino with a `service` field:

```json
{ "level": "info", "service": "odyssey", "cid": "abc:0", "method": "POST", "url": "/journeys", "status": 201, "durationMs": 42, "msg": "http-server: response sent" }
```

Promtail ships logs from the Docker socket to Loki. The `service` label is extracted as a Loki index label, enabling per-service filtering.

### Grafana Dashboards

Open Grafana at **http://localhost:4001** (no login required).

| Dashboard | UID | Description |
|-----------|-----|-------------|
| **Platform Status** | `platform-status` | ONLINE/OFFLINE indicator per service |
| **Services** | `service-detail` | Per-service: request rate, errors, latency |
| **HTTP Requests** | `http-requests` | All HTTP traffic across services |
| **Kafka Messages** | `kafka-messages` | Producer + consumer activity |
| **DLQ Monitor** | `dlq-monitor` | Dead letter queue ingestion rate + alerts |

### Kafka UI

Browse topics, consumer groups, and messages at **http://localhost:8081**.

### Alerts

Pre-configured Grafana alerts:
- Consumer crash (kafkajs `[Consumer] Crash`)
- DLQ message ingestion spike
- 5xx error spike
- Service down (absent logs for 3+ minutes)

---

## Shared Libraries (`@enxoval/*`)

All backend services use a common set of internal libraries:

| Package | Description |
|---------|-------------|
| `@enxoval/auth` | JWT middleware for Fastify — validates Bearer tokens |
| `@enxoval/db` | TypeORM wrapper — data source factory, migrations, `defineEntity` |
| `@enxoval/http` | Fastify wrapper — `listen`, `get`, `post`, typed route registration |
| `@enxoval/messaging` | Kafka wrapper — `subscribe`, `publish`, `publishRaw`, retry + DLQ policy |
| `@enxoval/observability` | Pino logger — structured JSON, `service` field from `SERVICE_NAME` env |
| `@enxoval/types` | Schema validation — `createSchema`, `field.*`, `asyncFn`, `fn` |

---

## Rebuilding a Service

After code changes:

```bash
docker compose build <service>
docker compose up -d <service>
```

---

## Repository Map

| Repo | Description |
|------|-------------|
| [dune-lab/arrakis](https://github.com/dune-lab/arrakis) | Frontend SPA |
| [dune-lab/imperium](https://github.com/dune-lab/imperium) | BFF |
| [dune-lab/janus](https://github.com/dune-lab/janus) | Auth |
| [dune-lab/atreides](https://github.com/dune-lab/atreides) | Users |
| [dune-lab/persona](https://github.com/dune-lab/persona) | Students |
| [dune-lab/odyssey](https://github.com/dune-lab/odyssey) | Journeys + DLQ |
| [dune-lab/platform](https://github.com/dune-lab/platform) | This repo |
| [dune-lab/enxoval](https://github.com/dune-lab/enxoval) | Shared libraries |
