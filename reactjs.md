# React 18 — Practical Best Practices 

---

## Table of contents

1. Introduction & goals
2. Upgrade & migration checklist (quick)
3. New runtime & root API
4. Concurrent features: principles & practical use
5. The important Hooks in React 18 (with examples)
6. State management patterns
7. Component design & folder structure
8. Performance: profiling, memoization, and common pitfalls
9. Effects, lifecycle, and strict mode considerations
10. Server-side rendering & Suspense (practical)
11. Accessibility, testing, and CI
12. TypeScript, linting, and dependency hygiene
13. Deployment & production monitoring
14. Appendix: useful utilities & quick snippets

---

## 1. Introduction & goals

This guide collects practical advice and examples for building modern React apps on React 18. It focuses on adoptable patterns, small-to-large app concerns, and how to use newer APIs safely.

Goals:

* Make apps robust, predictable and performant.
* Use React 18 features (automatic batching, transitions, Suspense SSR) where they help.
* Keep examples small and copy-pastable.

---

## 2. Upgrade & migration checklist (quick)

* Update `react` and `react-dom` to `^18.x` in `package.json`.
* Switch from legacy `ReactDOM.render` to the new root API: `createRoot`.
* Run your test and CI suites; fix components that break due to Strict Mode double-invocation in dev.
* Watch out for external libraries that expect old lifecycle behaviour; update or replace.

**Example:** new root

```js
import { createRoot } from 'react-dom/client';
import App from './App';

const container = document.getElementById('root');
const root = createRoot(container);
root.render(<App />);
```

---

## 3. New runtime & root API

React 18 introduces a new root API and enables automatic batching across more contexts. Use `createRoot` for new features and better compatibility with concurrent features.

**Why:** `createRoot` is required to opt-in to concurrent features and automatic batching across microtasks.

**Tip:** Keep `StrictMode` on in development — the double-invocation exposes fragile effects early.

---

## 4. Concurrent features: principles & practical use

React 18's “concurrent” renderer enables features like transitions and selective rendering priorities. Important principle: concurrency changes *scheduling* — it does not change your component’s logic.

**Do:**

* Use `startTransition` / `useTransition` for low-priority updates (e.g., changing filters, search input that triggers heavy renders).
* Prefer `useDeferredValue` for expensive derived values.

**Don't:**

* Use transitions to hide broken UX or bypass proper data loading/error handling.

**Example — startTransition:**

```js
import { startTransition } from 'react';

function onSearchChange(e) {
  const value = e.target.value;
  // update immediate state for controlled input
  setQuery(value);
  // expensive render/update can be deprioritized
  startTransition(() => {
    setFiltered(computeFiltered(value));
  });
}
```

**useTransition hook:** lets you track pending state to show spinners only when necessary.

---

## 5. Important React 18 Hooks (with examples)

### `useTransition` / `startTransition`

Use for transitions (non-urgent updates). Show a subtle loading state.

```js
const [isPending, startTransition] = useTransition();

function onFilter(val){
  startTransition(() => setFilter(val));
}
```

### `useDeferredValue`

Good when rendering a big list derived from user input — avoid janky typing.

```js
const deferredQuery = useDeferredValue(query);
const list = useMemo(() => expensiveFilter(items, deferredQuery), [items, deferredQuery]);
```

### `useId`

Generate stable ids for hydration-safe markup (forms, aria). Works both client & server.

```js
const id = useId();
<label htmlFor={`name-${id}`}>Name</label>
<input id={`name-${id}`} />
```

### `useSyncExternalStore`

For subscribing to external stores (Redux, Zustand) in a way that's SSR-friendly.

```js
import { useSyncExternalStore } from 'react';

function useCounterStore() {
  return useSyncExternalStore(subscribe, getSnapshot);
}
```

### `useInsertionEffect`

**Only** for CSS-in-JS library authors who need to insert styles before layout effects.

---

## 6. State management patterns

### Local state vs global state

* Prefer component/local state with hooks for UI-local concerns (open/close, value of inputs).
* Use global state (Redux / Zustand / Jotai / Recoil) for app-wide data and cross-cutting concerns.
* For server-derived data prefer React Query / TanStack Query / SWR for caching, deduping, and invalidation.

**Practical combo:** Keep caching & fetching in React Query; transform data in selectors; store UI state locally.

### Avoid over-centralizing

Centralized everything becomes a maintenance burden. Keep ephemeral UI state near where it's used.

### Example — react-query + local state

```js
const { data } = useQuery(['products', q], () => fetchProducts(q));
const [selected, setSelected] = useState(null);
```

---

## 7. Component design & folder structure

One practical, scalable structure:

```
/src
  /components  # shared presentational components (small)
  /features    # feature folders (domain + pages)
    /Cart
      CartPage.jsx
      cartAPI.js
      Cart.module.css
      components/
  /hooks
  /utils
  /services
  /app
    App.jsx
    routes.jsx
```

* Co-locate feature code (components, styles, tests) to reduce cognitive load.
* Keep `components` for primitives used across features.

**Naming:** prefer `PascalCase` files for components. Keep single responsibility per component.

---

## 8. Performance: profiling, memoization, and common pitfalls

### Measure first

Use React DevTools Profiler and browser performance tab to identify bottlenecks — premature optimization is harmful.

### Avoid common anti-patterns

* Avoid passing inline objects/arrays/functions to memoized children unless necessary.
* Use `useCallback`/`useMemo` when stable references matter (e.g., dependency arrays, memoized children), but don’t overuse them.

**Example — stable callback:**

```js
const onClick = useCallback(() => doThing(id), [id]);
```

### `React.memo`

Wrap pure components that receive props and re-render unnecessarily.

```js
const Avatar = React.memo(function Avatar({ user }) { ... });
```

### Virtualize large lists

Use `react-window` or `react-virtualized` for lists > few hundred rows.

---

## 9. Effects, lifecycle, and Strict Mode considerations

React 18’s Strict Mode simulates mount/unmount/mount in development. This surface issues with effects that have side effects or missing cleanup.

**Best practices:**

* Always return cleanup from effects that subscribe or mutate external resources.

```js
useEffect(() => {
  const id = subscribe(() => {});
  return () => unsubscribe(id);
}, []);
```

* Avoid doing non-idempotent operations in `useEffect` without guarding.

**Note:** `useLayoutEffect` still runs synchronously before paint; prefer only when measuring DOM.

---

## 10. Server-side rendering & Suspense (practical)

React 18 improves SSR with streaming and Suspense support.

**Guidelines:**

* Use Suspense boundaries to split UI and show fallback UIs for slow data.
* Use frameworks (Next.js, Remix) or streaming SSR solutions compatible with React 18.
* Use `useSyncExternalStore` for external stores to be SSR-safe.

**Example — Suspense boundary:**

```js
<Suspense fallback={<Spinner/>}>
  <HeavyComponent />
</Suspense>
```

**SSR note:** keep `useId()` for hydration-safe ids.

---

## 11. Accessibility, testing, and CI

### Accessibility

* Use semantic HTML and ARIA only when necessary.
* Use `eslint-plugin-jsx-a11y` and axe in CI.
* Test keyboard navigation and screen reader flow for critical paths.

### Testing

* Use React Testing Library + Jest for unit and integration tests. Prefer testing user behaviour over internal implementation.
* For E2E use Playwright or Cypress.

**React 18 testing tip:** when testing Suspense or asynchronous updates, prefer `waitFor` and `findBy` queries.

---

## 12. TypeScript, linting, and dependency hygiene

* Use TypeScript for medium+ codebases to catch bugs early. Keep `strict` turned on for safety.
* Use ESLint with `eslint-plugin-react` and rules for hooks (`react-hooks/rules-of-hooks`).
* Use `turbo` / `pnpm` / lockfiles for reproducible installs.

**Example eslint rule for hooks:** ensures dependencies are declared.

---

## 13. Deployment & production monitoring

* Use source maps for error reporting (Sentry, Bugsnag) — but keep them secure.
* Monitor RUM (Real User Monitoring) and backend metrics for performance regressions.
* Leverage CDNs and split bundles — use code-splitting (`React.lazy`) for large modules.

---

## 14. Appendix: useful utilities & quick snippets

### Debounce hook:

```js
import { useRef } from 'react';
function useDebouncedCallback(fn, delay) {
  const t = useRef();
  return (...args) => {
    clearTimeout(t.current);
    t.current = setTimeout(() => fn(...args), delay);
  };
}
```

### Safe async effect pattern

```js
useEffect(() => {
  let cancelled = false;
  async function load(){
    const data = await fetchData();
    if(!cancelled) setState(data);
  }
  load();
  return () => { cancelled = true };
}, [deps]);
```

---

