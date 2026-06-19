# Next.js — Awareness Notes

> **Scope note (junior job prep):** Next.js is a React meta-framework with its own server/SSR layer. For a **junior full-stack *Java* role**, the usual frontend is plain **React** (Vite/CRA) calling a **Spring Boot** API — Next.js's built-in backend would overlap Spring Boot, so it's **deferred unless a specific job posting asks for it**. Your core frontend (React, JavaScript, TypeScript, HTML, CSS) is kept in full. The deep Next.js guide remains in git history.

---

## What It Is (the 30-second version)

**Next.js** is a production framework built on React that adds **file-based routing**, **server-side rendering**, API routes, and build optimizations on top of plain React.

### Rendering strategies to recognize
- **CSR** (Client-Side Rendering) — plain React: the browser renders everything (what you do with Vite/CRA + a Spring Boot API).
- **SSR** (Server-Side Rendering) — HTML rendered on the server per request (good for SEO / fresh data).
- **SSG** (Static Site Generation) — HTML built once at build time (fastest, for content that rarely changes).
- **ISR** (Incremental Static Regeneration) — SSG that re-generates pages periodically.

### Other concepts (one-liner)
- **File-based routing** — files in the `app/` (or `pages/`) folder become routes automatically.
- **Server vs Client Components** — components render on the server by default (App Router); add `"use client"` for interactivity.

> **Interview soundbite:** "I know Next.js is a React framework adding SSR/SSG and file-based routing — useful for SEO and JS-centric full-stack apps. For a Java backend role I'd build the React frontend against the Spring Boot API; I'd pick up Next.js specifics if the team uses it."

---

*Trimmed to awareness level for junior Java full-stack prep. Restore the full Next.js guide from version control if a role calls for it.*
