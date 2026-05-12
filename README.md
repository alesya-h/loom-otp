# loom-otp

Erlang/OTP-style actor concurrency for Clojure on JVM virtual threads.

`loom-otp` gives Clojure programs a small actor toolkit: processes, mailboxes,
message passing, links, monitors, registered process names, `gen_server`,
supervisors, timers, tracing, and a compatibility layer for projects using the
[`otplike`](https://github.com/suprematic/otplike) API.

The library is designed for Java 21+ virtual threads. Blocking operations such as
`receive!`, `gen-server/call`, and timer sleeps block a virtual thread instead of
using `core.async` go blocks.

## Requirements

- JDK 21 or newer
- Clojure 1.12 or newer
- Leiningen, Clojure CLI, or another Maven-compatible build tool

## Installation

Leiningen:

```clojure
[org.clojars.arep-engineering/loom-otp "1.0.2"]
```

Clojure CLI / `deps.edn`:

```clojure
org.clojars.arep-engineering/loom-otp {:mvn/version "1.0.2"}
```

When working from a local checkout, run tests with:

```bash
lein test
```

## Quick start

Start the process system before using processes, and stop it during shutdown:

```clojure
(require '[loom-otp.core :as otp]
         '[loom-otp.process :as proc]
         '[loom-otp.process.match :as match])

(otp/start!)

(def result (promise))

(def echo
  (proc/spawn!
    (match/receive!
      [:hello from name]
      (proc/send from [:hello name]))))

(proc/spawn!
  (proc/send echo [:hello (proc/self) "Ada"])
  (match/receive!
    [:hello name] (deliver result name)
    (after 1000 (deliver result :timeout))))

@result
;; => "Ada"

(otp/stop!)
```

Most operations that receive messages or depend on the current process must run
inside a `loom-otp` process. For example, `proc/self`, `receive!`, `monitor`, and
`gen-server/call` require process context.

## Core concepts

| Concept | Purpose |
| --- | --- |
| Process | A virtual thread with a mailbox and process metadata. Pids are `Thread` objects. |
| Mailbox | FIFO queue of messages. Messages can carry context for trace propagation. |
| `receive!` | Blocks for the next message and pattern-matches it. |
| `selective-receive!` | Scans the mailbox for the first message matching a pattern. |
| Link | Bidirectional fault propagation between processes, created through `spawn-link` or `:link true`. |
| Monitor | One-way observation that sends `[:DOWN ref :process target reason]` when a target exits. |
| Registered name | Symbolic name that can be used instead of a pid when sending messages. |
| `gen_server` | Server behavior with synchronous calls, asynchronous casts, and callback state. |
| Supervisor | Process that starts children and restarts them according to a strategy. |
| Timer | Process-backed one-shot or interval callback. |

## Documentation

- [Getting started](doc/GETTING_STARTED.md) — install, run, and build your first processes.
- [API reference](doc/API.md) — public namespaces, functions, return values, and examples.
- [Supervision guide](doc/SUPERVISION.md) — child specs, restart policies, strategies, and operations.
- [OTPLike compatibility](doc/OTPLIKE_COMPATIBILITY.md) — migration notes and API differences.
- [Troubleshooting](doc/TROUBLESHOOTING.md) — common errors, likely causes, and fixes.
- [Design](doc/DESIGN.md) — implementation architecture and process model.
- [Compatibility test gaps](doc/OTPLIKE_COMPAT_TEST_GAPS.md) — parity notes for the otplike test suite.
- [Benchmark comparison](bench/benchmark-comparison.md) — measured comparisons with otplike.

## API map

| Namespace | Use it for |
| --- | --- |
| `loom-otp.core` | Starting, stopping, and resetting the system. |
| `loom-otp.process` | Spawning processes, sending messages, monitors, process info, and registration convenience. |
| `loom-otp.process.match` | Pattern-matching `receive!` and `selective-receive!` macros. |
| `loom-otp.registry` | Name lookup and registration management. |
| `loom-otp.gen-server` | Stateful server behavior. |
| `loom-otp.supervisor` | Supervision trees and child management. |
| `loom-otp.timer` | One-shot and interval timers. |
| `loom-otp.vfuture` | Virtual-thread-backed future values. |
| `loom-otp.trace` | Global process and message trace handler. |
| `loom-otp.otplike.*` | Compatibility layer for otplike-style APIs. |

## Development

Run the default test suite:

```bash
lein test
```

Run all tests, including tests marked with custom selectors:

```bash
lein test :all
```

The project uses `mount.lite` for lifecycle state. Tests typically start and
stop the system in a fixture so each test has a clean process table, registry,
and monitor table.

## License

Distributed under the Eclipse Public License 1.0. See
[License_EPL_1.0.html](License_EPL_1.0.html).
