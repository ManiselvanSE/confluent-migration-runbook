# Confluent Cloud Migration: Enterprise Guide
## Single-Zone to Multi-Zone Migration Using Cluster Linking

---

# 1. EXECUTIVE SUMMARY

## What is Confluent Cloud Migration?

Confluent Cloud migration is the process of moving Kafka workloads from a **single availability zone** (single-zone Dedicated cluster) to a **multi-zone architecture** (Enterprise cluster spanning three availability zones). This transformation elevates your streaming platform from basic availability to enterprise-grade resilience.

## Why Organizations Migrate

### Business Drivers

**Risk Mitigation**: Single-zone deployments create a single point of failure. An availability zone outage means complete service disruption, directly impacting revenue-generating applications and customer experience.

**Compliance & SLAs**: Regulated industries (financial services, healthcare) mandate 99.99% uptime and zero data loss. Multi-zone architecture provides the infrastructure to meet these requirements.

**Business Continuity**: Modern businesses operate 24/7 globally. Multi-zone deployments ensure streaming pipelines continue operating even during infrastructure failures, maintenance windows, or regional incidents.

**Cost of Downtime**: For high-velocity businesses (e-commerce, fintech, real-time analytics), even 5 minutes of downtime can cost tens of thousands in lost transactions and customer trust.

### Technical Drivers

**Durability Improvement**: Single-zone clusters typically run with replication factor 1-2, meaning data exists on only 1-2 servers. Multi-zone architecture distributes 3 copies across separate physical locations, surviving complete zone failures.

**Availability Enhancement**: Zone-level failures are rare but real (power failures, network outages, infrastructure maintenance). Multi-zone deployments continue operating when one zone fails.

**Scalability Requirements**: As data volumes grow, single-zone infrastructure hits capacity limits. Enterprise multi-zone clusters provide horizontal scaling across zones.

**Regulatory Requirements**: Data residency and disaster recovery regulations increasingly require multi-zone or multi-region architectures with proven failover capabilities.

## Systems Involved

The migration touches all components of your streaming ecosystem:

- **Kafka Clusters**: Core message brokers storing and routing event streams
- **Schema Registry**: Manages data schemas and evolution policies
- **Kafka Connect**: Integrates external systems (databases, cloud storage, SaaS applications)
- **ksqlDB** (if used): Stream processing engine for real-time analytics
- **Producer Applications**: Services writing events to Kafka
- **Consumer Applications**: Services reading and processing events
- **Security Infrastructure**: ACLs, API keys, role-based access controls

## Target State Architecture

**Current State**: 
- Single availability zone hosting all brokers
- Replication factor: 1-2 (limited redundancy)
- Zone failure = complete outage
- RTO (Recovery Time): Hours to restore from backups
- RPO (Recovery Point): Potential data loss since last backup

**Target State**:
- Three availability zones, each hosting broker replicas
- Replication factor: 3 (one replica per zone)
- Zone failure = zero downtime, automatic failover
- RTO: <5 minutes (consumer restart time)
- RPO: 0 (no data loss)
- 99.99% uptime SLA capability

## Migration Outcome

Upon completion, you achieve:

✅ **Zero Data Loss Architecture**: Every message replicated to 3 separate zones before acknowledgment  
✅ **Zone-Level Fault Tolerance**: Survive complete zone failures without service interruption  
✅ **Transparent Application Migration**: Consumers resume from exact offsets, producers continue without message loss  
✅ **Security Continuity**: All access controls, API keys, and permissions preserved  
✅ **Performance Optimization**: Modern enterprise cluster with auto-scaling and load balancing  

**Business Impact**: Transforms Kafka from a performance-focused platform to a resilient, enterprise-grade streaming backbone supporting mission-critical workloads.

---

# 2. SOLUTION ENGINEER VIEW (CONCEPTUAL DESIGN)

## Current State vs. Target State

### Current State: Single-Zone Dedicated Cluster

```
┌─────────────────────────────────────────┐
│     Single Availability Zone (AZ-A)     │
│                                          │
│  ┌──────┐  ┌──────┐  ┌──────┐          │
│  │Broker│  │Broker│  │Broker│          │
│  │  1   │  │  2   │  │  3   │          │
│  └──────┘  └──────┘  └──────┘          │
│                                          │
│  Replication Factor: 1-2                │
│  Durability: Single zone only           │
│  Availability: 99.5% - 99.9%            │
│                                          │
│  ⚠️  Zone Failure = Complete Outage     │
└─────────────────────────────────────────┘

Producers → Single Zone → Consumers
```

**Limitations**:
- No zone-level redundancy
- Data loss risk on zone failure
- Limited scalability ceiling
- No automatic failover

### Target State: Multi-Zone Enterprise Cluster

```
┌────────────────────────────────────────────────────────────────┐
│              Multi-Zone Enterprise Cluster                      │
│                                                                  │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│  │   Zone A    │   │   Zone B    │   │   Zone C    │          │
│  │             │   │             │   │             │          │
│  │  ┌──────┐   │   │  ┌──────┐   │   │  ┌──────┐   │          │
│  │  │Broker│   │   │  │Broker│   │   │  │Broker│   │          │
│  │  │  1   │◄──┼───┼─►│  2   │◄──┼───┼─►│  3   │   │          │
│  │  └──────┘   │   │  └──────┘   │   │  └──────┘   │          │
│  │             │   │             │   │             │          │
│  │  Replica 1  │   │  Replica 2  │   │  Replica 3  │          │
│  │  (Leader)   │   │  (Follower) │   │  (Follower) │          │
│  └─────────────┘   └─────────────┘   └─────────────┘          │
│                                                                  │
│  Replication Factor: 3 (distributed)                            │
│  min.insync.replicas: 2 (durability guarantee)                  │
│  Availability: 99.99%                                           │
│                                                                  │
│  ✅ Zone Failure = Zero Downtime (failover to Zones B & C)     │
└────────────────────────────────────────────────────────────────┘

Producers → Multi-Zone Cluster → Consumers
         (acks=all waits for 2+ zones)
```

**Improvements**:
- Survives complete zone failures
- Zero data loss with min.insync.replicas=2
- Automatic leader election and failover
- Horizontal scalability across zones

## High-Level Migration Approach

### Strategy: Active-Passive Cluster Linking

**Not** a lift-and-shift. **Not** a rebuild. This is a **synchronized replication migration**.

```
Phase 1: REPLICATE
┌─────────────┐         Cluster Link         ┌─────────────┐
│   Source    │─────────────────────────────►│ Destination │
│ (Single AZ) │    Continuous Replication    │ (Multi AZ)  │
│             │    Offset Translation        │             │
│ ACTIVE      │    ACL Sync                  │ STANDBY     │
└─────────────┘                               └─────────────┘
   ↑                                                 │
Producers/Consumers                            Read-only
   (Current Traffic)                             (Syncing)

Phase 2: CUTOVER (4-hour window)
┌─────────────┐         Cluster Link         ┌─────────────┐
│   Source    │         (Draining)            │ Destination │
│ (Single AZ) │                               │ (Multi AZ)  │
│             │                               │             │
│ DRAINING    │                               │ ACTIVE      │
└─────────────┘                               └─────────────┘
                                                     ↑
                                          Producers/Consumers
                                          (Switched Traffic)

Phase 3: DECOMMISSION (7-14 days later)
                                              ┌─────────────┐
Source Cluster Deleted                        │ Destination │
                                              │ (Multi AZ)  │
                                              │             │
                                              │ PRODUCTION  │
                                              └─────────────┘
                                                     ↑
                                          All Production Traffic
```

### Migration Timeline

**Total Duration**: 10 weeks (typical for 100TB cluster, 500 topics)

- **Weeks 1-2**: Assessment and planning
- **Week 3**: Staging environment rehearsal
- **Week 4**: Provision multi-zone cluster and establish replication
- **Weeks 4-5**: Wait for full data synchronization (lag → zero)
- **Week 5**: Application testing against destination
- **Week 6**: Production cutover (4-hour execution window)
- **Weeks 6-8**: Monitoring and validation period
- **Week 10**: Decommission source cluster

## Key Components Involved

### 1. Kafka Clusters

**Source Cluster** (Single-Zone Dedicated):
- Hosts current production workloads
- Remains active during replication phase
- Decommissioned after validation period

**Destination Cluster** (Multi-Zone Enterprise):
- Provisioned in parallel to source
- Receives replicated data via Cluster Link
- Becomes production after cutover

### 2. Cluster Linking Technology

Confluent's enterprise replication mechanism:

- **Continuous Replication**: Mirrors topics from source to destination in real-time
- **Offset Translation**: Automatically converts consumer offsets to destination cluster equivalents
- **Configuration Sync**: Replicates topic settings, ACLs, and schemas
- **Zero Code Changes**: Applications unaware of underlying migration

### 3. Schema Registry

Shared service across both clusters:
- Regional Schema Registry serves both source and destination
- No schema migration required
- Ensures schema compatibility during and after migration

### 4. Kafka Connect

Integration layer connecting Kafka to external systems:
- **Source Connectors**: Pull data from databases, cloud storage, SaaS apps into Kafka
- **Sink Connectors**: Push Kafka data to external systems

**Migration Requirement**: Reconfigure connectors to point to new cluster bootstrap servers

### 5. Producer & Consumer Applications

**Producers**: Applications writing events to Kafka
- Migration: Update bootstrap server configuration
- Strategy: Phased rollout (5% → 25% → 50% → 100%)

**Consumers**: Applications processing events from Kafka
- Migration: Update bootstrap server + validate offset translation
- Critical: Zero reprocessing or message loss

## Data Flow Overview

### During Replication Phase

```
┌────────────────────────────────────────────────────────────────┐
│                     DATA FLOW TIMELINE                          │
└────────────────────────────────────────────────────────────────┘

1. Producers write to SOURCE cluster
   │
   ▼
2. Cluster Link replicates to DESTINATION cluster
   │  - Topic data
   │  - Consumer offsets
   │  - ACLs
   ▼
3. Mirror lag decreases: 1M messages → 1K → 100 → 0
   │
   ▼
4. CUTOVER READY (lag stable at near-zero for 30+ min)
```

### During Cutover

```
Minute 0:   Pause Kafka Connect connectors
Minute 2:   Start switching producers (canary 5%)
Minute 5:   Validate canary, proceed with full producer switch
Minute 12:  All producers writing to DESTINATION
Minute 15:  Gracefully stop consumers from SOURCE
Minute 17:  Verify consumer offsets translated correctly
Minute 18:  Start consumers on DESTINATION (resume from exact offset)
Minute 25:  Resume connectors with new cluster config
Minute 30:  Cutover complete, begin monitoring
```

### After Migration

```
All traffic flows through DESTINATION cluster:

Producers → Multi-Zone Cluster → Consumers
              (3 AZs, RF=3)
              
External Systems → Kafka Connect → Multi-Zone Cluster
```

## Risks and Constraints

### Downtime

**Target**: Near-zero downtime
- **Producers**: Zero downtime (rolling restart with new config)
- **Consumers**: <5 minutes (time to restart with destination config)
- **Kafka Connect**: 2-5 minutes (pause → reconfigure → resume)

**Actual Downtime**: Typically 3-7 minutes of consumer unavailability during cutover

### Compatibility

**Version Alignment**: Source and destination must run compatible Kafka versions
- Minor version differences acceptable (e.g., 2.8 → 3.0)
- Major version jumps require careful schema/protocol testing

**Client Compatibility**: Ensure producer/consumer libraries support destination Kafka version

### Scaling Constraints

**Network Bandwidth**: Cluster Link replication consumes bandwidth proportional to source cluster write throughput
- Plan for sustained cross-zone data transfer during sync phase
- Monitor network saturation

**Cost Impact**: Multi-zone architecture costs 30-40% more than single-zone
- Higher replication factor (3x storage for RF=3 vs RF=1)
- Cross-zone data transfer charges
- Enterprise cluster premium

### Performance Considerations

**Latency Increase**: Cross-zone replication adds 3-7ms to producer latency (p99)
- Single-zone: 10-15ms produce latency
- Multi-zone: 13-22ms produce latency
- Mitigated via producer batching and compression

**Throughput**: Multi-zone clusters provide higher aggregate throughput due to more brokers and zones

## Key Talking Points for Customer Discussions

### For Business Stakeholders

1. **"We're eliminating your biggest risk"**: Single-zone architecture is a ticking time bomb. Zone failures are rare but catastrophic. Multi-zone provides peace of mind.

2. **"Near-zero downtime migration"**: We're not rebuilding—we're replicating in parallel. Your business continues operating throughout the migration.

3. **"Investment in resilience, not just technology"**: 30-40% cost increase buys insurance against revenue-impacting outages. Calculate downtime cost vs. migration cost.

4. **"Compliance-ready architecture"**: Multi-zone meets regulatory requirements for financial services, healthcare, and other regulated industries.

### For Technical Stakeholders

1. **"Cluster Linking is battle-tested"**: Confluent's replication technology powers thousands of enterprise migrations. Automatic offset translation prevents consumer reprocessing.

2. **"Minimal application changes"**: Only configuration updates required. No code changes. No message format changes.

3. **"Comprehensive rollback strategy"**: If issues arise, we can revert to source cluster in <15 minutes. Cluster Link remains active as safety net.

4. **"Performance optimized for multi-zone"**: Producer batching, compression, and tuning strategies maintain throughput despite cross-zone latency.

### For Security & Compliance Teams

1. **"Security posture preserved"**: All ACLs, RBAC policies, and API keys migrated automatically via Cluster Link.

2. **"Encryption maintained"**: TLS in-transit encryption throughout migration. Encryption at rest enabled on destination.

3. **"Audit trail complete"**: Full migration logs, validation reports, and cutover checklists for compliance documentation.

4. **"Zero data exposure"**: Replication occurs over private network (VPC peering or PrivateLink). No public internet exposure.

---

# 3. SOLUTION ARCHITECT VIEW (DETAILED DESIGN)

## A. Architecture Design

### Source Cluster Architecture (Single-Zone Dedicated)

```
┌─────────────────────────────────────────────────────────────┐
│               Source: Single-Zone Dedicated                  │
│                                                               │
│  Availability Zone: us-east-1a (example)                     │
│                                                               │
│  ┌─────────────────────────────────────────────────┐         │
│  │  Kafka Brokers (Dedicated CKUs)                 │         │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐        │         │
│  │  │ ID 1 │  │ ID 2 │  │ ID 3 │  │ ID 4 │        │         │
│  │  └──────┘  └──────┘  └──────┘  └──────┘        │         │
│  │                                                  │         │
│  │  Configuration:                                 │         │
│  │  - Replication Factor: 1 or 2                   │         │
│  │  - min.insync.replicas: 1                       │         │
│  │  - unclean.leader.election: false               │         │
│  └─────────────────────────────────────────────────┘         │
│                                                               │
│  Storage: EBS volumes (single AZ)                            │
│  Network: VPC private subnets                                │
│  Security: TLS encryption, SASL authentication               │
└─────────────────────────────────────────────────────────────┘

Connected Services:
- Schema Registry (regional, shared)
- Kafka Connect (managed or self-hosted)
- Producer/Consumer apps (VPC peered or PrivateLink)
```

**Architectural Characteristics**:

- **Fault Domain**: Single availability zone (1 failure domain)
- **Durability**: Limited to in-zone replication (RF=1 or 2)
- **Availability SLA**: 99.5% - 99.9% (zone-dependent)
- **Failure Mode**: Zone outage = complete cluster unavailability

### Target Cluster Architecture (Multi-Zone Enterprise)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Destination: Multi-Zone Enterprise                        │
│                                                                               │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐       │
│  │  Zone us-east-1a  │  │  Zone us-east-1b  │  │  Zone us-east-1c  │       │
│  │                   │  │                   │  │                   │       │
│  │  ┌──────┐         │  │  ┌──────┐         │  │  ┌──────┐         │       │
│  │  │Broker│         │  │  │Broker│         │  │  │Broker│         │       │
│  │  │ ID 1 │◄────────┼──┼─►│ ID 2 │◄────────┼──┼─►│ ID 3 │         │       │
│  │  └──────┘         │  │  └──────┘         │  │  └──────┘         │       │
│  │  ┌──────┐         │  │  ┌──────┐         │  │  ┌──────┐         │       │
│  │  │Broker│         │  │  │Broker│         │  │  │Broker│         │       │
│  │  │ ID 4 │◄────────┼──┼─►│ ID 5 │◄────────┼──┼─►│ ID 6 │         │       │
│  │  └──────┘         │  │  └──────┘         │  │  └──────┘         │       │
│  │                   │  │                   │  │                   │       │
│  │  EBS Storage      │  │  EBS Storage      │  │  EBS Storage      │       │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘       │
│                                                                               │
│  Configuration:                                                               │
│  - Replication Factor: 3 (rack-aware across zones)                           │
│  - min.insync.replicas: 2 (requires 2 zones to ack writes)                   │
│  - unclean.leader.election: false (never promote out-of-sync replica)        │
│  - rack.awareness: Replicas distributed one-per-zone                          │
└─────────────────────────────────────────────────────────────────────────────┘

Topic Partition Example:
  Partition 0: Leader on Broker 1 (Zone A), Followers on Broker 2 (Zone B), Broker 3 (Zone C)
  Partition 1: Leader on Broker 2 (Zone B), Followers on Broker 3 (Zone C), Broker 1 (Zone A)
  (Leaders distributed for load balancing)
```

**Architectural Characteristics**:

- **Fault Domains**: Three independent availability zones
- **Durability**: Cross-zone replication (RF=3)
- **Availability SLA**: 99.99% (survives single-zone failure)
- **Failure Mode**: Single zone failure = automatic failover, zero data loss

**Replica Distribution Logic**:

```
For topic "orders" with 12 partitions, 3 replicas each:

Partition 0:  [Broker 1 (Zone A) - Leader, Broker 2 (Zone B), Broker 3 (Zone C)]
Partition 1:  [Broker 2 (Zone B) - Leader, Broker 3 (Zone C), Broker 4 (Zone A)]
Partition 2:  [Broker 3 (Zone C) - Leader, Broker 4 (Zone A), Broker 5 (Zone B)]
...
(Leaders evenly distributed across zones for balanced load)
```

### Networking Considerations

#### VPC Architecture

**Option 1: VPC Peering** (AWS Example)

```
┌─────────────────────────────────────────────────────────┐
│              Customer VPC (Application VPC)              │
│                                                           │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐         │
│  │ Producers │   │ Consumers │   │  Connect  │         │
│  └─────┬─────┘   └─────┬─────┘   └─────┬─────┘         │
│        │               │               │               │
│        └───────────────┴───────────────┘               │
│                        │                                │
│                   VPC Peering                           │
│                        │                                │
└────────────────────────┼────────────────────────────────┘
                         │
┌────────────────────────┼────────────────────────────────┐
│                        │                                │
│           Confluent Cloud Multi-Zone VPC                │
│                        │                                │
│  ┌─────────────────────▼───────────────────────┐        │
│  │      Multi-Zone Kafka Cluster               │        │
│  │  (us-east-1a, us-east-1b, us-east-1c)       │        │
│  └──────────────────────────────────────────────┘        │
│                                                           │
│  Private IPs only, no internet exposure                  │
└───────────────────────────────────────────────────────────┘

Configuration Requirements:
- Peering connection accepted on both sides
- Route table updates for CIDR blocks
- Security group ingress: port 9092 (Kafka), 443 (Schema Registry)
- DNS resolution enabled (AWS: enableDnsSupport=true)
```

**Option 2: PrivateLink** (Recommended for Enterprise)

```
┌─────────────────────────────────────────────────────────┐
│              Customer VPC (Application VPC)              │
│                                                           │
│  ┌───────────┐   ┌───────────┐                          │
│  │ Producers │   │ Consumers │                          │
│  └─────┬─────┘   └─────┬─────┘                          │
│        │               │                                │
│        └───────────────┘                                │
│                │                                         │
│         VPC Endpoint (Interface)                        │
│                │                                         │
└────────────────┼─────────────────────────────────────────┘
                 │
            PrivateLink
                 │
┌────────────────┼─────────────────────────────────────────┐
│                │                                         │
│    PrivateLink Endpoint Service                         │
│                │                                         │
│  ┌─────────────▼───────────────────────┐                │
│  │  Multi-Zone Kafka Cluster           │                │
│  │  (Confluent Cloud Managed)          │                │
│  └──────────────────────────────────────┘                │
│                                                           │
│  Benefits:                                               │
│  - No CIDR overlap concerns                              │
│  - Simplified security (no route table changes)          │
│  - Private DNS names                                     │
└───────────────────────────────────────────────────────────┘
```

#### Security & TLS

**Encryption in Transit**:
- All Kafka broker communication: TLS 1.2+
- Client-to-broker: SASL_SSL (TLS + SASL authentication)
- Inter-broker replication: TLS encrypted
- Cluster Link replication: TLS encrypted over private network

**Authentication**:
- SASL/PLAIN with API keys (username + password)
- Service accounts per application/team
- API key rotation supported

**Authorization (ACLs)**:

```
ACL Model Example:

Producer Service Account: sa-producer-orders
  - Operation: WRITE
  - Resource: Topic "orders"
  - Permission: ALLOW

Consumer Service Account: sa-consumer-analytics
  - Operation: READ
  - Resource: Topic "orders"
  - Resource: Consumer Group "analytics-group"
  - Permission: ALLOW

Admin Service Account: sa-platform-admin
  - Operation: ALL
  - Resource: Cluster
  - Permission: ALLOW
```

**Network Security**:
- IP allowlists (if required for compliance)
- Private connectivity only (VPC peering or PrivateLink)
- No public internet exposure
- Security group rules limiting ingress to application VPCs

---

## B. Migration Strategy

### Component-Wise Migration Order

The migration follows a **dependency-based sequence** to ensure consistency and zero data loss.

#### Why Migration Order Matters

Incorrect sequencing causes:
- Consumer offset mismatches (consumers reprocess millions of messages)
- Schema incompatibilities (producers fail to serialize messages)
- ACL authorization failures (applications denied access)
- Connector failures (source/sink data loss)

**Correct Order** (dependency-driven):

```
1. Infrastructure & Networking (foundation)
   │
   ▼
2. Cluster Provisioning (destination cluster ready)
   │
   ▼
3. Security Setup (ACLs, API keys, RBAC)
   │
   ▼
4. Schema Registry Validation (schema compatibility confirmed)
   │
   ▼
5. Cluster Link Establishment (replication begins)
   │
   ▼
6. Data Synchronization (wait for lag → zero)
   │
   ▼
7. Producer Migration (write traffic switched)
   │
   ▼
8. Consumer Migration (read traffic switched, offset-aware)
   │
   ▼
9. Kafka Connect Migration (connectors reconfigured)
   │
   ▼
10. ksqlDB Migration (if applicable, state rebuild)
    │
    ▼
11. Validation & Monitoring (confirm zero data loss)
    │
    ▼
12. Source Cluster Decommission (after safety period)
```

### Detailed Component Migration

#### 1. Infrastructure & Networking (Week 4)

**Objective**: Establish secure connectivity between application VPC and Confluent Cloud multi-zone VPC

**Activities**:
- Provision VPC peering or PrivateLink connection
- Configure route tables for private CIDR routing
- Update security groups: allow port 9092 (Kafka), 443 (Schema Registry, Control Plane)
- Validate DNS resolution for bootstrap servers
- Test connectivity via `telnet` or `nc` to bootstrap endpoints

**Validation**:
- `telnet <destination-bootstrap-server> 9092` succeeds from application VPC
- DNS resolution returns private IPs

#### 2. Cluster Provisioning (Week 4)

**Objective**: Provision multi-zone Enterprise cluster matching source capacity

**Configuration**:
- Cluster type: Enterprise
- Availability: MULTI_ZONE
- Cloud/Region: Match source (e.g., AWS us-east-1)
- Capacity (CKUs): Match or exceed source cluster capacity
- Storage: Auto-scaling enabled

**Default Settings**:
```
default.replication.factor: 3
min.insync.replicas: 2
unclean.leader.election.enable: false
auto.create.topics.enable: false (prevent accidental topic creation)
```

**Validation**:
- Cluster status: PROVISIONED
- All 3 availability zones active
- Bootstrap URL accessible via private network

#### 3. Security Setup (Week 4)

**Objective**: Replicate source cluster security posture to destination

**Service Accounts**:
- Create service accounts matching source (e.g., sa-producer-app, sa-consumer-app)
- Generate API keys for each service account on destination cluster
- Store API keys securely (secrets manager, vault)

**ACL Replication**:
- Export ACLs from source cluster
- Enable ACL sync in Cluster Link (automatic replication)
- Validate ACLs replicated correctly via diff comparison

**RBAC** (if applicable):
- Export role bindings from source
- Recreate role bindings on destination

#### 4. Schema Registry Validation (Week 4)

**Objective**: Ensure Schema Registry accessible from destination cluster

**Activities**:
- Verify Schema Registry endpoint configured for destination cluster
- Validate schema compatibility mode matches source
- Test schema retrieval from destination cluster context
- Confirm no schema evolution breaking changes

**Validation**:
```bash
# List schemas accessible from destination context
curl -u <api-key>:<secret> https://<schema-registry-url>/subjects

# Verify compatibility mode
curl -u <api-key>:<secret> https://<schema-registry-url>/config
# Expected: {"compatibilityLevel":"BACKWARD"} (or FORWARD/FULL as per source)
```

#### 5. Cluster Link Establishment (Week 4)

**Objective**: Create bidirectional replication channel from source to destination

**Configuration**:
```
Link Name: prod-migration-link
Direction: Source → Destination
Mode: DESTINATION (initiated from destination cluster)

Settings:
- consumer.offset.sync.enable: true
- consumer.offset.sync.ms: 30000 (sync every 30 seconds)
- acl.sync.enable: true
- acl.sync.ms: 30000
```

**Mirror Topics**:
- Create mirror topics for all source topics
- Mirror topics are read-only (prevent accidental writes)
- Topic configs automatically replicated

**Validation**:
- Cluster link status: ACTIVE
- Mirror topics created successfully
- Replication lag decreasing

#### 6. Data Synchronization (Weeks 4-5)

**Objective**: Wait for mirror lag to reach near-zero across all topics

**Monitoring**:
- Track `mirror_lag_count` (number of messages behind)
- Track `mirror_lag_ms` (time lag in milliseconds)

**Readiness Criteria**:
- Mirror lag < 1000 messages for ALL topics
- Lag stable (not increasing) for 30+ minutes
- No mirror topics in ERROR state

**Timeline**: Typically 5-7 days for large clusters (100TB+) to fully sync

#### 7. Producer Migration (Week 6, Cutover Day)

**Objective**: Switch producer write traffic from source to destination

**Strategy**: Phased Rollout

```
Phase 1: Canary (5% of producers)
  - Deploy config change to 5% of producer instances
  - Monitor for 30 minutes
  - Metrics: error rate, latency, throughput
  - Rollback trigger: error rate >1%

Phase 2: Incremental (25%)
  - Deploy to additional 20% (total 25%)
  - Monitor for 15 minutes

Phase 3: Incremental (50%)
  - Deploy to additional 25% (total 50%)
  - Monitor for 15 minutes

Phase 4: Full Rollout (100%)
  - Deploy to remaining 50%
  - Monitor for 1 hour before proceeding to consumer migration
```

**Configuration Change**:
```
OLD: bootstrap.servers=<source-cluster-bootstrap>:9092
NEW: bootstrap.servers=<destination-cluster-bootstrap>:9092

OLD: sasl.jaas.config=... username='<source-api-key>' ...
NEW: sasl.jaas.config=... username='<destination-api-key>' ...
```

**Deployment Method**: Rolling restart or blue-green deployment (zero downtime)

#### 8. Consumer Migration (Week 6, Cutover Day)

**Objective**: Switch consumer read traffic to destination with exact offset preservation

**Critical Requirement**: Consumer offset translation MUST be validated before cutover

**Pre-Migration Validation**:
```
For each consumer group:
1. Retrieve current offset on source cluster
2. Retrieve translated offset on destination cluster
3. Compare (should be within replication lag window)
4. If mismatch > 1%, investigate before proceeding
```

**Migration Steps**:
```
1. Gracefully stop consumers on source cluster
   - Allow in-flight messages to complete processing
   - Commit final offsets

2. Verify final offset committed on source

3. Wait for Cluster Link to sync final offsets (30-60 seconds)

4. Update consumer configuration:
   - bootstrap.servers → destination cluster
   - API key → destination cluster key

5. Start consumers on destination cluster
   - Consumers automatically resume from translated offset
   - Zero reprocessing (if offset translation successful)
```

**Fallback**: If offset translation fails, manually reset to timestamp or specific offset

#### 9. Kafka Connect Migration (Week 6, Cutover Day)

**Objective**: Reconfigure connectors to use destination cluster

**Migration Steps**:

```
1. Pause all connectors (prevents data loss during reconfiguration)
   - Source connectors stop reading from external systems
   - Sink connectors stop writing to external systems

2. Export connector configurations

3. Update configurations:
   - kafka.bootstrap.servers → destination cluster
   - kafka.sasl.jaas.config → destination cluster API key

4. Apply updated configurations

5. Resume connectors
   - Validate tasks transition to RUNNING state
   - Monitor for errors
```

**Critical Note**: Connector offsets NOT automatically migrated by Cluster Link

**Connector Offset Handling**:
- **Source Connectors**: May reprocess from last committed offset or latest (depends on connector type)
- **Sink Connectors**: Resume from last committed Kafka offset (preserved via Cluster Link)

**Validation**:
- All connector tasks: RUNNING state
- No errors in connector logs
- Data flowing to/from external systems

#### 10. ksqlDB Migration (Week 6, Optional)

**Objective**: Migrate stream processing queries to destination cluster

**Challenge**: ksqlDB state stored in RocksDB and changelog topics (not auto-migrated)

**Strategy**: Rebuild

```
1. Export ksqlDB queries and schemas
   - SHOW STREAMS
   - SHOW TABLES
   - SHOW QUERIES
   - Save DDL statements

2. Provision new ksqlDB cluster connected to destination Kafka cluster

3. Recreate streams, tables, and queries
   - Execute saved DDL statements

4. Wait for state rebuild
   - ksqlDB reads from beginning of source topics
   - Rebuilds state stores
   - Can take hours for large state

5. Validate query output
```

**Alternative**: Advanced state migration (complex, not recommended without Confluent Professional Services)

---

## C. Data Consistency Strategy

### Replication Approach: Cluster Linking

**Technology**: Confluent Cluster Linking (enterprise replication)

**How It Works**:

```
Source Cluster                           Destination Cluster
┌─────────────────┐                      ┌─────────────────┐
│ Topic: orders   │                      │ Topic: orders   │
│ Partition 0     │                      │ Partition 0     │
│ ┌─────────────┐ │   Cluster Link       │ ┌─────────────┐ │
│ │Msg 1000-1500│─┼─────────────────────►│ │Msg 1000-1500│ │
│ └─────────────┘ │   (Async Replication)│ └─────────────┘ │
│                 │                      │                 │
│ Leader: Broker 1│                      │ Leader: Broker 4│
│ Offset: 1500    │                      │ Offset: 1500    │
└─────────────────┘                      └─────────────────┘

Replication Properties:
- Asynchronous (does not block source producers)
- Byte-for-byte copy (messages identical)
- Partition-level parallelism (each partition replicated independently)
- Checkpointing (tracks replication progress per partition)
```

**Consistency Guarantees**:

✅ **Message Ordering**: Preserved within each partition  
✅ **Message Content**: Byte-identical copies  
✅ **Topic Configuration**: Replicated (retention, compaction, partitions)  
✅ **Consumer Offsets**: Automatically translated  
✅ **ACLs**: Optionally synchronized  

❌ **Transactional State**: Not replicated (transactions are cluster-specific)  
❌ **Connector Offsets**: Not replicated (stored in internal topics not synced by default)  

### Offset Handling Strategy

**Challenge**: Consumer offsets on source cluster don't directly map to destination cluster offsets

**Solution**: Cluster Link Automatic Offset Translation

```
Source Cluster                    Destination Cluster
Consumer Group: analytics         Consumer Group: analytics
Topic: orders, Partition 0        Topic: orders, Partition 0
Current Offset: 50,000            Translated Offset: 50,000

┌───────────────────────────────────────────────────────────┐
│          Offset Translation Mechanism                      │
├───────────────────────────────────────────────────────────┤
│                                                             │
│  Source Offset 50,000 → Cluster Link → Dest Offset 50,000  │
│                                                             │
│  Translation Method:                                        │
│  - Cluster Link tracks message correspondence               │
│  - Maps source offset to equivalent destination offset      │
│  - Syncs every 30 seconds (configurable)                    │
│                                                             │
│  Consumer Cutover:                                          │
│  1. Consumer stops on source at offset 50,000              │
│  2. Consumer starts on destination                          │
│  3. Cluster Link provides translated offset: 50,000        │
│  4. Consumer resumes from 50,000 (zero reprocessing)       │
└───────────────────────────────────────────────────────────┘
```

**Offset Sync Configuration**:
```
consumer.offset.sync.enable: true
consumer.offset.sync.ms: 30000 (sync every 30 seconds)
consumer.offset.group.filters: {"groupFilters":[{"name":"*","patternType":"LITERAL"}]}
  (sync all consumer groups)
```

**Edge Cases**:

| Scenario | Handling |
|----------|----------|
| Consumer group doesn't exist on source | No offset to translate; consumer starts from `auto.offset.reset` |
| Offset sync lag > 60 seconds | Wait for sync to catch up before cutover |
| Translation fails for specific partition | Manual offset reset to timestamp or specific offset |
| Consumer commits offset between sync intervals | May reprocess up to 30 seconds of data (window defined by sync interval) |

### Schema Compatibility Handling

**Schema Registry Architecture**:

Schema Registry is a **regional shared service** (not cluster-specific)

```
┌─────────────────────────────────────────────────────────┐
│              Schema Registry (Regional)                  │
│         https://schema-registry.us-east-1.aws...         │
│                                                           │
│  ┌────────────────────────────────────────────┐          │
│  │  Schemas for all topics                    │          │
│  │  - orders-value (subject)                  │          │
│  │  - payments-key (subject)                  │          │
│  │  - ... (all Avro/Protobuf/JSON schemas)    │          │
│  └────────────────────────────────────────────┘          │
└─────────────────────┬──────────────┬────────────────────┘
                      │              │
        ┌─────────────┘              └─────────────┐
        │                                          │
┌───────▼──────────┐                    ┌──────────▼───────┐
│ Source Cluster   │                    │ Destination      │
│ (Single-Zone)    │                    │ Cluster          │
│                  │                    │ (Multi-Zone)     │
└──────────────────┘                    └──────────────────┘
```

**Migration Impact**: **ZERO** (Schema Registry migration not required)

**Validation Steps**:
1. Confirm destination cluster can access Schema Registry endpoint
2. Verify schema compatibility mode matches source
3. Test schema retrieval from destination cluster context
4. Validate producer/consumer schema serialization against Schema Registry

**Schema Evolution During Migration**:

```
Scenario: New schema version published during migration

T=0:   Schema version 5 exists
T=1:   Producer on source writes with schema v5
T=2:   Migration in progress (replication lag 1000 messages)
T=3:   Developer publishes schema v6 to Schema Registry
T=4:   Producer on destination writes with schema v6
T=5:   Cluster Link replicates messages with v5 (historical data)
       Consumers read v5 and v6 messages (backward compatible)

Result: No issues if schema compatibility rules enforced
        (BACKWARD/FORWARD/FULL compatibility prevents breaks)
```

**Best Practice**: Freeze schema changes during cutover window (optional but reduces risk)

---

## D. Risk Management

### Downtime Strategies

#### Rolling Migration (Recommended)

**Approach**: Migrate components incrementally with overlap (source and destination both active)

```
Timeline:
┌──────────────────────────────────────────────────────────┐
│ Component            │ Source │ Both   │ Destination     │
├──────────────────────┼────────┼────────┼─────────────────┤
│ Cluster Link         │        │ ██████ │ ████████████    │
│ Producers (Canary 5%)│ ████   │ ██     │                 │
│ Producers (Full)     │ ██████ │ ███    │                 │
│ Consumers            │ ██████ │ █      │                 │
│ Kafka Connect        │ ██████ │ █      │                 │
└──────────────────────┴────────┴────────┴─────────────────┘
         Week 4-5       Week 6    Cutover    Post-Migration
```

**Benefits**:
- Near-zero downtime (consumers down <5 min)
- Gradual risk exposure
- Easy rollback during overlap period

**Challenges**:
- Longer migration timeline
- Dual-cluster monitoring overhead

#### Hard Cutover (Not Recommended)

**Approach**: Stop all workloads on source, switch everything at once

```
T=0:   Stop all producers, consumers, connectors on source
T=10:  Verify all workloads stopped
T=15:  Update all configurations to destination
T=20:  Start all workloads on destination
T=30:  Validate and monitor

Downtime: 30+ minutes
Risk: High (all-or-nothing, no gradual rollback)
```

**When to Consider**: Very small clusters (<10 topics, <5 applications) where complexity of rolling migration outweighs downtime cost

### Rollback Strategy

#### Rollback Triggers

**Critical (Immediate Rollback)**:
- Data loss detected (message count mismatch >1%)
- Producer error rate >5% sustained for 10+ minutes
- Consumer offset corruption (consumers reprocessing from offset 0)
- Authentication/authorization failures affecting >10% of applications

**High (Rollback within 15 min)**:
- Consumer lag increasing uncontrollably (growing >100K messages/min)
- Destination cluster performance degradation (CPU >90%, disk full)
- Network connectivity failures (intermittent)

**Medium (Investigate, Consider Rollback)**:
- Latency increase >2x baseline
- Kafka Connect connectors failing to start
- Schema Registry intermittent availability

#### Rollback Procedure

**Pre-Requisite**: Cluster Link remains ACTIVE (do NOT delete until decommissioning)

```
Step 1: STOP Destination Traffic (Immediate)
  - Scale producer deployments to 0 replicas
  - Scale consumer deployments to 0 replicas
  - Pause Kafka Connect connectors

Step 2: Revert Producer Configurations
  - Update bootstrap.servers → source cluster
  - Update API keys → source cluster keys
  - Deploy via rolling restart (zero downtime)

Step 3: Restart Producers on Source
  - Scale producer deployments back to normal
  - Validate producers writing to source cluster
  - Monitor error rates

Step 4: Determine Consumer Restart Point
  - Option A: Resume from last committed offset on source (if tracked)
  - Option B: Reset to timestamp (at rollback time)
  - Option C: Reset to specific offset (calculated manually)

Step 5: Restart Consumers on Source
  - Update bootstrap.servers → source cluster
  - Update API keys → source cluster keys
  - Scale consumer deployments back to normal
  - Monitor consumer lag

Step 6: Resume Kafka Connect
  - Revert connector configs to source cluster
  - Resume connectors
  - Validate tasks RUNNING

Step 7: Post-Rollback Analysis
  - Document root cause
  - Plan remediation
  - Schedule retry migration
```

**Rollback SLA**: <15 minutes from decision to full production traffic on source

#### Data Divergence Handling

**Problem**: During failed migration, some messages written ONLY to destination cluster

```
Timeline of Divergence:

T=0:   Cutover begins, producers switched to destination
T=5:   Destination receives messages 100,001 - 100,500
T=10:  Critical issue detected, ROLLBACK initiated
T=15:  Producers reverted to source
T=16:  Source resumes at message 100,001

Result: Messages 100,001 - 100,500 exist ONLY on destination cluster
```

**Resolution Options**:

**Option 1: Accept Data Loss** (if acceptable for non-critical data)
- Messages written only to destination are lost
- Suitable for idempotent workloads where producers can replay

**Option 2: Manual Replay**
```bash
# Export messages from destination cluster (partition 0 example)
kafka-console-consumer \
  --bootstrap-server <destination-bootstrap> \
  --consumer.config destination.properties \
  --topic orders \
  --partition 0 \
  --offset 100001 \
  --max-messages 500 > divergent-messages.txt

# Replay to source cluster
kafka-console-producer \
  --bootstrap-server <source-bootstrap> \
  --producer.config source.properties \
  --topic orders < divergent-messages.txt
```

**Option 3: Application-Level Reconciliation**
- Use application transaction logs to identify lost business events
- Re-trigger business processes for divergent messages

**Prevention**: Ensure Cluster Link lag = 0 before cutover (minimizes divergence window)

### Failure Handling Scenarios

#### Scenario 1: Network Partition During Replication

**Symptom**: Cluster Link replication lag increases rapidly, state changes to DEGRADED

**Root Cause**: Network connectivity issue between source and destination VPCs

**Resolution**:
1. Validate VPC peering/PrivateLink connectivity
2. Check security group rules (port 9092 allowed)
3. Test network path: `traceroute`, `mtr`
4. Contact cloud provider if infrastructure issue
5. Wait for network recovery (Cluster Link auto-resumes)

**Impact**: Migration timeline delayed; cutover postponed until lag recovers

#### Scenario 2: Consumer Offset Translation Failure

**Symptom**: Consumers on destination start from offset 0 (reprocess all messages)

**Root Cause**: `consumer.offset.sync.enable` not configured or offset sync lag too high

**Resolution**:
1. Immediately stop consumers (prevent duplicate processing)
2. Enable offset sync: Update Cluster Link config
3. Wait for offset sync to complete (30-60 seconds)
4. Validate translated offsets match source
5. If still failing, manually reset to timestamp:
   ```bash
   kafka-consumer-groups --bootstrap-server <destination> \
     --reset-offsets --to-datetime 2026-04-20T14:30:00.000 \
     --group <consumer-group> --topic <topic> --execute
   ```

**Prevention**: Validate offset translation for ALL consumer groups before cutover

#### Scenario 3: Producer Latency Spike Post-Migration

**Symptom**: Producer p99 latency increases from 15ms → 50ms after cutover

**Root Cause**: Cross-zone replication latency with acks=all

**Resolution**:
1. Tune producer batching:
   ```
   linger.ms: 10 → 50 (increase batching window)
   batch.size: 16384 → 32768 (larger batches)
   compression.type: none → snappy (reduce bytes sent)
   ```
2. Validate application timeout settings accommodate new latency
3. Monitor for stabilization
4. If unacceptable, consider rollback (if business SLA breached)

**Prevention**: Performance testing against destination cluster in staging environment

#### Scenario 4: Kafka Connect Connector Stuck in FAILED State

**Symptom**: Connector transitions to FAILED after reconfiguration, tasks not starting

**Root Cause**: Invalid destination cluster credentials or missing ACLs

**Resolution**:
1. Check connector error logs (detailed error message)
2. Validate API key permissions (requires READ/WRITE on topic, CREATE on consumer group)
3. Test credentials manually:
   ```bash
   kafka-console-producer --bootstrap-server <destination> \
     --producer.config connector-credentials.properties --topic <topic>
   ```
4. Fix ACLs if authorization error
5. Restart connector

**Prevention**: Pre-validate all connector service account ACLs before cutover

---

# 4. IMPLEMENTATION CONSULTANT VIEW (STEP-BY-STEP GUIDE)

## Phase 1: Pre-Migration Setup

### Week 1-2: Environment Validation

#### Step 1.1: Inventory Source Cluster

**Objective**: Document complete source cluster state

**Commands**:

```bash
# List all topics
confluent kafka topic list --cluster <source-cluster-id> > topics-inventory.txt

# For each topic, export configuration
for topic in $(cat topics-inventory.txt); do
  confluent kafka topic describe $topic --cluster <source-cluster-id> -o json \
    > "topic-config-${topic}.json"
done

# Export consumer groups
confluent kafka consumer group list --cluster <source-cluster-id> \
  > consumer-groups-inventory.txt

# For each consumer group, capture lag
for group in $(cat consumer-groups-inventory.txt); do
  confluent kafka consumer group describe $group --cluster <source-cluster-id> \
    > "consumer-group-${group}.txt"
done

# Export ACLs
kafka-acls --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --list > acls-export.txt

# List Kafka Connect connectors
confluent connect list --cluster <connect-cluster-id> -o json \
  > connectors-inventory.json
```

**Deliverable**: Inventory spreadsheet with:
- Topic names, partition counts, replication factors, retention policies
- Consumer group names, lag, subscription patterns
- Producer count estimates (from metrics API)
- Connector names, types, configurations
- ACL count

#### Step 1.2: Version Compatibility Check

**Objective**: Ensure source and destination Kafka versions compatible

**Validation**:

```bash
# Check source cluster Kafka version
confluent kafka cluster describe <source-cluster-id> -o json | jq '.kafka_version'

# Check destination cluster Kafka version (after provisioning)
confluent kafka cluster describe <destination-cluster-id> -o json | jq '.kafka_version'

# Compare versions
# Compatible: Minor version differences (e.g., 3.3 → 3.4)
# Incompatible: Major version jumps (e.g., 2.8 → 3.5) require testing
```

**Compatibility Matrix**:

| Source Version | Destination Version | Cluster Link Support | Notes |
|----------------|---------------------|----------------------|-------|
| 2.8.x | 3.0.x - 3.4.x | ✅ Supported | Test in staging |
| 3.0.x - 3.4.x | 3.5.x+ | ✅ Supported | Recommended |
| <2.8 | Any | ❌ Not Supported | Upgrade source first |

#### Step 1.3: Network Connectivity Setup

**Objective**: Establish private connectivity between application VPC and Confluent Cloud

**Option A: VPC Peering (AWS Example)**

```bash
# Step 1: Create VPC peering connection request in Confluent Cloud Console
# Navigate to: Cluster → Networking → VPC Peering → Add Peering

# Record peering connection ID (output from Confluent Cloud)
PEERING_ID="pcx-0a1b2c3d4e5f6g7h8"

# Step 2: Accept peering request in your AWS account
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1

# Step 3: Update route tables in your VPC
# Add route for Confluent Cloud CIDR → peering connection
aws ec2 create-route \
  --route-table-id rtb-<your-route-table-id> \
  --destination-cidr-block <confluent-cloud-cidr> \
  --vpc-peering-connection-id $PEERING_ID

# Step 4: Update security groups
# Allow egress from application security group to Confluent Cloud CIDR:9092
aws ec2 authorize-security-group-egress \
  --group-id sg-<your-app-sg> \
  --protocol tcp \
  --port 9092 \
  --cidr <confluent-cloud-cidr>
```

**Validation**:
```bash
# From an EC2 instance in your VPC, test connectivity
telnet <destination-bootstrap-server> 9092
# Expected: Connected to <bootstrap-server>

# Test DNS resolution
nslookup <destination-bootstrap-server>
# Expected: Private IP address (not public)
```

**Option B: PrivateLink (Recommended)**

```bash
# Step 1: Create PrivateLink access in Confluent Cloud
confluent network private-link-access create \
  --cloud aws \
  --region us-east-1 \
  --network <network-id>

# Record endpoint service name (output)
ENDPOINT_SERVICE="com.amazonaws.vpce.us-east-1.vpce-svc-0a1b2c3d4e5f6g7h8"

# Step 2: Create VPC endpoint in your AWS account
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-<your-vpc-id> \
  --service-name $ENDPOINT_SERVICE \
  --subnet-ids subnet-<subnet-1> subnet-<subnet-2> subnet-<subnet-3> \
  --security-group-ids sg-<your-sg> \
  --vpc-endpoint-type Interface

# Step 3: Enable private DNS (optional, for simplified endpoint names)
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id vpce-<endpoint-id> \
  --private-dns-enabled
```

#### Step 1.4: Backup Strategy

**Objective**: Ensure ability to restore source cluster if catastrophic failure

**Activities**:

1. **Verify Confluent Cloud automated backups enabled** (default for Dedicated/Enterprise)
   - Confluent Cloud automatically backs up cluster metadata
   - Topic data retained per retention policy

2. **Export critical configurations** (already done in Step 1.1)
   - Topic configs
   - ACLs
   - Consumer group offsets (stored in `__consumer_offsets` topic, auto-backed up)

3. **Document rollback procedure** (see Section 3.D)

4. **Test restore capability** (if compliance-required)
   - Create test cluster
   - Restore topic configs
   - Validate

**Note**: Confluent Cloud does NOT offer point-in-time recovery. Backup strategy focuses on configuration preservation, not data restore.

---

## Phase 2: Infrastructure Preparation (Week 4)

### Step 2.1: Provision Multi-Zone Enterprise Cluster

**Objective**: Create destination cluster matching source capacity

**Via Confluent Cloud Console**:
1. Navigate to: **Environments** → Select environment → **Create Cluster**
2. Select: **Enterprise**
3. Configuration:
   - **Cloud Provider**: AWS (match source)
   - **Region**: us-east-1 (match source)
   - **Availability**: **Multi-Zone**
   - **Cluster Name**: `prod-enterprise-multi-zone`
4. Click **Launch Cluster**
5. Wait for provisioning (15-30 minutes)

**Via Confluent CLI**:
```bash
confluent kafka cluster create prod-enterprise-multi-zone \
  --cloud aws \
  --region us-east-1 \
  --type enterprise \
  --availability multi-zone

# Record cluster ID (output)
DESTINATION_CLUSTER_ID="lkc-a1b2c3d"
```

**Validation**:
```bash
# Check cluster status (wait for PROVISIONED)
confluent kafka cluster describe $DESTINATION_CLUSTER_ID -o json | jq '.status'
# Expected: "PROVISIONED"

# Verify availability zones
confluent kafka cluster describe $DESTINATION_CLUSTER_ID -o json | jq '.availability'
# Expected: "MULTI_ZONE"
```

### Step 2.2: Configure Cluster-Level Settings

**Objective**: Set default configurations for all topics on destination cluster

**Commands**:
```bash
# Set default replication factor (already 3 for multi-zone, verify)
confluent kafka cluster describe $DESTINATION_CLUSTER_ID -o json \
  | jq '.config."default.replication.factor"'
# Expected: "3"

# Set cluster-level configs (optional, based on source cluster)
# These are configured via Confluent Cloud Console: Cluster Settings → Edit

# Recommended settings:
# - auto.create.topics.enable: false (prevent accidental topic creation)
# - default.min.insync.replicas: 2 (durability)
```

### Step 2.3: Create Service Accounts and API Keys

**Objective**: Set up authentication for applications and Cluster Link

**Step 2.3.1: Create Service Accounts**

```bash
# Service account for Cluster Link
confluent iam service-account create cluster-link-sa \
  --description "Service account for Cluster Linking"

SA_LINK=$(confluent iam service-account list -o json \
  | jq -r '.[] | select(.name=="cluster-link-sa") | .id')

# Service accounts for producers
confluent iam service-account create producer-orders-sa \
  --description "Orders producer service account"

SA_PRODUCER_ORDERS=$(confluent iam service-account list -o json \
  | jq -r '.[] | select(.name=="producer-orders-sa") | .id')

# Service accounts for consumers
confluent iam service-account create consumer-analytics-sa \
  --description "Analytics consumer service account"

SA_CONSUMER_ANALYTICS=$(confluent iam service-account list -o json \
  | jq -r '.[] | select(.name=="consumer-analytics-sa") | .id')

# (Repeat for all application service accounts)
```

**Step 2.3.2: Create API Keys**

```bash
# API key for Cluster Link on SOURCE cluster
confluent api-key create \
  --service-account $SA_LINK \
  --resource <source-cluster-id> \
  --description "Cluster link source key"

# Record API key and secret (store securely!)
LINK_SOURCE_KEY="<api-key>"
LINK_SOURCE_SECRET="<api-secret>"

# API key for Cluster Link on DESTINATION cluster
confluent api-key create \
  --service-account $SA_LINK \
  --resource $DESTINATION_CLUSTER_ID \
  --description "Cluster link destination key"

LINK_DEST_KEY="<api-key>"
LINK_DEST_SECRET="<api-secret>"

# API keys for producers on destination
confluent api-key create \
  --service-account $SA_PRODUCER_ORDERS \
  --resource $DESTINATION_CLUSTER_ID \
  --description "Orders producer destination key"

PRODUCER_ORDERS_KEY="<api-key>"
PRODUCER_ORDERS_SECRET="<api-secret>"

# API keys for consumers on destination
confluent api-key create \
  --service-account $SA_CONSUMER_ANALYTICS \
  --resource $DESTINATION_CLUSTER_ID \
  --description "Analytics consumer destination key"

CONSUMER_ANALYTICS_KEY="<api-key>"
CONSUMER_ANALYTICS_SECRET="<api-secret>"

# (Repeat for all applications)
```

**Step 2.3.3: Store Secrets Securely**

```bash
# Example: AWS Secrets Manager
aws secretsmanager create-secret \
  --name confluent/destination/producer-orders \
  --secret-string "{\"api_key\":\"$PRODUCER_ORDERS_KEY\",\"api_secret\":\"$PRODUCER_ORDERS_SECRET\"}"

# Or: HashiCorp Vault, Kubernetes Secrets, etc.
```

### Step 2.4: Configure ACLs

**Objective**: Replicate source cluster ACLs to destination

**Option A: Automatic via Cluster Link** (Recommended)
- ACLs automatically synced when `acl.sync.enable=true` in Cluster Link config
- Configured in Step 3.1

**Option B: Manual ACL Migration**

```bash
# Export ACLs from source (already done in Phase 1)
# Parse and create ACLs on destination

# Example: Grant producer permissions
kafka-acls --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --add \
  --allow-principal User:$SA_PRODUCER_ORDERS \
  --operation WRITE \
  --topic orders

kafka-acls --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --add \
  --allow-principal User:$SA_PRODUCER_ORDERS \
  --operation DESCRIBE \
  --topic orders

# Example: Grant consumer permissions
kafka-acls --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --add \
  --allow-principal User:$SA_CONSUMER_ANALYTICS \
  --operation READ \
  --topic orders

kafka-acls --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --add \
  --allow-principal User:$SA_CONSUMER_ANALYTICS \
  --operation READ \
  --group analytics-group

# (Script this for all ACLs from acls-export.txt)
```

**Validation**:
```bash
# List ACLs on destination
kafka-acls --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --list

# Compare with source ACLs
diff acls-export.txt acls-destination.txt
```

### Step 2.5: Validate Schema Registry Access

**Objective**: Ensure destination cluster can access Schema Registry

**Configuration**:

Schema Registry is typically a **regional shared service**. No migration needed, just validation.

```bash
# Get Schema Registry endpoint from source cluster config
SR_ENDPOINT=$(confluent schema-registry cluster describe <sr-cluster-id> -o json | jq -r '.endpoint_url')

# Create API key for Schema Registry (if not already existing)
confluent api-key create \
  --service-account $SA_PRODUCER_ORDERS \
  --resource <sr-cluster-id>

SR_KEY="<api-key>"
SR_SECRET="<api-secret>"

# Test access from destination cluster context
curl -u $SR_KEY:$SR_SECRET $SR_ENDPOINT/subjects
# Expected: JSON array of schema subjects
```

**Validation**:
```bash
# Verify compatibility mode
curl -u $SR_KEY:$SR_SECRET $SR_ENDPOINT/config
# Expected: {"compatibilityLevel":"BACKWARD"} (or FORWARD/FULL)

# List all schemas
curl -u $SR_KEY:$SR_SECRET $SR_ENDPOINT/subjects | jq .
# Expected: ["orders-value", "payments-key", ...]
```

---

## Phase 3: Migration Execution (Week 5-6)

### Step 3.1: Create Cluster Link

**Objective**: Establish replication channel from source to destination

**Step 3.1.1: Create Link Configuration File**

```bash
cat > cluster-link.config << 'EOF'
# Source cluster connection details
bootstrap.servers=<source-cluster-bootstrap>:9092
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<LINK_SOURCE_KEY>' password='<LINK_SOURCE_SECRET>';

# Consumer offset sync (CRITICAL for zero-reprocessing cutover)
consumer.offset.sync.enable=true
consumer.offset.sync.ms=30000

# ACL sync (automatic ACL replication)
acl.sync.enable=true
acl.sync.ms=30000

# Cluster link mode
link.mode=DESTINATION

# Performance tuning
cluster.link.metadata.topic.replication.factor=3
cluster.link.metadata.topic.min.insync.replicas=2
EOF

# Replace placeholders with actual values
sed -i "s/<source-cluster-bootstrap>/$SOURCE_BOOTSTRAP/g" cluster-link.config
sed -i "s/<LINK_SOURCE_KEY>/$LINK_SOURCE_KEY/g" cluster-link.config
sed -i "s/<LINK_SOURCE_SECRET>/$LINK_SOURCE_SECRET/g" cluster-link.config
```

**Step 3.1.2: Create Cluster Link**

```bash
# Create link from destination cluster
confluent kafka link create prod-migration-link \
  --cluster $DESTINATION_CLUSTER_ID \
  --source-cluster <source-cluster-id> \
  --config-file cluster-link.config

# Validation: Check link status
confluent kafka link describe prod-migration-link \
  --cluster $DESTINATION_CLUSTER_ID

# Expected output:
# Link Name: prod-migration-link
# Link ID: <link-id>
# Source Cluster ID: <source-cluster-id>
# Destination Cluster ID: <destination-cluster-id>
# State: ACTIVE
```

**Troubleshooting**:
- If link state is FAILED: Check network connectivity, API key permissions
- If authentication error: Validate `LINK_SOURCE_KEY` has DeveloperRead permission on source cluster

### Step 3.2: Create Mirror Topics

**Objective**: Replicate all source topics to destination as read-only mirrors

**Step 3.2.1: List Source Topics**

```bash
# Get all topics from source (exclude internal topics)
SOURCE_TOPICS=$(confluent kafka topic list --cluster <source-cluster-id> -o json \
  | jq -r '.[].name' | grep -v '^_')

echo "Topics to mirror:"
echo "$SOURCE_TOPICS"
```

**Step 3.2.2: Create Mirror Topics**

```bash
# Create mirrors for all topics
for topic in $SOURCE_TOPICS; do
  echo "Creating mirror for topic: $topic"
  
  confluent kafka mirror create $topic \
    --link prod-migration-link \
    --cluster $DESTINATION_CLUSTER_ID
  
  # Brief pause to avoid rate limiting
  sleep 2
done
```

**Step 3.2.3: Validate Mirror Creation**

```bash
# Check mirror status for each topic
for topic in $SOURCE_TOPICS; do
  confluent kafka mirror describe $topic \
    --link prod-migration-link \
    --cluster $DESTINATION_CLUSTER_ID -o json | jq '{topic: .source_topic_name, state: .state_time_ms, lag: .num_partition_lag}'
done

# Expected: State should be ACTIVE or SYNCING, lag decreasing
```

**Verification**:
```bash
# List topics on destination (should include all mirrors)
confluent kafka topic list --cluster $DESTINATION_CLUSTER_ID

# Compare partition counts source vs destination
kafka-topics --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe --topic orders | grep PartitionCount

kafka-topics --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --describe --topic orders | grep PartitionCount

# Should match
```

### Step 3.3: Monitor Replication Lag

**Objective**: Wait for mirror lag to reach near-zero (cutover readiness)

**Step 3.3.1: Create Monitoring Script**

```bash
cat > monitor-mirror-lag.sh << 'EOF'
#!/bin/bash

DESTINATION_CLUSTER_ID="<destination-cluster-id>"
LINK_NAME="prod-migration-link"

while true; do
  echo "=== $(date) ==="
  
  # Get lag for all mirror topics
  confluent kafka mirror list --link $LINK_NAME --cluster $DESTINATION_CLUSTER_ID -o json \
    | jq -r '.[] | "\(.source_topic_name): \(.max_mirror_lag_ms // 0) ms, \(.num_partition_lag // 0) messages"'
  
  echo ""
  sleep 60
done
EOF

chmod +x monitor-mirror-lag.sh

# Run monitoring (in background or separate terminal)
./monitor-mirror-lag.sh
```

**Step 3.3.2: Cutover Readiness Criteria**

Wait until ALL topics meet:
- Mirror lag < 1000 messages
- Lag stable (not increasing) for 30+ minutes
- Mirror state: ACTIVE (no errors)

**Timeline**: Typically 5-7 days for 100TB cluster to fully sync

**Step 3.3.3: Validate Consumer Offset Sync**

```bash
# Check consumer group offsets on source
kafka-consumer-groups --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe --group analytics-group > source-offsets.txt

# Check translated offsets on destination
kafka-consumer-groups --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --describe --group analytics-group > destination-offsets.txt

# Compare (offsets should be within replication lag window)
diff source-offsets.txt destination-offsets.txt
```

---

### Step 3.4: Cutover Execution (Cutover Day - Week 6)

**Objective**: Switch all traffic from source to destination with <5 min downtime

**Pre-Cutover Checklist** (T-1 hour):
- [ ] Mirror lag < 100 messages for ALL topics
- [ ] Consumer offset sync lag < 30 seconds
- [ ] All application deployment pipelines ready (configs updated, tested)
- [ ] Rollback plan reviewed and ready
- [ ] Monitoring dashboards prepared
- [ ] Team assembled (incident bridge open)

**Cutover Timeline** (4-hour window, actual cutover ~30 min):

#### T-60 min: Final Validations

```bash
# Final mirror lag check
./monitor-mirror-lag.sh | head -20

# Verify NO deployments in progress
kubectl get deployments -n production | grep -i rolling

# Alert monitoring team
# (Send Slack/PagerDuty notification)
```

#### T-15 min: Freeze Changes

```bash
# Announce code freeze in team channels
# No deployments, config changes, or schema updates during cutover window
```

#### T=0: Begin Cutover - Pause Kafka Connect

```bash
# List all connectors
CONNECTORS=$(confluent connect list --cluster <connect-cluster-id> -o json | jq -r '.[].name')

# Pause each connector
for connector in $CONNECTORS; do
  echo "Pausing connector: $connector"
  confluent connect pause $connector --cluster <connect-cluster-id>
  sleep 2
done

# Validate all paused
for connector in $CONNECTORS; do
  confluent connect describe $connector --cluster <connect-cluster-id> | grep State
done
# Expected: State: PAUSED for all connectors
```

#### T+2 min: Deploy Producer Canary (5%)

```bash
# Update producer deployment with destination cluster config
# (Exact method depends on deployment system: Kubernetes, ECS, etc.)

# Example: Kubernetes ConfigMap update
kubectl create configmap producer-orders-config \
  --from-literal=KAFKA_BOOTSTRAP_SERVERS=<destination-bootstrap>:9092 \
  --from-literal=KAFKA_API_KEY=$PRODUCER_ORDERS_KEY \
  --from-literal=KAFKA_API_SECRET=$PRODUCER_ORDERS_SECRET \
  --dry-run=client -o yaml | kubectl apply -f -

# Rolling restart canary deployment (5% of pods)
kubectl rollout restart deployment/producer-orders-canary

# Wait for canary pods ready
kubectl rollout status deployment/producer-orders-canary
```

#### T+5 min: Validate Canary

```bash
# Check producer metrics (error rate, latency)
# Via Confluent Cloud Console or metrics API

# Validate messages arriving on destination cluster
kafka-console-consumer --bootstrap-server <destination-bootstrap>:9092 \
  --consumer.config destination-client.properties \
  --topic orders \
  --max-messages 10 \
  --timeout-ms 10000

# Expected: Messages appearing (from canary producers)
```

**Decision Point**: If canary success (error rate <1%, latency acceptable), proceed. If failure, ROLLBACK.

#### T+8 min: Deploy Full Producer Rollout

```bash
# Update main producer deployment (phased: 25% → 50% → 100%)

# Phase 1: 25%
kubectl rollout restart deployment/producer-orders-main --replicas=25%
kubectl rollout status deployment/producer-orders-main

# Monitor for 5 minutes (T+13)

# Phase 2: 50%
kubectl rollout restart deployment/producer-orders-main --replicas=50%
kubectl rollout status deployment/producer-orders-main

# Monitor for 3 minutes (T+16)

# Phase 3: 100%
kubectl rollout restart deployment/producer-orders-main --replicas=100%
kubectl rollout status deployment/producer-orders-main
```

**Validation**:
```bash
# All producers writing to destination
kafka-topics --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --describe --topic orders | grep "Leader:"

# Verify offsets increasing on destination
```

#### T+15 min: Stop Consumers on Source

```bash
# Gracefully scale down consumer deployments on source cluster
kubectl scale deployment/consumer-analytics --replicas=0

# Wait for pods to terminate (graceful shutdown)
kubectl wait --for=delete pod -l app=consumer-analytics --timeout=60s
```

#### T+17 min: Verify Final Consumer Offsets on Source

```bash
# Capture final committed offsets on source
kafka-consumer-groups --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe --group analytics-group > final-source-offsets.txt
```

#### T+18 min: Start Consumers on Destination

```bash
# Update consumer deployment with destination cluster config
kubectl create configmap consumer-analytics-config \
  --from-literal=KAFKA_BOOTSTRAP_SERVERS=<destination-bootstrap>:9092 \
  --from-literal=KAFKA_API_KEY=$CONSUMER_ANALYTICS_KEY \
  --from-literal=KAFKA_API_SECRET=$CONSUMER_ANALYTICS_SECRET \
  --dry-run=client -o yaml | kubectl apply -f -

# Scale up consumer deployment
kubectl scale deployment/consumer-analytics --replicas=10

# Wait for consumers to start
kubectl rollout status deployment/consumer-analytics
```

**Validation**:
```bash
# Check consumers joined group and consuming
kafka-consumer-groups --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --describe --group analytics-group

# Expected: Consumers assigned to partitions, LAG decreasing
```

#### T+22 min: Validate Consumer Offset Translation

```bash
# Compare final source offsets vs. destination starting offsets
kafka-consumer-groups --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --describe --group analytics-group > destination-starting-offsets.txt

# Compare
diff final-source-offsets.txt destination-starting-offsets.txt

# Expected: Offsets match (within replication lag window)
# If major mismatch (>1000 messages difference), investigate
```

#### T+25 min: Resume Kafka Connect Connectors

```bash
# Update connector configurations with destination cluster
for connector in $CONNECTORS; do
  echo "Updating connector: $connector"
  
  # Export current config
  confluent connect describe $connector --cluster <connect-cluster-id> -o json \
    > connector-$connector-config.json
  
  # Update bootstrap.servers and API keys (scripted or manual)
  jq '.config."kafka.bootstrap.servers" = "<destination-bootstrap>:9092"' \
    connector-$connector-config.json > connector-$connector-updated.json
  
  jq '.config."kafka.sasl.jaas.config" = "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"$CONNECTOR_KEY\" password=\"$CONNECTOR_SECRET\";"' \
    connector-$connector-updated.json > connector-$connector-final.json
  
  # Update connector
  confluent connect update $connector \
    --cluster <connect-cluster-id> \
    --config-file connector-$connector-final.json
done

# Resume connectors
for connector in $CONNECTORS; do
  echo "Resuming connector: $connector"
  confluent connect resume $connector --cluster <connect-cluster-id>
  sleep 2
done
```

**Validation**:
```bash
# Check all connectors RUNNING
for connector in $CONNECTORS; do
  confluent connect describe $connector --cluster <connect-cluster-id> | grep State
done

# Expected: State: RUNNING for all
```

#### T+30 min: Cutover Complete - Begin Monitoring

**Validation Checklist**:
- [ ] All producers writing to destination cluster (metrics show traffic)
- [ ] All consumers reading from destination cluster (lag decreasing)
- [ ] Kafka Connect connectors in RUNNING state (tasks active)
- [ ] No errors in application logs
- [ ] End-to-end latency within acceptable range (<500ms)
- [ ] Error rates within baseline (<0.01%)

**Monitoring Period**: 2-4 hours of intensive monitoring, then 24/7 for 2 weeks

---

## Phase 4: Post-Migration Validation (Week 6-8)

### Step 4.1: Data Integrity Validation

**Objective**: Confirm zero data loss during migration

**Step 4.1.1: Message Count Validation**

```bash
#!/bin/bash
# validate-message-counts.sh

TOPICS=$(confluent kafka topic list --cluster <source-cluster-id> -o json \
  | jq -r '.[].name' | grep -v '^_')

echo "Topic,Source High Water Mark,Destination High Water Mark,Diff,Status"

for topic in $TOPICS; do
  # Get high water mark (total messages) on source
  src_hwm=$(kafka-run-class kafka.tools.GetOffsetShell \
    --broker-list <source-bootstrap>:9092 \
    --topic $topic \
    --time -1 | awk -F: '{sum+=$3} END {print sum}')
  
  # Get high water mark on destination
  dest_hwm=$(kafka-run-class kafka.tools.GetOffsetShell \
    --broker-list <destination-bootstrap>:9092 \
    --topic $topic \
    --time -1 | awk -F: '{sum+=$3} END {print sum}')
  
  diff=$((src_hwm - dest_hwm))
  
  if [ $diff -eq 0 ]; then
    status="✓ MATCH"
  elif [ ${diff#-} -lt 100 ]; then  # Absolute value
    status="⚠ MINOR LAG"
  else
    status="✗ MISMATCH"
  fi
  
  echo "$topic,$src_hwm,$dest_hwm,$diff,$status"
done
```

**Expected Result**: All topics show MATCH or MINOR LAG (<100 messages due to in-flight at cutover)

### Step 4.2: Consumer Lag Validation

**Objective**: Ensure consumers processing at normal rate

```bash
# Monitor consumer lag for all groups
CONSUMER_GROUPS=$(kafka-consumer-groups --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --list)

for group in $CONSUMER_GROUPS; do
  echo "=== Consumer Group: $group ==="
  kafka-consumer-groups --bootstrap-server <destination-bootstrap>:9092 \
    --command-config destination-client.properties \
    --describe --group $group | grep -v "^GROUP"
  echo ""
done

# Expected: LAG column decreasing or stable (not increasing)
```

### Step 4.3: Schema Validation

**Objective**: Confirm Schema Registry integration working

```bash
# Test producer schema serialization
# (Application-specific test, example for Avro)

# Produce test message with schema
kafka-avro-console-producer \
  --bootstrap-server <destination-bootstrap>:9092 \
  --producer.config destination-client.properties \
  --topic test-schema-topic \
  --property schema.registry.url=<schema-registry-url> \
  --property basic.auth.credentials.source=USER_INFO \
  --property basic.auth.user.info=$SR_KEY:$SR_SECRET \
  --property value.schema='{"type":"record","name":"TestRecord","fields":[{"name":"id","type":"string"}]}' \
  <<< '{"id":"test-123"}'

# Consume and deserialize
kafka-avro-console-consumer \
  --bootstrap-server <destination-bootstrap>:9092 \
  --consumer.config destination-client.properties \
  --topic test-schema-topic \
  --from-beginning \
  --max-messages 1 \
  --property schema.registry.url=<schema-registry-url> \
  --property basic.auth.credentials.source=USER_INFO \
  --property basic.auth.user.info=$SR_KEY:$SR_SECRET

# Expected: Message deserialized successfully
```

### Step 4.4: Connector Validation

**Objective**: Confirm Kafka Connect data pipelines operating correctly

```bash
# Check connector throughput (compare pre/post migration)
# Via Confluent Cloud Console: Connectors → Metrics

# Validate source connector (e.g., JDBC Source)
# Check that new records from database appear in Kafka

# Validate sink connector (e.g., S3 Sink)
# Check that messages from Kafka appear in S3 bucket
```

### Step 4.5: Performance Benchmarking

**Objective**: Compare destination cluster performance to source baseline

**Producer Throughput Test**:
```bash
kafka-producer-perf-test \
  --topic perf-test \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=<destination-bootstrap>:9092 \
    security.protocol=SASL_SSL \
    sasl.mechanism=PLAIN \
    sasl.jaas.config='org.apache.kafka.common.security.plain.PlainLoginModule required username="<key>" password="<secret>";' \
    acks=all \
    compression.type=snappy

# Compare results to source cluster baseline
# Expected: Similar or better throughput (multi-zone has more brokers)
```

**Consumer Throughput Test**:
```bash
kafka-consumer-perf-test \
  --broker-list <destination-bootstrap>:9092 \
  --topic perf-test \
  --messages 1000000 \
  --consumer.config destination-client.properties

# Compare to source baseline
```

---

## Phase 5: Decommissioning (Week 10)

### Step 5.1: Wait Period

**Objective**: Ensure migration stability before decommissioning source

**Criteria for Decommission**:
- [ ] 7-14 days of stable operation on destination cluster
- [ ] Zero critical incidents related to migration
- [ ] Business stakeholders confirm data reconciliation complete
- [ ] Performance metrics meet or exceed baseline
- [ ] All applications confirmed migrated (no traffic to source)

### Step 5.2: Final Validation Before Decommission

```bash
# Verify NO traffic on source cluster
# Check metrics: produce rate, consume rate should be 0

# Verify all applications pointing to destination
# Audit application configs, environment variables

# Backup source cluster configurations (final snapshot)
confluent kafka cluster describe <source-cluster-id> -o json > source-cluster-final-snapshot.json
```

### Step 5.3: Delete Cluster Link

```bash
# Delete cluster link (no longer needed)
confluent kafka link delete prod-migration-link --cluster $DESTINATION_CLUSTER_ID

# Validation: Link should no longer appear
confluent kafka link list --cluster $DESTINATION_CLUSTER_ID
# Expected: Empty list or link absent
```

### Step 5.4: Delete Source Cluster

**⚠️ WARNING: This is IRREVERSIBLE. Triple-check before proceeding.**

```bash
# Final confirmation checklist:
# [ ] All traffic on destination for 7+ days
# [ ] Business approval obtained
# [ ] Backup configurations saved
# [ ] Finance approval (to stop billing)

# Delete source cluster
confluent kafka cluster delete <source-cluster-id>

# Confirmation prompt will appear - type cluster ID to confirm
```

### Step 5.5: Cleanup Associated Resources

```bash
# Delete VPC peering for source cluster (if separate from destination)
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id <source-peering-id>

# Delete API keys for source cluster
confluent api-key list --resource <source-cluster-id> -o json | jq -r '.[].id' | while read key_id; do
  confluent api-key delete $key_id
done

# Remove monitoring dashboards for source cluster

# Update documentation (runbooks, wikis) with new architecture
```

---

# 5. FUNCTION FLOW EXPLANATION

## Data Movement During Migration

### Phase 1: Replication (Weeks 4-5)

```
┌────────────────────────────────────────────────────────────────┐
│                   REPLICATION PHASE FLOW                        │
└────────────────────────────────────────────────────────────────┘

Step 1: Producer writes to SOURCE
  Producer App → Source Cluster (Single-Zone)
                 Topic: orders, Partition 0
                 Message: {id: 1001, amount: 50.00}
                 Offset: 100,000

Step 2: Cluster Link detects new message
  Cluster Link Consumer (on Destination) → Reads from Source
    - Consumes message 100,000 from orders:0
    - Stores in internal buffer

Step 3: Cluster Link writes to DESTINATION
  Cluster Link Producer (on Destination) → Writes to Destination
    - Topic: orders (mirror), Partition 0
    - Message: {id: 1001, amount: 50.00} (byte-identical)
    - Offset: 100,000 (same offset number)

Step 4: Consumer offset sync
  Cluster Link Offset Sync Process:
    - Reads consumer group "analytics" committed offset on Source: 99,500
    - Translates to Destination equivalent offset: 99,500
    - Writes translated offset to __consumer_offsets on Destination

Step 5: Consumer continues reading from SOURCE (unchanged)
  Consumer App → Source Cluster
    - Reads message 99,500 from orders:0
    - Processes message
    - Commits offset 99,501

Step 6: Lag monitoring
  Mirror Lag = Source Offset - Destination Offset
             = 100,000 - 99,900 = 100 messages (decreasing)

┌─────────────────────────────────────────────────────────────┐
│                   TIME PROGRESSION                           │
├─────────────────────────────────────────────────────────────┤
│ Day 1: Lag = 10M messages                                   │
│ Day 3: Lag = 1M messages                                    │
│ Day 5: Lag = 100K messages                                  │
│ Day 7: Lag = 100 messages → CUTOVER READY                   │
└─────────────────────────────────────────────────────────────┘
```

### Phase 2: Cutover (Day 0, Hour 0-4)

```
┌────────────────────────────────────────────────────────────────┐
│                   CUTOVER PHASE FLOW                            │
└────────────────────────────────────────────────────────────────┘

T=0 min: BEFORE CUTOVER
  Producers → SOURCE (writing)
  Consumers → SOURCE (reading)
  Cluster Link → Replicating SOURCE → DESTINATION (lag ~0)

T=2 min: PRODUCER SWITCH BEGINS (Canary 5%)
  5% Producers → DESTINATION (new messages: 100,001+)
  95% Producers → SOURCE (messages: up to 100,000)
  Consumers → SOURCE (reading up to 100,000)
  Cluster Link → Still replicating

T=12 min: ALL PRODUCERS SWITCHED
  100% Producers → DESTINATION (messages: 100,001+)
  0% Producers → SOURCE (stopped)
  Consumers → SOURCE (reading up to 100,000)
  Cluster Link → Draining final messages from SOURCE

T=15 min: CONSUMERS STOPPED ON SOURCE
  Producers → DESTINATION (messages: 100,150+)
  Consumers → STOPPED
    Final committed offset on SOURCE: 100,000 (example)

T=17 min: WAIT FOR CLUSTER LINK OFFSET SYNC
  Cluster Link syncs final consumer offset:
    SOURCE consumer offset 100,000 → DESTINATION offset 100,000

T=18 min: CONSUMERS STARTED ON DESTINATION
  Producers → DESTINATION (messages: 100,200+)
  Consumers → DESTINATION
    Resume from translated offset: 100,000
    Next message to read: 100,001 (no gap, no reprocessing)

T=30 min: CUTOVER COMPLETE
  Producers → DESTINATION (messages: 100,500+)
  Consumers → DESTINATION (consuming 100,001 → 100,500+)
  SOURCE cluster → Idle (no traffic)

DOWNTIME WINDOW:
  Producer downtime: 0 minutes (rolling restart)
  Consumer downtime: 3 minutes (T+15 to T+18)
  Kafka Connect downtime: 5 minutes (T+0 to T+25, paused)
```

## Replication Mechanism Details

### Cluster Link Internals

```
┌─────────────────────────────────────────────────────────────────┐
│              CLUSTER LINK REPLICATION ENGINE                     │
└─────────────────────────────────────────────────────────────────┘

Component 1: Link Consumer (runs on Destination Cluster)
  - Acts as a Kafka consumer reading from Source Cluster
  - Consumer group: internal (managed by Cluster Link)
  - Reads ALL partitions of ALL mirror topics
  - Parallelism: One consumer per partition

Component 2: Link Producer (runs on Destination Cluster)
  - Acts as a Kafka producer writing to Destination Cluster
  - Writes to mirror topics (read-only replicas)
  - Preserves message keys, values, headers, timestamps

Component 3: Offset Tracker
  - Maintains mapping: Source Offset → Destination Offset
  - Stored in internal topic: __cluster_link_offsets
  - Enables consumer offset translation

Component 4: Consumer Offset Sync Process
  - Periodically reads __consumer_offsets from Source
  - Translates offsets using Offset Tracker mapping
  - Writes translated offsets to __consumer_offsets on Destination
  - Frequency: Every 30 seconds (configurable)

Component 5: ACL Sync Process (if enabled)
  - Periodically reads ACLs from Source
  - Replicates to Destination
  - Frequency: Every 30 seconds

Flow Example (orders topic, Partition 0):

┌─────────────────┐
│ Source Cluster  │
│ orders:0        │
│ Offset: 50,000  │
│ Message: {...}  │
└────────┬────────┘
         │
         │ (1) Link Consumer reads message at offset 50,000
         ▼
┌─────────────────────────┐
│ Link Consumer           │
│ Fetches msg 50,000      │
│ Stores in buffer        │
└────────┬────────────────┘
         │
         │ (2) Link Producer writes to Destination
         ▼
┌─────────────────────────┐
│ Destination Cluster     │
│ orders:0 (mirror)       │
│ Offset: 50,000          │
│ Message: {...} (same)   │
└─────────────────────────┘
         │
         │ (3) Offset Tracker updates mapping
         ▼
┌─────────────────────────┐
│ Offset Tracker          │
│ Source 50,000 →         │
│ Destination 50,000      │
└─────────────────────────┘
```

### Offset Preservation Strategy

**Challenge**: Consumer must resume on Destination exactly where it left off on Source

**Solution**: Automatic Offset Translation

```
Scenario: Consumer group "analytics" consuming topic "orders"

SOURCE CLUSTER STATE (before cutover):
  Topic: orders, Partition 0
  Consumer Group: analytics
  Current Offset: 75,000
  
  Consumer has processed messages 0 → 74,999
  Next message to read: 75,000

CLUSTER LINK OFFSET SYNC:
  Every 30 seconds, Cluster Link:
    1. Reads committed offset from Source: 75,000
    2. Maps to Destination equivalent: 75,000
       (mapping stored in __cluster_link_offsets)
    3. Writes to __consumer_offsets on Destination:
       Group: analytics, Topic: orders, Partition: 0, Offset: 75,000

CUTOVER:
  1. Consumer stops on Source
     Final committed offset: 75,000

  2. Wait 30-60 seconds for offset sync to complete

  3. Consumer starts on Destination
     Reads committed offset from __consumer_offsets: 75,000
     Resumes from offset 75,000 (next message: 75,001)

RESULT: Zero reprocessing, zero message loss
```

### Cutover Traffic Switch Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   TRAFFIC SWITCH SEQUENCE                        │
└─────────────────────────────────────────────────────────────────┘

BEFORE CUTOVER:
┌──────────┐      ┌─────────┐      ┌──────────┐
│Producers │─────►│ SOURCE  │◄─────│Consumers │
│  (100%)  │      │ Cluster │      │  (100%)  │
└──────────┘      └────┬────┘      └──────────┘
                       │
              Cluster Link Replication
                       │
                       ▼
                  ┌─────────┐
                  │  DEST   │
                  │ Cluster │
                  │(Standby)│
                  └─────────┘

DURING CUTOVER (Producers switching):
┌──────────┐      ┌─────────┐      ┌──────────┐
│Producers │─────►│ SOURCE  │◄─────│Consumers │
│  (25%)   │      │ Cluster │      │  (100%)  │
└──────────┘      └────┬────┘      └──────────┘
     │                 │
     │        Cluster Link (draining)
     │                 │
     │                 ▼
     │            ┌─────────┐
     └───────────►│  DEST   │
       (75%)      │ Cluster │
                  │(Active) │
                  └─────────┘

AFTER PRODUCER SWITCH, BEFORE CONSUMER SWITCH:
                  ┌─────────┐      ┌──────────┐
                  │ SOURCE  │◄─────│Consumers │
                  │ Cluster │      │  (100%)  │
                  │ (Idle)  │      └──────────┘
                  └─────────┘      (Reading old data)
                       
┌──────────┐      ┌─────────┐
│Producers │─────►│  DEST   │
│  (100%)  │      │ Cluster │
└──────────┘      │(Active) │
                  └─────────┘
                  (Writing new data)

AFTER COMPLETE CUTOVER:
┌──────────┐      ┌─────────┐      ┌──────────┐
│Producers │─────►│  DEST   │◄─────│Consumers │
│  (100%)  │      │ Cluster │      │  (100%)  │
└──────────┘      │ (PROD)  │      └──────────┘
                  └─────────┘
                  
                  ┌─────────┐
                  │ SOURCE  │
                  │ Cluster │
                  │ (Idle)  │ ← Decommission after 7-14 days
                  └─────────┘
```

---

# 6. TROUBLESHOOTING GUIDE

## Common Migration Failures

### Issue 1: Cluster Link Fails to Create

**Symptom**:
```
Error: Failed to create cluster link
Reason: Authentication failed
```

**Root Cause**: Invalid API key or insufficient permissions

**Diagnosis**:
```bash
# Test API key manually
kafka-broker-api-versions --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-test.properties

# source-test.properties:
# bootstrap.servers=<source-bootstrap>:9092
# security.protocol=SASL_SSL
# sasl.mechanism=PLAIN
# sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<LINK_SOURCE_KEY>' password='<LINK_SOURCE_SECRET>';

# If authentication error, API key is invalid
```

**Resolution**:
1. Verify API key created for Cluster Link service account
2. Check service account has `DeveloperRead` role on source cluster:
   ```bash
   confluent iam rolebinding list --principal User:<sa-link-id> --kafka-cluster <source-cluster-id>
   ```
3. If missing, add role binding:
   ```bash
   confluent iam rolebinding create \
     --principal User:<sa-link-id> \
     --role DeveloperRead \
     --kafka-cluster <source-cluster-id>
   ```
4. Recreate cluster link with corrected API key

---

### Issue 2: Mirror Lag Not Decreasing

**Symptom**:
```
Mirror lag stuck at 10M messages for 24+ hours
```

**Root Cause**: Network bandwidth saturation or destination cluster under-provisioned

**Diagnosis**:
```bash
# Check network bandwidth utilization (via Confluent Cloud metrics)
# Navigate to: Destination Cluster → Metrics → Network Throughput

# Check cluster utilization
confluent kafka cluster describe $DESTINATION_CLUSTER_ID -o json | jq '.usage'

# Check for throttling (Destination cluster CPU/disk saturation)
# Metrics: CPU %, Disk %
```

**Resolution**:
1. **If network bandwidth saturated**:
   - Increase Cluster Link parallelism (automatic, but verify no rate limiting)
   - Reduce source cluster write load temporarily (if possible)

2. **If destination cluster under-provisioned**:
   - Scale up destination cluster CKUs:
     ```bash
     confluent kafka cluster update $DESTINATION_CLUSTER_ID --cku 8  # Example: 4 → 8
     ```
   - Wait for provisioning, monitor lag reduction

3. **If Cluster Link in ERROR state**:
   - Check link status:
     ```bash
     confluent kafka link describe prod-migration-link --cluster $DESTINATION_CLUSTER_ID
     ```
   - If ERROR, check detailed error message, resolve issue, restart link

---

### Issue 3: Consumer Offset Translation Failure

**Symptom**:
```
Consumers start from offset 0 on destination (reprocess all data)
```

**Root Cause**: `consumer.offset.sync.enable=false` or offset sync lag >60 seconds at cutover

**Diagnosis**:
```bash
# Check Cluster Link config
confluent kafka link describe prod-migration-link --cluster $DESTINATION_CLUSTER_ID -o json \
  | jq '.configs[] | select(.name=="consumer.offset.sync.enable")'

# Expected: "true"

# Check offset sync lag
# Compare source consumer group offset vs destination translated offset
kafka-consumer-groups --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe --group analytics-group | grep orders | awk '{print $6}'  # Current offset

kafka-consumer-groups --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --describe --group analytics-group | grep orders | awk '{print $6}'  # Translated offset

# If difference >1000, offset sync not working
```

**Resolution**:
1. **If `consumer.offset.sync.enable=false`**:
   - Update Cluster Link config:
     ```bash
     confluent kafka link update prod-migration-link \
       --cluster $DESTINATION_CLUSTER_ID \
       --config consumer.offset.sync.enable=true
     ```
   - Wait 30-60 seconds for sync to activate

2. **If offset sync lag high**:
   - Decrease sync interval:
     ```bash
     confluent kafka link update prod-migration-link \
       --cluster $DESTINATION_CLUSTER_ID \
       --config consumer.offset.sync.ms=10000  # 10 seconds instead of 30
     ```
   - Wait for sync to catch up before retrying cutover

3. **If offset translation still fails** (manual reset):
   ```bash
   # Reset to timestamp at cutover time
   kafka-consumer-groups --bootstrap-server <destination-bootstrap>:9092 \
     --command-config destination-client.properties \
     --reset-offsets --to-datetime 2026-04-20T14:30:00.000 \
     --group analytics-group \
     --all-topics \
     --execute
   ```

---

## Kafka Connectivity Issues

### Issue 4: Application Cannot Connect to Destination Cluster

**Symptom**:
```
Producer error: Failed to resolve bootstrap server
OR
Connection timeout to <destination-bootstrap>:9092
```

**Root Cause**: Network connectivity or DNS resolution failure

**Diagnosis**:
```bash
# From application server/pod, test DNS resolution
nslookup <destination-bootstrap-server>

# Expected: Private IP address (if VPC peering/PrivateLink)
# If timeout or public IP, DNS issue

# Test TCP connectivity
telnet <destination-bootstrap-server> 9092
# OR
nc -zv <destination-bootstrap-server> 9092

# Expected: Connection successful
# If timeout, network routing or security group issue
```

**Resolution**:
1. **DNS resolution fails**:
   - Verify VPC DNS settings enabled (AWS: `enableDnsSupport=true`)
   - Check VPC peering DNS resolution enabled
   - For PrivateLink, verify private DNS enabled on VPC endpoint

2. **Connection timeout**:
   - Check security group rules on application side (egress to 9092 allowed)
   - Check NACL rules (if using AWS Network ACLs)
   - Verify VPC peering route tables updated:
     ```bash
     aws ec2 describe-route-tables --route-table-ids rtb-<id>
     # Should show route for Confluent Cloud CIDR → peering connection
     ```
   - Test from bastion host in same VPC to isolate issue

3. **Authentication failure after connectivity established**:
   - Verify API key correct in application config
   - Test API key manually (see Issue 1 diagnosis)

---

## Schema Mismatch Problems

### Issue 5: Producer Fails with Schema Compatibility Error

**Symptom**:
```
Error: Schema being registered is incompatible with an earlier schema
```

**Root Cause**: Schema evolution violates compatibility rules

**Diagnosis**:
```bash
# Check schema compatibility mode
curl -u $SR_KEY:$SR_SECRET $SR_ENDPOINT/config/$SUBJECT
# Example subject: orders-value

# Expected: {"compatibilityLevel":"BACKWARD"} or FORWARD/FULL

# Check latest schema version
curl -u $SR_KEY:$SR_SECRET $SR_ENDPOINT/subjects/$SUBJECT/versions/latest

# Compare with schema producer is trying to register
```

**Resolution**:
1. **If schema change is intentional**:
   - Fix schema to be compatible (e.g., make new field optional for BACKWARD compatibility)
   - Re-register schema

2. **If compatibility mode too strict**:
   - Change compatibility mode (ONLY if business logic allows):
     ```bash
     curl -X PUT \
       -u $SR_KEY:$SR_SECRET \
       $SR_ENDPOINT/config/$SUBJECT \
       -H "Content-Type: application/json" \
       -d '{"compatibility":"NONE"}'
     ```
   - Re-register schema
   - **WARNING**: NONE compatibility allows breaking changes, risk consumer failures

3. **If schema change accidental**:
   - Revert producer code to previous schema
   - Redeploy

---

## Connector Failures

### Issue 6: Kafka Connect Connector Stuck in FAILED State

**Symptom**:
```
Connector State: FAILED
Task State: FAILED
Error: Not authorized to access topic 'orders'
```

**Root Cause**: Missing ACLs for connector service account on destination cluster

**Diagnosis**:
```bash
# Check connector error details
confluent connect describe <connector-name> --cluster <connect-cluster-id>

# Common errors:
# - "Not authorized" → ACL missing
# - "Unknown topic" → Topic doesn't exist on destination
# - "Connection refused" → Network issue

# Check ACLs for connector service account
kafka-acls --bootstrap-server <destination-bootstrap>:9092 \
  --command-config destination-client.properties \
  --list --principal User:<connector-sa-id>
```

**Resolution**:
1. **For authorization errors**:
   ```bash
   # Grant necessary ACLs (example for sink connector)
   kafka-acls --bootstrap-server <destination-bootstrap>:9092 \
     --command-config destination-client.properties \
     --add \
     --allow-principal User:<connector-sa-id> \
     --operation READ \
     --topic orders
   
   kafka-acls --bootstrap-server <destination-bootstrap>:9092 \
     --command-config destination-client.properties \
     --add \
     --allow-principal User:<connector-sa-id> \
     --operation READ \
     --group connect-<connector-name>
   ```

2. **For missing topic errors**:
   - Verify topic exists on destination:
     ```bash
     confluent kafka topic list --cluster $DESTINATION_CLUSTER_ID | grep orders
     ```
   - If missing, check mirror topic creation succeeded

3. **Restart connector**:
   ```bash
   confluent connect restart <connector-name> --cluster <connect-cluster-id>
   ```

---

## Rollback Scenarios

### Issue 7: Need to Rollback During Cutover

**Trigger**: Producer error rate >5% on destination cluster

**Immediate Actions** (within 5 minutes):

```bash
# STEP 1: STOP all traffic to destination (IMMEDIATE)
# Scale producers to 0
kubectl scale deployment/producer-orders --replicas=0

# Scale consumers to 0
kubectl scale deployment/consumer-analytics --replicas=0

# STEP 2: Revert producer config to source
kubectl set env deployment/producer-orders \
  KAFKA_BOOTSTRAP_SERVERS=<source-bootstrap>:9092 \
  KAFKA_API_KEY=<original-source-key> \
  KAFKA_API_SECRET=<original-source-secret>

# STEP 3: Restart producers on source
kubectl scale deployment/producer-orders --replicas=20

# STEP 4: Wait for producers to stabilize (2-3 min)
# Monitor error rates

# STEP 5: Revert consumer config to source
kubectl set env deployment/consumer-analytics \
  KAFKA_BOOTSTRAP_SERVERS=<source-bootstrap>:9092 \
  KAFKA_API_KEY=<original-source-key> \
  KAFKA_API_SECRET=<original-source-secret>

# STEP 6: Restart consumers on source
kubectl scale deployment/consumer-analytics --replicas=10

# STEP 7: Verify source cluster traffic restored
kafka-topics --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe --topic orders

# STEP 8: Post-rollback analysis
# - Investigate root cause of destination errors
# - Document lessons learned
# - Plan retry migration
```

**Total Rollback Time**: <15 minutes from decision to full traffic on source

---

# 7. BEST PRACTICES

## Zero-Downtime Migration Tips

### 1. Phased Producer Rollout

**Best Practice**: NEVER switch all producers at once

**Strategy**:
```
Canary (5%) → Monitor 30 min → Incremental (25%) → Monitor 15 min → 
Incremental (50%) → Monitor 15 min → Full (100%) → Monitor 1 hour
```

**Rationale**:
- Limits blast radius if issues arise
- Provides early warning signals
- Enables fast rollback with minimal impact

**Implementation**:
- Use separate deployments for canary vs. main
- Monitor key metrics during each phase: error rate, latency, throughput
- Automate rollout with CI/CD (gradual rollout strategies)

### 2. Consumer Offset Validation Before Cutover

**Best Practice**: Validate offset translation for EVERY consumer group before switching consumers

**Procedure**:
```bash
# Create validation script
cat > validate-all-offsets.sh << 'EOF'
#!/bin/bash
GROUPS=$(kafka-consumer-groups --bootstrap-server <source> --command-config source.properties --list)

for group in $GROUPS; do
  echo "Validating group: $group"
  
  src_offset=$(kafka-consumer-groups --bootstrap-server <source> --command-config source.properties --describe --group $group | grep -v "^GROUP" | awk '{sum+=$6} END {print sum}')
  
  dest_offset=$(kafka-consumer-groups --bootstrap-server <destination> --command-config destination.properties --describe --group $group | grep -v "^GROUP" | awk '{sum+=$6} END {print sum}')
  
  diff=$((src_offset - dest_offset))
  
  if [ ${diff#-} -gt 1000 ]; then
    echo "ERROR: Group $group offset mismatch: $diff"
    exit 1
  else
    echo "OK: Group $group offset difference: $diff"
  fi
done

echo "All consumer groups validated!"
EOF

chmod +x validate-all-offsets.sh
./validate-all-offsets.sh
```

**Rationale**: Prevents massive reprocessing (consumers starting from offset 0) which can:
- Overload downstream systems
- Cause duplicate transactions
- Violate business logic (idempotency assumptions)

### 3. Maintain Cluster Link Active Post-Cutover

**Best Practice**: Keep Cluster Link active for 7-14 days after cutover

**Rationale**:
- Enables fast rollback if issues discovered post-cutover
- Provides safety net during validation period
- Minimal cost (replication stopped once lag=0)

**Implementation**:
- Do NOT delete Cluster Link immediately after cutover
- Monitor link status (should show lag=0)
- Delete only after decommission approval

### 4. Dry Run in Staging Environment

**Best Practice**: Execute full migration rehearsal in staging BEFORE production

**Staging Rehearsal Checklist**:
- [ ] Create staging source and destination clusters (smaller size)
- [ ] Replicate production topic configurations
- [ ] Run full migration procedure (Cluster Link, cutover, validation)
- [ ] Measure cutover time (use as estimate for production)
- [ ] Test rollback procedure
- [ ] Document lessons learned, update runbook
- [ ] Conduct second rehearsal incorporating improvements

**Rationale**: Identifies issues in safe environment, builds team confidence, reduces production risk

---

## Monitoring Strategy During Migration

### Pre-Migration Baseline Metrics

**Capture baselines 7 days before migration**:

| Metric | Source | Threshold |
|--------|--------|-----------|
| Producer throughput | Confluent Cloud Metrics | 50 MB/s (example) |
| Producer p99 latency | Application metrics | 15ms |
| Consumer lag | Confluent Cloud Metrics | <5000 messages |
| Consumer throughput | Application metrics | 100 MB/s |
| Error rate | Application logs | <0.01% |
| End-to-end latency | Distributed tracing | <500ms |

**Purpose**: Compare post-migration performance to baseline to detect regressions

### During Migration Monitoring

**Critical Metrics to Monitor**:

1. **Cluster Link Replication Lag**:
   - Tool: Confluent Cloud Console → Cluster Linking → Metrics
   - Alert: Lag increasing for >30 minutes
   - Action: Investigate network/destination cluster capacity

2. **Producer Error Rate**:
   - Tool: Application metrics (Prometheus, Datadog, etc.)
   - Alert: Error rate >1% during canary rollout
   - Action: Pause rollout, investigate errors, consider rollback

3. **Consumer Lag**:
   - Tool: Confluent Cloud Console → Consumers → Lag
   - Alert: Lag increasing >10K messages/min
   - Action: Investigate consumer processing bottleneck

4. **Application Health**:
   - Tool: Kubernetes liveness/readiness probes
   - Alert: Pod restart rate >10% during migration
   - Action: Investigate application errors, check configs

5. **Network Connectivity**:
   - Tool: VPC flow logs, network monitoring
   - Alert: Connection failures to destination cluster
   - Action: Validate VPC peering/PrivateLink, security groups

**Monitoring Dashboard**: Create dedicated dashboard with all metrics in single view

### Post-Migration Monitoring

**Intensive monitoring period**: First 24 hours post-cutover

**Continued monitoring**: 2 weeks with daily reviews

**Metrics to Track**:
- All baseline metrics (compare to pre-migration baseline)
- Consumer offset drift (should be minimal)
- Schema Registry latency
- Kafka Connect connector task status

**Alerts**:
- Any baseline metric >20% worse than pre-migration
- Consumer lag >10K messages sustained for >10 minutes
- Error rate >0.1%

---

## Version Alignment Strategy

### Kafka Version Compatibility

**Best Practice**: Minimize version gap between source and destination

**Compatibility Matrix**:

| Source Version | Destination Version | Risk Level | Recommendation |
|----------------|---------------------|------------|----------------|
| 3.4.x | 3.4.x | Low | Ideal, proceed |
| 3.3.x | 3.4.x | Low | Safe, minor version jump |
| 3.0.x | 3.5.x | Medium | Test in staging first |
| 2.8.x | 3.5.x | High | Consider upgrading source first |
| <2.8 | Any | Not Supported | Upgrade source cluster before migration |

**If Source Version Too Old**:
1. Upgrade source cluster to compatible version (2.8+)
2. Wait 1-2 weeks for stability
3. Proceed with migration

### Client Library Version Alignment

**Best Practice**: Ensure producer/consumer libraries compatible with destination Kafka version

**Compatibility Check**:
```bash
# Check Kafka client library version in application
# Java example:
grep -r "kafka-clients" pom.xml  # Maven
grep -r "kafka-clients" build.gradle  # Gradle

# Expected: kafka-clients:3.4.0 or higher (matching destination cluster)
```

**If Incompatible**:
- Upgrade client libraries in application code
- Test thoroughly in staging
- Deploy updated applications BEFORE migration

---

## Security Best Practices

### 1. API Key Rotation Post-Migration

**Best Practice**: Rotate all API keys 30 days post-migration

**Rationale**: Migration exposes API keys in configs, logs, and deployment systems. Rotation ensures compromised keys expire.

**Procedure**:
```bash
# Create new API keys
confluent api-key create --service-account <sa-id> --resource $DESTINATION_CLUSTER_ID

# Update application configs with new keys
# (via secrets manager, Kubernetes secrets, etc.)

# Wait for all applications using new keys (gradual rollout)

# Delete old API keys
confluent api-key delete <old-api-key-id>
```

### 2. Principle of Least Privilege

**Best Practice**: Grant minimum required ACLs to each service account

**Example**:
```bash
# Producer: Only WRITE permission to specific topics (not all topics)
kafka-acls --add --allow-principal User:<producer-sa> --operation WRITE --topic orders
# NOT: --operation ALL --topic '*'

# Consumer: Only READ permission to specific topics and consumer group
kafka-acls --add --allow-principal User:<consumer-sa> --operation READ --topic orders
kafka-acls --add --allow-principal User:<consumer-sa> --operation READ --group analytics-group
# NOT: --operation ALL
```

### 3. Audit Logging

**Best Practice**: Enable audit logging on destination cluster for compliance

**Configuration**: Confluent Cloud automatically logs cluster events (topic create/delete, ACL changes, etc.)

**Access**: Via Confluent Cloud Console → Audit Logs

**Retention**: 90 days (default, configurable for Enterprise)

---

## Cost and Performance Considerations

### Cost Optimization

**Expected Cost Increase**: 30-40% from single-zone to multi-zone

**Breakdown**:
| Component | Single-Zone Cost | Multi-Zone Cost | Increase |
|-----------|------------------|-----------------|----------|
| Cluster compute (CKUs) | $1.50/CKU/hr × 4 CKUs | $2.00/CKU/hr × 4 CKUs | +33% |
| Storage (RF=1 → RF=3) | $0.10/GB × 1TB | $0.10/GB × 3TB | +200% |
| Cross-zone data transfer | $0 | $0.01/GB × transfer | +Variable |
| **Total** | ~$2,000/mo | ~$2,800/mo | +40% |

**Optimization Strategies**:

1. **Right-Size Cluster**:
   - After 2 weeks, analyze actual utilization
   - Scale down CKUs if utilization <40%
   - Example: 6 CKUs → 4 CKUs saves $1,200/month

2. **Optimize Retention Policies**:
   ```bash
   # Reduce retention for non-critical topics
   confluent kafka topic update logs-topic \
     --cluster $DESTINATION_CLUSTER_ID \
     --config retention.ms=259200000  # 3 days instead of 7
   ```
   - Reduces storage costs proportionally

3. **Enable Tiered Storage** (if available):
   ```bash
   # Move old data to cheaper object storage (S3/GCS)
   confluent kafka topic update orders \
     --cluster $DESTINATION_CLUSTER_ID \
     --config remote.storage.enable=true \
     --config local.retention.ms=86400000  # Keep 1 day local, rest in S3
   ```
   - Reduces local storage costs by 50-70%

4. **Compression**:
   ```bash
   # Enable compression on topics to reduce storage and bandwidth
   confluent kafka topic update orders \
     --cluster $DESTINATION_CLUSTER_ID \
     --config compression.type=snappy
   ```
   - Reduces storage by 30-50% (data-dependent)
   - Reduces cross-zone bandwidth costs

### Performance Tuning

**Challenge**: Multi-zone cross-zone replication adds latency

**Mitigation**:

1. **Producer Batching**:
   ```properties
   linger.ms=10-50         # Wait up to 50ms to batch messages
   batch.size=32768        # 32KB batches (or larger)
   compression.type=snappy # Compress batches
   ```
   - Reduces number of cross-zone requests
   - Amortizes latency overhead across many messages

2. **Consumer Fetch Optimization**:
   ```properties
   fetch.min.bytes=50000   # Wait for 50KB before fetch
   fetch.max.wait.ms=500   # Max wait 500ms
   max.partition.fetch.bytes=1048576  # 1MB per partition
   ```
   - Reduces cross-zone fetch requests
   - Increases throughput, slight latency increase acceptable

3. **Monitor Latency**:
   - Producer p99 latency: Target <50ms (vs <20ms single-zone)
   - Consumer lag: Target <5000 messages (same as single-zone)
   - End-to-end latency: Target <1s (acceptable increase from cross-zone)

**Acceptance Criteria**:
- Latency increase <2x baseline → acceptable tradeoff for resilience
- Latency increase >2x baseline → investigate producer tuning, consider rollback if business SLA breached

---

## Summary: Golden Rules for Successful Migration

1. **Rehearse in Staging**: Full dress rehearsal catches 90% of issues
2. **Validate Offsets Obsessively**: Consumer offset translation is #1 cause of migration failures
3. **Phased Rollout**: Never switch all traffic at once (canary → incremental)
4. **Keep Cluster Link Active**: Safety net for 7-14 days post-cutover
5. **Monitor Intensively**: First 24 hours post-cutover are critical
6. **Test Rollback**: Ensure team can execute rollback in <15 minutes
7. **Communicate Clearly**: Business stakeholders, SREs, developers all aligned on timeline and expectations
8. **Document Everything**: Runbook, lessons learned, metrics captured for future migrations
9. **Optimize Post-Migration**: Right-size cluster, tune retention, enable compression after stability confirmed
10. **Celebrate Success**: Migration is a major achievement—recognize team effort!

---

**END OF ENTERPRISE GUIDE**

*This guide is a simplified, restructured interpretation of the original runbook, tailored for enterprise stakeholders at varying technical levels. For detailed command examples and comprehensive troubleshooting, refer to the [original runbook](https://github.com/ManiselvanSE/confluent-migration-runbook).*
