# React — Every Concept Explained Simply

> A complete, beginner-friendly walkthrough of every core React concept — from components and JSX all the way through Suspense and Error Boundaries. Each idea is paired with simple analogies, working examples, and common pitfalls so you can build a real mental model, not just memorize syntax.

---

## Table of Contents

1. [What Is React & Why It Exists](#1-what-is-react--why-it-exists)
2. [Components — The Lego Pieces](#2-components--the-lego-pieces)
3. [JSX — JavaScript in Disguise](#3-jsx--javascript-in-disguise)
4. [Props — Passing Data Down](#4-props--passing-data-down)
5. [Composition & the `children` Prop](#5-composition--the-children-prop)
6. [The `key` Prop & Rendering Lists](#6-the-key-prop--rendering-lists)
7. [Rendering & the Virtual DOM](#7-rendering--the-virtual-dom)
8. [Reconciliation & Diffing](#8-reconciliation--diffing)
9. [State & `useState`](#9-state--usestate)
10. [Event Handling](#10-event-handling)
11. [Controlled vs Uncontrolled Components](#11-controlled-vs-uncontrolled-components)
12. [Hooks — The Rules & The Core Set](#12-hooks--the-rules--the-core-set)
13. [Purity & Pure Components](#13-purity--pure-components)
14. [Strict Mode](#14-strict-mode)
15. [Side Effects & `useEffect`](#15-side-effects--useeffect)
16. [Refs & `useRef`](#16-refs--useref)
17. [Context — Avoiding Prop Drilling](#17-context--avoiding-prop-drilling)
18. [Portals — Rendering Outside the Tree](#18-portals--rendering-outside-the-tree)
19. [Suspense — Showing Fallbacks While Loading](#19-suspense--showing-fallbacks-while-loading)
20. [Error Boundaries — Catching Render Errors](#20-error-boundaries--catching-render-errors)
21. [Other Hooks You'll Meet (`useMemo`, `useCallback`, `useReducer`, `useContext`)](#21-other-hooks-youll-meet)
22. [Custom Hooks](#22-custom-hooks)
23. [Performance — Memoization & Re-render Avoidance](#23-performance--memoization--re-render-avoidance)
24. [Putting It All Together — A Tiny App](#24-putting-it-all-together--a-tiny-app)
25. [Common Pitfalls & Interview Q&A](#25-common-pitfalls--interview-qa)

---

## 1. What Is React & Why It Exists

**React** is a JavaScript **library** (not a framework) for building **user interfaces**, created by Facebook in 2013. Instead of manually telling the browser *how* to update the DOM step-by-step, you describe *what* the UI should look like for a given state — and React figures out the minimum changes needed.

### The core idea in one sentence

> **UI = f(state)** — the screen is a pure function of your data.

When the data changes, you don't write DOM-mutation code. You just hand React a new description of the UI, and it updates the page for you.

### Why this matters

Before React, updating UIs meant code like:

```js
document.getElementById('count').innerText = newCount;
document.getElementById('button').classList.add('active');
```

Imagine doing this across hundreds of elements. Bugs everywhere. React replaces this with:

```jsx
<div>{count}</div>
```

You change `count`, the DOM updates. Done.

---

## 2. Components — The Lego Pieces

A **component** is a reusable, self-contained piece of UI — like a Lego brick. You build big UIs by snapping small components together.

A component is just a **JavaScript function** that returns JSX (markup).

```jsx
function Welcome() {
  return <h1>Hello, world!</h1>;
}
```

You use it like an HTML tag:

```jsx
<Welcome />
```

### Rules

1. **Component names must start with a capital letter** — `<Button />` is a component; `<button />` is an HTML element. React uses this to tell them apart.
2. **Return exactly one root element** (or a `<>...</>` fragment).
3. Components should be **pure** — same inputs → same output (more on this in §13).

### Analogy

Think of a webpage like a Lego castle. Each tower, wall, and door is a separate brick. You can reuse the same "window" brick 20 times — that's a React component.

```jsx
function App() {
  return (
    <div>
      <Header />
      <Sidebar />
      <Content />
      <Footer />
    </div>
  );
}
```

---

## 3. JSX — JavaScript in Disguise

**JSX** lets you write HTML-like markup inside JavaScript. It looks like HTML but is actually JavaScript:

```jsx
const element = <h1 className="title">Hello</h1>;
```

Under the hood, the build tool (Babel/Vite/Next) compiles this to:

```js
const element = React.createElement('h1', { className: 'title' }, 'Hello');
```

You don't have to write `createElement` by hand — that's the whole point of JSX.

### JSX rules (the ones beginners trip on)

| HTML | JSX | Why |
|---|---|---|
| `class="..."` | `className="..."` | `class` is a reserved word in JS |
| `for="..."` | `htmlFor="..."` | Same reason |
| `onclick="..."` | `onClick={...}` | camelCase, value is a JS function |
| `tabindex` | `tabIndex` | camelCase |
| `style="color: red"` | `style={{ color: 'red' }}` | Object, not string; camelCase props |

### Embedding JavaScript

Curly braces `{}` switch from JSX into JS:

```jsx
const name = 'Abhishek';
const age = 30;

return (
  <div>
    <h1>Hello, {name}!</h1>
    <p>You are {age * 2} years old in dog years.</p>
    <p>{age >= 18 ? 'Adult' : 'Minor'}</p>
  </div>
);
```

### Fragments

A component must return one element. To return multiple without an extra wrapper:

```jsx
return (
  <>
    <h1>Title</h1>
    <p>Paragraph</p>
  </>
);
```

`<>...</>` is a **fragment** — it produces no DOM node.

---

## 4. Props — Passing Data Down

**Props** ("properties") are how parent components pass data to children. They look like HTML attributes but can hold any JavaScript value: strings, numbers, objects, functions, even other components.

```jsx
function Greeting({ name, age }) {
  return <p>Hi {name}, you are {age}.</p>;
}

// Usage
<Greeting name="Abhishek" age={30} />
```

### Key points

- **Props are read-only.** A child must never modify its props. If it needs to change something, it asks the parent (via a callback prop).
- **Data flows one way** — from parent down to child ("unidirectional data flow"). This makes apps predictable.
- Use **destructuring** to make the function signature clean: `function Card({ title, body })` instead of `function Card(props)`.

### Passing functions as props

```jsx
function Button({ label, onClick }) {
  return <button onClick={onClick}>{label}</button>;
}

function App() {
  return <Button label="Click me" onClick={() => alert('Hi!')} />;
}
```

### Default props (modern way — defaults in destructuring)

```jsx
function Avatar({ size = 64, src }) {
  return <img src={src} width={size} height={size} />;
}
```

---

## 5. Composition & the `children` Prop

React doesn't have inheritance like classical OOP — it uses **composition**. The special `children` prop holds whatever JSX you nest inside a component.

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}

// Usage
<Card>
  <h2>Title</h2>
  <p>Some body text.</p>
</Card>
```

`Card` doesn't know or care what's inside it — it just provides the box. The parent decides the contents.

### Why this is powerful

Layout components, modals, panels, tooltips — all use `children`:

```jsx
function Modal({ title, children, onClose }) {
  return (
    <div className="modal">
      <header>
        <h2>{title}</h2>
        <button onClick={onClose}>×</button>
      </header>
      <div className="body">{children}</div>
    </div>
  );
}

// Usage
<Modal title="Confirm" onClose={() => setOpen(false)}>
  <p>Are you sure you want to delete this?</p>
  <button>Yes</button>
</Modal>
```

The same `Modal` can hold anything.

---

## 6. The `key` Prop & Rendering Lists

When you render a list, React needs a way to identify each item across renders — so it can tell "this item moved" vs "this item is new". That's the `key` prop.

```jsx
const fruits = ['apple', 'banana', 'cherry'];

return (
  <ul>
    {fruits.map((fruit) => (
      <li key={fruit}>{fruit}</li>
    ))}
  </ul>
);
```

### Rules for keys

1. **Must be unique among siblings** (not globally).
2. **Should be stable** — don't change between renders.
3. **Prefer real IDs** from your data (`item.id`), not array indexes.

### Why not array index?

If the list is reordered or items inserted at the top, using `index` as the key confuses React — it thinks items changed when they only moved. This causes subtle bugs with form state, animations, and focus.

```jsx
// ❌ Bad if list can change order
{items.map((item, i) => <Row key={i} data={item} />)}

// ✅ Good
{items.map((item) => <Row key={item.id} data={item} />)}
```

---

## 7. Rendering & the Virtual DOM

When state changes, React re-runs your component function and gets a new "description" of the UI — a tree of JavaScript objects. This is the **Virtual DOM**.

### Why a virtual DOM?

Manipulating the real DOM is slow. Manipulating plain JS objects is fast. React:

1. Builds a new virtual tree from your components.
2. **Diffs** it against the previous virtual tree.
3. Applies **only the minimum changes** to the real DOM.

```
state changes
   │
   ▼
re-run component → new virtual DOM
   │
   ▼
diff against previous virtual DOM
   │
   ▼
patch real DOM with only what's different
```

### What triggers a render?

- A component's **state** changes (via `useState`).
- A component's **props** change (because parent re-rendered).
- A `useContext` value changes.
- The parent re-renders (by default, children re-render too).

### Important: re-render ≠ DOM update

React re-runs your function on every render. But it only touches the real DOM where the virtual DOM diff says something changed. Re-rendering is cheap; DOM updates are not.

---

## 8. Reconciliation & Diffing

**Reconciliation** is React's process of figuring out what changed and updating the DOM accordingly. The algorithm relies on two assumptions:

1. **Different element types → different subtrees.** `<div>` replaced by `<span>` → throw away the whole subtree and rebuild.
2. **Stable `key` props** identify which children are which across renders.

### Practical example

```jsx
// Before
<ul>
  <li key="a">Apple</li>
  <li key="b">Banana</li>
</ul>

// After
<ul>
  <li key="c">Cherry</li>
  <li key="a">Apple</li>
  <li key="b">Banana</li>
</ul>
```

React sees that `a` and `b` still exist (matched by key) and a new `c` was inserted at the front. It inserts one `<li>` instead of rebuilding the whole list.

Without proper keys, React would naively assume every item changed.

---

## 9. State & `useState`

**State** is data that changes over time and causes the UI to update. When state changes, React re-renders the component.

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

### How `useState` works

- Call it inside a component to declare a "state variable".
- It returns an array: `[currentValue, setterFunction]`.
- The setter triggers a re-render with the new value.

### Functional updates (when next state depends on previous)

```jsx
// ❌ Risky — `count` might be stale if many updates happen quickly
setCount(count + 1);

// ✅ Safer — React passes the latest value
setCount((c) => c + 1);
```

### State is per-component-instance

If you render `<Counter />` twice, each has its own independent `count`. State is isolated.

### State updates are asynchronous and batched

```jsx
function handleClick() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
  // After all three, count is only +1, because each setCount used the same `count`
}

// To increment by 3, use the function form:
function handleClick() {
  setCount((c) => c + 1);
  setCount((c) => c + 1);
  setCount((c) => c + 1);
}
```

### State must be immutable (don't mutate)

```jsx
// ❌ Mutating — React won't detect the change
const [user, setUser] = useState({ name: 'Abhi', age: 30 });
user.age = 31;        // mutation
setUser(user);        // same reference; no re-render

// ✅ Create a new object
setUser({ ...user, age: 31 });
```

Same for arrays — use `[...arr, newItem]`, `arr.filter(...)`, `arr.map(...)`, never `.push()` / `.splice()`.

---

## 10. Event Handling

React events look like DOM events but with **camelCase names** and **functions as handlers**, not strings.

```jsx
<button onClick={handleClick}>Click</button>

function handleClick(e) {
  console.log('Clicked!', e);
}
```

### Common events

| Event | When |
|---|---|
| `onClick` | Click / tap |
| `onChange` | Input value changes |
| `onSubmit` | Form submitted |
| `onMouseEnter` / `onMouseLeave` | Hover |
| `onKeyDown` / `onKeyUp` | Key pressed |
| `onFocus` / `onBlur` | Focus changes |

### Passing arguments to handlers

```jsx
// Don't call the function — pass a reference
<button onClick={() => deleteItem(item.id)}>Delete</button>

// ❌ This calls immediately on render
<button onClick={deleteItem(item.id)}>Delete</button>
```

### Preventing default behavior

```jsx
function handleSubmit(e) {
  e.preventDefault();   // stops the page from reloading
  // do your thing
}

<form onSubmit={handleSubmit}>...</form>
```

### Synthetic events

React wraps native events in a cross-browser **SyntheticEvent**, so behavior is consistent everywhere. The API matches the native DOM event for the common properties (`e.target`, `e.preventDefault()`, `e.key`).

---

## 11. Controlled vs Uncontrolled Components

For form inputs, you choose who owns the value: React (controlled) or the DOM (uncontrolled).

### Controlled — React owns the value

```jsx
function NameForm() {
  const [name, setName] = useState('');

  return (
    <input
      value={name}
      onChange={(e) => setName(e.target.value)}
    />
  );
}
```

Every keystroke fires `onChange`, updates state, re-renders with the new value. The input is always in sync with state — you can validate, transform, or disable submission based on it.

### Uncontrolled — DOM owns the value

```jsx
function NameForm() {
  const inputRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    alert(inputRef.current.value);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="" />
      <button>Submit</button>
    </form>
  );
}
```

You read the value only when needed (on submit). Less re-rendering, but harder to validate live.

### When to use which

- **Controlled** — almost always. You need live validation, conditional disabling, formatting (phone masks), or syncing with other state.
- **Uncontrolled** — simple forms, file inputs (`<input type="file">` is always uncontrolled), or when integrating with non-React code.

---

## 12. Hooks — The Rules & The Core Set

**Hooks** are functions that let you "hook into" React features (state, lifecycle, context) from a function component. They were added in React 16.8 (2019) and replaced class components for almost all use cases.

### The two Rules of Hooks

1. **Only call hooks at the top level.** Never inside loops, conditions, or nested functions.
2. **Only call hooks from React functions** — components or other hooks.

```jsx
// ❌ Bad
function Bad({ flag }) {
  if (flag) {
    const [x, setX] = useState(0);   // conditional — breaks hook ordering
  }
}

// ✅ Good
function Good({ flag }) {
  const [x, setX] = useState(0);
  if (flag) { /* use x */ }
}
```

React relies on hook **call order** to associate state with each call. Conditional hooks break this.

### The core hooks

| Hook | What it does |
|---|---|
| `useState` | Local component state |
| `useEffect` | Run side effects after render |
| `useRef` | Hold a mutable value or DOM reference |
| `useContext` | Read context value |
| `useReducer` | State with reducer pattern (alt to useState) |
| `useMemo` | Memoize an expensive computation |
| `useCallback` | Memoize a function reference |

---

## 13. Purity & Pure Components

A **pure function** returns the same output for the same input and has no side effects. React components should behave like pure functions during render.

```jsx
// ✅ Pure
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}

// ❌ Impure — modifies external state during render
let counter = 0;
function Counter() {
  counter++;                        // side effect during render!
  return <p>{counter}</p>;
}
```

### Why this matters

React may call your component:
- Multiple times during a render.
- In a different order than expected.
- And throw the result away.

If render has side effects, you get inconsistent UI, lost updates, or duplicate API calls. Side effects belong in `useEffect` or event handlers — never in the render body.

### Pure component rules

- Don't mutate props.
- Don't mutate state directly.
- Don't make network calls, set timers, or touch the DOM during render.
- Don't read or write to variables defined outside the component.

---

## 14. Strict Mode

**Strict Mode** is a development-only tool that helps catch bugs. Wrap your app:

```jsx
import { StrictMode } from 'react';

<StrictMode>
  <App />
</StrictMode>
```

### What it does

- **Double-invokes** components (and some hooks) during development to surface impure render logic.
- Warns about deprecated APIs.
- Helps spot side effects in places they shouldn't be.

If your code breaks under Strict Mode, it likely has a hidden impurity. Strict Mode does **nothing in production builds** — it's purely a dev safety net.

> The "double render" trips beginners. If you `console.log` in render and see it twice, that's Strict Mode helping, not a bug.

---

## 15. Side Effects & `useEffect`

A **side effect** is anything that touches the world outside your component during render — API calls, subscriptions, timers, DOM manipulation, logging.

`useEffect` runs side effects **after** the component renders.

```jsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let cancelled = false;

    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((data) => {
        if (!cancelled) setUser(data);
      });

    return () => {
      cancelled = true;             // cleanup on unmount or re-run
    };
  }, [userId]);                     // re-run when userId changes

  if (!user) return <p>Loading...</p>;
  return <h1>{user.name}</h1>;
}
```

### The three parts of `useEffect`

1. **Effect function** — runs after render.
2. **Cleanup function** (returned) — runs before next effect or on unmount.
3. **Dependency array** — controls when the effect re-runs.

### Dependency array behaviors

| Dependency arg | Behavior |
|---|---|
| Omitted | Runs after every render |
| `[]` (empty) | Runs once after mount, cleans up on unmount |
| `[a, b]` | Runs when `a` or `b` changes |

### Common use cases

- Fetching data on mount.
- Subscribing to an event source (WebSocket, browser event) — cleanup unsubscribes.
- Setting up timers — cleanup clears them.
- Syncing with `localStorage`.
- Updating `document.title`.

### Gotcha: stale closures

The function "captures" variables from when it was defined. If you reference a state value inside, list it in deps or use the functional setter:

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((c) => c + 1);          // functional form — no stale closure
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### Avoid useEffect when you don't need it

- **Don't** use it for things derivable from props/state — compute them in render.
- **Don't** use it for event handlers — put the logic in the handler.
- **Do** use it for syncing with external systems.

---

## 16. Refs & `useRef`

A **ref** is a mutable container whose value persists across renders **without** triggering re-renders when changed.

```jsx
const inputRef = useRef(null);
```

`inputRef` is an object `{ current: null }`. You can mutate `current` freely.

### Use case 1 — accessing DOM elements

```jsx
function FocusInput() {
  const inputRef = useRef(null);

  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current.focus()}>
        Focus the input
      </button>
    </>
  );
}
```

### Use case 2 — holding mutable values that shouldn't cause re-renders

```jsx
function Timer() {
  const startTime = useRef(Date.now());

  useEffect(() => {
    const id = setInterval(() => {
      console.log(`Elapsed: ${Date.now() - startTime.current}ms`);
    }, 1000);
    return () => clearInterval(id);
  }, []);
  return <p>Timer running</p>;
}
```

`useRef` is the right tool when you want to **remember something** but don't want a re-render when it changes (think instance variables in classes).

### Ref vs state — quick rule

| Use state when | Use ref when |
|---|---|
| Value affects what's rendered | Value is purely internal |
| You want a re-render on change | You don't want a re-render |
| Value goes through React | You're talking to the DOM or external code |

---

## 17. Context — Avoiding Prop Drilling

When data needs to reach deep into the tree, passing it through every intermediate component is painful — that's **prop drilling**.

```
App ─ Layout ─ Sidebar ─ UserCard
       (theme)  (theme)   (uses theme)
```

`Context` lets any descendant read the value without intermediaries.

### Using Context

```jsx
import { createContext, useContext, useState } from 'react';

const ThemeContext = createContext('light');         // default

function App() {
  const [theme, setTheme] = useState('dark');

  return (
    <ThemeContext.Provider value={theme}>
      <Layout />
    </ThemeContext.Provider>
  );
}

function UserCard() {
  const theme = useContext(ThemeContext);            // read it directly
  return <div className={`card card-${theme}`}>Hi</div>;
}
```

### When to use Context

- **Theme** (light/dark mode).
- **Current user** / auth state.
- **Locale** / i18n.
- **Feature flags**.

### When NOT to use Context

- For **everything** — it can become a performance footgun (any consumer re-renders when value changes).
- As a substitute for proper state management for complex/frequently-changing data — use **Zustand**, **Redux Toolkit**, or **Jotai** for that.

---

## 18. Portals — Rendering Outside the Tree

A **portal** renders a component into a DOM node that lives **outside** its parent's DOM hierarchy, while keeping it in the React component tree (events bubble normally, context still works).

Perfect for modals, tooltips, dropdowns — things that visually need to escape `overflow: hidden` or stacking contexts of their parent.

```jsx
import { createPortal } from 'react-dom';

function Modal({ children, onClose }) {
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')   // any DOM element outside your app
  );
}
```

Add `<div id="modal-root"></div>` next to your app's mount point in `index.html`.

### Why it works

- The modal lives at the top of the DOM (no clipping issues).
- But it's still rendered from inside your component tree — props, context, and event bubbling all work as if it were inline.

---

## 19. Suspense — Showing Fallbacks While Loading

**Suspense** lets you declaratively show a fallback UI (like a spinner) while something is loading — lazy-loaded components or data.

```jsx
import { Suspense, lazy } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

function Dashboard() {
  return (
    <Suspense fallback={<p>Loading chart...</p>}>
      <HeavyChart />
    </Suspense>
  );
}
```

`React.lazy` defers loading the `HeavyChart` bundle until it's actually rendered. Suspense shows the fallback during the load.

### Suspense for data (modern frameworks)

In Next.js App Router, Remix, and modern React Server Components, Suspense also works for data fetching:

```jsx
<Suspense fallback={<Skeleton />}>
  <Posts />          {/* fetches data internally; Suspense waits */}
</Suspense>
```

### Why this is nice

You declare the loading UI **once** at a boundary instead of writing `if (loading) return <Spinner />` in every component.

---

## 20. Error Boundaries — Catching Render Errors

A normal `try/catch` doesn't catch errors thrown during React's render. An **Error Boundary** does — it's a class component (yes, still class) that catches errors in its child tree and shows a fallback UI.

```jsx
import { Component } from 'react';

class ErrorBoundary extends Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    console.error('Caught error:', error, info);
    // also send to Sentry / Datadog here
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <Dashboard />
</ErrorBoundary>
```

### What boundaries catch

- Errors during render.
- Errors in lifecycle methods.
- Errors in child component constructors.

### What they DON'T catch

- Event handlers (use try/catch inside).
- Async code (promises, `setTimeout`).
- Errors in the boundary itself.
- Server-side rendering errors.

### Tip

Wrap small sections, not just the whole app. That way one widget crashing doesn't blank the entire page — only that widget shows the fallback.

---

## 21. Other Hooks You'll Meet

### `useReducer` — when state logic gets complex

Like `useState`, but you describe transitions in a reducer function (similar to Redux).

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    case 'reset':     return { count: 0 };
    default: throw new Error();
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </>
  );
}
```

Use when state has multiple sub-values that change together, or when next state depends on current state in non-trivial ways.

### `useMemo` — cache expensive computations

```jsx
const sortedItems = useMemo(() => {
  return items.slice().sort((a, b) => a.price - b.price);
}, [items]);
```

Only recomputes when `items` changes. **Don't** use `useMemo` everywhere — it has its own cost. Use it when the computation is genuinely expensive or you need referential stability.

### `useCallback` — cache function references

```jsx
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
```

Returns the same function reference between renders unless `id` changes. Useful when passing callbacks to memoized children (otherwise they re-render every time because the function is a new reference).

### `useContext` — already covered in §17

```jsx
const value = useContext(MyContext);
```

### React 19's `use` hook

Modern React lets you `use(promise)` or `use(context)` inside components and conditional blocks — a more flexible alternative for reading async resources.

---

## 22. Custom Hooks

You can extract reusable stateful logic into your own hook. By convention, names start with `use`.

```jsx
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored !== null ? JSON.parse(stored) : initial;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// Use it just like useState
function Settings() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  return <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
    Toggle ({theme})
  </button>;
}
```

Custom hooks let you share **logic**, not UI. Two components using `useLocalStorage` each get their own independent state — hooks aren't a way to share state.

### Common custom hooks you'll see

- `useDebounce(value, delay)` — debounced version of a value.
- `useOnClickOutside(ref, handler)` — close dropdowns when clicking outside.
- `useFetch(url)` — wrap fetch + loading + error.
- `useMediaQuery(query)` — respond to viewport changes.
- `usePrevious(value)` — remember the previous render's value.

---

## 23. Performance — Memoization & Re-render Avoidance

React's renders are usually cheap — premature optimization is the bigger danger. But once a tree gets large, three tools help:

### `React.memo` — skip re-render if props are unchanged

```jsx
const Row = React.memo(function Row({ item }) {
  return <li>{item.name}</li>;
});
```

`Row` only re-renders when `item` reference changes (shallow comparison). Pair with `useCallback`/`useMemo` on parent so prop references stay stable.

### `useMemo` and `useCallback`

(See §21.) Use to keep referential equality of expensive values / callbacks passed to memoized children.

### Splitting components

Move state down. If only `<Counter />` cares about `count`, don't lift it to the parent — keep it local so parent doesn't re-render on every click.

### Virtualization

For lists with thousands of items, render only what's visible. Libraries: **TanStack Virtual**, **react-window**.

### Profiler

React DevTools' **Profiler** tab shows which components re-rendered and why. Always measure before optimizing.

---

## 24. Putting It All Together — A Tiny App

A todo app that uses most of the concepts:

```jsx
import { useState, useEffect, useRef, useContext, createContext, Suspense, lazy } from 'react';
import { createPortal } from 'react-dom';

// 1) Context for theme
const ThemeContext = createContext('light');

// 2) Lazy-loaded "About" page (Suspense)
const About = lazy(() => import('./About'));

// 3) Custom hook
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initial;
  });
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);
  return [value, setValue];
}

// 4) Portal-based modal
function Modal({ open, onClose, children }) {
  if (!open) return null;
  return createPortal(
    <div className="overlay" onClick={onClose}>
      <div className="dialog" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')
  );
}

// 5) Controlled input
function TodoInput({ onAdd }) {
  const [text, setText] = useState('');
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current?.focus();   // autofocus on mount
  }, []);

  function handleSubmit(e) {
    e.preventDefault();
    if (!text.trim()) return;
    onAdd(text);
    setText('');
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        ref={inputRef}
        value={text}
        onChange={(e) => setText(e.target.value)}
        placeholder="What to do?"
      />
      <button>Add</button>
    </form>
  );
}

// 6) List using key prop
function TodoList({ items, onToggle, onDelete }) {
  const theme = useContext(ThemeContext);
  return (
    <ul className={`list theme-${theme}`}>
      {items.map((todo) => (
        <li key={todo.id}>
          <label>
            <input
              type="checkbox"
              checked={todo.done}
              onChange={() => onToggle(todo.id)}
            />
            <span style={{ textDecoration: todo.done ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
          </label>
          <button onClick={() => onDelete(todo.id)}>×</button>
        </li>
      ))}
    </ul>
  );
}

// 7) Root component composing it all
export default function App() {
  const [todos, setTodos] = useLocalStorage('todos', []);
  const [theme, setTheme] = useState('light');
  const [showAbout, setShowAbout] = useState(false);

  function addTodo(text) {
    setTodos((t) => [...t, { id: crypto.randomUUID(), text, done: false }]);
  }

  function toggleTodo(id) {
    setTodos((t) => t.map((x) => x.id === id ? { ...x, done: !x.done } : x));
  }

  function deleteTodo(id) {
    setTodos((t) => t.filter((x) => x.id !== id));
  }

  return (
    <ThemeContext.Provider value={theme}>
      <header>
        <h1>Todos</h1>
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
          Toggle theme
        </button>
        <button onClick={() => setShowAbout(true)}>About</button>
      </header>

      <TodoInput onAdd={addTodo} />
      <TodoList items={todos} onToggle={toggleTodo} onDelete={deleteTodo} />

      <Modal open={showAbout} onClose={() => setShowAbout(false)}>
        <Suspense fallback={<p>Loading...</p>}>
          <About />
        </Suspense>
      </Modal>
    </ThemeContext.Provider>
  );
}
```

This 100-line app uses: components, JSX, props, state, controlled inputs, list keys, refs, effects, context, portals, suspense, lazy loading, and a custom hook.

---

## 25. Common Pitfalls & Interview Q&A

### Common pitfalls

| Pitfall | Why it bites | Fix |
|---|---|---|
| Mutating state directly | React compares by reference; same ref → no re-render | Use spread, map, filter |
| Using array index as key | Causes wrong items to re-render on reorder | Use stable IDs |
| Forgetting dependency in `useEffect` | Stale values, missed updates | Add them, or use functional setters |
| Side effects in render | Double renders in Strict Mode, weird bugs | Move to `useEffect` or handler |
| Conditional hooks | Breaks call-order invariant | Lift conditions outside the hook |
| Heavy work on every render | Slow UI | `useMemo`, split components |
| Over-using context | Performance issues, hard to debug | Local state, dedicated state libs |
| Calling setState during render | Infinite loop | Move into effect or handler |
| Forgetting cleanup in effects | Memory leaks, multiple subscriptions | Return cleanup function |

### Interview Q&A

**Q: What is React?**
A library for building UIs with components. You describe what the UI should look like for the current state; React efficiently updates the DOM.

**Q: Why is React fast?**
It uses a virtual DOM and a diffing algorithm to minimize real DOM updates, which are the expensive part.

**Q: What's the difference between props and state?**
Props are inputs from parent (read-only). State is internal data that can change and triggers re-renders.

**Q: Why must keys be unique and stable?**
React uses them to match items across renders. Unstable or duplicate keys cause incorrect updates, lost state, and animation bugs.

**Q: What does `useEffect` do?**
Runs side effects after render — data fetching, subscriptions, manual DOM updates. The dependency array controls when it re-runs; the return value is a cleanup function.

**Q: What's a controlled component?**
A form input whose value is held in React state. Every change updates state via `onChange`, and the input displays the state value.

**Q: When would you use `useRef`?**
To access DOM nodes (focus, scroll) or hold values that persist across renders without triggering re-renders.

**Q: What's Context for?**
Sharing values (theme, user, locale) deep in the tree without prop drilling. Not a general state management solution.

**Q: What does Strict Mode do?**
Development-only: double-invokes components to flush out impure render logic and warns about deprecated APIs. No effect in production.

**Q: When do you use a portal?**
For modals, tooltips, popovers — anywhere visual content needs to escape its parent's DOM/CSS hierarchy but stay in the React tree.

**Q: How do error boundaries help?**
Catch render errors in their child tree and show a fallback instead of a blank screen. Doesn't catch event handler or async errors.

**Q: What problem does `React.memo` solve?**
Prevents a component from re-rendering when its props haven't changed (shallow comparison) — useful for pure leaf components in big lists.

**Q: Why must components be pure?**
React may re-invoke them multiple times. If they have side effects in render, you get inconsistent UI and duplicate work. Move effects to `useEffect` or event handlers.

**Q: useState vs useReducer — when to choose which?**
`useState` for simple, independent values. `useReducer` when state has multiple sub-values that change together, or transitions are complex enough that "what action happened" beats "what to set".

**Q: What's prop drilling?**
Passing props through many intermediate components that don't use them, just to reach a deep child. Context or composition usually fixes it.

**Q: Why is `key` not a prop you can read inside the component?**
React uses `key` to identify the instance — it's not part of your props. To pass an ID to the child, send it as a separate prop too.

---

> **Final advice:** Don't try to memorize APIs. Build small projects — a counter, a todo, a search box, a fetch+list, a modal. Each forces you to confront 4–5 concepts at once and the muscle memory sticks far better than reading docs.
