---
inclusion: manual
---

# Getting Started with MSK Express Broker

This guide walks you through verifying prerequisites and retrieving cluster information for your Amazon MSK Express Broker cluster.

## Prerequisites Checklist

Before proceeding, ensure you have:

**AWS Setup:**
- AWS account with MSK permissions (describe, list operations)
- AWS CLI configured with appropriate credentials
- IAM permissions for MSK read operations

**Local Environment:**
- Python 3.10+ installed
- `uv` package manager installed
- AWS credentials configured (via `~/.aws/credentials` or environment variables)

**MCP Server Configuration:**
- Verify `mcp.json` contains the AWS MSK MCP Server configuration
- Ensure `AWS_PROFILE` environment variable is set correctly

## Step 1: Verify MCP Server Connection

Test connectivity by listing your MSK clusters in the target region.

**Action:** Use `get_global_info` tool
```json
{
  "region": "us-east-1",
  "info_type": "clusters"
}
```

This returns all MSK clusters in the specified region. Look for clusters with `ClusterType: "EXPRESS"`.

## Step 2: Identify Your Express Broker Cluster

From the cluster list, identify your Express Broker cluster by:
- Cluster name
- Cluster ARN
- Cluster type must be `EXPRESS`

**Important:** This power is specifically designed for MSK Express Brokers. Provisioned clusters have different characteristics and limits.

## Step 3: Get Cluster Metadata

Retrieve detailed information about your Express Broker cluster.

**Action:** Use `get_cluster_info` tool
```json
{
  "region": "us-east-1",
  "cluster_arn": "<your-cluster-arn>",
  "info_type": "metadata"
}
```

**Key information to note:**
- `State`: Should be `ACTIVE` for operational clusters
- `CurrentVersion`: Required for any update operations
- `Serverless.VpcConfigs`: VPC and subnet configuration
- `Serverless.ClientAuthentication`: Authentication method (IAM, SASL/SCRAM)

## Step 4: Get Broker Information

Retrieve broker endpoints and connection details.

**Action:** Use `get_cluster_info` tool
```json
{
  "region": "us-east-1",
  "cluster_arn": "<your-cluster-arn>",
  "info_type": "brokers"
}
```

**Key information to note:**
- Bootstrap broker endpoints for client connections
- Number of brokers in the cluster

## Step 5: Verify Cluster Health (Quick Check)

Perform a quick health check by retrieving key metrics.

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

**Note:** Replace `<1-hour-ago-ISO8601>` and `<now-ISO8601>` with actual ISO 8601 timestamps (e.g., `2026-02-04T14:00:00Z`).

If metrics return successfully, your cluster is operational and ready for monitoring.

## Express Broker Instance Sizes Reference

| Instance Size | Max Ingress (MBps) | Max Egress (MBps) |
|---------------|-------------------|-------------------|
| express.m7g.large | 15.6 | 31.2 |
| express.m7g.xlarge | 31.2 | 62.5 |
| express.m7g.2xlarge | 62.5 | 125.0 |
| express.m7g.4xlarge | 124.9 | 249.8 |
| express.m7g.8xlarge | 250.0 | 500.0 |
| express.m7g.12xlarge | 375.0 | 750.0 |
| express.m7g.16xlarge | 500.0 | 1000.0 |

## Next Steps

Once you've verified your cluster setup:
- **Monitor cluster health** → See `monitoring-best-practices.md`
- **Troubleshoot issues** → See `troubleshooting.md`
