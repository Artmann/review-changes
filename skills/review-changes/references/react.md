# React Guidelines

## useEffect

- Only use `useEffect` for side effects (data fetching, subscriptions, DOM mutations)
- Do not use it to derive or build state — use `useMemo` or compute during render instead
- Ensure dependency arrays contain diffable values (primitives, stable references)
- Objects and arrays created inline will cause infinite re-renders

## Hooks Rules

- **Critical**: Hooks must be called in the same order on every render
- Never place hooks inside conditionals, loops, or after early returns

## State Mutations

- **Critical**: React state must be treated as immutable
- Direct mutations (`.push()`, `.splice()`, `obj.prop = x`) won't trigger re-renders
- Always create new references when updating state

## Hook Dependencies

- Variables used inside `useEffect`, `useCallback`, or `useMemo` must be listed in the dependency array
- Missing dependencies cause stale closures

## Stale Closures in Callbacks

- Event handlers and callbacks may capture outdated state values
- Use functional `setState(prev => ...)` updates in intervals and callbacks
- Closures captured at mount time will always see the initial value

## Functional setState for Derived Updates

- When next state depends on previous state, always use functional form: `setState(prev => ...)`
- Without functional form, batched updates can lose intermediate values

## Async Updates After Unmount

- Async operations that update state after unmount cause memory leaks
- Use `AbortController` for fetch, or a `cancelled` flag for other async work
- Always clean up in the useEffect return function

## Inline Objects Breaking Memoization

- Inline object/array literals and arrow functions in JSX create new references every render
- This breaks `React.memo` and causes unnecessary child re-renders
- Hoist constants outside the component or use `useCallback`/`useMemo`

## Missing Form preventDefault

- Form submissions without `e.preventDefault()` cause full page reloads in SPAs

## Timer Cleanup in useEffect

- Timers created in `useEffect` must be cleared in the cleanup function
- For timers cleared from event handlers, store the ID in a `useRef`

## XSS and Unsafe HTML

- **Critical**: `dangerouslySetInnerHTML` bypasses React's built-in escaping — only use with sanitized content (e.g., DOMPurify)
- Validate protocol on user-controlled URLs in `href`/`src` (block `javascript:` URLs)

## Keys in Lists

- Missing `key` props in lists, or using array index as key when items can reorder
