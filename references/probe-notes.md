# Probe provenance

Recorded on 2026-09-09. Use this when updating the skill or distinguishing runtime observations from implementation evidence.

- Binary: `apmc 0.1.0`, installed by Cargo, native Mach-O arm64.
- Host: Apple M4 Max, macOS 26.6.2.
- Cargo install metadata identifies `https://github.com/0ax1/apmc` at `9cff99a9df2ae055f1e3065c64f633e38bb6beec`.
- Matching source was inspected in Cargo's local git checkout. No upstream version update was assumed.

## Executed against the installed binary

| Probe | Observation |
| --- | --- |
| `apmc --version`, top-level and subcommand help | Version and command/option spellings verified. |
| `apmc list` | Exit 0; 102 configurable events, two fixed event rows, aliases, descriptions, and slot masks on stdout. |
| `apmc list bRaNcH` | Exit 0; 16 configurable matches, showing case-insensitive discovery. |
| `apmc list contention` | Exit 0; description-only match for `ATOMIC_OR_EXCLUSIVE_FAIL`. |
| `apmc list no_such_apmc_event` | Exit 0; zero configurable matches, with aliases and fixed rows still shown. |
| `apmc stat --` | Exit 2; a command is required. |
| `apmc stat -r -s -- /usr/bin/true` | Exit 2; incompatible modes rejected. |
| `apmc stat -e NOT_AN_APMC_EVENT -- /usr/bin/true` | Exit 1; warning and no valid events. |
| Lowercase event names, lowercase aliases, a numeric ID, or an empty `-e` value | Rejected as unknown events. |
| `-e 'L1D_CACHE_MISS_LD, BRANCH_MISPRED_NONSPEC'` | Leading space in the second name preserved and rejected; valid first event reaches the root check. |
| `-e Cycles,Instructions` | Names resolve; execution reaches `NotRoot`. This does not verify counter reads. |
| `apmc stat --json /usr/bin/true` | Reaches `NotRoot`, illustrating that an unsupported hyphenated token can be consumed as the command. |
| `apmc --no-color stat /usr/bin/true` | Exit 1, `Error: NotRoot`. |
| `sudo -n true` | Exit 1, password required; privileged probes stopped here. |
| `csrutil status` | SIP enabled on this host. No security configuration changed. |

## Source-checked behavior

- [src/main.rs](https://github.com/0ax1/apmc/blob/9cff99a9df2ae055f1e3065c64f633e38bb6beec/src/main.rs): default events, stderr formatting, child exit reporting without propagation, identity handling, injection, whole-command timing, and descendant result pipe.
- [src/kpc.rs](https://github.com/0ax1/apmc/blob/9cff99a9df2ae055f1e3065c64f633e38bb6beec/src/kpc.rs): exclusive counter setup, actual slot capacity, assignment/skipping, fixed counters.
- [src/kpep.rs](https://github.com/0ax1/apmc/blob/9cff99a9df2ae055f1e3065c64f633e38bb6beec/src/kpep.rs): exact name/alias lookup and direct display of database metadata.
- [inject/kpc_inject.c](https://github.com/0ax1/apmc/blob/9cff99a9df2ae055f1e3065c64f633e38bb6beec/inject/kpc_inject.c): thread/fork tracking, SIGUSR2, slot limits, and region accumulation.
- [src/region.rs](https://github.com/0ax1/apmc/blob/9cff99a9df2ae055f1e3065c64f633e38bb6beec/src/region.rs) and [include/apmc.h](https://github.com/0ax1/apmc/blob/9cff99a9df2ae055f1e3065c64f633e38bb6beec/include/apmc.h): marker wrappers and no-op behavior.
- [Upstream README](https://github.com/0ax1/apmc/blob/9cff99a9df2ae055f1e3065c64f633e38bb6beec/README.md): installation command, SIP requirement, and private-entitlement caveat for configurable `INST_*` events.

Live counter collection, slot failures, child exit reporting during successful collection, and instrumented counter deltas were not runtime-verified. Do not present upstream sample counts as measurements from this host.

## Skill validation

- The skill-creator `quick_validate.py` accepted the completed skill; local Markdown links and code fences were checked.
- The C example in [regions.md](regions.md) compiled with both `cc` and `c++` using the matching upstream header. Both executables ran without injection and returned the expected result, `499999500000`.
- The Rust example compiled and ran in a temporary Cargo project using the matching cached source as a path dependency, with `--offline --release`. It returned the same expected result without injection. This checks the example and wrapper API, not a fresh network install or region counter accuracy.
- Temporary example projects and executables were removed after validation.
