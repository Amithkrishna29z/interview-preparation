# JavaScript Interview Questions & Study Guide

## Overview

JavaScript is the language of the web. This guide covers core JS fundamentals every frontend and full-stack developer is expected to know — from execution mechanics to modern ES6+ features.

---

## Table of Contents

1. [Core Fundamentals](#core-fundamentals)
2. [Execution Context & Call Stack](#execution-context--call-stack)
3. [Scope & Closures](#scope--closures)
4. [Hoisting](#hoisting)
5. [this Keyword](#this-keyword)
6. [Prototype & Inheritance](#prototype--inheritance)
7. [Asynchronous JavaScript](#asynchronous-javascript)
8. [Event Loop](#event-loop)
9. [ES6+ Features](#es6-features)
10. [Arrays & Functional Methods](#arrays--functional-methods)
11. [Objects & Destructuring](#objects--destructuring)
12. [Error Handling](#error-handling)
13. [Modules](#modules)
14. [Memory & Performance](#memory--performance)
15. [Common Interview Questions](#common-interview-questions)
16. [Coding Challenges](#coding-challenges)
17. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Core Fundamentals

### Data Types

```javascript
// Primitives (stored by value)
let num  = 42;        let str  = "hello";   let bool = true;
let undef = undefined; let nil = null;      let sym  = Symbol("id");
let big  = 9007199254740991n; // BigInt

// Reference types (stored by reference)
let obj = { name: "Alice" };  let arr = [1, 2, 3];
```

### typeof Operator

```javascript
typeof 42           // "number"
typeof "hello"      // "string"
typeof undefined    // "undefined"
typeof null         // "object"  ← famous bug
typeof []           // "object"  ← use Array.isArray() instead
typeof function(){} // "function"
```

### == vs ===

```javascript
// == coerces types;  === does not
0 == false        // true     |  0 === false       // false
null == undefined // true     |  null === undefined // false
1 == "1"          // true     |  1 === "1"         // false
// Always use === in production code
```

### Truthy & Falsy

```javascript
// Falsy (only 6): false, 0, "", null, undefined, NaN
// Everything else is truthy — including "0", [], {}
```

---

## Execution Context & Call Stack

Every time JS runs code it creates an **Execution Context** with two phases:
1. **Memory phase** — variables and functions are hoisted
2. **Execution phase** — code runs line by line

```javascript
function greet(name) { return `Hello, ${name}`; }
function main() { console.log(greet("Alice")); }
main();
// Stack: [main] → [greet, main] → [main] → []
```

> **Stack Overflow**: caused by infinite recursion — the call stack exceeds its limit.

---

## Scope & Closures

### var vs let vs const

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Yes (undefined) | Yes (TDZ) | Yes (TDZ) |
| Re-declare | Yes | No | No |
| Re-assign | Yes | Yes | No |

```javascript
function example() {
  console.log(x); // undefined (var hoisted)
  var x = 5;
  if (true) { var x = 10; } // same variable!
  console.log(x); // 10

  // let y = 5; — new block-scoped binding per block
}
```

### Closures

A closure is a function that **remembers variables from its outer scope** even after the outer function has returned.

```javascript
function makeCounter() {
  let count = 0;
  return function() { return ++count; };
}
const counter = makeCounter();
counter(); // 1
counter(); // 2  — count lives in memory as long as counter exists
```

**Classic var-in-loop bug:**

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // 3, 3, 3 — all share same i
}
// Fix: use let (new binding per iteration) → 0, 1, 2
```

**Practical uses:** data privacy, factory functions, memoization, event handlers with state.

---

## Hoisting

Variables and function declarations are moved to the top of their scope during the memory phase.

```javascript
greet();        // "Hello!" — function declarations fully hoisted
function greet() { console.log("Hello!"); }

console.log(x); // undefined — var hoisted but not initialized
var x = 5;

console.log(y); // ReferenceError: TDZ
let y = 5;

sayHi();        // TypeError: sayHi is not a function
var sayHi = function() { console.log("Hi!"); }; // var hoisted as undefined
```

---

## this Keyword

`this` is determined by **how** a function is called, not where it is defined.

```javascript
// Object method — this = calling object
const person = { name: "Alice", greet() { console.log(this.name); } };
person.greet(); // "Alice"

// Arrow function — this is lexically inherited (no own this)
const obj = {
  name: "Alice",
  greetMethod() {
    const inner = () => console.log(this.name); // "Alice" — inherits
    inner();
  }
};

// Constructor — this = new object
function Person(name) { this.name = name; }
const alice = new Person("Alice");

// Explicit binding
greet.call(user);          // immediate, individual args
greet.apply(user, [args]); // immediate, array of args
greet.bind(user);          // returns new fn, deferred
```

### call vs apply vs bind

| Method | Calls Immediately | Arguments |
|---|---|---|
| `call(ctx, a, b)` | Yes | Individual |
| `apply(ctx, [a, b])` | Yes | Array |
| `bind(ctx, a, b)` | No — returns fn | Individual |

---

## Prototype & Inheritance

Every object has a `__proto__` link forming a chain up to `Object.prototype → null`. Property lookup walks this chain.

```javascript
const animal = { breathe() { return "breathing"; } };
const dog = Object.create(animal); // dog.__proto__ = animal
dog.bark = function() { return "woof"; };
dog.breathe(); // found via prototype chain
```

### ES6 Classes (syntactic sugar over prototypes)

```javascript
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
  static create(name) { return new Animal(name); }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // must call before using this
    this.breed = breed;
  }
  speak() { return `${super.speak()} — Woof!`; }
}

const rex = new Dog("Rex", "Labrador");
rex instanceof Dog;    // true
rex instanceof Animal; // true
```

---

## Asynchronous JavaScript

### Callbacks

```javascript
// Error-first callback (Node.js style)
fetchData(1, (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});
```

### Promises

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => resolve("success"), 1000);
});
// States: pending → fulfilled | rejected (immutable once settled)
promise
  .then(result => console.log(result))
  .catch(err => console.error(err))
  .finally(() => console.log("done"));
```

### Promise Combinators

```javascript
Promise.all([p1, p2, p3])        // all succeed, or fail on first rejection
Promise.allSettled([p1, p2, p3]) // wait for all, collect results regardless
Promise.race([p1, p2, p3])       // first to settle (fulfilled or rejected)
Promise.any([p1, p2, p3])        // first to fulfill
```

### Async / Await

```javascript
async function loadUser(id) {
  try {
    const user   = await getUser(id);
    const orders = await getOrders(user.id);
    return orders;
  } catch (err) { throw err; }
}

// Run independent calls in parallel
async function loadAll() {
  const [users, products] = await Promise.all([fetchUsers(), fetchProducts()]);
  return { users, products };
}
```

> **Interview Tip**: `async` always returns a Promise. Forgetting `await` returns a Promise instead of the resolved value.

---

## Event Loop

JavaScript is **single-threaded** but handles async via the event loop.

```
Call Stack  ←  Event Loop (stack empty? dequeue next task)
               ↑
  Microtask Queue (higher priority)   Macrotask Queue (lower priority)
  · Promise .then                     · setTimeout / setInterval
  · queueMicrotask                    · I/O callbacks
```

```javascript
console.log("1 - sync");
setTimeout(() => console.log("2 - macrotask"), 0);
Promise.resolve().then(() => console.log("3 - microtask"));
console.log("4 - sync");
// Output: 1 - sync → 4 - sync → 3 - microtask → 2 - macrotask
```

All microtasks drain **before** the next macrotask runs.

---

## ES6+ Features

### Destructuring

```javascript
const [a, b, ...rest] = [1, 2, 3, 4];        // a=1, b=2, rest=[3,4]
const { name, age, city = "Unknown" } = user; // default value
const { name: fullName } = user;              // rename
```

### Spread & Rest

```javascript
const arr2 = [...arr1, 4, 5];          // spread — expand
const obj2 = { ...obj1, newProp: "v" }; // shallow clone

function sum(...nums) { return nums.reduce((a, b) => a + b, 0); } // rest
```

### Optional Chaining & Nullish Coalescing

```javascript
const city = user?.address?.city;  // undefined instead of TypeError
const name = user.name ?? "Anon";  // defaults only for null/undefined
const count = 0 ?? 10;  // 0   (0 is not null/undefined)
const count2 = 0 || 10; // 10  (0 is falsy — different!)
```

### Map & Set

```javascript
const map = new Map([["a", 1]]);
map.set("b", 2); map.get("a"); map.has("b"); map.size;

const set = new Set([1, 2, 2, 3]); // {1, 2, 3}
const unique = [...new Set(arr)];   // remove duplicates
```

### Generators

```javascript
function* range(start, end) {
  for (let i = start; i <= end; i++) yield i;
}
for (const n of range(1, 3)) console.log(n); // 1 2 3
```

---

## Arrays & Functional Methods

```javascript
const nums = [1, 2, 3, 4, 5];
nums.map(x => x * 2);              // [2,4,6,8,10] — transform
nums.filter(x => x % 2 === 0);    // [2,4] — keep matching
nums.reduce((acc, x) => acc + x, 0); // 15 — accumulate
nums.find(x => x > 3);            // 4 — first match
nums.some(x => x > 4);            // true — at least one
nums.every(x => x > 0);           // true — all match
nums.flatMap(x => [x, x * 2]);    // map + flat(1)
```

| Method | Returns | Mutates |
|---|---|---|
| `map`, `filter`, `slice`, `concat` | New array | No |
| `reduce` | Single value | No |
| `forEach` | undefined | No |
| `sort`, `splice` | Same array / removed items | **Yes** |

> **Interview Tip**: `sort()` is lexicographic by default. Use `[...arr].sort((a, b) => a - b)` for numeric sort.

---

## Objects & Destructuring

```javascript
const name = "Alice", age = 30;
const person = { name, age }; // shorthand

const key = "dynamic";
const obj = { [key]: "value" }; // computed property

Object.keys(obj);              // own enumerable keys
Object.values(obj);            // own enumerable values
Object.entries(obj);           // [key, value] pairs
Object.assign({}, src1, src2); // shallow merge
Object.freeze(obj);            // prevent mutation (shallow)
structuredClone(obj);          // deep clone (ES2022)
```

---

## Error Handling

```javascript
try {
  JSON.parse("invalid");
} catch (err) {
  console.log(err.name, err.message); // "SyntaxError", description
} finally {
  // always runs — cleanup here
}

// Custom error
class AppError extends Error {
  constructor(message, code) {
    super(message);
    this.name = "AppError";
    this.code = code;
  }
}

// Async
async function fetchData() {
  const res = await fetch("/api");
  if (!res.ok) throw new AppError(`HTTP ${res.status}`, res.status);
  return res.json();
}
```

---

## Modules

```javascript
// Named exports
export const PI = 3.14159;
export function add(a, b) { return a + b; }

// Default export (one per file)
export default class App { }

// Imports
import { add, PI } from "./math.js";
import { add as sum } from "./math.js";
import App from "./App.js";

// Dynamic import (lazy loading)
const module = await import("./heavy.js");
```

---

## Memory & Performance

### Common Memory Leaks

```javascript
// 1. Forgotten timers — data never freed
setInterval(() => process(data), 1000);

// 2. Detached DOM nodes — el removed from DOM but still referenced in JS
let el = document.getElementById("btn");
el.remove(); // el still holds reference

// 3. Closures over large data
function outer() {
  const huge = new Array(1000000).fill(0);
  return () => "done"; // huge never GC'd
}
```

### Debounce & Throttle

```javascript
// Debounce — wait until user STOPS for `delay` ms, then fire once
function debounce(fn, delay) {
  let timer;
  return (...args) => { clearTimeout(timer); timer = setTimeout(() => fn(...args), delay); };
}

// Throttle — fire at most once per `limit` ms
function throttle(fn, limit) {
  let inThrottle = false;
  return (...args) => {
    if (!inThrottle) { fn(...args); inThrottle = true; setTimeout(() => inThrottle = false, limit); }
  };
}
```

---

## Common Interview Questions

### Q: What is the difference between `null` and `undefined`?

`undefined` means a variable was declared but never assigned. `null` is an intentional empty value, explicitly set. `typeof null` returns `"object"` — a historical JS bug. `null == undefined` is true (loose), `null === undefined` is false (strict).

---

### Q: Explain `==` vs `===`.

`==` performs type coercion before comparing; `===` compares value AND type with no coercion. Always prefer `===` to avoid unexpected bugs.

---

### Q: What is event delegation?

```javascript
// One listener on parent instead of n listeners on n children
document.querySelector("ul").addEventListener("click", (e) => {
  if (e.target.matches("li")) console.log("Clicked:", e.target.textContent);
});
// Works for dynamically added items — events bubble up to the parent
```

---

### Q: Regular function vs arrow function?

| Feature | Regular Function | Arrow Function |
|---|---|---|
| `this` | Dynamic (call-site) | Lexical (enclosing scope) |
| `arguments` | Available | Not available |
| `new` | Can be constructor | Cannot |

---

### Q: What is `NaN` and how do you check for it?

```javascript
NaN === NaN            // false — NaN is not equal to itself
Number.isNaN(NaN)      // true  — use this
isNaN("hello")         // true  — unreliable (coerces first)
```

---

### Q: What is the Temporal Dead Zone (TDZ)?

The TDZ is the period between when a `let`/`const` variable is hoisted and when it is initialized. Accessing it during this period throws a `ReferenceError`.

---

## Coding Challenges

### Implement debounce

```javascript
function debounce(fn, delay) {
  let timer = null;
  return function(...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
```

### Flatten nested array without flat()

```javascript
function flatten(arr) {
  return arr.reduce((acc, val) =>
    Array.isArray(val) ? acc.concat(flatten(val)) : acc.concat(val), []);
}
flatten([1, [2, [3, [4]]]]); // [1, 2, 3, 4]
```

### Implement Promise.all

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = []; let settled = 0;
    if (!promises.length) return resolve([]);
    promises.forEach((p, i) => {
      Promise.resolve(p)
        .then(val => { results[i] = val; if (++settled === promises.length) resolve(results); })
        .catch(reject);
    });
  });
}
```

### Curry a function

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) return fn(...args);
    return (...more) => curried(...args, ...more);
  };
}
const add = curry((a, b, c) => a + b + c);
add(1)(2)(3); // 6
```

### Deep clone without structuredClone

```javascript
function deepClone(obj) {
  if (obj === null || typeof obj !== "object") return obj;
  if (Array.isArray(obj)) return obj.map(deepClone);
  return Object.fromEntries(Object.entries(obj).map(([k, v]) => [k, deepClone(v)]));
}
```

---

## Quick Reference Cheat Sheet

```
Primitives:  number, string, boolean, null, undefined, symbol, bigint
typeof null: "object" (bug)
Falsy:       false, 0, "", null, undefined, NaN
Hoisting:    var → undefined | function → full | let/const → TDZ
Scope:       var → function | let/const → block
Closure:     inner function retains outer scope after outer returns
this:        global | obj method | constructor | call/apply/bind
Arrow fn:    lexical this, no arguments, no new
Prototype:   lookup walks __proto__ chain → Object.prototype → null
Promise:     pending → fulfilled | rejected (immutable once settled)
Microtask:   Promise .then — drains before next macrotask
Macrotask:   setTimeout, setInterval, I/O
new keyword: {} → set __proto__ → run constructor → return object
== vs ===:   coercion vs strict
?? vs ||:    null/undefined only vs any falsy
```

---

*Last Updated: 2026-06-18*
