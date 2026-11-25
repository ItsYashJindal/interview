[Home](README.md) | [← 04 MongoDB & SQL](04-mongodb-sql.md) | Next: [06 TypeScript →](06-typescript.md)

# 05 – React.js (+ Next.js basics)

You are a backend engineer, so interviewers usually ask React at a "can you work full stack" level. Know the basics well and be honest that backend is your main strength.

## Contents
1. [Functional components](#1-functional-components)
2. [Props vs state](#2-props-vs-state)
3. [useState and useEffect](#3-usestate-and-useeffect)
4. [useMemo and useCallback](#4-usememo-and-usecallback)
5. [useRef](#5-useref)
6. [Context API](#6-context-api)
7. [Redux Toolkit](#7-redux-toolkit)
8. [React lifecycle](#8-react-lifecycle)
9. [Performance optimisation](#9-performance-optimisation)
10. [Virtual DOM](#10-virtual-dom)
- [Extra React questions (added)](#extra-react-questions)
- [Next.js basics (on your resume)](#nextjs-basics)

---

### 1. Functional components

> 🎤 **Interview answer:** "A functional component is just a JavaScript function that receives props and returns JSX describing the UI. Since hooks came in React 16.8, functional components can do everything class components could, with state, effects, context and refs, with less code and no `this` binding problems. I'd also be upfront: my main strength is backend, but I've built React and Next.js frontends, like the Air India GRN dashboard and the admin panels, so I'm comfortable working full stack."

A component is a function that takes **props** and returns **JSX** (UI).
```jsx
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}
```

**Q. Functional vs class components?**

> 🎤 **Interview answer:** "Class components extend `React.Component`, keep state in `this.state` and use lifecycle methods like componentDidMount. Functional components are plain functions that use hooks: `useState` for state, `useEffect` for side effects. Functional components are shorter, easier to test, avoid `this` binding issues, and let us reuse logic through custom hooks. They're the recommended approach today. The only thing that still needs a class is an error boundary."

| Functional | Class |
|---|---|
| Plain function | `class X extends React.Component` |
| Hooks for state/effects | `this.state`, lifecycle methods |
| Less code, no `this` | More boilerplate, `this` binding issues |
| Recommended today | Old code; still needed only for error boundaries |

**Q. What is JSX?**

> 🎤 **Interview answer:** "JSX is a syntax extension that lets us write HTML-like markup inside JavaScript. It's not valid JavaScript by itself: Babel or the TypeScript compiler transforms it into `React.createElement` calls, or the new JSX runtime functions, which create plain objects describing the UI. The differences from HTML are that we use `className` instead of `class`, camelCase event names like `onClick`, and curly braces for any JavaScript expression. JSX also escapes values by default, which helps prevent XSS."

HTML-like syntax inside JS. Babel converts `<h1>Hi</h1>` into `React.createElement("h1", null, "Hi")` (or the new JSX runtime). Use `className` instead of `class`, `{}` for JS expressions.

---

### 2. Props vs state

> 🎤 **Interview answer:** "Props are inputs passed from a parent to a child, like function arguments, and they're read-only for the child. State is data owned and managed inside the component, like local variables that survive re-renders, and it's changed with the setter function. When either props or state change, the component re-renders. A simple rule: if the data comes from outside, it's a prop. If the component itself changes it over time, like a form input or a toggle, it's state."

| Props | State |
|---|---|
| Passed from parent to child | Owned and managed inside the component |
| Read-only for the child | Can be changed with `setState` |
| Like function arguments | Like local variables that survive re-renders |
| Change in parent → child re-renders | Change → component re-renders |

```jsx
function Counter({ step }) {          // step = prop
  const [count, setCount] = useState(0); // count = state
  return <button onClick={() => setCount(c => c + step)}>{count}</button>;
}
```

**Q. How does a child send data to the parent?**

> 🎤 **Interview answer:** "Data in React flows down, so a child can't directly change the parent's state. Instead, the parent passes a callback function as a prop, like `onSearch`, and the child calls it with the data, like `onSearch(query)`. The parent then updates its own state. For data shared across many levels, I'd use Context or a store like Redux instead."

Parent passes a function as a prop; child calls it.
```jsx
<SearchBox onSearch={(q) => setQuery(q)} />
```

**Q. What is prop drilling?** Passing props through many layers that don't use them. Fix: Context API, Redux, or better component structure.

> 🎤 **Interview answer:** "Prop drilling is when we pass props through several intermediate components that don't use them, just to reach a deeply nested child. It makes the code noisy and fragile, because every layer has to know about that prop. The fixes are the Context API for things like the logged-in user or theme, a global store like Redux for frequently changing shared state, or restructuring with component composition, passing components as children."

**Q. What is "lifting state up"?** When two siblings need the same data, move the state to their common parent and pass it down.

> 🎤 **Interview answer:** "When two sibling components need the same data, or one needs to change data the other displays, we move that state up to their closest common parent. The parent holds the state and passes the value and the setter down as props. That gives a single source of truth, so the siblings never get out of sync."

---

### 3. useState and useEffect

**useState**

> 🎤 **Interview answer:** "`useState` adds local state to a function component. It returns the current value and a setter. A few things I keep in mind. State updates are asynchronous and batched, so the new value isn't available on the next line. When the new value depends on the old one, I use the functional form, `setCount(prev => prev + 1)`, to avoid stale values. I never mutate state directly, because React compares references, so I always create a new object or array with spread. And for an expensive initial value, I pass a function so it runs only once."

```jsx
const [user, setUser] = useState(null);
setUser({ name: "Yash" });

// Use functional update when new value depends on old value
setCount(prev => prev + 1);

// Lazy initial value (runs only once)
const [data] = useState(() => JSON.parse(localStorage.getItem("data")));
```
- State updates are **asynchronous and batched** – you don't see the new value on the next line.
- Never mutate state: `user.name = "x"; setUser(user)` will NOT re-render (same reference). Use `setUser({ ...user, name: "x" })`.

**useEffect** – run side effects (API calls, subscriptions, timers) after render.

> 🎤 **Interview answer:** "`useEffect` runs side effects after React renders: API calls, subscriptions, timers, or manually touching the DOM. The dependency array controls when it runs. With no array it runs after every render, with an empty array only once after mount, and with dependencies whenever one of them changes. The function I return is the cleanup, which runs before the next effect and on unmount, where I clear timers, remove listeners and close sockets. For data fetching I use an AbortController in the cleanup, so if the id changes quickly, an old response can't overwrite the new one."

```jsx
useEffect(() => {
  // runs after render
  return () => { /* cleanup */ };
}, [deps]);
```
| Dependency array | Runs |
|---|---|
| none | After every render |
| `[]` | Once after first render (mount); cleanup on unmount |
| `[id]` | On mount and whenever `id` changes |

**Fetching data correctly (with cleanup to avoid race conditions):**
```jsx
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then(r => r.json())
    .then(setUser)
    .catch(err => { if (err.name !== "AbortError") setError(err); });
  return () => controller.abort(); // cancels old request if id changes or component unmounts
}, [id]);
```

**Q. Why does useEffect run twice in development?**

> 🎤 **Interview answer:** "In React 18 with StrictMode, React deliberately mounts the component, unmounts it and mounts it again in development only. The goal is to expose effects that don't clean up properly, like subscriptions or duplicate API calls. It doesn't happen in production. The right response isn't to remove StrictMode but to write proper cleanup functions."

React 18 **StrictMode** mounts, unmounts and mounts again in development to help find missing cleanups. It does not happen in production.

**Q. Common useEffect mistakes**

> 🎤 **Interview answer:** "The common mistakes are missing dependencies, which causes stale values, and putting objects or functions created during render into the dependency array, which makes the effect run every render and can cause infinite loops. The fix is useMemo, useCallback, or moving them inside the effect. Another one is setting state that's in its own dependencies without a condition, which is also an infinite loop. Others are not cleaning up intervals, listeners or sockets, and race conditions in fetches without abort. I also try not to use effects for things that can be calculated during render."

- Missing dependencies → stale values.
- Object/function in deps created every render → infinite loop.
- Setting state that is in the deps without a condition → infinite loop.
- Not cleaning up intervals/listeners/sockets.

---

### 4. useMemo and useCallback

> 🎤 **Interview answer:** "Both are memoization hooks. `useMemo` caches a calculated value between renders and recalculates only when its dependencies change, like filtering a large product list by search text. `useCallback` caches a function, so it keeps the same reference between renders. That matters when passing callbacks to a child wrapped in `React.memo`, or using a function in a useEffect dependency array. `useCallback(fn, deps)` is basically `useMemo(() => fn, deps)`. I don't wrap everything in them, because they have their own cost. I use them when profiling shows unnecessary re-renders or expensive calculations."

- **useMemo** – remembers a **calculated value** between renders.
- **useCallback** – remembers a **function** (same reference) between renders.

```jsx
const filtered = useMemo(
  () => products.filter(p => p.name.includes(search)),
  [products, search]
);

const handleDelete = useCallback((id) => {
  setItems(items => items.filter(i => i.id !== id));
}, []);

<ItemList items={filtered} onDelete={handleDelete} />  // ItemList wrapped in React.memo
```

**Q. When to use them?**

> 🎤 **Interview answer:** "I use `useMemo` for expensive calculations, or to keep an object or array reference stable when it's passed to a memoized child or a context value. I use `useCallback` for functions passed to `React.memo` children or used as effect dependencies. I avoid using them by default on cheap values, because the comparison and memory cost can be more than the benefit. I measure with the React DevTools Profiler first."

- `useMemo` for expensive calculations, or to keep an object/array reference stable.
- `useCallback` when passing functions to a child wrapped in `React.memo`, or used in a `useEffect` dependency array.
- **Don't** wrap everything – they have their own cost. Use when you see a real problem.

`useCallback(fn, deps)` is the same as `useMemo(() => fn, deps)`.

---

### 5. useRef

> 🎤 **Interview answer:** "`useRef` returns a mutable object with a `.current` property that persists across renders, and changing it does not trigger a re-render. It has two main uses. First, accessing DOM elements directly, like focusing an input or measuring size. Second, storing mutable values that don't affect the UI, like a timer id, a socket instance, or the previous value of a prop. The key difference from state: changing state re-renders and updates the UI, and changing a ref doesn't."

Gives a box `{ current: value }` that **survives re-renders** and changing it does **not** cause a re-render.

**Uses:**
```jsx
// 1. Access a DOM element
const inputRef = useRef(null);
<input ref={inputRef} />
inputRef.current.focus();

// 2. Keep a mutable value without re-render (timer id, previous value, socket)
const timerRef = useRef();
timerRef.current = setInterval(tick, 1000);
clearInterval(timerRef.current);

// 3. Store previous value
const prev = useRef();
useEffect(() => { prev.current = value; }, [value]);
```

**useRef vs useState:** state change → re-render and UI updates. Ref change → no re-render.

---

### 6. Context API

> 🎤 **Interview answer:** "The Context API lets us share data with many components in a tree without passing props through every level. I create a context, wrap the part of the app with a Provider holding the value, and any component inside can read it with `useContext`, usually through a custom hook like `useAuth`. It's ideal for data that changes rarely, like the logged-in user, theme or language. One performance point: every consumer re-renders when the value changes, so I memoize the provider value and split large contexts."

Share data with many components without passing props at every level. Good for theme, logged-in user, language.

```jsx
const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const value = useMemo(() => ({ user, setUser }), [user]);
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

const useAuth = () => useContext(AuthContext);

// anywhere inside the provider
const { user } = useAuth();
```

**Q. Context vs Redux?**

> 🎤 **Interview answer:** "Context is built in and simple, and great for low-frequency global data like auth or theme. But every consumer re-renders when the value changes, and there's no middleware or devtools. Redux Toolkit is better for large, frequently changing shared state, like a cart, filters or complex dashboards. Components subscribe to just the slice they select, so re-renders are targeted. It also gives devtools, middleware and RTK Query for API caching. For server data specifically, I'd often use RTK Query or React Query instead of putting it in global state manually."

| Context | Redux Toolkit |
|---|---|
| Built-in, simple | Extra library |
| Good for data that changes rarely (theme, auth) | Good for large, frequently changing global state |
| Every consumer re-renders when value changes | Components re-render only for the slice they select |
| No devtools/middleware | Devtools, middleware, RTK Query for API caching |

---

### 7. Redux Toolkit

**Q. What is Redux?**

> 🎤 **Interview answer:** "Redux is a predictable state management library with a single global store. Components dispatch actions, which are plain objects describing what happened. Reducers, which are pure functions, take the current state and the action and return the new state. Components read data with selectors and re-render when their selected data changes. The one-way flow makes state changes traceable and easy to debug with Redux DevTools, including time-travel debugging."

One global **store** for app state. Components **dispatch actions**; **reducers** (pure functions) create the new state; components **select** what they need.

**Flow:** UI → `dispatch(action)` → reducer → new state in store → UI re-renders.

**Redux Toolkit (RTK)** is the official, shorter way to write Redux.

> 🎤 **Interview answer:** "Redux Toolkit is the official, recommended way to write Redux, and it removes most of the boilerplate. `configureStore` sets up the store with good defaults and devtools. `createSlice` generates the reducer and the action creators together. Thanks to Immer, I can write 'mutating' code like `state.items.push()` and it produces a new immutable state behind the scenes. `createAsyncThunk` handles async API calls with pending, fulfilled and rejected states. And RTK Query adds data fetching and caching, similar to React Query."

```js
// cartSlice.js
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

export const fetchCart = createAsyncThunk("cart/fetch", async () => {
  const res = await fetch("/api/cart");
  return res.json();
});

const cartSlice = createSlice({
  name: "cart",
  initialState: { items: [], status: "idle" },
  reducers: {
    addItem(state, action) { state.items.push(action.payload); }, // Immer allows "mutating" code
    removeItem(state, action) { state.items = state.items.filter(i => i.id !== action.payload); },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchCart.pending, (s) => { s.status = "loading"; })
      .addCase(fetchCart.fulfilled, (s, a) => { s.status = "done"; s.items = a.payload; });
  },
});
export const { addItem, removeItem } = cartSlice.actions;
export default cartSlice.reducer;

// store.js
export const store = configureStore({ reducer: { cart: cartReducer } });

// component
const items = useSelector((s) => s.cart.items);
const dispatch = useDispatch();
dispatch(addItem(product));
```

**Key points:**
- `createSlice` makes actions + reducer together.
- **Immer** lets you write "mutating" code in reducers; it creates a new immutable state for you.
- `createAsyncThunk` for API calls (pending/fulfilled/rejected).
- **RTK Query** – built-in data fetching + caching (like React Query).

---

### 8. React lifecycle

> 🎤 **Interview answer:** "A component goes through three phases: mounting, when it's added to the screen; updating, when it re-renders because props, state or context changed; and unmounting, when it's removed. In class components we had componentDidMount, componentDidUpdate and componentWillUnmount. With hooks, `useEffect` with an empty dependency array covers mount, `useEffect` with dependencies covers updates, and the cleanup function covers unmount. `React.memo` replaces shouldComponentUpdate. Error boundaries still need a class with componentDidCatch."

Three stages: **Mounting** (added to screen) → **Updating** (re-render on props/state change) → **Unmounting** (removed).

| Class method | Hook equivalent |
|---|---|
| `constructor` | `useState` initial value |
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `componentDidUpdate` | `useEffect(() => {...}, [dep])` |
| `componentWillUnmount` | cleanup function returned from `useEffect` |
| `shouldComponentUpdate` | `React.memo` |
| `getDerivedStateFromError` / `componentDidCatch` | No hook – use a class Error Boundary |

**Q. What causes a re-render?**

> 🎤 **Interview answer:** "A component re-renders when its own state changes, when its parent re-renders, even if the props are the same, unless it's wrapped in `React.memo` and the props are shallowly equal, or when a context it consumes changes value. A re-render doesn't always mean the DOM changes, because React only commits the differences. But unnecessary re-renders of big trees cost performance, and that's where memo, useMemo and useCallback help."

1. Its state changes.
2. Its parent re-renders (even if props are the same – unless wrapped in `React.memo`).
3. A context it uses changes.

---

### 9. Performance optimisation

> 🎤 **Interview answer:** "I start by measuring with the React DevTools Profiler and Lighthouse, instead of guessing. Then the usual techniques: `React.memo` with stable props from useMemo and useCallback to avoid unnecessary re-renders; stable unique keys in lists, not array indexes; code splitting with `React.lazy` and Suspense so pages load only when needed; virtualising long lists with react-window; debouncing search inputs; keeping state close to where it's used and splitting big contexts; and optimising images with lazy loading and Next.js Image. Since I'm a backend engineer, I also make sure the API side is efficient, with server-side pagination, small payloads and caching, because a slow API makes any frontend feel slow."

- **React.memo** – skip re-render if props didn't change (shallow compare).
- **useMemo / useCallback** – keep values/functions stable for memoized children.
- **Proper keys in lists** – stable unique ids, not array index (index breaks when items are added/removed/reordered).
- **Code splitting** – `React.lazy` + `Suspense` to load pages only when needed.
  ```jsx
  const Dashboard = lazy(() => import("./Dashboard"));
  <Suspense fallback={<Spinner />}><Dashboard /></Suspense>
  ```
- **Virtualise long lists** – render only visible rows (`react-window`).
- **Debounce** search inputs ([01 → debounce](01-javascript.md#7-debouncing-and-throttling)).
- **Keep state close** to where it is used; split big contexts.
- **Avoid creating objects/functions inline** in props for memoized children.
- **Images** – lazy load, right size, modern formats (Next.js `<Image>`).
- **Server-side pagination** and caching API data (RTK Query / React Query).
- Measure first with **React DevTools Profiler** and Lighthouse.

---

### 10. Virtual DOM

**Q. What is the Virtual DOM?**

> 🎤 **Interview answer:** "The Virtual DOM is a lightweight in-memory JavaScript representation of the real DOM. Updating the real DOM directly is expensive because of layout and repaint. So when state changes, React builds a new virtual tree, diffs it against the previous one, which is called reconciliation, and then applies only the minimal changes to the real DOM in a batch. The diffing uses a few rules: a different element type rebuilds that subtree, the same type updates only the changed attributes, and keys identify list items. That's why correct keys matter."

A lightweight JS copy of the real DOM. Updating the real DOM is slow, so React:
1. Creates a new Virtual DOM tree when state changes.
2. **Diffs** it with the previous tree (this is called **reconciliation**).
3. Updates **only the changed parts** in the real DOM (batched).

**Diffing rules (simple):**
- Different element type (`<div>` → `<span>`) → rebuild that subtree.
- Same type → update only changed attributes.
- Lists → use `key` to know which item is which.

**Q. What is React Fiber?**

> 🎤 **Interview answer:** "Fiber is React's reconciliation engine since React 16. Instead of rendering the whole tree in one uninterruptible pass, Fiber breaks the work into small units. React can then pause, prioritise urgent updates like typing over less urgent ones like rendering a big list, and resume later. That's the foundation for React 18's concurrent features like `useTransition` and `useDeferredValue`."

The newer reconciliation engine (React 16+). It splits rendering work into small units so React can pause, prioritise urgent updates (typing) and resume – this enables concurrent features like `useTransition`.

---

## Extra React questions

### Q. Controlled vs uncontrolled components

> 🎤 **Interview answer:** "In a controlled component, the input's value comes from React state through `value` and `onChange`, so React is the single source of truth. That makes validation, formatting and conditional logic easy. In an uncontrolled component, the DOM keeps the value and we read it with a ref when needed, which is less code and fewer re-renders. Libraries like React Hook Form use uncontrolled inputs for performance in large forms. I use controlled inputs for small forms with live validation, and React Hook Form for large ones."

- **Controlled** – input value comes from React state (`value` + `onChange`). Easy validation.
- **Uncontrolled** – DOM keeps the value; read with `ref`. Less code, used in libraries like React Hook Form for performance.

### Q. Why are keys needed in lists?

> 🎤 **Interview answer:** "Keys help React identify which list item is which between renders, so it can correctly reuse, move, add or remove elements. They must be stable and unique, ideally a database id. Using the array index is risky when items can be added, removed or reordered: React may attach the wrong state to the wrong item, so input values jump around or the wrong row gets updated."

So React can match old and new items correctly. Wrong keys (index) → wrong item updated, input values jump around.

### Q. Custom hooks

> 🎤 **Interview answer:** "A custom hook is a function whose name starts with `use` and that calls other hooks, to reuse stateful logic across components. For example, `useFetch` handles loading, data and error state, `useAuth` reads the auth context, and `useDebounce` delays a value. Each component that uses the hook gets its own independent state. It's the modern replacement for HOCs and render props, and it keeps components clean and focused on UI."

Functions starting with `use` that reuse stateful logic.
```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  useEffect(() => {
    let active = true;
    fetch(url).then(r => r.json()).then(d => { if (active) { setData(d); setLoading(false); } });
    return () => { active = false; };
  }, [url]);
  return { data, loading };
}
```

### Q. Rules of hooks

> 🎤 **Interview answer:** "There are two rules. First, only call hooks at the top level of the component, never inside conditions, loops or nested functions. Second, only call them from React function components or custom hooks. The reason is that React identifies each hook by its call order on every render. If a condition skips a hook, the order changes and the state gets mixed up. The ESLint plugin for hooks enforces this."

1. Call hooks only at the top level (not inside if/loops).
2. Call hooks only in React function components or custom hooks.
Reason: React tracks hooks by their call order.

### Q. useReducer

> 🎤 **Interview answer:** "`useReducer` is an alternative to useState for complex state logic: when the next state depends on the previous one, or there are many related values and actions. It takes a reducer function and an initial state, and returns the state and a `dispatch`. It's like a small Redux inside a component. I'd use it for multi-step forms or complex filters, where many fields update together."

Like `useState` but for complex state with many actions (`state, dispatch`). Good for forms and multi-step flows.

### Q. useEffect vs useLayoutEffect

> 🎤 **Interview answer:** "`useEffect` runs asynchronously after the browser has painted, so it doesn't block the UI, and it's right for almost everything, like data fetching and subscriptions. `useLayoutEffect` runs synchronously after DOM changes but before the browser paints. I use it only when I need to measure the DOM and adjust layout before the user sees it, like tooltip positioning, to avoid a visible flicker. Overusing it hurts performance."

`useEffect` runs after the browser paints. `useLayoutEffect` runs before paint – use only to measure DOM and avoid flicker.

### Q. Error boundaries

> 🎤 **Interview answer:** "An error boundary is a class component that implements `getDerivedStateFromError` or `componentDidCatch`. It catches errors thrown while rendering its child components and shows a fallback UI instead of crashing the whole app with a white screen. It can also log the error to a monitoring service. It doesn't catch errors in event handlers, async code or server-side rendering, so those need normal try/catch. Libraries like `react-error-boundary` make it easier to use."

Class components that catch errors in children during rendering and show a fallback UI. They don't catch errors in event handlers or async code.

### Q. Fragments

> 🎤 **Interview answer:** "Fragments, written as `<>...</>` or `React.Fragment`, let a component return multiple elements without adding an extra wrapper div to the DOM. That keeps the DOM clean and avoids breaking layouts like tables or flex containers. When rendering a list of fragments, we use the long form so we can pass a `key`."

`<>...</>` – group elements without adding an extra DOM node.

### Q. Higher-Order Component (HOC)

> 🎤 **Interview answer:** "A higher-order component is a function that takes a component and returns a new, enhanced component, like `withAuth(Page)`, which checks the login before rendering the page. It's the same concept as higher-order functions. They were common before hooks, but they can cause wrapper hell and prop name collisions, so today custom hooks are usually the better way to share logic."

A function that takes a component and returns a new component (e.g. `withAuth(Page)`). Today custom hooks are usually used instead.

### Q. Protected routes (React Router)

> 🎤 **Interview answer:** "I create a wrapper component, like `PrivateRoute`, that reads the auth state from context. If the user is logged in, it renders the children. Otherwise it redirects to the login page with `<Navigate>`, usually remembering where they wanted to go. For role-based pages it also checks the role. The important point, as a backend engineer: frontend route protection is only for user experience. Real security is enforced by the backend checking the token and permissions on every API."

```jsx
function PrivateRoute({ children }) {
  const { user } = useAuth();
  return user ? children : <Navigate to="/login" replace />;
}
```
Remember: frontend checks are only for UX – **real security is on the backend**.

### Q. How do you handle auth tokens in React?

> 🎤 **Interview answer:** "The refresh token lives in an httpOnly secure cookie that JavaScript can't read, and the access token is kept in memory. I set up an Axios interceptor: when an API returns 401 because the access token expired, it calls `/auth/refresh` once, gets a new access token and retries the original request. Concurrent requests wait for the same refresh instead of each triggering one. If the refresh fails, the user is logged out. I avoid localStorage for tokens because of the XSS risk."

Refresh token in httpOnly cookie, access token in memory. An Axios interceptor catches 401, calls `/auth/refresh`, retries the original request. See [03 → JWT](03-nodejs-express.md#7-jwt-and-refresh-tokens).

### Q. What is reconciliation vs rendering?

> 🎤 **Interview answer:** "Rendering means React calls the component functions to produce the new tree of elements. Reconciliation is comparing that new tree with the previous one to work out what changed. Commit is the final phase, where React applies those changes to the real DOM and runs layout effects and then effects. So a render doesn't always mean a DOM update: if nothing changed, the commit does nothing."

Rendering = calling your component functions to get the new tree. Reconciliation = comparing new tree with old and deciding DOM changes. Commit = applying changes to the DOM.

### Q. React 18 features

> 🎤 **Interview answer:** "The main React 18 features are automatic batching, concurrent rendering and the new root API. With automatic batching, multiple state updates are combined into one re-render even inside promises, timeouts and native event handlers, not just React events. Concurrent rendering enables `useTransition`, which marks updates as non-urgent so typing stays responsive, and `useDeferredValue`. The new `createRoot` API is what enables these. There's also streaming SSR with Suspense, which Next.js uses heavily."

Automatic batching (even in promises/timeouts), `useTransition`, `useDeferredValue`, concurrent rendering, `createRoot`.

---

## Next.js basics

You built a **Next.js GRN dashboard** for Air India, so expect a few questions.

### Q. Why Next.js over plain React?

> 🎤 **Interview answer:** "Next.js is a React framework that adds what plain React doesn't have built in: file-based routing, server-side rendering and static generation for better SEO and faster first loads, API route handlers so we can have backend endpoints in the same project, image and font optimisation, and middleware. For the Air India GRN dashboard, I chose Next.js because it gave routing, fast server-rendered pages and API routes in one codebase, and it fit well with our TypeScript backend."

File-based routing, server-side rendering, API routes, image optimisation, better SEO and first-load speed.

### Q. CSR vs SSR vs SSG vs ISR

> 🎤 **Interview answer:** "With CSR, client-side rendering, the browser downloads JavaScript and builds the page itself. It's good for dashboards behind login, where SEO doesn't matter. With SSR, server-side rendering, HTML is generated on the server for every request, so it's always fresh and SEO-friendly, but it costs server time on each request. With SSG, static site generation, HTML is generated once at build time, which is the fastest and good for blogs and docs. ISR, incremental static regeneration, is static pages that rebuild in the background after a set time, so product pages stay fast and still update. In Next.js, you can choose per page."

| Type | HTML made | Use |
|---|---|---|
| CSR (Client-Side Rendering) | In browser after JS loads | Dashboards behind login |
| SSR (Server-Side Rendering) | On server for every request | Fresh, personalised, SEO pages |
| SSG (Static Site Generation) | At build time | Blogs, docs, marketing pages |
| ISR (Incremental Static Regeneration) | Static, rebuilt in background after N seconds | Product pages |

### Q. App Router: Server vs Client components

> 🎤 **Interview answer:** "In the Next.js App Router, components are Server Components by default. They run only on the server, can access the database or secrets directly, and send no JavaScript to the browser, so bundles are smaller. Client Components start with `'use client'` and are needed whenever we use state, effects, event handlers or browser APIs. The usual pattern is to fetch data in server components and pass it down to small client components for the interactive parts."

- **Server components** (default) – run on server, can read DB/secrets directly, send no JS to browser.
- **Client components** – `"use client"` at top; needed for state, effects, event handlers, browser APIs.

### Q. API routes / Route handlers

> 🎤 **Interview answer:** "Route handlers in the App Router are files like `app/api/grn/route.ts` that export functions named after HTTP methods, like GET and POST, and they act as a small backend inside the Next.js app. They're good for things like form submissions, a backend-for-frontend layer or webhooks. For heavy business logic, integrations or background jobs, like the SAP sync, I keep a separate Node service and let Next.js call it."

`app/api/grn/route.ts` exports `GET`, `POST` functions – small backend inside Next.js. For heavy logic keep a separate Node service.

### Q. Middleware in Next.js

> 🎤 **Interview answer:** "`middleware.ts` runs before a request is completed, at the edge, for matching routes. It's commonly used for auth checks and redirects, like sending unauthenticated users to login before the page even renders, plus locale detection, rewrites and setting headers. It should stay lightweight, because it runs on every matched request."

`middleware.ts` runs before a request is completed – used for auth redirects, i18n, headers.

---

[Home](README.md) | [← 04 MongoDB & SQL](04-mongodb-sql.md) | Next: [06 TypeScript →](06-typescript.md)
