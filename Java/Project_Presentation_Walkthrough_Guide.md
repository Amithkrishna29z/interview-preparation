# Project Presentation Walkthrough Guide

## Overview

"Walk me through your project" is the question that can make or break a junior interview. For developers with limited work experience, a personal or portfolio project is often the **centerpiece** of the technical interview. This guide gives you a repeatable structure to present any full-stack Spring Boot + React project confidently, handle follow-up questions, and demonstrate the engineering judgment interviewers are actually looking for.

---

## Table of Contents

1. [What Interviewers Are Really Assessing](#what-interviewers-are-really-assessing)
2. [The 8-Step Walkthrough Framework](#the-8-step-walkthrough-framework)
3. [Worked Example: Task Manager App](#worked-example-task-manager-app)
4. [How to Talk About Decisions and Tradeoffs](#how-to-talk-about-decisions-and-tradeoffs)
5. [Sentence-Level Coaching](#sentence-level-coaching)
6. [Common Interview Questions](#common-interview-questions)
7. [Red Flags and Green Flags](#red-flags-and-green-flags)
8. [Pre-Interview Prep Checklist](#pre-interview-prep-checklist)
9. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What Interviewers Are Really Assessing

When an interviewer asks "walk me through your project," they are not just asking you to describe what the app does. They are probing four things:

| What They Ask | What They Are Really Testing |
|---|---|
| "Tell me about your project" | Can you communicate technical decisions clearly to a non-expert? |
| "What did YOU build?" | Do you own your work, or did you copy-paste without understanding? |
| "Why did you choose X?" | Do you make reasoned decisions, or do you just follow tutorials? |
| "What would you change?" | Do you have engineering judgment and self-awareness? |

The app itself matters less than how you explain it. A simple CRUD app explained with confidence and tradeoff awareness beats a complex app you cannot defend.

---

## The 8-Step Walkthrough Framework

Use this structure every time. Practice it out loud until it flows naturally. Aim for 4-6 minutes for the overview, leaving time for follow-up questions.

### Step 1 — Goal and Stack

State the problem and justify your technology choices.

- What problem does the app solve? One or two sentences.
- What is the stack? (React frontend, Spring Boot backend, PostgreSQL, deployed on AWS EC2 / Railway / Render)
- Why did you choose this stack? Give a brief, honest reason for each major choice.

> "I built a Task Manager app to practice full-stack development end to end. I used React because component-based UI maps well to a task list with reusable cards, Spring Boot because it gives you a production-ready REST API with very little boilerplate, and PostgreSQL because the task-user relationship is relational and I wanted to practice writing proper foreign keys and JPA mappings."

### Step 2 — Frontend Architecture

Explain your React structure, not just that you "used React."

- How is the UI broken into components? Name two or three key ones.
- What props do components accept? How is data passed down?
- How does routing work? (React Router `<Routes>` / `<Route>`, protected routes)
- How does data fetching work? (useEffect + fetch/Axios, loading state, error state)

> "The frontend has a TaskBoard component that fetches all tasks on mount. It passes each task as a prop to a TaskCard component. I used React Router for client-side navigation — the login page is public, everything else is behind a PrivateRoute wrapper that checks for a JWT in localStorage. When data is loading I show a spinner; on error I show an error message."

### Step 3 — Backend Architecture

Describe the layered architecture. Name the layers explicitly.

```
HTTP Request
    → Controller  (maps URL to method, validates request DTO)
    → Service     (business logic, transactions)
    → Repository  (Spring Data JPA interface, talks to DB)
    → Database    (PostgreSQL)
```

- What REST endpoints does the API expose? Give two or three examples.
- What does each layer's responsibility look like in your code?
- Why DTOs instead of exposing entities directly?

> "The backend follows a standard layered architecture. The TaskController maps HTTP requests to endpoints like POST /api/tasks. It hands off to TaskService, which contains the business logic — for example, only the task's owner can mark it complete. The service calls TaskRepository, which is a JPA interface that Spring generates queries for at runtime. I used DTOs to avoid exposing the database entity fields directly to the client, which also protects against mass assignment."

### Step 4 — Database Design

Show that you understand relational data modeling, not just that you "used a database."

- What tables do you have?
- What are the relationships (one-to-many, many-to-many)?
- How are foreign keys set up? What does referential integrity look like?
- Any important transaction considerations?

> "I have two main tables: users and tasks. Tasks has a user_id foreign key that references users.id with ON DELETE CASCADE, so deleting a user cleans up their tasks. In JPA this is a @ManyToOne relationship on the Task entity. I used @Transactional on the service method that creates a task and sends a confirmation email, so if the email fails, the task insert is rolled back."

### Step 5 — Integration and Security

Explain the full request-response cycle from frontend to database, and how auth works.

- How does the React app call the Spring Boot API? (Axios base URL, request headers)
- How does JWT authentication work end to end? (Login → get token → store → attach to headers → backend validates)
- How is CORS handled?
- Is any data validated on the backend?

> "The frontend uses Axios with a base URL pointing to the backend. On login, the backend authenticates the credentials, signs a JWT with a secret key, and returns it. The frontend stores the token in localStorage and attaches it as a Bearer token in the Authorization header on every subsequent request. A Spring Security filter intercepts every request, validates the JWT, and populates the SecurityContext. CORS is configured on the backend to allow requests from the React dev server origin only."

### Step 6 — Testing Approach

Describe what you tested and how, even if your coverage is limited.

- Unit tests: JUnit 5 + Mockito — test the service layer with mocked repositories.
- Integration tests: @SpringBootTest or @WebMvcTest — test the controller layer.
- What did you NOT test and why? (Honest acknowledgment is fine for a junior.)

> "I wrote unit tests for the TaskService using JUnit 5 and Mockito. I mocked the TaskRepository and verified that the service throws an AccessDeniedException when a user tries to update someone else's task. I used @WebMvcTest to test the controller layer and verify that 401 is returned when no JWT is provided. I did not write end-to-end tests, which is something I would add next."

### Step 7 — Deployment (Bonus)

If you deployed the project, mention it. Interviewers appreciate any production exposure.

- Where is it hosted? (AWS EC2, Railway, Render, Fly.io, Heroku)
- Is it containerized? (Dockerfile, docker-compose for local dev)
- Any CI/CD? (GitHub Actions running tests on push)

> "I containerized the backend with Docker and wrote a docker-compose file for local development that spins up Spring Boot and PostgreSQL together. The app is deployed on Railway — the backend and frontend are separate services. I added a GitHub Actions workflow that runs the JUnit tests on every push to main and blocks the merge if tests fail."

### Step 8 — Challenges, Tradeoffs, and What You Would Change

This is the highest-signal part of the walkthrough. Have a specific, honest answer prepared.

- What was the hardest bug or problem you faced? How did you solve it?
- What deliberate tradeoffs did you make?
- What would you improve with more time?

> "The hardest issue was CORS — the frontend was getting blocked because my Spring Security configuration was stripping the CORS headers before the CORS filter could add them. I fixed it by explicitly calling cors() in the SecurityFilterChain and providing a CorsConfigurationSource bean. If I had more time I would add refresh tokens so users are not logged out every hour, and I would add pagination to the task list so it does not degrade with large data sets."

---

## Worked Example: Task Manager App

Below is a condensed sample script for a Task Manager app. Adapt this to your own project.

---

**"I built a full-stack Task Manager. The problem it solves is organizing personal to-dos with user accounts so tasks are private to each user.**

**Stack: React for the frontend, Spring Boot for the REST API, PostgreSQL for the database, deployed on Railway. I chose Spring Boot because I wanted experience with a production-grade Java framework, and PostgreSQL because the user-task relationship fits naturally in a relational model.**

**On the frontend, I have three main components: a Login page, a TaskBoard that fetches and displays all tasks for the logged-in user, and a TaskCard that is a reusable component accepting a task object as a prop. React Router handles navigation with a PrivateRoute that redirects to login if no token is found in localStorage.**

**The backend is layered: TaskController handles HTTP routing, TaskService has the business logic, and TaskRepository is a Spring Data JPA interface. I used DTOs on the API boundary so internal entity fields like createdAt are not exposed unless I explicitly include them.**

**The database has two tables: users and tasks. Tasks.user_id is a foreign key to users.id. In JPA this is a @ManyToOne mapping. I added @Transactional to service methods that write to the DB.**

**For auth: on login, the backend validates credentials, creates a JWT signed with a secret, and returns it. The React app stores the token in localStorage and adds it as a Bearer header on every Axios request. A Spring Security OncePerRequestFilter validates the token before the request hits any controller.**

**I wrote JUnit tests for the service layer using Mockito to mock the repository, and @WebMvcTest tests for the controller. I did not write full integration tests, which I would add next.**

**The hardest problem was that deleting a user was failing because of a foreign key constraint violation on the tasks table. I fixed it by adding cascade = CascadeType.ALL and orphanRemoval = true on the @OneToMany relationship, and by setting ON DELETE CASCADE in the migration script.**

**If I rebuilt this I would add refresh tokens, server-side pagination, and move the JWT secret to an environment variable stored in a secrets manager rather than application.properties."**

---

### API Surface Reference (Task Manager)

Know your own endpoints cold. Here is a typical minimal set for this project:

| Method | Endpoint | Auth required | What it does |
|---|---|---|---|
| POST | /api/auth/register | No | Create a new user account |
| POST | /api/auth/login | No | Return JWT on valid credentials |
| GET | /api/tasks | Yes | Return all tasks for the logged-in user |
| POST | /api/tasks | Yes | Create a new task |
| PUT | /api/tasks/{id} | Yes | Update a task (owner only) |
| DELETE | /api/tasks/{id} | Yes | Delete a task (owner only) |

If asked "how does the frontend know which tasks belong to the user?" — the answer is that the backend reads the user identity from the JWT, not from a query parameter. The client never sends a userId; the server extracts it from the validated token.

---

## How to Talk About Decisions and Tradeoffs

Never say "I just used it because it's popular" or "the tutorial used it." Use this pattern:

> **Decision → Alternatives I considered → Why I chose this one**

| Decision | Alternative | Why I chose this |
|---|---|---|
| JWT for auth | Server-side sessions | JWTs are stateless — the backend does not need a session store, which is simpler for a single-server app |
| PostgreSQL | MySQL, MongoDB | The data is relational (users own tasks). Relational DB gives me foreign keys and JOIN queries natively |
| DTOs in API | Expose JPA entity directly | Entities can have lazy-loaded fields, bidirectional relationships that cause infinite JSON loops, and internal fields I do not want to expose |
| Spring Data JPA | Plain JDBC, MyBatis | JPA handles the ORM boilerplate. For this project complexity, derived queries and @Query are enough |
| React Router | Next.js routing | This is a client-side SPA; I did not need SSR, so React Router was simpler |

Even if your reason is "I wanted to learn it," frame it: "I chose X because I wanted hands-on experience with it, and it was appropriate for this scale of project."

---

## Sentence-Level Coaching

How you say something is as important as what you say. These patterns help you sound like an engineer, not a student reciting notes.

**Owning your decisions:**
- Weak: "I used JWT because that's what the tutorial did."
- Strong: "I chose JWT because the backend is stateless — there is no session store to manage, which keeps the deployment simpler."

**Talking about limitations honestly:**
- Weak: "I didn't have time to write tests."
- Strong: "I wrote unit tests for the service layer. I did not write integration tests for the full stack, which is the next thing I would add because right now I am testing the layers in isolation but not the wiring between them."

**Handling a question you cannot fully answer:**
- Weak: "I'm not sure, I would have to look it up."
- Strong: "I know that X is responsible for Y, but I would need to check the exact Spring Security interface name. What I do know is how the overall flow works: [explain the flow]."

**Talking about team vs. solo projects:**
- Solo: "I built this end to end myself — backend, frontend, deployment, and the CI pipeline. That was intentional: I wanted to own every layer so I understood where things fit together."
- Team: "My contribution was the entire backend: the JPA entity model, the Spring Security configuration, and the REST API. My teammate built the React frontend. I reviewed his API calls to make sure they matched what the backend expected."

**Closing the walkthrough:**
- Always end by inviting follow-up: "That is the high-level view — happy to go deeper on any part."

---

## Common Interview Questions

### "What was the hardest bug you fixed?"

Prepare one specific bug with a root cause and resolution. Use the structure: symptom → investigation → root cause → fix. Avoid vague answers like "there were many small bugs."

Example: "The app was returning a 403 for all authenticated requests even though the JWT was valid. I added logging to the security filter and found the token was being stripped from the Authorization header before reaching the filter. The root cause was that I had not configured Spring Security to explicitly permit the CORS pre-flight OPTIONS request, so the browser's preflight was being rejected and the actual request never reached the JWT filter. I fixed it by adding `.requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()` to the security config."

### "How does authentication work end to end?"

Walk through the full flow step by step:
1. User submits credentials (POST /api/auth/login)
2. Backend loads the user from the DB and verifies the password with BCrypt
3. Backend signs a JWT containing userId and email with an HMAC-SHA256 secret and sets an expiry (e.g., 1 hour)
4. JWT is returned in the response body
5. Frontend stores the token in localStorage
6. On every subsequent request, frontend attaches it as `Authorization: Bearer <token>`
7. Spring Security's `OncePerRequestFilter` reads the header, validates the signature and expiry, extracts the userId, and sets the `Authentication` in `SecurityContextHolder`
8. The controller method runs; if it needs the current user it reads from `SecurityContextHolder`

Know what is in your JWT payload: at minimum userId (or username), and expiry (exp claim).

### "How would you scale this if it got 10x more users?"

You do not need a perfect answer. Acceptable junior answer: "Right now the app runs on a single server. To scale I would first add database connection pooling if it is not already configured (HikariCP ships with Spring Boot), then add pagination so queries do not return unbounded result sets, then cache frequently read data with Redis. For stateless horizontal scaling, JWT already helps because there is no session state on the server — I could add more instances behind a load balancer without a shared session store."

### "Why did you choose this database?"

Use the decision pattern above. If you chose PostgreSQL: relational data model, ACID transactions, strong JPA/Hibernate support, free and widely used in production. If asked why not MongoDB: "The data has clear relationships and fixed structure. A document store would be a better fit if the schema were highly variable, but for users and tasks, a relational model with foreign keys is the right tool."

### "What would you change or improve?"

Have three concrete answers ready. Good examples: refresh tokens (so users are not forced to log in every hour), server-side pagination (so the task list does not degrade at scale), input validation with `@Valid` and Bean Validation annotations on DTOs, moving secrets to environment variables or a secrets manager, adding a CI/CD pipeline, writing more test coverage (especially integration tests with @SpringBootTest).

### "Did you work on this alone?"

Answer honestly. If alone: say so and own every part of it. If with others: clearly state YOUR specific contributions. "I built the entire backend including the Spring Security configuration and the JPA entity model. My partner built the React frontend."

### "How did you handle errors?"

Describe your error handling strategy: a `@ControllerAdvice` class with `@ExceptionHandler` methods that map application exceptions to HTTP status codes and return a consistent JSON error response body (e.g., `{ "error": "Task not found", "status": 404 }`). On the frontend, Axios intercepts non-2xx responses and displays the error message from the response body.

### "How is data validated?"

Backend: `@Valid` on controller method parameters with Bean Validation annotations (`@NotBlank`, `@Size`, `@Email`) on DTO fields. Spring returns 400 Bad Request automatically if validation fails. Frontend: basic client-side validation before submission (e.g., disabling the submit button if the title is empty), but the backend is the authoritative source of truth and always validates independently.

### "Walk me through a single request — from click to database and back."

This is a common deep-dive. Use this structure: user clicks "Create Task" → React calls `POST /api/tasks` with the task DTO in the request body and JWT in the header → Spring Security filter validates the JWT → TaskController receives the request, calls `@Valid` on the DTO → TaskService creates a Task entity, sets the owner from the SecurityContext, calls `taskRepository.save()` → Spring Data JPA generates an INSERT → PostgreSQL persists the row → the saved entity is mapped to a response DTO → 201 Created is returned → React updates the task list in state.

---

## Red Flags and Green Flags

| Red Flag | Green Flag |
|---|---|
| "I just followed a tutorial" | "I used a tutorial to start, then extended it with X, Y, Z" |
| Cannot explain what a library does | Knows why each dependency is in pom.xml |
| Takes credit for team work they did not do | Clearly scopes their own contribution |
| No tradeoffs — "I just used whatever seemed best" | Can state one alternative and why they did not pick it |
| Cannot explain the auth flow end to end | Can walk through the JWT lifecycle step by step |
| "The app kind of works but I am not sure why" | Can draw the architecture from memory |
| Vague answer to "what would you improve?" | Has 2-3 specific, technically grounded improvements ready |
| Never deployed or ran the app in a real environment | Has numbers: "the app handles X requests, the DB has Y rows" |

---

## Pre-Interview Prep Checklist

Complete this the evening before an interview where you will discuss a project.

**Know your code:**
- [ ] Re-read every class in the project — no surprises about your own code
- [ ] Can draw the architecture on a whiteboard from memory: frontend → API → service → repo → DB
- [ ] Know every dependency in pom.xml and package.json and why it is there
- [ ] Know your database schema: table names, column types, foreign keys, and any constraints

**Know your decisions:**
- [ ] Can explain the auth flow end to end without looking at code
- [ ] Have a one-sentence "why" for the three most important technology choices (DB, auth strategy, framework)
- [ ] Know one alternative for each major choice and why you did not pick it

**Have stories ready:**
- [ ] One specific bug story: symptom → investigation → root cause → fix
- [ ] Three concrete "what I would improve" answers (not vague — technically specific)
- [ ] Clear statement of your own contribution if it was a team project

**Have numbers ready:**
- [ ] Approximate number of API endpoints
- [ ] Number of DB tables and the key relationships
- [ ] Any performance observations ("loads in about 300ms locally", "the DB has around 5 tables")

**Final checks:**
- [ ] Run the app locally and verify it still works end to end
- [ ] Have the GitHub repository URL ready to share if asked
- [ ] Practice the 8-step walkthrough out loud — once all the way through, timed

---

## Quick Reference Cheat Sheet

**The 8 steps (in order):**
1. Goal + stack + why
2. Frontend: components, props, routing, data fetching, error states
3. Backend: Controller → Service → Repository → DB
4. Database: tables, relationships, foreign keys, transactions
5. Integration + security: Axios, JWT end to end, CORS
6. Testing: JUnit + Mockito, what you tested
7. Deployment: Docker, cloud host, CI/CD
8. Challenges, tradeoffs, what you would change

**Tradeoff pattern:** Decision → Alternatives → Why I chose this

**Auth flow (memorize this):**
Login → backend validates → signs JWT → frontend stores in localStorage → attaches as Bearer header → Spring Security filter validates → SecurityContext populated → controller runs

**Three safe "what I would improve" answers:**
- Refresh tokens (JWT expiry UX)
- Server-side pagination (performance)
- More test coverage (integration/E2E tests)

**One-line rule:** Every answer should show you understand your own code and made deliberate choices — even small ones.

**Common Spring Boot annotations to know cold:**

| Annotation | Where it goes | What it does |
|---|---|---|
| @RestController | Class | Marks as controller; @ResponseBody on every method |
| @RequestMapping / @GetMapping | Method | Maps HTTP verb + URL path to method |
| @Service | Class | Marks as service bean; business logic lives here |
| @Repository | Interface/Class | Marks as data access bean; enables exception translation |
| @Transactional | Method/Class | Wraps in a DB transaction; rolls back on unchecked exceptions |
| @Entity | Class | JPA entity mapped to a DB table |
| @ManyToOne / @OneToMany | Field | Defines the relationship side and join column |
| @Valid | Method parameter | Triggers Bean Validation on the annotated DTO |
| @ControllerAdvice | Class | Global exception handler across all controllers |
| @ExceptionHandler | Method | Handles a specific exception type and returns an HTTP response |

---

*Last Updated: 2026-06-18*
