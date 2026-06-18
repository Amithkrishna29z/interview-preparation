# CSS Interview Study Guide

## Overview

CSS (Cascading Style Sheets) controls the visual presentation of HTML documents. This guide covers the fundamentals commonly tested in junior frontend interviews.

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

CSS rules are applied in a **cascade** — the browser resolves conflicts using:
1. **Specificity** — More specific rules win
2. **Origin** — Author > User > Browser default
3. **Order** — Later rules override earlier ones (when specificity is equal)

### CSS Rule Structure

```css
selector {
  property: value;
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
1. Browser default styles
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
* { margin: 0; }                  /* Universal */
p { color: black; }               /* Type */
.card { border: 1px solid #ccc; } /* Class */
#header { background: blue; }     /* ID */
input[type="text"] { border: 1px solid gray; } /* Attribute */

div p { font-size: 14px; }        /* Descendant */
ul > li { list-style: none; }     /* Direct child */
h1 + p { font-weight: bold; }     /* Adjacent sibling */
h1 ~ p { color: gray; }           /* General sibling */
```

### Specificity Calculation

Specificity is scored as **(A, B, C)**:

| Selector | Score |
|---|---|
| `p` | 0,0,1 |
| `.card` | 0,1,0 |
| `#header` | 1,0,0 |
| `.card p` | 0,1,1 |
| `#nav .item` | 1,1,0 |
| Inline `style=""` | 1,0,0,0 |

```css
p { color: black; }              /* 0,0,1 */
.text { color: blue; }           /* 0,1,0 — wins */
#main p { color: red; }          /* 1,0,1 — wins */
p { color: green !important; }   /* wins over everything */
```

> **Interview Tip**: Inline styles beat ID selectors. `!important` beats inline styles.

---

## Box Model

Every HTML element is a rectangular box:

```
┌──────────── MARGIN ────────────┐
│  ┌────────── BORDER ─────────┐ │
│  │  ┌──────PADDING──────┐   │ │
│  │  │   CONTENT         │   │ │
│  │  └───────────────────┘   │ │
│  └───────────────────────────┘ │
└────────────────────────────────┘
```

### box-sizing

```css
/* Default — width applies to content only; total = width + padding + border */
box-sizing: content-box;

/* Intuitive — width includes padding and border */
box-sizing: border-box;

/* Best practice */
*, *::before, *::after { box-sizing: border-box; }
```

### Margin Collapse

Vertical margins between adjacent block elements collapse to the **larger** of the two values (20px + 20px = 20px gap, not 40px).

Margin collapse does NOT happen in flex/grid containers, floated elements, or absolutely positioned elements.

### Margin vs Padding

| Property | Affects Background | Can be Negative | Collapses |
|---|---|---|---|
| `margin` | No | Yes | Yes (vertical) |
| `padding` | Yes | No | No |

---

## Display & Positioning

### Display Values

```css
display: block;        /* Full width, new line: div, p, h1 */
display: inline;       /* Flows with text, no width/height: span, a */
display: inline-block; /* Flows inline but accepts width/height */
display: flex;
display: grid;
display: none;         /* Removed from layout (no space) */
```

| Display | Width | Height | Margin/Padding | Line Break |
|---|---|---|---|---|
| `block` | Full parent | Auto | All sides | Yes |
| `inline` | Content | Content | Left/Right only | No |
| `inline-block` | Content | Settable | All sides | No |

### Position Values

```css
position: static;    /* Default — normal document flow */
position: relative;  /* Offset from normal position; still takes up space */
position: absolute;  /* Removed from flow; relative to nearest positioned ancestor */
position: fixed;     /* Relative to viewport; doesn't scroll */
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

- Only works on **positioned** elements (not `static`)
- Higher value = closer to the viewer
- New stacking context created by: `position` + `z-index`, `opacity < 1`, `transform`, `filter`

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

### Container Properties

```css
.container {
  display: flex;
  flex-direction: row;          /* row | row-reverse | column | column-reverse */
  flex-wrap: nowrap;            /* nowrap | wrap */
  justify-content: flex-start;  /* main axis: flex-start | center | space-between | space-around | space-evenly */
  align-items: stretch;         /* cross axis: stretch | flex-start | flex-end | center | baseline */
  gap: 16px;
}
```

### Item Properties

```css
.item {
  flex-grow: 1;     /* how much item grows (0 = no grow) */
  flex-shrink: 1;   /* how much item shrinks */
  flex-basis: auto; /* initial size before grow/shrink */
  flex: 1;          /* shorthand: 1 1 0% */
  align-self: center;
  order: 2;
}
```

### Common Patterns

```css
/* Center anything */
.center { display: flex; justify-content: center; align-items: center; }

/* Navbar */
.navbar { display: flex; justify-content: space-between; align-items: center; }

/* Sticky footer */
body { display: flex; flex-direction: column; min-height: 100vh; }
main { flex: 1; }
```

```
Main axis  ──────────────────────►  (justify-content)
           [Item 1] [Item 2] [Item 3]
Cross axis ↕  (align-items)
```

---

## CSS Grid

CSS Grid is a **two-dimensional** layout system (rows AND columns).

### Container Properties

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  grid-template-rows: auto 1fr auto;
  gap: 16px;
  grid-template-areas:
    "header header header"
    "sidebar main main"
    "footer footer footer";
}
```

### Item Properties

```css
.item {
  grid-column: 1 / 3;       /* span from line 1 to 3 */
  grid-column: 1 / span 2;  /* start at 1, span 2 columns */
  grid-row: 2 / 4;
  grid-area: header;        /* named area */
}
```

### Named Areas Example

```css
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar content"
    "footer footer";
  grid-template-columns: 250px 1fr;
  min-height: 100vh;
}
.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.content { grid-area: content; }
.footer  { grid-area: footer; }
```

### fr Unit & auto-fill vs auto-fit

`fr` = fraction of remaining space. `200px 1fr 2fr` → 200px fixed, rest split 1/3 and 2/3.

```css
/* auto-fill: creates empty columns to fill space */
/* auto-fit: collapses empty columns, items stretch */
grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
```

### Grid vs Flexbox

| | Flexbox | Grid |
|---|---|---|
| Layout type | 1D (row or column) | 2D (rows and columns) |
| Best for | Navbars, card rows, small components | Page layouts, complex grids |

---

## Responsive Design & Media Queries

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

Always include this — without it, mobile browsers zoom out to fit desktop content.

### Media Query Syntax (Mobile-First)

```css
/* Base: mobile */
.container { padding: 16px; }

@media (min-width: 768px) { .container { padding: 24px; } }   /* Tablet */
@media (min-width: 1024px) { .container { padding: 32px; } }  /* Desktop */
```

### Common Breakpoints

| Device | Breakpoint |
|---|---|
| Mobile | < 768px |
| Tablet | 768px – 1023px |
| Desktop | 1024px+ |

### Useful Media Features

```css
@media (prefers-color-scheme: dark) { body { background: #1a1a1a; color: white; } }
@media (prefers-reduced-motion: reduce) { * { animation-duration: 0.01ms !important; } }
@media (min-width: 768px) and (max-width: 1023px) { ... }
```

### Fluid Typography

```css
h1 { font-size: clamp(1.5rem, 5vw, 3rem); }
```

---

## Typography

### Font & Text Properties

```css
p {
  font-family: 'Inter', Arial, sans-serif;
  font-size: 1rem;       /* relative to root (preferred) */
  font-weight: 400;      /* 100–900 */
  line-height: 1.5;      /* unitless (preferred) */
  letter-spacing: 0.05em;
  text-align: left;
  text-decoration: underline;
  text-transform: uppercase;
  text-overflow: ellipsis; /* clips overflow text with ... */
  white-space: nowrap;
  overflow: hidden;
}
```

### rem vs em vs px

| Unit | Relative To | Use Case |
|---|---|---|
| `px` | Absolute | Borders, shadows |
| `em` | Parent font-size | Component spacing |
| `rem` | Root font-size | Base typography |
| `vw`/`vh` | Viewport | Hero sections |
| `ch` | Width of "0" | Readable line lengths (~60-75ch) |

### Web Fonts

```css
@font-face {
  font-family: 'MyFont';
  src: url('myfont.woff2') format('woff2');
  font-display: swap; /* prevents invisible text while loading */
}
```

---

## Colors & Backgrounds

```css
color: #ff6347;
color: rgb(255, 99, 71);
color: rgba(255, 99, 71, 0.8);
color: hsl(9, 100%, 64%);

background: url('bg.jpg') no-repeat center/cover;
background-size: cover;    /* fill container, may crop */
background-size: contain;  /* fit inside, may letterbox */

/* Gradients */
background: linear-gradient(to right, #ff6347, #4682b4);
background: radial-gradient(circle, #ff6347, #4682b4);
```

---

## Transitions & Animations

### Transitions

```css
.button {
  background: blue;
  transition: background 0.3s ease, transform 0.2s ease;
}
.button:hover { background: darkblue; transform: translateY(-2px); }
```

| Timing | Behavior |
|---|---|
| `ease` | Slow start, fast middle, slow end |
| `linear` | Constant speed |
| `ease-in-out` | Slow start and end |

### CSS Animations

```css
@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}

.card {
  animation: fadeInUp 0.5s ease-out 0.1s both;
  /* name duration timing delay fill-mode */
}
```

### Transform

```css
transform: translateX(50px);
transform: scale(1.2);
transform: rotate(45deg);
transform: translateX(50px) rotate(45deg) scale(1.2); /* chain multiple */
```

> **Interview Tip**: Use `transform` and `opacity` for animations — they are compositor-only (GPU) and avoid triggering reflow.

---

## CSS Variables (Custom Properties)

```css
:root {
  --color-primary: #3b82f6;
  --spacing-md: 16px;
  --border-radius: 4px;
}

.button {
  background: var(--color-primary);
  padding: var(--spacing-md);
  border-radius: var(--border-radius);
  color: var(--color-accent, #ff6347); /* fallback */
}
```

### Theming

```css
:root    { --bg: #fff; --text: #1a1a1a; }
[data-theme="dark"] { --bg: #1a1a1a; --text: #fff; }
body { background: var(--bg); color: var(--text); }
```

```javascript
document.documentElement.setAttribute('data-theme', 'dark');
document.documentElement.style.setProperty('--color-primary', '#ff6347');
```

> **Interview Tip**: CSS variables are live in the DOM and can be changed at runtime with JavaScript. They cascade and inherit — unlike Sass variables.

---

## Pseudo-classes & Pseudo-elements

### Pseudo-classes (`:`) — target elements in specific states

```css
a:hover { color: blue; }
a:focus { outline: 2px solid blue; }
button:disabled { opacity: 0.5; }
input:valid   { border-color: green; }
input:invalid { border-color: red; }

li:first-child     { font-weight: bold; }
li:last-child      { margin-bottom: 0; }
li:nth-child(odd)  { background: #f0f0f0; }
:not(.active)      { opacity: 0.5; }
:is(h1, h2, h3)    { font-weight: bold; }
:has(img)          { border: 1px solid #ccc; } /* parent selector */
```

### Pseudo-elements (`::`) — create virtual elements

```css
.button::before { content: '→ '; }
.required::after { content: ' *'; color: red; }
p::first-letter { font-size: 2em; float: left; }
::selection { background: #3b82f6; color: white; }
input::placeholder { color: #9ca3af; }
```

> **Interview Tip**: Pseudo-classes (`:`) target existing elements by state. Pseudo-elements (`::`) create virtual content.

---

## CSS Architecture & Methodologies

### BEM (Block Element Modifier)

```css
.card { }             /* Block */
.card__title { }      /* Element — double underscore */
.card__image { }
.card--featured { }   /* Modifier — double dash */
.card__title--large { }
```

```html
<div class="card card--featured">
  <h2 class="card__title card__title--large">Title</h2>
</div>
```

### Other Methodologies (know the names)

- **SMACSS**: Categorizes CSS into Base, Layout, Module, State, Theme
- **OOCSS**: Separates structure from skin — `.btn` (structure) + `.btn--primary` (skin)
- **Utility-First (Tailwind)**: Single-purpose classes like `flex items-center p-4`
- **CSS Modules**: Scoped class names per component (`.button` compiles to `.Button_button__xK3f2`)

---

## Performance & Best Practices

### Reflow vs Repaint vs Composite

| Operation | Cost | Triggered By |
|---|---|---|
| **Reflow (Layout)** | Most expensive | width, height, margin, position |
| **Repaint** | Expensive | color, background, visibility |
| **Composite** | Cheap (GPU) | transform, opacity |

```css
/* Avoid — triggers reflow */
.animate:hover { left: 100px; }

/* Prefer — compositor only */
.animate:hover { transform: translateX(100px); }
```

### Accessibility

```css
:focus { outline: none; }           /* BAD */
:focus-visible { outline: 2px solid blue; } /* GOOD */

/* Visually hidden but accessible */
.sr-only {
  position: absolute; width: 1px; height: 1px;
  padding: 0; margin: -1px; overflow: hidden;
  clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0;
}
```

### CSS Reset vs Normalize

```css
/* Hard Reset */
* { margin: 0; padding: 0; box-sizing: border-box; }

/* Normalize.css — preserves useful defaults, fixes browser bugs */
```

---

## Common Interview Questions

### Q: What is the difference between `visibility: hidden` and `display: none`?

| Property | Space Occupied | Accessible |
|---|---|---|
| `display: none` | No | No |
| `visibility: hidden` | Yes | No |
| `opacity: 0` | Yes | Yes (still clickable!) |

---

### Q: What is the difference between `absolute` and `fixed` positioning?

`absolute` is positioned relative to the nearest positioned ancestor and scrolls with the page. `fixed` is positioned relative to the viewport and stays put on scroll.

---

### Q: How do you center a div horizontally and vertically?

```css
/* Flexbox (recommended) */
.parent { display: flex; justify-content: center; align-items: center; }

/* Grid */
.parent { display: grid; place-items: center; }

/* Absolute + Transform */
.child { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); }
```

---

### Q: What is a stacking context?

A stacking context groups elements that are painted together. A new one is created by: `position` + non-auto `z-index`, `opacity < 1`, `transform`, `filter`, `isolation: isolate`, `will-change`.

---

### Q: What is `em` vs `rem`?

`em` is relative to the **parent** element's font-size; `rem` is relative to the **root** (`html`) font-size. `rem` is more predictable for global sizing.

---

### Q: What is the difference between `min-width`, `max-width`, and `width`?

`width` sets a fixed size; `max-width` caps the size (can be smaller); `min-width` sets a floor (can be larger). Common pattern: `width: 100%; max-width: 1200px; margin: 0 auto;`

---

### Q: How does `z-index` work?

Only works on positioned elements. Higher value = in front. `z-index` is scoped within its stacking context — a child with `z-index: 9999` cannot escape a parent stacking context with `z-index: 1`.

---

### Q: What is `object-fit`?

Controls how an `<img>` or `<video>` fills its container. `cover` fills and crops; `contain` fits inside with letterboxing; `fill` stretches (distorts).

---

## Quick Reference Cheat Sheet

### Flexbox

```css
/* Container */
display: flex;
flex-direction: row | column;
flex-wrap: nowrap | wrap;
justify-content: flex-start | center | space-between | space-around | space-evenly;
align-items: stretch | flex-start | flex-end | center | baseline;
gap: <value>;

/* Item */
flex: <grow> <shrink> <basis>;
align-self: auto | flex-start | flex-end | center | stretch;
order: 0;
```

### Grid

```css
/* Container */
display: grid;
grid-template-columns: repeat(3, 1fr);
grid-template-areas: "..." "...";
gap: <row> <col>;
place-items: <align> <justify>;

/* Item */
grid-column: 1 / 3;
grid-row: 1 / span 2;
grid-area: name;
```

### Positioning

```css
position: static;    /* default */
position: relative;  /* offset from self */
position: absolute;  /* relative to nearest positioned ancestor */
position: fixed;     /* relative to viewport */
position: sticky;    /* relative until scroll threshold */
```

### Units

```
px   → absolute pixels
em   → relative to parent font-size
rem  → relative to root font-size
%    → relative to parent dimension
vw/vh → % of viewport width/height
fr   → fraction of grid container
clamp(min, val, max) → fluid value with bounds
```

### Common Patterns

```css
body { display: grid; grid-template-rows: auto 1fr auto; min-height: 100vh; }
.center { display: grid; place-items: center; }
.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(250px, 1fr)); gap: 1rem; }
.truncate { overflow: hidden; white-space: nowrap; text-overflow: ellipsis; }
.video { aspect-ratio: 16 / 9; }
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
- [ ] `z-index` and stacking context
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
- [ ] Centering techniques
- [ ] `object-fit` for images

---

*Last Updated: 2026-06-18*
