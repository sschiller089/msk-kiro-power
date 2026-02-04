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

- **POWER.md** - Main power documentation with:
  - Express broker instance limits and thresholds
  - Key metrics reference
  - MCP tools quick reference

- **steering/** - Detailed workflow guides:
  - `getting-started.md` - Prerequisites, cluster discovery, initial setup
  - `monitoring-best-practices.md` - Health monitoring, CPU thresholds, partition management
  - `troubleshooting.md` - Diagnose slow clients, high CPU, replication lag, auth failures

- **mcp.json** - AWS MSK MCP Server configuration

## Installation

### From GitHub

1. Open Kiro Powers UI (use command palette: "Open Powers Panel")
2. Click "Add Custom Power"
3. Select "GitHub Repository"
4. Enter: `https://github.com/sschiller089/msk-kiro-power`
5. Click "Add"

### Local Testing

1. Clone this repository
2. Open Kiro Powers UI
3. Click "Add Custom Power"
4. Select "Local Directory"
5. Enter the full path to the cloned directory
6. Click "Add"

### Configuration

Before using the power, update the AWS profile in `mcp.json`:

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

## Next Steps

1. Install the power using the instructions above
2. Configure your AWS profile in `mcp.json`
3. Ask Kiro to help you monitor or troubleshoot your MSK Express Broker cluster

