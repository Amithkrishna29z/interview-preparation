# Clean Architecture — Awareness Notes

> **Scope note (junior job prep):** Clean Architecture (the four layers, Ports & Adapters / Hexagonal, DDD alignment, dependency inversion at the architecture level) is an **architectural-design topic that's more mid-senior — deferred for later study**. This file is trimmed to the core idea. **SOLID principles** (which underpin it) are kept in **`Software_Engineering_Principles_Interview_Questions.md`**. The full deep-dive remains in git history.

---

## The Core Idea (the 30-second version)

Clean Architecture keeps your **business logic independent of frameworks, databases, and UI**, so those outer details can change without touching the core.

- **The Dependency Rule:** source-code dependencies point **inward** — outer layers (controllers, DB) depend on inner layers (business rules), never the reverse. The domain doesn't know about Spring or SQL.
- **The layers (inside → out):** Entities (core business rules) → Use Cases (application logic) → Interface Adapters (controllers, presenters, repositories) → Frameworks & Drivers (Spring, DB, web).
- **Ports & Adapters (Hexagonal)** is the same idea: the core defines interfaces ("ports") and the outside world plugs in via "adapters."

For most junior Spring Boot apps, the practical version is the familiar **Controller → Service → Repository** layering with DTOs at the edges — that already separates concerns reasonably.

> **Interview soundbite:** "Clean Architecture is about keeping business logic independent of frameworks and the database, with dependencies pointing inward. In Spring Boot I apply a lighter version — controllers, services, repositories, and DTOs — to keep the layers decoupled."

---

*Trimmed to awareness level for junior job prep. Restore the full Clean Architecture guide from version control when you're ready to study it.*
