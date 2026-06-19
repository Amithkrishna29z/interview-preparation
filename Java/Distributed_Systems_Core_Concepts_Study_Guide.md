# Distributed Systems Core Concepts — Awareness Notes

> **Scope note (junior job prep):** Deep distributed-systems / system-design material is **deferred for later study**. This file is trimmed to a one-line-per-concept awareness list. The junior-level versions of these topics (load balancing, CAP theorem, caching, monolith-vs-microservices, message queues) are covered at the right depth in **`System_Design_Microservices_Interview_Questions.md`**, which is kept in full. The full distributed-systems deep-dive remains in git history for when you study system design.

---

## Know These Exist (one-liner each)

- **Load balancing** — a "traffic cop" distributing requests across servers (Round Robin, Least Connections, IP-hash). Prefer a shared Redis session store over sticky sessions so servers stay stateless.
- **CAP theorem** — a distributed system can only guarantee 2 of {Consistency, Availability, Partition-tolerance}. Partitions are unavoidable, so the real choice is **CP** (e.g. Zookeeper, banking) vs **AP** (e.g. Cassandra, social feeds). ACID ≈ CP, BASE ≈ AP.
- **Eventual consistency** — replicas converge "eventually" after async replication; reads can be briefly stale (like DNS propagation).
- **Distributed locks** — ensure one process at a time across servers; commonly `SET key val NX PX ttl` in Redis (release with a Lua check-then-delete). The TTL prevents deadlock if the holder crashes.
- **Database sharding** — split data across servers (range / hash / consistent-hashing). A good shard key is high-cardinality, immutable, and evenly distributed. Exhaust vertical scaling + caching + read-replicas first.
- **Replication** — keep copies of data on multiple servers for availability and read scaling. Primary-replica (sync = safe/slow, async = fast/possible loss); watch for replication lag and split-brain.
- **Message queues** — async, decoupled service communication. At-least-once delivery is most common → make consumers **idempotent**; failed messages go to a Dead Letter Queue. Kafka (log, replay, high throughput) vs RabbitMQ (routing, task queues).

> **Interview soundbite:** "I understand these at a conceptual level — CAP, eventual consistency, sharding, replication, load balancing, message queues — but deep system design is something I'm growing into; my hands-on strength is building Spring Boot CRUD services."

---

*Trimmed to awareness level for junior job prep. Restore the full distributed-systems deep-dive from version control when you're ready to study system design.*
