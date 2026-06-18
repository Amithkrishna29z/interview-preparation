# AWS Cloud Interview Questions & Answers

> Commonly asked in cloud/DevOps/backend developer interviews. Focus on understanding **why** AWS services exist, not just **what** they are.

---

## Table of Contents

1. [AWS Global Infrastructure](#1-aws-global-infrastructure)
2. [IAM – Identity & Access Management](#2-iam--identity--access-management)
3. [EC2 – Elastic Compute Cloud](#3-ec2--elastic-compute-cloud)
4. [S3 – Simple Storage Service](#4-s3--simple-storage-service)
5. [VPC – Virtual Private Cloud](#5-vpc--virtual-private-cloud)
6. [RDS & Databases](#6-rds--databases)
7. [Load Balancing & Auto Scaling](#7-load-balancing--auto-scaling)
8. [Lambda & Serverless](#8-lambda--serverless)
9. [CloudWatch & Monitoring](#9-cloudwatch--monitoring)
10. [Route 53 & DNS](#10-route-53--dns)
11. [CloudFormation & IaC](#11-cloudformation--iac)
12. [Security & Compliance](#12-security--compliance)
13. [SNS, SQS & Messaging](#13-sns-sqs--messaging)
14. [ECS, EKS & Containers](#14-ecs-eks--containers)
15. [Cost Optimization](#15-cost-optimization)
16. [Quick Revision Summary](#16-quick-revision-summary)

---

## 1. AWS Global Infrastructure

### Q1: What is a Region in AWS?

- A Region is a **geographic area** containing multiple data centers
- Each Region is **independent** of others; AWS has 30+ Regions globally
- Choose a Region based on: latency, compliance (e.g., GDPR), cost, and available services

| Region Code | Location |
|---|---|
| `ap-south-1` | Mumbai, India |
| `us-east-1` | N. Virginia, USA (oldest, most services) |
| `eu-west-1` | Ireland, Europe |

**Interview Tip:** "I choose a Region close to my users for low latency and based on data residency laws."

---

### Q2: What is an Availability Zone (AZ)?

- Each Region has **2–6 AZs** — physically separate data centers connected by low-latency private fiber
- Deploying across multiple AZs provides **High Availability (HA)**

```
Region: ap-south-1 (Mumbai)
├── AZ: ap-south-1a
├── AZ: ap-south-1b
└── AZ: ap-south-1c
```

---

### Q3: What is an Edge Location?

- Used by **CloudFront** (CDN) and **Route 53**; 400+ locations globally
- Content is **cached** at the nearest edge location for low-latency delivery to users

---

## 2. IAM – Identity & Access Management

### Q4: What is IAM?

- **IAM** = Identity and Access Management — controls **who can access what** in AWS
- Handles **authentication** (who you are) and **authorization** (what you can do)
- IAM is a **global service** (not Region-specific)

| Component | What it is |
|---|---|
| **User** | A person or application with permanent credentials |
| **Group** | A collection of users sharing the same permissions |
| **Role** | A temporary identity assumed by services/users (no permanent credentials) |
| **Policy** | JSON document defining permissions |

---

### Q5: User vs Group vs Role — key difference?

- **User**: permanent credentials (password or access keys) for a person/app
- **Group**: attach one policy to many users (e.g., "Developers" group → EC2 read access)
- **Role**: assumed temporarily by EC2, Lambda, or cross-account — **no hardcoded keys needed**

```
Bad ❌:  EC2 stores AWS_ACCESS_KEY in env variables
Good ✅: EC2 is assigned an IAM Role; SDK uses temporary credentials automatically
```

---

### Q6: What is an IAM Policy?

A JSON document: Allow/Deny an **Action** on a **Resource**, optionally under a **Condition**.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

**Golden Rule:** Principle of **Least Privilege** — grant only what's needed.

---

### Q7: What is the Root User and why avoid it?

- Created when you first sign up; has **unrestricted access** to everything
- **Never use root for daily tasks** — create an IAM admin user instead
- Root-only tasks: change account email, enable MFA, close account
- Best practices: enable MFA on root immediately; never create root access keys

---

## 3. EC2 – Elastic Compute Cloud

### Q8: What is EC2?

- EC2 = **Elastic Compute Cloud** — virtual servers you rent by the hour/second
- Highly configurable: CPU, RAM, storage, OS, network
- **Lifecycle:** Launch → Pending → Running → Stopping → Stopped → Terminated

---

### Q9: EC2 Instance Types

| Family | Optimized For | Example Use Case |
|---|---|---|
| **General Purpose** (`t3`, `m6i`) | Balanced CPU/memory | Web servers, small DBs |
| **Compute Optimized** (`c6i`) | High CPU | Batch processing, gaming servers |
| **Memory Optimized** (`r6i`) | Large RAM | In-memory DBs (Redis, SAP) |
| **Storage Optimized** (`i3`) | High disk I/O | OLTP databases |
| **Accelerated** (`p4`, `g4`) | GPU | ML training, video rendering |

Naming: `m6i.xlarge` = family (`m`) + generation (`6`) + variant (`i`) + size.

---

### Q10: EC2 Purchasing Options

| Option | Best For | Discount |
|---|---|---|
| **On-Demand** | Short-term, unpredictable | Baseline |
| **Reserved Instances** | Steady usage (1 or 3 yr) | Up to 72% |
| **Savings Plans** | Flexible committed spend | Up to 72% |
| **Spot Instances** | Fault-tolerant, interruptible | Up to 90% |
| **Dedicated Hosts** | Compliance/licensing | Most expensive |

Spot Instances run on unused capacity; AWS can reclaim them with 2-minute warning — good for batch/ML, not for databases.

---

### Q11: Security Group vs NACL

| Feature | Security Group | NACL |
|---|---|---|
| **Level** | Instance (ENI) | Subnet |
| **State** | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| **Rules** | Allow only | Allow AND Deny |
| **Default** | All outbound allowed, inbound denied | All traffic allowed |

Traffic order: Internet → NACL → Security Group → EC2.

---

### Q12: What is an AMI?

- **AMI** = Amazon Machine Image — template (OS + software + config) used to launch EC2 instances
- Region-specific; can be AWS-provided, Marketplace, or your own custom image
- **Use case:** bake a configured server into a custom AMI → launch identical instances fast

---

### Q13: EBS vs Instance Store vs EFS

| Feature | EBS | Instance Store | EFS |
|---|---|---|---|
| **Type** | Block storage | Ephemeral block | File storage (NFS) |
| **Persistence** | Persists after stop | **Lost when instance stops** | Persists, shared |
| **Use case** | Root volumes, DBs | Temp caches/buffers | Shared across instances |
| **Replication** | Within 1 AZ | None | Multi-AZ |

---

## 4. S3 – Simple Storage Service

### Q14: What is S3?

- S3 = **Simple Storage Service** — object storage for any file type
- **Unlimited storage**, objects up to 5 TB; bucket names must be **globally unique**
- Object URL: `https://<bucket>.s3.<region>.amazonaws.com/<key>`

---

### Q15: S3 Storage Classes

| Storage Class | Access Pattern | Cost |
|---|---|---|
| **S3 Standard** | Frequent | Highest |
| **S3 Standard-IA** | Infrequent, fast retrieval | Lower + retrieval fee |
| **S3 Glacier** (Instant/Flexible/Deep Archive) | Archive (ms to 12 hrs) | Very low |
| **S3 Intelligent-Tiering** | Unknown pattern | Auto-moves between tiers |

---

### Q16: How do you secure an S3 bucket?

1. **Block Public Access** — on by default; never disable unless required
2. **Bucket Policy** — JSON policy attached to the bucket
3. **IAM Policy** — permissions for users/roles
4. **Encryption** — SSE-S3 (AWS managed), SSE-KMS (your key), SSE-C (client key)
5. **Versioning** — keeps all versions; protects against accidental deletes
6. **MFA Delete** — requires MFA to delete versions

---

### Q17: What is S3 Versioning?

- Keeps **every version** of every object — protects against accidental deletes/overwrites
- A "delete" adds a **delete marker**; you can restore the previous version
- Use **Lifecycle policies** to auto-expire old versions and control cost

---

## 5. VPC – Virtual Private Cloud

### Q18: What is a VPC?

- VPC = **Virtual Private Cloud** — your logically isolated private network in AWS
- You control: CIDR range, subnets, route tables, gateways, security
- Each AWS account gets a **default VPC** per Region

```
VPC (10.0.0.0/16)
├── Public Subnet  → has route to Internet Gateway (web servers, LBs)
├── Private Subnet → no direct internet (app servers, DBs)
├── Internet Gateway  → inbound/outbound internet for public subnet
├── NAT Gateway       → outbound-only internet for private subnet
└── Route Tables      → routing rules per subnet
```

---

### Q19: Public vs Private Subnet

| Feature | Public Subnet | Private Subnet |
|---|---|---|
| **Internet access** | Yes (via Internet Gateway) | No direct internet |
| **Use case** | Web servers, load balancers | App servers, databases |
| **Outbound internet** | Direct via IGW | Via NAT Gateway |

---

### Q20: NAT Gateway and VPC Peering

- **NAT Gateway**: sits in the public subnet; lets private EC2 instances reach the internet **outbound only** (e.g., download updates). AWS-managed.
- **VPC Peering**: connects two VPCs privately (same/cross-Region, cross-account). **Not transitive** — A↔B and B↔C does not give A↔C.

---

## 6. RDS & Databases

### Q21: What is Amazon RDS?

- RDS = **Relational Database Service** — managed relational DB
- Supports: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora
- AWS manages: hardware, OS patching, automated backups (up to 35-day retention), failover
- You manage: schema, indexes, query tuning

---

### Q22: Multi-AZ vs Read Replicas

| Feature | Multi-AZ | Read Replica |
|---|---|---|
| **Purpose** | High Availability | Read scaling / performance |
| **Replication** | Synchronous | Asynchronous |
| **Readable?** | No (passive standby) | Yes |
| **Failover** | Automatic (~1–2 min) | Manual promotion |
| **Cross-Region** | No | Yes |

---

### Q23: What is Amazon Aurora?

AWS's cloud-native MySQL/PostgreSQL-compatible DB — faster than standard RDS, auto-scaling storage with 6 copies across 3 AZs, up to 15 read replicas, fast failover. Aurora Serverless auto-scales compute to load.

---

## 7. Load Balancing & Auto Scaling

### Q24: What is Elastic Load Balancing (ELB)?

Distributes incoming traffic across multiple EC2 instances so no single instance is overwhelmed.

| Type | Layer | Use Case |
|---|---|---|
| **ALB** (Application LB) | Layer 7 (HTTP/HTTPS) | Web apps, microservices, path/host routing |
| **NLB** (Network LB) | Layer 4 (TCP/UDP) | Ultra-high performance, static IP |

ALB supports path-based (`/api/*`) and host-based (`api.example.com`) routing, target groups (EC2, ECS, Lambda).

---

### Q25: What is Auto Scaling?

Automatically adds/removes EC2 instances based on demand — scale out on high traffic, scale in to save cost.

- **Auto Scaling Group (ASG):** managed group of EC2 instances
- **Launch Template:** blueprint for new instances
- **Target Tracking** (most common): keep a metric at a target (e.g., CPU at 50%). Others: Step/Simple (on alarm), Scheduled, Predictive (ML).

**Flow:** CloudWatch alarm (CPU > 70%) → scaling policy triggers → instances launch from template → register with Load Balancer.

---

## 8. Lambda & Serverless

### Q26: What is AWS Lambda?

- Serverless compute — run code without managing servers; pay only for what runs
- Supports Python, Java, Node.js, Go, Ruby, .NET
- Max execution: **15 minutes**; Memory: 128 MB–10 GB

**Common triggers:** API Gateway, S3 events, SQS/SNS, EventBridge (cron), DynamoDB Streams, Kinesis.

---

### Q27: Lambda vs EC2

| | Lambda | EC2 |
|---|---|---|
| **Server management** | None | You manage OS/patches |
| **Scaling** | Automatic per-request | Manual or Auto Scaling |
| **Billing** | Per ms of execution | Per hr/sec (even idle) |
| **Max runtime** | 15 minutes | Unlimited |
| **Best for** | Event-driven, short tasks | Long-running apps |

**Cold Start:** first invocation after idle → container init adds latency. Fix: **Provisioned Concurrency**. Java/Spring Boot cold starts are notably longer than Node.js/Python.

---

## 9. CloudWatch & Monitoring

### Q28: What is Amazon CloudWatch?

Monitoring and observability for AWS — collects metrics, logs, and events.

| Component | Purpose |
|---|---|
| **Metrics** | Numerical data over time (CPU %, request count) |
| **Logs** | Log groups/streams from EC2, Lambda, etc. |
| **Alarms** | Trigger actions when a metric crosses a threshold |
| **Dashboards** | Visual monitoring panels |
| **EventBridge** | React to AWS state changes / schedule events |

Default EC2 metrics (free): CPU, Network In/Out, Disk Read/Write, Status Check. RAM/Disk space need the CloudWatch Agent (paid custom metrics).

---

### Q29: CloudWatch vs CloudTrail

| | CloudWatch | CloudTrail |
|---|---|---|
| **Purpose** | Performance & operational monitoring | Audit trail of API calls |
| **Tracks** | Metrics, logs, performance | Who did what, when, from where |
| **Use case** | Alert when CPU > 90% | "Who deleted that S3 bucket?" |
| **Retention** | Configurable | 90 days free; S3 for long-term |

- **CloudWatch** = "Is my app healthy?"
- **CloudTrail** = "Who changed what in my account?"

---

## 10. Route 53 & DNS

### Q30: What is Amazon Route 53?

- Highly available **DNS web service** — translates `www.myapp.com` → IP address
- Supports domain registration; named for DNS **port 53**

**Routing Policies:** Simple (one resource), Weighted (A/B split), Latency-based (lowest-latency Region), Failover (active-passive), Geolocation, Multivalue Answer.

---

## 11. CloudFormation & IaC

### Q31: What is AWS CloudFormation?

- **Infrastructure as Code** — define entire AWS infra (VPC, EC2, RDS) in YAML/JSON
- **Template** = the file; **Stack** = a deployed instance of a template
- If any resource creation fails, CloudFormation **automatically rolls back**

```yaml
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: ami-0abcdef1234567890
```

**CloudFormation vs Terraform:** CloudFormation is AWS-only (YAML/JSON), AWS manages state, auto-rollback. Terraform is multi-cloud (HCL), you manage state, huge community.

---

## 12. Security & Compliance

### Q32: What is AWS KMS?

KMS (Key Management Service) is a managed key vault. You manage encryption keys (CMKs); AWS services (S3, EBS, RDS, Lambda) use them to encrypt/decrypt data. Key usage is logged in CloudTrail; keys are Region-specific.

---

### Q33: AWS Shield, WAF, and Shared Responsibility

- **AWS Shield Standard**: free DDoS protection for everyone. **Advanced**: paid, extra protection + 24/7 support.
- **AWS WAF**: blocks web exploits (SQL injection, XSS) via Web ACL rules; attaches to CloudFront, ALB, API Gateway.

**Shared Responsibility Model:**
- **AWS** (security OF the cloud): physical hardware, data centers, hypervisor, managed-service infrastructure
- **You** (security IN the cloud): your data/encryption, IAM permissions, OS patching (EC2), Security Group config, app security

---

## 13. SNS, SQS & Messaging

### Q34: SQS vs SNS

| | SQS | SNS |
|---|---|---|
| **Pattern** | Queue (point-to-point) | Pub/Sub |
| **Delivery** | Pull (consumers poll) | Push |
| **Persistence** | Yes (up to 14 days) | No |
| **Multiple consumers** | No (one consumer per message) | Yes (all subscribers get it) |
| **Use case** | Task queues, decoupling services | Alerts, fan-out notifications |

- **SQS Standard**: at-least-once delivery, best-effort ordering, high throughput
- **SQS FIFO**: exactly-once, strict ordering, up to 3,000 msg/sec
- **Fan-out pattern**: one SNS topic → multiple SQS queues → separate consumers

---

## 14. ECS, EKS & Containers

### Q35: What is Amazon ECS?

- ECS = **Elastic Container Service** — runs and manages Docker containers on AWS
- **EC2 launch type**: you manage the EC2 instances
- **Fargate launch type**: serverless — define CPU/memory, AWS manages servers

| Concept | Description |
|---|---|
| **Task Definition** | Blueprint (image, CPU, memory, env vars) |
| **Task** | A running container instance |
| **Service** | Keeps N tasks running; integrates with ALB |
| **Cluster** | Group of tasks/services |

**ECS vs EKS:** ECS is AWS-only, simpler, no control-plane cost. EKS runs managed **Kubernetes** — portable but steeper learning curve and per-cluster fee.

---

## 15. Cost Optimization

### Q36: Key cost-saving strategies

- **Right-size** instances using CloudWatch metrics
- **Reserved Instances / Savings Plans** for steady workloads; **Spot** for fault-tolerant ones
- **Auto Scaling** to reduce capacity off-hours
- **S3 Lifecycle policies** to move data to cheaper tiers
- Delete unused resources (idle EIPs, unattached EBS volumes, old snapshots)
- Cache with **CloudFront**; monitor with **Cost Explorer + Budgets**

**Elastic IP vs Public IP:**
| | Elastic IP | Public IP |
|---|---|---|
| **Persistence** | Persists until released | Changes on stop/start |
| **Cost** | Free when attached; $0.005/hr when unattached | Free |
| **Use case** | Static IP for DNS, whitelisting | Temporary public access |

---

## 16. Quick Revision Summary

### AWS Services Cheat Sheet

| Category | Service | One-line Purpose |
|---|---|---|
| **Compute** | EC2 | Virtual servers |
| **Compute** | Lambda | Serverless function execution |
| **Compute** | ECS/EKS | Container orchestration |
| **Storage** | S3 | Object storage (files, backups) |
| **Storage** | EBS | Block storage for EC2 |
| **Storage** | EFS | Shared file system across EC2 |
| **Database** | RDS | Managed relational DB |
| **Database** | Aurora | AWS-native high-performance relational DB |
| **Database** | DynamoDB | Managed NoSQL key-value DB |
| **Database** | ElastiCache | Managed Redis/Memcached |
| **Networking** | VPC | Your private network in AWS |
| **Networking** | Route 53 | DNS and domain registration |
| **Networking** | CloudFront | CDN — cache at edge locations |
| **Load Balancing** | ALB | HTTP/HTTPS load balancer (Layer 7) |
| **Load Balancing** | NLB | TCP/UDP load balancer (Layer 4) |
| **Messaging** | SQS | Message queue (decouples services) |
| **Messaging** | SNS | Pub/Sub notification service |
| **Security** | IAM | User/role/permission management |
| **Security** | KMS | Encryption key management |
| **Security** | Shield / WAF | DDoS + web application firewall |
| **Monitoring** | CloudWatch | Metrics, logs, alarms |
| **Monitoring** | CloudTrail | API audit trail |
| **IaC** | CloudFormation | AWS infra as YAML/JSON templates |

---

### Common Interview Follow-up Questions

- *"What if an AZ goes down?"* → Traffic routes to other AZs if Multi-AZ is configured
- *"How do you make an app highly available?"* → Multi-AZ + Auto Scaling + Load Balancer
- *"How do you reduce latency for global users?"* → CloudFront CDN + Route 53 latency-based routing
- *"How to connect on-premise to AWS?"* → Direct Connect (dedicated line) or Site-to-Site VPN
- *"Horizontal vs vertical scaling?"* → Horizontal = add instances (scale out); Vertical = bigger instance (scale up)
- *"How do private subnet EC2s access the internet?"* → Via NAT Gateway in the public subnet

---

*Last updated: 2026-06-18 | Study path: YouTube → this guide → hands-on AWS Free Tier*
