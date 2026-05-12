# Supervision guide

Supervisors keep groups of child processes running. A supervisor starts children,
observes their exits through links, and restarts them according to a strategy and
restart policy.

Use this guide when you need a process tree rather than one-off processes.

## When to use a supervisor

Use a supervisor when:

- a process should be restarted after failure
- several related workers should start and stop together
- failure in one worker should affect a predictable subset of other workers
- you need a parent process to own child lifecycle

Use a plain `spawn!` or `spawn-monitor!` when work is one-off and should not be
restarted.

## Basic supervisor

```clojure
(require '[loom-otp.process :as proc]
         '[loom-otp.process.exit :as exit]
         '[loom-otp.process.match :as match]
         '[loom-otp.supervisor :as sup])

(defn worker []
  (match/receive!
    :stop (exit/exit :normal)
    [:crash reason] (exit/exit reason)))

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
      (println "Supervisor failed:" error)
      (println "Children:" (sup/which-children ok)))))
```

`start-link` returns `{:ok supervisor-pid}` or `{:error reason}`.

## Child specs

A child spec tells the supervisor how to start and stop one child.

```clojure
{:id       :worker-a
 :start    [worker-fn arg1 arg2]
 :restart  :permanent
 :shutdown 5000
 :type     :worker}
```

| Key | Required | Default | Description |
| --- | --- | --- | --- |
| `:id` | Yes | none | Unique identifier within the supervisor. |
| `:start` | Yes | none | Vector containing the function and any arguments. The supervisor calls `(apply proc/spawn-link f args)`. |
| `:restart` | No | `:permanent` | Restart policy. |
| `:shutdown` | No | `5000` | How to stop the child. |
| `:type` | No | `:worker` | `:worker` or `:supervisor`. |

Validate specs before starting a supervisor when you want early feedback:

```clojure
(sup/check-child-specs
  [{:id :worker :start [worker]}])
;; => {:ok (...normalized specs...)}
```

Validation returns `{:error errors}` for missing ids, missing starts, invalid
restart policies, invalid shutdown values, or invalid child types.

## Restart policies

The `:restart` policy decides whether a child should restart after it exits.

| Policy | Restarts after normal exit? | Restarts after abnormal exit? | Use for |
| --- | --- | --- | --- |
| `:permanent` | Yes | Yes | Long-running workers that should always exist. |
| `:transient` | No | Yes | Workers where normal completion is expected, but crashes should restart. |
| `:temporary` | No | No | One-off or optional children. |

The supervisor treats these reasons as normal shutdown for `:transient` workers:

- `:normal`
- `:shutdown`
- vectors whose first element is `:shutdown`, such as `[:shutdown :reason]`

All other reasons are abnormal.

Important: a `:permanent` child that returns normally is restarted. If you start
a short-lived task as `:permanent`, it can restart repeatedly until the restart
intensity is exceeded.

## Shutdown behavior

The `:shutdown` value controls how a child is stopped when the supervisor needs
to terminate or restart it.

| Value | Behavior |
| --- | --- |
| positive integer | Send `:shutdown`, wait that many milliseconds, then kill if still alive. |
| `:brutal-kill` | Send `:kill` immediately. The child exits with `:killed`. |
| `:infinity` | Send `:shutdown` and wait until the child exits. |

Choose a finite timeout for most workers so shutdown cannot block forever.

## Supervisor flags

Supervisor flags control restart scope and restart intensity.

```clojure
{:strategy  :one-for-one
 :intensity 5
 :period    10000}
```

| Key | Default | Description |
| --- | --- | --- |
| `:strategy` | `:one-for-one` | Restart scope. |
| `:intensity` | `1` | Maximum restarts allowed within `:period`. |
| `:period` | `5000` | Restart window in milliseconds. |

If more than `:intensity` restarts occur within `:period`, the supervisor exits
with `:max-intensity`. Because supervisors are usually linked to a parent, that
exit may propagate unless the parent traps exits.

## Restart strategies

### `:one-for-one`

Restarts only the child that exited.

Use it when workers are independent.

```clojure
{:strategy :one-for-one}
```

Example: if `:cache` crashes, only `:cache` restarts; `:listener` and
`:metrics` keep running.

### `:one-for-all`

Stops all children and restarts all children when one child exits abnormally and
its restart policy says it should restart.

Use it when children share state or must be reset together.

```clojure
{:strategy :one-for-all}
```

Example: if `:connection` crashes, stop and restart `:reader`, `:writer`, and
`:connection` so they all use a fresh connection state.

### `:rest-for-one`

Restarts the failed child and every child started after it.

Use it when children have an ordered dependency chain.

```clojure
{:strategy :rest-for-one}
```

Example child order:

```clojure
[:database :cache :api]
```

If `:cache` crashes, restart `:cache` and `:api`; keep `:database` running.

## Dynamic child management

Dynamic operations send requests to the supervisor and wait for replies, so call
them from inside a process.

### List children

```clojure
(sup/which-children supervisor-pid)
;; => [{:id :worker, :pid ..., :type :worker, :restart :permanent}]
```

### Start a child

```clojure
(sup/start-child supervisor-pid
  {:id :extra-worker
   :start [worker]
   :restart :temporary})
;; => {:ok pid}
```

### Terminate a child

```clojure
(sup/terminate-child supervisor-pid :extra-worker)
;; => :ok
```

The native supervisor removes the child from its current child list after
`terminate-child` succeeds.

### Delete a child

```clojure
(sup/delete-child supervisor-pid :extra-worker)
```

Deletes a child entry only when the child is present and not running. Returns
`{:error :running}` if the child is still alive, or `{:error :not-found}` when
the supervisor has no child with that id.

### Count children

```clojure
(sup/count-children supervisor-pid)
;; => {:specs 2, :active 2, :supervisors 0, :workers 2}
```

## Example: observe a restart

This example starts a supervised worker, crashes it, and compares the pid before
and after restart.

```clojure
(def restart-result (promise))

(proc/spawn!
  (let [{:keys [ok]} (sup/start-link
                       {:strategy :one-for-one
                        :intensity 5
                        :period 10000}
                       [{:id :worker
                         :start [worker]
                         :restart :permanent}])
        [{old-pid :pid}] (sup/which-children ok)]
    (proc/send old-pid [:crash :boom])
    (Thread/sleep 200)
    (let [[{new-pid :pid}] (sup/which-children ok)]
      (deliver restart-result {:old old-pid
                               :new new-pid
                               :restarted (not= old-pid new-pid)}))))

@restart-result
;; => {:old ..., :new ..., :restarted true}
```

## Supervising gen_servers

The native supervisor supervises the process that runs the child function in the
child spec. It does not adopt a pid returned by that function.

If a child function starts a separate `gen_server` process and then returns, the
supervisor observes the wrapper function's process, not the returned server pid.
For native supervision, run the long-lived worker logic in the child function
itself or build a wrapper that stays alive for the lifecycle you want to
supervise.

If you are porting code from `otplike`, review
[OTPLike compatibility](OTPLIKE_COMPATIBILITY.md). The compatibility supervisor
uses otplike-style child start results such as `[:ok pid]`.

## Operational guidance

- Keep child functions long-running unless their restart policy is `:temporary`
  or `:transient`.
- Choose `:one-for-one` by default for independent workers.
- Use `:one-for-all` only when every child must reset after any child failure.
- Use `:rest-for-one` for ordered dependencies.
- Set restart intensity high enough for expected transient failures but low
  enough to stop crash loops.
- Give workers a finite `:shutdown` timeout unless they must complete cleanup.
- Let supervisors own child lifecycle; avoid externally killing supervised
  children unless you want supervisor restart behavior.

## Troubleshooting supervision

| Symptom | Likely cause | Resolution |
| --- | --- | --- |
| Supervisor exits with `:max-intensity` | Child crashes repeatedly within `:period`. | Fix the child crash, lower restart frequency, or adjust `:intensity`/`:period`. |
| Child restarts after returning normally | Child is `:permanent`. | Use `:transient` or `:temporary` for tasks that may complete. |
| Parent exits when supervisor exits | Supervisor is linked to parent and parent is not trapping exits. | Start the parent with `:trap-exit true` if it should handle supervisor failure. |
| `start-link` returns `{:error ...}` | Invalid child specs or child startup failure. | Check `sup/check-child-specs` and inspect the returned reason. |
| Dynamic operation times out | Caller is not in process context, supervisor is dead, or mailbox is blocked. | Verify `proc/alive?`, call from a process, and inspect `process-info`. |
