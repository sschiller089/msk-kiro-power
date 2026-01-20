# AWS MSK Express Broker Power

A Kiro Power for building and operating Amazon MSK Express Brokers with best practices and troubleshooting guidance.

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

## Next Steps

1. Test the power locally using the installation steps above
2. Verify the documentation is clear and complete
3. Test the MCP tools with your AWS account
4. Share the power via GitHub repository or submit to Kiro recommended powers

