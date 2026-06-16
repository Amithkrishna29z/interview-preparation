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
5. [AWS Networking Deep Dive](#5-aws-networking-deep-dive)
6. [AWS Security Concepts](#6-aws-security-concepts)
7. [Terraform Workflow Commands](#7-terraform-workflow-commands)
8. [Top Interview Questions & Answers](#8-top-interview-questions--answers)

---

## 1. What is Terraform?

**Terraform** is an open-source **Infrastructure as Code (IaC)** tool by HashiCorp. It lets you define cloud resources in a **declarative** configuration language called **HCL (HashiCorp Configuration Language)** and then provision them automatically.

### Key Ideas

| Concept | Meaning |
|---|---|
| **Declarative** | You describe *what* you want, not *how* to build it step by step |
| **Idempotent** | Running `terraform apply` multiple times produces the same result — no duplicate resources |
| **Provider-agnostic** | Works with AWS, Azure, GCP, Kubernetes, GitHub, and hundreds more |
| **State-driven** | Tracks what it has created in a `terraform.tfstate` file |

### Why use IaC instead of clicking in the AWS Console?

- Version-controlled infrastructure (same as application code)
- Reproducible environments (dev, staging, prod are identical)
- Audit trail — every change is a git commit
- Team collaboration — pull requests for infra changes
- Disaster recovery — rebuild an entire environment from scratch in minutes

---

## 2. Core Terraform Concepts

### 2.1 Providers

A **provider** is a plugin that allows Terraform to talk to a specific platform's API.

```hcl
# This tells Terraform: "Use the AWS provider at version 6.44.0"
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.44.0"
    }
  }
}
```

Think of providers like **npm packages** — you declare what you need and Terraform downloads them into the `.terraform/` folder during `terraform init`.

---

### 2.2 Resources

A **resource** is a single piece of infrastructure you want to create.

```hcl
resource "<PROVIDER>_<TYPE>" "<LOCAL_NAME>" {
  # configuration arguments
}
```

Example:
```hcl
resource "aws_vpc" "demo_vpc" {
  cidr_block = "10.0.0.0/16"
}
```

- `aws_vpc` → the resource type (maps to an AWS VPC)
- `demo_vpc` → the local name used to reference this resource elsewhere in Terraform code
- `cidr_block` → the argument (what you're configuring)

---

### 2.3 Data Sources

A **data source** reads existing information from the provider *without creating anything*.

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["137112412989"]   # official Amazon account ID

  filter {
    name   = "name"
    values = ["al2023-ami-2023*-x86_64"]
  }
}
```

This finds the latest Amazon Linux 2023 AMI ID dynamically. You then reference it as:
```hcl
ami = data.aws_ami.amazon_linux.id
```

> **Interview tip:** Data sources are read-only. Resources create or manage. Know this distinction.

---

### 2.4 Outputs

**Outputs** print values after `terraform apply` and make data available to other Terraform modules.

```hcl
output "demo_app_vote_url" {
  value = "http://${aws_instance.demo_proxy_instance.public_ip}"
}
```

Think of outputs like the **return value** of your infrastructure code.

---

### 2.5 State File (`terraform.tfstate`)

The state file is Terraform's **memory**. It maps your HCL code to real cloud resources.

- **Never commit this to Git** — it contains sensitive data (IPs, credentials, secrets)
- In teams, store it in **S3 + DynamoDB** for remote state with locking
- If state is lost, Terraform doesn't know what it created (use `terraform import` to recover)

---

### 2.6 The Dependency Graph

Terraform automatically figures out the order to create resources by building a **dependency graph**.

```hcl
resource "aws_nat_gateway" "nat_gateway" {
  allocation_id = aws_eip.nat_eip.id         # depends on EIP
  subnet_id     = aws_subnet.public.id       # depends on public subnet
  depends_on    = [aws_internet_gateway.internet_gateway]  # explicit dependency
}
```

Here, Terraform knows:
1. Create the Internet Gateway first
2. Create the EIP and Public Subnet (can be done in parallel)
3. Create the NAT Gateway last (it needs both)

---

### 2.7 Variables, Locals, and Expressions

Not used directly in this project but essential to know:

```hcl
# Input variable
variable "region" {
  default = "ap-south-1"
}

# Local value (computed)
locals {
  full_name = "demo-${var.region}"
}

# Referencing
provider "aws" {
  region = var.region
}
```

---

## 3. Project Architecture Overview

```
Public Internet
       |
       | (Port 80 / Port 8080)
       v
[Internet Gateway]
       |
       v
[Public Subnet 10.0.1.0/24]
  ┌─────────────────────────┐
  │  Proxy Instance          │
  │  IP: 10.0.1.100          │
  │  Nginx Reverse Proxy     │
  │  Security Group:         │
  │   - Inbound: 22,80,8080  │
  │   - Outbound: all        │
  └─────────────────────────┘
       |
       | (Internal VPC traffic to 10.0.2.100)
       v
[Private Subnet 10.0.2.0/24]
  ┌─────────────────────────┐
  │  App Instance            │
  │  IP: 10.0.2.100          │
  │  Docker + Voting App     │
  │  Security Group:         │
  │   - Inbound: VPC only    │
  │   - Outbound: all        │
  └─────────────────────────┘
       |
       | (Outbound only — package downloads, git clone)
       v
[NAT Gateway] --> [Internet]
```

**Traffic Flow:**
1. User hits `http://<proxy-public-ip>` (port 80)
2. Nginx on Proxy Instance receives it and forwards to `http://10.0.2.100:8080` (Voting App)
3. User hits `http://<proxy-public-ip>:8080`
4. Nginx forwards to `http://10.0.2.100:8081` (Results App)
5. App Instance can reach the internet *outbound* via NAT Gateway (to download Docker images, git clone, etc.)

---

## 4. File-by-File Code Breakdown

### 4.1 `provider.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.44.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "4.2.1"
    }
  }
}

provider "aws" {
  region  = "ap-south-1"
  profile = "demo_devops"

  default_tags {
    tags = {
      Environment = "demo"
      Owner       = "devops"
      ManagedBy   = "terraform"
      Status      = "active"
    }
  }
}
```

**What each part does:**

| Part | Purpose |
|---|---|
| `required_providers` | Locks the exact provider versions to prevent breaking changes |
| `aws` provider | Tells Terraform to use the AWS API in the `ap-south-1` region (Mumbai) |
| `profile = "demo_devops"` | Uses a named AWS CLI profile from `~/.aws/credentials` instead of hardcoding keys |
| `tls` provider | Used to generate cryptographic keys (RSA key pair for SSH) |
| `default_tags` | Automatically applies these tags to **every** AWS resource created — eliminates repetitive tag blocks |

**Why `profile` and not hardcoded keys?**
Hardcoding `access_key` and `secret_key` in Terraform is a security risk (they get committed to git). Named profiles keep credentials on your local machine only.

---

### 4.2 `vpc.tf`

This file builds the entire network foundation. Think of a VPC as your own private data center inside AWS.

#### VPC

```hcl
resource "aws_vpc" "demo_vpc" {
  cidr_block = "10.0.0.0/16"
}
```

- `10.0.0.0/16` = a block of 65,536 private IP addresses (from `10.0.0.0` to `10.0.255.255`)
- Everything in this project lives inside this VPC

#### Subnets

```hcl
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.demo_vpc.id
  cidr_block              = "10.0.1.0/24"       # 256 IPs
  map_public_ip_on_launch = true                 # instances get a public IP automatically
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.demo_vpc.id
  cidr_block = "10.0.2.0/24"                    # 256 IPs
  # No map_public_ip_on_launch → no public IPs
}
```

#### Internet Gateway

```hcl
resource "aws_internet_gateway" "internet_gateway" {
  vpc_id = aws_vpc.demo_vpc.id
}
```

The IGW is the **door between your VPC and the public internet**. Without it, nothing in your VPC can reach the outside world and the outside world cannot reach your VPC.

#### Elastic IP + NAT Gateway

```hcl
resource "aws_eip" "nat_eip" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat_gateway" {
  allocation_id = aws_eip.nat_eip.id      # the static public IP for the NAT
  subnet_id     = aws_subnet.public.id    # NAT Gateway must live in the PUBLIC subnet
  depends_on    = [aws_internet_gateway.internet_gateway]
}
```

- **EIP (Elastic IP):** A static public IP address. The NAT Gateway needs one so it has a fixed identity on the internet.
- **NAT Gateway:** Lets private instances reach the internet outbound (for updates, downloads) while blocking all inbound connections. It translates the private IP to the EIP.
- **Why `depends_on`?** The NAT Gateway needs the IGW to exist first so it can actually reach the internet. Terraform doesn't always detect this automatically, so it's declared explicitly.

#### Route Tables

```hcl
# Public Route Table — sends internet traffic to IGW
resource "aws_route_table" "public_route_table" {
  vpc_id = aws_vpc.demo_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.internet_gateway.id
  }
}

# Private Route Table — sends internet traffic through NAT
resource "aws_route_table" "private_route_table" {
  vpc_id = aws_vpc.demo_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.nat_gateway.id
  }
}
```

A **route table** is like a GPS routing rule: "If traffic is going to `0.0.0.0/0` (anywhere on the internet), send it through X."

#### Route Table Associations

```hcl
resource "aws_route_table_association" "public_route_table_association" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public_route_table.id
}
```

This is what **links a route table to a subnet**. Without this, the routing rules exist but no subnet uses them.

---

### 4.3 `ec2.tf`

This file provisions the compute layer: instances, security groups, and SSH keys.

#### AMI Data Source

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["137112412989"]   # Amazon's official AWS account

  filter {
    name   = "name"
    values = ["al2023-ami-2023*-x86_64"]
  }
}
```

Instead of hardcoding an AMI ID (which is region-specific and changes over time), this dynamically fetches the latest Amazon Linux 2023 AMI. The `*` wildcard matches any version number.

#### Security Groups

**Proxy Security Group (Public-facing):**
```hcl
resource "aws_security_group" "demo_proxy_security_group" {
  vpc_id = aws_vpc.demo_vpc.id

  ingress { from_port = 22;   to_port = 22;   protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }  # SSH
  ingress { from_port = 80;   to_port = 80;   protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }  # HTTP
  ingress { from_port = 8080; to_port = 8080; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }  # Results

  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }         # All outbound
}
```

**App Security Group (Private — VPC-only):**
```hcl
resource "aws_security_group" "demo_app_security_group" {
  vpc_id = aws_vpc.demo_vpc.id

  ingress { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["10.0.0.0/16"] }  # Only from within VPC
  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }    # All outbound
}
```

Security groups are **stateful firewalls**. If you allow inbound traffic on a port, the response is automatically allowed back (you don't need a separate egress rule for it).

| Rule | Proxy SG | App SG |
|---|---|---|
| Port 22 (SSH) | Open to world | Blocked from internet |
| Port 80 (HTTP) | Open to world | VPC only |
| Port 8080 | Open to world | VPC only |
| Outbound | All allowed | All allowed |

#### SSH Key Generation

```hcl
# Step 1: Generate a 4096-bit RSA key pair
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

# Step 2: Register the PUBLIC key with AWS
resource "aws_key_pair" "demo_ssh_key" {
  key_name   = "demo-ssh-key"
  public_key = tls_private_key.ssh_key.public_key_openssh
}

# Step 3: Save the PRIVATE key to disk for SSH access
resource "local_file" "ssh_key" {
  content         = tls_private_key.ssh_key.private_key_pem
  filename        = "demo-ssh-key.pem"
  file_permission = "0600"   # Owner read-only — required by SSH
}
```

**How it works:** The `tls` provider generates a key pair in memory. The public key goes to AWS (so EC2 knows who to trust). The private key is saved locally as a `.pem` file (which you use to SSH in). This is 100% automated — no manual key creation needed.

#### EC2 Instances

**Proxy Instance (Public Subnet):**
```hcl
resource "aws_instance" "demo_proxy_instance" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  subnet_id                   = aws_subnet.public.id
  private_ip                  = "10.0.1.100"          # Static private IP
  associate_public_ip_address = true                   # Gets a public IP

  vpc_security_group_ids = [aws_security_group.demo_proxy_security_group.id]
  key_name               = aws_key_pair.demo_ssh_key.key_name

  user_data                   = file("scripts/proxy.sh")  # Bootstrap script
  user_data_replace_on_change = true
}
```

**App Instance (Private Subnet):**
```hcl
resource "aws_instance" "demo_app_instance" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  subnet_id                   = aws_subnet.private.id
  private_ip                  = "10.0.2.100"
  associate_public_ip_address = false                  # No public IP

  vpc_security_group_ids = [aws_security_group.demo_app_security_group.id]
  key_name               = aws_key_pair.demo_ssh_key.key_name

  user_data                   = file("scripts/app.sh")
  user_data_replace_on_change = true
}
```

`user_data_replace_on_change = true` means: if `proxy.sh` or `app.sh` changes, Terraform will **destroy and recreate** the instance (not just restart it), so the new script runs fresh.

---

### 4.4 `output.tf`

```hcl
output "demo_app_vote_url" {
  value = "http://${aws_instance.demo_proxy_instance.public_ip}"
}

output "demo_app_results_url" {
  value = "http://${aws_instance.demo_proxy_instance.public_ip}:8080"
}
```

After `terraform apply`, these print the actual URLs you can open in a browser. The `${...}` is Terraform's **string interpolation** syntax — it inserts the public IP of the proxy instance at runtime.

---

### 4.5 `scripts/proxy.sh`

This is a **bash bootstrap script** that runs once on the Proxy EC2 instance when it first starts (via `user_data`).

```bash
#!/bin/bash

# Redirect all output (stdout + stderr) to a log file AND system journal
exec > >(tee /var/log/user-data.log | logger -t user-data ) 2>&1
set -x    # Print every command before executing (debug mode)

sleep 30  # Wait for the network to fully come up before doing anything

# Install Nginx using dnf (Amazon Linux 2023's package manager)
dnf clean all
dnf makecache
dnf install -y nginx

# Enable Nginx so it restarts on reboot, then start it now
systemctl enable nginx
systemctl start nginx

# Delete the default Nginx site
rm -f /etc/nginx/conf.d/default.conf

# Write our custom proxy configuration using a here-document
cat <<EOF >/etc/nginx/conf.d/proxy.conf
server {
    listen 80;
    location / {
        proxy_pass http://10.0.2.100:8080;    # Forward port 80 → App Voting port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 8080;
    location / {
        proxy_pass http://10.0.2.100:8081;    # Forward port 8080 → App Results port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

nginx -t               # Test the config syntax before applying
systemctl restart nginx
```

**Key concepts:**
- `exec > >(tee ...) 2>&1` — captures all output for debugging even if the script fails silently
- `<<EOF ... EOF` — here-document for writing multi-line config without a separate file
- `proxy_set_header` — preserves the original client's IP and host through the proxy
- `nginx -t` — validates config before restarting, preventing outages from bad config

---

### 4.6 `scripts/app.sh`

This script bootstraps the **private App EC2 instance** with Docker and the voting application.

```bash
#!/bin/bash

exec > >(tee /var/log/user-data.log | logger -t user-data) 2>&1
set -euxo pipefail
# -e: exit on any error
# -u: treat undefined variables as errors
# -x: print each command
# -o pipefail: catch errors in piped commands

sleep 30   # Wait for network

# Install Docker and Git
dnf update -y
dnf install -y docker git

# Enable and start Docker daemon
systemctl enable docker
systemctl start docker

# Add ec2-user to docker group (avoids needing sudo for docker commands)
usermod -aG docker ec2-user

# Install Docker Compose plugin (v2 style — "docker compose" not "docker-compose")
mkdir -p /usr/libexec/docker/cli-plugins
curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64" \
     -o /usr/libexec/docker/cli-plugins/docker-compose
chmod +x /usr/libexec/docker/cli-plugins/docker-compose

# Install Docker Buildx plugin (multi-platform build support)
curl -SL "https://github.com/docker/buildx/releases/download/v0.23.0/buildx-v0.23.0.linux-amd64" \
     -o /usr/libexec/docker/cli-plugins/docker-buildx
chmod +x /usr/libexec/docker/cli-plugins/docker-buildx

# Clone the voting app (idempotent — only clones if not already there)
if [ ! -d "/home/ec2-user/example-voting-app" ]; then
  git clone https://github.com/dockersamples/example-voting-app.git /home/ec2-user/example-voting-app
fi

# Fix ownership so ec2-user can work with the files
chown -R ec2-user:ec2-user /home/ec2-user/example-voting-app

# Run docker compose as ec2-user (not root) — security best practice
sudo -u ec2-user bash -c '
  cd /home/ec2-user/example-voting-app
  sleep 10
  docker compose up -d --build
'
```

**Key concepts:**
- `set -euxo pipefail` — strict mode that catches most scripting bugs early
- `usermod -aG docker ec2-user` — Docker group membership takes effect at next login
- `sudo -u ec2-user bash -c '...'` — runs commands as a non-root user (Principle of Least Privilege)
- `-d` in `docker compose up -d` — detached mode, runs containers in the background

---

### 4.7 `.terraform.lock.hcl`

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "6.44.0"
  constraints = "6.44.0"
  hashes = [
    "h1:lJvGaC4...",
    "zh:0462747...",
    ...
  ]
}
```

This is Terraform's **dependency lock file** — similar to `package-lock.json` in Node.js or `Pipfile.lock` in Python.

**Why it matters:**
- Locks exact provider versions so every team member and CI/CD pipeline uses the same provider binary
- The `hashes` field contains cryptographic checksums to verify the downloaded provider hasn't been tampered with
- **Should be committed to git** (unlike `terraform.tfstate`)

---

## 5. AWS Networking Deep Dive

### 5.1 CIDR Notation

`10.0.0.0/16` — the `/16` is the **prefix length**.

| CIDR | Total IPs | Use Case |
|---|---|---|
| `/16` | 65,536 | VPC — large range |
| `/24` | 256 | Subnet — standard |
| `/32` | 1 | Single IP |
| `0.0.0.0/0` | All IPs | "Anywhere on the internet" |

### 5.2 Public vs Private Subnets

| Property | Public Subnet | Private Subnet |
|---|---|---|
| Route to internet | Via Internet Gateway | Via NAT Gateway |
| Inbound from internet | Yes | No |
| Outbound to internet | Yes | Yes (via NAT) |
| Public IP on instances | Yes (`map_public_ip_on_launch`) | No |
| Example use | Load balancers, bastion hosts, proxies | App servers, databases |

### 5.3 Internet Gateway vs NAT Gateway

| Feature | Internet Gateway | NAT Gateway |
|---|---|---|
| Direction | Bidirectional (in + out) | Outbound only |
| Used by | Public subnets | Private subnets |
| Required for | Public-facing services | Outbound internet from private |
| Managed by | AWS (free) | AWS (charged per hour + data) |
| Placement | Attached to VPC | Deployed inside public subnet |

### 5.4 Route Tables Explained

Every subnet must be associated with exactly one route table. A route table contains rules like:

| Destination | Target | Meaning |
|---|---|---|
| `10.0.0.0/16` | `local` | VPC-internal traffic stays local |
| `0.0.0.0/0` | `igw-xxxx` | Everything else goes to Internet Gateway |

The "most specific route wins" rule applies — `/16` matches before `/0`.

### 5.5 Elastic IP (EIP)

A static public IPv4 address assigned to your AWS account. Unlike a regular public IP (which changes on instance restart), an EIP persists. The NAT Gateway needs one so private instances always have a consistent public identity for outbound traffic.

---

## 6. AWS Security Concepts

### 6.1 Security Groups

- **Stateful** — if you allow inbound on port 80, the reply is automatically allowed out
- **Allow-only** — you can only add allow rules, never explicit deny rules
- **Applied at the instance level** (not the subnet level)
- Default behavior: deny all inbound, allow all outbound

### 6.2 Network ACLs (NACLs) — Not used here but important to know

- **Stateless** — you must explicitly allow both inbound AND outbound for each connection
- Applied at the **subnet level**
- Support both allow and deny rules
- Rules evaluated in number order (lower number = higher priority)

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Order | All rules evaluated | Rules evaluated in order |

### 6.3 Principle of Least Privilege in This Project

1. **App server has no public IP** — cannot be directly targeted from the internet
2. **App security group only allows VPC traffic** — even within AWS, only the proxy can talk to it
3. **SSH key is 4096-bit RSA** — strong cryptography
4. **Private key permissions are `0600`** — only the file owner can read it
5. **Docker runs as `ec2-user`**, not root — contains any container escape

### 6.4 IAM Concepts (Related)

The `profile = "demo_devops"` references an AWS CLI profile that holds IAM credentials. In production:
- Use **IAM Roles** for EC2 instances (not access keys)
- Use **IAM Instance Profiles** to attach roles to EC2
- Follow least-privilege IAM policies

---

## 7. Terraform Workflow Commands

```
terraform init      → Downloads providers, sets up backend
terraform validate  → Checks HCL syntax without connecting to AWS
terraform plan      → Shows what WILL be created/changed/destroyed (dry run)
terraform apply     → Actually creates/modifies resources in AWS
terraform destroy   → Tears down all resources Terraform created
terraform output    → Shows output values after apply
terraform state     → Inspect or manipulate the state file
terraform import    → Import existing AWS resources into Terraform state
terraform fmt       → Auto-formats HCL code
terraform graph     → Outputs dependency graph (use with Graphviz)
```

### Typical first-time workflow:

```bash
# 1. Initialize (only needed once or after changing providers)
terraform init

# 2. Preview what will be created
terraform plan

# 3. Create the infrastructure
terraform apply

# 4. See the URLs
terraform output

# 5. SSH into the proxy (once your key is saved)
ssh -i demo-ssh-key.pem ec2-user@<proxy-public-ip>

# 6. Destroy everything when done
terraform destroy
```

---

## 8. Top Interview Questions & Answers

### Terraform Fundamentals

**Q: What is the difference between `terraform plan` and `terraform apply`?**

`terraform plan` is a **dry run** — it compares the desired state (your `.tf` files) against the current state (`terraform.tfstate`) and shows you exactly what it will create, modify, or destroy without making any changes. `terraform apply` actually executes those changes against the cloud provider.

---

**Q: What is Terraform state and why is it important?**

State is a JSON file (`terraform.tfstate`) that maps your Terraform resource definitions to real-world cloud resources. Without it, Terraform can't know what already exists. It's important because: (1) it enables incremental updates (only changing what's different), (2) it stores metadata Terraform needs to manage resources, and (3) it can contain sensitive data so it must be secured.

---

**Q: What is a Terraform provider?**

A provider is a plugin that acts as an API client for a specific platform (AWS, GCP, Azure, GitHub, etc.). It translates Terraform's resource definitions into API calls. Providers are declared in the `terraform {}` block and downloaded during `terraform init`.

---

**Q: How do you manage Terraform state in a team?**

Use **remote state** with a backend like AWS S3 + DynamoDB:
- S3 stores the state file
- DynamoDB provides **state locking** (prevents two people running `apply` simultaneously)
- Enable versioning on the S3 bucket for rollback capability

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

`depends_on` explicitly declares a dependency between resources when Terraform cannot automatically detect it. In this project:
```hcl
resource "aws_nat_gateway" "nat_gateway" {
  depends_on = [aws_internet_gateway.internet_gateway]
}
```
Terraform can't infer that a NAT Gateway needs an IGW to function (since there's no direct attribute reference), so `depends_on` forces correct creation order.

---

**Q: What is the difference between a resource and a data source?**

- **Resource (`resource "aws_vpc" ...`)**: Creates and manages infrastructure. Terraform owns the lifecycle.
- **Data Source (`data "aws_ami" ...`)**: Read-only lookup of existing information. Terraform doesn't create or manage it.

---

**Q: What does `user_data_replace_on_change = true` do?**

When set to `true`, if the `user_data` script changes, Terraform will **destroy and recreate** the EC2 instance rather than just applying the change. By default, AWS ignores `user_data` changes on running instances. This flag ensures the new script always runs on a fresh instance.

---

**Q: What are Terraform modules?**

Modules are reusable packages of Terraform configurations. A module is any directory with `.tf` files. The `root` module is where you run Terraform commands. Child modules are called from the root module and enable code reuse (e.g., a VPC module you can call for dev, staging, and prod environments).

---

### AWS Networking Questions

**Q: What is a VPC?**

A Virtual Private Cloud (VPC) is a logically isolated section of the AWS cloud where you define your own virtual network. You control the IP address range, subnets, route tables, and gateways. Everything you launch in AWS goes into a VPC.

---

**Q: What is the difference between a public and private subnet?**

A **public subnet** has a route to an Internet Gateway, meaning resources in it can receive inbound traffic from the internet (and can reach it directly). A **private subnet** has no route to an IGW — resources can only reach the internet outbound through a NAT Gateway, and no inbound connections from the internet can reach them.

---

**Q: Why do we put the NAT Gateway in the PUBLIC subnet?**

The NAT Gateway needs to send traffic to the internet — therefore it needs internet access itself. It sits in the public subnet (which has IGW access), receives traffic from private subnet instances, and forwards it to the internet using its Elastic IP. The response comes back to the NAT Gateway and is forwarded back to the private instance.

---

**Q: What is the difference between an Internet Gateway and a NAT Gateway?**

An **Internet Gateway** allows **bidirectional** internet traffic — both inbound (someone connecting to your server) and outbound (your server making requests). A **NAT Gateway** is **outbound only** — private instances can initiate connections to the internet, but the internet cannot initiate connections back to them.

---

**Q: What is a Security Group and is it stateful or stateless?**

A Security Group is a virtual firewall at the **instance level** that controls inbound and outbound traffic. It is **stateful** — if you allow an inbound request on port 80, the response is automatically allowed out without needing a separate outbound rule.

---

**Q: Explain the nginx reverse proxy setup in this project.**

The Proxy EC2 instance (public subnet, IP `10.0.1.100`) runs Nginx with two server blocks:
1. Listens on port 80, forwards to `http://10.0.2.100:8080` (voting app)
2. Listens on port 8080, forwards to `http://10.0.2.100:8081` (results app)

The App EC2 instance (private subnet, IP `10.0.2.100`) has no public IP. Users hit the proxy's public IP, Nginx forwards the request internally to the app server, gets the response, and returns it to the user. This pattern hides the app server from the internet entirely.

---

**Q: Why use a static private IP (`private_ip = "10.0.1.100"`)?**

The Nginx config hardcodes `http://10.0.2.100:8080` as the backend address. If the app instance restarted and got a new dynamic IP, Nginx would be pointing at the wrong address. Static private IPs make the proxy config predictable and reliable without needing service discovery.

---

**Q: What is an Elastic IP (EIP)?**

An EIP is a static public IPv4 address that you can allocate to your AWS account. Unlike a regular public IP (which changes every time an instance restarts), an EIP persists. In this project, it's used by the NAT Gateway so private instances always have a consistent public identity for outbound connections.

---

### Advanced / Bonus Questions

**Q: What happens to the terraform.tfstate if two people run `terraform apply` at the same time?**

Without remote state locking, the state file could be corrupted — both would overwrite each other's state. With S3 + DynamoDB locking, the second `apply` is blocked until the first completes.

**Q: How would you make this infrastructure production-ready?**

Key improvements:
1. **Remote state** in S3 + DynamoDB
2. **Variables** for region, instance type, CIDR blocks (avoid hardcoding)
3. **Modules** for VPC and EC2 to enable environment reuse
4. **ALB (Application Load Balancer)** instead of single EC2 proxy
5. **Auto Scaling Group** for the app instances
6. **RDS** for the database instead of Docker-based DB
7. **ACM + HTTPS** instead of plain HTTP
8. **CloudWatch** for monitoring and alerting
9. **Bastion host or SSM Session Manager** instead of exposing port 22 publicly
10. **VPC Flow Logs** for network traffic audit

**Q: What is the `.terraform.lock.hcl` file and should you commit it?**

It's Terraform's **dependency lock file** that pins exact provider versions and their cryptographic hashes. **Yes, commit it to git.** It ensures every team member and CI/CD pipeline uses identical provider binaries, preventing "works on my machine" issues caused by provider version drift.

---

## Quick Reference Cheat Sheet

```
VPC            → Your private network in AWS
Subnet         → Segment of the VPC's IP range
IGW            → Door between VPC and public internet (bidirectional)
NAT Gateway    → Outbound internet for private subnets (outbound only)
Route Table    → Routing rules for subnets
Security Group → Stateful firewall at instance level
NACL           → Stateless firewall at subnet level
EIP            → Static public IP
AMI            → VM template (OS image) for EC2
user_data      → Bootstrap script that runs once at instance launch
Key Pair       → SSH public/private key for EC2 access
provider.tf    → Which cloud and which version
vpc.tf         → Networking (VPC, subnets, gateways, routes)
ec2.tf         → Compute (instances, security groups, keys)
output.tf      → Values to print after terraform apply
.tfstate       → Terraform's memory of what it created
.lock.hcl      → Provider version lock (commit to git)
```
