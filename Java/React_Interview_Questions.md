# React.js Interview Preparation Guide

> A guide covering React concepts for junior developers, with code examples and interview tips.

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
23. [Code Splitting & Lazy Loading](#23-code-splitting--lazy-loading)
24. [Testing React Components](#24-testing-react-components)
25. [Common Interview Questions](#25-common-interview-questions)
26. [Quick Revision Cheat Sheet](#26-quick-revision-cheat-sheet)

---

## 1. What is React?

React is a **JavaScript library** (not a framework) built by Meta for building **user interfaces** in single-page applications (SPAs).

**Key features:**
- **Component-based** — UI split into small, reusable pieces (like LEGO bricks)
- **Declarative** — describe WHAT the UI looks like, not HOW to update it
- **Virtual DOM** — lightweight copy of the real DOM for performance
- **One-way data flow** — data flows from parent to child via props

```bash
# Create a new React app (Vite is preferred)
npm create vite@latest my-app -- --template react
cd my-app && npm install && npm run dev
```

> **Interview tip:** React is a **library**, not a framework. Angular is a framework. React only handles the View layer.

---

## 2. JSX

JSX (JavaScript XML) lets you write HTML-like code inside JavaScript.

```jsx
const element = <h1 className="title">Hello, World!</h1>;

// Compiles to:
const element = React.createElement('h1', { className: 'title' }, 'Hello, World!');

// JSX rules:
// 1. Single root element — wrap with <> </> (Fragment)
// 2. Use className instead of class
// 3. Self-closing tags: <input />
// 4. JavaScript expressions: {}

const greeting = (
  <>
    <h1>Hello, {name}!</h1>
    <p>Today is {new Date().toDateString()}</p>
  </>
);
```

> **Interview tip:** JSX is syntactic sugar over `React.createElement()`. Since React 17+, you don't need `import React from 'react'` due to the new JSX transform.

---

## 3. Components — Functional vs Class

A **component** is a reusable piece of UI that accepts props and returns JSX.

```jsx
// Functional (modern — preferred)
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// Class (legacy — still common in production)
class Greeting extends Component {
  render() {
    return <h1>Hello, {this.props.name}!</h1>;
  }
}
```

| Feature | Functional | Class |
|---|---|---|
| State | `useState` hook | `this.state` |
| Lifecycle | `useEffect` | `componentDidMount`, etc. |
| `this` keyword | Not needed | Required |
| Modern usage | Preferred | Legacy |

> **Interview tip:** Always prefer functional components with hooks. Know class components because many production codebases still use them.

---

## 4. Props

Props are **read-only** inputs passed from parent to child. They flow **one-way** — parent to child only.

```jsx
function App() {
  return <UserCard name="Amith" age={25} isAdmin={true} onClick={() => alert('clicked')} />;
}

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

// Children prop
function Card({ children, title }) {
  return <div className="card"><h3>{title}</h3>{children}</div>;
}
// Usage: <Card title="My Card"><p>content</p></Card>
```

> **Interview tip:** Props are **immutable** — a child should never modify its own props. If data needs to change, lift state up to the parent.

---

## 5. State

State is **mutable data** owned by a component. When state changes, React **re-renders** that component.

```
Props  = data FROM parent (read-only)
State  = data INSIDE component (mutable, triggers re-render)

Use state for: counters, form values, toggles, fetched API data
```

---

## 6. useState Hook

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);  // [value, setter]

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}

// When new state depends on old state, use the functional form:
setCount(prev => prev + 1);  // Safe — always uses latest value

// State with objects — spread to preserve other fields
const [user, setUser] = useState({ name: '', email: '' });
setUser(prev => ({ ...prev, [e.target.name]: e.target.value }));
```

**Common mistakes:**
```jsx
// WRONG — direct mutation doesn't trigger re-render
items.push('new item');
user.name = 'John';

// RIGHT — always create a new reference
setItems(prev => [...prev, 'new item']);
setUser(prev => ({ ...prev, name: 'John' }));
```

> **Interview tip:** React uses **shallow comparison** — mutating in place keeps the same reference, so React won't detect the change and won't re-render.

---

## 7. useEffect Hook

`useEffect` handles **side effects** outside of rendering: API calls, subscriptions, timers.

```jsx
// 1. Runs after EVERY render (no dependency array)
useEffect(() => { console.log('rendered'); });

// 2. Runs ONCE on mount
useEffect(() => { fetchData(); }, []);

// 3. Runs when specific values change
useEffect(() => { fetchUser(userId); }, [userId]);

// 4. Cleanup — prevents memory leaks
useEffect(() => {
  const timer = setInterval(() => console.log('tick'), 1000);
  return () => clearInterval(timer);  // runs on unmount
}, []);

// Real example — fetch with cleanup
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => { if (!cancelled) { setUser(data); setLoading(false); } });
    return () => { cancelled = true; };
  }, [userId]);

  if (loading) return <p>Loading...</p>;
  return <div>{user?.name}</div>;
}
```

> **Interview tip:** Know the three dependency forms and understand **cleanup** — memory leaks from uncleared subscriptions/timers are a classic interview topic.

---

## 8. useRef Hook

`useRef` gives a **mutable reference** that persists across renders WITHOUT causing re-renders.

```jsx
// Use case 1: DOM access
function FocusInput() {
  const inputRef = useRef(null);
  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={() => inputRef.current.focus()}>Focus</button>
    </>
  );
}

// Use case 2: Store a value without triggering re-render
function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef(null);
  const start = () => { intervalRef.current = setInterval(() => setCount(c => c + 1), 1000); };
  const stop = () => { clearInterval(intervalRef.current); };
  return <><p>{count}</p><button onClick={start}>Start</button><button onClick={stop}>Stop</button></>;
}

// useRef vs useState:
// useRef  — .current is mutable, does NOT re-render
// useState — causes re-render when updated
```

> **Interview tip:** `ref.current` is mutable with no re-render; state changes trigger a re-render. Know which to use when.

---

## 9. useContext & Context API

Context lets you **share data across components** without prop drilling.

```jsx
import { createContext, useContext, useState } from 'react';

// Step 1: Create
const ThemeContext = createContext('light');

// Step 2: Provide
function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Navbar />
    </ThemeContext.Provider>
  );
}

// Step 3: Consume anywhere in the tree
function Navbar() {
  const { theme, setTheme } = useContext(ThemeContext);
  return (
    <nav style={{ background: theme === 'dark' ? '#333' : '#fff' }}>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>Toggle</button>
    </nav>
  );
}
```

**When NOT to use Context:** For frequently changing data (e.g., every keystroke) — it re-renders ALL consumers on every change. Use Zustand/Redux instead.

> **Interview tip:** Context solves **prop drilling** but is not a replacement for Redux. Mention this distinction.

---

## 10. useReducer Hook

`useReducer` is an alternative to `useState` for **complex state logic** with multiple transitions.

```jsx
function counterReducer(state, action) {
  switch (action.type) {
    case 'INCREMENT': return { count: state.count + 1 };
    case 'DECREMENT': return { count: state.count - 1 };
    case 'RESET':     return { count: 0 };
    case 'SET':       return { count: action.payload };
    default: throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });
  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'SET', payload: 10 })}>Set 10</button>
    </div>
  );
}

// useState   — simple, independent values
// useReducer — complex state, many fields, transitions depend on action type
```

---

## 11. useMemo & useCallback

These hooks **optimize performance** by caching computed values and function references.

```jsx
// useMemo — caches the RESULT of a computation
const filteredProducts = useMemo(() =>
  products.filter(p => p.name.toLowerCase().includes(filterText.toLowerCase())),
  [products, filterText]
);

// useCallback — caches the FUNCTION reference
// Prevents memoized child from re-rendering due to new function reference
const handleClick = useCallback(() => {
  setCount(c => c + 1);
}, []);

const ExpensiveChild = React.memo(({ onClick }) => (
  <button onClick={onClick}>Click</button>
));
```

| Hook | Caches | Use When |
|---|---|---|
| `useMemo` | Computed value | Expensive calculation, filtering, sorting |
| `useCallback` | Function reference | Passing callbacks to memoized children |
| `React.memo` | Entire component | Prevent re-render when props unchanged |

> **Interview tip:** Don't over-memoize — every call has overhead. Use only when you have a **measurable performance problem**. Interviewers often ask "when would you NOT use useMemo?"

---

## 12. Custom Hooks

Custom hooks are **reusable functions** that use built-in hooks. They must start with `use`.

```jsx
// useFetch — reusable data fetching
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

// Usage
function UserProfile({ id }) {
  const { data: user, loading, error } = useFetch(`/api/users/${id}`);
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <h1>{user.name}</h1>;
}

// useLocalStorage
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try { return JSON.parse(localStorage.getItem(key)) ?? initialValue; }
    catch { return initialValue; }
  });
  const setStoredValue = (newValue) => {
    setValue(newValue);
    localStorage.setItem(key, JSON.stringify(newValue));
  };
  return [value, setStoredValue];
}
```

> **Interview tip:** Custom hooks share **stateful logic** (not UI) between components. Must start with `use`.

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
function MyComponent({ id }) {
  // MOUNT — runs once
  useEffect(() => {
    console.log('Mounted');
    return () => console.log('Unmounted');  // UNMOUNT
  }, []);

  // UPDATE — runs when id changes
  useEffect(() => {
    fetchData(id);
  }, [id]);
}

// Class equivalents (for reference):
// componentDidMount()      → useEffect(fn, [])
// componentDidUpdate()     → useEffect(fn, [deps])
// componentWillUnmount()   → cleanup return in useEffect
```

---

## 14. Virtual DOM & Reconciliation

**Virtual DOM** is a lightweight JS object tree mirroring the real DOM. React uses it to minimize expensive real DOM operations.

**How it works:**
1. React renders a Virtual DOM tree
2. On state/props change, a **new** Virtual DOM tree is created
3. React **diffs** old vs new (reconciliation)
4. Only **changed parts** are updated in the real DOM

```jsx
// Diffing rules:
// - Different element types → rebuild entire subtree
// - Same type → update only changed attributes
// - Lists → use KEY prop to identify changes

// BAD — index as key causes bugs with dynamic lists
{items.map((item, index) => <Item key={index} data={item} />)}

// GOOD — stable unique IDs
{items.map(item => <Item key={item.id} data={item} />)}
```

> **Interview tip:** Top 5 interview question. Explain: Virtual DOM → diff → batch DOM updates. React 18 introduced **Concurrent Mode** and **Fiber** for better render scheduling.

---

## 15. Event Handling

```jsx
function Form() {
  const handleSubmit = (e) => {
    e.preventDefault();   // prevent default browser behavior
    e.stopPropagation();  // stop event bubbling
  };

  // Passing arguments to handlers
  const handleDelete = (id) => (e) => {
    e.stopPropagation();
    deleteItem(id);
  };

  return (
    <form onSubmit={handleSubmit}>
      <button onClick={handleDelete(42)}>Delete</button>
      <input onChange={(e) => console.log(e.target.value)} />
    </form>
  );
}
// Common events: onClick, onChange, onSubmit, onKeyDown, onFocus, onBlur, onMouseEnter
```

---

## 16. Conditional Rendering

```jsx
function Dashboard({ user, isLoading, hasError }) {
  // 1. if/else for complex logic
  if (isLoading) return <Spinner />;
  if (hasError) return <ErrorPage />;

  return (
    <div>
      {/* 2. Ternary — two outcomes */}
      {user ? <UserProfile user={user} /> : <LoginPrompt />}

      {/* 3. && short-circuit — render or nothing */}
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

  const toggleTodo = (id) =>
    setTodos(prev => prev.map(t => t.id === id ? { ...t, done: !t.done } : t));

  const removeTodo = (id) =>
    setTodos(prev => prev.filter(t => t.id !== id));

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <span style={{ textDecoration: todo.done ? 'line-through' : 'none' }}>{todo.text}</span>
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

A **controlled component** is one where React state drives the form value.

```jsx
function LoginForm() {
  const [formData, setFormData] = useState({ email: '', password: '' });
  const [errors, setErrors] = useState({});

  const handleChange = ({ target: { name, value } }) =>
    setFormData(prev => ({ ...prev, [name]: value }));

  const validate = () => {
    const errs = {};
    if (!formData.email.includes('@')) errs.email = 'Invalid email';
    if (formData.password.length < 6) errs.password = 'Too short';
    return errs;
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const errs = validate();
    if (Object.keys(errs).length > 0) { setErrors(errs); return; }
    console.log('Submit:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" value={formData.email} onChange={handleChange} />
      {errors.email && <span>{errors.email}</span>}
      <input name="password" type="password" value={formData.password} onChange={handleChange} />
      {errors.password && <span>{errors.password}</span>}
      <button type="submit">Login</button>
    </form>
  );
}
```

> **Interview tip:** **Controlled** = React state is source of truth. **Uncontrolled** = DOM manages itself via `ref`. Controlled is preferred for validation. React Hook Form uses uncontrolled components for performance.

---

## 19. React Router

```bash
npm install react-router-dom
```

```jsx
import { BrowserRouter, Routes, Route, Link, useNavigate, useParams } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <nav><Link to="/">Home</Link><Link to="/about">About</Link></nav>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/users/:id" element={<UserDetail />} />
        <Route path="/dashboard" element={<PrivateRoute><Dashboard /></PrivateRoute>} />
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}

function UserDetail() {
  const { id } = useParams();           // /users/42 → id = "42"
  const navigate = useNavigate();
  return (
    <div>
      <h1>User {id}</h1>
      <button onClick={() => navigate(-1)}>Back</button>
    </div>
  );
}

function PrivateRoute({ children }) {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? children : <Navigate to="/login" replace />;
}
```

---

## 20. State Management — Redux & Zustand

### Redux Toolkit (Modern Redux)

```jsx
// counterSlice.js
import { createSlice } from '@reduxjs/toolkit';
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },
    decrement: (state) => { state.value -= 1; },
    incrementByAmount: (state, action) => { state.value += action.payload; },
  },
});
export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;

// store.js
import { configureStore } from '@reduxjs/toolkit';
export const store = configureStore({ reducer: { counter: counterReducer } });

// Component
function Counter() {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();
  return <><p>{count}</p><button onClick={() => dispatch(increment())}>+</button></>;
}
```

### Zustand (Simpler alternative)

```jsx
const useStore = create((set) => ({
  count: 0,
  increment: () => set(state => ({ count: state.count + 1 })),
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
| Performance | Re-renders all consumers | Selective | Selective |
| Best for | Low-frequency global state | Large apps | Small/medium apps |

---

## 21. Performance Optimization

```jsx
// 1. React.memo — skip re-render if props unchanged
const ExpensiveComponent = React.memo(function({ data }) {
  return <div>{data.name}</div>;
});

// 2. useMemo — memoize expensive computation
const sortedList = useMemo(() => [...items].sort(), [items]);

// 3. useCallback — stable function for memoized children
const handleClick = useCallback(() => doSomething(id), [id]);

// 4. Code splitting — load components on demand
const HeavyPage = lazy(() => import('./HeavyPage'));
<Suspense fallback={<Loading />}><HeavyPage /></Suspense>

// 5. Virtualization — render only visible rows in long lists
import { FixedSizeList } from 'react-window';
<FixedSizeList height={500} itemCount={10000} itemSize={35}>
  {({ index, style }) => <div style={style}>Row {index}</div>}
</FixedSizeList>
```

---

## 22. Error Boundaries

Error Boundaries catch JS errors in child components and show a fallback instead of crashing the app. **Must be class components.**

```jsx
class ErrorBoundary extends Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    console.error('Error caught:', error, info.componentStack);
  }

  render() {
    if (this.state.hasError) return this.props.fallback || <h2>Something went wrong.</h2>;
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<ErrorPage />}><MyComponent /></ErrorBoundary>
```

> **Interview tip:** Error boundaries only catch errors in **rendering, lifecycle, and constructors** — NOT in event handlers or async code. Use try/catch for those.

---

## 23. Code Splitting & Lazy Loading

```jsx
import { lazy, Suspense } from 'react';

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
// Each page loads only when the user navigates to it — reduces initial bundle size
```

---

## 24. Testing React Components

```bash
npm install --save-dev @testing-library/react @testing-library/jest-dom
```

```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Counter from './Counter';

describe('Counter', () => {
  test('renders initial count of 0', () => {
    render(<Counter />);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });

  test('increments on click', async () => {
    const user = userEvent.setup();
    render(<Counter />);
    await user.click(screen.getByRole('button', { name: '+' }));
    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });

  test('fetches and displays user', async () => {
    global.fetch = jest.fn(() =>
      Promise.resolve({ json: () => Promise.resolve({ name: 'Amith' }) })
    );
    render(<UserProfile id={1} />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
    expect(await screen.findByText('Amith')).toBeInTheDocument();
  });
});
```

**RTL Query types:**
```
getBy...   — throws if not found (synchronous)
queryBy... — returns null if not found (assert absence)
findBy...  — async, waits for element

Priority: ByRole > ByLabelText > ByText > ByTestId
```

> **Interview tip:** RTL tests from the **user's perspective** — what they see and interact with, not implementation details.

---

## 25. Common Interview Questions

### Q1: What is the difference between state and props?
Props are read-only inputs passed from parent to child — a child cannot modify them. State is mutable data owned by the component — when it changes, React re-renders. Props flow down; state is local.

---

### Q2: Why shouldn't you mutate state directly?
React uses shallow reference comparison to detect changes. Mutating in place keeps the same reference, so React thinks nothing changed and won't re-render. Always return a new reference: `setItems([...items, newItem])`.

---

### Q3: What is the purpose of the key prop in lists?
Keys help React identify which items changed, were added, or removed. Without them, React re-renders the entire list. Keys must be stable, unique IDs — never use array indices for dynamic lists.

---

### Q4: What is the difference between useEffect and useLayoutEffect?
`useEffect` runs **after** the browser paints (async). `useLayoutEffect` runs **synchronously after DOM mutations, before** paint. Use `useLayoutEffect` when you need to measure or modify the DOM before the user sees it.

---

### Q5: Explain the React rendering process.
On state/props change, React creates a new Virtual DOM tree, diffs it against the previous one (reconciliation), and applies only the minimal changes to the real DOM. React 18's concurrent mode can pause renders to keep the UI responsive.

---

### Q6: What are the rules of Hooks?
1. Only call hooks at the **top level** — never inside loops, conditions, or nested functions.
2. Only call hooks from **React functions** (functional components or custom hooks).

These rules ensure hooks are called in the same order every render.

---

### Q7: How would you optimize a slow React app?
Use `React.memo` to prevent unnecessary child re-renders, `useMemo` for expensive computations, `useCallback` for stable function references, `react-window` for long list virtualization, and `lazy()`/`Suspense` for code splitting. Profile first with React DevTools.

---

### Q8: What is prop drilling and how do you solve it?
Prop drilling is passing props through many layers just to reach a deeply nested child. Solutions: Context API (low-frequency global state), Redux/Zustand (complex state), or component composition.

---

### Q9: Controlled vs Uncontrolled components?
**Controlled**: React state is the source of truth for form values — preferred for validation. **Uncontrolled**: the DOM manages its own state via ref — faster since it avoids re-renders on every keystroke (used by React Hook Form).

---

### Q10: What is React Strict Mode?
`<React.StrictMode>` intentionally double-invokes renders and effects in development to detect side effects in impure functions. It has no effect on production builds.

---

## 26. Quick Revision Cheat Sheet

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
- React 18 batches all state updates automatically

RENDERING
---------
- Component re-renders when: state changes, parent re-renders, context changes
- React.memo: skips re-render if props unchanged
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

> **Final Interview Tip:** Build 2-3 real projects (todo app, weather app, e-commerce cart) and be ready to walk through the code. Interviewers value hands-on understanding over memorized theory. Know how to use React DevTools to debug re-renders.

Good luck with your job search!

---

*Last Updated: 2026-06-18*
