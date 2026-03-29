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

Let's follow what happens when a user tries to create an environment:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 1: User clicks "Create Environment"                                   │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        │  User: "I want to create a new staging environment"
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 2: Environment Service receives request                               │
│                                                                              │
│  Before creating anything, it asks: "Does this hospital have quota left?"  │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        │  Environment Service calls Quota Service
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 3: Quota Service checks and RESERVES                                  │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Check: Hospital ABC has 4/5 environments                            │   │
│  │        4 + 1 = 5, which is ≤ 5                                      │   │
│  │        ✓ OK! Reserve 1 slot.                                        │   │
│  │                                                                      │   │
│  │ Now: 4 used + 1 reserved = 5 (no one else can grab this slot)      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Returns: reservation_id = "res_123"                                        │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        │  Quota reserved! Now safe to create.
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 4: Environment Service creates the environment                        │
│                                                                              │
│  This is the slow part (create database, deploy apps, etc.)                 │
│  Takes 5-10 minutes. Might fail!                                            │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        │  Success or failure?
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 5a: SUCCESS → Commit the reservation                                  │
│                                                                              │
│  Environment Service: "Creation worked! Commit res_123"                     │
│  Quota Service: Converts reservation to actual usage.                       │
│                 Now: 5 used, 0 reserved.                                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 5b: FAILURE → Release the reservation                                 │
│                                                                              │
│  Environment Service: "Creation failed! Release res_123"                    │
│  Quota Service: Deletes reservation, slot is free again.                   │
│                 Now: 4 used, 0 reserved.                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why reserve first?**

The environment creation takes minutes. Without reservation:
- User A starts creating (4/5 used)
- User B starts creating (still sees 4/5, allowed!)
- Both finish → 6/5 environments = over quota!

With reservation:
- User A reserves (4 used + 1 reserved = 5)
- User B tries to reserve → blocked (5 = limit)
- Only User A's environment is created → correct!

---

## What Happens Inside Quota Service?

When Environment Service calls `POST /quota/reserve`:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              QUOTA SERVICE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Input: "Hospital ABC wants to reserve 1 environment"                       │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ STEP 1: What's the limit?                                              │ │
│  │                                                                         │ │
│  │         Look up in database: Hospital ABC → max 5 environments         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                        │
│                                     ▼                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ STEP 2: What's currently used + reserved?                              │ │
│  │                                                                         │ │
│  │         Current usage:      4 environments                             │ │
│  │         Pending reservations: 0                                         │ │
│  │         ─────────────────────────                                       │ │
│  │         Total committed:    4                                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                        │
│                                     ▼                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ STEP 3: Is there room?                                                 │ │
│  │                                                                         │ │
│  │         4 (used) + 1 (requested) = 5                                   │ │
│  │         5 ≤ 5 (limit)?  ✓ YES                                          │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                        │
│                                     ▼                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ STEP 4: Create reservation                                             │ │
│  │                                                                         │ │
│  │         Save: "res_abc = 1 environment, expires in 1 hour"             │ │
│  │         Return: { reservation_id: "res_abc", status: "reserved" }      │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**After reservation, the state is:**
```
Hospital ABC:
  - Limit:        5
  - Used:         4  (actual environments that exist)
  - Reserved:     1  (being created right now)
  - Available:    0  (no one else can create until this finishes)
```

**When provisioning completes:**
```
POST /quota/commit (reservation_id: "res_abc")

  → Delete reservation (reserved: 1 → 0)
  → Increment usage (used: 4 → 5)
  → Done!
```

**If provisioning fails:**
```
POST /quota/release (reservation_id: "res_abc")

  → Delete reservation (reserved: 1 → 0)
  → Usage stays the same (used: 4)
  → Slot is free for someone else
```

---

## Why Separate Services?

**Q: Why not put quota logic inside Environment Service?**

If we embed quota checks in Environment Service:

```
┌─────────────────────────────────────────────────────────────────┐
│                     EMBEDDED APPROACH (Bad)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Environment Svc │  │   User Service  │  │ Storage Service │  │
│  │                 │  │                 │  │                 │  │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │
│  │ │Quota logic  │ │  │ │Quota logic  │ │  │ │Quota logic  │ │  │
│  │ │(duplicate!) │ │  │ │(duplicate!) │ │  │ │(duplicate!) │ │  │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
│  Problem: Same quota logic copy-pasted 3 times!                 │
│  - Bug fix? Update 3 places.                                    │
│  - New quota type? Update 3 places.                             │
│  - Different teams? Inconsistent implementations.               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

With a separate Quota Service:

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEPARATE SERVICE (Good)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Environment Svc │  │   User Service  │  │ Storage Service │  │
│  │                 │  │                 │  │                 │  │
│  │ "reserve 1 env" │  │ "reserve 1 user"│  │ "reserve 10 GB" │  │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  │
│           │                    │                    │           │
│           └────────────────────┼────────────────────┘           │
│                                ▼                                 │
│                    ┌─────────────────────┐                      │
│                    │   QUOTA SERVICE     │                      │
│                    │                     │                      │
│                    │ One place for all   │                      │
│                    │ quota logic         │                      │
│                    └─────────────────────┘                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**What does "reserve 1 user" mean?**

Hospitals have limits on different resources, not just environments:

```
Hospital ABC's Quota Limits:
┌─────────────────────────────────────┐
│  Resource       │ Limit │ Used     │
├─────────────────────────────────────┤
│  Environments   │   5   │   4      │  ← "Can create 1 more environment"
│  Users          │  100  │  87      │  ← "Can add 13 more users"
│  Storage (GB)   │  500  │  340     │  ← "Can use 160 more GB"
└─────────────────────────────────────┘
```

Example - Admin adds a new doctor to the hospital:

```
1. Admin: "Add Dr. Smith to our hospital"
           │
           ▼
2. User Service: "Let me check if they have room"
           │
           ▼
3. Quota Service: "Hospital ABC: 87/100 users. 87+1=88 ≤ 100? ✓ OK"
           │
           ▼
4. User Service: Creates Dr. Smith's account
           │
           ▼
5. Quota Service: Commit → Now 88/100 users
```

If they were at 100/100 users, step 3 would reject: "User quota exceeded."

**The rule:** If multiple services need the same logic, extract it into its own service.

---

**Summary:** Other services just call "Quota Service" — one service, simple name.

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

| Type | What it tracks |
|------|---------------|
| ENVIRONMENT | Count of environments |
| STORAGE_GB | Total storage in GB |
| USER | Count of users |
| API_CALLS_DAILY | Daily API call limit |

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

## 6.2 Why "Atomic" Matters

**The problem:** Check and reserve must happen as ONE operation, not two.

```
❌ BAD: Two separate operations

  Request A                         Request B
      │                                 │
      ▼                                 ▼
  Read: "4/5 used"                  Read: "4/5 used"
      │                                 │
      ▼                                 ▼
  "Room for 1? Yes!"                "Room for 1? Yes!"
      │                                 │
      ▼                                 ▼
  Reserve 1                         Reserve 1
      │                                 │
      ▼                                 ▼
  Now 5/5                           Now 6/5 ← OVER QUOTA!

Both read "4" before either reserved → both succeed → bad!
```

```
✓ GOOD: One atomic operation

  Request A                         Request B
      │                                 │
      ▼                                 │
  [Check AND Reserve]                   │  ← Redis locks this
  "4/5? Room for 1? Yes. Reserved."     │
  Now: 5/5                              │
      │                                 ▼
      │                           [Check AND Reserve]
      │                           "5/5? Room for 1? No."
      │                           REJECTED ✓
```

**How:** Use Redis with atomic operations (all-or-nothing). Redis processes one command at a time, so check+reserve can't be interrupted.

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
