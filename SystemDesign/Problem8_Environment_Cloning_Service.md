# Problem 8: Environment Cloning Service

## Interviewer Prompt

> "We need to design a service that lets customers clone their healthcare environments.
>
> For example, a hospital might want to clone their production environment to staging for testing new features, or clone to create a training environment.
>
> The tricky parts: clones can be large (hundreds of GBs), we need to handle PHI data masking for non-prod clones, and customers want to track progress.
>
> How would you design this?"

---

# Step 1: Clarify Requirements (3-5 min)

## Questions to Ask

| Question | Why It Matters | Example Answer |
|----------|----------------|----------------|
| What needs to be cloned? | Scope of work | DB, storage, configs, app state |
| Typical data size? | Architecture & timing | 10GB - 500GB |
| How often do customers clone? | Capacity planning | 2-3 times per week per tenant |
| Must clone be atomic? | Consistency guarantees | Yes, point-in-time snapshot |
| PHI masking required? | Compliance complexity | Required for non-prod targets |
| Can source change during clone? | Consistency model | Yes, clone is point-in-time |
| Max acceptable clone duration? | SLA definition | 1 hour for 100GB |

## Functional Requirements (Confirmed)

- **Full environment clone**: Copy database, storage, configurations
- **Selective clone**: Option to clone config-only (skip data)
- **PHI masking**: Automatic data masking for non-production targets
- **Progress tracking**: Real-time progress (0-100%) with current step
- **Cancellation**: Ability to cancel in-progress clone operations
- **Clone history**: Track past clone operations for auditing

## Non-Functional Requirements

- **Durability**: Clone operation survives service restarts
- **Isolation**: Clone operation doesn't impact source environment performance
- **HIPAA compliance**: Audit logging, proper PHI handling
- **Scalability**: Support 100 concurrent clone operations

---

# Step 2: Define Scope & Actors (2-3 min)

## Core Actors

| Actor | Scope | Key Actions |
|-------|-------|-------------|
| **Environment Admin** | Single environment | Initiate clone, track progress, cancel |
| **Hospital Admin** | Tenant-wide | View all clone operations, manage quotas |
| **System (Async)** | Internal | Execute clone steps, report progress |

## In Scope (This Interview)
- Clone orchestration and workflow
- Progress tracking API
- Data masking integration
- Cancellation flow

## Out of Scope
- PHI masking algorithm details
- Storage-level snapshot implementation
- Billing for clone storage

---

# Step 3: High-Level Architecture (10 min)

## Core Insight: Async Workflow with Progress Tracking

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLONE SERVICE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐    ┌─────────────────────┐    ┌──────────────────────┐   │
│   │   Client     │───▶│   Clone API         │───▶│   Clone Workflow     │   │
│   │              │    │   (REST)            │    │   Orchestrator       │   │
│   └──────────────┘    └─────────────────────┘    └──────────┬───────────┘   │
│          │                                                   │               │
│          │            ┌─────────────────────┐               │               │
│          └───────────▶│   Progress API      │◀──────────────┘               │
│            (poll)     │   (REST/WebSocket)  │     (updates)                 │
│                       └─────────────────────┘                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Orchestrates
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DOWNSTREAM SERVICES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
│   │  Database       │  │  Storage        │  │  Config         │            │
│   │  Clone Service  │  │  Clone Service  │  │  Clone Service  │            │
│   │                 │  │                 │  │                 │            │
│   │  - Snapshot     │  │  - Copy blobs   │  │  - Copy configs │            │
│   │  - Restore      │  │  - Sync files   │  │  - Transform    │            │
│   │  - Mask PHI     │  │                 │  │                 │            │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility |
|-----------|---------------|
| **Clone API** | Accept clone requests, validate permissions, return clone job ID |
| **Clone Workflow Orchestrator** | Execute clone steps, track state, handle retries |
| **Progress API** | Expose real-time progress, support polling and push |
| **Database Clone Service** | Snapshot DB, restore to target, apply PHI masking |
| **Storage Clone Service** | Copy storage buckets/blobs |
| **Config Clone Service** | Copy and transform environment configurations |

---

# Step 4: API Design (5 min)

## Clone Management APIs

```
POST   /api/v1/environments/{env_id}/clone           # Start clone operation
GET    /api/v1/clone-jobs/{job_id}                   # Get clone job status
GET    /api/v1/clone-jobs/{job_id}/progress          # Get detailed progress
POST   /api/v1/clone-jobs/{job_id}/cancel            # Cancel clone operation
GET    /api/v1/environments/{env_id}/clone-history   # List past clones
```

## Example: Start Clone Request

```json
POST /api/v1/environments/env_prod_123/clone
{
  "target_name": "staging-march-2024",
  "target_type": "staging",
  "clone_options": {
    "include_data": true,
    "mask_phi": true,
    "include_storage": true
  },
  "notification_webhook": "https://customer.com/webhooks/clone"
}
```

## Example: Start Clone Response

```json
{
  "job_id": "clone_job_abc123",
  "source_env_id": "env_prod_123",
  "target_env_id": "env_staging_456",
  "status": "IN_PROGRESS",
  "created_at": "2024-03-15T10:00:00Z",
  "progress": {
    "percent": 0,
    "current_step": "INITIALIZING",
    "steps": [
      {"name": "SNAPSHOT_DATABASE", "status": "PENDING"},
      {"name": "COPY_STORAGE", "status": "PENDING"},
      {"name": "MASK_PHI", "status": "PENDING"},
      {"name": "COPY_CONFIGS", "status": "PENDING"},
      {"name": "FINALIZE", "status": "PENDING"}
    ]
  }
}
```

## Example: Progress Response (Mid-Clone)

```json
GET /api/v1/clone-jobs/clone_job_abc123/progress
{
  "job_id": "clone_job_abc123",
  "status": "IN_PROGRESS",
  "percent": 45,
  "current_step": "COPY_STORAGE",
  "eta_seconds": 1800,
  "steps": [
    {"name": "SNAPSHOT_DATABASE", "status": "COMPLETED", "duration_ms": 120000},
    {"name": "COPY_STORAGE", "status": "IN_PROGRESS", "percent": 60, "bytes_copied": "30GB", "bytes_total": "50GB"},
    {"name": "MASK_PHI", "status": "PENDING"},
    {"name": "COPY_CONFIGS", "status": "PENDING"},
    {"name": "FINALIZE", "status": "PENDING"}
  ],
  "updated_at": "2024-03-15T10:25:00Z"
}
```

---

# Step 5: Data Model (5 min)

## Entity Relationships

```
┌─────────────────┐       ┌─────────────────────┐       ┌─────────────────┐
│   Environment   │1─────*│     CloneJob        │*─────1│   Environment   │
│   (Source)      │       │                     │       │   (Target)      │
├─────────────────┤       ├─────────────────────┤       ├─────────────────┤
│ env_id          │       │ job_id              │       │ env_id          │
│ tenant_id       │       │ source_env_id (FK)  │       │ cloned_from     │
│ ...             │       │ target_env_id (FK)  │       │ ...             │
└─────────────────┘       │ status              │       └─────────────────┘
                          │ clone_options (JSON)│
                          │ progress (JSON)     │
                          │ created_at          │
                          │ completed_at        │
                          │ error_message       │
                          └─────────────────────┘
                                    │
                                   1│
                                    │
                                   *│
                          ┌─────────────────────┐
                          │   CloneJobStep      │
                          ├─────────────────────┤
                          │ step_id             │
                          │ job_id (FK)         │
                          │ step_name           │
                          │ status              │
                          │ percent             │
                          │ started_at          │
                          │ completed_at        │
                          │ metadata (JSON)     │
                          └─────────────────────┘
```

## Clone Job States

```
           ┌──────────────────────────────────────┐
           │                                      │
           ▼                                      │
┌──────────────────┐    ┌──────────────────┐     │
│     PENDING      │───▶│   IN_PROGRESS    │─────┤
└──────────────────┘    └──────────────────┘     │
                               │                  │
              ┌────────────────┼────────────────┐│
              │                │                ││
              ▼                ▼                ▼│
     ┌──────────────┐  ┌──────────────┐  ┌─────────────┐
     │  COMPLETED   │  │    FAILED    │  │  CANCELLED  │
     └──────────────┘  └──────────────┘  └─────────────┘
```

## Schema & Indexes

**clone_jobs**
| Column | Type | Notes |
|--------|------|-------|
| job_id | VARCHAR | PK |
| source_env_id | VARCHAR | FK to environments |
| target_env_id | VARCHAR | FK to environments |
| tenant_id | VARCHAR | |
| status | VARCHAR | PENDING, IN_PROGRESS, COMPLETED, FAILED, CANCELLED |
| clone_options | JSONB | include_data, mask_phi, etc. |
| progress_percent | INT | 0-100 |
| current_step | VARCHAR | |
| eta_seconds | INT | |
| webhook_url | VARCHAR | |
| created_at, started_at, completed_at | TIMESTAMP | |
| error_message | TEXT | |
| created_by | VARCHAR | |

**clone_job_steps**
| Column | Type | Notes |
|--------|------|-------|
| step_id | VARCHAR | PK |
| job_id | VARCHAR | FK to clone_jobs |
| step_name | VARCHAR | SNAPSHOT_DATABASE, COPY_STORAGE, etc. |
| step_order | INT | |
| status | VARCHAR | |
| percent | INT | 0-100 |
| metadata | JSONB | bytes_copied, etc. |
| started_at, completed_at | TIMESTAMP | |

**Indexes**
- `clone_jobs(tenant_id)` — filter by tenant
- `clone_jobs(status) WHERE status = 'IN_PROGRESS'` — partial index for active jobs

---

# Step 6: Deep Dive - Core Flows (10 min)

## 6.1 Clone Initiation Flow

```
Environment Admin clicks "Clone Environment"
        │
        ▼
┌─────────────────────────────────────┐
│ 1. Clone API receives request       │
│    - Validate JWT, check permission │
│    - Check source env exists        │
│    - Check quota (max clones)       │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 2. Create CloneJob record           │
│    - Status: PENDING                │
│    - Generate job_id                │
│    - Create placeholder target env  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 3. Enqueue to Clone Workflow Queue  │
│    - Include job_id, options        │
│    - Return job_id to client        │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 4. Client polls progress or awaits  │
│    webhook notification             │
└─────────────────────────────────────┘
```

## 6.2 Clone Execution Flow (Orchestrator)

```
Orchestrator picks up job from queue
        │
        ▼
┌─────────────────────────────────────┐
│ 1. Update job status: IN_PROGRESS   │
│    Lock job for this worker         │
└──────────────────┬──────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌───────────────────┐  ┌───────────────────┐
│ 2a. Snapshot DB   │  │ 2b. Snapshot      │  (Parallel where possible)
│     - Point-in-   │  │     Storage       │
│       time snap   │  │     - Create blob │
│     - Store ref   │  │       snapshots   │
└────────┬──────────┘  └─────────┬─────────┘
         │                       │
         └───────────┬───────────┘
                     ▼
┌─────────────────────────────────────┐
│ 3. Restore snapshots to target      │
│    - Create new DB from snapshot    │
│    - Copy storage to target bucket  │
│    - Update progress: 60%           │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 4. Apply PHI Masking (if enabled)   │
│    - Run masking queries on DB      │
│    - Update progress: 80%           │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 5. Copy & Transform Configs         │
│    - Clone config documents         │
│    - Update target-specific values  │
│    - Update progress: 95%           │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 6. Finalize                         │
│    - Mark target env ACTIVE         │
│    - Update job status: COMPLETED   │
│    - Fire webhook notification      │
│    - Update progress: 100%          │
└─────────────────────────────────────┘
```

## 6.3 Cancellation Flow

```
Admin clicks "Cancel Clone"
        │
        ▼
┌─────────────────────────────────────┐
│ 1. API receives cancel request      │
│    - Validate permission            │
│    - Check job is cancellable       │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 2. Set cancellation flag            │
│    - Update DB: cancel_requested=T  │
│    - Orchestrator checks flag       │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 3. Orchestrator stops at next       │
│    checkpoint                       │
│    - Complete current atomic op     │
│    - Don't start next step          │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 4. Cleanup                          │
│    - Delete partial target env      │
│    - Release snapshots              │
│    - Mark job: CANCELLED            │
└─────────────────────────────────────┘
```

## 6.4 Progress Calculation

```python
def calculate_progress(job):
    """
    Progress is weighted by estimated step duration.
    """
    step_weights = {
        "SNAPSHOT_DATABASE": 10,   # Fast - just creates reference
        "SNAPSHOT_STORAGE": 5,     # Fast - blob snapshots
        "RESTORE_DATABASE": 30,    # Slow - full restore
        "RESTORE_STORAGE": 25,     # Medium - copy bytes
        "MASK_PHI": 20,            # Medium - DB transformations
        "COPY_CONFIGS": 5,         # Fast - small documents
        "FINALIZE": 5              # Fast - status updates
    }

    total_weight = sum(step_weights.values())
    completed_weight = 0

    for step in job.steps:
        weight = step_weights[step.name]
        if step.status == "COMPLETED":
            completed_weight += weight
        elif step.status == "IN_PROGRESS":
            # Partial credit based on step's internal progress
            completed_weight += weight * (step.percent / 100)

    return int((completed_weight / total_weight) * 100)
```

---

# Step 7: Scaling & Bottlenecks (5 min)

## Potential Bottlenecks

| Bottleneck | Solution |
|------------|----------|
| **Clone queue backlog** | Multiple orchestrator workers, priority queues |
| **Large database restores** | Use cloud-native snapshot restore (faster) |
| **PHI masking on large tables** | Parallel masking with batched updates |
| **Progress API hot path** | Cache current progress in Redis |
| **Storage copy bandwidth** | Use server-side copy (don't stream through service) |

## Scaling Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLONE SERVICE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐                                            │
│  │   Clone API     │ ◀── Stateless, horizontally scalable       │
│  │   (3 replicas)  │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐     ┌─────────────────┐                    │
│  │  Clone Queue    │────▶│  Orchestrators  │ ◀── Scale based on │
│  │  (SQS/Kafka)    │     │  (5 workers)    │     queue depth    │
│  └─────────────────┘     └─────────────────┘                    │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │  Progress Cache │ ◀── Redis, 30s TTL on progress data        │
│  │  (Redis)        │                                            │
│  └─────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Numbers to Know

| Metric | Target |
|--------|--------|
| Clone initiation response | < 500ms |
| Progress API response | < 100ms |
| Clone throughput | 100GB/hour |
| Max concurrent clones per tenant | 3 |
| Max concurrent clones system-wide | 100 |

---

# Step 8: Tradeoffs & Wrap-Up (5 min)

## Key Design Tradeoffs

### 1. Snapshot Strategy

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Cloud-native snapshots** | Fast, cheap, atomic | Cloud-specific | **Use this** |
| **Streaming copy** | Portable | Slow, resource-intensive | Avoid |
| **Logical dump/restore** | Flexible | Very slow for large DBs | Only for small envs |

### 2. Progress Tracking

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Polling API** | Simple, stateless | Wasteful requests | Default option |
| **WebSocket** | Real-time | Connection management | Optional add-on |
| **Webhook only** | No polling load | Customer must implement | Require for large clones |

### 3. PHI Masking

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **In-place on target** | Simpler | Briefly has real PHI | **Use this** (target is isolated) |
| **Stream through masking** | Never has real PHI | Complex, slow | Overkill |
| **Pre-masked snapshots** | Instant clones | Stale data | For repeated cloning |

## HIPAA Compliance Checklist

- [ ] Audit log all clone operations with user identity
- [ ] PHI masking uses approved algorithm
- [ ] Target environment has same security controls
- [ ] Clone job records retained for 7 years
- [ ] Access to clone API requires MFA

---

# Interviewer Follow-Up Questions (Prepare For)

1. **"Source environment changes during the clone. What happens to those changes?"**
   - Clone is point-in-time snapshot; changes after snapshot are not included
   - Clearly communicate this to users (snapshot timestamp in response)
   - Option: offer "live sync" clone mode for staging (more complex)

2. **"Clone is 80% done and fails. What do you do?"**
   - Don't auto-retry by default (could be quota or config issue)
   - Clean up partial resources (target DB, storage)
   - Preserve job record with error details for debugging
   - Offer manual retry button that resumes from last checkpoint

3. **"Customer wants to clone 500GB database. How long will it take?"**
   - Depends on cloud provider snapshot restore speed
   - Typical: 100GB/hour for restore, plus masking time
   - Provide estimated completion time in progress API
   - Consider offering "clone without data" option for faster iteration

4. **"How do you prevent clone from impacting source environment?"**
   - Use copy-on-write snapshots (zero impact on source)
   - Clone operations run on separate worker pool
   - Rate limit clones per source environment

---

# Summary: What Makes This Design Strong

1. **Async-first**: Long operation with proper progress tracking
2. **Resumable**: Checkpointed steps allow recovery from failures
3. **Compliant**: PHI masking built into the workflow
4. **Observable**: Clear progress with step-level granularity
5. **Cancellable**: Clean cancellation at safe checkpoints
