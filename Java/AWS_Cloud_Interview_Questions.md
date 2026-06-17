# AWS Cloud Interview Questions & Answers

> 🎯 Commonly asked in cloud/DevOps/backend developer interviews. Focus on understanding **why** AWS services exist, not just **what** they are.

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

**Easy Explanation:** Edge Locations are like delivery warehouses close to customers — the nearest one serves cached content fast instead of going back to the Region.

**Key Points:**
- Used by **Amazon CloudFront** (CDN) and **Route 53**
- 400+ Edge Locations globally; content is **cached** for low latency to users

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

**Easy Explanation:** A Policy is a JSON document that says "Allow (or Deny) this **Action** on this **Resource**, optionally under a **Condition**."

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

- **AWS managed** (e.g., `AmazonS3ReadOnlyAccess`), **customer managed** (reusable), or **inline** (one user/role).
- **Golden Rule:** Principle of **Least Privilege** — grant only what's needed.

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

**EC2 Lifecycle:** Launch → Pending → Running → Stopping → Stopped → Terminated (a reboot keeps it Running).

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

**Naming** (e.g. `m6i.xlarge`): family (`m`) + generation (`6`) + variant (`i`) + size (`xlarge`).

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

**Spot Instances:** run on unused capacity at up to 90% off, but AWS can reclaim them with a 2-minute warning. Good for batch/CI/ML; not for databases or stateful web servers.

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

Traffic order: Internet → NACL (subnet boundary) → Security Group (instance boundary) → EC2.

---

### Q12: What is an AMI?

**Key Points:**
- **AMI** = Amazon Machine Image — a **template** (OS + software + config) used to launch EC2 instances
- AMIs are **Region-specific**; can be AWS-provided, Marketplace, or your own **custom** image
- **Use case:** bake a custom AMI of your configured server → launch many identical instances fast

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

| Storage Class | Access Pattern | Cost |
|---|---|---|
| **S3 Standard** | Frequent access | Highest |
| **S3 Standard-IA** | Infrequent, fast when needed | Lower storage + retrieval fee |
| **S3 Glacier** (Instant/Flexible/Deep Archive) | Archive, retrieval ms to 12 hrs | Very low to cheapest |
| **S3 Intelligent-Tiering** | Unknown pattern, auto-moves tiers | Variable |

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

---

### Q17: What is S3 Versioning?

**Key Points:**
- When enabled, S3 keeps **every version** of every object
- Protects against accidental deletes and overwrites
- A "delete" just adds a **delete marker** — you can restore
- **Lifecycle policies** can auto-expire old versions to control cost

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

**Awareness summary:** A NAT (Network Address Translation) Gateway sits in a **public subnet** and lets **private** EC2 instances reach the internet **outbound only** (e.g., to download updates) while blocking inbound connections. It's AWS-managed.

---

### Q21: What is VPC Peering?

**Awareness summary:** VPC Peering privately connects two VPCs (same or cross-Region, even across accounts) so they communicate over the AWS network. It is **not transitive** (A↔B and B↔C does not give A↔C).

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
- Failover triggers: AZ failure, primary instance failure, OS patching

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

**Awareness summary:** Aurora is AWS's cloud-native, MySQL/PostgreSQL-compatible relational database — faster than standard RDS, with auto-scaling shared storage (6 copies across 3 AZs), up to 15 replicas, and fast failover. Aurora Serverless auto-scales compute to load.

---

## 7. Load Balancing & Auto Scaling

### Q26: What is Elastic Load Balancing (ELB)?

**Easy Explanation:** A Load Balancer distributes incoming traffic across multiple EC2 instances so no single instance gets overwhelmed. Like a traffic cop directing cars to multiple lanes.

**Types of Load Balancers:**

| Type | Layer | Use Case |
|---|---|---|
| **ALB** (Application Load Balancer) | Layer 7 (HTTP/HTTPS) | Web apps, microservices, path-based routing |
| **NLB** (Network Load Balancer) | Layer 4 (TCP/UDP) | Ultra-high performance, static IP |

(GWLB exists for third-party appliances; CLB is legacy — avoid for new projects.)

**ALB Features:** path-based (`/api/*`) and host-based (`api.example.com`) routing, HTTP→HTTPS redirects, and target groups (EC2, ECS tasks, Lambda, IPs).

---

### Q27: What is Auto Scaling?

**Easy Explanation:** Auto Scaling automatically adds or removes EC2 instances based on demand. During high traffic, it adds instances. During low traffic, it removes them to save cost.

**Key Components:**
- **Auto Scaling Group (ASG):** The group of EC2 instances managed together
- **Launch Template:** The blueprint used to create new instances
- **Scaling Policies:** Rules that trigger scaling

**Scaling Policy Types:** the common one is **Target Tracking** (keep a metric at a target, e.g. CPU at 50%). Others: **Step**/**Simple** (scale on alarm thresholds), **Scheduled** (scale at set times), and **Predictive** (ML forecast).

**Flow:** a CloudWatch alarm (e.g. CPU > 70%) triggers the scaling policy → new instances launch from the Launch Template → they register with the Load Balancer.

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

**Common Triggers (Event Sources):** API Gateway (REST backend), S3 events (process uploads), DynamoDB Streams, SQS/SNS (message processing), EventBridge (scheduled cron), and Kinesis (stream processing).

---

### Q29: What are the limitations of Lambda?

**Key limits:** 15-minute timeout, 128 MB–10 GB memory, 1,000 concurrent executions per region (can be raised), and a 50 MB (zipped) deployment package.

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

**Routing Policies (awareness):** **Simple** (one resource), **Weighted** (split traffic for A/B), **Latency-based** (lowest-latency Region), **Failover** (active-passive backup), plus **Geolocation**/**Geoproximity** and **Multivalue Answer**.

---

## 11. CloudFormation & IaC

### Q34: What is AWS CloudFormation?

**Easy Explanation:** CloudFormation lets you define your entire AWS infrastructure in a text file (YAML or JSON). You can spin up or tear down a complete environment — VPC, EC2, RDS, everything — with a single command.

**Key Points:**
- **Infrastructure as Code (IaC)** — version control your infra
- Resources are defined in **Templates** (YAML or JSON)
- A **Stack** is a deployed instance of a template
- If a resource creation fails, CloudFormation **automatically rolls back**

**Template Structure (YAML):** key sections are `Parameters` (inputs), `Resources` (the AWS resources to create), and `Outputs` (values to return).

```yaml
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: ami-0abcdef1234567890
```

---

### Q35: CloudFormation vs Terraform — key differences?

**Awareness summary:** **CloudFormation** is AWS-only (YAML/JSON), AWS manages state, auto-rollback on failure. **Terraform** is multi-cloud (HCL), you manage the state file, and has a huge community.

---

## 12. Security & Compliance

### Q36: What is AWS KMS?

**Easy Explanation:** KMS (Key Management Service) is a managed key vault. You manage encryption keys (CMKs) and AWS services (S3, EBS, RDS, Lambda, Secrets Manager) use them to encrypt/decrypt your data without you handling raw keys. Key usage is logged in CloudTrail and keys are Region-specific.

---

### Q37: What is AWS Shield and WAF?

**Awareness summary:**
- **AWS Shield** = DDoS protection. **Standard** is free for everyone; **Advanced** is paid with extra protection and 24/7 support.
- **AWS WAF** (Web Application Firewall) blocks web exploits (SQL injection, XSS, bad bots) via Web ACL rules; attaches to CloudFront, ALB, and API Gateway.

---

### Q38: What is the AWS Shared Responsibility Model?

**Easy Explanation:** AWS is responsible for the cloud infrastructure (hardware, networking, data centers). You are responsible for what you put IN the cloud (your OS configs, application, data, user access).

- **Customer — security IN the cloud:** your data/encryption, IAM users & permissions, OS patching (EC2), application security, Security Group config.
- **AWS — security OF the cloud:** physical data centers, hardware, hypervisor, and managed-service infrastructure (RDS, Lambda).

---

## 13. SNS, SQS & Messaging

### Q39: What is Amazon SQS?

**Easy Explanation:** SQS is a message queue. Producer services drop messages into the queue, consumer services pick them up and process them at their own pace. It decouples your application so components don't need to be available at the same time.

**Key Points:**
- SQS = **Simple Queue Service**
- Messages stay in queue until consumed (up to 14 days)
- **Standard Queue**: At-least-once delivery, best-effort ordering, high throughput
- **FIFO Queue**: Exactly-once delivery, strict ordering, up to 3,000 msg/sec

**Use case:** Order Service → SQS Queue → Payment Processor, so the consumer scales and processes independently of the producer.

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

**Fan-out Pattern:** one SNS topic fans a message out to multiple SQS queues, each feeding a separate consumer (e.g., email, DB update, push notification).

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

**Awareness summary:** **ECS** uses AWS's own orchestrator — simpler, AWS-only, no control-plane cost. **EKS** runs managed **Kubernetes** — portable across clusters but steeper learning curve and a per-cluster control-plane fee.

---

## 15. Cost Optimization

### Q43: What are the key strategies to reduce AWS costs?

**Awareness summary:** Right-size instances (CloudWatch metrics), commit with **Reserved Instances / Savings Plans** for steady workloads and use **Spot** for fault-tolerant ones, **Auto Scaling** down off-hours, **S3 lifecycle** policies to cheaper tiers, delete unused resources (EBS volumes, idle EIPs, snapshots), cache with **CloudFront**, and set **Cost Explorer + Budgets** alerts.

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

### Common Interview Follow-up Questions

- *"What happens if an Availability Zone goes down?"* → Traffic auto-routes to other AZs if Multi-AZ is configured
- *"How do you make an application highly available?"* → Multi-AZ deployment + Auto Scaling + Load Balancer
- *"How do you reduce latency for global users?"* → CloudFront CDN + Route 53 latency-based routing
- *"How do you connect your on-premise network to AWS?"* → AWS Direct Connect (dedicated line) or Site-to-Site VPN
- *"What is the difference between horizontal and vertical scaling?"* → Horizontal = add more instances (scale out); Vertical = bigger instance (scale up)
- *"How do EC2 instances in a private subnet access the internet?"* → Via NAT Gateway in the public subnet

---

*Last updated: June 2026 | Study path: YouTube → this guide → hands-on AWS Free Tier*
