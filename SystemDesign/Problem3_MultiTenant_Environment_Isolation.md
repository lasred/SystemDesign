# Problem 3: Multi-Tenant Environment Management

**Difficulty: Medium** | **Focus: Data Model & Access Control**

## Interviewer Prompt

> "We need to design a platform that manages healthcare environments for hospital customers.
>
> When a hospital subscribes, we create environments for them—things like their EHR instance, patient portal, analytics. Each hospital might have multiple environments: production, staging, dev.
>
> We need to make sure hospitals can't see each other's data, and we need role-based access—org admins, environment admins, regular users.
>
> How would you design the data model and access control for this?"

---

# Step 1: Clarify Requirements (3-5 min)

## Questions to Ask

| Question | Why It Matters | Example Answer |
|----------|----------------|----------------|
| How many hospitals/tenants? | Scale of system | 100-500 hospitals |
| Environments per hospital? | Resource planning | 3-5 (prod, staging, dev) |
| Users per environment? | Auth system scale | 50-200 per environment |
| What roles are needed? | RBAC design | Org admin, env admin, user |
| Isolation level needed? | Architecture | Separate databases per tenant |

## Functional Requirements (Confirmed)

- **Tenant isolation**: Hospital A cannot access Hospital B's data
- **Multi-environment**: Each hospital has prod/staging/dev environments
- **RBAC**: Three role levels with different permissions
- **Environment lifecycle**: Create, update, delete environments

## Non-Functional Requirements

- **Security**: No cross-tenant data leaks
- **Scalability**: Support 500+ hospitals
- **Audit**: Log all access for compliance

---

# Step 2: Define Scope & Actors (2-3 min)

## Core Actors

| Actor | Scope | Key Actions |
|-------|-------|-------------|
| **Hospital Admin** | Tenant-wide | Create/delete environments, manage users |
| **Environment Admin** | Single environment | Configure settings, manage env users |
| **Regular User** | Single environment | Access applications, view data |

## In Scope (This Interview)
- Tenant & environment data model
- RBAC and access control
- Environment provisioning flow

## Out of Scope
- Individual application design (EHR internals)
- Billing system
- Infrastructure provisioning details

---

# Step 3: High-Level Architecture (10 min)

## Core Insight: Control Plane / Data Plane Separation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CONTROL PLANE (Shared)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐    ┌─────────────────────┐    ┌──────────────────┐       │
│   │   Clients    │───▶│   Auth Service      │───▶│   Audit Service  │       │
│   │              │    │   (AuthN/AuthZ)     │    │                  │       │
│   └──────────────┘    └───────────┬─────────┘    └──────────────────┘       │
│                                   │                                          │
│                                   ▼                                          │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                    Tenant Management Service                       │     │
│   │              (Hospitals, Users, Roles, Permissions)               │     │
│   └───────────────────────────────┬───────────────────────────────────┘     │
│                                   │                                          │
│                                   ▼                                          │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                 Environment Management Service                     │     │
│   │                  (Environment CRUD, Lifecycle)                    │     │
│   └───────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                         Manages    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     DATA PLANE (Isolated per Hospital)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │   Hospital A            │  │   Hospital B            │    ...            │
│  │   ┌─────────────────┐   │  │   ┌─────────────────┐   │                   │
│  │   │ App Services    │   │  │   │ App Services    │   │                   │
│  │   │ (EHR, Portal)   │   │  │   │ (EHR, Portal)   │   │                   │
│  │   └─────────────────┘   │  │   └─────────────────┘   │                   │
│  │   ┌────────┬────────┐   │  │   ┌────────┬────────┐   │                   │
│  │   │Database│Storage │   │  │   │Database│Storage │   │                   │
│  │   └────────┴────────┘   │  │   └────────┴────────┘   │                   │
│  └─────────────────────────┘  └─────────────────────────┘                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility |
|-----------|---------------|
| **Auth Service** | JWT issuance, token validation |
| **Tenant Management** | Hospital CRUD, user management, role assignment |
| **Environment Management** | Environment lifecycle (create, update, delete) |
| **Audit Service** | Log all access and changes |

---

# Step 4: API Design (5 min)

## Tenant Management APIs

```
POST   /api/v1/tenants                    # Create new hospital
GET    /api/v1/tenants/{tenant_id}        # Get hospital details
DELETE /api/v1/tenants/{tenant_id}        # Delete hospital (soft delete)

POST   /api/v1/tenants/{tenant_id}/users  # Add user to hospital
PUT    /api/v1/tenants/{tenant_id}/users/{user_id}/role  # Update role
```

## Environment Management APIs

```
POST   /api/v1/tenants/{tenant_id}/environments           # Create environment
GET    /api/v1/tenants/{tenant_id}/environments           # List environments
GET    /api/v1/tenants/{tenant_id}/environments/{env_id}  # Get env details
DELETE /api/v1/tenants/{tenant_id}/environments/{env_id}  # Delete environment
```

## Example: Create Environment Request

```json
POST /api/v1/tenants/hospital_123/environments
{
  "name": "production",
  "type": "prod",
  "services": ["ehr", "patient-portal"]
}
```

## Example: Response

```json
{
  "env_id": "env_abc123",
  "tenant_id": "hospital_123",
  "name": "production",
  "type": "prod",
  "status": "provisioning",
  "created_at": "2024-01-15T10:00:00Z"
}
```

---

# Step 5: Data Model (5 min)

## Entity Relationships

```
┌─────────────┐         ┌─────────────────┐         ┌─────────────┐
│   Tenant    │────────▶│   Environment   │         │    User     │
│  (Hospital) │   1:N   │                 │         │             │
├─────────────┤         ├─────────────────┤         ├─────────────┤
│ tenant_id   │         │ env_id          │         │ user_id     │
│ name        │         │ tenant_id (FK)  │         │ tenant_id   │
│ status      │         │ name            │         │ email       │
│ created_at  │         │ type            │         │ role        │
└──────┬──────┘         │ status          │         └─────────────┘
       │                │ created_at      │                │
       │                └─────────────────┘                │
       │                         ▲                         │
       │ 1:N                     │ N:M                     │
       │                         │                         │
       └─────────────────────────┼─────────────────────────┘
                                 │
                    ┌────────────┴───────────┐
                    │    user_env_access     │
                    ├────────────────────────┤
                    │ user_id (FK)           │
                    │ env_id (FK)            │
                    │ granted_at             │
                    └────────────────────────┘
```

**Relationships:**
- **Tenant → Environment (1:N)**: One hospital has many environments (prod, staging, dev)
- **Tenant → User (1:N)**: One hospital has many users
- **User ↔ Environment (N:M)**: A user can access multiple environments, and an environment can have multiple users. The `user_env_access` table stores which users have access to which environments.

## Schema & Indexes

**tenants**
| Column | Type | Notes |
|--------|------|-------|
| tenant_id | VARCHAR | PK |
| name | VARCHAR | Hospital name |
| status | VARCHAR | ACTIVE, SUSPENDED, DELETED |
| created_at | TIMESTAMP | |

**environments**
| Column | Type | Notes |
|--------|------|-------|
| env_id | VARCHAR | PK |
| tenant_id | VARCHAR | FK to tenants |
| name | VARCHAR | "production", "staging", etc. |
| type | VARCHAR | prod, staging, dev |
| status | VARCHAR | CREATING, ACTIVE, DELETING |
| created_at | TIMESTAMP | |

**users**
| Column | Type | Notes |
|--------|------|-------|
| user_id | VARCHAR | PK |
| tenant_id | VARCHAR | FK to tenants |
| email | VARCHAR | |
| role | VARCHAR | hospital_admin, env_admin, user |

**user_env_access**
| Column | Type | Notes |
|--------|------|-------|
| user_id | VARCHAR | PK (composite) |
| env_id | VARCHAR | PK (composite) |
| granted_at | TIMESTAMP | |

**Indexes**
- `environments(tenant_id)` — list environments for a hospital
- `users(tenant_id)` — list users for a hospital
- `user_env_access(env_id)` — list users with access to an environment

## JWT Token Structure

```json
{
  "sub": "user_123",
  "tenant_id": "hospital_123",
  "role": "environment_admin",
  "env_access": ["env_prod", "env_staging"],
  "exp": 1234567890
}
```

---

# Step 6: Deep Dive - Core Flows (10 min)

## 6.1 Environment Provisioning Flow

```
Hospital Admin clicks "Create Environment"
        │
        ▼
┌─────────────────────────────────────┐
│ 1. API Gateway                      │
│    - Validate JWT                   │
│    - Check: role = hospital_admin?  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 2. Environment Service              │
│    - Validate request               │
│    - Check tenant quota             │
│    - Create env record (CREATING)   │
│    - Queue provisioning job         │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 3. Provisioner (async)              │
│    - Create database                │
│    - Deploy applications            │
│    - Update status → ACTIVE         │
│    - Notify admin                   │
└─────────────────────────────────────┘
```

## 6.2 Request Authorization Flow

```
Request: GET /api/environments/env_prod_123/patients
Headers: Authorization: Bearer <JWT>
        │
        ▼
┌─────────────────────────────────────┐
│ 1. Validate JWT signature & expiry  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 2. Extract claims from JWT:         │
│    - tenant_id: hospital_123        │
│    - env_access: [env_prod_123]     │
│    - role: environment_admin        │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 3. Authorization checks:            │
│    - Is env_prod_123 in env_access? │
│    - Does role allow this action?   │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 4. Forward to environment's         │
│    data plane                       │
└─────────────────────────────────────┘
```

## 6.3 RBAC Permission Matrix

| Action | Hospital Admin | Env Admin | User |
|--------|---------------|-----------|------|
| Create environment | ✓ | ✗ | ✗ |
| Delete environment | ✓ | ✗ | ✗ |
| Configure environment | ✓ | ✓ | ✗ |
| Add users to environment | ✓ | ✓ | ✗ |
| Access environment data | ✓ | ✓ | ✓ |
| View audit logs | ✓ | ✓ | ✗ |

---

# Step 7: Scaling & Bottlenecks (5 min)

## Potential Bottlenecks

### 1. Auth Service Overload

**The Problem:**
Every API request needs to validate the JWT token. If you have 10,000 requests/second, that's 10,000 calls to your auth service.

**Current approach (JWT):**
```
Request comes in with JWT
        │
        ▼
┌─────────────────────────────┐
│ Auth Service validates JWT: │
│ 1. Check signature (crypto) │
│ 2. Check expiry             │
│ 3. Extract claims           │
└─────────────────────────────┘
```

**Solutions:**

| Solution | How It Works |
|----------|--------------|
| **JWT signature validation locally** | JWT is self-contained. Any service can validate the signature using the public key—no need to call auth service. Just distribute the public key to all services. |
| **Cache validated tokens** | After validating a JWT once, cache the result (e.g., in Redis) for a few minutes. Next request with same token skips validation. |
| **Distributed auth** | Instead of one auth service, run multiple instances behind a load balancer. Each instance can validate independently since JWT validation is stateless. |

```
BEFORE: All requests hit one auth service
┌─────────┐
│ Service │──────▶ Auth Service (bottleneck!)
└─────────┘

AFTER: Each service validates JWT locally
┌─────────┐
│ Service │──▶ Local JWT validation (using cached public key)
└─────────┘    No network call needed!
```

---

### 2. Control Plane Database Overload

**The Problem:**
All tenant/environment queries hit one database. Reads are much more frequent than writes (e.g., "get environment details" vs "create environment").

**Solutions:**

| Solution | How It Works |
|----------|--------------|
| **Read replicas** | Create copies of the database that stay in sync. Route read queries (SELECT) to replicas, write queries (INSERT/UPDATE) to primary. Spreads the load. |
| **Connection pooling** | Instead of each request opening a new DB connection (expensive), reuse a pool of connections. Tools: PgBouncer, HikariCP. |

```
BEFORE: All queries hit primary DB
┌─────────┐
│ Service │──────▶ Primary DB (bottleneck!)
└─────────┘

AFTER: Reads go to replicas
┌─────────┐      ┌─────────────┐
│ Service │─────▶│ Primary DB  │◀── writes only
└─────────┘      └──────┬──────┘
     │                  │ replicates
     │           ┌──────▼──────┐
     └──────────▶│ Read Replica│◀── reads (most traffic)
                 └─────────────┘
```

---

### 3. Provisioning Backlog

**The Problem:**
Creating an environment takes 5-10 minutes. If 100 hospitals request environments at once, a single provisioner gets backed up.

**Solutions:**

| Solution | How It Works |
|----------|--------------|
| **Multiple workers** | Run N provisioner instances. Each pulls jobs from the same queue and works in parallel. |
| **Priority queues** | Paid/enterprise tenants get a high-priority queue that's processed first. Free tier waits longer. |

```
BEFORE: One worker, jobs pile up
Queue: [job1, job2, job3, job4, job5...]
              │
              ▼
        ┌──────────┐
        │ Worker 1 │  (slow!)
        └──────────┘

AFTER: Multiple workers process in parallel
Queue: [job1, job2, job3, job4, job5...]
         │     │     │
         ▼     ▼     ▼
      ┌────┐┌────┐┌────┐
      │ W1 ││ W2 ││ W3 │  (3x faster)
      └────┘└────┘└────┘
```

---

## Numbers to Know

| Metric | Target |
|--------|--------|
| Environment provisioning | < 10 min |
| API latency (P99) | < 200ms |
| Auth token validation | < 10ms |
| Max environments per tenant | 10 |

---

# Step 8: Tradeoffs & Wrap-Up (5 min)

## Key Design Tradeoffs

### 1. Tenant Isolation Strategy

**The Question:** How do we prevent Hospital A from seeing Hospital B's data?

---

**Option A: Shared DB + tenant_id column**

All tenants share one database. Every table has a `tenant_id` column, and every query must include `WHERE tenant_id = ?`.

```
┌─────────────────────────────────────────┐
│              Shared Database            │
├─────────────────────────────────────────┤
│  patients table                         │
│  ┌─────────┬───────────┬──────────┐    │
│  │tenant_id│ patient_id│  name    │    │
│  ├─────────┼───────────┼──────────┤    │
│  │ hosp_A  │  p_001    │  Alice   │    │
│  │ hosp_B  │  p_002    │  Bob     │ ◀── All mixed together!
│  │ hosp_A  │  p_003    │  Carol   │    │
│  └─────────┴───────────┴──────────┘    │
└─────────────────────────────────────────┘

Query: SELECT * FROM patients WHERE tenant_id = 'hosp_A'

⚠️  DANGER: If developer forgets WHERE clause, data leaks!
    SELECT * FROM patients  -- Returns ALL hospitals' data!
```

**Verdict:** Never for healthcare. One bug = HIPAA violation.

---

**Option B: Separate schema per tenant**

One database, but each tenant gets their own schema (namespace). Tables are isolated.

```
┌─────────────────────────────────────────┐
│              Shared Database            │
├─────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────┐│
│  │ schema: hosp_A  │ │ schema: hosp_B  ││
│  │ ┌─────────────┐ │ │ ┌─────────────┐ ││
│  │ │  patients   │ │ │ │  patients   │ ││
│  │ │  Alice      │ │ │ │  Bob        │ ││
│  │ │  Carol      │ │ │ └─────────────┘ ││
│  │ └─────────────┘ │ │                 ││
│  └─────────────────┘ └─────────────────┘│
└─────────────────────────────────────────┘

Query: SET search_path TO hosp_A; SELECT * FROM patients;
       -- Only sees hosp_A's patients
```

**Pros:** Better isolation, shared infrastructure
**Cons:** Schema migrations are complex (must update N schemas)

---

**Option C: Separate database per tenant** ✅ Recommended

Each tenant gets their own database instance. Complete isolation.

```
┌─────────────────┐     ┌─────────────────┐
│  hosp_A DB      │     │  hosp_B DB      │
├─────────────────┤     ├─────────────────┤
│  patients       │     │  patients       │
│  ┌───────────┐  │     │  ┌───────────┐  │
│  │ Alice     │  │     │  │ Bob       │  │
│  │ Carol     │  │     │  └───────────┘  │
│  └───────────┘  │     │                 │
└─────────────────┘     └─────────────────┘

Connection string per tenant:
  hosp_A → postgres://hosp_a_db:5432/patients
  hosp_B → postgres://hosp_b_db:5432/patients
```

**Pros:** Impossible to leak data across tenants (different servers)
**Cons:** Higher cost (more DB instances), more operational overhead

**Verdict:** Use this for healthcare. Cost is worth the security.

---

### 2. RBAC Storage: JWT vs Database Lookup

**The Question:** Where do we store the user's role and permissions?

---

**Option A: Roles embedded in JWT** ✅ Recommended

When user logs in, their role is baked into the JWT token.

```
┌─────────────────────────────────────────────────────┐
│  JWT Token (stored in browser, sent with requests)  │
├─────────────────────────────────────────────────────┤
│  {                                                  │
│    "sub": "user_123",                               │
│    "tenant_id": "hospital_123",                     │
│    "role": "environment_admin",    ◀── Role is HERE │
│    "exp": 1234567890                                │
│  }                                                  │
└─────────────────────────────────────────────────────┘

Authorization check (no DB needed):
  1. Parse JWT
  2. Read role from claims
  3. Check: does "environment_admin" allow this action?
  ✅ Fast! No database call.
```

**The catch:** If you change a user's role in the database, their JWT still has the old role until it expires (or they log out and back in).

---

**Option B: Roles fetched from database**

JWT only has user ID. On each request, look up their role from the database.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Request    │────▶│  Auth Check  │────▶│   Database   │
│  (has JWT)   │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
                            │                    │
                            │  SELECT role       │
                            │  FROM users        │
                            │  WHERE id = ?      │
                            │◀───────────────────┘
                            │
                     Check permissions
```

**Pros:** Role changes take effect immediately
**Cons:** Every request hits the database (slower, +5-20ms latency)

---

**When to use which:**

| Scenario | Recommendation |
|----------|---------------|
| Roles rarely change | JWT (most cases) |
| Need instant role revocation | DB lookup + short JWT expiry |
| High-security (revoke access NOW) | DB lookup or token blacklist |

---

### 3. Environment Access List: JWT vs Database

**The Question:** How do we know which environments a user can access?

---

**Option A: List environments in JWT** ✅ Recommended for <20 environments

```json
{
  "sub": "user_123",
  "tenant_id": "hospital_123",
  "role": "environment_admin",
  "env_access": ["env_prod", "env_staging", "env_dev"]  ◀── List is HERE
}
```

Authorization check:
```
Is "env_prod" in env_access array?
  → Yes? Allow access.
  → No? Reject.

✅ Fast! Just an array lookup, no DB call.
```

**The catch:** JWT has a size limit (~8KB). If a user has access to 100 environments, the token gets huge.

---

**Option B: Look up from database**

JWT only has user ID. Query the `user_env_access` table on each request.

```
SELECT env_id FROM user_env_access WHERE user_id = 'user_123'
```

**Pros:** Unlimited environments, changes take effect immediately
**Cons:** Database call on every request

---

**When to use which:**

| Scenario | Recommendation |
|----------|---------------|
| Users access <20 environments | JWT (fast, simple) |
| Users access 20+ environments | DB lookup (avoid huge tokens) |
| Need instant access revocation | DB lookup |

---

# Interviewer Follow-Up Questions (Prepare For)

1. **"How do you prevent one tenant from accessing another's data?"**
   - tenant_id is in JWT, checked on every request
   - Separate databases per tenant (no shared tables)
   - All queries scoped to tenant_id

2. **"User's role changes. How do they get new permissions?"**
   - Update role in DB
   - Next token refresh (or logout/login) gets new claims
   - For immediate effect: invalidate current tokens

3. **"How do you handle a user who needs access to multiple environments?"**
   - env_access array in JWT lists all accessible environments
   - user_env_access table tracks the M:M relationship
   - Hospital admin grants/revokes access

4. **"What happens when you delete an environment?"**
   - Soft delete first (status = DELETING)
   - Remove user access mappings
   - Queue async cleanup of resources
   - Audit log the deletion

---

# Summary: What Makes This Design Strong

1. **Clear data model**: Tenant → Environment → User hierarchy
2. **Simple RBAC**: Three roles with clear permission boundaries
3. **JWT-based auth**: Fast authorization without DB lookups
4. **Tenant isolation**: Separate databases, tenant_id in every request
5. **Audit trail**: All access logged for compliance
