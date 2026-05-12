# OTPLike compatibility

`loom-otp` includes compatibility namespaces for projects that use the
[`otplike`](https://github.com/suprematic/otplike) API but want to run on JVM
virtual threads.

Compatibility namespaces:

- `loom-otp.otplike.process`
- `loom-otp.otplike.gen-server`
- `loom-otp.otplike.supervisor`
- `loom-otp.otplike.timer`
- `loom-otp.otplike.trace`

Use this guide when migrating existing otplike code or when deciding whether to
use the native `loom-otp.*` API instead.

## Quick migration map

| otplike-style namespace | loom-otp compatibility namespace | Native alternative |
| --- | --- | --- |
| `otplike.process` | `loom-otp.otplike.process` | `loom-otp.process` and `loom-otp.process.match` |
| `otplike.gen-server` | `loom-otp.otplike.gen-server` | `loom-otp.gen-server` |
| `otplike.supervisor` | `loom-otp.otplike.supervisor` | `loom-otp.supervisor` |
| `otplike.timer` | `loom-otp.otplike.timer` | `loom-otp.timer` |
| `otplike.trace` | `loom-otp.otplike.trace` | `loom-otp.trace` |

Example require changes:

```clojure
;; Before
(require '[otplike.process :as process]
         '[otplike.gen-server :as gen-server])

;; After
(require '[loom-otp.otplike.process :as process]
         '[loom-otp.otplike.gen-server :as gen-server])
```

## Start the loom-otp system

The compatibility layer still uses `loom-otp` process state. Start it before
using compatibility APIs:

```clojure
(require '[loom-otp.core :as otp])

(otp/start!)
;; ... run processes ...
(otp/stop!)
```

## Major semantic differences from otplike

### Virtual threads replace core.async go blocks

`loom-otp` processes are Java virtual threads. Blocking operations block the
virtual thread directly.

Practical effects:

- There is no distinction between parking and blocking receive in the
  compatibility layer.
- `receive!!` delegates to the same implementation as `receive!`.
- Blocking inside a process is expected and cheap relative to platform threads.

### `async` executes immediately and synchronously

In original otplike, `async` used a `core.async` go block. In
`loom-otp.otplike.process`, `async` executes its body immediately on the calling
thread and captures the result or exception.

```clojure
(process/async
  (println "runs now"))
```

This preserves the single-process-thread invariant and lets async code access
the current process mailbox.

Consequences:

- Do not rely on `async` to introduce concurrency.
- Use `process/spawn` or the native `loom-otp.vfuture/vfuture` when you need
  concurrent execution.
- `realized?` is true for compatibility `Async` values because the body has
  already run.
- Awaiting an async created inside one process from a different process throws.

Related compatibility test notes are tracked in
[OTPLIKE_COMPAT_TEST_GAPS.md](OTPLIKE_COMPAT_TEST_GAPS.md).

### Return conventions differ between native and compatibility APIs

Compatibility APIs preserve otplike-style tuple returns.

| Operation | Compatibility return | Native return |
| --- | --- | --- |
| gen-server start success | `[:ok pid]` | `{:ok pid}` |
| gen-server start failure | `[:error reason]` | `{:error reason}` |
| supervisor start success | `[:ok pid]` | `{:ok pid}` |
| supervisor start failure | `[:error reason]` | `{:error reason}` |

The compatibility `gen-server/start` and `supervisor/start-link` functions return
compatibility async values. Use bang variants such as `start!`, `start-link!`,
and `call!` when you want the awaited result.

## Process compatibility API

Require it as:

```clojure
(require '[loom-otp.otplike.process :as process])
```

### Process identity and lookup

| Function | Purpose |
| --- | --- |
| `pid?` | True if value is a pid. |
| `ref?` | True if value is a monitor ref. |
| `self` | Current pid; requires process context. |
| `whereis` | Resolve registered name to pid. |
| `registered` | Registered names. |
| `resolve-pid` | Resolve pid or registered name. |
| `pid->str` | Debug string. |

Pids are Java `Thread` objects.

### Sending messages

```clojure
(process/! pid message)
(process/! :registered-name message)
```

Returns `true` when sent and `false` when the destination cannot be resolved.
Passing `nil` as the destination throws.

### Spawning processes

```clojure
(process/spawn proc-fn)
(process/spawn proc-fn [arg1 arg2])
(process/spawn-link proc-fn)
(process/spawn-opt proc-fn args opts)
```

Compatibility spawn options:

```clojure
{:flags {:trap-exit true}
 :link true
 :register :name}
```

Process function helpers:

```clojure
(process/proc-fn [arg] ...)

(process/proc-defn worker [arg]
  ...)

(process/proc-defn- private-worker [arg]
  ...)
```

In `loom-otp`, these create ordinary Clojure functions suitable for virtual
thread processes.

### Links and exits

The compatibility layer exposes `link` and `unlink`:

```clojure
(process/link pid)
(process/unlink pid)
```

The native API creates links only at spawn time.

Exit forms:

```clojure
(process/exit reason)
(process/exit pid reason)
```

The two-argument form returns `true` if the target was alive before signaling and
`false` if it was already terminated. Passing a non-pid target throws.

Set trap-exit with:

```clojure
(process/flag :trap-exit true)
```

The function returns the previous flag value.

### Monitors

```clojure
(def ref (process/monitor pid-or-name))

(process/demonitor ref)
(process/demonitor ref {:flush true})
```

Monitor messages have this shape:

```clojure
[:DOWN ref :process target reason]
```

With `{:flush true}`, `demonitor` attempts to remove a pending `:DOWN` message
for the same ref.

### Async and await

```clojure
(process/async body...)
(process/await! async-or-value)
(process/await!! async)
(process/await?! async-or-value)
(process/with-async [x async-expr] body...)
(process/map-async f async-val)
(process/async-value value)
```

Behavior summary:

| Function | Behavior |
| --- | --- |
| `async` | Runs body immediately, captures value or exception. |
| `await!` | Macro; returns resolved value, or value unchanged if not async. |
| `await!!` | Function; requires async and throws for non-async values. |
| `await?!` | Macro; awaits only if the expression returns async. |
| `with-async` | Transforms an async value. |
| `map-async` | Appends a transformation to an async value. |
| `async-value` | Wraps an already available value. |

### Receive macros

```clojure
(process/receive!
  pattern body
  (after timeout-ms timeout-body))

(process/receive!!
  pattern body
  (after timeout-ms timeout-body))

(process/selective-receive!
  pattern body
  (after timeout-ms timeout-body))
```

`receive!!` is the same as `receive!` in the compatibility layer.

### Exception helpers

```clojure
(process/ex-catch expr)
(process/ex->reason throwable)
```

`ex-catch` returns the expression result or `[:EXIT reason]` if an exception is
caught.

### Process info

```clojure
(process/process-info pid)
(process/process-info pid :status)
(process/process-info pid [:status :message-queue-len])
```

Compatibility `process-info` returns otplike-style `[key value]` pairs rather
than the native process-info map.

## Gen-server compatibility API

Require it as:

```clojure
(require '[loom-otp.otplike.gen-server :as gen-server])
```

### Implement a server

You can provide a map of callbacks:

```clojure
(def counter
  {:init (fn []
           [:ok 0])

   :handle-call (fn [request _from state]
                  (case request
                    :get [:reply state state]
                    :inc [:reply state (inc state)]
                    [:reply :unknown state]))

   :handle-cast (fn [request state]
                  (case request
                    :inc [:noreply (inc state)]
                    [:noreply state]))

   :handle-info (fn [_message state]
                  [:noreply state])

   :terminate (fn [_reason _state]
                nil)})
```

Or implement `IGenServer`.

### Start and call

```clojure
(process/spawn
  (process/proc-fn []
    (let [[:ok pid] (gen-server/start! counter)]
      (gen-server/call! pid :inc)
      (gen-server/call! pid :get))))
```

Non-bang forms return compatibility async values:

```clojure
(gen-server/start counter)
(gen-server/start-link counter)
(gen-server/call pid :get)
```

Bang forms await the result:

```clojure
(gen-server/start! counter)
(gen-server/start-link! counter)
(gen-server/call! pid :get)
```

### Namespace-based servers

The compatibility layer provides namespace start macros:

```clojure
(gen-server/start-ns)
(gen-server/start-ns!)
(gen-server/start-link-ns)
(gen-server/start-link-ns!)
```

The current namespace should define callback functions such as `init`,
`handle-call`, `handle-cast`, `handle-info`, and optionally `terminate`.

### Cast and inspect state

```clojure
(gen-server/cast pid :inc)
(gen-server/get! pid)
```

`get!` is intended for debugging.

## Supervisor compatibility API

Require it as:

```clojure
(require '[loom-otp.otplike.supervisor :as supervisor])
```

### Supervisor function

The compatibility supervisor expects a supervisor function that returns:

```clojure
[:ok [supervisor-flags child-specs]]
```

Example:

```clojure
(defn compat-worker []
  [:ok (process/spawn
         (process/proc-fn []
           (process/receive!
             :stop :ok)))])

(defn my-supervisor []
  [:ok [{:strategy :one-for-one
         :intensity 5
         :period 10000}
        [{:id :worker
          :start [compat-worker]
          :restart :permanent}]]])
```

### Start and manage children

```clojure
(supervisor/start-link my-supervisor)
(supervisor/start-link! my-supervisor)

(supervisor/start-child sup child-spec)
(supervisor/start-child! sup child-spec)

(supervisor/restart-child sup child-id)
(supervisor/restart-child! sup child-id)

(supervisor/terminate-child sup child-id)
(supervisor/terminate-child! sup child-id)

(supervisor/delete-child sup child-id)
(supervisor/delete-child! sup child-id)
```

Non-bang forms return compatibility async values. Bang forms return awaited
results such as `[:ok pid]`, `:ok`, or `[:error reason]`.

## Timer compatibility API

Require it as:

```clojure
(require '[loom-otp.otplike.timer :as timer])
```

Timer functions return a compatibility `TimerRef` wrapper.

| Function | Purpose |
| --- | --- |
| `(timer/apply-after msecs f args)` | Apply a function once after a delay. |
| `(timer/send-after msecs message)` | Send to the current process after a delay. |
| `(timer/send-after msecs pid message)` | Send to a target after a delay. |
| `(timer/exit-after msecs reason)` | Exit current process after a delay. |
| `(timer/exit-after msecs pid reason)` | Send exit signal after a delay. |
| `(timer/kill-after msecs)` | Kill current process after a delay. |
| `(timer/kill-after msecs pid)` | Kill target after a delay. |
| `(timer/apply-interval msecs f args)` | Repeatedly apply a function. |
| `(timer/send-interval msecs message)` | Repeatedly send to current process. |
| `(timer/send-interval msecs pid message)` | Repeatedly send to target. |
| `(timer/cancel timer-ref)` | Cancel; safe to call repeatedly. |

Notes:

- Timer callback exceptions are caught and ignored in the compatibility API.
- Interval timers require process context and are linked so they stop with the
  owning process.
- `cancel` returns `nil`, matching compatibility expectations.

## Trace compatibility

`loom-otp.otplike.trace` provides helper entry points:

```clojure
(trace/pid pid handler)
(trace/reg-name :name handler)
(trace/kind :kind handler)
(trace/crashed handler)
```

The native trace handler in `loom-otp.trace` receives raw event maps. If your
application depends on exact otplike trace filtering behavior, add migration
tests around the trace events you consume.

## Testing and parity notes

The compatibility test-gap document records the current parity status against
the original otplike tests:

- [OTPLIKE_COMPAT_TEST_GAPS.md](OTPLIKE_COMPAT_TEST_GAPS.md)

Notable recorded differences:

- `async` does not park or run concurrently.
- `receive!!` has the same blocking behavior as `receive!` on virtual threads.
- Tests use promises instead of core.async channels.
- Some original otplike tests were empty stubs and remain intentionally empty
  for parity tracking.

## Migration checklist

1. Replace otplike namespace requires with `loom-otp.otplike.*` namespaces.
2. Add `loom-otp.core/start!` to application startup and `stop!` to shutdown.
3. Search for `async` usages that depend on concurrent execution; replace those
   with `spawn`, native `vfuture`, or another concurrency primitive.
4. Keep bang/non-bang expectations straight: compatibility non-bang
   `gen-server` and supervisor functions often return async values.
5. Run the existing otplike-oriented test suite.
6. Add tests for process linking, monitor delivery, and timeout behavior that
   are important to your application.
