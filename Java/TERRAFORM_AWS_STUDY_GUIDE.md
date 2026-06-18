# Terraform + AWS Interview Study Guide
### Based on: `demo_aws_infra` Project

---

## Table of Contents

1. [What is Terraform?](#1-what-is-terraform)
2. [Core Terraform Concepts](#2-core-terraform-concepts)
3. [Project Architecture Overview](#3-project-architecture-overview)
4. [File-by-File Code Breakdown](#4-file-by-file-code-breakdown)
   - [provider.tf](#41-providertf)
   - [vpc.tf](#42-vpctf)
   - [ec2.tf](#43-ec2tf)
   - [output.tf](#44-outputtf)
   - [scripts/proxy.sh](#45-scriptsproxysh)
   - [scripts/app.sh](#46-scriptsappsh)
   - [.terraform.lock.hcl](#47-terraformlockhcl)
5. [AWS Networking](#5-aws-networking)
6. [AWS Security Concepts](#6-aws-security-concepts)
7. [Terraform Workflow Commands](#7-terraform-workflow-commands)
8. [Top Interview Questions & Answers](#8-top-interview-questions--answers)

---

## 1. What is Terraform?

**Terraform** is an open-source **Infrastructure as Code (IaC)** tool by HashiCorp. It lets you define cloud resources in **HCL (HashiCorp Configuration Language)** and provision them automatically.

| Concept | Meaning |
|---|---|
| **Declarative** | Describe *what* you want, not *how* to build it step by step |
| **Idempotent** | Running `terraform apply` multiple times produces the same result |
| **Provider-agnostic** | Works with AWS, Azure, GCP, Kubernetes, and more |
| **State-driven** | Tracks what it created in a `terraform.tfstate` file |

**Why IaC over clicking in the AWS Console?**
- Version-controlled, reproducible environments (dev/staging/prod identical)
- Audit trail via git; team collaboration via pull requests
- Rebuild an entire environment from scratch in minutes

---

## 2. Core Terraform Concepts

### 2.1 Providers

A **provider** is a plugin that lets Terraform talk to a specific platform's API. Think of it like an npm package — declare it, and Terraform downloads it during `terraform init`.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.44.0"
    }
  }
}
```

### 2.2 Resources

A **resource** is a single piece of infrastructure to create.

```hcl
resource "aws_vpc" "demo_vpc" {
  cidr_block = "10.0.0.0/16"
}
```

- `aws_vpc` → resource type
- `demo_vpc` → local name used to reference this resource in other `.tf` files
- `cidr_block` → configuration argument

### 2.3 Data Sources

A **data source** reads existing information *without creating anything*.

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["137112412989"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023*-x86_64"]
  }
}
```

Reference it as: `ami = data.aws_ami.amazon_linux.id`

> **Interview tip:** Data sources are read-only. Resources create or manage. Know this distinction.

### 2.4 Outputs

**Outputs** print values after `terraform apply` and expose data to other modules. Think of them as the return value of your infrastructure code.

```hcl
output "demo_app_vote_url" {
  value = "http://${aws_instance.demo_proxy_instance.public_ip}"
}
```

### 2.5 State File (`terraform.tfstate`)

The state file is Terraform's **memory** — it maps HCL code to real cloud resources.

- **Never commit to git** — contains sensitive data (IPs, secrets)
- In teams, store in **S3 + DynamoDB** for remote state with locking
- If state is lost, use `terraform import` to recover

### 2.6 The Dependency Graph

Terraform automatically determines resource creation order via a **dependency graph**.

```hcl
resource "aws_nat_gateway" "nat_gateway" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public.id
  depends_on    = [aws_internet_gateway.internet_gateway]
}
```

Use `depends_on` when Terraform can't detect the dependency automatically from attribute references.

### 2.7 Variables and Locals

```hcl
variable "region" {
  default = "ap-south-1"
}

locals {
  full_name = "demo-${var.region}"
}

provider "aws" {
  region = var.region
}
```

---

## 3. Project Architecture Overview

```
Public Internet
       |
[Internet Gateway]
       |
[Public Subnet 10.0.1.0/24]
  ┌─────────────────────────┐
  │  Proxy Instance          │
  │  IP: 10.0.1.100          │
  │  Nginx Reverse Proxy     │
  │  Inbound: 22, 80, 8080   │
  └─────────────────────────┘
       |  (internal VPC traffic)
[Private Subnet 10.0.2.0/24]
  ┌─────────────────────────┐
  │  App Instance            │
  │  IP: 10.0.2.100          │
  │  Docker + Voting App     │
  │  Inbound: VPC only       │
  └─────────────────────────┘
       |  (outbound only)
[NAT Gateway] --> [Internet]
```

**Traffic Flow:**
1. User hits `http://<proxy-public-ip>` → Nginx forwards to `10.0.2.100:8080` (Voting App)
2. User hits `http://<proxy-public-ip>:8080` → Nginx forwards to `10.0.2.100:8081` (Results App)
3. App Instance reaches internet outbound via NAT Gateway (Docker pulls, git clone, etc.)

---

## 4. File-by-File Code Breakdown

### 4.1 `provider.tf`

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws"; version = "6.44.0" }
    tls = { source = "hashicorp/tls"; version = "4.2.1" }
  }
}

provider "aws" {
  region  = "ap-south-1"
  profile = "demo_devops"

  default_tags {
    tags = {
      Environment = "demo"
      ManagedBy   = "terraform"
    }
  }
}
```

| Part | Purpose |
|---|---|
| `required_providers` | Locks exact provider versions to prevent breaking changes |
| `profile = "demo_devops"` | Uses a named AWS CLI profile — avoids hardcoding credentials in git |
| `tls` provider | Generates RSA key pairs for SSH |
| `default_tags` | Automatically applies tags to every AWS resource created |

### 4.2 `vpc.tf`

#### VPC + Subnets

```hcl
resource "aws_vpc" "demo_vpc" {
  cidr_block = "10.0.0.0/16"   # 65,536 private IPs
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.demo_vpc.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.demo_vpc.id
  cidr_block = "10.0.2.0/24"
}
```

#### Internet Gateway + NAT Gateway

```hcl
resource "aws_internet_gateway" "internet_gateway" {
  vpc_id = aws_vpc.demo_vpc.id
}

resource "aws_eip" "nat_eip" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat_gateway" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public.id   # Must live in the PUBLIC subnet
  depends_on    = [aws_internet_gateway.internet_gateway]
}
```

#### Route Tables

```hcl
resource "aws_route_table" "public_route_table" {
  vpc_id = aws_vpc.demo_vpc.id
  route { cidr_block = "0.0.0.0/0"; gateway_id = aws_internet_gateway.internet_gateway.id }
}

resource "aws_route_table" "private_route_table" {
  vpc_id = aws_vpc.demo_vpc.id
  route { cidr_block = "0.0.0.0/0"; gateway_id = aws_nat_gateway.nat_gateway.id }
}

resource "aws_route_table_association" "public_route_table_association" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public_route_table.id
}
```

A route table is a routing rule: "for traffic going to `0.0.0.0/0`, send it through X." The association is what actually links a route table to a subnet.

### 4.3 `ec2.tf`

#### AMI Data Source

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["137112412989"]
  filter { name = "name"; values = ["al2023-ami-2023*-x86_64"] }
}
```

Dynamically fetches the latest Amazon Linux 2023 AMI instead of hardcoding a region-specific ID.

#### Security Groups

```hcl
# Public-facing proxy
resource "aws_security_group" "demo_proxy_security_group" {
  vpc_id = aws_vpc.demo_vpc.id
  ingress { from_port = 22;   to_port = 22;   protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 80;   to_port = 80;   protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 8080; to_port = 8080; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0;    to_port = 0;    protocol = "-1";  cidr_blocks = ["0.0.0.0/0"] }
}

# Private app server — VPC traffic only
resource "aws_security_group" "demo_app_security_group" {
  vpc_id = aws_vpc.demo_vpc.id
  ingress { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["10.0.0.0/16"] }
  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}
```

Security groups are **stateful** — an allowed inbound connection automatically allows the response outbound.

#### SSH Key Generation

```hcl
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "demo_ssh_key" {
  key_name   = "demo-ssh-key"
  public_key = tls_private_key.ssh_key.public_key_openssh
}

resource "local_file" "ssh_key" {
  content         = tls_private_key.ssh_key.private_key_pem
  filename        = "demo-ssh-key.pem"
  file_permission = "0600"
}
```

The `tls` provider generates the key pair in memory. Public key goes to AWS; private key is saved locally as `.pem`.

#### EC2 Instances

```hcl
resource "aws_instance" "demo_proxy_instance" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public.id
  private_ip                  = "10.0.1.100"
  associate_public_ip_address = true
  vpc_security_group_ids      = [aws_security_group.demo_proxy_security_group.id]
  key_name                    = aws_key_pair.demo_ssh_key.key_name
  user_data                   = file("scripts/proxy.sh")
  user_data_replace_on_change = true
}

resource "aws_instance" "demo_app_instance" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.private.id
  private_ip                  = "10.0.2.100"
  associate_public_ip_address = false
  vpc_security_group_ids      = [aws_security_group.demo_app_security_group.id]
  key_name                    = aws_key_pair.demo_ssh_key.key_name
  user_data                   = file("scripts/app.sh")
  user_data_replace_on_change = true
}
```

`user_data_replace_on_change = true` — if the bootstrap script changes, Terraform destroys and recreates the instance so the new script runs fresh.

### 4.4 `output.tf`

```hcl
output "demo_app_vote_url" {
  value = "http://${aws_instance.demo_proxy_instance.public_ip}"
}

output "demo_app_results_url" {
  value = "http://${aws_instance.demo_proxy_instance.public_ip}:8080"
}
```

### 4.5 `scripts/proxy.sh`

Bash bootstrap script that runs once via `user_data` when the Proxy EC2 first starts.

```bash
#!/bin/bash
exec > >(tee /var/log/user-data.log | logger -t user-data) 2>&1
set -x
sleep 30  # Wait for network

dnf clean all && dnf makecache && dnf install -y nginx
systemctl enable nginx && systemctl start nginx

rm -f /etc/nginx/conf.d/default.conf
cat <<EOF >/etc/nginx/conf.d/proxy.conf
server {
    listen 80;
    location / { proxy_pass http://10.0.2.100:8080; }
}
server {
    listen 8080;
    location / { proxy_pass http://10.0.2.100:8081; }
}
EOF

nginx -t && systemctl restart nginx
```

- `exec > >(tee ...) 2>&1` — captures all output to a log file for debugging
- `nginx -t` — validates config syntax before restarting, preventing outages from bad config

### 4.6 `scripts/app.sh`

Bootstraps the private App EC2 with Docker and the voting application.

```bash
#!/bin/bash
exec > >(tee /var/log/user-data.log | logger -t user-data) 2>&1
set -euxo pipefail   # -e: exit on error; -u: error on undefined vars; -x: trace; -o pipefail: catch pipe errors
sleep 30

dnf update -y && dnf install -y docker git
systemctl enable docker && systemctl start docker
usermod -aG docker ec2-user

# Install Docker Compose plugin
mkdir -p /usr/libexec/docker/cli-plugins
curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" \
     -o /usr/libexec/docker/cli-plugins/docker-compose
chmod +x /usr/libexec/docker/cli-plugins/docker-compose

if [ ! -d "/home/ec2-user/example-voting-app" ]; then
  git clone https://github.com/dockersamples/example-voting-app.git /home/ec2-user/example-voting-app
fi

chown -R ec2-user:ec2-user /home/ec2-user/example-voting-app

sudo -u ec2-user bash -c '
  cd /home/ec2-user/example-voting-app
  docker compose up -d --build
'
```

- `sudo -u ec2-user` — runs Docker as a non-root user (Principle of Least Privilege)
- `-d` in `docker compose up -d` — detached mode, containers run in the background

### 4.7 `.terraform.lock.hcl`

Terraform's **dependency lock file** — similar to `package-lock.json` in Node.js.

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "6.44.0"
  constraints = "6.44.0"
  hashes = ["h1:lJvGaC4...", "zh:0462747..."]
}
```

- Locks exact provider versions so every team member uses identical binaries
- `hashes` are cryptographic checksums to verify the provider hasn't been tampered with
- **Should be committed to git** (unlike `terraform.tfstate`)

---

## 5. AWS Networking

### CIDR Notation

| CIDR | Total IPs | Use Case |
|---|---|---|
| `/16` | 65,536 | VPC |
| `/24` | 256 | Subnet |
| `/32` | 1 | Single IP |
| `0.0.0.0/0` | All IPs | Anywhere on the internet |

### Public vs Private Subnets

| Property | Public Subnet | Private Subnet |
|---|---|---|
| Route to internet | Via Internet Gateway | Via NAT Gateway |
| Inbound from internet | Yes | No |
| Public IP on instances | Yes | No |
| Example use | Proxies, load balancers | App servers, databases |

### Internet Gateway vs NAT Gateway

| Feature | Internet Gateway | NAT Gateway |
|---|---|---|
| Direction | Bidirectional | Outbound only |
| Used by | Public subnets | Private subnets |
| Cost | Free | Charged per hour + data |
| Placement | Attached to VPC | Deployed inside public subnet |

### Route Tables

Every subnet is associated with exactly one route table. Rules: most specific route wins (`/16` before `/0`).

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | `local` | VPC-internal traffic stays local |
| `0.0.0.0/0` | `igw-xxxx` | Everything else → Internet Gateway |

### Elastic IP (EIP)

A static public IPv4 address that persists across instance restarts. The NAT Gateway needs one so private instances always have a consistent public identity for outbound traffic.

---

## 6. AWS Security Concepts

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Rule evaluation | All rules checked | Evaluated in number order |

**Security Groups:** stateful — allow inbound on port 80, and the response is automatically allowed back. Default: deny all inbound, allow all outbound.

**NACLs:** stateless — must explicitly allow both inbound AND outbound for each connection.

### Principle of Least Privilege in This Project

1. App server has no public IP — can't be targeted from the internet
2. App security group only allows VPC traffic — only the proxy can reach it
3. SSH key is 4096-bit RSA; private key permissions are `0600`
4. Docker runs as `ec2-user`, not root

### IAM Note

The `profile = "demo_devops"` references an AWS CLI profile holding IAM credentials. In production, use **IAM Roles + Instance Profiles** instead of access keys.

---

## 7. Terraform Workflow Commands

```
terraform init      → Download providers, set up backend
terraform validate  → Check HCL syntax (no AWS connection)
terraform plan      → Dry run — show what will be created/changed/destroyed
terraform apply     → Create/modify resources in AWS
terraform destroy   → Tear down all Terraform-managed resources
terraform output    → Show output values after apply
terraform fmt       → Auto-format HCL code
terraform import    → Import existing AWS resources into state
```

### Typical Workflow

```bash
terraform init
terraform plan
terraform apply
terraform output

# SSH into proxy
ssh -i demo-ssh-key.pem ec2-user@<proxy-public-ip>

# Cleanup
terraform destroy
```

---

## 8. Top Interview Questions & Answers

### Terraform Fundamentals

**Q: What is the difference between `terraform plan` and `terraform apply`?**
`plan` is a dry run — it compares your `.tf` files against the state file and shows what will change without touching anything. `apply` actually executes those changes in the cloud.

---

**Q: What is Terraform state and why is it important?**
It's a JSON file mapping your HCL definitions to real cloud resources. Without it, Terraform can't know what already exists. It enables incremental updates and stores resource metadata — but it can contain secrets, so it must be secured and never committed to git.

---

**Q: What is a Terraform provider?**
A plugin that acts as an API client for a specific platform (AWS, GCP, Azure, etc.). Declared in the `terraform {}` block and downloaded during `terraform init`.

---

**Q: How do you manage Terraform state in a team?**
Use **remote state** with S3 + DynamoDB: S3 stores the state file, DynamoDB provides **locking** to prevent two people running `apply` simultaneously.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "prod/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

---

**Q: What is `depends_on` and when do you use it?**
It explicitly declares a dependency when Terraform can't detect it automatically from attribute references. In this project, the NAT Gateway needs the IGW to exist first, but there's no direct attribute linking them — so `depends_on` enforces the order.

---

**Q: What is the difference between a resource and a data source?**
A **resource** creates and manages infrastructure (Terraform owns the lifecycle). A **data source** is a read-only lookup of existing information (Terraform doesn't create or manage it).

---

**Q: What does `user_data_replace_on_change = true` do?**
If the `user_data` script changes, Terraform destroys and recreates the EC2 instance so the new script runs fresh. By default, AWS ignores `user_data` changes on already-running instances.

---

**Q: What are Terraform modules?**
Reusable packages of Terraform config. Any directory with `.tf` files is a module. The root module is where you run Terraform commands; child modules enable code reuse across dev/staging/prod.

---

### AWS Networking Questions

**Q: What is a VPC?**
A logically isolated virtual network in AWS where you control IP ranges, subnets, route tables, and gateways. Everything you launch in AWS lives inside a VPC.

---

**Q: What is the difference between a public and private subnet?**
A public subnet has a route to an Internet Gateway — resources can receive inbound internet traffic. A private subnet has no IGW route — resources can only reach the internet outbound via a NAT Gateway; no inbound connections are possible.

---

**Q: Why does the NAT Gateway go in the PUBLIC subnet?**
It needs internet access itself to forward traffic. It sits in the public subnet (which has IGW access), receives outbound requests from private instances, forwards them using its Elastic IP, and returns responses back to the private instance.

---

**Q: What is the difference between an Internet Gateway and a NAT Gateway?**
IGW is **bidirectional** — allows both inbound and outbound internet traffic. NAT Gateway is **outbound only** — private instances can initiate connections out, but the internet cannot initiate connections back in.

---

**Q: What is a Security Group?**
A stateful virtual firewall at the instance level. If you allow inbound on port 80, the response is automatically allowed out. Default: deny all inbound, allow all outbound.

---

**Q: Explain the Nginx reverse proxy setup.**
The Proxy instance (public, `10.0.1.100`) runs Nginx: port 80 forwards to `10.0.2.100:8080` (voting app), port 8080 forwards to `10.0.2.100:8081` (results app). The App instance has no public IP — users only ever see the proxy's public IP, keeping the app server hidden from the internet.

---

**Q: Why use static private IPs?**
The Nginx config hardcodes the backend address (`10.0.2.100`). If the app instance restarted with a dynamic IP, Nginx would point at the wrong address. Static IPs make the config predictable without needing service discovery.

---

**Q: What is an Elastic IP?**
A static public IPv4 address that persists even when an instance restarts. The NAT Gateway needs one so private instances always have a consistent public identity for outbound connections.

---

**Q: What happens if two people run `terraform apply` simultaneously without locking?**
The state file could be corrupted — both would overwrite each other's state. S3 + DynamoDB locking blocks the second `apply` until the first completes.

---

**Q: What is `.terraform.lock.hcl` and should you commit it?**
Terraform's dependency lock file — pins exact provider versions and their cryptographic hashes. **Yes, commit it.** It ensures everyone on the team and in CI/CD uses the same provider binaries.

---

## Quick Reference Cheat Sheet

```
VPC            → Your private network in AWS
Subnet         → Segment of the VPC's IP range
IGW            → Bidirectional door between VPC and public internet
NAT Gateway    → Outbound-only internet for private subnets
Route Table    → Routing rules linking subnets to gateways
Security Group → Stateful firewall at instance level
NACL           → Stateless firewall at subnet level
EIP            → Static public IP
AMI            → VM template (OS image) for EC2
user_data      → Bootstrap script that runs once at instance launch
Key Pair       → SSH public/private key for EC2 access

provider.tf    → Which cloud and which version
vpc.tf         → Networking (VPC, subnets, gateways, routes)
ec2.tf         → Compute (instances, security groups, keys)
output.tf      → Values printed after terraform apply
.tfstate       → Terraform's memory (never commit to git)
.lock.hcl      → Provider version lock (commit to git)
```

---

*Last Updated: 2026-06-18*
