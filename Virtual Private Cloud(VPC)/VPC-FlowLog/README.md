# VPC Flow Logs

## Overview

**VPC Flow Logs** is an AWS feature that captures information about the IP traffic flowing to and from network interfaces in your Virtual Private Cloud (VPC). Flow log data can be published to **Amazon CloudWatch Logs**, **Amazon S3**, or **Amazon Kinesis Data Firehose** for storage, analysis, and monitoring.

Flow logs help you:
- Diagnose overly restrictive or permissive security group rules
- Monitor the traffic reaching your instances
- Investigate network anomalies or security incidents
- Meet compliance and audit requirements

---

## Key Concepts

| Term | Description |
|------|-------------|
| **Flow Log** | A record of network traffic for a VPC, subnet, or ENI |
| **ENI** | Elastic Network Interface — the primary target of flow log capture |
| **Log Record** | A single line of space-separated fields representing a traffic flow |
| **Log Group** | CloudWatch container for flow log streams |
| **Aggregation Interval** | Time window for capturing traffic (1 min or 10 min) |

---

## Flow Log Levels

VPC Flow Logs can be enabled at three levels:

```
VPC Level       → Captures traffic for ALL subnets and ENIs in the VPC
Subnet Level    → Captures traffic for ALL ENIs in a specific subnet
ENI Level       → Captures traffic for a single network interface
```

---

## Default Log Record Format

Each flow log record contains the following fields (default format):

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

### Example Log Entry

```
2 123456789010 eni-0a1b2c3d4e5f 10.0.1.5 10.0.2.10 54321 443 6 20 4000 1620000000 1620000060 ACCEPT OK
```

| Field | Value | Description |
|-------|-------|-------------|
| version | `2` | Flow log version |
| account-id | `123456789010` | AWS account ID |
| interface-id | `eni-0a1b2c3d4e5f` | ENI ID |
| srcaddr | `10.0.1.5` | Source IP |
| dstaddr | `10.0.2.10` | Destination IP |
| srcport | `54321` | Source port |
| dstport | `443` | Destination port |
| protocol | `6` | Protocol number (6 = TCP) |
| packets | `20` | Packets transferred |
| bytes | `4000` | Bytes transferred |
| action | `ACCEPT` | ACCEPT or REJECT |
| log-status | `OK` | Logging status |

---

## Custom Log Format Fields

Beyond the default fields, you can include additional fields:

| Field | Description |
|-------|-------------|
| `vpc-id` | VPC ID |
| `subnet-id` | Subnet ID |
| `instance-id` | EC2 instance ID |
| `tcp-flags` | TCP flag bitmask |
| `type` | Traffic type (IPv4, IPv6, EFA) |
| `pkt-srcaddr` | Original source IP (for NATted traffic) |
| `pkt-dstaddr` | Original destination IP |
| `region` | AWS region |
| `az-id` | Availability Zone ID |
| `flow-direction` | `ingress` or `egress` |
| `traffic-path` | Path of egress traffic |

---

## Destination Options

| Destination | Use Case |
|------------|----------|
| **CloudWatch Logs** | Real-time monitoring, metric filters, alarms |
| **S3** | Long-term storage, Athena queries, cost-effective archival |
| **Kinesis Data Firehose** | Real-time streaming to third-party analytics tools |

---

## IAM Requirements

Flow logs require an IAM role with permissions to publish logs. The trust policy must allow the `vpc-flow-logs.amazonaws.com` service principal.

### Minimum IAM Policy (for CloudWatch)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

### Minimum IAM Policy (for S3)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::your-bucket-name/AWSLogs/*"
    }
  ]
}
```

---

## Limitations

- Flow logs **do not capture** real-time streaming — there is a delay of several minutes.
- Traffic to/from the following is **not logged**:
  - AWS DNS server (169.254.169.253)
  - DHCP traffic
  - Instance metadata service (169.254.169.254)
  - Amazon Windows license activation
  - Traffic to the default VPC router
- Flow logs **cannot be edited** after creation — delete and recreate to change settings.
- Flow logs do **not affect network throughput or latency**.

---

## Pricing

| Component | Cost |
|-----------|------|
| Log ingestion (CloudWatch) | Per GB ingested |
| Log storage (CloudWatch) | Per GB per month |
| Log storage (S3) | Standard S3 storage rates |
| Data transfer | Standard AWS data transfer rates |

> ℹ️ Enabling flow logs at the VPC level on large/busy environments can generate significant log volume — estimate costs before enabling in production.

---

## Common Use Cases

### 1. Security Auditing
Query logs to detect unexpected REJECT actions or traffic from unknown IPs.

### 2. Troubleshooting Connectivity
Confirm whether traffic is being accepted or rejected by security groups / NACLs.

### 3. Traffic Baseline & Anomaly Detection
Use CloudWatch Metric Filters or Athena to establish baselines and detect spikes.

### 4. Compliance
Retain logs in S3 with lifecycle policies to satisfy audit requirements (PCI DSS, HIPAA, etc.).

---

## Useful AWS CLI Commands

```bash
# Create a flow log to S3
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxxxxxxx \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::your-bucket/flow-logs/

# Create a flow log to CloudWatch Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxxxxxxx \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789010:role/FlowLogsRole

# Describe existing flow logs
aws ec2 describe-flow-logs

# Delete a flow log
aws ec2 delete-flow-logs --flow-log-ids fl-xxxxxxxx
```

---

## References

- [AWS Docs: VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [Flow Log Record Examples](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-records-examples.html)
- [Querying Flow Logs with Athena](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-athena.html)
- [CloudWatch Logs Insights for Flow Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)


# Lab Note: Setting Up VPC Flow Logs at the VPC Level

**Lab Type:** Hands-on AWS Console + CLI  
**Difficulty:** Beginner–Intermediate  
**Estimated Time:** 30–45 minutes  
**Prerequisites:** AWS account with admin or power-user permissions, an existing VPC

---

## Objective

Enable VPC Flow Logs at the **VPC level** to capture ALL inbound and outbound traffic across every subnet and ENI in the VPC. Logs will be delivered to both **Amazon S3** and optionally **CloudWatch Logs**.

---

## Architecture Overview

```
┌──────────────────────────────────────────┐
│                  VPC                     │
│  ┌────────────┐    ┌────────────────┐    │
│  │  Subnet A  │    │    Subnet B    │    │
│  │  [EC2 ENI] │    │  [EC2 ENI]    │    │
│  └────────────┘    └────────────────┘    │
│         │                  │             │
│         └──────┬───────────┘             │
│           Flow Logs (ALL traffic)        │
└───────────────────────────────────────── ┘
                  │
     ┌────────────┴────────────┐
     ▼                         ▼
Amazon S3 Bucket         CloudWatch Logs
(long-term storage)      (real-time alerts)
```

---

## Step 1 — Create an S3 Bucket for Flow Logs

> Skip if you already have a dedicated S3 bucket.

### Console

1. Navigate to **S3** → **Create bucket**
2. Set **Bucket name**: `vpc-flowlogs-<your-account-id>` (must be globally unique)
3. Set **Region**: same region as your VPC
4. Leave **Block all public access** enabled (default)
5. Click **Create bucket**

### CLI

```bash
aws s3api create-bucket \
  --bucket vpc-flowlogs-123456789010 \
  --region us-east-1
```

---

## Step 2 — (Optional) Create an IAM Role for CloudWatch Delivery

Only needed if you want logs sent to **CloudWatch Logs** in addition to S3.

### 2a. Create the Trust Policy file

```bash
cat > flow-logs-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

### 2b. Create the IAM Role

```bash
aws iam create-role \
  --role-name VPCFlowLogsRole \
  --assume-role-policy-document file://flow-logs-trust-policy.json
```

### 2c. Attach CloudWatch Permissions

```bash
cat > flow-logs-cw-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name VPCFlowLogsRole \
  --policy-name VPCFlowLogsCWPolicy \
  --policy-document file://flow-logs-cw-policy.json
```

> 📝 **Note down the Role ARN** — you'll need it in Step 3.  
> Run: `aws iam get-role --role-name VPCFlowLogsRole --query Role.Arn --output text`

---

## Step 3 — Enable Flow Logs at the VPC Level

### Option A: Console (Recommended for first-time setup)

1. Navigate to **VPC** → **Your VPCs**
2. Select the target VPC
3. Click **Actions** → **Create flow log**
4. Fill in the form:

| Field | Value |
|-------|-------|
| **Filter** | `All` (captures ACCEPT and REJECT) |
| **Maximum aggregation interval** | `1 minute` (for finer granularity) or `10 minutes` (default, lower cost) |
| **Destination** | `Send to an Amazon S3 bucket` |
| **S3 bucket ARN** | `arn:aws:s3:::vpc-flowlogs-123456789010` |
| **Log record format** | `AWS default format` (or custom — see below) |

5. Click **Create flow log**

> ✅ A green banner confirms the flow log was created. Status will show **Active**.

---

### Option B: CLI — S3 Destination

```bash
# Replace vpc-xxxxxxxx with your actual VPC ID
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxxxxxxx \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::vpc-flowlogs-123456789010 \
  --max-aggregation-interval 60
```

---

### Option C: CLI — CloudWatch Logs Destination

```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxxxxxxx \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789010:role/VPCFlowLogsRole \
  --max-aggregation-interval 60
```

---

### Option D: CLI — Custom Log Format

```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxxxxxxx \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::vpc-flowlogs-123456789010 \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status} ${vpc-id} ${subnet-id} ${instance-id} ${flow-direction}'
```

> 💡 Custom format adds fields like `vpc-id`, `subnet-id`, `instance-id`, and `flow-direction` which are not in the default format.

---

## Step 4 — Verify the Flow Log is Active

### Console

1. Go to **VPC** → **Your VPCs** → select your VPC
2. Click the **Flow logs** tab at the bottom
3. Confirm **Status** = `Active`

### CLI

```bash
aws ec2 describe-flow-logs \
  --filter "Name=resource-id,Values=vpc-xxxxxxxx" \
  --query 'FlowLogs[*].{ID:FlowLogId,Status:FlowLogStatus,Destination:LogDestination}' \
  --output table
```

Expected output:

```
-----------------------------------------------------------------------
|                         DescribeFlowLogs                            |
+-----------------+-----------+---------------------------------------+
|  ID             |  Status   |  Destination                          |
+-----------------+-----------+---------------------------------------+
|  fl-0abc123def  |  ACTIVE   |  arn:aws:s3:::vpc-flowlogs-...        |
+-----------------+-----------+---------------------------------------+
```

---

## Step 5 — Generate Test Traffic

To confirm logs are being captured, generate some traffic in your VPC.

```bash
# SSH into an EC2 instance in the VPC, then:
curl https://www.example.com         # generates outbound HTTPS traffic
ping 10.0.2.10                       # generates ICMP traffic to another instance
nc -zv 10.0.2.10 22                  # tests port 22 reachability
```

> ⏳ Wait **5–15 minutes** for the first logs to appear in S3 or CloudWatch.

---

## Step 6 — View Logs

### S3 Console

1. Navigate to **S3** → `vpc-flowlogs-123456789010`
2. Browse path: `AWSLogs/<account-id>/vpcflowlogs/<region>/<year>/<month>/<day>/`
3. Download and open a `.log.gz` file to inspect raw records

### CloudWatch Logs Console

1. Navigate to **CloudWatch** → **Log groups** → `/aws/vpc/flowlogs`
2. Click a log stream (named after the ENI ID)
3. Browse flow log entries in the event viewer

### CLI — Tail CloudWatch Logs

```bash
aws logs tail /aws/vpc/flowlogs --follow
```

---

## Step 7 — (Optional) Query Logs with Athena

If using S3, you can query logs with Athena for powerful analysis.

### Create Athena Table

```sql
CREATE EXTERNAL TABLE vpc_flow_logs (
  version int,
  account string,
  interfaceid string,
  sourceaddress string,
  destinationaddress string,
  sourceport int,
  destinationport int,
  protocol int,
  numpackets int,
  numbytes bigint,
  starttime int,
  endtime int,
  action string,
  logstatus string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ' '
LOCATION 's3://vpc-flowlogs-123456789010/AWSLogs/123456789010/vpcflowlogs/us-east-1/'
TBLPROPERTIES ("skip.header.line.count"="1");
```

### Sample Queries

```sql
-- Top source IPs by bytes
SELECT sourceaddress, SUM(numbytes) AS total_bytes
FROM vpc_flow_logs
WHERE action = 'ACCEPT'
GROUP BY sourceaddress
ORDER BY total_bytes DESC
LIMIT 10;

-- All REJECT events
SELECT * FROM vpc_flow_logs
WHERE action = 'REJECT'
ORDER BY starttime DESC
LIMIT 100;

-- Traffic to a specific port
SELECT sourceaddress, destinationaddress, destinationport, action
FROM vpc_flow_logs
WHERE destinationport = 22
ORDER BY starttime DESC;
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Flow log status is `ERROR` | Incorrect IAM role or S3 bucket policy | Check IAM role trust policy and permissions |
| No logs appearing in S3 | Logs take 5–15 min to appear initially | Wait and retry |
| S3 access denied error | Bucket policy blocking delivery | Add `s3:PutObject` permission for `delivery.logs.amazonaws.com` |
| Logs missing for certain ENIs | ENI in a different subnet not covered | Enable flow log at VPC (not subnet) level |
| Log file is empty | No traffic in aggregation window | Generate traffic, then wait one interval |

### S3 Bucket Policy Fix (if access denied)

Add this statement to your S3 bucket policy:

```json
{
  "Sid": "AWSLogDeliveryWrite",
  "Effect": "Allow",
  "Principal": {
    "Service": "delivery.logs.amazonaws.com"
  },
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::vpc-flowlogs-123456789010/AWSLogs/*",
  "Condition": {
    "StringEquals": {
      "s3:x-amz-acl": "bucket-owner-full-control"
    }
  }
}
```

---

## Cleanup

When finished, remove resources to avoid ongoing charges:

```bash
# 1. Delete the flow log
aws ec2 delete-flow-logs --flow-log-ids fl-xxxxxxxx

# 2. Delete CloudWatch log group (if created)
aws logs delete-log-group --log-group-name /aws/vpc/flowlogs

# 3. Empty and delete the S3 bucket (if created for this lab)
aws s3 rm s3://vpc-flowlogs-123456789010 --recursive
aws s3api delete-bucket --bucket vpc-flowlogs-123456789010

# 4. Delete IAM role (if created for this lab)
aws iam delete-role-policy --role-name VPCFlowLogsRole --policy-name VPCFlowLogsCWPolicy
aws iam delete-role --role-name VPCFlowLogsRole
```

---

## Summary

| Step | Action |
|------|--------|
| ✅ 1 | Created S3 bucket for log destination |
| ✅ 2 | Created IAM role for CloudWatch delivery (optional) |
| ✅ 3 | Enabled VPC Flow Log at the VPC level (all traffic) |
| ✅ 4 | Verified flow log is Active |
| ✅ 5 | Generated test traffic |
| ✅ 6 | Viewed and inspected log records |
| ✅ 7 | Queried logs with Athena (optional) |

> 🎉 **Lab complete!** You now have full visibility into all network traffic in your VPC.
