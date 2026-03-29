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
│   Tenant    │         │   Environment   │         │    User     │
│  (Hospital) │         │                 │         │             │
├─────────────┤         ├─────────────────┤         ├─────────────┤
│ tenant_id   │         │ env_id          │         │ user_id     │
│ name        │         │ tenant_id (FK)  │         │ tenant_id   │
│ status      │         │ name            │         │ email       │
│ created_at  │         │ type            │         │ role        │
└──────┬──────┘         │ status          │         └──────┬──────┘
       │                │ created_at      │                │
       │                └────────┬────────┘                │
       │                         │                         │
       │ 1:N                     │                         │ 1:N
       │ (one hospital has       │                         │ (one hospital has
       │  many environments)     │                         │  many users)
       │                         │                         │
       ▼                         │                         ▼
  ┌─────────┐                    │                    ┌─────────┐
  │ Hospital├────────────────────┘                    │ Hospital│
  │ owns    │                                         │ owns    │
  └─────────┘                                         └─────────┘
                                 │
                         N:M     │
                   (many users can access
                    many environments)
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │    user_env_access     │
                    │    (join table)        │
                    ├────────────────────────┤
                    │ user_id (FK)           │
                    │ env_id (FK)            │
                    │ granted_at             │
                    └────────────────────────┘
```

**Relationship Summary:**
| Relationship | Type | Meaning |
|--------------|------|---------|
| Tenant → Environment | 1:N | One hospital has many environments (prod, staging, dev) |
| Tenant → User | 1:N | One hospital has many users |
| User ↔ Environment | N:M | Many users can access many environments (via user_env_access) |

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

| Bottleneck | Solution |
|------------|----------|
| **Auth service overload** | Cache JWT validation, distributed auth |
| **Control plane DB** | Read replicas, connection pooling |
| **Provisioning backlog** | Multiple workers, priority queues |

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

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Shared DB + tenant_id column** | Cheapest | Query bug = data leak | Never for healthcare |
| **Separate schema per tenant** | Mid isolation | Migration complexity | Acceptable |
| **Separate DB per tenant** | Best isolation | Higher cost | **Use this** |

### 2. RBAC Storage

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Roles in JWT** | Fast, no DB lookup | Token refresh for changes | **Use this** |
| **Roles in DB lookup** | Always fresh | Adds latency | Only if roles change frequently |

### 3. Environment Access

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **List in JWT** | Fast authorization | Token size grows | **Use for <20 envs** |
| **DB lookup** | Unlimited envs | Adds latency | Use for >20 envs |

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
