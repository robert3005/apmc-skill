# apmc skill

A reusable agent skill for [apmc](https://github.com/0ax1/apmc), an Apple Silicon hardware performance counter CLI. It guides event discovery, command measurements, benchmark comparisons, and C/C++ or Rust region instrumentation.

The skill is in [SKILL.md](SKILL.md). It includes the practical details that are easy to miss in CLI help: exact event names, counter scope, output capture, slot limits, and how to avoid treating incomplete measurements as valid results.

## Requirements

- macOS on Apple Silicon.
- `apmc` available on PATH. The upstream installation command is:

  ```sh
  cargo install --git https://github.com/0ax1/apmc
  ```

- Root access for `apmc stat`; `apmc list` works without it. The matching upstream README also lists disabled SIP as a counter-access requirement. Installing this skill does not change system security settings.
- The upstream header or Rust dependency when adding region markers.

## Install the skill

Install with the [skills.sh](https://www.skills.sh/) CLI:

```sh
npx skills add robert3005/apmc-skill
```

Example requests:

- “Compare cache misses and branch mispredictions for these two release binaries.”
- “Find apmc events related to atomic contention on this Mac.”
- “Measure only this Rust kernel with apmc, excluding input generation.”

## CLI quick start

```sh
apmc --version
apmc list cache
apmc list branch

apmc_bin="$(command -v apmc)"
sudo "$apmc_bin" --no-color stat \
  -e L1D_CACHE_MISS_LD,BRANCH_MISPRED_NONSPEC \
  -- ./my_program input.dat >program.stdout 2>measurement.stderr
```

Build your executable first and replace the example path and arguments. Event names are case sensitive and must be supported by the current CPU. Counter results go to stderr, together with any target stderr. Default per-process mode runs the target with the elevated identity in the probed version.

Use `--region` with explicit markers to count selected code, or `--system-wide` to count across the machine during the command. These modes cannot be combined. `-r` means region mode, not repetitions.

## Contents and validation

| File | Purpose |
| --- | --- |
| [SKILL.md](SKILL.md) | Main agent workflow and scope selection. |
| [Measurement reference](references/measurement.md) | Event selection, result interpretation, and troubleshooting. |
| [Region reference](references/regions.md) | C/C++ and Rust instrumentation examples and semantics. |
| [Probe notes](references/probe-notes.md) | Reproducible observations, matching source revision, and validation limits. |

Created by probing `apmc 0.1.0` on an Apple M4 Max running macOS 26.6.2, and inspecting the matching Cargo source at revision `9cff99a9df2ae055f1e3065c64f633e38bb6beec`. Help, discovery, argument handling, and non-root failures were exercised. Live counter collection could not run because sudo required authentication; source-checked measurement behavior is documented separately from observed results.

The skill passed structural validation, and the C, C++, and Rust region examples compiled and ran without injection against the matching source/header. See the probe notes for the exact validation scope.
