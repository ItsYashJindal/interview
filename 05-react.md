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

> 🎤 **Interview answer:** "A functional component is just a JavaScript function. It takes props as input and returns JSX, which describes what to show on the screen. Since hooks came, functional components can do everything class components could, with less code. To be honest, my main strength is backend, but I've built React and Next.js frontends like the Air India GRN dashboard and admin panels, so I'm comfortable working full stack."

```jsx
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}
```

**Q. Functional vs class components?**

> 🎤 **Interview answer:** "Class components are the old way. They use `this.state` and methods like `componentDidMount`. Functional components are plain functions that use hooks: `useState` for data and `useEffect` for side work like API calls. Functional components are shorter, easier to test, and have no `this` problems. They are the recommended way today. The only thing that still needs a class is an error boundary."

| Functional | Class |
|---|---|
| Plain function | `class X extends React.Component` |
| Hooks for state/effects | `this.state`, lifecycle methods |
| Less code, no `this` | More code, `this` problems |
| Recommended today | Old code; only needed for error boundaries |

**Q. What is JSX?**

> 🎤 **Interview answer:** "JSX lets us write HTML-like code inside JavaScript. The browser doesn't understand it directly, so a tool like Babel converts it into normal JavaScript function calls. Small differences from HTML: we write `className` instead of `class`, `onClick` instead of `onclick`, and we put any JavaScript inside `{ }`."

---

### 2. Props vs state

> 🎤 **Interview answer:**
> - "**Props** are data **given by the parent** to the child, like function arguments. The child can only read them, not change them.
> - **State** is data **owned by the component itself**, which it can change with the setter function.
>
> When props or state change, the component re-renders (draws again). Simple rule: if the data comes from outside, it's a prop. If the component changes it itself, like a form input or a toggle, it's state."

| Props | State |
|---|---|
| Given by the parent | Owned by the component |
| Read-only for the child | Can be changed with the setter |
| Like function arguments | Like variables that are remembered between renders |

```jsx
function Counter({ step }) {             // step = prop (from parent)
  const [count, setCount] = useState(0); // count = state (own data)
  return <button onClick={() => setCount(c => c + step)}>{count}</button>;
}
```

**Q. How does a child send data to the parent?**

> 🎤 **Interview answer:** "In React, data flows **down** from parent to child. A child can't change the parent's data directly. So the parent passes a **function** as a prop, like `onSearch`, and the child calls it with the data: `onSearch(query)`. The parent then updates its own state."

```jsx
<SearchBox onSearch={(q) => setQuery(q)} />
```

**Q. What is prop drilling?**

> 🎤 **Interview answer:** "Prop drilling is when we pass a prop through many middle components that don't need it, just to reach a deep child. It makes the code messy. Fixes: the Context API for things like the logged-in user or theme, or Redux for big shared data."

**Q. What is "lifting state up"?**

> 🎤 **Interview answer:** "When two sibling components need the same data, I move that state up to their common parent. The parent keeps the data and passes it down to both as props. So both always show the same data."

---

### 3. useState and useEffect

**useState**

> 🎤 **Interview answer:** "`useState` gives a component its own data that it remembers between renders. It returns the current value and a function to change it. A few things to remember:
> - The update is not instant. The new value is not available on the very next line.
> - If the new value depends on the old one, I use `setCount(prev => prev + 1)`.
> - I never change state directly. I always create a new object or array, because React only re-renders if it gets a **new** object."

```jsx
const [user, setUser] = useState(null);
setUser({ name: "Yash" });

// When the new value depends on the old value
setCount(prev => prev + 1);

// Wrong – same object, React won't re-render
user.name = "x";
setUser(user);

// Right – new object
setUser({ ...user, name: "x" });
```

**useEffect** – run side work (API calls, timers, subscriptions) after the component shows on screen.

> 🎤 **Interview answer:** "`useEffect` runs code **after** React shows the component, like API calls, timers or event listeners. The dependency array decides **when** it runs:
> - No array → after every render.
> - Empty array `[]` → only once, when the component first appears.
> - `[id]` → first time, and again whenever `id` changes.
>
> The function I return from it is the **cleanup**. It runs when the component is removed, or before the effect runs again. There I clear timers, remove listeners and cancel old API calls."

```jsx
useEffect(() => {
  // runs after render
  return () => { /* cleanup */ };
}, [deps]);
```

| Dependency array | Runs |
|---|---|
| none | After every render |
| `[]` | Once, when the component first appears; cleanup when it is removed |
| `[id]` | First time and whenever `id` changes |

**Fetching data correctly (cancel the old request if `id` changes):**
```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then(r => r.json())
    .then(setUser)
    .catch(err => {
      if (err.name !== "AbortError") setError(err);
    });

  return () => controller.abort(); // cancel the old request
}, [id]);
```

**Q. Why does useEffect run twice in development?**

> 🎤 **Interview answer:** "In React 18 StrictMode, React shows the component, removes it, and shows it again, **only in development**. It does this on purpose, to find effects that don't clean up properly. It does not happen in production. The right fix is to write proper cleanup, not to remove StrictMode."

**Q. Common useEffect mistakes**

> 🎤 **Interview answer:**
> - "Missing a dependency → the effect uses an old value.
> - Putting an object or function made during render in the dependency array → it's 'new' every render, so the effect runs every time, sometimes in an endless loop.
> - Setting a state that is also in the dependency array, without a condition → endless loop.
> - Not cleaning up timers, listeners or sockets → memory leaks."

---

### 4. useMemo and useCallback

> 🎤 **Interview answer:** "Both help avoid repeating work on every render.
> - **`useMemo`** remembers a **calculated value**. It recalculates only when its inputs change. Example: filtering a big product list by search text.
> - **`useCallback`** remembers a **function**, so the same function is reused between renders. This matters when passing it to a child wrapped in `React.memo`, because a 'new' function would make the child re-render.
>
> I don't wrap everything in them, because they also have a cost. I use them when I see a real slowness."

```jsx
// Recalculate only when products or search change
const filtered = useMemo(
  () => products.filter(p => p.name.includes(search)),
  [products, search]
);

// Same function every render
const handleDelete = useCallback((id) => {
  setItems(items => items.filter(i => i.id !== id));
}, []);

<ItemList items={filtered} onDelete={handleDelete} />  // ItemList is wrapped in React.memo
```

**Q. When to use them?**

> 🎤 **Interview answer:** "`useMemo` for heavy calculations. `useCallback` for functions passed to a `React.memo` child, or used inside a `useEffect` dependency array. Not for simple, cheap things. I check with the React DevTools Profiler first."

---

### 5. useRef

> 🎤 **Interview answer:** "`useRef` gives a box `{ current: value }` that **is remembered between renders**, but changing it does **not** re-render the component. Two main uses:
> 1. **Access a DOM element** directly, like focusing an input.
> 2. **Keep a value that doesn't affect the UI**, like a timer id or a socket.
>
> Difference from state: changing state updates the screen. Changing a ref doesn't."

```jsx
// 1. Access a DOM element
const inputRef = useRef(null);
<input ref={inputRef} />
inputRef.current.focus();

// 2. Keep a value without re-render (timer id)
const timerRef = useRef();
timerRef.current = setInterval(tick, 1000);
clearInterval(timerRef.current);

// 3. Remember the previous value
const prev = useRef();
useEffect(() => { prev.current = value; }, [value]);
```

---

### 6. Context API

> 🎤 **Interview answer:** "Context lets many components share data **without passing props through every level**. I create a context, wrap part of the app in a Provider with the value, and any component inside can read it with `useContext`. Usually I make a small hook like `useAuth`. It is best for data that changes rarely, like the logged-in user, theme or language. One thing to know: when the context value changes, **every** component using it re-renders."

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

> 🎤 **Interview answer:** "Context is built into React and simple. Good for data that changes rarely, like auth or theme. Redux Toolkit is better for big data that changes often and is shared by many components, like a cart or filters. With Redux, a component re-renders only when the **part** of the data it uses changes. Redux also gives DevTools to see every change."

| Context | Redux Toolkit |
|---|---|
| Built-in, simple | Extra library |
| Good for data that changes rarely (theme, auth) | Good for big, often-changing shared data |
| Every user of the context re-renders on change | Only components using the changed part re-render |
| No DevTools | DevTools, RTK Query for API caching |

---

### 7. Redux Toolkit

**Q. What is Redux?**

> 🎤 **Interview answer:** "Redux keeps the app's shared data in **one central store**. The flow is always one way:
> 1. A component **dispatches an action**, which is an object saying what happened, like `{ type: 'cart/addItem' }`.
> 2. A **reducer** (a pure function) takes the old state and the action, and returns the new state.
> 3. Components **read** the data they need with `useSelector`, and re-render when it changes.
>
> Because the flow is always the same, it's easy to debug."

**Flow:** UI → `dispatch(action)` → reducer → new state in store → UI updates.

**Redux Toolkit (RTK)** is the official, shorter way to write Redux.

> 🎤 **Interview answer:** "Redux Toolkit removes most of the extra code of old Redux.
> - `createSlice` creates the reducer and the actions together.
> - Inside reducers I can write code like `state.items.push(item)`, and RTK safely creates a new state for me behind the scenes (using a library called Immer).
> - `createAsyncThunk` is for API calls, with loading / success / error states.
> - RTK Query handles API fetching and caching."

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
    addItem(state, action) {
      state.items.push(action.payload); // looks like changing, but RTK makes a new state
    },
    removeItem(state, action) {
      state.items = state.items.filter(i => i.id !== action.payload);
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchCart.pending, (state) => { state.status = "loading"; })
      .addCase(fetchCart.fulfilled, (state, action) => {
        state.status = "done";
        state.items = action.payload;
      });
  },
});
export const { addItem, removeItem } = cartSlice.actions;
export default cartSlice.reducer;

// store.js
export const store = configureStore({ reducer: { cart: cartReducer } });

// in a component
const items = useSelector((state) => state.cart.items);
const dispatch = useDispatch();
dispatch(addItem(product));
```

---

### 8. React lifecycle

> 🎤 **Interview answer:** "A component goes through three stages:
> 1. **Mounting** – it appears on the screen.
> 2. **Updating** – it re-renders because props or state changed.
> 3. **Unmounting** – it is removed from the screen.
>
> With hooks: `useEffect` with `[]` runs on mount, `useEffect` with `[dep]` runs on update, and the cleanup function runs on unmount."

| Class method | Hook equivalent |
|---|---|
| `constructor` | `useState` starting value |
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `componentDidUpdate` | `useEffect(() => {...}, [dep])` |
| `componentWillUnmount` | cleanup function returned from `useEffect` |
| `shouldComponentUpdate` | `React.memo` |
| `componentDidCatch` | No hook – use a class Error Boundary |

**Q. What causes a re-render?**

> 🎤 **Interview answer:** "A component re-renders when:
> 1. its own state changes,
> 2. its parent re-renders (even if props are the same, unless wrapped in `React.memo`),
> 3. a context it uses changes.
>
> A re-render doesn't always change the real page. React only updates what actually changed."

---

### 9. Performance optimisation

> 🎤 **Interview answer:** "First I **measure** with React DevTools Profiler and Lighthouse, instead of guessing. Then the usual fixes:
> - `React.memo` to skip re-rendering a child when its props didn't change.
> - Proper unique `key`s in lists (not the array index).
> - Load pages only when needed, with `React.lazy`.
> - For very long lists, render only the visible rows (`react-window`).
> - Debounce search inputs.
>
> As a backend engineer, I also make sure the API is fast: server-side pagination, small responses and caching. A slow API makes any frontend feel slow."

- **React.memo** – skip re-render if props didn't change.
- **useMemo / useCallback** – keep values and functions the same between renders.
- **Proper keys in lists** – use unique ids, not the array index.
- **Code splitting** – load a page only when the user opens it.
  ```jsx
  const Dashboard = lazy(() => import("./Dashboard"));
  <Suspense fallback={<Spinner />}><Dashboard /></Suspense>
  ```
- **Long lists** – render only visible rows (`react-window`).
- **Debounce** search inputs ([01 → debounce](01-javascript.md#7-debouncing-and-throttling)).
- **Keep state close** to where it is used.
- **Images** – lazy load, correct size (Next.js `<Image>`).
- **Server-side pagination** and caching API data (RTK Query / React Query).

---

### 10. Virtual DOM

**Q. What is the Virtual DOM?**

> 🎤 **Interview answer:** "The Virtual DOM is a light copy of the page kept in JavaScript memory. Changing the real page (DOM) is slow. So when data changes, React:
> 1. builds a new virtual copy,
> 2. compares it with the old copy to find what changed (this is called **reconciliation**),
> 3. updates **only the changed parts** on the real page.
>
> This is why `key`s in lists matter: they help React know which item is which."

**Comparing rules (simple):**
- Different element type (`<div>` → `<span>`) → rebuild that part.
- Same type → update only the changed attributes.
- Lists → use `key` to match items.

**Q. What is React Fiber?**

> 🎤 **Interview answer:** "Fiber is React's engine for updating the screen. It splits the work into small pieces, so React can pause less important work and do urgent work first, like showing what the user types. This makes features like `useTransition` possible."

---

## Extra React questions

### Q. Controlled vs uncontrolled components

> 🎤 **Interview answer:**
> - "**Controlled** – the input's value comes from React state (`value` + `onChange`). Easy to validate as the user types.
> - **Uncontrolled** – the browser keeps the value, and we read it with a `ref` when needed. Less code, fewer re-renders. React Hook Form uses this for big forms."

### Q. Why are keys needed in lists?

> 🎤 **Interview answer:** "Keys tell React which item is which between renders. They must be unique and stable, like a database id. If I use the array index and items are added, removed or re-ordered, React can mix them up, and the wrong row gets updated."

### Q. Custom hooks

> 🎤 **Interview answer:** "A custom hook is a function whose name starts with `use` and uses other hooks inside. It lets me reuse logic in many components. For example, `useFetch` handles loading, data and error. Each component that uses it gets its own separate state."

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let active = true;
    fetch(url)
      .then(r => r.json())
      .then(d => {
        if (active) {
          setData(d);
          setLoading(false);
        }
      });
    return () => { active = false; }; // ignore result if component is gone
  }, [url]);

  return { data, loading };
}
```

### Q. Rules of hooks

> 🎤 **Interview answer:** "Two rules:
> 1. Call hooks only at the **top** of the component, never inside `if`, loops or nested functions.
> 2. Call hooks only inside React components or custom hooks.
>
> Why: React remembers hooks by the **order** they are called. If an `if` skips a hook, the order changes and the data gets mixed up."

### Q. useReducer

> 🎤 **Interview answer:** "`useReducer` is like `useState`, but for complicated state with many actions. It's like a small Redux inside one component. I use it for multi-step forms or complex filters."

### Q. useEffect vs useLayoutEffect

> 🎤 **Interview answer:** "`useEffect` runs **after** the browser has drawn the screen. Right for almost everything. `useLayoutEffect` runs **before** the browser draws. I use it only when I need to measure an element and adjust it before the user sees it, to avoid a flicker."

### Q. Error boundaries

> 🎤 **Interview answer:** "An error boundary is a class component that catches errors in its child components while they render, and shows a fallback message instead of a blank white screen. It does **not** catch errors in click handlers or async code; those need normal try/catch."

### Q. Fragments

> 🎤 **Interview answer:** "`<>...</>` lets a component return many elements without adding an extra `<div>` to the page. This keeps the HTML clean."

### Q. Higher-Order Component (HOC)

> 🎤 **Interview answer:** "A HOC is a function that takes a component and returns a new, improved component, like `withAuth(Page)`, which checks login first. It was common before hooks. Today custom hooks are usually a simpler way to share logic."

### Q. Protected routes (React Router)

> 🎤 **Interview answer:** "I make a wrapper component, `PrivateRoute`. If the user is logged in, it shows the page. If not, it sends them to the login page. Important: this is only for user experience. **Real security is on the backend**, which checks the token on every API call."

```jsx
function PrivateRoute({ children }) {
  const { user } = useAuth();
  return user ? children : <Navigate to="/login" replace />;
}
```

### Q. How do you handle auth tokens in React?

> 🎤 **Interview answer:** "The refresh token is in an httpOnly cookie, which JavaScript can't read. The access token is kept in memory. I use an Axios interceptor: if an API returns 401 because the token expired, it calls `/auth/refresh` once, gets a new token, and retries the original request. If refresh fails, the user is logged out. I don't use localStorage for tokens, because injected scripts can read it."

See [03 → JWT](03-nodejs-express.md#7-jwt-and-refresh-tokens).

### Q. What is reconciliation vs rendering?

> 🎤 **Interview answer:**
> - "**Rendering** – React calls my component functions to get the new UI.
> - **Reconciliation** – React compares the new UI with the old one to find what changed.
> - **Commit** – React applies those changes to the real page.
>
> So a render doesn't always change the page. If nothing changed, nothing is updated."

### Q. React 18 features

> 🎤 **Interview answer:**
> - "**Automatic batching** – many state updates are combined into one re-render, even inside promises and timeouts.
> - **`useTransition`** – marks an update as 'not urgent', so typing stays smooth while a big list updates.
> - **`useDeferredValue`** – a similar idea for a value.
> - **`createRoot`** – the new way to start the app, which turns on these features."

---

## Next.js basics

You built a **Next.js GRN dashboard** for Air India, so expect a few questions.

### Q. Why Next.js over plain React?

> 🎤 **Interview answer:** "Next.js is built on React and adds things plain React doesn't have: routing based on the folder structure, pages rendered on the server for better SEO and faster first load, API routes inside the same project, and image optimisation. For the Air India GRN dashboard, I chose Next.js because it gave routing, fast pages and API routes in one codebase, and it fit well with our TypeScript backend."

### Q. CSR vs SSR vs SSG vs ISR

> 🎤 **Interview answer:**
> - "**CSR (Client-Side Rendering)** – the browser downloads JavaScript and builds the page. Good for dashboards behind login, where SEO doesn't matter.
> - **SSR (Server-Side Rendering)** – the server builds the HTML on **every request**. Always fresh, good for SEO, but costs server time.
> - **SSG (Static Site Generation)** – the HTML is built **once** at build time. Fastest. Good for blogs and docs.
> - **ISR (Incremental Static Regeneration)** – static pages that rebuild in the background every N seconds. Good for product pages.
>
> In Next.js, each page can use a different one."

| Type | When HTML is made | Use |
|---|---|---|
| CSR | In the browser, after JS loads | Dashboards behind login |
| SSR | On the server, for every request | Fresh, personal, SEO pages |
| SSG | At build time | Blogs, docs, marketing pages |
| ISR | At build time, rebuilt every N seconds | Product pages |

### Q. App Router: Server vs Client components

> 🎤 **Interview answer:**
> - "**Server components** (the default) – run only on the server. They can read the database or secrets directly, and send no JavaScript to the browser, so pages load faster.
> - **Client components** – start with `'use client'`. Needed when I use state, effects, click handlers or browser features.
>
> Usual pattern: get the data in a server component, and use small client components for the interactive parts."

### Q. API routes / Route handlers

> 🎤 **Interview answer:** "A file like `app/api/grn/route.ts` exports functions named `GET`, `POST` and so on. It works like a small backend inside the Next.js app. Good for forms or webhooks. For heavy logic, like the SAP sync, I keep a separate Node service."

### Q. Middleware in Next.js

> 🎤 **Interview answer:** "`middleware.ts` runs **before** a page or API is served. It's used for login checks and redirects, like sending a logged-out user to the login page before the page even loads. It should be small and fast, because it runs on every matching request."

---

[Home](README.md) | [← 04 MongoDB & SQL](04-mongodb-sql.md) | Next: [06 TypeScript →](06-typescript.md)
