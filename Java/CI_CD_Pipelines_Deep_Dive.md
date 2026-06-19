# CI/CD Pipelines — Awareness Notes

> **Scope note (junior job prep):** The deep CI/CD pipeline engineering (pipeline stages, deployment strategies, secrets management, runners, multi-environment promotion) is **deferred for later study**. This file is trimmed to the core definitions. The junior-level CI/CD basics with a working **GitHub Actions** example are kept in **`Git_Docker_DevOps_Interview_Questions.md`**. The full deep-dive remains in git history.

---

## The Three Definitions Worth Knowing

- **Continuous Integration (CI)** — every push merges to the shared branch and triggers an automated **build + test**. Goal: catch integration problems in minutes, not at sprint end.
- **Continuous Delivery (CD)** — every passing CI run produces an artifact that *could* be deployed with a single manual approval (a human clicks "deploy").
- **Continuous Deployment (CD)** — every commit that passes automated checks is deployed to production automatically, no human in the loop (needs high test coverage, feature flags, automated rollback).

```
CI            → every commit: build + test → artifact ready
Delivery      → artifact can be deployed any time (human approves)
Deployment    → artifact auto-deploys to production
```

A typical pipeline: **checkout → build → unit tests → package (e.g., Docker image) → deploy to staging → (approval) → deploy to production.** Deployment strategies you might hear of: **blue-green** and **canary** (gradual rollout).

> **Interview soundbite:** "CI builds and tests every push so integration issues surface fast; continuous delivery keeps an always-deployable artifact, continuous deployment ships it automatically. I've set up a basic GitHub Actions workflow to build and test a Spring Boot app."

---

*Trimmed to awareness level for junior job prep. Restore the full CI/CD deep-dive from version control when you're ready to study it.*
