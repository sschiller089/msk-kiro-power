---
inclusion: manual
---

# Monitoring MSK Express Broker: Best Practices

This guide covers how to monitor your MSK Express Broker cluster and compare its configuration against AWS best practices.

## Key Best Practice Thresholds

| Metric | Target | Why |
|--------|--------|-----|
| Total CPU (User + System) | < 60% | Reserve 40% headroom for failover and maintenance |
| Partition count per broker | Within instance limits | Prevents cluster overload and topic creation blocks |
| IAM connections per broker | < 3000 | Hard limit for IAM-authenticated connections |
| IAM connection rate | < 100/sec per broker | Prevents "Too many connects" errors |

## Step 1: Check CPU Utilization

**Target: Total CPU < 60%**

CPU headroom is critical for Express Brokers to handle:
- Planned events (version upgrades, broker restarts)
- Unplanned events (hardware failures, AZ failures)
- Partition leadership redistribution

**Action:** Use `get_cluster_telemetry` tool for CPU metrics
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

**Note:** Replace timestamps with actual ISO 8601 values (e.g., `2026-02-04T14:00:00Z`).

**Evaluation:**
- Calculate: `Total CPU = CpuUser + CpuSystem`
- ✅ **Healthy:** Total CPU < 60%
- ⚠️ **Warning:** Total CPU 60-80%
- ❌ **Critical:** Total CPU > 80%

## Step 2: Check Partition Distribution

**Target: Partitions within instance limits**

Excessive partitions cause:
- Inability to create new topics
- Inability to add partitions to existing topics
- Cluster configuration update blocks

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["GlobalPartitionCount", "PartitionCount"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**Evaluation:**
- Compare against instance size limits
- Check for even distribution across brokers
- Uneven distribution indicates potential hotspots

## Step 3: Check Throughput Against Limits

**Target: Throughput within instance capacity**

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

**Evaluation:**
Compare per-broker throughput against instance limits:

| Instance Size | Max Ingress (MBps) | Max Egress (MBps) |
|---------------|-------------------|-------------------|
| express.m7g.large | 15.6 | 31.2 |
| express.m7g.xlarge | 31.2 | 62.5 |
| express.m7g.2xlarge | 62.5 | 125.0 |
| express.m7g.4xlarge | 124.9 | 249.8 |
| express.m7g.8xlarge | 250.0 | 500.0 |
| express.m7g.12xlarge | 375.0 | 750.0 |
| express.m7g.16xlarge | 500.0 | 1000.0 |

## Step 4: Check Connection Metrics (IAM Authentication)

**Target: < 3000 connections per broker, < 100 new connections/sec**

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

**Evaluation:**
- ✅ **Healthy:** < 2500 connections (buffer for spikes)
- ⚠️ **Warning:** 2500-2900 connections
- ❌ **Critical:** > 2900 connections (approaching limit)

## Step 5: Check Replication Health

**Target: UnderReplicatedPartitions = 0**

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["UnderReplicatedPartitions"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**Evaluation:**
- ✅ **Healthy:** 0 under-replicated partitions
- ❌ **Unhealthy:** Any value > 0 requires investigation

## Step 6: Check Consumer Lag

**Target: Lag within acceptable bounds for your use case**

**Action:** Use `get_cluster_telemetry` tool
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<your-cluster-arn>",
  "kwargs": {
    "metrics": ["MaxOffsetLag", "EstimatedMaxTimeLag"],
    "start_time": "<1-hour-ago-ISO8601>",
    "end_time": "<now-ISO8601>",
    "period": 300
  }
}
```

**Evaluation:**
- Define acceptable lag based on your SLAs
- Sudden spikes indicate consumer issues or throughput problems

## Best Practices Summary Checklist

Run through this checklist regularly:

- [ ] Total CPU (User + System) < 60%
- [ ] Throughput within instance limits
- [ ] Partition count within limits
- [ ] Even partition distribution across brokers
- [ ] IAM connections < 3000 per broker
- [ ] UnderReplicatedPartitions = 0
- [ ] Consumer lag within acceptable bounds
- [ ] No throttling (ProduceThrottleTime, FetchThrottleTime = 0)

## Recommended CloudWatch Alarms

Set up alarms for:
1. **CPU utilization > 60%** - Warning threshold
2. **CPU utilization > 80%** - Critical threshold
3. **UnderReplicatedPartitions > 0** - Immediate investigation
4. **ClientConnectionCount > 2500** - Connection pressure
5. **MaxOffsetLag > [your threshold]** - Consumer lag

## Next Steps

- **Issues detected?** → See `troubleshooting.md`
- **Need to scale?** → Consider larger instance size or additional brokers
