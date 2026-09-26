# Waypoint Relay: backend spec (Hackathon)

The design is final. This file tells the backend team what to build, in what order, and which rules the data must follow.

| | |
|---|---|
| Deadline | **Sun 4 Oct 2026, 11:59 PM (Sri Lanka time)**. Code pushed after that is ignored |
| Design (the spec) | [Figma: LegendX_Designathon](https://www.figma.com/design/dtE7b7R8CPwZh5scsxu6zx/LegendX_Designathon?node-id=16-2) |
| ER diagram and user flows | [Excalidraw (finalised 25 Sep)](https://excalidraw.com/#json=38SJ1pFa3E_6EY4bnMtIf,pSYGqIPv4kXvP3tkBRSOHw) |
| Team brief | `TECH_TRIATHLON_2026_MASTER_BRIEF.md`, Sections 11, 14, 20–28 |

**What judges score that depends on the backend (booklet p.12–13):** planning and allocation engine 20% · engineering quality and architecture 25% · degradation, offline and recovery 10% · fidelity to the Day 5 design 10%. *"Plans must be runnable"* and *"respect every operating constraint"*.

---

## 1. Stack and layout

| Part | Choice |
|---|---|
| API | **NestJS** (TypeScript), one modular monolith |
| Database | **PostgreSQL 16** + **Prisma** (migrations + seed) |
| Shared rules | `packages/shared`: Zod schemas + the constraint validator, used by the API **and** the web app |
| Allocation | `packages/engine`: **HiGHS** (`highs-js`) with a greedy fallback |
| Auth | JWT, 4 roles; loaders sign in on the shared tablet with a 4-digit PIN |
| Live updates | Server-Sent Events (30 s heartbeat) |
| Files | **MinIO** (S3 API, presigned uploads) behind a `StorageService`, so a Docker volume can replace it |
| Docs | Swagger at `/api/docs`, `/health` endpoint |
| Not used | Redis, BullMQ, microservices, GraphQL, WebSockets, LLM or MCP (brief 26.13–27.5) |

```
apps/api/src/
├── auth/           login, JWT, role guards, loader PIN
├── master-data/    depots, districts, outlets, vehicles, calendar (read-only, seeded)
├── orders/         place, edit before cutoff, 4 PM cutoff, history
├── planning/       auto-plan, validator, deferrals, swap suggestion, publish
├── loading/        load list, load checks, vehicle ready
├── delivery/       driver run, proof of delivery, delivery problems
├── sync/           offline batch sync (idempotent), conflicts
├── receiving/      goods receipt, issues
├── live/           progress timeline, driver last contact
└── events/         SSE stream, event log
```

---

## 2. Data model

Entities and attributes come from the **finalised ER diagram**. Rows marked **(+)** are extra fields the Figma screens need; the ER does not have them yet. **Confirm the (+) rows with Kulasekara before the first migration.**

Times are `timestamptz` in **Asia/Colombo**. Every editable table gets `created_at`, `updated_at` and, where two people can edit the same row, a `version` int for optimistic locking.

### 2.1 Master data (seeded from CSV, read-only in the app)

**DEPOT**: `depot_id` PK (`Peliyagoda`, `Kandy`)

**DISTRICT** (12, from `district_travel.csv`)
| Field | Type | Note |
|---|---|---|
| district | string PK | |
| depot | FK → DEPOT | |
| depot_to_district_freeflow_min | int | trip-time rule |
| inter_stop_freeflow_min | int | trip-time rule |
| depot_to_district_km | float | fuel rule |
| (+) inter_stop_km | float | fuel rule (in the CSV) |
| (+) road_class, free_flow_kmh | string, float | in the CSV; keep for ETA |

**OUTLET** (120, from `outlets.csv`)
| Field | Type | Values |
|---|---|---|
| outlet_id | string PK | OUT001–OUT120 |
| brand | enum | Fresh / Style / Tech |
| district | FK → DISTRICT | |
| depot | FK → DEPOT | |
| dock_type | enum | rear_dock / street / mall_bay |
| parking_constraint | enum | normal / van_only / mall_dock |
| window_open_time, window_close_time | time | |
| (+) mall_window | string, nullable | e.g. `09:00-11:00` (in the CSV) |

**VEHICLE** (60, from `vehicles.csv`)
| Field | Type | Values |
|---|---|---|
| vehicle_id | string PK | VEH001–VEH060 |
| type | enum | truck / van |
| temp | enum | reefer / ambient |
| weight_cap_kg | int | |
| volume_cap_m3 | float | |
| km_per_l | float | |
| weekly_fuel_quota_l | int | |
| depot | FK → DEPOT | |
| status | enum | available / in_workshop (from `task2b_peak_day_fleet.csv` for the seeded day) |

**CALENDAR_DAY** (from `calendar.csv`): `date` PK, `is_operating`, `is_payday`, `festival`, `is_holiday`, `monsoon`, `iso_year`, `iso_week`.

**(+) SERVICE_ALLOWANCE** (lookup, from `service_allowance.csv`): `brand` + `dock_type` → `service_allowance_min`. Not drawn in the ER, but the trip-time rule cannot work without it.

### 2.2 People

**USER**
| Field | Type | Note |
|---|---|---|
| user_id | string PK | |
| role | enum | dispatcher / loader / driver / store_manager |
| depot | FK, nullable | dispatcher, loader, driver |
| outlet_id | FK, nullable | store manager only |
| pin_hash | string, nullable | loader quick sign-in |
| (+) name | string | shown in top bars and on records ("K. Silva") |
| (+) username, password_hash | string | dispatcher, driver, store manager sign-in |
| (+) last_seen_at | timestamptz | driver "last contact" on the Live screen |

### 2.3 Orders and plans

**ORDER**
| Field | Type | Note |
|---|---|---|
| order_id | string PK | client UUID (created on the device) |
| outlet_id | FK | |
| brand | enum | |
| temp_requirement | enum | chilled / ambient |
| order_units | int > 0 | cases |
| order_weight_kg | float > 0 | computed from units (Section 3.6) |
| order_volume_m3 | float > 0 | computed from units (Section 3.6) |
| delivery_date | date | the day requested |
| status | enum | Placed · Confirmed · Planned · Deferred · Loaded · OnTheWay · Delivered · Received · Disputed |
| placed_at | timestamptz | |
| (+) order_ref | string | shown in the UI, e.g. `S1-014` |
| (+) version | int | |

**DAILY_PLAN**: `plan_id` PK, `plan_date`, `depot` FK, `status` (draft / published), (+) `published_at`, (+) `published_by` FK → USER.

**TRIP**
| Field | Type | Note |
|---|---|---|
| trip_id | string PK | |
| plan_id | FK | |
| vehicle_id | FK | |
| trip_no | int | 1 or 2 |
| brand | enum | one brand per trip |
| district | FK | one district per trip |
| planned_minutes | int | within 270 (Fresh) or 480 (Style/Tech) |
| status | enum | (+) planned / loading / ready / departed / completed |
| (+) driver_id | FK → USER | the ER has the "drives" relationship but no field |
| (+) ready_at, departed_at | timestamptz | loader "Vehicle ready", Live screen |
| (+) at_risk | bool | realistic-time check (Section 3.4) |
| (+) version | int | |

**STOP**
| Field | Type | Note |
|---|---|---|
| stop_id | string PK | |
| trip_id | FK | |
| order_id | FK, UNIQUE | an order sits on one stop only |
| seq | int | 0 = first stop; the loader loads in reverse |
| planned_arrival | time | free-flow plan |
| eta | time | realistic (Section 3.4) or Datathon Task 1 |
| late_risk | float | Datathon Task 1, or the rule in 3.4 |
| (+) status | enum | pending / arrived / delivered / problem |
| (+) version | int | dispatcher vs driver conflicts |

**DEFERRAL**
| Field | Type | Note |
|---|---|---|
| deferral_id | string PK | |
| order_id | FK | |
| plan_id | FK | |
| reason | enum | reefer_full / larger_than_any_vehicle / van_only_no_van / no_time |
| consequence | string | e.g. "5th chilled deferral in 30 days" |
| decided_by | FK → USER | null = decided by auto-plan |
| (+) note | string, nullable | "Keep deferred" lets the dispatcher add a note |
| (+) next_delivery_date | date | shown to the store manager |
| (+) notified_at | timestamptz | set when the plan is published |

**FUEL_WEEK**: `vehicle_id` FK, `iso_week`, `planned_km`, `litres_used` (checked against `weekly_fuel_quota_l`).

### 2.4 The three counts and issues

Where the count drops is where the goods went missing: **LOAD_CHECK (loader) → DELIVERY (driver) → GOODS_RECEIPT (store)**.

**LOAD_CHECK**
| Field | Type | Note |
|---|---|---|
| stop_id | FK, UNIQUE | |
| loaded_units | int | |
| issue | enum | none / missing / damaged, (+) wrong_item / too_warm (the Flag shortfall screen has 4 reasons) |
| photo_url | string, nullable | |
| loaded_by | FK → USER | from the PIN session |
| checked_at | timestamptz | |
| (+) note | string, nullable | |

**DELIVERY** (works offline)
| Field | Type | Note |
|---|---|---|
| client_uuid | string PK, UNIQUE | blocks duplicate syncs |
| stop_id | FK | |
| delivered_units | int | |
| photo_url, signature_url | string | |
| delivered_at | timestamptz | real time on the phone |
| synced_at | timestamptz | when it reached the server |
| (+) signed_by_name | string | "N. Fernando, store staff" |
| (+) location | point, nullable | taken once at delivery (optional) |

**GOODS_RECEIPT** (the ER calls it RECEIPT): `order_id` FK, `received_units`, `confirmed_by` FK, `confirmed_at`.

**ISSUE**
| Field | Type | Note |
|---|---|---|
| issue_id | string PK | |
| order_id | FK | |
| raised_by | FK → USER | |
| type | enum | shortfall / damage / discrepancy / late, (+) delivery_problem / sync_conflict |
| status | enum | open / resolved |
| (+) reason_code | enum | store: damaged / short / wrong_item / too_warm · driver: access_blocked / store_closed / no_receiver / goods_damaged / vehicle_problem |
| (+) units_affected | int, nullable | |
| (+) note, photo_url | string, nullable | |
| (+) resolved_by, resolved_at, resolution | FK, timestamptz, string | Issues screen |

### 2.5 Proposed addition: ORDER_EVENT

The store manager's **Notifications** and **History** screens and the dispatcher's **Live** timeline need a list of what happened and when. The brief says notifications are not stored as an entity, so these screens need an append-only event log:

`event_id` PK · `order_id` FK · `trip_id` FK nullable · `type` · `from_status` · `to_status` · `actor_id` FK · `at` · `payload` json

Every status change writes one row inside the same transaction. The SSE stream sends the same events. **Needs team agreement** (brief 27.5 lists it as "consider").

### 2.6 In the ER but not built

**FORECAST** (Datathon Task 2A output) is in the ER, but no screen uses it: the forecast screen was left out on purpose (Figma page 04). Don't create the table unless the team integrates the Datathon later.

### 2.7 Constraints and indexes

```
UNIQUE  delivery.client_uuid
UNIQUE  stop.order_id
UNIQUE  load_check.stop_id
UNIQUE  trip (vehicle_id, plan_id, trip_no)
CHECK   trip.trip_no IN (1, 2)
CHECK   order_units > 0 AND order_weight_kg > 0 AND order_volume_m3 > 0
INDEX   order (outlet_id, delivery_date)      -- store manager screens
INDEX   order (delivery_date, status)         -- dispatcher queue
INDEX   stop (trip_id, seq)                   -- loader and driver lists
INDEX   trip (vehicle_id, plan_id)
INDEX   deferral (order_id)                   -- skip history
```
No triggers, stored procedures or DB functions: the rules live in the shared TypeScript validator (brief 26.8).

---

## 3. Business rules

### 3.1 Order cutoff
- Orders for tomorrow close at **16:00**. A store can edit its order until then.
- An order placed after 16:00 moves to the **next operating day** (`calendar.is_operating = 1`; Waypoint runs Monday to Saturday).
- At 16:00 a cron job (`@nestjs/schedule`) moves Placed → Confirmed and puts them in the dispatcher's queue.

### 3.2 The 7 feasibility rules (the validator)
One function in `packages/shared`, used by the engine, the API and the dispatcher's browser.

1. One **brand** and one **district** per trip.
2. **Chilled** orders only on `temp = reefer` (reefers may also carry ambient).
3. **van_only** outlets only on `type = van`.
4. A vehicle only serves outlets of **its own depot**.
5. **Whole orders** only; each served order sits on exactly one vehicle and trip.
6. **Weight AND volume** per trip within the vehicle's limits.
7. **Max 2 trips** per vehicle per day, within the time budgets: **Fresh 270 min** (03:30–08:00) for all of that vehicle's Fresh trips together, **480 min** for Style + Tech. Vehicles `in_workshop` can't be used.

Plus the **fuel quota**: the week's planned litres per vehicle ≤ `weekly_fuel_quota_l`.

### 3.3 Trip time and fuel (same formula as `check_allocation.py`)
```
trip_minutes = depot_to_district_freeflow_min
             + (stops - 1) * inter_stop_freeflow_min
             + sum(service_allowance_min[brand, dock_type] for each stop)

planned_km   = 2 * depot_to_district_km + (stops - 1) * inter_stop_km
litres       = planned_km / km_per_l
```
**Acceptance test:** export the seeded day's plan to the `submission_task2b.csv` format and run the official `check_allocation.py` in CI. It must print `FEASIBILITY: PASSED`.

### 3.4 Realistic ETA and "at risk"
- The checker counts free-flow, one-way time only. Real travel is **1.34×** the plan (**1.74×** in monsoon).
- `eta = planned_arrival` with travel × 1.34 (or × 1.74 when `calendar.monsoon = 1`). If Datathon Task 1 predictions are integrated, use them instead.
- **Fresh trip 2:** add the return leg of trip 1 before trip 2 starts. On S1 many trip 2 runs pass the checker but still reach stores after 08:00.
- Set `trip.at_risk` and a stop's late risk when the expected arrival is after `window_close_time`. The design shows VEH006 Trip 2 "at risk ~08:25".

### 3.5 Deferrals and skip history
- **Skip** = a past order with `dispatch_status = 'deferred'` (not `not_run`), per outlet and temperature.
- The deferred-order sheet shows the **last 10 runs**, the **30-day count**, and the **count since Jan 2024**.
- Checked against the data: OUT008 chilled = **5 of 17** in the last 30 days, **151 of 440** since Jan 2024; OUT066 and OUT067 = 0 in 30 days. These are the numbers in the Figma screens.
- Every deferral stores a **reason** and is sent to the store manager with the reason and the new date when the plan is **published**.

### 3.6 Cases to kg and m³
The store manager enters **cases** only. The server computes weight and volume from the median per case in `deliveries_train.csv`:

| Brand | Temp | kg per case | m³ per case |
|---|---|---|---|
| Fresh | ambient | 6.843 | 0.0370 |
| Fresh | chilled | 6.857 | 0.0371 |
| Style | ambient | 14.785 | 0.2384 |
| Tech | ambient | 211.763 | 0.7063 |

Put these in a small seeded lookup table, not in code.

### 3.7 Allocation engine
- **Auto-plan** at 16:05 (or on the dispatcher's "Re-run auto-plan"): HiGHS maximises served orders, chilled volume first, weighted by `deferred_yesterday`, `days_since_last_served` and the 30-day skip count, subject to all rules in 3.2. Time limit about 10 s; if HiGHS fails, fall back to the greedy plan (chilled first, then priority, then smallest volume).
- **Swap suggestion** (deferred-order sheet): for a deferred chilled order, try re-pointing one reefer trip to that order's district. Keep the swaps that pass every rule and serve orders with more skips. Mark the best one **"Suggested"**.
- **Consequences before applying:** return the change in served count, deferred count, chilled m³ served, the vehicle's trip minutes and the number of violations (design: 93.7 → 101.5 m³, 13 → 12 deferred, 0 violations).
- **Publish:** trips, stops, deferrals, order status changes and events are saved in **one transaction**. Publishing is refused if the validator finds any violation.

### 3.8 Dispatcher suggestions: rule-based symbolic AI
The dispatcher's suggestions come from a small **expert system** (symbolic AI): rules plus inference, with an explanation for every suggestion. No machine learning, no LLM, no external service; it runs on our server and gives the same answer for the same plan.

| Part | In our system |
|---|---|
| Knowledge base | Production rules stored as data in `packages/engine/rules`: `id`, `when` (condition on facts), `then` (action), `priority`, `explain` (text template) |
| Working memory (facts) | The draft plan: trips, stops, deferred orders, skip counts (3.5), vehicle limits, windows, ETAs (3.4) |
| Inference engine | Forward chaining: run the rules, add new facts (candidates) until nothing changes. Conflicts are settled by `priority`, then by score |
| Safety check | Every candidate must pass the shared validator (3.2); a candidate that breaks a rule is dropped |
| Explanation | The fired rules become the "why" text on the deferred-order sheet |

**Rules for the screens in the design:**
| Rule | When | Then | Shown in the design |
|---|---|---|---|
| R1 | A chilled order is deferred and its outlet has ≥ 3 chilled skips in 30 days | Look for a reefer-trip swap in the same depot | Deferred-order sheet |
| R2 | A candidate breaks any of the 7 rules | Drop it | (not shown) |
| R3 | A swap serves outlets with more skips than the outlets it defers | Prefer it; mark the best one **"Suggested"** | "Suggested" badge + consequences |
| R4 | An order is larger than any available vehicle | "Split the order or send it next run" | Deferred list |
| R5 | A van-only outlet and every van is full | "Van-only; both vans are full" → next run | Deferred list |
| R6 | Expected arrival is after the window close | Mark the trip **at risk** | Plan published, Live |

Example explanation: *"OUT008 chilled was deferred 5 times in 30 days; OUT066 and OUT067 had 0. The swap passes all 7 rules and serves 1 more order (+7.8 m³)."*

- Write the engine ourselves (about one file) so the team can explain every line; `json-rules-engine` is an acceptable alternative.
- Unit-test every rule, plus one golden test on S1: the engine must suggest the VEH006 Trip 1 swap for OUT008.
- Suggestions only appear where the design shows them. A new suggestion anywhere else is a design departure; write it in `docs/design-deviations.md`.

### 3.9 Order lifecycle
```
Placed → Confirmed (16:00) → Planned | Deferred (reason) → Loaded → OnTheWay → Delivered → Received
Deferred → Confirmed (next run)          Delivered → Disputed → Received (resolved with evidence)
```

---

## 4. API (REST, `/api/v1`)

Every endpoint checks the role **and** the scope: a store manager sees only their outlet, a driver only their trips, a loader only their depot, the dispatcher their depot.

### Auth
| Method | Path | Screen |
|---|---|---|
| POST | `/auth/login` | Sign in (phone, desktop) |
| POST | `/auth/pin` | Loader PIN sign-in: `{ user_id, pin }` on a depot tablet |
| POST | `/auth/logout` | "Switch loader" |
| GET | `/auth/me` | Top bar name, role, outlet or depot |

### Store manager
| Method | Path | Screen |
|---|---|---|
| GET | `/outlet/orders?date=` | Place order (today's order + recent orders) |
| POST | `/orders` | Place order (dry and chilled; idempotent on `order_id`) |
| PATCH | `/orders/:id` | Change cases before 16:00 |
| GET | `/outlet/tomorrow` | Tomorrow: both ETAs, deferral notice with reason and new date |
| GET | `/outlet/notifications` | Notifications (from ORDER_EVENT) |
| GET | `/orders/:id/proof` | Proof of delivery: photo, signature, 3 counts, times |
| POST | `/orders/:id/receipt` | Confirm receipt |
| POST | `/orders/:id/issues` | Report issue (reason, cases, photo) |
| GET | `/outlet/history?days=30` | History |

### Dispatcher
| Method | Path | Screen |
|---|---|---|
| POST | `/plans/auto?date=&depot=` | Tomorrow's plan (draft) |
| GET | `/plans/:id` | Plan: summary cards, trips, deferred list |
| GET | `/orders/:id/deferral-context` | Deferred-order sheet: history, counts, options with consequences |
| POST | `/plans/:id/simulate` | Consequences of a swap, without saving |
| POST | `/plans/:id/swaps` | Apply a swap |
| POST | `/deferrals/:id/keep` | Keep deferred (+ note) |
| POST | `/plans/:id/publish` | Publish plan; notifies the stores |
| GET | `/live?date=&depot=` | Live: per-vehicle timeline, at risk, offline with last contact |
| GET | `/issues?status=open` | Issues inbox (shortfalls, problems, sync conflicts) |
| POST | `/issues/:id/resolve` | Resolve an issue |

### Loader
| Method | Path | Screen |
|---|---|---|
| GET | `/loader/trips?date=` | Choose the vehicle (own depot) |
| GET | `/trips/:id/load-list` | Load list, **last stop first** |
| POST | `/stops/:id/load-check` | Loaded, or Flag shortfall (counted, reason, note, photo) |
| POST | `/trips/:id/ready` | Vehicle ready |

### Driver
| Method | Path | Screen |
|---|---|---|
| GET | `/driver/run` | Run: today's trip and stops, cached on the phone for offline use |
| POST | `/sync/batch` | Every driver write goes through here (Section 5) |
| POST | `/files/presign` | Upload URL for a photo or signature |
| POST | `/driver/heartbeat` | Every 60 s while online; sets `last_seen_at` |

### Shared
| Method | Path | Use |
|---|---|---|
| GET | `/events` | SSE stream, filtered by role and scope |
| GET | `/health` | Docker healthcheck |

---

## 5. Offline sync (the degradation scenario)

Design: *driver offline on the Kandy corridor* (Figma page 03). This is 10% of the Hackathon score.

- The phone stores the run and every action in IndexedDB (Dexie) with a **client UUID** and the **device time**.
- `POST /sync/batch` takes a list of `{ client_uuid, kind: delivery | arrival | problem, stop_id, payload, device_time }`.
- The whole batch runs in **one transaction**; each row uses `INSERT … ON CONFLICT (client_uuid) DO NOTHING`, so a retry never counts twice.
- Store both times: `delivered_at` (phone) and `synced_at` (server).
- A delivery counts as complete only after its photo is uploaded. Photos are compressed on the phone (about 200 KB).
- **Conflict rule: what happened on the ground wins.** If the dispatcher changed a stop while the driver was offline, keep the driver's delivery, reject the plan change for that stop, and create an ISSUE `type = sync_conflict` for the dispatcher. Detect the conflict with the `version` column.
- **Offline, not late:** during a departed trip, a driver with no request for **10 minutes** shows as "Offline since HH:MM" with the last contact, not as Late or Missing.
- The response returns a result per row (`saved`, `duplicate`, `conflict`), so the phone can clear its queue and show "4 records uploaded, 1 conflict explained".
- Deadlocks: keep transactions short, lock in the order ORDER → STOP → DELIVERY, retry `P2034` up to 3 times.

---

## 6. Events (SSE and ORDER_EVENT)

| Event | Who receives it |
|---|---|
| `order.confirmed` | dispatcher |
| `plan.published` | loaders of the depot, store managers with orders on it |
| `order.deferred` | the store manager (reason + new date) |
| `loadcheck.shortfall` | dispatcher, the trip's driver, the store manager |
| `trip.ready`, `trip.departed` | dispatcher, store manager (ETA) |
| `stop.delivered` | dispatcher, store manager |
| `sync.completed`, `sync.conflict` | dispatcher |
| `driver.offline`, `driver.online` | dispatcher |
| `issue.created`, `issue.resolved` | dispatcher, the reporter |

---

## 7. Database and infrastructure practices

### 7.1 Transactions (all or nothing)
| Operation | Why |
|---|---|
| Publish plan | Trips, stops, deferrals, order statuses and events are saved together |
| Offline sync batch | No half-saved batch |
| Defer + status change + event | The deferral, the order status and the store's notification always match |

Keep every transaction short: never wait for user input or a file upload inside one.

### 7.2 Concurrency
- **Idempotent sync:** `INSERT … ON CONFLICT (client_uuid) DO NOTHING` (Section 5).
- **Optimistic locking:** ORDER, TRIP and STOP have a `version` column. Update with `WHERE id = ? AND version = ?`; 0 rows updated means someone else changed it, so return a conflict instead of overwriting.
- **No pessimistic locking:** avoid `SELECT … FOR UPDATE`; it raises the deadlock risk.

### 7.3 Deadlocks
- PostgreSQL detects a deadlock by itself and cancels one transaction (error `40P01`, Prisma `P2034`). The system does not freeze.
- Risk is low: each driver writes only their own stops, and planning (evening) and driver syncs (early morning) happen at different times.
- Rules: short transactions · always lock in the order ORDER → STOP → DELIVERY (sort rows by id when updating many) · retry `P2034` up to 3 times with a short back-off (safe, because sync is idempotent).

### 7.4 No logic in the database
- No triggers, stored procedures or DB functions. The rules live in the shared TypeScript validator, which also runs in the browser; SQL copies would drift and are harder to test.
- `updated_at` uses Prisma's `@updatedAt`, not a trigger.
- The database still refuses bad data through foreign keys, `UNIQUE` and `CHECK` constraints (Section 2.7).

### 7.5 No database dump
No document asks for a backup or dump file. `prisma migrate deploy` creates the tables and an **idempotent seed script** loads the data on `docker compose up` (Section 8). Running `up` twice must not duplicate anything.

### 7.6 Docker Compose
- `db`: `postgres:16-alpine`, with a `pg_isready` healthcheck; the API uses `depends_on: db: condition: service_healthy`, so it never starts before the database (the judges' first run would crash).
- `api` start command: `prisma migrate deploy` → seed → server.
- Passwords live in `.env` (not committed); `.env.example` is committed.
- The **same compose file** runs on the VM, so local and production match.

### 7.7 MinIO (photos and signatures)
- **Create the bucket automatically** with a one-shot `minio-init` container (`mc mb --ignore-existing local/pod-photos`), or the first upload fails on a fresh install.
- **Pin the MinIO image version** (no `latest`), and check that it still pulls before relying on it.
- Configure **CORS**, because the browser uploads directly with presigned URLs.
- Photos are compressed on the phone (about 200 KB). Offline photos wait in IndexedDB and upload on sync.
- Everything goes through `StorageService.upload()`, so switching to a Docker volume is a one-file change.

### 7.8 No Redis or job queue
One API instance: scheduled jobs (16:00 cutoff, weekly fuel reset) use `@nestjs/schedule`, live events use the in-process event emitter, and PostgreSQL is fast enough without a cache. If a background queue is ever needed, use **pg-boss** (a queue inside PostgreSQL, no new container).

### 7.9 For semi-final Q&A: concurrency theory
We rely on PostgreSQL's **MVCC** (readers don't block writers) and its **built-in deadlock detection**, so we don't implement timestamp ordering, wait-die or wound-wait. At the application level, the `version` column is **optimistic concurrency control**, the fixed lock order is **deadlock prevention**, and the retry is **deadlock recovery**. Offline sync is idempotent, so retries never create duplicates.

---

## 8. Seed data (`docker compose up` on a fresh install must work)

Order: `prisma migrate deploy` → idempotent seed (skip if data exists) → start the API.

| Source | Loads |
|---|---|
| `General Data/outlets.csv`, `vehicles.csv`, `calendar.csv`, `district_travel.csv`, `service_allowance.csv` | master data |
| `Test Data/task2b_peak_day_scenarios.csv` | the **S1 peak day**: 85 orders from 59 outlets (Confirmed) |
| `Test Data/task2b_peak_day_fleet.csv` | vehicle status for S1 (28 available, 10 in the workshop) |
| cases-to-kg/m³ table (3.6) | lookup |
| skip-history summary per outlet and temp | the deferred-order sheet (see open question 2) |

**Seeded accounts** (names from the Figma personas):
| Role | Name | Scope |
|---|---|---|
| Dispatcher | Dilani Perera | Peliyagoda |
| Loader | Kasun Silva (+ 2 more names for the PIN picker) | Peliyagoda dock, tablet |
| Driver | Suresh Kumar | VEH006 |
| Driver (degradation) | a Kandy driver | VEH043 |
| Store manager | Fathima Rizwan | OUT008 |

The S1 plan date is the next operating day after the last history day in the data (Sat 14 Feb 2026 → **Mon 16 Feb 2026**). Check it against the Figma History screen before seeding.

**Never commit** the Datathon training or test files; only the seed subset in `seed/**/*.csv` (the `.gitignore` already allows that path).

---

## 9. Open questions (decide in the first team call)

1. **(+) fields and ORDER_EVENT** (Section 2): agree them with Kulasekara, then update the Excalidraw ER and `docs/data-model.md`.
2. **Skip history needs past deliveries.** Seeding it means committing a small derived summary (120 outlets × 2 temps). Booklet p.22 forbids publishing the datasets *"or any derivatives"*. Options: keep the repo private and share it with the judges, or ask tech-triathlon@rootcode.io first.
3. **Ambient orders in spare reefer space:** allowed by the booklet (p.5). Allow it in the engine?
4. **A 3 AM shortfall with no dispatcher on shift:** the design notifies an on-call dispatcher and lets the truck leave short. Confirm.
5. **Offline threshold:** 10 minutes without contact is a guess; agree it with the frontend team.

---

## 10. Build order (start now, finish by 3 Oct; 4 Oct is buffer and submission)

| Step | Backend work |
|---|---|
| 1 | Prisma schema, migrations, seed, auth (login, PIN, roles), `/health`, Swagger, Docker Compose with DB healthcheck |
| 2 | Orders and cutoff, shared validator, trip-time and fuel functions, `check_allocation.py` test in CI |
| 3 | Engine (HiGHS + greedy), deferral context, swap simulation, publish transaction |
| 4 | Loader endpoints, driver run, `/sync/batch`, MinIO presigned uploads, conflicts |
| 5 | Receipt and issues, Live and last contact, SSE, README walkthrough, deployment, tag `v1.0-submission` |

### Done means
- [ ] `docker compose up` on a clean machine starts DB, MinIO, API and web, with seed data
- [ ] A judge can run one full cycle with the 4 seeded accounts: order → plan → load → deliver (also offline) → receipt
- [ ] The seeded plan passes `check_allocation.py`
- [ ] A duplicated sync batch saves nothing twice
- [ ] Every screen in Figma has its endpoint; any departure is written in `docs/design-deviations.md`
