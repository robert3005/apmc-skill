---
name: apmc
description: Measure and interpret Apple Silicon hardware performance counters with the apmc CLI. Use for apmc commands, event discovery, counter-based benchmark comparisons, and C/C++ or Rust region instrumentation on macOS. Does not provide sampled profiles, flamegraphs, or assembly inspection.
---

# Apple Silicon performance counters

Use `apmc` to count hardware events while a command runs, or within explicitly instrumented regions. Match the counting scope and event definitions to the performance question.

## Establish the environment

```sh
command -v apmc
apmc --version
apmc stat --help
apmc list
```

These instructions describe `apmc 0.1.0`; use the installed help and event database when they differ. `list` works without root on macOS Apple Silicon. `stat` requires root. Resolve the binary before using sudo, since sudo's PATH may omit Cargo's bin directory:

```sh
apmc_bin="$(command -v apmc)"
sudo -n "$apmc_bin" --no-color stat -- ./target/release/my_program input.dat
```

If sudo requires authentication, retain the prepared command and report that counting could not run. Do not repeatedly retry it or treat event discovery as a successful measurement. The matching upstream README lists disabled SIP as a requirement for counter access; `csrutil status` can inspect it. Changing system security settings is a separate user decision.

In this version, the default and region modes run the target with the elevated identity. System-wide mode drops the child to `SUDO_UID`/`SUDO_GID` when available. Account for this when choosing workloads and interpreting environment-dependent behavior.

## Choose and run a measurement

1. Build the intended optimized executable separately. Measure the executable directly so build tools and benchmark setup do not dominate the count. Keep inputs, build flags, and useful work comparable between variants.
2. Use `apmc list cache`, `apmc list branch`, or another keyword to discover relevant events and read their complete descriptions. Filtering is case insensitive and searches descriptions as well as names; fixed counters and aliases remain visible even with no matching configurable events.
3. Omit `-e` for the default event set, or pass exact names in a comma-separated value with no spaces. Cycles and instructions are always reported. For just those fixed counters, use `-e FIXED_CYCLES,FIXED_INSTRUCTIONS`.
4. Put all `apmc` options before `--` and the executable and its arguments after it. Run counter measurements sequentially: `apmc` acquires and reprograms shared hardware counters.
5. Check stderr for warnings, missing requested events, and the child's reported exit status before interpreting results. A zero `apmc` exit code alone does not establish that the workload succeeded.

```sh
sudo -n "$apmc_bin" --no-color stat \
  -e L1D_CACHE_MISS_LD,BRANCH_MISPRED_NONSPEC \
  -- ./target/release/my_program input.dat \
  >program.stdout 2>measurement.stderr
```

The stderr file also includes the target's stderr. Keep raw output and the exact argument list for each run; the displayed command heading does not preserve shell quoting.

## Select the counting scope

| Scope | Invocation | Meaning |
| --- | --- | --- |
| Whole program | `apmc stat -- ./program` | Default; injects a dylib to aggregate thread counters, with support for descendants. Check injection and process-lifecycle limitations. |
| Instrumented regions | `apmc stat --region -- ./program` | Counts explicit start/stop pairs on the calling threads; requires instrumentation. |
| Whole machine | `apmc stat --system-wide -- ./program` | Sums counters across CPUs during the command, including background activity. |

Read [measurement.md](references/measurement.md) for event selection, output interpretation, comparisons, or failed/suspicious measurements. Read [regions.md](references/regions.md) when adding or reviewing region instrumentation. `--region` (`-r`) conflicts with `--system-wide` (`-s`); `-r` is not a repeat count.

## Interpret and report

Repeat comparable runs when assessing performance differences, and report a representative value and spread. Normalize counts by useful work when workloads differ in size. Report wall time alongside cycles, instructions, IPC, and the selected events; increased IPC alone does not establish a speedup.

Treat event changes as evidence for a hypothesis. A miss count alone is not a miss rate, and speculative micro-ops are not retired instructions. Missing, skipped, or unavailable counters are not zero. Region mode still reports whole-command wall time, so it cannot provide region throughput without a separate region timer.

Include the tool version, CPU, build/workload, counting scope, events actually collected, repetitions, and any relevant measurement limitation. Distinguish observed counter results from source-based expectations or commands that could not run.
