# Java/Kotlin Guidelines

- Unchecked exceptions crossing API boundaries without documentation
- Missing null handling — use `Optional` (Java) or nullable types with safe calls (Kotlin)
- Resource leaks — use try-with-resources (Java) or `.use {}` (Kotlin) for `Closeable` resources
- Mutable shared state without synchronization in concurrent contexts
