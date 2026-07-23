# Contribution 1: Working with Pydantic Model Instances

**Contribution Number:** 1  
**Student:** Vinay Surtani  
**Issue:** https://github.com/lance-format/lance/issues/1106  
**Status:** Changes Requested — Phase IV (Follow-Up Discussion Posted)

---

## Why I Chose This Issue

The Lance library currently requires users to manually call `pydantic_to_schema` when creating a table from a Pydantic model, and to explicitly convert model instances to dictionaries before writing data — both steps that the library could handle automatically. This friction makes the API harder to use for developers who are already working with Pydantic models, which are extremely common in Python ML and data pipelines. I chose this issue because it's a well-scoped ergonomic improvement with clear acceptance criteria, and it will push me to understand how Lance handles data ingestion and schema inference under the hood. It's also labeled "good first issue," which gives me confidence that the scope is right for a first open-source contribution.

---

## Understanding the Issue

### Problem Description

Lance does not support passing Pydantic model instances directly to `write_dataset()`. Users must manually convert each model instance to a dictionary using `.model_dump()` and also manually define the PyArrow schema themselves. There is also no convenience method like `LanceDataset.from_pydantic_model()` that can infer the table schema automatically from a Pydantic model class.

### Expected Behavior

Users should be able to pass a list of Pydantic model instances directly to `lance.write_dataset()` and have Lance automatically infer the schema and convert the instances to the correct format. A `LanceDataset.from_pydantic_model()` class method should also exist to create a dataset directly from a Pydantic model class.

### Current Behavior

Passing a list of Pydantic model instances to `lance.write_dataset()` raises: `ValueError: Must provide schema to write dataset from RecordBatch iterable`. Users are forced to manually call `.model_dump()` on each instance and define the PyArrow schema themselves before writing.

### Affected Components

- `python/python/lance/types.py` — `_coerce_reader()` function (line 55): the central function all data paths flow through; has no Pydantic handling
- `python/python/lance/dataset.py` — `LanceDataset` class and `write_dataset()` function: where the new `from_pydantic_model()` method would be added

---

## Reproduction Process

### Environment Setup

Lance is a Rust-backed Python library requiring three core dependencies: Rust, Python 3.10+, and protobuf v3.20+. Setup challenges encountered:

- **Rust not installed:** Installed via `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`, then sourced `$HOME/.cargo/env`
- **pip blocked by Ubuntu 24.04:** Ubuntu prevents system-wide pip installs. Used `uv` (installed via `curl -LsSf https://astral.sh/uv/install.sh | sh`) instead, which is what the project actually uses internally
- **Build time:** First build via `uv sync --all-extras` took ~29 minutes to compile the Rust extension. This is expected on first run.
- **pre-commit installation:** Used `pipx install pre-commit` to avoid the system-managed Python restriction, then ran `pre-commit install` in the repo root

Final working setup commands:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
sudo apt install protobuf-compiler -y
curl -LsSf https://astral.sh/uv/install.sh | sh
cd python && uv sync --all-extras
sudo apt install pipx -y && pipx install pre-commit
cd .. && pre-commit install
```

### Steps to Reproduce

1. Clone the fork and set up the environment as described above
2. Activate the virtual environment: `source python/.venv/bin/activate`
3. Create a Python script with a Pydantic model and attempt to write instances directly:
    ```python
    import lance
    from pydantic import BaseModel

    class MyModel(BaseModel):
        name: str
        score: float

    data = [MyModel(name="alice", score=0.9), MyModel(name="bob", score=0.8)]
    lance.write_dataset(data, "/tmp/test.lance")
    ```
4. Observe the error: `ValueError: Must provide schema to write dataset from RecordBatch iterable`
5. Note that the workaround requires manually calling `.model_dump()` on each instance and defining the PyArrow schema explicitly

### Reproduction Evidence

- **Branch:** https://github.com/vinaysurtani/lance/tree/fix-issue-1106
- **Reproduction script:** https://github.com/vinaysurtani/lance/blob/fix-issue-1106/reproduce.py
- **My findings:** Lance partially handles Pydantic instances — it recognizes them as an iterable — but fails because it cannot infer the schema from Pydantic model instances. The fix needs to detect Pydantic models and extract their schema automatically.

---

## Solution Approach

### Analysis

The root cause is in `python/python/lance/types.py` in the `_coerce_reader()` function (line 55). This is the central function that all data-writing paths funnel through. It has branches for pandas DataFrames, PyArrow Tables, HuggingFace datasets, Polars DataFrames, and others — but no branch for Pydantic model instances. When a list of Pydantic instances is passed, Python treats them as a generic iterable, which hits the RecordBatch iterable path that requires an explicit schema.

### Proposed Solution

Add Pydantic model instance support in two places:
1. In `_coerce_reader()` in `types.py`: detect if the input is a list/iterable of Pydantic `BaseModel` instances, automatically call `.model_dump()` on each, infer the PyArrow schema from the model class, and convert to a `pa.Table`
2. In `dataset.py`: add a `from_pydantic_model()` classmethod on `LanceDataset` that accepts a Pydantic model class and data, infers schema and table name automatically

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Lance cannot accept Pydantic model instances directly when writing data. Users must manually convert instances to dicts and define the schema themselves. The issue asks for automatic handling of both steps.

**Match:** The existing `_coerce_reader()` function in `types.py` already has a pattern for handling third-party types (e.g., the Polars DataFrame branch checks `type(data_obj).__module__.startswith("polars")`). The HuggingFace branch shows how to infer schema from a non-Arrow type. These are the patterns to follow for the Pydantic branch.

**Plan:**
1. In `python/python/lance/types.py`: add a branch in `_coerce_reader()` to detect `pydantic.BaseModel` instances, call `.model_dump()` on each, and use `pydantic`'s type info to build a PyArrow schema
2. In `python/python/lance/dataset.py`: add a `from_pydantic_model()` classmethod on `LanceDataset` that takes a model class and data, infers the schema, converts the table name to snake_case, and calls `write_dataset()`
3. Add `pydantic` as an optional dependency with a check similar to the existing `_check_for_pandas()` helper in `dependencies.py`
4. Write tests in the existing Python test suite covering both new entry points

**Implement:** https://github.com/vinaysurtani/lance/tree/fix-issue-1106 *(implementation in progress — Phase III)*

**Review:** Per CONTRIBUTING.md: all new public APIs must have documentation and examples, and all features must have corresponding tests. Will ensure both before submitting PR. Will also check that no breaking changes are introduced to existing APIs.

**Evaluate:** Will verify by running the reproduction script with the fix applied (should write successfully without manual `.model_dump()` calls), and by running the existing test suite via `pytest python/` to confirm no regressions.

---

## Testing Strategy

### Unit Tests

- [x] `test_input_data[NoneType-list]` — Pydantic instances added to existing parametrized suite; verifies auto-conversion produces correct PyArrow table
- [x] `test_from_pydantic_model` — verifies classmethod writes data, infers schema, and round-trips correctly

### Integration Tests

- [x] `reproduce.py` — end-to-end script confirming `write_dataset()` accepts Pydantic instances directly and the manual workaround still works

### Manual Testing

Ran `python reproduce.py` after implementing the fix. Output confirmed 2 rows written successfully without manual `.model_dump()` calls. Also confirmed the old manual workaround path still functions correctly (no regressions). All 8 parametrized `test_input_data` variants pass including the new Pydantic entry.

Ran the full `test_dataset.py` suite: **196 passed, 1 skipped, 4 failed**. The 4 failures (`test_random_dataset_recall_accelerated`, `test_random_dataset_recall_accelerated_one_pass`, `test_count_index_rows_accelerated`, `test_count_index_rows_accelerated_one_pass`) all fail with `PermissionError: [Errno 13] Permission denied: 'nvcc'` — the CUDA compiler is not installed in the dev environment. These tests are pre-existing failures unrelated to this change and do not touch any code modified in this PR.

---

## Implementation Notes

### Week 3 Progress

Implemented both parts of the issue:

**Part 1 — Auto-convert Pydantic instances in `write_dataset()`:**
Added a new branch in `_coerce_reader()` in `types.py` that detects when the input is a list of Pydantic `BaseModel` instances, automatically calls `.model_dump()` on each, and converts to a PyArrow RecordBatch. Also added `_check_for_pydantic()` to `dependencies.py` following the same optional-dependency pattern used for pandas, polars, and HuggingFace. The branch is placed before the generic `Iterable` branch to ensure Pydantic instances are caught specifically rather than falling through to the error path.

**Part 2 — `LanceDataset.from_pydantic_model()` classmethod:**
Added a classmethod to `LanceDataset` in `dataset.py` that accepts a Pydantic model class and a list of instances, converts the class name to snake_case for the default URI, and calls `write_dataset()` with the inferred schema.

**Challenges:**
- The pre-commit `ruff` linter auto-reformatted imports on first commit — learned to re-stage and commit again after auto-fixes
- Needed to support both Pydantic v1 (`.dict()`) and v2 (`.model_dump()`) — used `hasattr` check to handle both

### Code Changes

- **Files modified:**
  - `python/python/lance/dependencies.py` — added `_PYDANTIC_AVAILABLE` and `_check_for_pydantic()`
  - `python/python/lance/types.py` — added Pydantic branch in `_coerce_reader()`
  - `python/python/lance/dataset.py` — added `from_pydantic_model()` classmethod
  - `python/python/tests/test_dataset.py` — added Pydantic to `test_input_data` parametrize list and new `test_from_pydantic_model` test
- **Key commits:**
  - [Add Pydantic model instance support in _coerce_reader](https://github.com/vinaysurtani/lance/commit/5620779a)
  - [Add LanceDataset.from_pydantic_model() classmethod and tests](https://github.com/vinaysurtani/lance/commit/5bfdbf5a)
- **Approach decisions:** Followed the existing `_check_for_pandas` / `_check_for_hugging_face` pattern for optional dependency detection rather than importing pydantic unconditionally. Added Pydantic to the existing `test_input_data` parametrized suite rather than writing standalone tests, which ensures the new input type is tested consistently with all other supported types.

---

## Pull Request

**PR Link:** https://github.com/lance-format/lance/pull/7383

**PR Description Summary:**
- Added Pydantic model instance auto-conversion in `_coerce_reader()` so `write_dataset()` accepts lists of `BaseModel` instances directly, without requiring manual `.model_dump()` calls
- Added `LanceDataset.from_pydantic_model(model_class, data, uri=None, **kwargs)` classmethod that infers the dataset URI from the model class name (snake_case) and delegates to `write_dataset()`
- Pydantic treated as optional dependency using same `_PYDANTIC_AVAILABLE` / `_check_for_pydantic()` pattern as pandas and HuggingFace
- Supports both Pydantic v1 (`.dict()`) and v2 (`.model_dump()`)
- Added to existing `test_input_data` parametrized suite + new `test_from_pydantic_model` test; 196 passed, 4 pre-existing nvcc failures unrelated to this change

**Maintainer Feedback:**
- @westonpace (maintainer) left a `CHANGES_REQUESTED` review: 7 inline comments plus one top-level design question about whether `from_pydantic_model()` is necessary given `write_dataset()` already handles the core case
- @prrao87 raised a separate, out-of-scope question about continued Pydantic v1 support (not yet addressed)
- See "Code Review Feedback" section below for the full comment-by-comment breakdown

**Status:** Changes Requested — Follow-Up Discussion Posted (see below)

---

## Code Review Feedback

**Reviewer:** @westonpace (Lance maintainer) — `CHANGES_REQUESTED`, 7 inline comments + 1 top-level design question
**Also commented:** @prrao87 — separate scope question about continued Pydantic v1 support, not addressed yet

Worked through each comment individually: verified the underlying claim against the actual code (and, where he referenced a specific function name, against Lance's git history and the sibling `lancedb` project) before making any change, rather than assuming the comment was correct as written.

### Comment-by-Comment

1. **Missing `else` branch in the pydantic check (`types.py`)**
   His point: `_check_for_pydantic` can return false positives (it's a module-name heuristic, not a real `isinstance` check), and if it matches but the object isn't actually a `BaseModel`, the function silently falls through with no `else` and returns `None`.
   Fix: replaced `_check_for_pydantic` with a new `_is_pydantic_base_model()` helper that does a real `isinstance` check directly as the `elif` condition — so a false positive now falls through to the next branch instead of silently returning `None`. Verified with a list of `pydantic.AnyUrl` objects (a real pydantic type, but not a `BaseModel`) — now raises a clear error instead of returning `None`.
   **Fixed & tested.**

2. **Schema inferred from data instead of from the model class (`dataset.py`)**
   His point: referenced `pydantic_to_schema` as precedent. That function doesn't exist in this repo (confirmed by grep) — it's in the sibling `lancedb` project (`lancedb/pydantic.py`). Reproduced the actual bug independently: an `Optional` field that's `None` across an entire batch was getting typed as Arrow's literal `null` type instead of its real type, because schema was inferred from row data, not from the model.
   Fix: added `_pydantic_model_to_schema()`, deriving a real `pa.Schema` from the model class's own field annotations (`model_fields`/`__fields__`), handling `Optional`/`list` wrapping. Verified an `Optional[str]` field now correctly types as nullable `string` even when every value in a batch is `None`, and that appending a row with a real value afterward no longer breaks on schema mismatch.
   **Fixed & tested.**

3. **No error on empty `data` list (`dataset.py`)**
   His point: should raise if `data` is empty.
   Finding: before fixing #2, an empty list silently produced a dataset with **zero columns** — `pa.Table.from_pylist([])` succeeds with an empty schema rather than erroring. After fixing #2, an empty list now produces a correctly-shaped 0-row dataset instead, which is arguably valid on its own — but added the explicit guard anyway as a fail-safe, since that's what he asked for.
   Fix: added `if not data: raise ValueError(...)` at the top of `from_pydantic_model`.
   **Fixed & tested.**

4. **Why prefer `.model_dump()` over `.dict()`, and be consistent with `_into_pyarrow_reader` (`dataset.py`)**
   His point: referenced `_into_pyarrow_reader` as the place to be consistent with. That function also doesn't exist in this repo — it's a real function in `lancedb/table.py`, whose pydantic branch calls a shared `model_to_dict()` helper (also from `lancedb/pydantic.py`).
   Finding: verified empirically that `.dict()` on pydantic v2 emits a `PydanticDeprecatedSince20` warning while `.model_dump()` doesn't, and that pydantic v1 has no `.model_dump()` at all — so the `hasattr` fallback isn't arbitrary, it's the only way to support both without warnings or crashes.
   Fix: extracted the previously copy-pasted logic (identical in `types.py` and `dataset.py`) into one shared `model_to_dict()` helper in `dependencies.py`, matching `lancedb`'s naming. Kept the per-call `hasattr` check rather than `lancedb`'s import-time version branch, since pydantic is an optional/lazy dependency here, unlike in `lancedb` where it's a hard dependency.
   **Fixed & tested.**

5. **`import re` inside the method instead of at module scope (`dataset.py`)**
   Fix: moved to a top-level import (done as a side effect of the #2 fix, since the local import was inside the code already being changed).
   **Fixed & tested.**

6. **`pytest.importorskip("pydantic")` in the test file (`test_dataset.py`)**
   His point: the test file does a bare `from pydantic import BaseModel` — should skip gracefully if pydantic isn't installed.
   Finding: confirmed pydantic is **not** declared anywhere in `pyproject.toml`'s test dependencies (`[project.optional-dependencies].tests` or `[dependency-groups].tests`) — it's only present in the dev venv as a transitive dependency of something else, so this guard genuinely matters.
   Fix: replaced the bare import with `BaseModel = pytest.importorskip("pydantic").BaseModel`. Verified with a throwaway test file that the mechanism produces a clean `skipped` result rather than a collection error when the target module is missing. Note: this is module-level, so if pydantic ever really were unavailable, all ~200 tests in this file would skip rather than just the pydantic-specific ones — the standard tradeoff of file-scope `importorskip`, and what he explicitly asked for.
   **Fixed & tested.**

7. **Accidental `.gitignore` change**
   `reproduce.py` had been added to `.gitignore` — an unrelated leftover from local debugging.
   Fix: reverted.
   **Reverted.**

### Top-Level Design Question (open)

He also questioned whether `from_pydantic_model()` is needed at all, since the PR's own "After" example already shows plain `write_dataset(data, uri)` working directly (thanks to the `_coerce_reader` change). Confirmed by checking `upstream/main`: it has zero pydantic support before this PR, so the core capability is genuinely new and not in question — only the separate classmethod's justification is. Decided to put the keep-vs-drop decision back to him directly rather than presuming an answer, since it's his call as maintainer.

### Verification

Reran the full `python/tests/test_dataset.py` suite after all fixes: **196 passed, 4 failed, 1 skipped** — the 4 failures are the same pre-existing `nvcc`/CUDA-toolkit failures already noted before this review round, unrelated to any of these changes. No regressions introduced.

**Files touched in this round:** `python/python/lance/dependencies.py`, `python/python/lance/types.py`, `python/python/lance/dataset.py`, `python/python/tests/test_dataset.py`, `.gitignore` (reverted).

**Current state:** all fixes above are committed and pushed to the PR branch (`5620779` → `c998e03`).

---

## Follow-Up: Pydantic-to-Arrow Conversion Ownership Discussion

**Context:** After the inline-comment round above, @westonpace left a general (non-inline) comment on the PR clarifying that he'd initially assumed it targeted `lancedb/lancedb` rather than `lance-format/lance` — which explains why his earlier comments referenced `pydantic_to_schema` and `_into_pyarrow_reader`, functions that live in the sibling `lancedb` repo, not this one. That raised a new question: `lancedb` already has a much more complete pydantic-to-arrow conversion (`lancedb/python/python/lancedb/pydantic.py`), supporting nested models (struct), `Enum`, tz-aware `datetime`, and custom `Vector`/`MultiVector` fixed-size-list types — none of which this PR's implementation covers. He asked whether that code should be migrated into `lance-format/lance` to avoid two diverging "dialects" of pydantic-to-arrow conversion, laying out options: proceed as-is (two dialects), move the conversion code into pylance (with embedding-function handling necessarily staying in lancedb, proxying to the moved code), or drop pydantic support from lance entirely. He also flagged @prrao87's earlier, separate question about continued Pydantic v1 support as something to weigh alongside this.

**Analysis:** Read through `lancedb/pydantic.py` (pinned at commit `f8dc2f78`) and categorized its contents:
- **Portable to lance (no embedding-function dependency):** `Vector`, `MultiVector`, `FixedSizeListMixin`, the type-dispatch helpers (`_py_type_to_arrow_type`, `_pydantic_type_to_arrow_type`, `_pydantic_to_field`, `is_nullable`, `_unwrap_optional_annotation`), `pydantic_to_schema()`, `get_extras()`, and `model_to_dict()` — the last of which was already independently reimplemented in this PR's `dependencies.py`, confirming that was the right call.
- **Must stay lancedb-only:** `LanceModel.to_arrow_schema()`'s embedding-function parsing (`parse_embedding_functions`, `EmbeddingFunctionRegistry`) — lance has no concept of embedding functions, so this genuinely doesn't belong at the format level.

**Reply posted:** Recommended moving the pure conversion logic into pylance, with lancedb proxying to it and keeping only embedding-function handling on top. Flagged that completing this requires coordinated PRs across two separate repos (`lance-format/lance` and `lancedb/lancedb`, different maintainer groups) plus a version-coupling story (lancedb pinning a minimum lance version) — too large to fold into this PR. Proposed splitting the work: open a separate tracking issue for the full migration, and keep this PR scoped to the initial scalar/`Optional`/`List` conversion already implemented, with `Vector`/`MultiVector`/struct/`Enum` support following once the migration lands. Recommended keeping the pydantic v1 deprecation question separate too, since lancedb still actively branches on `PYDANTIC_VERSION.major < 2`, making v1 a live support target there — deprecation deserves its own justification rather than riding along with this migration decision.

**Status:** Reply posted to PR; awaiting maintainer response on the proposed minimal-scope + follow-up-issue split.

---

## Follow-Up: Migration Implementation (`pydantic-arrow-migration` branch)

**Context:** Rather than wait idle on the maintainer's response to the scope-split proposal above, started the migration itself on a separate branch (`pydantic-arrow-migration`), so the full conversion logic is ready regardless of which option he picks.

**What was ported**, from `lancedb/pydantic.py` (embedding-function registry code excluded — lance has no concept of embedding functions):
- **`python/python/lance/pydantic.py`** (new, 426 lines) — `Vector`/`MultiVector` fixed-size-list types, nested `BaseModel` → Arrow struct support, `Enum` handling, tz-aware `datetime`, and the full type-dispatch chain (`pydantic_to_schema()`, `_pydantic_to_field`, `is_nullable`, etc.)
- **`dataset.py`** — rewired `from_pydantic_model()` to call `lance.pydantic.pydantic_to_schema()` instead of the original PR's minimal `_pydantic_model_to_schema()`/`_pydantic_annotation_to_arrow_type()` helpers (which only covered scalars, `Optional`, and `List`). Net removal of ~54 lines of now-redundant scope-limited logic.
- **`tests/test_pydantic.py`** (new) — covers struct (nested models), `Enum`, `Vector`/`MultiVector`, tz-aware `datetime`, and a behavior change worth flagging below.

**Behavior change to note:** adopting lancedb's `is_nullable` logic tightens nullability inference — a field with a plain default value but a non-`Optional` type annotation is no longer treated as nullable (lancedb requires `Optional[...]` explicitly). This is a deliberate consistency choice (matches lancedb's semantics exactly, avoiding the two-dialect problem the maintainer raised) but is a behavior change from the original PR #7383 implementation, which inferred nullability more permissively.

**Status:** Implemented and tested locally on `pydantic-arrow-migration` (commits `e22a02a`, `b470b5f`, `67dc8f1`, 2026-07-09). Not yet opened as a PR or tracking issue — still waiting to hear which option @westonpace prefers before proposing this as the actual migration path.

---

## Learnings & Reflections

### Technical Skills Gained

- **Tracing an unfamiliar codebase:** Learned to follow the data path from `write_dataset()` down through `_coerce_reader()` to understand exactly where a new type needs to be handled, rather than guessing at the right insertion point.
- **Optional dependency patterns:** Understood how mature libraries gate optional integrations — the `_PYDANTIC_AVAILABLE` flag and `_check_for_pydantic()` helper follow the exact same pattern used for pandas, polars, and HuggingFace, keeping the core library lean while supporting rich integrations.
- **Pydantic v1/v2 compatibility:** Learned that Pydantic v2 renamed `.dict()` to `.model_dump()` and that a simple `hasattr` check is the idiomatic way to support both versions without version pinning.
- **PyArrow schema inference:** Learned how to derive a PyArrow schema from a Pydantic model class using `model_fields` and `pydantic_to_schema`, and how to build a `RecordBatch` from a list of dicts.
- **Building a Rust-backed Python library:** Got hands-on experience with `uv`, `maturin`, and the full Rust + protobuf build chain. Learned why the first build takes ~30 minutes and subsequent incremental builds are much faster.
- **Pre-commit and ruff:** Learned that ruff auto-reformats on commit and that the correct workflow is to re-stage the auto-fixed files and commit again rather than fighting the formatter.

### Challenges Overcome

- **Build environment from scratch:** Ubuntu 24.04 blocks system-wide `pip` installs and `pre-commit` is not available via `apt` — had to discover and use `uv` and `pipx` as the correct tools. This took significant trial and error before the build succeeded.
- **Finding the right place to make the change:** `write_dataset()` is a thin wrapper; the real logic is in `_coerce_reader()` three layers down. Reading the traceback carefully (the "Must provide schema" error) and grep-searching for that string led directly to the right function.
- **Test failures not caused by my change:** Four tests failed with `PermissionError: [Errno 13] Permission denied: 'nvcc'` — the CUDA compiler is absent in the dev environment. Had to confirm these were pre-existing failures by checking that the failing tests are unrelated to anything modified, before deciding it was safe to submit.
- **Schema inference edge cases:** The initial implementation only supported Pydantic v2's `model_dump()`. Discovered mid-testing that v1 uses `.dict()`, and added the compatibility shim.

### What I'd Do Differently Next Time

- **Read CONTRIBUTING.md first, before writing any code.** I spent time on the implementation before checking the project's conventions for tests, docstrings, and optional dependencies — discovering the existing patterns earlier would have saved a round of refactoring.
- **Grep for the error message immediately.** The "Must provide schema" error string led directly to the right function in under a minute. I should default to searching the codebase with the exact error message as the first investigative step.
- **Check if CUDA tests pass elsewhere before worrying about them.** The nvcc failures were a distraction — checking the repo's CI configuration upfront would have shown immediately that those tests require hardware not available in standard dev environments.

---

## Resources Used

- [Lance CONTRIBUTING.md](https://github.com/lance-format/lance/blob/main/CONTRIBUTING.md) — project conventions for tests, optional dependencies, and PR requirements
- [Pydantic v1/v2 migration guide](https://docs.pydantic.dev/latest/migration/) — reference for `.dict()` → `.model_dump()` rename and compatibility patterns
- [PyArrow documentation — Schema and RecordBatch](https://arrow.apache.org/docs/python/api/datatypes.html) — used for understanding how to construct a `pa.Table` from a list of dicts with an explicit schema
- [uv documentation](https://docs.astral.sh/uv/) — reference for setting up the Python environment without conflicting with Ubuntu's managed Python
- [Original GitHub issue #1106](https://github.com/lance-format/lance/issues/1106) — issue description and discussion that clarified the expected API design
