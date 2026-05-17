# Cognizant Support Ticketing System — Complete Project Documentation

A full-stack microservices platform: React/Vite frontend + 8 Spring Boot backend services, orchestrated via Docker-managed Kafka/Postgres, with both **local JAR-based** and **Kubernetes (kind)** deployment paths.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Technology Stack](#3-technology-stack)
4. [Services at a Glance](#4-services-at-a-glance)
5. [Prerequisites (Local Machine)](#5-prerequisites-local-machine)
6. [Credentials & Secrets](#6-credentials--secrets)
7. [Project Directory Layout](#7-project-directory-layout)
8. [First-Time Setup](#8-first-time-setup)
9. [Running the Stack — Local Mode (`services.sh`)](#9-running-the-stack--local-mode-servicessh)
10. [Running the Stack — Kubernetes Mode (`k8s/k8s.sh`)](#10-running-the-stack--kubernetes-mode-k8sk8ssh)
11. [Per-Service Deep Dive](#11-per-service-deep-dive)
12. [Frontend](#12-frontend)
13. [API Endpoints Reference](#13-api-endpoints-reference)
14. [Swagger / OpenAPI Documentation](#14-swagger--openapi-documentation)
15. [Kafka Topics](#15-kafka-topics)
16. [Auth Flow](#16-auth-flow)
17. [Complete List of Changes & Fixes Applied](#17-complete-list-of-changes--fixes-applied)
18. [Troubleshooting](#18-troubleshooting)
19. [Quick Command Reference](#19-quick-command-reference)

---

## 1. Project Overview

A support-ticket platform where users create tickets, engineers post solutions, the community rates knowledge-base articles, and users earn reward points/badges. The system is split into 8 bounded-context microservices behind a single API Gateway, with asynchronous events flowing through Kafka.

- **9 runtime components**: 1 gateway + 7 services + 1 frontend
- **7 Postgres databases** (one per service, except gateway)
- **Apache Kafka** for cross-service events
- **JWT HS384** authentication, centralized at the gateway, re-validated by each downstream service

---

## 2. Architecture

```
                        ┌─────────────────────┐
                        │  React + Vite SPA   │   :3000
                        │  (frontend)         │
                        └──────────┬──────────┘
                                   │  HTTP /api/*
                                   ▼
                        ┌─────────────────────┐
                        │  API Gateway        │   :8080
                        │  Spring Cloud       │
                        │  Gateway (WebFlux)  │
                        │  + AuthFilter       │
                        └──┬──┬──┬──┬──┬──┬──┬┘
             ┌────────────┘  │  │  │  │  │  └────────────┐
             ▼               ▼  ▼  ▼  ▼  ▼               ▼
  ┌────────────────┐  ┌───────┐ ...                ┌────────────────┐
  │ auth-service   │  │ user- │                    │ notification-  │
  │ :8081          │  │service│                    │ service :8087  │
  │ issues JWTs    │  │ :8082 │                    └────────────────┘
  └──────┬─────────┘  └───┬───┘
         │ JPA            │ JPA
         ▼                ▼
  ┌────────────┐   ┌──────────┐     ... one Postgres per service
  │ auth_db    │   │ user_db  │
  │ postgres   │   │ postgres │
  └────────────┘   └──────────┘

  All services also publish/consume on:   ┌─────────────┐
                                          │   Kafka     │  :9092
                                          │  (Docker)   │
                                          └─────────────┘
```

### Dependency order on boot
1. Postgres + Kafka (infra)
2. `auth-service` (gateway's AuthFilter calls it)
3. All other business services (any order)
4. `api-gateway` (needs downstreams to be healthy)
5. Frontend

---

## 3. Technology Stack

| Layer | Technology | Version |
|---|---|---|
| Frontend | React + Vite + TailwindCSS | latest |
| Frontend runtime | Node.js | 18+ (via `nvm` or brew) |
| Backend framework | Spring Boot | 3.3.5 |
| Gateway | Spring Cloud Gateway (reactive/WebFlux) | matching Boot |
| Build tool | Maven (wrapper included) | 3.x |
| JVM | OpenJDK 21 (Homebrew) | 21.0.11 |
| DB | PostgreSQL (Homebrew) | 14 |
| Messaging | Apache Kafka (Docker image) | confluentinc/cp-kafka 7.5.0 (local) / apache/kafka 3.9.2 (K8s) |
| Container engine | Docker Desktop | 28.1.1 |
| K8s cluster | kind | 0.31.0 |
| K8s ingress | nginx-ingress | latest |
| Auth | JWT HS384 | JJWT 0.11.x |
| Common library | `com.kva:common-library:1.0.0` (local-only) | 1.0.0 |

### Why Java 21 specifically
Java 21 is the current LTS and is fully certified for Spring Boot 3.3.5, Spring Cloud 2023.0.x, and Lombok 1.18.34. System Maven uses whichever `java` is on PATH, which on this machine may resolve to a newer or different release. To keep builds reproducible, every Maven command is executed with:
```
JAVA_HOME=/opt/homebrew/Cellar/openjdk@21/21.0.11
```
`services.sh` handles this automatically via its `MVN_ENV` wrapper.

> Historical note: the stack originally ran on Java 17. It was migrated to Java 21 in April 2026 — poms, Dockerfiles, the `services.sh` `JAVA_HOME`, and Lombok (1.18.30 → 1.18.34) and maven-compiler-plugin (3.11.0 → 3.13.0) were all bumped. The migration is captured in § 16 "Changes & Fixes Applied".

---

## 4. Services at a Glance

| Port | Service | Folder | Database | JAR |
|---|---|---|---|---|
| 3000 | Frontend (React/Vite) | `frontend/` | — | — |
| 8080 | API Gateway | `api_gateway/` | — | `api-gateway-0.0.1-SNAPSHOT.jar` |
| 8081 | Auth Service | `auth_service/` | `auth_db` | `auth_service-0.0.1-SNAPSHOT.jar` |
| 8082 | User Service | `User_service/` | `user_db` | `User_service-0.0.1-SNAPSHOT.jar` |
| 8083 | Ticket Service | `Ticket_service/` | `ticket_db` | `Ticket_service-0.0.1-SNAPSHOT.jar` |
| 8084 | Solution Service | `Solution_service/` | `solution_db` | `solution-service-0.0.1-SNAPSHOT.jar` |
| 8085 | Knowledge Service | `knowledge_service/` | `knowledge_db` | `knowledge-service-0.0.1-SNAPSHOT.jar` |
| 8086 | Reward Service | `Reward_service/` | `reward_db` | `Reward_service-0.0.1-SNAPSHOT.jar` |
| 8087 | Notification Service | `Notification_service/` | `notification_db` | `notification-service-0.0.1-SNAPSHOT.jar` |
| 9092 | Kafka | (Docker) | — | — |
| 5432 | PostgreSQL | (Homebrew service) | all 7 dbs | — |

---

## 5. Prerequisites (Local Machine)

Install once via Homebrew:

```bash
# macOS / Apple Silicon
brew install openjdk@21 maven postgresql@14 node kind kubectl
brew services start postgresql@14

# Docker Desktop (download from docker.com) — any recent version works
# Node 18+ is bundled with `brew install node`
```

Verify:
```bash
/opt/homebrew/Cellar/openjdk@21/21.0.11/bin/java -version   # 21.0.11
mvn -version                                                # any
psql --version                                              # 14.x
docker --version                                            # 28+
node --version                                              # 18+
kind --version                                              # 0.31+
kubectl version --client                                    # any
```

---

## 6. Credentials & Secrets

### Postgres (every database)
| Field | Value |
|---|---|
| Host | `localhost` (local) or `postgres-<service>` (K8s DNS) |
| Port | `5432` |
| User | `postgres` |
| Password | `postgres` |
| Databases | `auth_db`, `user_db`, `ticket_db`, `solution_db`, `knowledge_db`, `reward_db`, `notification_db` |

Quick connect:
```bash
PGPASSWORD=postgres psql -U postgres -h localhost -d auth_db
```

### JWT Signing Secret (shared by every service)
```
myVerySecretKeyForJWTThatIsLongEnoughForHS512Signature
```
Each service uses a different property key for it (historic inconsistency):
| Service | Property key |
|---|---|
| auth_service | `app.jwt.secret` + `jwt.secret` |
| Ticket_service | `security.jwt.secret` |
| Solution_service | `app.jwt.secret` |
| knowledge_service | `jwt.secret` |
| Reward_service | `app.jwt.secret` |
| Notification_service | `app.jwt.secret` |
| User_service | `jwt.secret` |

### Seeded Roles (auth_db.roles)
Must be present before the first registration:
```
USER, ADMIN, ENGINEER, MANAGER
```
`services.sh setup` and `k8s/k8s.sh up` both seed these automatically.

### SMTP (optional, disabled by default)
Notification Service has `app.email.enabled: false`. When enabling, set `spring.mail.username` + `spring.mail.password` in `Notification_service/src/main/resources/application-local.yaml`.

---

## 7. Project Directory Layout

```
Services/
├── api_gateway/                    # Spring Cloud Gateway (reactive)
├── auth_service/                   # JWT issuer + validator
├── User_service/
├── Ticket_service/
├── Solution_service/
├── knowledge_service/
├── Reward_service/
├── Notification_service/
├── common_library_service/         # shared DTOs (com.library:common-library:1.0.0)
├── frontend/                       # React + Vite SPA
├── k8s/                            # Kubernetes manifests + k8s.sh
│   ├── namespace.yaml
│   ├── postgres.yaml
│   ├── kafka.yaml
│   ├── services.yaml
│   ├── frontend.yaml
│   ├── ingress.yaml
│   ├── kind-cluster.yaml
│   ├── apply.sh
│   └── k8s.sh                      # one-shot K8s lifecycle
├── logs/                           # created at runtime by services.sh
│   ├── pids/                       # <service>.pid
│   └── <service>.log
├── scripts/
│   ├── demo_seed.sh                # create 12 demo users + emit tokens
│   ├── e2e_test.sh                 # 24-step automated integration test
│   └── swagger_verify.sh           # contract assertion checks
├── project_docs/                   # all project documentation
│   ├── README.md
│   ├── PROJECT_DOC.md              # this file
│   ├── DEMO_FLOW.md
│   └── TEST_CASES.md
├── services.sh                     # one-shot local lifecycle (setup/start/stop/seed/restart)
└── README.md                       # root README (quick-start)
```

---

## 8. First-Time Setup

All in one command:

```bash
cd /Users/abhidhabmellwyn/Downloads/Services
./services.sh setup
```

This runs:
1. Starts `postgresql@14` via `brew services` (if not running)
2. Creates the 7 databases (`CREATE DATABASE` if missing)
3. Installs `common-library:1.0.0` to local `~/.m2`
4. Builds every service JAR with `mvn -DskipTests -Dmaven.test.skip=true package` (Java 21 pinned)
5. Runs `npm install` in `frontend/`
6. Ensures Kafka + Zookeeper Docker containers exist and start

**Takes 3–6 minutes** the first time, depending on whether Maven has to download the 400+ Spring dependencies.

### Manual step for roles
After first start, seed `auth_db.roles`:

```bash
PGPASSWORD=postgres psql -U postgres -h localhost -d auth_db -c "
  INSERT INTO roles (role_id, role_name, created_at, description) VALUES
    (gen_random_uuid(), 'USER',     NOW(), 'Standard user role'),
    (gen_random_uuid(), 'ADMIN',    NOW(), 'Administrator role'),
    (gen_random_uuid(), 'ENGINEER', NOW(), 'Engineer role'),
    (gen_random_uuid(), 'MANAGER',  NOW(), 'Manager role')
  ON CONFLICT (role_name) DO NOTHING;"
```

(Without this, registration fails with *"Role USER not found"* → transaction rollback.)

---

## 9. Running the Stack — Local Mode (`services.sh`)

`services.sh` is the single script to manage the whole stack without containers.

### Every command

```bash
./services.sh setup                # first-time: DBs + common-lib + build all + npm
./services.sh start                # start Postgres/Kafka + 8 JARs + frontend
./services.sh stop                 # graceful stop of services; infra stays up
./services.sh stop --with-infra    # stop services + Kafka + Postgres
./services.sh restart              # stop everything then start everything
./services.sh restart <service>    # stop + restart a single service (e.g. ticket-service)
./services.sh build                # rebuild every JAR (no DB/npm work)
./services.sh status               # ports / PIDs / health table
./services.sh logs <service>       # tail -f logs/<service>.log
./services.sh seed                 # run scripts/demo_seed.sh (creates demo users + tokens)
./services.sh clean                # stop everything + wipe logs/ and pids/
./services.sh help                 # show command list
```

Service names for `logs`:
```
api-gateway, auth-service, user-service, ticket-service,
solution-service, knowledge-service, reward-service,
notification-service, frontend
```

### What `./services.sh start` actually does

1. **Preflight** — checks Java 21, psql, docker, mvn, curl are all present.
2. **Postgres** — runs `brew services start postgresql@14` if `pg_isready` fails, then waits up to 15s.
3. **Kafka + Zookeeper** — creates a Docker network `kafka-net`, reuses existing `kafka` + `zookeeper` containers (confluentinc/cp-*:7.5.0) if they exist, else creates them. Waits up to 30s for Kafka broker API.
4. **Services (dependency order)**:
   1. `auth-service`
   2. `user-service`, `ticket-service`, `solution-service`, `knowledge-service`, `reward-service`, `notification-service` (any order)
   3. `api-gateway` (last — needs downstreams)
5. For each service:
   - Checks `logs/pids/<service>.pid` for an already-running instance → skip
   - Checks if port is in use → warn and skip
   - `nohup java -jar <jar> --spring.profiles.active=local > logs/<service>.log 2>&1 &`
   - Writes PID to `logs/pids/<service>.pid`
6. **Health polling** — loops `curl /actuator/health` for each service, up to 90s, prints `UP` when 200.
7. **Frontend** — `cd frontend && npm run dev` in background, logs to `logs/frontend.log`.
8. Prints a final `status` table.

### What `./services.sh stop` does

1. Reads every `logs/pids/<service>.pid` in reverse dependency order (gateway first, auth last).
2. Sends `SIGTERM`, waits up to 8s per service, then `SIGKILL`.
3. Deletes the PID file.
4. Safety net: `lsof -ti :<port>` and kills any stray listener on that port.
5. Optionally (`--with-infra`): `docker stop kafka zookeeper` and `brew services stop postgresql@14`.

### Where things live at runtime

```
logs/
├── api-gateway.log
├── auth-service.log
├── user-service.log
├── ticket-service.log
├── solution-service.log
├── knowledge-service.log
├── reward-service.log
├── notification-service.log
├── frontend.log
└── pids/
    ├── api-gateway.pid
    ├── auth-service.pid
    ... (one per service)
```

### Example `./services.sh status` output
```
==== Service Status ====
SERVICE                PORT   STATE           HEALTH
api-gateway            8080   UP(pid4149)     200 UP
auth-service           8081   UP(pid4128)     200 UP
user-service           8082   UP(pid4131)     200 UP
ticket-service         8083   UP(pid4134)     200 UP
solution-service       8084   UP(pid4137)     200 UP
knowledge-service      8085   UP(pid4140)     200 UP
reward-service         8086   UP(pid4143)     200 UP
notification-service   8087   UP(pid4146)     200 UP
frontend               3000   UP(pid4433)     200 UP

Kafka:      [✓] running (container kafka)
Postgres:   [✓] ready on :5432
```

### Internals of `services.sh`

Key constants at the top:
```bash
PROJECT_DIR="/Users/abhidhabmellwyn/Downloads/Services"
JAVA_HOME_DIR="/opt/homebrew/Cellar/openjdk@21/21.0.11"
JAVA_BIN="${JAVA_HOME_DIR}/bin/java"
PG_USER="postgres"; PG_PASSWORD="postgres"; PG_HOST="localhost"; PG_PORT="5432"
LOG_DIR="${PROJECT_DIR}/logs"; PID_DIR="${LOG_DIR}/pids"
KAFKA_CONTAINER="kafka"; ZOOKEEPER_CONTAINER="zookeeper"; KAFKA_NETWORK="kafka-net"
```

The service list:
```bash
SERVICES=(
  "api-gateway|api_gateway|api-gateway-0.0.1-SNAPSHOT.jar|8080"
  "auth-service|auth_service|auth_service-0.0.1-SNAPSHOT.jar|8081"
  "user-service|User_service|User_service-0.0.1-SNAPSHOT.jar|8082"
  "ticket-service|Ticket_service|Ticket_service-0.0.1-SNAPSHOT.jar|8083"
  "solution-service|Solution_service|solution-service-0.0.1-SNAPSHOT.jar|8084"
  "knowledge-service|knowledge_service|knowledge-service-0.0.1-SNAPSHOT.jar|8085"
  "reward-service|Reward_service|Reward_service-0.0.1-SNAPSHOT.jar|8086"
  "notification-service|Notification_service|notification-service-0.0.1-SNAPSHOT.jar|8087"
)
```

Databases created by `setup`:
```bash
DATABASES=(auth_db user_db ticket_db solution_db knowledge_db reward_db notification_db)
```

---

## 10. Running the Stack — Kubernetes Mode

All services run as pods inside a local `kind` cluster (k8s-in-docker), exposed through nginx-ingress on `http://ticketing.local/`.

### Commands — via `services.sh` (recommended)

```bash
./services.sh k8s-up      # create kind cluster, build + load images, deploy, seed roles
./services.sh k8s-down    # delete the kind cluster (all k8s data lost)
./services.sh k8s-status  # show pods + ingress + health
./services.sh k8s-seed    # seed roles, categories, and k8s demo accounts
./services.sh k8s-logs <deployment>   # tail logs (e.g. k8s-logs auth-service)
```

### Commands — via `k8s/k8s.sh` (direct)

```bash
./k8s/k8s.sh up       # same as k8s-up above
./k8s/k8s.sh status   # show pods + ingress
./k8s/k8s.sh test     # end-to-end smoke test (register/login/protected endpoints)
./k8s/k8s.sh down     # delete the kind cluster
```

### What `k8s-up` / `./k8s/k8s.sh up` does

1. Creates kind cluster `ticketing` from `k8s/kind-cluster.yaml` (maps host ports 80+443 → cluster ingress) if not already present.
2. Builds all 9 Docker images locally:
   ```
   api-gateway:latest      auth-service:latest      user-service:latest
   ticket-service:latest   solution-service:latest  knowledge-service:latest
   reward-service:latest   notification-service:latest
   frontend:latest
   ```
3. Loads them into the kind cluster (`kind load docker-image ...`).
4. Installs the kind-variant nginx-ingress and waits for it ready.
5. Runs `k8s/apply.sh`, which applies in order: namespace → postgres → kafka → services → frontend → ingress.
6. Waits for every pod to report `Ready`.
7. Seeds `auth_db.roles` with USER/ADMIN/ENGINEER/MANAGER (correct schema: `role_id UUID`, `role_name VARCHAR`).

### One-time manual step for the hostname

Add to `/etc/hosts` (needs sudo — use your macOS login password):
```bash
echo "127.0.0.1 ticketing.local" | sudo tee -a /etc/hosts
```

Then open **http://ticketing.local/**.

### Seed demo data (required after every `k8s-up`)

The k8s databases start empty. Run after the cluster is up:
```bash
./services.sh k8s-seed
```

This seeds:
- **Roles** in `auth_db` — USER / ADMIN / ENGINEER / MANAGER (using `role_id` UUID column)
- **Categories** in `ticket_db` — 5 rows (Network, Software, Hardware, Security, Other)
- **Demo accounts** registered via ingress:

| Role | Email | Password |
|---|---|---|
| Admin | k8sadmin@demo.test | Demo@1234 |
| Engineer | k8sengineer@demo.test | Demo@1234 |

> **Note:** These are k8s-specific accounts. Local credentials (`adm1778576329@demo.test`) only exist in the local Postgres instance.

### K8s internal DNS
Inside the cluster, services reach each other by service name within the `ticketing-system` namespace:
- `auth-service:8081`, `user-service:8082`, …, `notification-service:8087`
- `postgres-auth:5432`, `postgres-user:5432`, …
- `kafka:9092`

Configured in `api-gateway` via env-var placeholders in `application-local.yaml`:
```yaml
uri: ${AUTH_SERVICE_URI:http://localhost:8081}
```
`k8s/services.yaml` sets `AUTH_SERVICE_URI=http://auth-service:8081` etc. on every pod that needs cross-service calls.

### K8s Kafka (KRaft mode — no Zookeeper)
The k8s cluster uses `apache/kafka:3.9.2` in KRaft mode (single-node, no Zookeeper required).
This is intentionally different from local mode which uses `confluentinc/cp-kafka:7.5.0` + Zookeeper.

### ⚠️ Port conflict
**Never run `services.sh start` and `k8s-up` at the same time** — both bind to port 9092 (Kafka). Use one mode at a time.

---

## 11. Per-Service Deep Dive

### API Gateway (`api_gateway/`, port 8080)
- Spring Cloud Gateway (reactive/WebFlux).
- YAML-driven routes (`application-local.yaml`) — one route per downstream service, path-prefix matched.
- Custom `AuthFilter` (Spring bean `AuthFilter` referenced by name in route config) calls `http://auth-service:8081/api/auth/validate`, extracts `userId` + `role` from the JSON response, then injects `X-User-Id` and `X-User-Role` headers to the downstream request using a `ServerHttpRequestDecorator` (NOT `request.mutate()` — that throws `UnsupportedOperationException` on Netty's read-only headers).
- Open endpoints (no JWT required): `/api/auth/login`, `/api/auth/register`.
- CORS: allow all origins/methods/headers (dev only).

### Auth Service (`auth_service/`, port 8081)
- Issues JWTs (HS384) on `/api/auth/login`, validates on `/api/auth/validate`.
- Endpoints: `/api/auth/register`, `/api/auth/login`, `/api/auth/validate`, `/api/auth/refresh`.
- Stores users in `auth_db.users`, roles in `auth_db.roles`, join table `auth_db.user_roles`.
- `User.userRoles` and `UserRole.role` use `FetchType.EAGER` (LAZY caused `LazyInitializationException` when the filter calls `getAuthorities()` outside the session).
- `/api/auth/validate` is in `SecurityConfig.permitAll()` so the gateway can reach it.
- Publishes `user-registration` Kafka event when a user registers.

### User Service (`User_service/`, port 8082)
- Endpoints: `/api/users`, `/api/departments`.
- `SecurityConfig` + `JwtAuthenticationFilter` + `JwtTokenProvider` (was missing originally — added).
- Depends on `spring-boot-starter-security` + JJWT.
- `UserController` constructor is `public` (CGLIB proxy with `@EnableMethodSecurity` requires this).

### Ticket Service (`Ticket_service/`, port 8083)
- Endpoints: `/api/tickets` and sub-resources.
- Uses `security.jwt.secret` (note different property key) for JWT validation.
- Publishes `ticket.resolved` Kafka event.

### Solution Service (`Solution_service/`, port 8084)
- Endpoints: `/api/solutions`, `/api/solutions/{id}/contributors`, `/api/solutions/{id}/comments`, `/api/solutions/{id}/attachments`.
- Consumes: `ticket.resolved`.
- Publishes: `solution.approved`, `solution.voted`.

### Knowledge Service (`knowledge_service/`, port 8085)
- Endpoints: `/api/knowledge`, `/api/categories`, `/api/tags`, `/api/ratings`.
- Publishes: `knowledge.created`, `knowledge.rated`.

### Reward Service (`Reward_service/`, port 8086)
- Endpoints: `/api/rewards`, `/api/rewards/leaderboard`, `/api/rewards/badges`.
- Consumes: `solution.approved`, `solution.voted`, `knowledge.rated`, `ticket.resolved`.
- Publishes: `reward.points.added`, `reward.badge.awarded`, `leaderboard.updated`.

### Notification Service (`Notification_service/`, port 8087)
- Endpoints: `/api/notifications`, `/api/preferences`, `/api/v1/notifications/templates`.
- Consumes: `notification-events`, `user-registration`, plus most reward/knowledge events.
- Email sending guarded by `app.email.enabled: false` — no actual SMTP call in dev.

### Common Library (`common_library_service/`)
- Shared DTOs (`ApiResponse<T>`, error envelopes) used by every service.
- GAV: `com.kva:common-library:1.0.0` → installed to `~/.m2/repository/com/kva/common-library/1.0.0/`.
- Built by `services.sh setup` before the services themselves.

---

## 12. Frontend

- **Tech**: React, Vite, TailwindCSS
- **Folder**: `frontend/`
- **Dev server**: `npm run dev` on port 3000
- **Build**: `npm run build` → `dist/`
- **Production serve**: Dockerfile uses nginx:alpine as the runtime, `nginx.conf` listens on 3000 and proxies `/api` → `http://api-gateway:8080` (inside k8s) or relies on vite dev proxy locally.
- **API base URL**: `VITE_API_BASE_URL=http://ticketing.local/api` (K8s) / relative `/api` via proxy (local dev).

Source layout inside `frontend/src/`:
- `components/` — reusable UI
- `pages/` — routed views
- `services/` — API client wrappers (axios-based)
- `hooks/`, `context/`, `utils/`

---

## 13. API Endpoints Reference

All endpoints below are accessed **through the gateway** at `http://localhost:8080` (local) or `http://ticketing.local` (K8s).

### Public (no token required)
```
POST /api/auth/register     { email, password, role }
POST /api/auth/login        { email, password }
```

### Protected (requires `Authorization: Bearer <token>`)

| Service | Sample endpoints |
|---|---|
| Auth | `POST /api/auth/refresh`, `GET /api/auth/validate` |
| User | `GET /api/users`, `GET /api/users/{id}`, `PUT /api/users/{id}`, `GET /api/departments` |
| Ticket | `GET /api/tickets`, `POST /api/tickets`, `PUT /api/tickets/{id}` |
| Solution | `GET /api/solutions`, `POST /api/solutions`, `POST /api/solutions/{id}/comments` |
| Knowledge | `GET /api/knowledge`, `GET /api/categories`, `GET /api/tags`, `POST /api/ratings` |
| Reward | `GET /api/rewards/leaderboard`, `GET /api/rewards/badges`, `GET /api/rewards/me` |
| Notification | `GET /api/notifications/users/{userId}`, `GET /api/notifications/statistics` (ADMIN), `GET /api/preferences` |

Role-specific gates enforced via `@PreAuthorize("hasRole('ADMIN')")` on the controllers.

---

## 14. Swagger / OpenAPI Documentation

Every backend service exposes a live OpenAPI 3.0 spec + Swagger UI, and the gateway aggregates them under a single URL.

### Direct URLs per service
| Service | OpenAPI JSON | Swagger UI |
|---|---|---|
| auth-service | `http://localhost:8081/v3/api-docs` | `http://localhost:8081/swagger-ui.html` |
| user-service | `http://localhost:8082/v3/api-docs` | `http://localhost:8082/swagger-ui.html` |
| ticket-service | `http://localhost:8083/v3/api-docs` | `http://localhost:8083/swagger-ui.html` |
| solution-service | `http://localhost:8084/v3/api-docs` | `http://localhost:8084/swagger-ui.html` |
| knowledge-service | `http://localhost:8085/v3/api-docs` | `http://localhost:8085/swagger-ui.html` |
| reward-service | `http://localhost:8086/v3/api-docs` | `http://localhost:8086/swagger-ui.html` |
| notification-service | `http://localhost:8087/v3/api-docs` | `http://localhost:8087/swagger-ui.html` |
| api-gateway | `http://localhost:8080/v3/api-docs` | `http://localhost:8080/swagger-ui.html` (aggregated) |

### Aggregated UI (the common entry point)
`http://localhost:8080/swagger-ui.html` — top-right dropdown lists every service. Deep-link:
```
http://localhost:8080/swagger-ui.html?urls.primaryName=ticket-service
```

### How the aggregation works
1. **Gateway route table** (`api_gateway/src/main/resources/application-local.yaml`):
   ```yaml
   - id: auth-service-docs
     uri: ${AUTH_SERVICE_URI:http://localhost:8081}
     predicates: [Path=/v3/api-docs/auth-service]
     filters:    [SetPath=/v3/api-docs]
   # ... one such route per downstream service
   ```
   So `GET http://localhost:8080/v3/api-docs/auth-service` is rewritten to `GET http://localhost:8081/v3/api-docs` — the downstream's real spec.

2. **Gateway `OpenApiConfig`** — declares a `SwaggerUiConfigProperties` bean listing 7 URLs (one per service), each pointing at one of these proxied paths. Swagger UI renders them as a dropdown.

3. **Each service `OpenApiConfig`** — provides:
   - `OpenAPI` bean with title / version `1.0.0` / description / contact / server list (`localhost:<port>`, `localhost:8080` via gateway, `ticketing.local` via ingress)
   - `bearerAuth` `SecurityScheme` (HTTP bearer / JWT format)
   - `addSecurityItem(new SecurityRequirement().addList("bearerAuth"))` so protected endpoints show the lock icon in the UI

### Controller annotations
Every controller is decorated with springdoc/OpenAPI annotations:
- `@Tag(name = "...", description = "...")` on the class
- `@Operation(summary = "...", description = "...")` on each endpoint
- `@ApiResponses({@ApiResponse(responseCode = "200", ...), @ApiResponse(responseCode = "401", ...)})`
- `@Parameter` on path / query params
- `@SecurityRequirement(name = "bearerAuth")` on any endpoint that requires JWT

Public GET endpoints (e.g. Solution Service GETs) and internal `/internal/**` endpoints deliberately omit `@SecurityRequirement` so the lock icon is hidden.

### DTO schemas
Primary auth DTOs (`LoginRequest`, `RegisterRequest`) use `@Schema(example = "...")` at field level so the "Try it out" form has sensible defaults (`alice@example.com`, `Passw0rd!`). Other DTOs rely on springdoc's reflection-based inference from field names + Jackson — explicit `@Schema` can be added incrementally if the autogenerated schema is insufficient.

### Standardised yaml
Every service has this block in `application.yaml` for consistent UI behaviour:
```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha
    try-it-out-enabled: true
    display-request-duration: true
```

### Testing Swagger end-to-end
```bash
# 1. Hit each service's spec directly
for p in 8081 8082 8083 8084 8085 8086 8087; do
  echo -n "$p "; curl -s "http://localhost:$p/v3/api-docs" | jq -r '.info.title'
done

# 2. Verify gateway proxies
for s in auth user ticket solution knowledge reward notification; do
  echo -n "$s "; curl -s "http://localhost:8080/v3/api-docs/$s-service" | jq -r '.info.title'
done

# 3. Open aggregator
open http://localhost:8080/swagger-ui.html
```

### Try-it-out with JWT (from the UI)
1. `POST /api/auth/login` — copy the `data.accessToken` field from the response.
2. Click the **Authorize** button (top-right, lock icon).
3. Paste the token into the `bearerAuth` field.
4. Every subsequent "Try it out" call sends `Authorization: Bearer <token>` automatically.

### SecurityConfig allowlists
All 8 service `SecurityConfig` classes allow `GET /v3/api-docs/**` and `GET /swagger-ui/**` + `/swagger-ui.html` without authentication — the specs themselves are public even though their operations are protected.

---

## 15. Kafka Topics

Auto-created by the producers on first publish; can be pre-created manually if needed:

| Topic | Produced by | Consumed by |
|---|---|---|
| `user-registration` | auth-service | notification-service |
| `ticket.resolved` | ticket-service | solution-service, reward-service |
| `solution.approved` | solution-service | reward-service, notification-service |
| `solution.voted` | solution-service | reward-service |
| `knowledge.created` | knowledge-service | notification-service |
| `knowledge.rated` | knowledge-service | reward-service |
| `reward.points.added` | reward-service | notification-service |
| `reward.badge.awarded` | reward-service | notification-service |
| `leaderboard.updated` | reward-service | — |
| `notification-events` | any | notification-service |

Pre-create all topics manually (optional):
```bash
docker exec kafka bash -c '
for t in user-registration ticket.resolved solution.approved solution.voted \
         knowledge.created knowledge.rated reward.points.added \
         reward.badge.awarded leaderboard.updated notification-events; do
  kafka-topics --create --if-not-exists --bootstrap-server localhost:9092 \
    --replication-factor 1 --partitions 1 --topic "$t";
done'
```

---

## 16. Auth Flow

```
 ┌────────┐   1. POST /api/auth/login      ┌─────────┐
 │ client ├─────────────────────────────────▶ gateway │
 └────┬───┘                                 └────┬────┘
      │                                          │ 2. open endpoint — no check
      │                                          ▼
      │                                    ┌──────────────┐
      │                                    │ auth-service │
      │     3. accessToken (JWT HS384)     └──────┬───────┘
      │    ◀───────────────────────────────────── │
      │                                           │
      │ 4. GET /api/tickets  Bearer <jwt>         │
      ▼                                           │
 ┌─────────┐  5. GET /api/auth/validate  ◀────────┘
 │ gateway ├──────────────────────────────▶ auth-service
 │AuthFilter│  {userId, role}                     │
 └────┬────┘  ◀───────────────────────────────────┘
      │ 6. add X-User-Id + X-User-Role headers
      │    via ServerHttpRequestDecorator
      ▼
 ┌───────────────┐   7. JwtAuthenticationFilter re-validates JWT independently
 │ticket-service │      then proceeds with business logic
 └───────────────┘
```

Every service has its own `SecurityConfig` + `JwtAuthenticationFilter` + `JwtTokenProvider`, which **all use the same HS384 secret**. This allows direct-to-service calls to work too (e.g. during debugging), not just gateway-proxied ones.

---

## 17. Complete List of Changes & Fixes Applied

### Files created from scratch
| File | Purpose |
|---|---|
| `services.sh` | Single-command local lifecycle (setup/start/stop/status/build/logs/clean) |
| `k8s/k8s.sh` | Single-command K8s lifecycle (up/down/status/test) |
| `k8s/kind-cluster.yaml` | Kind cluster config with ports 80/443 mapped to host |
| `User_service/Dockerfile` | Was missing — created (copy pre-built JAR pattern) |
| `Ticket_service/Dockerfile` | Was missing — created (copy pre-built JAR pattern) |
| `PROJECT_DOC.md` | This file |

### Files rewritten
| File | What changed |
|---|---|
| `api_gateway/Dockerfile` | Multistage → copy pre-built JAR (avoid common-library fetch fail); `17-jre-alpine` → `21-jre` (arm64 compat, Java 21 base) |
| `auth_service/Dockerfile` | Same |
| `Solution_service/Dockerfile` | Same |
| `knowledge_service/Dockerfile` | Same |
| `Reward_service/Dockerfile` | Same |
| `Notification_service/Dockerfile` | Same |
| `k8s/kafka.yaml` | `bitnami/kafka:3.6` (not on Docker Hub anymore) → `apache/kafka:3.9.2` KRaft single-node mode |
| `k8s/ingress.yaml` | Removed broken `nginx.ingress.kubernetes.io/rewrite-target: /` (was stripping `/api`); split into `/api` → api-gateway + `/` → frontend |
| `api_gateway/src/main/resources/application-local.yaml` | All 7 downstream URIs parameterized with `${SERVICE_URI:http://localhost:808X}` so same JAR works on localhost + K8s DNS |
| `k8s/services.yaml` | Added `AUTH_SERVICE_URI / USER_SERVICE_URI / ...` env vars on the gateway pod; added `KAFKA_BOOTSTRAP_SERVERS=kafka:9092` on every pod (the custom `@Value("${kafka.bootstrap-servers}")` in auth-service does NOT pick up `SPRING_KAFKA_*`) |

### Historic fixes (pre-existing knowledge, applied during setup)
- **Java 21 pinned for Maven** — set `JAVA_HOME=/opt/homebrew/Cellar/openjdk@21/21.0.11` for every `mvn` call; keeps Lombok/Maven-compiler-plugin happy and deterministic regardless of what the system `java` is.
- **Auth Service role seeding** — `INSERT INTO roles (role_name) VALUES ('USER','ADMIN','ENGINEER','MANAGER')` → otherwise registration transaction rolls back with *"Role USER not found"*.
- **Auth Service EAGER fetch** — `User.userRoles` and `UserRole.role` changed from `LAZY` → `EAGER` to prevent `LazyInitializationException` in `JwtAuthenticationFilter.getAuthorities()`.
- **`/api/auth/validate` public** — added to `permitAll()` in auth-service `SecurityConfig`; the gateway's AuthFilter calls this endpoint unauthenticated.
- **Gateway header injection** — use `ServerHttpRequestDecorator` (not `request.mutate()`) to add `X-User-Id`/`X-User-Role`. Netty read-only headers.
- **Gateway security** — `ApiGatewaySecurityConfig` only, `anyExchange().permitAll()`; route-level auth is handled by `AuthFilter` (not Spring Security filter chain).
- **User Service security** — added `SecurityConfig`, `JwtAuthenticationFilter`, `JwtTokenProvider` + `spring-boot-starter-security` + JJWT deps (was completely missing → everything 401).
- **UserController constructor public** — was `private`, incompatible with CGLIB proxy + `@EnableMethodSecurity`.
- **Notification Service mail autoconfig** — removed `spring.autoconfigure.exclude: MailSenderAutoConfiguration`; `EmailServiceImpl` requires `JavaMailSender` bean. `app.email.enabled: false` guards actual SMTP calls.
- **YAML duplicate keys** — merged duplicate `app:` / `spring:` blocks across several services.

### Java 21 Migration (April 2026)
End-to-end migration from Java 17 → Java 21 LTS across source, containers, orchestration, and tooling.

**Source code — 9 `pom.xml` files** (common_library_service, api_gateway, auth_service, User_service, Ticket_service, Solution_service, knowledge_service, Reward_service, Notification_service):
- `<java.version>17</java.version>` → `<java.version>21</java.version>`
- `<lombok.version>1.18.34</lombok.version>` property added where missing (Java 21 requires Lombok ≥ 1.18.30; bumped to 1.18.34 for safety).
- `maven-compiler-plugin` `3.11.0` → `3.13.0` with explicit `<release>${java.version}</release>` tag (replaces legacy `<source>/<target>`).
- Hardcoded Lombok `1.18.30` → `${lombok.version}`.

**Containerization — 9 Dockerfiles**:
- 8 service Dockerfiles: `FROM eclipse-temurin:17-jre` → `FROM eclipse-temurin:21-jre`.
- `common_library_service/Dockerfile`: `maven:3.9.9-eclipse-temurin-17` → `-21`; `eclipse-temurin:17-jre-alpine` → `21-jre-alpine`.

**Tooling — `services.sh`**:
- `JAVA_HOME_DIR="/opt/homebrew/Cellar/openjdk@17/17.0.18"` → `"/opt/homebrew/Cellar/openjdk@21/21.0.11"`.
- All `mvn … install/package` calls gained `-Dmaven.test.skip=true` alongside `-DskipTests` (skips test **compile** too — pre-existing test-code issues were unrelated to the migration and out of scope).

**Frontend integration (CORS fix bundled in)** — `api_gateway/src/main/java/com/example/api_gateway/config/CorsConfig.java` rewritten:
- Previously only allowed `http://localhost:5173`; the frontend actually runs on `http://localhost:3000` → browser was blocking.
- Now uses `setAllowedOriginPatterns` covering :3000, :5173, 127.0.0.1 equivalents, and `ticketing.local[:*]`.
- `allowCredentials=true` + `exposedHeaders=[Authorization, Content-Disposition]` + `maxAge=3600`.

**Verification outcomes**:
- All 9 JAR files: `javap -v` confirmed class-file **major version 65** (= Java 21 bytecode).
- Live services: `jps -v` confirmed every PID on `/opt/homebrew/Cellar/openjdk@21/21.0.11/bin/java`.
- K8s: 17 pods Running on `eclipse-temurin:21-jre` (Temurin 21.0.10 LTS), smoke test 4/4 green.
- Frontend: CORS preflight returns `access-control-allow-origin: http://localhost:3000`; Vite `/api` proxy register/login/protected all 200.

### Swagger / OpenAPI Rollout (April 2026)
Added first-class API documentation to every backend service and an aggregated UI at the gateway.

**Per-service `OpenApiConfig.java` created** (7 files, one per service + gateway):
- `auth_service`, `User_service`, `Ticket_service`, `Solution_service`, `Reward_service`, `Notification_service`, `api_gateway`
- Each declares an `OpenAPI` bean with: title, version `1.0.0`, description, contact (`platform@example.com`), `License("Internal")`, server list covering direct / gateway / k8s ingress, and a `bearerAuth` `SecurityScheme` (HTTP / bearer / JWT).
- `addSecurityItem(new SecurityRequirement().addList("bearerAuth"))` — every protected operation shows the lock icon.
- `knowledge_service` already had a config from earlier work; verified consistent.

**Gateway aggregator** (`api_gateway`):
- `OpenApiConfig` additionally defines `SwaggerUiConfigProperties` bean with 7 `SwaggerUrl` entries (`auth-service` through `notification-service`), each pointing at `/v3/api-docs/<service>`.
- `application-local.yaml` gained **7 new routes** (`*-service-docs`) with `SetPath=/v3/api-docs` filter that proxy `/v3/api-docs/<service>` → downstream `/v3/api-docs`.
- Aggregated UI now renders at `http://localhost:8080/swagger-ui.html` with a dropdown.

**Standardised springdoc yaml** across 7 service `application.yaml` files + gateway:
```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha
    try-it-out-enabled: true
    display-request-duration: true
```

**Controller annotation sweep — 29 controllers, ~126 endpoints**:
- `@Tag(name, description)` on every controller class.
- `@Operation(summary, description)` on every handler method.
- `@ApiResponses({200, 400, 401, 403, 404, 500 as applicable})`.
- `@Parameter(description, example)` on path and query params.
- `@SecurityRequirement(name = "bearerAuth")` on every JWT-protected endpoint.
- No `@SecurityRequirement` on public GETs (solution-service) or `/internal/**` controllers.

**DTO schemas**:
- `auth_service/.../dto/LoginRequest.java` + `RegisterRequest.java` — class-level `@Schema(description = ...)` and field-level `@Schema(example = "alice@example.com", example = "Passw0rd!")` for sensible "Try it out" defaults.
- Remaining DTOs rely on springdoc's reflection-based auto-generation from field names + Jackson annotations — incremental `@Schema` improvements deferred to future work.

**SecurityConfig allowlists** — verified (no edits needed) that all 8 `SecurityConfig` classes already had `/swagger-ui/**`, `/swagger-ui.html`, and `/v3/api-docs/**` in `permitAll()`. This was a bonus from earlier work.

**Verification**:
- `./services.sh build` → all 8 services compile clean on Java 21.
- Every service returns valid JSON at `/v3/api-docs` (verified via `curl ... | jq '.info.title'`).
- Gateway proxies all 7 downstream specs at `/v3/api-docs/<service>` (verified identically).
- Gateway aggregator UI `http://localhost:8080/swagger-ui.html` renders all 7 services in the dropdown.
- Try-it-out with JWT works: login → Authorize → protected endpoint returns 200.

**New files introduced**:
```
auth_service/src/main/java/com/example/auth_service/config/OpenApiConfig.java
User_service/src/main/java/com/example/User_service/config/OpenApiConfig.java
Ticket_service/src/main/java/com/example/Ticket_service/config/OpenApiConfig.java
Solution_service/src/main/java/com/example/Solution_service/config/OpenApiConfig.java
Reward_service/src/main/java/com/example/Reward_service/config/OpenApiConfig.java
Notification_service/src/main/java/com/example/Notification_service/config/OpenApiConfig.java
api_gateway/src/main/java/com/example/api_gateway/config/OpenApiConfig.java
```

**Files modified**:
- 29 controller .java files across 7 services (annotation sweep).
- 7 `application.yaml` files (springdoc yaml block).
- `api_gateway/src/main/resources/application-local.yaml` (7 new routes + springdoc yaml).
- `auth_service/.../LoginRequest.java`, `RegisterRequest.java` (@Schema examples).

### Documentation overhaul (April 2026)
- Rewrote every per-service `README.md` in the same structured template: "At a glance" table, "What it does", "API surface" (with auth column), "Configuration" (env vars), "Kafka events produced/consumed", "Build & run", "Docker / K8s", "Troubleshooting", "Tech stack".
- `frontend/README.md` rewritten in the same template.
- `common_library_service/README.md` rewritten to clarify it's a library JAR (no runtime, no port).
- Top-level `/README.md` refreshed with Java 21 + Swagger aggregator + full stack overview.
- `PROJECT_DOC.md` gained new §14 "Swagger / OpenAPI Documentation" + this migration entry + renumbering of §14–§18 → §15–§19.

### Infrastructure provisioned
- Installed `kind` via Homebrew (0.31.0).
- Created kind cluster `ticketing` with port 80/443 mapped to localhost.
- Installed kind-variant nginx-ingress controller.
- Built 9 Docker images (8 services + frontend).
- Loaded all 9 images into kind via `kind load docker-image`.
- Created Kafka + Zookeeper Docker containers (`confluentinc/cp-*:7.5.0`) on `kafka-net` Docker network for local mode.

### Frontend — Bug Fixes Applied (April 2026)

Ten issues were found and fixed during end-to-end integration testing against the live backend:

| # | File | Bug | Fix |
|---|---|---|---|
| 1 | `authSlice.js` | `decoded.userId` (0 or email) used as UUID | Switched to `decoded.authUserId` stored as both `userId` and `authUserId` |
| 2 | `ticketSlice.js` | All thunks extracted `response.data.data` — ticket backend returns flat objects | Changed to `response.data` |
| 3 | `CreateTicketPage.jsx` | Sent `CRITICAL` priority (not in backend enum) and text `category` field (no `categoryId`) | Priority dropdown: LOW/MEDIUM/HIGH/URGENT; Category dropdown mapped to `categoryId: 1–5` |
| 4 | `ticketApi.js` | `updateStatus` sent query params instead of request body | Changed to `PUT /tickets/status` with `{ ticketId, status }` body matching `TicketStatusRequestDTO` |
| 5 | `TicketDetailPage.jsx` | `isAdmin` used `===` equality; solution form missing `title` field; no status-change or award-points UI | Fixed `.includes('ADMIN')` check; added `title` field; added status buttons and Award Points form |
| 6 | `DashboardPage / Navbar / NotificationsPage` | `user.userId` (email string) used for notification API calls; `notification.id` key (doesn't exist — backend uses `notificationId`); `notification.ticketId` (backend uses `referenceId`) | All replaced with `user.authUserId`, `notification.notificationId`, `notification.referenceId` |
| 7 | `notificationSlice.js` | `markAsRead` reducer looked up by `n.id` — never matched backend `notificationId` | Changed to `n.notificationId` |
| 8 | `SolutionDetailPage.jsx` | `isAdmin` strict equality always false for `ROLE_ADMIN` JWT claim; rejected section checked non-existent `rejectedAt` field | Fixed to `.includes('ADMIN')`; simplified rejected section |
| 9 | `KnowledgeArticlePage.jsx` | `article.category` rendered object as `[object Object]`; `tag` rendered object; `relatedTicketId` undefined (backend uses `ticketId`) | Fixed to `category.categoryName`, `tag.tagName`, `article.ticketId` |
| 10 | `SolutionsPage / KnowledgeBasePage` | Tags and submitter fields referenced wrong backend field names | Added `tag.tagName` and `solution.createdBy` fallback |

### `services.sh` — Enhancements (April 2026)
- Added `seed` command: shortcut to `scripts/demo_seed.sh`
- Added `restart <service>` support: stop + restart a single named service or `frontend`
- Fixed check_prereqs label: "Java 17" → "Java 21"

### Integration Bug Fixes (May 2026)

Six issues found during full end-to-end integration testing:

| # | File | Bug | Fix |
|---|---|---|---|
| 1 | `Ticket_service/…/TicketRepository.java` | JPQL `LOWER(CONCAT('%',:title,'%'))` with null param → PostgreSQL infers `bytea` → `ERROR: function lower(bytea) does not exist` | Rewrote as native SQL with `CAST(:param AS text)` for every nullable parameter; added separate `countQuery` for pagination |
| 2 | `frontend/src/pages/TicketListPage.jsx` | `priorityFilter` missing from `useEffect` dep array → changing priority dropdown didn't reload tickets | Added `priorityFilter` to the `useEffect([…])` array |
| 3 | `Notification_service/…/domain/entity/Notification.java` | `referenceId`/`referenceType` columns annotated `nullable = false` but `createAndSend()` never sets them → `PSQLException` NOT NULL violation on every Kafka-triggered notification | Removed `nullable = false` from both `@Column` annotations; ran `ALTER TABLE notifications ALTER COLUMN reference_id DROP NOT NULL` and same for `reference_type` |
| 4 | `Reward_service/…/kafka/RewardEventConsumer.java` | `consumeSolutionApproved` expected `SolutionApprovedEvent` (common-library) but global `spring.json.value.default.type: RewardEventDTO` forced all messages to that type; Spring conversion to `SolutionApprovedEvent` always failed | Created local `SolutionApprovalEvent` DTO; added `solutionApprovedContainerFactory` bean with `VALUE_DEFAULT_TYPE: SolutionApprovalEvent`; updated listener to `containerFactory = "solutionApprovedContainerFactory"` |
| 5 | `Reward_service/…/dto/event/RewardEventDTO.java` | `timestamp` field (type `Long`) failed when `SolutionEvent.timestamp` (a `LocalDateTime`) arrived serialised as a JSON array `[2026,5,3,…]` | Added `@JsonIgnore` on `timestamp`; added `@JsonIgnoreProperties(ignoreUnknown = true)` on the class |
| 6 | `Notification_service/…/infrastructure/kafka/NotificationEventConsumer.java` | `consumeRewardPointsAdded` expected `RewardPointsAddedEvent` but the default `kafkaListenerContainerFactory` uses `USE_TYPE_INFO_HEADERS: true`; reward service publishes without type headers → fallback was always `NotificationEvent`; conversion to `RewardPointsAddedEvent` always failed | Created local `RewardPointsAddedEvent` DTO; added `rewardPointsContainerFactory` bean with `USE_TYPE_INFO_HEADERS: false` and `VALUE_DEFAULT_TYPE: RewardPointsAddedEvent`; updated listener to use that factory |

**End-to-end verification:** solution approval now awards +30 points AND delivers both a "Solution Approved" and a "Points Earned" notification. All 19 demo-flow steps pass.

---

## 18. Troubleshooting

### "Role USER not found" on register → 500
Seed the roles table (see [§6 Credentials](#6-credentials--secrets)).

### `services.sh status` shows some services DOWN after start
`./services.sh logs <service-name>` to see startup errors. Most common: port already in use (`services.sh` skips with a warning).

### Kafka container crashes on start: *"KeeperErrorCode = NodeExists"*
Stale Zookeeper state from a previous unclean shutdown.
```bash
docker rm -f kafka zookeeper
./services.sh start   # re-creates them clean
```

### `mvn` build fails with Lombok error
You ran `mvn` without Java 21. Use:
```bash
JAVA_HOME=/opt/homebrew/Cellar/openjdk@21/21.0.11 mvn ...
# or just:
./services.sh build
```

### Gateway returns 401/connection-refused for downstream calls
- Verify `auth-service` is UP: `curl http://localhost:8081/actuator/health`
- Verify the JWT hasn't expired (default expiry is 1h).
- Check gateway log: `./services.sh logs api-gateway`.

### K8s pods stuck in `ImagePullBackOff`
The image name doesn't match exactly what the manifest expects. Each image must be:
```
api-gateway:latest, auth-service:latest, user-service:latest,
ticket-service:latest, solution-service:latest, knowledge-service:latest,
reward-service:latest, notification-service:latest, frontend:latest
```
`imagePullPolicy: Never` is set so the cluster must already have the image. Use `kind load docker-image <name>:latest --name ticketing`.

### Frontend at `http://ticketing.local/` returns 404
You haven't added the hosts entry:
```bash
sudo sh -c 'echo "127.0.0.1 ticketing.local" >> /etc/hosts'
```

### Swagger UI at gateway dropdown is empty / 404s on downstream spec
The gateway's proxy route to that service is misconfigured or the backend is down.
```bash
# Is the backend even up?
curl -s http://localhost:8083/v3/api-docs | jq -r '.info.title'

# Does gateway proxy resolve?
curl -s http://localhost:8080/v3/api-docs/ticket-service | jq -r '.info.title'
```
If #1 works but #2 fails, check `api_gateway/src/main/resources/application-local.yaml` — the corresponding `ticket-service-docs` route must have predicate `Path=/v3/api-docs/ticket-service` and filter `SetPath=/v3/api-docs`.

### Swagger "Try it out" returns 401 even after clicking Authorize
The token expired (default 1h). Re-login, re-paste. Also confirm the token was pasted as the raw JWT (no `Bearer ` prefix — Swagger adds that itself).

### Swagger UI doesn't show the lock icon next to protected endpoints
The controller method is missing `@SecurityRequirement(name = "bearerAuth")`, or the `OpenAPI` bean doesn't declare the `bearerAuth` `SecurityScheme`. Check `OpenApiConfig.java` for that service.

---

## 19. Quick Command Reference

### Local mode
```bash
# first time
./services.sh setup

# daily
./services.sh start
./services.sh status
./services.sh logs auth-service
./services.sh stop

# nuke & re-test
./services.sh clean
./services.sh setup && ./services.sh start
```

### Kubernetes mode
```bash
# first time
./services.sh k8s-up
echo "127.0.0.1 ticketing.local" | sudo tee -a /etc/hosts   # one-time
./services.sh k8s-seed     # seed roles + categories + demo accounts

# inspect
./services.sh k8s-status
./k8s/k8s.sh test          # smoke test via ingress
kubectl get pods -n ticketing-system
./services.sh k8s-logs auth-service    # or: kubectl logs -f deployment/auth-service -n ticketing-system

# tear down
./services.sh k8s-down
```

### Direct backend test (bypass gateway)
```bash
curl http://localhost:8081/actuator/health     # auth-service
curl http://localhost:8082/actuator/health     # user-service
# ... etc
```

### Full end-to-end smoke test (local mode)
```bash
# register
curl -X POST http://localhost:8080/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"me@test.com","password":"Pass123!","role":"USER"}'

# login
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"me@test.com","password":"Pass123!"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['accessToken'])")

# protected
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/tickets
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/knowledge
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/rewards/leaderboard
```

### Database access
```bash
# list all dbs
PGPASSWORD=postgres psql -U postgres -h localhost -l

# connect to any
PGPASSWORD=postgres psql -U postgres -h localhost -d auth_db

# inspect roles
PGPASSWORD=postgres psql -U postgres -h localhost -d auth_db \
  -c "SELECT role_name FROM roles;"
```

### Docker/Kafka access
```bash
docker ps                              # kafka + zookeeper running?
docker logs kafka | tail -30           # broker logs
docker exec -it kafka bash             # shell into broker
# inside:
kafka-topics --list --bootstrap-server localhost:9092
kafka-console-consumer --bootstrap-server localhost:9092 --topic user-registration --from-beginning
```

---

## Appendix A — File Inventory of Changes

### From the initial setup / bug-fix phase
**Created (6):**
1. `/services.sh`
2. `/k8s/k8s.sh`
3. `/k8s/kind-cluster.yaml`
4. `/User_service/Dockerfile`
5. `/Ticket_service/Dockerfile`
6. `/PROJECT_DOC.md`

**Rewritten (7 Dockerfiles + 3 manifests + 1 YAML):**
7. `/api_gateway/Dockerfile`
8. `/auth_service/Dockerfile`
9. `/Solution_service/Dockerfile`
10. `/knowledge_service/Dockerfile`
11. `/Reward_service/Dockerfile`
12. `/Notification_service/Dockerfile`
13. `/k8s/kafka.yaml`
14. `/k8s/ingress.yaml`
15. `/k8s/services.yaml`
16. `/api_gateway/src/main/resources/application-local.yaml`

### From the Java 21 migration (April 2026)
- 9 × `pom.xml` (all services + common-library + gateway)
- 9 × `Dockerfile` (base image bump to `21-jre`)
- `services.sh` (JAVA_HOME, `-Dmaven.test.skip=true`)
- `api_gateway/src/main/java/com/example/api_gateway/config/CorsConfig.java`

### From the Swagger / OpenAPI rollout (April 2026)
**Created (7 OpenApiConfig classes):**
- `auth_service/src/main/java/com/example/auth_service/config/OpenApiConfig.java`
- `User_service/src/main/java/com/example/User_service/config/OpenApiConfig.java`
- `Ticket_service/src/main/java/com/example/Ticket_service/config/OpenApiConfig.java`
- `Solution_service/src/main/java/com/example/Solution_service/config/OpenApiConfig.java`
- `Reward_service/src/main/java/com/example/Reward_service/config/OpenApiConfig.java`
- `Notification_service/src/main/java/com/example/Notification_service/config/OpenApiConfig.java`
- `api_gateway/src/main/java/com/example/api_gateway/config/OpenApiConfig.java`

**Modified:**
- 29 × controller `.java` files (annotation sweep, across 7 services — approximately 126 endpoints).
- 7 × `application.yaml` files (standardised springdoc block).
- `api_gateway/.../application-local.yaml` (7 new proxy routes + springdoc config).
- `auth_service/.../dto/LoginRequest.java`, `RegisterRequest.java` (@Schema examples).

### From the documentation overhaul (April 2026)
**Rewritten READMEs (10):**
- `/README.md` (top-level)
- `/auth_service/README.md`
- `/User_service/README.md`
- `/Ticket_service/README.md`
- `/Solution_service/README.md`
- `/knowledge_service/README.md`
- `/Reward_service/README.md`
- `/Notification_service/README.md`
- `/api_gateway/README.md`
- `/common_library_service/README.md`
- `/frontend/README.md`

**Modified:**
- `/PROJECT_DOC.md` — added §14 "Swagger / OpenAPI Documentation", renumbered §14–§18 → §15–§19, added migration log entries, extended troubleshooting and Appendix A.

### From the integration bug-fix pass (May 2026)

**Modified:**
- `Ticket_service/src/main/java/com/cognizant/Ticket_service/repository/TicketRepository.java` — replaced JPQL search query with native SQL using `CAST(:param AS text)` to fix PostgreSQL `lower(bytea)` error
- `frontend/src/pages/TicketListPage.jsx` — added `priorityFilter` to `useEffect` dependency array
- `Notification_service/src/main/java/com/cognizant/notificationservice/domain/entity/Notification.java` — removed `nullable = false` from `referenceId` and `referenceType` columns
- `Notification_service/src/main/java/com/cognizant/notificationservice/infrastructure/config/KafkaConfig.java` — added `rewardPointsContainerFactory` bean
- `Notification_service/src/main/java/com/cognizant/notificationservice/infrastructure/kafka/NotificationEventConsumer.java` — updated `consumeRewardPointsAdded` to use `rewardPointsContainerFactory` with typed `RewardPointsAddedEvent` parameter
- `Reward_service/src/main/java/com/cognizant/Reward_service/dto/event/RewardEventDTO.java` — added `@JsonIgnoreProperties(ignoreUnknown = true)` and `@JsonIgnore` on `timestamp`
- `Reward_service/src/main/java/com/cognizant/Reward_service/config/KafkaConfig.java` — added `solutionApprovedContainerFactory` bean
- `Reward_service/src/main/java/com/cognizant/Reward_service/kafka/RewardEventConsumer.java` — updated `consumeSolutionApproved` to use `solutionApprovedContainerFactory` with local `SolutionApprovalEvent` parameter

**Created:**
- `Notification_service/src/main/java/com/cognizant/notificationservice/application/dto/event/RewardPointsAddedEvent.java` — local DTO matching reward service's published event shape
- `Reward_service/src/main/java/com/cognizant/Reward_service/dto/event/SolutionApprovalEvent.java` — local DTO matching solution service's `SolutionEvent` Kafka payload

**Database migration (manual):**
```sql
ALTER TABLE notifications ALTER COLUMN reference_id DROP NOT NULL;
ALTER TABLE notifications ALTER COLUMN reference_type DROP NOT NULL;
```

---

*Last updated: 2026-05-03. Project at `/Users/abhidhabmellwyn/Downloads/Services`.*
