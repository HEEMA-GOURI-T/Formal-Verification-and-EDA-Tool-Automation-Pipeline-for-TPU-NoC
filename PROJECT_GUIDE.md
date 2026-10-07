# Verification Automation Project: Complete Guide

This guide explains the repository section by section (your 14 requested sections, same numbering).
Every code reference points to a file you can run. **Real vs illustrative** is marked throughout.

| Label | Meaning |
|---|---|
| **REAL CONCEPT** | Standard industry practice you can state confidently |
| **WORKING CODE** | Runs in this repo against mock tools / synthetic data |
| **ILLUSTRATIVE** | Shape of the thing; vendor syntax must be filled from your site's docs |
| **MEASURED (MOCK)** | Number produced by this repo's mock/synthetic setup. **Not** a Cadence/Synopsys result |

> The most important sentence for interviews: *"The framework is vendor-agnostic; vendor specifics live in
> `configs/tools.yaml` (command template, return-code map, license feature) and `parsers/` (log regexes)."*

---

## 1. The overall project

### 1.1 Problem
Modern SoC verification is dominated by *execution management*, not just writing checks:
- **Formal tools** prove or disprove thousands of SVA properties. Each run costs a license seat, CPU, and RAM. Runs end as
  *proven*, *counterexample (cex)*, or *undetermined* (solver gave up at the time/memory bound).
- **Simulation** needs stimulus. Constrained-random (CR) is cheap but wastes cycles in already-covered space and rarely
  reaches narrow corners.
- **Floating licenses** are the scarcest resource. Idle seats are wasted money; a starved queue is wasted engineer time.

The project builds one infrastructure that (a) runs formal/simulation reliably and fast, (b) picks stimulus intelligently,
(c) watches the license economics, (d) gates changes in CI.

### 1.2 Why formal + simulation, and why automation
| | Simulation | Formal |
|---|---|---|
| Idea | Execute specific stimulus, check outputs/assertions | Mathematically explore *all* input sequences (bounded or unbounded) |
| Strength | Scales to full chip, software, performance | Exhaustive on blocks; finds corner bugs; proves absence |
| Weakness | Only covers what you stimulate | State explosion; complex properties go *undetermined* |

They are complementary; both generate large volumes of runs, so scheduling, retries, parsing, and cost control must be
automated.

### 1.3 Roles of Cadence and Synopsys tools (REAL CONCEPT)
Both vendors ship (a) RTL simulators, (b) formal property-verification apps, (c) license managers. In practice an
organisation may use both (different blocks, acquisitions, cross-checking). Your wrapper layer hides the differences so the
rest of the code sees one `FormalResult`. *Do not claim universal command syntax.* This repo uses mock tools named
`cadence_mock` / `synopsys_mock` with deliberately *different log formats* to exercise the vendor-adapter design.

### 1.4 Terminology (precise definitions)
- **Execution wrapper**: a program that turns a *job description* (tool, design, properties, seed, timeout) into a safe,
  reproducible tool invocation and returns a *structured result*. Covers: command construction, env isolation, timeout/kill,
  log capture, return-code classification, parsing, retry hooks.
- **"Dynamic assertion throughput"**: ⚠ *terminology trap.* Formal tools prove assertions **statically**; "dynamic" assertions
  are those evaluated during **simulation**. Be ready to define what *you* measured. A defensible definition for formal
  automation: **assertion (property) throughput = properties reaching a conclusive result per wall-clock hour** (optionally
  per license seat). For simulation: **assertion evaluations per second / assertions exercised per regression hour**.
  This repo measures the first (`benchmarks/metrics.py`).
- **Stimulus vector**: the tuple of knobs that defines a test: here `(burst_len, idle_gap, wr_prob, backpressure,
  num_masters, hotspot)`. In a real UVM bench these become plusargs / config-DB values / sequence parameters, *not* raw pin
  values.
- **Corner-case discovery**: finding stimulus that reaches rare/boundary behaviour (full FIFO + stalled reader, arbiter
  starvation, race windows) measured via assertion failures, coverage bins first hit, rare-event counters.
- **License allocation efficiency**: how well seats are turned into useful work: *utilization %* (busy seat-seconds ÷
  available seat-seconds), *wait time* (request→grant), *queue depth*, *idle seat-seconds*, *timeouts/denials*.

### 1.5 How the three components fit
```
 optimizer proposes stimulus ─► runner schedules jobs ─► wrapper runs tool ─► parser structures results
        ▲                              │ (needs seats)                                   │
        │                       license gate + monitor ◄────────────────────────────────┤
        └────────────── score (coverage/failures/rare events) ◄─────────────────────────┘
 CI orchestrates the whole loop on every change and publishes gates + reports.
```

---

## 2. Architecture

```
┌────────────────────────────────────────────────────────────────────────────────┐
│ Python automation layer   (ci/run_pipeline.py, benchmarks/*, optimization/*)   │
│   builds Job list, owns configs, collects metrics                               │
└───────────────┬────────────────────────────────────────────────────────────────┘
                ▼
┌───────────────────────────────┐        ┌────────────────────────────────────┐
│ JobRunner (wrappers/job_runner)│◄──────►│ LicensePool gate + event log       │
│  worker pool, retry, LPT order │ lease  │ (license_monitor/license_pool.py)  │
└───────────────┬───────────────┘        └──────────────────┬─────────────────┘
                ▼                                           ▼ events
┌───────────────────────────────┐                ┌──────────────────────────────┐
│ Execution wrapper             │                │ metrics.py → report.py       │
│ (wrappers/formal_wrapper.py)  │                │ utilization, wait, peak, idle │
└───────────────┬───────────────┘                └──────────────────────────────┘
                ▼   argv list, clean env, own process group, log file
┌───────────────────────────────┐   ┌─────────────────────────────────────────┐
│ Cadence / Synopsys formal tool│◄──│ SVA properties (assertions/*.sv) + RTL   │
│ (mock_eda/mock_formal.py here)│   │ (formal/rtl, formal/templates)           │
└───────────────┬───────────────┘   └─────────────────────────────────────────┘
                ▼ tool.log
┌───────────────────────────────┐
│ Result parser (parsers/)      │ vendor regexes → {prop: proven|cex|undetermined}
└───────────────┬───────────────┘
                ▼ FormalResult
┌───────────────────────────────┐   ┌──────────────────────────────────────────┐
│ Optimization engine           │──►│ Stimulus generation → simulation          │
│ (optimization/bayes_opt.py)   │   │ (stimulus/*, simulation/synthetic_dut.py) │
│ GP + constrained EI           │◄──│ SimMetrics → corner_case_score            │
└───────────────┬───────────────┘   └──────────────────────────────────────────┘
                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ CI/CD (ci/*, .github, .gitlab-ci.yml, Jenkinsfile): gates + reports/          │
└──────────────────────────────────────────────────────────────────────────────┘
```

| Component | Input → Output | Why it exists |
|---|---|---|
| Automation layer | repo state → Job list, reports | Single place that knows *what* to run |
| JobRunner | Jobs → `{id: FormalResult}` | Concurrency, retry policy, scheduling order |
| License gate | seat requests → grants + events | Throttles to owned seats; fairness; telemetry |
| Wrapper | Job → `FormalResult` | Isolates vendor/command/failure details |
| Parser | log text → normalized statuses | Vendor-neutral results |
| Optimizer | past (params, score) → next params | Sample-efficient exploration |
| Simulation | params+seed → `SimMetrics` | Produces coverage/failure evidence |
| Scoring | `SimMetrics` → scalar score | Turns "interesting" into a number |
| CI | commit → gates, artifacts | Prevents regressions; makes results visible |

---

## 3. Execution wrappers  (WORKING CODE + ILLUSTRATIVE commands)

### 3.1 Layout & config
```
wrappers/config.py         # load + validate configs/tools.yaml (fails early, precise messages)
wrappers/formal_wrapper.py # run_formal(...) -> FormalResult
wrappers/job_runner.py     # worker pool, retry, scheduling
wrappers/run_formal.sh     # Bash flavour with the same exit-code contract
configs/tools.yaml         # per-tool command template, env, returncode_map, license feature, parser
formal/templates/          # ILLUSTRATIVE Tcl skeletons with <placeholder> commands
```
`configs/tools.yaml` excerpt (MOCK; replace for real tools):
```yaml
cadence_mock:
  cmd_template: ["{python}", "{repo}/mock_eda/mock_formal.py", "--flavor", "cadence", "--design", "{design}", ...]
  env: {MOCK_STARTUP_S: "0.05"}
  returncode_map: {3: LICENSE_UNAVAILABLE}
  license_markers: ["license unavailable", "all licenses in use"]
  parser: cadence_mock
```
A real entry would look like `["<formal-binary>", "<batch-flag>", "{workdir}/run.tcl"]` and a Tcl script rendered from a
template in `formal/templates/` (ILLUSTRATIVE: command names differ by tool and version; verify with your docs).

### 3.2 `run_formal` line-by-line (`wrappers/formal_wrapper.py`)
1. **Signature** `run_formal(tool, design, assertions, config, timeout, *, seed, workdir, cache_dir, extra_env)`: the
   five requested arguments plus reproducibility knobs. Returns a dataclass; never a raw string.
2. `spec = config.tools.get(tool)`; unknown tool raises `KeyError` (a *programmer* error should be loud; *tool* errors never raise).
3. `run_id`, `workdir`, `cache_dir`, `logfile`: every run gets a unique directory → no log collisions across 50 parallel jobs;
   the cache dir is shared per tool so elaboration results can be reused.
4. `ctx` + `build_command`: fills `{design}`, `{assertions}`, `{seed}`… into the **argv list**. No shell string means no quoting
   bugs or injection (tested with a path containing `; rm -rf /`).
5. `build_env`: starts from a **whitelist** (`PATH`, `HOME`, license variables…), then per-tool env with `${VAR}` expansion. Whitelisting
   makes runs reproducible and keeps secrets out of tool environments.
6. `subprocess.Popen(..., stdout=lf, stderr=STDOUT, start_new_session=True)`: log streams to disk (a chatty tool can print GBs; never
   hold it in memory). A *new session* puts the tool and all its children in one process group.
7. `proc.wait(timeout)` → on `TimeoutExpired`, `_kill_group` sends SIGTERM to the **group**, waits 2 s, then SIGKILL. Killing only the
   parent would leave solver children holding a license seat.
8. `OSError` → `LAUNCH_ERROR` (binary missing, permission).
9. **Return-code classification, in this order**: negative rc (killed by signal) → `CRASH`; `returncode_map` hit (site knowledge, e.g. 3) →
   mapped status; license marker in the log with rc≠0 → `LICENSE_UNAVAILABLE`; other non-zero → `TOOL_ERROR`.
10. **Parse only on rc==0**, via the vendor parser. If rc==0 but *zero* properties parsed → `PARSE_ERROR` ("silent failure guard"): a tool that
    exits 0 after printing nothing useful must not count as "all proven".
11. Success returns `summary` counts (`proven/cex/undetermined/total`).

Statuses give the retry policy something precise to key on: `LICENSE_UNAVAILABLE` and `CRASH` are *transient* (retry);
`TIMEOUT` and `TOOL_ERROR` are usually *deterministic* (retrying wastes seats).

### 3.3 Parallel execution & regression management
`JobRunner` (§9) handles concurrency; regression management = `write_jobs()` in `benchmarks/throughput_bench.py` (batches properties
per tool session), per-batch assertion files, seeds recorded in each `FormalResult`, `logs/runs/<tool>_<id>/tool.log` per run,
and JSON/Markdown reports in `reports/`.

### 3.4 Connecting to real tools (checklist)
1. Capture one real log per outcome (proven / cex / undetermined / license-denied / crash).
2. Write a Tcl/script template in `formal/templates/`; render it into `{workdir}` before launch (add a render step to the wrapper).
3. Put the real launcher in `cmd_template`; record real exit codes in `returncode_map`.
4. Replace regexes in `parsers/formal_log_parser.py` using the captured logs; add them as **fixtures** in `tests/`.
5. Set `license_feature` to your site's feature name; set `configs/licenses.yaml` seat counts.
6. Run the same test-suite; only parser fixtures and config should change.

---

## 4. Assertion throughput

### 4.1 Where throughput is lost
| Factor | Mechanism | Evidence to collect |
|---|---|---|
| Number of assertions | linear job count; tool session setup per job | jobs/hour, props/job |
| Compilation/elaboration | each session re-reads, re-elaborates RTL | wall time before first property starts |
| Tool startup | process launch, license checkout, DB load | time-to-first-result per session |
| Property complexity | deep temporal ranges (`##[1:100]`), wide cones → large state | per-property solve time distribution |
| Solver complexity | engines time out → *undetermined* = effort with no conclusion | undetermined % (use *conclusive* throughput) |
| Parallel jobs | too few = idle CPUs; too many = memory thrash / license denial | CPU util, RSS, denials |
| CPU/memory | oversubscription slows everything; OOM kills jobs | `os.times`, cgroup stats |
| Incremental runs | re-proving unchanged properties after a small RTL edit | % properties whose cone of influence changed |
| Result parsing | regexes on GB logs, re-parsing | parser seconds |
| Scheduling | long jobs started last → tail latency | makespan vs ideal (Σtime/workers) |

### 4.2 Methodology that could credibly produce ≥25%
**Metric (primary):** conclusive properties per wall-clock hour on a *fixed* benchmark set. Secondary: regression wall time, undetermined %, CPU util, denied-license retries.

**Protocol:** freeze RTL + property set + seeds + tool version; ≥5 repetitions of each variant; cold caches each time for the cold case;
run baseline and candidate interleaved on the same host class; report medians + min/max; one change at a time (ablation) so you can say
*which* technique produced *which* part of the gain.

**Candidate techniques** (each toggleable in `benchmarks/throughput_bench.py`):
1. **Batch properties per session**: amortize startup/elaboration.
2. **Elaboration/compile cache** keyed by design hash (+ tool version + defines). Invalidate on any RTL/define change.
3. **Longest-processing-time-first scheduling** using historical runtimes (`Job.est_cost`).
4. **Right-sized worker pool**: workers ≤ seats, ≤ RAM/worker budget.
5. *(Real-tool extras, not mocked)*: tiered time limits (short first pass, escalate undetermined), grouping by shared cone of influence, only re-running properties whose cone changed.

**Why 25% is plausible, not guaranteed:** if profiling shows e.g. startup+elaboration are a large share of baseline wall time, removing most of that
overhead yields a large gain; if the solver dominates, gains will be smaller. *You must profile your own baseline first.* The 25% in your resume is
only defensible if you can show such a profile and a before/after.

### 4.3 MEASURED (MOCK): methodology demonstration
From `reports/throughput_bench.md` (median of 5, 48 properties, mock overheads: startup 50 ms, elaboration 80 ms, per-property 4–36 ms):

| variant | wall s | props/hour | cpu util % |
|---|---|---|---|
| baseline (1 prop per job, no cache, random order, 4 workers) | 3.31 | 52,283 | 12.1 |
| + batching | 0.70 | 246,187 | 7.5 |
| + elaboration cache | 0.60 | 286,713 | 8.7 |
| + LPT scheduling | 0.55 | 316,071 | 11.0 |
| + 8 workers (only 6 jobs → no extra benefit) | 0.63 | 274,723 | 4.6 |

⚠ **Do not quote these.** They are enormous because the mock's per-process overhead dominates by construction. They prove
the *measurement pipeline* and show ablation structure (batching is the biggest lever; the last row shows more workers do
nothing when jobs < workers). Real tools spend most time in solvers, so real gains are smaller and must be measured.

### 4.4 Collecting metrics (`benchmarks/metrics.py`)
```python
with CpuProbe() as probe:               # os.times() children CPU + wall
    results = runner.run(jobs)
m = collect("candidate", results, runner.stats, probe, workers=8)
m.throughput_props_per_hour, m.conclusive_per_hour, m.cpu_utilization_pct
pct_improvement(before, after, higher_is_better=True)
```

---

## 5. Bayesian Optimization

### 5.1 First principles
**Problem:** maximize an expensive, noisy black-box `f(x)` (here: corner-case score of a simulation) with few evaluations.
**Idea:** keep a *surrogate* model of `f` that also reports uncertainty; choose the next `x` where an *acquisition function* says
the trade-off between "probably high" and "very uncertain" is best; evaluate; update; repeat.

**Gaussian Process (GP)** places a prior over functions: `f ~ GP(m(x), k(x,x'))`. With kernel `k` (here Matérn-5/2 with a separate
length-scale per dimension) and data `(X, y)`, the posterior at a new point is Gaussian with mean `μ(x)` and std `σ(x)`. Near data
`σ` is small; far from data `σ` is large. A white-noise term models simulator randomness.

**Expected Improvement (maximization):** with incumbent best `f*` and `z = (μ − f* − ξ)/σ`:
```
EI(x) = (μ − f* − ξ)·Φ(z) + σ·φ(z)
```
First term = *exploitation* (high mean), second = *exploration* (high uncertainty). `ξ` (0.01 here) biases slightly toward exploring.

**Constraints** (`sim_cost`, `memory_mb`): a second GP per constraint gives `P(feasible) = Φ((limit − μ_g)/σ_g)`; acquisition = `EI × ΠP(feasible)`.
**Legality** (`burst_len*num_masters ≤ 160`) is a *hard* rule enforced by rejection in candidate generation, because illegal stimulus is
meaningless, not just costly.

### 5.2 Mapping to verification
| BO term | In this project |
|---|---|
| Search space | `configs/bo_search_space.yaml`, 6 typed knobs, encoded to the unit cube (`stimulus/space.py`) |
| Stimulus vector | decoded dict of knobs → testbench plusargs/config |
| Objective | `corner_case_score` (§6), maximize |
| Surrogate | `GaussianProcessRegressor`, Matérn 5/2 ARD + white noise, `normalize_y=True` |
| Acquisition | constrained EI |
| Loop | init → fit → candidates → acquire → evaluate → update → stop |

### 5.3 Code walk-through (`optimization/bayes_opt.py`)
- `make_gp`: kernel = `Constant × Matern(ARD) + WhiteKernel`; ARD length-scales tell you *which knobs matter* (a useful interview talking point).
- `expected_improvement`: the formula above, vectorized; `sigma` floored to avoid divide-by-zero.
- `run()` step 1: **initial design**: `random_legal(n_init)`; gives the GP something to fit and a first feasibility estimate.
- Step 2: `_fit` trains the score GP and one GP per constraint (ConvergenceWarnings silenced; they occur with few points).
- `_candidates`: 75 % Sobol low-discrepancy global samples (explore) + 25 % Gaussian perturbations of the top-5 feasible points (exploit), snapped to
  representable ints, filtered for **legality** and **novelty** (never re-run an identical stimulus).
- `_acquire`: if nothing feasible yet, maximize `P(feasible)` only (find the feasible region first); else constrained EI.
- Batch (`batch_size>1`): *kriging believer*: after picking a point, pretend its outcome equals the GP mean, refit, pick the next. This yields distinct
  points so several simulations can run in parallel on the farm.
- Stopping: `target_score_reached`, `ei_converged` (max acquisition < `ei_tol` for `patience` iterations), `budget_exhausted`, `no_candidates`.

### 5.4 Known limitations (say them before the interviewer does)
- **Needle-in-haystack / flat objective** (`tests/test_bayes.py::test_flat_plateau_is_a_known_failure_mode`): binary pass/fail gives the GP nothing to learn. Use a shaped score.
- **Noise**: each simulation depends on a random seed; the white-noise kernel helps, but you may need repeats for borderline points.
- **Dimension**: GPs degrade past ~20 dimensions; use TuRBO / random-embedding / feature screening.
- **Categorical/structured knobs** (sequence types): need other kernels or encodings (or tree-based surrogates like SMAC).
- **Cost**: GP fitting is O(n³); fine for hundreds of evaluations, not millions.

---

## 6. Corner-case discovery

### 6.1 Signals (what real environments provide)
| Signal | Source in a real flow | Pitfall |
|---|---|---|
| Assertion failure | simulator/formal log | A failure may be a TB bug; triage (check legality, reproduce with seed) |
| Functional coverage | covergroups, cross bins | Bins can be trivial; weight rare crosses higher |
| Code/toggle coverage | simulator coverage DB | Necessary, not sufficient; poor at temporal corners |
| State-space exploration | FSM state/transition coverage, abstract state tuples | Needs a good abstraction |
| Rare-event frequency | counters (full-FIFO cycles, retries, arbitration losses) | Heavy-tailed → log-compress |
| Newly reached states/bins | diff vs. merged coverage | History-dependent (see below) |
| Failure severity | rules: data-loss > protocol > performance | Define severities with the design owners |

### 6.2 Scoring function (`stimulus/scoring.py`)
```
score = 0.35·coverage_fraction + 0.20·rare_event_term + 0.30·failure_severity + 0.15·state_diversity
rare_event_term = min(1, log1p(events)/log1p(40))      # log compression
failure_severity = overflow:0.6–1.0, starvation:0.3–0.5, none:0
```
**Design decision:** the optimizer objective is *history-free* (a given stimulus always means the same thing). A "new bins vs. everything seen so far"
reward changes meaning as coverage fills, which breaks the GP's assumption that `f` is a fixed function. Cumulative novelty is tracked separately by
`CoverageLedger` for reporting. (Ask: "How do you integrate coverage feedback?" → see Q&A: use a fixed-reference coverage target set, or re-fit with
time-decayed observations.)

**Separating signal from noise:** (a) re-run top candidates with different seeds, (b) require the failure to reproduce with the *same seed*, (c) check
stimulus legality against the constraints, (d) cluster failures by signature (assertion name + first-failing cycle bucket).

### 6.3 End-to-end synthetic demo (`python -m optimization.run_demo`)
`simulation/synthetic_dut.py` is a small cycle model: FIFO depth 16, producer sees occupancy with a 4-cycle lag and throttles at 14, arbiter with hotspot bias.
Overflow needs a long burst, high write probability, high backpressure, and a simultaneous read stall → a narrow region (~8 % of uniform legal samples).

**MEASURED (SYNTHETIC)**, 6 seeds × 60 evaluations (10 initial), `reports/bo_demo.json`:

| metric | constrained-random | Bayesian optimization |
|---|---|---|
| median best feasible score | 0.852 | 0.985 |
| mean # failing stimuli found in 60 evals | 2.5 | 39 |
| median evals to first failure | 20 | 12.5 (−37.5 %) |
| runs that found a failure | 6/6 | 6/6 |

Interpretation: BO's big advantage here is *density*: once it learns where failures live it keeps sampling there, which is what you want for
regression-quality corner coverage. The "time-to-first-failure" gain is modest and noisy; do **not** equate this synthetic −37.5 % with your resume's 40 %.
Your 40 % must come from your own baseline (e.g. simulations-to-first-hit of a defined coverage bin, vs. your previous CR flow).

---

## 7. Continuous Integration

```
Git push/PR ─► CI trigger ─► validate configs ─► elaborate probe ─► formal regression ─► simulation (≥50 concurrent requests)
   ─► assertion analysis (gate: NEW cex vs baseline) ─► coverage gate ─► license report ─► reports/ + artifacts
```
Implemented in `ci/run_pipeline.py` (single entry point, identical locally and in CI); thin adapters:
- **GitHub Actions** `.github/workflows/verification.yml`: hosted runner for mock flow; `real-eda` job disabled by default and meant for a **self-hosted runner** that can reach the license server.
- **GitLab CI** `.gitlab-ci.yml`: `unit` + `regression` stages, JUnit artifacts; use `tags:` to select licensed runners and `resource_group:` to serialise scarce licenses.
- **Jenkins** `ci/Jenkinsfile`: agent label `eda-linux`, `disableConcurrentBuilds`, timeout, JUnit + artifacts.

**Gate semantics (important design point):** infrastructure failures (`LAUNCH_ERROR`, exhausted retries) fail the build; *functional* outcomes are compared to a
stored baseline (`ci/baseline_cex.json`): a **new** counterexample fails, a known/waived one does not. First run creates the baseline.
Each stage prints status/time/detail and writes `reports/ci_summary.{json,md}`.

---

## 8. License monitoring

### 8.1 How floating licenses work (REAL CONCEPT)
A license server holds *N* seats per *feature*. A tool process requests a seat at start (or at specific actions), receives it if one is free, and returns it on
exit (or after a timeout/heartbeat loss). If none are free, tools either queue, fail immediately, or retry depending on vendor/settings. Different features (formal engine
vs. simulator vs. coverage) have different pools. Wasted money = seats idle while work waits for something else; wasted time = jobs waiting/denied.

### 8.2 Design (`license_monitor/`)
| Layer | File | Role |
|---|---|---|
| Client-side gate | `license_pool.py` | Limit our submissions to owned seats; **FIFO fairness** (ticketed queue); records `request/grant/release/timeout` events |
| Metrics | `metrics.py` | From events: requested, granted, active, queued, peak, wait mean/p95, utilization %, idle seat-seconds, timeouts |
| Server reconciliation | `server_poller.py` | Polls the real server status command (`LICENSE_STATUS_COMMAND = "<vendor-license-status-command>"`: **placeholder**; no invented syntax) and parses to `{feature: issued/in_use/queued}` |
| Reports | `report.py` | JSON + Markdown (dashboard-ready) |

Definitions: `utilization = Σ(active_seat·dt) / (seats × window)`, `wait = grant − request`, `idle_seat_seconds = seats×window − busy_seat_seconds`,
`peak_concurrency = max active`, `peak_queue_depth = max queued`. Client-side events cover *our* jobs; server polling catches other users consuming the same pool, which is the main
reason the two views must be reconciled.

**MEASURED (MOCK)**, `python -m benchmarks.license_bench`: 60 jobs requested simultaneously against 20 seats (peak 60 threads):
`peak_concurrency=20, peak_queue_depth=40, utilization≈80.6 %, mean wait≈0.13 s, p95≈0.31 s, timeouts=0, seats never exceeded`
(verified by `tests/test_license.py`). The same code at 50+ is the "50+ concurrent simulation" scenario in `ci/run_pipeline.py::st_simulate`.

**Avoiding starvation:** FIFO tickets (a waiter at the head can't be overtaken), bounded waits with explicit timeout → retry, per-feature queues,
and in production priority classes (e.g., release-blocking > nightly > exploration BO runs) with aging.

---

## 9. Concurrency

| Option | Fit for "run 50+ EDA jobs" | Verdict |
|---|---|---|
| `multiprocessing` | Separate Python processes; pays pickling/IPC cost, buys CPU parallelism we don't need (the heavy work is in the *tool* process) | Overkill |
| `threading` | Threads mostly block on `proc.wait()`; GIL released; cheap, shared memory for license gate/metrics | **Best for orchestration** |
| `subprocess` | The mechanism that actually launches tools; always used *inside* the worker | Required building block |
| `asyncio` | Also fine (`asyncio.create_subprocess_exec`); gains appear at thousands of waiters; more friction with blocking libs | Good alternative |
| Job queue (LSF/Slurm/SGE, Celery, Redis queue) | Needed when jobs run on **many hosts**; scheduler enforces resources | **Right for the farm (scale-out)** |

**Chosen:** `ThreadPoolExecutor` + `subprocess` per job on one launcher host (this repo), designed so the "run one job" function can be replaced by "submit to farm scheduler + wait".
`JobRunner` provides: max concurrency, per-job timeout (wrapper), retry with exponential backoff on *transient* statuses, per-job exception isolation, logging, result dict,
license leasing, LPT ordering. Verified in `tests/test_job_runner.py` (concurrency never exceeds workers; license gate caps below workers; failures isolated).

---

## 10. Repository layout

| Dir | Purpose |
|---|---|
| `wrappers/` | config validation, `run_formal`, `JobRunner`, Bash wrapper |
| `formal/` | example RTL (`rtl/`) and ILLUSTRATIVE script templates (`templates/`) |
| `simulation/` | synthetic simulator stand-in (replace with real simulator launcher) |
| `assertions/` | SVA property files (mock-parsable) |
| `optimization/` | Bayesian optimizer, objective bridge, demo |
| `stimulus/` | search space, scoring, baseline generators |
| `ci/` | pipeline driver, Jenkinsfile, CI baseline file |
| `license_monitor/` | seat gate, metrics, server poller, reports |
| `parsers/` | vendor log parsers |
| `configs/` | tools, licenses, regression, search space |
| `mock_eda/` | **mock** formal tool with failure injection (never real vendor behaviour) |
| `benchmarks/` | metric collection, throughput A/B, license load |
| `tests/` | 72 tests (unit + integration with mock binary) |
| `reports/` | generated reports (MEASURED on mock) |
| `logs/` | per-run tool logs (gitignored in real use) |
| `docs/` | this guide, interview Q&A |

---

## 11. Testing

Run `python -m pytest` (~25 s). **Mock strategy:** two levels: (1) *fake run function* injected into `JobRunner` for fast deterministic policy tests; (2) the *mock tool binary*
(`mock_eda/mock_formal.py`) with failure injection env vars (`MOCK_HANG`, `MOCK_CRASH`, `MOCK_LICENSE_FAIL`) for true subprocess/timeout/signal behaviour.

| Area | File | What is verified |
|---|---|---|
| Config validation | `test_config.py` | missing keys, unknown placeholders, shell-string rejection, bad YAML/timeouts |
| Log parsing | `test_parser.py` | both vendor formats → same vocabulary; garbage → empty |
| Wrapper | `test_wrapper.py` | OK path ×2 vendors, determinism, elaboration cache effect, timeout kills group, crash/license/tool/launch/parse error classes, silent-failure guard, env whitelist, argv safety |
| Retry/failure | `test_job_runner.py` | transient retried then succeeds, retries exhausted, TIMEOUT not retried, wrapper exception isolated, concurrency cap, license throttle, LPT order, end-to-end retry with real mock binary |
| Stimulus | `test_stimulus_space.py` | roundtrip, legality, reproducibility, sandboxed expression, infeasible space |
| Scoring/DUT | `test_scoring_dut.py` | determinism, ordering, bounds, log-compression, ledger |
| Bayesian opt | `test_bayes.py` | EI properties, beats random, learns feasible region, stop rules, no duplicate points, batch distinctness, documented plateau limitation |
| License monitor | `test_license.py` | seats never exceeded, FIFO order, timeout cleanup, exact metrics on hand-built events, poller parse/placeholder guard |
| CI/bench | `test_ci_and_bench.py` | full pipeline pass + reports, new-cex gate fails build, stops at first failed stage, metric math |

**For real tools add:** golden-log fixtures per tool version (contract tests), property-based tests for parsers, soak test with 500 jobs, chaos test (kill license server).

---

## 12. Performance measurement: what you can and cannot claim

| Metric | Before | After | Improvement | Provenance |
|---|---|---|---|---|
| Assertion throughput (props/h) | 52,283 | 274,723 | +425 % | MEASURED (MOCK); not transferable |
| Regression wall time (s) | 3.31 | 0.63 | −81 % | MEASURED (MOCK) |
| Failing stimuli found / 60 evals | 2.5 | 39 | ×15.6 | MEASURED (SYNTHETIC DUT) |
| Evals to first failure | 20 | 12.5 | −37.5 % | MEASURED (SYNTHETIC DUT) |
| Failed jobs | 0 | 0 | n/a | MEASURED (MOCK) |
| License util at 20 seats / 60 requests | n/a | 80.6 % | n/a | MEASURED (MOCK) |
| CPU utilization | 12.1 % | 4.6–11 % | n/a | MEASURED (MOCK; sleep-dominated, meaningless on real tools) |

**Your resume's 25 % and 40 %** are *example/project metrics* until you have: baseline + candidate runs on the same inputs, repeated measurements, a stated definition,
and (ideally) a script/log you can show. Template for your real evidence:

| Metric | Definition | Before | After | Δ | Evidence file |
|---|---|---|---|---|---|
| Conclusive props/hour | proven+cex per wall-hour, fixed set | | | target ≥+25 % | `reports/throughput_bench.json` |
| Time-to-coverage | sims until bin X first hit (median of N campaigns) | | | target ≥−40 % | `reports/bo_demo.json`-style |
| Queue wait p95 | grant−request | | | | `license_report.json` |
| License utilization | busy÷available seat-seconds | | | | `license_report.json` |
| Failed/retried jobs | infra failures per 1000 | | | | `ci_summary.json` |

---
## 13. Interview preparation

See **`docs/INTERVIEW_QA.md`**: 20 questions (beginner → senior), each with *short answer / detailed answer / example from this project / likely follow-up*.

---

## 14. Resume validation

### 14.1 Bullet-by-bullet assessment

**Bullet 1: "Developed execution wrappers around Cadence and Synopsys formal tools for advanced technology nodes, increasing dynamic assertion throughput by 25%."**
- *Strong:* wrappers around two vendors; a quantified outcome; automation is a real, valued skill.
- *Vague / risky:* "advanced technology nodes" (formal tools are mostly node-agnostic: process node affects physical/signoff tools far more than RTL formal; an expert may probe this);
  **"dynamic assertion throughput"** conflicts with formal (static) proof; "25 %" lacks a baseline and definition.
- *Interviewer will ask:* What exactly did the wrapper do? What was throughput measured in? Baseline? Which technique gave the gain? How did you handle timeouts/undetermined/license denials?
- *Evidence needed:* baseline & after runs, definition of the metric, run count, tool versions, benchmark design set.
- *Know before claiming:* §3 (wrapper internals), §4 (bottlenecks, ablation), `JobRunner` retry policy, how your parsers handled each vendor, what "undetermined" means.

**Bullet 2: "Formulated Bayesian Optimization algorithms in Python to generate stimulus vectors, accelerating corner-case discovery by 40% in standard VLSI design flows."**
- *Strong:* a distinctive, modern technique; clear problem (sample efficiency).
- *Vague / risky:* "formulated algorithms" (did you *design* a new algorithm or *apply* BO? say apply); "standard VLSI design flows" is meaningless; "40 %" of what?
- *Interviewer will ask:* search space? objective? surrogate? acquisition? constraints? noise? how do you know a "corner case" is real? baseline (CR? directed?) and how many repeated campaigns?
- *Evidence needed:* campaign logs: evaluations-to-first-hit (or hits per budget) for BO vs baseline, multiple seeds, statistical summary.
- *Know before claiming:* §5, §6, the limitations list, why EI, how legality/cost constraints were handled.

**Bullet 3: "Implemented continuous integration scripts to monitor license allocation efficiency across 50+ concurrent simulation threads."**
- *Strong:* operational maturity; cost awareness.
- *Vague / risky:* no outcome; "threads" (licenses are per process/session; clarify what the 50+ were); "efficiency" undefined.
- *Interviewer will ask:* what metrics? how collected (server poll vs client events)? what decisions did it drive? did utilization or wait time improve?
- *Evidence needed:* utilization/wait/denial numbers before→after, even rough.
- *Know before claiming:* §8 definitions, how floating licenses behave, how starvation is avoided, how scripts ran in CI.

### 14.2 Rewrites (ATS-friendly; fill brackets only with facts you can prove)

1. *Built Python/Bash execution wrappers for Cadence and Synopsys formal tools (command templating, timeout/return-code handling, log parsing, retry), and tuned regression scheduling and batching to raise **conclusive-property throughput by [25]%** on a fixed [N]-property benchmark.*
2. *Applied Gaussian-process Bayesian Optimization (scikit-learn, Expected Improvement with cost/legality constraints) to select constrained-random stimulus parameters, reducing simulations needed to reach [coverage bin / first assertion failure] by **[40]%** versus the prior constrained-random baseline over [N] campaigns.*
3. *Developed Python/CI monitoring of EDA floating-license usage across 50+ concurrent simulation jobs (request/grant/wait/utilization/peak), [reducing license wait time by X% / raising utilization from A% to B%].*

If you cannot prove a number, **remove it** or state it qualitatively ("reduced…") rather than guessing.

### 14.3 Honest framing in an interview
"I built the orchestration and measurement; here is how I defined and measured the improvement; here are the limits (small benchmark set / specific designs)." Candidates who describe limits credibly are believed more than those quoting round numbers.
