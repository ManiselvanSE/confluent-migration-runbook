# Confluent Cloud Migration Runbook
## Single-Zone Dedicated → Multi-Zone Enterprise with Cluster Linking

**Migration Strategy:** Cluster Linking (Active-Passive)  
**Target RPO:** 0 (no data loss)  
**Target RTO:** < 5 minutes

---

## Migration Overview & Agenda

### Executive Summary

This runbook guides you through migrating a **Single-Zone Confluent Cloud Dedicated cluster** to a **Multi-Zone Enterprise cluster** using **Cluster Linking**. The migration achieves near-zero downtime (<5 minutes consumer interruption), zero data loss, and automatic consumer offset preservation.

**Business Outcomes:**
- 🎯 **Availability**: 99.5% → 99.99% uptime
- 🛡️ **Resilience**: Survive complete availability zone failures
- 📊 **Durability**: RF=1/2 → RF=3 with cross-zone replication
- ⚡ **Downtime**: <5 minutes (consumer restart only)

---

### Migration Timeline

**Total Duration**: ~10 weeks (for typical 100TB cluster, 500 topics)

```
┌─────────────────────────────────────────────────────────────┐
│                    MIGRATION PHASES                          │
└─────────────────────────────────────────────────────────────┘

Week 1-2   │ Pre-Migration Assessment
           │ • Inventory topics, consumers, connectors
           │ • Analyze traffic patterns and dependencies
           │ • Capacity planning and risk assessment
           │
Week 3     │ Architecture Design & Staging
           │ • Design target multi-zone architecture
           │ • Staging environment rehearsal
           │ • Network setup (VPC peering/PrivateLink)
           │
Week 4     │ Target Cluster Provisioning
           │ • Provision Multi-AZ Enterprise cluster
           │ • Configure networking and security
           │ • Create service accounts and API keys
           │
Week 4-5   │ Cluster Linking & Replication Sync
           │ • Establish Cluster Link (source → destination)
           │ • Create mirror topics
           │ • Monitor replication lag (10M messages → 0)
           │
Week 6     │ Production Cutover (4-hour window)
           │ • Migrate producers (phased rollout)
           │ • Migrate consumers (offset translation)
           │ • Reconfigure Kafka Connect
           │
Week 6-8   │ Validation & Monitoring
           │ • 24/7 monitoring period
           │ • Performance tuning
           │ • Data integrity validation
           │
Week 10    │ Decommission
           │ • Delete Cluster Link
           │ • Delete source cluster
           │ • Final documentation update
```

---

### High-Level Migration Steps

#### **Phase 1: Pre-Migration (Weeks 1-2)** 📋
**Objective**: Understand current state and plan migration

1. **Inventory & Assessment**
   - Topic configurations (partitions, RF, retention)
   - Consumer groups and lag analysis
   - Producer/consumer patterns
   - Dependencies (Schema Registry, Connect, ksqlDB)

2. **Risk & Capacity Planning**
   - Identify single points of failure
   - Calculate target cluster capacity
   - Security audit (ACLs, API keys)
   - Define rollback triggers

**Deliverable**: Complete inventory spreadsheet, risk assessment

---

#### **Phase 2: Architecture & Design (Week 3)** 🏗️
**Objective**: Design target state and validate approach

1. **Target Architecture Design**
   - Multi-zone cluster topology
   - Replication factor strategy (RF=3)
   - Network design (VPC peering vs PrivateLink)
   - Security architecture (ACLs, TLS, encryption)

2. **Staging Rehearsal**
   - Execute full migration in staging environment
   - Measure cutover timing
   - Test rollback procedure
   - Document lessons learned

**Deliverable**: Architecture diagrams, staging validation report

---

#### **Phase 3: Infrastructure Provisioning (Week 4)** ⚙️
**Objective**: Prepare destination cluster and networking

1. **Provision Multi-AZ Cluster**
   - Create Enterprise cluster (3 availability zones)
   - Configure cluster-level settings
   - Validate provisioning

2. **Network & Security Setup**
   - Establish VPC peering or PrivateLink
   - Configure security groups and route tables
   - Create service accounts and API keys
   - Replicate ACLs/RBAC policies

**Deliverable**: Operational multi-zone cluster, network connectivity validated

---

#### **Phase 4: Cluster Linking Setup (Weeks 4-5)** 🔗
**Objective**: Establish replication from source to destination

1. **Create Cluster Link**
   - Configure Cluster Link (destination → source)
   - Enable consumer offset sync
   - Enable ACL sync

2. **Create Mirror Topics**
   - Mirror all production topics
   - Validate configuration sync
   - Monitor replication lag

3. **Replication Sync (5-7 days)**
   - Wait for lag to reach near-zero
   - Monitor continuously
   - Validate offset translation

**Cutover Readiness Criteria**:
- ✅ Mirror lag < 1000 messages (all topics)
- ✅ Lag stable for 30+ minutes
- ✅ Consumer offset sync functional
- ✅ No errors in Cluster Link state

**Deliverable**: Fully synchronized destination cluster

---

#### **Phase 5: Application Migration (Week 6)** 🚀
**Objective**: Switch traffic to destination cluster

**Cutover Window**: 4 hours (actual execution ~30 minutes)

1. **Producer Migration** (Rolling, Zero Downtime)
   - Canary rollout (5% → 25% → 50% → 100%)
   - Update bootstrap servers and API keys
   - Validate traffic switch

2. **Consumer Migration** (<5 min downtime)
   - Graceful shutdown on source
   - Validate offset translation
   - Start consumers on destination
   - Resume from exact offset

3. **Kafka Connect Migration**
   - Pause connectors
   - Reconfigure bootstrap servers
   - Resume connectors
   - Validate data flow

**Deliverable**: All traffic on destination cluster, zero data loss

---

#### **Phase 6: Validation & Testing (Weeks 6-8)** ✅
**Objective**: Confirm migration success and stability

1. **Data Integrity Validation**
   - Compare message counts (source vs destination)
   - Validate consumer offsets
   - Schema Registry compatibility check

2. **Performance Validation**
   - Throughput benchmarks
   - Latency measurements
   - Consumer lag analysis
   - Failover testing (zone failure simulation)

3. **Monitoring Period**
   - 24/7 monitoring for 2 weeks
   - Performance tuning
   - Issue resolution

**Deliverable**: Validated production cluster, performance report

---

#### **Phase 7: Rollback Plan** 🔄
**Objective**: Revert to source cluster if critical issues arise

**Rollback Triggers**:
- Data loss detected (>1% message count mismatch)
- Producer error rate >5%
- Consumer lag uncontrollable
- Authentication failures affecting >10% of clients

**Rollback Procedure** (Target: <15 minutes):
1. Stop destination traffic
2. Revert producer/consumer configs to source
3. Restart on source cluster
4. Validate source cluster operational

**Note**: Cluster Link remains active for 7-14 days as safety net

---

#### **Phase 8: Post-Migration (Week 10)** 🎯
**Objective**: Optimize and decommission old infrastructure

1. **Decommission Source Cluster**
   - Wait 7-14 days for validation
   - Delete Cluster Link
   - Delete source cluster
   - Clean up networking resources

2. **Optimization**
   - Right-size cluster (CKU tuning)
   - Optimize retention policies
   - Enable tiered storage
   - Cost optimization (compression, batching)

**Deliverable**: Optimized production multi-zone cluster

---

#### **Phase 9: Common Pitfalls & Best Practices** ⚠️

**Top 5 Pitfalls to Avoid**:
1. **Incorrect Offset Translation** → Validate ALL consumer groups before cutover
2. **ACL Mismatches** → Enable ACL sync, diff before and after
3. **Latency Degradation** → Tune producer batching for cross-zone
4. **Under-Partitioned Topics** → Increase partitions before migration
5. **Skipped Staging Test** → ALWAYS rehearse in staging first

**Best Practices**:
- Use phased producer rollout (5% → 100%)
- Monitor replication lag continuously
- Keep Cluster Link active 7-14 days post-cutover
- Test rollback procedure in staging
- Freeze schema changes during cutover window

---

### Appendices

**Appendix A**: Command Reference (Confluent CLI, Kafka CLI)  
**Appendix B**: Troubleshooting Guide (common failures and resolutions)  
**Appendix C**: Migration Timeline Template (Excel/spreadsheet)

---

### Document Usage

**👨‍💼 For Executives**: Read Migration Overview & Timeline  
**🏗️ For Architects**: Focus on Phases 2-3 (Architecture & Design)  
**👨‍💻 For Implementation Teams**: Follow Phases 1-8 sequentially  
**🚨 For Incident Response**: Jump to Phase 7 (Rollback Plan)

---

## 1. Pre-Migration Assessment

### 1.1 Topic Inventory

```bash
# List all topics with configurations
confluent kafka topic list --cluster <source-cluster-id>

# Get detailed topic config for each topic
confluent kafka topic describe <topic-name> --cluster <source-cluster-id>

# Export topic configurations
for topic in $(confluent kafka topic list --cluster <source-cluster-id> -o json | jq -r '.[].name'); do
  confluent kafka topic describe $topic --cluster <source-cluster-id> -o json > "topic-config-${topic}.json"
done

# Analyze partition distribution
kafka-topics --bootstrap-server <bootstrap-server> \
  --command-config client.properties \
  --describe | tee topics-inventory.txt
```

**Collect:**
- Topic names, partition counts, replication factors
- Retention policies (time/size)
- Cleanup policies (delete/compact)
- Custom configurations (min.insync.replicas, compression.type, etc.)
- Topic size and message throughput
- **Critical:** Note topics with RF=1 (single-zone) → will become RF=3 (multi-zone)

### 1.2 Producer & Consumer Analysis

```bash
# List consumer groups
confluent kafka consumer group list --cluster <source-cluster-id>

# Get consumer group details and lag
for group in $(confluent kafka consumer group list --cluster <source-cluster-id> -o json | jq -r '.[].consumer_group_id'); do
  confluent kafka consumer group describe $group --cluster <source-cluster-id> > "consumer-group-${group}.txt"
done

# Analyze traffic patterns (requires Confluent Cloud metrics API)
curl -X GET "https://api.telemetry.confluent.cloud/v2/metrics/cloud/query" \
  -H "Authorization: Basic $(echo -n '<api-key>:<api-secret>' | base64)" \
  -d '{
    "aggregations": [{"metric": "io.confluent.kafka.server/received_bytes"}],
    "filter": {"field": "resource.kafka.id", "op": "EQ", "value": "<cluster-id>"},
    "granularity": "PT1H",
    "intervals": ["2026-04-13T00:00:00Z/2026-04-20T00:00:00Z"]
  }'
```

**Document:**
- Producer count and write patterns
- Consumer groups and subscription patterns
- Peak throughput (MB/s, messages/s)
- Consumer lag patterns
- Idempotency and transactional configurations
- Exactly-once vs at-least-once semantics

### 1.3 Dependency Mapping

**Schema Registry:**
```bash
# List all schemas
curl -u <sr-api-key>:<sr-api-secret> \
  https://<schema-registry-url>/subjects | jq .

# Export schema compatibility settings
curl -u <sr-api-key>:<sr-api-secret> \
  https://<schema-registry-url>/config
```

**Kafka Connect:**
```bash
# List connectors
confluent connect list --cluster <connect-cluster-id>

# Export connector configurations
for connector in $(confluent connect list --cluster <connect-cluster-id> -o json | jq -r '.[].name'); do
  confluent connect describe $connector --cluster <connect-cluster-id> -o json > "connector-${connector}.json"
done
```

**ksqlDB:**
```bash
# List ksqlDB clusters
confluent ksql cluster list

# Export queries and schemas (manual)
SHOW QUERIES;
SHOW STREAMS;
SHOW TABLES;
```

### 1.4 Current Architecture Limitations

**Single-Zone Dedicated Constraints:**
- **Availability:** RF=1 or RF=2 → single zone failure = data loss or unavailability
- **Durability:** min.insync.replicas typically ≤ 1
- **Fault tolerance:** No cross-zone redundancy
- **Failover:** No automatic zone-level failover

**Expected Multi-Zone Improvements:**
- **Availability:** RF=3 across 3 zones → survives single zone failure
- **Durability:** min.insync.replicas=2 (recommended)
- **Latency:** Expect +2-5ms p99 latency for cross-zone replication
- **Cost:** ~30-40% increase due to cross-zone data transfer and higher RF

### 1.5 Quotas & Capacity Planning

```bash
# Check current quotas
confluent kafka quota list --cluster <source-cluster-id>

# Review cluster capacity
confluent kafka cluster describe <source-cluster-id>
```

**Calculate target capacity:**
- Multiply storage by RF increase (RF=1 → RF=3 = 3x storage)
- Account for 20% growth buffer
- Cross-zone bandwidth costs (ingress free, egress charged)

### 1.6 Security Audit

```bash
# Export ACLs
kafka-acls --bootstrap-server <bootstrap-server> \
  --command-config client.properties \
  --list > acls-export.txt

# List API keys
confluent api-key list --resource <source-cluster-id>

# Check RBAC bindings (if using)
confluent iam rolebinding list --kafka-cluster <source-cluster-id>
```

**Validate:**
- Service account inventories
- API key rotation requirements
- Network security (IP allowlists, PrivateLink)
- Encryption at rest/in transit

### 1.7 Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Cluster Linking lag during peak hours | Medium | High | Schedule during low-traffic window; over-provision target |
| Consumer offset sync delay | Medium | Medium | Use offset translation; validate before cutover |
| Cross-zone latency increase | High | Medium | Test producer acks=all performance; tune request.timeout.ms |
| ACL mismatch post-migration | Low | High | Automated ACL replication script with validation |
| Ordering violation during cutover | Low | Critical | Use mirror topics in read-only mode until full cutover |
| Schema Registry compatibility break | Low | High | Test schema evolution in staging environment |

---

## 2. Architecture Design

### 2.1 Multi-Zone Architecture

```
┌─────────────────────────────────────────────────────────────┐
│             Multi-Zone Enterprise Cluster                    │
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Zone A    │    │   Zone B    │    │   Zone C    │     │
│  │             │    │             │    │             │     │
│  │  Broker 1   │    │  Broker 2   │    │  Broker 3   │     │
│  │  Broker 4   │    │  Broker 5   │    │  Broker 6   │     │
│  │             │    │             │    │             │     │
│  │ Replica 1   │◄──►│ Replica 2   │◄──►│ Replica 3   │     │
│  │ (Leader)    │    │ (Follower)  │    │ (Follower)  │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                               │
│  Replication Factor: 3 (one replica per zone)                │
│  min.insync.replicas: 2 (recommended)                        │
└─────────────────────────────────────────────────────────────┘
```

**Key Configuration Changes:**

| Parameter | Single-Zone | Multi-Zone | Impact |
|-----------|-------------|------------|--------|
| `replication.factor` | 1-2 | 3 | 3x storage, zone-level HA |
| `min.insync.replicas` | 1 | 2 | Prevents data loss if 1 zone fails |
| `acks` (producer) | 1 or all | all | Ensures writes to 2+ zones before ACK |
| `unclean.leader.election` | false | false | Never elect out-of-sync replica |

### 2.2 Cross-Zone Latency Considerations

**Expected Latency Profile:**
- Intra-zone: 1-3ms p99
- Cross-zone (same region): 2-5ms p99
- **Total p99 produce latency increase:** +3-7ms with acks=all

**Mitigation Strategies:**
```properties
# Producer tuning for multi-zone
linger.ms=10                    # Batch more aggressively
batch.size=32768               # Larger batches
compression.type=snappy        # Reduce cross-zone bandwidth
request.timeout.ms=30000       # Account for cross-zone latency

# Consumer tuning
fetch.min.bytes=1024           # Reduce cross-zone fetch requests
fetch.max.wait.ms=500          # Balance latency vs throughput
```

### 2.3 Cluster Linking Design

**Architecture Pattern:** Active-Passive with Cluster Link

```
┌───────────────────────┐         ┌───────────────────────┐
│  Source Cluster       │         │  Destination Cluster  │
│  (Single-Zone)        │         │  (Multi-Zone)         │
│                       │         │                       │
│  ┌─────────────────┐  │         │  ┌─────────────────┐  │
│  │ Topic: orders   │  │ Cluster │  │ Topic: orders   │  │
│  │ RF: 1           │  │  Link   │  │ RF: 3           │  │
│  │ Partitions: 12  │──┼────────►│  │ Partitions: 12  │  │
│  └─────────────────┘  │         │  └─────────────────┘  │
│                       │         │                       │
│  Producers ──────────►│         │                       │
│  Consumers ──────────►│         │  Consumers (standby)  │
│                       │         │                       │
└───────────────────────┘         └───────────────────────┘
                                           │
                       ┌───────────────────┘
                       │ CUTOVER
                       ▼
                  ┌────────────────────────┐
                  │ Producers switched     │
                  │ Consumers switched     │
                  │ Cluster link drained   │
                  └────────────────────────┘
```

**Cluster Link Configuration Strategy:**

1. **Link Direction:** Source (Single-Zone) → Destination (Multi-Zone)
2. **Mirror Topic Mode:** Read-only mirrors until cutover
3. **Offset Sync:** Auto-offset translation enabled
4. **Config Sync:** Mirror topic configurations automatically

**Key Properties:**
```properties
# Cluster link replication
link.mode=DESTINATION                          # Link initiated from destination
consumer.offset.sync.enable=true               # Sync consumer offsets
consumer.offset.sync.ms=30000                 # Sync every 30s
consumer.offset.group.filters={"groupFilters": [{"name": "*","patternType": "LITERAL"}]}
acl.sync.enable=true                          # Sync ACLs automatically
acl.sync.ms=30000
```

---

## 3. Target Cluster Setup

### 3.1 Provision Multi-Zone Enterprise Cluster

```bash
# Create Enterprise cluster (via Confluent Cloud Console or API)
# Note: Use API for automation

curl -X POST https://api.confluent.cloud/cmk/v2/clusters \
  -H "Authorization: Basic $(echo -n '<cloud-api-key>:<cloud-api-secret>' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "spec": {
      "display_name": "enterprise-multi-zone-prod",
      "availability": "MULTI_ZONE",
      "cloud": "AWS",
      "region": "us-east-1",
      "config": {
        "kind": "Enterprise"
      },
      "environment": {
        "id": "<environment-id>"
      }
    }
  }'

# Wait for cluster provisioning (typically 15-30 minutes)
confluent kafka cluster describe <target-cluster-id> --output json | jq -r '.status'
```

**Validation Checklist:**
- [ ] Cluster status = PROVISIONED
- [ ] Bootstrap URL accessible
- [ ] All 3 availability zones active
- [ ] Encryption at rest enabled

### 3.2 Configure Networking

**VPC Peering (if using private networking):**

```bash
# Create VPC peering (AWS example)
confluent network peering create \
  --cloud aws \
  --region us-east-1 \
  --customer-vpc <your-vpc-id> \
  --customer-account <your-aws-account-id> \
  --customer-routes <cidr-blocks>

# Accept peering request in your AWS account
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id <pcx-id>

# Update route tables
aws ec2 create-route \
  --route-table-id <rtb-id> \
  --destination-cidr-block <confluent-cidr> \
  --vpc-peering-connection-id <pcx-id>
```

**PrivateLink (preferred for enterprise):**

```bash
# Create PrivateLink connection
confluent network private-link-access create \
  --cloud aws \
  --region us-east-1 \
  --network <network-id>

# Create VPC endpoint in your AWS account (use endpoint service name from above)
aws ec2 create-vpc-endpoint \
  --vpc-id <vpc-id> \
  --service-name <confluent-endpoint-service-name> \
  --subnet-ids <subnet-1> <subnet-2> <subnet-3>
```

### 3.3 Create Service Accounts & API Keys

```bash
# Create service account for cluster linking
confluent iam service-account create cluster-link-sa \
  --description "Service account for cluster linking"

# Capture service account ID
LINK_SA_ID=$(confluent iam service-account list -o json | jq -r '.[] | select(.name=="cluster-link-sa") | .id')

# Create API key for cluster link on SOURCE cluster
confluent api-key create \
  --service-account $LINK_SA_ID \
  --resource <source-cluster-id> \
  --description "Cluster link source"

# Create API key for cluster link on DESTINATION cluster
confluent api-key create \
  --service-account $LINK_SA_ID \
  --resource <target-cluster-id> \
  --description "Cluster link destination"

# Create API keys for applications (producers/consumers)
for app in producer-app consumer-app; do
  confluent iam service-account create ${app}-sa
  SA_ID=$(confluent iam service-account list -o json | jq -r ".[] | select(.name==\"${app}-sa\") | .id")
  confluent api-key create --service-account $SA_ID --resource <target-cluster-id>
done
```

### 3.4 Replicate ACLs/RBAC

**Option A: Automated ACL Sync via Cluster Link**
```properties
# Enable during cluster link creation
acl.sync.enable=true
acl.sync.ms=30000
acl.filters={"aclFilters": [{"patternType": "LITERAL"}]}
```

**Option B: Manual ACL Migration Script**
```bash
#!/bin/bash
# Export ACLs from source
kafka-acls --bootstrap-server <source-bootstrap> \
  --command-config source-client.properties \
  --list | grep "User:" > acls-export.txt

# Parse and apply to destination
while IFS= read -r line; do
  # Parse ACL line and convert to kafka-acls command
  # Example: User:sa-123456 has Allow permission for operations: Read from hosts: *
  
  if [[ $line =~ User:([^ ]+).*operations:\ ([^ ]+).*resource:\ ([^ ]+):([^ ]+) ]]; then
    principal="${BASH_REMATCH[1]}"
    operation="${BASH_REMATCH[2]}"
    resource_type="${BASH_REMATCH[3]}"
    resource_name="${BASH_REMATCH[4]}"
    
    kafka-acls --bootstrap-server <target-bootstrap> \
      --command-config target-client.properties \
      --add \
      --allow-principal "User:$principal" \
      --operation "$operation" \
      --$resource_type "$resource_name"
  fi
done < acls-export.txt
```

**RBAC Migration (if using Role-Based Access Control):**
```bash
# Export role bindings from source
confluent iam rolebinding list \
  --kafka-cluster <source-cluster-id> \
  -o json > rbac-export.json

# Apply to destination (requires scripting)
jq -c '.[]' rbac-export.json | while read binding; do
  principal=$(echo $binding | jq -r '.principal')
  role=$(echo $binding | jq -r '.role_name')
  resource=$(echo $binding | jq -r '.crn_pattern')
  
  confluent iam rolebinding create \
    --principal $principal \
    --role $role \
    --resource $resource \
    --kafka-cluster <target-cluster-id>
done
```

### 3.5 Configure Schema Registry

**If using dedicated Schema Registry:**

```bash
# Link Schema Registry to new cluster
# Schema Registry is regional and can serve multiple clusters

# Update Schema Registry ACLs for new cluster
confluent iam rolebinding create \
  --principal User:<service-account-id> \
  --role ResourceOwner \
  --resource Topic:_schemas \
  --kafka-cluster <target-cluster-id>

# Verify schema compatibility
curl -u <sr-api-key>:<sr-api-secret> \
  https://<schema-registry-url>/subjects | jq .
```

**Schema compatibility settings:**
```bash
# Ensure compatibility mode matches source
curl -X PUT \
  -u <sr-api-key>:<sr-api-secret> \
  https://<schema-registry-url>/config \
  -H "Content-Type: application/json" \
  -d '{"compatibility": "BACKWARD"}'
```

### 3.6 Validate Quotas and Limits

```bash
# Apply quota configurations to destination
# Copy quotas from source cluster assessment

# Example: Set producer quota
kafka-configs --bootstrap-server <target-bootstrap> \
  --command-config target-client.properties \
  --alter \
  --add-config 'producer_byte_rate=10485760' \
  --entity-type clients \
  --entity-name <client-id>

# Set consumer quota
kafka-configs --bootstrap-server <target-bootstrap> \
  --command-config target-client.properties \
  --alter \
  --add-config 'consumer_byte_rate=20971520' \
  --entity-type clients \
  --entity-name <client-id>
```

---

## 4. Cluster Linking Implementation

### 4.1 Create Cluster Link

**Step 1: Create link configuration file**

```bash
# destination-link.config
cat > destination-link.config << EOF
bootstrap.servers=<source-cluster-bootstrap>:9092
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<source-api-key>' password='<source-api-secret>';

# Consumer offset sync
consumer.offset.sync.enable=true
consumer.offset.sync.ms=30000

# ACL sync
acl.sync.enable=true
acl.sync.ms=30000

# Link performance tuning
link.mode=DESTINATION
cluster.link.metadata.topic.replication.factor=3
cluster.link.metadata.topic.min.insync.replicas=2
EOF
```

**Step 2: Create cluster link from destination cluster**

```bash
# Using Confluent CLI
confluent kafka link create migration-link \
  --cluster <target-cluster-id> \
  --source-cluster <source-cluster-id> \
  --config-file destination-link.config

# Verify link creation
confluent kafka link describe migration-link \
  --cluster <target-cluster-id>

# Expected output:
# Link Name: migration-link
# Link ID: <link-id>
# Source Cluster ID: <source-cluster-id>
# Destination Cluster ID: <target-cluster-id>
# State: ACTIVE
```

**Step 3: Monitor link health**

```bash
# Check link state
confluent kafka link describe migration-link \
  --cluster <target-cluster-id> \
  -o json | jq -r '.link_state'

# Should return: ACTIVE
```

### 4.2 Create Mirror Topics

**Approach: Mirror all topics or selective mirroring**

```bash
# List all topics to mirror
SOURCE_TOPICS=$(confluent kafka topic list --cluster <source-cluster-id> -o json | jq -r '.[].name' | grep -v '^_')

# Create mirror topics with cluster link
for topic in $SOURCE_TOPICS; do
  echo "Creating mirror for topic: $topic"
  
  confluent kafka mirror create $topic \
    --link migration-link \
    --cluster <target-cluster-id>
    
  # Validate mirror creation
  sleep 2
  confluent kafka mirror describe $topic \
    --link migration-link \
    --cluster <target-cluster-id>
done
```

**Alternative: Using Kafka CLI for more control**

```bash
# destination-client.properties
cat > destination-client.properties << EOF
bootstrap.servers=<target-cluster-bootstrap>:9092
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<dest-api-key>' password='<dest-api-secret>';
EOF

# Create mirror topic with specific configs
kafka-cluster-links --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --create \
  --link migration-link \
  --topic orders \
  --config-file mirror-topic-config.properties
```

**mirror-topic-config.properties:**
```properties
# Override source configs if needed
replication.factor=3
min.insync.replicas=2

# Preserve source configs
auto.create.topics.enable=false
unclean.leader.election.enable=false
```

### 4.3 Sync Topic Configurations

**Topic configs are automatically synced by Cluster Link, but validate:**

```bash
#!/bin/bash
# validate-topic-configs.sh

echo "Topic,Source RF,Dest RF,Source Retention,Dest Retention,Config Match"

for topic in $SOURCE_TOPICS; do
  # Get source config
  src_config=$(confluent kafka topic describe $topic --cluster <source-cluster-id> -o json)
  src_rf=$(echo $src_config | jq -r '.replication_factor')
  src_retention=$(echo $src_config | jq -r '.configs[] | select(.name=="retention.ms") | .value')
  
  # Get destination config
  dest_config=$(confluent kafka topic describe $topic --cluster <target-cluster-id> -o json)
  dest_rf=$(echo $dest_config | jq -r '.replication_factor')
  dest_retention=$(echo $dest_config | jq -r '.configs[] | select(.name=="retention.ms") | .value')
  
  # Compare
  if [ "$src_retention" == "$dest_retention" ]; then
    match="✓"
  else
    match="✗"
  fi
  
  echo "$topic,$src_rf,$dest_rf,$src_retention,$dest_retention,$match"
done
```

### 4.4 Handle Replication Factor Differences

**Critical: RF increase from 1/2 → 3**

```bash
# For topics where source RF < destination RF
# Cluster Link automatically handles this, but validate:

kafka-topics --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --describe \
  --topic orders

# Expected output should show:
# ReplicationFactor: 3
# Configs: min.insync.replicas=2

# If RF is incorrect, ALTER the mirror topic (DANGER: stops replication temporarily)
kafka-configs --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --alter \
  --entity-type topics \
  --entity-name orders \
  --add-config min.insync.replicas=2
```

**Best Practice:** Cluster Link uses destination cluster's default RF (3 for multi-zone). Verify before cutover.

### 4.5 Enable Offset Sync

**Offset translation is critical for consumer migration.**

```bash
# Verify consumer offset sync is enabled (set during link creation)
confluent kafka link describe migration-link \
  --cluster <target-cluster-id> \
  -o json | jq -r '.configs[] | select(.name=="consumer.offset.sync.enable")'

# Should return: true

# Monitor offset sync lag
kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --list

# Describe each consumer group to see translated offsets
for group in $(kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> --command-config destination-client.properties --list); do
  echo "=== Consumer Group: $group ==="
  kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
    --command-config destination-client.properties \
    --describe \
    --group $group
done
```

**Offset Translation Validation Script:**

```bash
#!/bin/bash
# validate-offset-translation.sh

CONSUMER_GROUP="my-consumer-group"
TOPIC="orders"

# Get offsets from source
echo "Source Cluster Offsets:"
kafka-consumer-groups --bootstrap-server <source-cluster-bootstrap> \
  --command-config source-client.properties \
  --describe \
  --group $CONSUMER_GROUP | grep $TOPIC

# Get translated offsets from destination
echo "Destination Cluster Translated Offsets:"
kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --describe \
  --group $CONSUMER_GROUP | grep $TOPIC

# Compare (should be very close, accounting for replication lag)
```

### 4.6 Monitor Replication Lag

**Cluster Link lag monitoring is critical pre-cutover.**

```bash
# Monitor mirror topic lag
confluent kafka mirror describe orders \
  --link migration-link \
  --cluster <target-cluster-id> \
  -o json | jq '.[]'

# Key metrics:
# - mirror_lag_ms: Time lag between source and destination
# - mirror_lag_count: Number of messages behind

# Continuous monitoring script
#!/bin/bash
while true; do
  echo "=== $(date) ==="
  for topic in $SOURCE_TOPICS; do
    lag=$(confluent kafka mirror describe $topic \
      --link migration-link \
      --cluster <target-cluster-id> \
      -o json 2>/dev/null | jq -r '.[0].mirror_lag_count // "N/A"')
    
    echo "Topic: $topic, Lag: $lag messages"
  done
  echo ""
  sleep 30
done
```

**Using Confluent Cloud Metrics API:**

```bash
curl -X POST "https://api.telemetry.confluent.cloud/v2/metrics/cloud/query" \
  -H "Authorization: Basic $(echo -n '<api-key>:<api-secret>' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "aggregations": [{
      "metric": "io.confluent.kafka.server/cluster_link_mirror_topic_lag_count"
    }],
    "filter": {
      "op": "AND",
      "filters": [
        {"field": "resource.kafka.id", "op": "EQ", "value": "<target-cluster-id>"},
        {"field": "link_name", "op": "EQ", "value": "migration-link"}
      ]
    },
    "granularity": "PT1M",
    "intervals": ["2026-04-20T00:00:00Z/2026-04-20T23:59:59Z"]
  }' | jq .
```

**Cutover Readiness Criteria:**
- ✅ Mirror lag < 1000 messages for ALL topics
- ✅ Mirror lag stable for 30+ minutes
- ✅ No ERROR state in mirror status
- ✅ Consumer offset sync lag < 30 seconds

---

## 5. Application Migration

### 5.1 Producer Cutover Strategy

**Phase 1: Dual-Write Testing (Optional but Recommended)**

```java
// Java producer example with dual-write for testing
Properties sourceProps = new Properties();
sourceProps.put("bootstrap.servers", "<source-cluster-bootstrap>");
sourceProps.put("security.protocol", "SASL_SSL");
sourceProps.put("sasl.mechanism", "PLAIN");
sourceProps.put("sasl.jaas.config", "org.apache.kafka.common.security.plain.PlainLoginModule required username='<key>' password='<secret>';");
sourceProps.put("acks", "all");
sourceProps.put("enable.idempotence", "true");

Properties destProps = new Properties();
destProps.put("bootstrap.servers", "<target-cluster-bootstrap>");
// ... same configs

KafkaProducer<String, String> sourceProducer = new KafkaProducer<>(sourceProps);
KafkaProducer<String, String> destProducer = new KafkaProducer<>(destProps);

// Send to both (for validation only, not production approach)
ProducerRecord<String, String> record = new ProducerRecord<>("orders", key, value);
sourceProducer.send(record);  // Primary
destProducer.send(record);    // Test only - DO NOT USE IN PRODUCTION
```

**⚠️ WARNING:** Dual-write creates data duplication and ordering issues. Only use for testing. Production cutover should be a clean switch.

**Phase 2: Producer Cutover (Recommended Approach)**

```bash
# Cutover strategy: DNS/Config-based instant switch

# Option A: Update producer config via environment variable
# Deploy new config with target cluster bootstrap URL
export KAFKA_BOOTSTRAP_SERVERS=<target-cluster-bootstrap>:9092
export KAFKA_API_KEY=<new-api-key>
export KAFKA_API_SECRET=<new-api-secret>

# Restart producer application (zero downtime if using rolling restart)

# Option B: Feature flag / configuration management
# Use feature flag to switch at application level without restart
```

**Producer Configuration for Multi-Zone:**

```properties
# Optimized for multi-zone with acks=all
bootstrap.servers=<target-cluster-bootstrap>:9092
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<key>' password='<secret>';

# Durability (CRITICAL for multi-zone)
acks=all                              # Wait for all ISR replicas (min 2 zones)
enable.idempotence=true              # Prevent duplicates
max.in.flight.requests.per.connection=5

# Performance tuning for cross-zone latency
linger.ms=10                         # Batch messages for efficiency
batch.size=32768                     # Larger batches reduce requests
compression.type=snappy              # Reduce cross-zone bandwidth
request.timeout.ms=30000             # Account for cross-zone latency
delivery.timeout.ms=120000           # Total timeout including retries
```

**Phased Rollout Plan:**

```
1. Canary: Switch 5% of producers → monitor for 30 min
2. Incremental: Switch 25% → monitor for 15 min
3. Incremental: Switch 50% → monitor for 15 min
4. Full cutover: Switch remaining 50% → monitor for 1 hour
```

### 5.2 Consumer Migration with Offset Validation

**Pre-Migration: Validate Offset Translation**

```bash
# Before switching consumers, validate offsets are synced
CONSUMER_GROUPS=$(kafka-consumer-groups --bootstrap-server <source-cluster-bootstrap> \
  --command-config source-client.properties --list)

for group in $CONSUMER_GROUPS; do
  echo "=== Validating consumer group: $group ==="
  
  # Source offsets
  kafka-consumer-groups --bootstrap-server <source-cluster-bootstrap> \
    --command-config source-client.properties \
    --describe --group $group > source-offsets-$group.txt
  
  # Destination translated offsets
  kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
    --command-config destination-client.properties \
    --describe --group $group > dest-offsets-$group.txt
  
  # Compare (manual review or scripted diff)
  diff source-offsets-$group.txt dest-offsets-$group.txt
done
```

**Consumer Cutover Process:**

```bash
# Step 1: Stop consumers on source cluster (rolling restart)
# This prevents duplicate processing during cutover

# Step 2: Verify consumer group lag is stable
kafka-consumer-groups --bootstrap-server <source-cluster-bootstrap> \
  --command-config source-client.properties \
  --describe --group my-consumer-group

# Step 3: Update consumer configuration
cat > consumer-new.properties << EOF
bootstrap.servers=<target-cluster-bootstrap>:9092
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<key>' password='<secret>';
group.id=my-consumer-group
auto.offset.reset=earliest
enable.auto.commit=false
EOF

# Step 4: Start consumers on destination cluster
# Consumer will automatically use translated offsets from Cluster Link
```

**Consumer Configuration for Multi-Zone:**

```properties
# Consumer config
bootstrap.servers=<target-cluster-bootstrap>:9092
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='<key>' password='<secret>';

group.id=my-consumer-group
auto.offset.reset=earliest           # Fallback if no offset found
enable.auto.commit=false             # Manual commit for exactly-once
max.poll.records=500                 # Tune based on processing time
fetch.min.bytes=1024                 # Reduce cross-zone fetches
fetch.max.wait.ms=500
session.timeout.ms=45000             # Account for cross-zone latency
heartbeat.interval.ms=15000
```

**Offset Reset Strategy (if translation fails):**

```bash
# Option A: Reset to specific offset (if you tracked the cutover point)
kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --group my-consumer-group \
  --topic orders \
  --reset-offsets --to-offset 12345 \
  --execute

# Option B: Reset to timestamp (at cutover time)
kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --group my-consumer-group \
  --topic orders:0,orders:1,orders:2 \
  --reset-offsets --to-datetime 2026-04-20T14:30:00.000 \
  --execute

# Option C: Reset to earliest (last resort - reprocess all data)
kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --group my-consumer-group \
  --all-topics \
  --reset-offsets --to-earliest \
  --execute
```

### 5.3 Handling Exactly-Once / At-Least-Once Semantics

**Exactly-Once Semantics (EOS):**

```java
// Producer with transactional support
Properties props = new Properties();
props.put("bootstrap.servers", "<target-cluster-bootstrap>");
props.put("transactional.id", "orders-producer-1");  // Unique per producer instance
props.put("enable.idempotence", "true");
props.put("acks", "all");
props.put("max.in.flight.requests.per.connection", "5");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orders", key, value));
    producer.commitTransaction();
} catch (ProducerFencedException | OutOfOrderSequenceException | AuthorizationException e) {
    producer.close();
} catch (KafkaException e) {
    producer.abortTransaction();
}
```

**Consumer with Exactly-Once:**

```java
Properties props = new Properties();
props.put("bootstrap.servers", "<target-cluster-bootstrap>");
props.put("group.id", "orders-consumer-group");
props.put("enable.auto.commit", "false");
props.put("isolation.level", "read_committed");  // Only read committed transactions

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("orders"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }
    consumer.commitSync();  // Commit after processing
}
```

**Migration Considerations:**
- Transactional IDs remain the same across clusters
- Consumer isolation level must be `read_committed`
- Verify no in-flight transactions during cutover window

### 5.4 Connector Migration (Kafka Connect)

**Step 1: Inventory all connectors**

```bash
# List all connectors
confluent connect cluster list
CONNECT_CLUSTER_ID=<your-connect-cluster-id>

confluent connect list --cluster $CONNECT_CLUSTER_ID -o json > connectors-inventory.json
```

**Step 2: Pause connectors on source**

```bash
# Pause all connectors before cutover
for connector in $(jq -r '.[].name' connectors-inventory.json); do
  echo "Pausing connector: $connector"
  confluent connect pause $connector --cluster $CONNECT_CLUSTER_ID
  
  # Wait for tasks to stop
  sleep 5
  
  # Verify status
  confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID | grep State
done
```

**Step 3: Update connector configurations**

```bash
# For each connector, update bootstrap.servers and credentials
for connector in $(jq -r '.[].name' connectors-inventory.json); do
  # Get current config
  confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID -o json > connector-$connector.json
  
  # Update config (manual or scripted)
  jq '.config."kafka.bootstrap.servers" = "<target-cluster-bootstrap>"' connector-$connector.json > connector-$connector-new.json
  jq '.config."kafka.sasl.jaas.config" = "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"<new-key>\" password=\"<new-secret>\";"' connector-$connector-new.json > connector-$connector-final.json
  
  # Update connector
  confluent connect update $connector --cluster $CONNECT_CLUSTER_ID --config-file connector-$connector-final.json
done
```

**Step 4: Resume connectors on destination**

```bash
for connector in $(jq -r '.[].name' connectors-inventory.json); do
  echo "Resuming connector: $connector"
  confluent connect resume $connector --cluster $CONNECT_CLUSTER_ID
  
  # Monitor status
  watch -n 5 "confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID | grep State"
done
```

**Connector-Specific Considerations:**

| Connector Type | Migration Notes |
|----------------|-----------------|
| **Source Connectors** (e.g., JDBC, S3) | Update output topic cluster; consider offset/position tracking |
| **Sink Connectors** (e.g., Elasticsearch, JDBC) | Update input topic cluster; verify no duplicate writes |
| **Schema Registry** | Ensure Schema Registry URL remains same or update connector config |
| **Offset Storage** | Connectors store offsets in Kafka topics; Cluster Link does NOT sync connector offsets |

**Critical: Connector Offset Migration**

```bash
# Connector offsets are stored in internal topics and NOT migrated by Cluster Link
# Manual migration required:

# Step 1: Export connector offsets from source (complex - requires Kafka Connect API)
# Step 2: Import into destination cluster
# OR: Accept reprocessing from latest/earliest based on connector type
```

### 5.5 ksqlDB Migration Considerations

**ksqlDB migration is complex and typically requires rebuild:**

```bash
# Step 1: Export ksqlDB queries and schemas
ksql http://<source-ksqldb-endpoint> <<EOF
SHOW STREAMS;
SHOW TABLES;
SHOW QUERIES;
EOF

# Step 2: Create new ksqlDB cluster on destination Kafka cluster
confluent ksql cluster create ksqldb-prod \
  --cluster <target-cluster-id> \
  --csu 4

# Step 3: Recreate streams, tables, and queries
# This requires manual script execution of all DDL statements

# Step 4: Validate query state and output topics
```

**ksqlDB State Migration:**
- ksqlDB stores state in RocksDB and Kafka changelog topics
- Cluster Link does NOT migrate ksqlDB state topics automatically
- **Recommended:** Recreate queries and allow state rebuild (may take hours for large state stores)
- **Alternative:** Advanced state migration via topic import (high risk, not recommended)

---

## 6. Cutover Strategy

### 6.1 Step-by-Step Traffic Switch Plan

**T-minus 24 hours:**
- [ ] Final validation of mirror lag < 1000 messages
- [ ] Consumer offset sync validated
- [ ] ACLs and RBAC confirmed replicated
- [ ] Schema Registry validated
- [ ] Rollback plan reviewed with team
- [ ] Monitoring dashboards prepared

**T-minus 1 hour:**
- [ ] Alert monitoring team
- [ ] Verify mirror lag < 500 messages
- [ ] No ongoing deployments or maintenance

**T-minus 15 minutes:**
- [ ] Final mirror lag check: < 100 messages
- [ ] Producer canary deployment ready
- [ ] Consumer configuration files prepared

**T=0 (Cutover Start):**

```bash
# Minute 0: Pause Kafka Connect connectors
for connector in $(confluent connect list --cluster $CONNECT_CLUSTER_ID -o json | jq -r '.[].name'); do
  confluent connect pause $connector --cluster $CONNECT_CLUSTER_ID
done

# Minute 2: Deploy producer config change (canary - 5%)
# Rolling restart of 5% of producer instances with new bootstrap server
kubectl rollout restart deployment/orders-producer-canary

# Minute 5: Validate canary producer
# Check error rates, latency, message delivery

# Minute 8: Deploy remaining producers (phased rollout)
kubectl rollout restart deployment/orders-producer-main

# Minute 12: Validate producer migration complete
# All producers writing to destination cluster

# Minute 15: Stop consumers on source cluster (graceful shutdown)
kubectl scale deployment/orders-consumer --replicas=0

# Minute 17: Verify consumer group lag frozen on source
kafka-consumer-groups --bootstrap-server <source-cluster-bootstrap> \
  --command-config source-client.properties \
  --describe --group orders-consumer-group

# Minute 18: Start consumers on destination cluster
kubectl set env deployment/orders-consumer \
  KAFKA_BOOTSTRAP_SERVERS=<target-cluster-bootstrap> \
  KAFKA_API_KEY=<new-key> \
  KAFKA_API_SECRET=<new-secret>
kubectl scale deployment/orders-consumer --replicas=10

# Minute 22: Validate consumers processing from destination
kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --describe --group orders-consumer-group

# Minute 25: Resume Kafka Connect connectors with new config
for connector in $(jq -r '.[].name' connectors-inventory.json); do
  confluent connect resume $connector --cluster $CONNECT_CLUSTER_ID
done

# Minute 30: Validation phase - monitor for 30 minutes
# Check error rates, consumer lag, data integrity
```

**T+30 minutes: Post-Cutover Validation**
- [ ] All producers writing to destination
- [ ] All consumers reading from destination with correct offsets
- [ ] Kafka Connect connectors RUNNING
- [ ] No errors in application logs
- [ ] Consumer lag healthy
- [ ] End-to-end data flow validated

### 6.2 Handling In-Flight Messages

**In-flight message strategy:**

```bash
# Before stopping producers on source:
# 1. Drain in-flight messages (wait for producer buffers to flush)

# Java producer drain example:
producer.flush();  // Wait for all buffered messages to send
Thread.sleep(10000);  // Additional 10s buffer
producer.close(Duration.ofSeconds(30));  // Graceful close

# 2. Wait for mirror lag to reach zero
while true; do
  lag=$(confluent kafka mirror describe orders \
    --link migration-link \
    --cluster <target-cluster-id> \
    -o json | jq -r '.[0].mirror_lag_count')
  
  if [ "$lag" -eq "0" ]; then
    echo "Mirror lag is zero. Safe to proceed."
    break
  else
    echo "Mirror lag: $lag messages. Waiting..."
    sleep 5
  fi
done

# 3. Only after lag=0, start consumers on destination
# This ensures no message loss
```

### 6.3 Minimizing Downtime

**Target: < 5 minutes consumer downtime**

**Optimization techniques:**

1. **Pre-warm consumers:** Start consumer instances on destination in read-only mode before cutover
2. **Connection pooling:** Keep persistent connections to destination cluster before switch
3. **DNS cutover:** Use DNS CNAME for bootstrap server (instant switch)
4. **Blue-Green deployment:** Run consumers in parallel briefly (risk of duplicates, but zero downtime)

**DNS-based zero-downtime cutover:**

```bash
# Setup: Create DNS CNAME pointing to source cluster
kafka.internal.company.com → <source-cluster-bootstrap>

# Producers and consumers use: kafka.internal.company.com:9092

# At cutover: Update DNS CNAME to point to destination
kafka.internal.company.com → <target-cluster-bootstrap>

# Wait for DNS TTL (typically 60s)
# New connections go to destination; existing connections drain naturally
```

### 6.4 Validation Checklist

**Pre-Cutover Validation:**
- [ ] Mirror lag < 100 messages for all topics
- [ ] Consumer offset sync lag < 30 seconds
- [ ] All ACLs replicated and verified
- [ ] Schema Registry accessible from destination cluster
- [ ] API keys created and tested for all applications
- [ ] Network connectivity validated (VPC peering/PrivateLink)
- [ ] Monitoring dashboards updated with destination cluster
- [ ] Rollback plan tested and ready
- [ ] Team communication: migration window start time confirmed

**Post-Cutover Validation:**
- [ ] All producers successfully writing to destination cluster
  - Check: `confluent kafka topic describe <topic> --cluster <target-cluster-id>`
  - Verify: Partition offsets increasing
- [ ] All consumers successfully reading from destination cluster
  - Check: `kafka-consumer-groups --describe`
  - Verify: Consumer lag decreasing
- [ ] No duplicate message processing
  - Check: Application logs for duplicate IDs
  - Verify: Database idempotency checks
- [ ] Consumer offsets match expected values
  - Compare: Source cluster final offset vs destination cluster starting offset
- [ ] Kafka Connect connectors in RUNNING state
  - Check: `confluent connect describe <connector>`
  - Verify: Tasks all RUNNING, no errors
- [ ] ksqlDB queries running and producing output
  - Check: `SHOW QUERIES;` in ksqlDB CLI
- [ ] Schema Registry serving requests
  - Check: `curl https://<sr-url>/subjects`
- [ ] End-to-end latency within SLA
  - Check: Producer to consumer latency monitoring
  - Target: < 500ms p99
- [ ] Error rates within baseline
  - Check: Application error logs and metrics
  - Target: < 0.01% error rate
- [ ] No data loss
  - Validate: Message count source vs destination (within mirror lag window)
- [ ] Security: No authentication/authorization errors
  - Check: Application logs for auth failures
- [ ] Monitoring dashboards showing healthy metrics
  - CPU < 70%, Memory < 80%, Disk < 75%
  - No broker offline
- [ ] Cluster link still ACTIVE (for rollback capability)
  - Check: `confluent kafka link describe migration-link`

---

## 7. Validation & Testing

### 7.1 Data Integrity Validation

**Message Count Validation:**

```bash
#!/bin/bash
# validate-message-counts.sh

TOPICS=$(confluent kafka topic list --cluster <source-cluster-id> -o json | jq -r '.[].name' | grep -v '^_')

echo "Topic,Source Messages,Destination Messages,Diff,Status"

for topic in $TOPICS; do
  # Get partition count
  partitions=$(kafka-topics --bootstrap-server <source-cluster-bootstrap> \
    --command-config source-client.properties \
    --describe --topic $topic | grep "PartitionCount" | awk '{print $2}')
  
  # Sum offsets across all partitions for source
  src_total=0
  for i in $(seq 0 $((partitions-1))); do
    offset=$(kafka-run-class kafka.tools.GetOffsetShell \
      --broker-list <source-cluster-bootstrap> \
      --topic $topic:$i \
      --time -1 | awk -F: '{print $3}')
    src_total=$((src_total + offset))
  done
  
  # Sum offsets for destination
  dest_total=0
  for i in $(seq 0 $((partitions-1))); do
    offset=$(kafka-run-class kafka.tools.GetOffsetShell \
      --broker-list <target-cluster-bootstrap> \
      --topic $topic:$i \
      --time -1 | awk -F: '{print $3}')
    dest_total=$((dest_total + offset))
  done
  
  diff=$((src_total - dest_total))
  
  if [ $diff -eq 0 ]; then
    status="✓ MATCH"
  elif [ $diff -lt 100 ]; then
    status="⚠ MINOR LAG"
  else
    status="✗ MISMATCH"
  fi
  
  echo "$topic,$src_total,$dest_total,$diff,$status"
done
```

**Content Hash Validation (sample):**

```python
#!/usr/bin/env python3
# validate-content-hash.py

from kafka import KafkaConsumer
import hashlib

def calculate_topic_hash(bootstrap_servers, topic, sample_size=1000):
    consumer = KafkaConsumer(
        topic,
        bootstrap_servers=bootstrap_servers,
        auto_offset_reset='earliest',
        consumer_timeout_ms=10000,
        max_poll_records=sample_size
    )
    
    messages = []
    for msg in consumer:
        messages.append(msg.value)
        if len(messages) >= sample_size:
            break
    
    # Sort to ensure consistent ordering
    messages.sort()
    combined = b''.join(messages)
    return hashlib.sha256(combined).hexdigest()

# Validate
source_hash = calculate_topic_hash('<source-bootstrap>', 'orders')
dest_hash = calculate_topic_hash('<target-bootstrap>', 'orders')

print(f"Source hash: {source_hash}")
print(f"Dest hash:   {dest_hash}")
print(f"Match: {source_hash == dest_hash}")
```

### 7.2 Cross-Zone Replication Validation

**Verify replicas distributed across zones:**

```bash
# Check replica distribution
kafka-topics --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --describe --topic orders

# Expected output:
# Topic: orders  Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,2,3
#   Replica 1: Zone A (broker 1)
#   Replica 2: Zone B (broker 2)
#   Replica 3: Zone C (broker 3)

# Validate ISR (In-Sync Replicas) includes all 3 replicas
# If ISR < Replicas, investigate replication lag
```

**Zone failure simulation:**

```bash
# TEST ONLY - DO NOT RUN IN PRODUCTION WITHOUT APPROVAL

# Simulate zone failure by isolating brokers in one zone
# This requires network manipulation or broker shutdown

# Example: Stop broker in Zone A (controlled test)
# 1. Identify brokers in Zone A from Confluent Cloud Console
# 2. Contact Confluent Support to simulate zone failure (Enterprise feature)

# During simulation, verify:
# - Producers can still write (acks=all requires min.insync.replicas=2, survives 1 zone down)
# - Consumers can still read
# - Leader election happens within seconds
# - No data loss
```

### 7.3 Consumer Lag and Throughput

**Monitor consumer lag post-migration:**

```bash
# Real-time lag monitoring
watch -n 5 "kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --describe --group orders-consumer-group"

# Expected healthy lag: < 5000 messages under normal load

# Throughput validation
# Compare source vs destination throughput using Confluent Cloud metrics
```

**Throughput benchmark:**

```bash
# Producer throughput test
kafka-producer-perf-test \
  --topic perf-test \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=<target-cluster-bootstrap> \
    security.protocol=SASL_SSL \
    sasl.mechanism=PLAIN \
    sasl.jaas.config='org.apache.kafka.common.security.plain.PlainLoginModule required username="<key>" password="<secret>";' \
    acks=all \
    compression.type=snappy

# Consumer throughput test
kafka-consumer-perf-test \
  --broker-list <target-cluster-bootstrap> \
  --topic perf-test \
  --messages 1000000 \
  --consumer.config destination-client.properties

# Expected results:
# Producer: 50-100 MB/sec (depends on cluster size)
# Consumer: 100-200 MB/sec
# Compare with source cluster baseline
```

### 7.4 Failover Testing

**Controlled failover test plan:**

```bash
# Test 1: Single broker failure
# Via Confluent Cloud Console or API, restart one broker
# Expected: No impact, automatic leader election

# Test 2: Zone failure simulation (requires Confluent Support)
# Simulate entire zone outage
# Expected: 
#   - Producers with acks=all: continue writing (2 zones still available)
#   - Consumers: continue reading from new leaders
#   - Latency spike: +10-20ms during leader election
#   - Recovery time: < 30 seconds

# Test 3: Rolling restart of all brokers
for broker_id in {1..6}; do
  echo "Restarting broker $broker_id"
  # Use Confluent Cloud API or Console to restart
  # Wait for ISR to stabilize before next broker
  sleep 60
done
```

**Metrics to monitor during failover:**
- Producer error rate (should remain < 0.01%)
- Consumer rebalance count (expect 1-2 rebalances per zone failure)
- End-to-end latency (expect spike during election, recovery within 30s)
- Under-replicated partitions (should return to 0 after recovery)

### 7.5 Schema and Connector Validation

**Schema Registry validation:**

```bash
# Validate all schemas accessible
for subject in $(curl -u <sr-key>:<sr-secret> https://<sr-url>/subjects | jq -r '.[]'); do
  echo "Testing schema: $subject"
  curl -u <sr-key>:<sr-secret> https://<sr-url>/subjects/$subject/versions/latest
done

# Test schema evolution (create new version)
curl -X POST \
  -u <sr-key>:<sr-secret> \
  https://<sr-url>/subjects/orders-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"Order\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"timestamp\",\"type\":\"long\"}]}"
  }'

# Verify compatibility
curl -X POST \
  -u <sr-key>:<sr-secret> \
  https://<sr-url>/compatibility/subjects/orders-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema": "..."}'
```

**Connector validation:**

```bash
# Check all connectors RUNNING
for connector in $(confluent connect list --cluster $CONNECT_CLUSTER_ID -o json | jq -r '.[].name'); do
  status=$(confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID -o json | jq -r '.status.state')
  echo "Connector: $connector, Status: $status"
  
  if [ "$status" != "RUNNING" ]; then
    echo "ERROR: Connector $connector not running!"
    confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID
  fi
done

# Validate connector throughput (compare pre/post migration)
# Check connector task lag (for sink connectors)
```

---

## 8. Rollback Plan

### 8.1 Rollback Triggers

**When to rollback:**

| Trigger | Severity | Action |
|---------|----------|--------|
| Data loss detected (message count mismatch > 1%) | CRITICAL | Immediate rollback |
| Consumer lag increasing uncontrollably | HIGH | Rollback within 15 min |
| Producer error rate > 5% | HIGH | Rollback within 15 min |
| Authentication/authorization failures affecting > 10% of clients | HIGH | Rollback within 15 min |
| Cross-zone latency > 100ms p99 causing application timeouts | MEDIUM | Evaluate; rollback if SLA breached |
| Kafka Connect connectors failing to start | MEDIUM | Attempt fix; rollback if unresolved in 30 min |
| Schema Registry unavailable | CRITICAL | Immediate rollback |
| End-to-end latency > 2x baseline | MEDIUM | Evaluate; rollback if business impact |

**Decision authority:** Require sign-off from Migration Lead + Engineering Manager for rollback

### 8.2 Rollback Steps

**Assumption:** Cluster Link still active (DO NOT delete until post-migration validation complete)

```bash
# ROLLBACK PROCEDURE

# Step 1: STOP all producers on destination cluster (IMMEDIATE)
kubectl scale deployment/orders-producer --replicas=0

# Step 2: STOP all consumers on destination cluster
kubectl scale deployment/orders-consumer --replicas=0

# Step 3: Revert producer configuration to source cluster
kubectl set env deployment/orders-producer \
  KAFKA_BOOTSTRAP_SERVERS=<source-cluster-bootstrap> \
  KAFKA_API_KEY=<original-source-key> \
  KAFKA_API_SECRET=<original-source-secret>

# Step 4: Restart producers on source cluster
kubectl scale deployment/orders-producer --replicas=20

# Step 5: Verify producers writing to source
kafka-topics --bootstrap-server <source-cluster-bootstrap> \
  --command-config source-client.properties \
  --describe --topic orders
# Check offset increasing

# Step 6: Determine consumer restart point
# Option A: Resume from last committed offset on source (if offset tracking maintained)
# Option B: Reset to specific offset/timestamp

# Step 7: Restart consumers on source cluster
kubectl set env deployment/orders-consumer \
  KAFKA_BOOTSTRAP_SERVERS=<source-cluster-bootstrap> \
  KAFKA_API_KEY=<original-source-key> \
  KAFKA_API_SECRET=<original-source-secret>
kubectl scale deployment/orders-consumer --replicas=10

# Step 8: Resume Kafka Connect connectors on source cluster
for connector in $(jq -r '.[].name' connectors-inventory.json); do
  # Revert connector config to source cluster
  confluent connect update $connector --cluster $CONNECT_CLUSTER_ID --config-file connector-$connector-source.json
  confluent connect resume $connector --cluster $CONNECT_CLUSTER_ID
done

# Step 9: Monitor source cluster for stability
# - Producer success rate > 99.9%
# - Consumer lag decreasing
# - No errors in application logs

# Step 10: Post-rollback analysis
# - Investigate root cause of migration failure
# - Document lessons learned
# - Plan remediation before retry
```

**Rollback completion time:** Target < 15 minutes

### 8.3 Handling Data Divergence

**During rollback, destination cluster may have received some messages:**

```bash
# Identify divergence point
# Compare final offset on destination vs source at rollback time

# Example:
# Source cluster offset at rollback: 1,234,567
# Destination cluster offset at rollback: 1,234,890
# Divergence: 323 messages written to destination during failed migration

# Options to handle divergence:

# Option A: Ignore (if acceptable data loss of destination-only messages)
# - Messages written only to destination are lost
# - Acceptable for non-critical data or if producers can replay

# Option B: Manual replay
# - Export messages from destination cluster (partition 0 example):
kafka-console-consumer \
  --bootstrap-server <target-cluster-bootstrap> \
  --consumer.config destination-client.properties \
  --topic orders \
  --partition 0 \
  --offset 1234567 \
  --max-messages 323 > divergent-messages.txt

# - Replay to source cluster using producer
kafka-console-producer \
  --bootstrap-server <source-cluster-bootstrap> \
  --producer.config source-client.properties \
  --topic orders < divergent-messages.txt

# Option C: Application-level reconciliation
# - Use application logs to identify lost transactions
# - Re-trigger business processes for lost messages
```

**Prevention:** Always ensure Cluster Link lag is zero before cutover to minimize divergence risk

---

## 9. Post-Migration Tasks

### 9.1 Decommission Old Cluster

**Wait period:** 7-14 days after successful migration

**Pre-decommission checklist:**
- [ ] All applications confirmed running on destination cluster for 7+ days
- [ ] No rollback triggers activated
- [ ] Business validation complete (reconciliation, audits passed)
- [ ] Backup/archive strategy confirmed for source cluster data (if required)
- [ ] Finance approval for cluster deletion (cost savings confirmation)

**Decommission steps:**

```bash
# Step 1: Delete cluster link (prevents accidental data sync)
confluent kafka link delete migration-link --cluster <target-cluster-id>

# Step 2: Stop all remaining workloads on source cluster
# Verify no producers or consumers connected
kafka-broker-api-versions --bootstrap-server <source-cluster-bootstrap> \
  --command-config source-client.properties

# Step 3: Export audit logs (if compliance requirement)
# Download cluster logs from Confluent Cloud Console

# Step 4: Delete source cluster
confluent kafka cluster delete <source-cluster-id>

# IMPORTANT: Confirm deletion carefully - this is irreversible

# Step 5: Clean up associated resources
# - Delete API keys for source cluster
# - Remove VPC peering/PrivateLink for source cluster
# - Update DNS records (if any pointed to source)
# - Remove monitoring dashboards for source cluster
```

**Cost impact:** Expect 30-40% cost increase post-migration due to:
- Higher replication factor (RF=3 vs RF=1/2)
- Cross-zone data transfer costs
- Enterprise cluster premium

### 9.2 Performance Tuning for Multi-Zone

**Optimize producer batching:**

```properties
# Tune for cross-zone latency
linger.ms=10-50                      # Increase batching window
batch.size=32768-65536               # Larger batches (test optimal size)
compression.type=snappy              # Or lz4 for lower CPU
buffer.memory=67108864               # 64MB buffer for high throughput
max.in.flight.requests.per.connection=5

# Measure improvement:
# - Reduced requests/sec to cluster
# - Lower p99 latency despite cross-zone
# - Higher throughput (MB/s)
```

**Optimize consumer fetch:**

```properties
# Reduce cross-zone fetch frequency
fetch.min.bytes=50000                # Wait for more data before fetch
fetch.max.wait.ms=500                # Max wait time for fetch.min.bytes
max.partition.fetch.bytes=1048576    # 1MB per partition
```

**Partition rebalancing:**

```bash
# If partition distribution uneven across zones
# Use Confluent Auto Data Balancer (Enterprise feature)

# Enable via Confluent Cloud Console:
# Cluster Settings → Auto Data Balancer → Enable

# Or trigger manual rebalance during low-traffic window
confluent kafka cluster update <target-cluster-id> \
  --config auto.data.balancer.enable=true
```

**Monitor key metrics:**
- Produce latency p99: Target < 50ms
- Consumer lag: Target < 5000 messages
- Request rate: Lower is better (batching efficiency)
- Bandwidth utilization: < 70% of cluster capacity

### 9.3 Monitoring and Alerting Setup

**Confluent Cloud Metrics to monitor:**

```bash
# Key metrics for multi-zone cluster
# - cluster.active_connection_count
# - cluster.received_bytes (per zone)
# - cluster.sent_bytes (per zone)
# - partition.leader_election_rate (spike indicates instability)
# - partition.under_replicated_partition_count (should be 0)
# - consumer_lag (per consumer group)
# - produce.p99_latency
# - consume.p99_latency
```

**Alert rules (example using Prometheus/Grafana):**

```yaml
# Example alerting rules
groups:
  - name: kafka_multi_zone_alerts
    rules:
      - alert: UnderReplicatedPartitions
        expr: kafka_cluster_under_replicated_partition_count > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Under-replicated partitions detected"
          description: "{{ $value }} partitions are under-replicated"

      - alert: ConsumerLagHigh
        expr: kafka_consumer_group_lag > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Consumer lag exceeds threshold"
          description: "Consumer group {{ $labels.group }} lag: {{ $value }}"

      - alert: ProduceLatencyHigh
        expr: kafka_produce_p99_latency_ms > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Produce latency high"
          description: "P99 latency: {{ $value }}ms"

      - alert: CrossZoneReplicationLag
        expr: kafka_replica_max_lag > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Cross-zone replication lag detected"
```

**Dashboard setup:**

```bash
# Import Confluent Cloud metrics to Grafana (via Prometheus exporter)
# Or use Confluent Cloud built-in dashboards

# Key dashboard panels:
# 1. Cluster health: Broker status, partition distribution
# 2. Producer metrics: Throughput, latency, error rate
# 3. Consumer metrics: Lag, throughput, rebalance frequency
# 4. Topic metrics: Message rate, byte rate, retention
# 5. Zone health: Per-zone throughput, cross-zone traffic
# 6. Replication: ISR status, lag, under-replicated partitions
```

### 9.4 Cost Optimization

**Cost breakdown (multi-zone Enterprise):**

| Cost Component | Estimate | Optimization Strategy |
|----------------|----------|----------------------|
| Cluster compute (CKUs) | $1.50-3.00/CKU/hour | Right-size based on actual usage; use auto-scaling |
| Storage (per GB) | $0.10-0.25/GB/month | Optimize retention policies; use tiered storage |
| Cross-zone bandwidth | $0.01/GB | Reduce unnecessary replication; use compression |
| API requests | Minimal | Batch requests; optimize client configs |
| Schema Registry | $100-200/month | Shared across clusters |
| Kafka Connect | $0.50/connector/hour | Consolidate connectors; use managed connectors |

**Optimization actions:**

```bash
# 1. Review and optimize retention policies
for topic in $(confluent kafka topic list --cluster <target-cluster-id> -o json | jq -r '.[].name'); do
  # Reduce retention from 7 days to 3 days for non-critical topics
  confluent kafka topic update $topic \
    --cluster <target-cluster-id> \
    --config retention.ms=259200000  # 3 days
done

# 2. Enable compression on topics
confluent kafka topic update orders \
  --cluster <target-cluster-id> \
  --config compression.type=snappy

# 3. Use tiered storage (if available for your cluster type)
# Move old data to cheaper object storage (S3, GCS)
confluent kafka topic update orders \
  --cluster <target-cluster-id> \
  --config remote.storage.enable=true \
  --config local.retention.ms=86400000  # Keep 1 day local, rest in S3

# 4. Right-size cluster (after monitoring actual usage for 2 weeks)
# Scale down CKUs if utilization < 40%
confluent kafka cluster update <target-cluster-id> \
  --cku 4  # Reduce from 6 to 4 CKUs if underutilized
```

**Estimated monthly cost (example):**
- Single-zone Dedicated (4 CKUs): ~$2,000/month
- Multi-zone Enterprise (4 CKUs): ~$2,800/month
- Increase: ~40% (+$800/month)
- Benefit: 99.99% uptime SLA, zone-level HA, zero data loss

---

## 10. Common Pitfalls & Best Practices

### 10.1 Latency Increase in Multi-Zone

**Problem:** Cross-zone replication adds 3-7ms latency for `acks=all` producers

**Symptoms:**
- Producer timeout errors
- Application slowdowns
- Increased p99 latency

**Solutions:**

```properties
# Producer tuning
linger.ms=10                         # Accept slight delay for batching
batch.size=32768                     # Larger batches amortize latency
compression.type=snappy              # Reduce bytes sent cross-zone
request.timeout.ms=30000             # Increase timeout for cross-zone
delivery.timeout.ms=120000           # Total timeout including retries
max.in.flight.requests.per.connection=5  # Pipelining reduces latency impact

# Application-level
# - Use async send with callbacks instead of sync
# - Batch application logic before Kafka send
# - Monitor produce latency and set alerts
```

**Best practice:** Test producer performance in staging environment before migration

### 10.2 Incorrect Replication Factor

**Problem:** Topics created with RF=1 or RF=2 after migration to multi-zone

**Symptoms:**
- Unexpected data loss on zone failure
- Lower durability than expected

**Prevention:**

```bash
# Set cluster-level default RF
confluent kafka cluster update <target-cluster-id> \
  --config default.replication.factor=3 \
  --config min.insync.replicas=2

# Validate all topics have RF=3
kafka-topics --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --describe | grep "ReplicationFactor" | grep -v "ReplicationFactor: 3"

# Fix topics with incorrect RF (requires partition reassignment)
# For topic with RF < 3:
kafka-reassign-partitions --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --reassignment-json-file reassignment.json \
  --execute

# reassignment.json example:
# {
#   "version": 1,
#   "partitions": [
#     {"topic": "orders", "partition": 0, "replicas": [1,2,3]},
#     {"topic": "orders", "partition": 1, "replicas": [2,3,4]}
#   ]
# }
```

**Best practice:** Audit all topics post-migration for correct RF

### 10.3 Offset Sync Delays

**Problem:** Consumer offset sync lag causes consumers to reprocess messages

**Symptoms:**
- Consumers start from much earlier offsets than expected
- Duplicate message processing
- Consumer lag spikes after migration

**Solutions:**

```bash
# Monitor offset sync lag BEFORE cutover
# Ensure consumer.offset.sync.ms is frequent enough (default 30s)

# If offset sync lag > 60s:
# 1. Decrease sync interval
confluent kafka link update migration-link \
  --cluster <target-cluster-id> \
  --config consumer.offset.sync.ms=10000  # 10 seconds

# 2. Wait for lag to stabilize before cutover
while true; do
  echo "Checking offset sync lag..."
  # Compare source vs destination consumer group offsets
  # (manual comparison or scripted)
  sleep 10
done

# 3. If offset translation fails, manual reset
kafka-consumer-groups --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --group orders-consumer-group \
  --reset-offsets --to-offset <calculated-offset> \
  --topic orders \
  --execute
```

**Best practice:** Validate offset sync for ALL consumer groups before cutover, not just sampling

### 10.4 ACL Mismatches

**Problem:** ACLs not replicated correctly, causing authorization failures

**Symptoms:**
- Producers/consumers getting AUTHORIZATION_FAILED errors
- Kafka Connect connectors failing with auth errors
- Schema Registry access denied

**Prevention:**

```bash
# Enable ACL sync in cluster link
acl.sync.enable=true

# Validate ALL ACLs replicated
# Export source ACLs
kafka-acls --bootstrap-server <source-cluster-bootstrap> \
  --command-config source-client.properties \
  --list | sort > source-acls.txt

# Export destination ACLs
kafka-acls --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --list | sort > dest-acls.txt

# Compare
diff source-acls.txt dest-acls.txt

# If mismatches found, manually add missing ACLs
kafka-acls --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --add \
  --allow-principal User:sa-12345 \
  --operation Read \
  --topic orders
```

**Best practice:** Test all application service accounts against destination cluster before cutover

### 10.5 Under-Partitioned Topics

**Problem:** Topics with too few partitions become bottleneck in multi-zone

**Symptoms:**
- Consumer lag increasing despite adequate consumer instances
- Single partition handling disproportionate load
- Uneven broker utilization

**Detection:**

```bash
# Identify under-partitioned topics
kafka-topics --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --describe | awk '/PartitionCount/ {if ($4 < 12) print $2, $4}'

# Check partition size
kafka-log-dirs --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties \
  --describe --topic-list orders | grep size
```

**Solution (before migration):**

```bash
# Increase partition count on source cluster BEFORE migration
# Note: Cannot decrease partition count, only increase

kafka-topics --bootstrap-server <source-cluster-bootstrap> \
  --command-config source-client.properties \
  --alter \
  --topic orders \
  --partitions 24  # Increase from 12 to 24

# Cluster Link will mirror new partition count to destination

# Best practice partition count calculation:
# - Minimum: (Target throughput MB/s) / (Max throughput per partition MB/s)
# - Rule of thumb: 2-3 partitions per consumer instance for parallelism
# - Typical: 12-48 partitions for high-throughput topics
```

### 10.6 Network Configuration Issues

**Problem:** VPC peering or PrivateLink misconfigured causing connectivity failures

**Symptoms:**
- Producer/consumer timeouts
- Intermittent connection drops
- High latency

**Prevention checklist:**
- [ ] VPC peering route tables configured bidirectionally
- [ ] Security groups allow Kafka traffic (port 9092)
- [ ] DNS resolution working for bootstrap servers
- [ ] No IP overlap between VPCs
- [ ] PrivateLink endpoint services accepted and active

**Validation:**

```bash
# Test connectivity from application VPC
telnet <target-cluster-bootstrap> 9092

# Test DNS resolution
nslookup <target-cluster-bootstrap>

# Test full Kafka connectivity
kafka-broker-api-versions --bootstrap-server <target-cluster-bootstrap> \
  --command-config destination-client.properties
```

### 10.7 Mirror Topic Failover Confusion

**Problem:** Not understanding when to switch from source to mirror topic

**Clarification:**
- Mirror topics are READ-ONLY during migration
- Cluster Link continuously replicates from source → destination
- Cutover = redirect producers/consumers to destination cluster (same topic name)
- Mirror topics become regular topics after link deletion

**Best practice:**
- Keep cluster link active for 7+ days post-cutover (enables fast rollback)
- Monitor mirror lag until zero before link deletion
- Delete link only after decommission approval

### 10.8 Skipped Testing in Staging

**Problem:** Migrating production without testing in staging environment

**Consequences:**
- Unexpected issues during production cutover
- Prolonged downtime
- Data loss risk

**Best practice:**
1. Create staging source and destination clusters
2. Replicate production topic configurations
3. Run full migration rehearsal with synthetic data
4. Measure cutover time and rollback time
5. Identify issues and refine runbook
6. Conduct second rehearsal after fixes
7. Only then migrate production

**Staging migration checklist:**
- [ ] Test cluster link creation and mirror topics
- [ ] Test consumer offset sync and translation
- [ ] Test producer/consumer cutover
- [ ] Test rollback procedure
- [ ] Measure performance impact
- [ ] Validate monitoring and alerting
- [ ] Document lessons learned

---

## Appendix A: Command Reference

**Confluent CLI Commands:**

```bash
# Cluster operations
confluent kafka cluster list
confluent kafka cluster describe <cluster-id>
confluent kafka cluster create
confluent kafka cluster delete <cluster-id>

# Topic operations
confluent kafka topic list --cluster <cluster-id>
confluent kafka topic describe <topic> --cluster <cluster-id>
confluent kafka topic create <topic> --cluster <cluster-id> --partitions 12 --replication-factor 3
confluent kafka topic delete <topic> --cluster <cluster-id>

# Cluster link operations
confluent kafka link create <link-name> --cluster <dest-cluster-id> --source-cluster <source-cluster-id> --config-file link.config
confluent kafka link describe <link-name> --cluster <cluster-id>
confluent kafka link delete <link-name> --cluster <cluster-id>

# Mirror topic operations
confluent kafka mirror create <topic> --link <link-name> --cluster <cluster-id>
confluent kafka mirror describe <topic> --link <link-name> --cluster <cluster-id>
confluent kafka mirror promote <topic> --link <link-name> --cluster <cluster-id>

# Consumer group operations
confluent kafka consumer group list --cluster <cluster-id>
confluent kafka consumer group describe <group> --cluster <cluster-id>

# API key operations
confluent api-key create --service-account <sa-id> --resource <cluster-id>
confluent api-key list --resource <cluster-id>
confluent api-key delete <api-key-id>
```

**Apache Kafka CLI Commands:**

```bash
# Topic operations
kafka-topics --bootstrap-server <bootstrap> --command-config client.properties --list
kafka-topics --bootstrap-server <bootstrap> --command-config client.properties --describe --topic <topic>

# Consumer group operations
kafka-consumer-groups --bootstrap-server <bootstrap> --command-config client.properties --list
kafka-consumer-groups --bootstrap-server <bootstrap> --command-config client.properties --describe --group <group>
kafka-consumer-groups --bootstrap-server <bootstrap> --command-config client.properties --reset-offsets --group <group> --topic <topic> --to-earliest --execute

# ACL operations
kafka-acls --bootstrap-server <bootstrap> --command-config client.properties --list
kafka-acls --bootstrap-server <bootstrap> --command-config client.properties --add --allow-principal User:<principal> --operation Read --topic <topic>

# Performance testing
kafka-producer-perf-test --topic <topic> --num-records 1000000 --record-size 1024 --throughput -1 --producer-props bootstrap.servers=<bootstrap> ...
kafka-consumer-perf-test --broker-list <bootstrap> --topic <topic> --messages 1000000 --consumer.config client.properties
```

---

## Appendix B: Troubleshooting Guide

| Issue | Symptoms | Diagnosis | Resolution |
|-------|----------|-----------|------------|
| **Cluster Link not creating** | Error: "Failed to create link" | Check source cluster API key permissions | Grant DeveloperRead on source cluster |
| **Mirror lag increasing** | Lag > 10,000 messages and growing | Check network bandwidth, broker CPU | Increase cluster CKUs; optimize producer batching |
| **Consumer offset not syncing** | Offsets in destination = 0 | Check `consumer.offset.sync.enable=true` | Update link config and verify sync interval |
| **ACLs not replicated** | Auth errors on destination | Check `acl.sync.enable=true` | Enable ACL sync or manually replicate ACLs |
| **Producer timeout errors** | `org.apache.kafka.common.errors.TimeoutException` | Check `request.timeout.ms` too low for multi-zone | Increase to 30000ms; check network latency |
| **Consumer rebalancing constantly** | Frequent "Rebalance in progress" logs | Check `session.timeout.ms` too low | Increase to 45000ms for multi-zone |
| **Data loss detected** | Message count mismatch | Check `acks=all` and `min.insync.replicas=2` | Investigate; possible rollback needed |
| **High cross-zone bandwidth costs** | Cloud bill increase > 50% | Check compression disabled | Enable `compression.type=snappy` |
| **Under-replicated partitions** | ISR < RF persistently | Check broker health, network issues | Investigate broker logs; contact Confluent Support |
| **Schema Registry unavailable** | `Failed to connect to Schema Registry` | Check SR endpoint and API keys | Verify SR linked to new cluster; update client configs |

---

## Appendix C: Migration Timeline Template

**Example: 100 TB cluster, 500 topics, 50 consumer groups**

| Phase | Duration | Activities |
|-------|----------|------------|
| **Pre-Migration (Week 1-2)** | 2 weeks | - Assessment and inventory<br>- Architecture design<br>- Staging environment setup |
| **Staging Migration (Week 3)** | 1 week | - Full rehearsal in staging<br>- Performance testing<br>- Runbook refinement |
| **Target Cluster Provisioning (Week 4)** | 3 days | - Provision Enterprise cluster<br>- Network setup<br>- API keys and ACLs |
| **Cluster Link Setup (Week 4)** | 1 day | - Create cluster link<br>- Create mirror topics<br>- Initial sync |
| **Replication Sync (Week 4-5)** | 5-7 days | - Wait for mirror lag to reach near-zero<br>- Continuous monitoring |
| **Application Testing (Week 5)** | 2 days | - Producer/consumer testing against destination<br>- Connector validation |
| **Production Cutover (Week 6)** | 4 hours | - Execute cutover during low-traffic window<br>- Phased rollout<br>- Validation |
| **Monitoring Period (Week 6-8)** | 2 weeks | - 24/7 monitoring<br>- Performance tuning<br>- Issue resolution |
| **Decommission (Week 10)** | 1 day | - Delete cluster link<br>- Delete source cluster<br>- Cleanup |

**Total timeline:** 10 weeks from start to decommission

---

## Document Control

- **Version:** 1.0
- **Last Updated:** 2026-04-20
- **Owner:** Platform Engineering Team
- **Review Cycle:** Quarterly
- **Approvers:** Engineering Manager, VP Engineering, CISO

**Change Log:**
| Date | Version | Author | Changes |
|------|---------|--------|---------|
| 2026-04-20 | 1.0 | Migration Team | Initial runbook creation |

---

**END OF RUNBOOK**
