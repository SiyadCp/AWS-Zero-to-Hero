# VPC PrivateLink & VPC Lattice

## Table of Contents

- [VPC PrivateLink](#vpc-privatelink)
  - [Overview](#privatelink-overview)
  - [Key Concepts](#privatelink-key-concepts)
  - [Architecture](#privatelink-architecture)
  - [Endpoint Types](#endpoint-types)
  - [How It Works](#how-it-works)
  - [Supported Services](#supported-services)
  - [IAM & Security](#privatelink-iam--security)
  - [DNS Resolution](#dns-resolution)
  - [Limitations](#privatelink-limitations)
  - [Pricing](#privatelink-pricing)
  - [CLI Reference](#privatelink-cli-reference)
- [VPC Lattice](#vpc-lattice)
  - [Overview](#lattice-overview)
  - [Key Concepts](#lattice-key-concepts)
  - [Architecture](#lattice-architecture)
  - [Core Components](#core-components)
  - [Auth Policies](#auth-policies)
  - [Routing & Traffic Management](#routing--traffic-management)
  - [Observability](#observability)
  - [Limitations](#lattice-limitations)
  - [Pricing](#lattice-pricing)
  - [CLI Reference](#lattice-cli-reference)
- [PrivateLink vs VPC Lattice — Comparison](#privatelink-vs-vpc-lattice--comparison)

---

## VPC PrivateLink

### PrivateLink Overview

**AWS PrivateLink** provides private connectivity between VPCs, AWS services, and on-premises networks without exposing traffic to the public internet. Traffic between the consumer and the service never leaves the Amazon network.

PrivateLink is the underlying technology behind:
- AWS-managed service endpoints (e.g., S3, DynamoDB, EC2 APIs)
- Sharing your own services privately across accounts or organizations
- AWS Marketplace SaaS integrations

```
Consumer VPC                          Provider VPC
┌──────────────────┐                 ┌──────────────────────┐
│                  │                 │                      │
│  EC2 / Lambda    │                 │  NLB                 │
│       │          │                 │   │                  │
│  Interface       │   PrivateLink   │  Target Group        │
│  Endpoint  ──────┼─────────────────┼─► (EC2, ECS, Lambda) │
│  (ENI + DNS)     │                 │                      │
│                  │                 │                      │
└──────────────────┘                 └──────────────────────┘
         ▲
    Private IP,
    no internet
    exposure
```

---

### PrivateLink Key Concepts

| Term | Description |
|------|-------------|
| **VPC Endpoint** | Entry point in the consumer VPC to access a service privately |
| **Endpoint Service** | The provider's service, backed by an NLB, exposed via PrivateLink |
| **Interface Endpoint** | ENI-based endpoint with a private IP in the consumer subnet |
| **Gateway Endpoint** | Route-table based endpoint (S3 and DynamoDB only, free) |
| **Gateway Load Balancer Endpoint** | Used for inline traffic inspection (firewalls, IDS/IPS) |
| **Endpoint Policy** | Resource-based policy controlling access through the endpoint |
| **Principal** | AWS account or organization allowed to connect to an endpoint service |

---

### PrivateLink Architecture

#### Consumer Side
- An **Interface Endpoint** is created in one or more subnets
- Each endpoint creates an **ENI** with a private IP in the subnet
- AWS creates **private DNS entries** that resolve the service hostname to the endpoint's private IP
- Security groups on the ENI control which resources can use the endpoint

#### Provider Side
- A **Network Load Balancer (NLB)** fronts the service
- The **Endpoint Service** is registered using the NLB ARN
- The provider **whitelists** consumer AWS accounts or entire organizations
- Connections are one-directional: consumer → provider only

---

### Endpoint Types

#### 1. Interface Endpoints (PrivateLink)
- Backed by one or more ENIs in your subnets
- Supports most AWS services and custom endpoint services
- Billed per hour + per GB data processed
- Supports security groups and endpoint policies
- Supports private DNS (overrides the public hostname)

#### 2. Gateway Endpoints
- Route-table based; no ENI created
- Supported **only** for **Amazon S3** and **Amazon DynamoDB**
- Free of charge
- Does not support security groups; uses endpoint policies only
- Does not support private DNS

#### 3. Gateway Load Balancer Endpoints
- Routes traffic through a fleet of virtual appliances (firewalls, IPS/IDS)
- Traffic is inspected inline before reaching the destination
- Works with AWS Gateway Load Balancer (GWLB)

---

### How It Works

**Creating an Endpoint Service (Provider)**

```
1. Deploy your service behind a Network Load Balancer (NLB)
2. Create an Endpoint Service using the NLB ARN
3. Set acceptance mode: automatic or manual approval
4. Whitelist allowed AWS principals (accounts, IAM roles, org IDs)
5. Share the Endpoint Service name with consumers
```

**Creating an Interface Endpoint (Consumer)**

```
1. Navigate to VPC → Endpoints → Create Endpoint
2. Select the service (AWS service or endpoint service name)
3. Choose the VPC and subnets (one ENI per AZ)
4. Assign a security group to the ENI
5. Optionally enable Private DNS
6. Attach an endpoint policy
```

---

### Supported Services

PrivateLink natively supports 100+ AWS services, including:

| Category | Services |
|----------|----------|
| **Compute** | EC2, ECS, EKS, Lambda, Auto Scaling |
| **Storage** | S3 (Gateway), EBS, EFS, Backup |
| **Database** | RDS, Aurora, DynamoDB (Gateway), ElastiCache, Redshift |
| **Networking** | Route 53 Resolver, VPC Lattice, Transit Gateway |
| **Security** | KMS, Secrets Manager, ACM, GuardDuty, Security Hub |
| **Monitoring** | CloudWatch, CloudTrail, X-Ray |
| **Developer** | CodeBuild, CodeDeploy, CodePipeline |
| **AI/ML** | SageMaker, Bedrock, Rekognition |
| **Custom** | Your own services via Endpoint Service (NLB-backed) |

---

### PrivateLink IAM & Security

#### Endpoint Policy

Attached to the endpoint itself; controls what actions can be performed through it. This is additive to IAM policies — both must allow the action.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-private-bucket/*"
    }
  ]
}
```

#### Endpoint Service Principal Whitelisting

```bash
# Allow a specific account to connect
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-xxxxxxxxx \
  --add-allowed-principals arn:aws:iam::123456789010:root

# Allow an entire AWS Organization
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-xxxxxxxxx \
  --add-allowed-principals arn:aws:organizations::123456789010:organization/o-xxxxxxxxxx
```

#### Security Groups

Security groups are attached to the **Interface Endpoint ENI**, not to individual resources. Control inbound access to the endpoint, and outbound from it to your service.

---

### DNS Resolution

When **Private DNS** is enabled on an Interface Endpoint:

```
Public hostname:   ec2.us-east-1.amazonaws.com
                              ↓ (resolves via Route 53 in VPC)
Private DNS:       ec2.us-east-1.amazonaws.com → 10.0.1.45 (ENI IP)
```

- Requires **enableDnsSupport** and **enableDnsHostnames** set to `true` on the VPC
- Works transparently — no code or config changes needed in applications
- For custom endpoint services, AWS provides a default DNS name like:  
  `vpce-0abc123.vpce-svc-xyz.us-east-1.vpce.amazonaws.com`

---

### PrivateLink Limitations

- Interface endpoints are **regional** — they cannot span regions
- Each endpoint can only connect to **one service**
- Gateway endpoints support only **S3 and DynamoDB**
- **Transitive routing is not supported** — you cannot route from VPC A → VPC B's endpoint → service
- Maximum of **255 endpoints per VPC** (soft limit, can be increased)
- NLB backing an endpoint service must be in the **same region**
- Does not support IPv6 for endpoint services by default

---

### PrivateLink Pricing

| Component | Cost |
|-----------|------|
| Interface Endpoint (per AZ per hour) | ~$0.01/hr |
| Data processed (per GB) | ~$0.01/GB |
| Gateway Endpoint | **Free** |
| Gateway Load Balancer Endpoint | ~$0.01/hr + per GB |

> 💡 In multi-AZ deployments, create endpoints in each AZ for redundancy and to avoid cross-AZ data transfer charges.

---

### PrivateLink CLI Reference

```bash
# List available AWS services for endpoints
aws ec2 describe-vpc-endpoint-services --query 'ServiceNames[]'

# Create an interface endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-aaaa subnet-bbbb \
  --security-group-ids sg-xxxxxxxx \
  --private-dns-enabled

# Create a gateway endpoint (S3)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxxxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-xxxxxxxx

# Create an endpoint service (provider side)
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:us-east-1:123456789010:loadbalancer/net/my-nlb/xxxxx \
  --acceptance-required

# Describe all endpoints in your account
aws ec2 describe-vpc-endpoints \
  --query 'VpcEndpoints[*].{ID:VpcEndpointId,Service:ServiceName,State:State}'

# Delete an endpoint
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-xxxxxxxx
```

---

---

## VPC Lattice

### Lattice Overview

**Amazon VPC Lattice** is a fully managed application networking service that simplifies service-to-service connectivity, security, and monitoring across VPCs and AWS accounts — without requiring complex VPC peering, Transit Gateway configurations, or PrivateLink endpoint services for each pair of services.

VPC Lattice operates at **Layer 7 (application layer)**, supporting HTTP, HTTPS, and gRPC, and provides:
- Centralized auth and access control via IAM
- Weighted traffic routing, path-based routing, and failover
- Built-in observability (access logs, CloudWatch metrics)
- Cross-account and cross-VPC service mesh

```
Account A (Consumer)           Service Network           Account B (Provider)
┌─────────────────────┐       ┌──────────────┐          ┌───────────────────────┐
│  VPC A              │       │              │          │  VPC B                │
│  ┌───────────────┐  │       │  ┌────────┐  │          │  ┌─────────────────┐  │
│  │  Service A    ├──┼───────┤► │Service │◄─┼──────────┤  │  Service B      │  │
│  │  (EC2/Lambda) │  │       │  │Network │  │          │  │  (ECS/Lambda)   │  │
│  └───────────────┘  │       │  └────────┘  │          │  └─────────────────┘  │
└─────────────────────┘       └──────────────┘          └───────────────────────┘
                                     │
                               Auth + Routing
                               + Observability
```

---

### Lattice Key Concepts

| Term | Description |
|------|-------------|
| **Service Network** | Logical grouping of services and VPCs; the backbone of the mesh |
| **Service** | Represents a microservice or application registered in Lattice |
| **Service Directory** | Central catalog of all Lattice services in the account/org |
| **Listener** | Protocol and port on which a service accepts traffic (HTTP/HTTPS/gRPC) |
| **Listener Rule** | Condition-action routing rules (path, header, method matching) |
| **Target Group** | Backend targets (EC2, ECS, Lambda, ALB, IP) behind a service |
| **Association** | Links a VPC or service to a service network |
| **Auth Policy** | IAM-based policy controlling which principals can invoke a service |
| **Resource Policy** | RAM-based policy for cross-account sharing |

---

### Lattice Architecture

VPC Lattice uses a **service network** as the central hub:

```
                        ┌─────────────────────────────────────────┐
                        │            Service Network               │
                        │  (auth, routing, observability plane)    │
                        └────────────────┬────────────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
   ┌──────────▼─────────┐    ┌───────────▼────────┐    ┌───────────▼────────┐
   │   Service: Orders   │    │  Service: Payments  │    │  Service: Auth     │
   │  ┌───────────────┐  │    │  ┌───────────────┐  │    │  ┌──────────────┐  │
   │  │  Target Group │  │    │  │  Target Group │  │    │  │ Target Group │  │
   │  │  EC2 / Lambda │  │    │  │  ECS Fargate  │  │    │  │   Lambda     │  │
   │  └───────────────┘  │    │  └───────────────┘  │    │  └──────────────┘  │
   └─────────────────────┘    └────────────────────┘    └────────────────────┘
```

VPCs are **associated** to the service network, and services are **registered** into it. Any resource in an associated VPC can then call any registered service using its Lattice-generated DNS name.

---

### Core Components

#### Service Network

The central connectivity hub. Acts like a shared service mesh fabric.

- Multiple VPCs (same or different accounts) can be associated
- Services registered to the network are discoverable by all associated VPCs
- Supports auth policies at the network level (applies to all services)
- Shared across accounts using **AWS RAM**

```bash
aws vpc-lattice create-service-network --name my-service-network --auth-type AWS_IAM
```

#### Service

Represents one microservice or application. Each service has:
- A **Lattice-generated DNS name**: `service-name-<id>.lattice.amazonaws.com`
- One or more **Listeners** (HTTP port 80, HTTPS port 443, gRPC)
- **Listener Rules** for routing decisions
- One or more **Target Groups**

```bash
aws vpc-lattice create-service \
  --name orders-service \
  --auth-type AWS_IAM
```

#### Target Group

Defines the backend compute for a service. Supported target types:

| Target Type | Description |
|-------------|-------------|
| `INSTANCE` | EC2 instances |
| `IP` | IP addresses (any ENI in VPC) |
| `LAMBDA` | AWS Lambda function |
| `ALB` | Application Load Balancer |

```bash
aws vpc-lattice create-target-group \
  --name orders-targets \
  --type LAMBDA \
  --config '{"lambdaEventStructureVersion":"V2"}'
```

#### Listener & Rules

Listeners define the protocol/port. Rules define how traffic is routed.

```bash
# Create HTTPS listener
aws vpc-lattice create-listener \
  --service-identifier svc-xxxxxxxxx \
  --name https-listener \
  --protocol HTTPS \
  --port 443 \
  --default-action '{"forward":{"targetGroups":[{"targetGroupIdentifier":"tg-xxx","weight":100}]}}'
```

**Rule match conditions:**

| Condition Type | Example |
|----------------|---------|
| Path match | `/api/orders/*` |
| Header match | `x-env: production` |
| Method match | `POST` |
| Weighted routing | 80% → v1 target group, 20% → v2 target group |

---

### Auth Policies

VPC Lattice uses **IAM-based auth policies** — the same syntax as S3 bucket policies or resource policies — attached at the service network or individual service level.

#### Allow all traffic within the same organization

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "vpc-lattice-svcs:Invoke",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxx"
        }
      }
    }
  ]
}
```

#### Allow a specific IAM role only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789010:role/OrdersServiceRole"
      },
      "Action": "vpc-lattice-svcs:Invoke",
      "Resource": "arn:aws:vpc-lattice:us-east-1:123456789010:service/svc-xxxxxxxxx/*"
    }
  ]
}
```

> ℹ️ Auth policy evaluation: **Network policy** AND **Service policy** must both allow the request. If either denies, access is blocked.

---

### Routing & Traffic Management

#### Weighted Routing (Blue/Green or Canary)

Split traffic between two target groups by percentage:

```json
{
  "action": {
    "forward": {
      "targetGroups": [
        { "targetGroupIdentifier": "tg-v1-xxx", "weight": 90 },
        { "targetGroupIdentifier": "tg-v2-xxx", "weight": 10 }
      ]
    }
  }
}
```

#### Path-Based Routing

```json
{
  "match": {
    "pathMatch": {
      "match": { "prefix": "/api/v2/" },
      "caseSensitive": false
    }
  },
  "action": {
    "forward": {
      "targetGroups": [{ "targetGroupIdentifier": "tg-v2-xxx", "weight": 100 }]
    }
  }
}
```

#### Header-Based Routing

```json
{
  "match": {
    "headerMatches": [
      { "name": "x-env", "match": { "exact": "canary" } }
    ]
  }
}
```

---

### Observability

VPC Lattice provides built-in observability without additional agents:

#### Access Logs

Detailed per-request logs including:

| Field | Description |
|-------|-------------|
| `sourceVpcArn` | VPC the request originated from |
| `requestMethod` | HTTP method |
| `requestPath` | URL path |
| `responseCode` | HTTP status code |
| `duration` | Request latency in ms |
| `callerPrincipal` | IAM identity of the caller |
| `targetGroupArn` | Which target group handled the request |

Access logs can be sent to **CloudWatch Logs**, **S3**, or **Kinesis Data Firehose**.

```bash
aws vpc-lattice put-resource-policy \
  --resource-arn arn:aws:vpc-lattice:us-east-1:123456789010:service/svc-xxx \
  --policy file://access-log-policy.json
```

#### CloudWatch Metrics

VPC Lattice publishes metrics to CloudWatch under the `AWS/VpcLattice` namespace:

| Metric | Description |
|--------|-------------|
| `RequestCount` | Number of requests per service |
| `4XXCount` | Client error responses |
| `5XXCount` | Server error responses |
| `TargetResponseTime` | Latency from Lattice to target |

---

### Lattice Limitations

- Supports **HTTP, HTTPS, and gRPC** only — not raw TCP/UDP
- Each **service network** can be associated with up to **10 VPCs** by default (soft limit)
- Each **VPC** can be associated with only **1 service network**
- Maximum of **500 services per service network** (soft limit)
- **Private DNS** for Lattice services requires Route 53 private hosted zone integration
- Cross-region service discovery is **not natively supported** — services must be in the same region as the service network
- Lattice is **not a replacement for API Gateway** — it is for internal east-west service traffic, not north-south internet-facing APIs

---

### Lattice Pricing

| Component | Cost |
|-----------|------|
| Service network association (per VPC per hour) | ~$0.025/hr |
| Data processed (per GB) | ~$0.025/GB |
| Request (per million requests) | ~$2.50 |

> 💡 VPC Lattice pricing is higher per-GB than PrivateLink but includes Layer 7 routing, auth, and observability — factoring in the operational value is important when comparing costs.

---

### Lattice CLI Reference

```bash
# Create a service network
aws vpc-lattice create-service-network \
  --name prod-service-network \
  --auth-type AWS_IAM

# Associate a VPC with the service network
aws vpc-lattice create-service-network-vpc-association \
  --service-network-identifier sn-xxxxxxxxx \
  --vpc-identifier vpc-xxxxxxxx \
  --security-group-ids sg-xxxxxxxx

# Create a service
aws vpc-lattice create-service \
  --name payments-service \
  --auth-type AWS_IAM

# Associate a service with the service network
aws vpc-lattice create-service-network-service-association \
  --service-network-identifier sn-xxxxxxxxx \
  --service-identifier svc-xxxxxxxxx

# Create a target group (Lambda)
aws vpc-lattice create-target-group \
  --name payments-lambda-tg \
  --type LAMBDA

# Register a Lambda as a target
aws vpc-lattice register-targets \
  --target-group-identifier tg-xxxxxxxxx \
  --targets '[{"id":"arn:aws:lambda:us-east-1:123456789010:function:PaymentsFunction"}]'

# List all services in account
aws vpc-lattice list-services

# List service network associations
aws vpc-lattice list-service-network-vpc-associations \
  --service-network-identifier sn-xxxxxxxxx

# Put auth policy on a service
aws vpc-lattice put-auth-policy \
  --resource-identifier svc-xxxxxxxxx \
  --policy file://auth-policy.json

# Delete a service
aws vpc-lattice delete-service --service-identifier svc-xxxxxxxxx
```

---

---

## PrivateLink vs VPC Lattice — Comparison

| Feature | VPC PrivateLink | VPC Lattice |
|---------|----------------|-------------|
| **OSI Layer** | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS/gRPC) |
| **Primary Use Case** | Private access to a single service or AWS API | Service mesh for many microservices |
| **Traffic Routing** | Port-based only | Path, header, method, weighted |
| **Auth** | IAM endpoint policies + security groups | IAM auth policies (per network or service) |
| **Cross-Account** | Yes (whitelist principals) | Yes (AWS RAM sharing) |
| **Cross-VPC** | Yes | Yes |
| **Cross-Region** | No | No |
| **DNS** | Private DNS override for AWS services | Lattice-generated DNS per service |
| **Observability** | Via VPC Flow Logs only | Native access logs + CloudWatch metrics |
| **Blue/Green Routing** | No | Yes (weighted target groups) |
| **Backends Supported** | NLB → EC2, ECS, IP, Lambda | EC2, ECS, Lambda, ALB, IP |
| **Managed by** | Consumer and Provider configure separately | Centralized service network |
| **Best For** | Exposing a single service privately | Internal microservice communication at scale |
| **Pricing Model** | Per endpoint/hr + per GB | Per association/hr + per GB + per million requests |

### When to Use Which

**Choose PrivateLink when:**
- You need private access to a specific AWS service (S3, KMS, RDS, etc.)
- You want to expose a single service to other accounts or VPCs
- You need Layer 4 (TCP/UDP) connectivity (non-HTTP protocols)
- Cost sensitivity is high and traffic volume is large

**Choose VPC Lattice when:**
- You have multiple microservices that need to communicate across VPCs and accounts
- You need Layer 7 routing (path, header, weighted canary deployments)
- You want centralized auth, access control, and observability for all services
- You are building or migrating to a service mesh architecture on AWS

---

## References

- [AWS PrivateLink Documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/)
- [PrivateLink Supported Services](https://docs.aws.amazon.com/vpc/latest/privatelink/aws-services-privatelink-support.html)
- [VPC Lattice Documentation](https://docs.aws.amazon.com/vpc-lattice/latest/ug/)
- [VPC Lattice Pricing](https://aws.amazon.com/vpc/lattice/pricing/)
- [PrivateLink Pricing](https://aws.amazon.com/privatelink/pricing/)
- [VPC Lattice Auth Policies](https://docs.aws.amazon.com/vpc-lattice/latest/ug/auth-policies.html)
