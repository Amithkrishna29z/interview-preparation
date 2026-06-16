# Next.js Interview Preparation Guide

> A comprehensive guide covering Next.js concepts from basics to advanced, with real-world analogies, code examples, and interview tips.

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
9. [API Routes](#9-api-routes)
10. [Route Handlers (App Router)](#10-route-handlers-app-router)
11. [Layouts & Templates](#11-layouts--templates)
12. [Middleware](#12-middleware)
13. [Next.js Image Optimization](#13-nextjs-image-optimization)
14. [Next.js Font Optimization](#14-nextjs-font-optimization)
15. [Metadata & SEO](#15-metadata--seo)
16. [Server Actions](#16-server-actions)
17. [Caching in Next.js](#17-caching-in-nextjs)
18. [Streaming & Suspense](#18-streaming--suspense)
19. [Error Handling](#19-error-handling)
20. [Loading UI & Skeleton Screens](#20-loading-ui--skeleton-screens)
21. [Environment Variables](#21-environment-variables)
22. [Authentication in Next.js](#22-authentication-in-nextjs)
23. [Internationalization (i18n)](#23-internationalization-i18n)
24. [Performance Optimization](#24-performance-optimization)
25. [Deployment & Build](#25-deployment--build)
26. [Common Interview Questions & Answers](#26-common-interview-questions--answers)
27. [Quick Revision Cheat Sheet](#27-quick-revision-cheat-sheet)

---

## 1. What is Next.js?

Next.js is a **React framework** built by Vercel that adds server-side capabilities, file-based routing, and production optimizations on top of React.

**Real-world analogy:** If React is the engine of a car, Next.js is the full vehicle — it adds the chassis, wheels, GPS (routing), fuel system (data fetching), and safety features (performance optimization) so you can drive immediately without assembling everything yourself.

**Key features:**
- **File-based routing** — folders and files define your URL structure
- **Multiple rendering modes** — SSR, SSG, ISR, CSR per page
- **Full-stack** — API routes / Route Handlers run on the server
- **Built-in optimizations** — image, font, script, and bundle optimization
- **Server Components** — components that run only on the server (App Router)

```bash
# Create a new Next.js app
npx create-next-app@latest my-app
cd my-app
npm run dev

# Project structure (App Router, TypeScript)
my-app/
├── app/
│   ├── layout.tsx       # Root layout
│   ├── page.tsx         # Home page (/)
│   ├── globals.css
│   └── api/
│       └── route.ts     # API Route Handler
├── public/              # Static assets
├── next.config.js
└── package.json
```

> **Interview tip:** Next.js is a **framework** (opinionated, full-stack), while React is a **library** (only UI). Next.js is what teams choose when they need SSR, SEO, or a backend alongside React.

---

## 2. Pages Router vs App Router

Next.js has two routing paradigms. The **App Router** (introduced in v13) is the modern standard; the **Pages Router** is legacy but still fully supported.

| Feature | Pages Router (`/pages`) | App Router (`/app`) |
|---|---|---|
| Introduced | Next.js v1 | Next.js v13 |
| Default exports | Required (`export default`) | Not required |
| Server Components | No | Yes (default) |
| Data fetching | `getStaticProps`, `getServerSideProps` | `async/await` in component |
| Layouts | Custom `_app.tsx` | Nested `layout.tsx` files |
| Loading states | Manual | `loading.tsx` |
| Error handling | `_error.tsx` | `error.tsx` |
| Streaming | No | Yes (React Suspense) |
| Route Handlers | `pages/api/*` | `app/api/*/route.ts` |

```
# Pages Router structure
pages/
├── index.tsx          → /
├── about.tsx          → /about
├── blog/
│   ├── index.tsx      → /blog
│   └── [slug].tsx     → /blog/:slug
└── api/
    └── users.ts       → /api/users

# App Router structure
app/
├── page.tsx           → /
├── about/
│   └── page.tsx       → /about
├── blog/
│   ├── page.tsx       → /blog
│   └── [slug]/
│       └── page.tsx   → /blog/:slug
└── api/
    └── users/
        └── route.ts   → /api/users
```

> **Interview tip:** New projects should use the **App Router**. If asked why, say: it enables React Server Components, co-located loading/error states, and nested layouts — all of which reduce boilerplate and improve performance.

---

## 3. File-Based Routing

Next.js derives your URL structure directly from the file system — no router configuration needed.

**Real-world analogy:** Your file system IS your sitemap. Folders are URL segments, `page.tsx` files are the pages at those segments.

```
# App Router — Special files per segment
app/
├── layout.tsx         # Shared UI wrapping all children
├── page.tsx           # The UI for this route (required to make it public)
├── loading.tsx        # Loading skeleton shown via Suspense
├── error.tsx          # Error boundary for this segment
├── not-found.tsx      # 404 for this segment
└── route.ts           # API endpoint (no page.tsx at same level)
```

**Route groups** — organize routes without affecting the URL:
```
app/
├── (marketing)/       # () = not part of URL
│   ├── page.tsx       → /
│   └── about/
│       └── page.tsx   → /about
└── (dashboard)/
    ├── layout.tsx      # Layout only for dashboard routes
    └── settings/
        └── page.tsx   → /settings
```

**Parallel routes** — render multiple pages in the same layout simultaneously:
```
app/
├── layout.tsx
├── @sidebar/
│   └── page.tsx       # Rendered in @sidebar slot
└── @main/
    └── page.tsx       # Rendered in @main slot
```

```tsx
// app/layout.tsx — consuming parallel route slots
export default function Layout({
  children,
  sidebar,
  main,
}: {
  children: React.ReactNode;
  sidebar: React.ReactNode;
  main: React.ReactNode;
}) {
  return (
    <div>
      {sidebar}
      {main}
      {children}
    </div>
  );
}
```

> **Interview tip:** Know the difference between **route groups** `(name)`, **dynamic segments** `[slug]`, **catch-all** `[...slug]`, and **optional catch-all** `[[...slug]]`.

---

## 4. Rendering Strategies — SSR, SSG, ISR, CSR

This is one of the most important Next.js interview topics.

**Real-world analogy:**
- **SSG** = Printing a book in advance (fast delivery, same for everyone)
- **SSR** = Printing the book on-demand when ordered (personalized, slower)
- **ISR** = Printing a book then reprinting it every night (fresh but cached)
- **CSR** = Giving someone a blank book and instructions to write it themselves (browser does the work)

### Static Site Generation (SSG)
HTML generated at **build time**. Fastest — served from CDN.

```tsx
// App Router: any async Server Component with no dynamic data = SSG by default
// app/blog/page.tsx
export default async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    cache: 'force-cache', // explicitly static (default)
  }).then(r => r.json());

  return <PostList posts={posts} />;
}
```

### Server-Side Rendering (SSR)
HTML generated on **every request**. Fresh data, but slower — cannot be CDN-cached.

```tsx
// App Router: opt-in per fetch or per segment
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic'; // entire page is dynamic

export default async function DashboardPage() {
  const data = await fetch('https://api.example.com/live', {
    cache: 'no-store', // SSR — bypass cache entirely
  }).then(r => r.json());

  return <Dashboard data={data} />;
}
```

### Incremental Static Regeneration (ISR)
HTML generated at build time, then **regenerated in the background** after a time interval.

```tsx
// App Router: revalidate option on fetch
export default async function ProductPage() {
  const product = await fetch('https://api.example.com/product/1', {
    next: { revalidate: 60 }, // re-generate every 60 seconds
  }).then(r => r.json());

  return <Product data={product} />;
}

// OR segment-level config
export const revalidate = 60;
```

### Client-Side Rendering (CSR)
Data fetched in the **browser** after the page loads. Used for user-specific or real-time data.

```tsx
'use client'; // must be a Client Component

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

| Strategy | When HTML is built | Data freshness | Best for |
|---|---|---|---|
| SSG | Build time | Stale until rebuild | Blog posts, docs, marketing |
| SSR | Every request | Always fresh | Dashboards, personalized pages |
| ISR | Build + background | Fresh within interval | E-commerce, news |
| CSR | In browser | Always fresh | Private, real-time, interactive UI |

---

## 5. Server Components vs Client Components

The App Router's biggest paradigm shift — components are **Server Components by default**.

**Real-world analogy:** Server Components are like a kitchen (you prepare food there but customers don't see inside). Client Components are the dining area — visible and interactive.

### Server Components
- Run **only on the server** — code never shipped to browser
- Can be `async` and `await` data directly
- No access to browser APIs, `useState`, `useEffect`, event handlers
- Smaller JavaScript bundle on client

```tsx
// app/posts/page.tsx — Server Component (default)
// No 'use client' directive needed

import { db } from '@/lib/db'; // Safe — never sent to browser

export default async function PostsPage() {
  // Direct database access — works because this runs on the server
  const posts = await db.query('SELECT * FROM posts');

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### Client Components
- Run on **server (for initial HTML) AND client (for interactivity)**
- Required for: `useState`, `useEffect`, event handlers, browser APIs, third-party UI libs

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
// app/page.tsx — Server Component
import InteractiveButton from './InteractiveButton'; // Client Component

export default async function Page() {
  const data = await fetchData(); // Server-only work

  return (
    <div>
      <h1>{data.title}</h1>
      <InteractiveButton label="Click me" /> {/* Client Component nested inside */}
    </div>
  );
}
```

> **Common mistake:** Importing a Server Component inside a Client Component breaks the server-only guarantee. Instead, pass Server Components as `children` props to Client Components.

```tsx
// WRONG — importing server component into client component
'use client';
import ServerComp from './ServerComp'; // breaks server-only guarantee

// CORRECT — pass as children
'use client';
export default function ClientShell({ children }) {
  return <div onClick={...}>{children}</div>;
}

// Parent (Server Component) passes server content as children
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

The App Router uses native `fetch` with extended options for caching and revalidation.

```tsx
// 1. Static fetch (SSG) — cached indefinitely
const data = await fetch('https://api.example.com/data', {
  cache: 'force-cache', // default behavior
});

// 2. Dynamic fetch (SSR) — never cached
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store',
});

// 3. Revalidated fetch (ISR) — cached, refreshed after N seconds
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 }, // 1 hour
});

// 4. Tag-based revalidation — revalidate on-demand
const data = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] },
});

// Trigger revalidation from a Server Action or Route Handler:
import { revalidateTag } from 'next/cache';
revalidateTag('posts'); // invalidates all fetches tagged 'posts'
```

**Parallel data fetching** — avoid request waterfalls:

```tsx
export default async function Page() {
  // WRONG — sequential (waterfall)
  const user = await fetchUser();
  const posts = await fetchPosts(user.id);

  // CORRECT — parallel
  const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);

  return <div>{/* ... */}</div>;
}
```

---

## 7. getStaticProps & getServerSideProps (Pages Router)

Legacy Pages Router data fetching functions — still valid but App Router is preferred.

```tsx
// pages/blog/[slug].tsx

// SSG — runs at build time
export async function getStaticProps({ params }) {
  const post = await fetchPost(params.slug);
  return {
    props: { post },
    revalidate: 60, // ISR — regenerate every 60s
  };
}

// Must pair with getStaticPaths for dynamic SSG routes
export async function getStaticPaths() {
  const slugs = await fetchAllSlugs();
  return {
    paths: slugs.map(slug => ({ params: { slug } })),
    fallback: 'blocking', // 'false' | true | 'blocking'
  };
}

// SSR — runs on every request
export async function getServerSideProps({ req, res, params, query }) {
  const data = await fetchData(params.slug);
  return { props: { data } };
}

export default function BlogPost({ post }) {
  return <article>{post.title}</article>;
}
```

**`fallback` values in `getStaticPaths`:**

| Value | Behavior |
|---|---|
| `false` | Unbuilt paths → 404 |
| `true` | Unbuilt paths → loading state, then generated |
| `'blocking'` | Unbuilt paths → SSR on first hit, then cached as static |

---

## 8. Dynamic Routes

Routes with URL parameters are defined with square brackets.

```
app/
├── blog/
│   └── [slug]/
│       └── page.tsx       → /blog/hello-world
├── shop/
│   └── [...categories]/
│       └── page.tsx       → /shop/a/b/c  (catch-all)
└── docs/
    └── [[...path]]/
        └── page.tsx       → /docs  OR  /docs/a/b  (optional catch-all)
```

```tsx
// app/blog/[slug]/page.tsx
interface Props {
  params: { slug: string };
  searchParams: { page?: string };
}

export default async function BlogPost({ params, searchParams }: Props) {
  const post = await fetchPost(params.slug);
  const page = searchParams.page ?? '1';

  return <article>{post.title}</article>;
}

// Generate static pages at build time (replaces getStaticPaths)
export async function generateStaticParams() {
  const posts = await fetchAllPosts();
  return posts.map(post => ({ slug: post.slug }));
}

// Dynamic metadata per route
export async function generateMetadata({ params }: Props) {
  const post = await fetchPost(params.slug);
  return { title: post.title, description: post.excerpt };
}
```

---

## 9. API Routes

Pages Router API routes — files in `pages/api/` become HTTP endpoints.

```tsx
// pages/api/users/[id].ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { id } = req.query;

  if (req.method === 'GET') {
    const user = await db.findUser(id as string);
    if (!user) return res.status(404).json({ error: 'Not found' });
    return res.status(200).json(user);
  }

  if (req.method === 'PUT') {
    const updated = await db.updateUser(id as string, req.body);
    return res.status(200).json(updated);
  }

  if (req.method === 'DELETE') {
    await db.deleteUser(id as string);
    return res.status(204).end();
  }

  res.setHeader('Allow', ['GET', 'PUT', 'DELETE']);
  res.status(405).end(`Method ${req.method} Not Allowed`);
}
```

---

## 10. Route Handlers (App Router)

Modern replacement for API Routes in the App Router — defined in `route.ts` files.

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const page = searchParams.get('page') ?? '1';

  const users = await db.getUsers({ page: parseInt(page) });
  return NextResponse.json(users);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const user = await db.createUser(body);
  return NextResponse.json(user, { status: 201 });
}

// app/api/users/[id]/route.ts — dynamic route handler
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const user = await db.findUser(params.id);
  if (!user) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json(user);
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  await db.deleteUser(params.id);
  return new NextResponse(null, { status: 204 });
}
```

> **Interview tip:** Route Handlers use Web APIs (`Request`, `Response`) unlike Pages Router API Routes which use Node.js-specific `req`/`res`. This makes them more portable and aligned with the Edge runtime.

---

## 11. Layouts & Templates

**Layouts** persist across route changes — state is preserved, they don't re-render. **Templates** create a new instance on each navigation.

```tsx
// app/layout.tsx — Root layout (required)
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'My App',
  description: 'My Next.js application',
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <nav>/* persistent nav — won't re-render on navigation */</nav>
        <main>{children}</main>
        <footer>/* persistent footer */</footer>
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx — Nested layout
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="dashboard">
      <aside>/* Sidebar — persists within /dashboard/* routes */</aside>
      <section>{children}</section>
    </div>
  );
}

// app/dashboard/template.tsx — Template (re-mounts on navigation)
export default function DashboardTemplate({ children }: { children: React.ReactNode }) {
  // This runs fresh on every navigation — useful for page transition animations
  return <div className="fade-in">{children}</div>;
}
```

---

## 12. Middleware

Middleware runs **before a request is processed** — on the Edge runtime. Used for auth, redirects, rewrites, A/B testing.

```tsx
// middleware.ts (root of project, not inside /app)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value;
  const { pathname } = request.nextUrl;

  // Redirect unauthenticated users away from protected routes
  if (pathname.startsWith('/dashboard') && !token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // Add custom headers to all responses
  const response = NextResponse.next();
  response.headers.set('X-Custom-Header', 'my-value');
  return response;
}

// Control which paths middleware runs on
export const config = {
  matcher: [
    '/dashboard/:path*',
    '/api/:path*',
    // Exclude static files and Next.js internals
    '/((?!_next/static|_next/image|favicon.ico).*)',
  ],
};
```

**Key middleware use cases:**

| Use Case | How |
|---|---|
| Authentication | Check cookies/headers, redirect to login |
| Geolocation | Read `request.geo`, serve localized content |
| A/B testing | Split traffic with cookies, rewrite to variant |
| Rate limiting | Track requests, return 429 when exceeded |
| Feature flags | Read flag from headers/cookies, rewrite route |

> **Interview tip:** Middleware runs on the **Edge runtime** — it cannot use Node.js APIs. It's extremely fast (no cold start) but limited. Don't put heavy database logic here; use it for routing decisions only.

---

## 13. Next.js Image Optimization

The `<Image>` component automatically optimizes images — lazy loading, WebP conversion, responsive sizing.

```tsx
import Image from 'next/image';

// Local image — dimensions inferred automatically
import profilePic from './profile.jpg';

export default function Avatar() {
  return (
    <Image
      src={profilePic}
      alt="Profile picture"
      // width and height inferred from import
      placeholder="blur" // shows blurred placeholder while loading
    />
  );
}

// Remote image — must specify width and height
export default function ProductImage({ url }: { url: string }) {
  return (
    <Image
      src={url}
      alt="Product"
      width={800}
      height={600}
      sizes="(max-width: 768px) 100vw, 50vw" // responsive sizes hint
      priority // load eagerly (for above-the-fold images)
    />
  );
}
```

```js
// next.config.js — allow remote image domains
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
        pathname: '/images/**',
      },
    ],
  },
};
```

**Benefits over `<img>`:**
- Automatic WebP/AVIF conversion
- Lazy loading by default
- Prevents Cumulative Layout Shift (CLS) — reserves space
- Automatic `srcSet` for responsive images
- Served via Next.js Image Optimization API

---

## 14. Next.js Font Optimization

`next/font` automatically self-hosts fonts — zero layout shift, no external network requests.

```tsx
// app/layout.tsx
import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-inter', // expose as CSS variable
});

const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  variable: '--font-roboto-mono',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${robotoMono.variable}`}>
      <body className={inter.className}>{children}</body>
    </html>
  );
}

// Local fonts
import localFont from 'next/font/local';

const myFont = localFont({
  src: './fonts/MyFont.woff2',
  variable: '--font-my-font',
});
```

---

## 15. Metadata & SEO

App Router uses a `Metadata` API — both static and dynamic.

```tsx
// Static metadata — app/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'My App',
  description: 'Welcome to my app',
  openGraph: {
    title: 'My App',
    description: 'Welcome to my app',
    images: [{ url: 'https://example.com/og.png' }],
  },
  twitter: {
    card: 'summary_large_image',
  },
  robots: {
    index: true,
    follow: true,
  },
};

// Dynamic metadata — app/blog/[slug]/page.tsx
export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const post = await fetchPost(params.slug);

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      images: [{ url: post.coverImage }],
    },
  };
}

// Metadata template — app/layout.tsx
export const metadata: Metadata = {
  title: {
    template: '%s | My App', // child pages: "Post Title | My App"
    default: 'My App',       // fallback when no title set
  },
};
```

---

## 16. Server Actions

Functions that run **on the server**, triggered from Client Components — no manual API route needed.

```tsx
// app/actions.ts
'use server'; // marks all exports as Server Actions

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const body = formData.get('body') as string;

  // Input validation
  if (!title || !body) throw new Error('Title and body are required');

  const post = await db.posts.create({ title, body });

  revalidatePath('/blog'); // invalidate cached /blog page
  redirect(`/blog/${post.slug}`); // redirect to new post
}

export async function deletePost(id: string) {
  await db.posts.delete(id);
  revalidatePath('/blog');
}
```

```tsx
// Using Server Action in a form — no JavaScript required for submission
// app/blog/new/page.tsx
import { createPost } from '@/app/actions';

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" required />
      <textarea name="body" placeholder="Body" required />
      <button type="submit">Create Post</button>
    </form>
  );
}

// Using Server Action from a Client Component
'use client';
import { deletePost } from '@/app/actions';

export default function DeleteButton({ postId }: { postId: string }) {
  return (
    <button onClick={() => deletePost(postId)}>
      Delete
    </button>
  );
}
```

> **Interview tip:** Server Actions are POST requests under the hood. They progressively enhance forms — the form still works without JavaScript. They replace the need for `/api` routes in many CRUD scenarios.

---

## 17. Caching in Next.js

Next.js has **four caching layers** — understanding all four is a senior-level topic.

| Cache | What it stores | Default | Invalidation |
|---|---|---|---|
| Request Memoization | `fetch` dedup within a single render | Automatic | Per request |
| Data Cache | `fetch` results across requests | Persistent | `revalidate`, `revalidateTag`, `revalidatePath` |
| Full Route Cache | Static HTML + RSC payload | Persistent | Redeploy or revalidation |
| Router Cache | RSC payload on client | Session | Navigate, `router.refresh()` |

```tsx
// Request Memoization — same fetch() called in multiple components
// returns same result without duplicate network requests
async function getUser(id: string) {
  // This fetch is automatically deduplicated within a single request
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

// Data Cache — control with cache / revalidate options
fetch(url, { cache: 'force-cache' });        // store in Data Cache indefinitely
fetch(url, { cache: 'no-store' });           // bypass Data Cache entirely
fetch(url, { next: { revalidate: 3600 } });  // revalidate after 1 hour

// On-demand revalidation
import { revalidatePath, revalidateTag } from 'next/cache';

revalidatePath('/blog');           // invalidate Full Route Cache for /blog
revalidateTag('posts');            // invalidate all fetches tagged 'posts'
```

---

## 18. Streaming & Suspense

Streaming lets Next.js send HTML to the browser in **chunks** — critical content first, slower content later.

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { DashboardSkeleton, RevenueChartSkeleton } from './skeletons';

// These components fetch their own data independently
import Summary from './Summary';       // fast (1 DB query)
import RevenueChart from './RevenueChart'; // slow (complex aggregation)
import RecentOrders from './RecentOrders'; // medium speed

export default function Dashboard() {
  return (
    <div>
      {/* Summary loads fast — streamed first */}
      <Suspense fallback={<DashboardSkeleton />}>
        <Summary />
      </Suspense>

      {/* RevenueChart is slow — shows skeleton until ready */}
      <Suspense fallback={<RevenueChartSkeleton />}>
        <RevenueChart />
      </Suspense>

      {/* RecentOrders streams independently */}
      <Suspense fallback={<p>Loading orders...</p>}>
        <RecentOrders />
      </Suspense>
    </div>
  );
}
```

**Without streaming:** Browser waits for the slowest component — TTFB is the total of all fetches.

**With streaming:** Browser receives fast components immediately, slow ones trickle in — better Time to First Byte and user experience.

---

## 19. Error Handling

Each route segment can have its own `error.tsx` — a React error boundary.

```tsx
// app/dashboard/error.tsx
'use client'; // error boundaries must be Client Components

import { useEffect } from 'react';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void; // retry the segment
}) {
  useEffect(() => {
    // Log error to monitoring service
    console.error(error);
  }, [error]);

  return (
    <div>
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// app/not-found.tsx — custom 404 page
export default function NotFound() {
  return (
    <div>
      <h2>Page Not Found</h2>
      <p>Could not find the requested resource.</p>
    </div>
  );
}

// Trigger 404 from a Server Component
import { notFound } from 'next/navigation';

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await fetchPost(params.slug);
  if (!post) notFound(); // renders the nearest not-found.tsx

  return <article>{post.title}</article>;
}
```

---

## 20. Loading UI & Skeleton Screens

`loading.tsx` automatically wraps the page in a `<Suspense>` boundary.

```tsx
// app/dashboard/loading.tsx — shown while page.tsx is loading
export default function DashboardLoading() {
  return (
    <div className="animate-pulse">
      <div className="h-8 bg-gray-200 rounded w-1/4 mb-4" />
      <div className="h-64 bg-gray-200 rounded mb-4" />
      <div className="h-4 bg-gray-200 rounded w-3/4" />
    </div>
  );
}
```

This is equivalent to wrapping `page.tsx` in `<Suspense fallback={<DashboardLoading />}>`, but automatic.

---

## 21. Environment Variables

```bash
# .env.local (never commit — git ignored)
DATABASE_URL=postgresql://...
JWT_SECRET=supersecret

# Available on server only (no NEXT_PUBLIC_ prefix)
API_SECRET_KEY=server-only-secret

# Available on client AND server (exposed to browser!)
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_ANALYTICS_ID=UA-123456
```

```tsx
// Server Component — access any env var
const db = new Database(process.env.DATABASE_URL);

// Client Component — only NEXT_PUBLIC_* vars
const apiUrl = process.env.NEXT_PUBLIC_API_URL; // works in browser
const secret = process.env.JWT_SECRET; // undefined in browser!
```

| Variable type | Available in Server | Available in Client |
|---|---|---|
| `MY_SECRET` | Yes | No |
| `NEXT_PUBLIC_MY_VAR` | Yes | Yes |

> **Interview tip:** Never prefix sensitive secrets with `NEXT_PUBLIC_` — they'll be embedded in the JavaScript bundle and visible to anyone.

---

## 22. Authentication in Next.js

**NextAuth.js (Auth.js)** is the most common auth solution for Next.js.

```bash
npm install next-auth
```

```tsx
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';
import Credentials from 'next-auth/providers/credentials';

const handler = NextAuth({
  providers: [
    GitHub({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
    Credentials({
      async authorize(credentials) {
        const user = await verifyUser(credentials.email, credentials.password);
        return user ?? null;
      },
    }),
  ],
  callbacks: {
    async session({ session, token }) {
      session.user.id = token.sub!;
      return session;
    },
  },
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

## 23. Internationalization (i18n)

```js
// next.config.js — Pages Router i18n
const nextConfig = {
  i18n: {
    locales: ['en', 'fr', 'de'],
    defaultLocale: 'en',
  },
};

// pages/index.tsx — access locale
export default function Home({ locale }) { ... }

export async function getStaticProps({ locale }) {
  return {
    props: { locale, messages: await loadMessages(locale) },
  };
}
```

For App Router, `next-intl` is the most popular library:

```tsx
// middleware.ts — detect locale and rewrite
import createMiddleware from 'next-intl/middleware';

export default createMiddleware({
  locales: ['en', 'fr', 'de'],
  defaultLocale: 'en',
});

// app/[locale]/layout.tsx
import { NextIntlClientProvider } from 'next-intl';

export default async function LocaleLayout({ children, params: { locale } }) {
  const messages = await getMessages();
  return (
    <NextIntlClientProvider locale={locale} messages={messages}>
      {children}
    </NextIntlClientProvider>
  );
}
```

---

## 24. Performance Optimization

### Script Optimization

```tsx
import Script from 'next/script';

export default function Page() {
  return (
    <>
      {/* afterInteractive — loads after page is interactive (default) */}
      <Script src="https://analytics.example.com/script.js" strategy="afterInteractive" />

      {/* lazyOnload — loads during browser idle time */}
      <Script src="https://ads.example.com/script.js" strategy="lazyOnload" />

      {/* beforeInteractive — loads before hydration (use sparingly) */}
      <Script src="https://polyfills.example.com/script.js" strategy="beforeInteractive" />
    </>
  );
}
```

### Bundle Analysis

```bash
npm install @next/bundle-analyzer
```

```js
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});
module.exports = withBundleAnalyzer({});

// Run: ANALYZE=true npm run build
```

### Dynamic Imports

```tsx
import dynamic from 'next/dynamic';

// Lazy load heavy components
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <p>Loading chart...</p>,
  ssr: false, // skip server-side rendering for this component
});

// Only loads when rendered
export default function Dashboard() {
  return <HeavyChart />;
}
```

### Core Web Vitals Checklist

| Metric | Target | Next.js Feature |
|---|---|---|
| LCP (Largest Contentful Paint) | < 2.5s | `<Image priority>`, font optimization |
| CLS (Cumulative Layout Shift) | < 0.1 | `<Image>` with dimensions, `next/font` |
| FID / INP (Interaction to Next Paint) | < 200ms | Server Components, reduce JS bundle |
| TTFB (Time to First Byte) | < 800ms | SSG, ISR, CDN caching |

---

## 25. Deployment & Build

```bash
# Build for production
npm run build

# Start production server (requires Node.js)
npm start

# Output build info
.next/
├── server/          # Server-side code
├── static/          # Static assets (hashed filenames)
└── cache/           # Build cache
```

```js
// next.config.js — static export (no Node.js server needed)
const nextConfig = {
  output: 'export', // generates /out directory
};
```

| Deployment target | Config | Use case |
|---|---|---|
| Vercel | None (auto-detected) | Best integration, edge functions |
| Docker / Node.js | `output: 'standalone'` | Self-hosted, custom infra |
| Static hosting (Netlify/S3) | `output: 'export'` | No SSR needed |
| AWS Lambda | Use `@sst/next.js` or `open-next` | AWS-native deployment |

```dockerfile
# Dockerfile for Next.js standalone output
FROM node:20-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

```js
// next.config.js — enable standalone for Docker
const nextConfig = {
  output: 'standalone',
};
```

---

## 26. Common Interview Questions & Answers

**Q: What is the difference between `getStaticProps` and `getServerSideProps`?**

> `getStaticProps` runs at **build time** and generates static HTML — fastest delivery via CDN, best for content that doesn't change often. `getServerSideProps` runs on **every request** — always fresh data but slower, cannot be CDN cached. Use `getStaticProps` + ISR for most cases; reserve `getServerSideProps` for pages that need per-request data (e.g., user-specific dashboards).

---

**Q: What is the difference between `revalidate` in `getStaticProps` vs `fetch`?**

> In Pages Router, `revalidate: 60` in `getStaticProps` triggers ISR — the page is regenerated in the background after 60 seconds. In App Router, `next: { revalidate: 60 }` on a `fetch` call does the same at the data level. Both implement ISR but the mechanism differs — Pages Router is page-level, App Router is per-fetch.

---

**Q: When would you use a Client Component vs a Server Component?**

> Default to **Server Components** — they reduce the client bundle and can access databases/secrets directly. Switch to `'use client'` only when you need: `useState`, `useEffect`, event handlers (`onClick`, `onChange`), browser-only APIs, or third-party libraries that use those features. The pattern is: fetch data on the server, pass to client for interactivity.

---

**Q: How does Next.js handle hydration?**

> Next.js renders HTML on the server (initial paint) then "hydrates" it on the client — React attaches event listeners to the existing HTML without re-rendering. Server Components are never hydrated (they don't ship JavaScript). Client Components are hydrated. A hydration mismatch (server HTML ≠ client render) causes a warning and full client re-render.

---

**Q: What is the purpose of the `layout.tsx` file?**

> `layout.tsx` wraps all pages in its subtree with shared UI (nav, sidebar, footer). Unlike `template.tsx`, layouts **persist across navigation** — they don't unmount and remount. This preserves state (e.g., scroll position, open dropdowns) when the user navigates between child routes.

---

**Q: How does caching work in Next.js App Router?**

> Next.js has four caching layers: (1) **Request Memoization** deduplicates identical `fetch` calls within a single render tree. (2) **Data Cache** persists `fetch` responses across requests on the server. (3) **Full Route Cache** stores rendered HTML and RSC payload for static routes. (4) **Router Cache** stores RSC payloads in the browser between navigations. Each layer has different invalidation strategies.

---

**Q: What is a Server Action and when would you use one?**

> Server Actions are `async` functions marked with `'use server'` that run on the server but can be called from Client Components or HTML forms. They replace simple API routes for mutations — creating, updating, deleting data. Benefits: no manual `fetch` to an API route, works without JavaScript (progressive enhancement), and can call `revalidatePath`/`revalidateTag` to invalidate cache after mutation.

---

**Q: How does Middleware differ from API Routes?**

> Middleware runs **before** a request reaches any route — on the Edge runtime, globally across all routes. It's for routing decisions: auth checks, redirects, rewrites, A/B testing. API Routes / Route Handlers run **as** a route — they handle specific endpoints and can use Node.js APIs. Middleware cannot read from databases efficiently (Edge runtime limitations); Route Handlers can.

---

**Q: What is ISR and how does it work?**

> Incremental Static Regeneration generates static pages at build time, then re-generates them in the background after a `revalidate` interval. The first request after the interval triggers a background rebuild — the current user still gets the stale page (stale-while-revalidate). The next request gets the fresh page. This gives you SSG performance with near-fresh data.

---

**Q: How do you optimize images in Next.js?**

> Use the `<Image>` component from `next/image` instead of `<img>`. It automatically: converts to WebP/AVIF, generates responsive `srcSet`, lazy loads by default, prevents CLS by reserving space, and serves via a built-in optimization API. For above-the-fold images, add `priority` to preload them.

---

## 27. Quick Revision Cheat Sheet

```
ROUTING
├── app/page.tsx          → page at that URL
├── app/layout.tsx        → persistent wrapper
├── app/loading.tsx       → Suspense fallback
├── app/error.tsx         → error boundary (must be 'use client')
├── app/not-found.tsx     → 404 page
├── app/route.ts          → API endpoint
├── [slug]                → dynamic segment
├── [...slug]             → catch-all
├── [[...slug]]           → optional catch-all
└── (group)               → route group (no URL segment)

RENDERING
├── SSG    → static, build-time, cache: 'force-cache'
├── SSR    → per-request, cache: 'no-store' or dynamic = 'force-dynamic'
├── ISR    → static + revalidate: N seconds
└── CSR    → 'use client' + useEffect fetch

COMPONENTS
├── Server Component (default) — async, no useState/useEffect, zero JS to client
└── Client Component ('use client') — stateful, interactive, eventhandlers

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

IMAGE OPTIMIZATION
├── import Image from 'next/image'
├── priority → preload (above the fold)
├── placeholder="blur" → blur-up effect
└── sizes → responsive image hints

ENVIRONMENT VARIABLES
├── MY_SECRET        → server only
└── NEXT_PUBLIC_VAR  → server + client (visible in bundle!)
```

---

> **Final interview tip:** Next.js interviews often test **why** you'd choose one approach over another, not just **how** to use it. Frame your answers around trade-offs: SSG vs SSR vs ISR (performance vs freshness), Server vs Client Components (bundle size vs interactivity), Middleware vs API Routes (routing vs data handling).
