# dune-lab · platform

Architecture overview and global infrastructure for the adaptive learning platform.

## Services

| Service | Repo | Port | Description |
|---------|------|------|-------------|
| **arrakis** | [dune-lab/arrakis](https://github.com/dune-lab/arrakis) | 5173 | Frontend SPA — React + Vite |
| **imperium** | [dune-lab/imperium](https://github.com/dune-lab/imperium) | 3004 | BFF — single entry point for the client |
| **janus** | [dune-lab/janus](https://github.com/dune-lab/janus) | 3003 | Auth — issues JWT tokens |
| **atreides** | [dune-lab/atreides](https://github.com/dune-lab/atreides) | 3002 | User identity & authentication |
| **persona** | [dune-lab/persona](https://github.com/dune-lab/persona) | 3000 | Student profiles |
| **odyssey** | [dune-lab/odyssey](https://github.com/dune-lab/odyssey) | 3001 | Learning journey state machine |

## Reference

| Repo | Description |
|------|-------------|
| [dune-lab/student-journey](https://github.com/dune-lab/student-journey) | Original monolith — Diplomat Architecture reference |
| [dune-lab/enxoval](https://github.com/dune-lab/enxoval) | Shared libraries (`@enxoval/*`) |

## Communication

```
arrakis (browser)
  │
  └─ ALL requests ───────────────────────────► imperium
                                                 │
                                                 ├─ POST /auth/login ──────────► janus
                                                 │                                 │
                                                 │                                 └─ POST /users/authenticate ──► atreides
                                                 │
                                                 ├─ POST /users/register ──────► atreides
                                                 │
                                                 ├─ GET  /me ──────────────────► atreides + persona + odyssey
                                                 │
                                                 ├─ POST /students ────────────► persona
                                                 ├─ GET  /students ────────────► persona
                                                 │
                                                 ├─ POST /journeys ────────────► odyssey
                                                 ├─ GET  /journeys ────────────► odyssey
                                                 └─ POST /journeys/republish ──► odyssey

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
1. arrakis  POST /auth/login { email, password }  → imperium → janus
2. janus validates credentials against atreides   → returns JWT { userId, role }
3. imperium returns the token to arrakis
4. arrakis sends  Authorization: Bearer <token>  on every subsequent request to imperium
5. imperium validates the token via @enxoval/auth, then proxies Bearer to downstream
6. Each downstream service validates the token independently via @enxoval/auth
```

## Run Everything

```bash
docker-compose up
```

| Container | Port |
|-----------|------|
| arrakis | 5173 |
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
              │   arrakis    │  ← SPA (browser)
              │  port 5173   │
              └──────┬───────┘
                     │ all requests
                     ▼
              ┌──────────────┐
              │   imperium   │  ← single backend entry point
              │  port 3004   │
              └──┬──┬──┬──┬──┘
                 │  │  │  │
       ┌─────────┘  │  │  └──────────┐
       ▼            ▼  ▼             ▼
┌──────────┐  ┌──────┐ ┌──────────┐ ┌──────────────┐
│ atreides │  │janus │ │ persona  │ │   odyssey    │
│ port 3002│  │ 3003 │ │ port 3000│ │  port 3001   │
│  users   │  │ auth │ │ students │ │  journeys    │
│ Postgres │  │no DB │ │ Postgres │ │  Postgres    │
│  Kafka ──┼─►│      │ │          │ │  + Kafka saga│
└──────────┘  └──────┘ └──────────┘ └──────────────┘
```

Each service runs independently with its own database. No shared state, no direct service-to-service coupling. The client never talks to services directly — all traffic goes through imperium.
