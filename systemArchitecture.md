# systemArchitecture.md — QueueSpace

A practical, lightweight architecture to let people join queues from their phones (App + USSD) and help staff manage the line smoothly.

# (1) Architecture at a glance
[ Mobile App ]            [ USSD Menu ]             [ Staff Web ]
     |                         |                         |
     |  HTTPS / JSON           |  Telco → Webhook       |  HTTPS / JSON
     v                         v                         v
                 ┌─────────────────────────────────────────────┐
                 │            QueueSpace API (Backend)         │
                 │  - Auth & Users     - Queue/Ticket Service  │
                 │  - Branch Service   - Notification Service  │
                 │  - Admin/Staff      - Audit Logs            │
                 └──────────────┬───────────────┬──────────────┘
                                │               │
                          SQL Database      SMS/Push Provider
                        (Postgres/MySQL)     (e.g., Twilio)

Why this is feasible: one API serves three clients (App, USSD, Staff web). USSD calls arrive as telco webhooks to our API. We keep the core logic in the backend so all channels behave the same.

# (2) Frontend clients
**Mobile App (React Native or Flutter)**
Join queue, see live position, receive push alerts.
Simple screens: Select Location → Join Queue → Queue Status.

**USSD**
For basic phones/no data. Flow mirrors the app:
Dial *shortcode#
Choose location
Confirm “Join queue”
Receive ticket number + SMS updates

**Staff Web (very simple)**
Secure login for staff.
Buttons: Call Next, Skip, Pause, Close Queue.
Shows live list of tickets and who is “Up Next”.

# (3) Backend (API) — services
Auth & Users: email/phone OTP now; OAuth later.
Organization/Branch: locations where queues run.
Queue/Ticket Service: create ticket, compute live position, assign next.
Notification Service: send SMS/push when user is near turn.
USSD Controller: receives telco webhooks, maps sessions to users.
Audit Logs: record staff actions (called/skip/close).
Tech suggestion: Node.js (Express/Fastify) or Django/FastAPI. Either fits MVP in 3–4 sprints.

# (4) Data model (MVP)
| Table           | Key fields                                                                                  | Notes                              |
| --------------- | ------------------------------------------------------------------------------------------- | ---------------------------------- |
| `users`         | id, name, phone, email(optional), role(user/staff)                                          | Phone is primary for USSD.         |
| `organizations` | id, name                                                                                    | For multi-branch groups.           |
| `branches`      | id, org_id, name, address, is_open                                                          | A queue runs per branch.           |
| `queues`        | id, branch_id, status(open/paused/closed), current_number                                   | One active queue per branch (MVP). |
| `tickets`       | id, queue_id, user_id, number, status(waiting/called/skipped/completed/no_show), created_at | Core of the product.               |
| `notifications` | id, user_id, ticket_id, channel(sms/push), template, status                                 | Audit what went out.               |
| `ussd_sessions` | id, phone, session_id, last_step, temp_branch                                               | Track users mid-USSD flow.         |
| `audit_logs`    | id, actor_id, action, entity, entity_id, created_at                                         | Who did what, when.                |
Minimal indexes: tickets(queue_id, status, number), users(phone).

# (5) Key user flows
**(5.1) Join Queue (App or USSD)**
User selects Branch and taps Join Queue (or dials USSD and chooses options).
API checks: queue is open, user not already queued.
API assigns next ticket number and stores ticket.
API returns: ticket number + people ahead.
Notification service schedules “you’re close” alert (e.g., when ≤ 3 ahead).

**Edge cases**
Queue paused/closed → show friendly message + come-back time.
User already has an active ticket → show existing status.

**(5.2) Staff: Call Next / Skip / Pause / Close**
Saff opens Staff Web → authenticates.
Press Call Next → API sets the next waiting ticket to called and increments current_number.
Press Skip → mark current as skipped, move to next.
Pause stops new joins; Close ends the day’s queue and prevents new tickets.

**(5.3) “You’re almost up” Notification**
Triggered when a ticket moves within a threshold (configurable).
Channel: SMS for USSD users; Push for app users.

# (6) Minimal API contract (illustrative)
POST /auth/otp/request           { phone }
POST /auth/otp/verify            { phone, code }

GET  /branches                   → list branches
POST /queues/:branchId/join      → create ticket
GET  /tickets/:id/status         → position, status

POST /staff/queues/:id/call-next
POST /staff/tickets/:id/skip
POST /staff/queues/:id/pause
POST /staff/queues/:id/close

USSD webhooks (from telco) hit:
POST /ussd/inbound  (body carries session_id, phone, user selection)

# (7) Notifications
Provider: SMS (e.g., Twilio / Africa’s Talking) + push (Firebase/APNs).
Strategy: send join confirmation and “you’re close” alerts.
Retry: 3 attempts with back-off; log to notifications.

# (8) Security & privacy (MVP)
Phone verification (OTP).
Role-based access: user vs staff.
Limit staff actions to their branch.
Store only what we need (no sensitive PII beyond contact details).
Basic rate-limit on USSD/web endpoints.

# (9) Observability
Logs for every staff action (audit).
Error logging with request IDs.
Basic metrics: tickets/day per branch, average wait time, no-show rate.

# (10) Scaling path
Single DB is fine for MVP.
Later: queue operations wrapped in transactions to avoid race conditions.
Scale read traffic with caching (e.g., Redis for “people ahead”).
Shard by branch/org if needed in future.

# (11) Non-functional targets (MVP)
Availability: 99.5% (working hours).
Latency: < 400ms for ticket status API.
SMS delivery: attempt within 5s of trigger.
Data retention: tickets & logs kept 180 days (configurable).

# (12) Risks & mitigations
USSD dependency (telco downtime) → always send SMS confirmation with ticket; allow re-entry by phone number.
Skipped/No-show abuse → cap skips per ticket; auto-expire after X minutes.
Multiple joins → one active ticket per user per branch.

# (13) MVP vs Later
**MVP**
App + USSD join, live position, SMS/push, simple staff panel.

**Later**
Payments before service, multi-queue per branch (departments),
Public display screens (token TV), analytics dashboards,
Priority queues (elderly/pregnant/emergencies).

# (14) Why this approach is technically feasible
Well-known stacks (Node/Django + Postgres/MySQL) with large ecosystem support.
Queue logic is simple: incrementing ticket numbers with safe DB transactions.
USSD/SMS are standard integrations with clear SDKs/webhooks.
All channels share one backend; no duplicate logic.

# Architecture Diagram
graph LR
  subgraph Clients
    A1[Mobile App<br/>(React Native/Flutter)]
    A2[USSD Phone<br/>(*shortcode#)]
    A3[Staff Web Panel]
  end

  subgraph Platform (Backend/API)
    B1[API Gateway / Auth]
    B2[Queue Service<br/>- create ticket<br/>- live position<br/>- call next/skip]
    B3[Notification Service<br/>(SMS & Push)]
  end

  subgraph Integrations
    C1[Telco USSD Gateway]
    C2[SMS Provider]
  end

  subgraph Data Layer
    D1[(PostgreSQL/MySQL)]
  end

  A1 -->|REST| B1
  A3 -->|REST| B1
  A2 -->|USSD Session| C1 -->|Webhook| B1

  B1 --> B2
  B1 --> B3
  B2 --> D1
  B3 --> C2



# (15) Dev notes / setup (quick)
Env vars: DB_URL, SMS_API_KEY, JWT_SECRET, USSD_WEBHOOK_SECRET.
Seed data: organizations, branches, one open queue for testing.
Local: run DB in Docker, start API, hit endpoints with Postman.

# (16) Open questions (for mentors/devs)
Ticket expiry window (e.g., 3–5 minutes after “called”)?
How many “skips” before marking no_show?
USSD short code length/cost in Nigeria (shared vs dedicated)?