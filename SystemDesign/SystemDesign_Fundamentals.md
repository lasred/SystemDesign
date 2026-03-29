# System Design Fundamentals

A reference guide for core concepts with OHCP-relevant examples.

---

## Table of Contents
1. [Control Plane vs Data Plane](#1-control-plane-vs-data-plane)
2. [Synchronous vs Asynchronous](#2-synchronous-vs-asynchronous)
3. [State Machines](#3-state-machines)
4. [Idempotency](#4-idempotency)
5. [Consistency Models](#5-consistency-models)
6. [Common Patterns](#6-common-patterns)
7. [Failure Handling](#7-failure-handling)
8. [Scaling Strategies](#8-scaling-strategies)

---

## 1. Control Plane vs Data Plane

### Definition

| Aspect | Control Plane | Data Plane |
|--------|--------------|------------|
| **Purpose** | Manages and configures the system | Handles actual user data/traffic |
| **Frequency** | Infrequent operations | High-frequency operations |
| **Latency tolerance** | Can tolerate higher latency | Must be low latency |
| **Consistency** | Strong consistency preferred | Eventually consistent often OK |
| **Scale** | Lower request volume | High request volume |

### Mental Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CONTROL PLANE                                   │
│                                                                              │
│   "The brain that decides WHAT should exist and HOW it should be configured"│
│                                                                              │
│   - Create/delete resources                                                 │
│   - Configure settings                                                      │
│   - Manage lifecycle states                                                 │
│   - Handle provisioning workflows                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Provisions & Configures
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               DATA PLANE                                     │
│                                                                              │
│   "The muscles that DO the actual work with user data"                      │
│                                                                              │
│   - Serve API requests                                                      │
│   - Store/retrieve patient records                                          │
│   - Process transactions                                                    │
│   - Run application logic                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### OHCP Examples

| Control Plane (OHCP) | Data Plane (Customer's Environment) |
|---------------------|-------------------------------------|
| Create new hospital environment | EHR serving patient records |
| Provision database instance | Database storing PHI |
| Configure VPC networking | Network traffic between services |
| Manage subscription lifecycle | Patient portal serving users |
| Deploy application versions | Application processing requests |
| Set resource quotas | Actual resource consumption |

### Real-World Analogies

| System | Control Plane | Data Plane |
|--------|--------------|------------|
| **Kubernetes** | API Server, Scheduler, Controller Manager | Kubelet, Container Runtime, Pods |
| **AWS** | CloudFormation, IAM, Resource APIs | EC2 instances, S3 buckets, RDS |
| **Networking** | BGP routing, SDN controllers | Packet forwarding, switches |
| **DNS** | Zone management, record updates | DNS query resolution |

### Why This Separation Matters

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BENEFITS OF SEPARATION                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. INDEPENDENT SCALING                                                     │
│     ┌─────────────────┐     ┌─────────────────┐                            │
│     │ Control Plane   │     │ Data Plane      │                            │
│     │ 3 instances     │     │ 100+ instances  │                            │
│     │ (low traffic)   │     │ (high traffic)  │                            │
│     └─────────────────┘     └─────────────────┘                            │
│                                                                              │
│  2. DIFFERENT AVAILABILITY REQUIREMENTS                                     │
│     - Control plane down: Can't create new resources (annoying)             │
│     - Data plane down: Users can't access data (critical!)                  │
│                                                                              │
│  3. SECURITY ISOLATION                                                      │
│     - Control plane: Admin access, highly privileged                        │
│     - Data plane: User access, tenant-isolated                              │
│                                                                              │
│  4. DIFFERENT CONSISTENCY REQUIREMENTS                                      │
│     - Control plane: Strong consistency (don't create duplicate resources) │
│     - Data plane: Can often use eventual consistency for performance        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### OHCP Architecture Example

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OHCP CONTROL PLANE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Tenant     │  │ Environment  │  │  Workflow    │  │ Subscription │    │
│  │   Service    │  │   Service    │  │  Orchestrator│  │   Service    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
│         │                │                  │                 │             │
│         └────────────────┴──────────────────┴─────────────────┘             │
│                                    │                                         │
│                                    ▼                                         │
│                          ┌──────────────────┐                               │
│                          │   Kiev (Entity   │                               │
│                          │   Store)         │                               │
│                          └──────────────────┘                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                     Provisions     │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATA PLANES (Per Hospital)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────┐    ┌─────────────────────────┐                 │
│  │   Hospital A            │    │   Hospital B            │                 │
│  │   ┌─────┐ ┌─────┐      │    │   ┌─────┐ ┌─────┐      │                 │
│  │   │ EHR │ │Portal│      │    │   │ EHR │ │Portal│      │                 │
│  │   └─────┘ └─────┘      │    │   └─────┘ └─────┘      │                 │
│  │   ┌─────────────┐      │    │   ┌─────────────┐      │                 │
│  │   │  Database   │      │    │   │  Database   │      │                 │
│  │   └─────────────┘      │    │   └─────────────┘      │                 │
│  └─────────────────────────┘    └─────────────────────────┘                 │
│                                                                              │
│  Hospital A users ONLY access Hospital A's data plane                       │
│  Hospital B users ONLY access Hospital B's data plane                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Synchronous vs Asynchronous

### When to Use Each

| Synchronous | Asynchronous |
|-------------|--------------|
| Simple CRUD operations | Long-running operations (>5 sec) |
| User needs immediate response | Can notify user later |
| Operation is fast (<500ms) | Operation may fail and retry |
| Single service involved | Multiple services coordinated |

### OHCP Examples

| Operation | Sync or Async? | Why |
|-----------|---------------|-----|
| Get environment status | Sync | Fast, single lookup |
| Create environment | Async | Takes 5-10 min, multiple services |
| Update environment config | Sync | Fast DB update |
| Clone environment | Async | Copies large data, takes minutes |
| Delete environment | Async | Cleanup across multiple services |
| Check quota | Sync | Fast, needs immediate answer |

### Async Pattern: Request → Job → Poll/Webhook

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      ASYNC OPERATION PATTERN                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. REQUEST (Synchronous - returns immediately)                          │
│     ─────────────────────────────────────────                            │
│     Client: POST /environments                                           │
│     Server: 202 Accepted { "job_id": "job_123", "status": "PENDING" }   │
│                                                                           │
│  2. PROCESS (Asynchronous - happens in background)                       │
│     ──────────────────────────────────────────────                       │
│     Worker picks up job, executes steps, updates status                  │
│                                                                           │
│  3. NOTIFY (Client gets result)                                          │
│     ────────────────────────────                                         │
│                                                                           │
│     Option A: Polling                                                    │
│     Client: GET /jobs/job_123 (every 5 sec)                             │
│     Server: { "status": "IN_PROGRESS", "percent": 60 }                  │
│     ...                                                                   │
│     Server: { "status": "COMPLETED", "result": {...} }                  │
│                                                                           │
│     Option B: Webhook                                                    │
│     Server: POST https://customer.com/webhook                           │
│             { "job_id": "job_123", "status": "COMPLETED" }              │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. State Machines

### Why State Machines?

- **Explicit states**: Know exactly what states are valid
- **Controlled transitions**: Define what transitions are allowed
- **Audit trail**: Track state changes over time
- **Error handling**: Know what to do when stuck in a state

### OHCP Environment Lifecycle

```
                                    ┌─────────────┐
                         create     │             │
                      ─────────────▶│  CREATING   │
                                    │             │
                                    └──────┬──────┘
                                           │
                              success      │      failure
                         ┌─────────────────┼─────────────────┐
                         │                 │                 │
                         ▼                 │                 ▼
                  ┌─────────────┐          │          ┌─────────────┐
                  │             │          │          │             │
                  │   ACTIVE    │          │          │   FAILED    │
                  │             │          │          │             │
                  └──────┬──────┘          │          └──────┬──────┘
                         │                 │                 │
            ┌────────────┼────────────┐    │                 │ retry
            │            │            │    │                 │
         suspend      delete       update  │                 ▼
            │            │            │    │          ┌─────────────┐
            ▼            │            ▼    │          │             │
     ┌─────────────┐     │     ┌─────────────┐       │  CREATING   │
     │             │     │     │             │       │  (retry)    │
     │  SUSPENDED  │     │     │  UPDATING   │       └─────────────┘
     │             │     │     │             │
     └──────┬──────┘     │     └──────┬──────┘
            │            │            │
         resume          │         success
            │            │            │
            ▼            │            ▼
     ┌─────────────┐     │     ┌─────────────┐
     │             │     │     │             │
     │   ACTIVE    │     │     │   ACTIVE    │
     │             │     │     │             │
     └─────────────┘     │     └─────────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │             │
                  │  DELETING   │
                  │             │
                  └──────┬──────┘
                         │
                      success
                         │
                         ▼
                  ┌─────────────┐
                  │             │
                  │   DELETED   │
                  │             │
                  └─────────────┘
```

### State Transition Table

| Current State | Event | Next State | Side Effects |
|---------------|-------|------------|--------------|
| - | create | CREATING | Start provisioning workflow |
| CREATING | success | ACTIVE | Notify customer |
| CREATING | failure | FAILED | Alert ops, log error |
| ACTIVE | suspend | SUSPENDED | Stop compute, keep data |
| ACTIVE | delete | DELETING | Start cleanup workflow |
| ACTIVE | update | UPDATING | Apply config changes |
| SUSPENDED | resume | ACTIVE | Start compute |
| SUSPENDED | delete | DELETING | Start cleanup workflow |
| UPDATING | success | ACTIVE | - |
| UPDATING | failure | ACTIVE | Rollback, alert |
| DELETING | success | DELETED | Purge records after retention |
| FAILED | retry | CREATING | Restart workflow |
| FAILED | delete | DELETED | Cleanup partial resources |

### Implementation Pattern

```python
class EnvironmentStateMachine:
    TRANSITIONS = {
        None: ['CREATING'],
        'CREATING': ['ACTIVE', 'FAILED'],
        'ACTIVE': ['SUSPENDED', 'UPDATING', 'DELETING'],
        'SUSPENDED': ['ACTIVE', 'DELETING'],
        'UPDATING': ['ACTIVE'],  # Always returns to ACTIVE (success or rollback)
        'DELETING': ['DELETED'],
        'FAILED': ['CREATING', 'DELETED'],
        'DELETED': [],  # Terminal state
    }

    def can_transition(self, from_state: str, to_state: str) -> bool:
        return to_state in self.TRANSITIONS.get(from_state, [])

    def transition(self, env: Environment, new_state: str) -> None:
        if not self.can_transition(env.state, new_state):
            raise InvalidTransitionError(
                f"Cannot transition from {env.state} to {new_state}"
            )

        old_state = env.state
        env.state = new_state
        env.state_changed_at = datetime.utcnow()

        # Audit log
        self.audit_log.record(
            entity_id=env.id,
            old_state=old_state,
            new_state=new_state,
            timestamp=env.state_changed_at
        )
```

---

## 4. Idempotency

### Definition

> An operation is **idempotent** if performing it multiple times has the same effect as performing it once.

### Why It Matters

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     WITHOUT IDEMPOTENCY                                   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Client ──POST /environments──▶ Server                                   │
│         ◀── (network timeout) ──                                         │
│                                                                           │
│  Client: "Did it work? I'll retry..."                                    │
│                                                                           │
│  Client ──POST /environments──▶ Server                                   │
│         ◀── 201 Created ────────                                         │
│                                                                           │
│  Result: TWO environments created! 💥                                    │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                      WITH IDEMPOTENCY                                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Client ──POST /environments──▶ Server                                   │
│          (Idempotency-Key: abc)                                          │
│         ◀── (network timeout) ──                                         │
│                                                                           │
│  Client: "Did it work? I'll retry with same key..."                      │
│                                                                           │
│  Client ──POST /environments──▶ Server                                   │
│          (Idempotency-Key: abc)                                          │
│         ◀── 201 Created ────────                                         │
│                                                                           │
│  Result: ONE environment created ✓                                       │
│          (Server recognized duplicate request)                           │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Implementation Approaches

**1. Idempotency Key (Client-Provided)**
```python
def create_environment(request):
    idempotency_key = request.headers.get('Idempotency-Key')

    # Check if we've seen this key before
    existing = db.query(
        "SELECT * FROM idempotency_cache WHERE key = %s",
        idempotency_key
    )

    if existing:
        return existing.response  # Return cached response

    # Process request
    result = do_create_environment(request.body)

    # Cache response
    db.execute(
        "INSERT INTO idempotency_cache (key, response, expires_at) VALUES (%s, %s, %s)",
        idempotency_key, result, datetime.utcnow() + timedelta(hours=24)
    )

    return result
```

**2. Natural Idempotency (Operation is inherently safe)**
```python
# DELETE is naturally idempotent
def delete_environment(env_id):
    env = get_environment(env_id)
    if env is None or env.status == 'DELETED':
        return {"status": "already deleted"}  # Safe to call multiple times

    do_delete(env)
    return {"status": "deleted"}
```

**3. Conditional Updates (Optimistic Locking)**
```python
def update_environment(env_id, new_config, expected_version):
    result = db.execute("""
        UPDATE environments
        SET config = %s, version = version + 1
        WHERE id = %s AND version = %s
    """, new_config, env_id, expected_version)

    if result.rows_affected == 0:
        raise ConflictError("Version mismatch - resource was modified")
```

### OHCP Idempotency Examples

| Operation | Idempotency Approach |
|-----------|---------------------|
| Create environment | Idempotency key from client |
| Delete environment | Natural (deleting deleted = no-op) |
| Update config | Version/ETag check |
| Start environment | Check current state first |
| Workflow step execution | Step ID as idempotency key |

---

## 5. Consistency Models

### The Spectrum

```
STRONG ◀─────────────────────────────────────────────────────▶ EVENTUAL

┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Linearizable │  │  Sequential  │  │    Causal    │  │   Eventual   │
│              │  │              │  │              │  │              │
│  - Strictest │  │  - Ordered   │  │  - Preserves │  │  - Fastest   │
│  - Slowest   │  │    per client│  │    causality │  │  - May read  │
│  - Read own  │  │  - Can read  │  │  - Good for  │  │    stale     │
│    writes    │  │    stale     │  │    most apps │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### When to Use What

| Use Case | Consistency | Why |
|----------|-------------|-----|
| Quota checks | Strong | Can't over-provision |
| Environment state | Strong | Must reflect reality |
| Audit logs | Eventual | Can tolerate delay |
| Usage metrics | Eventual | Approximate is fine |
| User preferences | Causal | User should see own changes |
| Dashboard status | Eventual | Slight staleness OK |

### Read-Your-Writes Consistency

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    READ-YOUR-WRITES PATTERN                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Problem: User writes to primary, reads from replica before replication  │
│                                                                           │
│  User ──write──▶ Primary DB ──replicates──▶ Replica DB                  │
│       ◀──read stale────────────────────────────┘                         │
│                                                                           │
│  Solutions:                                                               │
│                                                                           │
│  1. Read from primary after write (for N seconds)                        │
│     - Track last write timestamp per user                                │
│     - If within N seconds, route to primary                              │
│                                                                           │
│  2. Include version in response, require in subsequent reads             │
│     - Write returns: { version: 5 }                                      │
│     - Read with: If-Min-Version: 5                                       │
│     - Replica waits until it has version 5                               │
│                                                                           │
│  3. Session stickiness                                                    │
│     - Route all requests from same session to same replica               │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Common Patterns

### 6.1 Saga Pattern

For distributed transactions across multiple services.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         SAGA PATTERN                                      │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  "Create Environment" Saga:                                              │
│                                                                           │
│  Step 1: Reserve Quota ──────▶ Success ──▶ Step 2                       │
│              │                                                            │
│              ▼ Failure                                                    │
│          (Nothing to undo)                                                │
│                                                                           │
│  Step 2: Create VPC ─────────▶ Success ──▶ Step 3                       │
│              │                                                            │
│              ▼ Failure                                                    │
│          Compensate: Release Quota                                       │
│                                                                           │
│  Step 3: Create Database ────▶ Success ──▶ Step 4                       │
│              │                                                            │
│              ▼ Failure                                                    │
│          Compensate: Delete VPC, Release Quota                           │
│                                                                           │
│  Step 4: Deploy App ─────────▶ Success ──▶ DONE                         │
│              │                                                            │
│              ▼ Failure                                                    │
│          Compensate: Delete DB, Delete VPC, Release Quota                │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Outbox Pattern

For reliable event publishing with database transactions.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        OUTBOX PATTERN                                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Problem: Need to update DB AND publish event atomically                 │
│                                                                           │
│  ❌ Wrong way:                                                            │
│     1. Update database ✓                                                 │
│     2. Publish to Kafka ✗ (fails - now DB and events are inconsistent)  │
│                                                                           │
│  ✓ Outbox pattern:                                                       │
│     1. BEGIN TRANSACTION                                                 │
│        - Update entity in main table                                     │
│        - Insert event into outbox table                                  │
│     2. COMMIT                                                             │
│                                                                           │
│     3. Separate process reads outbox, publishes to Kafka, marks done    │
│                                                                           │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                │
│  │  Service    │────▶│  Database   │     │   Kafka     │                │
│  │             │     │ ┌─────────┐ │     │             │                │
│  │ 1. Write    │     │ │ Entity  │ │     │             │                │
│  │    entity + │     │ └─────────┘ │     │             │                │
│  │    outbox   │     │ ┌─────────┐ │     │             │                │
│  │             │     │ │ Outbox  │─┼────▶│  3. Event   │                │
│  └─────────────┘     │ └─────────┘ │     │     published               │
│                      │      ▲      │     │             │                │
│                      └──────┼──────┘     └─────────────┘                │
│                             │                                            │
│                      2. Outbox processor                                 │
│                         reads & publishes                                │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Circuit Breaker

Prevent cascading failures when downstream is unhealthy.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       CIRCUIT BREAKER                                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│                    ┌─────────────────┐                                   │
│                    │                 │                                   │
│         success    │     CLOSED      │  failure threshold               │
│        ┌──────────▶│   (normal)      │────────────────┐                 │
│        │           │                 │                │                  │
│        │           └─────────────────┘                │                  │
│        │                                              ▼                  │
│        │           ┌─────────────────┐         ┌─────────────────┐      │
│        │           │                 │         │                 │      │
│        │           │   HALF-OPEN     │◀────────│     OPEN        │      │
│        │           │   (testing)     │  timeout│   (failing)     │      │
│        │           │                 │         │                 │      │
│        │           └────────┬────────┘         └─────────────────┘      │
│        │                    │                           │                │
│        │              test request                      │                │
│        │                    │                     fast fail              │
│        └────────────────────┘                     all requests           │
│              success                                                     │
│                                                                           │
│  OHCP Example:                                                           │
│  - Calling downstream provisioning service                               │
│  - If it fails 5 times in 1 minute → OPEN circuit                       │
│  - Return "service unavailable" without calling                         │
│  - After 30 seconds → HALF-OPEN, try one request                        │
│  - If succeeds → CLOSED, resume normal operation                        │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.4 Bulkhead

Isolate failures to prevent resource exhaustion.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         BULKHEAD PATTERN                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Without bulkhead:                                                       │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │  Shared Thread Pool (100 threads)                           │        │
│  │                                                              │        │
│  │  Service A calls ████████████████████████████████ (slow)    │        │
│  │  Service B calls ██ (starved - no threads left!)            │        │
│  │  Service C calls ██ (starved)                               │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                           │
│  With bulkhead:                                                          │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐         │
│  │ Service A Pool   │ │ Service B Pool   │ │ Service C Pool   │         │
│  │ (40 threads)     │ │ (30 threads)     │ │ (30 threads)     │         │
│  │                  │ │                  │ │                  │         │
│  │ ████████████████ │ │ ██████           │ │ ████             │         │
│  │ (A is slow but   │ │ (B works fine)   │ │ (C works fine)   │         │
│  │  contained)      │ │                  │ │                  │         │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘         │
│                                                                           │
│  OHCP Example:                                                           │
│  - Separate thread pools for each downstream service                    │
│  - If EHR provisioning is slow, it doesn't block Portal provisioning   │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Failure Handling

### Types of Failures

| Type | Example | Handling |
|------|---------|----------|
| **Transient** | Network timeout, 503 | Retry with backoff |
| **Permanent** | 400 Bad Request, 404 | Don't retry, fail fast |
| **Partial** | 3 of 5 services succeeded | Saga compensation |
| **Byzantine** | Corrupted data, wrong response | Validation, checksums |

### Retry Strategies

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       RETRY STRATEGIES                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. IMMEDIATE RETRY                                                      │
│     Attempt: 1 ─▶ 2 ─▶ 3 ─▶ FAIL                                        │
│     Delay:    0    0    0                                                │
│     Use: Unlikely to help, rarely used                                   │
│                                                                           │
│  2. FIXED DELAY                                                          │
│     Attempt: 1 ─▶ 2 ─▶ 3 ─▶ FAIL                                        │
│     Delay:   1s   1s   1s                                                │
│     Use: Simple, predictable                                             │
│                                                                           │
│  3. EXPONENTIAL BACKOFF                                                  │
│     Attempt: 1 ─▶ 2 ─▶ 3 ─▶ 4 ─▶ FAIL                                   │
│     Delay:   1s   2s   4s   8s                                           │
│     Use: Most common, gives system time to recover                       │
│                                                                           │
│  4. EXPONENTIAL BACKOFF + JITTER                                         │
│     Attempt: 1 ─▶ 2 ─▶ 3 ─▶ 4 ─▶ FAIL                                   │
│     Delay:  0.8s 2.3s 3.7s 9.1s  (randomized)                           │
│     Use: BEST - prevents thundering herd                                 │
│                                                                           │
│  Implementation:                                                          │
│  delay = min(base * (2 ** attempt) + random(0, 1), max_delay)           │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Dead Letter Queue (DLQ)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      DEAD LETTER QUEUE                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                │
│  │   Queue     │────▶│   Worker    │────▶│   Success   │                │
│  │             │     │             │     │             │                │
│  └─────────────┘     └──────┬──────┘     └─────────────┘                │
│                             │                                            │
│                        Fails 3x                                          │
│                             │                                            │
│                             ▼                                            │
│                      ┌─────────────┐                                    │
│                      │    DLQ      │──▶ Alert ops                       │
│                      │             │──▶ Manual review                   │
│                      │             │──▶ Fix & replay                    │
│                      └─────────────┘                                    │
│                                                                           │
│  OHCP Example:                                                           │
│  - Subscription event fails to process after 5 retries                  │
│  - Move to DLQ instead of blocking queue                                │
│  - Alert on-call, investigate, fix data, replay                         │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Scaling Strategies

### Horizontal vs Vertical

| Horizontal (Scale Out) | Vertical (Scale Up) |
|-----------------------|---------------------|
| Add more machines | Add more resources to machine |
| Better for stateless services | Simpler, has limits |
| Requires load balancing | No distribution complexity |
| Linear cost scaling | Exponential cost at high end |

### Scaling Different Components

| Component | Scaling Approach |
|-----------|------------------|
| **Stateless APIs** | Horizontal, behind load balancer |
| **Databases** | Read replicas, sharding, vertical |
| **Queues** | Partitioning, more consumers |
| **Caches** | Distributed cache (Redis Cluster) |
| **Workflows** | More workers, partition by tenant |

### Sharding Strategies

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      SHARDING STRATEGIES                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. RANGE-BASED                                                          │
│     Tenant A-M → Shard 1                                                 │
│     Tenant N-Z → Shard 2                                                 │
│     ❌ Problem: Uneven distribution (more hospitals start with "S")     │
│                                                                           │
│  2. HASH-BASED                                                           │
│     Shard = hash(tenant_id) % num_shards                                │
│     ✓ Even distribution                                                  │
│     ❌ Resharding is painful                                             │
│                                                                           │
│  3. CONSISTENT HASHING                                                   │
│     Hash ring with virtual nodes                                         │
│     ✓ Adding/removing shards only moves some data                       │
│     ✓ Good for caches, distributed storage                              │
│                                                                           │
│  4. DIRECTORY-BASED                                                      │
│     Lookup table: tenant_id → shard_id                                  │
│     ✓ Full control over placement                                       │
│     ✓ Can move tenants easily                                           │
│     ❌ Lookup service is dependency                                      │
│                                                                           │
│  OHCP Recommendation: Directory-based                                    │
│  - Can isolate large tenants to dedicated shards                        │
│  - Can move noisy tenants                                                │
│  - Can comply with data residency requirements                          │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Rate Limiting

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       RATE LIMITING                                       │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Token Bucket Algorithm:                                                  │
│                                                                           │
│  ┌─────────────────────────┐                                            │
│  │  Bucket (capacity: 10)  │                                            │
│  │  ┌───────────────────┐  │   Refill: 1 token/second                   │
│  │  │ 🪙🪙🪙🪙🪙🪙🪙      │  │                                            │
│  │  │ (7 tokens)        │  │   Request needs 1 token                    │
│  │  └───────────────────┘  │                                            │
│  └─────────────────────────┘                                            │
│                                                                           │
│  Request arrives:                                                        │
│  - Has token? → Allow request, remove token                             │
│  - No token? → Reject with 429 Too Many Requests                        │
│                                                                           │
│  OHCP Rate Limits:                                                       │
│  - Per tenant: 100 requests/minute to control plane                     │
│  - Per operation: 5 environment creates/hour                            │
│  - Global: 10,000 requests/minute across all tenants                    │
│                                                                           │
│  Response headers:                                                        │
│  X-RateLimit-Limit: 100                                                  │
│  X-RateLimit-Remaining: 45                                               │
│  X-RateLimit-Reset: 1623456789                                           │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card

### Control Plane vs Data Plane
- **Control Plane**: Management operations (create, configure, delete)
- **Data Plane**: User data operations (read records, serve requests)

### Sync vs Async
- **Sync**: Fast operations, user needs immediate response
- **Async**: Long operations, can notify later

### Idempotency
- Same request multiple times = same result
- Use idempotency keys for non-idempotent operations

### Consistency
- **Strong**: For quotas, state, anything that can't be wrong
- **Eventual**: For metrics, logs, dashboards

### Key Patterns
- **Saga**: Distributed transactions with compensation
- **Outbox**: Reliable event publishing
- **Circuit Breaker**: Prevent cascading failures
- **Bulkhead**: Isolate failure domains

### Failure Handling
- Retry with exponential backoff + jitter
- Dead letter queue for poison messages
- Fail fast for permanent errors

### Scaling
- Stateless services: Scale horizontally
- Databases: Read replicas, then shard
- Rate limit to protect the system
