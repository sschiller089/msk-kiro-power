---
name: "aws-msk-express-broker"
displayName: "Build and Operate MSK Express Broker"
description: "Guide developers through creating and operating Amazon MSK Express Brokers with best practices, monitoring strategies, and troubleshooting workflows for common operational challenges."
keywords: ["aws", "msk", "kafka", "express", "broker", "troubleshooting", "monitoring", "cloudwatch"]
author: "Stephan Schiller"
---

# Build and Operate MSK Express Broker

## Overview

This power guides you through building and operating Amazon MSK Express Brokers according to AWS best practices. Express brokers are pre-configured for high availability and durability with data distributed across three availability zones, replication set to 3, and minimum in-sync replicas set to 2.

This power combines the AWS MSK MCP Server tools with comprehensive guidance on:
- Creating Express broker clusters with proper sizing
- Implementing client-side and server-side best practices
- Monitoring cluster health with CloudWatch metrics
- Troubleshooting common operational challenges (slow clients, replication lag, high CPU usage)
- Optimizing performance and reliability

Whether you're setting up a new Express broker cluster or troubleshooting an existing one, this power provides the tools and knowledge to operate MSK Express Brokers effectively.

## Onboarding

### Prerequisites

Before using this power, ensure you have:

**AWS Setup:**
- AWS account with permissions to manage MSK clusters
- AWS CLI configured with appropriate credentials
- IAM permissions for MSK operations (create, describe, update clusters)
- VPC with subnets across multiple availability zones

**Local Environment:**
- Python 3.10+ installed
- `uv` package manager installed ([installation guide](https://docs.astral.sh/uv/getting-started/installation/))
- AWS credentials configured (via `~/.aws/credentials` or environment variables)

**Knowledge:**
- Basic understanding of Apache Kafka concepts (topics, partitions, brokers)
- Familiarity with AWS networking (VPCs, subnets, security groups)

### Installation

The AWS MSK MCP Server is already configured in your workspace. Verify the configuration:

1. Check that `mcp.json` contains the AWS MSK MCP Server configuration
2. Ensure your AWS profile is set correctly in the `AWS_PROFILE` environment variable
3. Test connectivity by listing your MSK clusters (see Common Workflows below)

### Configuration

**AWS Profile Setup:**

Replace the placeholder in `mcp.json` with your actual AWS profile name:

```json
{
  "mcpServers": {
    "awslabs.aws-msk-mcp-server": {
      "env": {
        "AWS_PROFILE": "your-actual-profile-name"
      }
    }
  }
}
```

**AWS Region:**

Most MCP tools require specifying an AWS region. Common regions include:
- `us-east-1` (US East - N. Virginia)
- `us-west-2` (US West - Oregon)
- `eu-west-1` (Europe - Ireland)
- `ap-southeast-1` (Asia Pacific - Singapore)

## Best Practices for Express Brokers

### Right-Sizing Your Cluster

**Number of Brokers:**

Express brokers come with defined throughput capacity. Size your cluster based on your application's ingress (write) and egress (read) requirements.

**Recommended Throughput Limits per Broker:**

| Instance Size | Ingress (MBps) | Egress (MBps) |
|---------------|----------------|---------------|
| express.m7g.large | 15.6 | 31.2 |
| express.m7g.xlarge | 31.2 | 62.5 |
| express.m7g.2xlarge | 62.5 | 125.0 |
| express.m7g.4xlarge | 124.9 | 249.8 |
| express.m7g.8xlarge | 250.0 | 500.0 |
| express.m7g.12xlarge | 375.0 | 750.0 |
| express.m7g.16xlarge | 500.0 | 1000.0 |

**Example Sizing:**
If your application needs 45 MBps ingress and 90 MBps egress, use 3 `express.m7g.large` brokers (each handling 15 MBps ingress and 30 MBps egress).

**Partition Count:**

For high partition, low throughput use cases, you can pack more partitions per broker if you're not sending traffic across all partitions. However, perform thorough testing to validate cluster health.

If partition count exceeds maximum allowed values and your cluster becomes overloaded, you'll be prevented from:
- Creating new topics
- Adding partitions to existing topics
- Updating cluster configuration

### CPU Usage Guidelines

**Target: Keep total CPU utilization under 60%**

Maintain at least 40% available CPU capacity to handle:
- Planned events (version upgrades, broker restarts)
- Unplanned events (hardware failures, AZ failures)
- Partition leadership redistribution

Total CPU = CPU User + CPU System

Monitor using CloudWatch metrics: `CpuUser` and `CpuSystem`

### Connection Management

**Connection Limits:**

| Authentication Type | Limit |
|---------------------|-------|
| IAM Access Control | 3000 TCP connections per broker |
| IAM (new connections) | 100 per second per broker |
| Non-IAM | No enforced limit (monitor CPU/memory) |

**Client Configuration:**

Set `reconnect.backoff.ms` on clients to handle connection retries gracefully. Example: Set to 1000 for 1-second retry interval.

### Client-Side Best Practices

**Producer Configuration:**
- Set appropriate `retries` value for transient errors
- Use proper partitioning keys to distribute data evenly
- Configure `acks=all` for durability (Express brokers handle this efficiently)
- Monitor producer metrics for throttling

**Consumer Configuration:**
- Use static membership protocol for stable consumer groups
- Set reasonable `session.timeout.ms` (e.g., 4 minutes for 5-minute tolerance)
- Reduce offset commit frequency to minimize broker load
- Monitor consumer lag metrics

### Partition Reassignment

When moving partitions between brokers:
- **Limit to 20 partitions per reassignment operation**
- Use `kafka-reassign-partitions.sh` tool
- Monitor cluster performance during reassignment
- Plan reassignments during low-traffic periods

## Common Workflows

### Workflow 1: Create Express Broker Cluster

**Goal:** Create a new MSK Express broker cluster with proper configuration

**Steps:**

1. **Determine cluster requirements:**
   - Calculate required throughput (ingress/egress)
   - Choose appropriate broker instance size
   - Determine number of brokers needed
   - Identify VPC and subnets (must span 3 AZs)

2. **Create the cluster using MCP tools:**

Use the `create_cluster` tool with cluster type "SERVERLESS" for Express brokers:

```json
{
  "region": "us-east-1",
  "cluster_name": "my-express-cluster",
  "cluster_type": "SERVERLESS",
  "kwargs": {
    "serverless": {
      "vpcConfigs": [{
        "subnetIds": ["subnet-xxx", "subnet-yyy", "subnet-zzz"],
        "securityGroupIds": ["sg-xxx"]
      }],
      "clientAuthentication": {
        "sasl": {
          "iam": {"enabled": true}
        }
      }
    }
  }
}
```

3. **Monitor cluster creation:**

Use `get_cluster_info` to check cluster status:

```json
{
  "region": "us-east-1",
  "cluster_arn": "arn:aws:kafka:us-east-1:123456789012:cluster/my-express-cluster/...",
  "info_type": "metadata"
}
```

Wait for cluster state to become "ACTIVE" (typically takes 15-30 minutes).

4. **Verify cluster configuration:**

Check broker endpoints and configuration:

```json
{
  "region": "us-east-1",
  "cluster_arn": "arn:aws:kafka:us-east-1:123456789012:cluster/my-express-cluster/...",
  "info_type": "brokers"
}
```

### Workflow 2: Monitor Cluster Health

**Goal:** Establish monitoring for Express broker cluster health and performance

**Key Metrics to Monitor:**

**Throughput Metrics:**
- `BytesInPerSec` - Ingress traffic per broker
- `BytesOutPerSec` - Egress traffic per broker
- Compare against recommended limits for your instance size

**CPU Metrics:**
- `CpuUser` - CPU in user space
- `CpuSystem` - CPU in kernel space
- **Target:** Total (User + System) < 60%

**Connection Metrics:**
- `ClientConnectionCount` - Active authenticated connections
- `ConnectionCount` - Total connections (authenticated + unauthenticated + inter-broker)

**Partition Metrics:**
- `GlobalPartitionCount` - Total partitions across cluster
- `PartitionCount` - Partitions per broker (including replicas)
- `LeaderCount` - Leader partitions per broker

**Consumer Lag Metrics:**
- `MaxOffsetLag` - Maximum offset lag across partitions
- `EstimatedMaxTimeLag` - Time estimate to drain lag
- `SumOffsetLag` - Aggregated offset lag for topic

**Steps:**

1. **Get available metrics for your cluster:**

```json
{
  "region": "us-east-1",
  "action": "available_metrics",
  "cluster_arn": "arn:aws:kafka:us-east-1:123456789012:cluster/my-express-cluster/..."
}
```

2. **Retrieve current metrics:**

```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "arn:aws:kafka:us-east-1:123456789012:cluster/my-express-cluster/...",
  "kwargs": {
    "metric_name": "CpuUser",
    "start_time": "2024-01-15T00:00:00Z",
    "end_time": "2024-01-15T23:59:59Z",
    "period": 300
  }
}
```

3. **Set up CloudWatch alarms:**

Create alarms for critical thresholds:
- CPU utilization > 60%
- BytesInPerSec approaching broker limit
- MaxOffsetLag > acceptable threshold
- ClientConnectionCount approaching 3000 (for IAM)

### Workflow 3: Update Cluster Configuration

**Goal:** Modify cluster settings or scale brokers

**Scaling Brokers (Serverless/Express):**

For Express brokers, scaling is automatic based on throughput. However, you can update other configurations:

1. **Get current cluster version:**

```json
{
  "region": "us-east-1",
  "cluster_arn": "arn:aws:kafka:us-east-1:123456789012:cluster/my-express-cluster/...",
  "info_type": "metadata"
}
```

Note the `currentVersion` field.

2. **Update monitoring settings:**

```json
{
  "region": "us-east-1",
  "cluster_arn": "arn:aws:kafka:us-east-1:123456789012:cluster/my-express-cluster/...",
  "current_version": "K3AEGXETSR30VB",
  "enhanced_monitoring": "PER_BROKER",
  "open_monitoring": {
    "prometheus": {
      "jmxExporter": {"enabledInBroker": true},
      "nodeExporter": {"enabledInBroker": true}
    }
  }
}
```

3. **Monitor the update operation:**

```json
{
  "region": "us-east-1",
  "cluster_operation_arn": "arn:aws:kafka:us-east-1:123456789012:cluster-operation/..."
}
```

## Troubleshooting

### Slow Running Clients

**Problem:** Clients experiencing slow produce or consume operations

**Metrics to Check:**

1. **Producer Latency:**
   - `ProduceTotalTimeMsMean` - Mean produce time
   - `ProduceRequestQueueTimeMsMean` - Time in request queue
   - `ProduceResponseSendTimeMsMean` - Time sending response

2. **Consumer Latency:**
   - `FetchConsumerTotalTimeMsMean` - Total fetch time
   - `FetchConsumerRequestQueueTimeMsMean` - Time in request queue
   - `FetchConsumerLocalTimeMsMean` - Processing time at leader

3. **Network Metrics:**
   - `NetworkRxErrors` / `NetworkTxErrors` - Network errors
   - `NetworkRxDropped` / `NetworkTxDropped` - Dropped packets
   - `RequestTime` - Average request processing time

**Diagnostic Steps:**

1. Check if broker CPU is under 60%:
   ```
   Monitor: CpuUser + CpuSystem < 60%
   ```

2. Verify throughput is within limits:
   ```
   Check: BytesInPerSec and BytesOutPerSec per broker
   Compare against instance size limits
   ```

3. Check for throttling:
   ```
   Monitor: ProduceThrottleTime, FetchThrottleTime
   If > 0, clients are being throttled
   ```

4. Examine network processor idle time:
   ```
   Monitor: NetworkProcessorAvgIdlePercent
   Low values indicate network thread saturation
   ```

**Solutions:**

- **If CPU is high:** Scale to larger instance size or add more brokers
- **If throttled:** Reduce client request rate or increase cluster capacity
- **If network saturated:** Optimize batch sizes, reduce connection count
- **If request queue time high:** Increase broker capacity or reduce load

### Replication Lag

**Problem:** Follower replicas falling behind leader partitions

**Metrics to Check:**

1. **Replication Throughput:**
   - `ReplicationBytesInPerSec` - Bytes received from other brokers
   - `ReplicationBytesOutPerSec` - Bytes sent to other brokers

2. **Follower Fetch Performance:**
   - `FetchFollowerTotalTimeMsMean` - Total follower fetch time
   - `FetchFollowerLocalTimeMsMean` - Processing time at leader
   - `FetchFollowerRequestQueueTimeMsMean` - Time in request queue

3. **Partition Status:**
   - `UnderReplicatedPartitions` - Should be 0 in healthy cluster
   - `LeaderCount` - Distribution of leaders across brokers

**Diagnostic Steps:**

1. Check for under-replicated partitions:
   ```
   Monitor: UnderReplicatedPartitions metric
   If > 0, investigate cause
   ```

2. Verify broker health:
   ```
   Check all brokers are ACTIVE
   Monitor CPU, memory, and disk usage
   ```

3. Check for volume saturation (after volume replacement):
   ```
   Monitor: Disk throughput approaching 250 MiB/s limit
   This can impact caught-up partitions
   ```

4. Examine leader distribution:
   ```
   Monitor: LeaderCount per broker
   Uneven distribution can cause hotspots
   ```

**Solutions:**

- **Volume replacement recovery:** Wait for replication to complete; consider temporarily reducing producer traffic
- **Uneven leader distribution:** Trigger preferred leader election
- **Broker overload:** Scale cluster or redistribute partitions
- **Network issues:** Check security groups, VPC configuration, and inter-AZ connectivity

### High CPU Usage

**Problem:** One or more brokers showing CPU utilization > 60%

**Common Causes and Solutions:**

**1. High Incoming/Outgoing Traffic**

**Symptoms:**
- `BytesInPerSec` or `BytesOutPerSec` metrics are high or skewed
- CPU correlates with traffic spikes

**Diagnostic Steps:**
- Check if traffic exceeds recommended throughput limits
- Verify partition distribution across brokers
- Examine producer partitioning key for even distribution

**Solutions:**
- Update producer partitioning logic to distribute data evenly
- Scale to larger instance size
- Add more brokers to distribute load
- Reduce consumer group offset commit frequency

**2. Excessive Partition Count**

**Symptoms:**
- `PartitionCount` per broker exceeds recommended values
- Performance degradation across cluster
- Unable to create topics or add partitions

**Diagnostic Steps:**
- Check `PartitionCount` and `GlobalPartitionCount` metrics
- Calculate partitions per broker (including replicas)

**Solutions:**
- Add more brokers to distribute partitions
- Consolidate topics with low traffic
- Use `kafka-reassign-partitions.sh` to rebalance (max 20 partitions per operation)
- Scale to larger instance size

**3. High Connection Count**

**Symptoms:**
- `ClientConnectionCount` or `ConnectionCount` metrics are high
- `IAMTooManyConnections` > 0 (for IAM auth)
- CPU spikes correlate with connection activity

**Diagnostic Steps:**
- Monitor `ConnectionCreationRate` and `ConnectionCloseRate`
- Check for connection churn (frequent connect/disconnect)
- Verify client `reconnect.backoff.ms` configuration

**Solutions:**
- Reduce number of client connections
- Implement connection pooling in applications
- Increase `reconnect.backoff.ms` on clients (e.g., 1000ms)
- Scale to larger instance size
- Consolidate consumer groups

**4. Skewed Data Distribution**

**Symptoms:**
- CPU high on specific brokers, normal on others
- `LeaderCount` unevenly distributed
- `BytesInPerSec` varies significantly between brokers

**Diagnostic Steps:**
- Check partition distribution using `kafka-topics.sh --describe`
- Verify leader distribution across brokers
- Examine producer partitioning strategy

**Solutions:**
- Rebalance partitions across brokers
- Fix producer partitioning key to use round-robin
- Trigger preferred leader election
- Reassign partitions to balance load

**5. Open Monitoring with Prometheus**

**Symptoms:**
- CPU increased after enabling Prometheus monitoring
- High metric emission rate

**Diagnostic Steps:**
- Check Prometheus scrape interval configuration
- Monitor metric collection frequency

**Solutions:**
- Increase scrape interval to minimum 1 minute per broker
- Disable unnecessary metric exporters
- Use PER_BROKER monitoring level instead of PER_TOPIC_PER_PARTITION

**6. Broker Maintenance or Failures**

**Symptoms:**
- CPU spike during cluster operations
- Recent version upgrade or broker restart
- Partition leadership redistribution

**Diagnostic Steps:**
- Check cluster operation history
- Monitor for recent broker restarts or failures
- Verify partition leadership changes

**Solutions:**
- Wait for maintenance to complete (CPU will normalize)
- Ensure 40% CPU headroom for future operations
- Plan maintenance during low-traffic periods

### Consumer Group Stuck in PreparingRebalance

**Problem:** Consumer group perpetually rebalancing, unable to consume messages

**Cause:** Apache Kafka issue KAFKA-9752 (affects versions 2.3.1 and 2.4.1)

**Solutions:**

**Option 1: Upgrade Cluster (Recommended)**
- Upgrade to Amazon MSK bug-fix version 2.4.1.1 or later
- Use `update_cluster_configuration` or version upgrade tools

**Option 2: Implement Static Membership Protocol**
- Configure clients with `group.instance.id`
- Set `session.timeout.ms` to allow recovery time (e.g., 240000 for 4 minutes)
- Prevents premature rebalancing during consumer restarts

**Option 3: Reboot Coordinating Broker**
- Identify coordinator broker for stuck consumer group
- Reboot the specific broker node
- Monitor consumer group state after reboot

### Failed Authentication: Too Many Connects

**Problem:** IAM clients receiving "Too many connects" authentication errors

**Cause:** Broker protecting itself from aggressive connection rate (> 100 connections/second)

**Metrics to Check:**
- `IAMNumberOfConnectionRequests` - IAM auth requests per second
- `IAMTooManyConnections` - Connections attempted beyond limit

**Solutions:**
- Increase `reconnect.backoff.ms` on clients (e.g., 1000-5000ms)
- Implement exponential backoff for connection retries
- Reduce number of concurrent client connections
- Stagger client startup times
- Use connection pooling

### Disk Space Running Low

**Problem:** Partitions going offline or replicas out of sync due to low disk space

**Metrics to Check:**
- `StorageUsed` - Total storage across cluster

**Solutions:**

**Adjust Data Retention:**
- Reduce `retention.ms` for topics
- Reduce `retention.bytes` for topics
- Enable log compaction for appropriate topics

**Monitor and Alert:**
- Set CloudWatch alarms for storage thresholds
- Monitor storage growth trends
- Plan capacity increases proactively

**For Express Brokers:**
- Storage scales automatically with throughput
- Ensure retention policies align with storage capacity

## MCP Tools Reference

### Cluster Management Tools

**create_cluster** - Create new MSK cluster (provisioned or serverless)
- Parameters: region, cluster_name, cluster_type, kwargs
- Use cluster_type="SERVERLESS" for Express brokers

**get_cluster_info** - Get comprehensive cluster information
- Parameters: region, cluster_arn, info_type (metadata, brokers, nodes, etc.)
- Returns cluster state, configuration, and operational details

**describe_cluster_operation** - Check status of cluster operations
- Parameters: region, cluster_operation_arn
- Monitor create, update, or delete operations

### Monitoring Tools

**get_cluster_telemetry** - Retrieve CloudWatch metrics
- Parameters: region, action (metrics, available_metrics), cluster_arn, kwargs
- Access all Express broker metrics

**get_cluster_best_practices** - Get sizing and configuration recommendations
- Parameters: instance_type, number_of_brokers
- Returns best practices for cluster configuration

### Configuration Tools

**update_monitoring** - Update monitoring settings
- Parameters: region, cluster_arn, current_version, enhanced_monitoring
- Configure CloudWatch and Prometheus monitoring levels

**update_security** - Update security settings
- Parameters: region, cluster_arn, current_version, client_authentication, encryption_info
- Modify authentication and encryption configuration

**create_configuration** - Create MSK configuration
- Parameters: region, name, server_properties, kafka_versions
- Define custom Kafka broker configurations

**update_configuration** - Update existing configuration
- Parameters: region, arn, server_properties
- Modify broker configuration settings

### VPC and Networking Tools

**create_vpc_connection** - Create VPC connection
- Parameters: region, cluster_arn, vpc_id, subnet_ids, security_groups
- Enable cross-VPC access to cluster

**describe_vpc_connection** - Get VPC connection details
- Parameters: region, vpc_connection_arn
- Check VPC connection status and configuration

### Security Tools

**associate_scram_secret** - Associate SCRAM secrets
- Parameters: region, cluster_arn, secret_arns
- Configure SASL/SCRAM authentication

**put_cluster_policy** - Attach resource policy
- Parameters: region, cluster_arn, policy
- Define cluster access policies

**list_customer_iam_access** - List IAM access information
- Parameters: region, cluster_arn
- View IAM-based access configuration

### Operational Tools

**reboot_broker** - Reboot specific brokers
- Parameters: region, cluster_arn, broker_ids
- Restart brokers for maintenance or troubleshooting

**tag_resource** / **untag_resource** - Manage resource tags
- Parameters: region, resource_arn, tags/tag_keys
- Organize and categorize MSK resources

## Additional Resources

- [AWS MSK Express Best Practices](https://docs.aws.amazon.com/msk/latest/developerguide/bestpractices-express.html)
- [AWS MSK Troubleshooting Guide](https://docs.aws.amazon.com/msk/latest/developerguide/troubleshooting.html)
- [AWS MSK Express Metrics](https://docs.aws.amazon.com/msk/latest/developerguide/metrics-details-express.html)
- [AWS MSK MCP Server Documentation](https://awslabs.github.io/mcp/servers/aws-msk-mcp-server)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

---

**MCP Server:** awslabs.aws-msk-mcp-server
**Installation:** `uvx awslabs.aws-msk-mcp-server@latest --allow-writes`
