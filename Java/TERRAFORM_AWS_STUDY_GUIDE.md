# Terraform + AWS — Awareness Notes

> **Scope note (junior job prep):** Terraform / Infrastructure-as-Code is **cloud-provisioning tooling deferred for later study** — not needed to get hired as a junior full-stack developer. This file is trimmed to a one-paragraph awareness note. The full study guide (provider/vpc/ec2 `.tf` breakdowns, networking, security groups, workflow commands) remains in git history.

---

## What It Is (the 30-second version)

**Terraform** (by HashiCorp) is an **Infrastructure as Code (IaC)** tool: you describe cloud resources (servers, networks, databases) in declarative `.tf` files written in **HCL**, and Terraform creates/updates/destroys them to match. Instead of clicking around the AWS console, your infrastructure lives in version control.

- Core workflow: `terraform init` → `terraform plan` (preview changes) → `terraform apply` (make them real) → `terraform destroy` (tear down).
- **State file** — Terraform tracks what it has provisioned in a state file so it knows what to change.
- IaC benefits: repeatable environments, code review for infra changes, easy teardown/rebuild.

> **Interview soundbite:** "Terraform is Infrastructure as Code — you declare cloud resources in HCL and it provisions them, with `plan`/`apply`/`destroy`. I understand the concept; provisioning infrastructure is something I'd learn on the job rather than as part of building Spring Boot apps."

---

*Trimmed to awareness level for junior job prep. Restore the full Terraform study guide from version control when you're ready to study it.*
