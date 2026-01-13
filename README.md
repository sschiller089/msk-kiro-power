# AWS MSK Kiro Power

A comprehensive Kiro Power for managing and troubleshooting AWS MSK (Amazon Managed Streaming for Kafka) clusters. This power provides health checks, performance analysis, and operational guidance for developers working with Kafka on AWS.

## Features

- 🔍 **Cluster Health Monitoring** - Quick status checks, broker health, and performance metrics
- 🔌 **Connection Configuration** - Get bootstrap servers, authentication details, and client configs
- 🐛 **Troubleshooting** - Diagnose consumer lag, broker problems, and connectivity issues
- 📊 **Capacity Planning** - Analyze usage and make informed scaling decisions
- ⚙️ **Operational Tasks** - Update configurations, scale brokers, manage security settings

## Prerequisites

Before installing this power, ensure you have:

1. **AWS Account** - Valid AWS credentials with MSK access
2. **AWS CLI** - Installed and configured with appropriate profile
3. **Python & uv** - Required for running the MCP server
   - Install uv: https://docs.astral.sh/uv/getting-started/installation/
4. **IAM Permissions** - See [Required Permissions](#required-permissions) below

## Installation

### Step 1: Install the Power

1. Open Kiro and access the Powers panel:
   - Use Command Palette: Search for "Powers"
   - Or click the Powers icon in the sidebar

2. Click **"Add Custom Power"** button at the top

3. Select **"Local Directory"** option

4. Provide the full path to the power directory:
   ```
   /path/to/your/powers/aws-msk
   ```

5. Click **"Add"** to install

### Step 2: Configure AWS Profile

The power uses AWS credentials from your configured profile. You need to update the `mcp.json` file with your AWS profile name.

1. Open `powers/aws-msk/mcp.json`

2. Replace `YOUR_AWS_PROFILE_NAME` with your actual AWS profile:
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

3. Find your AWS profile name:
   ```bash
   aws configure list-profiles
   ```

4. Verify your profile has MSK access:
   ```bash
   aws kafka list-clusters --region us-east-1 --profile your-profile-name
   ```

### Step 3: Verify Installation

1. Restart Kiro or reconnect the MCP server from the MCP Server view

2. Test the power by asking Kiro:
   ```
   "List my MSK clusters in us-east-1"
   ```

3. If successful, you should see your cluster information

## Configuration Options

### Read-Only Mode

If you only need to view cluster information without making changes, remove the `--allow-writes` flag:

```json
"args": [
  "awslabs.aws-msk-mcp-server@latest"
]
```

### Logging Level

Adjust the logging level for debugging:

```json
"env": {
  "FASTMCP_LOG_LEVEL": "INFO"  // Options: ERROR, INFO, DEBUG
}
```

### Multiple Regions

The power works with any AWS region. Specify the region in each query:

```
"Show me clusters in eu-west-1"
```

## Required Permissions

### Minimum (Read-Only)

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

### Full Access (With --allow-writes)

Add these permissions for write operations:

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

## Usage Examples

### Get Cluster Overview

```
"Show me all my MSK clusters"
"What's the status of my MSK cluster?"
"Get details for cluster MSKCluster-production"
```

### Get Connection Details

```
"Get bootstrap servers for my MSK cluster"
"Show me the connection details for cluster XYZ"
"What authentication methods are enabled?"
```

### Monitor Performance

```
"Check CPU utilization for my MSK cluster"
"Show me the current metrics for cluster ABC"
"Is my cluster healthy?"
```

### Troubleshoot Issues

```
"My Kafka consumers are slow, help me diagnose"
"Check for under-replicated partitions"
"Why is my cluster slow?"
```

### Scale Cluster

```
"Add 3 more brokers to my cluster"
"Upgrade broker instance type to kafka.m5.xlarge"
"Increase storage to 3000 GB"
```

### Manage Configuration

```
"Show me the current cluster configuration"
"Update cluster configuration with new settings"
"What's the replication factor?"
```

## Common Workflows

### 1. Health Check Workflow

1. List clusters to get cluster ARN
2. Check cluster state (should be ACTIVE)
3. Get current metrics (CPU, memory, disk)
4. Review broker node status
5. Check for under-replicated partitions

### 2. Application Setup Workflow

1. Get cluster information
2. Retrieve bootstrap broker endpoints
3. Check authentication methods enabled
4. Get connection strings for your auth method
5. Configure your application with connection details

### 3. Performance Troubleshooting Workflow

1. Check broker CPU and memory utilization
2. Review network throughput metrics
3. Check consumer lag metrics
4. Analyze partition distribution
5. Get best practices recommendations
6. Scale if needed (add brokers, upgrade instance type)

### 4. Scaling Workflow

1. Get current cluster version
2. Review current capacity and metrics
3. Decide on scaling approach:
   - Add more brokers (horizontal scaling)
   - Upgrade instance type (vertical scaling)
   - Expand storage
4. Execute scaling operation
5. Monitor operation status until complete

## Troubleshooting

### MCP Server Won't Connect

**Error**: "Connection refused" or "Token has expired"

**Solution**:
```bash
# Verify AWS credentials
aws sts get-caller-identity --profile your-profile-name

# Refresh SSO token (if using AWS SSO)
aws sso login --profile your-profile-name

# Restart Kiro
```

### Permission Denied Errors

**Error**: "User is not authorized to perform: kafka:DescribeCluster"

**Solution**:
1. Verify your IAM permissions include required MSK actions
2. Check your AWS profile has the correct role/permissions
3. See [Required Permissions](#required-permissions) section

### Metrics Not Loading

**Error**: Telemetry queries fail or return empty results

**Solution**:
1. Ensure CloudWatch permissions are included
2. Check that enhanced monitoring is enabled on your cluster
3. Verify the time range is valid (not in the future)
4. Use correct metric names from available_metrics list

### Power Not Appearing

**Solution**:
1. Verify the power directory path is correct
2. Check that POWER.md and mcp.json files exist
3. Restart Kiro
4. Check MCP Server view for connection status

## Available Tools

The power provides 27 MCP tools organized into categories:

### Information Retrieval (9 tools)
- get_global_info
- get_cluster_info
- describe_cluster_operation
- describe_vpc_connection
- get_configuration_info
- list_tags_for_resource
- get_cluster_telemetry
- list_customer_iam_access
- get_cluster_best_practices

### Cluster Management (8 tools)
- create_cluster
- update_broker_storage
- update_broker_type
- update_cluster_configuration
- update_monitoring
- update_security
- put_cluster_policy
- update_broker_count

### Authentication & Security (2 tools)
- associate_scram_secret
- disassociate_scram_secret

### Operations (1 tool)
- reboot_broker

### Configuration Management (2 tools)
- create_configuration
- update_configuration

### Tagging (2 tools)
- tag_resource
- untag_resource

### VPC Connections (3 tools)
- create_vpc_connection
- delete_vpc_connection
- reject_client_vpc_connection

## Best Practices

### Security
- Use IAM authentication when possible for AWS-native applications
- Enable encryption in transit (TLS) for all production clusters
- Enable encryption at rest using AWS KMS
- Implement least-privilege access with IAM policies
- Use private subnets and security groups to restrict network access

### Monitoring
- Enable enhanced monitoring (PER_TOPIC_PER_BROKER level)
- Set up CloudWatch alarms for critical metrics
- Monitor consumer lag continuously
- Track under-replicated partitions (should always be 0)
- Review broker metrics regularly (CPU, network, disk)

### Operations
- Always get current cluster version before updates
- Perform updates during maintenance windows when possible
- Monitor operations until completion
- Test scaling operations in non-production first
- Document cluster configurations and changes

### Performance
- Keep CPU utilization below 60% regularly
- Monitor disk usage (warning at 85%, critical at 90%)
- Set appropriate replication factor (3 for production)
- Configure min.insync.replicas (typically replication factor - 1)
- Enable compression (snappy or lz4) to reduce storage and network usage

## Support & Resources

- **Official AWS MSK Documentation**: https://docs.aws.amazon.com/msk/
- **MCP Server Repository**: https://github.com/awslabs/aws-msk-mcp-server
- **Kiro Documentation**: https://kiro.dev/docs/
- **Apache Kafka Documentation**: https://kafka.apache.org/documentation/

## Contributing

Found an issue or want to improve this power? Contributions are welcome!

1. Fork the repository
2. Make your changes
3. Test thoroughly
4. Submit a pull request

## License

This power uses the AWS MSK MCP Server which is licensed under the Apache License 2.0.

## Version

**Current Version**: 1.0.0
**Last Updated**: January 2026
**MCP Server**: awslabs.aws-msk-mcp-server@latest
