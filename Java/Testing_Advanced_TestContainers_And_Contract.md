# Advanced Testing: TestContainers, Contract Testing & Integration — Awareness Notes

> **Scope note (junior job prep):** Advanced testing tools (TestContainers, Pact contract testing, WireMock, REST Assured, Cucumber BDD) are **deferred for later study**. Your junior-level testing essentials — JUnit 5, Mockito, the Spring Boot test slices (`@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`), AssertJ, parameterized tests, and TDD — are kept in full in **`Java_Testing_JUnit_Mockito_Guide.md`**. The full advanced-testing deep-dive remains in git history.

---

## Tools to Recognize (one-liner each)

- **TestContainers** — spins up real dependencies (a real PostgreSQL, Kafka, Redis) in throwaway Docker containers for integration tests, instead of mocks or in-memory H2. Needs Docker installed.
- **Pact (contract testing)** — verifies that a consumer and provider agree on an API "contract," catching breaking changes between microservices without full end-to-end tests.
- **WireMock** — stubs/mocks external HTTP services so you can test your code against fake-but-realistic responses.
- **REST Assured** — a fluent Java library for writing readable API integration tests (`given().when().then()`).
- **Cucumber (BDD)** — write tests as plain-English Given/When/Then scenarios that map to step definitions.

> **Interview soundbite:** "I'm strong on unit testing with JUnit and Mockito and the Spring Boot test slices. I'm aware of TestContainers for real-dependency integration tests and contract testing with Pact for microservices — tools I'd adopt as the project needs them."

---

*Trimmed to awareness level for junior job prep. Restore the full advanced-testing deep-dive from version control when you're ready to study it.*
