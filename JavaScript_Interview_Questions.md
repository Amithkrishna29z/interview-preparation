# JavaScript Interview Questions & Study Guide

## Overview

JavaScript is the language of the web. This guide covers core JS fundamentals that every frontend and full-stack developer is expected to know — from execution mechanics to modern ES6+ features.

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

---

## Core Fundamentals

### Data Types

```javascript
// Primitive types (immutable, stored by value)
let num    = 42;                // Number
let str    = "hello";           // String
let bool   = true;              // Boolean
let undef  = undefined;         // Undefined
let nil    = null;              // Null
let sym    = Symbol("id");      // Symbol (ES6)
let big    = 9007199254740991n; // BigInt (ES2020)

// Reference type (mutable, stored by reference)
let obj    = { name: "Alice" };
let arr    = [1, 2, 3];
let fn     = function() {};
```

### typeof Operator

```javascript
typeof 42            // "number"
typeof "hello"       // "string"
typeof true          // "boolean"
typeof undefined     // "undefined"
typeof null          // "object"   ← famous bug, null is NOT an object
typeof Symbol()      // "symbol"
typeof function(){}  // "function"
typeof {}            // "object"
typeof []            // "object"   ← use Array.isArray() instead
```

### == vs ===

```javascript
// == (loose equality) — coerces types before comparing
0 == false        // true
"" == false       // true
null == undefined // true
1 == "1"          // true

// === (strict equality) — no type coercion
0 === false       // false
1 === "1"         // false
null === undefined // false

// Always use === in production code
```

### Truthy & Falsy Values

```javascript
// Falsy values (only 6)
false, 0, "", null, undefined, NaN

// Everything else is truthy
"0"   // truthy (non-empty string)
[]    // truthy (empty array)
{}    // truthy (empty object)
-1    // truthy (non-zero number)
```

---

## Execution Context & Call Stack

### Execution Context

Every time JavaScript runs code, it creates an **Execution Context**:

```
Global Execution Context
  ├── this = window (browser) / global (Node.js)
  ├── Memory Phase (hoisting): variables and functions stored
  └── Execution Phase: code runs line by line

Function Execution Context (created on each call)
  ├── this = depends on how function is called
  ├── arguments object
  ├── Memory Phase: local variables hoisted
  └── Execution Phase: function body runs
```

### Call Stack

```javascript
function greet(name) {
  return `Hello, ${name}`;
}

function main() {
  let result = greet("Alice");
  console.log(result);
}

main();

/*
Call Stack progression:
  [main]           ← pushed when main() called
  [greet, main]    ← pushed when greet() called inside main
  [main]           ← greet() returned, popped
  []               ← main() returned, popped
*/
```

> **Stack Overflow**: Occurs when the call stack exceeds its limit — usually caused by infinite recursion.

---

## Scope & Closures

### Scope Types

```javascript
// Global scope
let globalVar = "I'm global";

function outer() {
  // Function scope
  let outerVar = "I'm in outer";

  function inner() {
    // Nested function scope
    let innerVar = "I'm in inner";
    console.log(outerVar);  // accessible via lexical scope
    console.log(globalVar); // accessible
  }
  // console.log(innerVar); // ReferenceError
}
```

### var vs let vs const

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Yes (undefined) | Yes (TDZ) | Yes (TDZ) |
| Re-declare | Yes | No | No |
| Re-assign | Yes | Yes | No |
| Global object property | Yes | No | No |

```javascript
// var — function scoped, hoisted to undefined
function example() {
  console.log(x); // undefined (not ReferenceError)
  var x = 5;
  if (true) {
    var x = 10; // same variable! overwrites
  }
  console.log(x); // 10
}

// let — block scoped, Temporal Dead Zone
function example2() {
  // console.log(y); // ReferenceError: TDZ
  let y = 5;
  if (true) {
    let y = 10; // different block-scoped variable
  }
  console.log(y); // 5
}
```

### Closures

A closure is a function that **remembers variables from its outer scope** even after the outer function has returned.

```javascript
function makeCounter() {
  let count = 0;        // outer variable

  return function() {   // inner function closes over count
    count++;
    return count;
  };
}

const counter = makeCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
// count lives in memory as long as counter references it
```

**Classic closure bug with var in loops:**

```javascript
// Bug: all callbacks share the same i (var is function-scoped)
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // 3, 3, 3
}

// Fix 1: use let (block-scoped, new binding per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // 0, 1, 2
}

// Fix 2: IIFE to capture current i
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100); // 0, 1, 2
  })(i);
}
```

**Practical closure uses:**
- Data privacy / encapsulation
- Factory functions
- Memoization / caching
- Event handlers with state
- Partial application / currying

---

## Hoisting

Variables and function declarations are moved to the top of their scope during the **memory phase** before code executes.

```javascript
// Function declarations are FULLY hoisted
greet(); // "Hello!" — works before declaration
function greet() { console.log("Hello!"); }

// var is hoisted but initialized to undefined
console.log(x); // undefined (not ReferenceError)
var x = 5;

// let/const are hoisted but stay in the Temporal Dead Zone
console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 5;

// Function expressions are NOT fully hoisted
sayHi(); // TypeError: sayHi is not a function
var sayHi = function() { console.log("Hi!"); };
// var sayHi is hoisted as undefined, calling undefined() = TypeError
```

---

## this Keyword

`this` refers to the **object that is executing the current function**. Its value is determined by **how** the function is called, not where it is defined.

```javascript
// 1. Global context
console.log(this); // window (browser) / global (Node.js)

// 2. Object method — this = the calling object
const person = {
  name: "Alice",
  greet() { console.log(this.name); } // "Alice"
};
person.greet();

// 3. Regular function (non-strict) — this = global
function show() { console.log(this); } // window

// 4. Arrow function — this is lexically inherited
const obj = {
  name: "Alice",
  greetArrow: () => console.log(this.name),  // undefined (arrow, no own this)
  greetMethod() {
    const inner = () => console.log(this.name); // "Alice" (inherits from greetMethod)
    inner();
  }
};

// 5. Constructor — this = newly created object
function Person(name) { this.name = name; }
const alice = new Person("Alice");

// 6. Explicit binding
function greet() { console.log(this.name); }
const user = { name: "Bob" };
greet.call(user);             // "Bob" — immediate call
greet.apply(user, []);        // "Bob" — immediate call, args as array
const bound = greet.bind(user);
bound();                      // "Bob" — deferred call
```

### call vs apply vs bind

| Method | Calls Immediately | Arguments Format |
|---|---|---|
| `call(ctx, a, b)` | Yes | Individual args |
| `apply(ctx, [a, b])` | Yes | Array of args |
| `bind(ctx, a, b)` | No — returns new fn | Individual args (partial application) |

---

## Prototype & Inheritance

### Prototype Chain

```javascript
const animal = {
  breathe() { return "breathing"; }
};

const dog = Object.create(animal); // dog.__proto__ = animal
dog.bark = function() { return "woof"; };

dog.bark();    // own property
dog.breathe(); // found via prototype chain
// chain: dog → animal → Object.prototype → null
```

### Constructor Functions

```javascript
function Animal(name) {
  this.name = name; // own property per instance
}
Animal.prototype.speak = function() { // shared across all instances
  return `${this.name} makes a sound`;
};

const dog = new Animal("Rex");
// new: 1) creates {} 2) sets __proto__ 3) runs constructor 4) returns object
```

### ES6 Classes (syntactic sugar over prototypes)

```javascript
class Animal {
  #name; // private field (ES2022)

  constructor(name) {
    this.#name = name;
  }

  speak() { return `${this.#name} makes a sound`; }

  get name() { return this.#name; }

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
// Error-first callback convention (Node.js style)
function fetchData(id, callback) {
  setTimeout(() => {
    if (!id) callback(new Error("No id"), null);
    else callback(null, { id, data: "result" });
  }, 1000);
}

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

// Promise states: pending → fulfilled | rejected (immutable once settled)
promise
  .then(result => console.log(result))  // "success"
  .catch(err => console.error(err))
  .finally(() => console.log("done"));  // always runs
```

### Promise Combinators

```javascript
// All succeed or fail together
Promise.all([p1, p2, p3])
  .then(([r1, r2, r3]) => { /* all fulfilled */ })
  .catch(err => { /* first rejection */ });

// Wait for all, collect results regardless of outcome
Promise.allSettled([p1, p2, p3])
  .then(results => results.forEach(r => {
    if (r.status === "fulfilled") console.log(r.value);
    if (r.status === "rejected")  console.log(r.reason);
  }));

// First to settle (fulfilled or rejected) wins
Promise.race([p1, p2, p3]).then(first => { });

// First to FULFILL wins (ignores rejections unless all reject)
Promise.any([p1, p2, p3]).then(first => { });
```

### Async / Await

```javascript
async function loadUser(id) {
  try {
    const user    = await getUser(id);       // pauses here
    const orders  = await getOrders(user.id);
    return orders;
  } catch (err) {
    console.error(err);
    throw err;
  }
}

// Run in parallel — don't await sequentially when independent
async function loadAll() {
  const [users, products] = await Promise.all([
    fetchUsers(),
    fetchProducts()
  ]);
  return { users, products };
}
```

> **Interview Tip**: `async` always returns a Promise. Forgetting `await` is a common bug — the function returns a Promise instead of the resolved value.

---

## Event Loop

JavaScript is **single-threaded** but handles async operations via the event loop.

```
┌─────────────────────┐
│     Call Stack       │  ← JS executes here (one at a time)
└─────────────────────┘
          ▲
          │  Event Loop checks: is stack empty?
          │  If yes → dequeue next task
          │
┌─────────────────────┐    ┌──────────────────────────┐
│   Microtask Queue   │    │     Macrotask Queue       │
│  (higher priority)  │    │   (lower priority)        │
│  · Promise .then    │    │  · setTimeout             │
│  · queueMicrotask   │    │  · setInterval            │
│  · MutationObserver │    │  · I/O callbacks          │
└─────────────────────┘    │  · UI render events       │
                            └──────────────────────────┘
```

### Execution Order

```javascript
console.log("1 - sync");

setTimeout(() => console.log("2 - macrotask"), 0);

Promise.resolve().then(() => console.log("3 - microtask"));

queueMicrotask(() => console.log("4 - microtask"));

console.log("5 - sync");

// Output:
// 1 - sync
// 5 - sync
// 3 - microtask   ← all microtasks drain before next macrotask
// 4 - microtask
// 2 - macrotask
```

| Queue | Examples | Priority |
|---|---|---|
| Microtask | Promise `.then`, `queueMicrotask` | Runs before next macrotask |
| Macrotask | `setTimeout`, `setInterval`, I/O | Runs after all microtasks |

---

## ES6+ Features

### Destructuring

```javascript
// Array destructuring
const [a, b, ...rest] = [1, 2, 3, 4, 5]; // a=1, b=2, rest=[3,4,5]
const [, second] = [10, 20];              // skip first → second=20

// Object destructuring
const { name, age, city = "Unknown" } = { name: "Alice", age: 30 };
const { name: fullName } = { name: "Alice" }; // rename

// Nested
const { address: { street } } = { address: { street: "Main St" } };
```

### Spread & Rest

```javascript
// Spread — expand iterables
const arr2 = [...arr1, 4, 5];
const obj2 = { ...obj1, newProp: "val" }; // shallow clone

// Rest — collect remaining into array
function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}
```

### Optional Chaining & Nullish Coalescing

```javascript
const city = user?.address?.city;    // undefined instead of TypeError
const fn   = obj?.method?.();        // safe method call

// ?? — defaults only for null/undefined (not 0 or "")
const name  = user.name ?? "Anonymous";
const count = 0 ?? 10;   // 0 (0 is not null/undefined)
const count2 = 0 || 10;  // 10 (0 is falsy — different behavior!)
```

### Map & Set

```javascript
// Map — key-value, any type as key, insertion-order preserved
const map = new Map([["a", 1], ["b", 2]]);
map.set("c", 3);
map.get("a");         // 1
map.has("b");         // true
map.size;             // 3
for (const [k, v] of map) { }

// Set — unique values
const set = new Set([1, 2, 2, 3, 3]);  // {1, 2, 3}
set.add(4); set.delete(1); set.has(2); set.size;
// Remove duplicates from array
const unique = [...new Set(arr)];
```

### Generators

```javascript
function* range(start, end) {
  for (let i = start; i <= end; i++) yield i;
}

const gen = range(1, 3);
gen.next(); // { value: 1, done: false }
gen.next(); // { value: 2, done: false }
gen.next(); // { value: 3, done: false }
gen.next(); // { value: undefined, done: true }

for (const n of range(1, 5)) console.log(n); // 1 2 3 4 5
```

---

## Arrays & Functional Methods

```javascript
const nums = [1, 2, 3, 4, 5];

nums.map(x => x * 2);             // [2,4,6,8,10] — transform
nums.filter(x => x % 2 === 0);   // [2,4] — keep matching
nums.reduce((acc, x) => acc + x, 0); // 15 — accumulate
nums.find(x => x > 3);           // 4 — first match
nums.findIndex(x => x > 3);      // 3 — index of first match
nums.some(x => x > 4);           // true — at least one
nums.every(x => x > 0);          // true — all match
nums.flat();                      // flatten one level
nums.flatMap(x => [x, x * 2]);   // map + flat(1)
nums.includes(3);                 // true
```

| Method | Returns | Mutates |
|---|---|---|
| `map` | New array | No |
| `filter` | New array | No |
| `reduce` | Single value | No |
| `forEach` | undefined | No |
| `sort` | Same array | **Yes** |
| `splice` | Removed items | **Yes** |
| `slice` | New array | No |
| `concat` | New array | No |

> **Interview Tip**: `sort()` mutates in-place and sorts lexicographically by default. Use `[...arr].sort((a, b) => a - b)` for numeric sort.

---

## Objects & Destructuring

```javascript
// Property shorthand
const name = "Alice", age = 30;
const person = { name, age }; // { name: "Alice", age: 30 }

// Computed properties
const key = "dynamic";
const obj = { [key]: "value" };

// Object utility methods
Object.keys(obj);             // own enumerable keys
Object.values(obj);           // own enumerable values
Object.entries(obj);          // [key, value] pairs
Object.assign({}, src1, src2); // shallow merge
Object.freeze(obj);           // prevent mutation (shallow)
Object.fromEntries(entries);  // entries array → object
structuredClone(obj);         // deep clone (ES2022)
```

---

## Error Handling

```javascript
try {
  JSON.parse("invalid");
} catch (err) {
  console.log(err.name);    // "SyntaxError"
  console.log(err.message); // description
  console.log(err.stack);   // stack trace
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

// Async error handling
async function fetchData() {
  try {
    const res = await fetch("/api");
    if (!res.ok) throw new AppError(`HTTP ${res.status}`, res.status);
    return await res.json();
  } catch (err) {
    console.error(err);
    throw err;
  }
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
import * as MathUtils from "./math.js";

// Dynamic import (lazy loading)
const module = await import("./heavy.js");
```

---

## Memory & Performance

### Common Memory Leaks

```javascript
// 1. Forgotten timers
const data = getHeavyData();
setInterval(() => process(data), 1000); // data never freed

// 2. Detached DOM nodes
let el = document.getElementById("btn");
el.remove();
// el still holds a reference — not GC'd

// 3. Closures over large data
function outer() {
  const huge = new Array(1000000).fill(0);
  return () => "done"; // huge stays in memory
}
```

### Debounce & Throttle

```javascript
// Debounce — execute only after user STOPS triggering for `delay` ms
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

// Throttle — execute at most once per `limit` ms
function throttle(fn, limit) {
  let inThrottle = false;
  return (...args) => {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}
```

---

## Common Interview Questions

### Q: What is the difference between `null` and `undefined`?

```javascript
undefined  // variable declared but never assigned
null       // intentional empty value, explicitly set

typeof undefined  // "undefined"
typeof null       // "object" (historical bug in JS)
null == undefined  // true (loose)
null === undefined // false (strict)
```

---

### Q: Explain the difference between `==` and `===`.

`==` performs type coercion before comparing. `===` compares value AND type with no coercion. Always prefer `===` to avoid unexpected coercion bugs.

---

### Q: What is event delegation?

```javascript
// Instead of n listeners on n children, one listener on parent
document.querySelector("ul").addEventListener("click", (e) => {
  if (e.target.matches("li")) {
    console.log("Clicked:", e.target.textContent);
  }
});
// Works for dynamically added items too — event bubbles up to ul
```

---

### Q: What is the difference between a regular function and an arrow function?

| Feature | Regular Function | Arrow Function |
|---|---|---|
| `this` | Dynamic (call-site) | Lexical (enclosing scope) |
| `arguments` | Available | Not available |
| `new` | Can be constructor | Cannot |
| `prototype` | Has it | Does not |

---

### Q: What is `NaN` and how do you check for it?

```javascript
NaN === NaN            // false — NaN is not equal to itself
Number.isNaN(NaN)      // true — strict check
Number.isNaN("hello")  // false — "hello" is not NaN (it's a string)
isNaN("hello")         // true — coerces first, unreliable
```

---

### Q: What is the Temporal Dead Zone (TDZ)?

The TDZ is the period between when a `let`/`const` variable is hoisted and when it is initialized. Accessing the variable during this period throws a `ReferenceError`. This encourages declaring variables before using them.

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
    const results = [];
    let settled = 0;
    if (!promises.length) return resolve([]);
    promises.forEach((p, i) => {
      Promise.resolve(p)
        .then(val => {
          results[i] = val;
          if (++settled === promises.length) resolve(results);
        })
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
add(1, 2)(3); // 6
```

### Deep clone without structuredClone

```javascript
function deepClone(obj) {
  if (obj === null || typeof obj !== "object") return obj;
  if (Array.isArray(obj)) return obj.map(deepClone);
  return Object.fromEntries(
    Object.entries(obj).map(([k, v]) => [k, deepClone(v)])
  );
}
```

---

## Quick Reference Cheat Sheet

```
Primitives:    number, string, boolean, null, undefined, symbol, bigint
typeof null:   "object" (bug — null is not an object)
Falsy:         false, 0, "", null, undefined, NaN
Hoisting:      var → undefined | function → full | let/const → TDZ
Scope:         var → function | let/const → block
Closure:       inner function retains access to outer scope after return
this:          global | obj method | constructor | explicit (call/apply/bind)
Arrow fn:      lexical this, no arguments, no new, no prototype
Prototype:     property lookup walks __proto__ chain to Object.prototype → null
Promise:       pending → fulfilled | rejected (immutable once settled)
Microtask:     Promise .then — runs before next macrotask
Macrotask:     setTimeout, setInterval, I/O
new keyword:   {} → set __proto__ → run constructor → return object
== vs ===:     coercion vs strict
?? vs ||:      null/undefined only vs any falsy
```

---

*Last Updated: 2026-06-04*
