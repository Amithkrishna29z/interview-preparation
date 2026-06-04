# CSS Interview Study Guide

## Overview

CSS (Cascading Style Sheets) controls the visual presentation of HTML documents. This guide covers everything from fundamentals to advanced topics commonly tested in frontend interviews.

---

## Table of Contents

1. [CSS Fundamentals](#css-fundamentals)
2. [Selectors & Specificity](#selectors--specificity)
3. [Box Model](#box-model)
4. [Display & Positioning](#display--positioning)
5. [Flexbox](#flexbox)
6. [CSS Grid](#css-grid)
7. [Responsive Design & Media Queries](#responsive-design--media-queries)
8. [Typography](#typography)
9. [Colors & Backgrounds](#colors--backgrounds)
10. [Transitions & Animations](#transitions--animations)
11. [CSS Variables (Custom Properties)](#css-variables-custom-properties)
12. [Pseudo-classes & Pseudo-elements](#pseudo-classes--pseudo-elements)
13. [CSS Architecture & Methodologies](#css-architecture--methodologies)
14. [Performance & Best Practices](#performance--best-practices)
15. [Common Interview Questions](#common-interview-questions)
16. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## CSS Fundamentals

### How CSS Works

CSS rules are applied in a **cascade** — the browser resolves conflicts using three factors:
1. **Specificity** — More specific rules win
2. **Origin** — Author > User > Browser default
3. **Order** — Later rules override earlier rules (when specificity is equal)

### CSS Rule Structure

```css
selector {
  property: value;
}

/* Example */
h1 {
  color: red;
  font-size: 2rem;
}
```

### Ways to Apply CSS

| Method | Syntax | Use Case |
|---|---|---|
| **Inline** | `<p style="color:red">` | Quick overrides (avoid in production) |
| **Internal** | `<style>` tag in `<head>` | Single-page styles |
| **External** | `<link rel="stylesheet" href="style.css">` | Preferred — reusable across pages |
| **@import** | `@import url('style.css')` | Modular CSS (blocks rendering, avoid) |

### The Cascade Order (low to high priority)

```
1. Browser default styles (user-agent stylesheet)
2. External stylesheets
3. Internal <style> tag
4. Inline styles
5. !important declarations  ← overrides everything (use sparingly)
```

> **Interview Tip**: `!important` overrides the cascade. Two `!important` rules resolve by specificity, then source order.

---

## Selectors & Specificity

### Selector Types

```css
/* Universal */
* { margin: 0; }

/* Type (element) */
p { color: black; }

/* Class */
.card { border: 1px solid #ccc; }

/* ID */
#header { background: blue; }

/* Attribute */
input[type="text"] { border: 1px solid gray; }
a[href^="https"] { color: green; }   /* starts with */
a[href$=".pdf"] { color: red; }      /* ends with */
a[href*="docs"] { color: blue; }     /* contains */

/* Descendant */
div p { font-size: 14px; }

/* Direct Child */
ul > li { list-style: none; }

/* Adjacent Sibling (immediately after) */
h1 + p { font-weight: bold; }

/* General Sibling (all siblings after) */
h1 ~ p { color: gray; }
```

### Specificity Calculation

Specificity is scored as **(A, B, C)**:

| Selector | A (IDs) | B (Classes/Attrs/Pseudo-classes) | C (Elements/Pseudo-elements) | Score |
|---|---|---|---|---|
| `p` | 0 | 0 | 1 | 0,0,1 |
| `.card` | 0 | 1 | 0 | 0,1,0 |
| `#header` | 1 | 0 | 0 | 1,0,0 |
| `div p` | 0 | 0 | 2 | 0,0,2 |
| `.card p` | 0 | 1 | 1 | 0,1,1 |
| `#nav .item` | 1 | 1 | 0 | 1,1,0 |
| `style=""` (inline) | — | — | — | 1,0,0,0 |

```css
/* Specificity example */
p { color: black; }              /* 0,0,1 */
.text { color: blue; }           /* 0,1,0 — wins over p */
#main p { color: red; }          /* 1,0,1 — wins over .text */
p { color: green !important; }   /* !important — wins over everything */
```

> **Interview Tip**: Inline styles beat ID selectors. `!important` beats inline styles. Two `!important` rules fall back to specificity.

### Combinators Summary

```
div p       → descendant (any level deep)
div > p     → direct child only
h1 + p      → adjacent sibling (immediately after)
h1 ~ p      → general sibling (all p after h1 in same parent)
```

---

## Box Model

Every HTML element is a rectangular box made of:

```
┌─────────────────────────────────────┐
│              MARGIN                 │
│   ┌─────────────────────────────┐   │
│   │           BORDER            │   │
│   │   ┌─────────────────────┐   │   │
│   │   │       PADDING       │   │   │
│   │   │   ┌─────────────┐   │   │   │
│   │   │   │   CONTENT   │   │   │   │
│   │   │   │  width x    │   │   │   │
│   │   │   │  height     │   │   │   │
│   │   │   └─────────────┘   │   │   │
│   │   └─────────────────────┘   │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

### box-sizing

```css
/* Default — width applies to content only */
box-sizing: content-box;
/* Total width = width + padding + border */

/* Intuitive — width includes padding and border */
box-sizing: border-box;
/* Total width = width (padding + border included) */

/* Best practice — apply globally */
*, *::before, *::after {
  box-sizing: border-box;
}
```

### Margin Collapse

Vertical margins between adjacent block elements collapse to the **larger** of the two values.

```css
/* Both paragraphs have margin-bottom/top of 20px */
/* Gap between them is 20px, NOT 40px */
p { margin: 20px 0; }
```

**Margin collapse does NOT happen:**
- Flex or Grid children
- Floated elements
- Absolutely positioned elements
- Elements with `overflow` other than `visible`

### Margin vs Padding

| Property | Affects Background | Can be Negative | Collapses |
|---|---|---|---|
| `margin` | No (outside box) | Yes | Yes (vertical) |
| `padding` | Yes (inside box) | No | No |

---

## Display & Positioning

### Display Values

```css
display: block;        /* Full width, starts on new line: div, p, h1 */
display: inline;       /* Flows with text, no width/height: span, a */
display: inline-block; /* Flows inline but accepts width/height */
display: flex;         /* Flex container */
display: grid;         /* Grid container */
display: none;         /* Removed from layout (no space) */
display: contents;     /* Element itself disappears, children remain */
```

| Display | Width | Height | Margin/Padding | Line Break |
|---|---|---|---|---|
| `block` | Full parent width | Auto | All sides | Yes |
| `inline` | Content width | Content height | Left/Right only | No |
| `inline-block` | Content width | Settable | All sides | No |

### Position Values

```css
position: static;    /* Default — in normal document flow */
position: relative;  /* Offset from normal position; still takes up space */
position: absolute;  /* Removed from flow; positioned relative to nearest positioned ancestor */
position: fixed;     /* Removed from flow; positioned relative to viewport */
position: sticky;    /* Relative until scroll threshold, then fixed */
```

```css
/* Centering with absolute + transform */
.centered {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
```

### z-index

- Only works on **positioned** elements (relative, absolute, fixed, sticky)
- Higher value = closer to the viewer
- Stacking context is created by: `position` + `z-index`, `opacity < 1`, `transform`, `filter`

```css
.modal { position: fixed; z-index: 1000; }
.overlay { position: fixed; z-index: 999; }
```

### Float & Clear

```css
.image { float: left; margin-right: 16px; }

/* Clearfix — prevent parent collapsing around floats */
.clearfix::after {
  content: '';
  display: table;
  clear: both;
}
```

> **Interview Tip**: Floats are largely replaced by Flexbox and Grid. Know clearfix for legacy code questions.

---

## Flexbox

Flexbox is a **one-dimensional** layout system (row OR column).

### Flex Container Properties

```css
.container {
  display: flex;

  /* Direction */
  flex-direction: row;            /* row | row-reverse | column | column-reverse */

  /* Wrapping */
  flex-wrap: nowrap;              /* nowrap | wrap | wrap-reverse */

  /* Shorthand: flex-direction + flex-wrap */
  flex-flow: row wrap;

  /* Alignment along main axis */
  justify-content: flex-start;   /* flex-start | flex-end | center | space-between | space-around | space-evenly */

  /* Alignment along cross axis */
  align-items: stretch;          /* stretch | flex-start | flex-end | center | baseline */

  /* Multi-line cross axis alignment */
  align-content: flex-start;     /* stretch | flex-start | flex-end | center | space-between | space-around */

  gap: 16px;                     /* gap between flex items */
}
```

### Flex Item Properties

```css
.item {
  /* flex-grow: how much item grows relative to siblings */
  flex-grow: 1;     /* 0 = no grow (default) */

  /* flex-shrink: how much item shrinks relative to siblings */
  flex-shrink: 1;   /* 1 = can shrink (default) */

  /* flex-basis: initial size before grow/shrink */
  flex-basis: auto; /* auto | 0 | 200px | 30% */

  /* Shorthand: grow shrink basis */
  flex: 1;          /* flex: 1 1 0% */
  flex: auto;       /* flex: 1 1 auto */
  flex: none;       /* flex: 0 0 auto */

  /* Override container's align-items for this item */
  align-self: center;

  /* Reorder items visually (default: 0) */
  order: 2;
}
```

### Common Flexbox Patterns

```css
/* Center anything */
.center {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Navbar: logo left, links right */
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

/* Equal-width columns */
.columns > * {
  flex: 1;
}

/* Sticky footer */
body {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}
main { flex: 1; }
```

### Flexbox Axes Diagram

```
flex-direction: row (default)

Main axis ──────────────────────────►
┌──────────────────────────────────┐
│  [Item 1]  [Item 2]  [Item 3]    │  ▲
│                                  │  │ Cross axis
└──────────────────────────────────┘  ▼

justify-content controls → main axis
align-items controls     → cross axis
```

---

## CSS Grid

CSS Grid is a **two-dimensional** layout system (rows AND columns).

### Grid Container Properties

```css
.container {
  display: grid;

  /* Define columns */
  grid-template-columns: 200px 1fr 1fr;
  grid-template-columns: repeat(3, 1fr);
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));

  /* Define rows */
  grid-template-rows: auto 1fr auto;

  /* Gap */
  gap: 16px;
  column-gap: 24px;
  row-gap: 16px;

  /* Named areas */
  grid-template-areas:
    "header header header"
    "sidebar main main"
    "footer footer footer";

  /* Align items in cells */
  justify-items: stretch;   /* horizontal within cell */
  align-items: stretch;     /* vertical within cell */

  /* Align entire grid in container */
  justify-content: start;
  align-content: start;
}
```

### Grid Item Properties

```css
.item {
  /* Place by line numbers */
  grid-column: 1 / 3;        /* span from line 1 to 3 */
  grid-column: 1 / span 2;   /* start at 1, span 2 columns */
  grid-row: 2 / 4;

  /* Place in named area */
  grid-area: header;

  /* Self alignment */
  justify-self: center;
  align-self: end;
}
```

### Named Grid Areas Example

```css
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar content"
    "footer footer";
  grid-template-columns: 250px 1fr;
  grid-template-rows: 60px 1fr 60px;
  min-height: 100vh;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.content { grid-area: content; }
.footer  { grid-area: footer; }
```

### fr Unit

`fr` = fraction of remaining space after fixed sizes are allocated.

```css
grid-template-columns: 200px 1fr 2fr;
/* 200px fixed, remaining split: 1/3 and 2/3 */
```

### auto-fill vs auto-fit

```css
/* auto-fill: creates empty columns to fill space */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));

/* auto-fit: collapses empty columns, items stretch */
grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
```

### Grid vs Flexbox

| Dimension | Flexbox | Grid |
|---|---|---|
| Layout type | 1D (row or column) | 2D (rows and columns) |
| Content-driven | Yes — size adapts to content | No — define structure first |
| Alignment | Cross-axis only | Both axes precisely |
| Best for | Navigation bars, card rows, small components | Page layouts, complex grids |
| Browser Support | Excellent | Excellent (IE11 partial) |

---

## Responsive Design & Media Queries

### Viewport Meta Tag

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

Always include this — without it, mobile browsers zoom out to fit desktop content.

### Media Query Syntax

```css
/* Breakpoint: mobile first (min-width) */
/* Base styles: mobile */
.container { padding: 16px; }

/* Tablet */
@media (min-width: 768px) {
  .container { padding: 24px; }
}

/* Desktop */
@media (min-width: 1024px) {
  .container { padding: 32px; }
}

/* Large desktop */
@media (min-width: 1280px) {
  .container { max-width: 1200px; margin: 0 auto; }
}
```

### Common Breakpoints

| Device | Breakpoint |
|---|---|
| Mobile (default) | < 768px |
| Tablet | 768px – 1023px |
| Desktop | 1024px – 1279px |
| Large Desktop | 1280px+ |

### Media Query Features

```css
/* Orientation */
@media (orientation: landscape) { ... }
@media (orientation: portrait) { ... }

/* Dark mode */
@media (prefers-color-scheme: dark) {
  body { background: #1a1a1a; color: white; }
}

/* Reduced motion (accessibility) */
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: 0.01ms !important; }
}

/* High resolution (Retina) */
@media (-webkit-min-device-pixel-ratio: 2), (min-resolution: 192dpi) {
  .logo { background-image: url('logo@2x.png'); }
}

/* Multiple conditions */
@media (min-width: 768px) and (max-width: 1023px) { ... }
@media (max-width: 600px), (orientation: portrait) { ... }
```

### Mobile-First vs Desktop-First

```css
/* Mobile-first (recommended) */
/* Base = mobile, add complexity for larger screens */
.card { font-size: 14px; }
@media (min-width: 768px) { .card { font-size: 16px; } }

/* Desktop-first */
/* Base = desktop, simplify for smaller screens */
.card { font-size: 16px; }
@media (max-width: 767px) { .card { font-size: 14px; } }
```

### Fluid Typography

```css
/* clamp(min, preferred, max) */
h1 {
  font-size: clamp(1.5rem, 5vw, 3rem);
}

/* Fluid values between breakpoints */
p {
  font-size: clamp(1rem, 1rem + 0.5vw, 1.25rem);
}
```

---

## Typography

### Font Properties

```css
p {
  font-family: 'Inter', Arial, sans-serif; /* font stack */
  font-size: 16px;           /* absolute */
  font-size: 1rem;           /* relative to root (preferred) */
  font-size: 1.2em;          /* relative to parent */
  font-weight: 400;          /* 100–900, or normal/bold */
  font-style: italic;        /* normal | italic | oblique */
  font-variant: small-caps;
  line-height: 1.5;          /* unitless (preferred) — 1.5 × font-size */
  letter-spacing: 0.05em;
  word-spacing: 0.1em;

  /* Shorthand: style variant weight size/line-height family */
  font: italic small-caps bold 16px/1.5 'Inter', sans-serif;
}
```

### Text Properties

```css
p {
  text-align: left;           /* left | right | center | justify */
  text-decoration: underline; /* none | underline | line-through | overline */
  text-transform: uppercase;  /* none | uppercase | lowercase | capitalize */
  text-indent: 2em;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
  white-space: nowrap;        /* nowrap | pre | pre-wrap | pre-line */
  overflow: hidden;
  text-overflow: ellipsis;    /* clips overflow text with ... */
  word-break: break-word;
}
```

### rem vs em vs px

| Unit | Relative To | Use Case |
|---|---|---|
| `px` | Absolute pixels | Borders, shadows |
| `em` | Parent element's font-size | Component-level spacing |
| `rem` | Root (`html`) font-size | Base typography, global spacing |
| `%` | Parent element's dimension | Widths, fluid layouts |
| `vw` / `vh` | Viewport width / height | Hero sections, full-screen layouts |
| `vmin` / `vmax` | Smaller / larger viewport dimension | Responsive sizing |
| `ch` | Width of the `0` character | Readable line lengths (~60-75ch) |

```css
/* Setting the root font size for easy rem math */
html { font-size: 62.5%; }  /* 1rem = 10px */
body { font-size: 1.6rem; } /* 16px */
h1   { font-size: 3.2rem; } /* 32px */
```

### Web Fonts

```css
@font-face {
  font-family: 'MyFont';
  src: url('myfont.woff2') format('woff2'),
       url('myfont.woff') format('woff');
  font-weight: 400;
  font-display: swap; /* prevents invisible text while loading */
}
```

---

## Colors & Backgrounds

### Color Formats

```css
color: red;                       /* named */
color: #ff6347;                   /* hex */
color: #f63;                      /* shorthand hex */
color: rgb(255, 99, 71);          /* RGB */
color: rgba(255, 99, 71, 0.8);    /* RGB with alpha */
color: hsl(9, 100%, 64%);         /* hue, saturation, lightness */
color: hsla(9, 100%, 64%, 0.8);   /* HSL with alpha */
color: oklch(0.65 0.2 30);        /* modern perceptually uniform */
```

### Background Properties

```css
.box {
  background-color: #f0f0f0;
  background-image: url('bg.jpg');
  background-repeat: no-repeat;     /* repeat | repeat-x | repeat-y | no-repeat */
  background-position: center top;  /* x y */
  background-size: cover;           /* cover | contain | auto | 100px 200px */
  background-attachment: fixed;     /* scroll | fixed | local */
  background-origin: border-box;    /* padding-box | border-box | content-box */
  background-clip: text;            /* padding-box | border-box | content-box | text */

  /* Shorthand */
  background: url('bg.jpg') no-repeat center/cover;
}
```

### Gradients

```css
/* Linear gradient */
background: linear-gradient(to right, #ff6347, #4682b4);
background: linear-gradient(135deg, #ff6347 0%, #4682b4 100%);

/* Radial gradient */
background: radial-gradient(circle, #ff6347, #4682b4);
background: radial-gradient(ellipse at center, #ff6347 0%, #4682b4 100%);

/* Conic gradient */
background: conic-gradient(from 0deg, red, yellow, green, blue, red);

/* Multiple backgrounds */
background:
  linear-gradient(rgba(0,0,0,0.3), rgba(0,0,0,0.3)),
  url('hero.jpg') no-repeat center/cover;
```

---

## Transitions & Animations

### Transitions

Transitions animate a change from one state to another smoothly.

```css
.button {
  background: blue;
  transition: background 0.3s ease, transform 0.2s ease;
}

.button:hover {
  background: darkblue;
  transform: translateY(-2px);
}

/* Transition shorthand: property duration timing-function delay */
transition: all 0.3s ease-in-out 0s;
```

### Timing Functions

| Function | Behavior |
|---|---|
| `ease` | Slow start, fast middle, slow end (default) |
| `linear` | Constant speed |
| `ease-in` | Slow start, fast end |
| `ease-out` | Fast start, slow end |
| `ease-in-out` | Slow start and end |
| `cubic-bezier(x1,y1,x2,y2)` | Custom curve |

### CSS Animations

```css
/* Define keyframes */
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Apply animation */
.card {
  animation-name: fadeInUp;
  animation-duration: 0.5s;
  animation-timing-function: ease-out;
  animation-delay: 0.1s;
  animation-iteration-count: 1;        /* infinite | number */
  animation-direction: normal;         /* normal | reverse | alternate */
  animation-fill-mode: both;           /* none | forwards | backwards | both */
  animation-play-state: running;       /* running | paused */

  /* Shorthand: name duration timing-function delay iteration direction fill-mode */
  animation: fadeInUp 0.5s ease-out 0.1s 1 normal both;
}
```

### Transform

```css
.element {
  transform: translateX(50px);
  transform: translateY(-20px);
  transform: translate(50px, -20px);
  transform: scale(1.2);
  transform: scaleX(0.8);
  transform: rotate(45deg);
  transform: skew(10deg, 5deg);

  /* Chain multiple transforms */
  transform: translateX(50px) rotate(45deg) scale(1.2);

  /* 3D transforms */
  transform: rotateX(45deg);
  transform: rotateY(45deg);
  transform: perspective(500px) rotateY(45deg);
}
```

### will-change (Performance Hint)

```css
.animated {
  will-change: transform, opacity;
}
```

Tells the browser to create a new compositor layer in advance — use sparingly as it consumes memory.

---

## CSS Variables (Custom Properties)

### Defining and Using Variables

```css
/* Define on :root for global scope */
:root {
  --color-primary: #3b82f6;
  --color-text: #1f2937;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --border-radius: 4px;
  --shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

/* Use with var() */
.button {
  background: var(--color-primary);
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--border-radius);
  box-shadow: var(--shadow);
}

/* Fallback value */
color: var(--color-accent, #ff6347);
```

### Local Scope

```css
.card {
  --card-padding: 24px;  /* scoped to .card */
  padding: var(--card-padding);
}

.card--compact {
  --card-padding: 12px;  /* override for compact variant */
}
```

### Dynamic Theming with Variables

```css
:root {
  --bg: #ffffff;
  --text: #1a1a1a;
}

[data-theme="dark"] {
  --bg: #1a1a1a;
  --text: #ffffff;
}

body {
  background: var(--bg);
  color: var(--text);
}
```

```javascript
// Toggle theme in JavaScript
document.documentElement.setAttribute('data-theme', 'dark');

// Update variable at runtime
document.documentElement.style.setProperty('--color-primary', '#ff6347');
```

> **Interview Tip**: CSS variables are different from preprocessor variables (Sass/Less). CSS variables are **live** in the DOM and can be changed at runtime with JavaScript. They also cascade and inherit.

---

## Pseudo-classes & Pseudo-elements

### Pseudo-classes (state-based)

```css
/* User interaction */
a:hover     { color: blue; }
a:focus     { outline: 2px solid blue; }
a:active    { color: red; }
button:disabled { opacity: 0.5; }

/* Form states */
input:checked   { ... }
input:valid     { border-color: green; }
input:invalid   { border-color: red; }
input:required  { ... }
input:focus-within { ... }  /* parent when child is focused */

/* Structural */
li:first-child   { font-weight: bold; }
li:last-child    { margin-bottom: 0; }
li:nth-child(2)  { color: red; }
li:nth-child(odd)     { background: #f0f0f0; }
li:nth-child(even)    { background: white; }
li:nth-child(3n+1)    { color: blue; }  /* every 3rd starting at 1 */
li:nth-last-child(1)  { ... }           /* from the end */

p:first-of-type  { ... }
p:last-of-type   { ... }
p:nth-of-type(2) { ... }

/* Other */
:not(.active)    { opacity: 0.5; }
:is(h1, h2, h3)  { font-weight: bold; }   /* matches any in list */
:where(h1, h2)   { margin: 0; }            /* like :is() but 0 specificity */
:has(img)        { border: 1px solid #ccc; } /* parent selector */
```

### Pseudo-elements (virtual elements)

```css
/* Before and after content */
.button::before {
  content: '→ ';
}
.required::after {
  content: ' *';
  color: red;
}

/* Styling first line / letter */
p::first-line   { font-weight: bold; }
p::first-letter { font-size: 2em; float: left; }

/* Selection highlighting */
::selection {
  background: #3b82f6;
  color: white;
}

/* Placeholder text */
input::placeholder { color: #9ca3af; }

/* Scrollbar (WebKit) */
::-webkit-scrollbar       { width: 8px; }
::-webkit-scrollbar-track { background: #f1f1f1; }
::-webkit-scrollbar-thumb { background: #888; border-radius: 4px; }
```

> **Interview Tip**: Pseudo-classes use `:` (single colon) and target existing elements in specific states. Pseudo-elements use `::` (double colon) and create virtual elements.

---

## CSS Architecture & Methodologies

### BEM (Block Element Modifier)

Naming convention for maintainable, scalable CSS.

```css
/* Block: standalone component */
.card { }

/* Element: part of a block (double underscore) */
.card__title { }
.card__image { }
.card__body  { }

/* Modifier: variant of block or element (double dash) */
.card--featured { }
.card--dark { }
.card__title--large { }
```

```html
<div class="card card--featured">
  <img class="card__image" src="..." alt="...">
  <div class="card__body">
    <h2 class="card__title card__title--large">Title</h2>
  </div>
</div>
```

### SMACSS (Scalable and Modular Architecture for CSS)

Categorizes CSS into 5 types:

```
Base       → Default element styles (body, h1, a)
Layout     → Page structure (.l-header, .l-sidebar)
Module     → Reusable components (.card, .nav, .modal)
State      → State changes (.is-active, .is-hidden)
Theme      → Visual theme overrides (.theme-dark)
```

### OOCSS (Object-Oriented CSS)

Separates structure from skin, container from content.

```css
/* Structure */
.btn { display: inline-block; padding: 8px 16px; border-radius: 4px; }

/* Skin (separate from structure) */
.btn--primary { background: blue; color: white; }
.btn--danger  { background: red; color: white; }
```

### Utility-First CSS (Tailwind approach)

```html
<!-- Single-purpose utility classes -->
<div class="flex items-center justify-between p-4 bg-white rounded-lg shadow">
  <h2 class="text-xl font-bold text-gray-900">Title</h2>
  <button class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
    Click me
  </button>
</div>
```

### CSS Modules (Component-scoped CSS)

```css
/* Button.module.css */
.button { background: blue; }

/* Compiled to: .Button_button__xK3f2 */
```

```javascript
import styles from './Button.module.css';
<button className={styles.button}>Click</button>
```

---

## Performance & Best Practices

### CSS Performance Tips

```css
/* Avoid expensive selectors */
* { }                    /* Universal — avoid for styling */
[hidden="true"] { }      /* Attribute — slow */
.parent .child .deep { } /* Deep nesting — slower than flat */

/* Prefer class selectors */
.nav-item { }            /* Fast */

/* Avoid expensive properties (trigger layout/paint) */
/* Layout-triggering: width, height, margin, padding, display, position, top, left */
/* Paint-triggering: background, color, border, box-shadow */
/* Compositor-only (GPU): transform, opacity, filter */
```

### Critical Rendering Path

```
HTML Parsing → DOM
CSS Parsing  → CSSOM
              → Render Tree → Layout → Paint → Composite
```

- CSS is **render-blocking** — browser won't render until CSS is parsed
- Minimize critical CSS; defer non-critical stylesheets
- Inline critical CSS in `<head>` for above-the-fold content

### Reflow vs Repaint vs Composite

| Operation | Cost | Triggered By |
|---|---|---|
| **Reflow (Layout)** | Most expensive | width, height, margin, padding, position |
| **Repaint** | Expensive | color, background, visibility |
| **Composite** | Cheap (GPU) | transform, opacity |

```css
/* WRONG — triggers reflow */
.animate { left: 0; }
.animate:hover { left: 100px; }

/* RIGHT — compositor only */
.animate { transform: translateX(0); }
.animate:hover { transform: translateX(100px); }
```

### Accessibility Best Practices

```css
/* Never remove outline without replacement */
:focus { outline: none; }           /* BAD */
:focus { outline: 2px solid blue; } /* GOOD */

/* Use focus-visible for mouse vs keyboard */
:focus-visible { outline: 2px solid blue; }

/* Ensure color contrast (WCAG AA: 4.5:1 for text) */
.text { color: #595959; background: #ffffff; } /* 7:1 — passes AAA */

/* Avoid hiding content visually but accessibly */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

### CSS Reset vs Normalize

```css
/* Hard Reset — removes all browser defaults */
* { margin: 0; padding: 0; box-sizing: border-box; }

/* Normalize.css — preserves useful defaults, fixes bugs */
/* Use a library like normalize.css or modern-normalize */
```

---

## Common Interview Questions

### Q: What is the difference between `visibility: hidden` and `display: none`?

| Property | Space Occupied | Accessible | Use Case |
|---|---|---|---|
| `display: none` | No | No (removed from tree) | Hide/show toggling |
| `visibility: hidden` | Yes | No | Placeholder preservation |
| `opacity: 0` | Yes | Yes (still clickable!) | Fade animations |

---

### Q: What is the difference between `absolute` and `fixed` positioning?

- **`absolute`**: Positioned relative to the nearest **positioned ancestor** (any element with `position` other than `static`). Scrolls with the page.
- **`fixed`**: Positioned relative to the **viewport**. Does NOT scroll with the page.

---

### Q: Explain the CSS specificity of `#id .class element`

```
#id       → (1, 0, 0)
.class    → (0, 1, 0)
element   → (0, 0, 1)
Total     → (1, 1, 1)
```

---

### Q: How do you center a div horizontally and vertically?

```css
/* Method 1: Flexbox (recommended) */
.parent {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Method 2: Grid */
.parent {
  display: grid;
  place-items: center;
}

/* Method 3: Absolute + Transform */
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* Method 4: Absolute + Margin Auto */
.child {
  position: absolute;
  inset: 0;
  margin: auto;
  width: fit-content;
  height: fit-content;
}
```

---

### Q: What is the stacking context?

A stacking context is a 3D conceptual space. Elements within a stacking context are painted together as a group. A new stacking context is created by:

- `position: relative/absolute/fixed/sticky` + `z-index` other than `auto`
- `opacity` less than 1
- `transform`, `filter`, `perspective`, `clip-path`
- `isolation: isolate`
- `will-change`

---

### Q: What is the difference between CSS Grid and Flexbox?

```
Flexbox:
  └── One dimension: either a row OR a column
  └── Content-driven: item sizes adapt to content
  └── Best for: navigation, toolbars, card rows

Grid:
  └── Two dimensions: rows AND columns simultaneously
  └── Layout-driven: define structure, content fills it
  └── Best for: page layouts, complex grid systems
```

---

### Q: What is `em` vs `rem`?

```css
html { font-size: 16px; }

.parent { font-size: 20px; }

.child {
  font-size: 1.5em;   /* 1.5 × 20px = 30px (relative to PARENT) */
  padding: 1.5rem;    /* 1.5 × 16px = 24px (relative to ROOT) */
}
```

---

### Q: What is the difference between `min-width`, `max-width`, and `width`?

```css
.box {
  width: 300px;       /* Always 300px (can overflow parent) */
  max-width: 300px;   /* At most 300px, can be smaller */
  min-width: 300px;   /* At least 300px, can be larger */
}

/* Common pattern: fluid but capped */
.container {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
}
```

---

### Q: How does `z-index` work?

- Only works on **positioned** elements
- Elements with higher `z-index` appear in front
- `z-index` is scoped within a stacking context

```css
/* z-index: auto creates NO new stacking context */
.a { position: relative; z-index: 100; }
.b { position: relative; z-index: 50; }
/* .a is in front of .b */
```

---

### Q: What is `clip-path`?

```css
/* Circle */
.circle { clip-path: circle(50%); }

/* Polygon (triangle) */
.triangle { clip-path: polygon(50% 0%, 0% 100%, 100% 100%); }

/* Inset (rounded rect) */
.inset { clip-path: inset(10px 20px round 8px); }
```

---

### Q: What is `object-fit`?

Controls how an `<img>` or `<video>` fills its container:

```css
img {
  width: 300px;
  height: 200px;
  object-fit: cover;     /* fill container, crop if needed */
  object-fit: contain;   /* fit inside container, letterbox */
  object-fit: fill;      /* stretch to fill (distorts) */
  object-fit: none;      /* original size */
  object-position: top;  /* where to anchor the image */
}
```

---

## Quick Reference Cheat Sheet

### Flexbox Quick Reference

```css
/* Container */
display: flex;
flex-direction: row | column | row-reverse | column-reverse;
flex-wrap: nowrap | wrap;
justify-content: flex-start | flex-end | center | space-between | space-around | space-evenly;
align-items: stretch | flex-start | flex-end | center | baseline;
align-content: flex-start | flex-end | center | space-between | space-around | stretch;
gap: <value>;

/* Item */
flex: <grow> <shrink> <basis>;
flex-grow: 0;
flex-shrink: 1;
flex-basis: auto;
align-self: auto | flex-start | flex-end | center | stretch;
order: 0;
```

### Grid Quick Reference

```css
/* Container */
display: grid;
grid-template-columns: repeat(3, 1fr);
grid-template-rows: auto;
grid-template-areas: "..." "...";
gap: <row-gap> <col-gap>;
justify-items: start | end | center | stretch;
align-items: start | end | center | stretch;
place-items: <align> <justify>;

/* Item */
grid-column: 1 / 3;
grid-row: 1 / span 2;
grid-area: name;
justify-self: start | end | center | stretch;
align-self: start | end | center | stretch;
```

### Positioning Quick Reference

```css
position: static;    /* default */
position: relative;  /* offset from self; creates stacking context with z-index */
position: absolute;  /* relative to nearest positioned ancestor */
position: fixed;     /* relative to viewport */
position: sticky;    /* relative until scrolled to threshold */

/* Offset properties */
top | right | bottom | left | inset
```

### Units Quick Reference

```
px   → absolute pixels
em   → relative to parent font-size
rem  → relative to root font-size
%    → relative to parent dimension
vw   → % of viewport width
vh   → % of viewport height
fr   → fraction of grid/flex container
ch   → width of "0" character
clamp(min, val, max) → fluid value with bounds
```

### Common Patterns Quick Reference

```css
/* Full-page layout */
body { display: grid; grid-template-rows: auto 1fr auto; min-height: 100vh; }

/* Center anything */
.center { display: grid; place-items: center; }

/* Responsive grid */
.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(250px, 1fr)); gap: 1rem; }

/* Truncate text */
.truncate { overflow: hidden; white-space: nowrap; text-overflow: ellipsis; }

/* Aspect ratio */
.video { aspect-ratio: 16 / 9; }

/* Custom scrollbar */
.scroll { overflow-y: auto; scrollbar-width: thin; scrollbar-color: #888 #f1f1f1; }

/* Visually hidden (accessible) */
.sr-only { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0,0,0,0); white-space: nowrap; }
```

---

## CSS Interview Topics Checklist

- [ ] Box model: `content-box` vs `border-box`
- [ ] Specificity calculation: inline > ID > class > element
- [ ] Cascade, inheritance, and `!important`
- [ ] Margin collapse and when it doesn't occur
- [ ] `display`: block, inline, inline-block, flex, grid, none
- [ ] Position: static, relative, absolute, fixed, sticky
- [ ] `z-index` and stacking context creation
- [ ] Flexbox: container and item properties
- [ ] Grid: template definitions, named areas, `fr` unit
- [ ] `em` vs `rem` vs `px`
- [ ] Media queries: mobile-first vs desktop-first
- [ ] Pseudo-classes vs pseudo-elements (`:` vs `::`)
- [ ] CSS variables: scope, fallback, JS manipulation
- [ ] Transitions vs animations
- [ ] `transform`: translate, scale, rotate (compositor-only)
- [ ] Performance: reflow vs repaint vs composite
- [ ] BEM naming convention
- [ ] `visibility: hidden` vs `display: none` vs `opacity: 0`
- [ ] Centering techniques (flexbox, grid, absolute)
- [ ] `object-fit` for images
- [ ] `clamp()` for fluid typography

---

*Last Updated: 2026-06-04*
