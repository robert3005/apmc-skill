# Measurement details

Use this reference for event selection, benchmark comparisons, and troubleshooting. Behavior below is grounded in the binary probes and matching source recorded in [probe-notes.md](probe-notes.md); privileged measurement paths were not executed during skill creation.

## Events and CLI behavior

The default configurable events are:

```text
L1D_CACHE_MISS_LD,L1D_CACHE_MISS_ST,ATOMIC_OR_EXCLUSIVE_FAIL,MAP_STALL,LDST_X64_UOP,BRANCH_MISPRED_NONSPEC,MAP_SIMD_UOP,SCHEDULE_EMPTY
```

`-e` selects a custom set in place of the defaults. Fixed cycles and instructions remain in the output. Event names and aliases are exact and case sensitive: the probed database accepts `Cycles` and `Instructions`, but not `cycles` or `instructions`. Copy names from `apmc list`; the displayed numeric event IDs are not accepted as names. An empty string is an invalid event, not a request for fixed counters only.

Unknown names generate warnings and are skipped. If no valid names remain, `stat` exits 1 before running the target. A partially valid list can still run, so compare the requested events with the rows actually reported.

The hardware has limited configurable slots, typically eight for the supported M-series machines. `stat` queries actual capacity after acquiring counters. The `list` header prints database metadata directly: the probed machine displayed `Fixed counters: 3, Configurable counters: 1020`. Do not interpret that as 1,020 usable slots. More requested configurable events than available slots causes `TooManyEvents`; conflicting slot masks can cause individual events to be skipped even below that limit. Split these into separate runs with identical work. Results can be reordered by slot assignment; parse by event name.

There are no built-in JSON/CSV, repetition, PID-attach, or sampling options in this version. The trailing command accepts hyphenated arguments, so an unsupported flag can become the executable name instead of producing an option error. Use only options advertised in help and always put `--` before the workload.

## Choose a small event set

Discover these on the actual machine before using them; availability and definitions vary by CPU.

| Question | Candidate events | Interpretation constraint |
| --- | --- | --- |
| L1 data misses | `L1D_CACHE_MISS_LD,L1D_CACHE_MISS_ST` | Counts missed loads/stores; a rate requires a compatible access denominator. |
| Branch prediction | `BRANCH_MISPRED_NONSPEC` | Retired mispredictions; a branch miss rate needs a working retired-branch counter with matching scope. |
| Atomic contention | `ATOMIC_OR_EXCLUSIVE_FAIL,ATOMIC_OR_EXCLUSIVE_SUCC` | Read the database caveat about undercounting exclusive operations in some cache states. |
| SIMD/FP work | `MAP_SIMD_UOP,MAP_INT_UOP` | Mapped micro-ops, not a retired instruction mix or percentage of vector utilization. |
| Stalls | `MAP_STALL,SCHEDULE_EMPTY` | Different conditions; do not add overlapping cycle events into a stall percentage. |
| Boundary crossings | `LDST_X64_UOP,LDST_XPG_UOP` | Crosses 64-byte or 16-KiB boundaries respectively; not cache miss counts. |
| Address translation | `L1D_TLB_MISS,L2_TLB_MISS_DATA` | Different translation levels; use the full event descriptions. |

The matching upstream README reports that configurable `INST_*` events can silently read zero without Apple's private `com.apple.private.kperf` entitlement. Treat such zeros as potentially unavailable. `MAP_*_UOP` events can provide related evidence, but their semantics differ; do not substitute them as denominators for retired-instruction rates. Fixed `instructions` is a separate counter.

## Output and comparisons

- `list` writes its catalog to stdout. `stat` writes the CPU summary, counter table, IPC, wall time, warnings, and failed-child status to stderr. The target inherits stdout and stderr.
- Use `--no-color` for capture. `NO_COLOR` also disables styling, but sudo can filter environment variables. Counts use comma thousands separators.
- A failed child is printed as `(exit status Some(N))`, or `None` for signal termination. If collection succeeds, `apmc` itself returns 0. Exclude failed or incomplete work based on this status and workload validation.
- IPC is fixed instructions divided by fixed cycles. These aggregate across measured threads and are not wall-clock cycles. Compare the same scope and work, and show runtime as well as counts.
- Preserve the executable, input, build settings, thread count, and relevant environment for each variant. Use a warm-up appropriate to the question and repeat runs sequentially; report median and range or another stated summary.
- Do not mix system-wide and per-process results into one comparison. Background load affects system-wide counts. Region counts exclude unmarked work, but the printed wall time spans the whole command.

## Counting limitations and recovery

| Symptom or workload | Action |
| --- | --- |
| `Error: NotRoot`, or sudo needs a password | Counting did not run. Provide the resolved command for an authenticated session; retain any completed discovery work. |
| `ApiError`, `LoadError`, or `MissingSymbol` | Check macOS/architecture, tool version, SIP status, and whether another counter tool is running. The implementation uses private kperf APIs. Report the exact error before proposing environment changes. |
| `unknown event` or `No valid events to monitor` | Correct spelling/case, remove spaces after commas, and rediscover events on this CPU. |
| `TooManyEvents` or `cannot assign event ... skipping` | Reduce or split the event set. Do not report a skipped event as zero. |
| `per-process counting failed: no results from inject dylib` | Confirm the target ran and can load the injected library. Prefer a directly built executable; protected binaries or runtime restrictions can prevent injection. Use system-wide mode only if its broader scope answers the user's question, and label the change. |
| Unexpected zero counts | Check workload success, injection, region markers, and entitlement-sensitive events before concluding the workload performs none of that activity. |
| Target uses `SIGUSR2` or other injected tooling | The injected library installs a `SIGUSR2` handler, and the launcher replaces `DYLD_INSERT_LIBRARIES`. Consider these interactions when choosing the measurement approach. |
| Long-lived descendants or many short-lived threads | Per-process collection waits for EOF on a pipe inherited by descendants. Its thread-slot allocator has a 1,024-slot per-process limit without recycling; heavy thread churn can undercount. |

Default per-process mode attempts to collect natural thread exits, live thread pools at teardown, and fork descendants. Abrupt termination, failed injection in descendants, and the limits above can produce incomplete data. Do not assume process-tree coverage merely because the top-level program returned a table.
