# Next.js Interview Preparation Guide

> A focused guide covering the Next.js concepts a junior developer needs for interviews.

---

## Table of Contents

1. [What is Next.js?](#1-what-is-nextjs)
2. [Pages Router vs App Router](#2-pages-router-vs-app-router)
3. [File-Based Routing](#3-file-based-routing)
4. [Rendering Strategies — SSR, SSG, ISR, CSR](#4-rendering-strategies--ssr-ssg-isr-csr)
5. [Server Components vs Client Components](#5-server-components-vs-client-components)
6. [Data Fetching in App Router](#6-data-fetching-in-app-router)
7. [getStaticProps & getServerSideProps (Pages Router)](#7-getstaticprops--getserversideprops-pages-router)
8. [Dynamic Routes](#8-dynamic-routes)
9. [API Routes & Route Handlers](#9-api-routes--route-handlers)
10. [Layouts & Templates](#10-layouts--templates)
11. [Middleware](#11-middleware)
12. [Image & Font Optimization](#12-image--font-optimization)
13. [Metadata & SEO](#13-metadata--seo)
14. [Server Actions](#14-server-actions)
15. [Caching in Next.js](#15-caching-in-nextjs)
16. [Streaming & Suspense](#16-streaming--suspense)
17. [Error Handling & Loading UI](#17-error-handling--loading-ui)
18. [Environment Variables](#18-environment-variables)
19. [Authentication in Next.js](#19-authentication-in-nextjs)
20. [Performance Optimization](#20-performance-optimization)
21. [Deployment & Build](#21-deployment--build)
22. [Common Interview Questions & Answers](#22-common-interview-questions--answers)
23. [Quick Revision Cheat Sheet](#23-quick-revision-cheat-sheet)

---

## 1. What is Next.js?

Next.js is a **React framework** built by Vercel that adds server-side capabilities, file-based routing, and production optimizations on top of React.

**Key features:**
- **File-based routing** — folders and files define your URL structure
- **Multiple rendering modes** — SSR, SSG, ISR, CSR per page
- **Full-stack** — API routes / Route Handlers run on the server
- **Built-in optimizations** — image, font, script, and bundle optimization
- **Server Components** — components that run only on the server (App Router)

```bash
npx create-next-app@latest my-app

# App Router project structure
my-app/
├── app/
│   ├── layout.tsx       # Root layout
│   ├── page.tsx         # Home page (/)
│   └── api/
│       └── route.ts     # API Route Handler
├── public/
└── next.config.js
```

> **Interview tip:** Next.js is a **framework** (opinionated, full-stack), while React is a **library** (only UI). Choose Next.js when you need SSR, SEO, or a backend alongside React.

---

## 2. Pages Router vs App Router

Next.js has two routing paradigms. The **App Router** (v13+) is the modern standard; the **Pages Router** is legacy but still fully supported.

| Feature | Pages Router (`/pages`) | App Router (`/app`) |
|---|---|---|
| Server Components | No | Yes (default) |
| Data fetching | `getStaticProps`, `getServerSideProps` | `async/await` in component |
| Layouts | Custom `_app.tsx` | Nested `layout.tsx` files |
| Loading states | Manual | `loading.tsx` |
| Error handling | `_error.tsx` | `error.tsx` |
| Streaming | No | Yes (React Suspense) |
| Route Handlers | `pages/api/*` | `app/api/*/route.ts` |

> **Interview tip:** New projects should use the **App Router** — it enables React Server Components, co-located loading/error states, and nested layouts, all of which reduce boilerplate and improve performance.

---

## 3. File-Based Routing

Next.js derives your URL structure from the file system — no router config needed.

```
# App Router special files per segment
app/
├── layout.tsx         # Shared UI wrapping all children
├── page.tsx           # The UI for this route (required to make it public)
├── loading.tsx        # Loading skeleton shown via Suspense
├── error.tsx          # Error boundary for this segment
├── not-found.tsx      # 404 for this segment
└── route.ts           # API endpoint

# Route groups — organize without affecting the URL
app/
├── (marketing)/       # () = not part of URL
│   └── about/page.tsx → /about
└── (dashboard)/
    ├── layout.tsx      # Layout only for dashboard routes
    └── settings/page.tsx → /settings
```

> **Interview tip:** Know the difference between **route groups** `(name)`, **dynamic segments** `[slug]`, **catch-all** `[...slug]`, and **optional catch-all** `[[...slug]]`.

---

## 4. Rendering Strategies — SSR, SSG, ISR, CSR

This is one of the most important Next.js interview topics.

### Static Site Generation (SSG)
HTML generated at **build time**. Fastest — served from CDN.

```tsx
// app/blog/page.tsx — SSG by default (force-cache is the default)
export default async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    cache: 'force-cache',
  }).then(r => r.json());
  return <PostList posts={posts} />;
}
```

### Server-Side Rendering (SSR)
HTML generated on **every request**. Fresh data, but slower — cannot be CDN-cached.

```tsx
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic';

export default async function DashboardPage() {
  const data = await fetch('https://api.example.com/live', {
    cache: 'no-store',
  }).then(r => r.json());
  return <Dashboard data={data} />;
}
```

### Incremental Static Regeneration (ISR)
HTML generated at build time, then **regenerated in the background** after an interval.

```tsx
export default async function ProductPage() {
  const product = await fetch('https://api.example.com/product/1', {
    next: { revalidate: 60 }, // re-generate every 60 seconds
  }).then(r => r.json());
  return <Product data={product} />;
}
```

### Client-Side Rendering (CSR)
Data fetched in the **browser** after the page loads.

```tsx
'use client';
import { useState, useEffect } from 'react';

export default function UserProfile() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch('/api/me').then(r => r.json()).then(setUser);
  }, []);
  if (!user) return <p>Loading...</p>;
  return <p>{user.name}</p>;
}
```

| Strategy | When HTML is built | Best for |
|---|---|---|
| SSG | Build time | Blog posts, docs, marketing |
| SSR | Every request | Dashboards, personalized pages |
| ISR | Build + background refresh | E-commerce, news |
| CSR | In browser | Private, real-time, interactive UI |

---

## 5. Server Components vs Client Components

The App Router's biggest paradigm shift — components are **Server Components by default**.

### Server Components
- Run **only on the server** — code never shipped to browser
- Can be `async` and `await` data directly
- No access to `useState`, `useEffect`, event handlers, or browser APIs

```tsx
// app/posts/page.tsx — Server Component (no directive needed)
import { db } from '@/lib/db'; // Safe — never sent to browser

export default async function PostsPage() {
  const posts = await db.query('SELECT * FROM posts');
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

### Client Components
- Required for: `useState`, `useEffect`, event handlers, browser APIs

```tsx
'use client'; // Must be the first line

import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### Composition pattern — Server wraps Client

```tsx
// WRONG — importing a server component inside a client component
'use client';
import ServerComp from './ServerComp'; // breaks server-only guarantee

// CORRECT — pass server content as children
'use client';
export default function ClientShell({ children }) {
  return <div onClick={...}>{children}</div>;
}

// Parent (Server Component)
import ClientShell from './ClientShell';
import ServerComp from './ServerComp';

export default function Page() {
  return (
    <ClientShell>
      <ServerComp /> {/* stays a Server Component */}
    </ClientShell>
  );
}
```

---

## 6. Data Fetching in App Router

The App Router uses native `fetch` with extended options for caching.

```tsx
// SSG — cached indefinitely
const data = await fetch(url, { cache: 'force-cache' });

// SSR — never cached
const data = await fetch(url, { cache: 'no-store' });

// ISR — cached, refreshed after N seconds
const data = await fetch(url, { next: { revalidate: 3600 } });

// Tag-based revalidation — invalidate on demand
const data = await fetch(url, { next: { tags: ['posts'] } });
import { revalidateTag } from 'next/cache';
revalidateTag('posts');
```

**Avoid waterfalls — fetch in parallel:**

```tsx
// WRONG — sequential
const user = await fetchUser();
const posts = await fetchPosts(user.id);

// CORRECT — parallel
const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);
```

---

## 7. getStaticProps & getServerSideProps (Pages Router)

Legacy Pages Router data fetching — still valid but App Router is preferred.

```tsx
// pages/blog/[slug].tsx

// SSG — runs at build time
export async function getStaticProps({ params }) {
  const post = await fetchPost(params.slug);
  return { props: { post }, revalidate: 60 }; // ISR with 60s interval
}

// Required for dynamic SSG routes
export async function getStaticPaths() {
  const slugs = await fetchAllSlugs();
  return {
    paths: slugs.map(slug => ({ params: { slug } })),
    fallback: 'blocking',
  };
}

// SSR — runs on every request
export async function getServerSideProps({ params }) {
  const data = await fetchData(params.slug);
  return { props: { data } };
}

export default function BlogPost({ post }) {
  return <article>{post.title}</article>;
}
```

**`fallback` values:**

| Value | Behavior |
|---|---|
| `false` | Unbuilt paths → 404 |
| `true` | Unbuilt paths → loading state, then generated |
| `'blocking'` | Unbuilt paths → SSR on first hit, then cached as static |

---

## 8. Dynamic Routes

```
app/
├── blog/[slug]/page.tsx        → /blog/hello-world
├── shop/[...categories]/page.tsx → /shop/a/b/c  (catch-all)
└── docs/[[...path]]/page.tsx   → /docs  OR  /docs/a/b  (optional catch-all)
```

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogPost({ params, searchParams }) {
  const post = await fetchPost(params.slug);
  return <article>{post.title}</article>;
}

// Generate static pages at build time (replaces getStaticPaths)
export async function generateStaticParams() {
  const posts = await fetchAllPosts();
  return posts.map(post => ({ slug: post.slug }));
}

// Dynamic metadata per route
export async function generateMetadata({ params }) {
  const post = await fetchPost(params.slug);
  return { title: post.title, description: post.excerpt };
}
```

---

## 9. API Routes & Route Handlers

**Pages Router** — files in `pages/api/` become HTTP endpoints.

```tsx
// pages/api/users/[id].ts
export default async function handler(req, res) {
  const { id } = req.query;
  if (req.method === 'GET') {
    const user = await db.findUser(id);
    if (!user) return res.status(404).json({ error: 'Not found' });
    return res.status(200).json(user);
  }
  res.setHeader('Allow', ['GET']);
  res.status(405).end(`Method ${req.method} Not Allowed`);
}
```

**App Router** — `route.ts` files, uses Web APIs.

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const users = await db.getUsers();
  return NextResponse.json(users);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const user = await db.createUser(body);
  return NextResponse.json(user, { status: 201 });
}

// app/api/users/[id]/route.ts
export async function DELETE(request, { params }) {
  await db.deleteUser(params.id);
  return new NextResponse(null, { status: 204 });
}
```

> **Interview tip:** Route Handlers use Web APIs (`Request`/`Response`) unlike Pages Router which uses Node.js `req`/`res`. This makes them more portable and Edge-compatible.

---

## 10. Layouts & Templates

**Layouts** persist across route changes — state is preserved, they don't re-render. **Templates** create a new instance on each navigation.

```tsx
// app/layout.tsx — Root layout (required)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <nav>/* persistent — won't re-render on navigation */</nav>
        <main>{children}</main>
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx — Nested layout
export default function DashboardLayout({ children }) {
  return (
    <div className="dashboard">
      <aside>/* Sidebar — persists within /dashboard/* */</aside>
      <section>{children}</section>
    </div>
  );
}
```

---

## 11. Middleware

Middleware runs **before a request is processed** — on the Edge runtime. Used for auth, redirects, rewrites.

```tsx
// middleware.ts (at project root)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value;
  const { pathname } = request.nextUrl;

  if (pathname.startsWith('/dashboard') && !token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
};
```

> **Interview tip:** Middleware runs on the **Edge runtime** — no Node.js APIs, no database calls. Use it for routing decisions only (auth checks, redirects, rewrites).

---

## 12. Image & Font Optimization

### Image

```tsx
import Image from 'next/image';

// Remote image
export default function ProductImage({ url }) {
  return (
    <Image
      src={url}
      alt="Product"
      width={800}
      height={600}
      sizes="(max-width: 768px) 100vw, 50vw"
      priority // preload above-the-fold images
    />
  );
}
```

```js
// next.config.js — allow remote domains
const nextConfig = {
  images: {
    remotePatterns: [{ protocol: 'https', hostname: 'cdn.example.com' }],
  },
};
```

Benefits over `<img>`: auto WebP/AVIF, lazy loading, CLS prevention, responsive `srcSet`.

### Font

```tsx
// app/layout.tsx
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] });

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

`next/font` self-hosts fonts — zero layout shift, no external network requests at runtime.

---

## 13. Metadata & SEO

```tsx
// Static metadata — app/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: { template: '%s | My App', default: 'My App' },
  description: 'Welcome to my app',
  openGraph: {
    images: [{ url: 'https://example.com/og.png' }],
  },
};

// Dynamic metadata — app/blog/[slug]/page.tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await fetchPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
  };
}
```

---

## 14. Server Actions

Functions that run **on the server**, triggered from Client Components or forms — no manual API route needed.

```tsx
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  if (!title) throw new Error('Title is required');
  const post = await db.posts.create({ title });
  revalidatePath('/blog');
  redirect(`/blog/${post.slug}`);
}
```

```tsx
// Using in a form — works without JavaScript (progressive enhancement)
import { createPost } from '@/app/actions';

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <button type="submit">Create Post</button>
    </form>
  );
}
```

> **Interview tip:** Server Actions replace simple API routes for mutations. They progressively enhance forms and can call `revalidatePath`/`revalidateTag` to refresh cache after mutation.

---

## 15. Caching in Next.js

Next.js has **four caching layers**:

| Cache | What it stores | Invalidation |
|---|---|---|
| Request Memoization | `fetch` dedup within a single render | Per request (automatic) |
| Data Cache | `fetch` results across requests | `revalidate`, `revalidateTag`, `revalidatePath` |
| Full Route Cache | Static HTML + RSC payload | Redeploy or revalidation |
| Router Cache | RSC payload on client | Navigate, `router.refresh()` |

```tsx
// On-demand revalidation
import { revalidatePath, revalidateTag } from 'next/cache';

revalidatePath('/blog');    // invalidate Full Route Cache for /blog
revalidateTag('posts');     // invalidate all fetches tagged 'posts'
```

---

## 16. Streaming & Suspense

Streaming sends HTML to the browser in **chunks** — critical content first, slower content later.

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div>
      <Suspense fallback={<p>Loading summary...</p>}>
        <Summary />       {/* fast */}
      </Suspense>
      <Suspense fallback={<p>Loading chart...</p>}>
        <RevenueChart />  {/* slow */}
      </Suspense>
    </div>
  );
}
```

Without streaming: browser waits for the slowest component. With streaming: fast components arrive immediately, slow ones trickle in.

---

## 17. Error Handling & Loading UI

```tsx
// app/dashboard/error.tsx — must be a Client Component
'use client';

export default function Error({ error, reset }) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// app/dashboard/loading.tsx — auto Suspense fallback
export default function Loading() {
  return <div className="animate-pulse">Loading...</div>;
}

// Trigger 404 from a Server Component
import { notFound } from 'next/navigation';

export default async function BlogPost({ params }) {
  const post = await fetchPost(params.slug);
  if (!post) notFound();
  return <article>{post.title}</article>;
}
```

`loading.tsx` automatically wraps `page.tsx` in a `<Suspense>` boundary — no manual wiring needed.

---

## 18. Environment Variables

```bash
# .env.local (never commit)
DATABASE_URL=postgresql://...
JWT_SECRET=supersecret
NEXT_PUBLIC_API_URL=https://api.example.com  # exposed to browser!
```

```tsx
// Server Component — access any env var
const db = new Database(process.env.DATABASE_URL);

// Client Component — only NEXT_PUBLIC_* vars
const apiUrl = process.env.NEXT_PUBLIC_API_URL; // works
const secret = process.env.JWT_SECRET;          // undefined in browser!
```

| Variable type | Server | Client |
|---|---|---|
| `MY_SECRET` | Yes | No |
| `NEXT_PUBLIC_MY_VAR` | Yes | Yes |

> **Interview tip:** Never prefix sensitive secrets with `NEXT_PUBLIC_` — they'll be embedded in the JS bundle and visible to anyone.

---

## 19. Authentication in Next.js

**NextAuth.js (Auth.js)** is the most common auth solution.

```tsx
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';

const handler = NextAuth({
  providers: [
    GitHub({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
  ],
});

export { handler as GET, handler as POST };

// Protecting a page — Server Component
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';

export default async function ProtectedPage() {
  const session = await getServerSession();
  if (!session) redirect('/login');
  return <p>Welcome, {session.user?.name}</p>;
}

// Protecting via Middleware (recommended for performance)
// middleware.ts
export { default } from 'next-auth/middleware';
export const config = { matcher: ['/dashboard/:path*'] };
```

---

## 20. Performance Optimization

### Dynamic Imports

```tsx
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <p>Loading chart...</p>,
  ssr: false, // skip SSR for this component
});
```

### Script Loading

```tsx
import Script from 'next/script';

// afterInteractive — loads after page is interactive (default)
<Script src="https://analytics.example.com/script.js" strategy="afterInteractive" />

// lazyOnload — loads during browser idle time
<Script src="https://ads.example.com/script.js" strategy="lazyOnload" />
```

### Core Web Vitals

| Metric | Target | Next.js Feature |
|---|---|---|
| LCP | < 2.5s | `<Image priority>`, font optimization |
| CLS | < 0.1 | `<Image>` with dimensions, `next/font` |
| INP | < 200ms | Server Components, reduce JS bundle |
| TTFB | < 800ms | SSG, ISR, CDN caching |

---

## 21. Deployment & Build

```bash
npm run build   # build for production
npm start       # start production server
```

| Deployment target | Config | Use case |
|---|---|---|
| Vercel | None (auto-detected) | Best integration |
| Docker / Node.js | `output: 'standalone'` | Self-hosted |
| Static hosting (S3/Netlify) | `output: 'export'` | No SSR needed |

```js
// next.config.js
const nextConfig = {
  output: 'standalone', // for Docker
  // output: 'export',  // for static hosting
};
```

---

## 22. Common Interview Questions & Answers

**Q: What is the difference between `getStaticProps` and `getServerSideProps`?**

> `getStaticProps` runs at **build time** — generates static HTML, served from CDN, fastest delivery. `getServerSideProps` runs on **every request** — always fresh but slower, cannot be CDN cached. Use `getStaticProps` + ISR for most cases; `getServerSideProps` only when you need per-request personalized data.

---

**Q: When would you use a Client Component vs a Server Component?**

> Default to **Server Components** — they reduce the client bundle and can access databases/secrets directly. Switch to `'use client'` only when you need `useState`, `useEffect`, event handlers, or browser APIs. The pattern is: fetch data on the server, pass to client for interactivity.

---

**Q: How does Next.js handle hydration?**

> Next.js renders HTML on the server then "hydrates" it on the client — React attaches event listeners to the existing HTML without re-rendering. Server Components are never hydrated (no JS shipped). A hydration mismatch (server HTML ≠ client render) causes a warning and full client re-render.

---

**Q: What is the purpose of `layout.tsx`?**

> It wraps all pages in its subtree with shared UI (nav, sidebar, footer). Unlike `template.tsx`, layouts **persist across navigation** — they don't unmount, so state like scroll position or open dropdowns is preserved when navigating between child routes.

---

**Q: How does caching work in Next.js App Router?**

> Four layers: (1) **Request Memoization** deduplicates identical `fetch` calls within a render. (2) **Data Cache** persists `fetch` responses across requests. (3) **Full Route Cache** stores rendered HTML for static routes. (4) **Router Cache** stores RSC payloads in the browser between navigations. Each layer has different invalidation mechanisms.

---

**Q: What is a Server Action and when would you use one?**

> Server Actions are `async` functions marked `'use server'` that run on the server but can be called from Client Components or HTML forms. They replace simple API routes for mutations — create, update, delete. Benefits: no manual `fetch` to an API route, works without JavaScript (progressive enhancement), and can call `revalidatePath`/`revalidateTag` after mutation.

---

**Q: How does Middleware differ from Route Handlers?**

> Middleware runs **before** a request reaches any route — on the Edge runtime, for routing decisions like auth checks, redirects, and rewrites. Route Handlers run **as** a route — they handle specific endpoints and can use Node.js APIs. Middleware cannot call databases efficiently; Route Handlers can.

---

**Q: What is ISR and how does it work?**

> ISR generates static pages at build time, then re-generates them in the background after a `revalidate` interval. The first request after the interval triggers a background rebuild — the current user still gets the stale page (stale-while-revalidate). The next request gets the fresh page. SSG performance with near-fresh data.

---

**Q: How do you optimize images in Next.js?**

> Use `<Image>` from `next/image` instead of `<img>`. It automatically converts to WebP/AVIF, generates responsive `srcSet`, lazy loads by default, and prevents CLS by reserving space. Add `priority` for above-the-fold images to preload them.

---

## 23. Quick Revision Cheat Sheet

```
ROUTING
├── app/page.tsx          → page at that URL
├── app/layout.tsx        → persistent wrapper
├── app/loading.tsx       → Suspense fallback
├── app/error.tsx         → error boundary ('use client' required)
├── app/not-found.tsx     → 404 page
├── app/route.ts          → API endpoint
├── [slug]                → dynamic segment
├── [...slug]             → catch-all
├── [[...slug]]           → optional catch-all
└── (group)               → route group (no URL segment)

RENDERING
├── SSG    → build-time, cache: 'force-cache'
├── SSR    → per-request, cache: 'no-store' or dynamic = 'force-dynamic'
├── ISR    → static + revalidate: N seconds
└── CSR    → 'use client' + useEffect fetch

COMPONENTS
├── Server Component (default) — async, no useState/useEffect, zero JS to client
└── Client Component ('use client') — stateful, interactive, event handlers

DATA FETCHING (App Router)
├── fetch(url, { cache: 'force-cache' })      → SSG
├── fetch(url, { cache: 'no-store' })         → SSR
├── fetch(url, { next: { revalidate: 60 } })  → ISR
└── fetch(url, { next: { tags: ['tag'] } })   → tag-based invalidation

CACHE INVALIDATION
├── revalidatePath('/path')   → invalidate full route cache
├── revalidateTag('tag')      → invalidate tagged fetches
└── router.refresh()          → invalidate router cache client-side

SPECIAL EXPORTS (App Router pages)
├── export const dynamic = 'force-dynamic' | 'force-static' | 'auto'
├── export const revalidate = 60
├── export const runtime = 'edge' | 'nodejs'
└── export async function generateStaticParams() { ... }

SERVER ACTIONS
├── 'use server' at top of file or function
├── Called from forms (action={fn}) or event handlers
├── Can call revalidatePath, revalidateTag, redirect
└── Progressive enhancement — works without JS

MIDDLEWARE
├── middleware.ts at project root
├── Runs on Edge (no Node.js APIs)
├── Use for auth, redirects, rewrites
└── export const config = { matcher: [...] }

IMAGE & FONT
├── import Image from 'next/image' — auto WebP, lazy load, CLS prevention
├── priority → preload above-the-fold images
└── import { Inter } from 'next/font/google' — self-hosted, zero layout shift

ENVIRONMENT VARIABLES
├── MY_SECRET        → server only
└── NEXT_PUBLIC_VAR  → server + client (visible in bundle!)
```

---

> **Final interview tip:** Next.js interviews often test **why** you'd choose one approach over another. Frame answers around trade-offs: SSG vs SSR vs ISR (performance vs freshness), Server vs Client Components (bundle size vs interactivity), Middleware vs Route Handlers (routing vs data handling).

---

*Last Updated: 2026-06-18*
