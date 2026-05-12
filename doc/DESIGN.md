# loom-otp design

`loom-otp` implements Erlang/OTP-style actor concurrency for Clojure on JVM
virtual threads. It provides process spawning, message passing, links, monitors,
registered names, `gen_server`, supervisors, timers, tracing, and an otplike
compatibility layer.

This document describes the current implementation architecture. For user-facing
usage, start with [Getting started](GETTING_STARTED.md) and [API reference](API.md).

## Goals

- Make blocking actor-style code natural on Java 21+ virtual threads.
- Keep the main process API small: spawn, send, receive, link at spawn time,
  monitor, and inspect.
- Preserve important OTP semantics: links, monitors, trap exits, restart
  strategies, and server callbacks.
- Provide an otplike compatibility layer for migration while keeping native
  `loom-otp` APIs straightforward.

## Non-goals

- Full Erlang VM compatibility.
- Distribution across JVM nodes.
- A core.async implementation. The compatibility layer avoids requiring callers
  to use go blocks.
- Public access to every low-level process table and mailbox operation.

## Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│ Application layer                                                │
│                                                                 │
│  loom-otp.supervisor       Supervision trees and restart logic   │
├─────────────────────────────────────────────────────────────────┤
│ Behavior layer                                                   │
│                                                                 │
│  loom-otp.gen-server       Stateful request/cast/info servers    │
│  loom-otp.timer            Process-backed timers                 │
│  loom-otp.vfuture          Virtual-thread futures                │
├─────────────────────────────────────────────────────────────────┤
│ Public process layer                                             │
│                                                                 │
│  loom-otp.process          Main process API                      │
│  loom-otp.process.match    Pattern receive macros                │
│  loom-otp.registry         Registered names                      │
│  loom-otp.trace            Global trace handler                  │
├─────────────────────────────────────────────────────────────────┤
│ Process implementation layer                                     │
│                                                                 │
│  process.spawn             Virtual-thread lifecycle              │
│  process.core              Current pid, send, exit signals       │
│  process.receive           FIFO and predicate receive functions  │
│  process.link              Bidirectional links                   │
│  process.monitor           Monitors and :DOWN delivery           │
│  process.exit              Exit exception and reason helpers     │
├─────────────────────────────────────────────────────────────────┤
│ Foundation layer                                                 │
│                                                                 │
│  state                     mount.lite-managed global state       │
│  mailbox                   Blocking mailbox implementation       │
│  context-mailbox           [context message] mailbox wrapper     │
│  types                     TRef and helper predicates            │
└─────────────────────────────────────────────────────────────────┘
```

The compatibility layer under `loom-otp.otplike.*` wraps the native layers and
adapts return values, names, and selected semantics to otplike-style APIs.

## Runtime state

Global state is managed by `mount.lite` in `loom-otp.state`.

```clojure
{:processes        ConcurrentHashMap ; Thread -> process map
 :monitors         ConcurrentHashMap ; ref-id -> monitor info
 :registry-forward ConcurrentHashMap ; name -> Thread
 :registry-reverse ConcurrentHashMap ; Thread -> name
 :trace-fn         Atom              ; trace handler or nil
 :ref-counter      AtomicLong        ; monitor/timer ref ids
 :cleanup-thread   Atom}             ; delayed cleanup worker
```

Start state with `loom-otp.core/start!` and stop it with
`loom-otp.core/stop!`.

### Process map

Each process is represented by one Java virtual thread plus metadata:

```clojure
{:mailbox          Atom<Queue<[ctx msg]>>
 :exit-reason      Atom<nil | reason>
 :cleanup-after    Atom<nil | Instant>
 :user-result      Promise
 :message-context  Atom<Map>
 :last-control-ctx Atom<Map>
 :flags            Atom<{:trap-exit boolean}>
 :links            Atom<#{Thread}>
 :ex->reason-fn    Function}
```

Pids are the virtual `Thread` objects themselves. There is no pid wrapper type
in the native API.

Exited processes are marked for delayed cleanup. `state/get-proc` hides fully
exited processes, while lower-level cleanup code can still see raw entries until
the cleanup delay expires.

## Process lifecycle

```text
caller thread
  │
  ├─ spawn/spawn-opt
  │   ├─ capture caller message context
  │   ├─ create process map
  │   ├─ start virtual thread
  │   └─ wait until setup succeeds or fails
  │
virtual process thread
  │
  ├─ add self to process table
  ├─ register name, if requested
  ├─ create link to parent, if requested
  ├─ signal spawn completion
  ├─ run bound user function
  ├─ record normal return, explicit exit, interrupt, or exception
  ├─ notify linked processes and monitors
  ├─ remove owned monitors and links
  ├─ unregister name
  └─ mark process for delayed cleanup
```

`bound-fn*` preserves dynamic bindings from the caller in spawned process
functions.

## Message passing

`loom-otp.process/send` resolves the destination through `loom-otp.registry`.
The destination can be a pid or registered name.

Messages are stored internally as `[ctx msg]` pairs. `ctx` is the sender's
current message context. When a process receives a message, the context is merged
into the receiver's process context. That context is then propagated through
future sends, exits, and monitor notifications.

### FIFO receive

`loom-otp.process.receive/receive!` removes the next mailbox entry in FIFO order
and returns the unwrapped message. The pattern macro
`loom-otp.process.match/receive!` matches that message with `core.match`.

If the next message does not match any macro clause, the macro throws. The
message has already been removed because FIFO receive has consumed it.

### Selective receive

`selective-receive!` scans the mailbox for the first matching message. It removes
that entry and leaves non-matching entries in their original order.

For the macro form, selection is pattern-based. Clause bodies do not participate
in selection.

## Exit reasons

Exit reasons are plain Clojure values.

| Scenario | Exit reason |
| --- | --- |
| User function returns normally | `:normal` |
| User calls `(exit/exit reason)` or `(proc/exit reason)` | `reason` |
| User function throws an uncaught exception | `[:exception throwable]` by default |
| `:ex->reason-fn` spawn option converts an exception | Function return value |
| Link setup fails because target is gone | `:noproc` |
| Process receives untrappable `:kill` | `:killed` |

The normal return value of a process is delivered to the process's internal
`:user-result` promise. It is not part of the native process exit reason.

## Links

Links are bidirectional relationships used for fault propagation.

Native links are created at spawn time:

- `proc/spawn-link`
- `proc/spawn-link!`
- `proc/spawn-opt` with `{:link true}`

The native public API intentionally does not expose general `link` and `unlink`
functions. The otplike compatibility layer exposes them for compatibility.

### Link signal handling

When a process exits, cleanup sends exit signals to linked processes.

| Receiver `:trap-exit` | Reason `:normal` | Reason `:kill` | Any other reason |
| --- | --- | --- | --- |
| `false` | Ignored | Exit with `:killed` | Exit with reason |
| `true` | Receive `[:EXIT from reason]` | Exit with `:killed` | Receive `[:EXIT from reason]` |

## Monitors

Monitors are one-way observations. They do not propagate failure.

```clojure
[:DOWN ref :process target reason]
```

Properties:

- A monitor fires once and is removed.
- Monitoring a missing target sends an immediate `:DOWN` with `:noproc`.
- Multiple monitors to the same target are allowed.
- Monitors are cleaned up when the watcher exits.

## Registry

The registry maintains two maps:

- name -> pid
- pid -> name

This enforces one name per pid and one pid per name. Registrations are removed
during process cleanup.

## `gen_server`

`loom-otp.gen-server` runs a server loop inside a process.

Message types:

- `[::call [ref caller-pid] request]` for synchronous calls
- `[::cast request]` for asynchronous casts
- any other message for `handle-info`
- `:timeout` synthetic info message after a callback timeout

`call` monitors the server, sends a call message, and selectively waits for
either a reply or a `:DOWN`. It throws on timeout or server death.

Callbacks return OTP-style vectors such as `[:reply response new-state]`,
`[:noreply new-state]`, and `[:stop reason new-state]`. See [API reference](API.md)
for the full table.

## Supervisors

`loom-otp.supervisor` starts a supervisor process that traps exits and starts
children with `proc/spawn-link`. Children are stored as ordered entries:

```clojure
{:spec child-spec
 :pid  child-pid}
```

On `[:EXIT child reason]`, the supervisor:

1. finds the child entry
2. checks the child restart policy
3. checks restart intensity
4. applies the configured strategy
5. exits with `:max-intensity` or another reason if restart cannot proceed

Supported strategies:

- `:one-for-one`
- `:one-for-all`
- `:rest-for-one`

See [Supervision guide](SUPERVISION.md) for examples and operational guidance.

## Timers

Timers are processes that sleep using `receive!` with a timeout. A cancel message
prevents the timeout branch from firing.

Timer defaults:

- one-shot timers are unlinked by default
- interval timers are linked by default
- `:catch-all true` catches callback exceptions and lets interval timers
  continue

Intervals schedule the next fire from the previous target fire time, not from the
end of callback execution. This keeps intervals closer to the requested cadence
when callback execution is shorter than the interval.

## Tracing

`loom-otp.trace/trace` installs a global handler. Internal code emits event maps
for operations such as spawn, send, exit signals, and process exit.

Each emitted event receives:

```clojure
{:event event-type
 :timestamp current-time-ms
 ...event-specific-data}
```

Trace handlers should be fast and should not throw. Exceptions from handlers are
caught and ignored.

## Compatibility layer design

The `loom-otp.otplike.*` namespaces adapt native behavior to otplike-style APIs:

- `process/!` wraps native `send`.
- `proc-fn` and `proc-defn` produce ordinary functions.
- `async` executes immediately and captures value or exception.
- `receive!!` maps to the same implementation as `receive!`.
- `gen-server` and supervisor start APIs use tuple returns and async/bang
  conventions.
- Timers return a compatibility `TimerRef` wrapper.

See [OTPLike compatibility](OTPLIKE_COMPATIBILITY.md) for migration details.

## Concurrency and cleanup considerations

- Process tables, monitor tables, and registry maps use `ConcurrentHashMap`.
- Per-process links, flags, mailbox, and context are stored in atoms.
- Spawning waits until registration and link setup complete, so callers do not
  receive a pid before setup succeeds.
- Process cleanup notifies links and monitors before unregistering and marking
  the process for delayed cleanup.
- `mount.lite` stop interrupts all live process threads and clears state.

## Public vs implementation namespaces

Documented application APIs live in:

- `loom-otp.core`
- `loom-otp.process`
- `loom-otp.process.match`
- `loom-otp.registry`
- `loom-otp.gen-server`
- `loom-otp.supervisor`
- `loom-otp.timer`
- `loom-otp.vfuture`
- `loom-otp.trace`
- `loom-otp.types`
- `loom-otp.otplike.*`

Other namespaces are implementation details unless a function is explicitly
referenced by the public API documentation.
