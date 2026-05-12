# Getting started with loom-otp

This guide shows how to install `loom-otp`, start the process system, send
messages, handle exits, and use the higher-level server, supervisor, and timer
APIs.

## Prerequisites

- JDK 21 or newer
- Clojure 1.12 or newer
- A project using Leiningen, Clojure CLI, or another Maven-compatible tool

Expected result: by the end of this guide, you will have a small process that
receives a message, a monitored worker, a registered server, and a supervised
child.

## Add the dependency

Leiningen:

```clojure
[org.clojars.arep-engineering/loom-otp "1.0.2"]
```

Clojure CLI / `deps.edn`:

```clojure
org.clojars.arep-engineering/loom-otp {:mvn/version "1.0.2"}
```

## Start and stop the system

`loom-otp` stores process, monitor, registry, and trace state in `mount.lite`
state. Start it before using processes:

```clojure
(require '[loom-otp.core :as otp])

(otp/start!)
```

Stop it during shutdown or at the end of a REPL session:

```clojure
(otp/stop!)
```

In tests, use a fixture so each test starts with an empty process table:

```clojure
(use-fixtures :each
  (fn [f]
    (otp/start!)
    (try
      (f)
      (finally
        (otp/stop!)))))
```

## Spawn your first process

Use `loom-otp.process/spawn!` when you want to run a body in a new process.
The returned pid is a Java `Thread` object.

```clojure
(require '[loom-otp.process :as proc]
         '[loom-otp.process.match :as match])

(def result (promise))

(def pid
  (proc/spawn!
    (deliver result (proc/self))))

@result
;; => #object[java.lang.VirtualThread ...]

(proc/alive? pid)
;; => false, after the process has returned and exited normally
```

Use `spawn` when you already have a function and arguments:

```clojure
(def sum-result (promise))

(proc/spawn
  (fn [a b c]
    (deliver sum-result (+ a b c)))
  1 2 3)

@sum-result
;; => 6
```

## Send and receive messages

Use `proc/send` to deliver a message to a pid or registered name. It returns
`true` when the destination exists and `false` otherwise.

Use `loom-otp.process.match/receive!` for pattern-matching receive:

```clojure
(def greeting (promise))

(def greeter
  (proc/spawn!
    (match/receive!
      [:hello name] (deliver greeting (str "Hello, " name "!")))))

(proc/send greeter [:hello "Ada"])
;; => true

@greeting
;; => "Hello, Ada!"
```

Add an `(after timeout-ms body...)` clause when a receive should not block
forever:

```clojure
(proc/spawn!
  (match/receive!
    [:work item] (do-work item)
    (after 1000 :timed-out)))
```

### FIFO receive vs selective receive

`receive!` reads the next message in FIFO order. If the next message does not
match any clause, the macro throws an exception.

`selective-receive!` scans the mailbox and removes the first message whose
pattern matches. Non-matching messages stay in their original order:

```clojure
(def selected (promise))

(def receiver
  (proc/spawn!
    ;; Give the sender time to enqueue all messages in this example.
    (Thread/sleep 50)
    (let [target (match/selective-receive!
                   [:target value] value
                   (after 1000 :not-found))]
      (deliver selected target))))

(proc/send receiver :noise)
(proc/send receiver [:target 42])
(proc/send receiver :later)

@selected
;; => 42
```

Important: selectivity is based on the pattern, not on the body expression. Do
not use the body to reject a message after it has matched; matched messages are
removed from the mailbox.

## Register a process name

Register a process when callers should not need to hold its pid.

```clojure
(def reply (promise))

(proc/spawn-opt!
  {:reg-name :echo}
  (match/receive!
    [:echo from msg]
    (proc/send from [:echo-reply msg])))

(proc/spawn!
  (proc/send :echo [:echo (proc/self) "hello"])
  (match/receive!
    [:echo-reply msg] (deliver reply msg)
    (after 1000 (deliver reply :timeout))))

@reply
;; => "hello"
```

Names are unregistered when the process exits. Registering a name that is
already in use throws an exception.

## Monitor a process

A monitor observes a process without linking failures. When the target exits,
the watcher receives one `:DOWN` message:

```clojure
(require '[loom-otp.process.exit :as exit])

(def down-message (promise))

(proc/spawn!
  (let [[worker ref] (proc/spawn-monitor!
                       (exit/exit :finished))]
    (match/receive!
      [:DOWN received-ref :process target reason]
      (when (= received-ref ref)
        (deliver down-message {:target target
                               :reason reason})))))

@down-message
;; => {:target #object[java.lang.VirtualThread ...], :reason :finished}
```

Monitor messages have this shape:

```clojure
[:DOWN ref :process target reason]
```

Monitoring a missing target sends an immediate `:DOWN` with reason `:noproc`.

## Link a child and trap exits

Links propagate exits between processes. In the main API, create links when
spawning a child with `spawn-link`, `spawn-link!`, or `spawn-opt` with
`:link true`.

If the parent does not trap exits, an abnormal linked child exit terminates the
parent. Set `:trap-exit true` when you want exits to arrive as messages:

```clojure
(def exit-message (promise))

(proc/spawn-opt!
  {:trap-exit true}
  (proc/spawn-link!
    (exit/exit :child-failed))
  (match/receive!
    [:EXIT child reason]
    (deliver exit-message {:child child :reason reason})
    (after 1000 (deliver exit-message :timeout))))

@exit-message
;; => {:child #object[java.lang.VirtualThread ...], :reason :child-failed}
```

`:kill` is untrappable and is delivered to the target as `:killed`.

## Use a gen_server for stateful logic

`loom-otp.gen-server` provides a stateful server behavior. Define callbacks by
implementing `IGenServer` or by using namespace functions with `ns->gen-server`.

```clojure
(require '[clojure.core.match :as cm]
         '[loom-otp.gen-server :as gs])

(def counter
  (reify gs/IGenServer
    (init [_ args]
      [:ok (or (:initial args) 0)])

    (handle-call [_ request _from state]
      (cm/match request
        :get [:reply state state]
        :inc [:reply state (inc state)]
        [:add n] [:reply state (+ state n)]
        :else [:reply :unknown state]))

    (handle-cast [_ request state]
      (cm/match request
        :inc [:noreply (inc state)]
        :else [:noreply state]))

    (handle-info [_ _message state]
      [:noreply state])

    (terminate [_ _reason _state]
      nil)))
```

Start and call the server from inside a process:

```clojure
(def counter-result (promise))

(proc/spawn!
  (let [{:keys [ok error]} (gs/start counter {:initial 10})]
    (if error
      (deliver counter-result {:error error})
      (do
        (gs/call ok [:add 5])
        (gs/cast ok :inc)
        (Thread/sleep 50)
        (deliver counter-result (gs/call ok :get))))))

@counter-result
;; => 16
```

`gs/start` returns `{:ok pid}` or `{:error reason}`. `gs/call` throws on timeout
or server death.

## Supervise children

Use a supervisor when child processes should be restarted after failures.

```clojure
(require '[loom-otp.supervisor :as sup])

(defn worker []
  (match/receive!
    :stop (exit/exit :normal)
    [:crash reason] (exit/exit reason)))

(def sup-result (promise))

(proc/spawn!
  (let [{:keys [ok error]}
        (sup/start-link
          {:strategy :one-for-one
           :intensity 5
           :period 10000}
          [{:id :worker
            :start [worker]
            :restart :permanent
            :shutdown 5000
            :type :worker}])]
    (if error
      (deliver sup-result {:error error})
      (deliver sup-result (sup/which-children ok)))))

@sup-result
;; => [{:id :worker, :pid ..., :type :worker, :restart :permanent}]
```

See [Supervision guide](SUPERVISION.md) for restart strategies and dynamic child
management.

## Run delayed and periodic work

Timers are implemented as processes.

One-shot timer:

```clojure
(require '[loom-otp.timer :as timer])

(def delayed (promise))

(proc/spawn!
  (timer/start-timer {:after-ms 100}
                     proc/send
                     (proc/self)
                     :done)
  (match/receive!
    :done (deliver delayed :ok)
    (after 1000 (deliver delayed :timeout))))

@delayed
;; => :ok
```

Interval timer:

```clojure
(def ticks (promise))

(proc/spawn!
  (let [timer (timer/start-timer {:after-ms 50 :every-ms 50}
                                 proc/send
                                 (proc/self)
                                 :tick)
        count (atom 0)]
    (dotimes [_ 3]
      (match/receive!
        :tick (swap! count inc)
        (after 1000 nil)))
    (timer/cancel timer)
    (deliver ticks @count)))

@ticks
;; => 3
```

One-shot timers are unlinked by default. Interval timers are linked to the
calling process by default, so they stop when the caller dies.

## Next steps

- Read the [API reference](API.md) for namespace-by-namespace details.
- Read the [Supervision guide](SUPERVISION.md) before using restart strategies
  in production code.
- Read [OTPLike compatibility](OTPLIKE_COMPATIBILITY.md) if you are porting from
  `otplike`.
- Use [Troubleshooting](TROUBLESHOOTING.md) when a process blocks, a call times
  out, or a linked process exits unexpectedly.
