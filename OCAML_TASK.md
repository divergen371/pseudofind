# Test Improvements for Quasifind

## Refactoring
- [x] Refactor `Ghost` module to separate logic from system calls

## New Tests
- [x] Create `test/test_ghost.ml` with mock data
  - [x] Test parsing of `lsof` output
  - [x] Test filtering by root directory
- [x] Create `test/test_suspicious.ml` for Suspicious mode
  - [x] Default rules typecheck
  - [-] [x] Investigating -d option not working
  - [x] Reproduce the issue (script created)
  - [x] Debug DFS logic (verified correct against `find`)
  - [x] Fix Parallel logic (Found bug: depth ignored)
  - [x] Verify fix with `repro_depth.sh`
  - [x] Add regression tests to `test/test_traversal.ml`
  - [x] Comprehensive Option Combination Tests
    - [x] Test Parallel + Hidden/Ignore interactions
    - [x] Test Follow Symlinks (File and Directory behavior)
    - [x] Test Ignore Pattern Pruning
    - [x] Test Max Depth + Symlinks
    - [x] Test Timestamp Preservation
- [ ] Phase 3: Low-Level Optimization (Lock-Free & GC-Less)
  - [x] Add `saturn` dependency
- [x] **Phase 3: Core Parallelism (Multicore)**
  - [x] Implement Work Stealing or Static Partitioning
  - [x] Thread-safe Result Collection
  - [x] C Stubs for `readdir` with d_type (Native Readdir)
  - [x] **Verified Speedup**: Parallel matches `fd` (1.34s vs 1.32s). ~5x faster than `find`.

- [x] **Phase 4: Lazy Stat Optimization & Profiling**
  - [x] Analyze Query Dependencies (`Eval.requires_metadata`)
  - [x] Conditional `lstat`
  - [x] **Critical Fix**: Pre-compile Ignore Regexes (was O(N*M) recompilation)
  - [x] **Verified Speedup**: Reduced overhead significantly.
- [x] Add `qcheck` to dependencies
- [x] Create `test/test_props.ml` for Property-Based Testing
  - [x] PBT for `Size` comparisons (boundary values)
  - [x] PBT for `Time` comparisons

## Performance Optimization
- [x] Create benchmark suite (`bench/`)
- [x] Optimize data generation (Python parallel generator)
- [x] Implement Multicore Parallelism (Static Partitioning with Eio Domains)
- [x] Implement Native Readdir (C Extension for `d_type`)
- [x] Implement Dynamic Work Stealing (Thread-safe Queue & Result Stream)
- [x] Verify speedup (4.14x achieved)
