---
inclusion: manual
---

# Troubleshooting MSK Express Broker

This guide provides workflows for diagnosing and resolving common MSK Express Broker issues.

## Quick Diagnostic: Cluster Health Check

Before diving into specific issues, gather baseline metrics:

**Action:** Use `get_cluster_info` tool
```json
{
  "region": "us-east-1",
  "cluster_arn": "<your-cluster-arn>",
  "info_type": "all"
}
```

**Key metrics to collect:**
1. CPU: `CpuUser` + `CpuSystem`
2. Throughput: `BytesInPerSec`, `BytesOutPerSec`
3. Partitions: `GlobalPartitionCount`, `PartitionCount`
4. Leaders: `LeaderCount` (check distribution)
5. Connections: `ClientConnectionCount`

**Note:** Some metrics require PER_BROKER monitoring level. See `monitoring-best-practices.md` for the full metrics reference.

---

## Troubleshooting Workflow 1: Slow Running Clients

**Symptoms:**
- High producer latency
- Slow consumer fetch times
- Client timeouts

### Step 1: Check Broker CPU

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["CpuUser", "CpuSystem"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**If CPU > 60%:** See "High CPU Usage" workflow below.

### Step 2: Check Throughput Limits

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["BytesInPerSec", "BytesOutPerSec"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

Compare against instance limits:
| Instance Size | Max Ingress (MBps) | Max Egress (MBps) |
|---------------|-------------------|-------------------|
| express.m7g.large | 15.6 | 31.2 |
| express.m7g.xlarge | 31.2 | 62.5 |
| express.m7g.2xlarge | 62.5 | 125.0 |
| express.m7g.4xlarge | 124.9 | 249.8 |

**If approaching limits:** Scale to larger instance or add brokers.

### Step 3: Check for Throttling

**Note:** Requires PER_BROKER monitoring level. Metrics only appear after throttling occurs.

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["ProduceThrottleTime", "FetchThrottleTime"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**If throttling > 0:**
- Reduce client request rate
- Increase cluster capacity
- Optimize batch sizes

### Step 4: Check Network Processor

**Note:** Requires PER_BROKER monitoring level.

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["NetworkProcessorAvgIdlePercent"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**If idle < 30%:** Network threads are saturated.
- Reduce connection count
- Optimize batch sizes
- Scale cluster

### Step 5: Check Request Queue Time

**Note:** Requires PER_BROKER monitoring level.

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["ProduceRequestQueueTimeMsMean"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**If queue time is high:**
- Broker is overloaded
- Scale cluster or reduce load

### Resolution Summary for Slow Clients

| Root Cause | Solution |
|------------|----------|
| High CPU | Scale instance size or add brokers |
| Throughput limit | Scale instance size |
| Throttling | Reduce request rate or scale |
| Network saturation | Reduce connections, optimize batches |
| High queue time | Scale cluster capacity |

---

## Troubleshooting Workflow 2: High CPU Usage

**Symptoms:**
- Total CPU (User + System) > 60%
- Performance degradation
- Increased latency

### Step 1: Identify CPU Distribution

**Action:** Use `get_cluster_telemetry` tool for each broker
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["CpuUser", "CpuSystem"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**Determine pattern:**
- All brokers high → Cluster-wide issue
- Single broker high → Skewed distribution

### Step 2: Check Traffic Distribution

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["BytesInPerSec", "BytesOutPerSec"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**If traffic is skewed:**
- Check producer partitioning key
- Verify leader distribution
- Consider partition reassignment

### Step 3: Check Partition Count

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["PartitionCount", "GlobalPartitionCount"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**If partition count is excessive:**
- Add more brokers
- Consolidate low-traffic topics
- Reassign partitions (max 20 per operation)

### Step 4: Check Connection Count

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["ClientConnectionCount", "ConnectionCreationRate"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**If connections are high:**
- Implement connection pooling
- Reduce client instances
- Increase `reconnect.backoff.ms` on clients

### Step 5: Check for Recent Operations

**Action:** Use `get_cluster_info` tool
```json
{
  "region": "us-east-1",
  "cluster_arn": "<your-cluster-arn>",
  "info_type": "operations"
}
```

**If recent maintenance:**
- CPU spike during operations is normal
- Wait for operation to complete
- Ensure 40% headroom for future operations

### Resolution Summary for High CPU

| Root Cause | Solution |
|------------|----------|
| High traffic | Scale instance size or add brokers |
| Skewed distribution | Fix partitioning, rebalance leaders |
| Excessive partitions | Add brokers, consolidate topics |
| High connections | Connection pooling, reduce clients |
| Maintenance | Wait for completion |

---

## Troubleshooting Workflow 3: Replication Lag

**Symptoms:**
- Leader distribution imbalance
- Data durability concerns
- Follower replicas behind

**Note:** `UnderReplicatedPartitions` is not available for Express brokers. Monitor `LeaderCount` distribution and `ReplicationBytesInPerSec`/`ReplicationBytesOutPerSec` instead.

### Step 1: Check Leader Distribution

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["LeaderCount"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**Evaluation:**
- Leaders should be evenly distributed across brokers
- Significant imbalance indicates potential replication issues

### Step 2: Check Broker Health

**Action:** Use `get_cluster_info` tool
```json
{
  "region": "us-east-1",
  "cluster_arn": "<your-cluster-arn>",
  "info_type": "nodes"
}
```

**Verify:**
- All brokers are ACTIVE
- No broker failures or restarts

### Step 3: Check Replication Throughput

**Note:** Requires PER_BROKER monitoring level.

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["ReplicationBytesInPerSec", "ReplicationBytesOutPerSec"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

### Resolution Summary for Replication Lag

| Root Cause | Solution |
|------------|----------|
| Broker failure | Wait for recovery, check broker status |
| Volume recovery | Wait for replication, reduce producer traffic |
| Uneven leaders | Preferred leader election |
| Broker overload | Scale cluster, redistribute partitions |

---

## Troubleshooting Workflow 4: Authentication Failures (IAM)

**Symptoms:**
- "Too many connects" errors
- Authentication timeouts
- Connection failures

### Step 1: Check Connection Rate

**Note:** `ConnectionCreationRate` and `ConnectionCloseRate` require PER_BROKER monitoring level.

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["ConnectionCreationRate", "ConnectionCloseRate"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**Limit:** 100 new IAM connections per second per broker

### Step 2: Check Connection Count

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["ClientConnectionCount"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**Limit:** 3000 IAM connections per broker

### Step 3: Check for Connection Churn

The metrics from Step 1 (`ConnectionCreationRate` and `ConnectionCloseRate`) will show connection churn.

**If high churn:**
- Clients are reconnecting frequently
- Check client `reconnect.backoff.ms` setting

### Resolution Summary for Authentication Failures

| Root Cause | Solution |
|------------|----------|
| Connection rate > 100/sec | Increase `reconnect.backoff.ms` to 1000-5000ms |
| Connection count > 3000 | Reduce clients, use connection pooling |
| Connection churn | Implement exponential backoff |
| Simultaneous startup | Stagger client startup times |

**Client Configuration Fix:**
```properties
reconnect.backoff.ms=1000
reconnect.backoff.max.ms=10000
```

---

## Troubleshooting Workflow 5: Consumer Group Stuck in Rebalance

**Symptoms:**
- Consumer group perpetually in `PreparingRebalance` state
- Consumers unable to consume messages
- Frequent rebalancing

### Root Cause

Apache Kafka issue KAFKA-9752 (affects versions 2.3.1 and 2.4.1)

### Solutions

**Option 1: Upgrade Cluster (Recommended)**
- Upgrade to Amazon MSK bug-fix version 2.4.1.1 or later

**Option 2: Implement Static Membership**

Configure clients with:
```properties
group.instance.id=unique-instance-id
session.timeout.ms=240000
```

This prevents premature rebalancing during consumer restarts.

**Option 3: Reboot Coordinator Broker**

**Action:** Use `reboot_broker` tool
```json
{
  "region": "us-east-1",
  "cluster_arn": "<your-cluster-arn>",
  "broker_ids": ["<coordinator-broker-id>"]
}
```

⚠️ **Caution:** Only reboot after identifying the coordinator broker for the stuck consumer group.

---

## Quick Reference: Metrics to Check First

| Issue | Primary Metrics | Monitoring Level |
|-------|-----------------|------------------|
| Slow clients | CpuUser, BytesInPerSec | DEFAULT |
| Slow clients (detailed) | ProduceThrottleTime, NetworkProcessorAvgIdlePercent | PER_BROKER |
| High CPU | CpuUser, CpuSystem, BytesInPerSec, PartitionCount | DEFAULT |
| Replication health | LeaderCount | DEFAULT |
| Replication (detailed) | ReplicationBytesInPerSec | PER_BROKER |
| Auth failures | ClientConnectionCount | DEFAULT |
| Auth failures (detailed) | ConnectionCreationRate, IAMTooManyConnections | PER_BROKER |
| Consumer issues | MaxOffsetLag, EstimatedMaxTimeLag | DEFAULT |

**Note:** `UnderReplicatedPartitions` is not available for Express brokers. Use `LeaderCount` distribution instead.

## When to Escalate

Consider AWS Support if:
- UnderReplicatedPartitions persists after broker recovery
- Cluster operations fail repeatedly
- Metrics indicate hardware issues
- Performance doesn't improve after scaling
