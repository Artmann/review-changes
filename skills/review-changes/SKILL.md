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

### 1. Determine Review Target

- **Remote PR**: If user provides PR number or URL (e.g., "review PR #123", "review https://github.com/org/repo/pull/123"):
  1. Checkout the PR: `gh pr checkout <PR_NUMBER>`
  2. Read PR context: `gh pr view <PR_NUMBER> --json title,body,comments`
  3. Use the PR description and comments as additional context for the review

- **Local Changes**: If no PR specified, review current branch against default branch (continue to step 2)

### 1.5 Load Repository Guidelines

Search for repository-specific coding guidelines. These take precedence over built-in guidelines.

**Discovery order** (highest to lowest priority):
1. `CLAUDE.md`, `.claude/CLAUDE.md`
2. `CODE_GUIDELINES.md`, `.github/CODE_GUIDELINES.md`, `docs/CODE_GUIDELINES.md`
3. `STYLE_GUIDE.md`, `.github/STYLE_GUIDE.md`, `docs/STYLE_GUIDE.md`
4. `CONTRIBUTING.md`, `.github/CONTRIBUTING.md`, `docs/CONTRIBUTING.md`

Extract review-relevant content (coding standards, error handling, testing, security, naming conventions). Skip non-review content (issue templates, code of conduct). User guidelines take precedence over built-in guidelines and are additive. When reviewing a remote PR, load guidelines from the remote repository.

### 2. Detect Branches

```bash
# Get current branch
git branch --show-current

# Detect default branch (try remote HEAD, fall back to main)
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main"
```

### 3. Get Changed Files

```bash
# Assess scope first
git diff --stat <default-branch>...HEAD

# Get full diff
git diff <default-branch>...HEAD
```

**Performance guardrails:**
1. Skip lock files, `.min.js`, `.min.css`, generated files, compiled output
2. If >30 changed files, prioritize source over tests/config; summarize skipped files
3. If a single file has >500 lines changed, summarize rather than line-by-line review
4. If total diff >3000 lines, select the ~20 most important files and summarize the rest

### 4. Analyze Changes

Use the diff as the primary input. Apply guidelines holistically across the diff rather than file-by-file. Only read full files when surrounding context is needed to understand a change.

**Review process:**
1. Identify applicable guidelines: repository-specific (from step 1.5), general (below), and language-specific (from reference files)
2. Check for critical issues first — report before scanning for minor ones
3. For each issue found, record: guideline violated, file and line number, problem description, fix suggestion
4. Skip inapplicable guidelines (no database operations → skip Database & Persistence)

### 5. Check Test Coverage

For each new or modified file containing business logic:
- Check if corresponding test file exists
- If tests exist, verify new code paths have coverage
- Flag missing tests for critical paths

### 6. Format Output

Present findings grouped by severity, ordered Critical → Major → Minor.

```
## Branch Review: `feature/xyz` → `main`

**Repository guidelines loaded:**
- `.github/CONTRIBUTING.md` - coding standards, testing requirements

*(These take precedence over built-in rules where they conflict)*

### 🔴 Critical (X issues)

**1. [Brief title]**
- **File**: `path/to/file.ts:42`
- **Problem**: Clear description of what's wrong
- **Fix**: Specific solution

### 🟠 Major (X issues)

**1. [Brief title]** `[repo]`
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

If no issues found in a category, omit that section. End with clear merge recommendation. Issues marked `[repo]` were flagged based on repository-specific guidelines. If no repository guidelines were loaded, omit the "Repository guidelines loaded" section.

**Feedback tone:** Be constructive — explain *why* a change is needed. Provide actionable suggestions. Assume positive intent. For approvals, acknowledge the specific value of the contribution.

---

## General Guidelines

These apply to every review regardless of language.

#### API & Breaking Changes

- Removed or renamed public functions/methods without deprecation period
- Changed function signatures (new required parameters, changed return types)
- Modified response shapes in API endpoints (removed fields, changed types)
- Changed default values that alter existing behavior
- Database schema changes without migration scripts

#### Authentication & Authorization (Critical)

- Missing auth checks on new endpoints or routes
- Downgraded permissions (admin-only → public)
- Hardcoded credentials or API keys
- JWT/session issues: missing expiry, weak secrets, improper validation
- CORS misconfigurations: overly permissive origins (`*` in production)

#### Database & Persistence

- Missing transactions for multi-step atomic operations
- N+1 query patterns: fetching related data in loops instead of joins/eager loading
- Missing indexes on frequently queried columns
- Unbounded queries: `SELECT *` without `LIMIT` on large tables
- SQL injection: string concatenation in queries instead of parameterized queries

#### Concurrency (Critical for data corruption)

- Shared mutable state accessed without synchronization
- Missing locks/mutexes when modifying shared resources
- Check-then-act patterns without atomicity (TOCTOU)
- Deadlock potential: acquiring multiple locks in inconsistent order

#### Async Code

- After any `await`, verify assumptions are still valid — state may have changed
- Flag when code returns success without verifying the expected outcome of an async operation

#### External API Handling

- Missing timeouts on HTTP requests
- No retry logic for transient failures
- Missing circuit breakers for repeatedly failing services
- Not distinguishing between 4xx and 5xx errors
- Missing rate limiting awareness (no backoff on 429)

#### Edge Cases & Boundaries

- Code assumes arrays/lists are non-empty
- Missing null/undefined checks on optional values
- Off-by-one errors in loops, incorrect range checks
- Type coercion issues leading to unexpected behavior
- Unicode/encoding issues: assuming ASCII, incorrect string length

#### Defensive Coding

- Using external input (APIs/users) without validation
- Not handling failure cases for operations that can fail
- Array access without verifying index is valid
- Code relying on undocumented behavior or ordering

#### Input Sanitization (Critical)

- **Command injection**: unsanitized input in shell commands — use argument arrays, not string interpolation
- **Path traversal**: user input in file paths escaping intended directories — resolve and validate paths
- **XSS**: user input rendered as HTML — escape or use safe APIs, validate URL protocols
- **Log injection**: unsanitized input forging log entries — use structured logging
- **SQL injection**: string concatenation in queries — use parameterized queries

#### Memory & Performance

- Unbounded collections (arrays/maps growing without limits)
- Memory leaks: event listeners not removed, closures holding references
- Large allocations in loops that could be reused
- Blocking synchronous I/O or CPU-heavy work on main thread
- Missing pagination: loading entire datasets

#### File System Operations

- Reading files without verifying they exist
- Writing files without ensuring parent directory exists
- Missing error handling on file operations

#### Logging & Observability

- Insufficient logging for important operations
- Excessive logging creating noise or performance issues
- Missing correlation IDs for cross-service tracing
- **Critical**: Logging sensitive data (PII, passwords, tokens)
- New features without observability hooks

#### Testing Quality

- Tests that don't assert anything meaningful
- Missing edge case coverage (only happy path)
- Flaky tests: race conditions, time dependencies, external dependencies
- Test pollution: shared state, missing cleanup
- Mocking too much: tests don't exercise real code paths

#### Accessibility

- Missing alt text on images
- Non-semantic HTML (divs for buttons, missing form labels)
- Interactive elements not reachable via keyboard
- Missing ARIA labels on icon-only buttons
- Color-only indicators

#### Code Style

- Prefer early returns over nested if statements — flat code is easier to read

#### Error Messages

- Error messages must be actionable, contextual, and specific
- Include identifiers (order IDs, user IDs) for debugging
- Don't expose stack traces or internal details to end users
- Include error codes for programmatic handling

---

## Language-Specific Guidelines

Based on file extensions in the diff, load the relevant reference:

- `.ts`, `.js`, `.mjs`, `.cjs` → Read [references/typescript.md](references/typescript.md)
- `.tsx`, `.jsx` → Read [references/react.md](references/react.md) (also load typescript.md)
- `.py` → Read [references/python.md](references/python.md)
- `.go` → Read [references/go.md](references/go.md)
- `.rs` → Read [references/rust.md](references/rust.md)
- `.java`, `.kt` → Read [references/java-kotlin.md](references/java-kotlin.md)

Only load references for languages present in the changed files.

---

## Severity Definitions

- **Critical** — Block the merge. Broken code, security vulnerabilities, data leaks, runtime failures that will crash production.
- **Major** — Should fix. Unhandled async, missing error handling, resource leaks, race conditions, missing validation.
- **Minor** — Nice to fix. Code clarity, consistency, performance, dead code, duplication, unresolved TODOs.
