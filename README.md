# Contribution 1: Working with Pydantic Model Instances

**Contribution Number:** 1  
**Student:** Vinay Surtani  
**Issue:** https://github.com/lance-format/lance/issues/1106  
**Status:** Phase III In Progress

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

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
