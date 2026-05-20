# Manual Testing Guide
**Project:** Services — Spring Boot Microservices Platform  
**Total Assertions:** 171 | **Intentionally Skipped:** 1 (UC-141 — no retry endpoint)  
**Base URL (Gateway):** `http://localhost:8080`  
**Tool:** Postman or curl

---

## Pre-Test Checklist

Before starting, verify all services are running:

| Port | Service |
|------|---------|
| 8080 | API Gateway |
| 8081 | Auth Service |
| 8082 | User Service |
| 8083 | Ticket Service |
| 8084 | Solution Service |
| 8085 | Knowledge Service |
| 8086 | Reward Service |
| 8087 | Notification Service |

```
curl http://localhost:8080/actuator/health
```
Expected: `{"status":"UP"}`

---

## Variable Reference

Store these as Postman collection variables as you progress through the tests.

| Variable | Set In | Used In |
|----------|--------|---------|
| `TOKEN_USER1` | Phase 0 login | All user1 requests |
| `TOKEN_USER2` | Phase 0 login | All user2 requests |
| `TOKEN_USER3` | Phase 0 login | All user3 requests |
| `TOKEN_USER4` | Phase 0 login | All user4 requests |
| `TOKEN_MGR1` | Phase 0 login | Manager approvals |
| `TOKEN_MGR2` | Phase 0 login | Manager rejections |
| `TOKEN_MGR3` | Phase 0 login | Multi-contributor approval |
| `TOKEN_MGR4` | Phase 0 login | Re-approval tests |
| `TOKEN_ADMIN1` | Phase 0 login | Admin user management |
| `TOKEN_ADMIN2` | Phase 0 login | Admin ticket assign |
| `TOKEN_ADMIN3` | Phase 0 login | KB management |
| `TOKEN_ADMIN4` | Phase 0 login | Notifications broadcast |
| `AUTH_ID_USER1` | Phase 0 register | Ticket assign, rewards, notifications |
| `AUTH_ID_USER2` | Phase 0 register | Contributor, voter |
| `AUTH_ID_USER3` | Phase 0 register | Multi-contributor |
| `AUTH_ID_USER4` | Phase 0 register | Security tests |
| `TICKET1_ID` | UC-24 | Solutions, comments, contributors |
| `TICKET2_ID` | UC-24 | Solution 2 (rejection flow) |
| `TICKET3_ID` | UC-24 | Assignment, solution 3 |
| `TICKET4_ID` | UC-24 | Status change, rating |
| `SOLUTION1_ID` | UC-48 | Approve, vote, comment, KB |
| `SOLUTION2_ID` | UC-48 prep | Reject, resubmit |
| `SOLUTION3_ID` | UC-48 prep | Multi-contributor |
| `DRAFT_SOL_ID` | UC-82 | DRAFT approval block test |
| `ATTACHMENT_ID` | UC-69 | Delete attachment (UC-71) |
| `KB_ARTICLE_ID` | UC-76 / UC-84 | KB read, rate, view |
| `MANUAL_KB_ID` | UC-85 | Publish, archive, update, delete (UC-89) |
| `TAG_TEST_ARTICLE_ID` | UC-94 | Tag render assertion |
| `RATING_ID` | UC-97 | Delete rating |
| `CATEGORY_ID` | UC-102 | Subcategories, update, delete |
| `TAG_ID` | UC-107 | Delete tag |
| `BADGE_ID` | UC-123 | Assign badge |
| `NOTIF_ID` | UC-131 | View, mark read, delete |
| `TEMPLATE_ID` | UC-145 | Update, delete template |
| `COMMENT_ID` | UC-61 | Edit, delete comment |

---

## Phase 0 — Account Setup (Run Once Before All Tests)

Register all 12 accounts and save their tokens and auth IDs.  
**No precondition** — do this fresh against a running system.

### Step 0.1 — Register testuser1 (USER)

**Request:**
```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{
  "email": "testuser1@corp.com",
  "password": "Test@1234",
  "role": "USER"
}
```
**Expected:** HTTP 200  
**Action:** Save `data.authUserId` → `AUTH_ID_USER1`

---

### Step 0.2 — Register testuser2 (USER)

```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "testuser2@corp.com", "password": "Test@1234", "role": "USER" }
```
**Expected:** HTTP 200  
**Action:** Save `data.authUserId` → `AUTH_ID_USER2`

---

### Step 0.3 — Register testuser3 (USER)

```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "testuser3@corp.com", "password": "Test@1234", "role": "USER" }
```
**Expected:** HTTP 200  
**Action:** Save `data.authUserId` → `AUTH_ID_USER3`

---

### Step 0.4 — Register testuser4 (USER)

```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "testuser4@corp.com", "password": "Test@1234", "role": "USER" }
```
**Expected:** HTTP 200  
**Action:** Save `data.authUserId` → `AUTH_ID_USER4`

---

### Step 0.5 — Register testmgr1 (MANAGER)

```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "testmgr1@corp.com", "password": "Test@1234", "role": "MANAGER" }
```
**Expected:** HTTP 200

---

### Step 0.6 — Register testmgr2, testmgr3, testmgr4 (MANAGER)

Repeat Step 0.5 for:
- `testmgr2@corp.com`
- `testmgr3@corp.com`
- `testmgr4@corp.com`

---

### Step 0.7 — Register testadmin1, testadmin2, testadmin3, testadmin4 (ADMIN)

```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "testadmin1@corp.com", "password": "Test@1234", "role": "ADMIN" }
```
Repeat for testadmin2, testadmin3, testadmin4.

---

### Step 0.8 — Login All Accounts and Save Tokens

For each account, call:
```
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{ "email": "testuser1@corp.com", "password": "Test@1234" }
```
**Expected:** HTTP 200, body contains `accessToken`  
**Action:** Save `data.accessToken` as the corresponding TOKEN variable.

| Account | Token Variable |
|---------|---------------|
| testuser1@corp.com | TOKEN_USER1 |
| testuser2@corp.com | TOKEN_USER2 |
| testuser3@corp.com | TOKEN_USER3 |
| testuser4@corp.com | TOKEN_USER4 |
| testmgr1@corp.com | TOKEN_MGR1 |
| testmgr2@corp.com | TOKEN_MGR2 |
| testmgr3@corp.com | TOKEN_MGR3 |
| testmgr4@corp.com | TOKEN_MGR4 |
| testadmin1@corp.com | TOKEN_ADMIN1 |
| testadmin2@corp.com | TOKEN_ADMIN2 |
| testadmin3@corp.com | TOKEN_ADMIN3 |
| testadmin4@corp.com | TOKEN_ADMIN4 |

> All subsequent requests use `Authorization: Bearer {{TOKEN_XXXX}}` in the header.

---

## Phase 1 — Authentication & Account Management

---

test_user1

### UC-01 — Register new USER

**Precondition:** None  
**Actor:** Anonymous

**Steps:**
1. Use a unique email (timestamp-based to avoid conflicts)

**Request:**
```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "newuser_001@test.com", "password": "Passw0rd!", "role": "USER" }
```
**Expected Code:** 200  
**Expected Body:** Contains `authUserId`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-02 — Register new ADMIN

**Precondition:** None  
**Actor:** Anonymous

**Request:**
```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "newadmin_001@test.com", "password": "Passw0rd!", "role": "ADMIN" }
```
**Expected Code:** 200  
**Expected Body:** Contains `authUserId`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-03 — Register new MANAGER

**Precondition:** None  
**Actor:** Anonymous

**Request:**
```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "newmgr_001@test.com", "password": "Passw0rd!", "role": "MANAGER" }
```
**Expected Code:** 200  
**Expected Body:** Contains `authUserId`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-04 — Register with blank email returns error

**Precondition:** None  
**Actor:** Anonymous

**Request:**
```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "", "password": "Passw0rd!", "role": "USER" }
```
**Expected Code:** 400 (NOT 200)  
**Expected Body:** Contains validation error

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-05 — Register duplicate email returns error

**Precondition:** testuser1@corp.com already registered (Phase 0)  
**Actor:** Anonymous

**Request:**
```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{ "email": "testuser1@corp.com", "password": "Passw0rd!", "role": "USER" }
```
**Expected Code:** 409 (NOT 200)  
**Expected Body:** Contains duplicate/conflict error

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-06 — Login correct credentials returns accessToken

**Precondition:** testuser1@corp.com registered  
**Actor:** Anonymous

**Request:**
```
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{ "email": "testuser1@corp.com", "password": "Test@1234" }
```
**Expected Code:** 200  
**Expected Body:** Contains `accessToken`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-07 — Login wrong password returns error

**Precondition:** testuser1@corp.com registered  
**Actor:** Anonymous

**Request:**
```
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{ "email": "testuser1@corp.com", "password": "WrongPassword!" }
```
**Expected Code:** 401 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-08 — Refresh token

**Precondition:** testuser1 logged in  
**Actor:** Anonymous (uses refresh token)

**Steps:**
1. From the UC-06 login response, copy the `refreshToken` field

**Request:**
```
POST http://localhost:8080/api/auth/refresh
Content-Type: application/json

{ "refreshToken": "{{REFRESH_TOKEN_FROM_LOGIN}}" }
```
**Expected Code:** 200  
**Expected Body:** Contains new `accessToken`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-09 — Change password (same password) returns non-500

**Precondition:** testadmin1 logged in  
**Actor:** testadmin1

**Request:**
```
PUT http://localhost:8080/api/auth/change-password
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{ "oldPassword": "Test@1234", "newPassword": "Test@1234" }
```
**Expected Code:** 400 (NOT 500) — same password rejected  
**Expected Body:** Contains error message

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-10 — Reset password request returns non-500

**Precondition:** None  
**Actor:** Anonymous

**Request:**
```
POST http://localhost:8080/api/auth/reset-password
Content-Type: application/json

{ "email": "testuser1@corp.com" }
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-11 — Logout returns non-500

**Precondition:** testadmin1 logged in  
**Actor:** testadmin1

**Request:**
```
POST http://localhost:8080/api/auth/logout
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{}
```
**Expected Code:** 200 (NOT 500)

> Note: After this, TOKEN_ADMIN1 may be invalidated. Re-login testadmin1 if needed.

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-12 — Lock user account

**Precondition:** testuser4 registered, testadmin1 logged in. Get testuser4's auth UUID.  
**Actor:** testadmin1

**Steps:**
1. Query the auth DB or use the `AUTH_ID_USER4` saved from registration
2. Call the lock endpoint with that UUID

**Request:**
```
PUT http://localhost:8080/api/auth/users/{{AUTH_ID_USER4}}/lock
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** User locked confirmation

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-13 — Unlock user account

**Precondition:** testuser4 locked (UC-12 done), testadmin1 logged in  
**Actor:** testadmin1

**Request:**
```
PUT http://localhost:8080/api/auth/users/{{AUTH_ID_USER4}}/unlock
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-14 — View login history

**Precondition:** testadmin1 has logged in at least once  
**Actor:** testadmin1

**Steps:**
1. Use `AUTH_ID_ADMIN1` (from registration response or look it up)

**Request:**
```
GET http://localhost:8080/api/auth/users/{{AUTH_ID_ADMIN1}}/login-history
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Array of login history entries

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-15 — Unauthenticated access blocked

**Precondition:** None  
**Actor:** Anonymous (no token)

**Request:**
```
GET http://localhost:8080/api/tickets
(No Authorization header)
```
**Expected Code:** 401 or 403

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-16 — USER cannot approve solution

**Precondition:** testuser1 logged in  
**Actor:** testuser1

**Request:**
```
PUT http://localhost:8080/api/solutions/00000000-0000-0000-0000-000000000001/approve
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{}
```
**Expected Code:** 403 (or 404) — NOT 200

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-17 — MANAGER cannot list all users

**Precondition:** testmgr1 logged in  
**Actor:** testmgr1

**Request:**
```
GET http://localhost:8080/api/users
Authorization: Bearer {{TOKEN_MGR1}}
```
**Expected Code:** 403 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-18 — Expired token handling

**Precondition:** Restart auth-service with the test profile (10-second JWT expiry):
```
cd auth_service
mvn -DskipTests spring-boot:run -Dspring-boot.run.profiles=test
```
Wait for it to start on :8081, then proceed.

**Step 1 — Login to get a short-lived token:**
```
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{
  "email": "testuser1@corp.com",
  "password": "Test@1234"
}
```
**Expected:** HTTP 200 — save `data.accessToken` as `TOKEN_EXPIRED`

**Step 2 — Wait 15 seconds** (token expires after 10s)

**Step 3 — Use the expired token on any protected endpoint:**
```
GET http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_EXPIRED}}
```
**Expected Code:** 401  
**Expected Body:** `{"success": false, "message": "...token expired..."}` or similar

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

> **After test:** Restart auth-service normally with `--spring.profiles.active=local` to restore the 1-hour expiry.

---

## Phase 2 — User Profiles & Departments

---

### UC-19 — Create user profile

**Precondition:** testuser2 logged in; a department must exist  
**Actor:** testuser2

**Steps:**
1. First create a department (Step 2a below — direct call to user-service port 8082)
2. Save the department ID as `DEPT_ID`

**Step 2a — Create department (prerequisite):**
```
POST http://localhost:8082/api/departments
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{ "name": "Engineering", "description": "Engineering department" }
```
Save response `id` or `departmentId` → `DEPT_ID`

**Request (UC-19):**
```
POST http://localhost:8080/api/users
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{
  "name": "Test User Two",
  "email": "testuser2@corp.com",
  "departmentId": "{{DEPT_ID}}",
  "technicalLevel": "MID",
  "skillLevel": "INTERMEDIATE"
}
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-20 — Get user by ID

**Precondition:** At least one user profile exists in user-service  
**Actor:** testuser1

**Steps:**
1. Call `GET /api/users` as testadmin1 to get a list of users
2. Copy the first user's `id` → `USER_PROFILE_ID`

**Request:**
```
GET http://localhost:8080/api/users/{{USER_PROFILE_ID}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains user details

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-21 — Update user profile

**Precondition:** testuser2 profile created (UC-19), have `USER_PROFILE_ID`  
**Actor:** testuser2

**Request:**
```
PUT http://localhost:8080/api/users/{{USER_PROFILE_ID}}
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{ "name": "Test User Two Updated" }
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-22 — Admin list all users

**Precondition:** testadmin1 logged in  
**Actor:** testadmin1

**Request:**
```
GET http://localhost:8080/api/users
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 200  
**Expected Body:** Array/page of users

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-23 — Create department (direct user-service call)

**Precondition:** testadmin3 logged in  
**Actor:** testadmin3 (direct call — not via gateway)

**Request:**
```
POST http://localhost:8082/api/departments
Authorization: Bearer {{TOKEN_ADMIN3}}
Content-Type: application/json

{ "name": "QA", "description": "Quality Assurance" }
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

## Phase 3 — Ticket Management

**Prerequisite for this phase:** Create 4 tickets before running UC-25 onwards.

### Ticket Setup — Create Tickets

Run each of these and save the returned `ticketId` values:

```
POST http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{
  "title": "VPN Connection Drops",
  "description": "VPN drops every 30 minutes on Windows",
  "priority": "HIGH",
  "categoryId": 1,
  "difficultyLevel": "MEDIUM",
  "visibility": "ORGANIZATION"
}
```
Save `ticketId` → `TICKET1_ID`

```
POST http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{
  "title": "Printer Offline",
  "description": "Floor 2 printer not responding",
  "priority": "LOW",
  "categoryId": 2,
  "difficultyLevel": "EASY",
  "visibility": "ORGANIZATION"
}
```
Save `ticketId` → `TICKET2_ID`

```
POST http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{
  "title": "Login Page 500 Error",
  "description": "Users cannot login — 500 on submit",
  "priority": "URGENT",
  "categoryId": 3,
  "difficultyLevel": "HARD",
  "visibility": "ORGANIZATION"
}
```
Save `ticketId` → `TICKET3_ID`

```
POST http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_USER4}}
Content-Type: application/json

{
  "title": "Email Not Sending",
  "description": "Outbound SMTP failing since 9am",
  "priority": "HIGH",
  "categoryId": 1,
  "difficultyLevel": "MEDIUM",
  "visibility": "ORGANIZATION"
}
```
Save `ticketId` → `TICKET4_ID`

---

### UC-24 — Create ticket

**Precondition:** testuser1 logged in  
**Actor:** testuser1

**Steps:**
1. Use the Ticket Setup block above
2. Verify the first `POST /api/tickets` returns HTTP 201 and a `ticketId`

**Expected Code:** 201  
**Expected Body:** Contains `ticketId`, `status: "OPEN"`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-25 — View all tickets (paginated)

**Precondition:** At least one ticket exists (from Setup)  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Paginated list containing at least 1 ticket

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-26 — View ticket detail

**Precondition:** TICKET1_ID saved  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/tickets/{{TICKET1_ID}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Contains `ticketId`, `title: "VPN Connection Drops"`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-27 — View my tickets

**Precondition:** testuser1 has created at least one ticket  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/tickets/my
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Contains tickets created by testuser1

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-28 — Search tickets by title

**Precondition:** TICKET1_ID exists (title contains "VPN")  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/tickets/search?title=VPN
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Contains at least one result with "VPN" in the title

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-29 — Search tickets by status=OPEN

**Precondition:** At least one ticket with status OPEN  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/tickets/search?status=OPEN
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** All results have `status: "OPEN"`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-30 — Search tickets by priority=HIGH

**Precondition:** Tickets with HIGH priority exist  
**Actor:** testadmin2

**Request:**
```
GET http://localhost:8080/api/tickets/search?priority=HIGH
Authorization: Bearer {{TOKEN_ADMIN2}}
```
**Expected Code:** 200

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-31 — Search tickets by difficultyLevel=MEDIUM

**Precondition:** Tickets with MEDIUM difficulty exist  
**Actor:** testadmin2

**Request:**
```
GET http://localhost:8080/api/tickets/search?difficultyLevel=MEDIUM
Authorization: Bearer {{TOKEN_ADMIN2}}
```
**Expected Code:** 200

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-32 — Search tickets by categoryId=1

**Precondition:** Tickets with categoryId=1 exist  
**Actor:** testadmin2

**Request:**
```
GET http://localhost:8080/api/tickets/search?categoryId=1
Authorization: Bearer {{TOKEN_ADMIN2}}
```
**Expected Code:** 200

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-33 — Search tickets multi-filter combined

**Precondition:** TICKET1_ID (VPN, OPEN, HIGH) exists  
**Actor:** testadmin2

**Request:**
```
GET http://localhost:8080/api/tickets/search?title=VPN&status=OPEN&priority=HIGH
Authorization: Bearer {{TOKEN_ADMIN2}}
```
**Expected Code:** 200  
**Expected Body:** Narrowed list matching all filters

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-34 — Search tickets with no filters

**Precondition:** Tickets exist  
**Actor:** testadmin2

**Request:**
```
GET http://localhost:8080/api/tickets/search
Authorization: Bearer {{TOKEN_ADMIN2}}
```
**Expected Code:** 200  
**Expected Body:** All non-deleted tickets returned

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-35 — Ticket list page 2 size 3

**Precondition:** At least 4 tickets exist  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/tickets?page=1&size=3
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Page 2 with up to 3 items

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-36 — Update ticket

**Precondition:** TICKET2_ID exists  
**Actor:** testadmin1

**Request:**
```
PUT http://localhost:8080/api/tickets/{{TICKET2_ID}}
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{ "title": "Printer Offline — Updated", "priority": "MEDIUM" }
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-37 — Assign ticket to user

**Precondition:** TICKET3_ID exists, AUTH_ID_USER1 known  
**Actor:** testadmin1

**Steps:**
1. Use `AUTH_ID_USER1` as the assignee UUID

**Request:**
```
POST http://localhost:8080/api/tickets/{{TICKET3_ID}}/assign
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{ "assignedTo": "{{AUTH_ID_USER1}}" }
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Ticket updated with `assignedTo` set  
**Side effect:** After ~4 seconds, testuser1 should receive a "ticket assigned" notification

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-38 — Change ticket status to IN_PROGRESS

**Precondition:** TICKET4_ID exists (status=OPEN)  
**Actor:** testadmin2

**Request:**
```
PUT http://localhost:8080/api/tickets/status
Authorization: Bearer {{TOKEN_ADMIN2}}
Content-Type: application/json

{ "ticketId": "{{TICKET4_ID}}", "status": "IN_PROGRESS" }
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Ticket with `status: "IN_PROGRESS"`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-39 — Change ticket status to RESOLVED

**Precondition:** TICKET4_ID is IN_PROGRESS (UC-38 done)  
**Actor:** testadmin2

**Request:**
```
PUT http://localhost:8080/api/tickets/status
Authorization: Bearer {{TOKEN_ADMIN2}}
Content-Type: application/json

{ "ticketId": "{{TICKET4_ID}}", "status": "RESOLVED" }
```
**Expected Code:** 200 (NOT 500)  
**Side effect:** Kafka fires `ticket.resolved` → reward service awards +50 points to the assignee

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-40 — Resolve ticket via /resolve endpoint

**Precondition:** A ticket exists (not yet resolved — use TICKET3_ID)  
**Actor:** testadmin2

**Request:**
```
PUT http://localhost:8080/api/tickets/resolve
Authorization: Bearer {{TOKEN_ADMIN2}}
Content-Type: application/json

{ "ticketId": "{{TICKET3_ID}}" }
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-41 — Ticket statistics

**Precondition:** Tickets exist  
**Actor:** testadmin2

**Request:**
```
GET http://localhost:8080/api/tickets/statistics
Authorization: Bearer {{TOKEN_ADMIN2}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains `totalTickets`, `openTickets`, `resolvedTickets`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-42 — Add comment to ticket

**Precondition:** TICKET1_ID exists, testuser1 logged in  
**Actor:** testuser1

**Request:**
```
POST http://localhost:8080/api/tickets/{{TICKET1_ID}}/comments
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{ "content": "This also happens on mobile VPN" }
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-43 — View ticket comments

**Precondition:** Comment added in UC-42  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/tickets/{{TICKET1_ID}}/comments
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains the comment from UC-42

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-44 — Add contributor to ticket

**Precondition:** TICKET1_ID exists, AUTH_ID_USER2 known  
**Actor:** testadmin1

**Request:**
```
POST http://localhost:8080/api/tickets/{{TICKET1_ID}}/contributors
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{ "userId": "{{AUTH_ID_USER2}}" }
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-45 — Remove contributor from ticket

**Precondition:** testuser2 added as contributor in UC-44  
**Actor:** testadmin1

**Request:**
```
DELETE http://localhost:8080/api/tickets/{{TICKET1_ID}}/contributors/{{AUTH_ID_USER2}}
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-46 — Rate resolved ticket

**Precondition:** TICKET4_ID is RESOLVED (UC-39 done)  
**Actor:** testuser1

**Request:**
```
POST http://localhost:8080/api/tickets/rating
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{
  "ticketId": "{{TICKET4_ID}}",
  "rating": 4,
  "feedback": "Resolved quickly and efficiently"
}
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-47 — Delete ticket (soft delete)

**Precondition:** Create a disposable ticket first  
**Actor:** testadmin1

**Steps:**
1. Create a fresh ticket to delete:
```
POST http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{
  "title": "Disposable Ticket",
  "description": "Created only to be deleted",
  "priority": "LOW",
  "categoryId": 1,
  "difficultyLevel": "EASY",
  "visibility": "ORGANIZATION"
}
```
Save `ticketId` → `DISPOSE_TICKET_ID`

**Request (UC-47):**
```
DELETE http://localhost:8080/api/tickets/{{DISPOSE_TICKET_ID}}
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

## Phase 4 — Solution Workflow (State Machine)

**Prerequisite:** TICKET1_ID, TICKET2_ID, TICKET3_ID must exist from Phase 3 setup.

### Solution Setup — Create 3 Solutions

Before running UC-49 onwards, create these 3 solutions and save their IDs:

**Solution 1** (by testuser1, for TICKET1):
```
POST http://localhost:8080/api/solutions
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{
  "title": "Fix VPN by updating client",
  "solutionContent": "Update VPN client to v3.2 and restart the VPN service",
  "ticketId": "{{TICKET1_ID}}"
}
```
Save `data.solutionId` → `SOLUTION1_ID`

**Solution 2** (by testuser2, for TICKET2 — will be rejected then re-approved):
```
POST http://localhost:8080/api/solutions
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{
  "title": "Printer driver reinstall",
  "solutionContent": "Reinstall driver from manufacturer site",
  "ticketId": "{{TICKET2_ID}}"
}
```
Save `data.solutionId` → `SOLUTION2_ID`

**Solution 3** (by testuser3, for TICKET3 — multi-contributor):
```
POST http://localhost:8080/api/solutions
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{
  "title": "Fix login 500 by rolling back deployment",
  "solutionContent": "Rollback to v2.1.4 using: kubectl rollout undo deployment/auth-svc",
  "ticketId": "{{TICKET3_ID}}"
}
```
Save `data.solutionId` → `SOLUTION3_ID`

---

### UC-48 — Create solution starts as DRAFT

**Precondition:** TICKET1_ID exists  
**Actor:** testuser1

**Steps:**
1. Use the Solution Setup above
2. Verify the `POST /api/solutions` for Solution 1 returns HTTP 201

**Expected Code:** 201  
**Expected Body:** `data.status: "DRAFT"`, `data.solutionId` present

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-49 — View solution detail

**Precondition:** SOLUTION1_ID saved  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/solutions/{{SOLUTION1_ID}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Contains `solutionId`, `title`, `status: "DRAFT"`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-50 — View solutions for a ticket

**Precondition:** TICKET1_ID has at least one solution  
**Actor:** testuser2

**Request:**
```
GET http://localhost:8080/api/solutions/ticket/{{TICKET1_ID}}
Authorization: Bearer {{TOKEN_USER2}}
```
**Expected Code:** 200  
**Expected Body:** Paginated list containing Solution 1

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-51 — View all solutions

**Precondition:** Solutions exist  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/solutions
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Paginated list of all non-deleted solutions

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-52 — View my solutions

**Precondition:** testuser1 has created at least one solution  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/solutions/my
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Only solutions created by testuser1

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-53 — View pending solutions (manager)

**Precondition:** At least one solution has status UNDER_REVIEW  
**Actor:** testmgr1

**Steps:**
1. First submit Solution 1 for review (do this now if not done yet):
```
PATCH http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/submit
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{}
```

**Request (UC-53):**
```
GET http://localhost:8080/api/solutions/pending
Authorization: Bearer {{TOKEN_MGR1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains solutions with `status: "UNDER_REVIEW"`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-54 — Solution statistics

**Precondition:** Solutions exist  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/solutions/statistics
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains `totalSolutions`, `approvedSolutions`, `pendingSolutions`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-55 — Update own solution (while in DRAFT)

**Precondition:** SOLUTION1_ID exists and is in DRAFT state (NOT yet submitted in this test run)  
**Actor:** testuser1

> If Solution 1 was already submitted for review in UC-53, use Solution 2 (which is still DRAFT) with TOKEN_USER2 instead.

**Request:**
```
PUT http://localhost:8080/api/solutions/{{SOLUTION1_ID}}
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{
  "title": "Fix VPN by updating client — Revised",
  "solutionContent": "Update VPN client to v3.2.1 and restart the service. Also clear DNS cache.",
  "changeSummary": "Added DNS cache clearing step"
}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Updated `title` and incremented `currentVersion`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-56 — Submit solution for review (DRAFT → UNDER_REVIEW)

**Precondition:** SOLUTION1_ID is in DRAFT  
**Actor:** testuser1

**Request:**
```
PATCH http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/submit
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{}
```
**Expected Code:** 200 (NOT 500 and NOT 400)  
**Expected Body:** `status: "UNDER_REVIEW"`

> Also submit Solution 2 and Solution 3 for review now:
```
PATCH http://localhost:8080/api/solutions/{{SOLUTION2_ID}}/submit
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{}
```
```
PATCH http://localhost:8080/api/solutions/{{SOLUTION3_ID}}/submit
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{}
```

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-57 — Add contributor to solution

**Precondition:** SOLUTION3_ID exists, AUTH_ID_USER4 known  
**Actor:** testuser3 (author of Solution 3)

**Request:**
```
POST http://localhost:8080/api/solutions/{{SOLUTION3_ID}}/contributors
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{
  "solutionId": "{{SOLUTION3_ID}}",
  "userId": "{{AUTH_ID_USER4}}"
}
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-58 — Remove contributor from solution

**Precondition:** testuser4 is a contributor on Solution 3 (UC-57 done)  
**Actor:** testuser3

**Request:**
```
DELETE http://localhost:8080/api/solutions/{{SOLUTION3_ID}}/contributors/{{AUTH_ID_USER4}}
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200 or 204 (NOT 500)

> **Important:** Re-add testuser4 as contributor after this test, so the multi-contributor reward test (UC-126) works:
```
POST http://localhost:8080/api/solutions/{{SOLUTION3_ID}}/contributors
Authorization: Bearer {{TOKEN_MGR1}}
Content-Type: application/json

{ "solutionId": "{{SOLUTION3_ID}}", "userId": "{{AUTH_ID_USER4}}" }
```

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-59 — Add duplicate contributor (idempotency)

**Precondition:** testuser4 already a contributor on Solution 3  
**Actor:** testuser3

**Request:**
```
POST http://localhost:8080/api/solutions/{{SOLUTION3_ID}}/contributors
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{ "solutionId": "{{SOLUTION3_ID}}", "userId": "{{AUTH_ID_USER4}}" }
```
**Expected Code:** 200 or 409 (NOT 500) — duplicate handled gracefully

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-60 — View contributors

**Precondition:** SOLUTION3_ID has contributors  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/solutions/{{SOLUTION3_ID}}/contributors
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains contributor list including testuser3 and testuser4

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-61 — Add comment to solution

**Precondition:** SOLUTION1_ID exists  
**Actor:** testmgr1 (comment requires MANAGER/ADMIN/ENGINEER role)

**Request:**
```
POST http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/comments
Authorization: Bearer {{TOKEN_MGR1}}
Content-Type: application/json

{
  "solutionId": "{{SOLUTION1_ID}}",
  "commentText": "Does this fix IPv6 VPN tunnelling as well?"
}
```
**Expected Code:** 200 or 201 (NOT 500)  
**Action:** Save `data.commentId` → `COMMENT_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-62 — View comments paginated

**Precondition:** Comment added in UC-61  
**Actor:** testmgr1

**Request:**
```
GET http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/comments/paginated
Authorization: Bearer {{TOKEN_MGR1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Paginated list containing the comment from UC-61

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-63 — Edit own comment

**Precondition:** COMMENT_ID saved from UC-61  
**Actor:** testmgr1

**Request:**
```
PUT http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/comments/{{COMMENT_ID}}
Authorization: Bearer {{TOKEN_MGR1}}
Content-Type: application/json

"Updated: Does this fix IPv6 tunnelling and split-tunnel configs?"
```
> Note: Body is a plain string (not JSON object)

**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-64 — Delete own comment

**Precondition:** COMMENT_ID exists  
**Actor:** testmgr1

**Request:**
```
DELETE http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/comments/{{COMMENT_ID}}
Authorization: Bearer {{TOKEN_MGR1}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-65 — Upvote a solution

**Precondition:** SOLUTION1_ID is UNDER_REVIEW; testuser2 is NOT the author  
**Actor:** testuser2

**Request:**
```
POST http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/votes
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{ "voteType": "UPVOTE" }
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-66 — Downvote a solution

**Precondition:** SOLUTION1_ID exists; testuser3 not the author  
**Actor:** testuser3

**Request:**
```
POST http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/votes
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{ "voteType": "DOWNVOTE" }
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-67 — Remove own vote

**Precondition:** testuser3 has voted on SOLUTION1_ID (UC-66 done)  
**Actor:** testuser3

**Request:**
```
DELETE http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/votes
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-68 — Vote on own solution (documents behaviour)

**Precondition:** SOLUTION1_ID created by testuser1  
**Actor:** testuser1 (voting on own solution)

**Request:**
```
POST http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/votes
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{ "voteType": "UPVOTE" }
```
**Expected Code:** Any (400 is expected — self-vote blocked). Document the actual code returned.  
**Expected Behaviour:** Not 500; gracefully rejected or allowed as documented

**Result:** [ ] PASS  [ ] FAIL  Notes: Actual HTTP code: ___________

---

### UC-69 — Upload attachment

**Precondition:** SOLUTION1_ID exists, testadmin1 logged in (endpoint requires ENGINEER/MANAGER/ADMIN role — no ENGINEER accounts in Phase 0, use ADMIN)  
**Actor:** testadmin1

> **Note:** This endpoint takes a **JSON body** (not multipart/form-data). Provide a URL reference to the file — no binary upload needed.

**Request:**
```
POST http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/attachments
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{
  "fileName": "fix-notes.txt",
  "fileUrl": "https://example.com/attachments/fix-notes.txt",
  "fileType": "text/plain",
  "fileSize": 1024
}
```
**Expected Code:** 201  
**Expected Body:** Contains `attachmentId`, `fileName`, `fileUrl`, `uploadedBy`  
**Action:** Save `data.attachmentId` → `ATTACHMENT_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-70 — View attachments

**Precondition:** SOLUTION1_ID exists (even with no attachments)  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/attachments
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Empty array or list of attachments

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-71 — Delete attachment

**Precondition:** ATTACHMENT_ID saved from UC-69, testadmin1 logged in (same role requirement as UC-69)  
**Actor:** testadmin1

**Request:**
```
DELETE http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/attachments/{{ATTACHMENT_ID}}
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 200  
**Expected Body:** `{"success": true, "message": "Attachment deleted successfully"}`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-72 — Approve solution

**Precondition:** SOLUTION1_ID is in UNDER_REVIEW state  
**Actor:** testmgr1

**Request:**
```
PUT http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/approve
Authorization: Bearer {{TOKEN_MGR1}}
Content-Type: application/json

{}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `status: "APPROVED"`, `isApproved: true`  
**Side effects (wait 4–5 seconds):**
- Reward service awards points to testuser1 (+30 for solution approval)
- Notification service creates "Solution Approved" notification for testuser1
- Knowledge service auto-creates a KB article from this solution

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-73 — Points awarded after solution approval

**Precondition:** UC-72 done, wait ~5 seconds for Kafka to process  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/points
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `totalPoints` is greater than 0

**Result:** [ ] PASS  [ ] FAIL  Actual points: ___________

---

### UC-74 — Solution approved notification exists

**Precondition:** UC-72 done, wait ~5 seconds  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER1}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains at least one notification

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-75 — Points Earned notification found

**Precondition:** UC-72 done and Kafka has processed reward event  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER1}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** At least one notification contains "points" or "earned" (case-insensitive)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-76 — KB article auto-created from approved solution

**Precondition:** UC-72 done, wait ~5–8 seconds for Kafka  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/knowledge/by-solution/{{SOLUTION1_ID}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Contains `id`/`articleId` of the auto-generated KB article  
**Action:** Save the article ID → `KB_ARTICLE_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-77 — Reject solution

**Precondition:** SOLUTION2_ID is in UNDER_REVIEW state  
**Actor:** testmgr2

**Request:**
```
PUT http://localhost:8080/api/solutions/{{SOLUTION2_ID}}/reject
Authorization: Bearer {{TOKEN_MGR2}}
Content-Type: application/json

{ "reason": "Solution is incomplete — missing rollback steps and testing plan" }
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `status: "REJECTED"`, `rejectionReason` present

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-78 — Rejection notification with reason found

**Precondition:** UC-77 done, wait ~4 seconds  
**Actor:** testuser2 (author of Solution 2)

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER2}}
Authorization: Bearer {{TOKEN_USER2}}
```
**Expected Code:** 200  
**Expected Body:** Contains a notification with "reject" or "rejected" (case-insensitive)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-79 — Resubmit after rejection (REJECTED → UNDER_REVIEW)

**Precondition:** SOLUTION2_ID is REJECTED (UC-77 done)  
**Actor:** testuser2

**Steps:**
1. Optionally update the solution first to fix the issues:
```
PUT http://localhost:8080/api/solutions/{{SOLUTION2_ID}}
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{
  "title": "Printer driver reinstall — Revised",
  "solutionContent": "Reinstall driver from manufacturer site. Rollback: restore previous driver from backup.",
  "changeSummary": "Added rollback steps and testing plan"
}
```
2. Then submit for review:

**Request (UC-79):**
```
PATCH http://localhost:8080/api/solutions/{{SOLUTION2_ID}}/submit
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `status: "UNDER_REVIEW"`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-80 — Re-approve resubmitted solution

**Precondition:** SOLUTION2_ID is UNDER_REVIEW again (UC-79 done)  
**Actor:** testmgr4

**Request:**
```
PUT http://localhost:8080/api/solutions/{{SOLUTION2_ID}}/approve
Authorization: Bearer {{TOKEN_MGR4}}
Content-Type: application/json

{}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `status: "APPROVED"`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-81 — Cannot re-approve already-approved solution

**Precondition:** SOLUTION1_ID is already APPROVED (UC-72 done)  
**Actor:** testmgr1

**Request:**
```
PUT http://localhost:8080/api/solutions/{{SOLUTION1_ID}}/approve
Authorization: Bearer {{TOKEN_MGR1}}
Content-Type: application/json

{}
```
**Expected Code:** 409 (NOT 200)  
**Expected Body:** Error message about already approved

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-82 — Cannot approve DRAFT solution (must be UNDER_REVIEW)

**Precondition:** Create a DRAFT solution (do NOT submit it for review)  
**Actor:** testmgr1

**Steps:**
1. Create a new DRAFT solution:
```
POST http://localhost:8080/api/solutions
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{
  "title": "Draft Only Solution",
  "solutionContent": "This solution will not be submitted for review",
  "ticketId": "{{TICKET1_ID}}"
}
```
Save `data.solutionId` → `DRAFT_SOL_ID`

2. Attempt to approve it while still in DRAFT:

**Request (UC-82):**
```
PUT http://localhost:8080/api/solutions/{{DRAFT_SOL_ID}}/approve
Authorization: Bearer {{TOKEN_MGR1}}
Content-Type: application/json

{}
```
**Expected Code:** 400 (NOT 200)  
**Expected Body:** Error stating solution must be UNDER_REVIEW

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-83 — Delete solution (admin)

**Precondition:** DRAFT_SOL_ID exists (from UC-82 setup)  
**Actor:** testadmin1

**Request:**
```
DELETE http://localhost:8080/api/solutions/{{DRAFT_SOL_ID}}
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### Multi-Contributor Setup (for UC-126 / Flow E)

Before proceeding to Phase 5, set up the multi-contributor reward scenario:

**Step 1:** Ensure testuser4 is a contributor on SOLUTION3_ID using a manager token (manager token bypasses `solutionId` validation):
```
POST http://localhost:8080/api/solutions/{{SOLUTION3_ID}}/contributors
Authorization: Bearer {{TOKEN_MGR1}}
Content-Type: application/json

{ "solutionId": "{{SOLUTION3_ID}}", "userId": "{{AUTH_ID_USER4}}" }
```

**Step 2:** Approve Solution 3 (multi-contributor approval):
```
PUT http://localhost:8080/api/solutions/{{SOLUTION3_ID}}/approve
Authorization: Bearer {{TOKEN_MGR3}}
Content-Type: application/json

{}
```
**Expected:** HTTP 200, `status: "APPROVED"`  
**Side effect:** After ~5 seconds, both testuser3 and testuser4 should each receive +30 reward points

---

## Phase 5 — Knowledge Base

---

### UC-84 — KB article auto-created (verify again)

**Precondition:** SOLUTION1_ID approved (UC-72 done), waited ~8 seconds  
**Actor:** testuser3 (any authenticated user)

**Request:**
```
GET http://localhost:8080/api/knowledge/by-solution/{{SOLUTION1_ID}}
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200  
**Expected Body:** `data.id` or `data.articleId` present  
**Action:** Confirm or update `KB_ARTICLE_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-85 — Admin manually create KB article

**Precondition:** testadmin3 logged in  
**Actor:** testadmin3

**Request:**
```
POST http://localhost:8080/api/knowledge
Authorization: Bearer {{TOKEN_ADMIN3}}
Content-Type: application/json

{
  "title": "How to reset VPN credentials",
  "content": "Step 1: Open VPN portal at vpn.corp.com. Step 2: Click Forgot Password. Step 3: Follow email instructions.",
  "summary": "Step-by-step guide for resetting VPN credentials"
}
```
**Expected Code:** 200 or 201 (NOT 500)  
**Action:** Save the article ID → `MANUAL_KB_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-86 — Publish KB article

**Precondition:** MANUAL_KB_ID exists (UC-85 done)  
**Actor:** testadmin3

**Request:**
```
PUT http://localhost:8080/api/knowledge/{{MANUAL_KB_ID}}/publish
Authorization: Bearer {{TOKEN_ADMIN3}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `status: "PUBLISHED"` or similar

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-87 — Archive KB article

**Precondition:** MANUAL_KB_ID published (UC-86 done)  
**Actor:** testadmin3

**Request:**
```
PUT http://localhost:8080/api/knowledge/{{MANUAL_KB_ID}}/archive
Authorization: Bearer {{TOKEN_ADMIN3}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `status: "ARCHIVED"` or similar

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-88 — Update KB article

**Precondition:** MANUAL_KB_ID exists  
**Actor:** testadmin3

**Request:**
```
PUT http://localhost:8080/api/knowledge/{{MANUAL_KB_ID}}
Authorization: Bearer {{TOKEN_ADMIN3}}
Content-Type: application/json

{ "title": "How to reset VPN credentials — Revised" }
```
**Expected Code:** 200 or 400 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-89 — Delete KB article

> **Run this after UC-101** (ratings depend on the article existing). See Phase 5 Cleanup section below.

---

### UC-90 — Browse KB articles (paginated)

**Precondition:** At least one KB article exists  
**Actor:** testuser3

**Request:**
```
GET http://localhost:8080/api/knowledge
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200  
**Expected Body:** Paginated list of articles

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-91 — Search KB by keyword

**Precondition:** An article with "VPN" in the title exists  
**Actor:** testuser3

**Request:**
```
GET http://localhost:8080/api/knowledge/search?keyword=VPN
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200  
**Expected Body:** Contains articles matching "VPN"

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-92 — View KB article detail

**Precondition:** KB_ARTICLE_ID or MANUAL_KB_ID known  
**Actor:** testuser3

**Request:**
```
GET http://localhost:8080/api/knowledge/{{KB_ARTICLE_ID}}
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200  
**Expected Body:** Full article with `title`, `content`, `summary`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-93 — Category field renders correctly

**Precondition:** KB_ARTICLE_ID known  
**Actor:** testuser3

**Request:**
```
GET http://localhost:8080/api/knowledge/{{KB_ARTICLE_ID}}
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200  
**Check:** The `category` field in the response should NOT be `"[object Object]"`. It should be `null`, a string name, or a proper object with `id` and `name`.

**Result:** [ ] PASS  [ ] FAIL  Notes: Category value: ___________

---

### UC-94 — Tag render on KB article

**Precondition:** A KB article exists with at least one tag. Create one if needed (requires ADMIN or MANAGER role):
```
POST http://localhost:8080/api/knowledge
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{
  "title": "Tag Render Test Article",
  "content": "Verifying tags appear in GET response",
  "tags": ["networking", "vpn"]
}
```
Save returned `data.articleId` → `TAG_TEST_ARTICLE_ID`

**Request:**
```
GET http://localhost:8080/api/knowledge/{{TAG_TEST_ARTICLE_ID}}
Authorization: Bearer {{TOKEN_ADMIN1}}
```
**Expected Code:** 200  
**Expected Body:** Response contains a `tags` field that is a **non-empty array** (e.g. `["networking","vpn"]` or objects with `id`/`name`). Must NOT be `null` or `[]`.

**Result:** [ ] PASS  [ ] FAIL  Notes: tags value: ___________

---

### UC-95 — Track article view count

**Precondition:** KB_ARTICLE_ID known  
**Actor:** testuser3

**Request:**
```
POST http://localhost:8080/api/knowledge/{{KB_ARTICLE_ID}}/view
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** View count incremented or acknowledged

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-96 — Get KB article by solution ID

**Precondition:** SOLUTION1_ID approved and KB article exists  
**Actor:** testuser3

**Request:**
```
GET http://localhost:8080/api/knowledge/by-solution/{{SOLUTION1_ID}}
Authorization: Bearer {{TOKEN_USER3}}
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-97 — Rate KB article

**Precondition:** KB_ARTICLE_ID known  
**Actor:** testuser2

> Ratings endpoint is on the knowledge-service directly (port 8085), not via gateway.

**Request:**
```
POST http://localhost:8085/api/ratings
Authorization: Bearer {{TOKEN_USER2}}
X-User-Id: {{AUTH_ID_USER2}}
Content-Type: application/json

{
  "articleId": "{{KB_ARTICLE_ID}}",
  "rating": 5,
  "feedback": "Very helpful, solved my issue immediately!"
}
```
**Expected Code:** 200 or 201 (NOT 500)  
**Action:** Save `data.ratingId` → `RATING_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-98 — Rate same article twice (upsert)

**Precondition:** testuser2 already rated KB_ARTICLE_ID (UC-97)  
**Actor:** testuser2

**Request:**
```
POST http://localhost:8085/api/ratings
Authorization: Bearer {{TOKEN_USER2}}
X-User-Id: {{AUTH_ID_USER2}}
Content-Type: application/json

{
  "articleId": "{{KB_ARTICLE_ID}}",
  "rating": 4,
  "feedback": "Slightly updated opinion — good but could have more detail"
}
```
**Expected Code:** 200 or 201 (NOT 500)  
**Expected Behaviour:** Rating upserted (not duplicate error)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-99 — Get all ratings for article

**Precondition:** KB_ARTICLE_ID has at least one rating  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8085/api/ratings/article/{{KB_ARTICLE_ID}}
Authorization: Bearer {{TOKEN_USER1}}
X-User-Id: {{AUTH_ID_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Array of ratings for the article

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-100 — Get average rating

**Precondition:** KB_ARTICLE_ID has at least one rating  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8085/api/ratings/article/{{KB_ARTICLE_ID}}/average
Authorization: Bearer {{TOKEN_USER1}}
X-User-Id: {{AUTH_ID_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains an average score value

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-101 — Delete a rating

**Precondition:** RATING_ID saved from UC-97  
**Actor:** testuser2

**Request:**
```
DELETE http://localhost:8085/api/ratings/{{RATING_ID}}
Authorization: Bearer {{TOKEN_USER2}}
X-User-Id: {{AUTH_ID_USER2}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### Phase 5 Cleanup — UC-89 — Delete KB article

> **Run after UC-101.** All rating tests that depend on the article are now complete.

**Precondition:** MANUAL_KB_ID exists (created in UC-85), testadmin3 logged in  
**Actor:** testadmin3

**Request:**
```
DELETE http://localhost:8080/api/knowledge/{{MANUAL_KB_ID}}
Authorization: Bearer {{TOKEN_ADMIN3}}
```
**Expected Code:** 200 or 204  
**Expected Body:** Success confirmation or empty body

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-102 — Create KB category

**Precondition:** testadmin3 logged in  
**Actor:** testadmin3

> Categories endpoint is on knowledge-service directly (port 8085).

**Request:**
```
POST http://localhost:8085/api/categories
Authorization: Bearer {{TOKEN_ADMIN3}}
X-User-Id: {{AUTH_ID_ADMIN3}}
X-User-Role: ROLE_ADMIN
Content-Type: application/json

{ "categoryName": "Networking", "description": "Network-related articles and guides" }
```
**Expected Code:** 200 or 201 (NOT 500)  
**Action:** Save category ID → `CATEGORY_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-103 — List KB categories

**Precondition:** At least one category exists  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8085/api/categories
Authorization: Bearer {{TOKEN_USER1}}
X-User-Id: {{AUTH_ID_USER1}}
X-User-Role: ROLE_USER
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Array of categories

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-104 — Get category subcategories

**Precondition:** CATEGORY_ID known  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8085/api/categories/{{CATEGORY_ID}}/subcategories
Authorization: Bearer {{TOKEN_USER1}}
X-User-Id: {{AUTH_ID_USER1}}
X-User-Role: ROLE_USER
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Array (may be empty)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-105 — Update KB category

**Precondition:** CATEGORY_ID known  
**Actor:** testadmin3

**Request:**
```
PUT http://localhost:8085/api/categories/{{CATEGORY_ID}}
Authorization: Bearer {{TOKEN_ADMIN3}}
X-User-Id: {{AUTH_ID_ADMIN3}}
X-User-Role: ROLE_ADMIN
Content-Type: application/json

{ "categoryName": "Networking — Updated" }
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-106 — Delete KB category

**Precondition:** CATEGORY_ID known  
**Actor:** testadmin3

**Request:**
```
DELETE http://localhost:8085/api/categories/{{CATEGORY_ID}}
Authorization: Bearer {{TOKEN_ADMIN3}}
X-User-Id: {{AUTH_ID_ADMIN3}}
X-User-Role: ROLE_ADMIN
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-107 — Create KB tag

**Precondition:** testadmin3 logged in  
**Actor:** testadmin3

**Request:**
```
POST http://localhost:8085/api/tags
Authorization: Bearer {{TOKEN_ADMIN3}}
X-User-Id: {{AUTH_ID_ADMIN3}}
X-User-Role: ROLE_ADMIN
Content-Type: application/json

{ "tagName": "vpn-fix" }
```
**Expected Code:** 200 or 201 (NOT 500)  
**Action:** Save tag ID → `TAG_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-108 — List KB tags

**Precondition:** At least one tag exists  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8085/api/tags
Authorization: Bearer {{TOKEN_USER1}}
X-User-Id: {{AUTH_ID_USER1}}
X-User-Role: ROLE_USER
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Array of tags including "vpn-fix"

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-109 — Delete KB tag

**Precondition:** TAG_ID known  
**Actor:** testadmin3

**Request:**
```
DELETE http://localhost:8085/api/tags/{{TAG_ID}}
Authorization: Bearer {{TOKEN_ADMIN3}}
X-User-Id: {{AUTH_ID_ADMIN3}}
X-User-Role: ROLE_ADMIN
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

## Phase 6 — Rewards & Gamification

**Prerequisite:** Solution approvals from Phase 4 must be complete and Kafka events processed (wait 5+ seconds).

---

### UC-110 — Points exist after ticket.resolved

**Precondition:** A ticket was RESOLVED in Phase 3 (TICKET4_ID resolved); testuser1 must have been assigned or involved  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/points
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `totalPoints` field present (may be 0 if user1 wasn't the ticket assignee)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-111 — Points exist after solution.approved

**Precondition:** SOLUTION1_ID approved (UC-72)  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/points
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** `totalPoints > 0`

**Result:** [ ] PASS  [ ] FAIL  Actual points: ___________

---

### UC-112 — Reward transactions include knowledge.created event

**Precondition:** KB article auto-created from approved solution (UC-76)  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/transactions
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Transaction list (check for a KNOWLEDGE_CREATED event type entry)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-113 — Transactions include vote reward

**Precondition:** SOLUTION1_ID was upvoted in UC-65  
**Actor:** testuser1 (author receives vote reward)

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/transactions
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains a SOLUTION_VOTED type transaction

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-114 — Admin manually award points

**Precondition:** testadmin1 logged in, AUTH_ID_USER4 known  
**Actor:** testadmin1 (or testadmin2)

**Request:**
```
POST http://localhost:8080/api/rewards/points
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{
  "userId": "{{AUTH_ID_USER4}}",
  "points": 25,
  "reason": "Bonus points for helping colleagues"
}
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-115 — View own total points

**Precondition:** testuser1 has earned points  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/points
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** `totalPoints` field with numeric value

**Result:** [ ] PASS  [ ] FAIL  Actual total: ___________

---

### UC-116 — View reward transactions

**Precondition:** testuser1 has transactions  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/transactions
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** List of reward transactions with `eventType`, `points`, `timestamp`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-117 — View contributions

**Precondition:** testuser1 has solution/ticket contributions  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/contributions
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-118 — View global leaderboard

**Precondition:** Points have been awarded  
**Actor:** testuser4 (any authenticated user)

**Request:**
```
GET http://localhost:8080/api/rewards/leaderboard
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Leaderboard array (may be empty on first run)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-119 — View top contributors

**Precondition:** Solutions have been approved  
**Actor:** testmgr1

**Request:**
```
GET http://localhost:8080/api/rewards/top-contributors
Authorization: Bearer {{TOKEN_MGR1}}
```
**Expected Code:** 200 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-120 — Generate/refresh leaderboard

**Precondition:** testadmin2 logged in  
**Actor:** testadmin2

**Request:**
```
POST http://localhost:8080/api/rewards/leaderboard/generate
Authorization: Bearer {{TOKEN_ADMIN2}}
Content-Type: application/json

{}
```
**Expected Code:** 200 or 400 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-121 — View badge catalog

**Precondition:** None  
**Actor:** testuser4

**Request:**
```
GET http://localhost:8080/api/badges
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200 or 404 (NOT 500)  
**Expected Body:** Array of badge definitions or empty

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-122 — View own badges

**Precondition:** testuser1 may or may not have badges  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/badges
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Array of badges (may be empty)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-123 — Admin assign badge manually

**Precondition:** A badge exists in the system  
**Actor:** testadmin1

**Steps:**
1. First create a badge if none exists:
```
POST http://localhost:8080/api/rewards/badges
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{
  "badgeName": "TICKET_SLAYER",
  "description": "Solved 10 or more tickets",
  "pointsRequired": 100
}
```
Save `badgeId` → `BADGE_ID`

2. Assign to testuser1:

**Request (UC-123):**
```
POST http://localhost:8080/api/rewards/badges/assign
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{
  "badgeId": "{{BADGE_ID}}",
  "userId": "{{AUTH_ID_USER1}}"
}
```
**Expected Code:** 200 or 409 (if already assigned — NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-124 — Create new badge definition

**Precondition:** testadmin1 logged in  
**Actor:** testadmin1

**Request:**
```
POST http://localhost:8080/api/rewards/badges
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{
  "badgeName": "SOLUTION_ARCHITECT",
  "description": "Contributed solutions that reached top 3 on leaderboard",
  "pointsRequired": 200
}
```
**Expected Code:** 200 or 409 (if already exists — NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-125 — Reward statistics

**Precondition:** Reward events have been processed  
**Actor:** testmgr1

**Request:**
```
GET http://localhost:8080/api/rewards/statistics
Authorization: Bearer {{TOKEN_MGR1}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains statistics like `totalPointsAwarded`, `totalUsers`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-126 — Multi-contributor reward verified

**Precondition:** SOLUTION3_ID approved with testuser3 (author) AND testuser4 (contributor). Wait ~5 seconds.  
**Actor:** testuser3 and testuser4

**Request 1 — testuser3 points:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER3}}/points
Authorization: Bearer {{TOKEN_USER3}}
```
**Request 2 — testuser4 points:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER4}}/points
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected:** Both users have `totalPoints > 0`  
**Note:** testuser4 should have received contributor-share reward points

**Result:** [ ] PASS  [ ] FAIL  user3 points: ___  user4 points: ___

---

## Phase 7 — Notifications

**Prerequisite:** UC-37 (ticket assigned), UC-72 (solution approved), UC-77 (solution rejected) must all be done. Wait 5+ seconds for Kafka processing.

---

### UC-127 — Ticket assigned notification exists

**Precondition:** Ticket was assigned to testuser1 (UC-37); waited ~5 seconds  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER1}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Contains at least one notification with "ticket" or "assign" in the title/message/type

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-128 — Solution approved notification exists

**Precondition:** SOLUTION1_ID approved (UC-72), waited ~5 seconds  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER1}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Contains a notification with "approved" or "solution" in the content

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-129 — Solution rejected notification exists

**Precondition:** SOLUTION2_ID rejected (UC-77); testuser2 is the author  
**Actor:** testuser2

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER2}}
Authorization: Bearer {{TOKEN_USER2}}
```
**Expected Code:** 200  
**Expected Body:** Contains a notification with "reject" or "rejected" in the content

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-130 — Points earned notification exists

**Precondition:** Solution approved → points awarded → notification sent (UC-72 + UC-73)  
**Actor:** testuser1

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER1}}
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected Code:** 200  
**Expected Body:** Contains a notification with "points" or "earned" in title/message

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-131 — View own notifications

**Precondition:** testuser4 may have notifications (from UC-114 points award or UC-139 broadcast)  
**Actor:** testuser4

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER4}}
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200  
**Action:** Save `content[0].notificationId` or `data[0].notificationId` → `NOTIF_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-132 — View unread notifications

**Precondition:** testuser4 has notifications  
**Actor:** testuser4

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER4}}?read=false
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Only unread notifications

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-133 — Get unread count (bell badge)

**Precondition:** testuser4 has unread notifications  
**Actor:** testuser4

**Request:**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER4}}/unread-count
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200  
**Expected Body:** Numeric count value (e.g. `{"data": 3}`)

**Result:** [ ] PASS  [ ] FAIL  Unread count: ___________

---

### UC-134 — View single notification

**Precondition:** NOTIF_ID saved from UC-131  
**Actor:** testuser4

**Request:**
```
GET http://localhost:8080/api/notifications/{{NOTIF_ID}}
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Full notification detail

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-135 — Mark single notification read

**Precondition:** NOTIF_ID known, AUTH_ID_USER4 known  
**Actor:** testuser4

**Request:**
```
PUT http://localhost:8080/api/notifications/{{NOTIF_ID}}/read?userId={{AUTH_ID_USER4}}
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-136 — Mark all notifications read

**Precondition:** testuser4 has unread notifications  
**Actor:** testuser4

**Request:**
```
PUT http://localhost:8080/api/notifications/users/{{AUTH_ID_USER4}}/read-all
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Verification (check unread count is now 0):**
```
GET http://localhost:8080/api/notifications/users/{{AUTH_ID_USER4}}/unread-count
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected:** Count = 0

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-137 — Delete notification

**Precondition:** NOTIF_ID known  
**Actor:** testuser4

**Request:**
```
DELETE http://localhost:8080/api/notifications/{{NOTIF_ID}}
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-138 — Notification statistics

**Precondition:** testadmin4 logged in  
**Actor:** testadmin4

**Request:**
```
GET http://localhost:8080/api/notifications/statistics
Authorization: Bearer {{TOKEN_ADMIN4}}
```
**Expected Code:** 200 (NOT 500)  
**Expected Body:** Contains `totalNotifications`, `sentNotifications`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-139 — Admin broadcast to all users

**Precondition:** testadmin4 logged in  
**Actor:** testadmin4

> Broadcast uses the internal endpoint directly on notification-service (not via gateway).

**Request:**
```
POST http://localhost:8087/internal/notifications/broadcast
Authorization: Bearer {{TOKEN_ADMIN4}}
Content-Type: application/json

{
  "title": "System Update",
  "message": "Scheduled maintenance at midnight tonight. Please save your work.",
  "type": "SYSTEM_ALERT",
  "sendToAll": true
}
```
**Expected Code:** 200 or 201 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-140 — Direct send notification to user

**Precondition:** testadmin4 logged in, AUTH_ID_USER4 known  
**Actor:** testadmin4

**Request:**
```
POST http://localhost:8080/api/notifications/send
Authorization: Bearer {{TOKEN_ADMIN4}}
Content-Type: application/json

{
  "title": "Direct Message",
  "message": "Hello testuser4 — this is a direct notification from admin",
  "recipientUserIds": ["{{AUTH_ID_USER4}}"],
  "type": "SYSTEM_ALERT"
}
```
**Expected Code:** 200 or 201 or 400 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-141 — Resend failed notification

> **PERMANENT SKIP** — No `/resend` or `/retry` REST endpoint is exposed by notification-service. Retry logic is handled internally via the `app.notification.retry` configuration (`max-attempts: 3`, `delay-ms: 1000`) applied at the service layer. Creating a deterministic delivery failure requires fault injection (invalid SMTP credentials, network drop) which is outside the scope of functional REST testing. Verified by code review of `NotificationServiceImpl` — not testable through REST in a normal flow.

---

### UC-142 — View notification preferences

**Precondition:** testuser4 logged in. Run UC-144 first to ensure preferences exist.  
**Actor:** testuser4

**Request:**
```
GET http://localhost:8080/api/notification-preferences/{{AUTH_ID_USER4}}
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 200  
**Expected Body:** Contains `emailEnabled`, `inAppEnabled`, `ticketUpdates`, `solutionUpdates`, `knowledgeUpdates`, `rewardUpdates` fields

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-143 — Update notification preferences

**Precondition:** Preferences exist for testuser4 (run UC-144 first), testuser4 logged in  
**Actor:** testuser4

**Request:**
```
PUT http://localhost:8080/api/notification-preferences/{{AUTH_ID_USER4}}
Authorization: Bearer {{TOKEN_USER4}}
Content-Type: application/json

{
  "emailEnabled": false,
  "inAppEnabled": true,
  "ticketUpdates": true,
  "solutionUpdates": true,
  "knowledgeUpdates": false,
  "rewardUpdates": true
}
```
**Expected Code:** 200  
**Expected Body:** Updated preference object reflecting the new values

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-144 — Create default preferences

**Precondition:** testuser4 logged in. Run this BEFORE UC-142 and UC-143.  
**Actor:** testuser4

**Request:**
```
POST http://localhost:8080/api/notification-preferences/{{AUTH_ID_USER4}}
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 201 (created) or 200 (already exists — idempotent)  
**Expected Body:** Default preferences object with all channels enabled

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-145 — Create notification template

**Precondition:** testadmin4 logged in  
**Actor:** testadmin4

> Templates are managed on the notification-service directly (port 8087).

**Request:**
```
POST http://localhost:8087/api/v1/notifications/templates
Authorization: Bearer {{TOKEN_ADMIN4}}
Content-Type: application/json

{
  "eventType": "BADGE_AWARDED",
  "titleTemplate": "Badge Earned: {{badge}}",
  "messageTemplate": "Congratulations {{userName}}! You earned the {{badge}} badge."
}
```
**Expected Code:** 200 or 201 (NOT 500)  
**Action:** Save `templateId` or `data.templateId` → `TEMPLATE_ID`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-146 — Get template by event type

**Precondition:** testadmin4 logged in  
**Actor:** testadmin4

**Request:**
```
GET http://localhost:8087/api/v1/notifications/templates/event-type/TICKET_ASSIGNED
Authorization: Bearer {{TOKEN_ADMIN4}}
```
**Expected Code:** 200 or 404 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-147 — Update notification template

**Precondition:** TEMPLATE_ID saved from UC-145  
**Actor:** testadmin4

**Request:**
```
PUT http://localhost:8087/api/v1/notifications/templates/{{TEMPLATE_ID}}
Authorization: Bearer {{TOKEN_ADMIN4}}
Content-Type: application/json

{
  "eventType": "BADGE_AWARDED",
  "titleTemplate": "UPDATED: New Badge Earned!",
  "messageTemplate": "Well done {{userName}} — you just earned the {{badge}} badge!"
}
```
**Expected Code:** 200 or 400 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-148 — Delete notification template

**Precondition:** TEMPLATE_ID known  
**Actor:** testadmin4

**Request:**
```
DELETE http://localhost:8087/api/v1/notifications/templates/{{TEMPLATE_ID}}
Authorization: Bearer {{TOKEN_ADMIN4}}
```
**Expected Code:** 200 or 204 (NOT 500)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

## Phase 8 — Gateway & Infrastructure

---

### UC-149 — Gateway health endpoint

**Precondition:** Gateway running on port 8080  
**Actor:** Anonymous

**Request:**
```
GET http://localhost:8080/actuator/health
(No Authorization)
```
**Expected Code:** 200  
**Expected Body:** `{"status":"UP"}`

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-150 — Gateway dashboard

**Precondition:** testadmin2 logged in  
**Actor:** testadmin2

**Request:**
```
GET http://localhost:8080/dashboard
Authorization: Bearer {{TOKEN_ADMIN2}}
```
**Expected Code:** Any except 500 (404 is acceptable)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-151 — Gateway metrics

**Precondition:** testadmin2 logged in  
**Actor:** testadmin2

**Request:**
```
GET http://localhost:8080/metrics
Authorization: Bearer {{TOKEN_ADMIN2}}
```
**Expected Code:** Any except 500 (404 is acceptable)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-152 — Swagger UI accessible

**Precondition:** Gateway running  
**Actor:** Browser or curl

**Request:**
```
GET http://localhost:8080/swagger-ui.html
(No Authorization)
```
**Expected Code:** 200 or 302 (redirect to swagger-ui/index.html)

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-153 — OpenAPI spec accessible

**Precondition:** Gateway running  
**Actor:** Anonymous

**Request:**
```
GET http://localhost:8080/v3/api-docs
(No Authorization)
```
**Expected Code:** 200  
**Expected Body:** JSON OpenAPI specification

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### UC-154 — All 7 backend services healthy

**Precondition:** All services started  
**Actor:** Anonymous

Check each service's health endpoint:

| Port | Service | URL | Expected |
|------|---------|-----|---------|
| 8081 | Auth | `http://localhost:8081/actuator/health` | 200 |
| 8082 | User | `http://localhost:8082/actuator/health` | 200 |
| 8083 | Ticket | `http://localhost:8083/actuator/health` | 200 |
| 8084 | Solution | `http://localhost:8084/actuator/health` | 200 |
| 8085 | Knowledge | `http://localhost:8085/actuator/health` | 200 |
| 8086 | Reward | `http://localhost:8086/actuator/health` | 200 |
| 8087 | Notification | `http://localhost:8087/actuator/health` | 200 |

**Result:** [ ] PASS  [ ] FAIL  Notes: Failed ports: ___________

---

## Phase 9 — End-to-End Flows

### Flow A — Full USER Journey

**Goal:** User creates ticket → writes solution → gets it approved → earns points.  
**Actor:** testuser1 (or any fresh user with no prior actions)

**Steps:**

1. **Create ticket:**
```
POST http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{
  "title": "Flow A: Network Latency Issue",
  "description": "High latency on internal network during peak hours",
  "priority": "HIGH",
  "categoryId": 1,
  "difficultyLevel": "MEDIUM",
  "visibility": "ORGANIZATION"
}
```
Save `ticketId` → `FLOW_A_TICKET_ID`

2. **Create solution:**
```
POST http://localhost:8080/api/solutions
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{
  "title": "Flow A: QoS Configuration Fix",
  "solutionContent": "Implement QoS rules on the core switch to prioritize critical traffic",
  "ticketId": "{{FLOW_A_TICKET_ID}}"
}
```
Save `data.solutionId` → `FLOW_A_SOL_ID`

3. **Submit for review:**
```
PATCH http://localhost:8080/api/solutions/{{FLOW_A_SOL_ID}}/submit
Authorization: Bearer {{TOKEN_USER1}}
Content-Type: application/json

{}
```

4. **testuser2 upvotes:**
```
POST http://localhost:8080/api/solutions/{{FLOW_A_SOL_ID}}/votes
Authorization: Bearer {{TOKEN_USER2}}
Content-Type: application/json

{ "voteType": "UPVOTE" }
```

5. **testadmin1 approves:**
```
PUT http://localhost:8080/api/solutions/{{FLOW_A_SOL_ID}}/approve
Authorization: Bearer {{TOKEN_ADMIN1}}
Content-Type: application/json

{}
```

6. **Wait 5 seconds, then check points:**
```
GET http://localhost:8080/api/rewards/users/{{AUTH_ID_USER1}}/points
Authorization: Bearer {{TOKEN_USER1}}
```
**Expected:** `totalPoints > 0`

**Result:** [ ] PASS  [ ] FAIL  Final points: ___________

---

### Flow B — Full ADMIN Journey

**Goal:** Admin creates ticket → moves through statuses → resolves → triggers leaderboard.  
**Actor:** testadmin2

**Steps:**

1. **Create ticket:**
```
POST http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_ADMIN2}}
Content-Type: application/json

{
  "title": "Flow B: Disk Space Alert on DB Server",
  "description": "DB server at 95% disk capacity",
  "priority": "MEDIUM",
  "categoryId": 2,
  "difficultyLevel": "EASY",
  "visibility": "ORGANIZATION"
}
```
Save `ticketId` → `FLOW_B_TICKET_ID`

2. **Set IN_PROGRESS:**
```
PUT http://localhost:8080/api/tickets/status
Authorization: Bearer {{TOKEN_ADMIN2}}
Content-Type: application/json

{ "ticketId": "{{FLOW_B_TICKET_ID}}", "status": "IN_PROGRESS" }
```

3. **Set RESOLVED:**
```
PUT http://localhost:8080/api/tickets/status
Authorization: Bearer {{TOKEN_ADMIN2}}
Content-Type: application/json

{ "ticketId": "{{FLOW_B_TICKET_ID}}", "status": "RESOLVED" }
```

4. **Wait 5 seconds, then generate leaderboard:**
```
POST http://localhost:8080/api/rewards/leaderboard/generate
Authorization: Bearer {{TOKEN_ADMIN2}}
Content-Type: application/json

{}
```

**Expected:** Each step responds with non-500 codes. Leaderboard generation returns non-500.

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### Flow C — Knowledge Base Chain

**Goal:** Approve a solution → KB auto-created → published → rated.

**Steps:**

1. **Create and submit a solution (testuser3):**
```
POST http://localhost:8080/api/tickets
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{
  "title": "Flow C: SSL Handshake Failure",
  "description": "Clients cannot connect due to SSL handshake error",
  "priority": "URGENT",
  "categoryId": 4,
  "difficultyLevel": "HARD",
  "visibility": "ORGANIZATION"
}
```
Save `ticketId` → `FLOW_C_TICKET_ID`

```
POST http://localhost:8080/api/solutions
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{
  "title": "Flow C: Renew SSL Certificate",
  "solutionContent": "Use certbot to renew: certbot renew --force-renewal. Restart nginx after.",
  "ticketId": "{{FLOW_C_TICKET_ID}}"
}
```
Save `data.solutionId` → `FLOW_C_SOL_ID`

```
PATCH http://localhost:8080/api/solutions/{{FLOW_C_SOL_ID}}/submit
Authorization: Bearer {{TOKEN_USER3}}
Content-Type: application/json

{}
```

2. **testmgr3 approves:**
```
PUT http://localhost:8080/api/solutions/{{FLOW_C_SOL_ID}}/approve
Authorization: Bearer {{TOKEN_MGR3}}
Content-Type: application/json

{}
```

3. **Wait 8 seconds, then find KB article:**
```
GET http://localhost:8080/api/knowledge/by-solution/{{FLOW_C_SOL_ID}}
Authorization: Bearer {{TOKEN_USER3}}
```
Save `data.id` → `FLOW_C_KB_ID`

4. **testadmin3 publishes the KB article:**
```
PUT http://localhost:8080/api/knowledge/{{FLOW_C_KB_ID}}/publish
Authorization: Bearer {{TOKEN_ADMIN3}}
```

5. **testuser2 rates the article (direct to knowledge-service):**
```
POST http://localhost:8085/api/ratings
Authorization: Bearer {{TOKEN_USER2}}
X-User-Id: {{AUTH_ID_USER2}}
Content-Type: application/json

{ "articleId": "{{FLOW_C_KB_ID}}", "rating": 5, "feedback": "Excellent guide!" }
```

**Expected:** All steps return non-500. KB article found after approval.

**Result:** [ ] PASS  [ ] FAIL  KB article found: ___________

---

### Flow D — Rejection + Resubmit (Verified in Phase 4)

This flow is exercised through UC-77 (reject), UC-79 (resubmit), and UC-80 (re-approve).

Confirm the following occurred:
- [ ] Solution 2 was rejected by testmgr2 with reason
- [ ] Solution 2 was resubmitted by testuser2
- [ ] Solution 2 was re-approved by testmgr4

**Result:** [ ] PASS  [ ] FAIL  Notes: ___________

---

### Flow E — Multi-Contributor Rewards (Verified in Phase 4 + UC-126)

Confirm the following occurred:
- [ ] Solution 3 was created by testuser3
- [ ] testuser4 was added as a contributor to Solution 3
- [ ] testmgr3 approved Solution 3
- [ ] Both testuser3 and testuser4 received reward points (UC-126)

**Result:** [ ] PASS  [ ] FAIL  user3 pts: ___  user4 pts: ___

---

## Phase 10 — Security Sweep (Flow G)

---

### SEC-01 — USER cannot approve solution

**Actor:** testuser4 (USER role)

**Request:**
```
PUT http://localhost:8080/api/solutions/00000000-0000-0000-0000-000000000099/approve
Authorization: Bearer {{TOKEN_USER4}}
Content-Type: application/json

{}
```
**Expected Code:** 403 or 401 or 404 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

### SEC-02 — USER cannot reject solution

**Actor:** testuser4

**Request:**
```
PUT http://localhost:8080/api/solutions/00000000-0000-0000-0000-000000000099/reject
Authorization: Bearer {{TOKEN_USER4}}
Content-Type: application/json

{ "reason": "test" }
```
**Expected Code:** 403 or 401 or 404 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

### SEC-03 — USER cannot list all users

**Actor:** testuser4

**Request:**
```
GET http://localhost:8080/api/users
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 403 or 401 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

### SEC-04 — USER cannot lock a user account

**Actor:** testuser4

**Request:**
```
PUT http://localhost:8080/api/auth/users/00000000-0000-0000-0000-000000000099/lock
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 403 or 401 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

### SEC-05 — USER cannot manually award points

**Actor:** testuser4

**Request:**
```
POST http://localhost:8080/api/rewards/points
Authorization: Bearer {{TOKEN_USER4}}
Content-Type: application/json

{
  "userId": "00000000-0000-0000-0000-000000000099",
  "points": 1000,
  "reason": "Unauthorized self-award"
}
```
**Expected Code:** 403 or 401 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

### SEC-06 — USER cannot delete another user's ticket

**Actor:** testuser4

**Request:**
```
DELETE http://localhost:8080/api/tickets/00000000-0000-0000-0000-000000000099
Authorization: Bearer {{TOKEN_USER4}}
```
**Expected Code:** 403 or 401 or 404 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

### SEC-07 — USER cannot create system notifications

**Actor:** testuser4

**Request:**
```
POST http://localhost:8080/api/notifications
Authorization: Bearer {{TOKEN_USER4}}
Content-Type: application/json

{
  "title": "Hack Attempt",
  "message": "This should be blocked",
  "type": "SYSTEM_ALERT",
  "referenceId": "00000000-0000-0000-0000-000000000099",
  "referenceType": "TICKET",
  "recipientUserIds": ["00000000-0000-0000-0000-000000000099"]
}
```
**Expected Code:** 403 or 401 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

### SEC-08 — MANAGER cannot list all users

**Actor:** testmgr1

**Request:**
```
GET http://localhost:8080/api/users
Authorization: Bearer {{TOKEN_MGR1}}
```
**Expected Code:** 403 or 401 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

### SEC-09 — MANAGER cannot lock a user account

**Actor:** testmgr1

**Request:**
```
PUT http://localhost:8080/api/auth/users/00000000-0000-0000-0000-000000000099/lock
Authorization: Bearer {{TOKEN_MGR1}}
```
**Expected Code:** 403 or 401 (NOT 200)

**Result:** [ ] PASS  [ ] FAIL  Actual code: ___________

---

## Summary Scorecard

Fill this in after completing all phases:

| Phase | Tests | PASS | FAIL | SKIP | Notes |
|-------|-------|------|------|------|-------|
| Phase 0 — Setup | 12 | | | | Setup steps (no pass/fail checkboxes) |
| Phase 1 — Auth | 18 | | | 0 | UC-18 now testable via test profile |
| Phase 2 — Profiles | 5 | | | | |
| Phase 3 — Tickets | 24 | | | | |
| Phase 4 — Solutions | 36 | | | 0 | UC-69 (JSON body), UC-71 (delete) restored |
| Phase 5 — Knowledge Base | 27 | | | 0 | UC-89 moved to cleanup, UC-94 restored, UC-89 cleanup added |
| Phase 6 — Rewards | 17 | | | | |
| Phase 7 — Notifications | 24 | | | 1 | UC-141 permanent skip (no retry endpoint); UC-142/143/144 restored |
| Phase 8 — Infrastructure | 6 | | | | |
| Phase 9 — E2E Flows | 5 | | | | |
| Phase 10 — Security | 9 | | | | |
| **TOTAL** | **171** | | | **1** | Only UC-141 permanently skipped |

---

## Important Notes

### Kafka Event Timing
After any of these actions, **wait 4–8 seconds** before checking for side effects:
- Ticket assignment (→ notification)
- Solution approval (→ points, KB article, notifications)
- Solution rejection (→ notification)
- Ticket resolution (→ points)

### Direct Service Calls (Not via Gateway)
Some endpoints are only accessible directly on service ports:

| Endpoint | Direct URL | Reason |
|----------|-----------|--------|
| Departments | `http://localhost:8082/api/departments` | Not routed via gateway |
| Ratings | `http://localhost:8085/api/ratings` | Not routed via gateway |
| Categories | `http://localhost:8085/api/categories` | Not routed via gateway |
| Tags | `http://localhost:8085/api/tags` | Not routed via gateway |
| Notification templates | `http://localhost:8087/api/v1/notifications/templates` | Not routed via gateway |
| Broadcast | `http://localhost:8087/internal/notifications/broadcast` | Internal endpoint |

### Required Headers for Direct Knowledge Service Calls
When calling knowledge-service (port 8085) directly, include:
```
Authorization: Bearer {{TOKEN}}
X-User-Id: {{AUTH_ID}}
X-User-Role: ROLE_USER   (or ROLE_MANAGER or ROLE_ADMIN)
```

### Solution Body Fields
- Create solution body uses `solutionContent` (not `description`)
- Rejection body uses `reason` (not `rejectionReason`)
- Ticket assignment body uses `assignedTo` (not `assignedUserId`)
- Add contributor body requires both `solutionId` AND `userId` fields

---

## Appendix K — Kubernetes Mode Testing

All 162 test assertions in this guide apply equally to k8s mode. The only differences are:

### Base URL
Replace `http://localhost:8080` with `http://ticketing.local` in every curl command.

### Credentials
Use k8s-specific accounts (created by `./services.sh k8s-seed`):

| Role | Email | Password |
|---|---|---|
| Admin | k8sadmin@demo.test | Demo@1234 |
| Engineer | k8sengineer@demo.test | Demo@1234 |

### Pre-test checklist
```bash
# 1. Confirm cluster is up and all pods running
./services.sh k8s-status

# 2. Confirm /etc/hosts has the ticketing.local entry
grep "ticketing.local" /etc/hosts

# 3. Confirm seed data exists
curl -s http://ticketing.local/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"k8sadmin@demo.test","password":"Demo@1234"}' | python3 -m json.tool

# 4. Get a token for testing
TOKEN=$(curl -s -H "Content-Type: application/json" \
  -X POST http://ticketing.local/api/auth/login \
  -d '{"email":"k8sadmin@demo.test","password":"Demo@1234"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['accessToken'])")
echo $TOKEN
```

### Phase 0 equivalent — infrastructure check
```bash
# All 17 pods should show 1/1 READY
kubectl get pods -n ticketing-system

# Ingress rule should show ticketing.local
kubectl get ingress -n ticketing-system

# Frontend should return HTTP 200
curl -s -o /dev/null -w "%{http_code}" http://ticketing.local/
```

### Phase 1–10 — identical steps, different base URL
Replace `localhost:8080` → `ticketing.local` in every request. All expected responses, status codes, and Kafka event timings are identical to local mode.

### Observability during k8s testing
```bash
# Tail logs for the service under test
./services.sh k8s-logs ticket-service
./services.sh k8s-logs solution-service
./services.sh k8s-logs reward-service
./services.sh k8s-logs notification-service

# Check Kafka topics inside the k8s cluster
kubectl exec -n ticketing-system deploy/kafka -- /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list

# Check consumer group lag
kubectl exec -n ticketing-system deploy/kafka -- /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 --list
```
