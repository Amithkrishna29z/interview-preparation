# AWS Cloud Interview Questions & Answers

> 🎯 Commonly asked in cloud/DevOps/backend developer interviews. Focus on understanding **why** AWS services exist, not just **what** they are.

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

**Easy Explanation:** A Region is a physical location in the world where AWS has data centers. Think of it like AWS has offices in different countries — Mumbai, US, London, etc.

**Key Points:**
- A Region is a **geographic area** containing multiple data centers
- Each Region is completely **independent** of other Regions
- AWS has **30+ Regions** globally (as of 2024)
- You choose a Region based on: latency, compliance, cost, and available services

**Examples:**
| Region Code | Location |
|---|---|
| `ap-south-1` | Mumbai, India |
| `us-east-1` | N. Virginia, USA (oldest, most services) |
| `eu-west-1` | Ireland, Europe |
| `ap-southeast-1` | Singapore |

**Interview Tip:** Always say "I choose a Region close to my users to reduce latency, and based on data residency laws (e.g., GDPR for Europe)."

---

### Q2: What is an Availability Zone (AZ)?

**Easy Explanation:** An AZ is like a separate building inside the same city. If one building catches fire, the other buildings in that city still work.

**Key Points:**
- Each Region has **2 to 6 AZs**
- Each AZ is one or more **physically separate data centers**
- AZs in the same Region are connected by **low-latency, high-bandwidth** private fiber links
- Deploying across multiple AZs provides **High Availability (HA)**

```
Region: ap-south-1 (Mumbai)
├── AZ: ap-south-1a  → Data Center A
├── AZ: ap-south-1b  → Data Center B
└── AZ: ap-south-1c  → Data Center C
```

**Real-world use:** Deploy your EC2 instances in 2+ AZs. If AZ-a goes down, traffic automatically routes to AZ-b.

---

### Q3: What is an Edge Location / CloudFront POP?

**Easy Explanation:** Edge Locations are like delivery warehouses close to customers. Instead of shipping from a main warehouse (Region) every time, the nearest warehouse (Edge Location) delivers fast.

**Key Points:**
- Used by **Amazon CloudFront** (CDN) and **Route 53**
- There are **400+ Edge Locations** globally — far more than Regions
- Content is **cached** at Edge Locations for fast delivery
- Reduces latency for end users worldwide

```
User in Chennai
    ↓
Edge Location (Chennai)  ← serves cached content (fast!)
    ↓ (on cache miss)
Origin Server (ap-south-1 Region)
```

---

## 2. IAM – Identity & Access Management

### Q4: What is IAM and why is it important?

**Easy Explanation:** IAM is the security system that controls **who can access what** in your AWS account. It's like a key card system in an office — different people have access to different rooms.

**Key Points:**
- **IAM** = Identity and Access Management
- Controls **authentication** (who you are) and **authorization** (what you can do)
- IAM is a **global service** — not Region-specific
- Everything in AWS passes through IAM

**Core IAM Components:**

| Component | What it is | Analogy |
|---|---|---|
| **User** | A person who logs into AWS | An employee |
| **Group** | A collection of users | A department |
| **Role** | Temporary identity assumed by services/users | A security badge for a visitor |
| **Policy** | JSON document that defines permissions | A rulebook |

---

### Q5: What is the difference between IAM User, Group, and Role?

**IAM User:**
- Represents a **person or application**
- Has permanent long-term credentials (username/password or access keys)
- Example: A developer who logs into the AWS Console

**IAM Group:**
- A **collection of IAM users**
- Permissions assigned to the group apply to all members
- Example: "Developers" group → all developers get EC2 read access

**IAM Role:**
- An identity with permissions, but **no permanent credentials**
- Assumed **temporarily** by users, services, or applications
- Example: An EC2 instance assumes a role to access S3 — no hardcoded keys needed

```
Bad practice ❌:
  EC2 instance stores AWS_ACCESS_KEY and AWS_SECRET_KEY in env variables

Good practice ✅:
  EC2 instance is assigned an IAM Role with S3 permissions
  SDK automatically uses the role's temporary credentials
```

---

### Q6: What is an IAM Policy? Explain the structure.

**Easy Explanation:** A Policy is a JSON document that says "Allow this action on this resource under these conditions."

**Policy Structure:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",           // Allow or Deny
      "Action": [                  // Which API calls
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*",  // Which resource
      "Condition": {               // Optional: extra conditions
        "IpAddress": {
          "aws:SourceIp": "192.168.1.0/24"
        }
      }
    }
  ]
}
```

**Types of Policies:**
| Type | Description |
|---|---|
| **Managed Policy (AWS)** | Pre-built by AWS (e.g., `AmazonS3ReadOnlyAccess`) |
| **Managed Policy (Customer)** | Custom policies you create and reuse |
| **Inline Policy** | Policy embedded directly into one user/role — not reusable |

**Golden Rule:** Follow the **Principle of Least Privilege** — give only the permissions needed, nothing more.

---

### Q7: What is the Root User and why should you not use it?

**Key Points:**
- The **Root User** is the account created when you first sign up for AWS
- It has **unrestricted access** to everything — billing, account closure, all services
- Best practice: **Never use root for daily tasks**
- Root should only be used for: changing account email, enabling MFA, closing the account

**Root User Best Practices:**
- Enable MFA (Multi-Factor Authentication) on root immediately
- Create an IAM admin user for daily operations
- Delete or never create root access keys

---

## 3. EC2 – Elastic Compute Cloud

### Q8: What is EC2?

**Easy Explanation:** EC2 gives you a virtual computer in the cloud. You can rent it by the hour, install your software, and run your applications — just like having your own server but without buying hardware.

**Key Points:**
- EC2 = **Elastic Compute Cloud**
- Provides **resizable compute capacity** in the cloud
- You only pay for what you use (per second/hour)
- Highly configurable: CPU, RAM, storage, OS, network

**EC2 Lifecycle:**
```
Launch → Pending → Running → (Stopping) → Stopped → Terminated
                                  ↑
                              Rebooting
```

---

### Q9: What are EC2 Instance Types?

**Easy Explanation:** Instance types are like choosing a car — sedan, SUV, sports car, truck. Each is optimized for different workloads.

**Instance Type Families:**

| Family | Optimized For | Example Use Case |
|---|---|---|
| **General Purpose** (`t3`, `m6i`) | Balanced CPU/memory/network | Web servers, small DBs |
| **Compute Optimized** (`c6i`, `c5`) | High CPU performance | Batch processing, gaming servers |
| **Memory Optimized** (`r6i`, `x2`) | Large RAM | In-memory DBs, big data (Redis, SAP) |
| **Storage Optimized** (`i3`, `d3`) | High disk I/O | OLTP databases, data warehousing |
| **Accelerated Computing** (`p4`, `g4`) | GPU workloads | ML training, video rendering |

**Naming convention:**
```
m6i.xlarge
│││  └─ Size (nano < micro < small < medium < large < xlarge < 2xlarge...)
││└── Generation (6 = 6th gen)
│└─── Family variant (i = Intel)
└──── Family (m = General Purpose)
```

---

### Q10: What are the EC2 Purchasing Options?

**Key Options:**

| Option | Best For | Discount vs On-Demand |
|---|---|---|
| **On-Demand** | Short-term, unpredictable workloads | 0% (baseline price) |
| **Reserved Instances (RI)** | Steady, predictable usage (1 or 3 year) | Up to 72% off |
| **Savings Plans** | Flexible, committed spend | Up to 72% off |
| **Spot Instances** | Fault-tolerant, flexible, interruptible | Up to 90% off |
| **Dedicated Hosts** | Compliance/licensing needs | Most expensive |

**Spot Instances — Important to know:**
- You bid on unused EC2 capacity
- AWS can **terminate your instance with 2-minute warning** if capacity is needed back
- Great for: batch jobs, data analysis, CI/CD pipelines, ML training
- NOT suitable for: databases, web servers that can't handle interruption

---

### Q11: What is the difference between a Security Group and NACL?

**Easy Explanation:**
- **Security Group** = Firewall at the **instance level** (like a door lock)
- **NACL** = Firewall at the **subnet level** (like a building entrance security)

| Feature | Security Group | NACL (Network ACL) |
|---|---|---|
| **Level** | Instance (ENI) level | Subnet level |
| **State** | Stateful (return traffic auto-allowed) | Stateless (must allow inbound AND outbound) |
| **Rules** | Allow rules only | Allow AND Deny rules |
| **Rule order** | All rules evaluated | Rules evaluated in order (lowest number first) |
| **Default** | All outbound allowed, all inbound denied | All traffic allowed |

```
Internet
    ↓
NACL (Subnet boundary) → checks inbound rule
    ↓
Security Group (Instance boundary) → checks inbound rule
    ↓
EC2 Instance
```

---

### Q12: What is an AMI?

**Key Points:**
- **AMI** = Amazon Machine Image
- A **template** (snapshot) used to launch EC2 instances
- Contains: OS, application software, configurations, EBS volume mappings
- AMIs are **Region-specific** (copy to another Region to use there)

**Types:**
| Type | Description |
|---|---|
| **AWS-provided** | Amazon Linux 2, Ubuntu, Windows Server |
| **AWS Marketplace** | Third-party software (e.g., pre-installed Jenkins) |
| **Custom AMI** | Your own image with your app pre-configured |
| **Community AMI** | Public images shared by others |

**Use case:** Create a custom AMI of your configured web server → launch 10 identical instances in seconds.

---

### Q13: What is the difference between EBS, Instance Store, and EFS?

| Feature | EBS | Instance Store | EFS |
|---|---|---|---|
| **Type** | Block storage | Ephemeral block storage | File storage (NFS) |
| **Persistence** | Persists after stop/terminate | **Lost when instance stops** | Persists, shared |
| **Use case** | Root volumes, databases | Temp buffers, caches | Shared file system across instances |
| **Replication** | Within 1 AZ | None | Multi-AZ |
| **Cost** | Moderate | Included in instance price | Higher |

---

## 4. S3 – Simple Storage Service

### Q14: What is S3?

**Easy Explanation:** S3 is like an infinite hard drive in the cloud. You store files (called **objects**) in folders (called **buckets**). It never runs out of space and your data is automatically replicated.

**Key Points:**
- S3 = **Simple Storage Service** — object storage
- Stores any type of file: images, videos, backups, logs, static websites
- **Unlimited storage**, objects can be 0 bytes to 5 TB each
- Object URL format: `https://<bucket-name>.s3.<region>.amazonaws.com/<key>`
- Bucket names must be **globally unique** across all AWS accounts

---

### Q15: What are S3 Storage Classes?

**Easy Explanation:** Like mail delivery options — express, standard, economy. Choose based on how often you need to access the data.

| Storage Class | Access Pattern | Retrieval Time | Cost |
|---|---|---|---|
| **S3 Standard** | Frequent access | Milliseconds | Highest |
| **S3 Standard-IA** | Infrequent, but fast when needed | Milliseconds | Lower storage, retrieval fee |
| **S3 One Zone-IA** | Infrequent, non-critical | Milliseconds | Cheaper (1 AZ only) |
| **S3 Glacier Instant** | Archive with instant access | Milliseconds | Low |
| **S3 Glacier Flexible** | Archive, 1-5 minute retrieval | Minutes | Very low |
| **S3 Glacier Deep Archive** | Long-term archive (years) | Up to 12 hours | Cheapest |
| **S3 Intelligent-Tiering** | Unknown access pattern | Milliseconds | Auto-moves between tiers |

---

### Q16: How do you secure an S3 bucket?

**Key Security Controls:**

1. **Block Public Access** — enabled by default on new buckets; never turn off unless required
2. **Bucket Policy** — JSON policy attached to the bucket (resource-based policy)
3. **IAM Policy** — attached to users/roles
4. **S3 ACLs** — older method, mostly replaced by bucket policies
5. **Encryption** — SSE-S3 (AWS managed), SSE-KMS (your KMS key), SSE-C (your own key)
6. **Versioning** — keeps all versions; protects against accidental delete/overwrite
7. **MFA Delete** — requires MFA to delete versions

```
// Bucket policy: Allow only specific account
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringNotEquals": {
        "aws:PrincipalAccount": "123456789012"
      }
    }
  }]
}
```

---

### Q17: What is S3 Versioning?

**Key Points:**
- When enabled, S3 keeps **every version** of every object
- Protects against accidental deletes and overwrites
- A "delete" just adds a **delete marker** — you can restore
- **Lifecycle policies** can auto-expire old versions to control cost

```
bucket/photo.jpg  (version: abc123)  ← current
bucket/photo.jpg  (version: def456)  ← previous
bucket/photo.jpg  (version: ghi789)  ← oldest
```

---

## 5. VPC – Virtual Private Cloud

### Q18: What is a VPC?

**Easy Explanation:** A VPC is your own private section of the AWS cloud, like having a private office floor in a large shared building. Other tenants can't access your floor.

**Key Points:**
- VPC = **Virtual Private Cloud**
- Logically isolated network within AWS
- You control: IP address range (CIDR), subnets, route tables, gateways, security
- Each AWS account gets a **default VPC** per Region

**VPC Components:**
```
VPC (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24)   → has route to Internet Gateway
│     └── EC2 (web server, load balancer)
├── Private Subnet (10.0.2.0/24)  → no direct internet route
│     └── EC2 (app servers, DBs)
├── Internet Gateway               → allows internet traffic
├── NAT Gateway                    → lets private subnet access internet (outbound only)
└── Route Tables                   → routing rules per subnet
```

---

### Q19: What is the difference between a Public and Private Subnet?

| Feature | Public Subnet | Private Subnet |
|---|---|---|
| **Internet access** | Yes (via Internet Gateway) | No direct internet |
| **Use case** | Web servers, load balancers, bastion hosts | App servers, databases |
| **Outbound internet** | Direct via IGW | Via NAT Gateway (in public subnet) |
| **Inbound internet** | Yes (with Security Group rules) | No |

---

### Q20: What is a NAT Gateway?

**Easy Explanation:** NAT Gateway lets your private EC2 instances **download updates from the internet** without exposing them to inbound internet traffic. Like a one-way mirror.

**Key Points:**
- NAT = **Network Address Translation**
- Sits in the **public subnet**
- Private subnet instances → NAT Gateway → Internet (outbound only)
- Inbound internet connections cannot reach the private instance
- NAT Gateway is **managed by AWS** (NAT Instance is self-managed, older approach)

```
Private EC2 → NAT Gateway (Public Subnet) → Internet Gateway → Internet
                                                               ↓ (response)
Private EC2 ← NAT Gateway ← Internet Gateway ← Internet
```

---

### Q21: What is VPC Peering?

**Easy Explanation:** VPC Peering connects two VPCs privately so they can communicate as if they were on the same network. Like connecting two office floors with a private corridor.

**Key Points:**
- Connects VPCs within the **same Region or across Regions**
- Can peer across **different AWS accounts**
- Traffic stays on the AWS private network (never goes through internet)
- **NOT transitive**: If A ↔ B and B ↔ C, A cannot reach C through B

---

## 6. RDS & Databases

### Q22: What is Amazon RDS?

**Easy Explanation:** RDS manages your relational database so you don't have to worry about OS patches, backups, hardware failures — AWS handles it. You just write SQL.

**Key Points:**
- RDS = **Relational Database Service**
- Supports: **MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora**
- AWS manages: hardware, OS patching, automated backups, software updates
- You manage: schema, indexes, query tuning, application logic

**What RDS automates:**
- Automated backups (up to 35 days retention)
- Multi-AZ failover
- Read replicas for scaling reads
- Storage auto-scaling

---

### Q23: What is Multi-AZ in RDS?

**Easy Explanation:** Multi-AZ is like having a hot spare — an identical standby database running in another AZ. If the primary fails, the standby automatically takes over in about 1-2 minutes.

**Key Points:**
- **Synchronous replication** to standby in a different AZ
- Automatic failover (DNS endpoint stays the same — your app reconnects)
- Purpose: **High Availability (HA)**, NOT performance (standby is not readable)
- Failover triggers: AZ failure, primary instance failure, instance type change, OS patching

```
Primary (AZ-a) → Synchronous replication → Standby (AZ-b)
     ↓ (fails)
DNS automatically points to Standby → Standby becomes Primary
```

---

### Q24: What is the difference between Multi-AZ and Read Replicas?

| Feature | Multi-AZ | Read Replica |
|---|---|---|
| **Purpose** | High Availability | Performance / Read Scaling |
| **Replication** | Synchronous | Asynchronous |
| **Readable?** | No (standby is passive) | Yes |
| **Failover** | Automatic | Manual promotion |
| **Cross-Region** | No | Yes |
| **Cost** | Higher (2x instance cost) | Moderate |

---

### Q25: What is Amazon Aurora?

**Key Points:**
- AWS's **cloud-native relational database** — MySQL and PostgreSQL compatible
- Up to **5x faster than MySQL**, **3x faster than PostgreSQL**
- Storage: auto-scales from 10 GB to 128 TB
- **6 copies of data** across 3 AZs (2 copies per AZ)
- Aurora Serverless: auto-scales compute capacity based on load

**Aurora vs RDS:**
| | Aurora | Standard RDS |
|---|---|---|
| Performance | Much higher | Good |
| Storage | Shared cluster storage | Per-instance EBS |
| Replicas | Up to 15 | Up to 5 |
| Failover | ~30 seconds | ~1-2 minutes |

---

## 7. Load Balancing & Auto Scaling

### Q26: What is Elastic Load Balancing (ELB)?

**Easy Explanation:** A Load Balancer distributes incoming traffic across multiple EC2 instances so no single instance gets overwhelmed. Like a traffic cop directing cars to multiple lanes.

**Types of Load Balancers:**

| Type | Layer | Use Case |
|---|---|---|
| **ALB** (Application Load Balancer) | Layer 7 (HTTP/HTTPS) | Web apps, microservices, path-based routing |
| **NLB** (Network Load Balancer) | Layer 4 (TCP/UDP) | Ultra-high performance, gaming, IoT, static IP |
| **GWLB** (Gateway Load Balancer) | Layer 3 | Third-party virtual appliances (firewalls) |
| **CLB** (Classic Load Balancer) | Layer 4/7 | Legacy, avoid for new projects |

**ALB Features:**
- Path-based routing: `/api/*` → API servers, `/web/*` → Web servers
- Host-based routing: `api.example.com` vs `app.example.com`
- HTTP to HTTPS redirects
- Target groups: EC2, ECS tasks, Lambda, IP addresses

---

### Q27: What is Auto Scaling?

**Easy Explanation:** Auto Scaling automatically adds or removes EC2 instances based on demand. During high traffic, it adds instances. During low traffic, it removes them to save cost.

**Key Components:**
- **Auto Scaling Group (ASG):** The group of EC2 instances managed together
- **Launch Template:** The blueprint used to create new instances
- **Scaling Policies:** Rules that trigger scaling

**Scaling Policy Types:**
| Policy | How it works |
|---|---|
| **Target Tracking** | Keep a metric at a target (e.g., CPU at 50%) |
| **Step Scaling** | Scale by X instances when alarm breaches thresholds |
| **Simple Scaling** | Add/remove fixed number on alarm |
| **Scheduled Scaling** | Scale at specific times (e.g., more capacity every Monday 9am) |
| **Predictive Scaling** | ML-based forecast of future traffic |

```
CloudWatch Alarm (CPU > 70%)
        ↓
Auto Scaling Policy triggered
        ↓
New EC2 instances launched from Launch Template
        ↓
Registered with Load Balancer
```

---

## 8. Lambda & Serverless

### Q28: What is AWS Lambda?

**Easy Explanation:** Lambda lets you run code without managing any servers. You upload your function, AWS runs it when triggered, and you only pay for the exact milliseconds it runs.

**Key Points:**
- **Serverless** compute — no EC2 to manage, patch, or scale
- Supports: Python, Java, Node.js, Go, Ruby, .NET, custom runtimes
- Max execution time: **15 minutes**
- Memory: 128 MB to 10 GB (CPU scales with memory)
- Billed per **number of requests** + **duration (GB-seconds)**

**Lambda Triggers (Event Sources):**
```
API Gateway        → Lambda (REST API backend)
S3 Event           → Lambda (process uploaded file)
DynamoDB Stream    → Lambda (react to DB changes)
SQS/SNS            → Lambda (message processing)
EventBridge (cron) → Lambda (scheduled job)
Kinesis            → Lambda (stream processing)
```

---

### Q29: What are the limitations of Lambda?

| Limit | Value |
|---|---|
| Execution timeout | 15 minutes max |
| Deployment package size | 50 MB (zipped), 250 MB (unzipped) |
| Ephemeral storage `/tmp` | 512 MB to 10 GB |
| Concurrent executions | 1,000 per region (can request increase) |
| Memory | 128 MB – 10 GB |

**Cold Start Issue:**
- First invocation after idle period → Lambda must initialize the container → adds latency
- Solution: Use **Provisioned Concurrency** to keep functions warm
- Java/Spring Boot cold starts are notably longer than Node.js/Python

---

### Q30: What is the difference between Lambda and EC2?

| | Lambda | EC2 |
|---|---|---|
| **Server management** | None (serverless) | You manage the OS, patches |
| **Scaling** | Automatic, per-request | Manual or Auto Scaling |
| **Billing** | Per millisecond of execution | Per hour/second (even when idle) |
| **Max runtime** | 15 minutes | Unlimited |
| **Best for** | Event-driven, short tasks | Long-running apps, custom OS |

---

## 9. CloudWatch & Monitoring

### Q31: What is Amazon CloudWatch?

**Easy Explanation:** CloudWatch is the monitoring and observability service for AWS. It collects logs, metrics, and events from your AWS resources — like a security camera system for your cloud.

**CloudWatch Components:**

| Component | Purpose |
|---|---|
| **Metrics** | Numerical data over time (CPU %, request count) |
| **Logs** | Log groups and streams from EC2, Lambda, etc. |
| **Alarms** | Trigger actions when a metric crosses a threshold |
| **Dashboards** | Visual monitoring panels |
| **Events / EventBridge** | React to AWS state changes on a schedule |
| **Container Insights** | Metrics for ECS/EKS |

**Default EC2 Metrics (free):** CPU, Network In/Out, Disk Read/Write, Status Check
**Custom Metrics (paid):** RAM usage, Disk space (need CloudWatch Agent installed)

---

### Q32: What is CloudWatch vs CloudTrail?

| | CloudWatch | CloudTrail |
|---|---|---|
| **Purpose** | Performance & operational monitoring | Audit trail of API calls |
| **What it tracks** | Metrics, logs, resource performance | Who did what, when, from where |
| **Use case** | Alert when CPU > 90% | Audit "who deleted that S3 bucket?" |
| **Log type** | Application/system logs | Management events, data events |
| **Retention** | Configurable | 90 days free, S3 for long-term |

**Simple rule:**
- **CloudWatch** = "Is my application healthy?"
- **CloudTrail** = "Who changed what in my account?"

---

## 10. Route 53 & DNS

### Q33: What is Amazon Route 53?

**Easy Explanation:** Route 53 is AWS's DNS service. It translates human-friendly domain names (`www.myapp.com`) into IP addresses that computers can use.

**Key Points:**
- Highly available and scalable **DNS web service**
- Supports domain registration
- Named "Route 53" because DNS uses **port 53**

**Route 53 Routing Policies:**

| Policy | Use Case |
|---|---|
| **Simple** | Single resource, no health checks |
| **Weighted** | A/B testing, split traffic (70/30) |
| **Latency-based** | Route to lowest-latency Region |
| **Failover** | Active-passive: switch to backup if primary fails |
| **Geolocation** | Route by user's country/continent |
| **Geoproximity** | Route by distance, with bias adjustment |
| **Multivalue Answer** | Return multiple IPs (basic load balancing) |

---

## 11. CloudFormation & IaC

### Q34: What is AWS CloudFormation?

**Easy Explanation:** CloudFormation lets you define your entire AWS infrastructure in a text file (YAML or JSON). You can spin up or tear down a complete environment — VPC, EC2, RDS, everything — with a single command.

**Key Points:**
- **Infrastructure as Code (IaC)** — version control your infra
- Resources are defined in **Templates** (YAML or JSON)
- A **Stack** is a deployed instance of a template
- If a resource creation fails, CloudFormation **automatically rolls back**

**Template Structure:**
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "My web app infrastructure"

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro

Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: ami-0abcdef1234567890

Outputs:
  PublicIP:
    Value: !GetAtt MyEC2Instance.PublicIp
```

---

### Q35: CloudFormation vs Terraform — key differences?

| | CloudFormation | Terraform |
|---|---|---|
| **Provider** | AWS only | Multi-cloud (AWS, Azure, GCP, etc.) |
| **Language** | YAML / JSON | HCL (HashiCorp Configuration Language) |
| **State management** | AWS manages state | You manage `terraform.tfstate` |
| **Rollback** | Automatic on failure | Manual (`terraform destroy`) |
| **Cost** | Free | Free (open source) |
| **Ecosystem** | AWS native, tight integration | Huge community, multi-cloud |

---

## 12. Security & Compliance

### Q36: What is AWS KMS?

**Easy Explanation:** KMS is like a master key vault. You store encryption keys here and AWS services use them to encrypt/decrypt your data — without you ever touching the raw keys.

**Key Points:**
- KMS = **Key Management Service**
- Create and manage **Customer Managed Keys (CMK)**
- Integrated with S3, EBS, RDS, Lambda, Secrets Manager, and more
- Every key usage is **logged in CloudTrail** (audit trail)
- Keys are **Region-specific** — need to copy to another Region to use there

**Encryption types:**
| Type | Key managed by |
|---|---|
| **SSE-S3** | AWS (automatic, no control) |
| **SSE-KMS** | You (via KMS CMK) |
| **SSE-C** | You (provide your own key per request) |

---

### Q37: What is AWS Shield and WAF?

**AWS Shield:**
- DDoS (Distributed Denial of Service) protection
- **Shield Standard**: free, automatically protects all AWS customers
- **Shield Advanced**: paid ($3,000/month), advanced DDoS protection + 24/7 DRT team support

**AWS WAF (Web Application Firewall):**
- Protects against web exploits: **SQL injection, XSS, bad bots**
- Define **Web ACL rules**: block specific IPs, rate limit, geo-block
- Integrates with: CloudFront, ALB, API Gateway, AppSync

```
User request
    ↓
CloudFront + WAF (filters malicious requests)
    ↓
ALB + Shield (DDoS protection)
    ↓
EC2 / Application
```

---

### Q38: What is the AWS Shared Responsibility Model?

**Easy Explanation:** AWS is responsible for the cloud infrastructure (hardware, networking, data centers). You are responsible for what you put IN the cloud (your OS configs, application, data, user access).

```
┌─────────────────────────────────────────────────────┐
│              CUSTOMER RESPONSIBILITY                 │
│  "Security IN the cloud"                            │
│  ✔ Your data (encryption)                           │
│  ✔ IAM users, roles, permissions                    │
│  ✔ OS patching (for EC2)                            │
│  ✔ Application security                             │
│  ✔ Security Group configurations                    │
├─────────────────────────────────────────────────────┤
│              AWS RESPONSIBILITY                      │
│  "Security OF the cloud"                            │
│  ✔ Physical data centers                            │
│  ✔ Hardware (servers, networking)                   │
│  ✔ Hypervisor / virtualization layer                │
│  ✔ Managed service infrastructure (RDS, Lambda)     │
└─────────────────────────────────────────────────────┘
```

---

## 13. SNS, SQS & Messaging

### Q39: What is Amazon SQS?

**Easy Explanation:** SQS is a message queue. Producer services drop messages into the queue, consumer services pick them up and process them at their own pace. It decouples your application so components don't need to be available at the same time.

**Key Points:**
- SQS = **Simple Queue Service**
- Messages stay in queue until consumed (up to 14 days)
- **Standard Queue**: At-least-once delivery, best-effort ordering, high throughput
- **FIFO Queue**: Exactly-once delivery, strict ordering, up to 3,000 msg/sec

**Use case:**
```
Order Service → SQS Queue → Payment Processor (scales independently)
                         └→ Notification Service (also reads)
```

---

### Q40: What is Amazon SNS?

**Easy Explanation:** SNS is a pub/sub messaging service. One publisher sends a message to a **topic**, and all subscribers (SQS, Lambda, email, SMS, HTTP) receive it instantly.

**Key Points:**
- SNS = **Simple Notification Service**
- **Push-based** (vs SQS which is poll-based)
- Fan-out pattern: one message → many subscribers

**SNS vs SQS:**
| | SNS | SQS |
|---|---|---|
| **Pattern** | Pub/Sub | Queue (point-to-point) |
| **Delivery** | Push | Pull (consumers poll) |
| **Persistence** | No (message not stored) | Yes (up to 14 days) |
| **Multiple consumers** | Yes (all subscribers get it) | No (one consumer per message) |
| **Use case** | Alerts, fan-out notifications | Task queues, decoupling |

**Fan-out Pattern (SNS + SQS):**
```
Event → SNS Topic → SQS Queue A → Lambda (send email)
                 └→ SQS Queue B → Lambda (update DB)
                 └→ SQS Queue C → Lambda (push notification)
```

---

## 14. ECS, EKS & Containers

### Q41: What is Amazon ECS?

**Easy Explanation:** ECS is AWS's container management service. Instead of running Docker containers manually on EC2 instances, ECS handles scheduling, scaling, and health checks for you.

**Key Points:**
- ECS = **Elastic Container Service**
- Runs Docker containers on AWS
- **Launch types:**
  - **EC2 Launch Type**: You manage the EC2 instances (servers)
  - **Fargate Launch Type**: Serverless — you only define CPU/memory, AWS handles servers

**ECS Concepts:**
| Concept | Description |
|---|---|
| **Task Definition** | Blueprint for your container (image, CPU, memory, env vars) |
| **Task** | A running instance of a Task Definition |
| **Service** | Ensures N tasks are always running, integrates with ALB |
| **Cluster** | Group of tasks/services |

---

### Q42: What is the difference between ECS and EKS?

| | ECS | EKS |
|---|---|---|
| **Full name** | Elastic Container Service | Elastic Kubernetes Service |
| **Orchestrator** | AWS proprietary | Kubernetes (open-source) |
| **Learning curve** | Lower | Higher (K8s knowledge needed) |
| **Portability** | AWS-only | Portable across any K8s cluster |
| **Cost** | No control plane cost | $0.10/hour per cluster |
| **Best for** | Simple container workloads, AWS-native teams | Complex microservices, K8s expertise |

---

## 15. Cost Optimization

### Q43: What are the key strategies to reduce AWS costs?

**Key Strategies:**

1. **Right-sizing**: Match instance size to actual workload (use CloudWatch metrics to find over-provisioned instances)

2. **Reserved Instances / Savings Plans**: Commit to 1 or 3 years for steady workloads → up to 72% savings

3. **Spot Instances**: Use for fault-tolerant workloads (batch, ML training) → up to 90% savings

4. **Auto Scaling**: Scale down during off-hours to avoid paying for idle capacity

5. **S3 Lifecycle Policies**: Auto-move old data to cheaper storage classes (Standard → Glacier)

6. **Delete unused resources**: Unattached EBS volumes, idle EIPs (Elastic IPs), old snapshots

7. **CloudFront**: Reduce S3/EC2 data transfer costs by caching at edge

8. **AWS Cost Explorer + Budgets**: Set budget alerts before overspending

---

### Q44: What is the difference between Elastic IP and Public IP?

| | Elastic IP | Public IP |
|---|---|---|
| **Persistence** | Persists until released | Changes when instance stops/starts |
| **Cost** | Free when attached to running instance; charged when unattached ($0.005/hr) | Free |
| **Control** | You own it — can move between instances | AWS assigns randomly |
| **Use case** | Static IP for DNS records, whitelisting | Temporary public access |

---

## 16. Quick Revision Summary

### AWS Services Cheat Sheet

| Category | Service | One-line Purpose |
|---|---|---|
| **Compute** | EC2 | Virtual servers in the cloud |
| **Compute** | Lambda | Serverless function execution |
| **Compute** | ECS/EKS | Container orchestration |
| **Storage** | S3 | Object storage (files, backups) |
| **Storage** | EBS | Block storage for EC2 (like a hard drive) |
| **Storage** | EFS | Shared file system across EC2 instances |
| **Database** | RDS | Managed relational DB (MySQL, Postgres) |
| **Database** | Aurora | AWS-native high-performance relational DB |
| **Database** | DynamoDB | Managed NoSQL key-value database |
| **Database** | ElastiCache | Managed Redis/Memcached (in-memory cache) |
| **Networking** | VPC | Your private network in AWS |
| **Networking** | Route 53 | DNS service and domain registration |
| **Networking** | CloudFront | CDN — cache content at edge locations |
| **Load Balancing** | ALB | HTTP/HTTPS load balancer (Layer 7) |
| **Load Balancing** | NLB | TCP/UDP load balancer (Layer 4) |
| **Messaging** | SQS | Message queue (decouples services) |
| **Messaging** | SNS | Pub/Sub notification service |
| **Security** | IAM | User/role/permission management |
| **Security** | KMS | Encryption key management |
| **Security** | Shield | DDoS protection |
| **Security** | WAF | Web application firewall |
| **Monitoring** | CloudWatch | Metrics, logs, alarms |
| **Monitoring** | CloudTrail | API audit trail |
| **IaC** | CloudFormation | AWS infrastructure as YAML/JSON templates |
| **IaC** | CDK | Define infra in TypeScript/Python code |

---

### Key Concepts to Always Remember

| Concept | Remember This |
|---|---|
| **High Availability** | Deploy across multiple AZs |
| **Disaster Recovery** | Deploy across multiple Regions |
| **Security** | Principle of Least Privilege in IAM |
| **Scaling** | Use Auto Scaling + Load Balancer together |
| **Cost** | Spot for batch, Reserved for steady, On-Demand for unpredictable |
| **Serverless** | Lambda for event-driven, short tasks (<15 min) |
| **Shared Responsibility** | AWS secures the cloud; you secure what's in it |
| **Stateful vs Stateless** | Security Groups are stateful; NACLs are stateless |

---

### Common Interview Follow-up Questions

- *"What happens if an Availability Zone goes down?"* → Traffic auto-routes to other AZs if Multi-AZ is configured
- *"How do you make an application highly available?"* → Multi-AZ deployment + Auto Scaling + Load Balancer
- *"How do you reduce latency for global users?"* → CloudFront CDN + Route 53 latency-based routing
- *"How do you connect your on-premise network to AWS?"* → AWS Direct Connect (dedicated line) or Site-to-Site VPN
- *"What is the difference between horizontal and vertical scaling?"* → Horizontal = add more instances (scale out); Vertical = bigger instance (scale up)
- *"How do EC2 instances in a private subnet access the internet?"* → Via NAT Gateway in the public subnet

---

*Last updated: June 2026 | Study path: YouTube → this guide → hands-on AWS Free Tier*
