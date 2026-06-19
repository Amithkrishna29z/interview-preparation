# DevOps Core Concepts — Awareness Notes

> **Scope note (junior job prep):** DevOps as a discipline (Infrastructure as Code, configuration management, container orchestration, SRE, DevSecOps, artifact management, observability) is largely a **platform/ops role deferred for later study**. This file is trimmed to a short awareness section. The junior-practical pieces you actually use — **Git workflows, Docker, and CI/CD basics** — are kept in **`Git_Docker_DevOps_Interview_Questions.md`**. The full DevOps survey remains in git history.

---

## What It Is (the 30-second version)

DevOps is a **culture + set of practices** that unites development and operations to ship software faster and more reliably, leaning heavily on **automation** (automated testing, builds, and deployments).

- **CI (Continuous Integration)** — every push is automatically built and tested.
- **CD (Continuous Delivery/Deployment)** — passing builds are automatically released to an environment.
- **The lifecycle:** Plan → Code → Build → Test → Release → Deploy → Operate → Monitor → (feedback loop back to Plan).

## Terms to Recognize (one-liner each)

- **Infrastructure as Code (IaC)** — define servers/infra in version-controlled files (e.g., Terraform) instead of clicking in a console.
- **Container orchestration** — Kubernetes runs/scales/heals containers across machines (you won't operate this as a junior).
- **Observability** — logs, metrics, and traces that tell you what your running app is doing.
- **Artifact registry** — stores built artifacts (Docker images, JARs) — e.g., Docker Hub, Nexus.

> **Interview soundbite:** "I understand the DevOps flow — CI builds and tests every push, CD deploys passing builds — and I've used Git and Docker hands-on. Deeper platform work (IaC, Kubernetes, SRE) is something I'd pick up on the job."

---

*Trimmed to awareness level for junior job prep. Restore the full DevOps survey from version control when you're ready to study it.*
