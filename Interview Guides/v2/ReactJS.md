# React Interview Questions – Sorted by Importance for SDE Interviews

This guide organizes the most frequently asked React interview questions into four tiers, from **must-know basics** to **advanced deep dives**. Each question includes a polished answer, a one‑line TL;DR, and key terms to remember.

---

## Top 10 – The Essentials (Q1–Q10)

These questions form the backbone of any React interview. You should be able to answer all of them fluently.

### Q1. How does React work? (Virtual DOM & Reconciliation)
**Polished answer:**  
React builds a lightweight in‑memory copy of the real DOM called the **Virtual DOM**. When state or props change, React creates a new Virtual DOM tree, diffs it against the previous one using a heuristic O(n) algorithm, and computes the minimal set of DOM mutations. Only those changes are applied to the actual DOM, making updates fast and efficient. This process is called **reconciliation**.

**TL;DR:** React updates a virtual DOM, diffs it with the previous version, and patches only the changed parts of the real DOM.

**Keywords:** Virtual DOM, diffing, reconciliation, efficient updates, O(n).

---

### Q2. What are components, props, and state? How do props and state differ?
**Polished answer:**  
Components are the building blocks of a React UI. They can be **functional** (plain functions returning JSX) or **class‑based** (extend `React.Component`).  
- **Props** are read‑only data passed from parent to child. They are immutable and used to configure a component.  
- **State** is internal, mutable data that belongs to a component and can change over time. When state changes, the component re‑renders.  

The key difference: props are passed in and cannot be changed by the component itself; state is owned and managed by the component using `useState` or `setState`.

**TL;DR:** Props are external and immutable; state is internal and mutable. Both trigger re‑renders when changed.

**Keywords:** Component, props, state, functional vs class, immutability, re‑render.

---

### Q3. What are React Hooks and why were they introduced?
**Polished answer:**  
Hooks are functions that let functional components use state, lifecycle features, and other React capabilities that were previously only available in class components. They were introduced in React 16.8 to:  
- Reuse stateful logic without changing component hierarchy (no more wrapper hell).  
- Simplify complex class components into smaller, more readable functions.  
- Avoid confusion around `this` binding.  

**Rules of Hooks:**  
1. Only call hooks at the top level (not inside loops/conditions).  
2. Only call hooks from React functions or custom hooks.

**TL;DR:** Hooks bring state and lifecycle to functional components, making code cleaner and more reusable.

**Keywords:** Hooks, useState, useEffect, functional components, rules of hooks.

---

### Q4. Explain `useState` and `useEffect` with examples.
**Polished answer:**  
**`useState`** adds local state to a functional component. It returns a state value and a setter function.  
```jsx
const [count, setCount] = useState(0);
```
Calling `setCount(newValue)` updates the state and triggers a re‑render.

**`useEffect`** handles side effects (data fetching, subscriptions, timers). It runs after render and can optionally clean up.  
```jsx
useEffect(() => {
  const timer = setTimeout(() => console.log('Tick'), 1000);
  return () => clearTimeout(timer);   // cleanup
}, []); // empty array → runs once after mount
```
The dependency array controls when the effect re‑runs.

**TL;DR:** `useState` manages state; `useEffect` manages side effects after rendering.

**Keywords:** useState, useEffect, dependency array, side effects, cleanup.

---

### Q5. What is Redux and what are its core principles? When would you use Redux over Context API?
**Polished answer:**  
Redux is a predictable state container for JavaScript apps, commonly used with React. It stores the entire application state in a single **store**, and state changes are made by dispatching **actions** that are processed by **reducers** (pure functions).  

**Three principles:**  
1. **Single source of truth** – the store is the only place state lives.  
2. **State is read‑only** – changes only happen via actions.  
3. **Changes are made with pure reducers** – no side effects inside reducers.

Use **Redux** when you have complex global state, many shared updates, or need middleware for async logic and devtools. Use **Context API** for simpler, low‑frequency global data (theme, auth token) to avoid boilerplate.

**TL;DR:** Redux provides a centralized store with predictable updates; Context is simpler for smaller apps.

**Keywords:** Redux, store, action, reducer, single source of truth, Context API.

---

### Q6. How does routing work in React? Explain React Router basics.
**Polished answer:**  
React Router enables client‑side routing in single‑page applications. It listens to the browser URL and renders the appropriate component without a full page reload.  

Key components (v6):  
- `<BrowserRouter>` wraps the app and provides routing context.  
- `<Routes>` groups all `<Route>` definitions.  
- `<Route path="/about" element={<About/>} />` defines what to render for a URL.  
- `<Link to="/about">` creates navigation links.  
- `useNavigate()` lets you programmatically navigate.  
- `useParams()` extracts dynamic URL parameters (e.g., `/users/:id`).

**TL;DR:** React Router matches URLs to components and enables SPA navigation.

**Keywords:** React Router, BrowserRouter, Routes, Route, Link, useNavigate, useParams.

---

### Q7. How do you optimize React performance? Name specific techniques.
**Polished answer:**  
Common optimizations:  
- **Memoization:** `React.memo` prevents re‑renders of components when props are unchanged. `useMemo` caches expensive computations; `useCallback` stabilizes function references.  
- **Code splitting:** Use `React.lazy` and `<Suspense>` to load components only when needed.  
- **Virtualized lists:** Render only visible rows using libraries like `react-window` or `react-virtualized`.  
- **State colocation:** Keep state as close as possible to where it’s used to reduce re‑render scope.  
- **Avoid inline functions/objects** in JSX props when combined with `React.memo`.  
- **Use `key` prop properly** in lists.

**TL;DR:** Reduce unnecessary re‑renders and initial bundle size with memoization, code splitting, and virtualization.

**Keywords:** React.memo, useMemo, useCallback, code splitting, lazy loading, virtualization.

---

### Q8. What are Higher‑Order Components (HOC)? Give an example.
**Polished answer:**  
An HOC is a function that takes a component and returns a new enhanced component. It’s used to share common logic (e.g., data fetching, authentication) across components without duplicating code.  
```jsx
function withLoading(Component) {
  return function WithLoading({ isLoading, ...props }) {
    if (isLoading) return <div>Loading...</div>;
    return <Component {...props} />;
  };
}
```
HOCs are a legacy pattern; modern React prefers hooks for logic reuse, but HOCs are still common in libraries.

**TL;DR:** HOC wraps a component to add shared behavior; it’s a function returning a new component.

**Keywords:** Higher‑Order Component, HOC, code reuse, wrapping, props.

---

### Q9. Explain the component lifecycle in React (class and functional).
**Polished answer:**  
**Class components** have three main phases:  
- **Mounting:** `constructor()`, `render()`, `componentDidMount()`.  
- **Updating:** `render()`, `componentDidUpdate()`, `shouldComponentUpdate()`.  
- **Unmounting:** `componentWillUnmount()`.  

**Functional components** use hooks to replicate lifecycle behavior:  
- `useEffect(() => {...}, [])` ≈ `componentDidMount` + `componentWillUnmount` (with cleanup).  
- `useEffect(() => {...}, [dep])` ≈ `componentDidUpdate` for specific dependencies.  
- `useLayoutEffect` runs before paint, useful for layout measurements.

**TL;DR:** Lifecycle methods manage what happens when a component is created, updated, or destroyed; hooks provide equivalent functionality in functional components.

**Keywords:** Lifecycle, componentDidMount, componentWillUnmount, useEffect, useLayoutEffect.

---

### Q10. What is the difference between controlled and uncontrolled components?
**Polished answer:**  
- **Controlled components:** The form input’s value is controlled by React state. Every change triggers an `onChange` handler that updates state. This gives React full control over the input and makes validation/synchronization easier.  
```jsx
<input value={name} onChange={e => setName(e.target.value)} />
```
- **Uncontrolled components:** The DOM maintains the input’s value. React accesses it via a `ref` when needed (e.g., on submit). Less code but less control.  
```jsx
<input ref={inputRef} />
```

**TL;DR:** Controlled = React owns the value; Uncontrolled = DOM owns the value, React reads it via ref.

**Keywords:** Controlled, uncontrolled, form handling, state, ref.

---

## Top 25 – Continuation (Q11–Q25)

### Q11. What is JSX and how is it converted to JavaScript?
**Polished answer:**  
JSX is a syntax extension that lets you write HTML‑like code inside JavaScript. It improves readability by mixing markup with logic. Browsers can’t run JSX directly; build tools like Babel transpile it to `React.createElement()` calls.  
```jsx
const element = <h1>Hello</h1>;
// becomes
const element = React.createElement('h1', null, 'Hello');
```

**TL;DR:** JSX is syntactic sugar for `React.createElement()`, compiled by Babel.

**Keywords:** JSX, Babel, createElement, transpile.

---

### Q12. Why is the `key` prop important in React lists? What happens if you use array index as key?
**Polished answer:**  
The `key` prop helps React identify which list items changed, were added, or removed. It’s essential for efficient reconciliation and preserving component state.  
Using the array index as a key is dangerous when the list can be reordered, filtered, or have items inserted/deleted – React may reuse component instances incorrectly, leading to wrong state or UI bugs. Always prefer a stable, unique ID.

**TL;DR:** Keys enable efficient list diffing; index keys break when order changes.

**Keywords:** key prop, list rendering, reconciliation, index key, stable ID.

---

### Q13. How are forms handled in React compared to plain HTML?
**Polished answer:**  
In plain HTML, form data is stored in the DOM. In React, forms are typically **controlled components**: each input’s value is tied to state, and `onChange` handlers update that state. This makes data predictable and easy to validate. Submission is handled with `onSubmit` and `e.preventDefault()` to avoid page reload.

**TL;DR:** React forms use state + onChange; submission is intercepted with preventDefault.

**Keywords:** Forms, controlled inputs, onChange, onSubmit, preventDefault.

---

### Q14. What is conditional rendering in React? Show different ways.
**Polished answer:**  
Conditional rendering displays different UI based on a condition. Common techniques:  
- **if/else statements** (outside JSX)  
- **Ternary operator** inside JSX: `{isLoggedIn ? <User/> : <Guest/>}`  
- **Logical &&** for render‑or‑nothing: `{show && <Alert/>}`  
- **Element variables** to store JSX conditionally.

**TL;DR:** Use JavaScript conditions to decide what JSX to render.

**Keywords:** Conditional rendering, ternary, logical &&, if‑else.

---

### Q15. Explain `useEffect` cleanup and dependency array behavior.
**Polished answer:**  
`useEffect` can return a cleanup function that runs before the component unmounts or before the next effect execution (if dependencies change). This is used to clear timers, cancel subscriptions, or abort requests.  
The dependency array controls when the effect runs:  
- `[]` – runs once on mount, cleanup on unmount.  
- `[dep1, dep2]` – runs when any dependency changes.  
- Omitted – runs after every render.

**TL;DR:** Cleanup prevents memory leaks; dependencies decide when effects re‑run.

**Keywords:** useEffect, cleanup function, dependencies, unmount.

---

### Q16. `useReducer` vs `useState` – when to use which?
**Polished answer:**  
`useState` is for simple, independent state values. `useReducer` is better for complex state logic where the next state depends on the previous one, or when multiple sub‑values update together. It centralizes update logic in a reducer function and uses `dispatch` to trigger changes, similar to Redux but local to a component.

**TL;DR:** useState for simple state; useReducer for complex, interdependent state logic.

**Keywords:** useState, useReducer, reducer, dispatch, complex state.

---

### Q17. What are custom hooks? Give an example.
**Polished answer:**  
Custom hooks are functions that let you extract reusable logic from components. They must start with `use` and can call other hooks. Example:  
```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch(url).then(res => res.json()).then(setData);
  }, [url]);
  return data;
}
```
Then any component can call `useFetch('/api/users')`.

**TL;DR:** Custom hooks encapsulate shared logic and follow the Rules of Hooks.

**Keywords:** Custom hooks, use prefix, logic reuse, useState/useEffect inside.

---

### Q18. What is the Context API and when should you use it?
**Polished answer:**  
Context provides a way to pass data through the component tree without prop drilling. You create a context with `createContext`, provide a value with `<Provider>`, and consume it with `useContext`.  
Use Context for global, relatively stable data like theme, locale, or authenticated user. Avoid it for high‑frequency updates because all consumers re‑render when the value changes.

**TL;DR:** Context avoids prop drilling; best for infrequently changing global data.

**Keywords:** Context API, Provider, useContext, prop drilling, re‑render.

---

### Q19. What is Redux middleware? Explain `redux-thunk`.
**Polished answer:**  
Redux middleware sits between action dispatch and reducers. It can intercept, modify, delay, or perform side effects on actions.  
`redux-thunk` allows action creators to return a function instead of an action object. That function receives `dispatch` and `getState`, enabling asynchronous logic like API calls:  
```js
const fetchUser = (id) => async dispatch => {
  dispatch({ type: 'FETCH_START' });
  const user = await api.fetchUser(id);
  dispatch({ type: 'FETCH_SUCCESS', payload: user });
};
```

**TL;DR:** Thunk enables async actions by letting action creators return functions.

**Keywords:** Redux middleware, redux-thunk, async actions, dispatch, side effects.

---

### Q20. How do you navigate programmatically in React Router v6?
**Polished answer:**  
Use the `useNavigate` hook, which returns a function to change routes.  
```jsx
const navigate = useNavigate();
navigate('/dashboard');            // go to /dashboard
navigate(-1);                      // go back
navigate('/user', { state: { id: 1 } }); // pass state
```

**TL;DR:** `useNavigate` replaces `useHistory` for programmatic navigation.

**Keywords:** useNavigate, programmatic navigation, React Router v6.

---

### Q21. What is code splitting and how do you implement it with `React.lazy` and `Suspense`?
**Polished answer:**  
Code splitting breaks the main bundle into smaller chunks loaded on demand, reducing initial load time. In React:  
```jsx
const LazyComponent = React.lazy(() => import('./LazyComponent'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <LazyComponent />
    </Suspense>
  );
}
```
`React.lazy` dynamically imports the component; `Suspense` shows a fallback while loading.

**TL;DR:** Load components only when needed; Suspense handles the loading state.

**Keywords:** Code splitting, React.lazy, Suspense, dynamic import, bundle size.

---

### Q22. What are error boundaries in React?
**Polished answer:**  
Error boundaries are React components that catch JavaScript errors in their child component tree, log them, and display a fallback UI instead of crashing the whole app. They are implemented with `componentDidCatch` or `static getDerivedStateFromError` in class components. They do **not** catch errors in event handlers, async code, or the boundary itself.

**TL;DR:** Error boundaries prevent entire app crashes by catching child render errors.

**Keywords:** Error boundary, componentDidCatch, fallback UI, class component.

---

### Q23. What is `React.memo` and how does it differ from `useMemo` and `useCallback`?
**Polished answer:**  
- `React.memo` is a higher‑order component that memoizes a whole component; it re‑renders only if props change (shallow compare).  
- `useMemo` memoizes a computed **value** to avoid expensive recalculations.  
- `useCallback` memoizes a **function** reference, preventing child re‑renders when passed as a prop.  

**TL;DR:** `React.memo` = memoized component; `useMemo` = memoized value; `useCallback` = memoized function.

**Keywords:** React.memo, useMemo, useCallback, memoization, props stability.

---

### Q24. What is the difference between class components and functional components?
**Polished answer:**  
- **Class components** extend `React.Component`, use `render()`, have lifecycle methods, and manage state with `this.state` and `this.setState`.  
- **Functional components** are plain functions that return JSX. With hooks, they can use state (`useState`), lifecycle (`useEffect`), and other features, making them simpler and more testable.  
Functional components are the modern standard.

**TL;DR:** Functional components + hooks are now preferred over class components.

**Keywords:** Class component, functional component, hooks, lifecycle, state.

---

### Q25. What is the purpose of the `key` prop in list rendering?
**Polished answer:**  
The `key` prop gives React a stable identity for each list item. It allows React to efficiently update the DOM by reusing existing elements instead of re‑rendering the entire list. Keys should be unique among siblings and preferably be a stable ID (not index).

**TL;DR:** Keys let React track items and apply minimal updates.

**Keywords:** key prop, list diffing, stable identity, index key issues.

---

## Top 50 – Continuation (Q26–Q50)

### Q26. Explain React’s reconciliation process and diffing algorithm.
**Polished answer:**  
Reconciliation is the process React uses to update the DOM when state or props change. It compares the previous Virtual DOM tree with the new one using a heuristic O(n) diffing algorithm. The algorithm assumes:  
- Elements of different types produce different trees → replace entire subtree.  
- Elements of the same type are compared attribute‑wise; only changed attributes are updated.  
- `key` prop is used to match children in lists.  

These heuristics make updates fast without exhaustive comparisons.

**TL;DR:** React diffs virtual trees and patches minimal changes; keys help match list items.

**Keywords:** Reconciliation, diffing, Virtual DOM, keys, O(n), subtree replacement.

---

### Q27. What are React Portals and when would you use them?
**Polished answer:**  
Portals allow rendering a component’s DOM node outside its parent component’s DOM hierarchy. They are created with `ReactDOM.createPortal(child, container)`. Common use cases: modals, tooltips, dropdowns – any UI that needs to escape CSS constraints like `overflow: hidden` or z‑index stacking.

**TL;DR:** Portals render children into a different DOM node while preserving React’s component tree behavior.

**Keywords:** Portals, createPortal, modal, tooltip, escape layout.

---

### Q28. What is `forwardRef` and when is it needed?
**Polished answer:**  
`forwardRef` allows a parent component to pass a `ref` to a child functional component, which then attaches it to a DOM element or another component. It’s needed because `ref` is not a regular prop and cannot be passed directly to functional components. It enables imperative actions like focusing an input, measuring dimensions, or integrating with third‑party DOM libraries.

**TL;DR:** `forwardRef` forwards refs to child components for imperative DOM access.

**Keywords:** forwardRef, ref, imperative handle, DOM access.

---

### Q29. What is StrictMode in React and why is it used?
**Polished answer:**  
`<StrictMode>` is a development‑only wrapper that activates additional checks and warnings for its children. It helps identify unsafe lifecycle methods, legacy string refs, deprecated `findDOMNode`, and unexpected side effects. It double‑invokes certain functions (like `render`, `useState` updater, `useEffect`) in development to surface potential issues, but has no effect in production.

**TL;DR:** StrictMode is a dev tool that flags potential problems and encourages best practices.

**Keywords:** StrictMode, development checks, deprecated APIs, double invocation.

---

### Q30. Explain server‑side rendering (SSR) vs client‑side rendering (CSR).
**Polished answer:**  
- **SSR:** The server generates the initial HTML for each request and sends it to the browser. The browser displays content immediately, then React **hydrates** to attach event listeners. Advantages: faster first paint, better SEO. Disadvantages: higher server load, more complex setup.  
- **CSR:** The browser downloads a minimal HTML page and then renders everything using JavaScript. The page is blank until JS loads and executes. Advantages: simpler deployment, smoother transitions. Disadvantages: slower initial load, SEO challenges.

**TL;DR:** SSR renders on server first; CSR renders entirely in browser. SSR better for SEO/first load.

**Keywords:** SSR, CSR, hydration, initial load, SEO, Next.js.

---

### Q31. What is hydration in React?
**Polished answer:**  
Hydration is the process by which React attaches event listeners and restores component state to server‑rendered HTML. It converts static markup into a fully interactive React application. The server‑rendered HTML must exactly match what the client expects; mismatches cause errors.

**TL;DR:** Hydration makes SSR‑generated HTML interactive.

**Keywords:** Hydration, SSR, event listeners, state restoration.

---

### Q32. What are render props and how do they compare to HOCs?
**Polished answer:**  
A render prop is a technique where a component’s prop is a function that returns JSX. This allows sharing code by injecting dynamic content.  
```jsx
<DataProvider render={data => <Display data={data} />} />
```
HOCs wrap components; render props make the logic more explicit and avoid “wrapper hell”, but both are largely replaced by hooks in modern React.

**TL;DR:** Render props pass a function as a prop to control rendering; hooks often replace them now.

**Keywords:** Render props, function as prop, HOC, code reuse, hooks.

---

### Q33. When should you use `useLayoutEffect` instead of `useEffect`?
**Polished answer:**  
`useLayoutEffect` runs synchronously after DOM mutations but before the browser paints. Use it when you need to read layout information (e.g., element dimensions) and make changes that must be visible immediately without flicker. `useEffect` runs after paint and is suitable for most side effects (data fetching, subscriptions). Using `useLayoutEffect` for heavy work can block rendering and hurt performance.

**TL;DR:** `useLayoutEffect` for pre‑paint layout adjustments; `useEffect` for post‑paint side effects.

**Keywords:** useLayoutEffect, useEffect, paint, synchronous, layout measurement.

---

### Q34. What is `useImperativeHandle` and when would you use it?
**Polished answer:**  
`useImperativeHandle` customizes the instance value exposed to parent components when using `ref`. It’s used with `forwardRef` to expose specific methods or properties from a child component. Example: exposing a `focus` method on a custom input.  
```jsx
useImperativeHandle(ref, () => ({
  focus: () => inputRef.current.focus()
}));
```
This avoids exposing internal DOM details.

**TL;DR:** `useImperativeHandle` lets you define a custom API for a ref‑forwarded component.

**Keywords:** useImperativeHandle, forwardRef, ref, custom instance.

---

### Q35. Explain `useMemo` and `useCallback` in detail. When would you use each?
**Polished answer:**  
- `useMemo(callback, deps)` memoizes the **result** of an expensive computation. It recalculates only when dependencies change. Use for heavy data transformations, filtering, or derived values.  
- `useCallback(callback, deps)` memoizes the **function reference** itself. Use when passing callbacks to memoized child components to prevent unnecessary re‑renders.  

Both help stabilize references/values and avoid repeated work.

**TL;DR:** `useMemo` caches values; `useCallback` caches functions.

**Keywords:** useMemo, useCallback, memoization, dependencies, re‑render.

---

### Q36. How does Redux Toolkit simplify Redux?
**Polished answer:**  
Redux Toolkit (RTK) is the official, opinionated way to write Redux. It reduces boilerplate by providing:  
- `configureStore()` – sets up store with good defaults (e.g., Redux DevTools, thunk middleware).  
- `createSlice()` – generates actions and reducers in one place using Immer for immutable updates.  
- `createAsyncThunk()` – standardizes async action handling with pending/fulfilled/rejected states.  
RTK is the recommended approach for all new Redux code.

**TL;DR:** RTK removes Redux boilerplate with `createSlice` and `createAsyncThunk`.

**Keywords:** Redux Toolkit, createSlice, configureStore, createAsyncThunk, Immer.

---

### Q37. What is `createAsyncThunk` in Redux Toolkit?
**Polished answer:**  
`createAsyncThunk` is a function that generates an action creator for asynchronous operations. It automatically dispatches `pending`, `fulfilled`, and `rejected` action types based on the promise lifecycle. Example:  
```js
const fetchUser = createAsyncThunk('user/fetch', async (id) => {
  return await api.fetchUser(id);
});
```
Reducers can listen for these types to update state accordingly.

**TL;DR:** `createAsyncThunk` streamlines async action handling in Redux.

**Keywords:** createAsyncThunk, async actions, pending/fulfilled/rejected.

---

### Q38. What are selectors in Redux and why use them?
**Polished answer:**  
Selectors are functions that extract and derive data from the Redux store state. They encapsulate state shape and transformation logic, promoting reuse and testability. `reselect` library provides memoized selectors to avoid recalculating derived data if the underlying state hasn’t changed.

**TL;DR:** Selectors read and transform state; memoized selectors improve performance.

**Keywords:** Selectors, state derivation, reselect, memoization.

---

### Q39. Why is immutability important in Redux?
**Polished answer:**  
Immutability ensures that state is never modified directly; instead, a new state object is returned. This allows Redux to efficiently detect changes by reference comparison (`oldState === newState`), enables time‑travel debugging, and keeps reducers predictable. Libraries like Immer (in RTK) let you write mutable‑looking code while preserving immutability.

**TL;DR:** Immutability makes state changes predictable, performant, and debuggable.

**Keywords:** Immutability, reducer purity, reference equality, time‑travel.

---

### Q40. How do you combine multiple reducers in Redux?
**Polished answer:**  
Use `combineReducers` from Redux to merge several reducers into a single root reducer. Each reducer manages a slice of the state, and the resulting state object has a key for each slice.  
```js
const rootReducer = combineReducers({
  users: usersReducer,
  posts: postsReducer
});
```
In Redux Toolkit, you can pass an object to `configureStore` and it automatically combines them.

**TL;DR:** `combineReducers` merges reducers into one, each handling its own slice.

**Keywords:** combineReducers, root reducer, state slices, Redux Toolkit.

---

### Q41. What is the difference between Redux Thunk and Redux Saga?
**Polished answer:**  
- **Thunk** is simpler: action creators can return functions that receive `dispatch` and `getState`. It’s good for basic async flows like API calls.  
- **Saga** uses ES6 generators to manage complex side effects (e.g., concurrency, cancellation, orchestration). It is more powerful but adds complexity and a larger learning curve.  

Choose Thunk for most apps; Saga when you need advanced control.

**TL;DR:** Thunk = simple async functions; Saga = generator‑based, handles complex flows.

**Keywords:** Redux Thunk, Redux Saga, async side effects, generators, cancellation.

---

### Q42. What is time‑travel debugging in Redux?
**Polished answer:**  
Time‑travel debugging lets you step back and forth through the history of dispatched actions and state changes. Redux DevTools records every action and state snapshot. Because state updates are pure and immutable, you can rewind to any previous state and replay actions, making it easy to reproduce and fix bugs.

**TL;DR:** Redux DevTools allows rewinding state history for debugging.

**Keywords:** Time‑travel, Redux DevTools, action history, state snapshots.

---

### Q43. How do you implement authentication in React Router?
**Polished answer:**  
Create a **protected route** component that checks authentication status (e.g., from context or Redux). If authenticated, render the requested component; otherwise redirect to login.  
```jsx
function ProtectedRoute({ children }) {
  const { user } = useAuth();
  return user ? children : <Navigate to="/login" />;
}
```
Wrap routes that need protection with this component.

**TL;DR:** Protected routes check auth and redirect unauthenticated users.

**Keywords:** Protected route, authentication, redirect, Navigate, useAuth.

---

### Q44. What are route guards in React Router?
**Polished answer:**  
Route guards are mechanisms that control navigation by executing checks before rendering a route. They can enforce authentication, authorization, or fetch required data. Implementation: a wrapper component (like `ProtectedRoute`) that conditionally renders the route component or redirects.

**TL;DR:** Route guards protect routes by verifying conditions before access.

**Keywords:** Route guard, protected route, authorization, redirect.

---

### Q45. Difference between `BrowserRouter` and `HashRouter`.
**Polished answer:**  
- `BrowserRouter` uses the HTML5 history API (`pushState`, `popstate`). URLs look clean (`/about`). Requires server configuration to handle deep links.  
- `HashRouter` uses the hash portion of the URL (`/#/about`). Works without server config and is suitable for static hosting or environments without server‑side routing. Slightly less SEO‑friendly.

**TL;DR:** BrowserRouter uses history API; HashRouter uses URL hash.

**Keywords:** BrowserRouter, HashRouter, history API, hash, server config.

---

### Q46. What are CSS Modules and how do they help?
**Polished answer:**  
CSS Modules are CSS files where class names are scoped locally by default. At build time, each class name is renamed to a unique identifier, preventing style collisions across components. Usage: create a file like `styles.module.css`, import it as an object, and apply classes with `className={styles.button}`.

**TL;DR:** CSS Modules scope styles locally to avoid global conflicts.

**Keywords:** CSS Modules, local scope, unique class names, style isolation.

---

### Q47. How do you handle multiple input fields in a React form?
**Polished answer:**  
Use a single state object with all field values and a generic `onChange` handler that updates the specific field using the input’s `name` attribute.  
```jsx
const [form, setForm] = useState({ name: '', email: '' });
const handleChange = (e) => {
  setForm({ ...form, [e.target.name]: e.target.value });
};
```
Each input sets `name` and `value`.

**TL;DR:** One state object + computed property names to update fields dynamically.

**Keywords:** Multiple inputs, dynamic state, `[e.target.name]`, form handling.

---

### Q48. What is event bubbling and how do you stop it in React?
**Polished answer:**  
Event bubbling is when an event triggered on a child element propagates up to its ancestors. In React, you can stop it by calling `event.stopPropagation()` inside the handler.  
```jsx
const handleChildClick = (e) => {
  e.stopPropagation();
  // child logic
};
```
This prevents parent handlers from being called.

**TL;DR:** Stop bubbling with `e.stopPropagation()`.

**Keywords:** Event bubbling, stopPropagation, synthetic events.

---

### Q49. How do you render large lists efficiently in React?
**Polished answer:**  
For very large lists, use **virtualization** libraries like `react-window` or `react-virtualized`. These render only the visible portion of the list, drastically reducing the number of DOM nodes and improving performance. Combined with `React.memo` and proper keys, you can handle thousands of items smoothly.

**TL;DR:** Virtualization renders only visible items, avoiding DOM overload.

**Keywords:** Virtualization, react-window, react-virtualized, large lists, performance.

---

### Q50. What is the difference between `React.PureComponent` and `React.Component`?
**Polished answer:**  
`React.PureComponent` implements `shouldComponentUpdate` with a shallow prop and state comparison. It re‑renders only if something actually changed at a shallow level. `React.Component` always re‑renders when `setState` is called, even if values are identical. Use `PureComponent` to optimize class components when props/state are simple and immutable.

**TL;DR:** PureComponent skips re‑renders on shallow equality; Component always re‑renders.

**Keywords:** PureComponent, shouldComponentUpdate, shallow comparison, class optimization.

---

## Top 100 – Continuation (Q51–Q100)

### Q51. What is the difference between ReactDOM and React?
**Polished answer:**  
`React` provides the core library for creating components, handling JSX, and managing the virtual DOM. `ReactDOM` is the renderer that interacts with the browser’s DOM, providing methods like `createRoot`, `render`, `hydrate`, and `unmountComponentAtNode`. In React 18, `ReactDOM.createRoot` is used for concurrent rendering.

**TL;DR:** React = component logic; ReactDOM = browser DOM rendering.

**Keywords:** ReactDOM, React core, createRoot, renderer.

---

### Q52. What are React Fragments and why use them?
**Polished answer:**  
Fragments allow grouping multiple elements without adding an extra DOM node. Use `<React.Fragment>` or the short syntax `<> ... </>`. They help avoid unnecessary wrapper `<div>`s that can break layouts or add noise to the DOM. The long form supports a `key` prop when rendering lists.

**TL;DR:** Fragments group elements without extra DOM nodes.

**Keywords:** Fragment, `<>`, avoid extra div, key.

---

### Q53. What is the purpose of `ReactDOM.createRoot`?
**Polished answer:**  
`ReactDOM.createRoot` is the React 18 way to create a root for concurrent rendering. It enables new features like automatic batching, `startTransition`, and Suspense improvements.  
```jsx
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
```
The older `ReactDOM.render` is deprecated.

**TL;DR:** `createRoot` enables concurrent rendering features in React 18.

**Keywords:** createRoot, concurrent rendering, React 18, root.render.

---

### Q54. What is the `exact` prop in React Router v5? (Note: not used in v6)
**Polished answer:**  
In React Router v5, the `exact` prop on a `<Route>` ensures that the route matches only when the URL path is exactly equal to the route’s path, not a prefix. For example, `<Route exact path="/" component={Home} />` would not match `/about`. In v6, exact matching is the default behavior.

**TL;DR:** `exact` forces full path matching (v5 only; v6 does this by default).

**Keywords:** exact prop, React Router v5, route matching.

---

### Q55. Difference between `<Switch>` (v5) and `<Routes>` (v6).
**Polished answer:**  
`<Switch>` (v5) renders the first matching route from top to bottom; order matters. `<Routes>` (v6) uses a ranking algorithm to find the best match automatically; order doesn’t matter. `<Routes>` also supports `element` prop instead of `component`/`render`, and nested routes are simpler. `<Switch>` is deprecated.

**TL;DR:** `<Routes>` is the v6 replacement with better matching and cleaner syntax.

**Keywords:** Switch, Routes, v6 changes, route matching.

---

### Q56. What is `activeClassName` in React Router? (v5)
**Polished answer:**  
`activeClassName` is a prop on `<NavLink>` that specifies the CSS class to apply when the link’s route is active. In v6, this prop is removed; you can instead use a function in `className` or the `style` prop to conditionally style active links.

**TL;DR:** `activeClassName` styles active NavLinks (v5 only).

**Keywords:** NavLink, activeClassName, active styling, v6 changes.

---

### Q57. What is the `location` object in React Router?
**Polished answer:**  
The `location` object represents the current URL location. It contains properties like `pathname`, `search`, `hash`, and `state`. You can access it via `useLocation()` hook or `props.location` (in class components) to read or respond to route changes.

**TL;DR:** `location` tells you where the app currently is (URL details).

**Keywords:** location, useLocation, pathname, search, state.

---

### Q58. What is the `history` object in React Router?
**Polished answer:**  
The `history` object manages the browser’s navigation history. It provides methods like `push`, `replace`, `go`, `goBack`, and `goForward` for programmatic navigation. In v6, the `useNavigate` hook replaces direct history manipulation in most cases, but the history object is still available via context.

**TL;DR:** `history` lets you navigate and control the browser history stack.

**Keywords:** history, push, replace, navigation, v5/v6 differences.

---

### Q59. What is the `match` object in React Router?
**Polished answer:**  
The `match` object contains information about how a route’s path matched the current URL. It includes `params` (URL parameters), `path`, `url`, and `isExact`. In v6, you primarily use hooks like `useParams` and `useRouteMatch` (replaced by `useMatch`) to access match data.

**TL;DR:** `match` provides route matching details, especially URL params.

**Keywords:** match, params, route matching, useParams.

---

### Q60. What is the `withRouter` HOC in React Router? (v5)
**Polished answer:**  
`withRouter` was a higher‑order component that injected `history`, `location`, and `match` props into components that were not directly rendered by a `<Route>`. In v6, `withRouter` is removed; you can use hooks (`useNavigate`, `useLocation`, `useParams`) inside functional components instead.

**TL;DR:** `withRouter` gave routing props to non‑route components (v5 only).

**Keywords:** withRouter, routing props, v6 hooks.

---

### Q61. How do you access query parameters in React Router?
**Polished answer:**  
Use `useLocation` to get the `search` string, then parse it with `URLSearchParams`.  
```jsx
const location = useLocation();
const query = new URLSearchParams(location.search);
const id = query.get('id');
```
For class components (v5), you can use `this.props.location.search`.

**TL;DR:** Parse `location.search` with `URLSearchParams`.

**Keywords:** query parameters, useLocation, URLSearchParams.

---

### Q62. How do you handle 404 errors (page not found) in React Router?
**Polished answer:**  
Add a catch‑all route at the end of your `<Routes>` with a `path="*"` and an element that renders a 404 component.  
```jsx
<Routes>
  <Route path="/" element={<Home/>} />
  <Route path="/about" element={<About/>} />
  <Route path="*" element={<NotFound/>} />
</Routes>
```

**TL;DR:** Use a wildcard `path="*"` route to match undefined URLs.

**Keywords:** 404, catch‑all route, wildcard path.

---

### Q63. What is nested routing in React Router?
**Polished answer:**  
Nested routing allows defining routes inside other routes, reflecting a hierarchical UI. In v6, you can nest `<Route>` elements inside a parent route and use `<Outlet>` to render child components.  
```jsx
<Route path="/dashboard" element={<Dashboard/>}>
  <Route path="stats" element={<Stats/>} />
  <Route path="settings" element={<Settings/>} />
</Route>
```
This keeps URL structure aligned with component tree.

**TL;DR:** Nested routes mirror component hierarchy; `<Outlet>` renders child routes.

**Keywords:** Nested routing, Outlet, child routes, hierarchical URLs.

---

### Q64. Difference between `react-router-dom` and `react-router-native`.
**Polished answer:**  
`react-router-dom` is for web applications; it provides components like `BrowserRouter`, `Link`, and uses the browser’s DOM and history API. `react-router-native` is for React Native apps; it provides `NativeRouter`, uses platform navigation, and renders native components. Both share the core `react-router` logic.

**TL;DR:** `react-router-dom` for web, `react-router-native` for mobile (React Native).

**Keywords:** react-router-dom, react-router-native, platform‑specific routers.

---

### Q65. What is the purpose of the `Redirect` component? (v5) / `Navigate` in v6
**Polished answer:**  
In v5, `<Redirect to="/login" />` renders a redirect to another route. In v6, it has been replaced by `<Navigate to="/login" />`, which works similarly but is a component that must be rendered inside a route or component. It can also take a `replace` prop to replace the history entry.

**TL;DR:** `Redirect`/`Navigate` declaratively redirects to another route.

**Keywords:** Redirect, Navigate, v6 replacement, declarative redirect.

---

### Q66. How to implement automatic redirect after login?
**Polished answer:**  
In the login component, after successful authentication, use `useNavigate` to programmatically go to the desired page. Alternatively, conditionally render `<Navigate to="/dashboard" />` when the login state becomes `true`.  
```jsx
if (loggedIn) return <Navigate to="/dashboard" replace />;
```

**TL;DR:** Use `useNavigate` or `<Navigate>` after login succeeds.

**Keywords:** Automatic redirect, login, useNavigate, Navigate.

---

### Q67. How to pass data between sibling components using React Router?
**Polished answer:**  
One approach is to use route parameters or query strings. Example: from HomePage, navigate to `/about/${data}`; in AboutPage, extract `data` from `useParams`. For more complex data, you can use `navigate('/about', { state: { ... } })` and read it with `useLocation().state`.

**TL;DR:** Pass data via URL params or location state when navigating between siblings.

**Keywords:** Sibling communication, route params, location state, navigate.

---

### Q68. How to re‑render the view when the browser is resized?
**Polished answer:**  
Attach a `resize` event listener in `useEffect` and update state with the new dimensions. Clean up the listener on unmount.  
```jsx
useEffect(() => {
  const handleResize = () => setSize({ width: window.innerWidth, height: window.innerHeight });
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

**TL;DR:** Use `resize` event listener to update state; cleanup on unmount.

**Keywords:** resize event, window dimensions, useEffect cleanup.

---

### Q69. What are the different ways to style React components?
**Polished answer:**  
Common methods:  
- **Inline styles** (object in `style` prop)  
- **External CSS** files (global or scoped with CSS Modules)  
- **CSS Modules** (local scoping)  
- **Styled‑components** (CSS‑in‑JS, dynamic props)  
- **Tailwind CSS** / utility classes  
- **CSS preprocessors** (Sass/Less)  

Each has trade‑offs; CSS Modules and CSS‑in‑JS are popular in large apps.

**TL;DR:** Inline, CSS files, CSS Modules, styled‑components, Tailwind, etc.

**Keywords:** Styling, CSS Modules, styled-components, Tailwind, inline styles.

---

### Q70. What is Material UI and how do you use it in React?
**Polished answer:**  
Material UI (MUI) is a popular React component library implementing Google’s Material Design guidelines. It provides pre‑built, customizable components like buttons, forms, tables, and themes. Install with `@mui/material` and use components directly:  
```jsx
import Button from '@mui/material/Button';
<Button variant="contained">Click</Button>
```

**TL;DR:** MUI gives ready‑made, themeable UI components following Material Design.

**Keywords:** Material UI, MUI, component library, Material Design.

---

### Q71. What is Axios and how do you use it for HTTP requests?
**Polished answer:**  
Axios is a promise‑based HTTP client for making requests from the browser or Node.js. It supports interceptors, cancellation, automatic JSON parsing, and a simpler API than `fetch`. Common usage:  
```jsx
import axios from 'axios';
useEffect(() => {
  axios.get('/api/data')
    .then(res => setData(res.data))
    .catch(err => console.error(err));
}, []);
```

**TL;DR:** Axios is a feature‑rich HTTP client; easier and more capable than fetch.

**Keywords:** Axios, HTTP requests, interceptors, JSON parsing.

---

### Q72. What is CORS and how do you handle it in React with Axios?
**Polished answer:**  
CORS (Cross‑Origin Resource Sharing) is a security mechanism that restricts web pages from making requests to a different domain. To handle it, the **server** must include appropriate CORS headers (like `Access-Control-Allow-Origin`). On the React/Axios side, you can set `withCredentials` for cookies, but the actual fix is server configuration. During development, you can use a proxy in `package.json` to avoid CORS.

**TL;DR:** CORS is a server‑side issue; React can only handle errors or use a dev proxy.

**Keywords:** CORS, Axios, cross‑origin, proxy, Access-Control-Allow-Origin.

---

### Q73. How do you test React components that use hooks?
**Polished answer:**  
Use testing libraries like **React Testing Library** (recommended) or Enzyme. Render the component with `render()`, interact with it using `fireEvent` or `userEvent`, and assert on the output with `expect`. Hooks are tested indirectly by verifying component behavior (e.g., state updates cause UI changes). For custom hooks, you can use `renderHook` from `@testing-library/react`.

**TL;DR:** Use React Testing Library to render, interact, and assert on hook‑driven components.

**Keywords:** Testing, React Testing Library, renderHook, fireEvent.

---

### Q74. What are the rules of Hooks in detail? Why are they important?
**Polished answer:**  
1. **Only call hooks at the top level** – never inside loops, conditions, or nested functions. This ensures hooks are called in the same order on every render, which React relies on to correctly associate state.  
2. **Only call hooks from React functions** – functional components or custom hooks. Don’t call from regular JavaScript functions.  
Custom hooks must start with `use` so React can identify them.

**TL;DR:** Hooks must be called unconditionally at the top level and only from React functions.

**Keywords:** Rules of hooks, order, top level, custom hooks.

---

### Q75. What happens if you omit the dependency array in `useEffect`?
**Polished answer:**  
If you omit the array (pass no second argument), the effect runs after **every render**. This can lead to performance issues or infinite loops if the effect updates state that causes re‑renders. It’s rarely what you want; most effects should have at least an empty array `[]` (run once) or specific dependencies.

**TL;DR:** No dependency array = effect runs after every render.

**Keywords:** useEffect, dependency array, every render, infinite loop.

---

### Q76. What is lazy initialization with `useState`?
**Polished answer:**  
`useState` accepts either a direct value or a function that returns the initial state. The function form is called only once on the initial render, which is useful for expensive initial computations.  
```jsx
const [state, setState] = useState(() => {
  const initial = expensiveComputation();
  return initial;
});
```
This avoids recalculating the initial value on every render.

**TL;DR:** Pass a function to `useState` to lazily initialize state (runs once).

**Keywords:** lazy initialization, useState, expensive initial state, function argument.

---

### Q77. Can you use Hooks in class components?
**Polished answer:**  
No. Hooks are only for functional components and custom hooks. They rely on the call order and functional component execution model, which is incompatible with class component lifecycle and instance methods. Class components have their own state and lifecycle methods.

**TL;DR:** Hooks are exclusive to functional components.

**Keywords:** Hooks, class components, functional components, incompatibility.

---

### Q78. Difference between `useRef` and `createRef`.
**Polished answer:**  
- `useRef` is a hook that returns a **persistent** ref object across re‑renders in functional components.  
- `createRef` creates a **new** ref object on every render; it’s used in class components (typically in the constructor).  
Both return `{ current: ... }`. For functional components, always use `useRef`.

**TL;DR:** `useRef` persists across renders; `createRef` creates a new ref each render.

**Keywords:** useRef, createRef, ref, persistence, class vs functional.

---

### Q79. What is the purpose of `useCallback`?
**Polished answer:**  
`useCallback` returns a memoized version of a callback function that only changes if its dependencies change. It’s used to prevent unnecessary re‑renders of child components that rely on referential equality (e.g., when wrapped with `React.memo`). Without it, inline functions are recreated on every render, causing children to re‑render.

**TL;DR:** `useCallback` stabilizes function references to optimize child re‑renders.

**Keywords:** useCallback, memoized function, referential stability, React.memo.

---

### Q80. What is the purpose of `useMemo`?
**Polished answer:**  
`useMemo` memoizes the **result** of a computation. It recalculates the value only when its dependencies change. It’s used for expensive calculations (e.g., filtering large arrays, complex data transformations) to avoid repeating work on every render.

**TL;DR:** `useMemo` caches computed values to avoid expensive recalculations.

**Keywords:** useMemo, memoized value, expensive computation, dependencies.

---

### Q81. What is the purpose of `useLayoutEffect`?
**Polished answer:**  
`useLayoutEffect` is similar to `useEffect`, but it fires synchronously after DOM mutations and before the browser paints. Use it for layout measurements or adjustments that must happen before the user sees anything. It blocks painting, so avoid heavy operations.

**TL;DR:** `useLayoutEffect` runs before paint, ideal for layout‑related effects.

**Keywords:** useLayoutEffect, synchronous, pre‑paint, layout measurement.

---

### Q82. What is the purpose of `useReducer`?
**Polished answer:**  
`useReducer` is a hook for managing complex state logic. It takes a reducer function and an initial state, and returns the current state and a `dispatch` function. It’s preferable over `useState` when the next state depends on the previous one or when multiple sub‑values are updated together.

**TL;DR:** `useReducer` centralizes complex state updates through a reducer.

**Keywords:** useReducer, reducer, dispatch, complex state.

---

### Q83. What is the purpose of `useContext`?
**Polished answer:**  
`useContext` lets a functional component read the value from a React Context. It subscribes the component to context updates. Used to avoid prop drilling and share global data like theme, user, or locale.

**TL;DR:** `useContext` consumes a Context value inside functional components.

**Keywords:** useContext, Context API, global data, prop drilling.

---

### Q84. How do you create a custom hook that accepts parameters?
**Polished answer:**  
Custom hooks are just functions; they can receive any parameters. Inside the hook, use those parameters to configure state or effects.  
```jsx
function useDocumentTitle(title) {
  useEffect(() => {
    document.title = title;
  }, [title]);
}
```
Call it as `useDocumentTitle('My Page')`.

**TL;DR:** Custom hooks accept parameters like normal functions; use them inside hook logic.

**Keywords:** Custom hooks, parameters, function arguments.

---

### Q85. What is the second argument to `useState`?
**Polished answer:**  
`useState` actually takes only one argument – the initial state value (or a lazy initializer function). The “second argument” often refers to the **setter function** returned from the hook, not an argument passed to it. The setter (`setState`) is used to update the state.

**TL;DR:** There is no second argument to `useState`; the return value has state and setter.

**Keywords:** useState, initial state, setter function.

---

### Q86. What is the purpose of `useDebugValue`?
**Polished answer:**  
`useDebugValue` is used in custom hooks to display a label in React DevTools. It helps developers debug custom hooks by showing additional information about the hook’s internal state.  
```jsx
useDebugValue(isOnline ? 'Online' : 'Offline');
```

**TL;DR:** `useDebugValue` adds debug info for custom hooks in DevTools.

**Keywords:** useDebugValue, custom hooks, React DevTools, debugging.

---

### Q87. What are pure functions in Redux and why do reducers need them?
**Polished answer:**  
A pure function always returns the same output for the same input and has no side effects. Reducers must be pure to keep state updates predictable, enable time‑travel debugging, and allow efficient reference‑based change detection. They should not modify arguments, perform I/O, or call non‑pure functions.

**TL;DR:** Pure reducers ensure predictable, testable state transitions.

**Keywords:** Pure functions, reducers, predictability, side effects.

---

### Q88. What is the purpose of the `Provider` component in React Redux?
**Polished answer:**  
`Provider` is a component from `react-redux` that makes the Redux store available to all nested components via context. It should wrap the root component of your app.  
```jsx
<Provider store={store}>
  <App />
</Provider>
```

**TL;DR:** `Provider` passes the store down to all connected components.

**Keywords:** Provider, react-redux, store, context.

---

### Q89. What is the `connect` function in React Redux?
**Polished answer:**  
`connect` is a higher‑order function that connects a React component to the Redux store. It takes two optional arguments: `mapStateToProps` and `mapDispatchToProps`, and returns a new component that receives the selected state and dispatch functions as props. In modern Redux with hooks (`useSelector`, `useDispatch`), `connect` is less common but still used in class components.

**TL;DR:** `connect` links a component to the Redux store by mapping state/dispatch to props.

**Keywords:** connect, mapStateToProps, mapDispatchToProps, HOC.

---

### Q90. What is the `dispatch` function in Redux?
**Polished answer:**  
`dispatch` is a method of the Redux store that sends an action to the store. When called, it runs the reducer with the current state and the action, and updates the state. In React Redux, you access dispatch via `useDispatch` hook or `connect`’s `mapDispatchToProps`.

**TL;DR:** `dispatch` sends actions to the store to trigger state updates.

**Keywords:** dispatch, action, store, useDispatch.

---

### Q91. What is `mapStateToProps`?
**Polished answer:**  
`mapStateToProps` is a function used with `connect` that selects a slice of the Redux store state and passes it as props to the connected component. It receives the entire store state and returns an object of props. It helps avoid prop drilling and keep components decoupled from the store.

**TL;DR:** `mapStateToProps` maps Redux state to component props.

**Keywords:** mapStateToProps, connect, state selection.

---

### Q92. What is `mapDispatchToProps`?
**Polished answer:**  
`mapDispatchToProps` is a function used with `connect` that maps action creators (or dispatch calls) to props. It allows components to dispatch actions without directly accessing the store. It can be an object of action creators or a function receiving `dispatch`.

**TL;DR:** `mapDispatchToProps` provides functions to dispatch actions as props.

**Keywords:** mapDispatchToProps, connect, action creators, dispatch.

---

### Q93. What is Redux DevTools and what can you do with it?
**Polished answer:**  
Redux DevTools is a browser extension that provides real‑time inspection of the Redux store. You can view dispatched actions, state changes, jump to any previous state (time‑travel), replay actions, and even export/import state history. It greatly simplifies debugging complex state logic.

**TL;DR:** Redux DevTools enables time‑travel debugging and action inspection.

**Keywords:** Redux DevTools, time‑travel, debugging, actions, state history.

---

### Q94. How can you access a Redux store outside a React component?
**Polished answer:**  
Import the store instance from your Redux setup file. Then you can call `store.getState()` to read the current state or `store.dispatch(action)` to update it. This is useful in non‑React modules like middleware, API clients, or tests.

**TL;DR:** Import the store and use `getState`/`dispatch` directly.

**Keywords:** store, getState, dispatch, outside React.

---

### Q95. Can you dispatch an action inside a reducer?
**Polished answer:**  
Technically you can, but it is strongly discouraged. Reducers must be pure functions; dispatching inside a reducer causes side effects, breaks predictability, and can lead to infinite loops. If you need to dispatch after a state change, use middleware (like thunk) or handle it in the component.

**TL;DR:** Never dispatch inside reducers; keep them pure.

**Keywords:** Reducer purity, dispatch, anti‑pattern, side effects.

---

### Q96. How does Redux work with server‑side rendering (SSR)?
**Polished answer:**  
On the server, create a new Redux store, fetch required data, and render the React app to HTML with that store. Send the HTML and the serialized store state to the client. On the client, create a store with the received initial state (rehydration) and hydrate the app. This ensures the client starts with the same state as the server.

**TL;DR:** Server creates store with fetched data, passes state to client for hydration.

**Keywords:** SSR, Redux, rehydration, initial state, server store.

---

### Q97. How do you implement authentication in Redux?
**Polished answer:**  
Store authentication data (token, user) in the Redux store. Use middleware (Thunk/Saga) to handle async login/logout actions that call APIs and dispatch success/failure actions. Protect routes by checking the auth state in a wrapper component or route guard.

**TL;DR:** Auth state in store, async actions via middleware, protected routes.

**Keywords:** Authentication, Redux, token, thunk, protected routes.

---

### Q98. How do you test Redux reducers and actions?
**Polished answer:**  
- **Reducers:** Write unit tests that dispatch actions to the reducer and assert the returned state. Use a testing framework like Jest.  
- **Actions:** Test action creators by asserting they return the correct action object. For async actions (with thunk), you can mock dispatch and call the returned function to check dispatched actions.  
Tools: Jest, redux-mock-store.

**TL;DR:** Test reducers with direct calls; test actions by checking returned objects/function behavior.

**Keywords:** Testing, reducers, actions, Jest, redux-mock-store.

---

### Q99. What is the `reselect` library and why use it?
**Polished answer:**  
`reselect` provides **memoized selectors** for Redux. A selector computes derived data from the store state; memoization ensures it recalculates only when the input state changes, avoiding expensive recalculations on every render. This improves performance in large applications.

**TL;DR:** `reselect` memoizes selectors to prevent redundant calculations.

**Keywords:** reselect, memoized selectors, derived data, performance.

---

### Q100. What is action chaining in Redux middleware?
**Polished answer:**  
Action chaining refers to dispatching multiple actions in sequence in response to a single user or system event. For example, a thunk might dispatch `FETCH_START`, then after an API call, dispatch `FETCH_SUCCESS` or `FETCH_FAILURE`. This pattern makes async flows explicit and manageable.

**TL;DR:** Dispatching a series of actions to handle an async flow step by step.

**Keywords:** Action chaining, async actions, thunk, dispatch sequence.

---

## Final Notes

These 100 questions cover the core React ecosystem: fundamentals, hooks, state management, routing, performance, and advanced patterns. Prioritise mastering the **Top 10** and **Top 25** first; the rest build on that foundation. For interview practice, answer each question aloud, explaining the concept with a small code example if possible.

Good luck with your preparation!
