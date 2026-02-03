# AWS MSK Express Broker Power

A Kiro Power for building and operating Amazon MSK Express Brokers with best practices and troubleshooting guidance.

## About Express Brokers

Amazon MSK Express Brokers are a newer cluster type designed for simplified operations and automatic scaling. Key characteristics:

- **KRaft-based:** No ZooKeeper dependency, using Kafka's native consensus protocol
- **Pre-configured HA:** Data distributed across 3 AZs with replication factor of 3 and min.insync.replicas of 2
- **Automatic scaling:** Storage scales automatically based on throughput
- **Instance types:** Use `express.m7g.*` instances (large, xlarge, 2xlarge, 4xlarge, 8xlarge, 12xlarge, 16xlarge)
- **Simplified management:** Reduced operational overhead compared to provisioned clusters

Express brokers are ideal for workloads that benefit from automatic scaling and simplified operations, while provisioned clusters offer more granular control over broker configuration.

## What's Included

- **POWER.md** - Comprehensive guide covering:
  - Express broker best practices (sizing, CPU management, connections)
  - Common workflows (create cluster, monitor health, update configuration)
  - Troubleshooting guides (slow clients, replication lag, high CPU, authentication issues)
  - MCP tools reference for AWS MSK operations

- **mcp.json** - AWS MSK MCP Server configuration

## Installation

### Local Testing

1. Open Kiro Powers UI (use command palette: "Open Powers Panel")
2. Click "Add Custom Power"
3. Select "Local Directory"
4. Enter the full path to this directory: `{workspace}/powers/aws-msk-express-broker`
5. Click "Add"

### Configuration

Before using the power, update the AWS profile placeholder in `mcp.json`:

```json
{
  "env": {
    "AWS_PROFILE": "your-actual-aws-profile-name"
  }
}
```

## Usage

Once installed, the power provides:

- **Best practices** for Express broker sizing, CPU management, and connection handling
- **Workflows** for creating clusters, monitoring health, and updating configurations
- **Troubleshooting** for common issues like slow clients, replication lag, and high CPU
- **MCP tools** for interacting with AWS MSK clusters

## Included Hooks

This power includes a Kiro hook for automated health monitoring:

**MSK Post-Cluster Health Check** (`.kiro/hooks/msk-post-cluster-health-check.json`)

Automatically prompts a health check after MSK cluster operations complete. When Kiro detects cluster creation or activation patterns in the conversation, it offers to:

1. Check cluster status and metadata
2. Verify broker endpoints are accessible
3. Review current CPU and throughput metrics
4. Confirm monitoring is properly configured

This helps ensure new clusters are healthy before you start using them.

## Next Steps

1. Test the power locally using the installation steps above
2. Verify the documentation is clear and complete
3. Test the MCP tools with your AWS account
4. Share the power via GitHub repository or submit to Kiro recommended powers

