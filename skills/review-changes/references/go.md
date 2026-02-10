# Go Guidelines

- Ignored error returns (`_ = err`) — always handle or propagate errors
- Unclosed resources — use `defer` to close files, connections, response bodies immediately after opening
- Data races in goroutines — protect shared state with mutexes or channels
- Nil pointer dereferences — check interface values and pointer returns before use
- Missing `context.Context` propagation in request handlers and long-running operations
