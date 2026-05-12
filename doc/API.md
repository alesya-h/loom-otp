# loom-otp API reference

This reference documents the public API used by applications. Implementation
namespaces under `loom-otp.process.*`, `loom-otp.mailbox`, `loom-otp.state`, and
`loom-otp.context-mailbox` are described only where their behavior affects the
public API.

## Conventions

### Process context

Several functions must be called from inside a `loom-otp` process because they
need the current pid:

- `proc/self`
- `receive!` and `selective-receive!`
- `proc/monitor`
- `proc/exit` when sending an exit signal to another process
- `gen-server/call`
- `gen-server/start-link` and `gen-server/stop`
- supervisor dynamic operations such as `which-children` and `start-child`

When calling these APIs from a REPL or test main thread, spawn a small process
and use a `promise` to observe the result.

### Pids and refs

- Pids are Java `Thread` objects.
- Monitor references are `loom_otp.types.TRef` values.
- Registered names can be used as destinations for `send`, `exit`, `monitor`,
  and selected higher-level APIs.

### Timeouts

Timeouts are expressed in milliseconds unless a function says otherwise.
`receive!` accepts `:infinity` in lower-level function APIs; macro examples
usually omit the `(after ...)` clause to wait indefinitely.

### Return styles

The native API uses maps such as `{:ok pid}` and `{:error reason}` for
`gen-server` and supervisor start results. The `loom-otp.otplike.*`
compatibility layer uses otplike-style tuples such as `[:ok pid]` and
`[:error reason]`.

## `loom-otp.core`

System lifecycle management.

Require it as:

```clojure
(require '[loom-otp.core :as otp])
```

### `start!`

```clojure
(otp/start!)
```

Starts the process system. Call this before spawning processes.

### `stop!`

```clojure
(otp/stop!)
```

Stops the process system and interrupts running process threads.

### `status`

```clojure
(otp/status)
```

Returns `mount.lite` state status.

### `with-session`

```clojure
(otp/with-session
  (otp/start!)
  ...)
```

Creates an isolated `mount.lite` session. This is useful in tests that run in
parallel.

### `reset-all!`

```clojure
(otp/reset-all!)
```

Stops and restarts global state. Intended for tests and development utilities.

## `loom-otp.process`

Main process API.

Require it as:

```clojure
(require '[loom-otp.process :as proc])
```

### `self`

```clojure
(proc/self)
```

Returns the current process pid. Throws if called outside a process.

### `send`

```clojure
(proc/send pid message)
(proc/send :registered-name message)
```

Sends `message` to a pid or registered name.

Returns:

- `true` if the message was delivered
- `false` if the destination could not be resolved or is no longer alive

Messages are wrapped with the sender's message context so trace context can
propagate through message chains.

### `spawn`

```clojure
(proc/spawn f)
(proc/spawn f arg1 arg2)
```

Starts a new virtual-thread process and runs `(f arg1 arg2 ...)`.

Returns the child pid.

### `spawn!`

```clojure
(proc/spawn!
  (do-work)
  (more-work))
```

Macro version of `spawn`. Wraps the body in an anonymous function.

### `spawn-opt`

```clojure
(proc/spawn-opt opts f & args)
```

Starts a process with options.

Options:

| Option | Type | Purpose |
| --- | --- | --- |
| `:link` | boolean | Link the child to the current process. |
| `:monitor` | boolean | Monitor the child from the current process. |
| `:reg-name` | any registry key | Register the child under this name. |
| `:trap-exit` | boolean | Set the child process flag so exit signals become `[:EXIT pid reason]` messages. |
| `:async` | boolean | Return a virtual future that resolves to the spawn result. |
| `:ex->reason-fn` | function | Convert uncaught exceptions to process exit reasons. |

Return value:

- pid when `:monitor` is false
- `[pid ref]` when `:monitor` is true
- virtual future of either result when `:async` is true

Use `:link` and `:monitor` from inside a process so there is a current parent or
watcher pid.

### `spawn-opt!`

```clojure
(proc/spawn-opt! {:trap-exit true}
  (body))
```

Macro version of `spawn-opt`.

### `spawn-link` and `spawn-link!`

```clojure
(proc/spawn-link f & args)

(proc/spawn-link!
  (body))
```

Starts a child linked to the current process. Linked process exits propagate to
the other side unless the receiver traps exits or the reason is normal.

The native API does not expose general-purpose public `link` and `unlink`
functions. Create links at spawn time with `spawn-link` or `spawn-opt`.

### `spawn-trap` and `spawn-trap!`

```clojure
(proc/spawn-trap f & args)

(proc/spawn-trap!
  (body))
```

Starts a child with `:trap-exit true`. This sets the child process flag; it does
not automatically link the child to the caller.

### `spawn-monitor` and `spawn-monitor!`

```clojure
(let [[pid ref] (proc/spawn-monitor f & args)]
  ...)

(let [[pid ref] (proc/spawn-monitor!
                  (body))]
  ...)
```

Starts a child and monitors it from the current process.

Returns `[pid ref]`. The watcher receives:

```clojure
[:DOWN ref :process target reason]
```

### `spawn-async!`

```clojure
(let [f (proc/spawn-async!
          (body))]
  @f)
```

Starts the process asynchronously and returns a virtual future that resolves to
the pid.

### `exit`

```clojure
(proc/exit reason)
(proc/exit pid reason)
(proc/exit :registered-name reason)
```

One-argument form exits the current process with `reason`.

Two-argument form sends an exit signal to another process. It requires process
context because the signal includes the sender pid and message context.

Exit signal behavior:

| Receiver state | Reason | Result |
| --- | --- | --- |
| Not trapping exits | `:normal` | Ignored. |
| Not trapping exits | `:kill` | Receiver exits with `:killed`. |
| Not trapping exits | Any other reason | Receiver exits with that reason. |
| Trapping exits | `:normal` | Receiver gets `[:EXIT from-pid :normal]`. |
| Trapping exits | `:kill` | Receiver exits with `:killed`. |
| Trapping exits | Any other reason | Receiver gets `[:EXIT from-pid reason]`. |

### `monitor`

```clojure
(def ref (proc/monitor pid))
(def ref (proc/monitor :registered-name))
```

Creates a one-shot monitor from the current process to a target pid or name.

Returns a `TRef`. When the target exits, the watcher receives:

```clojure
[:DOWN ref :process target reason]
```

If the target cannot be resolved, the watcher receives an immediate `:DOWN`
message with reason `:noproc`.

### `demonitor`

```clojure
(proc/demonitor ref)
```

Removes a monitor. Returns `true`. This does not flush any `:DOWN` message that
may already be in the mailbox.

### `register`

```clojure
(proc/register :name)
(proc/register :name pid)
```

Registers the current process or supplied pid under `:name`.

Throws if the name is already registered or if the pid already has a name.

### `alive?`

```clojure
(proc/alive? pid)
(proc/alive?)
```

Returns `true` if a process is alive. The zero-argument form checks the current
process and therefore requires process context.

### `processes`

```clojure
(proc/processes)
```

Returns active process pids.

### `process-info`

```clojure
(proc/process-info pid)
```

Returns a map for a live process, or `nil` for an unknown or fully exited pid.

Map keys include:

| Key | Meaning |
| --- | --- |
| `:pid` | Process pid. |
| `:links` | Linked pids. |
| `:monitors` | Monitors owned by this process. |
| `:monitored-by` | Monitors watching this process. |
| `:registered-name` | Registered name or `nil`. |
| `:status` | `:running` or `:exiting`. |
| `:message-queue-len` | Number of queued messages. |
| `:flags` | Process flags, including `:trap-exit`. |

### Function-based `receive!`

```clojure
(proc/receive!)
(proc/receive! timeout-ms timeout-val)
```

Receives the next FIFO message from the current process mailbox. Prefer
`loom-otp.process.match/receive!` for application code that wants pattern
matching.

### Function-based `selective-receive!`

```clojure
(proc/selective-receive! pred)
(proc/selective-receive! pred timeout-ms)
```

Receives the first queued message where `(pred message)` is truthy. Returns
`:timeout` if no matching message arrives within `timeout-ms`.

## `loom-otp.process.match`

Pattern-matching receive macros backed by `clojure.core.match`.

Require it as:

```clojure
(require '[loom-otp.process.match :as match])
```

### `receive!`

```clojure
(match/receive!
  pattern-1 body-1
  pattern-2 body-2
  (after timeout-ms timeout-body))
```

Receives the next FIFO message and matches it with `core.match` clauses.

Example:

```clojure
(match/receive!
  [:add a b] (+ a b)
  [:ping from] (proc/send from :pong)
  (after 1000 :timeout))
```

Without an `after` clause, the receive waits indefinitely for the next message.
If the next message does not match any clause, it throws.

### `selective-receive!`

```clojure
(match/selective-receive!
  pattern-1 body-1
  pattern-2 body-2
  (after timeout-ms timeout-body))
```

Scans the mailbox for the first message matching any supplied pattern.
Non-matching messages remain queued in their original order.

Use patterns to discriminate messages. If a pattern matches, the message is
removed before the body runs.

## `loom-otp.registry`

Process name registry.

Require it as:

```clojure
(require '[loom-otp.registry :as reg])
```

| Function | Purpose |
| --- | --- |
| `(reg/whereis name)` | Return pid registered under `name`, or `nil`. |
| `(reg/registered)` | Return a set of registered names. |
| `(reg/get-registered-name pid)` | Return pid's registered name, or `nil`. |
| `(reg/register! name pid)` | Register `pid` under `name`; throws on conflict. |
| `(reg/unregister! pid)` | Remove pid's registration. |
| `(reg/resolve-pid dest)` | Resolve pid or registered name to pid. |

## `loom-otp.gen-server`

Generic server behavior.

Require it as:

```clojure
(require '[loom-otp.gen-server :as gs])
```

### `IGenServer`

Implement the protocol to define server behavior:

```clojure
(gs/init this args)
(gs/handle-call this request from state)
(gs/handle-cast this request state)
(gs/handle-info this message state)
(gs/terminate this reason state)
```

Callback return values:

| Callback | Return values |
| --- | --- |
| `init` | `[:ok state]`, `[:ok state timeout]`, `[:stop reason]` |
| `handle-call` | `[:reply response new-state]`, `[:reply response new-state timeout]`, `[:noreply new-state]`, `[:noreply new-state timeout]`, `[:stop reason response new-state]`, `[:stop reason new-state]` |
| `handle-cast` | `[:noreply new-state]`, `[:noreply new-state timeout]`, `[:stop reason new-state]` |
| `handle-info` | Same as `handle-cast` |
| `terminate` | Return value ignored |

`timeout` controls how long the server waits for the next message before
calling `handle-info` with `:timeout`.

### `ns->gen-server`

```clojure
(gs/ns->gen-server 'my.app.counter)
(gs/ns->gen-server (find-ns 'my.app.counter))
```

Builds an `IGenServer` implementation from namespace functions named `init`,
`handle-call`, `handle-cast`, `handle-info`, and `terminate`. Missing functions
use default implementations.

### `start`

```clojure
(gs/start impl)
(gs/start impl args)
(gs/start impl args {:name :counter :timeout 5000})
```

Starts a server. Returns `{:ok pid}` or `{:error reason}`.

Options:

| Option | Default | Purpose |
| --- | --- | --- |
| `:name` | `nil` | Registered name for the server. |
| `:timeout` | `5000` | Startup timeout in milliseconds. |

### `start-link`

```clojure
(gs/start-link impl args opts)
```

Starts a server linked to the current process. Requires process context. Returns
`{:ok pid}` or `{:error reason}`.

### `call`

```clojure
(gs/call server request)
(gs/call server request timeout-ms)
```

Sends a synchronous request and returns the reply. Requires process context.
Throws `ExceptionInfo` on timeout or server death.

### `cast`

```clojure
(gs/cast server request)
```

Sends an asynchronous request. Returns `:ok`.

### `reply`

```clojure
(gs/reply from response)
```

Sends an explicit reply when `handle-call` returns `[:noreply new-state]`.
`from` is the value passed to `handle-call`.

### `stop`

```clojure
(gs/stop server)
(gs/stop server reason)
(gs/stop server reason timeout-ms)
```

Sends an exit signal, waits for the server to go down, and returns `:ok`.
Requires process context. If the timeout expires, the server is killed and `:ok`
is still returned.

## `loom-otp.supervisor`

Supervisor behavior for restartable child processes.

Require it as:

```clojure
(require '[loom-otp.supervisor :as sup])
```

See [Supervision guide](SUPERVISION.md) for detailed examples.

### Supervisor flags

```clojure
{:strategy  :one-for-one    ; default
 :intensity 1               ; default
 :period    5000}           ; default, milliseconds
```

Strategies:

- `:one-for-one` — restart only the failed child.
- `:one-for-all` — stop and restart all children.
- `:rest-for-one` — stop and restart the failed child and children started
  after it.

### Child specs

```clojure
{:id       :worker
 :start    [worker-fn arg1 arg2]
 :restart  :permanent
 :shutdown 5000
 :type     :worker}
```

Defaults:

| Key | Default | Values |
| --- | --- | --- |
| `:restart` | `:permanent` | `:permanent`, `:transient`, `:temporary` |
| `:shutdown` | `5000` | positive milliseconds, `:brutal-kill`, `:infinity` |
| `:type` | `:worker` | `:worker`, `:supervisor` |

### `check-child-specs`

```clojure
(sup/check-child-specs child-specs)
```

Returns `{:ok normalized-specs}` or `{:error errors}`.

### `start-link`

```clojure
(sup/start-link sup-flags child-specs)
(sup/start-link sup-flags child-specs {:name :sup :timeout 5000})
```

Starts a supervisor linked to the current process. Use it from process context
when the supervisor should be linked to a parent. Returns `{:ok pid}` or
`{:error reason}`.

### Dynamic child operations

These functions require process context because they send a request and receive
a reply.

| Function | Return value |
| --- | --- |
| `(sup/which-children sup)` | Vector of child maps with `:id`, `:pid`, `:type`, `:restart`. |
| `(sup/start-child sup child-spec)` | `{:ok pid}` or `{:error reason}`. |
| `(sup/terminate-child sup child-id)` | `:ok` or `{:error reason}`. |
| `(sup/delete-child sup child-id)` | `:ok` or `{:error reason}`. |
| `(sup/count-children sup)` | Map with `:specs`, `:active`, `:supervisors`, and `:workers`. |

## `loom-otp.timer`

Timer functions for delayed and periodic work.

Require it as:

```clojure
(require '[loom-otp.timer :as timer])
```

### `start-timer`

```clojure
(timer/start-timer opts f & args)
```

Starts a timer process that applies `(f arg1 arg2 ...)` after a delay and,
optionally, at an interval.

Options:

| Option | Default | Purpose |
| --- | --- | --- |
| `:after-ms` | `0` | Delay before first fire. |
| `:every-ms` | `nil` | Interval between repeated fires. `nil` means one-shot. |
| `:link` | `true` for intervals, `false` for one-shot timers | Link timer to caller. |
| `:catch-all` | `false` | Catch and ignore exceptions from `f`. |

Returns a `Timer` record with `:pid`, `:start-time`, `:last-fire-time`, and
`:opts`.

### `cancel`

```clojure
(timer/cancel timer)
```

Sends a cancel message to the timer process. Returns `true` if the timer process
existed and `false` otherwise.

### `tc`

```clojure
(timer/tc 5 :seconds)
;; => 5000
```

Converts supported units to milliseconds.

Supported units:

- `:ms`, `:milliseconds`
- `:s`, `:sec`, `:seconds`
- `:min`, `:minutes`
- `:hr`, `:hours`

## `loom-otp.vfuture`

Virtual-thread-backed future values.

Require it as:

```clojure
(require '[loom-otp.vfuture :as vf])
```

### `vfuture`

```clojure
(let [f (vf/vfuture (expensive-work))]
  @f)
```

Runs the body on a virtual thread and returns a derefable value.

Behavior:

- `@f` blocks until completion.
- `(deref f timeout-ms timeout-val)` returns `timeout-val` on timeout.
- Exceptions thrown in the body are rethrown on deref.
- Dynamic bindings from the caller are preserved.

## `loom-otp.trace`

Global trace handler for debugging process lifecycle and message passing.

Require it as:

```clojure
(require '[loom-otp.trace :as trace])
```

### `trace`

```clojure
(trace/trace
  (fn [event]
    (println (:event event) (:timestamp event))))
```

Sets a global trace handler. The handler receives event maps. Events emitted by
the current implementation include process spawn, message send, exit signals,
and process exit.

### `untrace`

```clojure
(trace/untrace)
```

Removes the trace handler.

### `trace-event!`

```clojure
(trace/trace-event! :custom {:k :v})
```

Emits a trace event if a handler is installed. Intended primarily for internal
or advanced instrumentation.

## `loom-otp.types`

Type helpers.

Require it as:

```clojure
(require '[loom-otp.types :as types])
```

| Function | Purpose |
| --- | --- |
| `(types/pid? x)` | True when `x` is a pid (`Thread`). |
| `(types/pid->str pid)` | Debug string for a pid. |
| `(types/ref? x)` | True when `x` is a `TRef`. |
| `(types/ref->id ref)` | Internal numeric id for a `TRef`, or `nil`. |
| `(types/->ref id)` | Construct a `TRef`. Mostly useful for tests and internals. |
| `(types/async? x)` | True when `x` is a `loom-otp.types.Async`. |
| `(types/->async promise value)` | Construct an `Async` wrapper. Mostly internal. |
| `(types/async-promise async)` | Get wrapped promise, if any. |
| `(types/async-value async)` | Get wrapped value, if any. |

The compatibility layer defines its own `Async` type in
`loom-otp.otplike.process`; do not rely on `loom-otp.types.Async` for otplike
compatibility behavior.

## `loom-otp.otplike.*`

Compatibility namespaces for projects migrating from `otplike`:

- `loom-otp.otplike.process`
- `loom-otp.otplike.gen-server`
- `loom-otp.otplike.supervisor`
- `loom-otp.otplike.timer`
- `loom-otp.otplike.trace`

See [OTPLike compatibility](OTPLIKE_COMPATIBILITY.md) for migration notes and
differences from the native API.
