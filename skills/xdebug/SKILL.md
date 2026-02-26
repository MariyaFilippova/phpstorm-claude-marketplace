---
name: xdebug
description: Debug PHP applications with Xdebug - set breakpoints, inspect variables, step through code, debug web requests and CLI scripts
---

# Xdebug Debugging Skill

Debug PHP code interactively using the Xdebug MCP tools exposed through PhpStorm.

## Two Workflows

### 1. Dev Server Debugging (web apps)

For debugging HTTP requests to a PHP application:

```
xdebug_set_breakpoint(file, line)     # 1. Set breakpoints BEFORE starting
xdebug_start_server(projectRoot)      # 2. Start PHP built-in server + Xdebug listener
xdebug_request(url)                   # 3. Make HTTP request — pauses at breakpoint
  ... inspect / step ...              # 4. Inspection loop (see below)
xdebug_detach()                       # 5. Let the request finish, keep server running
xdebug_request(url)                   # 6. Debug another request
  ... inspect / step ...
xdebug_detach()
xdebug_stop_server()                  # 7. Tear down server when done
```

Key rules:
- Set breakpoints **before** `xdebug_request` — they are auto-applied to each new request.
- Use `xdebug_detach()` (not `xdebug_stop()`) between requests to let the HTTP response complete.
- `xdebug_stop_server()` at the end to clean up both the dev server and the Xdebug listener.

### 2. Single Script Debugging (CLI)

For debugging a standalone PHP script:

```
xdebug_set_breakpoint(file, line)     # 1. Set breakpoints BEFORE running
xdebug_single_file(path)             # 2. Run script — pauses at breakpoint
  ... inspect / step ...              # 3. Inspection loop (see below)
xdebug_stop()                         # 4. Terminate when done
```

Key rules:
- `xdebug_single_file` both starts and runs the script.
- Use `xdebug_stop()` to terminate (not `xdebug_detach`).

## Inspection Loop

When paused at a breakpoint (status = `break`), use these tools in any order:

| Tool | Purpose | When to use |
|------|---------|-------------|
| `xdebug_stack` | View call stack with file paths and line numbers | Understand where you are and how you got there |
| `xdebug_context(stackDepth?)` | List all variables in scope | See local vars and superglobals at current or caller frame |
| `xdebug_property_get(name, stackDepth?)` | Get a specific variable value | Inspect `$obj->field`, `$arr[0]`, or any named variable |
| `xdebug_eval(expression)` | Evaluate arbitrary PHP expression | Run `count($items)`, `$user->getName()`, `$a + $b`, etc. |
| `xdebug_property_set(name, value)` | Change a variable's value | Modify state mid-execution for testing |

### Stepping commands

| Tool | Effect |
|------|--------|
| `xdebug_step_over` | Execute current line, pause at next line in same scope |
| `xdebug_step_into` | Step into the next function call |
| `xdebug_step_out` | Run until current function returns, pause in caller |
| `xdebug_run` | Resume execution until next breakpoint or end of script |

### Typical inspection sequence

1. `xdebug_stack` — orient yourself
2. `xdebug_context` — see all local variables
3. `xdebug_property_get` or `xdebug_eval` — drill into specific values
4. Step or run to the next point of interest

## Tool Reference

### Session Management

| Tool | Parameters | Description |
|------|-----------|-------------|
| `xdebug_start_server` | `projectRoot` (string, required), `host` (string, default `"127.0.0.1"`), `port` (int, default `8081`) | Start PHP built-in dev server + Xdebug listener |
| `xdebug_stop_server` | *(none)* | Stop the dev server and Xdebug listener |
| `xdebug_request` | `url` (string, full URL e.g. `"http://127.0.0.1:8081/index.php"`) | HTTP request to dev server; starts debugging it. Returns `break` (hit breakpoint) or `stopping` (script completed) |
| `xdebug_single_file` | `path` (string, absolute path to PHP script) | Run and debug a single PHP script |
| `xdebug_status` | *(none)* | Current session state: `starting`, `running`, `break`, `stopping` |

### Breakpoints

| Tool | Parameters | Description |
|------|-----------|-------------|
| `xdebug_set_breakpoint` | `line` (int), `file` (string, optional — absolute or relative to project root) | Set a line breakpoint. Persists across requests |
| `xdebug_breakpoint_list` | *(none)* | List all active breakpoints |
| `xdebug_breakpoint_remove` | `id` (string, breakpoint ID from `breakpoint_list`) | Remove a breakpoint by ID |

### Execution Control

| Tool | Parameters | Description |
|------|-----------|-------------|
| `xdebug_run` | *(none)* | Resume until next breakpoint or script end |
| `xdebug_step_into` | *(none)* | Step into next function call |
| `xdebug_step_over` | *(none)* | Execute current line, pause at next line in same scope |
| `xdebug_step_out` | *(none)* | Run until current function returns |
| `xdebug_pause` | *(none)* | Pause execution at current point |

### Inspection

| Tool | Parameters | Description |
|------|-----------|-------------|
| `xdebug_stack` | *(none)* | Call stack with file paths and line numbers |
| `xdebug_context` | `stackDepth` (int, default `0`) | All variables in scope at given stack depth |
| `xdebug_property_get` | `name` (string, e.g. `"$myVar"`, `"$this->field"`), `stackDepth` (int, default `0`) | Get value of a specific variable |
| `xdebug_property_set` | `name` (string), `value` (string, PHP literal e.g. `"42"`, `"\"hello\""`, `"true"`) | Change a variable's value in current scope |
| `xdebug_eval` | `expression` (string, e.g. `"count($items)"`, `"$a + $b"`) | Evaluate PHP expression in current scope |

### Session Termination

| Tool | Parameters | Description |
|------|-----------|-------------|
| `xdebug_detach` | *(none)* | End debugging, let script finish normally. Use between requests in dev server workflow |
| `xdebug_stop` | *(none)* | Terminate PHP script immediately. Use for single-file workflow |

## Session Lifecycle Rules

1. **Breakpoints before sessions** — always call `xdebug_set_breakpoint` before `xdebug_request` or `xdebug_single_file`.
2. **Breakpoints persist** — once set, breakpoints apply to all subsequent `xdebug_request` calls until removed.
3. **`detach` vs `stop`**:
   - `xdebug_detach` = let the PHP script finish and send the HTTP response; server stays running for more requests.
   - `xdebug_stop` = kill the script immediately; use for single-file debugging or when you want to abort.
4. **One active session** — only one debug session at a time. Detach or stop before starting a new request.
5. **Inspection requires `break` status** — `stack`, `context`, `property_get`, `eval`, `property_set`, and stepping commands only work when paused at a breakpoint.

## Error Handling

| Situation | Symptom | Recovery |
|-----------|---------|----------|
| No active session | Tool returns error about missing session | Start a session with `xdebug_request` or `xdebug_single_file` |
| Not paused at breakpoint | Inspection tools fail | Set breakpoints and re-run; or use `xdebug_pause` |
| `xdebug_request` returns `stopping` | Script completed without hitting breakpoints | Verify breakpoint file paths and line numbers; re-set and retry |
| Server not reachable | Connection error on `xdebug_request` | Check `xdebug_start_server` was called; verify host/port |
| Breakpoint on wrong line | Breakpoint not hit | PHP may adjust to nearest executable line; check with `xdebug_breakpoint_list` |
| Stale session | Unexpected state after errors | Call `xdebug_stop` or `xdebug_stop_server`, then start fresh |

## Debugging Strategy Tips

- **Start broad, narrow down**: set breakpoints at entry points first, then step into suspect code.
- **Use `eval` liberally**: test hypotheses without modifying code — e.g. `xdebug_eval("isset($config['key'])")`.
- **Check `xdebug_status`** when unsure about session state.
- **Read the stack first**: `xdebug_stack` tells you where you are and which file/line to examine.
- **Use `context` before `property_get`**: see what variables exist before drilling into specific ones.
- **For web debugging**: after inspecting, always `xdebug_detach` before the next `xdebug_request`.
