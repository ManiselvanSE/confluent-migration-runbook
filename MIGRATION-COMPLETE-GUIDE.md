# Complete Migration Guide: Dedicated to Enterprise
## Confluent Cloud Cluster Linking Migration

**Migration Type**: Single-AZ Dedicated → Multi-AZ Enterprise  
**Strategy**: Cluster Linking (Zero Downtime)  
**Target RPO**: 0 (No Data Loss)  
**Target RTO**: <5 minutes (Consumer Restart)

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Pre-Migration Assessment](#pre-migration-assessment)
3. [Migration Steps](#migration-steps)
4. [Post-Migration Validation](#post-migration-validation)
5. [Discussion Questions](#discussion-questions)
6. [Things to Know Before Migration](#things-to-know-before-migration)
7. [Things to Know After Migration](#things-to-know-after-migration)
8. [Rollback Procedure](#rollback-procedure)

---

## Prerequisites

### 1. Access Requirements

**Confluent Cloud Access**:
- [ ] Admin access to Confluent Cloud organization
- [ ] Permission to create Enterprise clusters
- [ ] Permission to create Cluster Links
- [ ] Permission to create API keys and service accounts

**Cloud Provider Access** (if using private networking):
- [ ] AWS/Azure/GCP admin access for VPC peering or PrivateLink
- [ ] Permission to modify security groups and route tables
- [ ] Access to create VPC endpoints

---

### 2. Tool Requirements

**Required Tools**:
```bash
# Confluent CLI (v3.0+)
confluent --version

# Kafka CLI tools (for validation)
kafka-topics --version

# jq (for JSON parsing)
jq --version

# (Optional) kubectl (if using Kubernetes)
kubectl version
```

**Installation**:
```bash
# Install Confluent CLI
curl -sL --http1.1 https://cnfl.io/cli | sh -s -- latest

# Add to PATH
export PATH=$(pwd)/bin:$PATH

# Login to Confluent Cloud
confluent login --save
```

---

### 3. Source Cluster Requirements

**Minimum Requirements**:
- [ ] Source cluster: Confluent Cloud **Dedicated** tier
- [ ] Kafka version: 2.8+ (check cluster details)
- [ ] Cluster status: PROVISIONED and healthy
- [ ] No pending upgrades or maintenance

**Verification**:
```bash
# Check source cluster details
confluent kafka cluster describe <source-cluster-id>

# Verify cluster is healthy
confluent kafka cluster list --output json | jq '.[] | select(.id=="<source-cluster-id>") | .status'
# Expected: "PROVISIONED"
```

---

### 4. Network Requirements

**Private Networking** (Recommended for Production):

**Option A: VPC Peering**
- [ ] Application VPC created and configured
- [ ] No CIDR overlap with Confluent Cloud VPC
- [ ] Route tables prepared for peering
- [ ] Security groups allow egress on port 9092

**Option B: PrivateLink** (Preferred)
- [ ] VPC endpoints supported in your region
- [ ] Private DNS enabled in VPC
- [ ] Security groups configured

**Validation**:
```bash
# Test network connectivity (after setup)
telnet <destination-bootstrap-server> 9092

# Or
nc -zv <destination-bootstrap-server> 9092
```

---

### 5. Capacity Planning

**Calculate Destination Cluster Capacity**:

| Metric | Source (Dedicated) | Destination (Enterprise) | Notes |
|--------|-------------------|--------------------------|-------|
| **Replication Factor** | 1-2 | 3 | Triples storage |
| **Storage** | X GB | 3X GB | Account for RF=3 |
| **CKUs** | N CKUs | N+ CKUs | Add 20% buffer |
| **Throughput** | Y MB/s | Y MB/s | Same or higher |

**CKU Sizing Formula**:
```
Destination CKUs = Source CKUs × 1.2 (20% buffer)
Minimum: 4 CKUs for Enterprise
```

**Storage Calculation**:
```
Destination Storage = Source Storage × 3 (RF) × 1.2 (growth buffer)
```

---

### 6. Security Preparation

**Service Accounts**:
- [ ] Create service account for Cluster Link
- [ ] Create service accounts for applications (producers/consumers)
- [ ] Grant appropriate ACLs

**API Key Planning**:
```bash
# List current API keys
confluent api-key list --resource <source-cluster-id>

# Plan to create equivalent keys on destination
# - Cluster Link service account
# - Producer service accounts
# - Consumer service accounts
# - Kafka Connect service accounts
```

---

### 7. Application Inventory

**Complete Application Inventory**:

**Producers**:
- [ ] List all producer applications
- [ ] Document producer configs (acks, compression, batching)
- [ ] Identify exactly-once producers (transactional.id)
- [ ] Note deployment method (Kubernetes, VMs, serverless)

**Consumers**:
- [ ] List all consumer groups
- [ ] Document consumer offsets and lag
- [ ] Identify consumers using exactly-once semantics
- [ ] Note auto.offset.reset settings

**Kafka Connect**:
- [ ] List all connectors (source and sink)
- [ ] Export connector configurations
- [ ] Note connector offsets (stored in internal topics)

**ksqlDB** (if applicable):
- [ ] List ksqlDB queries
- [ ] Document ksqlDB state stores
- [ ] Plan for query recreation

---

### 8. Pre-Migration Testing

**Staging Environment Validation**:
- [ ] Create staging Dedicated and Enterprise clusters
- [ ] Execute full migration in staging
- [ ] Time each phase (replication sync, cutover, validation)
- [ ] Test rollback procedure
- [ ] Document lessons learned

**Expected Staging Results**:
- Total migration time: ~7-10 days (replication sync)
- Cutover window: ~4 hours (actual execution ~30 min)
- Rollback time: <15 minutes

---

## Pre-Migration Assessment

### Step 1: Inventory Current State

#### 1.1 Topic Inventory

```bash
# Export all topic names
confluent kafka topic list --cluster <source-cluster-id> > topics-list.txt

# For each topic, export configuration
while read topic; do
  echo "Exporting config for: $topic"
  confluent kafka topic describe $topic --cluster <source-cluster-id> -o json \
    > "topic-config-${topic}.json"
done < topics-list.txt

# Create inventory spreadsheet
echo "Topic,Partitions,Replication Factor,Retention (ms),Size (GB)" > topic-inventory.csv

# Analyze topics (using Kafka CLI for detailed stats)
kafka-topics --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe | grep "PartitionCount\|ReplicationFactor" >> topic-inventory.csv
```

**Output**: `topic-inventory.csv` with complete topic details

---

#### 1.2 Consumer Group Inventory

```bash
# List all consumer groups
confluent kafka consumer group list --cluster <source-cluster-id> \
  -o json > consumer-groups.json

# For each group, capture lag
for group in $(jq -r '.[].consumer_group_id' consumer-groups.json); do
  echo "=== Consumer Group: $group ===" >> consumer-group-lag.txt
  confluent kafka consumer group describe $group \
    --cluster <source-cluster-id> >> consumer-group-lag.txt
  echo "" >> consumer-group-lag.txt
done
```

**Key Metrics to Capture**:
- Total consumer groups: ______
- Groups with lag >10K messages: ______
- Groups with exactly-once semantics: ______

---

#### 1.3 Throughput Analysis

```bash
# Get cluster metrics (last 7 days)
# Via Confluent Cloud Console → Cluster → Metrics

# Or via Metrics API
curl -X POST "https://api.telemetry.confluent.cloud/v2/metrics/cloud/query" \
  -H "Authorization: Basic $(echo -n '<cloud-api-key>:<cloud-api-secret>' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "aggregations": [
      {"metric": "io.confluent.kafka.server/received_bytes"},
      {"metric": "io.confluent.kafka.server/sent_bytes"}
    ],
    "filter": {
      "field": "resource.kafka.id",
      "op": "EQ",
      "value": "<source-cluster-id>"
    },
    "granularity": "PT1H",
    "intervals": ["2024-01-01T00:00:00Z/2024-01-08T00:00:00Z"]
  }' | jq . > cluster-metrics.json
```

**Key Metrics**:
- Peak ingress throughput: ______ MB/s
- Peak egress throughput: ______ MB/s
- Average message rate: ______ msgs/s

---

#### 1.4 ACL Export

```bash
# Export all ACLs
kafka-acls --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --list > acls-export.txt

# Count ACLs
echo "Total ACLs: $(grep -c "User:" acls-export.txt)"
```

---

#### 1.5 Schema Registry Inventory

```bash
# List all schemas
curl -u <sr-api-key>:<sr-api-secret> \
  https://<schema-registry-url>/subjects | jq . > schemas-list.json

# Get compatibility mode
curl -u <sr-api-key>:<sr-api-secret> \
  https://<schema-registry-url>/config > schema-compatibility.json

echo "Total schemas: $(jq '. | length' schemas-list.json)"
```

---

#### 1.6 Kafka Connect Inventory

```bash
# List connectors
confluent connect cluster list

# For each connector cluster
CONNECT_CLUSTER_ID="<your-connect-cluster-id>"

confluent connect list --cluster $CONNECT_CLUSTER_ID -o json \
  > connectors-list.json

# Export each connector config
for connector in $(jq -r '.[].name' connectors-list.json); do
  confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID -o json \
    > "connector-${connector}.json"
done

echo "Total connectors: $(jq '. | length' connectors-list.json)"
```

---

### Step 2: Risk Assessment

**Create Risk Matrix**:

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Consumer offset translation failure | Medium | High | Validate ALL groups before cutover |
| Network connectivity issue during sync | Low | High | Pre-test VPC peering/PrivateLink |
| ACL mismatch post-migration | Low | High | Enable acl.sync.enable=true |
| Application config error | Medium | Critical | Canary rollout (5% → 100%) |
| Cluster Link lag during peak hours | Medium | Medium | Schedule cutover during low traffic |
| Cross-zone latency degrades SLA | Medium | Medium | Test in staging, tune producer batching |
| Schema Registry compatibility break | Low | High | Freeze schema changes during cutover |

**Risk Mitigation Plan**:
- [ ] Tested in staging environment
- [ ] Rollback plan documented and tested
- [ ] Monitoring dashboards prepared
- [ ] Team training completed
- [ ] Runback plan reviewed

---

### Step 3: Cutover Window Planning

**Recommended Cutover Window**:
- **Day of Week**: Tuesday or Wednesday (avoid Friday/Monday)
- **Time**: Low-traffic period (e.g., 2 AM - 6 AM local time)
- **Duration**: 4-hour window
- **Team Availability**: Full team on bridge for 6 hours (before/during/after)

**Cutover Timeline**:
```
T-1 week:   Final staging rehearsal
T-1 day:    Final validation (lag <1000 messages)
T-1 hour:   Team assembly, final go/no-go
T=0:        Cutover begins
T+30 min:   Cutover complete
T+4 hours:  Intensive monitoring
T+24 hours: Continued monitoring
```

---

## Migration Steps

### Phase 1: Provision Destination Cluster (Week 1)

#### Step 1.1: Create Enterprise Cluster

**Via Confluent Cloud Console**:
1. Navigate to: **Environments** → Select environment → **Add Cluster**
2. Select: **Enterprise**
3. Configure:
   - **Cloud Provider**: AWS (or Azure/GCP - match source)
   - **Region**: us-east-1 (match source region)
   - **Availability**: **Multi-Zone**
   - **Cluster Name**: `prod-enterprise-multizone`
4. Click **Launch Cluster**

**Via Confluent CLI**:
```bash
# Create Enterprise cluster
confluent kafka cluster create prod-enterprise-multizone \
  --cloud aws \
  --region us-east-1 \
  --type enterprise \
  --availability multi-zone

# Capture cluster ID
DEST_CLUSTER_ID=$(confluent kafka cluster list -o json \
  | jq -r '.[] | select(.name=="prod-enterprise-multizone") | .id')

echo "Destination Cluster ID: $DEST_CLUSTER_ID"
```

**Expected Time**: 15-30 minutes for provisioning

**Validation**:
```bash
# Check cluster status
confluent kafka cluster describe $DEST_CLUSTER_ID -o json | jq '.status'
# Expected: "PROVISIONED"

# Verify multi-zone
confluent kafka cluster describe $DEST_CLUSTER_ID -o json | jq '.availability'
# Expected: "MULTI_ZONE"

# Get bootstrap server
DEST_BOOTSTRAP=$(confluent kafka cluster describe $DEST_CLUSTER_ID -o json \
  | jq -r '.endpoint' | sed 's/SASL_SSL:\/\///')

echo "Destination Bootstrap: $DEST_BOOTSTRAP"
```

---

#### Step 1.2: Configure Networking

**Option A: VPC Peering (AWS Example)**

**Step 1**: Create Peering Request
```bash
# Via Confluent Cloud Console
# Navigate to: Cluster → Networking → Peering → Add Peering

# Record peering connection ID
PEERING_ID="pcx-xxxxxxxxxxxxx"
```

**Step 2**: Accept Peering in AWS
```bash
# Accept peering connection
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-east-1

# Verify peering active
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEERING_ID \
  --query 'VpcPeeringConnections[0].Status.Code'
# Expected: "active"
```

**Step 3**: Update Route Tables
```bash
# Get Confluent Cloud CIDR
CONFLUENT_CIDR=$(confluent network describe <network-id> -o json | jq -r '.cidr')

# Add route to Confluent CIDR
aws ec2 create-route \
  --route-table-id rtb-<your-rtb-id> \
  --destination-cidr-block $CONFLUENT_CIDR \
  --vpc-peering-connection-id $PEERING_ID
```

**Step 4**: Update Security Groups
```bash
# Allow egress to Confluent on port 9092
aws ec2 authorize-security-group-egress \
  --group-id sg-<your-app-sg> \
  --protocol tcp \
  --port 9092 \
  --cidr $CONFLUENT_CIDR
```

---

**Option B: PrivateLink (Preferred for Enterprise)**

```bash
# Create PrivateLink access
confluent network private-link-access create \
  --cloud aws \
  --region us-east-1 \
  --network <network-id>

# Get endpoint service name
ENDPOINT_SERVICE=$(confluent network private-link-access list -o json \
  | jq -r '.[0].aws.vpc_endpoint_service_name')

# Create VPC endpoint in your AWS account
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-<your-vpc-id> \
  --service-name $ENDPOINT_SERVICE \
  --subnet-ids subnet-<id1> subnet-<id2> subnet-<id3> \
  --security-group-ids sg-<your-sg> \
  --vpc-endpoint-type Interface \
  --private-dns-enabled
```

**Validation**:
```bash
# Test connectivity from application server
telnet $DEST_BOOTSTRAP 9092
# Expected: Connected

# Test DNS resolution
nslookup $DEST_BOOTSTRAP
# Expected: Private IP address
```

---

#### Step 1.3: Create Service Accounts and API Keys

**Create Service Account for Cluster Link**:
```bash
# Create service account
confluent iam service-account create cluster-link-sa \
  --description "Service account for Cluster Linking"

# Get service account ID
SA_LINK=$(confluent iam service-account list -o json \
  | jq -r '.[] | select(.name=="cluster-link-sa") | .id')

echo "Cluster Link SA: $SA_LINK"

# Create API key on SOURCE cluster
confluent api-key create \
  --service-account $SA_LINK \
  --resource <source-cluster-id> \
  --description "Cluster link source key"

# Save output (example):
# API Key: ABC123XYZ
# API Secret: xyz789abc456def...

LINK_SOURCE_KEY="ABC123XYZ"
LINK_SOURCE_SECRET="xyz789abc456def..."

# Store in secrets manager
echo "Storing in AWS Secrets Manager..."
aws secretsmanager create-secret \
  --name confluent/cluster-link/source \
  --secret-string "{\"api_key\":\"$LINK_SOURCE_KEY\",\"api_secret\":\"$LINK_SOURCE_SECRET\"}"

# Create API key on DESTINATION cluster
confluent api-key create \
  --service-account $SA_LINK \
  --resource $DEST_CLUSTER_ID \
  --description "Cluster link destination key"

LINK_DEST_KEY="DEF456UVW"
LINK_DEST_SECRET="uvw123def789ghi..."

# Store in secrets manager
aws secretsmanager create-secret \
  --name confluent/cluster-link/destination \
  --secret-string "{\"api_key\":\"$LINK_DEST_KEY\",\"api_secret\":\"$LINK_DEST_SECRET\"}"
```

**Create Application Service Accounts**:
```bash
# For each application, create service account and API keys
# Example: Producer application

confluent iam service-account create producer-orders-sa \
  --description "Orders producer service account"

SA_PRODUCER=$(confluent iam service-account list -o json \
  | jq -r '.[] | select(.name=="producer-orders-sa") | .id')

# Create API key on destination
confluent api-key create \
  --service-account $SA_PRODUCER \
  --resource $DEST_CLUSTER_ID \
  --description "Orders producer destination key"

# Save keys securely
```

---

#### Step 1.4: Configure ACLs

**Option A: Automatic ACL Sync (Recommended)**
- ACLs will sync automatically via Cluster Link (configured in Phase 2)
- Set `acl.sync.enable=true` in Cluster Link config

**Option B: Manual ACL Migration**
```bash
# Parse exported ACLs and recreate on destination

# Example: Grant producer permissions
kafka-acls --bootstrap-server $DEST_BOOTSTRAP:9092 \
  --command-config dest-client.properties \
  --add \
  --allow-principal User:$SA_PRODUCER \
  --operation WRITE \
  --operation DESCRIBE \
  --topic orders

# Example: Grant consumer permissions
kafka-acls --bootstrap-server $DEST_BOOTSTRAP:9092 \
  --command-config dest-client.properties \
  --add \
  --allow-principal User:$SA_CONSUMER \
  --operation READ \
  --topic orders \
  --group analytics-group
```

---

### Phase 2: Cluster Linking Setup (Week 2)

#### Step 2.1: Create Cluster Link Configuration

```bash
# Create link configuration file
cat > cluster-link.properties << EOF
# Source cluster connection
bootstrap.servers=<source-bootstrap>:9092
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username='${LINK_SOURCE_KEY}' password='${LINK_SOURCE_SECRET}';

# Consumer offset sync (CRITICAL)
consumer.offset.sync.enable=true
consumer.offset.sync.ms=30000

# ACL sync
acl.sync.enable=true
acl.sync.ms=30000

# Link mode
link.mode=DESTINATION

# Metadata topic settings
cluster.link.metadata.topic.replication.factor=3
cluster.link.metadata.topic.min.insync.replicas=2
EOF

echo "✅ Cluster Link config created"
```

---

#### Step 2.2: Create Cluster Link

```bash
# Create cluster link from destination
confluent kafka link create migration-link \
  --cluster $DEST_CLUSTER_ID \
  --source-cluster <source-cluster-id> \
  --config-file cluster-link.properties

echo "⏳ Cluster Link creation initiated..."

# Wait for link to become active
sleep 10

# Validate link status
confluent kafka link describe migration-link \
  --cluster $DEST_CLUSTER_ID

# Expected output:
# Link Name: migration-link
# Link ID: <link-id>
# Source Cluster ID: <source-cluster-id>
# Destination Cluster ID: <dest-cluster-id>
# State: ACTIVE ✅
```

**Troubleshooting**:
```bash
# If link state is FAILED
confluent kafka link describe migration-link --cluster $DEST_CLUSTER_ID

# Common issues:
# 1. Authentication failed → Verify API key has DeveloperRead on source
# 2. Network connectivity → Verify VPC peering/PrivateLink
# 3. Cluster not found → Verify source cluster ID
```

---

#### Step 2.3: Create Mirror Topics

**List Source Topics**:
```bash
# Get all topics (exclude internal topics)
SOURCE_TOPICS=$(confluent kafka topic list --cluster <source-cluster-id> -o json \
  | jq -r '.[].name' | grep -v '^_')

echo "Topics to mirror:"
echo "$SOURCE_TOPICS" | head -10
echo "..."
echo "Total: $(echo "$SOURCE_TOPICS" | wc -l) topics"
```

**Create Mirrors**:
```bash
# Create mirror topics (one at a time for monitoring)
for topic in $SOURCE_TOPICS; do
  echo "Creating mirror for: $topic"
  
  confluent kafka mirror create $topic \
    --link migration-link \
    --cluster $DEST_CLUSTER_ID
  
  # Brief pause to avoid rate limiting
  sleep 2
done

echo "✅ Mirror topic creation complete"
```

**Validation**:
```bash
# List all mirror topics
confluent kafka mirror list \
  --link migration-link \
  --cluster $DEST_CLUSTER_ID

# Check specific topic
confluent kafka mirror describe orders \
  --link migration-link \
  --cluster $DEST_CLUSTER_ID

# Expected:
# Source Topic: orders
# Mirror Topic: orders
# State: ACTIVE
# Mirror Lag: (decreasing number)
```

---

#### Step 2.4: Monitor Replication Lag

**Create Monitoring Script**:
```bash
cat > monitor-mirror-lag.sh << 'SCRIPT'
#!/bin/bash

DEST_CLUSTER_ID="<dest-cluster-id>"
LINK_NAME="migration-link"

echo "=== Cluster Link Replication Monitor ==="
echo "Press Ctrl+C to exit"
echo ""

while true; do
  clear
  echo "=== $(date) ==="
  echo ""
  
  # Get lag for all mirror topics
  confluent kafka mirror list --link $LINK_NAME --cluster $DEST_CLUSTER_ID -o json \
    | jq -r '.[] | "\(.source_topic_name): \(.num_partition_lag // "N/A") messages"' \
    | column -t
  
  echo ""
  echo "Refreshing in 60 seconds..."
  sleep 60
done
SCRIPT

chmod +x monitor-mirror-lag.sh

# Run monitoring
./monitor-mirror-lag.sh
```

**Expected Timeline**:
- Initial lag: High (millions of messages for large clusters)
- Day 1: Lag decreases significantly (50-70% reduction)
- Day 3: Lag <1M messages
- Day 5-7: Lag <1000 messages (cutover ready)

---

### Phase 3: Validation Before Cutover (Week 2-3)

#### Step 3.1: Verify Replication Lag

**Cutover Readiness Criteria**:
```bash
# Check lag for all topics
confluent kafka mirror list --link migration-link --cluster $DEST_CLUSTER_ID -o json \
  | jq -r '.[] | select(.num_partition_lag > 1000) | "\(.source_topic_name): \(.num_partition_lag)"'

# Expected: Empty output (all topics <1000 messages lag)
```

**If lag is high**:
- Wait longer (replication still in progress)
- Check network bandwidth
- Verify destination cluster capacity

---

#### Step 3.2: Verify Consumer Offset Sync

```bash
# For each consumer group, verify offset translation
CONSUMER_GROUPS=$(kafka-consumer-groups --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties --list)

for group in $CONSUMER_GROUPS; do
  echo "=== Consumer Group: $group ==="
  
  # Source offsets
  echo "Source offsets:"
  kafka-consumer-groups --bootstrap-server <source-bootstrap>:9092 \
    --command-config source-client.properties \
    --describe --group $group | head -5
  
  # Destination translated offsets
  echo "Destination translated offsets:"
  kafka-consumer-groups --bootstrap-server $DEST_BOOTSTRAP:9092 \
    --command-config dest-client.properties \
    --describe --group $group | head -5
  
  echo ""
done

# Offsets should match (within replication lag window)
```

**Critical**: If offsets don't match or are missing, investigate before proceeding.

---

#### Step 3.3: Verify ACL Sync

```bash
# Export ACLs from source (already have in acls-export.txt)

# Export ACLs from destination
kafka-acls --bootstrap-server $DEST_BOOTSTRAP:9092 \
  --command-config dest-client.properties \
  --list > acls-destination.txt

# Compare
diff acls-export.txt acls-destination.txt

# Expected: Minimal differences (same ACLs present)
```

---

#### Step 3.4: Pre-Cutover Checklist

**Complete Checklist** (all must be ✅):

- [ ] Mirror lag <1000 messages for ALL topics
- [ ] Mirror lag stable for 30+ minutes (not increasing)
- [ ] Consumer offset sync verified for all consumer groups
- [ ] ACLs replicated correctly
- [ ] Network connectivity validated (telnet test)
- [ ] Application API keys created on destination
- [ ] Deployment configs updated in staging
- [ ] Rollback plan reviewed and ready
- [ ] Monitoring dashboards prepared
- [ ] Team trained on cutover procedure
- [ ] Cutover window scheduled and communicated
- [ ] Change management approval obtained

---

### Phase 4: Cutover Execution (Week 3 - Cutover Day)

**Pre-Cutover** (T-1 hour):

```bash
# Final validation
echo "=== Pre-Cutover Final Checks ==="

# 1. Mirror lag check
echo "1. Checking mirror lag..."
confluent kafka mirror list --link migration-link --cluster $DEST_CLUSTER_ID -o json \
  | jq -r '.[] | select(.num_partition_lag > 100) | "\(.source_topic_name): FAIL"'
# Expected: Empty (all topics <100 messages)

# 2. Cluster Link state
echo "2. Checking Cluster Link state..."
confluent kafka link describe migration-link --cluster $DEST_CLUSTER_ID | grep "State:"
# Expected: State: ACTIVE

# 3. Consumer offset sync lag
echo "3. Verifying offset sync..."
# (Manual review of consumer groups)

# 4. No ongoing deployments
echo "4. Verifying no deployments in progress..."
kubectl get deployments -n production | grep -i rolling
# Expected: No rolling updates

# GO/NO-GO Decision
echo ""
echo "=== GO/NO-GO Decision ==="
echo "All checks passed? (yes/no)"
read GO_DECISION

if [ "$GO_DECISION" != "yes" ]; then
  echo "❌ Aborting cutover. Investigate issues."
  exit 1
fi

echo "✅ Proceeding with cutover..."
```

---

#### Step 4.1: Pause Kafka Connect (T=0)

```bash
# List all connectors
CONNECT_CLUSTER_ID="<your-connect-cluster-id>"

CONNECTORS=$(confluent connect list --cluster $CONNECT_CLUSTER_ID -o json \
  | jq -r '.[].name')

# Pause each connector
for connector in $CONNECTORS; do
  echo "Pausing connector: $connector"
  confluent connect pause $connector --cluster $CONNECT_CLUSTER_ID
  sleep 2
done

# Validate all paused
echo ""
echo "=== Connector Status ==="
for connector in $CONNECTORS; do
  confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID | grep "State:"
done

# Expected: All show "State: PAUSED"
```

---

#### Step 4.2: Producer Migration (T+2 min)

**Canary Deployment (5%)**:
```bash
# Update Kubernetes ConfigMap with destination cluster config
kubectl create configmap producer-orders-config \
  --from-literal=KAFKA_BOOTSTRAP_SERVERS=$DEST_BOOTSTRAP:9092 \
  --from-literal=KAFKA_API_KEY=$PRODUCER_KEY \
  --from-literal=KAFKA_API_SECRET=$PRODUCER_SECRET \
  --dry-run=client -o yaml | kubectl apply -f -

# Rolling restart canary deployment (5% of pods)
kubectl rollout restart deployment/producer-orders-canary -n production

# Wait for rollout
kubectl rollout status deployment/producer-orders-canary -n production

echo "✅ Canary deployment complete"
```

**Validation (T+5 min)**:
```bash
# Check messages appearing on destination
kafka-console-consumer --bootstrap-server $DEST_BOOTSTRAP:9092 \
  --consumer.config dest-client.properties \
  --topic orders \
  --max-messages 10 \
  --timeout-ms 10000

# Expected: Messages from canary producers

# Check producer metrics (error rate should be <1%)
# Via Confluent Cloud Console or metrics API
```

**Decision Point**: If canary error rate >1%, STOP and investigate. Else, proceed.

**Full Producer Rollout (T+8 min)**:
```bash
# Phase 1: 25%
echo "Deploying to 25% of producers..."
kubectl set env deployment/producer-orders -n production \
  KAFKA_BOOTSTRAP_SERVERS=$DEST_BOOTSTRAP:9092 \
  KAFKA_API_KEY=$PRODUCER_KEY \
  KAFKA_API_SECRET=$PRODUCER_SECRET

kubectl scale deployment/producer-orders -n production --replicas=5  # 25% of 20
kubectl rollout status deployment/producer-orders -n production

# Monitor for 5 minutes
echo "Monitoring for 5 minutes..."
sleep 300

# Phase 2: 50%
echo "Deploying to 50% of producers..."
kubectl scale deployment/producer-orders -n production --replicas=10
kubectl rollout status deployment/producer-orders -n production

# Monitor for 3 minutes
sleep 180

# Phase 3: 100%
echo "Deploying to 100% of producers..."
kubectl scale deployment/producer-orders -n production --replicas=20
kubectl rollout status deployment/producer-orders -n production

echo "✅ All producers switched to destination cluster"
```

---

#### Step 4.3: Consumer Migration (T+15 min)

**Stop Consumers on Source**:
```bash
# Gracefully scale down consumers
echo "Stopping consumers on source cluster..."
kubectl scale deployment/consumer-analytics -n production --replicas=0

# Wait for graceful shutdown (consumers commit final offsets)
kubectl wait --for=delete pod -l app=consumer-analytics -n production --timeout=120s

echo "✅ Consumers stopped on source"
```

**Capture Final Source Offsets**:
```bash
# Capture final committed offsets
kafka-consumer-groups --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe --group analytics-group > final-source-offsets.txt

echo "✅ Final source offsets captured"
```

**Start Consumers on Destination** (T+18 min):
```bash
# Update consumer config
kubectl set env deployment/consumer-analytics -n production \
  KAFKA_BOOTSTRAP_SERVERS=$DEST_BOOTSTRAP:9092 \
  KAFKA_API_KEY=$CONSUMER_KEY \
  KAFKA_API_SECRET=$CONSUMER_SECRET

# Scale up consumers
kubectl scale deployment/consumer-analytics -n production --replicas=10

# Wait for consumers to start
kubectl rollout status deployment/consumer-analytics -n production

echo "✅ Consumers started on destination"
```

**Validation**:
```bash
# Verify consumers joined and consuming
kafka-consumer-groups --bootstrap-server $DEST_BOOTSTRAP:9092 \
  --command-config dest-client.properties \
  --describe --group analytics-group

# Expected:
# - Consumers assigned to partitions
# - LAG decreasing
# - No errors

echo "✅ Consumers validated"
```

---

#### Step 4.4: Resume Kafka Connect (T+22 min)

**Update Connector Configurations**:
```bash
# For each connector, update config
for connector in $CONNECTORS; do
  echo "Updating connector: $connector"
  
  # Export current config
  confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID -o json \
    > "connector-${connector}-orig.json"
  
  # Update bootstrap.servers and API keys (use jq)
  jq --arg bootstrap "$DEST_BOOTSTRAP:9092" \
     --arg key "$CONNECTOR_KEY" \
     --arg secret "$CONNECTOR_SECRET" \
     '.config."kafka.bootstrap.servers" = $bootstrap |
      .config."kafka.sasl.jaas.config" = "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"\($key)\" password=\"\($secret)\";"' \
     "connector-${connector}-orig.json" > "connector-${connector}-updated.json"
  
  # Update connector
  confluent connect update $connector \
    --cluster $CONNECT_CLUSTER_ID \
    --config-file "connector-${connector}-updated.json"
  
  sleep 2
done

echo "✅ Connectors updated"
```

**Resume Connectors**:
```bash
# Resume all connectors
for connector in $CONNECTORS; do
  echo "Resuming connector: $connector"
  confluent connect resume $connector --cluster $CONNECT_CLUSTER_ID
  sleep 2
done

# Validate
echo ""
echo "=== Connector Status ==="
for connector in $CONNECTORS; do
  confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID | grep "State:"
done

# Expected: All show "State: RUNNING"

echo "✅ Kafka Connect migration complete"
```

---

#### Step 4.5: Cutover Complete (T+30 min)

**Final Validation**:
```bash
echo "=== Cutover Complete - Final Validation ==="

# 1. Producers on destination
echo "1. Verifying producers..."
# Check metrics show traffic on destination

# 2. Consumers on destination
echo "2. Verifying consumers..."
kafka-consumer-groups --bootstrap-server $DEST_BOOTSTRAP:9092 \
  --command-config dest-client.properties \
  --list | wc -l

# 3. Kafka Connect operational
echo "3. Verifying Kafka Connect..."
confluent connect list --cluster $CONNECT_CLUSTER_ID

# 4. No errors in application logs
echo "4. Checking application logs..."
kubectl logs -l app=producer-orders -n production --tail=20 | grep -i error
kubectl logs -l app=consumer-analytics -n production --tail=20 | grep -i error

echo ""
echo "✅ CUTOVER COMPLETE!"
echo "Begin post-migration monitoring period"
```

---

## Post-Migration Validation

### Step 1: Data Integrity Validation

#### 1.1 Message Count Comparison

```bash
#!/bin/bash
# validate-message-counts.sh

echo "Topic,Source Count,Dest Count,Difference,Status" > message-count-validation.csv

for topic in $SOURCE_TOPICS; do
  echo "Validating: $topic"
  
  # Get high water mark from source
  src_count=$(kafka-run-class kafka.tools.GetOffsetShell \
    --broker-list <source-bootstrap>:9092 \
    --topic $topic --time -1 \
    | awk -F: '{sum+=$3} END {print sum}')
  
  # Get high water mark from destination
  dest_count=$(kafka-run-class kafka.tools.GetOffsetShell \
    --broker-list $DEST_BOOTSTRAP:9092 \
    --topic $topic --time -1 \
    | awk -F: '{sum+=$3} END {print sum}')
  
  diff=$((src_count - dest_count))
  
  if [ $diff -eq 0 ]; then
    status="✅ MATCH"
  elif [ ${diff#-} -lt 100 ]; then
    status="⚠️ MINOR"
  else
    status="❌ MISMATCH"
  fi
  
  echo "$topic,$src_count,$dest_count,$diff,$status" >> message-count-validation.csv
done

# Summary
echo ""
echo "=== Validation Summary ==="
grep "❌" message-count-validation.csv && echo "ISSUES FOUND" || echo "✅ ALL TOPICS VALIDATED"
```

---

#### 1.2 Consumer Offset Validation

```bash
# For each consumer group, compare final source offset vs destination starting offset

for group in $CONSUMER_GROUPS; do
  echo "=== Validating Consumer Group: $group ==="
  
  # Get final source offsets (from final-source-offsets.txt)
  grep "$group" final-source-offsets.txt > source-final-$group.txt
  
  # Get destination starting offsets
  kafka-consumer-groups --bootstrap-server $DEST_BOOTSTRAP:9092 \
    --command-config dest-client.properties \
    --describe --group $group > dest-starting-$group.txt
  
  # Manual comparison (offsets should match)
  diff source-final-$group.txt dest-starting-$group.txt
done

echo "✅ Consumer offset validation complete"
```

---

### Step 2: Application Health Validation

#### 2.1 Producer Health

```bash
# Check producer metrics
echo "=== Producer Health Check ==="

# 1. Deployment status
kubectl get deployments -n production | grep producer

# 2. Pod status
kubectl get pods -n production -l app=producer-orders

# 3. Error logs
kubectl logs -l app=producer-orders -n production --tail=100 | grep -i error

# 4. Throughput metrics (via Confluent Cloud Console)
# Navigate to: Destination Cluster → Metrics → Throughput

echo "✅ Producer health validated"
```

---

#### 2.2 Consumer Health

```bash
echo "=== Consumer Health Check ==="

# 1. Consumer lag
for group in $CONSUMER_GROUPS; do
  echo "Group: $group"
  kafka-consumer-groups --bootstrap-server $DEST_BOOTSTRAP:9092 \
    --command-config dest-client.properties \
    --describe --group $group | grep "LAG"
done

# Expected: Lag decreasing or stable (not increasing)

# 2. Pod status
kubectl get pods -n production -l app=consumer-analytics

# 3. Error logs
kubectl logs -l app=consumer-analytics -n production --tail=100 | grep -i error

echo "✅ Consumer health validated"
```

---

#### 2.3 Kafka Connect Health

```bash
echo "=== Kafka Connect Health Check ==="

# Check all connector states
for connector in $CONNECTORS; do
  status=$(confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID -o json \
    | jq -r '.status.state')
  echo "$connector: $status"
done

# Expected: All RUNNING

# Check task status
for connector in $CONNECTORS; do
  confluent connect describe $connector --cluster $CONNECT_CLUSTER_ID | grep "Task"
done

echo "✅ Kafka Connect health validated"
```

---

### Step 3: Performance Validation

#### 3.1 Latency Check

```bash
# Producer latency (via Confluent Cloud metrics)
# Navigate to: Cluster → Metrics → Request Latency

# Expected:
# - Produce p99 latency: <50ms (acceptable for multi-zone)
# - Increase from single-zone: +3-7ms (expected)

echo "Latency validation:"
echo "- Single-zone baseline: 15ms p99"
echo "- Multi-zone current: 22ms p99"
echo "- Increase: 7ms (acceptable) ✅"
```

---

#### 3.2 Throughput Check

```bash
# Compare throughput: source vs destination

echo "=== Throughput Validation ==="

# Producer throughput test
kafka-producer-perf-test \
  --topic perf-test \
  --num-records 100000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=$DEST_BOOTSTRAP:9092 \
    security.protocol=SASL_SSL \
    sasl.mechanism=PLAIN \
    sasl.jaas.config="..." \
    acks=all \
    compression.type=snappy

# Expected: ≥ source cluster throughput

echo "✅ Throughput validated"
```

---

### Step 4: Schema Registry Validation

```bash
# Verify Schema Registry integration

# 1. List schemas (should match source)
curl -u $SR_KEY:$SR_SECRET \
  https://<schema-registry-url>/subjects | jq . > schemas-post-migration.json

# Compare with pre-migration
diff schemas-list.json schemas-post-migration.json

# Expected: No differences

# 2. Test schema serialization
# Produce test message with schema
kafka-avro-console-producer \
  --bootstrap-server $DEST_BOOTSTRAP:9092 \
  --producer.config dest-client.properties \
  --topic test-avro \
  --property schema.registry.url=https://<schema-registry-url> \
  --property basic.auth.credentials.source=USER_INFO \
  --property basic.auth.user.info=$SR_KEY:$SR_SECRET \
  --property value.schema='{"type":"record","name":"Test","fields":[{"name":"id","type":"string"}]}' \
  <<< '{"id":"test-123"}'

# 3. Consume and deserialize
kafka-avro-console-consumer \
  --bootstrap-server $DEST_BOOTSTRAP:9092 \
  --consumer.config dest-client.properties \
  --topic test-avro \
  --from-beginning \
  --max-messages 1 \
  --property schema.registry.url=https://<schema-registry-url> \
  --property basic.auth.credentials.source=USER_INFO \
  --property basic.auth.user.info=$SR_KEY:$SR_SECRET

# Expected: Message deserialized successfully

echo "✅ Schema Registry validated"
```

---

## Discussion Questions

### Pre-Migration Planning Questions

**Business Questions**:

1. **What is the business impact of downtime?**
   - Cost per hour of downtime: $_______
   - Critical business hours: _______
   - Acceptable downtime window: _______

2. **What is the data criticality?**
   - Can we afford to lose any messages? (Yes/No)
   - Are there regulatory requirements? (HIPAA, PCI-DSS, SOC 2)
   - What is the required RPO (Recovery Point Objective)? _______

3. **What is the timeline flexibility?**
   - Hard deadline for migration? _______
   - Can we delay for staging tests? (Yes/No)
   - Budget for migration execution? $_______

---

**Technical Questions**:

4. **What is the current cluster configuration?**
   - Number of topics: _______
   - Number of partitions (total): _______
   - Total data size: _______ GB
   - Peak throughput: _______ MB/s
   - Number of consumer groups: _______

5. **What applications are involved?**
   - Producer count: _______
   - Consumer count: _______
   - Kafka Connect connectors: _______
   - ksqlDB queries: _______
   - Deployment method: Kubernetes / VMs / Serverless / Other

6. **What is the network setup?**
   - Current connectivity: Public / Private (VPC Peering / PrivateLink)
   - Target connectivity: Public / Private (VPC Peering / PrivateLink)
   - Cloud provider: AWS / Azure / GCP
   - Region: _______

7. **What are the security requirements?**
   - Authentication method: API Keys / RBAC / mTLS
   - Number of service accounts: _______
   - Number of ACLs: _______
   - Encryption at rest required? (Yes/No)

---

**Operational Questions**:

8. **What is the team's experience level?**
   - Have we done Confluent migrations before? (Yes/No)
   - Team familiar with Cluster Linking? (Yes/No)
   - Access to Confluent Support? (Yes/No)
   - Need external consultants? (Yes/No)

9. **What is the testing strategy?**
   - Do we have a staging environment? (Yes/No)
   - Can we rehearse the migration? (Yes/No)
   - Time allocated for staging test: _______ days

10. **What is the monitoring setup?**
    - Current monitoring tools: Confluent Cloud Console / Datadog / Prometheus / Other
    - Alerting configured? (Yes/No)
    - On-call team available during cutover? (Yes/No)

---

### During Migration Questions

11. **Is replication lag acceptable?**
    - Current lag: _______ messages
    - Target lag for cutover: <1000 messages
    - Lag stable for 30+ minutes? (Yes/No)

12. **Are consumer offsets syncing correctly?**
    - Offset sync enabled? (Yes/No)
    - Offsets match between source and destination? (Yes/No)
    - Any consumer groups missing? _______

13. **Are applications ready for cutover?**
    - Deployment configs updated? (Yes/No)
    - API keys created on destination? (Yes/No)
    - Deployment tested in staging? (Yes/No)

---

### Post-Migration Questions

14. **Is the migration successful?**
    - All producers on destination? (Yes/No)
    - All consumers on destination? (Yes/No)
    - Kafka Connect operational? (Yes/No)
    - No data loss? (Yes/No)

15. **What is the performance impact?**
    - Latency increase: _______ ms (acceptable: <10ms)
    - Throughput maintained? (Yes/No)
    - Consumer lag healthy? (Yes/No)

16. **When can we decommission the source cluster?**
    - Days of stable operation: _______ (recommended: 7-14 days)
    - Business validation complete? (Yes/No)
    - Ready to delete Cluster Link? (Yes/No)

---

## Things to Know Before Migration

### 1. Cluster Linking Capabilities

**What Cluster Linking DOES**:
- ✅ Replicates topic data byte-for-byte
- ✅ Syncs consumer offsets automatically
- ✅ Syncs ACLs (if enabled)
- ✅ Syncs topic configurations
- ✅ Preserves message ordering within partitions
- ✅ Enables zero-downtime migration

**What Cluster Linking DOES NOT DO**:
- ❌ Sync Kafka Connect connector offsets (stored in internal topics)
- ❌ Sync transactional state (transactions are cluster-specific)
- ❌ Sync ksqlDB state (requires query recreation)
- ❌ Sync Schema Registry (it's regional, already shared)
- ❌ Automatically switch application configs (manual step)

---

### 2. Replication Factor Changes

**Critical Understanding**:

| Aspect | Source (Dedicated) | Destination (Enterprise) | Impact |
|--------|-------------------|--------------------------|--------|
| **Replication Factor** | 1 or 2 | 3 | 3x storage |
| **min.insync.replicas** | 1 | 2 | Higher durability |
| **Storage Cost** | X | 3X | Budget accordingly |
| **Latency** | Baseline | +3-7ms | Tune producers |

**Action Required**: Budget for 30-40% cost increase

---

### 3. Cross-Zone Latency

**Expected Latency Increase**:
- Single-zone produce latency: 10-15ms p99
- Multi-zone produce latency: 13-22ms p99
- **Increase**: +3-7ms

**Mitigation**:
```properties
# Producer tuning for multi-zone
linger.ms=10               # Increase batching
batch.size=32768          # Larger batches
compression.type=snappy   # Reduce bytes
request.timeout.ms=30000  # Account for cross-zone latency
```

---

### 4. Consumer Offset Translation

**How It Works**:
- Cluster Link automatically maps source offsets → destination offsets
- Consumers resume from exact position (no reprocessing)
- Sync interval: 30 seconds (configurable)

**Validation Critical**:
- **MUST validate** offset translation for ALL consumer groups before cutover
- Missing or incorrect offsets = massive reprocessing

---

### 5. Network Connectivity

**Private Connectivity Required** (Production):
- **VPC Peering**: Requires route table updates, no CIDR overlap
- **PrivateLink**: Preferred, no CIDR conflicts, simpler security

**Testing**:
```bash
# MUST test connectivity before migration
telnet <destination-bootstrap> 9092
```

---

### 6. ACL Sync Behavior

**Automatic ACL Sync**:
- Enabled via `acl.sync.enable=true` in Cluster Link
- Syncs every 30 seconds (configurable)
- **Important**: Test all application permissions after cutover

**Manual Verification Required**:
- Export source ACLs
- Export destination ACLs
- Diff and validate

---

### 7. Kafka Connect Limitations

**Connector Offset NOT Synced**:
- Connector offsets stored in `connect-offsets` topic
- This topic is **NOT** synced by Cluster Link
- **Result**: Connectors may reprocess data after migration

**Mitigation**:
- Accept reprocessing (idempotent connectors)
- Manual offset migration (complex, not recommended)
- Use connector checkpointing features if available

---

### 8. ksqlDB State

**ksqlDB State NOT Migrated**:
- ksqlDB stores state in RocksDB and changelog topics
- Cluster Link does not migrate ksqlDB-specific state
- **Approach**: Recreate queries on destination cluster

**Timeline**:
- State rebuild can take hours for large datasets
- Plan accordingly

---

### 9. Schema Registry

**No Migration Needed**:
- Schema Registry is **regional** (not cluster-specific)
- Same Schema Registry serves both source and destination
- **Validation**: Ensure destination can access SR endpoint

---

### 10. Rollback Capability

**Rollback Window**: 7-14 days (while Cluster Link active)

**Rollback Time**: <15 minutes

**Important**: Do NOT delete Cluster Link until decommission approval

---

## Things to Know After Migration

### 1. Monitoring Changes

**New Metrics to Monitor**:
- **Cross-zone replication lag**: Should be 0
- **Under-replicated partitions**: Should be 0
- **Producer p99 latency**: Expect +3-7ms increase
- **Consumer lag**: Should remain stable

**Alert Thresholds**:
```yaml
# Update alerting rules
alerts:
  - name: UnderReplicatedPartitions
    threshold: 0
    duration: 5m
    severity: critical
  
  - name: ConsumerLagHigh
    threshold: 10000
    duration: 10m
    severity: warning
  
  - name: ProduceLatencyHigh
    threshold: 100ms  # Adjusted for multi-zone
    duration: 5m
    severity: warning
```

---

### 2. Performance Tuning

**Producer Optimization**:
```properties
# Fine-tune for multi-zone
linger.ms=10-50           # Increase based on testing
batch.size=32768-65536    # Larger batches
compression.type=snappy   # Or lz4 for lower CPU
```

**Consumer Optimization**:
```properties
# Reduce cross-zone fetches
fetch.min.bytes=50000
fetch.max.wait.ms=500
```

---

### 3. Cost Optimization

**Expected Cost Breakdown**:
| Component | Single-Zone | Multi-Zone | Increase |
|-----------|-------------|------------|----------|
| Cluster (CKUs) | $1.50/CKU/hr | $2.00/CKU/hr | +33% |
| Storage (RF) | 1TB × $0.10/GB | 3TB × $0.10/GB | +200% |
| Bandwidth | Minimal | Cross-zone charges | +Variable |
| **Total** | ~$2,000/mo | ~$2,800/mo | +40% |

**Optimization Strategies**:
1. **Right-size cluster**: Monitor for 2 weeks, adjust CKUs
2. **Reduce retention**: Non-critical topics (7d → 3d)
3. **Enable tiered storage**: Move old data to S3/GCS
4. **Compression**: Enable on topics to reduce storage

---

### 4. Operational Changes

**New Runbooks Needed**:
- Multi-zone failure scenarios
- Zone failure testing procedures
- Cross-zone replication monitoring
- Updated disaster recovery procedures

**Team Training**:
- Multi-zone architecture concepts
- New monitoring dashboards
- Updated alerting thresholds

---

### 5. Decommission Timeline

**Wait Period**: 7-14 days minimum

**Decommission Checklist**:
- [ ] 7+ days stable operation
- [ ] Zero critical incidents
- [ ] Business validation complete
- [ ] All applications confirmed on destination
- [ ] Performance meets baseline
- [ ] Monitoring dashboards updated
- [ ] Runbooks updated
- [ ] Finance approval for cluster deletion

**Decommission Steps**:
```bash
# 1. Delete Cluster Link
confluent kafka link delete migration-link --cluster $DEST_CLUSTER_ID

# 2. Delete source cluster
confluent kafka cluster delete <source-cluster-id>

# 3. Clean up networking
# - Delete VPC peering
# - Delete PrivateLink endpoints
# - Remove security group rules
```

---

### 6. Success Metrics

**Track These Metrics**:

| Metric | Pre-Migration | Post-Migration | Target |
|--------|---------------|----------------|--------|
| **Availability** | 99.5% | 99.99% | ✅ Improved |
| **Data Loss Events** | Possible | 0 | ✅ Eliminated |
| **Recovery Time** | Hours | <30s | ✅ Automated |
| **Consumer Lag** | <5K msgs | <5K msgs | ✅ Maintained |
| **Producer Latency** | 15ms p99 | 22ms p99 | ⚠️ +7ms (acceptable) |
| **Monthly Cost** | $2,000 | $2,800 | ⚠️ +40% (expected) |

---

## Rollback Procedure

### When to Rollback

**Critical Triggers (Immediate Rollback)**:
- Data loss detected (>1% message count mismatch)
- Producer error rate >5%
- Consumer offset corruption (consumers at offset 0)
- Authentication failures >10% of applications

**High Priority Triggers (Rollback within 15 min)**:
- Consumer lag increasing uncontrollably
- Destination cluster performance degradation (CPU >90%)
- Network connectivity failures

---

### Rollback Steps (Target: <15 minutes)

#### Step 1: STOP Destination Traffic (IMMEDIATE)

```bash
echo "🚨 ROLLBACK INITIATED 🚨"

# Stop all producers
kubectl scale deployment/producer-orders -n production --replicas=0

# Stop all consumers
kubectl scale deployment/consumer-analytics -n production --replicas=0

echo "✅ Destination traffic stopped"
```

---

#### Step 2: Revert Producer Configs

```bash
# Revert to source cluster config
kubectl set env deployment/producer-orders -n production \
  KAFKA_BOOTSTRAP_SERVERS=<source-bootstrap>:9092 \
  KAFKA_API_KEY=<original-source-key> \
  KAFKA_API_SECRET=<original-source-secret>

echo "✅ Producer configs reverted"
```

---

#### Step 3: Restart Producers on Source

```bash
# Scale up producers on source
kubectl scale deployment/producer-orders -n production --replicas=20

# Wait for pods ready
kubectl rollout status deployment/producer-orders -n production

# Validate producers writing to source
kafka-topics --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe --topic orders | grep "Leader"

echo "✅ Producers operational on source"
```

---

#### Step 4: Restart Consumers on Source

```bash
# Revert consumer configs
kubectl set env deployment/consumer-analytics -n production \
  KAFKA_BOOTSTRAP_SERVERS=<source-bootstrap>:9092 \
  KAFKA_API_KEY=<original-source-key> \
  KAFKA_API_SECRET=<original-source-secret>

# Scale up consumers
kubectl scale deployment/consumer-analytics -n production --replicas=10

# Wait for consumers ready
kubectl rollout status deployment/consumer-analytics -n production

# Validate consumers processing
kafka-consumer-groups --bootstrap-server <source-bootstrap>:9092 \
  --command-config source-client.properties \
  --describe --group analytics-group

echo "✅ Consumers operational on source"
```

---

#### Step 5: Resume Kafka Connect

```bash
# Revert connector configs to source (reverse of migration)
for connector in $CONNECTORS; do
  # Load original source config
  confluent connect update $connector \
    --cluster $CONNECT_CLUSTER_ID \
    --config-file "connector-${connector}-orig.json"
  
  confluent connect resume $connector --cluster $CONNECT_CLUSTER_ID
done

echo "✅ Kafka Connect operational on source"
```

---

#### Step 6: Validate Rollback Complete

```bash
echo "=== ROLLBACK VALIDATION ==="

# 1. Producers on source
kubectl get pods -n production -l app=producer-orders

# 2. Consumers on source
kubectl get pods -n production -l app=consumer-analytics

# 3. No errors
kubectl logs -l app=producer-orders -n production --tail=20 | grep -i error
kubectl logs -l app=consumer-analytics -n production --tail=20 | grep -i error

echo "✅ ROLLBACK COMPLETE - Source cluster operational"
```

---

#### Step 7: Post-Rollback Analysis

**Incident Report Template**:
```markdown
# Migration Rollback Incident Report

**Date**: _______________
**Time**: _______________
**Duration**: _______________

## Rollback Trigger
- [ ] Data loss detected
- [ ] Producer error rate >5%
- [ ] Consumer offset issues
- [ ] Other: _______________

## Root Cause
[Detailed explanation of what went wrong]

## Impact
- Applications affected: _______________
- Data loss (if any): _______________
- Downtime duration: _______________

## Remediation Steps Taken
1. _______________
2. _______________
3. _______________

## Lessons Learned
[What we learned and what we'll do differently]

## Action Items
- [ ] Fix root cause issue
- [ ] Update migration procedure
- [ ] Retry migration on: _______________
```

---

## Summary Checklists

### Pre-Migration Checklist

- [ ] Source cluster inventoried (topics, consumers, connectors)
- [ ] Destination cluster provisioned (Multi-AZ Enterprise)
- [ ] Networking configured (VPC peering or PrivateLink tested)
- [ ] Service accounts and API keys created
- [ ] ACLs planned (auto-sync or manual)
- [ ] Staging migration completed successfully
- [ ] Rollback procedure tested in staging
- [ ] Team trained on cutover procedure
- [ ] Monitoring dashboards prepared
- [ ] Cutover window scheduled and approved

---

### Cutover Day Checklist

**T-1 Hour**:
- [ ] Mirror lag <100 messages (all topics)
- [ ] Mirror lag stable for 30+ minutes
- [ ] Consumer offset sync validated
- [ ] Cluster Link state: ACTIVE
- [ ] No ongoing deployments
- [ ] Team on bridge
- [ ] Rollback plan ready

**During Cutover**:
- [ ] Kafka Connect paused
- [ ] Producers switched (canary → full rollout)
- [ ] Producers validated on destination
- [ ] Consumers stopped on source
- [ ] Final source offsets captured
- [ ] Consumers started on destination
- [ ] Consumer offsets validated
- [ ] Kafka Connect updated and resumed

**T+30 Min (Cutover Complete)**:
- [ ] All producers on destination
- [ ] All consumers on destination
- [ ] Kafka Connect RUNNING
- [ ] No application errors
- [ ] Consumer lag healthy
- [ ] End-to-end validation passed

---

### Post-Migration Checklist

**Week 1**:
- [ ] Data integrity validated (message counts match)
- [ ] Consumer offsets correct (no reprocessing)
- [ ] Application health confirmed
- [ ] Performance baseline met
- [ ] No critical errors

**Week 2**:
- [ ] 7+ days stable operation
- [ ] Cost analysis completed
- [ ] Performance tuning applied
- [ ] Monitoring updated
- [ ] Runbooks updated

**Week 10 (Decommission)**:
- [ ] 7-14 days stable validation period complete
- [ ] Business sign-off obtained
- [ ] Cluster Link deleted
- [ ] Source cluster deleted
- [ ] Networking resources cleaned up
- [ ] Documentation updated
- [ ] Post-mortem completed

---

**END OF MIGRATION GUIDE**
