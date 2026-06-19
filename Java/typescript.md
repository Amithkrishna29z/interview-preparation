# TypeScript: From Beginner to Advanced

> A comprehensive guide covering core TypeScript concepts with practical examples.

---

## Table of Contents

1. [What is TypeScript?](#1-what-is-typescript)
2. [Basic Types](#2-basic-types)
3. [Type Inference](#3-type-inference)
4. [Functions](#4-functions)
5. [Interfaces](#5-interfaces)
6. [Type Aliases](#6-type-aliases)
7. [Union & Intersection Types](#7-union--intersection-types)
8. [Literal Types](#8-literal-types)
9. [Enums](#9-enums)
10. [Arrays & Tuples](#10-arrays--tuples)
11. [Classes](#11-classes)
12. [Generics](#12-generics)
13. [Type Narrowing & Guards](#13-type-narrowing--guards)
14. [Utility Types](#14-utility-types)

---

## 1. What is TypeScript?

TypeScript is a **strongly typed superset of JavaScript** developed by Microsoft. It compiles to plain JavaScript and adds static typing, interfaces, and generics.

**Why TypeScript?**
- Catches errors at **compile time** instead of runtime
- Better **IDE autocomplete & refactoring**
- Makes large codebases more **maintainable**

```bash
npm install -g typescript
tsc hello.ts       # Compile
tsc --watch        # Watch mode
```

---

## 2. Basic Types

```typescript
let name: string = "Alice";
let age: number = 30;
let isActive: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;

let anything: any = "avoid this";  // Opt-out of type checking
let unknown: unknown = 42;         // Like any, but must narrow before use
let neverReturns: never;           // Function that never returns
let voidFn: void;                  // Function that returns nothing

let sym: symbol = Symbol("id");
let bigNum: bigint = 9007199254740991n;
```

---

## 3. Type Inference

TypeScript infers types automatically — you don't always need annotations.

```typescript
let city = "New York";  // inferred as string
let count = 42;         // inferred as number

function add(a: number, b: number) {
  return a + b;  // return type inferred as number
}
```

---

## 4. Functions

```typescript
function greet(name: string): string {
  return `Hello, ${name}!`;
}

// Optional, default, and rest parameters
function greetUser(name: string, title?: string): string {
  return title ? `Hello, ${title} ${name}` : `Hello, ${name}`;
}

function createUser(name: string, role: string = "guest"): string {
  return `${name} is a ${role}`;
}

function sum(...numbers: number[]): number {
  return numbers.reduce((acc, n) => acc + n, 0);
}

// Arrow function
const multiply = (a: number, b: number): number => a * b;

// Function overloads
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value).trim();
}

function throwError(message: string): never {
  throw new Error(message);
}
```

---

## 5. Interfaces

Interfaces define the **shape of an object**.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age?: number;             // Optional
  readonly createdAt: Date; // Cannot be modified after creation
}

// Extending interfaces
interface Admin extends User {
  permissions: string[];
}

// Interface for functions
interface MathOperation {
  (a: number, b: number): number;
}
const add: MathOperation = (a, b) => a + b;

// Declaration merging (unique to interfaces)
interface Window {
  myCustomProp: string;
}
```

---

## 6. Type Aliases

Type aliases create **custom names for types**, including primitives, unions, and complex shapes.

```typescript
type ID = string | number;

type Point = { x: number; y: number };

type Formatter = (value: string) => string;
const toUpper: Formatter = (s) => s.toUpperCase();

// Recursive type
type TreeNode = { value: number; left?: TreeNode; right?: TreeNode };
```

### Interface vs Type Alias

| Feature                  | Interface | Type Alias |
|--------------------------|-----------|------------|
| Extends / Implements     | ✅ Yes     | ✅ (via &)  |
| Declaration merging      | ✅ Yes     | ❌ No       |
| Primitives, unions       | ❌ No      | ✅ Yes      |
| Computed property names  | ❌ No      | ✅ Yes      |

---

## 7. Union & Intersection Types

```typescript
// Union — value can be one of several types
type StringOrNumber = string | number;

// Intersection — value must satisfy ALL types
type Named = { name: string };
type Aged = { age: number };
type Person = Named & Aged;
const person: Person = { name: "Alice", age: 30 };

// Discriminated Unions
type Circle = { kind: "circle"; radius: number };
type Square = { kind: "square"; side: number };
type Shape = Circle | Square;

function getArea(shape: Shape): number {
  if (shape.kind === "circle") return Math.PI * shape.radius ** 2;
  return shape.side ** 2;
}
```

---

## 8. Literal Types

Restrict a variable to specific **exact values**.

```typescript
type Direction = "north" | "south" | "east" | "west";
type DiceValue = 1 | 2 | 3 | 4 | 5 | 6;

function move(direction: Direction, steps: number): void {
  console.log(`Moving ${steps} steps to the ${direction}`);
}

// as const — narrows inferred types to literals
const config = { host: "localhost", port: 3000 } as const;
// config.host is type "localhost", not string
```

---

## 9. Enums

Enums define **named constant sets**.

```typescript
// Numeric Enum (starts at 0 by default)
enum Direction { Up, Down, Left, Right }
console.log(Direction.Up);  // 0
console.log(Direction[0]);  // "Up" (reverse mapping)

// String Enum (no reverse mapping)
enum Color { Red = "RED", Green = "GREEN", Blue = "BLUE" }

// Const Enum (inlined at compile time)
const enum Weekday { Monday = 1, Tuesday, Wednesday, Thursday, Friday }
let day = Weekday.Monday; // Compiled to: let day = 1;
```

---

## 10. Arrays & Tuples

```typescript
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ["a", "b", "c"];
const readonlyArr: readonly number[] = [1, 2, 3];

// Tuple — fixed-length array with known types at each position
let pair: [string, number] = ["Alice", 30];

// Named tuple elements (TS 4.0+)
type Point = [x: number, y: number];

// Rest in tuples
type StringAndNumbers = [string, ...number[]];
const data: StringAndNumbers = ["label", 1, 2, 3];
```

---

## 11. Classes

```typescript
class Animal {
  public name: string;
  private age: number;
  protected sound: string;
  readonly species: string;

  constructor(name: string, age: number, species: string) {
    this.name = name;
    this.age = age;
    this.sound = "...";
    this.species = species;
  }

  get info(): string { return `${this.name} (${this.species})`; }

  set animalAge(value: number) {
    if (value < 0) throw new Error("Age cannot be negative");
    this.age = value;
  }
}

// Shorthand constructor (parameter properties)
class Dog extends Animal {
  constructor(name: string, age: number, public breed: string) {
    super(name, age, "Canis lupus familiaris");
    this.sound = "Woof";
  }
}

// Abstract class
abstract class Shape {
  abstract getArea(): number;
  describe(): string { return `Area: ${this.getArea()}`; }
}

class Circle extends Shape {
  constructor(private radius: number) { super(); }
  getArea(): number { return Math.PI * this.radius ** 2; }
}

// Static members
class Counter {
  static count = 0;
  static increment(): void { Counter.count++; }
}
```

---

## 12. Generics

Generics allow **reusable, type-safe code** that works with any type.

```typescript
function identity<T>(value: T): T { return value; }

// Generic class
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  get size(): number { return this.items.length; }
}

// Generic constraints
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Default generic parameter
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}
```

---

## 13. Type Narrowing & Guards

Narrowing helps TypeScript understand more specific types within control flow.

```typescript
// typeof narrowing
function process(value: string | number): string {
  if (typeof value === "string") return value.toUpperCase();
  return value.toFixed(2);
}

// instanceof narrowing
class Cat { meow() { return "Meow"; } }
class Dog { bark() { return "Woof"; } }
function makeNoise(animal: Cat | Dog): string {
  return animal instanceof Cat ? animal.meow() : animal.bark();
}

// in operator narrowing
type Fish = { swim: () => void };
type Bird = { fly: () => void };
function move(creature: Fish | Bird): void {
  "swim" in creature ? creature.swim() : creature.fly();
}

// Custom Type Guard (type predicate)
interface Car { drive: () => void }
interface Boat { sail: () => void }
function isCar(vehicle: Car | Boat): vehicle is Car {
  return (vehicle as Car).drive !== undefined;
}

// Nullish check
function getLength(str: string | null | undefined): number {
  if (str == null) return 0;
  return str.length;
}
```

---

## 14. Utility Types

TypeScript ships with **built-in generic utility types**.

```typescript
interface User {
  id: number; name: string; email: string;
  age: number; role: "admin" | "user" | "guest";
}

type PartialUser   = Partial<User>;           // All optional
type RequiredUser  = Required<PartialUser>;   // All required
type ReadonlyUser  = Readonly<User>;          // All readonly
type UserPreview   = Pick<User, "id" | "name">;     // Keep only these
type UserWithoutId = Omit<User, "id">;              // Remove these
type RoleMap       = Record<User["role"], string[]>; // Keys → values

type NonAdmin   = Exclude<User["role"], "admin">;  // "user" | "guest"
type AdminOnly  = Extract<User["role"], "admin">;  // "admin"
type SafeString = NonNullable<string | null | undefined>; // string

type FetchResult = ReturnType<typeof fetchUser>;   // User
type FetchParams = Parameters<typeof fetchUser>;   // []
type Resolved    = Awaited<Promise<string>>;       // string
```

---

## Quick Reference Cheatsheet

```typescript
// ── TYPES ──────────────────────────────────────────────
let s: string, n: number, b: boolean;
let a: any, uk: unknown, nv: never, v: void;

// ── ARRAYS & TUPLES ────────────────────────────────────
let arr: number[] = [];
let tup: [string, number] = ["hello", 42];

// ── UNION & INTERSECTION ───────────────────────────────
type U = string | number;
type I = TypeA & TypeB;

// ── GENERICS ───────────────────────────────────────────
function identity<T>(x: T): T { return x; }
function getKey<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }

// ── UTILITY TYPES ──────────────────────────────────────
Partial<T>       // All optional
Required<T>      // All required
Readonly<T>      // All readonly
Pick<T, K>       // Keep only K
Omit<T, K>       // Remove K
Record<K, V>     // Object with keys K → values V
Exclude<T, U>    // Remove U from union T
Extract<T, U>    // Keep U from union T
NonNullable<T>   // Remove null/undefined
ReturnType<F>    // Return type of function F
Parameters<F>    // Param types of function F
Awaited<T>       // Resolved type of Promise<T>

// ── CONDITIONAL TYPES ──────────────────────────────────
type IsArray<T> = T extends any[] ? true : false;
type Unwrap<T>  = T extends Promise<infer U> ? U : T;

// ── MAPPED TYPES ───────────────────────────────────────
type MyPartial<T>  = { [K in keyof T]?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };

// ── TEMPLATE LITERALS ──────────────────────────────────
type EventName<T extends string> = `on${Capitalize<T>}`;
```

---

> **Further Reading**
> - [Official TypeScript Docs](https://www.typescriptlang.org/docs/)
> - [TypeScript Playground](https://www.typescriptlang.org/play)
> - [TypeScript Deep Dive (Book)](https://basarat.gitbook.io/typescript/)
