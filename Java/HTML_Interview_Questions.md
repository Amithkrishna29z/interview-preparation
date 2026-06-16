# HTML Interview Preparation Guide

> A comprehensive guide covering HTML concepts from basics to advanced, with real-world analogies, code examples, and interview tips.

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

**HTML (HyperText Markup Language)** is the standard language for creating web pages. It describes the **structure and content** of a web page using a system of elements represented by tags.

**Real-world analogy:** Think of HTML as the **skeleton** of a building. Just like a skeleton gives shape and structure to a body, HTML gives structure to a web page. CSS is the skin/clothing (appearance), and JavaScript is the muscles (behaviour).

**Key facts:**
- HTML is **not a programming language** — it is a markup language
- Current standard: **HTML5** (released 2014, continuously updated)
- Maintained by: **W3C** (World Wide Web Consortium) and **WHATWG**
- Files use `.html` or `.htm` extension
- Rendered by the browser's **rendering engine** (e.g., Blink in Chrome, Gecko in Firefox)

```html
<!-- Simplest valid HTML5 document -->
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

> **Interview tip:** "HTML is a markup language, not a programming language" is a classic gotcha. It has no logic, loops, or variables — it only describes structure.

---

## 2. HTML Document Structure

Every HTML document follows a standard structure that the browser expects.

**Real-world analogy:** An HTML document is like a letter — it has a header section (meta info, subject) and a body section (the actual content).

```html
<!DOCTYPE html>                     <!-- Tells browser: use HTML5 -->
<html lang="en">                    <!-- Root element; lang helps screen readers & SEO -->
  <head>                            <!-- Metadata — NOT visible to the user -->
    <meta charset="UTF-8" />        <!-- Character encoding -->
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="Page description for SEO" />
    <title>Page Title</title>       <!-- Shown in browser tab -->
    <link rel="stylesheet" href="style.css" />   <!-- External CSS -->
    <script src="app.js" defer></script>          <!-- JS loaded after HTML parses -->
  </head>
  <body>                            <!-- Visible page content -->
    <header>...</header>
    <main>...</main>
    <footer>...</footer>
  </body>
</html>
```

**`<!DOCTYPE html>` explained:**
- Not an HTML tag — it is a **declaration** to the browser
- Prevents the browser from switching to **quirks mode** (legacy rendering)
- In HTML4, DOCTYPE was long and complex. HTML5 simplified it to just `<!DOCTYPE html>`

**`<head>` vs `<body>`:**

| `<head>` | `<body>` |
|----------|----------|
| Not rendered on screen | Rendered on screen |
| Contains metadata | Contains page content |
| `<title>`, `<meta>`, `<link>`, `<script>` | `<div>`, `<p>`, `<img>`, `<form>` |

> **Interview tip:** Always include `<meta charset="UTF-8">` as the **first** tag inside `<head>` so the browser knows encoding before rendering any text.

---

## 3. HTML Elements & Tags

An **element** is everything from the opening tag to the closing tag, including content.

```
<p class="intro">Hello World</p>
 ^                ^             ^
 Opening tag      Content       Closing tag
```

**Void elements (self-closing — no closing tag needed):**
```html
<br />      <!-- Line break -->
<hr />      <!-- Horizontal rule -->
<img />     <!-- Image -->
<input />   <!-- Form input -->
<link />    <!-- Link to external resource -->
<meta />    <!-- Metadata -->
```

**Nested elements:**
```html
<div>
  <p>This is a <strong>bold</strong> word inside a paragraph.</p>
</div>
```

**Common elements quick reference:**

| Element | Purpose |
|---------|---------|
| `<h1>`–`<h6>` | Headings (h1 = most important) |
| `<p>` | Paragraph |
| `<div>` | Generic block container |
| `<span>` | Generic inline container |
| `<a>` | Hyperlink/anchor |
| `<img>` | Image |
| `<ul>`, `<ol>`, `<li>` | Lists |
| `<table>`, `<tr>`, `<td>` | Tables |
| `<form>`, `<input>`, `<button>` | Forms |
| `<script>` | JavaScript |
| `<style>` | Inline CSS |

> **Interview tip:** The difference between `<div>` and `<span>` is purely about **display type** — `<div>` is block-level, `<span>` is inline. Neither has semantic meaning.

---

## 4. Block vs Inline Elements

**Real-world analogy:** Block elements are like **paragraphs in a book** — each starts on a new line and takes up the full width. Inline elements are like **words within a sentence** — they flow with the text and only take up as much space as needed.

**Block-level elements:**
- Always start on a **new line**
- Take up the **full available width**
- Can contain other block and inline elements
- Examples: `<div>`, `<p>`, `<h1>`–`<h6>`, `<ul>`, `<ol>`, `<li>`, `<table>`, `<form>`, `<header>`, `<section>`, `<article>`

**Inline elements:**
- Do **not** start on a new line
- Take up only as much **width as necessary**
- Should only contain data or other inline elements (not block elements)
- Examples: `<span>`, `<a>`, `<strong>`, `<em>`, `<img>`, `<input>`, `<label>`, `<button>`

```html
<!-- Block elements stack vertically -->
<p>First paragraph</p>
<p>Second paragraph</p>

<!-- Inline elements flow horizontally -->
<p>This is <strong>bold</strong> and this is <em>italic</em> text.</p>
```

**Inline-block:**
- Behaves like inline (flows with text) but respects `width`/`height`/`margin`/`padding`
- Set via CSS: `display: inline-block`

> **Interview tip:** Placing a block element inside an inline element (e.g., `<a><div>...</div></a>`) is invalid HTML — except in HTML5 where `<a>` can wrap block elements as a special case.

---

## 5. Semantic HTML

**Semantic HTML** means using HTML elements that carry **meaning** about the content they contain, rather than just defining how it looks.

**Real-world analogy:** Imagine a newspaper. It has a masthead (header), main story (article), sidebars (aside), and a footer. Semantic tags are like labels on each section — they tell everyone (browsers, screen readers, search engines) what each part is.

**Why semantics matter:**
- **Accessibility** — screen readers use semantic tags to navigate pages
- **SEO** — search engines weight content in semantic tags differently
- **Maintainability** — code is easier to read and understand
- **Browser defaults** — some semantic tags carry built-in styling/behaviour

**Semantic vs non-semantic:**

```html
<!-- ❌ Non-semantic — structure but no meaning -->
<div id="header">
  <div class="nav">...</div>
</div>
<div id="main">
  <div class="article">...</div>
  <div class="sidebar">...</div>
</div>
<div id="footer">...</div>

<!-- ✅ Semantic — self-describing structure -->
<header>
  <nav>...</nav>
</header>
<main>
  <article>...</article>
  <aside>...</aside>
</main>
<footer>...</footer>
```

**HTML5 Semantic Elements:**

| Element | Meaning |
|---------|---------|
| `<header>` | Introductory content / navigation for a page or section |
| `<nav>` | Primary navigation links |
| `<main>` | Main content of the page (only one per page) |
| `<article>` | Self-contained, independently distributable content (blog post, news article) |
| `<section>` | Thematic grouping of content with a heading |
| `<aside>` | Content tangentially related to surrounding content (sidebar, callout) |
| `<footer>` | Footer for a page or section |
| `<figure>` | Self-contained media content (image, diagram, code) |
| `<figcaption>` | Caption for a `<figure>` |
| `<time>` | A specific time or date |
| `<mark>` | Highlighted/relevant text |
| `<details>` / `<summary>` | Collapsible disclosure widget |
| `<address>` | Contact information for nearest `<article>` or `<body>` |

```html
<article>
  <header>
    <h2>How HTML5 Changed the Web</h2>
    <time datetime="2024-01-15">January 15, 2024</time>
  </header>
  <p>HTML5 introduced semantic elements that...</p>
  <figure>
    <img src="html5-logo.png" alt="HTML5 logo" />
    <figcaption>The official HTML5 logo</figcaption>
  </figure>
  <footer>
    <address>Written by <a href="mailto:author@example.com">Jane Doe</a></address>
  </footer>
</article>
```

> **Interview tip:** `<article>` vs `<section>` — `<article>` is for content that makes sense **on its own** (you could copy-paste it and it still makes sense). `<section>` is for **grouping related content** that is part of a larger whole.

---

## 6. Forms & Input Elements

Forms are the primary way users **submit data** to a server or trigger JavaScript actions.

**Real-world analogy:** An HTML form is like a paper application form — it has fields you fill in, and a submit button that sends it somewhere.

**Basic form structure:**
```html
<form action="/submit" method="POST" enctype="multipart/form-data">
  <!-- action: where to send data -->
  <!-- method: GET (visible in URL) or POST (hidden in body) -->
  <!-- enctype: needed when uploading files -->

  <label for="username">Username:</label>
  <input type="text" id="username" name="username" required placeholder="Enter username" />

  <button type="submit">Submit</button>
</form>
```

**All `<input>` types:**

| Type | Use Case |
|------|---------|
| `text` | Single-line text |
| `password` | Masked password input |
| `email` | Email (with built-in validation) |
| `number` | Numeric input |
| `tel` | Phone number |
| `url` | URL (with validation) |
| `search` | Search box |
| `checkbox` | Boolean on/off selection |
| `radio` | One-of-many selection |
| `range` | Slider |
| `date` | Date picker |
| `time` | Time picker |
| `datetime-local` | Date and time picker |
| `color` | Color picker |
| `file` | File upload |
| `hidden` | Hidden field (sent with form but not shown) |
| `submit` | Submit button |
| `reset` | Reset form button |
| `image` | Image as submit button |

```html
<!-- Radio buttons — same `name` groups them -->
<input type="radio" id="male" name="gender" value="male" />
<label for="male">Male</label>
<input type="radio" id="female" name="gender" value="female" />
<label for="female">Female</label>

<!-- Checkbox -->
<input type="checkbox" id="terms" name="terms" value="agreed" />
<label for="terms">I agree to the terms</label>

<!-- Select dropdown -->
<select name="country">
  <option value="">-- Choose country --</option>
  <option value="us">United States</option>
  <option value="in">India</option>
</select>

<!-- Multi-line text -->
<textarea name="message" rows="4" cols="50" placeholder="Your message..."></textarea>

<!-- Datalist — autocomplete suggestions -->
<input list="browsers" name="browser" />
<datalist id="browsers">
  <option value="Chrome">
  <option value="Firefox">
  <option value="Safari">
</datalist>
```

**Form validation attributes:**

```html
<input type="email" required />                   <!-- Must be filled -->
<input type="text" minlength="3" maxlength="20" /> <!-- Length constraints -->
<input type="number" min="1" max="100" step="5" /> <!-- Value constraints -->
<input type="text" pattern="[A-Za-z]{3,}" />       <!-- Regex pattern -->
```

**`GET` vs `POST`:**

| GET | POST |
|-----|------|
| Data in URL query string | Data in request body |
| Bookmarkable | Not bookmarkable |
| Max ~2000 chars | No practical size limit |
| Use for search/filter | Use for sensitive/large data |
| Cached by browser | Not cached |

> **Interview tip:** Use `<label for="id">` linking to `<input id="id">` — this makes the label clickable to focus the input, which is crucial for accessibility.

---

## 7. Tables

Tables should only be used for **tabular data** (data that has rows and columns), not for page layout.

```html
<table>
  <caption>Monthly Sales Report</caption>   <!-- Table title -->
  <thead>                                    <!-- Header rows -->
    <tr>
      <th scope="col">Month</th>
      <th scope="col">Revenue</th>
      <th scope="col">Units Sold</th>
    </tr>
  </thead>
  <tbody>                                    <!-- Data rows -->
    <tr>
      <td>January</td>
      <td>$10,000</td>
      <td>200</td>
    </tr>
    <tr>
      <td>February</td>
      <td>$12,500</td>
      <td>250</td>
    </tr>
  </tbody>
  <tfoot>                                    <!-- Summary row -->
    <tr>
      <td>Total</td>
      <td>$22,500</td>
      <td>450</td>
    </tr>
  </tfoot>
</table>
```

**Spanning cells:**
```html
<td colspan="2">Spans 2 columns</td>    <!-- Merge horizontally -->
<td rowspan="3">Spans 3 rows</td>       <!-- Merge vertically -->
```

> **Interview tip:** Using tables for layout (as was done in the 1990s) is considered **bad practice** today. Use CSS Flexbox or Grid instead. Tables break responsive design and accessibility.

---

## 8. Lists

HTML provides three types of lists:

```html
<!-- Unordered list (bullet points) -->
<ul>
  <li>Apples</li>
  <li>Bananas</li>
  <li>Oranges</li>
</ul>

<!-- Ordered list (numbered) -->
<ol type="1" start="3">   <!-- type: 1, A, a, I, i — start: begin numbering at 3 -->
  <li>First step</li>
  <li>Second step</li>
  <li>Third step</li>
</ol>

<!-- Description list (term + definition pairs) -->
<dl>
  <dt>HTML</dt>
  <dd>HyperText Markup Language — the structure of the web</dd>
  <dt>CSS</dt>
  <dd>Cascading Style Sheets — the styling of the web</dd>
</dl>

<!-- Nested lists -->
<ul>
  <li>Fruits
    <ul>
      <li>Apple</li>
      <li>Mango</li>
    </ul>
  </li>
  <li>Vegetables</li>
</ul>
```

---

## 9. Links & Anchors

The `<a>` (anchor) tag creates hyperlinks.

```html
<!-- External link — always use target="_blank" with rel="noopener noreferrer" -->
<a href="https://example.com" target="_blank" rel="noopener noreferrer">Visit Example</a>

<!-- Internal link (relative path) -->
<a href="/about.html">About Us</a>

<!-- Same-page anchor (jump to section) -->
<a href="#section2">Go to Section 2</a>
<h2 id="section2">Section 2</h2>

<!-- Email link -->
<a href="mailto:hello@example.com">Send Email</a>

<!-- Phone link (mobile-friendly) -->
<a href="tel:+919876543210">Call Us</a>

<!-- Download link -->
<a href="/report.pdf" download="Annual_Report_2024.pdf">Download Report</a>

<!-- Link with no href — used with JavaScript -->
<a href="#" onclick="doSomething()">Click Me</a>
```

**`target` attribute values:**

| Value | Behaviour |
|-------|-----------|
| `_self` | Opens in same tab (default) |
| `_blank` | Opens in new tab/window |
| `_parent` | Opens in parent frame |
| `_top` | Opens in full body of window |

> **Interview tip:** Always add `rel="noopener noreferrer"` when using `target="_blank"`. Without it, the new page can access the opener via `window.opener`, which is a **security vulnerability** (reverse tabnapping).

---

## 10. Images & Media

```html
<!-- Basic image — alt is REQUIRED for accessibility -->
<img src="photo.jpg" alt="A mountain landscape at sunset" width="800" height="600" />

<!-- Responsive image with srcset (different sizes for different screens) -->
<img
  src="photo-800.jpg"
  srcset="photo-400.jpg 400w, photo-800.jpg 800w, photo-1200.jpg 1200w"
  sizes="(max-width: 600px) 400px, (max-width: 1000px) 800px, 1200px"
  alt="Responsive photo"
/>

<!-- Picture element (art direction — different images for different contexts) -->
<picture>
  <source media="(max-width: 600px)" srcset="small.jpg" />
  <source media="(max-width: 1200px)" srcset="medium.jpg" />
  <img src="large.jpg" alt="Fallback image" />   <!-- Fallback -->
</picture>

<!-- Lazy loading (defer off-screen images) -->
<img src="photo.jpg" alt="..." loading="lazy" />
```

**`alt` attribute rules:**
- **Informative image**: describe what the image shows — `alt="Bar chart showing 2024 sales data"`
- **Decorative image**: use empty alt — `alt=""` (screen readers skip it)
- **Image with text**: include that text in alt — `alt="Submit"`
- **Missing alt**: screen readers read out the filename — bad UX

---

## 11. HTML Attributes

Attributes provide additional information about elements.

**Global attributes** (work on any HTML element):

| Attribute | Purpose |
|-----------|---------|
| `id` | Unique identifier (must be unique per page) |
| `class` | CSS class (space-separated, multiple allowed) |
| `style` | Inline CSS styles |
| `title` | Tooltip text on hover |
| `lang` | Language of element content |
| `dir` | Text direction: `ltr` or `rtl` |
| `hidden` | Hides element (like `display:none`) |
| `tabindex` | Tab key focus order (`0`=natural, `-1`=programmatic only, positive=explicit order) |
| `contenteditable` | Makes element editable by user |
| `draggable` | Makes element draggable |
| `spellcheck` | Enable/disable spell checking |

```html
<p id="intro" class="lead highlight" title="Introduction paragraph" lang="en">
  Welcome to the page.
</p>

<!-- tabindex usage -->
<div tabindex="0" role="button">Focusable Div</div>      <!-- Focusable via tab -->
<div tabindex="-1" id="modal">Focus programmatically</div><!-- Only via JS focus() -->
```

> **Interview tip:** `id` must be **unique** on a page. `class` can be reused. Using the same `id` on multiple elements is invalid HTML and causes unpredictable CSS/JS behaviour.

---

## 12. Meta Tags & SEO

`<meta>` tags provide metadata about the HTML document — mostly used for SEO and social sharing.

```html
<head>
  <!-- Essential meta tags -->
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="description" content="Learn HTML from basics to advanced. Interview prep guide." />
  <meta name="keywords" content="HTML, web development, interview" />   <!-- Mostly ignored now -->
  <meta name="author" content="Amith Krishnan" />
  <meta name="robots" content="index, follow" />   <!-- Tell search engines to index this page -->

  <!-- Open Graph (Facebook, LinkedIn sharing) -->
  <meta property="og:title" content="HTML Interview Guide" />
  <meta property="og:description" content="Complete HTML interview prep" />
  <meta property="og:image" content="https://example.com/preview.jpg" />
  <meta property="og:url" content="https://example.com/html-guide" />
  <meta property="og:type" content="website" />

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:title" content="HTML Interview Guide" />
  <meta name="twitter:image" content="https://example.com/preview.jpg" />

  <!-- Canonical URL (avoid duplicate content penalties) -->
  <link rel="canonical" href="https://example.com/html-guide" />

  <!-- Favicon -->
  <link rel="icon" href="/favicon.ico" type="image/x-icon" />
  <link rel="apple-touch-icon" href="/apple-icon.png" />
</head>
```

**`<meta name="viewport">`** — critical for responsive design:
- `width=device-width` — sets viewport width to device screen width
- `initial-scale=1.0` — prevents mobile browsers from zooming out

> **Interview tip:** Without the viewport meta tag, mobile browsers render the page as if it were a desktop (~980px wide) and zoom it out — making text tiny. This tag is **mandatory** for responsive websites.

---

## 13. HTML5 New Features

HTML5 (2008–2014) introduced major new features over HTML4.

**New structural/semantic elements:**
- `<header>`, `<footer>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<figure>`, `<figcaption>`, `<time>`, `<mark>`, `<details>`, `<summary>`

**New form input types:**
- `email`, `url`, `tel`, `number`, `range`, `date`, `time`, `color`, `search`, `datetime-local`

**New form attributes:**
- `placeholder`, `required`, `autofocus`, `autocomplete`, `novalidate`, `pattern`, `min`, `max`, `step`

**New media elements:**
- `<audio>`, `<video>`, `<source>`, `<track>`, `<canvas>`, `<svg>`

**New APIs (via JavaScript):**
- **Local Storage / Session Storage** — persistent client-side data storage
- **Geolocation API** — get user's location
- **Drag & Drop API** — native drag-and-drop
- **Canvas API** — 2D drawing via JavaScript
- **WebSockets** — full-duplex communication
- **Web Workers** — background JS threads
- **History API** — manipulate browser history without reload
- **File API** — read files from user's device

**Removed in HTML5:**
- `<font>`, `<center>`, `<big>`, `<strike>`, `<frameset>`, `<frame>`, `<noframes>` — all deprecated, use CSS instead

> **Interview tip:** HTML5 also changed the doctype to simply `<!DOCTYPE html>` — compared to the long HTML4 doctype that required a URL reference.

---

## 14. Canvas & SVG

Both are used for graphics on the web but work differently.

**Canvas:**
- Raster-based (pixel-by-pixel)
- Drawn via JavaScript
- Good for games, real-time data visualization, image manipulation
- Not accessible by default

```html
<canvas id="myCanvas" width="400" height="200"></canvas>

<script>
  const canvas = document.getElementById('myCanvas');
  const ctx = canvas.getContext('2d');

  // Draw a filled rectangle
  ctx.fillStyle = '#3498db';
  ctx.fillRect(10, 10, 150, 100);

  // Draw text
  ctx.fillStyle = '#fff';
  ctx.font = '20px Arial';
  ctx.fillText('Hello Canvas!', 20, 60);

  // Draw a circle
  ctx.beginPath();
  ctx.arc(300, 100, 50, 0, Math.PI * 2);
  ctx.fillStyle = '#e74c3c';
  ctx.fill();
</script>
```

**SVG (Scalable Vector Graphics):**
- Vector-based (mathematical shapes)
- Resolution-independent — scales without quality loss
- Part of the DOM — can be styled with CSS, manipulated with JS
- Good for icons, logos, charts, illustrations
- Accessible (can have `<title>` and `aria-label`)

```html
<svg width="200" height="200" xmlns="http://www.w3.org/2000/svg">
  <!-- Circle -->
  <circle cx="100" cy="100" r="50" fill="#3498db" stroke="#2c3e50" stroke-width="3" />
  <!-- Rectangle -->
  <rect x="10" y="10" width="80" height="50" fill="#e74c3c" rx="5" />
  <!-- Text -->
  <text x="100" y="170" text-anchor="middle" fill="#2c3e50">SVG Text</text>
  <!-- Path (custom shape) -->
  <path d="M 10 80 Q 95 10 180 80" stroke="#27ae60" fill="none" stroke-width="3" />
</svg>
```

**Canvas vs SVG comparison:**

| Feature | Canvas | SVG |
|---------|--------|-----|
| Type | Raster (pixels) | Vector (shapes) |
| Scalability | Loses quality on zoom | Perfect at any size |
| Performance | Better for many objects | Better for few, complex objects |
| DOM interaction | No (just a `<canvas>` tag) | Yes (each shape is a DOM node) |
| Events | Manual (coordinate math) | Native event listeners |
| Accessibility | Poor (needs ARIA workarounds) | Good (native roles/labels) |
| Best for | Games, image editing | Icons, charts, illustrations |

---

## 15. Audio & Video

HTML5 introduced native `<audio>` and `<video>` elements, replacing Flash.

```html
<!-- Video -->
<video
  width="640"
  height="360"
  controls
  autoplay
  muted
  loop
  poster="thumbnail.jpg"
  preload="metadata"
>
  <source src="video.mp4" type="video/mp4" />
  <source src="video.webm" type="video/webm" />   <!-- Fallback format -->
  <track kind="subtitles" src="subtitles-en.vtt" srclang="en" label="English" />
  <p>Your browser does not support video. <a href="video.mp4">Download it</a>.</p>
</video>

<!-- Audio -->
<audio controls preload="none">
  <source src="podcast.mp3" type="audio/mpeg" />
  <source src="podcast.ogg" type="audio/ogg" />
  Your browser does not support audio.
</audio>
```

**Key attributes:**

| Attribute | Purpose |
|-----------|---------|
| `controls` | Show play/pause/volume UI |
| `autoplay` | Play automatically (requires `muted` in most browsers) |
| `muted` | Start muted |
| `loop` | Repeat when finished |
| `poster` | Image to show before video plays |
| `preload` | `none` / `metadata` / `auto` — how much to load upfront |

> **Interview tip:** Browsers block `autoplay` with sound to prevent jarring user experiences. `autoplay` only works if `muted` is also set. To auto-play with sound, the user must have interacted with the page first.

---

## 16. Iframe & Embedding

`<iframe>` (inline frame) embeds another web page inside the current page.

```html
<!-- Basic iframe -->
<iframe
  src="https://www.youtube.com/embed/VIDEO_ID"
  width="560"
  height="315"
  title="YouTube video"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope"
  allowfullscreen
  loading="lazy"
></iframe>

<!-- Sandbox for security (restrict iframe capabilities) -->
<iframe
  src="untrusted-page.html"
  sandbox="allow-scripts allow-same-origin"
></iframe>
```

**`sandbox` attribute values:**
- `allow-scripts` — allows JavaScript
- `allow-forms` — allows form submission
- `allow-same-origin` — allows same-origin access
- `allow-popups` — allows popups
- Empty `sandbox` — most restrictive (no JS, no forms, no same-origin)

**Security concern:** Never load untrusted content in an iframe without `sandbox`. An iframe has access to `window.parent` and could do **clickjacking** attacks.

**`X-Frame-Options` header:** Servers set this to prevent their pages from being embedded in iframes elsewhere — a clickjacking defence.

---

## 17. Accessibility (a11y)

**Accessibility** ensures web content is usable by people with disabilities (visual, motor, hearing, cognitive).

**Real-world analogy:** Accessibility is like building ramps alongside stairs — it doesn't remove the stairs, it makes the building usable for everyone.

**WCAG (Web Content Accessibility Guidelines)** levels: A (minimum), AA (standard), AAA (enhanced).

**Key accessibility practices:**

```html
<!-- 1. Always use semantic HTML -->
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<!-- 2. Always include alt text for images -->
<img src="chart.png" alt="Bar chart showing 30% increase in Q4 revenue" />
<img src="decorative-divider.png" alt="" role="presentation" />  <!-- Decorative -->

<!-- 3. Associate labels with inputs -->
<label for="email">Email address</label>
<input type="email" id="email" name="email" required aria-describedby="email-hint" />
<span id="email-hint">We'll never share your email.</span>

<!-- 4. ARIA roles and attributes -->
<button aria-expanded="false" aria-controls="dropdown-menu">Menu</button>
<ul id="dropdown-menu" role="menu" hidden>
  <li role="menuitem"><a href="/profile">Profile</a></li>
</ul>

<!-- 5. Skip navigation link for keyboard users -->
<a href="#main-content" class="skip-link">Skip to main content</a>
<main id="main-content">...</main>

<!-- 6. Focus management for modals -->
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Confirm Delete</h2>
  ...
</div>

<!-- 7. Live regions for dynamic content -->
<div aria-live="polite" aria-atomic="true">
  <!-- Screen readers announce changes to this region -->
  <p>Form submitted successfully!</p>
</div>
```

**Common ARIA attributes:**

| Attribute | Purpose |
|-----------|---------|
| `aria-label` | Names an element when no visible label exists |
| `aria-labelledby` | Points to element(s) that label this element |
| `aria-describedby` | Points to element(s) that describe this element |
| `aria-hidden="true"` | Hides from screen readers (for decorative elements) |
| `aria-expanded` | Indicates if a collapsible element is open |
| `aria-required` | Indicates required field (same as `required` attribute) |
| `aria-invalid` | Indicates validation error |
| `aria-live` | Announces dynamic content changes |
| `role` | Overrides or provides semantic role |

> **Interview tip:** The first rule of ARIA — **don't use ARIA if a native HTML element does the job**. `<button>` is better than `<div role="button">` because native elements have built-in keyboard support, focus, and accessibility already.

---

## 18. Data Attributes

Data attributes let you store **custom data** on HTML elements without using non-standard attributes or hidden inputs.

```html
<!-- Define with data-* prefix -->
<div
  id="product-card"
  data-product-id="42"
  data-category="electronics"
  data-price="29.99"
  data-in-stock="true"
>
  Wireless Headphones
</div>

<button data-action="add-to-cart" data-product-id="42">Add to Cart</button>
```

```javascript
// Access in JavaScript via dataset property (camelCase conversion)
const card = document.getElementById('product-card');

console.log(card.dataset.productId);   // "42"   (data-product-id → productId)
console.log(card.dataset.category);    // "electronics"
console.log(card.dataset.price);       // "29.99"

// Modify
card.dataset.price = "24.99";

// Delete
delete card.dataset.inStock;

// Access via getAttribute (original kebab-case)
card.getAttribute('data-product-id');  // "42"

// CSS can also read data attributes
// [data-in-stock="true"] { border: 2px solid green; }
```

> **Interview tip:** Data attributes are always **strings** in the DOM. Numbers and booleans are stored as strings and must be parsed — `parseInt(card.dataset.productId)` or `card.dataset.inStock === 'true'`.

---

## 19. HTML APIs

### Drag & Drop API

```html
<div draggable="true" id="drag-item" ondragstart="dragStart(event)">Drag me</div>
<div id="drop-zone" ondrop="drop(event)" ondragover="allowDrop(event)">Drop here</div>

<script>
  function dragStart(e) {
    e.dataTransfer.setData('text/plain', e.target.id);
  }
  function allowDrop(e) {
    e.preventDefault();   // Required to allow dropping
  }
  function drop(e) {
    e.preventDefault();
    const id = e.dataTransfer.getData('text/plain');
    e.target.appendChild(document.getElementById(id));
  }
</script>
```

### Geolocation API

```javascript
if ('geolocation' in navigator) {
  navigator.geolocation.getCurrentPosition(
    position => {
      console.log('Lat:', position.coords.latitude);
      console.log('Lng:', position.coords.longitude);
    },
    error => {
      console.error('Error:', error.message);
    },
    { enableHighAccuracy: true, timeout: 5000 }
  );
}
```

### Web Storage API

```javascript
// localStorage — persists until explicitly cleared
localStorage.setItem('theme', 'dark');
const theme = localStorage.getItem('theme');   // "dark"
localStorage.removeItem('theme');
localStorage.clear();

// sessionStorage — cleared when tab/browser closes
sessionStorage.setItem('formData', JSON.stringify({ name: 'Amith' }));
const data = JSON.parse(sessionStorage.getItem('formData'));
```

**localStorage vs sessionStorage vs cookies:**

| Feature | localStorage | sessionStorage | Cookies |
|---------|-------------|----------------|---------|
| Capacity | ~5–10 MB | ~5 MB | ~4 KB |
| Expiry | Never (manual) | Tab close | Set by expiry date |
| Accessible from JS | Yes | Yes | Yes (unless HttpOnly) |
| Sent with HTTP requests | No | No | Yes (automatically) |
| Accessible across tabs | Yes | No | Yes |

---

## 20. Character Encoding & Entities

**Character encoding** tells the browser how to interpret the bytes in the HTML file. Always use **UTF-8**.

```html
<meta charset="UTF-8" />   <!-- Required — goes first in <head> -->
```

**HTML entities** — special characters that have meaning in HTML must be escaped:

| Character | Entity Name | Entity Number | When to use |
|-----------|-------------|---------------|-------------|
| `<` | `&lt;` | `&#60;` | Inside content (would open a tag) |
| `>` | `&gt;` | `&#62;` | Inside content |
| `&` | `&amp;` | `&#38;` | Inside content (would start an entity) |
| `"` | `&quot;` | `&#34;` | Inside attribute values |
| `'` | `&apos;` | `&#39;` | Inside attribute values |
| ` ` (space) | `&nbsp;` | `&#160;` | Non-breaking space |
| `©` | `&copy;` | `&#169;` | Copyright symbol |
| `®` | `&reg;` | `&#174;` | Registered trademark |
| `™` | `&trade;` | `&#8482;` | Trademark |
| `€` | `&euro;` | `&#8364;` | Euro sign |
| `→` | `&rarr;` | `&#8594;` | Right arrow |

```html
<!-- Without entities — INVALID, browser misinterprets -->
<p>5 < 10 & 10 > 5</p>

<!-- With entities — correct -->
<p>5 &lt; 10 &amp; 10 &gt; 5</p>

<!-- Non-breaking space prevents line break between words -->
<p>10&nbsp;kg</p>   <!-- "10 kg" won't break onto separate lines -->
```

---

## 21. Performance & Best Practices

### Script Loading Strategies

```html
<!-- ❌ Blocks HTML parsing — old approach -->
<script src="app.js"></script>

<!-- ✅ defer — downloads in parallel, executes AFTER HTML is parsed, in order -->
<script src="app.js" defer></script>

<!-- ✅ async — downloads in parallel, executes immediately when ready (order not guaranteed) -->
<script src="analytics.js" async></script>
```

**When to use:**
- `defer` — scripts that depend on DOM or other scripts (most scripts)
- `async` — independent scripts (analytics, ads)
- Neither — critical scripts that must run before DOM renders (rare)

### Resource Hints

```html
<!-- DNS prefetch — resolve domain early -->
<link rel="dns-prefetch" href="https://fonts.googleapis.com" />

<!-- Preconnect — full TCP + TLS handshake early -->
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />

<!-- Preload — fetch critical resource early (but don't execute yet) -->
<link rel="preload" href="hero-image.webp" as="image" />
<link rel="preload" href="font.woff2" as="font" type="font/woff2" crossorigin />

<!-- Prefetch — fetch for future navigation (low priority) -->
<link rel="prefetch" href="/next-page.html" />
```

### Best Practices Summary

```html
<!-- ✅ Use loading="lazy" for below-fold images -->
<img src="photo.jpg" alt="..." loading="lazy" />

<!-- ✅ Specify image dimensions to prevent layout shift (CLS) -->
<img src="photo.jpg" alt="..." width="800" height="600" />

<!-- ✅ Use WebP/AVIF for modern browsers with JPEG fallback -->
<picture>
  <source srcset="image.avif" type="image/avif" />
  <source srcset="image.webp" type="image/webp" />
  <img src="image.jpg" alt="..." />
</picture>

<!-- ✅ Use minified CSS/JS in production -->
<link rel="stylesheet" href="styles.min.css" />

<!-- ✅ External CSS in <head>, scripts before </body> or use defer -->
<head>
  <link rel="stylesheet" href="styles.css" />
</head>
<body>
  ...
  <script src="app.js" defer></script>
</body>
```

---

## 22. Common Interview Questions

### Q1: What is the difference between `<b>` and `<strong>`, `<i>` and `<em>`?

| Tag | Visual | Semantic meaning |
|-----|--------|-----------------|
| `<b>` | Bold | No semantic meaning |
| `<strong>` | Bold | **Important** content |
| `<i>` | Italic | No semantic meaning |
| `<em>` | Italic | **Emphasized** content |

> Use `<strong>` and `<em>` for semantic meaning. Screen readers may read `<strong>` with emphasis; they ignore `<b>` as purely visual.

---

### Q2: What is the difference between `id` and `class`?

- **`id`** — must be **unique** per page. Used for unique elements, URL fragments (`#id`), JavaScript targeting
- **`class`** — can be **reused** on multiple elements. Used for CSS styling groups, JavaScript targeting

---

### Q3: What are void elements? Give examples.

Void elements are elements that cannot have children and have no closing tag:
`<br>`, `<hr>`, `<img>`, `<input>`, `<link>`, `<meta>`, `<area>`, `<base>`, `<col>`, `<embed>`, `<param>`, `<source>`, `<track>`, `<wbr>`

---

### Q4: What is the difference between `localStorage`, `sessionStorage`, and cookies?

See the comparison table in [Section 19](#19-html-apis). Key summary:
- **localStorage**: Persistent, JS-only, 5–10 MB
- **sessionStorage**: Tab-scoped, JS-only, 5 MB
- **Cookies**: Sent with every HTTP request, 4 KB, set expiry date

---

### Q5: What is the difference between `<script defer>` and `<script async>`?

- **`defer`**: Script downloads in parallel with HTML parsing, executes **after** HTML is fully parsed, scripts execute **in order**
- **`async`**: Script downloads in parallel, executes **immediately** when downloaded (may interrupt parsing), order **not guaranteed**

---

### Q6: What is `<!DOCTYPE html>` and why is it needed?

It is a **declaration** (not a tag) that tells browsers to render the page in **standards mode** (HTML5). Without it, browsers enter **quirks mode** — they mimic old, buggy browser behaviour to maintain backward compatibility with 1990s websites.

---

### Q7: What are data attributes? When would you use them?

`data-*` attributes store custom private data on HTML elements. Use them to:
- Pass data from HTML to JavaScript without hidden inputs
- Store configuration for UI widgets
- Track state in HTML-first approaches

---

### Q8: What is the purpose of `alt` attribute on images?

1. **Accessibility** — screen readers read it aloud for visually impaired users
2. **SEO** — search engines use it to understand image content
3. **Fallback** — displayed when image fails to load
4. **Context** — provides meaning even when images are disabled

---

### Q9: What is the difference between `<section>`, `<article>`, and `<div>`?

- **`<article>`** — fully self-contained content that makes sense on its own (blog post, news story)
- **`<section>`** — thematic grouping that requires surrounding context
- **`<div>`** — generic container with no semantic meaning (use when no semantic element fits)

---

### Q10: What is ARIA and when should you use it?

ARIA (Accessible Rich Internet Applications) adds accessibility semantics to HTML via attributes like `role`, `aria-label`, `aria-expanded`. Use it **only when native HTML semantics are insufficient** — for custom interactive widgets (tabs, accordions, modals) that don't have native HTML equivalents.

---

### Q11: What are web workers in HTML5?

Web Workers allow running **JavaScript in a background thread**, separate from the main UI thread. This prevents heavy computations from blocking the UI.

```javascript
// main.js
const worker = new Worker('worker.js');
worker.postMessage({ data: bigArray });
worker.onmessage = e => console.log('Result:', e.data);

// worker.js
onmessage = e => {
  const result = heavyComputation(e.data);
  postMessage(result);
};
```

---

### Q12: What is the difference between `<link>` and `<a>`?

- **`<link>`** — goes in `<head>`, links external resources (CSS, icons, preload). Not clickable
- **`<a>`** — goes in `<body>`, creates clickable hyperlinks for users. Not for loading resources

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
  POST              → Data in body, secure, no size limit
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

HTML5 ENTITIES
  &lt;   <     &gt;   >
  &amp;  &     &quot; "
  &nbsp; (non-breaking space)
  &copy; ©     &reg;  ®
```

---

*Last updated: June 2026 | Part of the Interview Preparation Documentation Repository*
