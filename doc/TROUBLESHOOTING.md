# Troubleshooting loom-otp

Use this guide to diagnose common problems with processes, message delivery,
links, monitors, `gen_server`, supervisors, and timers.

Each entry follows this pattern: symptom → likely cause → resolution →
verification.

## `self called outside of process`

### Symptom

An operation throws an `ExceptionInfo` similar to:

```text
self called outside of process
```

or:

```text
not in a process
```

### Likely cause

The code called a process-context API from the REPL main thread, a test thread,
or another non-`loom-otp` thread.

APIs that require process context include:

- `proc/self`
- `receive!` and `selective-receive!`
- `proc/monitor`
- `proc/exit` when signaling another process
- `gen-server/call`
- supervisor dynamic operations such as `which-children`

### Resolution

Run the operation inside a `proc/spawn!` body and deliver the result to a
`promise` if the caller is outside the process system.

```clojure
(def result (promise))

(proc/spawn!
  (deliver result (proc/self)))

@result
```

### Verification

The operation returns a pid or expected value instead of throwing.

## Process operations fail before startup

### Symptom

Spawning, sending, registering, or receiving fails unexpectedly near application
startup.

### Likely cause

The `mount.lite` state for `loom-otp` has not been started.

### Resolution

Call `loom-otp.core/start!` before using processes:

```clojure
(require '[loom-otp.core :as otp])

(otp/start!)
```

Stop the system during shutdown:

```clojure
(otp/stop!)
```

### Verification

`(otp/status)` reports started states, and `proc/spawn!` returns a pid.

## `send` returns `false`

### Symptom

```clojure
(proc/send dest msg)
;; => false
```

### Likely cause

The destination is not a live pid and does not resolve to a registered process
name.

### Resolution

Check the destination:

```clojure
(proc/alive? pid)
(reg/whereis :name)
(reg/registered)
```

If the process should be registered, register it after it starts:

```clojure
(proc/spawn!
  (proc/register :server)
  ...)
```

### Verification

`proc/send` returns `true`, and the receiver observes the message.

## A receive blocks forever

### Symptom

A process waits indefinitely in `receive!` or `selective-receive!`.

### Likely cause

No matching message is arriving, the message is being sent to the wrong pid or
name, or the receive pattern does not match the message shape.

### Resolution

Add an `after` clause while debugging:

```clojure
(match/receive!
  [:expected value] value
  (after 1000 :timeout))
```

Inspect the process mailbox length:

```clojure
(proc/process-info pid)
;; Check :message-queue-len
```

Confirm the sender returns `true` from `proc/send`.

### Verification

The receive returns a value or `:timeout`, and the mailbox length behaves as
expected.

## `receive!` throws `no matching clause for message`

### Symptom

The process throws after a message arrives.

### Likely cause

`receive!` reads the next FIFO message, but that message does not match any
clause.

### Resolution

Either add a clause for the message shape or use `selective-receive!` when the
target message may be behind unrelated messages.

```clojure
(match/selective-receive!
  [:target value] value
  (after 1000 :timeout))
```

### Verification

The process handles the expected message and leaves non-matching messages queued
when using selective receive.

## Selective receive removes the wrong message

### Symptom

A selective receive consumes a message even though the body returns `nil` or
decides it is not the intended message.

### Likely cause

`selective-receive!` selects by pattern. Once the pattern matches, the message is
removed before the body runs.

### Resolution

Put all selection criteria in the pattern. Do not use the body as a post-match
filter.

For complex matching, use the function-based API with a predicate:

```clojure
(proc/selective-receive!
  (fn [msg]
    (and (vector? msg)
         (= :reply (first msg))
         (= expected-ref (second msg))))
  1000)
```

### Verification

Only messages satisfying the predicate or exact pattern are removed.

## A linked parent exits when a child fails

### Symptom

A parent process dies after a linked child exits with an abnormal reason.

### Likely cause

Links propagate abnormal exits. The parent is not trapping exits.

### Resolution

Start the parent with `:trap-exit true` if it should handle linked child exits:

```clojure
(proc/spawn-opt!
  {:trap-exit true}
  (proc/spawn-link!
    (exit/exit :failed))
  (match/receive!
    [:EXIT child reason] (handle-exit child reason)))
```

Use monitors instead of links when you need observation without fault
propagation.

### Verification

The parent receives `[:EXIT child reason]` and remains alive.

## A trapped `:kill` still terminates the process

### Symptom

A process with `:trap-exit true` exits after receiving `:kill`.

### Likely cause

`:kill` is untrappable by design. The target exits with reason `:killed`.

### Resolution

Use another reason if the receiver should be able to trap it:

```clojure
(proc/exit pid :shutdown)
```

### Verification

With a non-`:kill` reason, the trapping process receives an `[:EXIT ...]`
message.

## A monitor does not deliver `:DOWN`

### Symptom

The watcher never receives a `[:DOWN ref :process target reason]` message.

### Likely cause

The monitor was created outside process context, the watcher exited before the
target, or the code is waiting for the wrong ref.

### Resolution

Create monitors from inside a process and match the exact ref:

```clojure
(proc/spawn!
  (let [ref (proc/monitor target)]
    (match/receive!
      [:DOWN received-ref :process _ reason]
      (when (= received-ref ref)
        (handle-down reason)))))
```

### Verification

The watcher process is alive, `process-info` shows monitor data, and the `:DOWN`
message arrives after target exit.

## `gen-server/call` times out

### Symptom

`gen-server/call` throws with timeout data.

### Likely cause

The server is not alive, the server callback did not reply, the request is stuck
behind other messages, or the call is made from outside process context.

### Resolution

Check server liveness and queue length:

```clojure
(proc/alive? server)
(proc/process-info server)
```

Make sure `handle-call` returns one of the supported reply forms:

```clojure
[:reply response new-state]
[:reply response new-state timeout]
[:noreply new-state]
[:noreply new-state timeout]
[:stop reason response new-state]
[:stop reason new-state]
```

If returning `:noreply`, call `gen-server/reply` later with the `from` value.

### Verification

The server receives the request and the caller receives a response before the
timeout.

## `gen-server/start` returns `{:error :timeout}`

### Symptom

Starting a server times out.

### Likely cause

`init` did not return before the startup timeout, blocked, or returned an invalid
shape.

### Resolution

Make `init` quick and return one of:

```clojure
[:ok state]
[:ok state timeout]
[:stop reason]
```

Increase the startup timeout only when slow initialization is expected:

```clojure
(gs/start impl args {:timeout 10000})
```

### Verification

`gs/start` returns `{:ok pid}` and `(proc/alive? pid)` is true.

## Supervisor exits with `:max-intensity`

### Symptom

A supervisor stops after repeated child failures.

### Likely cause

The number of restarts exceeded `:intensity` within `:period`.

### Resolution

Fix the child crash first. Then tune restart flags if the failure rate is
expected:

```clojure
{:strategy :one-for-one
 :intensity 5
 :period 10000}
```

### Verification

The child stays alive or restarts within the configured intensity window.

## A child restarts even though it completed normally

### Symptom

A child that returns normally is started again.

### Likely cause

The child has `:restart :permanent`.

### Resolution

Use `:transient` for tasks that should restart only after abnormal exits, or
`:temporary` for tasks that should never restart.

```clojure
{:id :job
 :start [run-job]
 :restart :temporary}
```

### Verification

The child exits normally and does not restart.

## Timer does not stop

### Symptom

An interval timer continues after the process that created it exits.

### Likely cause

The timer was created with `:link false`, or it was created outside process
context as an unlinked timer.

### Resolution

Use the default linked interval behavior or set `:link true` explicitly:

```clojure
(timer/start-timer {:after-ms 1000
                    :every-ms 1000
                    :link true}
                   f)
```

Cancel timers you no longer need:

```clojure
(timer/cancel timer)
```

### Verification

After the owner exits or `cancel` is called, no further interval callbacks run.

## Timer callback crashes the timer

### Symptom

A timer fires once and then stops, or `proc/alive?` returns false for the timer
pid.

### Likely cause

The timer callback threw and `:catch-all` was false.

### Resolution

Use `:catch-all true` when callback exceptions should be ignored:

```clojure
(timer/start-timer {:after-ms 1000
                    :every-ms 1000
                    :catch-all true}
                   f)
```

Or catch exceptions inside the callback and send errors to another process for
handling.

### Verification

The interval continues after a callback error.

## Registered name is already in use

### Symptom

Registering or spawning with a name throws a registration error.

### Likely cause

Another live process already owns the name, or the pid already has a registered
name.

### Resolution

Inspect the registry:

```clojure
(reg/registered)
(reg/whereis :name)
```

Use a unique name, wait for the old process to exit, or explicitly stop the old
process.

### Verification

`reg/whereis` returns the expected pid for the name.

## Compatibility `async` does not run concurrently

### Symptom

Code ported from otplike expects `process/async` to run later or concurrently,
but it runs immediately.

### Likely cause

`loom-otp.otplike.process/async` executes synchronously on the calling thread.

This is intentional for virtual-thread process semantics.

### Resolution

Use `process/spawn` for process concurrency or `loom-otp.vfuture/vfuture` for a
virtual-thread future.

### Verification

Code that needs concurrency no longer depends on compatibility `async` for
scheduling.

## Java version errors

### Symptom

The project fails to compile or virtual-thread APIs are missing.

### Likely cause

The application is running on a JDK older than 21.

### Resolution

Install and select JDK 21 or newer. Verify:

```bash
java -version
```

### Verification

The version output reports Java 21 or newer, and code using
`Thread/startVirtualThread` runs.
