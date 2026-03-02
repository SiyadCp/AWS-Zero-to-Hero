<div align="center">

![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![VPC](https://img.shields.io/badge/VPC-Networking-blue?style=for-the-badge)
![PrivateLink](https://img.shields.io/badge/PrivateLink-Secure-green?style=for-the-badge)
![S3](https://img.shields.io/badge/S3-Storage-569A31?style=for-the-badge&logo=amazon-s3&logoColor=white)

# AWS VPC Endpoints — In-Depth Guide

### 🔒 Access AWS services privately — without touching the public internet.

</div>

---

## 📋 Table of Contents

- [What is a VPC Endpoint?](#-what-is-a-vpc-endpoint)
- [Why VPC Endpoints?](#-why-vpc-endpoints--the-problem-they-solve)
- [Types of VPC Endpoints](#-types-of-vpc-endpoints)
  - [Gateway Endpoint](#1-gateway-endpoint)
  - [Interface Endpoint](#2-interface-endpoint-aws-privatelink)
  - [Gateway Load Balancer Endpoint](#3-gateway-load-balancer-endpoint)
- [Gateway vs Interface Comparison](#-gateway-vs-interface-endpoint-comparison)
- [How VPC Endpoints Work Internally](#-how-vpc-endpoints-work-internally)
- [Endpoint Policies](#-endpoint-policies)
- [Lab 1 — Gateway Endpoint for S3](#-lab-1--gateway-endpoint-for-s3)
- [Lab 2 — Interface Endpoint for SSM](#-lab-2--interface-endpoint-for-ssm)
- [Troubleshooting](#-troubleshooting)
- [Pricing Overview](#-pricing-overview)
- [Best Practices](#-best-practices)
- [Summary](#-summary)

---

## 💡 What is a VPC Endpoint?

A **VPC Endpoint** allows resources inside your VPC to **privately connect to supported AWS services** without requiring:

- An Internet Gateway
- A NAT Gateway
- A VPN connection
- AWS Direct Connect

Traffic between your VPC and the AWS service travels entirely **within the AWS network backbone** — it never leaves AWS infrastructure and never hits the public internet.

```
                    ❌ WITHOUT VPC ENDPOINT

  EC2 Instance
      │
      │  Private IP traffic
      ▼
  NAT Gateway  ──▶  Internet Gateway  ──▶  🌐 Public Internet  ──▶  S3 / SSM / etc.
  (costs money)     (public route)          (leaves AWS network)

```

```
                    ✅ WITH VPC ENDPOINT

  EC2 Instance
      │
      │  Private IP traffic
      ▼
  VPC Endpoint  ──▶  AWS Private Network  ──▶  S3 / SSM / etc.
  (no NAT needed)    (never leaves AWS)
```

---

## 🔥 Why VPC Endpoints? — The Problem They Solve

### 1. Security — Keep Traffic Off the Internet

In a properly secured architecture, EC2 instances in **private subnets** have no internet access. But they often still need to reach AWS services like S3, DynamoDB, SSM, Secrets Manager, and ECR.

Without a VPC endpoint, the only way to reach these services from a private subnet is through a **NAT Gateway** — which routes traffic out through an Internet Gateway, exposing it to the public internet. This creates unnecessary attack surface.

VPC Endpoints eliminate this by routing traffic directly from your VPC to the AWS service through AWS's private network.

### 2. Cost — Eliminate NAT Gateway Data Processing Fees

NAT Gateway charges **$0.045 per GB** of data processed. In workloads that transfer large volumes of data to S3 or other AWS services, this adds up quickly.

```
  Example: EC2 transfers 10 TB/month to S3

  ❌ Via NAT Gateway:
     10,000 GB × $0.045 = $450/month in NAT data processing fees
     + S3 data transfer costs

  ✅ Via S3 Gateway Endpoint:
     $0/month in data processing fees (Gateway endpoints are free)
     + S3 data transfer costs only
```

### 3. Compliance — Data Sovereignty and Network Isolation

Many compliance frameworks (PCI-DSS, HIPAA, SOC2) require that sensitive data never traverse the public internet. VPC Endpoints provide a straightforward way to enforce this network boundary.

---

## 🗂️ Types of VPC Endpoints

AWS offers three types of VPC endpoints, each suited to different use cases.

---

### 1. Gateway Endpoint

A **Gateway Endpoint** is a route table entry that directs traffic destined for a specific AWS service to the endpoint rather than through an Internet Gateway or NAT.

```
VPC Route Table (with Gateway Endpoint):

Destination           Target
──────────────────    ─────────────────────
10.0.0.0/16           local
0.0.0.0/0             nat-gateway-id
pl-xxxxxxxx (S3)      vpce-xxxxxxxxx       ← Gateway Endpoint route
```

**Supported services (only two):**
- **Amazon S3**
- **Amazon DynamoDB**

**Key characteristics:**
- Free — no hourly charge, no data processing charge
- Works by injecting a **prefix list** route into your VPC route tables
- Does **not** use an ENI — no IP address is assigned inside your VPC
- Does **not** support access from on-premises (VPN/Direct Connect)
- Does **not** support cross-region access
- Controlled via **Endpoint Policies** (IAM-style JSON)

---

### 2. Interface Endpoint (AWS PrivateLink)

An **Interface Endpoint** provisions one or more **Elastic Network Interfaces (ENIs)** with private IP addresses inside your VPC subnets. Traffic to the AWS service is routed to this ENI.

```
VPC Subnet (10.0.1.0/24)
┌─────────────────────────────────────────┐
│  EC2 Instance          Interface ENI    │
│  10.0.1.10   ─────▶   10.0.1.50        │
│                         (ssm.us-east-1  │
│                          .vpce.aws.com) │
└─────────────────────────────────────────┘
         │
         │  Private IP traffic via ENI
         ▼
   AWS PrivateLink Network
         │
         ▼
   AWS Service (SSM, ECR, Secrets Manager, etc.)
```

**Supported services:** 100+ AWS services including:

| Category | Services |
|----------|---------|
| **Management** | SSM, Secrets Manager, KMS, CloudWatch, CloudTrail |
| **Containers** | ECR (API + DKR), ECS |
| **Compute** | EC2 API, Auto Scaling, Lambda |
| **Storage** | S3 (interface variant), EFS, Backup |
| **Security** | STS, IAM (limited), GuardDuty, Security Hub |
| **Data** | Kinesis, SQS, SNS, Glue, Athena |
| **Developer** | CodeBuild, CodeDeploy, CodePipeline |

**Key characteristics:**
- Creates **private DNS** entries — existing SDK/CLI calls work automatically
- Supports access **from on-premises** via VPN or Direct Connect
- Supports **cross-region** access (with DNS)
- Has **hourly billing** per AZ plus data processing charges
- Can be shared across accounts via **PrivateLink**

---

### 3. Gateway Load Balancer Endpoint

A **Gateway Load Balancer (GWLB) Endpoint** is used to route traffic to **third-party virtual appliances** — firewalls, intrusion detection systems (IDS), deep packet inspection tools — transparently.

```
  Traffic flow with GWLB Endpoint (Inspection VPC pattern):

  EC2 ──▶ GWLB Endpoint ──▶ Gateway Load Balancer ──▶ Firewall Appliance
                              (in Inspection VPC)        (Palo Alto, Fortinet)
                                                              │
                                                              │ Inspected
                                                              ▼
                                                        Original Destination
```

This is an advanced networking pattern — primarily used by security teams implementing centralized traffic inspection. It is not typically part of basic VPC endpoint usage.

---

## ⚖️ Gateway vs Interface Endpoint Comparison

| Feature | Gateway Endpoint | Interface Endpoint |
|---------|-----------------|-------------------|
| Mechanism | Route table entry (prefix list) | ENI with private IP in your subnet |
| Supported Services | S3, DynamoDB only | 100+ AWS services |
| Cost | ✅ Free | ~$0.01/AZ/hour + $0.01/GB |
| DNS Resolution | Not required | ✅ Private DNS created automatically |
| On-Premises Access | ❌ Not supported | ✅ Supported via VPN/DX |
| Cross-Region | ❌ Not supported | ✅ Supported |
| Security Group | ❌ Not applicable | ✅ Apply security groups to ENI |
| Endpoint Policy | ✅ Supported | ✅ Supported |
| HA / Multi-AZ | Automatically available | Deploy one ENI per AZ |
| Setup Complexity | Simple (route table edit) | Moderate (ENI + DNS + SG) |

---

## 🔧 How VPC Endpoints Work Internally

### Gateway Endpoint — Prefix Lists

When you create an S3 Gateway Endpoint, AWS creates a **Managed Prefix List** — a group of IP address ranges that collectively represent the S3 service in your region. This prefix list is injected into the route tables you select.

```
  AWS Managed Prefix List for S3 (us-east-1):
  pl-63a5400a  →  {52.216.0.0/15, 54.231.0.0/17, 3.5.0.0/19, ...}

  Your route table entry:
  pl-63a5400a  →  vpce-0abc123def456      (your Gateway Endpoint)
```

The OS and SDK on your EC2 instance don't know anything changed — they still make requests to `s3.amazonaws.com`. The route table intercepts those packets and redirects them to the endpoint.

### Interface Endpoint — Private DNS

When you create an Interface Endpoint with **Private DNS** enabled, AWS creates a **private hosted zone** in Route 53 that overrides the public DNS for that service:

```
  Public DNS (without endpoint):
  ssm.us-east-1.amazonaws.com  →  52.46.x.x  (public IP)

  Private DNS (with Interface Endpoint):
  ssm.us-east-1.amazonaws.com  →  10.0.1.50  (ENI private IP in your VPC)
```

Because the DNS name resolves to a private IP, all existing SDK and CLI calls automatically use the endpoint with **zero application changes**.

---

## 🔐 Endpoint Policies

**Endpoint Policies** are IAM-style resource policies attached directly to the VPC endpoint. They act as an **additional layer of access control** on top of IAM user/role policies.

A request must be allowed by **both** the endpoint policy AND the IAM policy to succeed.

### Default Policy (allows all)

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

### Restrictive Policy — Allow Only a Specific S3 Bucket

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ]
    }
  ]
}
```

This policy ensures that even if an EC2 instance has broad S3 IAM permissions, it can **only** access `my-company-bucket` through this endpoint — not any other S3 bucket.

### Deny All Except Through Endpoint (S3 Bucket Policy)

You can also enforce endpoint usage from the S3 bucket side by denying all requests that don't come through the endpoint:

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0abc123def456"
        }
      }
    }
  ]
}
```

---

## 🧪 Lab 1 — Gateway Endpoint for S3

### Objective

Create an **S3 Gateway Endpoint** so that an EC2 instance in a **private subnet** (no NAT Gateway, no internet access) can read and write to S3 entirely within the AWS private network.

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16)                     │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │            Private Subnet (10.0.1.0/24)         │    │
│  │                                                  │    │
│  │   ┌─────────────┐                               │    │
│  │   │  EC2 Instance│                               │    │
│  │   │  No public IP│                               │    │
│  │   │  No NAT GW   │                               │    │
│  │   └──────┬───────┘                               │    │
│  └──────────┼────────────────────────────────────── ┘    │
│             │                                            │
│             │  Route: pl-xxxxxx → vpce-xxxxxxx           │
│             ▼                                            │
│   ┌──────────────────┐                                   │
│   │  S3 Gateway      │                                   │
│   │  Endpoint        │                                   │
│   │  (vpce-xxxxxxx)  │                                   │
│   └────────┬─────────┘                                   │
└────────────┼─────────────────────────────────────────────┘
             │
             │  AWS Private Network (no internet)
             ▼
        Amazon S3
```

### Prerequisites

- An existing VPC with a private subnet (no route to an IGW)
- An EC2 instance in that private subnet with an IAM role that has S3 permissions
- Access to the EC2 instance via SSM Session Manager or a bastion host

---

### Step 1 — Create the VPC and Private Subnet

If you don't already have one, create a VPC and private subnet:

1. Go to **VPC Console → Create VPC**

| Field | Value |
|-------|-------|
| Name | `Lab-VPC` |
| CIDR | `10.0.0.0/16` |

2. Create a **private subnet** (no route to Internet Gateway):

| Field | Value |
|-------|-------|
| Name | `Private-Subnet` |
| CIDR | `10.0.1.0/24` |
| Route Table | Should have no `0.0.0.0/0` route |

---

### Step 2 — Launch an EC2 Instance with an IAM Role

1. Launch an EC2 instance in `Private-Subnet`
2. Attach an **IAM instance profile** with this policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

3. **No public IP** — this instance should be completely private

Before creating the endpoint, verify the instance **cannot** reach S3:

```bash
# Connect via SSM Session Manager (also needs an Interface Endpoint — see Lab 2)
# Or connect via a bastion in a public subnet

aws s3 ls    # This should fail or hang — no route to S3
```

---

### Step 3 — Create the S3 Gateway Endpoint

1. Go to **VPC Console → Endpoints → Create Endpoint**
2. Configure:

| Field | Value |
|-------|-------|
| Name | `S3-Gateway-Endpoint` |
| Service Category | AWS Services |
| Service Name | Search `s3` → Select `com.amazonaws.us-east-1.s3` with Type **Gateway** |
| VPC | `Lab-VPC` |
| Route Tables | Select the route table associated with `Private-Subnet` |
| Policy | Full Access (default) — we will restrict later |

3. Click **Create Endpoint**

---

### Step 4 — Verify the Route Table Entry

1. Go to **VPC Console → Route Tables** → select the private subnet's route table
2. Click the **Routes** tab

You should now see a new entry automatically added by AWS:

| Destination | Target |
|-------------|--------|
| `pl-63a5400a` (S3 prefix list) | `vpce-xxxxxxxxx` |

This is the Gateway Endpoint route. Traffic to S3 will now use this route.

---

### Step 5 — Create an S3 Bucket and Test

1. Create an S3 bucket (e.g., `my-vpc-endpoint-test-bucket`) in the same region

2. Connect to the EC2 instance and test:

```bash
# Upload a file to S3
echo "Hello from private EC2!" > test.txt
aws s3 cp test.txt s3://my-vpc-endpoint-test-bucket/test.txt

# List the bucket
aws s3 ls s3://my-vpc-endpoint-test-bucket/

# Download the file
aws s3 cp s3://my-vpc-endpoint-test-bucket/test.txt downloaded.txt
cat downloaded.txt
```

All three commands should succeed — traffic flows from the EC2 instance through the VPC endpoint directly to S3, **never touching the internet**. ✅

---

### Step 6 — Restrict the Endpoint with a Policy (Optional but Recommended)

Apply an endpoint policy to restrict access to only your specific bucket:

1. Go to **VPC Console → Endpoints** → select `S3-Gateway-Endpoint`
2. Click **Actions → Edit Policy**
3. Replace the default policy with:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-vpc-endpoint-test-bucket",
        "arn:aws:s3:::my-vpc-endpoint-test-bucket/*"
      ]
    }
  ]
}
```

4. Now test that access to a **different** S3 bucket is denied:

```bash
aws s3 ls s3://some-other-bucket/
# Should return: Access Denied ✅
```

---

## 🧪 Lab 2 — Interface Endpoint for SSM

### Objective

Create **AWS Systems Manager (SSM) Interface Endpoints** so that an EC2 instance in a **fully private subnet** (no internet, no NAT) can be managed via SSM Session Manager — without any public IP or bastion host.

SSM requires **three** Interface Endpoints to function:

| Endpoint | Service |
|----------|---------|
| `ssm` | Core SSM service |
| `ssmmessages` | Session Manager messaging |
| `ec2messages` | EC2 message delivery |

### Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      VPC (10.0.0.0/16)                        │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Private Subnet (10.0.1.0/24)            │    │
│  │                                                       │    │
│  │  ┌─────────────┐    ┌──────────────────────────────┐ │    │
│  │  │  EC2 Instance│    │    Interface Endpoints (ENIs)│ │    │
│  │  │  No internet │    │                              │ │    │
│  │  │  No NAT GW   │    │  ssm           10.0.1.10    │ │    │
│  │  │  No pub IP   ├───▶│  ssmmessages   10.0.1.11    │ │    │
│  │  │              │    │  ec2messages   10.0.1.12    │ │    │
│  │  └─────────────┘    └──────────────────────────────┘ │    │
│  └──────────────────────────────────────────────────────┘    │
│                              │                                │
│                Private DNS resolves service names             │
│                to ENI private IPs automatically               │
└──────────────────────────────────────────────────────────────┘
                              │
                              │ AWS Private Network
                              ▼
                      AWS Systems Manager
```

### Prerequisites

- VPC and private subnet from Lab 1 (or create new ones)
- EC2 instance with **AmazonSSMManagedInstanceCore** IAM policy attached
- EC2 must have **SSM Agent** installed (pre-installed on Amazon Linux 2/2023 and recent Ubuntu AMIs)

---

### Step 1 — Create a Security Group for the Endpoints

The Interface Endpoint ENIs need a Security Group that allows HTTPS traffic from the VPC.

1. Go to **EC2 Console → Security Groups → Create Security Group**
2. Configure:

| Field | Value |
|-------|-------|
| Name | `SSM-Endpoint-SG` |
| Description | Security group for SSM Interface Endpoints |
| VPC | `Lab-VPC` |

3. Add **Inbound Rule:**

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| HTTPS | TCP | 443 | `10.0.0.0/16` (VPC CIDR) |

4. Click **Create**

> 💡 Interface Endpoint ENIs communicate with your EC2 instances over **HTTPS (port 443)**. The source should be your VPC CIDR so all resources in the VPC can use the endpoints.

---

### Step 2 — Verify EC2 IAM Role

Your EC2 instance needs the **AmazonSSMManagedInstanceCore** managed policy. This grants SSM the permissions to manage the instance.

1. Go to **IAM Console → Roles** → find the role attached to your EC2 instance
2. Ensure this policy is attached:

```
arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

This policy allows the SSM agent on the EC2 to communicate with the SSM service — registering the instance, receiving commands, and streaming session data.

---

### Step 3 — Create the Three SSM Interface Endpoints

You need to create one endpoint for each of the three required services. Repeat this process three times.

**Go to VPC Console → Endpoints → Create Endpoint for each:**

#### Endpoint 1 — SSM

| Field | Value |
|-------|-------|
| Name | `SSM-Endpoint` |
| Service Category | AWS Services |
| Service Name | `com.amazonaws.us-east-1.ssm` |
| VPC | `Lab-VPC` |
| Subnets | Select `Private-Subnet` in `us-east-1a` |
| Private DNS | ✅ Enable |
| Security Group | `SSM-Endpoint-SG` |
| Policy | Full Access |

#### Endpoint 2 — SSM Messages

| Field | Value |
|-------|-------|
| Name | `SSMMessages-Endpoint` |
| Service Name | `com.amazonaws.us-east-1.ssmmessages` |
| VPC | `Lab-VPC` |
| Subnets | Select `Private-Subnet` |
| Private DNS | ✅ Enable |
| Security Group | `SSM-Endpoint-SG` |

#### Endpoint 3 — EC2 Messages

| Field | Value |
|-------|-------|
| Name | `EC2Messages-Endpoint` |
| Service Name | `com.amazonaws.us-east-1.ec2messages` |
| VPC | `Lab-VPC` |
| Subnets | Select `Private-Subnet` |
| Private DNS | ✅ Enable |
| Security Group | `SSM-Endpoint-SG` |

> ⏳ Wait for all three endpoints to show status `available` before testing.

---

### Step 4 — Verify Private DNS Resolution

SSH into any instance that can reach the VPC (or use another method), and verify DNS resolves to private IPs:

```bash
# Verify SSM endpoint DNS resolves to a private IP (not a public AWS IP)
nslookup ssm.us-east-1.amazonaws.com

# Expected output — should show a 10.x.x.x IP (your ENI private IP)
# Server:  10.0.0.2
# Address: 10.0.0.2#53
# Name:    ssm.us-east-1.amazonaws.com
# Address: 10.0.1.10    ← Private IP of the Interface Endpoint ENI ✅
```

If you see a public IP here, **Private DNS is not enabled** on the endpoint — go back and enable it.

---

### Step 5 — Connect via SSM Session Manager

1. Go to **EC2 Console → Instances** → select your private EC2 instance
2. Click **Connect → Session Manager → Connect**

A terminal session opens directly in your browser — **no SSH key, no bastion, no public IP needed**. ✅

You can also connect via AWS CLI:

```bash
aws ssm start-session --target i-0abc123def456789
```

---

### Step 6 — Validate No Internet Traffic

From within the SSM session on the EC2 instance, verify there is no internet connectivity — confirming that SSM works purely through the VPC endpoints:

```bash
# This should FAIL (no internet access)
curl https://google.com --max-time 5
# curl: (28) Connection timed out ✅ (expected — no internet)

# This should SUCCEED (SSM already working — you're connected!)
echo "SSM Session via VPC Endpoint is working!" ✅
```

---

### Step 7 — Check Endpoint Network Interfaces

Verify the ENIs that were created for your endpoints:

1. Go to **EC2 Console → Network Interfaces**
2. Filter by Description: `VPC Endpoint`
3. You should see three ENIs — one for each SSM endpoint — with private IPs in your subnet

---

## 🛠️ Troubleshooting

### Gateway Endpoint Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| S3 access still fails after creating endpoint | Route not added to correct route table | Re-check which route table is associated with the instance's subnet — ensure that route table was selected during endpoint creation |
| Access Denied to S3 | Endpoint policy too restrictive | Review and update the endpoint policy — check if bucket ARN in policy is correct |
| Can access S3 from some subnets but not others | Route only added to one route table | Edit endpoint and add the other route tables |
| Cross-region S3 bucket not accessible | Gateway endpoints are region-specific | Use an Interface Endpoint for cross-region S3, or configure S3 Cross-Region Replication |

---

### Interface Endpoint Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| SSM instance shows as offline | One or more of the 3 endpoints missing | Ensure all three endpoints exist: `ssm`, `ssmmessages`, `ec2messages` |
| DNS still resolving to public IP | Private DNS not enabled | Edit endpoint → enable Private DNS |
| Connection refused on port 443 | Security Group blocking HTTPS | Ensure `SSM-Endpoint-SG` allows TCP 443 inbound from VPC CIDR |
| Session Manager connect button greyed out | IAM role missing SSM policy | Attach `AmazonSSMManagedInstanceCore` to the EC2 IAM role |
| Endpoint stuck in `pending` | Normal — takes a few minutes | Wait 3-5 minutes for endpoint to become `available` |
| Endpoint works in one AZ but not another | Endpoint only deployed in one AZ | Create endpoint ENI in every AZ where you have instances |

---

## 💰 Pricing Overview

| Endpoint Type | Hourly Charge | Data Processing |
|---------------|--------------|----------------|
| **Gateway Endpoint** | ✅ Free | ✅ Free |
| **Interface Endpoint** | ~$0.01 per AZ per hour | ~$0.01 per GB |

**Interface Endpoint cost example:**
- 3 SSM endpoints × 2 AZs × $0.01/hour × 730 hours/month = **~$0.44/month** per endpoint set
- Plus data processing for session data transferred

> 💡 Always delete Interface Endpoints when not needed — they are billed hourly even when idle.

---

## ✅ Best Practices

| Practice | Why It Matters |
|----------|---------------|
| Deploy Interface Endpoints in every AZ | Avoids cross-AZ traffic charges and single-AZ failure |
| Use dedicated Security Groups for endpoints | Fine-grained control over which resources can use each endpoint |
| Always enable Private DNS on Interface Endpoints | Allows existing code/SDK calls to work without changes |
| Apply restrictive Endpoint Policies | Limit which S3 buckets or services can be accessed through the endpoint |
| Use S3 bucket policies with `aws:sourceVpce` condition | Force all S3 access to go through the endpoint — deny public access |
| Use Gateway Endpoints for S3 and DynamoDB | They're free — there's no reason not to |
| Monitor with VPC Flow Logs | Verify traffic is flowing through the endpoint, not through NAT |
| Tag endpoints clearly | Multiple endpoints in large environments are hard to manage without tags |

---

## 📌 Summary

| Concept | Key Point |
|---------|-----------|
| **VPC Endpoint** | Private connectivity to AWS services without internet or NAT |
| **Gateway Endpoint** | Route table entry — free — supports S3 and DynamoDB only |
| **Interface Endpoint** | ENI in your subnet — supports 100+ services — charged hourly |
| **Private DNS** | Interface Endpoints override public DNS so existing code works transparently |
| **Endpoint Policy** | IAM-style policy on the endpoint itself — additional access control layer |
| **SSM Endpoints** | Three endpoints needed: `ssm`, `ssmmessages`, `ec2messages` |
| **S3 Gateway Endpoint** | Best practice for any VPC that accesses S3 — eliminates NAT costs |
| **Security** | Traffic stays on AWS backbone — never crosses the public internet |

---

<div align="center">

*Part of the AWS Networking series*

</div>
