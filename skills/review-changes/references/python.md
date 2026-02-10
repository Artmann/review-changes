# Python Guidelines

- Bare `except:` clauses swallowing all errors — always catch specific exceptions
- Missing context managers for resources — use `with` statements for files, connections, locks
- Mutable default arguments — use `None` as default and create inside the function:

```python
# ❌ Bad
def append_to(item, target=[]):
    target.append(item)
    return target

# ✅ Good
def append_to(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

- Missing type hints on public API functions and methods
