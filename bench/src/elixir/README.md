# BenchElixir

Elixir/BEAM benchmark runner used by the top-level loom-otp benchmark suite.

This project provides the Erlang/OTP baseline for the benchmark comparison in
[`../../benchmark-comparison.md`](../../benchmark-comparison.md). It measures the
same broad categories as the JVM runners where practical:

- process spawn throughput
- process tree spawning
- linked process trees
- idle process memory
- spawn/exit memory stability
- message queue growth
- ping-pong latency
- send and receive throughput
- selective receive
- many-to-one and one-to-many messaging

## Requirements

- Erlang/OTP 28 or compatible version
- Elixir 1.19 or compatible version

The checked-in result file was produced with Erlang/OTP 28 and Elixir 1.19.4.

## Run from the loom-otp benchmark wrapper

From the repository root:

```bash
bench/bench-compare.sh --impl=elixir
bench/bench-compare.sh --impl=elixir spawn
bench/bench-compare.sh --impl=elixir messaging
bench/bench-compare.sh --impl=elixir memory
```

To run every benchmark implementation and write result files:

```bash
bench/run-all-benchmarks.sh --force
```

## Run directly with Mix

From this directory:

```bash
mix run -e 'BenchElixir.Runner.main([])'
mix run -e 'BenchElixir.Runner.main(["spawn"])'
mix run -e 'BenchElixir.Runner.main(["messaging"])'
mix run -e 'BenchElixir.Runner.main(["memory"])'
mix run -e 'BenchElixir.Runner.main(["scaling"])'
```

Supported modes are `spawn`, `memory`, `messaging`, and `scaling`. Running with
an empty argument list runs spawn, memory, and messaging benchmarks.

## Output

The top-level wrapper writes Elixir results to:

```text
bench/results/elixir.txt
```

Use this file with the JVM result files when updating the comparison report.
