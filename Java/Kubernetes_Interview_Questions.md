# Kubernetes — Awareness Notes

> **Scope note (junior job prep):** Kubernetes is **container orchestration / infrastructure** — mostly a DevOps responsibility you won't operate as a junior developer, so it's **deferred for later study**. This file is trimmed to a short awareness section (enough to recognize the terms). The full deep-dive (architecture, RBAC, Helm, autoscaling, storage, rollouts, kubectl) remains in git history. **Docker** — which you *do* use day-to-day — is kept in full in `Docker_Concepts_Study_Guide.md` and `Git_Docker_DevOps_Interview_Questions.md`.

---

## What It Is (the 30-second version)

Kubernetes (K8s) automates running containers across many machines: **deployment, scaling, self-healing (restart failed containers), load balancing, and zero-downtime rolling updates**. Docker packages and runs *one* container; Kubernetes orchestrates *many* across a cluster.

| | Docker | Kubernetes |
|---|---|---|
| Scope | Single-container management | Multi-container orchestration across machines |
| Scaling | Manual | Automatic |
| Self-healing | No | Yes |
| Load balancing | Basic | Built-in |

## The Objects to Recognize

- **Pod** — the smallest deployable unit; wraps one (or a few tightly-coupled) containers.
- **Deployment** — manages a set of identical Pods: handles scaling and rolling updates.
- **Service** — a stable network endpoint / load balancer in front of a set of Pods (Pods come and go; the Service IP is stable).
- **ConfigMap / Secret** — externalized configuration and sensitive values injected into Pods.
- **Ingress** — routes external HTTP traffic to Services (the cluster's "front door").
- **Namespace** — a logical partition to isolate resources within a cluster.

> **Interview soundbite:** "I understand Kubernetes conceptually — Pods run containers, Deployments manage and scale them, Services give stable networking, and it self-heals and does rolling updates. I've worked hands-on with Docker; operating a K8s cluster is something I'd grow into on the job."

---

*Trimmed to awareness level for junior job prep. Restore the full Kubernetes deep-dive from version control when you're ready to study it.*
