# Ticketing System — Microservices Platform

An 8-service Spring Boot microservices stack + React frontend + Spring Cloud Gateway, tied together with Kafka for event-driven flows and a Postgres-per-service data pattern. Supports ticket lifecycle, collaborative solutions, auto-generated knowledge articles, gamified rewards/leaderboards, and multi-channel notifications.

> **Status:** ✅ Java 21 • Spring Boot 3.2.x / 3.3.5 • springdoc-openapi 2.6.0 • aggregated Swagger UI at `http://localhost:8080/swagger-ui.html` • single `./services.sh` start command.

---

## Documentation

Full project docs live in [`project_docs/`](project_docs/):

| Document | Description |
|---|---|
| [README.md](project_docs/README.md) | This file (copy) |
| [PROJECT_DOC.md](project_docs/PROJECT_DOC.md) | Deep-dive architecture, setup, troubleshooting |
| [DEMO_FLOW.md](project_docs/DEMO_FLOW.md) | Step-by-step demo script for evaluation panel |
| [TEST_CASES.md](project_docs/TEST_CASES.md) | 45 manual test cases across all modules |
| [MANUAL_TESTING.md](project_docs/MANUAL_TESTING.md) | Detailed 162-assertion manual testing guide (Phases 0–10) |

---

## Table of Contents
- [At a glance](#at-a-glance)
- [Architecture](#architecture)
- [Services](#services)
- [Swagger / OpenAPI](#swagger--openapi)
- [Kafka topics](#kafka-topics)
- [Ports, databases, secrets](#ports-databases-secrets)
- [Quick start (local)](#quick-start-local)
- [Quick start (Kubernetes)](#quick-start-kubernetes)
- [Demo walkthrough](#demo-walkthrough)
- [Common operations](#common-operations)
- [Troubleshooting](#troubleshooting)

---

## At a glance

| | |
|---|---|
| **Backend services** | 8 (auth, user, ticket, solution, knowledge, reward, notification + gateway) |
| **Backend language** | Java 21 (Temurin) |
| **Backend framework** | Spring Boot 3.2.0 / 3.2.4 / 3.3.5 (per service — see individual READMEs) |
| **Spring Cloud** | 2023.0.3 (gateway only) |
| **Frontend** | React 18 + Redux Toolkit + Vite 5 + Tailwind 3 |
| **Persistence** | PostgreSQL 16, one DB per service |
| **Messaging** | Apache Kafka 3.6 (Bitnami image) |
| **API docs** | springdoc-openapi 2.6.0 per service + gateway aggregator |
| **Auth** | JWT HS384, shared secret across all backends |
| **Build tool** | Maven 3.9+ |
| **Orchestration** | Docker + Docker Compose / Kubernetes (manifests in `k8s/`) |
| **Runner script** | `./services.sh {start|stop|logs|restart|status} [service]` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             FRONTEND (React)                                 │
│                                 :3000                                        │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │ /api/*
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      API GATEWAY (Spring Cloud Gateway)                      │
│                                 :8080                                        │
│  • AuthFilter → calls auth-service /api/auth/validate → injects X-User-Id   │
│  • CORS (localhost:3000/5173 + ticketing.local)                             │
│  • Aggregated Swagger UI at /swagger-ui.html                                │
│  • Proxies /v3/api-docs/<service> to each downstream /v3/api-docs           │
└──────┬──────┬──────┬──────┬──────┬──────┬──────┬─────────────────────────────┘
       │      │      │      │      │      │      │
   ┌───▼──┬───▼──┬───▼──┬───▼──┬───▼──┬───▼──┬───▼──┐
   │ auth │ user │ticket│solut.│knowl.│reward│notif.│
   │ :8081│ :8082│ :8083│ :8084│ :8085│ :8086│ :8087│
   └──┬───┴──┬───┴──┬───┴──┬───┴──┬───┴──┬───┴──┬───┘
      │      │      │      │      │      │      │
   ┌──▼──────▼──────▼──────▼──────▼──────▼──────▼──┐
   │   7 × PostgreSQL 16 (one DB per service)       │
   └────────────────────────────────────────────────┘

                      ┌───────────────────────┐
     ticket.*         │                       │   consumed by
     solution.*   ───▶│    Apache Kafka       │──▶ reward-service,
     reward.*         │       :9092           │    notification-service,
     user.*           │                       │    knowledge-service
                      └───────────────────────┘
```

### Event flows
```
ticket.created     → notification  (in-app + email to assignee)
ticket.assigned    → notification
ticket.resolved    → reward (+50 pts) + notification
solution.created   → notification
solution.approved  → reward (+30 pts, split across contributors) + notification
solution.voted     → reward (+5 pts on upvote)
knowledge.created  → reward (+20 pts to author)
knowledge.rated    → reward (+5 pts)
reward.badge.awarded → notification
user.password-reset → notification (email)
leaderboard.updated → (internal fan-out)
```

---

## Services

Every service has its own detailed README in its folder. Summary here:

| # | Service | Port | DB | Kafka (in) | Kafka (out) | Boot | Swagger |
|---|---|---|---|---|---|---|---|
| 1 | [**auth-service**](auth_service/README.md) | 8081 | auth_db | — | `user.registered`, `user.password-reset` | 3.3.5 | [`:8081/swagger-ui.html`](http://localhost:8081/swagger-ui.html) |
| 2 | [**user-service**](User_service/README.md) | 8082 | user_db | `user.registered` | — | 3.3.5 | [`:8082/swagger-ui.html`](http://localhost:8082/swagger-ui.html) |
| 3 | [**ticket-service**](Ticket_service/README.md) | 8083 | ticket_db | — | `ticket.created/assigned/resolved/closed` | 3.3.5 | [`:8083/swagger-ui.html`](http://localhost:8083/swagger-ui.html) |
| 4 | [**solution-service**](Solution_service/README.md) | 8084 | solution_db | — | `solution.created/approved/voted` | 3.2.0 | [`:8084/swagger-ui.html`](http://localhost:8084/swagger-ui.html) |
| 5 | [**knowledge-service**](knowledge_service/README.md) | 8085 | knowledge_db | `solution.approved` | `knowledge.created/rated` | 3.2.0 | [`:8085/swagger-ui.html`](http://localhost:8085/swagger-ui.html) |
| 6 | [**reward-service**](Reward_service/README.md) | 8086 | reward_db | `ticket.resolved`, `solution.approved`, `solution.voted`, `knowledge.created`, `knowledge.rated` | `reward.points.added`, `reward.badge.awarded`, `leaderboard.updated` | 3.2.4 | [`:8086/swagger-ui.html`](http://localhost:8086/swagger-ui.html) |
| 7 | [**notification-service**](Notification_service/README.md) | 8087 | notification_db | 6 inbound topics | `notification.sent` | 3.2.4 | [`:8087/swagger-ui.html`](http://localhost:8087/swagger-ui.html) |
| 8 | [**api-gateway**](api_gateway/README.md) | 8080 | — | — | — | 3.3.5 | [`:8080/swagger-ui.html`](http://localhost:8080/swagger-ui.html) (aggregated) |
| 9 | [**frontend**](frontend/README.md) | 3000 | — | — | — | — | — |
| – | [common-library](common_library_service/README.md) | n/a | n/a | n/a | n/a | 3.3.5 BOM | — |

> Role shorthand in the per-service docs: `JWT` = any authenticated user, `JWT + ADMIN` = requires `ROLE_ADMIN`, similarly `MANAGER` / `ENGINEER`. Roles come from the JWT `role` claim, propagated as the `X-User-Role` header by the gateway.
>
> **JWT claims**: the token payload includes `sub` (email), `role`, and `authUserId` (the UUID primary key in `auth_db.users`). Downstream services and scripts use `authUserId` — not `sub` — when calling user-scoped endpoints like `GET /api/rewards/users/{userId}/points` and `GET /api/notifications/users/{userId}`. The `POST /api/auth/validate` response exposes this as `authUserId` in its JSON body.

---

## Swagger / OpenAPI

Every backend service exposes its OpenAPI 3.0 spec and UI:

- **Direct**: `http://localhost:<port>/swagger-ui.html` and `/v3/api-docs`
- **Aggregated via gateway**: `http://localhost:8080/swagger-ui.html` with a top-right dropdown listing all 7 services

Deep-link to a specific service:
```
http://localhost:8080/swagger-ui.html?urls.primaryName=ticket-service
```

### How the aggregator works
1. Gateway declares 7 URLs in `SwaggerUiConfigProperties` (see `api_gateway/.../OpenApiConfig.java`), each named `<service>-service` → `/v3/api-docs/<service>-service`.
2. Gateway's reactive route table proxies `/v3/api-docs/<service>` → downstream `:<port>/v3/api-docs` via `SetPath` filter.
3. Swagger UI loads, renders the dropdown, lazy-loads each spec on selection.

### Try-it-out with JWT
1. `POST /api/auth/login` → copy the `accessToken`.
2. Click the **Authorize** button (lock icon) in any Swagger UI.
3. Paste the token — Swagger stores it and sends `Authorization: Bearer …` on every "Try it out" call.

---

## Kafka topics

### Published (across services)
| Topic | Producer | Payload |
|---|---|---|
| `ticket.created` | ticket-service | `{ticketId, creatorId, assigneeId?, difficulty}` |
| `ticket.assigned` | ticket-service | `{ticketId, engineerId}` |
| `ticket.resolved` | ticket-service | `{ticketId, resolverId, contributorIds[]}` |
| `ticket.closed` | ticket-service | `{ticketId}` |
| `solution.created` | solution-service | `{solutionId, ticketId, authorId}` |
| `solution.approved` | solution-service | `{solutionId, ticketId, authorId, contributorIds[]}` |
| `solution.voted` | solution-service | `{solutionId, voterId, up/down}` |
| `knowledge.created` | knowledge-service | `{articleId, authorId}` |
| `knowledge.rated` | knowledge-service | `{articleId, raterId, stars}` |
| `reward.points.added` | reward-service | `{userId, points, reason, sourceEventId}` |
| `reward.badge.awarded` | reward-service | `{userId, badgeKey}` |
| `leaderboard.updated` | reward-service | `{top: [...]}` |
| `notification.sent` | notification-service | audit record |
| `user.password-reset` | auth-service | `{email, token}` |

### Consumption table
- **reward-service** ← `ticket.resolved`, `solution.approved`, `solution.voted`, `knowledge.created`, `knowledge.rated`
- **notification-service** ← `ticket.created`, `ticket.assigned`, `ticket.resolved`, `solution.approved`, `reward.badge.awarded`, `user.password-reset`

See individual service READMEs for point values and idempotency rules.

---

## Ports, databases, secrets

| Port | Service | Container DB |
|---|---|---|
| 3000 | frontend (dev or nginx) | — |
| 8080 | api-gateway | — |
| 8081 | auth-service | postgres-auth / `auth_db` |
| 8082 | user-service | postgres-user / `user_db` |
| 8083 | ticket-service | postgres-ticket / `ticket_db` |
| 8084 | solution-service | postgres-solution / `solution_db` |
| 8085 | knowledge-service | postgres-knowledge / `knowledge_db` |
| 8086 | reward-service | postgres-reward / `reward_db` |
| 8087 | notification-service | postgres-notification / `notification_db` |
| 9092 | kafka | — |
| 5432 | postgres (each in its own container) | — |

### Shared secrets (env vars)
| Var | Default | Used by |
|---|---|---|
| `JWT_SECRET` | `myVerySecretKeyForJWTThatIsLongEnoughForHS512Signature` | Every backend + gateway |
| `SPRING_KAFKA_BOOTSTRAP_SERVERS` | `kafka:9092` (in docker) / `localhost:9092` (local) | All Kafka consumers/producers |
| `SPRING_DATASOURCE_*` | per-service | Each backend |
| `APP_EMAIL_ENABLED` | `false` | notification-service only |

> **Change the JWT secret in one place** and the whole stack must be restarted — mismatched keys = `401` on every `/api/**` call.

---

## Quick start (local)

### Prerequisites
- JDK 21 (Temurin)
- Maven 3.9+
- Docker Desktop (for Kafka + 7 × Postgres)
- Node 18+ (frontend)

### One-shot via `services.sh`
```bash
./services.sh setup             # first-time: create DBs, build all JARs, npm install
./services.sh start             # start Postgres, Kafka, 8 services + frontend
./services.sh status            # health table
./services.sh seed              # create demo users (admin1, user1–user10, password: Passw0rd!)
./services.sh restart           # stop everything then start everything
./services.sh restart <svc>     # hot-restart a single service (e.g. restart ticket-service)
./services.sh logs ticket-service
./services.sh stop
```

### Manual steps (if you prefer)
```bash
# 0. Ensure Java 21 active
export JAVA_HOME=/opt/homebrew/Cellar/openjdk@21/21.0.11

# 1. Install the common library into your local ~/.m2
cd common_library_service && ./mvnw clean install && cd ..

# 2. Bring up Postgres + Kafka (docker compose in each service dir, or one root compose)
# …see services.sh for the canonical wiring

# 3. Build + run each service (separate terminals or ./services.sh)
cd auth_service && mvn -DskipTests spring-boot:run
# repeat for User_service, Ticket_service, Solution_service, knowledge_service,
# Reward_service, Notification_service, api_gateway

# 4. Frontend
cd frontend && npm install && npm run dev
```

Once all up:
- Frontend → http://localhost:3000
- Gateway + aggregated Swagger → http://localhost:8080/swagger-ui.html
- Any individual service's Swagger → `http://localhost:80XX/swagger-ui.html`

### Seed demo users (first run or after schema wipe)
```bash
./services.sh seed
# or directly: bash scripts/demo_seed.sh
```
Creates 12 demo accounts (admin1, admin2, user1–user10) with password `Passw0rd!`.
Writes auth tokens to `scripts/.seed/*.token` and a user manifest to `scripts/.seed/users.json`.

> **After a full schema wipe** the `categories` table in `ticket_db` is empty — the ticket service has
> no data initializer. Re-insert the 5 seed categories:
> ```sql
> INSERT INTO categories (category_id, category_name, description, created_at)
>   VALUES (1,'Network','Network issues',now()),(2,'Software','Software issues',now()),
>          (3,'Hardware','Hardware issues',now()),(4,'Security','Security issues',now()),
>          (5,'Other','Other issues',now());
> SELECT setval(pg_get_serial_sequence('categories','category_id'), 5);
> ```

### Run the end-to-end test suite
```bash
bash scripts/e2e_test.sh
```
Requires `demo_seed.sh` to have been run first (tokens in `scripts/.seed/`). Exercises **24 steps**
covering ticketing, solutions, votes, ticket workflow, rewards, leaderboard, knowledge articles,
notifications, the full Kafka `solution.approved → KB article` chain, and multi-user validation.

---

## Quick start (Kubernetes)

```bash
# Builds all images, applies manifests, waits for pods to be Ready
./k8s/k8s.sh up

# Status
kubectl get pods -n ticketing-system

# Access via ingress
echo "127.0.0.1 ticketing.local" | sudo tee -a /etc/hosts
open http://ticketing.local
```

Manifests live in `k8s/` — one YAML per service + `postgres.yaml`, `kafka.yaml`, `ingress.yaml`, `namespace.yaml`.

Tear down:
```bash
./k8s/k8s.sh down
```

---

## Demo walkthrough

1. **Register** at http://localhost:3000/register — create one `ADMIN`/`MANAGER` and one `USER`.
2. **Login** as the user → Dashboard.
3. **Create ticket** → `POST /api/tickets` → Kafka `ticket.created` → notification-service drops an in-app row for the assignee.
4. **Login as engineer/admin**, open the ticket, click **Resolve** → Kafka `ticket.resolved` → reward-service awards +50 pts → notification-service notifies the requester → frontend leaderboard refreshes.
5. **Submit a solution** on the ticket, have the manager **approve** → Kafka `solution.approved` → reward-service awards +30 pts (split across contributors) → notification fans out.
6. **Create a knowledge article** at `/knowledge` → +20 pts. Rate it → +5 pts.
7. Watch the **Leaderboard** and **Notifications** pages update.

---

## Common operations

### Build everything (no start)
```bash
./services.sh build
```

### Restart one service (after code change)
```bash
./services.sh restart ticket-service   # stops + restarts that service only
./services.sh restart frontend         # restart the Vite dev server
```

### Run tests for a single service
```bash
cd Ticket_service && mvn test
```

### Inspect Kafka consumer lag
```bash
docker exec kafka kafka-consumer-groups --bootstrap-server kafka:9092 --list
docker exec kafka kafka-consumer-groups --bootstrap-server kafka:9092 \
  --describe --group reward-service-group
```

### Health probes
```bash
for port in 8080 8081 8082 8083 8084 8085 8086 8087; do
  echo -n "$port "
  curl -s http://localhost:$port/actuator/health | jq -r '.status'
done
```

### Grab a JWT and hit an endpoint
```bash
# Login returns { success, data: { accessToken, refreshToken, ... } }
TOKEN=$(curl -s -X POST http://localhost:8081/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"user1@demo.local","password":"Passw0rd!"}' | jq -r '.data.accessToken')

# Hit the gateway (port 8080) — auth filter validates token and forwards to downstream
curl http://localhost:8080/api/tickets -H "Authorization: Bearer $TOKEN" | jq
```

---

## Troubleshooting

**Gateway returns 401 for every call**
- `jwt.secret` drifted between gateway and a backend. Must match exactly.
- auth-service unreachable from gateway. Check `AUTH_SERVICE_URI` and `docker ps`.
- Gateway calls `GET /api/auth/validate` (not POST) — verify `permitAll()` covers this path in auth-service `SecurityConfig`.

**Service won't start: "address already in use" on 808X**
`lsof -ti:808X | xargs kill -9` — or `./services.sh stop` and restart.

**Maven build fails with `release version 21 not supported`**
`JAVA_HOME` is pointing at 17. Fix with:
```bash
export JAVA_HOME=/opt/homebrew/Cellar/openjdk@21/21.0.11
java -version   # verify
```

**Maven build fails with `TypeTag::UNKNOWN` (Lombok error)**
System Maven is using Java 25 (Homebrew default `openjdk`). Lombok 1.18.34 does not support Java 25.
Force Java 21 for every build in this project:
```bash
export JAVA_HOME=/Users/<you>/Library/Java/JavaVirtualMachines/ms-21.0.10/Contents/Home
# or Homebrew:
export JAVA_HOME=/opt/homebrew/Cellar/openjdk@21/21.0.11
mvn -DskipTests spring-boot:run
```

**Kafka `UnknownTopicOrPartitionException`**
Topics are auto-created on first publish. Either restart the producer (ticket-service / solution-service / etc.) or pre-create them:
```bash
docker exec kafka kafka-topics --bootstrap-server kafka:9092 \
  --create --topic ticket.resolved --partitions 1 --replication-factor 1
```

**Swagger UI dropdown at gateway is empty**
- Backend not up. `curl http://localhost:8081/v3/api-docs | head` on each port.
- Gateway proxy route misconfigured. Check `application-local.yaml` `…-service-docs` routes exist with `SetPath=/v3/api-docs`.

**Notification-service fails to start: "JavaMailSender bean missing"**
Don't exclude `MailSenderAutoConfiguration`. Set `app.email.enabled=false` to disable dispatch — the bean still exists, sending no-ops.

**Ticket search returns 500 — `ERROR: function lower(bytea) does not exist`**
JPQL with `LOWER(:param)` or `LOWER(CONCAT('%', :param, '%'))` on a null parameter causes PostgreSQL to infer `bytea` as the bind type. Fix: use a native SQL query with `CAST(:param AS text)` for every nullable bind variable. See `Ticket_service/README.md` for the exact query.

**Kafka events deserialise to wrong type in consumers**
If a service sets `spring.json.value.default.type` globally (e.g. `RewardEventDTO`), that type is forced for ALL topics consumed by that service. Listeners that need a different type (e.g. `SolutionApprovalEvent`) will receive a `RewardEventDTO` instance; Spring then fails to convert it.
Fix: create a dedicated `ConcurrentKafkaListenerContainerFactory` bean with its own `DefaultKafkaConsumerFactory` and set `JsonDeserializer.VALUE_DEFAULT_TYPE` to the required class. Reference the factory via `containerFactory = "beanName"` on `@KafkaListener`.

**"Points Earned" notifications never arrive**
The reward service publishes without Kafka type headers (`ADD_TYPE_INFO_HEADERS: false` in its `ProducerFactory`). The notification service's default consumer factory falls back to `NotificationEvent` instead of `RewardPointsAddedEvent`. Fix: the notification service needs a dedicated `rewardPointsContainerFactory` with `USE_TYPE_INFO_HEADERS: false`. Already applied — see `Notification_service/README.md`.

**Frontend CORS error**
- In dev, use the Vite proxy (hit `/api/...`, not `http://localhost:8080/api/...` directly).
- In prod, ensure gateway's `CorsConfig` whitelists the origin.

**Everything 500s after a commit**
Common-library changed? Rebuild it: `cd common_library_service && ./mvnw clean install` — consumers pin the version but need the jar in `~/.m2` to resolve.

---

## Project docs
- [`PROJECT_DOC.md`](PROJECT_DOC.md) — detailed architecture, event schemas, fixes applied, migration history
- Per-service `README.md` — API surface + configuration + troubleshooting
- [`MANUAL_TESTING.md`](MANUAL_TESTING.md) — step-by-step 162-assertion manual testing guide

---

## Tech stack (full)
- **Languages**: Java 21, JavaScript (ES2022)
- **Backend**: Spring Boot 3.2.x / 3.3.5, Spring Cloud Gateway 2023.0.3, Spring Security 6, Spring Data JPA 3, Spring Kafka, Spring Mail, Spring WebSocket (STOMP)
- **Auth**: JJWT 0.11.5 (HS384 shared secret)
- **Database**: PostgreSQL 16 (one per service)
- **Messaging**: Apache Kafka 3.6 (Bitnami image)
- **API docs**: springdoc-openapi 2.6.0 (webmvc + webflux UI)
- **Library**: `com.kva:common-library:1.0.0`
- **Frontend**: React 18.2, Redux Toolkit 2.0, React Router 6.21, axios 1.6, jwt-decode 4.0, Tailwind 3.4, Vite 5.0
- **Build**: Maven 3.9+, npm
- **Containerization**: Docker, Docker Compose, Kubernetes (1.28+)
- **Observability**: Spring Boot Actuator, Micrometer

---

## License
Internal / MIT (see repo).
