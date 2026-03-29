# Problem 11: Scheduled Environment Operations

## Interviewer Prompt

> "We want to let customers schedule operations on their healthcare environments.
>
> For example, a hospital might want to automatically stop their dev environments at 6 PM every day to save costs, then start them again at 8 AM. Or they might want to schedule a one-time maintenance operation.
>
> We need to handle timezones correctly, allow customers to cancel or modify schedules, and make sure jobs run reliably even if our service restarts.
>
> How would you design this?"

---

# Step 1: Clarify Requirements (3-5 min)

## Questions to Ask

| Question | Why It Matters | Example Answer |
|----------|----------------|----------------|
| What operations can be scheduled? | Scope of work | Start, stop, restart, scale |
| Recurring or one-time? | Scheduling complexity | Both |
| How precise must timing be? | SLA definition | Within 60 seconds of scheduled time |
| How many scheduled jobs? | Scale requirements | 50K jobs, 5K executions/hour peak |
| Can jobs overlap? | Conflict handling | No, queue if previous still running |
| What if execution fails? | Retry policy | Retry 3x with backoff |

## Functional Requirements (Confirmed)

- **Schedule types**: One-time and recurring (cron-like)
- **Operations**: Start, stop, restart environment
- **Timezone support**: Customer specifies timezone, system handles DST
- **Management**: Create, update, pause, resume, delete schedules
- **Execution history**: Track all executions with status
- **Notifications**: Webhook or email on execution complete/fail

## Non-Functional Requirements

- **Reliability**: Jobs execute even after service restart
- **Timing accuracy**: Within 60 seconds of scheduled time
- **Scalability**: 50K active schedules, 5K executions/hour
- **Idempotency**: Same job doesn't execute twice

---

# Step 2: Define Scope & Actors (2-3 min)

## Core Actors

| Actor | Scope | Key Actions |
|-------|-------|-------------|
| **Environment Admin** | Single environment | Create/manage schedules for their env |
| **Hospital Admin** | Tenant-wide | View all schedules, set policies |
| **Scheduler (System)** | Internal | Execute jobs on time, handle retries |

## In Scope (This Interview)
- Schedule CRUD APIs
- Job execution engine
- Timezone handling
- Retry and failure handling

## Out of Scope
- Cost calculation for start/stop
- Approval workflows for scheduled changes
- Integration with external calendar systems

---

# Step 3: High-Level Architecture (10 min)

## Core Insight: Scheduler + Executor Separation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SCHEDULE SERVICE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐    ┌─────────────────────┐                              │
│   │   Client     │───▶│   Schedule API      │                              │
│   │              │    │   (CRUD)            │                              │
│   └──────────────┘    └──────────┬──────────┘                              │
│                                  │                                          │
│                                  ▼                                          │
│                       ┌─────────────────────┐                              │
│                       │   Schedule Store    │◀── PostgreSQL                │
│                       │   (Persistent)      │    (source of truth)         │
│                       └──────────┬──────────┘                              │
│                                  │                                          │
│                                  │ Scheduler reads                          │
│                                  ▼                                          │
│   ┌──────────────────────────────────────────────────────────────────┐     │
│   │                      JOB SCHEDULER                                │     │
│   │                                                                   │     │
│   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │     │
│   │   │  Schedule   │───▶│   Due Job   │───▶│  Execution  │         │     │
│   │   │  Scanner    │    │   Queue     │    │  Workers    │         │     │
│   │   │  (1/min)    │    │  (Redis)    │    │  (N nodes)  │         │     │
│   │   └─────────────┘    └─────────────┘    └──────┬──────┘         │     │
│   │                                                 │                │     │
│   └─────────────────────────────────────────────────┼────────────────┘     │
│                                                     │                       │
└─────────────────────────────────────────────────────┼───────────────────────┘
                                                      │
                                                      │ Executes
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ENVIRONMENT CONTROL PLANE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐                │
│   │  Start Env    │   │   Stop Env    │   │  Restart Env  │                │
│   │  API          │   │   API         │   │  API          │                │
│   └───────────────┘   └───────────────┘   └───────────────┘                │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility |
|-----------|---------------|
| **Schedule API** | CRUD for schedules, validation, timezone handling |
| **Schedule Store** | Persistent storage of schedule definitions |
| **Schedule Scanner** | Periodically finds due jobs, enqueues them |
| **Due Job Queue** | Redis queue of jobs ready to execute |
| **Execution Workers** | Pick jobs from queue, call control plane APIs |

---

# Step 4: API Design (5 min)

## Schedule Management APIs

```
POST   /api/v1/environments/{env_id}/schedules           # Create schedule
GET    /api/v1/environments/{env_id}/schedules           # List schedules
GET    /api/v1/schedules/{schedule_id}                   # Get schedule details
PUT    /api/v1/schedules/{schedule_id}                   # Update schedule
DELETE /api/v1/schedules/{schedule_id}                   # Delete schedule
POST   /api/v1/schedules/{schedule_id}/pause             # Pause schedule
POST   /api/v1/schedules/{schedule_id}/resume            # Resume schedule
GET    /api/v1/schedules/{schedule_id}/executions        # Execution history
```

## Example: Create Recurring Schedule

```json
POST /api/v1/environments/env_dev_123/schedules
{
  "name": "Nightly Shutdown",
  "operation": "STOP",
  "schedule_type": "RECURRING",
  "cron_expression": "0 18 * * 1-5",    // 6 PM Mon-Fri
  "timezone": "America/New_York",
  "enabled": true,
  "notification": {
    "webhook_url": "https://hospital.com/webhooks/schedule",
    "notify_on": ["FAILURE", "SUCCESS"]
  }
}
```

## Example: Response

```json
{
  "schedule_id": "sched_abc123",
  "env_id": "env_dev_123",
  "tenant_id": "hospital_456",
  "name": "Nightly Shutdown",
  "operation": "STOP",
  "schedule_type": "RECURRING",
  "cron_expression": "0 18 * * 1-5",
  "timezone": "America/New_York",
  "enabled": true,
  "status": "ACTIVE",
  "next_run_utc": "2024-03-15T22:00:00Z",
  "next_run_local": "2024-03-15T18:00:00-04:00",
  "created_at": "2024-03-14T10:00:00Z"
}
```

## Example: Create One-Time Schedule

```json
POST /api/v1/environments/env_prod_789/schedules
{
  "name": "Maintenance Window Start",
  "operation": "STOP",
  "schedule_type": "ONE_TIME",
  "run_at": "2024-03-20T02:00:00",
  "timezone": "America/Chicago",
  "enabled": true
}
```

## Example: Execution History Response

```json
GET /api/v1/schedules/sched_abc123/executions
{
  "schedule_id": "sched_abc123",
  "executions": [
    {
      "execution_id": "exec_001",
      "scheduled_at": "2024-03-14T22:00:00Z",
      "started_at": "2024-03-14T22:00:12Z",
      "completed_at": "2024-03-14T22:00:45Z",
      "status": "SUCCESS",
      "duration_ms": 33000
    },
    {
      "execution_id": "exec_002",
      "scheduled_at": "2024-03-15T22:00:00Z",
      "started_at": "2024-03-15T22:00:08Z",
      "completed_at": "2024-03-15T22:01:30Z",
      "status": "FAILED",
      "error": "Environment already stopped",
      "retry_count": 2
    }
  ]
}
```

---

# Step 5: Data Model (5 min)

## Entity Relationships

```
┌─────────────────┐       ┌─────────────────────┐       ┌─────────────────────┐
│   Environment   │1─────*│      Schedule       │1─────*│  ScheduleExecution  │
├─────────────────┤       ├─────────────────────┤       ├─────────────────────┤
│ env_id          │       │ schedule_id         │       │ execution_id        │
│ tenant_id       │       │ env_id (FK)         │       │ schedule_id (FK)    │
│ status          │       │ tenant_id           │       │ scheduled_at        │
│ ...             │       │ name                │       │ started_at          │
└─────────────────┘       │ operation           │       │ completed_at        │
                          │ schedule_type       │       │ status              │
                          │ cron_expression     │       │ error_message       │
                          │ run_at (one-time)   │       │ retry_count         │
                          │ timezone            │       └─────────────────────┘
                          │ enabled             │
                          │ status              │
                          │ next_run_utc        │
                          │ last_run_utc        │
                          │ notification_config │
                          │ created_at          │
                          └─────────────────────┘
```

## Schedule States

```
                    ┌──────────────────┐
          create    │                  │   enable
       ────────────▶│     ACTIVE       │◀───────────
                    │                  │            │
                    └────────┬─────────┘            │
                             │                      │
                    pause    │               ┌──────┴──────┐
                             ▼               │             │
                    ┌──────────────────┐     │   PAUSED    │
                    │                  │     │             │
                    │    PAUSED        │─────┘             │
                    │                  │     resume        │
                    └──────────────────┘                   │
                                                           │
                                        delete             │
                    ┌──────────────────┐◀──────────────────┘
                    │                  │◀─── (from any state)
                    │    DELETED       │
                    │                  │
                    └──────────────────┘
```

## Schema & Indexes

**schedules**
| Column | Type | Notes |
|--------|------|-------|
| schedule_id | VARCHAR | PK |
| env_id | VARCHAR | FK to environments |
| tenant_id | VARCHAR | |
| name | VARCHAR | Human-readable name |
| operation | VARCHAR | START, STOP, RESTART |
| schedule_type | VARCHAR | ONE_TIME, RECURRING |
| cron_expression | VARCHAR | NULL for one-time |
| run_at | TIMESTAMP | NULL for recurring |
| timezone | VARCHAR | e.g., "America/New_York" |
| enabled | BOOLEAN | |
| status | VARCHAR | ACTIVE, PAUSED, DELETED |
| next_run_utc | TIMESTAMP | Computed, used for scanning |
| last_run_utc | TIMESTAMP | |
| notification_config | JSONB | webhook_url, notify_on |
| created_at, updated_at | TIMESTAMP | |
| created_by | VARCHAR | |

**schedule_executions**
| Column | Type | Notes |
|--------|------|-------|
| execution_id | VARCHAR | PK |
| schedule_id | VARCHAR | FK to schedules |
| scheduled_at | TIMESTAMP | When it was supposed to run |
| started_at, completed_at | TIMESTAMP | |
| status | VARCHAR | PENDING, RUNNING, SUCCESS, FAILED |
| error_message | TEXT | |
| retry_count | INT | |
| worker_id | VARCHAR | Which worker executed |

**Indexes**
- `schedules(next_run_utc) WHERE enabled = true AND status = 'ACTIVE'` — partial index for scanner queries
- `schedules(env_id)` — list schedules for an environment
- `schedules(tenant_id)` — list schedules for a tenant
- `schedule_executions(schedule_id, scheduled_at DESC)` — execution history

---

# Step 6: Deep Dive - Core Flows (10 min)

## 6.1 Schedule Creation Flow

```
Admin creates schedule via API
        │
        ▼
┌─────────────────────────────────────┐
│ 1. Validate request                 │
│    - Valid cron expression?         │
│    - Valid timezone?                │
│    - Environment exists?            │
│    - Permission check               │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 2. Calculate next_run_utc           │
│                                     │
│    // Parse cron in customer TZ     │
│    cron = parse("0 18 * * 1-5")     │
│    tz = "America/New_York"          │
│                                     │
│    // Get next occurrence           │
│    next_local = cron.next(tz)       │
│    // "2024-03-15T18:00:00-04:00"   │
│                                     │
│    // Convert to UTC for storage    │
│    next_utc = next_local.to_utc()   │
│    // "2024-03-15T22:00:00Z"        │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 3. Store schedule                   │
│    - Insert into schedules table    │
│    - next_run_utc indexed for       │
│      efficient scanning             │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 4. Return schedule to client        │
│    - Include next_run in both       │
│      UTC and local time             │
└─────────────────────────────────────┘
```

## 6.2 Job Scanning and Execution Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SCHEDULE SCANNER (Every 1 min)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │ 1. Query due schedules                                          │       │
│   │                                                                  │       │
│   │    SELECT * FROM schedules                                      │       │
│   │    WHERE enabled = true                                         │       │
│   │      AND status = 'ACTIVE'                                      │       │
│   │      AND next_run_utc <= NOW() + INTERVAL '1 minute'            │       │
│   │    FOR UPDATE SKIP LOCKED                                       │       │
│   │                                                                  │       │
│   └──────────────────────────────┬──────────────────────────────────┘       │
│                                  │                                           │
│                                  ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │ 2. For each due schedule:                                        │       │
│   │    - Create execution record (PENDING)                          │       │
│   │    - Enqueue job to Redis                                       │       │
│   │    - Calculate and update next_run_utc                          │       │
│   │      (for recurring) or disable (for one-time)                  │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ Jobs enqueued
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXECUTION WORKER (N instances)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │ 1. Dequeue job from Redis                                        │       │
│   │    BRPOP schedule:jobs:queue 30                                 │       │
│   └──────────────────────────────┬──────────────────────────────────┘       │
│                                  │                                           │
│                                  ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │ 2. Acquire execution lock (Redis)                                │       │
│   │    SET schedule:lock:{execution_id} {worker_id} NX EX 300       │       │
│   │                                                                  │       │
│   │    (Prevents duplicate execution if worker crashes mid-job)     │       │
│   └──────────────────────────────┬──────────────────────────────────┘       │
│                                  │                                           │
│                                  ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │ 3. Update execution status: RUNNING                              │       │
│   └──────────────────────────────┬──────────────────────────────────┘       │
│                                  │                                           │
│                                  ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │ 4. Call environment control plane API                           │       │
│   │                                                                  │       │
│   │    POST /api/v1/environments/{env_id}/stop                      │       │
│   │    Headers: X-Idempotency-Key: {execution_id}                   │       │
│   └──────────────────────────────┬──────────────────────────────────┘       │
│                                  │                                           │
│                     ┌────────────┴────────────┐                             │
│                     │                         │                              │
│                SUCCESS                     FAILURE                           │
│                     │                         │                              │
│                     ▼                         ▼                              │
│   ┌─────────────────────────┐   ┌─────────────────────────────┐             │
│   │ 5a. Update execution:   │   │ 5b. Retry logic:            │             │
│   │     status = SUCCESS    │   │     if retry_count < 3:     │             │
│   │     Send success webhook│   │       re-enqueue with delay │             │
│   └─────────────────────────┘   │     else:                   │             │
│                                 │       status = FAILED       │             │
│                                 │       Send failure webhook  │             │
│                                 └─────────────────────────────┘             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 6.3 Timezone and DST Handling

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      TIMEZONE HANDLING EXAMPLES                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Schedule: "0 2 * * *" (2 AM daily) in America/New_York                     │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ Normal day:                                                        │      │
│  │   Local:  2024-03-14 02:00:00 EDT (UTC-4)                         │      │
│  │   UTC:    2024-03-14 06:00:00Z                                    │      │
│  │   Stored: next_run_utc = 2024-03-14 06:00:00Z                     │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ Spring forward (Mar 10, 2024 - 2 AM becomes 3 AM):                │      │
│  │   Cron "0 2 * * *" → 2 AM doesn't exist!                          │      │
│  │                                                                    │      │
│  │   Option A: Skip (cron library default)                           │      │
│  │   Option B: Run at 3 AM instead                                   │      │
│  │   Recommendation: Run at 3 AM (next valid time)                   │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ Fall back (Nov 3, 2024 - 2 AM happens twice):                     │      │
│  │   First 2 AM: 2024-11-03 02:00:00 EDT (UTC-4) = 06:00:00Z        │      │
│  │   Second 2 AM: 2024-11-03 02:00:00 EST (UTC-5) = 07:00:00Z       │      │
│  │                                                                    │      │
│  │   Recommendation: Use first occurrence (earlier UTC)              │      │
│  │   Document behavior clearly for customers                         │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  Implementation:                                                             │
│  - Store customer's timezone string (e.g., "America/New_York")              │
│  - Use proper TZ library (pytz, moment-timezone, java.time.ZoneId)          │
│  - Recalculate next_run_utc after EVERY execution                           │
│  - Never store local time; always convert to UTC for comparisons            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 6.4 Preventing Duplicate Execution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DUPLICATE PREVENTION MECHANISMS                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Level 1: Scanner deduplication                                             │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ - After enqueuing, immediately update next_run_utc                │      │
│  │ - Use FOR UPDATE SKIP LOCKED to prevent parallel scanners         │      │
│  │   from picking same schedule                                      │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  Level 2: Execution lock (Redis)                                            │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ - Before executing, acquire lock with NX (only if not exists)    │      │
│  │ - Lock key: schedule:lock:{execution_id}                         │      │
│  │ - TTL: 5 minutes (longer than max execution time)                │      │
│  │ - If lock fails, another worker is already executing             │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  Level 3: Idempotency key to downstream                                     │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ - Pass execution_id as idempotency key to control plane          │      │
│  │ - Control plane dedupes based on this key                        │      │
│  │ - Even if our dedup fails, downstream won't double-execute       │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  Level 4: Execution record status check                                     │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ - Before starting, check if execution already SUCCESS/FAILED     │      │
│  │ - If already terminal state, skip silently                       │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Step 7: Scaling & Bottlenecks (5 min)

## Potential Bottlenecks

| Bottleneck | Solution |
|------------|----------|
| **8 AM thundering herd** | Spread jobs with jitter (±30 sec) |
| **Scanner DB load** | Index on next_run_utc, batch queries |
| **Single scanner** | Multiple scanners with partition by tenant_id |
| **Worker overload** | Auto-scale workers based on queue depth |
| **Control plane overload** | Rate limit executions per downstream service |

## Handling Peak Load (8 AM Start)

```
Problem: 500 environments scheduled to start at 8:00 AM

┌─────────────────────────────────────────────────────────────────────────────┐
│                         THUNDERING HERD MITIGATION                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Strategy 1: Execution Jitter                                               │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ - Add random delay 0-60 seconds to each job                       │      │
│  │ - Jobs scheduled for 8:00 execute between 8:00-8:01               │      │
│  │                                                                    │      │
│  │ execution_delay = random(0, 60) # seconds                         │      │
│  │ actual_run = scheduled_time + execution_delay                     │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  Strategy 2: Priority Queues                                                │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ - Premium tenants get high-priority queue                         │      │
│  │ - Workers process high-priority first                             │      │
│  │                                                                    │      │
│  │ Queues: schedule:jobs:high, schedule:jobs:normal                 │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  Strategy 3: Auto-scaling Workers                                           │
│  ┌───────────────────────────────────────────────────────────────────┐      │
│  │ - Monitor queue depth                                             │      │
│  │ - Scale workers based on pending jobs                             │      │
│  │                                                                    │      │
│  │ if queue_depth > 100: scale_workers(+5)                          │      │
│  │ if queue_depth > 500: scale_workers(+20)                         │      │
│  └───────────────────────────────────────────────────────────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Numbers to Know

| Metric | Target |
|--------|--------|
| Schedule API latency | < 100ms |
| Job execution accuracy | Within 60 sec of scheduled time |
| Scanner interval | 1 minute |
| Max schedules per environment | 20 |
| Execution history retention | 90 days |

---

# Step 8: Tradeoffs & Wrap-Up (5 min)

## Key Design Tradeoffs

### 1. Scheduling Precision

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **1 min granularity** | Simple, low overhead | Max 60 sec late | **Use this** |
| **1 sec granularity** | Very precise | High scanner load | Overkill |
| **5 min granularity** | Very simple | Too imprecise | Too coarse |

### 2. Cron vs. Custom Format

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Standard cron** | Familiar, powerful | Syntax can be confusing | **Use this** |
| **Custom JSON** | More explicit | Another format to learn | Only for simple cases |
| **ISO 8601 intervals** | Standard | Less flexible | Not enough |

### 3. Scanner Architecture

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Single scanner** | Simple | SPOF | Only for <10K schedules |
| **Partitioned scanners** | Scalable | More complex | **Use this** |
| **DB-driven triggers** | No scanner needed | DB-specific | Vendor lock-in |

## Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Scanner down | Jobs not enqueued | Multiple scanners, health checks |
| Worker crash mid-job | Job stuck in RUNNING | Stale job detector, auto-retry |
| Control plane down | Executions fail | Retry with backoff, alert |
| Redis queue lost | Pending jobs lost | Scanner will re-enqueue on next cycle |

---

# Interviewer Follow-Up Questions (Prepare For)

1. **"Control plane is overloaded at 8 AM when 500 environments need to start. What do you do?"**
   - Add execution jitter (0-60 sec random delay)
   - Implement priority queues (premium tenants first)
   - Rate limit calls to control plane
   - Auto-scale workers but cap concurrent calls to downstream

2. **"Customer changes timezone from EST to PST. What happens to existing schedules?"**
   - Recalculate next_run_utc based on new timezone
   - Cron expression stays the same ("0 18 * * *" = 6 PM)
   - But 6 PM PST is different UTC than 6 PM EST
   - Update happens immediately on save

3. **"How do you prevent the same job from running twice?"**
   - Four levels of protection (see flow above)
   - Scanner updates next_run immediately after enqueue
   - Redis lock before execution
   - Idempotency key to downstream
   - Terminal state check before starting

4. **"Previous scheduled job is still running when next one is due. What happens?"**
   - Option A: Queue the new one (may cause backlog)
   - Option B: Skip the new one (recommended for most cases)
   - Option C: Cancel the running one and start new
   - Decision depends on operation type (stop is idempotent, start may not be)

---

# Summary: What Makes This Design Strong

1. **Reliable**: Survives crashes with persistent state and locks
2. **Timezone-aware**: Proper DST handling with standard TZ names
3. **Scalable**: Partitioned scanners, auto-scaling workers
4. **Idempotent**: Multiple layers prevent duplicate execution
5. **Observable**: Full execution history with status tracking
