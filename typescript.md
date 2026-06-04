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
15. [Mapped Types](#15-mapped-types)
16. [Conditional Types](#16-conditional-types)
17. [Template Literal Types](#17-template-literal-types)
18. [Decorators](#18-decorators)
19. [Modules & Namespaces](#19-modules--namespaces)
20. [Declaration Files (.d.ts)](#20-declaration-files-dts)

---

## 1. What is TypeScript?

TypeScript is a **strongly typed superset of JavaScript** developed by Microsoft. It compiles down to plain JavaScript and adds optional static typing, interfaces, generics, and more.

**Why TypeScript?**
- Catches errors at **compile time** instead of runtime
- Provides better **IDE autocomplete & refactoring**
- Makes large codebases more **maintainable**
- Supports modern JavaScript features with backward compatibility

```bash
# Install TypeScript globally
npm install -g typescript

# Compile a TypeScript file
tsc hello.ts

# Run in watch mode
tsc --watch
```

---

## 2. Basic Types

TypeScript provides a set of primitive and object types.

```typescript
// Primitive Types
let name: string = "Alice";
let age: number = 30;
let isActive: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;

// Special Types
let anything: any = "can be anything"; // Opt-out of type checking (avoid if possible)
let unknown: unknown = 42;             // Like any, but safer — must narrow before use
let neverReturns: never;               // Function that never returns (throws or infinite loop)
let voidFn: void;                      // Function that returns nothing

// Object & Symbol
let obj: object = { key: "value" };
let sym: symbol = Symbol("id");

// BigInt
let bigNum: bigint = 9007199254740991n;
```

---

## 3. Type Inference

TypeScript can **automatically infer** types without explicit annotations.

```typescript
let city = "New York";        // inferred as string
let count = 42;               // inferred as number
let isOpen = true;            // inferred as boolean

// TypeScript knows this is wrong:
// city = 100;  ❌ Error: Type 'number' is not assignable to type 'string'

// Inferred return type
function add(a: number, b: number) {
  return a + b;  // return type inferred as number
}
```

---

## 4. Functions

```typescript
// Basic function with type annotations
function greet(name: string): string {
  return `Hello, ${name}!`;
}

// Optional parameters (use ?)
function greetUser(name: string, title?: string): string {
  return title ? `Hello, ${title} ${name}` : `Hello, ${name}`;
}

// Default parameters
function createUser(name: string, role: string = "guest"): string {
  return `${name} is a ${role}`;
}

// Rest parameters
function sum(...numbers: number[]): number {
  return numbers.reduce((acc, n) => acc + n, 0);
}

// Arrow functions
const multiply = (a: number, b: number): number => a * b;

// Function overloads
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value).trim();
}

// Void return type
function logMessage(msg: string): void {
  console.log(msg);
}

// Never return type (always throws)
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
  age?: number;          // Optional property
  readonly createdAt: Date; // Cannot be modified after creation
}

const user: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
  createdAt: new Date(),
};

// Interface extending another interface
interface Admin extends User {
  permissions: string[];
}

const admin: Admin = {
  id: 2,
  name: "Bob",
  email: "bob@example.com",
  createdAt: new Date(),
  permissions: ["read", "write", "delete"],
};

// Interface for functions
interface MathOperation {
  (a: number, b: number): number;
}

const add: MathOperation = (a, b) => a + b;
const subtract: MathOperation = (a, b) => a - b;

// Interface declaration merging (unique to interfaces)
interface Window {
  myCustomProp: string;
}
// Now the global Window type has myCustomProp
```

---

## 6. Type Aliases

Type aliases create **custom names for types**, including primitives, unions, and complex shapes.

```typescript
// Alias for primitive
type ID = string | number;

// Alias for object shape
type Point = {
  x: number;
  y: number;
};

// Alias for function type
type Formatter = (value: string) => string;

const toUpper: Formatter = (s) => s.toUpperCase();

// Recursive type alias
type TreeNode = {
  value: number;
  left?: TreeNode;
  right?: TreeNode;
};
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
// Union Type — value can be one of several types
type StringOrNumber = string | number;

function display(value: StringOrNumber): void {
  console.log(value);
}

display("hello"); // ✅
display(42);      // ✅

// Intersection Type — value must satisfy ALL types
type Named = { name: string };
type Aged = { age: number };

type Person = Named & Aged;

const person: Person = { name: "Alice", age: 30 }; // Must have both

// Union with objects (Discriminated Unions)
type Circle = { kind: "circle"; radius: number };
type Square = { kind: "square"; side: number };
type Shape = Circle | Square;

function getArea(shape: Shape): number {
  if (shape.kind === "circle") {
    return Math.PI * shape.radius ** 2;
  }
  return shape.side ** 2;
}
```

---

## 8. Literal Types

Restrict a variable to specific **exact values**.

```typescript
// String literal type
type Direction = "north" | "south" | "east" | "west";
let dir: Direction = "north"; // Only these 4 values are valid

// Numeric literal type
type DiceValue = 1 | 2 | 3 | 4 | 5 | 6;
let roll: DiceValue = 4;

// Boolean literal
type AlwaysTrue = true;

// Literal type in functions
function move(direction: Direction, steps: number): void {
  console.log(`Moving ${steps} steps to the ${direction}`);
}

move("north", 3); // ✅
// move("up", 3); ❌ Error: "up" is not assignable to Direction

// As const — narrow inferred types to literals
const config = {
  host: "localhost",
  port: 3000,
} as const;
// config.host is type "localhost", not string
```

---

## 9. Enums

Enums define **named constant sets**.

```typescript
// Numeric Enum (default: starts at 0)
enum Direction {
  Up,    // 0
  Down,  // 1
  Left,  // 2
  Right, // 3
}

console.log(Direction.Up);   // 0
console.log(Direction[0]);   // "Up" (reverse mapping)

// Custom numeric values
enum StatusCode {
  OK = 200,
  NotFound = 404,
  ServerError = 500,
}

// String Enum (no reverse mapping)
enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}

function paintWall(color: Color): void {
  console.log(`Painting with ${color}`);
}

paintWall(Color.Red); // ✅

// Const Enum (inlined at compile time for performance)
const enum Weekday {
  Monday = 1,
  Tuesday,
  Wednesday,
  Thursday,
  Friday,
}

let day = Weekday.Monday; // Compiled to: let day = 1;
```

---

## 10. Arrays & Tuples

```typescript
// Array types
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ["a", "b", "c"]; // Generic syntax

// Read-only array
const readonlyArr: readonly number[] = [1, 2, 3];
// readonlyArr.push(4); ❌ Error

// Multi-dimensional
let matrix: number[][] = [[1, 2], [3, 4]];

// Tuple — fixed-length array with known types at each position
let pair: [string, number] = ["Alice", 30];

let rgb: [number, number, number] = [255, 128, 0];

// Named tuple elements (TypeScript 4.0+)
type Point = [x: number, y: number];
const origin: Point = [0, 0];

// Optional tuple elements
type OptionalTuple = [string, number?];
const t1: OptionalTuple = ["hello"];         // ✅
const t2: OptionalTuple = ["hello", 42];     // ✅

// Rest in tuples
type StringAndNumbers = [string, ...number[]];
const data: StringAndNumbers = ["label", 1, 2, 3, 4];
```

---

## 11. Classes

```typescript
class Animal {
  // Access modifiers: public (default), private, protected
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

  // Getter & Setter
  get info(): string {
    return `${this.name} (${this.species})`;
  }

  set animalAge(value: number) {
    if (value < 0) throw new Error("Age cannot be negative");
    this.age = value;
  }

  makeSound(): void {
    console.log(this.sound);
  }
}

// Shorthand constructor (parameter properties)
class Dog extends Animal {
  constructor(
    name: string,
    age: number,
    public breed: string  // Automatically declares & assigns
  ) {
    super(name, age, "Canis lupus familiaris");
    this.sound = "Woof";
  }

  fetch(item: string): string {
    return `${this.name} fetched the ${item}!`;
  }
}

const dog = new Dog("Rex", 3, "Labrador");
console.log(dog.info);        // "Rex (Canis lupus familiaris)"
console.log(dog.fetch("ball")); // "Rex fetched the ball!"

// Abstract classes — cannot be instantiated directly
abstract class Shape {
  abstract getArea(): number;

  describe(): string {
    return `This shape has an area of ${this.getArea()}`;
  }
}

class Circle extends Shape {
  constructor(private radius: number) {
    super();
  }

  getArea(): number {
    return Math.PI * this.radius ** 2;
  }
}

// Implementing interfaces
interface Serializable {
  serialize(): string;
}

class Rectangle extends Shape implements Serializable {
  constructor(private width: number, private height: number) {
    super();
  }

  getArea(): number {
    return this.width * this.height;
  }

  serialize(): string {
    return JSON.stringify({ width: this.width, height: this.height });
  }
}

// Static members
class Counter {
  static count = 0;

  static increment(): void {
    Counter.count++;
  }

  static getCount(): number {
    return Counter.count;
  }
}

Counter.increment();
Counter.increment();
console.log(Counter.getCount()); // 2
```

---

## 12. Generics

Generics allow you to write **reusable, type-safe code** that works with any type.

```typescript
// Generic function
function identity<T>(value: T): T {
  return value;
}

identity<string>("hello");  // Explicit
identity(42);               // Inferred as number

// Generic with multiple type params
function pair<K, V>(key: K, value: V): [K, V] {
  return [key, value];
}

const result = pair("name", "Alice"); // [string, string]

// Generic interface
interface Repository<T> {
  findById(id: number): T | undefined;
  findAll(): T[];
  save(item: T): void;
  delete(id: number): void;
}

// Generic class
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
console.log(numStack.pop()); // 2

// Generic constraints (extends)
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Alice", age: 30 };
getProperty(user, "name"); // ✅ Returns string
// getProperty(user, "salary"); ❌ Error: "salary" not in user

// Default generic parameter
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

type UserResponse = ApiResponse<{ id: number; name: string }>;
```

---

## 13. Type Narrowing & Guards

Narrowing helps TypeScript understand more specific types within control flow.

```typescript
// typeof narrowing
function process(value: string | number): string {
  if (typeof value === "string") {
    return value.toUpperCase(); // TypeScript knows it's string here
  }
  return value.toFixed(2);     // TypeScript knows it's number here
}

// instanceof narrowing
class Cat { meow() { return "Meow"; } }
class Dog { bark() { return "Woof"; } }

function makeNoise(animal: Cat | Dog): string {
  if (animal instanceof Cat) {
    return animal.meow();
  }
  return animal.bark();
}

// in operator narrowing
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(creature: Fish | Bird): void {
  if ("swim" in creature) {
    creature.swim();
  } else {
    creature.fly();
  }
}

// Custom Type Guard (type predicate)
interface Car { make: string; drive: () => void; }
interface Boat { brand: string; sail: () => void; }

function isCar(vehicle: Car | Boat): vehicle is Car {
  return (vehicle as Car).drive !== undefined;
}

function useVehicle(vehicle: Car | Boat): void {
  if (isCar(vehicle)) {
    vehicle.drive(); // TypeScript knows it's Car
  } else {
    vehicle.sail();  // TypeScript knows it's Boat
  }
}

// Assertion functions
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

// Nullish checks
function getLength(str: string | null | undefined): number {
  if (str == null) return 0; // Narrows out null and undefined
  return str.length;
}
```

---

## 14. Utility Types

TypeScript ships with powerful **built-in generic utility types**.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  role: "admin" | "user" | "guest";
}

// Partial<T> — makes all properties optional
type PartialUser = Partial<User>;
const update: PartialUser = { name: "Bob" }; // Only name, rest are optional

// Required<T> — makes all properties required
type RequiredUser = Required<PartialUser>;

// Readonly<T> — makes all properties read-only
type ReadonlyUser = Readonly<User>;
const frozenUser: ReadonlyUser = { id: 1, name: "Alice", email: "a@b.com", age: 25, role: "user" };
// frozenUser.name = "Bob"; ❌ Error

// Pick<T, K> — picks a subset of properties
type UserPreview = Pick<User, "id" | "name">;
const preview: UserPreview = { id: 1, name: "Alice" };

// Omit<T, K> — removes specified properties
type UserWithoutId = Omit<User, "id">;

// Record<K, V> — creates an object type with keys K and values V
type RoleMap = Record<User["role"], string[]>;
const permissions: RoleMap = {
  admin: ["read", "write", "delete"],
  user: ["read", "write"],
  guest: ["read"],
};

// Exclude<T, U> — removes types from a union
type NonAdmin = Exclude<User["role"], "admin">; // "user" | "guest"

// Extract<T, U> — extracts matching types from a union
type AdminOnly = Extract<User["role"], "admin">; // "admin"

// NonNullable<T> — removes null and undefined
type SafeString = NonNullable<string | null | undefined>; // string

// ReturnType<T> — extracts the return type of a function
function fetchUser(): User {
  return { id: 1, name: "Alice", email: "a@b.com", age: 30, role: "user" };
}
type FetchResult = ReturnType<typeof fetchUser>; // User

// Parameters<T> — extracts the parameter types of a function as a tuple
type FetchParams = Parameters<typeof fetchUser>; // []

// InstanceType<T> — gets the instance type of a constructor
class Logger { log(msg: string) { console.log(msg); } }
type LoggerInstance = InstanceType<typeof Logger>; // Logger

// Awaited<T> — unwraps Promise types (TypeScript 4.5+)
type ResolvedType = Awaited<Promise<string>>; // string
type DeepResolved = Awaited<Promise<Promise<number>>>; // number
```

---

## 15. Mapped Types

Mapped types transform existing types by iterating over their keys.

```typescript
// Basic mapped type
type Stringify<T> = {
  [K in keyof T]: string;
};

type User = { id: number; name: string; age: number };
type StringifiedUser = Stringify<User>;
// { id: string; name: string; age: string }

// Add optional modifier
type Optional<T> = {
  [K in keyof T]?: T[K];
};

// Add readonly modifier
type Immutable<T> = {
  readonly [K in keyof T]: T[K];
};

// Remove modifiers with - prefix
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type Concrete<T> = {
  [K in keyof T]-?: T[K]; // Removes optional
};

// Remap keys with "as" (TypeScript 4.1+)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// { getId: () => number; getName: () => string; getAge: () => number }

// Filter keys using Exclude in remapping
type NonIdFields<T> = {
  [K in keyof T as Exclude<K, "id">]: T[K];
};

type UserWithoutId = NonIdFields<User>; // { name: string; age: number }
```

---

## 16. Conditional Types

Conditional types select types based on a **condition**.

```typescript
// Basic syntax: T extends U ? TrueType : FalseType
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<string>;  // "yes"
type B = IsString<number>;  // "no"

// Practical use: flatten array types
type Flatten<T> = T extends Array<infer U> ? U : T;

type Str = Flatten<string[]>;   // string
type Num = Flatten<number>;     // number (not an array, returns itself)

// infer keyword — extract types within conditions
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type StrReturn = ReturnType<() => string>;        // string
type NumReturn = ReturnType<(x: number) => void>; // void

// Distributive conditional types (applied to each union member)
type ToArray<T> = T extends any ? T[] : never;

type StrOrNumArray = ToArray<string | number>; // string[] | number[]

// Non-distributive (wrap in tuple to prevent distribution)
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type Combined = ToArrayNonDist<string | number>; // (string | number)[]

// Real-world example: DeepPartial
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

interface Config {
  server: { host: string; port: number };
  db: { name: string; password: string };
}

type PartialConfig = DeepPartial<Config>;
// All nested fields become optional
```

---

## 17. Template Literal Types

Build **string types dynamically** using template literal syntax.

```typescript
// Basic template literal type
type Greeting = `Hello, ${string}!`;
const g: Greeting = "Hello, World!"; // ✅

// Combining unions in template literals
type Size = "small" | "medium" | "large";
type Color = "red" | "blue";

type SizedColor = `${Size}-${Color}`;
// "small-red" | "small-blue" | "medium-red" | "medium-blue" | "large-red" | "large-blue"

// Utility: CSS property builder
type CSSProperty = "margin" | "padding";
type CSSDirection = "top" | "right" | "bottom" | "left";

type CSSProps = `${CSSProperty}-${CSSDirection}`;
// "margin-top" | "margin-right" | ... | "padding-left"

// Built-in string manipulation types
type Upper = Uppercase<"hello">;           // "HELLO"
type Lower = Lowercase<"WORLD">;           // "world"
type Cap = Capitalize<"typescript">;       // "Typescript"
type Uncap = Uncapitalize<"TypeScript">;   // "typeScript"

// Generate event handler names
type EventName = "click" | "focus" | "blur";
type EventHandler = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"

// Real world: typed event emitter
type EventMap = {
  userCreated: { id: number; name: string };
  userDeleted: { id: number };
};

type EventKeys = keyof EventMap; // "userCreated" | "userDeleted"
type ListenerKey = `on:${EventKeys}`; // "on:userCreated" | "on:userDeleted"
```

---

## 18. Decorators

Decorators are special **metadata annotations** for classes, methods, properties, and parameters.

> ⚠️ Enable in `tsconfig.json`: `"experimentalDecorators": true`

```typescript
// Class decorator
function Sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@Sealed
class BankAccount {
  balance: number = 0;
}

// Decorator factory (returns a decorator)
function Log(prefix: string) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    descriptor.value = function (...args: any[]) {
      console.log(`[${prefix}] Calling ${key} with`, args);
      const result = original.apply(this, args);
      console.log(`[${prefix}] Result:`, result);
      return result;
    };
    return descriptor;
  };
}

class Calculator {
  @Log("MATH")
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(2, 3);
// [MATH] Calling add with [2, 3]
// [MATH] Result: 5

// Property decorator
function ReadOnly(target: any, key: string) {
  Object.defineProperty(target, key, {
    writable: false,
  });
}

class Config {
  @ReadOnly
  version = "1.0.0";
}

// Parameter decorator
function Required(target: any, key: string, index: number) {
  console.log(`Parameter ${index} in ${key} is required`);
}

class UserService {
  createUser(@Required name: string, role: string = "user") {
    return { name, role };
  }
}
```

---

## 19. Modules & Namespaces

```typescript
// ─── math.ts ─────────────────────────────────────
export function add(a: number, b: number): number {
  return a + b;
}

export const PI = 3.14159;

export default class MathHelper {
  static square(n: number): number {
    return n * n;
  }
}

// ─── main.ts ─────────────────────────────────────
import MathHelper, { add, PI } from "./math";
import * as Math from "./math";  // Import all as namespace

console.log(add(2, 3));           // 5
console.log(PI);                  // 3.14159
console.log(MathHelper.square(4)); // 16
console.log(Math.PI);             // 3.14159

// Re-exporting
export { add as sum } from "./math";

// Namespaces (for grouping related code, often in global scripts)
namespace Validation {
  export interface StringValidator {
    isValid(s: string): boolean;
  }

  export class EmailValidator implements StringValidator {
    isValid(s: string): boolean {
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(s);
    }
  }
}

const validator = new Validation.EmailValidator();
console.log(validator.isValid("test@example.com")); // true
```

---

## 20. Declaration Files (.d.ts)

Declaration files teach TypeScript about types in **third-party JavaScript libraries**.

```typescript
// ─── myLib.d.ts ──────────────────────────────────
// Declare a global variable
declare const VERSION: string;

// Declare a global function
declare function greet(name: string): void;

// Declare a module
declare module "my-library" {
  export interface Options {
    timeout: number;
    retries: number;
  }

  export function connect(url: string, options?: Options): Promise<void>;
  export function disconnect(): void;
}

// Declare module augmentation (extend existing types)
declare module "express" {
  interface Request {
    userId?: number;  // Add custom property to Express Request
  }
}

// Declare global type augmentation
declare global {
  interface Window {
    analytics: {
      track: (event: string, data?: object) => void;
    };
  }
}

// ─── Usage ───────────────────────────────────────
// In your .ts file:
import { connect } from "my-library";

connect("https://api.example.com", { timeout: 5000, retries: 3 });
window.analytics.track("page_view", { page: "/home" });
```

---

## Quick Reference Cheatsheet

```typescript
// ── TYPES ────────────────────────────────────────────────────────
let s: string, n: number, b: boolean, u: undefined, nil: null;
let a: any, uk: unknown, nv: never, v: void;

// ── ARRAYS & TUPLES ──────────────────────────────────────────────
let arr: number[] = [];
let tup: [string, number] = ["hello", 42];

// ── UNION & INTERSECTION ─────────────────────────────────────────
type U = string | number;
type I = TypeA & TypeB;

// ── GENERICS ─────────────────────────────────────────────────────
function identity<T>(x: T): T { return x; }
function getKey<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }

// ── UTILITY TYPES ────────────────────────────────────────────────
Partial<T>       // All optional
Required<T>      // All required
Readonly<T>      // All readonly
Pick<T, K>       // Keep only K
Omit<T, K>       // Remove K
Record<K, V>     // Object with keys K and values V
Exclude<T, U>    // Remove U from T (union)
Extract<T, U>    // Keep U from T (union)
NonNullable<T>   // Remove null/undefined
ReturnType<F>    // Return type of function F
Parameters<F>    // Param types of function F
Awaited<T>       // Resolved type of Promise<T>

// ── CONDITIONAL TYPES ────────────────────────────────────────────
type IsArray<T> = T extends any[] ? true : false;
type Unwrap<T> = T extends Promise<infer U> ? U : T;

// ── MAPPED TYPES ─────────────────────────────────────────────────
type MyPartial<T> = { [K in keyof T]?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };

// ── TEMPLATE LITERALS ────────────────────────────────────────────
type EventName<T extends string> = `on${Capitalize<T>}`;
```

---

> 📘 **Further Reading**
> - [Official TypeScript Docs](https://www.typescriptlang.org/docs/)
> - [TypeScript Playground](https://www.typescriptlang.org/play)
> - [TypeScript Deep Dive (Book)](https://basarat.gitbook.io/typescript/)
