# HTML Interview Preparation Guide

> A guide covering core HTML concepts with code examples and interview tips for junior developers.

---

## Table of Contents

1. [What is HTML?](#1-what-is-html)
2. [HTML Document Structure](#2-html-document-structure)
3. [HTML Elements & Tags](#3-html-elements--tags)
4. [Block vs Inline Elements](#4-block-vs-inline-elements)
5. [Semantic HTML](#5-semantic-html)
6. [Forms & Input Elements](#6-forms--input-elements)
7. [Tables](#7-tables)
8. [Lists](#8-lists)
9. [Links & Anchors](#9-links--anchors)
10. [Images & Media](#10-images--media)
11. [HTML Attributes](#11-html-attributes)
12. [Meta Tags & SEO](#12-meta-tags--seo)
13. [HTML5 New Features](#13-html5-new-features)
14. [Canvas & SVG](#14-canvas--svg)
15. [Audio & Video](#15-audio--video)
16. [Iframe & Embedding](#16-iframe--embedding)
17. [Accessibility (a11y)](#17-accessibility-a11y)
18. [Data Attributes](#18-data-attributes)
19. [HTML APIs (Drag & Drop, Geolocation, Web Storage)](#19-html-apis)
20. [Character Encoding & Entities](#20-character-encoding--entities)
21. [Performance & Best Practices](#21-performance--best-practices)
22. [Common Interview Questions](#22-common-interview-questions)
23. [Quick Revision Cheat Sheet](#23-quick-revision-cheat-sheet)

---

## 1. What is HTML?

**HTML (HyperText Markup Language)** is the standard language for creating web pages. It describes the **structure and content** of a web page using elements represented by tags.

**Analogy:** HTML is the **skeleton** of a building — it gives structure. CSS is the skin/clothing (appearance), JavaScript is the muscles (behaviour).

**Key facts:**
- HTML is **not a programming language** — it is a markup language (no logic, loops, or variables)
- Current standard: **HTML5** (released 2014, continuously updated)
- Maintained by **W3C** and **WHATWG**
- Rendered by the browser's **rendering engine** (e.g., Blink in Chrome, Gecko in Firefox)

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>My Page</title>
  </head>
  <body>
    <h1>Hello, World!</h1>
  </body>
</html>
```

> **Interview tip:** "HTML is a markup language, not a programming language" is a classic gotcha.

---

## 2. HTML Document Structure

```html
<!DOCTYPE html>                     <!-- Tells browser: use HTML5 -->
<html lang="en">                    <!-- Root element -->
  <head>                            <!-- Metadata — NOT visible to the user -->
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Page Title</title>
    <link rel="stylesheet" href="style.css" />
    <script src="app.js" defer></script>
  </head>
  <body>                            <!-- Visible page content -->
    <header>...</header>
    <main>...</main>
    <footer>...</footer>
  </body>
</html>
```

**`<!DOCTYPE html>`** — not a tag, it's a declaration that prevents the browser from switching to **quirks mode** (legacy rendering). HTML5 simplified the old verbose HTML4 doctype to just `<!DOCTYPE html>`.

| `<head>` | `<body>` |
|----------|----------|
| Not rendered | Rendered on screen |
| `<title>`, `<meta>`, `<link>`, `<script>` | `<div>`, `<p>`, `<img>`, `<form>` |

> **Interview tip:** Always put `<meta charset="UTF-8">` as the **first** tag inside `<head>`.

---

## 3. HTML Elements & Tags

An **element** is everything from the opening tag to the closing tag, including content.

**Void elements** (self-closing — no closing tag):
```html
<br />  <hr />  <img />  <input />  <link />  <meta />
```

**Common elements:**

| Element | Purpose |
|---------|---------|
| `<h1>`–`<h6>` | Headings |
| `<p>` | Paragraph |
| `<div>` | Generic block container |
| `<span>` | Generic inline container |
| `<a>` | Hyperlink |
| `<img>` | Image |
| `<ul>`, `<ol>`, `<li>` | Lists |
| `<table>`, `<tr>`, `<td>` | Tables |
| `<form>`, `<input>`, `<button>` | Forms |

> **Interview tip:** `<div>` is block-level; `<span>` is inline. Neither has semantic meaning.

---

## 4. Block vs Inline Elements

**Block-level elements:** start on a new line, take full available width. Examples: `<div>`, `<p>`, `<h1>`–`<h6>`, `<ul>`, `<table>`, `<header>`, `<section>`

**Inline elements:** flow with text, take only as much width as needed. Examples: `<span>`, `<a>`, `<strong>`, `<em>`, `<img>`, `<input>`, `<button>`

```html
<!-- Block elements stack vertically -->
<p>First paragraph</p>
<p>Second paragraph</p>

<!-- Inline elements flow horizontally -->
<p>This is <strong>bold</strong> and this is <em>italic</em> text.</p>
```

**Inline-block:** flows like inline but respects `width`/`height` — set via CSS `display: inline-block`.

> **Interview tip:** In HTML5, `<a>` can wrap block elements as a special exception to the rule that block elements cannot go inside inline elements.

---

## 5. Semantic HTML

**Semantic HTML** uses elements that carry **meaning** about their content, not just how it looks.

**Why it matters:** accessibility (screen readers navigate by semantic tags), SEO (search engines weight content differently), and readability.

```html
<!-- Non-semantic -->
<div id="header"><div class="nav">...</div></div>
<div id="main"><div class="article">...</div></div>

<!-- Semantic -->
<header><nav>...</nav></header>
<main><article>...</article><aside>...</aside></main>
<footer>...</footer>
```

**HTML5 Semantic Elements:**

| Element | Meaning |
|---------|---------|
| `<header>` | Intro content / navigation for a page or section |
| `<nav>` | Primary navigation links |
| `<main>` | Main content (only one per page) |
| `<article>` | Self-contained, independently distributable content |
| `<section>` | Thematic grouping with a heading |
| `<aside>` | Sidebar / tangentially related content |
| `<footer>` | Footer for a page or section |
| `<figure>` / `<figcaption>` | Media with caption |
| `<time>` | Specific time or date |
| `<mark>` | Highlighted/relevant text |
| `<details>` / `<summary>` | Collapsible widget |

> **Interview tip:** `<article>` is for content that makes sense **on its own**. `<section>` is for **grouping related content** that is part of a larger whole.

---

## 6. Forms & Input Elements

Forms are the primary way users **submit data** to a server.

```html
<form action="/submit" method="POST" enctype="multipart/form-data">
  <label for="username">Username:</label>
  <input type="text" id="username" name="username" required placeholder="Enter username" />
  <button type="submit">Submit</button>
</form>
```

**Common `<input>` types:**

| Type | Use Case |
|------|---------|
| `text`, `password`, `email`, `number`, `tel`, `url` | Text inputs with optional built-in validation |
| `checkbox` | Boolean on/off |
| `radio` | One-of-many (group by same `name`) |
| `date`, `time`, `datetime-local`, `color` | Pickers |
| `file` | File upload |
| `hidden` | Sent with form but not shown |
| `submit`, `reset` | Form action buttons |

```html
<!-- Select dropdown -->
<select name="country">
  <option value="us">United States</option>
</select>

<!-- Multi-line text -->
<textarea name="message" rows="4"></textarea>
```

**Form validation attributes:** `required`, `minlength`, `maxlength`, `min`, `max`, `pattern`

**GET vs POST:**

| GET | POST |
|-----|------|
| Data in URL | Data in request body |
| Bookmarkable | Not bookmarkable |
| ~2000 char limit | No practical size limit |
| Use for search/filter | Use for sensitive/large data |

> **Interview tip:** Always link `<label for="id">` to `<input id="id">` — it makes the label clickable and is essential for accessibility.

---

## 7. Tables

Tables are for **tabular data only** — not for page layout.

```html
<table>
  <caption>Monthly Sales</caption>
  <thead>
    <tr><th scope="col">Month</th><th scope="col">Revenue</th></tr>
  </thead>
  <tbody>
    <tr><td>January</td><td>$10,000</td></tr>
  </tbody>
  <tfoot>
    <tr><td>Total</td><td>$10,000</td></tr>
  </tfoot>
</table>
```

**Spanning cells:** `colspan="2"` (merge columns), `rowspan="3"` (merge rows).

> **Interview tip:** Using tables for layout is bad practice — it breaks responsive design and accessibility. Use CSS Flexbox or Grid instead.

---

## 8. Lists

```html
<ul><li>Apples</li><li>Bananas</li></ul>           <!-- Unordered (bullets) -->

<ol type="1" start="3"><li>Step</li></ol>          <!-- Ordered (numbered) -->

<dl>                                               <!-- Description list -->
  <dt>HTML</dt>
  <dd>HyperText Markup Language</dd>
</dl>
```

Lists can be nested by placing a `<ul>` or `<ol>` inside an `<li>`.

---

## 9. Links & Anchors

```html
<!-- External link -->
<a href="https://example.com" target="_blank" rel="noopener noreferrer">Visit</a>

<!-- Same-page anchor -->
<a href="#section2">Go to Section 2</a>
<h2 id="section2">Section 2</h2>

<!-- Email / phone -->
<a href="mailto:hello@example.com">Email</a>
<a href="tel:+919876543210">Call</a>

<!-- Download -->
<a href="/report.pdf" download="Annual_Report.pdf">Download</a>
```

| `target` value | Behaviour |
|----------------|-----------|
| `_self` | Same tab (default) |
| `_blank` | New tab/window |

> **Interview tip:** Always add `rel="noopener noreferrer"` with `target="_blank"`. Without it, the new page can access `window.opener` — a **security vulnerability** (reverse tabnapping).

---

## 10. Images & Media

```html
<!-- Basic image — alt is REQUIRED -->
<img src="photo.jpg" alt="A mountain at sunset" width="800" height="600" />

<!-- Responsive image with srcset -->
<img
  src="photo-800.jpg"
  srcset="photo-400.jpg 400w, photo-800.jpg 800w"
  sizes="(max-width: 600px) 400px, 800px"
  alt="Responsive photo"
/>

<!-- Art direction with <picture> -->
<picture>
  <source media="(max-width: 600px)" srcset="small.jpg" />
  <img src="large.jpg" alt="Fallback" />
</picture>

<!-- Lazy loading -->
<img src="photo.jpg" alt="..." loading="lazy" />
```

**`alt` rules:** describe informative images; use `alt=""` for purely decorative images; missing alt causes screen readers to read the filename.

---

## 11. HTML Attributes

**Global attributes** (work on any element):

| Attribute | Purpose |
|-----------|---------|
| `id` | Unique identifier per page |
| `class` | CSS class (reusable, space-separated) |
| `style` | Inline CSS |
| `title` | Tooltip text |
| `hidden` | Hides element (like `display:none`) |
| `tabindex` | Tab focus order (`0`=natural, `-1`=programmatic only) |
| `contenteditable` | Makes element editable |

> **Interview tip:** `id` must be **unique** per page. Using the same `id` on multiple elements is invalid and causes unpredictable CSS/JS behaviour.

---

## 12. Meta Tags & SEO

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="description" content="Page description for SEO." />
  <meta name="robots" content="index, follow" />

  <!-- Open Graph (social sharing) -->
  <meta property="og:title" content="HTML Interview Guide" />
  <meta property="og:image" content="https://example.com/preview.jpg" />
  <meta property="og:url" content="https://example.com/html-guide" />

  <link rel="canonical" href="https://example.com/html-guide" />
  <link rel="icon" href="/favicon.ico" />
</head>
```

**Viewport meta tag:** `width=device-width` sets the viewport to device width; `initial-scale=1.0` prevents mobile browsers from zooming out. Without it, mobile browsers render the page at ~980px wide and shrink it — making text tiny.

> **Interview tip:** The viewport meta tag is **mandatory** for responsive websites.

---

## 13. HTML5 New Features

**New semantic elements:** `<header>`, `<footer>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<figure>`, `<time>`, `<mark>`, `<details>`

**New form input types:** `email`, `url`, `tel`, `number`, `range`, `date`, `time`, `color`, `search`

**New form attributes:** `placeholder`, `required`, `autofocus`, `pattern`, `min`, `max`

**New media elements:** `<audio>`, `<video>`, `<canvas>`, `<svg>`

**New JS APIs:** Local/Session Storage, Geolocation, Drag & Drop, Canvas, WebSockets, Web Workers, History API

**Removed:** `<font>`, `<center>`, `<big>`, `<strike>`, `<frameset>` — use CSS instead.

> **Interview tip:** HTML5 simplified `<!DOCTYPE html>` from the verbose HTML4 version that required a URL reference.

---

## 14. Canvas & SVG

| Feature | Canvas | SVG |
|---------|--------|-----|
| Type | Raster (pixels) | Vector (shapes) |
| Scalability | Loses quality on zoom | Perfect at any size |
| DOM interaction | No | Yes (each shape is a DOM node) |
| Events | Manual (coordinate math) | Native event listeners |
| Accessibility | Poor | Good |
| Best for | Games, image editing | Icons, charts, illustrations |

```html
<!-- Canvas — drawn via JavaScript -->
<canvas id="myCanvas" width="400" height="200"></canvas>
<script>
  const ctx = document.getElementById('myCanvas').getContext('2d');
  ctx.fillStyle = '#3498db';
  ctx.fillRect(10, 10, 150, 100);
</script>

<!-- SVG — declarative, part of the DOM -->
<svg width="200" height="200" xmlns="http://www.w3.org/2000/svg">
  <circle cx="100" cy="100" r="50" fill="#3498db" />
  <text x="100" y="170" text-anchor="middle">SVG Text</text>
</svg>
```

---

## 15. Audio & Video

```html
<video width="640" height="360" controls muted autoplay poster="thumbnail.jpg">
  <source src="video.mp4" type="video/mp4" />
  <source src="video.webm" type="video/webm" />
  <track kind="subtitles" src="subtitles-en.vtt" srclang="en" label="English" />
  <p>Your browser does not support video. <a href="video.mp4">Download it</a>.</p>
</video>

<audio controls>
  <source src="podcast.mp3" type="audio/mpeg" />
</audio>
```

**Key attributes:** `controls` (show UI), `autoplay` (requires `muted`), `loop`, `poster` (video thumbnail), `preload` (`none`/`metadata`/`auto`).

> **Interview tip:** Browsers block `autoplay` with sound. `autoplay` only works if `muted` is also set.

---

## 16. Iframe & Embedding

```html
<iframe
  src="https://www.youtube.com/embed/VIDEO_ID"
  width="560" height="315"
  title="YouTube video"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media"
  allowfullscreen
  loading="lazy"
></iframe>

<!-- Sandbox restricts iframe capabilities -->
<iframe src="untrusted.html" sandbox="allow-scripts allow-same-origin"></iframe>
```

**`sandbox` values:** `allow-scripts`, `allow-forms`, `allow-same-origin`, `allow-popups`. Empty `sandbox` is most restrictive.

**Security:** Never embed untrusted content without `sandbox`. Use `X-Frame-Options` header to prevent your own pages from being framed elsewhere (clickjacking defence).

---

## 17. Accessibility (a11y)

**Accessibility** ensures web content is usable by people with disabilities. WCAG levels: A (minimum), AA (standard), AAA (enhanced).

```html
<!-- Semantic HTML + ARIA label -->
<nav aria-label="Main navigation">
  <ul><li><a href="/">Home</a></li></ul>
</nav>

<!-- Label linked to input -->
<label for="email">Email</label>
<input type="email" id="email" required aria-describedby="email-hint" />
<span id="email-hint">We'll never share your email.</span>

<!-- Skip link for keyboard users -->
<a href="#main-content" class="skip-link">Skip to main content</a>
<main id="main-content">...</main>

<!-- Live region for dynamic content -->
<div aria-live="polite">Form submitted successfully!</div>
```

**Common ARIA attributes:**

| Attribute | Purpose |
|-----------|---------|
| `aria-label` | Names an element with no visible label |
| `aria-labelledby` | Points to element(s) that label this one |
| `aria-describedby` | Points to element(s) that describe this one |
| `aria-hidden="true"` | Hides from screen readers |
| `aria-expanded` | Indicates if a collapsible is open |
| `aria-live` | Announces dynamic content changes |
| `role` | Overrides or provides semantic role |

> **Interview tip:** First rule of ARIA — **don't use ARIA if a native HTML element does the job**. `<button>` beats `<div role="button">` because native elements have built-in keyboard and focus support.

---

## 18. Data Attributes

Data attributes store **custom data** on HTML elements.

```html
<div id="product" data-product-id="42" data-category="electronics" data-price="29.99">
  Wireless Headphones
</div>
```

```javascript
const card = document.getElementById('product');
console.log(card.dataset.productId);   // "42"  (kebab-case → camelCase)
console.log(card.dataset.category);    // "electronics"
card.dataset.price = "24.99";          // Modify
delete card.dataset.category;          // Delete
```

> **Interview tip:** Data attributes are always **strings**. Parse numbers/booleans explicitly: `parseInt(card.dataset.productId)` or `card.dataset.inStock === 'true'`.

---

## 19. HTML APIs

### Drag & Drop API

```html
<div draggable="true" id="item" ondragstart="dragStart(event)">Drag me</div>
<div id="zone" ondrop="drop(event)" ondragover="allowDrop(event)">Drop here</div>

<script>
  function dragStart(e) { e.dataTransfer.setData('text/plain', e.target.id); }
  function allowDrop(e) { e.preventDefault(); }
  function drop(e) {
    e.preventDefault();
    e.target.appendChild(document.getElementById(e.dataTransfer.getData('text/plain')));
  }
</script>
```

### Geolocation API

```javascript
if ('geolocation' in navigator) {
  navigator.geolocation.getCurrentPosition(
    pos => console.log(pos.coords.latitude, pos.coords.longitude),
    err => console.error(err.message)
  );
}
```

### Web Storage API

```javascript
localStorage.setItem('theme', 'dark');
const theme = localStorage.getItem('theme');  // "dark"
localStorage.removeItem('theme');

sessionStorage.setItem('formData', JSON.stringify({ name: 'Amith' }));
```

**localStorage vs sessionStorage vs cookies:**

| Feature | localStorage | sessionStorage | Cookies |
|---------|-------------|----------------|---------|
| Capacity | ~5–10 MB | ~5 MB | ~4 KB |
| Expiry | Never (manual) | Tab close | Set by expiry date |
| Sent with HTTP | No | No | Yes (automatically) |
| Across tabs | Yes | No | Yes |

---

## 20. Character Encoding & Entities

Always use **UTF-8**: `<meta charset="UTF-8" />` — goes first in `<head>`.

**HTML entities** escape characters that have special meaning in HTML:

| Character | Entity | Use |
|-----------|--------|-----|
| `<` | `&lt;` | Inside content |
| `>` | `&gt;` | Inside content |
| `&` | `&amp;` | Inside content |
| `"` | `&quot;` | Inside attributes |
| ` ` | `&nbsp;` | Non-breaking space |
| `©` | `&copy;` | Copyright |
| `®` | `&reg;` | Registered trademark |

```html
<!-- Wrong — browser misinterprets -->
<p>5 < 10 & 10 > 5</p>

<!-- Correct -->
<p>5 &lt; 10 &amp; 10 &gt; 5</p>
```

---

## 21. Performance & Best Practices

### Script Loading

```html
<!-- Blocks HTML parsing — avoid -->
<script src="app.js"></script>

<!-- defer: downloads in parallel, executes after HTML parsed, in order -->
<script src="app.js" defer></script>

<!-- async: downloads in parallel, executes immediately (order not guaranteed) -->
<script src="analytics.js" async></script>
```

Use `defer` for most scripts. Use `async` for independent scripts like analytics.

### Resource Hints & Best Practices

```html
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link rel="preload" href="hero-image.webp" as="image" />

<!-- Lazy load below-fold images -->
<img src="photo.jpg" alt="..." loading="lazy" />

<!-- Specify dimensions to prevent layout shift -->
<img src="photo.jpg" alt="..." width="800" height="600" />

<!-- Modern formats with fallback -->
<picture>
  <source srcset="image.webp" type="image/webp" />
  <img src="image.jpg" alt="..." />
</picture>
```

---

## 22. Common Interview Questions

### Q1: What is the difference between `<b>` and `<strong>`, `<i>` and `<em>`?

| Tag | Visual | Semantic |
|-----|--------|----------|
| `<b>` | Bold | None |
| `<strong>` | Bold | Important content |
| `<i>` | Italic | None |
| `<em>` | Italic | Emphasized content |

Use `<strong>` and `<em>` for meaning. Screen readers may read `<strong>` with emphasis; `<b>` is purely visual.

---

### Q2: What is the difference between `id` and `class`?

`id` must be **unique per page** — use for unique elements, URL fragments, and JS targeting. `class` can be **reused** on multiple elements — use for CSS styling groups.

---

### Q3: What are void elements? Give examples.

Elements with no children and no closing tag: `<br>`, `<hr>`, `<img>`, `<input>`, `<link>`, `<meta>`, `<area>`, `<base>`, `<col>`, `<embed>`, `<source>`, `<track>`.

---

### Q4: What is the difference between `localStorage`, `sessionStorage`, and cookies?

- **localStorage**: Persistent, JS-only, 5–10 MB, not sent with HTTP requests
- **sessionStorage**: Cleared on tab close, JS-only, 5 MB
- **Cookies**: Sent automatically with every HTTP request, 4 KB, configurable expiry

---

### Q5: What is the difference between `<script defer>` and `<script async>`?

`defer` downloads in parallel, executes **after** full HTML parsing, preserves script order. `async` downloads in parallel, executes **immediately** when ready — order not guaranteed. Use `defer` for most scripts, `async` for independent ones like analytics.

---

### Q6: What is `<!DOCTYPE html>` and why is it needed?

A declaration (not a tag) that tells the browser to use **HTML5 standards mode**. Without it, browsers enter **quirks mode** — mimicking old, buggy rendering to stay compatible with 1990s websites.

---

### Q7: What are data attributes? When would you use them?

`data-*` attributes store custom data on HTML elements. Use them to pass data from HTML to JavaScript without hidden inputs, store widget configuration, or track UI state.

---

### Q8: What is the purpose of the `alt` attribute on images?

1. **Accessibility** — screen readers read it for visually impaired users
2. **SEO** — search engines use it to understand image content
3. **Fallback** — displayed when the image fails to load

---

### Q9: What is the difference between `<section>`, `<article>`, and `<div>`?

`<article>` is for fully self-contained content (blog post, news story). `<section>` groups related content that is part of a larger whole. `<div>` is a generic container with no semantic meaning — use it when no semantic element fits.

---

### Q10: What is ARIA and when should you use it?

ARIA (Accessible Rich Internet Applications) adds accessibility semantics via `role`, `aria-label`, `aria-expanded`, etc. Use it **only when native HTML semantics are insufficient** — for custom widgets like tabs, accordions, and modals that have no native HTML equivalent.

---

### Q11: What are Web Workers?

Web Workers run **JavaScript in a background thread**, preventing heavy computations from blocking the UI.

```javascript
// main.js
const worker = new Worker('worker.js');
worker.postMessage({ data: bigArray });
worker.onmessage = e => console.log('Result:', e.data);

// worker.js
onmessage = e => postMessage(heavyComputation(e.data));
```

---

### Q12: What is the difference between `<link>` and `<a>`?

`<link>` goes in `<head>` and loads external resources (CSS, icons) — not clickable. `<a>` goes in `<body>` and creates clickable hyperlinks for users.

---

## 23. Quick Revision Cheat Sheet

```
DOCTYPE
  <!DOCTYPE html>         → HTML5 standard mode declaration

DOCUMENT STRUCTURE
  <html lang="en">        → Root element
  <head>                  → Metadata (not rendered)
  <body>                  → Page content (rendered)

SEMANTIC LAYOUT
  <header>                → Page/section header
  <nav>                   → Navigation links
  <main>                  → Main content (once per page)
  <article>               → Self-contained content
  <section>               → Thematic grouping
  <aside>                 → Sidebar / tangential content
  <footer>                → Page/section footer

DISPLAY TYPES
  Block: div, p, h1-h6, ul, ol, table, form, header, section
  Inline: span, a, strong, em, img, input, label, button

FORMS
  GET               → Data in URL, bookmarkable, max 2000 chars
  POST              → Data in body, no size limit
  <label for="id">  → Always link labels to inputs
  required / pattern / min / max / minlength → Built-in validation

IMAGES
  alt=""            → Required (empty for decorative images)
  loading="lazy"    → Defer off-screen images
  srcset            → Responsive images
  <picture>         → Art direction (different images per breakpoint)

LINKS
  target="_blank"   → Always add rel="noopener noreferrer"
  href="#id"        → Jump to same-page anchor
  download          → Force file download

SCRIPTS
  defer             → After HTML, in order     ← use for most scripts
  async             → Immediately, any order   ← use for analytics

STORAGE
  localStorage      → Persistent, JS-only, 5-10 MB
  sessionStorage    → Tab-scoped, JS-only, 5 MB
  cookies           → Sent with HTTP, 4 KB

META
  charset="UTF-8"              → Always first in <head>
  viewport content=...         → Required for responsive design
  description                  → SEO snippet text
  og:* / twitter:*             → Social sharing cards

ACCESSIBILITY
  Semantic HTML first          → Don't use ARIA if native element works
  alt on all <img>             → Empty alt="" for decorative
  <label for="id">             → Link all form labels
  tabindex="0"                 → Make custom elements focusable
  aria-label / aria-labelledby → Name unlabelled elements
  role="..."                   → Only when semantics are wrong/missing

HTML ENTITIES
  &lt;   <     &gt;   >
  &amp;  &     &quot; "
  &nbsp; (non-breaking space)
  &copy; ©     &reg;  ®
```

---

*Last updated: 2026-06-18 | Part of the Interview Preparation Documentation Repository*
