# TraceOpt

[![tests](https://github.com/pranathibandi37-cyber/trace/actions/workflows/tests.yml/badge.svg)](https://github.com/pranathibandi37-cyber/trace/actions/workflows/tests.yml)

**TraceOpt: A Proving-Cost-Aware Compiler Optimization Framework for Zero-Knowledge Virtual Machines**

> Invention-disclosure title: *Method and System for Proving-Cost-Aware
> Compilation for Zero-Knowledge Virtual Machine Targets*

TraceOpt is a research prototype. It asks whether a compiler can make better
optimization decisions for zero-knowledge virtual machines (zkVMs) by
estimating the **cost of proving** a program's execution, instead of relying
only on conventional goals such as native speed or code size.

> **Status: research prototype, working end to end (Phases 0–5).**
> `traceopt optimize` compiles a C kernel with Clang, generates loop-unroll
> variants with LLVM, predicts each one's padded proving size with a model
> calibrated on RISC Zero measurements, picks one (or keeps the baseline),
> and can measure the result in the RISC Zero executor. On a first
> evaluation of 16 unseen cases it halved the padded size once, kept the
> baseline 15 times, and never chose a variant that measured worse; it also
> missed 3 available wins. See [Results](#results) and
> [Project progress](#project-progress).

## Contents

- [The problem](#the-problem)
- [Approach](#approach)
- [How TraceOpt works](#how-traceopt-works)
- [Project progress](#project-progress)
- [Findings so far](#findings-so-far)
- [Results](#results)
- [Benchmarks](#benchmarks)
- [Cost model](#cost-model-initial-proposal)
- [Installation](#installation)
- [Usage](#usage)
- [Testing](#testing)
- [Documentation](#documentation)
- [Repository structure](#repository-structure)
- [Limitations](#limitations)

# The problem

A zkVM runs a program and produces a cryptographic proof that the run was
correct. Generating that proof usually costs far more than running the program,
and the cost depends on properties of the execution trace, such as the number
of cycles and memory paging, rather than on native wall-clock time.

Conventional compiler heuristics optimize for native hardware. Loop unrolling is
an example: it removes loop-control instructions, which can reduce zkVM cycles,
but it also enlarges the code. In RISC Zero, touching additional memory pages
carries a significant per-page cycle cost, so an optimization that helps on a
CPU can be neutral or harmful inside a zkVM.

TraceOpt explores using a calibrated, target-specific proving-cost estimate to
decide **whether and how** to apply an optimization.

## Approach

For the first prototype, the scope is deliberately narrow:

- **One backend:** [RISC Zero](https://dev.risczero.com/) 3.0.6. SP1 is the
  fallback; Phase 0 found no blocker that requires it.
- **One optimization:** loop unrolling, using LLVM's existing `loop-unroll` pass.
  TraceOpt chooses the unroll factor; it does not implement its own unroller.
- **One cost model:** a linear model over a small set of IR features, calibrated
  against measured RISC Zero cycle counts, combined with a model of RISC Zero's
  segment padding.

What is new here, and what is not, is set out in
[docs/novelty.md](docs/novelty.md).

Design choices and their reasons are recorded in
[docs/design-decisions.md](docs/design-decisions.md).

## How TraceOpt works

```
Source (C) → LLVM IR → Program Analysis → Candidate Optimizations
          → Proving-Cost Estimation → Optimization Selection
          → Target Compilation (RISC Zero guest) → Evaluation Report
```

| Stage | Module | Status |
|---|---|---|
| Source → LLVM IR (Clang 21) | `traceopt/toolchain.py` | Implemented |
| Program analysis (loops, trip counts, instruction classes, footprint) | `traceopt/ir.py`, `loops.py`, `features.py`, `footprint.py` | Implemented; matches LLVM LoopInfo on all benchmarks |
| Dynamic count estimation | `traceopt/frequency.py`, `evaluate.py` | Implemented; exact vs. an IR interpreter except data-dependent branches |
| Candidate generation (`opt` loop-unroll factors) | `traceopt/toolchain.py` | Implemented |
| Proving-cost estimation | `traceopt/model.py`, `segments.py`, `calibrate.py` | Implemented; calibrated, 8.3% mean held-out cycle error |
| Optimization selection | `traceopt/decision.py`, `optimize.py` | Implemented, with differential testing and an uncertainty margin |
| Target compilation into a RISC Zero guest | `backend/risc0/`, `traceopt/backend.py` | Implemented (Clang and GCC routes verified) |
| Evaluation and reports (JSON) | `scripts/evaluate.py`, `traceopt/report.py` | Implemented; first evaluation done |

Module responsibilities are described in
[docs/architecture.md](docs/architecture.md).

## Project progress

The work is split into seven phases over about eight weeks. Full task lists
are in the [development roadmap](docs/development-roadmap.md), and work is
tracked as [GitHub issues](https://github.com/prakartii/TraceOpt/issues)
labelled by phase. The percentages below are rough estimates against the
roadmap checklists, not measurements.

| Phase | Goal | Status | Estimate |
|---|---|---|---|
| 0. Feasibility | Toolchain works; a C kernel runs in a RISC Zero guest | Done except proving memory vs. segment size | ~95% |
| 1. IR analysis and baseline | Analyzer, benchmarks, reproducible baseline | Done: validated on Clang 21 IR against LLVM LoopInfo and an IR interpreter | 100% |
| 2. Cost model | Calibrated model with reported error | Done: 64 measured variants, held-out error reported | 100% |
| 3. Cost-aware optimization | Pick an unroll factor or keep the baseline | Done: `traceopt optimize` with differential tests and an uncertainty margin | 100% |
| 4. Backend integration and reporting | Automated build → run → report | Done: `--measure`, guest-only builds (17 s per variant) | ~90% |
| 5. Evaluation | Baseline vs. `-O2` vs. TraceOpt | First evaluation done (16 cases); larger benchmark set pending | ~70% |
| 6. Finalization and demo | Clean setup check, demo, write-up | Narrated demo, HTML report, presenting guide done; clean-install check pending | ~60% |

**Overall: roughly 85%.** The full pipeline works; what remains is
broadening the evaluation and polishing.

**Next steps:**
1. Verify the setup on a clean WSL install ([#26](https://github.com/prakartii/TraceOpt/issues/26)).
2. Measure proving time and memory against segment size ([#3](https://github.com/prakartii/TraceOpt/issues/3)); needs care with the ~7.6 GB memory limit.
3. Improve the model where it fails: memory-heavy loops (strided_rw) and
   multi-segment paging (docs/experiments.md §5).
4. Add benchmarks so the model is calibrated and evaluated on more than four
   kernels.

## Findings so far

All measured in the RISC Zero 3.0.6 executor. Commands and full tables are in
[docs/feasibility.md](docs/feasibility.md) §4.

- **Total cycles are padded to powers of two.** Each segment's size is
  rounded up to 2^k, so cutting user cycles lowers proving work only when it
  crosses a boundary. For example, xorshift at n = 1000 has 12,100 user
  cycles and 18,635 paging cycles, and is padded to 65,536.
- **The segment model matches every measured configuration.**
  `traceopt/segments.py` reproduces the segment count and padded total of all
  9 measured runs, including runs with smaller segment limits
  (`tests/test_segments.py`).
- **Loop overhead is visible and exactly linear.** The xorshift kernel costs
  `95 + 8n` cycles: 8 cycles per iteration, two of which are loop control.
  This is the overhead unrolling targets.
- **Paging depends on memory footprint, not access count.** `strided_rw`
  pays 61,154 paging cycles whether it makes 0 or 1000 passes over its 64 KiB
  array; xorshift pays 18,635. The memory feature in the cost model should be
  distinct pages touched, not memory accesses.
- **Compilers can delete the loop entirely.** rustc/LLVM replaced a
  sum-of-squares loop with its closed form, so its cycle count did not grow
  with n. Benchmarks are designed with loop-carried dependences to avoid this.
- **Segment size changes paging cost.** The same run pays 18,635 paging
  cycles with one segment and 31,825 with four (limit 2^18), so variants must
  be compared at the same segment limit.
- **Proving memory is the hardware limit.** A 2^15-cycle proof took 7.0 s and
  peaked at 303 MB. A 2^20-cycle segment was killed after using 7.5 GB, the
  memory available to WSL on the development machine.

## Results

All numbers below come from committed result files; the commands that
regenerate them are in [docs/cost-model.md](docs/cost-model.md) and
[docs/experiments.md](docs/experiments.md).

**Cost model** (`results/model-clang21.json`, fitted on 64 measured
variants: 4 benchmarks × unroll factors 1, 2, 4, 8 × n = 1, 10, 100, 1000):

| Held-out benchmark | Mean error of predicted kernel cycles |
|---|---|
| branchy | 1.5% |
| nested_fnv | 6.4% |
| xorshift | 7.1% |
| strided_rw | 18.3% |

Each benchmark is predicted by a model fitted without it. Exact instruction
counts from the IR interpreter give almost the same errors, so the remaining
error is in the IR-to-cycles mapping rather than the static analysis.

**Evaluation** (`results/evaluation-clang21.json`, 16 cases at inputs not
used for calibration, all outputs correct):

| Outcome | Cases |
|---|---|
| No unroll factor could lower the padded total; TraceOpt kept the baseline | 12 |
| A lower padded total was available and TraceOpt chose it (strided_rw, n = 5: −50%) | 1 |
| A lower padded total was available and TraceOpt missed it | 3 |
| TraceOpt's choice measured worse than the baseline | 0 |

Clang `-O2` lowered the padded total in one case (nested_fnv, n = 3000) and
matched the baseline in the other 15. The missed wins are analyzed in
[docs/experiments.md](docs/experiments.md#5-results): two are near a
power-of-two boundary, where the 10% safety margin declined to act, and one
is a model failure on a memory-heavy, multi-segment run.

These results cover four small kernels on one machine and one toolchain;
they show the approach working, not a general speedup.

## Benchmarks

Each kernel in `benchmarks/` exports `uint32_t traceopt_kernel(uint32_t n)`
and has a Python reference in `tests/reference.py`. Baseline numbers below
are from [results/baseline-gcc.json](results/baseline-gcc.json): compiled
with RISC Zero's GCC 13.2.0 at `-O2`, default segment limit, 1 segment each.
Every run matched its reference and was identical across repeats.

| Kernel | What it exercises | Code (bytes) | Kernel cycles, n = 1000 | Padded total, n = 1000 |
|---|---|---|---|---|
| `sum_squares` | simple arithmetic loop | 44 | 4,094 | 32,768 |
| `xorshift` | loop-carried bit operations | 64 | 8,095 | 65,536 |
| `nested_fnv` | nested loop (n × 16), hashing with multiply | 72 | 67,098 | 131,072 |
| `branchy` | data-dependent three-way branch | 124 | 14,814 | 65,536 |
| `strided_rw` | strided reads and writes over a 64 KiB array | 140 | 645,641 | 1,048,576 |

These are GCC baselines for the measurement backend. The Clang baseline,
`-O2`, and TraceOpt comparison is in [Results](#results) and
[docs/experiments.md](docs/experiments.md). `sum_squares` is left out of the
Clang experiments because Clang folds its loop into a closed form.

## Cost model (initial proposal)

```
C(F) = Σᵢ wᵢ·nᵢ(F) + w_M·M(F) + w_B·B(F) + w_L·L(F)
```

- `nᵢ(F)`: estimated dynamic counts of instruction classes
- `M(F)`: memory and code-footprint features related to paging
- `B(F)`: estimated dynamic branch count
- `L(F)`: lookup/precompile features (omitted for the first backend)

In the implemented model, the instruction classes are grouped into
compute, memory, and control counts, and their weights are fitted to
measured kernel cycles by non-negative least squares on relative errors
(`traceopt/calibrate.py`, [docs/cost-model.md](docs/cost-model.md)).
Paging is fitted against the number of data pages; `B(F)` is part of the
control group and `L(F)` is omitted. The predicted user and paging cycles go
through the segment model to give a padded total, which is what candidates
are compared on. It is an empirical model for these kernels, with the
held-out errors in [Results](#results). Precise feature definitions are in
[architecture.md §4](docs/architecture.md#4-cost-model-initial-proposal-not-validated).

TraceOpt keeps these quantities separate and never reports one as another:

- **Predicted cost**: the model's output.
- **Execution cycles and segment count**: measured by the RISC Zero executor.
- **Proving time**: measured wall-clock time of proof generation. It depends on
  the hardware and may be unavailable when memory is limited.
- **Native execution time**: not used as a proxy for proving cost.

### Decision rule

`traceopt/decision.py` keeps the baseline unless a candidate:

1. has every prediction it needs,
2. passed differential testing (same outputs as the baseline),
3. is at most 16× the baseline's code size (a loose safety limit; extra
   1 KiB code pages are charged as paging in the prediction), and
4. has a predicted padded total at least 5% below the baseline's.

Among the candidates that qualify, the lowest padded total wins; ties go to
fewer user cycles, then smaller code. Every candidate gets a row explaining
why it was or was not chosen.

## Installation

RISC Zero's installer supports x86-64 Linux and arm64 macOS, so on Windows
the project is developed inside WSL 2 (Ubuntu). The commands below were
verified on Ubuntu 26.04 in WSL.

### Verified toolchain

| Tool | Version |
|---|---|
| rzup | 0.5.0 |
| cargo-risczero, r0vm, `risc0-zkvm` | 3.0.6 |
| RISC Zero guest Rust | 1.97.0 |
| RISC Zero C/C++ toolchain | 2024.1.5 (GCC 13.2.0, `riscv32-unknown-elf`) |
| Host Rust | 1.99.0 |
| Python | 3.10+ (CI: 3.10 and 3.12) |
| LLVM/Clang/LLD | 21.1.8 (Ubuntu packages; CI runs the LLVM tests and the demo with LLVM 21 from apt.llvm.org) |

### Backend (WSL)

```bash
# Toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --profile minimal
curl -L https://risczero.com/install | bash
~/.risc0/bin/rzup install

# Build (first build takes ~15 minutes; uses 4 jobs to stay within memory)
scripts/build_backend.sh
```

LLVM, needed for `traceopt optimize` and the measurement scripts:

```bash
sudo apt-get install -y clang-21 llvm-21 lld-21   # with sudo, or:
scripts/install_llvm_user.sh                      # without sudo, into ~/.local/llvm
source scripts/llvm_env.sh                        # puts clang, opt, llc on PATH
```

### Python tools (any OS)

Standard library only, no dependencies:

```bash
pip install .            # installs the `traceopt` command
# or run without installing:
python -m traceopt --help
```

## Usage

### Demo (WSL, about a minute after the backend is built)

```bash
scripts/build_backend.sh   # once
scripts/demo.sh            # narrated; press Enter between steps
scripts/demo.sh --no-pause --html results/demo-report.html   # also writes the HTML report
```

The demo explains every step in plain language for two kernels,
`strided_rw` (n = 5: the proved size halves) and `xorshift` (n = 1000: the
original is kept):

1. the C program, its input, and the expected answer (from the Python
   reference);
2. the unrolled versions LLVM builds, their code size, and the answer
   check in TraceOpt's IR interpreter;
3. the operations each version executes, its predicted cycles, and a bar
   chart of the power-of-two size the prover charges;
4. the decision, with a reason for every version;
5. the RISC Zero run: the answer the zkVM returned, and measured next to
   predicted cycles for the original, the choice, and the most tempting
   rejected version.

It ends with the 16-case evaluation from `results/evaluation-clang21.json`.
`--no-measure` skips the zkVM runs.

**Report page:** [`results/demo-report.html`](results/demo-report.html) is
the same walk-through as one page with charts, generated from a measured run
(`results/demo-run.json`). Open it in any browser, or rebuild it without
WSL: `python scripts/make_report.py`.
A PDF copy for sending is in
[`results/demo-report.pdf`](results/demo-report.pdf) (printed from the HTML
page in Chrome).

Presenting it to someone new? See [docs/presenting.md](docs/presenting.md)
for what to say at each step and likely questions.

### Live mode: any program, any input (WSL)

```bash
scripts/live.sh                                # asks for a program and n, repeatedly
scripts/live.sh xorshift 777                   # a built-in kernel
scripts/live.sh examples/kernels/poly_hash.c 12345   # your own C kernel
```

Each run prints eight stages as they happen, each as a table, with their
timings:

| Stage | What happens |
|---|---|
| 1 | The program and what the input means |
| 2 | The **expected answer**, computed without TraceOpt: a native build with the host clang, and the Python reference for built-in kernels |
| 3 | Compile to LLVM IR; the loops TraceOpt finds and how many times each runs for this n; how the cost model turns the executed operations into cycles |
| 4 | Build the 1x/2x/4x/8x versions; check each gives the same answers in the IR interpreter (including at this n, up to 2,000) |
| 5 | Predicted cycles and proved (padded) size of each version, with bar charts |
| 6 | The decision and a reason for every version |
| 7 | Run every version in the RISC Zero executor (`--only-chosen`: the original and the choice only) |
| 8 | Predicted vs measured cycles and proved size for every version; zkVM answer vs expected; proving work saved; whether the decision was right (a justified rejection or a missed win) |

**Your own kernel:** any C file defining `uint32_t traceopt_kernel(uint32_t n)`.
It runs without an operating system, so it may use headers such as
`<stdint.h>` but no library functions (`printf`, `malloc`). Loops, `if` and
`switch`, helper functions and recursion, 8- to 64-bit integers (signed or
unsigned), global and local arrays (also 2D), structs, lookup tables and
strings all work: every one of these was tried in live mode with the zkVM
answer checked. Floating point compiles and runs in the zkVM, but TraceOpt's
answer checker does not support it, so unrolled versions are rejected to be
safe and the original is kept. Inputs are 0 … 4,294,967,295. Inputs whose predicted proved size exceeds 2^31 cycles
skip the zkVM run (`--max-cycles` changes the limit). Undefined behavior such
as division by zero is reported instead of compared. Examples are in
[`examples/kernels/`](examples/kernels/).

### Run a benchmark in the zkVM (WSL)

```bash
# Build the guest with one benchmark, run it for each n, check against the reference
scripts/check_kernel.sh nested_fnv 1 100

# Run the most recently built kernel directly: host <execute|prove> <probe|kernel> [n]
~/.cache/traceopt-target/release/host execute kernel 1000

# Cap the segment size (2^18 cycles)
TRACEOPT_SEGMENT_PO2=18 ~/.cache/traceopt-target/release/host execute kernel 100000
```

The host prints JSON with the kernel's output, the cycles spent inside the
kernel, and the session statistics (segments and total, user, paging, and
reserved cycles). `prove` works for small inputs; for larger inputs, cap the
segment size with `TRACEOPT_SEGMENT_PO2` (see
[docs/feasibility.md](docs/feasibility.md) §5).

### TraceOpt commands

```bash
# Choose an unroll factor for a C kernel (needs LLVM on PATH and a fitted model)
source scripts/llvm_env.sh
traceopt optimize benchmarks/xorshift.c --model results/model-clang21.json --arg 0=1000
# ... and build, run, and compare the choice with its prediction
traceopt optimize benchmarks/strided_rw.c --model results/model-clang21.json --arg 0=5 --measure

# Features of an LLVM IR function (JSON). --arg sets loop-bound parameters.
traceopt analyze tests/fixtures/clang21/xorshift.ll --arg 0=1000

# Exact counts from the IR interpreter, next to the static estimate
traceopt profile tests/fixtures/clang21/branchy.ll --arg 0=100

# Predicted segments and padded total for user and per-segment paging cycles
traceopt segments 12100 18635            # → 1 segment, padded total 65536
traceopt segments 804100 7956 --po2 18   # → 4 segments

# Cost-aware choice among candidates
traceopt decide examples/candidates.json [--min-improvement 0.05] [--po2 20]
```

`candidates.json` format (see `examples/candidates.json`):

```json
{
  "baseline":   {"name": "unroll1", "predicted_user_cycles": 30000, "paging_per_segment": 18635, "code_bytes": 140},
  "candidates": [{"name": "unroll2", "predicted_user_cycles": 9000, "paging_per_segment": 18635, "code_bytes": 200}]
}
```

Use `null` for a prediction that is unavailable; that candidate is then
never selected.

## Testing

Unit tests (162, standard library only; run from the repository root). CI
runs them on every push; tests that need LLVM run when clang, opt, and llc
are on PATH and are skipped otherwise:

```bash
python -m unittest discover -s tests
```

They cover the IR parser, loops and trip counts (cross-checked against
simulation over more than 1,000 cases), the analyzer on real Clang 21 IR
(against LLVM LoopInfo and exact counts from the IR interpreter), the
interpreter against the Python references, the segment model (against
measured data), calibration, the decision rule, candidate generation, and
every CLI command.

zkVM checks (inside WSL, after building):

```bash
scripts/smoke_test.sh                     # probe and kernel guests vs. reference
scripts/check_kernel.sh nested_fnv 1 100  # one benchmark in the zkVM
python3 scripts/run_baseline.py --out results/baseline-gcc.json
```

## Documentation

| Document | Contents |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Pipeline, modules, cost-model features, predicted vs. measured quantities |
| [docs/feasibility.md](docs/feasibility.md) | Phase 0 environment, commands, measurements, and blockers |
| [docs/cost-model.md](docs/cost-model.md) | Calibration procedure, feature-set comparison, fitted model, held-out errors, limits |
| [docs/experiments.md](docs/experiments.md) | Evaluation method, results, missed wins, threats to validity |
| [docs/design-decisions.md](docs/design-decisions.md) | Why RISC Zero, C kernels via FFI, Python over textual IR, LLVM's unroller, and later decisions |
| [docs/development-roadmap.md](docs/development-roadmap.md) | Phases 0–6, tasks, dependencies, risks, work split |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Workflow, commit conventions, rules for reporting results |

## Repository structure

```
TraceOpt/
├── .github/workflows/      CI: unit tests (Python 3.10, 3.12) and LLVM tests
├── CONTRIBUTING.md
├── README.md
├── pyproject.toml          package metadata and `traceopt` command
├── backend/risc0/          RISC Zero workspace
│   ├── host/               runs guests, prints JSON measurements
│   └── methods/
│       ├── guest/          Rust-only probe guest
│       └── kernel/         Rust guest that links a C kernel via FFI
├── benchmarks/             C kernels exporting traceopt_kernel(uint32_t)
├── docs/
│   ├── architecture.md
│   ├── cost-model.md       calibration and model accuracy
│   ├── design-decisions.md
│   ├── development-roadmap.md
│   ├── experiments.md      evaluation results
│   └── feasibility.md      Phase 0 measurements and blockers
├── examples/
│   ├── candidates.json     input for `traceopt decide`
│   └── kernels/            custom kernels to try in live mode
├── results/                curated measurement outputs (JSON)
│   ├── baseline-gcc.json           GCC baseline, all five benchmarks
│   ├── variants-clang21.json       calibration measurements (64 variants)
│   ├── variants-clang21-u8-rerun.json  re-measured unroll=8 records
│   ├── fit-clang21.json            fit report (static features)
│   ├── fit-clang21-exact.json      fit report (exact interpreter counts)
│   ├── model-clang21.json          the cost model `traceopt optimize` uses
│   ├── evaluation-clang21.json     baseline / -O2 / TraceOpt evaluation
│   ├── demo-run.json               a measured run of the narrated demo
│   └── demo-report.html            HTML report built from demo-run.json
├── scripts/
│   ├── build_backend.sh    builds the backend (TRACEOPT_GUEST_ONLY=1: guest only)
│   ├── check_kernel.sh     runs one benchmark and checks its output
│   ├── compile_kernel.sh   C -> IR -> unroll -> RISC-V object
│   ├── crosscheck_loops.py analyzer loops vs. LLVM LoopInfo
│   ├── demo.sh             narrated demonstration (runs demo.py)
│   ├── demo.py             the demo itself; --html writes the report
│   ├── live.sh / live.py   live mode: any program and input, stage by stage
│   ├── evaluate.py         baseline / -O2 / TraceOpt comparison
│   ├── fit_model.py        fits the cost model, reports held-out error
│   ├── install_llvm_user.sh  LLVM without sudo
│   ├── llvm_env.sh         puts LLVM on PATH
│   ├── make_report.py      HTML report from saved results (no WSL needed)
│   ├── merge_results.py    replaces re-measured records
│   ├── run_baseline.py     GCC baseline measurements
│   ├── run_variants.py     calibration measurements
│   └── smoke_test.sh       end-to-end backend checks
├── tests/                  unit tests, IR fixtures (tests/fixtures/clang21), references
└── traceopt/               Python package
    ├── ir.py               textual LLVM IR parser
    ├── loops.py            dominators, natural loops, trip counts
    ├── evaluate.py         values computed from known arguments
    ├── features.py         instruction classes and static counts
    ├── frequency.py        block frequencies, dynamic counts
    ├── footprint.py        data pages touched (paging feature)
    ├── interp.py           IR interpreter (ground truth for the analyzer)
    ├── segments.py         RISC Zero segment and padding model
    ├── present.py          plain-language labels and text bar charts
    ├── html_report.py      self-contained HTML demo report
    ├── native.py           native host build: independent expected answers
    ├── calibrate.py        least-squares fit, held-out error
    ├── model.py            the calibrated cost model
    ├── decision.py         cost-aware candidate selection
    ├── toolchain.py        clang / opt / llc wrapper
    ├── optimize.py         candidate generation, prediction, selection
    ├── backend.py          build and run the RISC Zero guest
    ├── report.py           predictions next to measurements
    └── cli.py              `traceopt optimize | analyze | profile | segments | decide`
```

## Limitations

- A single backend (RISC Zero) and a single transformation (loop unrolling).
- C benchmark kernels only. Rust guest code is not optimized by TraceOpt.
- The IR parser handles the subset of LLVM IR the benchmarks need and rejects
  `switch`, `invoke`, and other constructs it does not model.
- Correctness is checked by differential testing, not formal equivalence proofs.
- Proving time may be measurable only for small programs on the development
  machine (about 7.6 GB of RAM available to WSL).
- The cost model is calibrated on four small kernels and may not
  generalize; it under-predicts memory-heavy loops and multi-segment paging
  (docs/experiments.md).
- `env::cycle_count()`, used to measure kernel cycles, is reported by the host
  and not constrained by the proof. It is a measurement aid, not a proven value.

**Future work** (not part of the current plan): other zkVMs such as SP1, more
transformations (inlining, strength reduction), and cost models that account
for precompiles.

## Contributors

- Prakarti Dubey
- Pranathi Bandi

<!-- Add the other team members here. -->

## References

- RISC Zero, *Installation*: https://dev.risczero.com/api/zkvm/install
- RISC Zero, *Optimizing guest programs* (cycles and paging costs): https://dev.risczero.com/api/zkvm/optimization
- Succinct, *SP1 installation*: https://docs.succinct.xyz/docs/sp1/getting-started/install
- LLVM, *LLVM's Analysis and Transform Passes* (`loop-unroll`): https://llvm.org/docs/Passes.html
