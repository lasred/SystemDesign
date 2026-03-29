# Problem 10: Environment Resource Quota System

## Interviewer Prompt

> "We need to design a quota system for our healthcare control plane.
>
> Customers have limits on how many environments they can create, how much storage they can use, how many users they can have. We need to enforce these limits in real-time when they make requests.
>
> The tricky part is handling concurrent requests—two people might try to create environments at the same time, and we can't let them both succeed if it would exceed the quota.
>
> How would you design this?"

---

# Step 1: Clarify Requirements (3-5 min)

## Questions to Ask

| Question | Why It Matters | Example Answer |
|----------|----------------|----------------|
| What resources are quota-limited? | Scope of system | Environments, storage (GB), users, API calls |
| Hard limit vs soft limit? | Enforcement behavior | Both - soft warns, hard blocks |
| Quota per tenant or per env? | Hierarchy design | Per tenant, some per environment |
| How are quotas set? | Admin workflow | Default by tier, admin can override |
| Real-time or eventual? | Consistency requirements | Real-time for provisioning, eventual OK for usage display |
| What about pending resources? | Edge case handling | Count pending provisions against quota |

**Quota Hierarchy Example:**

Some quotas apply to the whole hospital (tenant), others to each environment:

```
Hospital ABC (tenant)
├── Tenant-level quotas (shared across all environments):
│   ├── Max 5 environments        ← Hospital can create 5 total
│   ├── Max 500 GB storage        ← 500 GB shared across all envs
│   └── Max 100 users             ← 100 users for the whole hospital
│
├── Production (environment)
│   └── Env-level quotas:
│       └── Max 50 API calls/sec  ← This env's rate limit
│
├── Staging (environment)
│   └── Env-level quotas:
│       └── Max 20 API calls/sec  ← Lower limit for staging
│
└── Dev (environment)
    └── Env-level quotas:
        └── Max 10 API calls/sec  ← Lowest for dev
```

**Why both levels?**
- **Tenant-level**: Controls total resource consumption for billing (hospital pays for 500 GB total, not per environment)
- **Environment-level**: Controls performance isolation (prod gets more API capacity than dev)

---

**Real-time vs Eventual Consistency for Quotas:**

"Consistency" = how quickly all parts of the system see the same data.

```
Real-time (strong consistency):
  User A creates environment → Quota updated immediately → User B sees new count instantly

Eventual consistency:
  User A creates environment → Quota updates "soon" (maybe 5 sec delay) → User B might see stale count briefly
```

**Why real-time for provisioning?**

When checking "can this hospital create another environment?", we MUST have accurate count.

```
Hospital ABC: 4/5 environments used

         User A                              User B
            │                                   │
   "Create environment"                "Create environment"
            │                                   │
            ▼                                   ▼
   ┌─────────────────┐                ┌─────────────────┐
   │ Check quota:    │                │ Check quota:    │
   │ 4 < 5? Yes! ✓   │                │ 4 < 5? Yes! ✓   │  ← Both see "4" (stale!)
   └────────┬────────┘                └────────┬────────┘
            │                                   │
            ▼                                   ▼
      Create env #5                       Create env #6   ← OVER QUOTA!

With eventual consistency, both requests see "4" and both succeed → 6 environments (bad!)
With real-time consistency, second request sees "5" and is blocked → 5 environments (correct!)
```

**Why eventual OK for usage display?**

The dashboard showing "You've used 4 of 5 environments" can be slightly stale. If it shows "4" when the real count is "5", the user just refreshes. No harm done—the actual enforcement at provisioning time is still real-time.

```
Dashboard (eventual):          Provisioning (real-time):
┌─────────────────────┐        ┌─────────────────────┐
│ Environments: 4/5   │        │ Can I create?       │
│ Storage: 340/500 GB │        │ → Check REAL count  │
│                     │        │ → Block if over     │
│ (updated every 30s) │        │ (must be accurate)  │
└─────────────────────┘        └─────────────────────┘
     ↑                              ↑
     │                              │
  Slight delay OK               Must be exact
  (just a display)              (enforces the rule)
```

## Functional Requirements (Confirmed)

- **Quota enforcement**: Block requests that would exceed quota
- **Quota types**: Environment count, storage GB, user count, API rate limits
- **Soft limits**: Warn at 80%, block at 100%
- **Admin override**: Support temporary or permanent quota increases
- **Usage tracking**: Show current usage vs quota in real-time
- **Pending reservation**: Reserve quota for in-progress provisioning

## Non-Functional Requirements

- **Low latency**: Quota check < 10ms (in critical path)
- **Strong consistency**: No over-provisioning due to race conditions
- **High availability**: Quota service can't be single point of failure
- **Auditability**: Log all quota changes and overrides

---

# Step 2: Define Scope & Actors (2-3 min)

## Core Actors

| Actor | Scope | Key Actions |
|-------|-------|-------------|
| **Control Plane Services** | Internal | Check quota, reserve, release |
| **Hospital Admin** | Tenant-wide | View usage, request increases |
| **Platform Admin** | System-wide | Set quotas, approve overrides |

## In Scope (This Interview)
- Quota check and enforcement API
- Reservation/release mechanism
- Usage tracking
- Admin override flow

## Out of Scope
- Billing integration
- Usage analytics and forecasting
- Self-service quota purchase

---

# Step 3: High-Level Architecture (10 min)

## Core Insight: Reserve-Commit-Release Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QUOTA SERVICE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                      Quota API                                    │      │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │      │
│   │  │   Check     │  │  Reserve    │  │  Release    │              │      │
│   │  │   Quota     │  │  Quota      │  │  Quota      │              │      │
│   │  └─────────────┘  └─────────────┘  └─────────────┘              │      │
│   └──────────────────────────────────────────────────────────────────┘      │
│                                    │                                         │
│                                    ▼                                         │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                    Quota Enforcement Engine                       │      │
│   │                                                                   │      │
│   │   - Atomic check-and-reserve                                     │      │
│   │   - Soft limit warnings                                          │      │
│   │   - Hard limit blocking                                          │      │
│   │   - Override handling                                            │      │
│   └──────────────────────────────────────────────────────────────────┘      │
│                                    │                                         │
│              ┌─────────────────────┼─────────────────────┐                  │
│              ▼                     ▼                     ▼                  │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│   │  Quota Limits   │   │  Current Usage  │   │  Reservations   │          │
│   │  (Config DB)    │   │  (Fast Store)   │   │  (Fast Store)   │          │
│   └─────────────────┘   └─────────────────┘   └─────────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Called by
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE SERVICES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐                │
│   │  Environment  │   │    User       │   │   Storage     │                │
│   │  Provisioner  │   │   Service     │   │   Service     │                │
│   │               │   │               │   │               │                │
│   │  1. Reserve   │   │  1. Reserve   │   │  1. Reserve   │                │
│   │  2. Create    │   │  2. Create    │   │  2. Allocate  │                │
│   │  3. Commit    │   │  3. Commit    │   │  3. Commit    │                │
│   └───────────────┘   └───────────────┘   └───────────────┘                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility |
|-----------|---------------|
| **Quota API** | REST interface for check/reserve/release operations |
| **Enforcement Engine** | Business logic for limit checking, warnings, blocking |
| **Quota Limits Store** | Tenant quota configurations (PostgreSQL) |
| **Usage Store** | Current usage counters (Redis for speed) |
| **Reservations Store** | Pending reservations with TTL (Redis) |

---

# Step 4: API Design (5 min)

## Internal APIs (Service-to-Service)

```
POST   /api/v1/quota/check             # Check if action would exceed quota
POST   /api/v1/quota/reserve           # Reserve quota (returns reservation_id)
POST   /api/v1/quota/commit            # Confirm reservation, update usage
POST   /api/v1/quota/release           # Cancel reservation
GET    /api/v1/quota/usage/{tenant_id} # Get current usage summary
```

## Admin APIs

```
GET    /api/v1/admin/quota/{tenant_id}              # Get tenant quotas
PUT    /api/v1/admin/quota/{tenant_id}              # Update tenant quotas
POST   /api/v1/admin/quota/{tenant_id}/override     # Temporary override
GET    /api/v1/admin/quota/{tenant_id}/history      # Quota change audit log
```

## Example: Reserve Quota Request

```json
POST /api/v1/quota/reserve
{
  "tenant_id": "hospital_123",
  "resource_type": "ENVIRONMENT",
  "quantity": 1,
  "operation_id": "provision_job_abc",   // Idempotency key
  "ttl_seconds": 3600                    // Auto-release if not committed
}
```

## Example: Reserve Quota Response (Success)

```json
{
  "reservation_id": "res_xyz789",
  "status": "RESERVED",
  "resource_type": "ENVIRONMENT",
  "quantity": 1,
  "current_usage": 5,
  "reserved": 2,           // Including this reservation
  "limit": 10,
  "available_after": 3,    // 10 - 5 - 2 = 3
  "expires_at": "2024-03-15T11:00:00Z",
  "warnings": []
}
```

## Example: Reserve Quota Response (Soft Limit)

```json
{
  "reservation_id": "res_xyz789",
  "status": "RESERVED",
  "resource_type": "ENVIRONMENT",
  "quantity": 1,
  "current_usage": 8,
  "limit": 10,
  "warnings": [
    {
      "code": "SOFT_LIMIT_EXCEEDED",
      "message": "Usage at 90% of quota. Consider requesting increase.",
      "threshold": 80
    }
  ]
}
```

## Example: Reserve Quota Response (Hard Limit)

```json
{
  "status": "REJECTED",
  "error": {
    "code": "QUOTA_EXCEEDED",
    "message": "Environment quota exceeded. Current: 10, Limit: 10, Requested: 1",
    "resource_type": "ENVIRONMENT",
    "current_usage": 10,
    "limit": 10,
    "requested": 1
  },
  "suggestion": "Contact administrator to request quota increase"
}
```

---

# Step 5: Data Model (5 min)

## Entity Relationships

```
┌─────────────────┐       ┌─────────────────────┐
│     Tenant      │1─────*│    QuotaLimit       │
├─────────────────┤       ├─────────────────────┤
│ tenant_id       │       │ tenant_id (FK)      │
│ name            │       │ resource_type       │
│ tier            │       │ hard_limit          │
│ ...             │       │ soft_limit_percent  │
└─────────────────┘       │ override_limit      │
                          │ override_expires_at │
                          └─────────────────────┘

┌─────────────────────┐       ┌─────────────────────┐
│   QuotaUsage        │       │   QuotaReservation  │
│   (Redis)           │       │   (Redis + TTL)     │
├─────────────────────┤       ├─────────────────────┤
│ tenant_id           │       │ reservation_id      │
│ resource_type       │       │ tenant_id           │
│ current_count       │       │ resource_type       │
│ last_updated        │       │ quantity            │
└─────────────────────┘       │ operation_id        │
                              │ expires_at          │
                              │ created_at          │
                              └─────────────────────┘
```

## Resource Types

```python
class ResourceType(Enum):
    ENVIRONMENT = "ENVIRONMENT"       # Count of environments
    STORAGE_GB = "STORAGE_GB"         # Total storage in GB
    USER = "USER"                     # Count of users
    API_CALLS_DAILY = "API_CALLS"     # Daily API call limit
```

## Default Quotas by Tier

```json
{
  "STARTER": {
    "ENVIRONMENT": {"hard_limit": 3, "soft_limit_percent": 80},
    "STORAGE_GB": {"hard_limit": 100, "soft_limit_percent": 80},
    "USER": {"hard_limit": 50, "soft_limit_percent": 80}
  },
  "PROFESSIONAL": {
    "ENVIRONMENT": {"hard_limit": 10, "soft_limit_percent": 80},
    "STORAGE_GB": {"hard_limit": 500, "soft_limit_percent": 80},
    "USER": {"hard_limit": 200, "soft_limit_percent": 80}
  },
  "ENTERPRISE": {
    "ENVIRONMENT": {"hard_limit": 50, "soft_limit_percent": 90},
    "STORAGE_GB": {"hard_limit": 5000, "soft_limit_percent": 90},
    "USER": {"hard_limit": 1000, "soft_limit_percent": 90}
  }
}
```

## Schema & Indexes

**quota_limits** (PostgreSQL)
| Column | Type | Notes |
|--------|------|-------|
| tenant_id | VARCHAR | PK (composite) |
| resource_type | VARCHAR | PK (composite) — ENVIRONMENT, STORAGE_GB, USER |
| hard_limit | BIGINT | |
| soft_limit_percent | INT | Default 80 |
| override_limit | BIGINT | Temporary override |
| override_expires_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |
| updated_by | VARCHAR | |

**quota_audit_log** (PostgreSQL)
| Column | Type | Notes |
|--------|------|-------|
| id | SERIAL | PK |
| tenant_id | VARCHAR | |
| resource_type | VARCHAR | |
| action | VARCHAR | SET_LIMIT, OVERRIDE, USAGE_CHANGE |
| old_value, new_value | BIGINT | |
| reason | TEXT | |
| performed_by | VARCHAR | |
| performed_at | TIMESTAMP | |

**Indexes**
- `quota_audit_log(tenant_id, performed_at)` — query audit history by tenant

## Redis Data Structures

| Key Pattern | Type | Purpose |
|-------------|------|---------|
| `quota:usage:{tenant_id}:{resource_type}` | Integer | Current usage count |
| `quota:reservation:{reservation_id}` | String (JSON) + TTL | Reservation details, auto-expires |
| `quota:reservations:{tenant_id}:{resource_type}` | Set | Index of active reservation IDs for summing |

---

# Step 6: Deep Dive - Core Flows (10 min)

## 6.1 Reserve-Commit-Release Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROVISIONING SERVICE                                 │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        │ 1. Reserve quota before starting work
        ▼
┌─────────────────────────────────────┐
│ POST /quota/reserve                 │
│   resource: ENVIRONMENT             │
│   quantity: 1                       │──────┐
│   operation_id: "prov_123"          │      │
└──────────────────┬──────────────────┘      │
                   │                          │
                   ▼                          │
        ┌──────────────────┐                  │
        │ Reservation OK   │                  │ If rejected: fail fast,
        │ res_id: res_xyz  │                  │ return error to user
        └────────┬─────────┘                  │
                 │                            │
                 │ 2. Do the actual work      │
                 ▼                            │
┌─────────────────────────────────────┐      │
│ Create environment in downstream    │      │
│ (This is the slow, risky part)      │      │
└──────────────────┬──────────────────┘      │
                   │                          │
         ┌─────────┴─────────┐                │
         │                   │                │
    SUCCESS              FAILURE              │
         │                   │                │
         ▼                   ▼                │
┌─────────────────┐  ┌─────────────────┐     │
│ 3a. Commit      │  │ 3b. Release     │     │
│ POST /commit    │  │ POST /release   │◀────┘
│ res_id: res_xyz │  │ res_id: res_xyz │
└────────┬────────┘  └────────┬────────┘
         │                    │
         ▼                    ▼
┌─────────────────┐  ┌─────────────────┐
│ Usage count     │  │ Reservation     │
│ incremented     │  │ deleted         │
│ Reservation     │  │ Usage unchanged │
│ deleted         │  └─────────────────┘
└─────────────────┘
```

## 6.2 Atomic Check-and-Reserve (Redis Lua Script)

```lua
-- KEYS[1] = quota:usage:{tenant}:{resource}
-- KEYS[2] = quota:reservations:{tenant}:{resource}
-- KEYS[3] = quota:reservation:{reservation_id}
-- ARGV[1] = hard_limit
-- ARGV[2] = quantity
-- ARGV[3] = reservation_id
-- ARGV[4] = ttl_seconds
-- ARGV[5] = reservation_json

local current_usage = tonumber(redis.call('GET', KEYS[1]) or '0')
local reservations = redis.call('SMEMBERS', KEYS[2])
local reserved_total = 0

for _, res_id in ipairs(reservations) do
    local res_data = redis.call('GET', 'quota:reservation:' .. res_id)
    if res_data then
        local res = cjson.decode(res_data)
        reserved_total = reserved_total + res.quantity
    end
end

local total_committed = current_usage + reserved_total + tonumber(ARGV[2])

if total_committed > tonumber(ARGV[1]) then
    return {err = 'QUOTA_EXCEEDED', current = current_usage, reserved = reserved_total}
end

-- Create reservation
redis.call('SETEX', KEYS[3], ARGV[4], ARGV[5])
redis.call('SADD', KEYS[2], ARGV[3])

return {ok = true, current = current_usage, reserved = reserved_total + tonumber(ARGV[2])}
```

## 6.3 Handling Concurrent Requests

```
Time ──────────────────────────────────────────────────────────────▶

Request A                          Request B
    │                                  │
    ▼                                  ▼
┌─────────────────┐              ┌─────────────────┐
│ Reserve 1 env   │              │ Reserve 1 env   │
│ (Lua atomic)    │              │ (Lua atomic)    │
└────────┬────────┘              └────────┬────────┘
         │                                │
         │    Tenant quota: 10            │
         │    Current usage: 9            │
         │    Reservations: 0             │
         │                                │
         ▼                                │
┌─────────────────┐                       │
│ Check: 9+0+1≤10 │                       │
│ ✓ RESERVED      │                       │
│ res_id: A       │                       │
└─────────────────┘                       │
                                          ▼
                               ┌─────────────────┐
                               │ Check: 9+1+1≤10 │
                               │ ✗ REJECTED      │ ◀── Sees A's reservation
                               │ (would be 11)   │
                               └─────────────────┘

Result: Only one succeeds, no over-provisioning
```

## 6.4 Reservation Expiry and Cleanup

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     RESERVATION CLEANUP (Background)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Redis TTL handles automatic expiration:                                    │
│                                                                              │
│  1. Reservation created with TTL (e.g., 1 hour)                             │
│     SETEX quota:reservation:res_xyz 3600 {...}                              │
│                                                                              │
│  2. If commit/release not called, key auto-expires                          │
│                                                                              │
│  3. Background cleanup job (every 5 min):                                   │
│     - Scan reservation index sets                                           │
│     - Remove references to expired reservations                             │
│     - Log orphaned reservations for alerting                                │
│                                                                              │
│  ┌─────────────────┐                                                        │
│  │ Cleanup Job     │                                                        │
│  │                 │                                                        │
│  │ FOR each tenant:│                                                        │
│  │   FOR each res in SMEMBERS quota:reservations:{tenant}:{resource}:      │
│  │     IF NOT EXISTS quota:reservation:{res_id}:                           │
│  │       SREM quota:reservations:{tenant}:{resource} {res_id}              │
│  │       LOG "Cleaned up expired reservation {res_id}"                     │
│  └─────────────────┘                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 6.5 Usage Reconciliation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  USAGE RECONCILIATION (Background - Hourly)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Redis counters can drift from reality due to bugs or missed releases.      │
│  Reconciliation ensures eventual consistency.                                │
│                                                                              │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐        │
│  │ 1. Query actual │────▶│ 2. Compare to   │────▶│ 3. If mismatch, │        │
│  │    counts from  │     │    Redis count  │     │    update Redis │        │
│  │    source of    │     │                 │     │    + alert      │        │
│  │    truth (DB)   │     │                 │     │                 │        │
│  └─────────────────┘     └─────────────────┘     └─────────────────┘        │
│                                                                              │
│  Example reconciliation query:                                               │
│                                                                              │
│  SELECT tenant_id, COUNT(*) as actual_count                                 │
│  FROM environments                                                           │
│  WHERE status NOT IN ('DELETED', 'DELETING')                                │
│  GROUP BY tenant_id                                                          │
│                                                                              │
│  Compare with: GET quota:usage:{tenant_id}:ENVIRONMENT                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# Step 7: Scaling & Bottlenecks (5 min)

## Potential Bottlenecks

| Bottleneck | Solution |
|------------|----------|
| **Redis single point of failure** | Redis Cluster with replicas |
| **Hot tenant (many requests)** | Per-tenant rate limiting on quota API |
| **Lua script blocking** | Keep scripts simple, < 1ms execution |
| **Quota DB reads** | Cache quota limits in Redis (5 min TTL) |
| **Reconciliation at scale** | Partition by tenant, run in parallel |

## High Availability Design

```
┌─────────────────────────────────────────────────────────────────┐
│                      QUOTA SERVICE HA                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐     ┌─────────────────┐                    │
│  │  Quota API      │     │  Quota API      │  (Stateless,       │
│  │  Instance 1     │     │  Instance 2     │   load balanced)   │
│  └────────┬────────┘     └────────┬────────┘                    │
│           │                       │                              │
│           └───────────┬───────────┘                              │
│                       ▼                                          │
│           ┌─────────────────────┐                               │
│           │   Redis Cluster     │                               │
│           │                     │                               │
│           │  ┌─────┐  ┌─────┐  │                               │
│           │  │ M1  │  │ M2  │  │  (3 masters, each with        │
│           │  │ R1a │  │ R2a │  │   1-2 replicas)               │
│           │  │ R1b │  │ R2b │  │                               │
│           │  └─────┘  └─────┘  │                               │
│           └─────────────────────┘                               │
│                       │                                          │
│                       ▼                                          │
│           ┌─────────────────────┐                               │
│           │   PostgreSQL        │                               │
│           │   (Quota limits,    │                               │
│           │    audit logs)      │                               │
│           └─────────────────────┘                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Numbers to Know

| Metric | Target |
|--------|--------|
| Quota check latency (P99) | < 10ms |
| Reserve operation latency | < 20ms |
| Throughput | 10K checks/sec |
| Redis memory per tenant | ~1KB |
| Max concurrent reservations | 100K |

---

# Step 8: Tradeoffs & Wrap-Up (5 min)

## Key Design Tradeoffs

### 1. Consistency Model

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Strong (Redis atomic)** | No over-provision | Slightly slower | **Use for reserves** |
| **Eventual (async update)** | Faster | Risk of drift | Use for usage display only |

### 2. Reservation TTL

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Short (5 min)** | Less wasted quota | Fails long operations | Too risky |
| **Medium (1 hour)** | Balances both | Some waste | **Use this** |
| **Long (24 hours)** | Never fails | Lots of wasted quota | Only for very long ops |

### 3. Storage Backend

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Redis only** | Fastest | Durability concerns | Reservations only |
| **PostgreSQL only** | Durable | Too slow for hot path | Limits + audit only |
| **Redis + PostgreSQL** | Best of both | Complexity | **Use this** |

## Failure Modes and Handling

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Redis down | Can't reserve | Fail closed (reject all), alert |
| Reservation leak | Quota slowly fills | TTL auto-cleanup + reconciliation |
| Counter drift | Inaccurate display | Hourly reconciliation from source of truth |
| Commit lost | Resource exists but quota not counted | Reconciliation catches it |

---

# Interviewer Follow-Up Questions (Prepare For)

1. **"Two requests come in at exact same time. How do you prevent both from succeeding?"**
   - Redis Lua scripts are atomic - only one executes at a time per key
   - First request reserves, second sees the reservation and is rejected
   - No locks needed, Redis handles serialization

2. **"Provisioning succeeds but commit call fails. What happens?"**
   - Resource exists, quota not incremented
   - Reservation will expire (TTL), freeing the "reserved" quota
   - Reconciliation job will detect mismatch and fix usage counter
   - Net effect: works correctly, just slightly delayed accuracy

3. **"Customer is at 10/10 environments, deletes one, immediately tries to create. What if delete is still in progress?"**
   - Don't decrement usage until delete is fully complete
   - Customer sees 10/10 until delete commits
   - Alternative: allow "soft" over-provision during transitions (not recommended for healthcare)

4. **"How do you handle quota for storage, which changes continuously?"**
   - Don't reserve/commit for every byte
   - Reserve in chunks (e.g., 10GB increments)
   - Background job updates actual usage periodically
   - Alert when approaching limit rather than blocking writes mid-operation

---

# Summary: What Makes This Design Strong

1. **Atomic operations**: Lua scripts prevent race conditions
2. **Reserve-commit pattern**: Handles long-running operations safely
3. **Self-healing**: TTL expiry + reconciliation handles edge cases
4. **Fast path**: Redis for hot operations, PostgreSQL for durability
5. **Observable**: Audit log for all quota changes
