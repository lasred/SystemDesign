# OHCP/OHMP System Design Interview Problems

Based on Oracle Health Control Plane / Management Plane responsibilities:
- Control plane for healthcare product provisioning (EHR, Patient Portal, Clinical AI)
- Async workflow orchestration with multi-product fan-out
- Kiev entity storage, RASL subscription lifecycle integration
- Environment groups (OHEGs) and environments (OHEs)
- Lifecycle state machines (CREATING → ACTIVE → DELETING)

---

## Problem 1: Workflow Orchestration Engine
**Difficulty: Medium** | **Focus: Core OHCP Pattern**

### Scenario
Design a workflow orchestration system that provisions healthcare environments by coordinating multiple downstream services.

### Requirements
- Execute multi-step provisioning workflows with parallel and sequential tasks
- Support polling/waiting for long-running downstream operations (5-30 min)
- Handle retries, timeouts, and partial failures
- Provide dry-run mode to validate without executing
- Track workflow state persistently (survives crashes)
- Support workflow cancellation mid-execution

### Scale
- 500 concurrent workflows
- Each workflow has 5-15 steps
- Downstream services have 99.5% availability

### Key Concepts to Discuss
- Saga pattern vs. 2PC
- State machines and checkpointing
- Idempotency tokens
- Compensation/rollback logic
- Poison pill handling

### Sample Questions
1. How do you handle a workflow that's 80% complete when a downstream service goes down for 2 hours?
2. How do you implement dry-run without duplicating all your logic?
3. How do you prevent duplicate resource creation on retry?

---

## Problem 2: Subscription Lifecycle Event Processor
**Difficulty: Medium** | **Focus: RASL-like Integration**

### Scenario
Design a system that processes subscription lifecycle events from an external billing/subscription system and orchestrates environment changes.

### Requirements
- Handle events: PROVISION, SUSPEND, RESUME, TERMINATE
- Guarantee exactly-once processing
- Handle out-of-order events (e.g., TERMINATE arrives before PROVISION completes)
- Coordinate state changes with multiple downstream services
- Provide event replay capability for debugging

### Scale
- 10K+ subscription changes per day
- Events must be processed within 5 minutes of receipt
- Zero data loss requirement

### Key Concepts to Discuss
- Event sourcing
- Idempotency keys
- State reconciliation
- Dead letter queues
- Outbox pattern

### Sample Questions
1. A SUSPEND event arrives while PROVISION is still running. What happens?
2. How do you handle events that fail processing after 5 retries?
3. How do you replay events without re-triggering side effects?

---

## Problem 3: Multi-Tenant Environment Isolation Platform
**Difficulty: Hard** | **Focus: Data Model & Access Control**

### Scenario
Design the data model and access control system for managing isolated healthcare environments for multiple hospital customers.

### Requirements
- Environment Groups (OHEGs) contain multiple Environments (OHEs)
- Each environment has resources across 4+ downstream services
- Strict tenant isolation (Hospital A cannot access Hospital B's data)
- Hierarchical permissions (Org Admin → Env Group Admin → Env Admin → User)
- Support environment "move" between groups
- Audit all access attempts

### Scale
- 10K+ organizations
- 100K+ environments
- 1M+ daily API calls

### Key Concepts to Discuss
- Multi-tenancy patterns (silo vs. pool vs. hybrid)
- RBAC vs. ABAC
- Compartment hierarchies
- Resource tagging and filtering
- Tenant context propagation

### Sample Questions
1. How do you prevent a bug in your code from leaking data across tenants?
2. How do you implement "move environment to different group" without downtime?
3. How do you handle a query like "list all environments I have access to" efficiently?

---

## Problem 4: Distributed Entity Store for Control Plane State
**Difficulty: Hard** | **Focus: Kiev-like Storage**

### Scenario
Design a distributed key-value/entity store optimized for control plane entity management.

### Requirements
- Store control plane entities (environments, subscriptions, workflows)
- Strongly consistent reads within a region
- Entity versioning with optimistic concurrency control
- Change notifications (watch/subscribe to entity updates)
- Support composite keys and secondary indexes
- Point-in-time recovery

### Scale
- 50K reads/sec, 5K writes/sec
- 10M+ entities
- 99.99% availability target

### Key Concepts to Discuss
- Consensus protocols (Paxos/Raft)
- Sharding strategies
- Version vectors / ETags
- Watch/notify implementation
- LSM trees vs B-trees

### Sample Questions
1. How do you implement "read-your-writes" consistency?
2. How do you handle a watch that has 10K subscribers?
3. How do you shard by tenant while supporting cross-tenant admin queries?

---

## Problem 5: Control Plane High Availability & Disaster Recovery
**Difficulty: Hard** | **Focus: HA/DR Architecture**

### Scenario
Design the HA/DR strategy for a control plane managing healthcare environments across multiple regions.

### Requirements
- RPO < 1 minute, RTO < 15 minutes
- Active-passive across regions (with considerations for active-active)
- Handle "split-brain" scenarios gracefully
- Downstream services may be in different regions
- HIPAA compliance: full audit trail, no data loss

### Scale
- 3 regions (primary + 2 DR)
- 1000s of environments being actively managed
- Must handle region-wide outages

### Key Concepts to Discuss
- Leader election and fencing
- Cross-region replication (sync vs. async)
- Failover orchestration
- Consistency vs. availability tradeoffs
- Runbook automation

### Sample Questions
1. Primary region goes down mid-workflow. How do you recover?
2. How do you prevent both regions from thinking they're the leader?
3. How do you test DR without impacting production?

---

## Problem 6: Service Availability Matrix & Dependency Management
**Difficulty: Medium** | **Focus: Feature/Region Availability**

### Scenario
Design a system that tracks which products and features are available in which regions and controls provisioning accordingly.

### Requirements
- Track product availability per region (EHR available in us-phoenix-1, not in eu-frankfurt-1)
- Block provisioning if required dependencies aren't available
- Support gradual rollouts (10% → 50% → 100%)
- Real-time availability status for API callers
- Distinguish "soft" dependencies (degraded experience) vs "hard" (blocks provisioning)

### Scale
- 50 products/features
- 20 regions
- 1000s of availability checks per minute

### Key Concepts to Discuss
- Feature flags at scale
- Dependency graphs
- Circuit breakers
- Configuration management and propagation
- Rollout strategies (canary, percentage-based)

### Sample Questions
1. How do you handle circular dependencies?
2. How do you roll back a feature that's causing issues?
3. How do you ensure config changes propagate to all nodes within seconds?

---

## Problem 7: Async Status Aggregation Service
**Difficulty: Medium-Easy** | **Focus: API Design & Performance**

### Scenario
Design an API that returns aggregated environment health status by querying multiple downstream services.

### Requirements
- Environment health depends on 4+ downstream services
- Each downstream has its own health/status endpoint
- API responses must return in < 500ms even if downstream is slow
- Show granular status: "healthy" vs "degraded" vs "down"
- Don't show stale data older than 30 seconds

### Scale
- 10K status checks per minute
- 4-6 downstream services per check
- Downstream P99 latency: 200-2000ms

### Key Concepts to Discuss
- Fan-out/fan-in patterns
- Timeout and fallback strategies
- Caching with TTL
- Health check aggregation logic
- Circuit breakers for downstream services

### Sample Questions
1. Downstream service is timing out. Do you show "unknown" or cached "healthy"?
2. How do you prevent a thundering herd when cache expires?
3. How do you handle a downstream that's flapping (up/down/up/down)?

---

## How to Use These Problems

### For Practice
1. Pick a problem and set a 45-minute timer
2. Spend first 5 minutes clarifying requirements (write down assumptions)
3. Draw the high-level architecture (boxes and arrows)
4. Deep dive into 2-3 components
5. Discuss tradeoffs and alternatives

### For Interview Prep
- Problems 1, 2, 7 are most directly relevant to day-to-day OHCP work
- Problems 3, 4, 5 test deeper infrastructure knowledge
- Be ready to discuss: idempotency, state machines, async patterns, failure handling

### Diagram Suggestions
- Use draw.io, Excalidraw, or whiteboard
- Start with: Client → API Gateway → Control Plane → Downstream Services
- Add: Database, Message Queue, Workflow Engine as needed
- Show data flow for happy path first, then failure scenarios
