# Benchmark comparison: Elixir/BEAM, loom-otp, otplike-compat, and otplike

This report summarizes the checked-in benchmark results under
[`bench/results/`](results/). It includes the Elixir/BEAM comparison that is easy
to miss because the raw analysis also exists as `perf-compare-analysis.txt`.

## Implementations compared

| Implementation | Description | Runtime in checked-in result |
| --- | --- | --- |
| Elixir/BEAM | Native Erlang/OTP process implementation | Erlang/OTP 28, Elixir 1.19.4, ERTS 16.2 |
| `loom-otp` | Native loom-otp API on JVM virtual threads | JVM 25.0.1 in result files |
| `otplike-compat` | otplike API backed by loom-otp | JVM 25.0.1 in result files |
| `otplike` | Original core.async-based otplike implementation | JVM 25.0.1 in result files |

Java implementations were run with G1GC, ZGC, and Shenandoah. Tables below use
G1GC unless another collector is named.

## Running benchmarks

Run all available benchmarks and preserve existing results:

```bash
bench/run-all-benchmarks.sh
```

Overwrite existing results:

```bash
bench/run-all-benchmarks.sh --force
```

Preview commands without running benchmarks:

```bash
bench/run-all-benchmarks.sh --dry-run --force
```

Run one implementation/category:

```bash
bench/bench-compare.sh --impl=elixir messaging
bench/bench-compare.sh --impl=loom-otp --gc=G1GC spawn
bench/bench-compare.sh --impl=otplike-compat --gc=ZGC memory
```

To rerun the original otplike implementation, set `OTPLIKE_BENCH_DIR` to a
separate otplike checkout. This repository contains the loom-otp compatibility
layer and historical otplike result files, but not a full original otplike
project.

## Summary

| Aspect | Winner | Approximate margin |
| --- | --- | --- |
| Large process-tree spawning | Elixir/BEAM | up to 28x over loom-otp at 524K processes |
| Per-process memory | Elixir/BEAM / original otplike | about 2.8x smaller than loom-otp G1GC |
| GC pressure | Elixir/BEAM | about 13x faster in the checked-in allocation benchmark |
| Ping-pong latency | Elixir/BEAM | about 4.2x lower than loom-otp G1GC |
| Raw send throughput | loom-otp / otplike-compat on JVM | about 4x higher than BEAM in this benchmark |
| Receive throughput | Elixir/BEAM and loom-otp | comparable at 1M messages |
| Many-to-one fan-in | Elixir/BEAM and loom-otp | comparable |
| Link/exit propagation | Elixir/BEAM | about 30x faster than Java implementations |

## Spawn performance

### Immediate exit throughput

Fire-and-forget process spawn throughput.

| Implementation | n=1K | n=10K | n=100K |
| --- | ---: | ---: | ---: |
| Elixir/BEAM | 718K/sec | 1,104K/sec | 1,350K/sec |
| loom-otp (G1GC) | 37K/sec | 63K/sec | 84K/sec |
| loom-otp (ZGC) | 70K/sec | 90K/sec | 97K/sec |
| loom-otp (Shenandoah) | 54K/sec | 71K/sec | 87K/sec |
| otplike-compat (G1GC) | 60K/sec | 80K/sec | 85K/sec |
| otplike-compat (ZGC) | 53K/sec | 81K/sec | 87K/sec |
| otplike (G1GC) | 333K/sec | 345K/sec | 273K/sec |

Elixir/BEAM is strongest for raw process spawn throughput. Original otplike is
also fast for fire-and-forget spawns because core.async go blocks are lighter
than JVM virtual-thread process setup in this benchmark.

### Process tree throughput

Binary tree spawn benchmark.

| Implementation | depth=10, 2K procs | depth=15, 65K procs | depth=18, 524K procs |
| --- | ---: | ---: | ---: |
| Elixir/BEAM | 1,778K/sec | 4,152K/sec | 3,705K/sec |
| loom-otp (G1GC) | 499K/sec | 332K/sec | 134K/sec |
| loom-otp (ZGC) | 501K/sec | 549K/sec | 136K/sec |
| loom-otp (Shenandoah) | 477K/sec | 407K/sec | 119K/sec |
| otplike-compat (G1GC) | 384K/sec | 356K/sec | 186K/sec |
| otplike-compat (ZGC) | 395K/sec | 474K/sec | 221K/sec |
| otplike (G1GC) | 87K/sec | 80K/sec | 50K/sec |

Elixir/BEAM is substantially faster for large process trees. Among Java results,
ZGC improves medium-scale loom-otp spawn throughput, and otplike-compat with ZGC
has the best checked-in Java result at depth 18.

### Linked process tree throughput

Binary tree spawn benchmark with links and exit propagation.

| Implementation | depth=10 | depth=15 | depth=18 |
| --- | ---: | ---: | ---: |
| Elixir/BEAM | 986K/sec | 2,745K/sec | 2,691K/sec |
| loom-otp (G1GC) | 88K/sec | 97K/sec | 90K/sec |
| loom-otp (ZGC) | 87K/sec | 197K/sec | 120K/sec |
| loom-otp (Shenandoah) | 104K/sec | 139K/sec | 83K/sec |
| otplike-compat (G1GC) | 105K/sec | 137K/sec | 110K/sec |
| otplike-compat (ZGC) | 77K/sec | 155K/sec | 169K/sec |
| otplike (G1GC) | 26K/sec | 45K/sec | 45K/sec |

BEAM's linked process implementation is much faster for exit cascades.

## Memory performance

### Idle process footprint

| Implementation | n=1K | n=10K | n=50K |
| --- | ---: | ---: | ---: |
| Elixir/BEAM | 2.18 KB/proc | 2.45 KB/proc | 2.54 KB/proc |
| loom-otp (G1GC) | 7.14 KB/proc | 7.14 KB/proc | 7.15 KB/proc |
| loom-otp (ZGC) | 14.34 KB/proc | 9.83 KB/proc | 9.46 KB/proc |
| loom-otp (Shenandoah) | 8.05 KB/proc | 7.77 KB/proc | 7.70 KB/proc |
| otplike-compat (G1GC) | 7.27 KB/proc | 7.27 KB/proc | 7.27 KB/proc |
| otplike-compat (Shenandoah) | 8.17 KB/proc | 7.78 KB/proc | 7.86 KB/proc |
| otplike (G1GC) | 2.79 KB/proc | 2.80 KB/proc | 2.80 KB/proc |

Elixir/BEAM and original otplike have the smallest per-process footprint in the
checked-in results. loom-otp G1GC uses about 7.1 KB per idle process.

### GC pressure

10K spawn/exit operations with allocation pressure.

| Implementation | Time | Final memory |
| --- | ---: | ---: |
| Elixir/BEAM | 9 ms | 63 MB |
| loom-otp (G1GC) | 120 ms | 13 MB |
| loom-otp (ZGC) | 112 ms | 56 MB |
| loom-otp (Shenandoah) | 105 ms | 17 MB |
| otplike-compat (G1GC) | 117 ms | 14 MB |
| otplike (G1GC) | 112 ms | 20 MB |

## Messaging performance

### Ping-pong latency

Round-trip latency converted to microseconds per message.

| Implementation | n=100K | Throughput |
| --- | ---: | ---: |
| Elixir/BEAM | 0.28 µs/msg | 3,607K msg/sec |
| loom-otp (G1GC) | 1.17 µs/msg | 856K msg/sec |
| loom-otp (ZGC) | 1.57 µs/msg | 637K msg/sec |
| loom-otp (Shenandoah) | 1.34 µs/msg | 746K msg/sec |
| otplike-compat (G1GC) | 1.20 µs/msg | 833K msg/sec |
| otplike-compat (Shenandoah) | 1.38 µs/msg | 727K msg/sec |
| otplike (G1GC) | 8.16 µs/msg | 123K msg/sec |

BEAM has the lowest ping-pong latency. loom-otp and otplike-compat are much
faster than original otplike in this benchmark.

### Raw send throughput

| Implementation | n=1M |
| --- | ---: |
| Elixir/BEAM | 5.1M msg/sec |
| loom-otp (G1GC) | 20.5M msg/sec |
| loom-otp (ZGC) | 16.3M msg/sec |
| loom-otp (Shenandoah) | 15.1M msg/sec |
| otplike-compat (G1GC) | 20.7M msg/sec |
| otplike (G1GC) | 19.3M msg/sec |

The JVM implementations win this fire-and-forget send benchmark.

### Receive throughput

| Implementation | n=1M |
| --- | ---: |
| Elixir/BEAM | 7.4M msg/sec |
| loom-otp (G1GC) | 6.8M msg/sec |
| loom-otp (ZGC) | 4.3M msg/sec |
| loom-otp (Shenandoah) | 5.0M msg/sec |
| otplike-compat (G1GC) | 6.8M msg/sec |
| otplike (G1GC) | 0.87M msg/sec |

Elixir/BEAM, loom-otp, and otplike-compat are in the same range for receiving
from a pre-filled queue. Original otplike is slower here.

### Many-to-one fan-in

| Implementation | 10 senders | 100 senders |
| --- | ---: | ---: |
| Elixir/BEAM | 1.85M msg/sec | 1.76M msg/sec |
| loom-otp (G1GC) | 1.90M msg/sec | 1.73M msg/sec |
| loom-otp (ZGC) | 1.77M msg/sec | 1.52M msg/sec |
| otplike-compat (G1GC) | 1.86M msg/sec | 1.33M msg/sec |
| otplike (G1GC) | 0.15M msg/sec | 0.15M msg/sec |

loom-otp matches BEAM closely for this fan-in workload.

## Recommendations from the checked-in results

- Use Elixir/BEAM when you need BEAM-scale process counts, link propagation, and
  low process/message latency outside the JVM.
- Use loom-otp with G1GC for JVM applications that need actor-style message
  passing and Java ecosystem access.
- Use the otplike compatibility layer when migrating existing otplike-style code
  and you want loom-otp's faster messaging path.
- Use ZGC if your loom-otp workload is spawn-heavy and your local benchmark run
  confirms the checked-in trend on your hardware.

## Source data

- Elixir/BEAM: [`results/elixir.txt`](results/elixir.txt)
- loom-otp: `results/loom-otp-*.txt`
- otplike compatibility layer: `results/otplike-compat-*.txt`
- original otplike: `results/otplike-*.txt`
- Longer raw analysis: [`perf-compare-analysis.txt`](perf-compare-analysis.txt)
