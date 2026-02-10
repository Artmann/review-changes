# TypeScript/JavaScript Guidelines

## General

- Unhandled promise rejections, missing `await`
- `any` types hiding bugs, incorrect type assertions
- Missing null checks (`?.` and `??` usage)
- Event listener cleanup in components

## Disposable Resources

Flag when objects with `dispose()` methods are created but never disposed:

- `CancellationTokenSource` — must call `dispose()` in finally block
- `Disposable` subscriptions from event listeners
- Timer handles (`setTimeout`/`setInterval`) without cleanup

## Filesystem

- Always use `path.resolve()` or `path.join()` to build paths
- Hardcoded path strings with `/` or `\` separators break cross-platform compatibility

## URLs

- Always use `URLSearchParams` to construct query strings
- Manual string concatenation is error-prone and fails to encode special characters

## Timers and Intervals

- Always store timer IDs and clear them on cleanup — uncleaned timers cause memory leaks
- Prefer recursive `setTimeout` over `setInterval` for precise timing (intervals can drift and stack if callbacks take longer than the interval)
- In Node.js, use `unref()` for background timers that shouldn't keep the process alive
