---
name: review-changes
description: Review code changes in a feature branch before merging. Use when asked to review a branch, review changes, check a PR, or audit code before merge. Compares the current branch against the default branch (main/master) and categorizes issues by severity (Critical, Major, Minor) with actionable solutions.
license: MIT
metadata:
  author: Christoffer Artmann
  version: "1.0.0"
---

# Review Changes

Review code changes in a feature branch and identify issues before merging.

## Workflow

### 1. Detect Branches

```bash
# Get current branch
git branch --show-current

# Detect default branch (try remote HEAD, fall back to main)
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main"
```

### 2. Get Changed Files

```bash
# List files changed between default branch and current branch
git diff --name-only <default-branch>...HEAD

# Get full diff for analysis
git diff <default-branch>...HEAD
```

### 3. Analyze Changes

Read each changed file and review for issues. Adapt review criteria to the detected language.

#### Critical Issues (Must Fix)

Block the merge. These will break production or cause security incidents.

- **Broken code**: Syntax errors, undefined variables, missing imports, type errors
- **Security vulnerabilities**: SQL injection, XSS, exposed secrets/credentials, insecure auth
- **Data leaks**: Logging sensitive data, exposing PII, missing access controls
- **Runtime failures**: Code paths that will definitely crash in production

#### Major Issues (Should Fix)

Will likely cause problems for users or developers.

- **Unhandled async**: Promises without `.catch()` or `try/catch`, missing `await`
- **Missing error handling**: Bare try blocks, swallowed exceptions, no fallback for failures
- **No timeouts/retries**: Network calls without timeout, no retry logic for flaky operations
- **Resource leaks**: Unclosed connections, file handles, event listeners not cleaned up
- **Race conditions**: Shared state mutations, missing locks, concurrent access issues
- **Missing validation**: User input not validated, missing null checks on external data

#### Minor Issues (Nice to Fix)

Won't break anything but would improve the code.

- **Code clarity**: Confusing variable names, overly complex logic, missing comments on non-obvious code
- **Consistency**: Mixed patterns, inconsistent naming, style drift from codebase norms
- **Performance**: Obvious inefficiencies, N+1 patterns, unnecessary re-computation
- **Dead code**: Unused variables, unreachable branches, commented-out code
- **TODO/FIXME**: Unresolved todos that should be addressed

### 4. Check Test Coverage

For each new or modified file containing business logic:

- Check if corresponding test file exists
- If tests exist, verify new code paths have coverage
- Flag missing tests as **Major** if for critical paths, **Minor** otherwise

### 5. Format Output

Present findings grouped by severity, ordered Critical → Major → Minor.

```
## Branch Review: `feature/xyz` → `main`

### 🔴 Critical (X issues)

**1. [Brief title]**
- **File**: `path/to/file.ts:42`
- **Problem**: Clear description of what's wrong
- **Fix**: Specific solution

### 🟠 Major (X issues)

**1. [Brief title]**
- **File**: `path/to/file.ts:87`
- **Problem**: Description
- **Fix**: Solution

### 🟡 Minor (X issues)

**1. [Brief title]**
- **File**: `path/to/file.ts:123`
- **Problem**: Description
- **Fix**: Solution

---
**Summary**: X critical, Y major, Z minor issues found.
[Ready to merge / Needs fixes before merge]
```

If no issues found in a category, omit that section. End with clear merge recommendation.

## Things to look for

Adapt review focus based on detected language and framework.

### Async Code

After any `await`, verify that assumptions made before the await are still valid. The world may have changed: files may have been deleted, state may have been modified, or resources may no longer be available.

### Code Style

Prefer early returns over nested if statements. Flat code is easier to read and reason about.

### File System Operations

Flag these as **Major** issues:

- **Missing file existence checks**: Reading files without verifying they exist first
- **Missing directory checks**: Writing files without ensuring parent directory exists
- **No error handling on fs operations**: File operations that assume success without try/catch or error callbacks

Look for patterns like:

- `fs.readFile(path)` without prior `fs.existsSync(path)` or `fs.access()` check
- `fs.writeFile(path)` without ensuring `path.dirname(path)` exists
- `open()`, `fopen()`, `File.read()` etc. without existence validation

### TypeScript/JavaScript

- Unhandled promise rejections, missing `await`
- `any` types hiding bugs, incorrect type assertions
- Missing null checks (`?.` and `??` usage)
- Event listener cleanup in components

#### Filesystem

Always use `path.resolve()` or `path.join()` to build paths. Hardcoded path strings with `/` or `\` separators break cross-platform compatibility.

```ts
// ❌ Bad - hardcoded path separators
const modelPath = "data/models";
const filePath = dir + "/" + filename;

// ✅ Good - path module handles separators
import path from "path";

const modelPath = path.join("data", "models");
const filePath = path.join(dir, filename);
```

#### URLs

Always use `URLSearchParams` to construct query strings. Manual string concatenation is error-prone and fails to properly encode special characters.

```ts
// ❌ Bad - manual concatenation
const url = `/api/search?q=${query}&page=${page}`;

// ✅ Good - URLSearchParams handles encoding
const params = new URLSearchParams({ q: query, page: String(page) });
const url = `/api/search?${params}`;
```

#### React

- Only use `useEffect` for side effects (data fetching, subscriptions, DOM mutations). Do not use it to derive or build state—use `useMemo` or compute during render instead.
- Ensure dependency arrays contain diffable values (primitives, stable references). Objects and arrays created inline will cause infinite re-renders.

### Python

- Bare `except:` clauses swallowing errors
- Missing context managers for resources (`with` statements)
- Mutable default arguments
- Missing type hints on public APIs

### Go

- Ignored error returns (`_ = err`)
- Unclosed resources (defer patterns)
- Data races in goroutines
- Nil pointer dereferences

### Rust

- `unwrap()` on Result/Option in non-test code
- Missing error propagation (`?` operator)
- Unsafe blocks without justification

### Java/Kotlin

- Unchecked exceptions, missing null handling
- Resource leaks (try-with-resources)
- Mutable shared state
