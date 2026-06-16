# React.js Interview Preparation Guide

> A comprehensive guide covering React concepts from basics to advanced, with real-world analogies, code examples, and interview tips.

---

## Table of Contents

1. [What is React?](#1-what-is-react)
2. [JSX](#2-jsx)
3. [Components — Functional vs Class](#3-components--functional-vs-class)
4. [Props](#4-props)
5. [State](#5-state)
6. [useState Hook](#6-usestate-hook)
7. [useEffect Hook](#7-useeffect-hook)
8. [useRef Hook](#8-useref-hook)
9. [useContext & Context API](#9-usecontext--context-api)
10. [useReducer Hook](#10-usereducer-hook)
11. [useMemo & useCallback](#11-usememo--usecallback)
12. [Custom Hooks](#12-custom-hooks)
13. [Component Lifecycle](#13-component-lifecycle)
14. [Virtual DOM & Reconciliation](#14-virtual-dom--reconciliation)
15. [Event Handling](#15-event-handling)
16. [Conditional Rendering](#16-conditional-rendering)
17. [Lists & Keys](#17-lists--keys)
18. [Forms & Controlled Components](#18-forms--controlled-components)
19. [React Router](#19-react-router)
20. [State Management — Redux & Zustand](#20-state-management--redux--zustand)
21. [Performance Optimization](#21-performance-optimization)
22. [Error Boundaries](#22-error-boundaries)
23. [Higher-Order Components (HOC)](#23-higher-order-components-hoc)
24. [Render Props Pattern](#24-render-props-pattern)
25. [Code Splitting & Lazy Loading](#25-code-splitting--lazy-loading)
26. [Testing React Components](#26-testing-react-components)
27. [Common Interview Questions](#27-common-interview-questions)
28. [Quick Revision Cheat Sheet](#28-quick-revision-cheat-sheet)

---

## 1. What is React?

React is a **JavaScript library** (not a framework) built by Facebook/Meta for building **user interfaces**, specifically single-page applications (SPAs).

**Real-world analogy:** Think of React like LEGO bricks. Each brick (component) is independent and reusable. You snap them together to build complex structures (UIs).

**Key features:**
- **Component-based** — UI split into small, reusable pieces
- **Declarative** — you describe WHAT the UI should look like, not HOW to update it
- **Virtual DOM** — React maintains a lightweight copy of the real DOM for performance
- **One-way data flow** — data flows from parent to child via props

```bash
# Create a new React app (Vite is now preferred over CRA)
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev

# OR with Create React App (older approach)
npx create-react-app my-app
cd my-app
npm start
```

> **Interview tip:** Interviewers often ask "Is React a framework or library?" — it's a **library**. Angular is a framework. React only handles the View layer.

---

## 2. JSX

JSX (JavaScript XML) is a syntax extension that lets you write HTML-like code inside JavaScript.

**Real-world analogy:** JSX is like a template language — it looks like HTML but it's actually JavaScript under the hood.

```jsx
// JSX
const element = <h1 className="title">Hello, World!</h1>;

// What JSX compiles to (Babel transforms it):
const element = React.createElement('h1', { className: 'title' }, 'Hello, World!');

// JSX rules:
// 1. Must return a SINGLE root element — use <> </> (Fragment) to wrap
// 2. Use className instead of class
// 3. Self-closing tags must have /  e.g. <input />
// 4. JavaScript expressions use {}

const name = "Amith";
const greeting = (
  <>
    <h1>Hello, {name}!</h1>
    <p>Today is {new Date().toDateString()}</p>
    <input type="text" />
  </>
);
```

> **Interview tip:** Know that JSX is syntactic sugar over `React.createElement()`. Since React 17+, you don't need `import React from 'react'` at the top because of the new JSX transform.

---

## 3. Components — Functional vs Class

A **component** is a reusable piece of UI. Components accept inputs (props) and return JSX.

### Functional Component (Modern — preferred)

```jsx
// Simple functional component
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// Arrow function style
const Greeting = ({ name }) => <h1>Hello, {name}!</h1>;

// Usage
<Greeting name="Amith" />
```

### Class Component (Legacy — still in many codebases)

```jsx
import React, { Component } from 'react';

class Greeting extends Component {
  render() {
    return <h1>Hello, {this.props.name}!</h1>;
  }
}
```

| Feature | Functional | Class |
|---|---|---|
| Syntax | Simple function | Extends `Component` |
| State | `useState` hook | `this.state` |
| Lifecycle | `useEffect` | `componentDidMount`, etc. |
| `this` keyword | Not needed | Required |
| Performance | Slightly better | Slightly more overhead |
| Modern usage | Preferred | Legacy |

> **Interview tip:** Always say you prefer functional components with hooks. Class components are still important to know because many production codebases still use them.

---

## 4. Props

Props (properties) are **read-only** inputs passed from a parent component to a child. They flow **one-way** — parent to child only.

**Real-world analogy:** Props are like function arguments — you pass data in, but the function can't change the caller's variables.

```jsx
// Parent passes data
function App() {
  return (
    <UserCard
      name="Amith"
      age={25}
      isAdmin={true}
      onClick={() => alert('clicked')}
    />
  );
}

// Child receives via props object
function UserCard({ name, age, isAdmin, onClick }) {
  return (
    <div onClick={onClick}>
      <h2>{name}</h2>
      <p>Age: {age}</p>
      {isAdmin && <span>Admin</span>}
    </div>
  );
}

// Default props
function Button({ label = "Click Me", color = "blue" }) {
  return <button style={{ backgroundColor: color }}>{label}</button>;
}

// Children prop — content between tags
function Card({ children, title }) {
  return (
    <div className="card">
      <h3>{title}</h3>
      {children}
    </div>
  );
}

// Usage
<Card title="My Card">
  <p>This is passed as children</p>
</Card>
```

> **Interview tip:** Emphasize that **props are immutable** — a child component should never modify its own props. If you need to change data, lift state up to the parent.

---

## 5. State

State is **mutable data** that belongs to a component. When state changes, React **re-renders** the component.

**Real-world analogy:** Think of state like a whiteboard in your room. You can write and erase on it (mutable), and every time you change it, whoever is watching the room (React) updates the view.

```jsx
// State vs Props summary
// Props  = data FROM parent (read-only, like function arguments)
// State  = data INSIDE component (mutable, triggers re-render)

// When to use state:
// - Counter values
// - Form input values
// - Toggle (open/close, show/hide)
// - Data fetched from an API
// - Any data that changes over time and affects the UI
```

---

## 6. useState Hook

`useState` is the most fundamental hook for managing state in functional components.

```jsx
import { useState } from 'react';

// Basic counter
function Counter() {
  const [count, setCount] = useState(0);  // [currentValue, setter]

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
      <button onClick={() => setCount(count - 1)}>-</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}

// IMPORTANT: State updates are ASYNCHRONOUS
// If new state depends on old state, use the functional form:
function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(prev => prev + 1);  // Safe — uses latest value
    setCount(prev => prev + 1);  // This WORKS — count goes up by 2
    // setCount(count + 1);       // This would NOT work — count is stale
    // setCount(count + 1);       // Both would still only go up by 1
  };
  return <button onClick={increment}>+2</button>;
}

// State with objects
function Form() {
  const [user, setUser] = useState({ name: '', email: '' });

  const handleChange = (e) => {
    setUser(prev => ({
      ...prev,                          // Spread existing values
      [e.target.name]: e.target.value   // Override changed field
    }));
  };

  return (
    <>
      <input name="name" value={user.name} onChange={handleChange} />
      <input name="email" value={user.email} onChange={handleChange} />
    </>
  );
}
```

**Common mistakes:**
```jsx
// WRONG — Direct mutation doesn't trigger re-render
const [items, setItems] = useState([]);
items.push('new item');           // Never do this!

// RIGHT — Create new array
setItems(prev => [...prev, 'new item']);

// WRONG — Mutating object
user.name = 'John';               // Never do this!

// RIGHT — Create new object
setUser(prev => ({ ...prev, name: 'John' }));
```

> **Interview tip:** A very common interview question is "why shouldn't you mutate state directly?" — answer: React uses **shallow comparison** to detect changes. If you mutate in place, the reference stays the same, React thinks nothing changed, and won't re-render.

---

## 7. useEffect Hook

`useEffect` handles **side effects** — things that happen outside of rendering: API calls, subscriptions, timers, DOM manipulation.

**Real-world analogy:** useEffect is like an "after-render" notification system. "After the UI is painted, go do this extra work."

```jsx
import { useState, useEffect } from 'react';

// Syntax: useEffect(callback, [dependencies])

// 1. Runs after EVERY render (no dependency array)
useEffect(() => {
  console.log('Component rendered');
});

// 2. Runs ONCE on mount (empty dependency array)
useEffect(() => {
  console.log('Component mounted');
  fetchData();
}, []);

// 3. Runs when specific values change
useEffect(() => {
  console.log('userId changed:', userId);
  fetchUser(userId);
}, [userId]);

// 4. Cleanup — runs before component unmounts or before re-running
useEffect(() => {
  const timer = setInterval(() => console.log('tick'), 1000);

  return () => {
    clearInterval(timer);  // Cleanup prevents memory leaks
    console.log('cleaned up');
  };
}, []);

// Real example — fetch data from API
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;  // Prevent state update after unmount

    const fetchUser = async () => {
      try {
        setLoading(true);
        const response = await fetch(`/api/users/${userId}`);
        const data = await response.json();
        if (!cancelled) setUser(data);
      } catch (err) {
        if (!cancelled) setError(err.message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    };

    fetchUser();

    return () => { cancelled = true; };  // Cleanup
  }, [userId]);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <div>{user?.name}</div>;
}
```

> **Interview tip:** Know the three forms (no deps, empty deps, specific deps). Also understand **cleanup** and why it matters — memory leaks from subscriptions/timers are a classic interview topic.

---

## 8. useRef Hook

`useRef` gives you a **mutable reference** that persists across renders WITHOUT causing re-renders.

**Two main use cases:**
1. Accessing DOM elements directly
2. Storing mutable values that shouldn't trigger re-renders

```jsx
import { useRef, useEffect, useState } from 'react';

// Use case 1: DOM access
function FocusInput() {
  const inputRef = useRef(null);

  const handleClick = () => {
    inputRef.current.focus();   // Direct DOM manipulation
  };

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={handleClick}>Focus Input</button>
    </>
  );
}

// Use case 2: Store previous value (doesn't cause re-render)
function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef(null);

  const start = () => {
    intervalRef.current = setInterval(() => setCount(c => c + 1), 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
  };

  return (
    <>
      <p>{count}</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </>
  );
}

// useRef vs useState
// useRef  — mutable, does NOT re-render, access via .current
// useState — mutable, DOES re-render, access directly
```

> **Interview tip:** useRef is commonly used in interviews to demonstrate DOM access without re-rendering. Know the difference between `ref.current` (mutable, no re-render) and `state` (causes re-render).

---

## 9. useContext & Context API

Context lets you **share data across components** without prop drilling (passing props through many levels).

**Real-world analogy:** Context is like a global announcement system. Instead of passing a note person-to-person down a chain, you broadcast it and anyone who's tuned in receives it.

```jsx
import { createContext, useContext, useState } from 'react';

// Step 1: Create the context
const ThemeContext = createContext('light');   // 'light' is the default

// Step 2: Provide the context (wrap components that need it)
function App() {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Navbar />
      <MainContent />
    </ThemeContext.Provider>
  );
}

// Step 3: Consume the context anywhere in the tree
function Navbar() {
  const { theme, setTheme } = useContext(ThemeContext);

  return (
    <nav style={{ background: theme === 'dark' ? '#333' : '#fff' }}>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </nav>
  );
}

// Common contexts in real apps:
// - Auth context (current user, login/logout)
// - Theme context (dark/light mode)
// - Language context (i18n)
// - Cart context (e-commerce)
```

**When NOT to use Context:**
- For frequently changing data (every keystroke) — prefer Zustand/Redux
- Context re-renders ALL consumers when the value changes

> **Interview tip:** Context solves **prop drilling** but is not a replacement for dedicated state management like Redux. Mention this distinction.

---

## 10. useReducer Hook

`useReducer` is an alternative to `useState` for **complex state logic** — multiple sub-values or when the next state depends on the previous.

**Real-world analogy:** Think of Redux store in miniature. A reducer is like a customer service agent: you send an "action" (your request), and they follow rules to update the "state" (the account).

```jsx
import { useReducer } from 'react';

// Define the reducer (pure function — no side effects)
function counterReducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };
    case 'DECREMENT':
      return { count: state.count - 1 };
    case 'RESET':
      return { count: 0 };
    case 'SET':
      return { count: action.payload };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <button onClick={() => dispatch({ type: 'RESET' })}>Reset</button>
      <button onClick={() => dispatch({ type: 'SET', payload: 10 })}>Set 10</button>
    </div>
  );
}

// useState vs useReducer
// useState   — simple, independent state values
// useReducer — complex state, multiple fields, state transitions depend on action type
```

---

## 11. useMemo & useCallback

These hooks **optimize performance** by memoizing (caching) values and functions.

```jsx
import { useMemo, useCallback, useState } from 'react';

// useMemo — caches the RESULT of a computation
function ProductList({ products, filterText }) {
  // Without useMemo: this runs on EVERY render
  // With useMemo: only runs when products or filterText changes
  const filteredProducts = useMemo(() => {
    console.log('Filtering...');
    return products.filter(p =>
      p.name.toLowerCase().includes(filterText.toLowerCase())
    );
  }, [products, filterText]);

  return <ul>{filteredProducts.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}

// useCallback — caches the FUNCTION itself
// Prevents child from re-rendering because a new function reference is created
function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // Without useCallback: new function created every render → Child re-renders
  // With useCallback: same function reference → Child does NOT re-render
  const handleClick = useCallback(() => {
    setCount(c => c + 1);
  }, []);  // No dependencies — function never changes

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <ExpensiveChild onClick={handleClick} />
    </>
  );
}

const ExpensiveChild = React.memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click</button>;
});
```

| Hook | Caches | Use When |
|---|---|---|
| `useMemo` | Computed value | Expensive calculation, filtering, sorting |
| `useCallback` | Function reference | Passing callbacks to memoized child components |
| `React.memo` | Entire component | Prevent re-render when props haven't changed |

> **Interview tip:** Don't over-memoize. Every `useMemo`/`useCallback` call has overhead. Use them only when you have a **measurable performance problem**. Interviewers love asking "when would you NOT use useMemo?"

---

## 12. Custom Hooks

Custom hooks are **reusable functions** that use built-in hooks. They must start with `use`.

**Real-world analogy:** Custom hooks are like creating your own power tool. You combine basic tools (useState, useEffect) into a specialized tool for a specific job.

```jsx
// Custom hook for fetching data
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    fetch(url)
      .then(res => res.json())
      .then(data => { if (!cancelled) setData(data); })
      .catch(err => { if (!cancelled) setError(err.message); })
      .finally(() => { if (!cancelled) setLoading(false); });

    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Usage — much cleaner!
function UserProfile({ id }) {
  const { data: user, loading, error } = useFetch(`/api/users/${id}`);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <h1>{user.name}</h1>;
}

// Custom hook for local storage
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      return JSON.parse(localStorage.getItem(key)) ?? initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = (newValue) => {
    setValue(newValue);
    localStorage.setItem(key, JSON.stringify(newValue));
  };

  return [value, setStoredValue];
}

// Usage
const [theme, setTheme] = useLocalStorage('theme', 'light');

// Custom hook for window size
function useWindowSize() {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });

  useEffect(() => {
    const handleResize = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}
```

> **Interview tip:** Custom hooks demonstrate your ability to write clean, reusable code. Always mention that custom hooks must start with `use` and that they allow you to share stateful logic (not UI) between components.

---

## 13. Component Lifecycle

React components go through three phases: **Mount → Update → Unmount**

```
MOUNT                   UPDATE                  UNMOUNT
-----                   ------                  -------
Component created  →    Props/state changed  →  Component removed
useEffect(fn, [])       useEffect(fn, [dep])    useEffect cleanup fn
```

```jsx
// Functional component lifecycle with hooks
function MyComponent({ id }) {
  const [data, setData] = useState(null);

  // MOUNT — runs once after first render
  useEffect(() => {
    console.log('Mounted');
    return () => console.log('Unmounted');  // UNMOUNT cleanup
  }, []);

  // UPDATE — runs when `id` changes
  useEffect(() => {
    console.log('id changed to', id);
    fetchData(id).then(setData);
  }, [id]);

  return <div>{data}</div>;
}

// Class component lifecycle (for reference)
class MyComponent extends Component {
  componentDidMount() { /* same as useEffect(fn, []) */ }
  componentDidUpdate(prevProps, prevState) { /* same as useEffect(fn, [deps]) */ }
  componentWillUnmount() { /* same as cleanup function in useEffect */ }
}
```

---

## 14. Virtual DOM & Reconciliation

**Virtual DOM** is a lightweight JavaScript object representation of the real DOM. React uses it to minimize expensive real DOM operations.

**How it works:**
1. React renders the Virtual DOM (a JS tree)
2. On state/props change, React creates a **new** Virtual DOM tree
3. React **diffs** the old and new trees (reconciliation)
4. Only the **changed parts** are updated in the real DOM (diffing algorithm)

**Real-world analogy:** Instead of repainting an entire room every time you want to change one wall color, you create a blueprint of the room, find only what changed, and repaint just that wall.

```jsx
// React's diffing rules:
// 1. Elements of different types → destroy old tree, build new one
// 2. Same type elements → update only changed attributes
// 3. Lists → use KEY prop to identify which items changed

// WHY keys matter in lists:
// Without key: React has to re-render the entire list on change
// With key:    React knows exactly which item was added/removed/moved

// BAD — using index as key causes bugs with dynamic lists
{items.map((item, index) => <Item key={index} data={item} />)}

// GOOD — use stable, unique IDs
{items.map(item => <Item key={item.id} data={item} />)}
```

> **Interview tip:** This is a top 5 interview question. Explain: Virtual DOM → diff → batch DOM updates. Mention that React 18 introduced **Concurrent Mode** and **Fiber** architecture for better scheduling of renders.

---

## 15. Event Handling

```jsx
// React uses synthetic events (cross-browser wrapper around native events)

function Form() {
  const handleClick = (e) => {
    e.preventDefault();   // Prevent form submit / default behavior
    e.stopPropagation();  // Stop event from bubbling up
    console.log(e.target.value);
  };

  // Passing arguments to handlers
  const handleDelete = (id) => (e) => {
    e.stopPropagation();
    deleteItem(id);
  };

  return (
    <form onSubmit={handleClick}>
      <button onClick={handleDelete(42)}>Delete</button>
      <input onChange={(e) => console.log(e.target.value)} />
    </form>
  );
}

// Common React events:
// onClick, onDoubleClick
// onChange, onInput, onSubmit
// onKeyDown, onKeyUp, onKeyPress
// onMouseEnter, onMouseLeave, onMouseOver
// onFocus, onBlur
// onScroll, onWheel
// onDragStart, onDrop
```

---

## 16. Conditional Rendering

```jsx
function Dashboard({ user, isLoading, hasError }) {

  // 1. if/else (for complex logic)
  if (isLoading) return <Spinner />;
  if (hasError) return <ErrorPage />;

  // 2. Ternary operator (inline, two outcomes)
  return (
    <div>
      {user ? <UserProfile user={user} /> : <LoginPrompt />}

      {/* 3. && short-circuit (render or nothing) */}
      {user?.isAdmin && <AdminPanel />}

      {/* 4. Nullish coalescing for defaults */}
      <p>{user?.bio ?? 'No bio provided'}</p>
    </div>
  );
}
```

---

## 17. Lists & Keys

```jsx
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Learn React', done: false },
    { id: 2, text: 'Build project', done: true },
  ]);

  const toggleTodo = (id) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      )
    );
  };

  const removeTodo = (id) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>   {/* KEY must be stable and unique */}
          <span style={{ textDecoration: todo.done ? 'line-through' : 'none' }}>
            {todo.text}
          </span>
          <button onClick={() => toggleTodo(todo.id)}>Toggle</button>
          <button onClick={() => removeTodo(todo.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

---

## 18. Forms & Controlled Components

A **controlled component** is one where React controls the form element's value via state.

```jsx
// Controlled component — React is the "single source of truth"
function LoginForm() {
  const [formData, setFormData] = useState({ email: '', password: '' });
  const [errors, setErrors] = useState({});

  const handleChange = ({ target: { name, value } }) => {
    setFormData(prev => ({ ...prev, [name]: value }));
    setErrors(prev => ({ ...prev, [name]: '' }));  // Clear error on change
  };

  const validate = () => {
    const newErrors = {};
    if (!formData.email.includes('@')) newErrors.email = 'Invalid email';
    if (formData.password.length < 6) newErrors.password = 'Too short';
    return newErrors;
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const validationErrors = validate();
    if (Object.keys(validationErrors).length > 0) {
      setErrors(validationErrors);
      return;
    }
    console.log('Submit:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="email"
        type="email"
        value={formData.email}      {/* Controlled — value from state */}
        onChange={handleChange}
      />
      {errors.email && <span>{errors.email}</span>}

      <input
        name="password"
        type="password"
        value={formData.password}
        onChange={handleChange}
      />
      {errors.password && <span>{errors.password}</span>}

      <button type="submit">Login</button>
    </form>
  );
}

// Uncontrolled component — DOM manages its own state (use useRef)
function UncontrolledForm() {
  const inputRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log(inputRef.current.value);  // Read directly from DOM
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="initial" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

> **Interview tip:** Know the difference — **controlled** (React state drives the input), **uncontrolled** (DOM drives itself via ref). Controlled is preferred for validation. Libraries like **React Hook Form** use uncontrolled components for better performance.

---

## 19. React Router

React Router v6 is the standard routing library for React SPAs.

```bash
npm install react-router-dom
```

```jsx
import { BrowserRouter, Routes, Route, Link, useNavigate, useParams, useLocation } from 'react-router-dom';

// Setup
function App() {
  return (
    <BrowserRouter>
      <Navbar />
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/users/:id" element={<UserDetail />} />
        <Route path="/dashboard" element={<PrivateRoute><Dashboard /></PrivateRoute>} />
        <Route path="*" element={<NotFound />} />   {/* Catch-all 404 */}
      </Routes>
    </BrowserRouter>
  );
}

// Navigation
function Navbar() {
  return (
    <nav>
      <Link to="/">Home</Link>               {/* Declarative navigation */}
      <Link to="/about">About</Link>
    </nav>
  );
}

// URL params
function UserDetail() {
  const { id } = useParams();               // /users/42 → id = "42"
  const location = useLocation();           // Current location object
  const navigate = useNavigate();           // Programmatic navigation

  return (
    <div>
      <h1>User {id}</h1>
      <button onClick={() => navigate(-1)}>Back</button>
      <button onClick={() => navigate('/home', { replace: true })}>Home</button>
    </div>
  );
}

// Protected route pattern
function PrivateRoute({ children }) {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? children : <Navigate to="/login" replace />;
}
```

---

## 20. State Management — Redux & Zustand

### Redux Toolkit (Modern Redux)

```bash
npm install @reduxjs/toolkit react-redux
```

```jsx
// store/counterSlice.js
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },   // Immer allows "mutation"
    decrement: (state) => { state.value -= 1; },
    incrementByAmount: (state, action) => { state.value += action.payload; },
  },
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;

// store/index.js
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';

export const store = configureStore({
  reducer: { counter: counterReducer },
});

// main.jsx
import { Provider } from 'react-redux';
ReactDOM.createRoot(document.getElementById('root')).render(
  <Provider store={store}><App /></Provider>
);

// In a component
import { useSelector, useDispatch } from 'react-redux';

function Counter() {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();

  return (
    <>
      <p>{count}</p>
      <button onClick={() => dispatch(increment())}>+</button>
    </>
  );
}
```

### Zustand (Simpler alternative)

```bash
npm install zustand
```

```jsx
import { create } from 'zustand';

const useStore = create((set) => ({
  count: 0,
  increment: () => set(state => ({ count: state.count + 1 })),
  decrement: () => set(state => ({ count: state.count - 1 })),
}));

function Counter() {
  const { count, increment } = useStore();
  return <button onClick={increment}>{count}</button>;
}
```

| | Context API | Redux | Zustand |
|---|---|---|---|
| Setup | Built-in | Verbose | Minimal |
| DevTools | No | Yes | Yes |
| Performance | Re-renders all | Selective | Selective |
| Best for | Low-frequency global state | Large apps | Small/medium apps |

---

## 21. Performance Optimization

```jsx
// 1. React.memo — prevent re-render if props haven't changed
const ExpensiveComponent = React.memo(function({ data }) {
  return <div>{data.name}</div>;
});

// 2. useMemo — memoize expensive calculations
const sortedList = useMemo(() => [...items].sort(), [items]);

// 3. useCallback — stable function references for memoized children
const handleClick = useCallback(() => doSomething(id), [id]);

// 4. Code splitting — load components lazily
const HeavyPage = lazy(() => import('./HeavyPage'));

<Suspense fallback={<Loading />}>
  <HeavyPage />
</Suspense>

// 5. Virtualization — render only visible rows for long lists
// Use libraries: react-window or react-virtual
import { FixedSizeList } from 'react-window';

<FixedSizeList height={500} itemCount={10000} itemSize={35}>
  {({ index, style }) => <div style={style}>Row {index}</div>}
</FixedSizeList>

// 6. Avoid anonymous functions in JSX (creates new reference each render)
// BAD
<button onClick={() => handleClick(id)}>Click</button>

// BETTER — for lists
const handleClick = useCallback((id) => () => doDelete(id), []);
<button onClick={handleClick(id)}>Click</button>
```

---

## 22. Error Boundaries

Error Boundaries catch JavaScript errors in child components and display a fallback UI instead of crashing the entire app. **Must be class components** (no hook equivalent yet).

```jsx
class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    console.error('Error caught:', error, info.componentStack);
    // Send to error tracking: Sentry, LogRocket, etc.
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <h2>Something went wrong.</h2>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<ErrorPage />}>
  <MyComponent />
</ErrorBoundary>
```

> **Interview tip:** Error boundaries only catch errors in **rendering, lifecycle methods, and constructors** — NOT in event handlers or async code. Use try/catch for those.

---

## 23. Higher-Order Components (HOC)

An HOC is a function that takes a component and returns a **new enhanced component**.

**Real-world analogy:** HOC is like a coffee machine. You put in plain coffee (component), it adds milk and sugar (behavior), and returns enhanced coffee (component with extra features).

```jsx
// HOC for adding loading state
function withLoading(WrappedComponent) {
  return function WithLoadingComponent({ isLoading, ...props }) {
    if (isLoading) return <div>Loading...</div>;
    return <WrappedComponent {...props} />;
  };
}

// HOC for requiring authentication
function withAuth(WrappedComponent) {
  return function WithAuthComponent(props) {
    const { isAuthenticated } = useAuth();
    if (!isAuthenticated) return <Navigate to="/login" />;
    return <WrappedComponent {...props} />;
  };
}

// Usage
const UserListWithLoading = withLoading(UserList);
const ProtectedDashboard = withAuth(Dashboard);

<UserListWithLoading isLoading={loading} users={users} />
```

> **Interview tip:** HOCs were the pattern before hooks. Today, custom hooks solve most of the same problems more cleanly. Interviewers may ask you to compare HOC vs custom hooks.

---

## 24. Render Props Pattern

A technique where a component's prop is a **function that returns JSX**, allowing sharing of behavior.

```jsx
// Component with render prop
function MouseTracker({ render }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  const handleMouseMove = (e) => setPosition({ x: e.clientX, y: e.clientY });

  return (
    <div onMouseMove={handleMouseMove} style={{ height: '100vh' }}>
      {render(position)}
    </div>
  );
}

// Usage
<MouseTracker render={({ x, y }) => (
  <p>Mouse at ({x}, {y})</p>
)} />

// Children as a function (same pattern, different syntax)
<MouseTracker>
  {({ x, y }) => <p>Mouse at ({x}, {y})</p>}
</MouseTracker>
```

---

## 25. Code Splitting & Lazy Loading

```jsx
import { lazy, Suspense } from 'react';

// Dynamically import heavy components
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading page...</div>}>
        <Routes>
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// Each page is loaded only when the user navigates to it
// This reduces the initial bundle size significantly
```

---

## 26. Testing React Components

```bash
# Tools: Jest (test runner) + React Testing Library (RTL)
npm install --save-dev @testing-library/react @testing-library/jest-dom
```

```jsx
// Counter.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Counter from './Counter';

describe('Counter', () => {
  test('renders initial count of 0', () => {
    render(<Counter />);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });

  test('increments count on button click', async () => {
    const user = userEvent.setup();
    render(<Counter />);

    await user.click(screen.getByRole('button', { name: '+' }));
    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });

  test('fetches and displays user', async () => {
    // Mock API
    global.fetch = jest.fn(() =>
      Promise.resolve({ json: () => Promise.resolve({ name: 'Amith' }) })
    );

    render(<UserProfile id={1} />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();

    const name = await screen.findByText('Amith');  // Wait for async
    expect(name).toBeInTheDocument();
  });
});
```

**RTL Queries:**
```
getBy...     — throws if not found (for synchronous)
queryBy...   — returns null if not found (for asserting absence)
findBy...    — async, waits for element (for async operations)

ByRole, ByText, ByLabelText, ByPlaceholderText, ByTestId
```

> **Interview tip:** RTL encourages testing from the **user's perspective** (what they see/interact with), not implementation details. Know the query priority: `ByRole > ByLabelText > ByText > ByTestId`.

---

## 27. Common Interview Questions

### Q1: What is the difference between state and props?
**Answer:** Props are read-only inputs passed from parent to child — a child cannot modify its own props. State is mutable data owned by a component — when it changes, React re-renders that component. Props flow down (parent → child), state is local.

---

### Q2: Why shouldn't you mutate state directly?
**Answer:** React uses shallow reference comparison to detect state changes. If you mutate an object/array in place, the reference stays the same, so React thinks nothing changed and won't re-render. You must always return a new reference: `setItems([...items, newItem])`.

---

### Q3: What is the purpose of the key prop in lists?
**Answer:** Keys help React identify which items have changed, been added, or removed in a list. Without keys, React would have to re-render the entire list on any change. Keys should be stable, unique IDs — never use array indices for dynamic lists.

---

### Q4: What is the difference between useEffect and useLayoutEffect?
**Answer:** `useEffect` runs **after** the browser has painted the screen (asynchronous). `useLayoutEffect` runs **synchronously after DOM mutations but before** the browser paints. Use `useLayoutEffect` when you need to read/modify the DOM (e.g., measure element size) before the user sees any flash.

---

### Q5: Explain the React rendering process.
**Answer:** When state/props change, React creates a new Virtual DOM tree, diffs it against the previous one (reconciliation), and applies only the minimal set of changes to the real DOM (commit phase). React 18's concurrent mode can pause/interrupt renders to keep the UI responsive.

---

### Q6: What are the rules of Hooks?
**Answer:**
1. Only call hooks at the **top level** — never inside loops, conditions, or nested functions
2. Only call hooks from **React functions** (functional components or custom hooks)

These rules ensure hooks are called in the same order every render, which is how React maintains their state.

---

### Q7: How would you optimize a React app that's rendering slowly?
**Answer:**
1. Use `React.memo` to prevent unnecessary re-renders of child components
2. Use `useMemo` for expensive computations
3. Use `useCallback` for stable function references
4. Implement virtualization (`react-window`) for long lists
5. Code-split with `lazy()` and `Suspense` to reduce bundle size
6. Profile with React DevTools to identify bottlenecks

---

### Q8: What is prop drilling and how do you solve it?
**Answer:** Prop drilling is when you pass props through many layers of components just to get data to a deeply nested child. Solutions: Context API (for low-frequency global state), Redux/Zustand (for complex state), or component composition.

---

### Q9: Controlled vs Uncontrolled components?
**Answer:** In a **controlled** component, React state is the source of truth for form values. In an **uncontrolled** component, the DOM manages its own state (accessed via ref). Controlled is preferred for validation and real-time feedback. Uncontrolled (used by React Hook Form) is faster as it avoids re-renders on every keystroke.

---

### Q10: What is React Strict Mode?
**Answer:** `<React.StrictMode>` is a development tool that intentionally double-invokes renders and effects to help detect side effects in impure render functions. It does NOT affect production builds.

---

## 28. Quick Revision Cheat Sheet

```
HOOKS
-----
useState(init)              → [value, setter] — local state
useEffect(fn, [deps])       → side effects (fetch, subscriptions, timers)
useRef(init)                → .current — DOM ref or mutable value (no re-render)
useContext(Context)         → consume context value
useReducer(reducer, init)   → [state, dispatch] — complex state logic
useMemo(() => val, [deps])  → memoized computed value
useCallback(fn, [deps])     → memoized function reference
useLayoutEffect(fn, [deps]) → like useEffect but fires before paint

EFFECT DEPENDENCY PATTERNS
---------------------------
useEffect(fn)          → runs after EVERY render
useEffect(fn, [])      → runs ONCE on mount
useEffect(fn, [a, b])  → runs when a or b changes
return () => {}        → cleanup (runs on unmount or before re-run)

STATE UPDATE RULES
------------------
- Never mutate state directly
- Use functional form when new state depends on old: setCount(c => c + 1)
- Batch updates: React 18 batches all state updates automatically

RENDERING
---------
- Component re-renders when: state changes, parent re-renders, context changes
- React.memo: wraps component, skips re-render if props unchanged
- Keys in lists: must be stable, unique IDs (not array index)

COMMON PATTERNS
---------------
Lifting state up     → Move shared state to closest common parent
Prop drilling fix    → Context API or state management library
Composition          → Pass components as props/children instead of deep nesting
Guard clause         → if (loading) return <Spinner /> early

REACT ECOSYSTEM
---------------
Routing              → React Router v6
State management     → Redux Toolkit, Zustand, Jotai
Forms                → React Hook Form, Formik
Styling              → Tailwind CSS, CSS Modules, styled-components
Data fetching        → React Query (TanStack Query), SWR
Testing              → Jest + React Testing Library
Build tool           → Vite (fast), CRA (legacy)
Meta-framework       → Next.js (SSR/SSG + routing)
```

---

> **Final Interview Tip:** For frontend job interviews, build at least **2-3 real projects** (todo app, weather app, e-commerce cart) and be ready to walk through the code. Interviewers value hands-on understanding over memorized theory. Know how to use React DevTools to debug re-renders and understand the component tree.

Good luck with your job search!
