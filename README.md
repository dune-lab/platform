# dune-lab · platform

Architecture overview and global infrastructure for the adaptive learning platform.

## Services

| Service | Repo | Port | Description |
|---------|------|------|-------------|
| **persona** | [dune-lab/persona](https://github.com/dune-lab/persona) | 3000 | Student identity |
| **odyssey** | [dune-lab/odyssey](https://github.com/dune-lab/odyssey) | 3001 | Learning journey state machine |

## Reference

| Repo | Description |
|------|-------------|
| [dune-lab/student-journey](https://github.com/dune-lab/student-journey) | Original monolith — Diplomat Architecture reference |
| [orezende/enxoval](https://github.com/orezende/enxoval) | Shared libraries (`@enxoval/*`) published on npm |

## Communication

```
Client → POST /students (persona:3000)
       → POST /journeys { studentId } (odyssey:3001)

persona  ──── no coupling ────  odyssey
                  ↕
               (Kafka — odyssey only)
```

- **Commands** (do something) → HTTP
- **Events** (something happened) → Kafka

## Run Everything

```bash
docker-compose up
```

Starts:
- `persona` on port 3000
- `odyssey` on port 3001
- `persona-db` (Postgres) on port 5433
- `odyssey-db` (Postgres) on port 5434
- `kafka` on port 29092
- `kafka-ui` on port 8080
- `grafana` on port 4000

## Architecture

```
┌─────────────┐        ┌──────────────────────────────────┐
│   persona   │        │            odyssey               │
│             │        │                                  │
│ POST /stud  │        │ POST /journeys { studentId }     │
│ GET  /stud  │        │ GET  /journeys                   │
│             │        │ POST /journeys/republish         │
│  Postgres   │        │                                  │
│  (no Kafka) │        │  Postgres + Kafka saga           │
└─────────────┘        └──────────────────────────────────┘
       ↑                            ↑
       └────── Client orchestrates ─┘
```

Each service runs independently with its own database. No shared state, no direct coupling.
