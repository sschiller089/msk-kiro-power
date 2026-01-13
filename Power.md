---
name: "aws-msk"
displayName: "AWS MSK"
description: "Manage and troubleshoot AWS MSK (Managed Streaming for Kafka) clusters with health checks, performance analysis, and operational guidance for developers working with Kafka on AWS."
keywords: ["msk", "kafka", "aws", "streaming", "managed-kafka", "troubleshooting"]
author: "stepesch"
---

# AWS MSK

## Overview

AWS MSK (Amazon Managed Streaming for Kafka) is a fully managed service that makes it easy to build and run applications that use Apache Kafka to process streaming data. This power provides comprehensive tools and guidance for managing MSK clusters, from initial setup through production operations.

This power helps developers and operators with:
- **Cluster Health Monitoring** - Quick status checks, broker health, and performance metrics
- **Connection Configuration** - Getting bootstrap servers, authentication details, and client configs
- **Troubleshooting** - Diagnosing common issues like consumer lag, broker problems, and connectivity
- **Capacity Planning** - Analyzing usage and making informed scaling decisions
- **Operational Tasks** - Updating configurations, scaling brokers, managing security settings

Whether you're setting up a new application to connect to MSK, troubleshooting production issues, or optimizing cluster performance, this power provides the workflows and best practices you need.

## Onboarding

### Prerequisites

Before using this power, ensure you have:

1. **AWS Account Access** - Valid AWS credentials with permissions to access MSK
2. **AWS CLI Configured** - AWS CLI installed and configured with appropriate profile
3. **MSK Cluster** - At least one MSK cluster deployed in your AWS account
4. **IAM Permissions** - Required permissions include:
   - `kafka:DescribeCluster`
   - `kafka:ListClusters`
   - `kafka:GetBootstrapBrokers`
   - Additional permissions for write operations if using `--allow-writes`

### Installation

This power uses the AWS MSK MCP server, which should already be configured in your Kiro settings.

**Verify MCP Server Configuration:**

Check that `awslabs.aws-msk-mcp-server` is enabled in your MCP configuration (`.kiro/settings/mcp.json` or `~/.kiro/settings/mcp.json`).

### Configuration

**AWS Profile Setup:**

The MCP server uses AWS credentials from your configured profile. Ensure your AWS profile is set correctly:

```bash
# Verify AWS profile
aws configure list --profile your-profile-name

# Test AWS access
aws kafka list-clusters --region us-east-1 --profile your-profile-name
```

**Environment Variables:**

The MCP server configuration should include:
- `AWS_PROFILE`: Your AWS profile name (e.g., "amplifyuser", "default")
- `AWS_REGION`: Default region for operations (optional, can be specified per-call)

## Common Workflows

### Workflow 1: Get Cluster Overview

**Goal:** Quickly understand what MSK clusters you have and their current status.

**Steps:**

1. List all clusters in a region:


```
Use tool: get_global_info
Parameters:
  region: "us-east-1"
  info_type: "clusters"
```

2. Review the cluster information returned:
   - Cluster name and ARN
   - Current state (ACTIVE, CREATING, UPDATING, etc.)
   - Kafka version
   - Number of broker nodes
   - Instance type
   - Creation date

**Example Output Interpretation:**

```json
{
  "ClusterName": "MSKCluster-production",
  "State": "ACTIVE",
  "CurrentVersion": "K2CL7FJ6VNEXJ1",
  "NumberOfBrokerNodes": 6,
  "BrokerNodeGroupInfo": {
    "InstanceType": "kafka.m5.large"
  }
}
```

This shows a healthy production cluster with 6 brokers running on m5.large instances.

### Workflow 2: Get Connection Details for Applications

**Goal:** Retrieve bootstrap servers and authentication details needed to connect your application to MSK.

**Steps:**

1. Get detailed cluster information including broker endpoints:

```
Use tool: get_cluster_info
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  info_type: "brokers"
```

2. Extract connection information:
   - Bootstrap broker endpoints (plaintext, TLS, SASL)
   - Zookeeper connection string (if needed)
   - Authentication methods enabled

**Example Connection Strings:**

For different authentication methods:

```bash
# TLS authentication
bootstrap.servers=b-1.mycluster.kafka.us-east-1.amazonaws.com:9094,b-2.mycluster.kafka.us-east-1.amazonaws.com:9094

# SASL/SCRAM authentication
bootstrap.servers=b-1.mycluster.kafka.us-east-1.amazonaws.com:9096,b-2.mycluster.kafka.us-east-1.amazonaws.com:9096

# IAM authentication
bootstrap.servers=b-1.mycluster.kafka.us-east-1.amazonaws.com:9098,b-2.mycluster.kafka.us-east-1.amazonaws.com:9098

# Plaintext (unauthenticated)
bootstrap.servers=b-1.mycluster.kafka.us-east-1.amazonaws.com:9092,b-2.mycluster.kafka.us-east-1.amazonaws.com:9092
```

**Client Configuration Examples:**

Java (with IAM authentication):
```java
Properties props = new Properties();
props.put("bootstrap.servers", "b-1.mycluster.kafka.us-east-1.amazonaws.com:9098");
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "AWS_MSK_IAM");
props.put("sasl.jaas.config", "software.amazon.msk.auth.iam.IAMLoginModule required;");
props.put("sasl.client.callback.handler.class", "software.amazon.msk.auth.iam.IAMClientCallbackHandler");
```

Python (with SASL/SCRAM):
```python
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers=['b-1.mycluster.kafka.us-east-1.amazonaws.com:9096'],
    security_protocol='SASL_SSL',
    sasl_mechanism='SCRAM-SHA-512',
    sasl_plain_username='your-username',
    sasl_plain_password='your-password'
)
```

### Workflow 3: Troubleshoot Consumer Lag

**Goal:** Identify and diagnose consumer lag issues affecting application performance.

**Steps:**

1. Check broker CPU utilization:

```
Use tool: get_cluster_telemetry
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  action: "metrics"
  kwargs: {
    "metrics": ["CpuUser"],
    "start_time": "2024-01-01T00:00:00Z",
    "end_time": "2024-01-01T23:59:59Z",
    "period": 300,
    "stat": "Average"
  }
```

2. Check network throughput and consumer lag:

```
Use tool: get_cluster_telemetry
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  action: "metrics"
  kwargs: {
    "metrics": ["BytesInPerSec", "BytesOutPerSec", "EstimatedMaxTimeLag"],
    "start_time": "2024-01-01T00:00:00Z",
    "end_time": "2024-01-01T23:59:59Z",
    "period": 300,
    "stat": "Average"
  }
```

**Note:** The `metrics` parameter accepts an array of metric names, allowing you to query multiple metrics in a single call.

3. Review partition distribution across brokers:

```
Use tool: get_cluster_info
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  info_type: "nodes"
```

**Common Causes and Solutions:**

- **High consumer lag + Low broker CPU** → Consumer application is slow or under-provisioned
  - Solution: Scale consumer instances, optimize consumer code
  
- **High consumer lag + High broker CPU** → Broker capacity issue
  - Solution: Scale brokers (add nodes or upgrade instance type)
  
- **Uneven partition distribution** → Imbalanced load across brokers
  - Solution: Rebalance partitions or review topic partition strategy

### Workflow 4: Scale Cluster Capacity

**Goal:** Increase cluster capacity by adding brokers, upgrading instance types, or expanding storage.

**Important:** These operations require the `--allow-writes` flag in your MCP server configuration.

**Option A: Add More Brokers**

```
Use tool: update_broker_count
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  current_version: "K2CL7FJ6VNEXJ1"
  target_number_of_broker_nodes: 9
```

**Option B: Upgrade Broker Instance Type**

```
Use tool: update_broker_type
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  current_version: "K2CL7FJ6VNEXJ1"
  target_instance_type: "kafka.m5.xlarge"
```

**Option C: Expand Storage**

```
Use tool: update_broker_storage
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  current_version: "K2CL7FJ6VNEXJ1"
  target_broker_ebs_volume_info: '[{"KafkaBrokerNodeId": "1", "VolumeSizeGB": 3000}]'
```

**Best Practices for Scaling:**

- Always get the current cluster version before making updates
- Monitor cluster operations with `describe_cluster_operation` tool
- Scale during low-traffic periods when possible
- Test scaling operations in non-production environments first
- Consider auto-scaling storage with provisioned throughput for predictable growth

### Workflow 5: Update Cluster Configuration

**Goal:** Modify Kafka broker configurations for performance tuning or feature enablement.

**Steps:**

1. Review current configuration:

```
Use tool: get_configuration_info
Parameters:
  region: "us-east-1"
  action: "describe"
  arn: "arn:aws:kafka:us-east-1:123456789012:configuration/my-config/uuid"
```

2. Create or update configuration:

```
Use tool: create_configuration
Parameters:
  region: "us-east-1"
  name: "my-optimized-config"
  server_properties: "auto.create.topics.enable=false\ndefault.replication.factor=3\nmin.insync.replicas=2\nlog.retention.hours=168"
  kafka_versions: ["2.8.1", "3.4.0"]
```

3. Apply configuration to cluster:

```
Use tool: update_cluster_configuration
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  configuration_arn: "arn:aws:kafka:us-east-1:123456789012:configuration/my-optimized-config/uuid"
  configuration_revision: 1
  current_version: "K2CL7FJ6VNEXJ1"
```

**Common Configuration Changes:**

- **Disable auto-topic creation**: `auto.create.topics.enable=false`
- **Increase retention**: `log.retention.hours=336` (14 days)
- **Adjust replication**: `default.replication.factor=3`
- **Set minimum in-sync replicas**: `min.insync.replicas=2`
- **Compression**: `compression.type=snappy`

### Workflow 6: Configure Authentication and Security

**Goal:** Set up or modify authentication methods and encryption settings.

**Steps:**

1. Review current security settings:

```
Use tool: get_cluster_info
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  info_type: "metadata"
```

2. Update security configuration:

```
Use tool: update_security
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  current_version: "K2CL7FJ6VNEXJ1"
  client_authentication: {
    "Sasl": {
      "Scram": {"Enabled": true},
      "Iam": {"Enabled": true}
    }
  }
```

3. For SCRAM authentication, associate secrets:

```
Use tool: associate_scram_secret
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  secret_arns: ["arn:aws:secretsmanager:us-east-1:123456789012:secret:AmazonMSK_myuser"]
```

**Authentication Methods:**

- **IAM** - Best for AWS-native applications, no credential management
- **SASL/SCRAM** - Username/password stored in AWS Secrets Manager
- **mTLS** - Certificate-based authentication using AWS Private CA
- **Unauthenticated** - For development/testing only, not recommended for production

### Workflow 7: Monitor Cluster Health

**Goal:** Proactively monitor cluster health and identify potential issues.

**Key Metrics to Monitor:**

1. **Broker CPU Utilization**
   - Target: < 60% average
   - Alert threshold: > 80%

2. **Network Throughput**
   - Monitor: BytesInPerSec, BytesOutPerSec
   - Compare against instance type limits

3. **Disk Space**
   - Target: < 70% utilization
   - Alert threshold: > 85%

4. **Consumer Lag**
   - Target: < 1000 messages or < 5 seconds
   - Depends on use case

5. **Under-Replicated Partitions**
   - Target: 0
   - Alert immediately if > 0

**Get Available Metrics:**

```
Use tool: get_cluster_telemetry
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  action: "available_metrics"
```

**Get Current Metrics:**

```
Use tool: get_cluster_telemetry
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  action: "metrics"
  kwargs: {
    "metrics": ["CpuUser", "KafkaDataLogsDiskUsed", "HeapMemoryAfterGC"],
    "start_time": "2024-01-01T15:00:00Z",
    "end_time": "2024-01-01T16:00:00Z",
    "period": 300,
    "stat": "Average"
  }
```

**Parameters:**
- `metrics`: Array of metric names (e.g., ["CpuUser", "BytesInPerSec"])
- `start_time`: Start time in ISO 8601 format (e.g., "2024-01-01T15:00:00Z")
- `end_time`: End time in ISO 8601 format
- `period`: Data point interval in seconds (e.g., 300 for 5-minute intervals)
- `stat`: Statistic type ("Average", "Sum", "Maximum", "Minimum")

**Enable Enhanced Monitoring:**

```
Use tool: update_monitoring
Parameters:
  region: "us-east-1"
  cluster_arn: "arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/uuid"
  current_version: "K2CL7FJ6VNEXJ1"
  enhanced_monitoring: "PER_TOPIC_PER_BROKER"
  open_monitoring: {
    "Prometheus": {
      "JmxExporter": {"EnabledInBroker": true},
      "NodeExporter": {"EnabledInBroker": true}
    }
  }
```

## Troubleshooting

### MCP Server Connection Issues

**Problem:** MCP server won't start or connect

**Symptoms:**
- Error: "Connection refused"
- Error: "Token has expired and refresh failed"
- Server not responding

**Solutions:**

1. **Verify AWS credentials:**
   ```bash
   aws sts get-caller-identity --profile your-profile-name
   ```

2. **Refresh SSO token (if using AWS SSO):**
   ```bash
   aws sso login --profile your-profile-name
   ```

3. **Check MCP server configuration:**
   - Verify `AWS_PROFILE` environment variable is set correctly
   - Ensure profile exists in `~/.aws/config`
   - Confirm AWS CLI is installed and accessible

4. **Restart Kiro:**
   - MCP servers reconnect automatically on config changes
   - Or manually reconnect from MCP Server view in Kiro

### Cluster Operation Failures

**Error:** "Cluster is in UPDATING state"

**Cause:** Another operation is in progress

**Solution:**
1. Check operation status:
   ```
   Use tool: describe_cluster_operation
   Parameters:
     region: "us-east-1"
     cluster_operation_arn: "arn:aws:kafka:us-east-1:123456789012:cluster-operation/uuid"
   ```

2. Wait for current operation to complete
3. Operations typically take 15-30 minutes
4. Retry your operation after completion

**Error:** "Invalid cluster version"

**Cause:** Cluster version has changed since you retrieved it

**Solution:**
1. Get current cluster version:
   ```
   Use tool: get_cluster_info
   Parameters:
     region: "us-east-1"
     cluster_arn: "your-cluster-arn"
     info_type: "metadata"
   ```

2. Use the `CurrentVersion` field in your update operation
3. Cluster version changes with each successful update

### Permission Errors

**Error:** "User is not authorized to perform: kafka:DescribeCluster"

**Cause:** Insufficient IAM permissions

**Solution:**

1. Verify your IAM permissions include required MSK actions
2. Common required permissions:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": [
         "kafka:DescribeCluster",
         "kafka:ListClusters",
         "kafka:GetBootstrapBrokers",
         "kafka:DescribeClusterV2",
         "kafka:ListNodes"
       ],
       "Resource": "*"
     }]
   }
   ```

3. For write operations, add:
   ```json
   "kafka:UpdateBrokerCount",
   "kafka:UpdateBrokerType",
   "kafka:UpdateBrokerStorage",
   "kafka:UpdateClusterConfiguration",
   "kafka:UpdateMonitoring",
   "kafka:UpdateSecurity"
   ```

### Connection Issues from Applications

**Problem:** Application can't connect to MSK cluster

**Diagnostic Steps:**

1. **Verify network connectivity:**
   - Check security group rules allow traffic on Kafka ports
   - Verify application is in same VPC or has VPC peering/VPN
   - Test connectivity: `telnet broker-endpoint 9092`

2. **Check authentication configuration:**
   - Verify correct authentication method (IAM, SASL, mTLS)
   - Confirm credentials are valid
   - Check client library supports authentication method

3. **Review bootstrap servers:**
   - Ensure using correct port for authentication method
   - Port 9092: Plaintext
   - Port 9094: TLS
   - Port 9096: SASL/SCRAM
   - Port 9098: IAM

4. **Check cluster state:**
   ```
   Use tool: get_cluster_info
   Parameters:
     region: "us-east-1"
     cluster_arn: "your-cluster-arn"
     info_type: "metadata"
   ```
   - Cluster must be in ACTIVE state

### Performance Issues

**Problem:** Slow message production or consumption

**Diagnostic Approach:**

1. **Check broker metrics:**
   - CPU utilization (should be < 80%)
   - Network throughput (check against instance limits)
   - Disk I/O (check for throttling)

2. **Review partition distribution:**
   - Ensure partitions are evenly distributed
   - Check for hot partitions (uneven load)

3. **Analyze consumer lag:**
   - Identify slow consumers
   - Check consumer group rebalancing

4. **Consider scaling:**
   - Add brokers if CPU/network constrained
   - Upgrade instance type for better performance
   - Increase storage if disk I/O is bottleneck

**Get best practices recommendations:**

```
Use tool: get_cluster_best_practices
Parameters:
  instance_type: "kafka.m5.large"
  number_of_brokers: 6
```

## Best Practices

### Cluster Design

- **Use at least 3 brokers** for production workloads (enables replication)
- **Distribute brokers across multiple AZs** for high availability
- **Size brokers appropriately** - start with m5.large or m5.xlarge for production
- **Plan for growth** - storage can be expanded but not reduced
- **Use provisioned throughput** for predictable I/O performance

### Security

- **Enable encryption in transit** (TLS) for all production clusters
- **Enable encryption at rest** using AWS KMS
- **Use IAM authentication** when possible for AWS-native applications
- **Implement least-privilege access** with IAM policies
- **Rotate SCRAM credentials** regularly if using username/password auth
- **Use private subnets** and security groups to restrict network access

### Configuration

- **Disable auto-topic creation** in production (`auto.create.topics.enable=false`)
- **Set appropriate replication factor** (3 for production)
- **Configure min.insync.replicas** (typically replication factor - 1)
- **Set retention policies** based on data requirements
- **Enable compression** (snappy or lz4) to reduce storage and network usage
- **Test configuration changes** in non-production environments first

### Monitoring

- **Enable enhanced monitoring** (PER_TOPIC_PER_BROKER level)
- **Set up CloudWatch alarms** for critical metrics
- **Monitor consumer lag** continuously
- **Track under-replicated partitions** (should always be 0)
- **Review broker metrics** regularly (CPU, network, disk)
- **Enable Prometheus exporters** for detailed metrics

### Operations

- **Always get current cluster version** before updates
- **Perform updates during maintenance windows** when possible
- **Monitor operations** until completion
- **Test scaling operations** in non-production first
- **Document cluster configurations** and changes
- **Implement backup strategies** for critical topics
- **Plan for disaster recovery** with cross-region replication if needed

### Cost Optimization

- **Right-size broker instances** based on actual usage
- **Use tiered storage** for older data (if available)
- **Set appropriate retention policies** to avoid unnecessary storage costs
- **Monitor and delete unused topics**
- **Consider reserved capacity** for predictable workloads
- **Review CloudWatch metrics retention** settings

## MCP Config Placeholders

**IMPORTANT:** Before using this power, replace the following placeholder in `mcp.json` with your actual value:

- **`YOUR_AWS_PROFILE_NAME`**: Your AWS CLI profile name that has access to MSK clusters.
  - **How to get it:**
    1. List your AWS profiles: `aws configure list-profiles`
    2. Choose the profile that has MSK access (e.g., "default", "amplifyuser", "production")
    3. Replace `YOUR_AWS_PROFILE_NAME` in mcp.json with your profile name
    4. Verify access: `aws kafka list-clusters --region us-east-1 --profile your-profile-name`

**After replacing the placeholder, your mcp.json should look like:**
```json
{
  "mcpServers": {
    "awslabs.aws-msk-mcp-server": {
      "command": "uvx",
      "args": [
        "awslabs.aws-msk-mcp-server@latest",
        "--allow-writes"
      ],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR",
        "AWS_PROFILE": "amplifyuser"
      },
      "disabled": false
    }
  }
}
```

## Configuration

### MCP Server Configuration

The AWS MSK MCP server should be configured in your Kiro MCP settings with:

```json
{
  "mcpServers": {
    "awslabs.aws-msk-mcp-server": {
      "command": "uvx",
      "args": [
        "awslabs.aws-msk-mcp-server@latest",
        "--allow-writes"
      ],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR",
        "AWS_PROFILE": "your-profile-name"
      },
      "disabled": false
    }
  }
}
```

**Configuration Options:**

- `--allow-writes`: Enables write operations (update, create, delete). Remove for read-only access.
- `AWS_PROFILE`: Your AWS CLI profile name
- `FASTMCP_LOG_LEVEL`: Logging level (ERROR, INFO, DEBUG)

### AWS Regions

The MCP server works with any AWS region where MSK is available. Specify the region in each tool call:

```
region: "us-east-1"
region: "eu-west-1"
region: "ap-southeast-1"
```

### Required IAM Permissions

Minimum permissions for read-only access:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "kafka:DescribeCluster",
      "kafka:DescribeClusterV2",
      "kafka:ListClusters",
      "kafka:ListClustersV2",
      "kafka:GetBootstrapBrokers",
      "kafka:ListNodes",
      "kafka:DescribeConfiguration",
      "kafka:ListConfigurations",
      "kafka:ListConfigurationRevisions",
      "kafka:DescribeClusterOperation",
      "kafka:ListClusterOperations",
      "kafka:ListTagsForResource",
      "cloudwatch:GetMetricData",
      "cloudwatch:ListMetrics"
    ],
    "Resource": "*"
  }]
}
```

Additional permissions for write operations (when using `--allow-writes`):

```json
{
  "Effect": "Allow",
  "Action": [
    "kafka:CreateCluster",
    "kafka:CreateClusterV2",
    "kafka:UpdateBrokerCount",
    "kafka:UpdateBrokerType",
    "kafka:UpdateBrokerStorage",
    "kafka:UpdateClusterConfiguration",
    "kafka:UpdateMonitoring",
    "kafka:UpdateSecurity",
    "kafka:CreateConfiguration",
    "kafka:UpdateConfiguration",
    "kafka:TagResource",
    "kafka:UntagResource",
    "kafka:RebootBroker",
    "kafka:PutClusterPolicy",
    "kafka:CreateVpcConnection",
    "kafka:DeleteVpcConnection"
  ],
  "Resource": "*"
}
```

---

**Package:** `awslabs.aws-msk-mcp-server`
**MCP Server:** awslabs.aws-msk-mcp-server
