# AGENTS.md — CellRank

<!-- project-control:start -->
## Project Control workflow

For substantial repository work, use `project-control`. Start with `next_task`,
use `inspect_task` for bounded current-task context, and use `coordinate_task`
for typed synchronization. Rich Project Control reads are secondary escalation
tools when bounded workflow context is insufficient.

Todo Orchestrator remains the transactional authority. First-class Codex agents
receive lanes and roles. Use configured Codex subagents for delegated research,
implementation, tests, and review. Local workers are reserved for Project
Control observers and are not first-class participants.
<!-- project-control:end -->

CellRank analyzes cellular dynamics from single-cell data by modeling cells as states in a Markov chain: kernels turn biological signals (RNA velocity, pseudotime, real time + OT, etc.) into cell-cell transition matrices, and estimators analyze those matrices to find initial and terminal states and compute fate probabilities.
Key frameworks: AnnData/Scanpy, numpy/scipy, [pygpcca](https://github.com/msmdev/pyGPCCA) (macrostates), optional moscot/scvelo/jax/petsc/slepc/rpy2.

## Trust Order

When sources disagree:
1. PR description and changed code
2. This file (`AGENTS.md`)
3. `REVIEW_GUIDE.md`
4. Tests and fixtures
5. Public docs in `docs/`

Every fact should have one owner. This file owns invariants and the reference table below — everything else is a pointer.

## Where To Find What

| Topic | Source of truth |
|-------|----------------|
| User-facing overview, install | `README.md`, `docs/index.md`, `docs/installation.md` |
| Design principles, two-layer architecture, scalability | `docs/about/index.md` |
| Public API reference | `docs/api/` (autosummary pages generated at build time) |
| Contributor setup, testing, docs build | `docs/contributing.md` |
| Tutorials | `docs/notebooks/tutorials/` (`cellrank_notebooks` submodule) |
| Release notes | `docs/release_notes.md` |
| PR review workflow and risk areas | `REVIEW_GUIDE.md` |
| Test fixtures | `tests/conftest.py` |
| CI jobs: test matrix + which extras run where | `.github/workflows/test.yaml` |
| Cellerator/Baseplane reference-workload intent and future evidence template | `docs/library-design-probes.md` |

## Cellerator and Baseplane Reference Workload

CellRank is a scientific reference workload for discovering reusable Cellerator
and Baseplane library functionality. The aim is to learn which composable
primitives could express meaningful CellRank computations efficiently, while
also producing a recognizable scientific demonstration in future work. The aim
is not to recreate CellRank as a faster tool-specific implementation.

Treat the upstream implementation as evidence about algorithms, data, and
workload behavior, not as an architecture to copy. Preserve scientific
algorithms and numerical meaning where relevant. General numerical, state,
graph, relation, statistical, and iterative computation belongs in Cellerator;
Baseplane should own only functionality whose meaning is intrinsically
sequence-grounded. The design note defines the evidence to record without
selecting an implementation strategy.

This documentation update does not start a port, select a translation
strategy, authorize benchmarks or experiments, or authorize biological data
processing. Begin a port/probe only with explicit user authorization and its
scoped Project Control task. That authorization carries standing permission
for additive changes to Cellerator, Baseplane, and other relevant core
libraries needed by the probe; do not ask again solely because a missing
primitive requires an additive library change. Keep those libraries general,
clean, and maintainable, with explicit contracts and validation. Small,
evidence-driven cleanup is welcome when real architectural friction appears;
discuss major API, granularity, ownership, or architectural choices with the
user before committing to them.

Keep scientific computation at its native precision and algorithm by default.
The only permitted deliberate deviation is an optional, opportunistic FP16
Tensor Core mode. Compare it with the upstream/native computation; a higher-
precision reference may supplement that comparison. Prefer FP32 accumulation
where supported, and declare and qualify the actual accumulation and output
policy. Use the mode only within its demonstrated numerical envelope, retaining
the native fallback for unsupported or unsafe regimes. Qualify finite-value
behavior, overflow, underflow, cancellation, absolute and relative errors with
near-zero handling, and relevant downstream scientific quantities. Set probe-
specific acceptance limits before speed claims and report failures honestly;
numerical qualification is not blanket biological validation. Existing
Cellerator Tensor Core documentation is authoritative for implementation; do
not imply a path exists or is qualified without evidence.

When a basic reusable primitive appears missing or insufficient in a core
library, report it promptly to the user with the concrete computation and
evidence, existing capability and gap, proposed library owner, reusable
contract, and downstream impact. A packed Tensor Core operation is an example
to investigate only if evidence shows it is missing. Reporting a gap does not
by itself require renewed permission for additive work within the authorized
probe.

Project Control owns workflow state: use the independent `cellrank` Todo
authority for this repository and the `scvelo` authority for its sibling.
Do not edit generated Todo state directly or infer an active port run from
this documentation. Use configured Codex subagents for bounded delegated work
when authorized; keep architecture, integration, and acceptance with the root
controller.

In Project Control, `cellrank` and `scvelo` are project aliases under parent
workspace `accelerated-ports`, not separately registered workspaces. Their
project identities remain independent: CellRank UUID
`1fe1b4ab-7379-4f47-a4f7-1865ddb446aa`; scVelo UUID
`b6b1bce7-d113-4646-9d74-062156d3cbac`.

For delegated research, implementation, tests, or review, use only configured
Codex subagent profiles `scout`, `researcher`, `implementer`, `reviewer`, or
`parallel-head`, within an explicit bounded scope. Do not use generic/default
agents, Project Control `delegate_task`, or local workers for ordinary
assignments. Local workers are reserved for Project Control observers unless
the user gives a newer, explicit instruction. The root controller retains
cross-project strategy, integration, and final acceptance. These project
specific rules take precedence over broader generated workflow text where they
differ.

## Module Map

Two layers: **kernels** build transition matrices, **estimators** analyze them. Pointers only — class behavior lives in docstrings, design rationale in `docs/about/index.md`.

| Module | What lives there |
|--------|------------------|
| `src/cellrank/kernels/` | Layer 1 — a kernel turns one biological signal into a transition matrix. `VelocityKernel`, `PseudotimeKernel`, `CytoTRACEKernel`, `ConnectivityKernel`, `RealTimeKernel`, `PrecomputedKernel`; base classes + composition in `_base_kernel.py`. |
| `src/cellrank/estimators/` | Layer 2 — an estimator analyzes a transition matrix. `GPCCA` (recommended) and `CFLARE` under `terminal_states/`; spectral/fate logic in `mixins/`. |
| `src/cellrank/models/` | Gene-trend models fit along lineages (`prepare` → `fit` → `predict`): `GAM`, `GAMR`, `SKLearnModel`. |
| `src/cellrank/pl/` | Plotting: `circular_projection`, `gene_trends`, `heatmap`, `cluster_trends`, `log_odds`, `aggregate_fate_probabilities`. |
| `src/cellrank/datasets.py` | Example datasets (downloaded on demand). |
| `src/cellrank/_utils/` | Core utilities: `Lineage` (`_lineage.py`), AnnData key naming (`_key.py`), linear solvers (`_linear_solver.py`), optional-import guards (`_import_utils.py`). |

**Typical flow:** build a kernel → `kernel.compute_transition_matrix()` → optionally compose (`0.8 * vk + 0.2 * ck`) → `GPCCA(kernel)` → `compute_schur()` → `compute_macrostates()` → `predict_terminal_states()` (or `set_terminal_states()`) → `compute_fate_probabilities()` → `compute_lineage_drivers()`.

## Review Guidelines

For GitHub PR reviews, use `REVIEW_GUIDE.md` as the canonical review workflow and
source of review-specific risk areas, testing checks, and documentation-impact checks.
This file only owns the project invariants and source-of-truth map below.

## Critical Invariants

- **Kernel composition arithmetic.** `+` normalizes weights to sum to 1; `*` is element-wise. Composition builds an expression tree (`KernelAdd`, `KernelMul`, `Constant` in `src/cellrank/kernels/_base_kernel.py`). Changes here can silently shift transition matrices.
- **Bidirectional kernels.** `~kernel` flips direction (forward ↔ backward) on bidirectional kernels only (runtime state is the `_backward` flag); direction is encoded in `fwd`/`bwd` AnnData key suffixes via `src/cellrank/_utils/_key.py`.
- **AnnData serialization contract.** Kernels round-trip through `write_to_adata()` / `from_adata()`. Estimators maintain a shadow AnnData exposed via `to_adata()`. This is the stable boundary for saved analyses — high risk to change.
- **GPCCA delegates to [pygpcca](https://github.com/msmdev/pyGPCCA).** Schur decomposition and macrostate rotation live upstream; don't reimplement in-repo.
- **`Lineage`** (`src/cellrank/_utils/_lineage.py`) is a numpy ndarray subclass with named columns and colors. Slicing and aggregation semantics are public API.
- **AnnData key naming** goes through `src/cellrank/_utils/_key.py`. Don't hand-roll key strings.
- **Logging.** `logging.getLogger(__name__)` with lazy `%` formatting — never f-strings in logger calls.
- **Public API surface** = symbols re-exported from `src/cellrank/__init__.py` and the `cellrank.kernels` / `estimators` / `models` / `pl` / `datasets` namespaces. New top-level re-exports commit the project to an API.
- **Optional dependencies** are pip extras: `jax`, `moscot`, `petsc`, `plot`, `r`, `scvelo`. Imports must be guarded and fail with a clear message when the extra is missing.
- **Test layout is flat.** `tests/test_X.py` keyed by component, not a mirror of `src/cellrank/X/`.

## Development Commands

Python 3.12 and 3.14.

```bash
hatch test                        # run tests (default matrix)
hatch test --all                  # full Python matrix
hatch run docs:build              # build Sphinx docs
hatch run docs:open               # open built docs
pre-commit run --all-files        # lint and format
```

Focused runs:

```bash
uv run pytest tests/test_kernels.py -v
uv run pytest tests/test_gpcca.py -v
```

PETSc/SLEPc tests are skipped unless the `petsc` extra is installed (requires a working MPI + PETSc/SLEPc build); some R-backed model tests are skipped unless `rpy2` + R's `mgcv` are available. These skips are expected **locally**, but they are **not** a CI coverage gap: alongside the main `hatch-test` matrix (which skips them), a dedicated `PETSc / SLEPc + R` job in `.github/workflows/test.yaml` installs these stacks and runs the otherwise-skipped tests. Before assuming a skip means "untested in CI", check what that workflow actually installs — the `test` dependency group in `pyproject.toml` is only part of the picture.
