# dune-lab · platform

Architecture overview and global infrastructure for the adaptive learning platform.

## Services

| Service | Repo | Port | Description |
|---------|------|------|-------------|
| **imperium** | [dune-lab/imperium](https://github.com/dune-lab/imperium) | 3004 | BFF — single entry point for the client |
| **janus** | [dune-lab/janus](https://github.com/dune-lab/janus) | 3003 | Auth — issues JWT tokens |
| **atreides** | [dune-lab/atreides](https://github.com/dune-lab/atreides) | 3002 | User identity & authentication |
| **persona** | [dune-lab/persona](https://github.com/dune-lab/persona) | 3000 | Student profiles |
| **odyssey** | [dune-lab/odyssey](https://github.com/dune-lab/odyssey) | 3001 | Learning journey state machine |

## Reference

| Repo | Description |
|------|-------------|
| [dune-lab/student-journey](https://github.com/dune-lab/student-journey) | Original monolith — Diplomat Architecture reference |
| [orezende/enxoval](https://github.com/orezende/enxoval) | Shared libraries (`@enxoval/*`) |

## Communication

```
Client
  │
  ├─ POST /auth/login  ──────────────────────► janus
  │                                              │
  │                                              └─ POST /users/authenticate ──► atreides
  │
  └─ GET  /me  ─────────────────────────────► imperium
                                                 │
                                                 ├─ GET /users/:id ────────────► atreides
                                                 ├─ GET /students/by-user/:id ─► persona
                                                 └─ GET /journeys/by-student/:id ► odyssey

atreides ──── userCreated, mailConfirmed ────► (Kafka — future consumers)

odyssey internal saga (Kafka):
  journeyInitiated → diagnosticTriggered → diagnosticCompleted
  → analysisStarted → analysisFinished → curriculumGenerated
  → contentDispatched → studentEngagementReceived
  → progressMilestoneReached → journeyCompleted
```

- **Commands** (do something) → HTTP
- **Events** (something happened) → Kafka

## Auth Flow

```
1. Client  POST /auth/login { email, password }  → janus
2. janus validates credentials against atreides  → returns JWT { userId, role }
3. Client sends  Authorization: Bearer <token>  on every request to imperium
4. imperium decodes the token (no re-validation) → proxies Bearer to downstream
5. Each downstream service validates the token independently via @enxoval/auth
```

## Run Everything

```bash
docker-compose up
```

| Container | Port |
|-----------|------|
| imperium | 3004 |
| janus | 3003 |
| atreides | 3002 |
| persona | 3000 |
| odyssey | 3001 |
| atreides-db (Postgres) | 5435 |
| persona-db (Postgres) | 5433 |
| odyssey-db (Postgres) | 5434 |
| kafka | 29092 |
| kafka-ui | 8080 |
| grafana | 4000 |

## Architecture

```
                    ┌──────────────┐
                    │   imperium   │  ← single client entry point
                    │  port 3004   │
                    └──┬───┬───┬───┘
                       │   │   │
           ┌───────────┘   │   └────────────┐
           ▼               ▼                ▼
    ┌──────────┐   ┌──────────────┐   ┌──────────────┐
    │ atreides │   │   persona    │   │   odyssey    │
    │ port 3002│   │  port 3000   │   │  port 3001   │
    │  users   │   │  students    │   │  journeys    │
    │ Postgres │   │  Postgres    │   │  Postgres    │
    │  Kafka ──┼──►│  (no Kafka)  │   │  + Kafka saga│
    └────▲─────┘   └──────────────┘   └──────────────┘
         │
  ┌──────────┐
  │  janus   │  ← auth only, no DB
  │ port 3003│
  └──────────┘
```

Each service runs independently with its own database. No shared state, no direct service-to-service coupling except `janus → atreides` for credential validation.
