# Manual Test Cases — Ticketing System

**Base URL:** `http://localhost:3000`  
**Credentials:** All demo accounts use password `Passw0rd!`  
**Demo accounts:** `user1@demo.local` (USER), `admin1@demo.local` (ADMIN), `user2@demo.local` (USER)

---

## Frontend Bug Fixes Applied (April 2026)

The following bugs were identified during integration testing and corrected before final demo:

| # | File(s) | Bug Description | Fix Applied |
|---|---|---|---|
| 1 | `authSlice.js` | `decoded.userId` returned `0` (numeric) instead of the real UUID | Switched to `decoded.authUserId` throughout |
| 2 | `ticketSlice.js` | All ticket thunks unwrapped `response.data.data` (ticket-service returns flat, no ApiResponse wrapper) | Changed to `response.data` |
| 3 | `CreateTicketPage.jsx` | `CRITICAL` priority and string `category` field sent to backend | Fixed enums (LOW/MEDIUM/HIGH/URGENT) and `categoryId` number field |
| 4 | `ticketApi.js` | `updateStatus` sent status as query param instead of request body | Fixed to `{ ticketId, status }` JSON body |
| 5 | `TicketDetailPage.jsx` | `isAdmin` used `===` (always false for `"ROLE_ADMIN"`); title and status-change buttons missing | Fixed to `.includes('ADMIN')`; added title + admin buttons |
| 6 | `DashboardPage / Navbar / NotificationsPage` | Used `user.userId` (0), `notification.id` (undefined), `notification.ticketId` (undefined) | → `user.authUserId`, `notification.notificationId`, `notification.referenceId` |
| 7 | `notificationSlice.js` | `markAsRead` reducer looked up by `n.id` (always fails) | → `n.notificationId` |
| 8 | `SolutionDetailPage.jsx` | `isAdmin` used strict `===`; rejected section checked non-existent `rejectedAt` | Fixed `.includes()`; simplified rejected display |
| 9 | `KnowledgeArticlePage.jsx` | `article.category` rendered `[object Object]`; `tag` rendered `[object Object]`; used `relatedTicketId` (undefined) | → `category.categoryName`, `tag.tagName`, `article.ticketId` |
| 10 | `SolutionsPage / KnowledgeBasePage` | Tags and submitter field names wrong (`submittedBy`, raw `tag`) | → `tag.tagName`, `solution.createdBy` fallback |

---

## Setup Checklist (Before Testing)

- [ ] All 8 backend services healthy: `for p in 8080 8081 8082 8083 8084 8085 8086 8087; do curl -s http://localhost:$p/actuator/health | grep -q UP && echo "$p UP" || echo "$p DOWN"; done`
- [ ] Frontend running at `http://localhost:3000`
- [ ] Kafka + Postgres containers running: `docker ps`
- [ ] Demo users seeded: `./services.sh seed`
- [ ] Open browser console (F12 → Console) and Network tab — watch for red errors throughout testing

---

## Module 1 — Authentication

### TC-01: Successful login as USER
**Actor:** user1  
**Steps:**
1. Navigate to `http://localhost:3000`
2. Should redirect to `/login`
3. Enter email: `user1@demo.local`, password: `Passw0rd!`
4. Click **Login**

**Expected:**
- Redirects to `/dashboard`
- Navbar shows `(ROLE_USER)` and **Logout** button
- No console errors, no 4xx/5xx in Network tab

---

### TC-02: Successful login as ADMIN
**Actor:** admin1  
**Steps:**
1. Navigate to `http://localhost:3000/login`
2. Enter email: `admin1@demo.local`, password: `Passw0rd!`
3. Click **Login**

**Expected:**
- Redirects to `/dashboard`
- Navbar shows `(ROLE_ADMIN)` and **Logout** button

---

### TC-03: Login with wrong password
**Actor:** anyone  
**Steps:**
1. Navigate to `/login`
2. Enter email: `user1@demo.local`, password: `wrongpassword`
3. Click **Login**

**Expected:**
- Stays on login page
- Error message displayed ("Invalid email or password" or similar)
- No redirect to dashboard

---

### TC-04: Logout
**Actor:** user1 (logged in)  
**Steps:**
1. Log in as user1 (TC-01)
2. Click **Logout** in navbar

**Expected:**
- Redirects to `/login`
- `localStorage` cleared (token, user gone)
- Attempting to navigate to `/dashboard` redirects back to `/login`

---

### TC-05: Access protected route while unauthenticated
**Actor:** unauthenticated  
**Steps:**
1. Ensure logged out
2. Navigate directly to `http://localhost:3000/tickets`

**Expected:**
- Redirects to `/login` (route guard active)

---

## Module 2 — Tickets

### TC-06: Create a ticket (USER)
**Actor:** user1  
**Steps:**
1. Log in as user1
2. Click **Tickets** in navbar → click **Create Ticket**
3. Fill in:
   - Title: `Cannot connect to VPN`
   - Description: `Getting authentication error from home network`
   - Priority: `High`
   - Difficulty: `Medium`
   - Category: `Network`
   - Visibility: `Organization`
4. Click **Create Ticket**

**Expected:**
- Success toast: "Ticket created successfully!"
- Redirected to `/tickets`
- New ticket appears in the list with status `OPEN`
- No 4xx/5xx in Network tab (POST /api/tickets returns 201)

---

### TC-07: Create ticket with all priorities
**Actor:** user1  
**Steps:**
1. Repeat TC-06 four times, selecting Priority: `Low`, `Medium`, `High`, `Urgent` in turn

**Expected:**
- All four tickets created successfully
- Each appears in ticket list with correct priority displayed

---

### TC-08: Create ticket with all categories
**Actor:** user1  
**Steps:**
1. Create tickets selecting each category: `Network`, `Software`, `Hardware`, `Security`, `Other`

**Expected:**
- All five tickets created without errors

---

### TC-09: View ticket list
**Actor:** user1  
**Steps:**
1. Log in as user1
2. Click **Tickets** in navbar

**Expected:**
- List displays all tickets
- Each ticket shows title, status badge (OPEN/IN_PROGRESS/RESOLVED), difficulty badge
- Created timestamp visible
- No loading spinner stuck indefinitely

---

### TC-10: View ticket detail
**Actor:** user1  
**Steps:**
1. From ticket list, click on any ticket

**Expected:**
- Ticket detail page loads
- Shows title, description, priority, difficulty, created by, created date
- "Submit Solution" button visible
- No approve/reject/status-change buttons visible (USER role)

---

### TC-11: Admin views all tickets
**Actor:** admin1  
**Steps:**
1. Log in as admin1
2. Click **Tickets**

**Expected:**
- All tickets from all users appear (not just admin1's own)
- Status change buttons visible on ticket detail

---

### TC-12: Admin changes ticket status — IN_PROGRESS
**Actor:** admin1  
**Steps:**
1. Log in as admin1
2. Open any ticket with status `OPEN`
3. Click **→ In Progress**

**Expected:**
- Success toast: "Ticket moved to IN_PROGRESS"
- Status badge on the page updates to `IN_PROGRESS` (may require page reload)

---

### TC-13: Admin changes ticket status — RESOLVED
**Actor:** admin1  
**Steps:**
1. From an `IN_PROGRESS` ticket, click **→ Resolved**

**Expected:**
- Success toast: "Ticket moved to RESOLVED"
- Status badge updates to `RESOLVED`

---

### TC-14: Search tickets by title
**Actor:** user1  
**Steps:**
1. Navigate to `http://localhost:8080/api/tickets/search?title=VPN` (gateway direct or via Network tab observation)

**Expected:**
- Only tickets with "VPN" in title returned

---

## Module 3 — Solutions

### TC-15: Submit a solution to a ticket
**Actor:** admin1 (or any logged-in user)  
**Steps:**
1. Log in as admin1
2. Open any `OPEN` or `IN_PROGRESS` ticket
3. Click **Submit Solution**
4. Fill in:
   - Solution Title: `Reset MFA and re-authenticate`
   - Solution Description: `1. Sign out of VPN client 2. Re-pair authenticator app 3. Sign in again`
5. Click **Submit Solution**

**Expected:**
- Success toast: "Solution submitted successfully!"
- Solution appears in the solutions list below with status `PENDING`
- No 4xx/5xx (POST /api/solutions returns 201)

---

### TC-16: Submit solution without title (validation)
**Actor:** user1  
**Steps:**
1. Open a ticket → click Submit Solution
2. Leave **Solution Title** empty
3. Fill in description
4. Click **Submit Solution**

**Expected:**
- Form validation prevents submission (browser required field warning or toast error)
- No API call made / 400 returned if submitted

---

### TC-17: Admin approves a solution
**Actor:** admin1  
**Steps:**
1. Log in as admin1
2. Open a ticket that has a `PENDING` solution
3. Click **Approve** next to the solution

**Expected:**
- Success toast: "Solution approved! Knowledge article will be auto-created and rewards distributed."
- Solution status changes to `APPROVED`
- Within ~20 seconds: a Knowledge Base article titled "Solution for Ticket #..." auto-appears (Kafka event chain)

---

### TC-18: Admin rejects a solution
**Actor:** admin1  
**Steps:**
1. Open a ticket with a `PENDING` solution
2. Click **Reject**

**Expected:**
- Toast: "Solution rejected"
- Solution status changes to `REJECTED`
- No approve/reject buttons appear for that solution anymore

---

### TC-19: USER cannot see approve/reject buttons
**Actor:** user1  
**Steps:**
1. Log in as user1
2. Open a ticket with a `PENDING` solution

**Expected:**
- No **Approve** or **Reject** buttons visible
- No status-change buttons visible

---

### TC-20: View all solutions (Solutions page)
**Actor:** user1  
**Steps:**
1. Click **Solutions** in navbar (if visible) or navigate to `/solutions`

**Expected:**
- All solutions listed with status badges
- Filter tabs (ALL / PENDING / APPROVED / REJECTED) work correctly

---

## Module 4 — Knowledge Base

### TC-21: View knowledge base articles
**Actor:** user1  
**Steps:**
1. Log in as user1
2. Click **Knowledge Base** in navbar

**Expected:**
- Articles listed (including auto-created ones from approved solutions)
- Each article shows title, status (PUBLISHED/DRAFT), tags, created date
- "No articles found" empty state if none exist

---

### TC-22: Search knowledge base
**Actor:** user1  
**Steps:**
1. Navigate to Knowledge Base
2. Type `VPN` in the search box
3. Press Enter or click **Search**

**Expected:**
- Only articles containing "VPN" in title or content appear
- Empty state if no matches

---

### TC-23: View article detail
**Actor:** user1  
**Steps:**
1. From Knowledge Base, click on any article

**Expected:**
- Article detail page loads: title, status badge, version, content, tags (each tag shows `tagName` string, not `[object Object]`)
- Category displays as a name string (not `[object Object]`)
- "Rate this Article" section at the bottom with 5 star buttons
- "View Related Ticket" link appears only if `article.ticketId` is set (uses `ticketId` field, not `relatedTicketId`)

---

### TC-24: Admin creates a KB article manually
**Actor:** admin1  
**Steps:**
1. Log in as admin1
2. Navigate to Knowledge Base
3. Click **+ New Article**
4. Fill in:
   - Title: `How to fix VPN authentication errors`
   - Content: `Step 1: Re-pair authenticator. Step 2: Clear VPN cache. Step 3: Reconnect.`
   - Tags: `vpn, authentication, network`
   - Visibility: `Public`
5. Click **Create Article**

**Expected:**
- Toast: "Knowledge article created!"
- New article appears in the list

---

### TC-25: USER cannot see "+ New Article" button
**Actor:** user1  
**Steps:**
1. Log in as user1
2. Navigate to Knowledge Base

**Expected:**
- No **+ New Article** button visible

---

### TC-26: Rate an article
**Actor:** user2  
**Steps:**
1. Log in as `user2@demo.local` / `Passw0rd!`
2. Navigate to Knowledge Base → open any PUBLISHED article
3. Click 4 stars in the "Rate this Article" section
4. Type feedback: `Very helpful, solved my issue`
5. Click **Submit Rating**

**Expected:**
- Toast: "Rating submitted!"
- Average rating updates (shows e.g. `4.0 average`)

---

### TC-27: Rate same article twice (upsert)
**Actor:** user2  
**Steps:**
1. As user2, open an article already rated in TC-26
2. Click 5 stars → click **Submit Rating**

**Expected:**
- Toast: "Rating submitted!" (upsert, not duplicate error)
- Average rating updates to reflect the new value

---

### TC-28: Auto-created KB article from Kafka (end-to-end event chain)
**Actor:** admin1  
**Steps:**
1. Create a new ticket (as user1)
2. Log in as admin1, open the ticket, submit a solution
3. Approve the solution
4. Wait up to 20 seconds
5. Navigate to Knowledge Base → search for "Solution for Ticket"

**Expected:**
- An article titled `Solution for Ticket #<ticketId>` appears automatically
- This proves: solution-service publishes `solution.approved` → Kafka → knowledge-service consumes → article created

---

## Module 5 — Rewards & Leaderboard

### TC-29: View leaderboard
**Actor:** user1  
**Steps:**
1. Log in as user1
2. Click **Leaderboard** in navbar

**Expected:**
- Leaderboard table displayed with ranked entries
- Each entry shows userId, points, rank
- Top 3 entries have gold/silver/bronze styling
- Empty state if no points awarded yet

---

### TC-30: Admin awards points manually
**Actor:** admin1  
**Steps:**
1. Log in as admin1
2. Open any ticket
3. Click **Award Points**
4. Set Points: `50`, Reason: `Outstanding contribution`
5. Click **Award**

**Expected:**
- Toast: "50 points awarded!"
- Reward form closes

---

### TC-31: Points reflected after solution approval (auto-reward)
**Actor:** user1 (observe)  
**Steps:**
1. Complete TC-28 (solution approved)
2. Navigate to Leaderboard

**Expected:**
- The user who submitted the solution appears on the leaderboard with non-zero points
- Points match what the reward service awarded automatically

---

## Module 6 — Notifications

### TC-32: User receives notifications
**Actor:** user1  
**Steps:**
1. Log in as user1
2. Click the **bell icon** in the navbar or navigate to notifications
3. Alternatively navigate to `/notifications`

**Expected:**
- Notification list loads (may be empty if no events triggered yet)
- If any solution was approved for user1's tickets: notification "Solution Approved" appears
- No 4xx/5xx (GET /api/notifications/users/`{authUserId}` returns 200 — uses `user.authUserId`, not `user.userId`)

---

### TC-33: Notification delivered after solution approval (Kafka chain)
**Actor:** user1  
**Steps:**
1. Log in as user1, create a ticket
2. Log out, log in as admin1
3. Submit and approve a solution for user1's ticket
4. Log out, log in as user1
5. Check notifications

**Expected:**
- Within 20–30 seconds of approval: a notification about "Solution Approved" appears in user1's notifications list
- Proves the Kafka chain: solution-service → Kafka → notification-service → DB

---

### TC-34: Notification bell badge
**Actor:** user1  
**Steps:**
1. After TC-33, check the bell icon in the navbar

**Expected:**
- Bell shows an unread count badge (e.g. red circle with number)
- Clicking a notification navigates to the related ticket using `notification.referenceId` (not `ticketId`)
- Marking as read uses `notification.notificationId` (UUID) as the path variable

---

## Module 7 — Multi-User Validation

### TC-35: Multiple users can each create tickets independently
**Actor:** user1, user2, user3  
**Steps:**
1. Log in as `user1@demo.local` → create a ticket → log out
2. Log in as `user2@demo.local` → create a ticket → log out
3. Log in as `user3@demo.local` → create a ticket → log out
4. Log in as `admin1@demo.local` → view ticket list

**Expected:**
- Admin sees all three tickets from three different users
- Each ticket attributed to the correct `createdBy` email

---

### TC-36: User cannot see another user's PRIVATE visibility tickets
**Actor:** user1, user2  
**Steps:**
1. (Backend test via curl) Create a ticket as user1 with `visibility: PRIVATE`
2. Log in as user2 → view ticket list

**Expected:**
- user2 does not see user1's private ticket

---

## Module 8 — Full End-to-End Journey

### TC-37: Complete E2E — USER journey
**Actor:** user1  
**Steps:**
1. Login as user1
2. Create a ticket (Priority: HIGH, Category: Network)
3. Verify ticket appears in ticket list with status OPEN
4. Click into the ticket → submit a solution (add title + description)
5. Check notifications (may be empty at this point)
6. Check Leaderboard (user1 may not appear yet until solution approved)

**Expected:** All steps complete without errors

---

### TC-38: Complete E2E — ADMIN journey
**Actor:** admin1  
**Steps:**
1. Login as admin1
2. View ticket list → open a ticket from user1
3. Change status to `IN_PROGRESS`
4. Submit a solution
5. Approve the solution
6. Change status to `RESOLVED`
7. Award 25 points to yourself (using Award Points button)
8. View leaderboard — verify points appear

**Expected:** All steps complete without errors. After solution approval, KB article auto-created within 20s.

---

### TC-39: Complete E2E — KNOWLEDGE journey
**Actor:** admin1 + user2  
**Steps:**
1. Login as admin1 → Knowledge Base → create an article (Title: `VPN Troubleshooting Guide`, tags: `vpn,network`)
2. Search for "VPN" → article appears
3. Click article → verify content renders correctly
4. Logout → login as user2
5. Navigate to Knowledge Base → search "VPN" → article visible
6. Open article → rate it 5 stars + feedback "Excellent guide"
7. Verify "Rating submitted!" toast and average rating updates

**Expected:** All steps complete without errors

---

## Module 9 — Negative / Edge Cases

### TC-40: Empty title on ticket creation
**Actor:** user1  
**Steps:**
1. Navigate to Create Ticket
2. Leave Title empty, fill other fields
3. Click Create Ticket

**Expected:**
- Browser validation prevents form submission (HTML required field)

---

### TC-41: Submit solution without solution content
**Actor:** user1  
**Steps:**
1. Open a ticket → click Submit Solution
2. Fill title but leave description empty
3. Click Submit Solution

**Expected:**
- HTML required field validation blocks submission

---

### TC-42: Rate article with no star selected
**Actor:** user1  
**Steps:**
1. Open an article → go to Rate section
2. Leave rating at 0 stars (no star clicked)
3. Click Submit Rating

**Expected:**
- Toast error: "Please select a star rating"
- No API call made

---

### TC-43: Expired token handling
**Actor:** user1  
**Steps:**
1. Login as user1
2. Wait 1 hour (or manually clear/expire token in localStorage)
3. Attempt any API action (e.g. create a ticket)

**Expected:**
- 401 detected by axios interceptor
- Automatically redirected to `/login`
- localStorage cleared

---

### TC-44: Access admin action as USER
**Actor:** user1  
**Steps:**
1. Login as user1
2. Open a ticket with a PENDING solution
3. Verify no Approve/Reject buttons visible
4. Directly call via Network tab or curl: `PUT /api/solutions/{id}/approve` with user1's token

**Expected:**
- UI: buttons not shown
- API direct: 403 Forbidden returned

---

### TC-45: Creating ticket with invalid priority (direct API)
**Actor:** user1  
**Steps:**
1. Via browser Network tab, replay a POST /api/tickets request with `priority: "INVALID"`

**Expected:**
- Backend returns 400/500 with message "Invalid Priority: INVALID"
- Frontend shows toast error (not stuck in "Creating..." state)

---

---

## Bug Fixes Applied (May 2026)

The following backend and frontend bugs were found during end-to-end integration testing and fixed:

| # | Component | Bug | Fix |
|---|---|---|---|
| 1 | `TicketRepository.java` | JPQL `LOWER(CONCAT('%', :title, '%'))` with a null parameter caused PostgreSQL to infer the bind parameter type as `bytea` → `ERROR: function lower(bytea) does not exist` | Rewrote query as native SQL with explicit `CAST(:param AS text)` for every nullable parameter; added matching `countQuery` |
| 2 | `TicketListPage.jsx` | `priorityFilter` state was missing from the `useEffect` dependency array so changing the Priority dropdown never triggered a data reload | Added `priorityFilter` to the `useEffect` dependency array |
| 3 | `Notification.java` (entity) | `referenceId` and `referenceType` columns were annotated `nullable = false` but the `createAndSend()` helper never populates them → `PSQLException: null value in column "reference_id" violates not-null constraint` | Removed `nullable = false` from both `@Column` annotations; dropped the NOT NULL constraint from the live DB (`ALTER TABLE notifications ALTER COLUMN … DROP NOT NULL`) |
| 4 | `Reward_service` Kafka | `consumeSolutionApproved` declared `SolutionApprovedEvent` (common-library) as parameter type, but the global `spring.json.value.default.type: RewardEventDTO` forced all messages to deserialise as `RewardEventDTO`; Spring then failed to convert the instance to `SolutionApprovedEvent` | Created local `SolutionApprovalEvent` DTO matching the solution-service event shape; added dedicated `solutionApprovedContainerFactory` bean with `VALUE_DEFAULT_TYPE: SolutionApprovalEvent`; updated consumer to use that factory |
| 5 | `Reward_service` Kafka | `RewardEventDTO.timestamp` (type `Long`) failed to deserialise when a `SolutionEvent.timestamp` arrived as a Jackson `LocalDateTime` array `[2026, 5, 3, …]` | Added `@JsonIgnore` on `RewardEventDTO.timestamp`; added `@JsonIgnoreProperties(ignoreUnknown = true)` on the class |
| 6 | `Notification_service` Kafka | `consumeRewardPointsAdded` declared `RewardPointsAddedEvent` as parameter but the default `kafkaListenerContainerFactory` used `USE_TYPE_INFO_HEADERS: true` with fallback `NotificationEvent`; reward service publishes without type headers (`ADD_TYPE_INFO_HEADERS: false`) → consumer always got a `NotificationEvent` and Spring failed to convert it | Created local `RewardPointsAddedEvent` DTO; added dedicated `rewardPointsContainerFactory` bean with `USE_TYPE_INFO_HEADERS: false` and `VALUE_DEFAULT_TYPE: RewardPointsAddedEvent`; updated consumer to use that factory |

**Verified end-to-end after all fixes:** approving a solution now awards +30 reward points to the contributor AND produces two notifications — "Solution Approved" and "Points Earned: 30 points! Total: 95".

---

## Regression Checks (Run After Every Code Change)

| Check | How |
|---|---|
| All backend services healthy | `for p in 8080..8087; do curl http://localhost:$p/actuator/health; done` |
| Ticket create → list → detail flow | TC-06 + TC-09 + TC-10 |
| Solution submit → approve → KB auto-create | TC-15 + TC-17 + TC-28 |
| Notifications delivered | TC-33 |
| Leaderboard shows after award | TC-30 + TC-29 |
| Login/logout | TC-01 + TC-04 |
| Rating flow | TC-26 |

---

## Test Execution Log Template

| TC ID | Description | Tester | Date | Result | Notes |
|---|---|---|---|---|---|
| TC-01 | Login as USER | | | PASS / FAIL | |
| TC-02 | Login as ADMIN | | | PASS / FAIL | |
| TC-03 | Login wrong password | | | PASS / FAIL | |
| TC-04 | Logout | | | PASS / FAIL | |
| TC-05 | Unauthenticated access | | | PASS / FAIL | |
| TC-06 | Create ticket | | | PASS / FAIL | |
| TC-07 | All priorities | | | PASS / FAIL | |
| TC-08 | All categories | | | PASS / FAIL | |
| TC-09 | Ticket list | | | PASS / FAIL | |
| TC-10 | Ticket detail (user) | | | PASS / FAIL | |
| TC-11 | Admin views all tickets | | | PASS / FAIL | |
| TC-12 | Status → IN_PROGRESS | | | PASS / FAIL | |
| TC-13 | Status → RESOLVED | | | PASS / FAIL | |
| TC-14 | Search tickets | | | PASS / FAIL | |
| TC-15 | Submit solution | | | PASS / FAIL | |
| TC-16 | Solution without title | | | PASS / FAIL | |
| TC-17 | Approve solution | | | PASS / FAIL | |
| TC-18 | Reject solution | | | PASS / FAIL | |
| TC-19 | USER no approve buttons | | | PASS / FAIL | |
| TC-20 | Solutions page | | | PASS / FAIL | |
| TC-21 | KB article list | | | PASS / FAIL | |
| TC-22 | KB search | | | PASS / FAIL | |
| TC-23 | Article detail | | | PASS / FAIL | |
| TC-24 | Admin creates article | | | PASS / FAIL | |
| TC-25 | USER no create button | | | PASS / FAIL | |
| TC-26 | Rate article | | | PASS / FAIL | |
| TC-27 | Rate same article twice | | | PASS / FAIL | |
| TC-28 | Kafka auto-KB article | | | PASS / FAIL | |
| TC-29 | Leaderboard view | | | PASS / FAIL | |
| TC-30 | Admin award points | | | PASS / FAIL | |
| TC-31 | Auto-reward after approval | | | PASS / FAIL | |
| TC-32 | User notifications | | | PASS / FAIL | |
| TC-33 | Kafka notification chain | | | PASS / FAIL | |
| TC-34 | Bell badge unread count | | | PASS / FAIL | |
| TC-35 | Multi-user tickets | | | PASS / FAIL | |
| TC-36 | Private visibility | | | PASS / FAIL | |
| TC-37 | E2E USER journey | | | PASS / FAIL | |
| TC-38 | E2E ADMIN journey | | | PASS / FAIL | |
| TC-39 | E2E KNOWLEDGE journey | | | PASS / FAIL | |
| TC-40 | Empty ticket title | | | PASS / FAIL | |
| TC-41 | Empty solution content | | | PASS / FAIL | |
| TC-42 | Rating without stars | | | PASS / FAIL | |
| TC-43 | Expired token | | | PASS / FAIL | |
| TC-44 | USER accesses admin API | | | PASS / FAIL | |
| TC-45 | Invalid priority value | | | PASS / FAIL | |
