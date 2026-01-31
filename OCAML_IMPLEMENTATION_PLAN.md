# Implementation Plan - Enhanced Testing

Add mock-based tests for Ghost file detection and Property-Based Testing (PBT) for core logic validation.

## User Review Required
> [!IMPORTANT]
> This plan requires adding `qcheck` and `qcheck-alcotest` to the project dependencies. Please ensure these packages are available or installable via opam.

## Proposed Changes

### Refactoring
#### [MODIFY] [lib/quasifind/ghost.ml](file:///Users/atsushi/OCaml/quasifind/lib/quasifind/ghost.ml)
- Extract command output parsing logic into a pure function `parse_lsof_output`.
- This allows testing the parsing logic without running actual `lsof` commands.

### Dependencies
#### [MODIFY] [dune-project](file:///Users/atsushi/OCaml/quasifind/dune-project)
- Add `qcheck` and `qcheck-alcotest` to `depends`.

### New Tests
#### [NEW] [test/test_ghost.ml](file:///Users/atsushi/OCaml/quasifind/test/test_ghost.ml)
- Unit tests for `Ghost.parse_lsof_output`.
- Mock various `lsof` output scenarios (valid, empty, mixed garbage).

#### [NEW] [test/test_props.ml](file:///Users/atsushi/OCaml/quasifind/test/test_props.ml)
- Property-Based Tests using `QCheck`.
- **Properties**:
  - `Size`: `SizeGt n` should return true for all files with size > n.
  - `Time`: `TimeLt n` should return true for all files with age < n.
  - `Parsing`: `Parser.parse (show_expr e) == Ok e` (Round-trip, if `show` is valid DSL).

## Performance Optimization

### 1. Reduce `stat` Calls (Pre-stat Filtering)
- **Goal**: Avoid calling `lstat` on files that will definitely not match the `name` criteria.
- **Approach**:
  - Enhance `Planner` module to extract `name` constraints (e.g., `name =~ /.../` or `name == "..."`) from the AST.
  - In `Traversal`, apply these name filters immediately after `readdir` and **before** `make_entry` (which calls `stat`).

### 2. Intelligent Scheduling (Adaptive Parallelism)
- **Goal**: Reduce Eio fiber creation overhead. Current implementation forks a fiber for *every* directory, then blocks on a semaphore.
- **Approach**:
  - Use an atomic counter (or Eio semaphore inspection if possible) to track active workers.
  - **Logic**:
    - If `active_workers < max_concurrency`: Fork a new fiber for the subdirectory.
    - Else: Process the subdirectory synchronously in the current fiber (DFS fallback).
  - This limits total fibers to roughly `max_concurrency`, drastically reducing memory and scheduler pressure.

### 3. Native Readdir Optimization (C Extension)
- **Goal**: Retrieve file type (`d_type`) during `readdir` to eliminate `lstat` calls for non-matching files.
- **Why**: OCaml's `Sys.readdir` only returns names. `fd` is fast because it uses `d_type` to skip `stat`.
- **Implementation**:
  - Create `lib/quasifind/dirent_stubs.c`:
    - Implement `caml_readdir_with_type` using `opendir`, `readdir`, `closedir` (POSIX).
    - Return `(string * int)` array or list, where int represents file kind (DT_DIR, DT_REG, etc).
  - Create `lib/quasifind/dirent.ml`: OCaml interface.
  - Update `lib/dune` to include `dirent_stubs`.
  - Update `Traversal.ml` to use `Dirent.readdir`.
    - If `d_type` is available and indicates "File", and name doesn't match filter, skip `make_entry` (which does `stat`).
    - If `d_type` indicates "Dir", we still traverse.

## Verification Plan
### Automated Tests
- Run `dune runtest` to verify all new and existing tests pass.

## Phase 3: Low-Level Optimization (Lock-Free & GC-Less)

**Goal**: Maximize parallelism scaling and minimize GC pressure for huge directory trees.

### 1. Lock-Free Work Stealing (Reduce Contention)
Currently, `Work_pool` uses a single variable `queue` protected by a `Mutex`. This causes contention when many workers push/pop simultaneously.
- **Solution**: Implement **Work-Stealing Deque (Chase-Lev)**.
  - Each worker (Domain) has its own local lock-free deque.
  - Pushing to own deque is wait-free/fast.
  - Popping from own deque is fast.
  - Only when local deque is empty does the worker "steal" from others (using `Saturn` library).
- **Dependencies**: Add `saturn` to `dune-project`.

### 2. GC-Less / Batch Readdir (Reduce Allocation & Overhead)
`readdir` currently allocates a `list` node, a `string`, and a `tuple` for *every single file*. For 1M files, this is huge GC pressure.
- **Solution**: **Batched C-Stub with Buffer**.
  - Modify C stub to accept a pre-allocated `Bigarray` (buffer).
  - `readdir_batch` fills this buffer with multiple entries (packed binary format: `[kind:1][len:1][name...]`).
  - OCaml side iterates over this buffer.
  - **Memory Saving**: Only allocate OCaml `string` objects for entries that *pass the filter* (ignore patterns) or are directories we strictly need to traverse.
  - Reduces C call overhead (1 call per batch vs 1 per file).

### 3. Path Buffer Optimization
- Avoid repeated `Filename.concat` (which allocates new strings) during traversal if possible, or assume short-lived minor heap allocation is acceptable given Batch Readdir savings.
