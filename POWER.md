---
name: "aws-msk-express-broker"
displayName: "Build and Operate MSK Express Broker"
description: "Guide developers through creating and operating Amazon MSK Express Brokers with best practices, monitoring strategies, and troubleshooting workflows."
keywords: ["aws", "msk", "kafka", "express", "broker", "troubleshooting", "monitoring", "cloudwatch"]
author: "Stephan Schiller"
---

# Build and Operate MSK Express Broker

## Prerequisites

Before using this power, set your AWS profile as an environment variable:

```bash
export AWS_PROFILE=your-profile-name
```

Add this to your shell profile (`~/.zshrc` or `~/.bashrc`) for persistence. When Kiro first loads the power, it will prompt you to approve the `AWS_PROFILE` environment variable expansion.

Required IAM permissions: `kafka:Describe*`, `kafka:List*`, `cloudwatch:GetMetricData`

## Overview

This power guides you through building and operating Amazon MSK Express Brokers according to AWS best practices. Express brokers are pre-configured for high availability and durability with data distributed across three availability zones, replication set to 3, and minimum in-sync replicas set to 2.

## Steering Files

This power includes detailed workflow guides:

| Guide | Use Case |
|-------|----------|
| `getting-started.md` | Prerequisites, cluster discovery, initial setup verification |
| `monitoring-best-practices.md` | Health monitoring, CPU thresholds, partition management |
| `troubleshooting.md` | Diagnose slow clients, high CPU, replication lag, auth failures |

## Quick Reference

### Express Broker Instance Limits

| Instance Size | Max Ingress (MBps) | Max Egress (MBps) |
|---------------|-------------------|-------------------|
| express.m7g.large | 15.6 | 31.2 |
| express.m7g.xlarge | 31.2 | 62.5 |
| express.m7g.2xlarge | 62.5 | 125.0 |
| express.m7g.4xlarge | 124.9 | 249.8 |
| express.m7g.8xlarge | 250.0 | 500.0 |
| express.m7g.12xlarge | 375.0 | 750.0 |
| express.m7g.16xlarge | 500.0 | 1000.0 |

### Key Thresholds

| Metric | Target | Reason |
|--------|--------|--------|
| Total CPU | < 60% | Reserve 40% for failover/maintenance |
| IAM connections | < 3000/broker | Hard limit |
| IAM connection rate | < 100/sec/broker | Rate limit |
| UnderReplicatedPartitions | 0 | Data durability |

### Critical Metrics

**Performance:** `CpuUser`, `CpuSystem`, `BytesInPerSec`, `BytesOutPerSec`

**Health:** `UnderReplicatedPartitions`, `GlobalPartitionCount`, `PartitionCount`

**Connections:** `ClientConnectionCount`, `ConnectionCreationRate`

**Consumer:** `MaxOffsetLag`, `EstimatedMaxTimeLag`

## MCP Tools Reference

### Cluster Information

**get_global_info** - List clusters in a region
```json
{"region": "us-east-1", "info_type": "clusters"}
```

**get_cluster_info** - Get cluster details
```json
{"region": "us-east-1", "cluster_arn": "<arn>", "info_type": "metadata"}
```
Info types: `metadata`, `brokers`, `nodes`, `operations`, `all`

### Monitoring

**get_cluster_telemetry** - Retrieve CloudWatch metrics
```json
{
  "region": "us-east-1",
  "action": "metrics",
  "cluster_arn": "<arn>",
  "kwargs": {
    "metrics": ["CpuUser", "CpuSystem"],
    "start_time": "2026-02-04T14:00:00Z",
    "end_time": "2026-02-04T15:00:00Z",
    "period": 300
  }
}
```

**Note:** `start_time` and `end_time` are required. Use ISO 8601 format. The `metrics` parameter accepts an array of metric names.

**get_cluster_best_practices** - Get sizing recommendations
```json
{"instance_type": "express.m7g.large", "number_of_brokers": 3}
```

### Cluster Operations

**create_cluster** - Create Express broker cluster
```json
{
  "region": "us-east-1",
  "cluster_name": "my-cluster",
  "cluster_type": "SERVERLESS",
  "kwargs": {
    "serverless": {
      "vpcConfigs": [{"subnetIds": ["subnet-xxx"], "securityGroupIds": ["sg-xxx"]}],
      "clientAuthentication": {"sasl": {"iam": {"enabled": true}}}
    }
  }
}
```

**update_monitoring** - Update monitoring settings
```json
{
  "region": "us-east-1",
  "cluster_arn": "<arn>",
  "current_version": "<version>",
  "enhanced_monitoring": "PER_BROKER"
}
```

**reboot_broker** - Reboot specific brokers
```json
{"region": "us-east-1", "cluster_arn": "<arn>", "broker_ids": ["1"]}
```

## Additional Resources

- [AWS MSK Express Best Practices](https://docs.aws.amazon.com/msk/latest/developerguide/bestpractices-express.html)
- [AWS MSK Troubleshooting Guide](https://docs.aws.amazon.com/msk/latest/developerguide/troubleshooting.html)
- [AWS MSK Express Metrics](https://docs.aws.amazon.com/msk/latest/developerguide/metrics-details-express.html)
- [AWS MSK MCP Server Documentation](https://awslabs.github.io/mcp/servers/aws-msk-mcp-server)

---

**MCP Server:** awslabs.aws-msk-mcp-server
**Installation:** `uvx awslabs.aws-msk-mcp-server@latest --allow-writes`
