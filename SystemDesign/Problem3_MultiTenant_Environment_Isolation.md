# Problem 3: Multi-Tenant Healthcare Environment Platform

## Interviewer Prompt

> "We need to design a platform that manages isolated healthcare environments for hospital customers.
>
> When a hospital subscribes to our products, we create environments for them—things like their EHR instance, patient portal, analytics. Each hospital might have multiple environments: production, staging, dev.
>
> We need to make sure hospitals can't see each other's data. We also need role-based access—org admins, environment admins, regular users.
>
> How would you design this?"

---

# Step 1: Clarify Requirements (3-5 min)

## Questions to Ask

| Question | Why It Matters | Example Answer |
|----------|----------------|----------------|
| How many hospitals/tenants? | Isolation strategy, infra cost | 100-1000 hospitals |
| Environments per hospital? | Resource planning | 3-5 (prod, staging, dev, etc.) |
| Users per environment? | Auth system scale | 50-500 per environment |
| Read/write ratio? | DB optimization | 80/20 read-heavy |
| Isolation level needed? | Network vs DB vs app | Full network isolation (HIPAA) |
| Latency requirements? | Architecture decisions | <200ms P99 |
| Availability target? | Redundancy planning | 99.95% for prod |

## Functional Requirements (Confirmed)

- **Tenant isolation**: Hospital A cannot access Hospital B's data
- **Multi-environment**: Each hospital has prod/staging/dev environments
- **RBAC**: Three role levels with different permissions
- **Environment lifecycle**: Create, update, delete, clone environments
- **Onboarding/Offboarding**: Hospital signup and churn flows

## Non-Functional Requirements

- **HIPAA compliance**: Encryption, audit logs, access controls
- **High availability**: 99.95% for production environments
- **Scalability**: Support 1000+ hospitals, 5000+ environments
- **Security**: Zero tolerance for cross-tenant data leaks

---

# Step 2: Define Scope & Actors (2-3 min)

## Core Actors

| Actor | Scope | Key Actions |
|-------|-------|-------------|
| **Hospital Admin** | Tenant-wide | Create/delete environments, manage users, view billing |
| **Environment Admin** | Single environment | Configure services, deploy apps, manage env settings |
| **Regular User** | Single environment | Access patient data, use applications |

## In Scope (This Interview)
- Tenant & environment management
- Isolation architecture
- Access control system
- Environment provisioning flow

## Out of Scope
- Individual application design (EHR, portal internals)
- Billing system details
- Specific cloud provider implementation

---

# Step 3: High-Level Architecture (10 min)

## Core Insight: Control Plane / Data Plane Separation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CONTROL PLANE (Shared)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐    ┌─────────────────────────┐    ┌──────────────────┐   │
│   │   Clients    │───▶│  Security & Access      │───▶│   Monitoring/    │   │
│   │              │    │  Control (AuthN/AuthZ)  │    │   Audit Service  │   │
│   └──────────────┘    └───────────┬─────────────┘    └──────────────────┘   │
│                                   │                                          │
│                                   ▼                                          │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                    Tenant Management Service                       │     │
│   │         (Hospital/Tenant level interactions & RBAC)               │     │
│   └───────────────────────────────┬───────────────────────────────────┘     │
│                                   │ Provision environment                    │
│                                   ▼                                          │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                 Environment Management Service                     │     │
│   │      (Handle internal environments, deploy app/data)              │     │
│   └───────────────────────────────┬───────────────────────────────────┘     │
│                                   │                                          │
│                                   ▼                                          │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                   Infrastructure Provisioner                       │     │
│   │          (Creates separate VPCs, databases, storage buckets)      │     │
│   └───────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          DATA PLANE (Isolated per Hospital)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │   Hospital A Environment│  │   Hospital B Environment│    ...            │
│  │   ┌─────────────────┐   │  │   ┌─────────────────┐   │                   │
│  │   │ App Services    │   │  │   │ App Services    │   │                   │
│  │   │ (EHR, Portal)   │   │  │   │ (EHR, Portal)   │   │                   │
│  │   └─────────────────┘   │  │   └─────────────────┘   │                   │
│  │   ┌────────┬────────┐   │  │   ┌────────┬────────┐   │                   │
│  │   │Storage │Database│   │  │   │Storage │Database│   │                   │
│  │   └────────┴────────┘   │  │   └────────┴────────┘   │                   │
│  └─────────────────────────┘  └─────────────────────────┘                   │
│                                                                              │
│  Each environment instance provisioned independently (data isolation)        │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility |
|-----------|---------------|
| **Auth Service** | JWT issuance, token validation, session management |
| **Tenant Management** | Hospital CRUD, user management, RBAC policies |
| **Environment Management** | Environment lifecycle, deployment orchestration |
| **Infrastructure Provisioner** | VPC creation, DB provisioning, storage setup |
| **Monitoring/Audit** | Centralized logging, compliance reporting |

---

# Step 4: API Design (5 min)

## Tenant Management APIs

```
POST   /api/v1/tenants                    # Onboard new hospital
GET    /api/v1/tenants/{tenant_id}        # Get hospital details
DELETE /api/v1/tenants/{tenant_id}        # Offboard hospital (soft delete)

POST   /api/v1/tenants/{tenant_id}/users  # Add user to hospital
PUT    /api/v1/tenants/{tenant_id}/users/{user_id}/role  # Update role
```

## Environment Management APIs

```
POST   /api/v1/tenants/{tenant_id}/environments           # Create environment
GET    /api/v1/tenants/{tenant_id}/environments           # List environments
GET    /api/v1/tenants/{tenant_id}/environments/{env_id}  # Get env details
DELETE /api/v1/tenants/{tenant_id}/environments/{env_id}  # Delete environment
POST   /api/v1/tenants/{tenant_id}/environments/{env_id}/clone  # Clone env
```

## Example: Create Environment Request

```json
POST /api/v1/tenants/hospital_123/environments
{
  "name": "production",
  "type": "prod",
  "region": "us-east-1",
  "services": ["ehr", "patient-portal", "analytics"],
  "config": {
    "db_size": "large",
    "backup_enabled": true
  }
}
```

## Example: Response

```json
{
  "env_id": "env_abc123",
  "tenant_id": "hospital_123",
  "status": "provisioning",
  "estimated_ready": "2024-01-15T10:30:00Z",
  "endpoints": {
    "ehr": "pending",
    "portal": "pending"
  }
}
```

---

# Step 5: Data Model (5 min)

## Entity Relationships

```
┌─────────────┐       ┌─────────────────┐       ┌─────────────┐
│   Tenant    │1─────*│   Environment   │1─────*│    User     │
│  (Hospital) │       │                 │       │             │
├─────────────┤       ├─────────────────┤       ├─────────────┤
│ tenant_id   │       │ env_id          │       │ user_id     │
│ name        │       │ tenant_id (FK)  │       │ tenant_id   │
│ status      │       │ type (prod/stg) │       │ email       │
│ tier        │       │ status          │       │ role        │
│ created_at  │       │ vpc_id          │       │ env_access[]│
└─────────────┘       │ db_endpoint     │       └─────────────┘
                      │ storage_bucket  │
                      │ created_at      │
                      └─────────────────┘
```

## Token Structure (JWT)

```json
{
  "sub": "user_id",
  "tenant_id": "hospital_123",
  "env_access": ["env_prod", "env_staging"],
  "role": "environment_admin",
  "permissions": ["env:read", "env:write", "users:read"],
  "exp": 1234567890
}
```

## Access Control Database

```sql
-- Role definitions
CREATE TABLE roles (
  role_id VARCHAR PRIMARY KEY,
  role_name VARCHAR,  -- hospital_admin, env_admin, user
  permissions JSONB   -- ["tenant:*", "env:*", "users:*"]
);

-- User-environment access mapping
CREATE TABLE user_env_access (
  user_id VARCHAR,
  env_id VARCHAR,
  granted_by VARCHAR,
  granted_at TIMESTAMP,
  PRIMARY KEY (user_id, env_id)
);
```

---

# Step 6: Deep Dive - Core Flows (10 min)

## 6.1 Environment Provisioning Flow

```
Hospital Admin clicks "Create Environment"
        │
        ▼
┌─────────────────────────────────────┐
│ 1. API Gateway receives request     │
│    - Validate JWT                   │
│    - Check: role = hospital_admin?  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 2. Environment Service              │
│    - Validate request params        │
│    - Check tenant quota             │
│    - Create env record (PENDING)    │
│    - Publish to provisioning queue  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 3. Infrastructure Provisioner       │
│    - Create VPC + subnets           │
│    - Provision RDS instance         │
│    - Create S3 bucket               │
│    - Deploy app containers          │
│    - Configure security groups      │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 4. Update env record (READY)        │
│    - Store endpoints                 │
│    - Notify admin via webhook/email │
└─────────────────────────────────────┘
```

## 6.2 Request Authorization Flow

```
Request: GET /api/env/env_prod_123/patients
Headers: Authorization: Bearer <JWT>
        │
        ▼
┌─────────────────────────────────────┐
│ 1. Validate JWT signature & expiry  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 2. Extract claims:                  │
│    tenant_id: hospital_123          │
│    env_access: [env_prod_123, ...]  │
│    role: environment_admin          │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 3. Authorization checks:            │
│    - env_prod_123 IN env_access? ✓  │
│    - role has patients:read? ✓      │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 4. Route to data plane:             │
│    - Lookup env_prod_123 endpoint   │
│    - Forward request to VPC         │
└─────────────────────────────────────┘
```

## 6.3 Hospital Offboarding Flow

```
1. Soft Delete
   └─► Mark tenant status = "PENDING_DELETION"
   └─► Disable all user logins
   └─► Stop billing

2. Grace Period (30 days)
   └─► Data export API available
   └─► Admin can request restore

3. Hard Delete
   └─► Delete all environments (parallel)
       ├─► Terminate compute instances
       ├─► Delete databases (final backup first)
       ├─► Delete storage buckets
       └─► Delete VPCs
   └─► Purge control plane records
   └─► Audit log: "Tenant X fully deleted"
```

---

# Step 7: Scaling & Bottlenecks (5 min)

## Potential Bottlenecks

| Bottleneck | Solution |
|------------|----------|
| **Provisioning queue backlog** | Multiple provisioner workers, priority queues |
| **Control plane DB** | Read replicas, connection pooling |
| **Auth service overload** | Token caching, distributed auth |
| **Cross-region latency** | Regional control plane deployments |

## Scaling Strategy

```
                    ┌─────────────────┐
                    │  Global LB      │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ US-East Region  │ │ US-West Region  │ │ EU Region       │
│ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
│ │Control Plane│ │ │ │Control Plane│ │ │ │Control Plane│ │
│ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │
│ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
│ │Data Planes  │ │ │ │Data Planes  │ │ │ │Data Planes  │ │
│ │(Hospitals)  │ │ │ │(Hospitals)  │ │ │ │(Hospitals)  │ │
│ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

## Numbers to Know

| Metric | Target |
|--------|--------|
| Environment provisioning | < 10 min |
| API latency (P99) | < 200ms |
| Auth token validation | < 10ms |
| Max environments per tenant | 10 |
| Max tenants per region | 500 |

---

# Step 8: Tradeoffs & Wrap-Up (5 min)

## Key Design Tradeoffs

### 1. Isolation Strategy

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Separate VPCs** | Lower cost, easier mgmt | Softer boundary | Default for most |
| **Separate AWS Accounts** | Strongest isolation | Complex, costly | Enterprise tier |
| **Namespace only** | Cheapest | Weak isolation | Dev/test only |

### 2. Database Strategy

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Shared DB + tenant_id** | Cheapest | Query bug = data leak | Never for healthcare |
| **Separate schema** | Mid isolation | Migration pain | Acceptable |
| **Separate DB** | Best isolation | Highest cost | **Use this** |

### 3. App Deployment

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Shared app** | Single deploy | Bug affects all | Not for healthcare |
| **Dedicated per tenant** | Full isolation | Complex deploys | **Use this** |

## HIPAA Compliance Checklist

- [ ] Encryption at rest (AES-256)
- [ ] Encryption in transit (TLS 1.3)
- [ ] Audit logging (all PHI access)
- [ ] Access reviews (quarterly)
- [ ] BAA with cloud provider
- [ ] Backup & DR tested

## SLAs

| Environment | Availability | RPO | RTO |
|-------------|--------------|-----|-----|
| Production | 99.95% | 1 hr | 4 hr |
| Staging | 99.5% | 24 hr | 24 hr |
| Dev | 99% | 24 hr | 48 hr |

---

# Interviewer Follow-Up Questions (Prepare For)

1. **"How do you handle a hospital that needs to share data across environments?"**
   - Cross-environment data sync service with explicit grants
   - Audit all cross-env access

2. **"What if provisioning fails halfway through?"**
   - Saga pattern with compensating transactions
   - Cleanup orphaned resources, retry from checkpoint

3. **"How do you handle noisy neighbor in shared control plane?"**
   - Rate limiting per tenant
   - Priority queues for paid tiers
   - Auto-scaling based on queue depth

4. **"What about disaster recovery?"**
   - Cross-region replication for control plane
   - Per-tenant backup policies in data plane
   - Annual DR drills

---

# Summary: What Makes This Design Strong

1. **Clear separation**: Control plane (shared) vs Data plane (isolated)
2. **Defense in depth**: Network + DB + app isolation for healthcare
3. **Scalable**: Horizontal scaling at each layer
4. **Compliant**: HIPAA requirements baked into design
5. **Operationally sound**: Clear provisioning and offboarding flows
