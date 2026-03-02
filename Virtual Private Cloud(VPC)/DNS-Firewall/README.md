<div align="center">

![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Route53](https://img.shields.io/badge/Route53-DNS-8C4FFF?style=for-the-badge&logo=amazon-route53&logoColor=white)
![VPC](https://img.shields.io/badge/VPC-Networking-blue?style=for-the-badge)
![Security](https://img.shields.io/badge/Security-DNS%20Firewall-red?style=for-the-badge)

# AWS VPC DNS Firewall — In-Depth Guide

### 🛡️ Filter and control outbound DNS queries from your VPC — block malicious domains before the connection is ever made.

</div>

---

## 📋 Table of Contents

- [What is DNS Firewall?](#-what-is-dns-firewall)
- [Why DNS Firewall?](#-why-dns-firewall--the-threat-it-addresses)
- [How DNS Firewall Works](#-how-dns-firewall-works)
- [Core Components](#-core-components)
  - [Domain Lists](#1-domain-lists)
  - [Firewall Rules](#2-firewall-rules)
  - [Rule Groups](#3-rule-groups)
  - [VPC Association](#4-vpc-association)
- [Rule Actions](#-rule-actions)
- [DNS Firewall Architecture](#-dns-firewall-architecture)
- [Managed Domain Lists](#-aws-managed-domain-lists)
- [DNS Firewall vs Other AWS Security Tools](#-dns-firewall-vs-other-aws-security-tools)
- [Lab — Block google.com from EC2 Instances](#-lab--block-googlecom-from-ec2-instances)
- [Monitoring and Logging](#-monitoring-and-logging)
- [Troubleshooting](#-troubleshooting)
- [Pricing Overview](#-pricing-overview)
- [Best Practices](#-best-practices)
- [Summary](#-summary)

---

## 💡 What is DNS Firewall?

**AWS Route 53 Resolver DNS Firewall** is a managed security service that allows you to **filter and control outbound DNS queries** made by resources inside your VPC.

Every time an EC2 instance, Lambda function, or container resolves a domain name (e.g., `google.com`, `s3.amazonaws.com`, or a malicious C2 domain), that query passes through Route 53 Resolver. DNS Firewall sits in the path of these queries and can:

- **Allow** the query to proceed normally
- **Block** the query and return a custom response (NXDOMAIN, NODATA, or a custom IP)
- **Alert** and pass the query through for monitoring purposes

```
  EC2 Instance
      │
      │  DNS Query: "resolve google.com"
      ▼
  Route 53 Resolver (VPC DNS at 169.254.169.253 / .2)
      │
      ▼
  ┌─────────────────────────────────┐
  │       DNS FIREWALL              │
  │                                 │
  │  Rule 1: Block *.malware.com    │
  │  Rule 2: Block google.com   ◀───┼──  Match found!
  │  Rule 3: Allow *                │
  └─────────────────────────────────┘
      │
      ▼  (if blocked)
  Response: NXDOMAIN  ──▶  EC2 gets "domain not found"
  (connection never made to google.com)
```

---

## 🔥 Why DNS Firewall? — The Threat it Addresses

### DNS is the Foundation of All Network Communication

Before any TCP/IP connection is made, a DNS query is sent. This makes DNS a uniquely powerful control point — **if you block the DNS query, the connection can never be established**, even if the EC2 instance has full network access to the internet.

### Threats DNS Firewall Defends Against

**Command and Control (C2) Traffic**
Malware on a compromised EC2 instance typically phones home to a C2 server using a domain name. DNS Firewall can block these queries, severing the malware's communication channel before any data is exfiltrated.

**DNS Tunneling**
Attackers encode data inside DNS queries to exfiltrate information or bypass network firewalls. Since DNS is usually allowed through firewalls, this technique can bypass traditional controls. DNS Firewall with AWS Managed lists can detect and block tunneling patterns.

**Data Exfiltration**
Blocking access to unauthorized external domains ensures sensitive data cannot be sent to attacker-controlled servers.

**Shadow IT / Unauthorized Services**
Prevent EC2 instances or containers from communicating with unauthorized SaaS services, social media, or external APIs that violate your organization's policies.

```
  Attack Scenario — Without DNS Firewall:

  Compromised EC2
      │
      │  DNS: "resolve c2-server.attacker.com"
      ▼
  Route 53 Resolver  ──▶  Resolves to 1.2.3.4
      │
      ▼
  EC2 connects to 1.2.3.4  ──▶  Data exfiltrated ❌

  ──────────────────────────────────────────────────

  Attack Scenario — With DNS Firewall:

  Compromised EC2
      │
      │  DNS: "resolve c2-server.attacker.com"
      ▼
  Route 53 Resolver  ──▶  DNS Firewall intercepts
      │
      ▼
  Rule matches: Block *.attacker.com
      │
      ▼
  Response: NXDOMAIN  ──▶  EC2 gets "domain not found"
  Connection never established ✅
```

---

## ⚙️ How DNS Firewall Works

DNS Firewall integrates directly with **Route 53 Resolver** — the DNS server built into every VPC (accessible at the VPC base address + 2, e.g., `10.0.0.2`, and also at `169.254.169.253`).

**The query flow:**

```
  1. Resource in VPC makes a DNS query
           │
           ▼
  2. Query reaches Route 53 Resolver
           │
           ▼
  3. DNS Firewall evaluates the query against rule groups
     associated with the VPC — rules are checked by priority
           │
     ┌─────┴──────────────────────┐
     │                            │
     ▼                            ▼
  Rule MATCHES               No rule matches
  Apply action:              Default behavior applies
  BLOCK / ALERT / ALLOW      (ALLOW if fail-open,
                              BLOCK if fail-closed)
           │
           ▼
  4. Response returned to the resource
     (real answer, NXDOMAIN, custom IP, or NODATA)
```

DNS Firewall processes queries **before** Route 53 Resolver forwards them to upstream resolvers or returns cached responses. It operates at **Layer 7 (Application Layer)** — specifically the DNS protocol.

---

## 🧩 Core Components

### 1. Domain Lists

A **Domain List** is a collection of domain names that you want to match in firewall rules. There are two types:

**Custom Domain Lists** — You create and manage these. Add specific domains you want to block or allow.

```
Example Custom Block List:
  google.com
  facebook.com
  *.social-media.com
  evil-domain.net
```

**AWS Managed Domain Lists** — Pre-built lists maintained by AWS threat intelligence. Updated automatically as new threats are discovered. (See [Managed Domain Lists](#-aws-managed-domain-lists) section.)

Domain list entries support:
- Exact match: `google.com`
- Wildcard subdomain: `*.google.com` (matches `mail.google.com`, `www.google.com`)
- Note: `*.google.com` does NOT match `google.com` itself — add both if needed

---

### 2. Firewall Rules

A **Firewall Rule** ties a domain list to an action. Each rule has:

| Attribute | Description |
|-----------|-------------|
| **Name** | Descriptive label for the rule |
| **Priority** | Numeric value — lower number = evaluated first (1-65535) |
| **Domain List** | Which list of domains to match against |
| **Action** | What to do when a query matches: ALLOW, BLOCK, or ALERT |
| **Block Response** | If action is BLOCK: NXDOMAIN, NODATA, or OVERRIDE |
| **Override Domain** | If block response is OVERRIDE: return this IP instead |

---

### 3. Rule Groups

A **Rule Group** is a container for one or more firewall rules. Rule groups are the unit that gets associated with VPCs.

```
  Rule Group: "Corporate-DNS-Policy"
  ├── Rule 1  Priority: 10  Block-List → BLOCK (NXDOMAIN)
  ├── Rule 2  Priority: 20  Allow-List → ALLOW
  └── Rule 3  Priority: 30  Suspicious  → ALERT

  This rule group is then associated with one or more VPCs.
```

You can associate **multiple rule groups** with a single VPC, and each rule group can have its own priority relative to other groups.

---

### 4. VPC Association

Once a rule group is created, you **associate** it with one or more VPCs. From that point on, all DNS queries from resources in that VPC are evaluated against the rule group's rules.

```
  Rule Group: "Corp-Block-List"
  │
  ├── Associated with: VPC-Production
  ├── Associated with: VPC-Development
  └── Associated with: VPC-Staging

  All three VPCs now enforce the same DNS policy.
```

---

## 🎯 Rule Actions

| Action | Behavior | Use Case |
|--------|----------|----------|
| **ALLOW** | Query passes through to be resolved normally | Whitelist known-good domains |
| **BLOCK — NXDOMAIN** | Returns "domain does not exist" | Standard block — application thinks domain doesn't exist |
| **BLOCK — NODATA** | Returns empty answer (domain exists but no records) | Subtle block — less likely to trigger error logs |
| **BLOCK — OVERRIDE** | Returns a custom IP address you specify | Redirect blocked queries to a warning page or sinkhole |
| **ALERT** | Logs the match but resolves normally | Monitor suspicious queries without breaking connectivity |

### Block Response Types Explained

```
  Query: resolve "google.com"

  NXDOMAIN response:
  → "Non-Existent Domain" — application sees the domain as completely gone
  → curl: (6) Could not resolve host: google.com

  NODATA response:
  → Domain exists but has no A record returned
  → Application gets an empty DNS response

  OVERRIDE response (e.g., override IP = 10.0.0.100):
  → google.com resolves to 10.0.0.100 (your custom sinkhole IP)
  → Traffic is redirected to your warning page or logging server
```

---

## 🏗️ DNS Firewall Architecture

### Single VPC — Basic Setup

```
┌─────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16)                    │
│                                                         │
│   EC2-A          EC2-B           Lambda                 │
│   ┌─────┐        ┌─────┐         ┌──────┐              │
│   │     │        │     │         │      │              │
│   └──┬──┘        └──┬──┘         └──┬───┘              │
│      │              │               │                   │
│      └──────────────┴───────────────┘                   │
│                       │                                 │
│                       │  All DNS queries                │
│                       ▼                                 │
│             ┌──────────────────┐                        │
│             │  Route 53        │                        │
│             │  Resolver        │                        │
│             │  10.0.0.2        │                        │
│             └────────┬─────────┘                        │
│                      │                                  │
│                      ▼                                  │
│         ┌────────────────────────┐                      │
│         │     DNS FIREWALL       │                      │
│         │                        │                      │
│         │  Priority 10: BLOCK    │                      │
│         │  → malware-list        │                      │
│         │                        │                      │
│         │  Priority 20: BLOCK    │                      │
│         │  → custom-block-list   │                      │
│         │                        │                      │
│         │  Priority 30: ALLOW    │                      │
│         │  → approved-list       │                      │
│         └────────────────────────┘                      │
│                      │                                  │
│           Match? ────┤                                  │
│           Block/Allow│                                  │
└──────────────────────┼──────────────────────────────────┘
                       │ (if allowed)
                       ▼
               Public DNS Resolution
               (google.com → 142.250.x.x)
```

### Multi-VPC — Centralized Policy

```
  Shared Rule Groups (managed centrally by Security team)
  ┌─────────────────────────────────────────────┐
  │  Group 1: AWS-Managed-ThreatIntel (Priority 1)│
  │  Group 2: Corporate-Block-Policy  (Priority 2)│
  │  Group 3: Approved-Services       (Priority 3)│
  └─────────────────────────────────────────────┘
           │              │               │
           ▼              ▼               ▼
       VPC-Prod       VPC-Dev        VPC-Staging
   (all associated with same rule groups)
```

---

## 🛡️ AWS Managed Domain Lists

AWS maintains several threat intelligence domain lists that are automatically updated. These are available as **Managed Domain Lists** in DNS Firewall:

| List Name | What It Blocks |
|-----------|---------------|
| `AWSManagedDomainsMalwareDomainList` | Known malware distribution and C2 domains |
| `AWSManagedDomainsBotnetCommandandControl` | Botnet command and control domains |
| `AWSManagedDomainsAggregateThreatList` | Combination of all AWS threat intelligence lists |

**Benefits of Managed Lists:**
- Updated automatically by AWS threat intelligence — no maintenance required
- Based on real-world threat data
- Free to use (you only pay for the DNS Firewall association, not the list itself)

---

## ⚖️ DNS Firewall vs Other AWS Security Tools

| Tool | Layer | What It Controls |
|------|-------|-----------------|
| **DNS Firewall** | DNS (Layer 7) | Which domain names can be resolved |
| **Security Groups** | Transport (Layer 4) | Inbound/outbound TCP/UDP ports and IPs |
| **Network ACLs** | Network (Layer 3/4) | Subnet-level IP CIDR and port rules |
| **AWS WAF** | Application (Layer 7) | HTTP/HTTPS web traffic patterns |
| **AWS Network Firewall** | Network/Transport (Layer 3-7) | Deep packet inspection on traffic flows |
| **GuardDuty** | Detection (all layers) | Threat detection and anomaly alerting |

> 💡 DNS Firewall is **complementary**, not a replacement. Use it alongside Security Groups and Network Firewall for defense in depth. DNS Firewall stops threats at the earliest possible point — before a connection is ever made.

---

## 🧪 Lab — Block google.com from EC2 Instances

### Objective

Configure **AWS VPC DNS Firewall** to block DNS resolution of `google.com` and all its subdomains from EC2 instances inside a VPC. Verify that:

1. Before the firewall: `google.com` resolves normally
2. After the firewall: `google.com` returns `NXDOMAIN`
3. Other domains (e.g., `amazon.com`) continue to resolve normally

### Architecture

```
┌────────────────────────────────────────────────────┐
│                  VPC (10.0.0.0/16)                 │
│                                                    │
│   ┌────────────────┐                               │
│   │   EC2 Instance │                               │
│   │  (Test client) │                               │
│   └───────┬────────┘                               │
│           │                                        │
│           │  DNS query: google.com                 │
│           ▼                                        │
│   ┌───────────────────┐                            │
│   │  Route 53 Resolver│                            │
│   └────────┬──────────┘                            │
│            │                                       │
│            ▼                                       │
│   ┌─────────────────────────────┐                  │
│   │      DNS FIREWALL           │                  │
│   │                             │                  │
│   │  Rule: Block-Google         │                  │
│   │  Domain: google.com         │◀── MATCH         │
│   │          *.google.com       │                  │
│   │  Action: BLOCK (NXDOMAIN)   │                  │
│   └─────────────────────────────┘                  │
│            │                                       │
│            ▼                                       │
│   EC2 receives: NXDOMAIN                           │
│   "could not resolve host: google.com"  ✅         │
└────────────────────────────────────────────────────┘
```

### Prerequisites

- An existing VPC with at least one subnet
- An EC2 instance running in that VPC with internet access (for baseline testing)
- IAM permissions: `route53resolver:*`
- EC2 must have a public IP or be reachable via Session Manager for testing

---

### Step 1 — Baseline Test (Before Firewall)

Connect to your EC2 instance and verify that `google.com` currently resolves:

```bash
# Test DNS resolution — should succeed BEFORE the firewall
nslookup google.com

# Expected output:
# Server:   10.0.0.2
# Address:  10.0.0.2#53
#
# Non-authoritative answer:
# Name:    google.com
# Address: 142.250.x.x   ← Real IP returned ✅

# Also test with dig
dig google.com +short
# Returns: 142.250.x.x

# Test HTTP connectivity
curl -I https://google.com --max-time 5
# Returns: HTTP/2 301  ← google.com is reachable ✅
```

Record the results — you will compare these against the post-firewall output.

---

### Step 2 — Create a Custom Domain List

1. Go to **AWS Console → VPC → DNS Firewall → Domain lists → Create domain list**

2. Configure:

| Field | Value |
|-------|-------|
| Domain list name | `Block-Google-Domains` |
| Description | `Blocks google.com and all subdomains` |

3. Under **Domains**, add the following entries one per line:

```
google.com
*.google.com
```

> ⚠️ You must add **both** entries:
> - `google.com` — blocks the apex domain itself
> - `*.google.com` — blocks all subdomains (www.google.com, mail.google.com, etc.)
>
> The wildcard `*.google.com` alone does **not** match `google.com` without the dot prefix.

4. Click **Create domain list**

---

### Step 3 — Create a Firewall Rule Group

1. Go to **DNS Firewall → Rule groups → Create rule group**

2. Configure:

| Field | Value |
|-------|-------|
| Rule group name | `Block-Unauthorized-Domains` |
| Description | `Corporate DNS policy blocking unauthorized domains` |
| Region | Your current region (e.g., `us-east-1`) |

3. Click **Create rule group** (you will add rules next)

---

### Step 4 — Add Rules to the Rule Group

After creating the rule group, add rules to it:

1. Select `Block-Unauthorized-Domains` → click **Add rule**

#### Rule 1 — Block Google

| Field | Value |
|-------|-------|
| Rule name | `Block-Google` |
| Priority | `10` |
| Domain list | `Block-Google-Domains` |
| Action | `Block` |
| Block response | `NXDOMAIN` |

Click **Add rule**.

#### Rule 2 — Allow Everything Else (Optional but Recommended)

Add an explicit allow-all rule at a lower priority to document your intent and make the policy readable:

| Field | Value |
|-------|-------|
| Rule name | `Allow-All-Other` |
| Priority | `100` |
| Domain list | Create a new list with `*` (wildcard) OR skip this rule |
| Action | `Allow` |

> 💡 Without an explicit ALLOW rule, non-matching queries still resolve normally (DNS Firewall only acts on matches). The ALLOW rule is optional but documents intent clearly.

Click **Save changes** on the rule group.

---

### Step 5 — Associate the Rule Group with Your VPC

1. Go to **DNS Firewall → Rule groups** → select `Block-Unauthorized-Domains`
2. Click **Associate VPCs → Associate**
3. Select your VPC from the dropdown
4. Set the **Priority** for this rule group (e.g., `100`)

> 💡 Rule group priority matters when multiple rule groups are associated with the same VPC. Lower number = evaluated first.

5. Click **Associate**

The association status will show as **Complete** within a few seconds. DNS Firewall is now **active on your VPC**.

---

### Step 6 — Test the Block (After Firewall)

Connect to your EC2 instance and repeat the DNS tests:

```bash
# Test 1: Resolve google.com — should be BLOCKED
nslookup google.com

# Expected output after firewall:
# Server:   10.0.0.2
# Address:  10.0.0.2#53
#
# ** server can't find google.com: NXDOMAIN   ← Blocked! ✅
```

```bash
# Test 2: Try with dig — shows NXDOMAIN explicitly
dig google.com

# Expected output:
# ;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 12345
# ;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0
#
# Status: NXDOMAIN  ← Blocked! ✅
```

```bash
# Test 3: Try curl — should fail to resolve
curl -I https://google.com --max-time 5

# Expected output:
# curl: (6) Could not resolve host: google.com  ← Blocked! ✅
```

```bash
# Test 4: Try a subdomain — should also be blocked
nslookup www.google.com
# ** server can't find www.google.com: NXDOMAIN  ✅

nslookup mail.google.com
# ** server can't find mail.google.com: NXDOMAIN  ✅
```

```bash
# Test 5: Verify other domains STILL work
nslookup amazon.com
# Returns real IP — NOT blocked ✅

nslookup github.com
# Returns real IP — NOT blocked ✅

curl -I https://amazon.com --max-time 5
# Returns HTTP response — reachable ✅
```

---

### Step 7 — Change Block Response to OVERRIDE (Bonus)

Instead of returning NXDOMAIN, redirect blocked domains to a custom IP — useful for showing a "blocked page" warning.

1. Go to **DNS Firewall → Rule groups** → select `Block-Unauthorized-Domains`
2. Select the `Block-Google` rule → click **Edit**
3. Change:

| Field | Value |
|-------|-------|
| Block response | `Override` |
| Override domain | `10.0.0.100` (your sinkhole server IP) |

4. Save changes

Now test again:

```bash
nslookup google.com

# Expected output:
# Name:    google.com
# Address: 10.0.0.100   ← Redirected to your sinkhole IP ✅
```

This technique is used to redirect blocked traffic to an internal warning page.

---

### Step 8 — Enable Query Logging (Recommended)

Enable DNS Firewall logging to see which queries are being blocked:

1. Go to **VPC → DNS Firewall → Query logging**

   OR go to **Route 53 → Resolver → Query logging**

2. Click **Configure query logging**

3. Configure:

| Field | Value |
|-------|-------|
| Name | `DNS-Firewall-Logs` |
| Destination | CloudWatch Logs (easiest for Lab) |
| Log group name | `/aws/route53resolver/dns-firewall` |
| VPC | Select your VPC |

4. Click **Create**

5. After triggering some blocked queries, check CloudWatch Logs:

```bash
# Each log entry looks like this:
{
  "version": "1.100000",
  "account_id": "123456789012",
  "region": "us-east-1",
  "vpc_id": "vpc-0abc123",
  "query_name": "google.com.",
  "query_type": "A",
  "query_class": "IN",
  "rcode": "NXDOMAIN",          ← Blocked
  "firewall_rule_action": "BLOCK",
  "firewall_domain_list_id": "rslvr-fdl-xxxxx",
  "firewall_rule_group_id": "rslvr-frg-xxxxx"
}
```

---

### Step 9 — Clean Up

Remove resources after the lab to avoid charges:

1. **Disassociate the rule group** from the VPC:
   - DNS Firewall → Rule groups → select group → VPC associations → Disassociate

2. **Delete the rule group:**
   - DNS Firewall → Rule groups → select → Delete

3. **Delete the domain list:**
   - DNS Firewall → Domain lists → select → Delete

4. **Disable query logging** (if enabled)

---

### Lab Summary — What Happened

```
BEFORE DNS FIREWALL:
  EC2 → Route 53 Resolver → Public DNS → 142.250.x.x (google.com)
  Result: google.com resolved successfully ✅

AFTER DNS FIREWALL:
  EC2 → Route 53 Resolver → DNS Firewall (MATCH: google.com)
                                    │
                                    ▼
                             Action: BLOCK
                                    │
                                    ▼
  Result: NXDOMAIN returned to EC2 ✅
  google.com was NEVER contacted — blocked at DNS layer
```

---

## 📊 Monitoring and Logging

### Route 53 Resolver Query Logs

Every DNS query (allowed and blocked) can be logged to:
- **Amazon CloudWatch Logs** — for real-time monitoring and alerts
- **Amazon S3** — for long-term storage and compliance
- **Amazon Kinesis Data Firehose** — for streaming to SIEM tools

**Key fields in DNS Firewall logs:**

| Field | Description |
|-------|-------------|
| `query_name` | The domain that was queried |
| `rcode` | DNS response code (NOERROR, NXDOMAIN, etc.) |
| `firewall_rule_action` | ALLOW, BLOCK, or ALERT |
| `firewall_domain_list_id` | Which domain list triggered the match |
| `src_addr` | Private IP of the resource that made the query |
| `vpc_id` | Which VPC the query came from |

### CloudWatch Metrics

DNS Firewall publishes metrics to CloudWatch:

| Metric | Description |
|--------|-------------|
| `FirewallQueryVolume` | Total DNS queries evaluated by the firewall |
| `FirewallQueryPassVolume` | Queries that were allowed |
| `FirewallQueryBlockVolume` | Queries that were blocked |
| `FirewallQueryAlertVolume` | Queries that triggered ALERT rules |

### Create a CloudWatch Alarm for Blocked Queries

```
Metric: FirewallQueryBlockVolume
Threshold: > 100 blocks in 5 minutes
Action: SNS notification to security team
```

This alerts your team when an unusual number of DNS blocks are occurring — potentially indicating a compromised instance attempting to reach C2 infrastructure.

---

## 🛠️ Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Domain not blocked after creating firewall | Rule group not associated with VPC | Go to Rule Groups → VPC Associations and verify the association exists |
| Domain not blocked after association | DNS cache — query was cached before firewall activated | Wait for TTL to expire, or flush DNS on EC2: `sudo systemd-resolve --flush-caches` |
| Legitimate domain accidentally blocked | Wildcard too broad (e.g., `*.com`) | Review domain list entries — be specific with domains |
| All DNS queries failing | `FAIL_CLOSE` behavior on firewall misconfiguration | Check Route 53 Resolver DNS Firewall fail-open setting on the VPC |
| Wildcard not matching subdomains | Wildcard entry missing `*.` prefix | Ensure entry is `*.google.com` not `google.com` for subdomain matching |
| Firewall rule not evaluated | Rule priority conflict with another rule group | Check priorities — lower number is evaluated first |
| Logs not appearing in CloudWatch | Query logging not enabled or log group not created | Enable Resolver Query Logging and associate it with the VPC |
| OVERRIDE not working | Override IP not reachable from VPC | Ensure the override IP is accessible from the VPC's route tables |

---

## 💰 Pricing Overview

| Component | Cost |
|-----------|------|
| **DNS Firewall Rule Group Association** | ~$0.60 per VPC per month |
| **DNS Queries processed** | ~$0.60 per million queries |
| **AWS Managed Domain Lists** | Free (included in association cost) |
| **Custom Domain Lists** | Free to create — queries charged above |
| **Resolver Query Logging** | CloudWatch / S3 / Kinesis standard rates apply |

**Cost example — medium workload:**
- 1 VPC: $0.60/month
- 50 million queries/month: $30/month
- **Total: ~$30.60/month for DNS-level protection**

---

## ✅ Best Practices

| Practice | Why It Matters |
|----------|---------------|
| Enable AWS Managed Threat Lists | Instant protection from known malware/botnet domains with no maintenance |
| Use ALERT action before BLOCK for new rules | Validate that blocking a domain won't break production before enforcing |
| Add both `domain.com` and `*.domain.com` to block lists | Wildcard alone doesn't match the apex domain |
| Enable Resolver Query Logging | Essential for security auditing and investigating incidents |
| Set up CloudWatch alarms on block volume | Detect compromised instances reaching out to blocked domains |
| Use OVERRIDE action to redirect to a warning sinkhole | Better user experience and allows logging which users hit blocked domains |
| Apply firewall to all VPCs centrally | Share rule groups across VPCs — maintain one policy, apply everywhere |
| Review and update custom domain lists regularly | Threat landscape changes — keep your block lists current |
| Test with ALERT before production enforcement | Avoid breaking legitimate DNS resolution |
| Separate rule groups by policy type | "Security threats" group vs "Corporate policy" group — easier to manage |

---

## 📌 Summary

| Concept | Key Point |
|---------|-----------|
| **DNS Firewall** | Filters outbound DNS queries from VPC resources before connections are made |
| **Domain List** | Collection of domains to match — custom or AWS managed |
| **Firewall Rule** | Ties a domain list to an action (ALLOW/BLOCK/ALERT) with a priority |
| **Rule Group** | Container for rules — associated with one or more VPCs |
| **BLOCK — NXDOMAIN** | Returns "domain not found" — standard block response |
| **BLOCK — OVERRIDE** | Redirects blocked domain to a custom IP — sinkhole pattern |
| **ALERT** | Logs the match but resolves normally — monitoring without breaking |
| **AWS Managed Lists** | Pre-built threat intel lists — auto-updated, free to use |
| **Lab Result** | `google.com` → NXDOMAIN — connection never made, blocked at DNS layer |
| **Key Advantage** | Blocks threats at the earliest possible point — before any TCP connection |

---

<div align="center">

*Part of the AWS Networking and Security series*

</div>
