# DEMO_FLOW.md — Final-Sem Walkthrough Script

**Audience:** evaluation panel  
**Duration:** ~25 minutes (5 min setup checkpoint, 15 min live demo, 5 min Q&A buffer)  
**Goal:** prove the platform — 8 Spring Boot microservices + React frontend + Postgres + Kafka — is genuinely integrated end-to-end, not a façade.

---

## 0. Pre-flight (do this 30 minutes BEFORE the panel arrives)

Run these from `/Users/abhidhabmellwyn/Downloads/Services` and **do not skip the verification step**.

```bash
# 1. infra + services
./services.sh start               # brings up Postgres, Kafka, all 8 services + frontend
                                  # takes ~90s on warm JARs

# 2. health gate
./services.sh status              # every row must say UP / 200 UP
                                  # if anything is DOWN: ./services.sh restart <service-name>

# 3. seed demo users
./services.sh seed                # creates admin1, admin2, user1..user10 (password: Passw0rd!)
                                  # tokens land in scripts/.seed/*.token

# 4. dry-run the e2e
./scripts/e2e_test.sh             # ALL CLEAR — 24/24 steps PASS
```

**If e2e_test.sh fails any step, stop and fix BEFORE the demo.** A red step in front of the panel is worse than postponing.

Open these tabs in advance so you don't fumble during the demo:
- http://localhost:3000 — React app (login screen)
- http://localhost:8080/swagger-ui.html — gateway aggregated docs (optional fallback)
- http://localhost:8083/swagger-ui.html — ticket-service Swagger
- http://localhost:8084/swagger-ui.html — solution-service Swagger
- http://localhost:8086/swagger-ui.html — reward-service Swagger

A terminal at `cd /Users/abhidhabmellwyn/Downloads/Services` with three split panes:
1. `tail -f logs/ticket-service.log`
2. `tail -f logs/solution-service.log`
3. `tail -f logs/reward-service.log`

---

## 1. Architecture intro (2 min)

> "The system is a 9-component microservices platform. One API gateway, seven domain services, and a React frontend. Postgres for state, Kafka for cross-service events. Every service ships its own Swagger and its own JWT verifier — auth-service mints tokens; the others trust them."

Show this diagram (sketch on whiteboard or hand printout):

```
                        ┌────────────────┐
                        │  React (3000)  │
                        └───────┬────────┘
                                │ HTTP+JWT
                  ┌─────────────▼──────────────┐
                  │   API Gateway (8080)        │  ← single ingress, injects X-User-Id
                  └──┬─────┬─────┬─────┬───────┘
            ┌────────┘     │     │     └────────┐
            ▼              ▼     ▼              ▼
   ┌──────────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐
   │ auth (8081)  │ │user (82) │ │tkt (83)  │ │ sol (8084)  │
   └──────────────┘ └──────────┘ └──────────┘ └──────┬──────┘
                                                      │ Kafka
   ┌──────────────┐ ┌──────────┐ ┌──────────┐        │ solution.approved
   │ kb (8085)    │ │rwd (86)  │ │noti (87) │ ◄───────┘
   └──────────────┘ └──────────┘ └──────────┘
            │              │             │
            └──────────────┴──────┬──────┘
                                  ▼
                       ┌──────────────────┐
                       │  Postgres (5432) │   1 db per service
                       └──────────────────┘
```

Talking points:
- **9 services × 1 DB each** — proper bounded contexts, no shared schema.
- **Gateway is the only public ingress** — every other service is internal.
- **JWT secret is shared** but issuance is centralised in auth-service. Others only verify.
- **Kafka is the async bus** — `solution.approved` fans out to knowledge-service (article auto-creation), reward-service (points), and notification-service (push).

---

## 2. Live demo — Customer journey (8 min)

We follow **one ticket from creation to resolution** through the UI, then narrate the data trail.

### 2.1 — user1 logs in and files a ticket

1. Open http://localhost:3000 (incognito tab #1).
2. Sign in as `user1@demo.local` / `Passw0rd!`.
3. Observe: Dashboard loads with open ticket count and leaderboard.
4. Click **Tickets** → **Create Ticket**.
5. Fill in:
   - **Title:** `Cannot connect to corporate VPN from coffee shop`
   - **Description:** `MFA prompt fails with "auth error". Tried: restarting client, fresh wifi.`
   - **Priority:** HIGH
   - **Difficulty:** MEDIUM
   - **Category:** Network (id=1)
   - **Visibility:** Organization
6. Click **Create Ticket**.

> *Narration:* "The frontend POSTed `/api/tickets` through the gateway. The gateway extracted the JWT, injected `X-User-Id` (the user's authUserId UUID), and routed to ticket-service on 8083. Ticket-service stored the row."

**Show the proof:** flip to terminal pane 1 (`logs/ticket-service.log`):
```
INFO  TicketServiceImpl - Creating ticket for user: <UUID>
```

### 2.2 — admin1 triages and changes status

1. Open incognito tab #2 → sign in as `admin1@demo.local` / `Passw0rd!`.
2. Click **Tickets**. Open user1's ticket.
3. Observe: **Change status** buttons visible (→ In Progress, → Resolved) — admin-only UI.
4. Click **→ In Progress**.

> *Narration:* "Admin1 has ROLE_ADMIN in their JWT. The frontend checks `role.includes('ADMIN')` to show the status buttons. Ticket-service's `@PreAuthorize` enforces the same check on the API side."

### 2.3 — admin1 submits a solution

1. From admin1's tab, on the same ticket, click **Submit Solution**.
2. Fill in:
   - **Solution Title:** `Reset MFA token + reauth`
   - **Solution Description:** `1. Sign out everywhere. 2. Re-pair the authenticator app. 3. Sign in from VPN client.`
3. Click **Submit Solution**.

> *Narration:* "Solution-service on 8084 accepted the POST. The `SolutionRequestDTO` requires a non-blank `title` AND `solutionContent` — we validate both on the frontend form and via `@NotBlank` on the backend."

### 2.4 — admin1 approves the solution → Kafka fans out

1. Click **Approve** next to the PENDING solution.

> *Narration:* "Approval triggers a Kafka publish on topic `solution.approved`. Three downstream services subscribe:
> - **knowledge-service** auto-creates a KB article from this solution
> - **notification-service** emits an in-app notification to the contributor
> - **reward-service** tallies points for the solution author"

**Show the proof:** flip to logs pane 2 (`solution-service.log`):
```
INFO  SolutionEventProducer - Published solution.approved event for solution: <UUID>
```

**Wait ~5–10 seconds**, then navigate to **Knowledge Base** → the auto-created article titled `Solution for Ticket #...` appears.

### 2.5 — user1 views the auto-created KB article and rates it

1. Switch back to user1's tab. Click **Knowledge Base**.
2. Open the auto-created article (`Solution for Ticket #...`).
3. Click 5 stars in the **Rate this Article** section → add feedback → click **Submit Rating**.

> *Narration:* "The article was created entirely by the Kafka consumer in knowledge-service — no human created it. The rating POSTs to `/api/ratings`, which uses an upsert: the same user rating again updates their rating rather than creating a duplicate."

### 2.6 — admin1 closes the ticket and awards points

1. admin1 tab → ticket → click **→ Resolved**.
2. Click **Award Points** → set Points: `25`, Reason: `Outstanding VPN solution` → click **Award**.

> *Narration:* "The reward endpoint is `POST /api/rewards/points` — admin-only. It creates a `RewardTransaction` row and the leaderboard scheduler will pick it up on the next MONTHLY generation cycle."

---

## 3. Cross-service integration — show the data trail (3 min)

This is the part that separates "real microservices" from "monolith with extra steps".

```bash
# Drop into psql and prove every service has its own DB and the data flowed
PGPASSWORD=postgres psql -U postgres -h localhost -d ticket_db \
  -c "SELECT ticket_id, title, status FROM tickets ORDER BY created_at DESC LIMIT 3;"

PGPASSWORD=postgres psql -U postgres -h localhost -d solution_db \
  -c "SELECT solution_id, title, status, approved_by FROM solutions ORDER BY created_at DESC LIMIT 3;"

PGPASSWORD=postgres psql -U postgres -h localhost -d knowledge_db \
  -c "SELECT article_id, title, status FROM knowledge_articles ORDER BY created_at DESC LIMIT 3;"

PGPASSWORD=postgres psql -U postgres -h localhost -d reward_db \
  -c "SELECT user_id, points, reason FROM reward_transactions ORDER BY created_at DESC LIMIT 5;"

PGPASSWORD=postgres psql -U postgres -h localhost -d notification_db \
  -c "SELECT notification_id, title, type FROM notifications ORDER BY created_at DESC LIMIT 5;"
```

> *Narration:* "Every row was written by a different service. The ticket lives in `ticket_db`. The approved solution in `solution_db`. The knowledge article was auto-created from the Kafka event by knowledge-service. The reward row was written by reward-service. None of them share a transaction — they're glued together by Kafka."

---

## 4. Rewards & leaderboard (2 min)

1. Open the **Leaderboard** page (visible to all users).
2. User1 should appear with points (from the auto-reward when solution was approved + manual award from step 2.6).

> *Narration:* "Reward-service exposes both a write endpoint (admin-gated, used by managers to recognise contributions) and read endpoints (open to all authenticated users). The leaderboard shows per-period rankings."

To generate a fresh MONTHLY leaderboard snapshot (if the list is empty):
```bash
# via Swagger at http://localhost:8086/swagger-ui.html → POST /api/rewards/leaderboard/generate
# or directly:
TOKEN=$(cat scripts/.seed/admin1.token)
curl -s -X POST http://localhost:8080/api/rewards/leaderboard/generate \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"period":"MONTHLY"}' | jq
```

---

## 5. Swagger & API discoverability (2 min)

Open http://localhost:8084/swagger-ui.html (solution-service).

> *Narration:* "Every service ships springdoc-openapi. Pick the `Solutions` tag, expand `POST /api/solutions`. Click **Authorize**, paste an admin token from `scripts/.seed/admin1.token`, and execute. The same call you saw in the UI is now reproducible from Swagger — that's the integration test surface."

---

## 6. Run the automated end-to-end (3 min)

> *Narration:* "We've shown the journey by hand. Here's the same journey executed by a script, hitting the live gateway, with PASS/FAIL counters."

```bash
./scripts/e2e_test.sh
```

Read the section headers as it runs:
- Tickets → 4 PASS
- Solutions → 4 PASS
- Ticket workflow → 2 PASS
- Rewards → 5 PASS
- Knowledge → 2 PASS
- Notifications → 1 PASS
- Kafka event flow → 1 PASS
- Multi-user validation → 5 PASS

Final line: **ALL CLEAR — 24/24 steps PASS**.

> "Each step is a real HTTP call against the gateway with a real JWT. The script reuses the seeded users from `./services.sh seed`, so every panel run starts from the same baseline."

---

## 7. Wrap-up (1 min)

Three sentences:

1. "9 services, 7 databases, Kafka-driven async events, JWT-secured everywhere — and a unified React frontend. It's a real distributed system."
2. "Each service is independently deployable, has its own Swagger contract, and was validated by the 24-step e2e test you just saw."
3. "Source is at `/Users/abhidhabmellwyn/Downloads/Services`, full docs in `project_docs/`, and the demo is reproducible with `./services.sh seed && ./scripts/e2e_test.sh`."

---

## Backup plan — if something dies on stage

| Symptom | Quick recovery |
|---------|---------------|
| Frontend won't load | `./services.sh restart frontend` (wait ~15s) |
| Login fails 401 | `./services.sh restart auth-service`, retry login |
| Ticket POST 500 | usually categories missing — `PGPASSWORD=postgres psql -U postgres -d ticket_db -c "SELECT * FROM categories;"` should show 5 rows |
| Solution approve 403 | JWT shape issue — show Swagger Authorize dialog, paste fresh token from `scripts/.seed/admin1.token` |
| Leaderboard empty | Call `POST /api/rewards/leaderboard/generate` from Swagger or the curl above |
| KB article missing after approve | Kafka event may have been slow — wait 20s then refresh Knowledge Base |
| Kafka unreachable | `docker ps` should show kafka container; if not: `./services.sh stop && ./services.sh start` |
| Any single service down | `./services.sh restart <service-name>` |

If multiple things break: **switch to running `./scripts/e2e_test.sh` and narrate the PASS lines as it goes.** That proves end-to-end integration even if the UI is misbehaving.

---

## Roles & users cheat sheet (memorise this)

| Nick | Email | Role | Demo purpose |
|------|-------|------|--------------|
| admin1 | admin1@demo.local | ADMIN | Approves solutions, awards points, manages tickets |
| admin2 | admin2@demo.local | ADMIN | Stand-in admin for second-approver scenario |
| user1 | user1@demo.local | USER | The ticket creator throughout the demo |
| user2–user10 | userN@demo.local | USER | Filler population for multi-user validation |

**Password for everyone:** `Passw0rd!`

---

## Things NOT to do on stage

- Don't `./services.sh stop` mid-demo — it takes 30s to come back up.
- Don't run `mvn clean` anywhere — you'll lose the cached JARs.
- Don't open a database admin UI live — too easy to fat-finger a DELETE.

---

## 8. Kubernetes Demo Mode (Alternative to Local)

Use this section if the demo is running on the Kubernetes (kind) cluster instead of local JARs.
Everything in sections 1–7 applies identically; only the **access URL and credentials change**.

### 8.0 — Pre-flight (30 min before panel)

```bash
# Check cluster is up
./services.sh k8s-status

# If not up yet:
./services.sh k8s-up          # ~5–8 min first time, ~2 min thereafter

# Seed demo data (required on a fresh cluster)
./services.sh k8s-seed

# One-time /etc/hosts entry (if not already done)
grep -q "ticketing.local" /etc/hosts || echo "Run: echo '127.0.0.1 ticketing.local' | sudo tee -a /etc/hosts"

# Smoke test
./k8s/k8s.sh test             # prints register / login / protected endpoint HTTP codes
```

Open these tabs:
- **http://ticketing.local/** — React frontend (via Nginx ingress)
- **http://ticketing.local/api/tickets** — raw API check (expect 401 unauthenticated)

---

### 8.1 — k8s credentials (memorise)

| Nick | Email | Role | Demo purpose |
|------|-------|------|--------------|
| k8sadmin | k8sadmin@demo.test | ADMIN | Approves solutions, awards points, manages tickets |
| k8sengineer | k8sengineer@demo.test | ENGINEER | Submits solutions, earns points |

**Password for both:** `Demo@1234`

---

### 8.2 — URL differences vs local

| | Local | Kubernetes |
|---|---|---|
| Frontend | http://localhost:3000 | http://ticketing.local |
| API entry | http://localhost:8080/api | http://ticketing.local/api |
| Aggregated Swagger UI | http://localhost:8080/swagger-ui.html | http://ticketing.local/swagger-ui.html |
| OpenAPI JSON | http://localhost:8080/v3/api-docs | http://ticketing.local/v3/api-docs |
| Per-service OpenAPI JSON | http://localhost:808X/v3/api-docs | http://ticketing.local/v3/api-docs/\<service\> |
| Individual service Swagger | http://localhost:808X/swagger-ui.html | Not exposed (pods are internal) |

> The ingress routes `/swagger-ui`, `/v3/api-docs`, and `/webjars` to the gateway before the frontend catch-all rule — so the aggregated Swagger UI works directly at `http://ticketing.local/swagger-ui.html`.

---

### 8.3 — Live demo steps (k8s version)

Follow the same steps as **Section 2** but:
- Replace all `http://localhost:3000` → `http://ticketing.local`
- Use `k8sadmin@demo.test` (instead of `admin1@demo.local`)
- Use `k8sengineer@demo.test` as the second user

Token retrieval for terminal demos:
```bash
TOKEN=$(curl -s -H "Host: ticketing.local" -H "Content-Type: application/json" \
  -X POST http://ticketing.local/api/auth/login \
  -d '{"email":"k8sadmin@demo.test","password":"Demo@1234"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['accessToken'])")
echo $TOKEN
```

---

### 8.4 — Show the k8s platform itself

```bash
# All 17 pods running (7 postgres + kafka + 8 services + frontend)
kubectl get pods -n ticketing-system

# Ingress routing rules
kubectl get ingress -n ticketing-system -o yaml | grep -A5 "rules:"

# Individual service pod details
kubectl describe pod -n ticketing-system -l app=auth-service | grep -E "Image:|Ready:|Restart"

# Live logs during demo (run in a split pane)
./services.sh k8s-logs solution-service
```

**Key talking points:**
- "17 pods running — 7 PostgreSQL StatefulSets (one per service), Apache Kafka in KRaft mode, 8 Spring Boot services and the React frontend."
- "The Nginx ingress is the single entry point. `/api/*` routes to the gateway pod; `/*` routes to the frontend pod."
- "All images are built locally and loaded into the kind cluster — `imagePullPolicy: Never`. This mirrors how a private registry would work in production."
- "Pod-to-pod communication uses Kubernetes DNS: `auth-service:8081`, `kafka:9092` — no hardcoded IPs anywhere."

---

### 8.5 — Backup plan (k8s-specific)

| Symptom | Quick recovery |
|---------|---------------|
| Pod `CrashLoopBackOff` | `./services.sh k8s-logs <svc>` to see error; restart pod: `kubectl rollout restart deployment/<svc> -n ticketing-system` |
| Frontend 404 / ingress not routing | `kubectl get ingress -n ticketing-system` — check host is `ticketing.local`; check `/etc/hosts` |
| Login 401 after seed | Roles not seeded — run `./services.sh k8s-seed` again |
| Ticket create fails | Categories not seeded — run `./services.sh k8s-seed` again |
| KB article missing after approve | Wait 15–20s for Kafka fan-out; check `./services.sh k8s-logs knowledge-service` |
| Cluster completely broken | `./services.sh k8s-down && ./services.sh k8s-up && ./services.sh k8s-seed` (~5 min) |
