# AWS Cloud — Awareness Notes

> **Scope note (junior job prep):** Deep AWS (IAM policies, VPC, CloudFormation, Lambda, ECS/EKS, cost optimization) is **cloud-engineering / DevOps territory deferred for later study** — a junior full-stack dev rarely needs more than awareness to get hired. This file is trimmed to a short "recognize the core services" section. The full AWS deep-dive remains in git history.

---

## The Core Services to Recognize (one-liner each)

| Service | What it is |
|---|---|
| **EC2** | Virtual servers in the cloud (rent a machine to run your app) |
| **S3** | Object storage for files (images, backups, static assets) |
| **RDS** | Managed relational databases (MySQL, PostgreSQL) — AWS handles backups/patching |
| **IAM** | Identity & Access Management — users, roles, and permissions (least privilege) |
| **Lambda** | Serverless functions — run code without managing servers, pay per invocation |
| **VPC** | Your private network in AWS |
| **CloudWatch** | Monitoring, metrics, and logs |
| **Route 53** | AWS's DNS service |
| **SQS / SNS** | Managed message queue / pub-sub |
| **ELB** | Elastic Load Balancer — distributes traffic across servers |

### A couple of concepts worth knowing
- **Region vs Availability Zone** — a Region is a geographic area; each contains multiple isolated data centers (AZs) for fault tolerance.
- **Managed service** — AWS runs the undifferentiated heavy lifting (e.g., RDS patches/backs up your DB) so you focus on your app.

> **Interview soundbite:** "I know the core AWS building blocks — EC2 for servers, S3 for storage, RDS for managed databases, IAM for permissions, Lambda for serverless. I haven't operated cloud infrastructure in depth; my focus has been building the Spring Boot apps that run on it."

---

*Trimmed to awareness level for junior job prep. Restore the full AWS deep-dive from version control when you're ready to study it.*
