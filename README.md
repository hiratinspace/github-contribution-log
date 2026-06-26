# Contribution 1: heap tracker: color or mark pointers?

**Contribution Number:** 1  
**Student:** Hirat Rahman Rahi   
**Issue:** https://github.com/pwndbg/pwndbg/issues/3090  
**Status:** Phase I Complete, Phase II Complete, Phase III Complete

---

## Why I Chose This Issue

As someone passionate about cybersecurity, I felt it's more meaningful to contribute on a tool that is used by real exploit developers. Also it would be a great chance for to deepen my understanding of low-level memory architecture.

---

## Understanding the Issue

### Problem Description

The heap tracker ('track-heap enable') prints a line for every 'malloc', 'free', and 'realloc' a program makes. The pointers in that report were plain text, so when a program does many allocations it's hard to visually maatch an allocation with the 'free' that releases it. Issue #3090 asks for each pointer to be colored uniquely and consistently. Coloring was already added for 'malloc'/'free' (PR #3176), but the 'realloc' path was left incomplete and buggy.

### Expected Behavior

The same pointer address should always render in the same color across its whole lifetime ('malloc' -> 'realloc' -> 'free'), so an allocation can be paired with its free by eye. 'realloc''s returned pointer should be colored just like 'malloc''s, and 'realloc(ptr, 0)' should print a warning rather than crashing.

### Current Behavior

A successful `realloc` printed its returned pointer **uncolored** (`{ret_ptr:#x}`), so it couldn't be color-matched to a later `free` — defeating the feature for realloc'd chunks.
`realloc(ptr, 0)` raised `AttributeError` (the printer referenced a non-existent `self.freed_pointer`).
The same `realloc(ptr, 0)` branch also leaked tracker state (missing `exit_memory_management()`), leaving the tracker flagged "in progress" and breaking the next allocation.


### Affected Components

`pwndbg/gdblib/ptmalloc2_tracking.py` — the heap-tracker logic (GDB-only; it hooks libc `malloc`/`calloc`/`realloc`/`free` with breakpoints). Specifically the `ReallocEnterBreakpoint` and `ReallocExitBreakpoint` classes.
`pwndbg/commands/ptmalloc2_tracking.py` — the user-facing `track-heap` command wrapper (unchanged, but where the feature is invoked).


---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. Compile a small C program that does `malloc`, `malloc`, `realloc(ptr, n)` (forced to move), and `realloc(ptr, 0)`.
2. Launch it under pwndbg: break before the allocations, run `track-heap enable`, then `continue`.
3. Observe the `[*]`/`[-]` tracker report lines.


### Reproduction Evidence

- **Commit showing reproduction:** (https://github.com/hiratinspace/pwndbg)
- **Screenshots/logs:** [If applicable]
- **My findings:** confirmed both bugs live — the `realloc` return pointer printed with no color, and `realloc(ptr, 0)` aborted with `Python Exception <AttributeError>: 'ReallocEnterBreakpoint' object has no attribute 'freed_pointer'`. I also found the state-leak bug while confirming that subsequent allocations behaved correctly.

---

## Solution Approach

### Analysis

The root cause is that the `realloc` printers diverged from the established `malloc`/`free` pattern. `malloc` colorizes via `colorize_ptr(ret_ptr)`, but `realloc`'s success printer used the raw `{ret_ptr:#x}`. The `realloc(ptr, 0)` branch referenced an attribute (`self.freed_pointer`) that is never assigned — the value lives in the local `freed_pointer` — and it returned without the `exit_memory_management()` call that balances its earlier `enter_memory_management()`.

### Proposed Solution

Route the `realloc` return through the existing `colorize_ptr()` helper (consistent with `malloc`/`free`); use the correct local variable in the `realloc(ptr, 0)` warning; and add the missing `exit_memory_management()` so the tracker isn't left in a broken state.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** 
Issue #3090 asks that the heap tracker (track-heap enable) color pointers uniquely so a given allocation can be visually matched to its later free. Reading the current code, this was already largely delivered by PR #3176, which added Tracker.colorize_ptr() and the PTRS_COLORS palette and wired them into the malloc and free printers. The remaining problem is that the realloc path is incomplete and broken: (1) realloc(ptr, 0) raises AttributeError because the printer references a non-existent attribute, and (2) a successful realloc prints its returned pointer uncolored, so it can't be matched to a later free, defeating the feature's purpose for realloc'd chunks. So the task is to finish and fix the realloc coloring, not implement coloring from scratch.

**Match:** 
colorize_ptr(ptr) in pwndbg/gdblib/ptmalloc2_tracking.py (line ~250) already exists: it assigns a color from PTRS_COLORS round-robin, caches per-address in self.colorized_heap_ptrs, and supports --relative-addresses (rel_addr).
The correct pattern to copy is the malloc printer: AllocExitBreakpoint.stop() (lines 481–482) does ptr_str = self.tracker.colorize_ptr(ret_ptr) then prints with ptr_str.
The free printer (FreeExitBreakpoint, lines 609/633) colorizes in __init__ via self.ptr_str = tracker.colorize_ptr(self.ptr) — same caching guarantees one consistent color per address across its lifetime.
The realloc printers (ReallocEnterBreakpoint line ~517, ReallocExitBreakpoint line ~574) deviate from these patterns — that's the gap to close.



**Plan:** Reproduce: 
build a small C program exercising malloc, free, realloc(ptr, n), and realloc(ptr, 0); run under GDB + pwndbg with track-heap enable; confirm the AttributeError on realloc(ptr, 0) and the uncolored realloc return pointer.
Fix ptmalloc2_tracking.py:517: in ReallocEnterBreakpoint.stop(), change self.tracker.colorize_ptr(self.freed_pointer) to use the local freed_pointer (the attribute is never set; the local from line 504 is the intended value).
Fix ptmalloc2_tracking.py:574: in ReallocExitBreakpoint.stop(), replace the raw {ret_ptr:#x} in the success print with self.tracker.colorize_ptr(ret_ptr), mirroring the malloc printer, so the realloc result shares one color with its eventual free.
Verify consistency: confirm via the repro that a single address keeps one color across malloc → realloc → free, that input vs. output pointers get distinct colors when the address changes, and that --relative-addresses mode still renders correctly.
Update/add tests: ptmalloc2_tracking currently has no test coverage. Add at minimum a unit test for colorize_ptr caching/consistency (no live inferior needed) and, if feasible, a regression test for the realloc path.
Submit PR referencing #3090 and #3176, framed as completing realloc coloring and fixing the realloc(ptr, 0) crash, with before/after terminal output.



**Implement:** https://github.com/hiratinspace/pwndbg/tree/fix/heap-tracker-colorize-realloc (commit `2f9769d4`). During implementation I found a third bug beyond the two planned: the `realloc(ptr, 0)` branch also leaked tracker state, so I added the missing `exit_memory_management()`.

**Review:** change lives in the correct layer (GDB-only logic stays in `gdblib/`); reuses the existing `colorize_ptr()` helper rather than inventing a new mechanism;

**Evaluate:** Verified the tests fail on the unfixed code and pass on the fixed code

---

## Testing Strategy

### Unit Tests

- [ ] The heap tracker requires a live GDB inferior (it hooks libc functions with breakpoints), so a pure no-inferior unit test of `colorize_ptr()` was not practical; coverage is provided via the integration tests below, which assert the observable colored output rather than internal state.

### Integration Tests

- [x] `test_track_heap_colorizes_realloc_pointer` — enables the tracker, runs a program whose `realloc` is forced to move, and asserts the returned pointer is colorized **and** shares the exact same ANSI color as its later `free()`.

### Manual Testing

Ran the tracker live on the fixture and captured colored before/after output:
- **Before (unfixed):** output stopped after the `realloc(ptr, 0)` crash; the realloc return was printed uncolored.
- **After (fixed):** the realloc return is colored and matches the color of its later `free`; `realloc(ptr, 0)` prints a clean warning; the program runs to completion.
- Verified the tests genuinely guard the bug by confirming they FAIL on the unfixed code and PASS on the fixed code (`git stash` the fix, re-run, restore).

---

## Implementation Notes

### Week [III] Progress

Completed the heap-tracker pointer-coloring feature (`track-heap enable`)
requested in issue #3090. The coloring mechanism (`Tracker.colorize_ptr()`,
which caches one color per pointer so the same address always renders the same
color) already existed for `malloc`/`free`, but the `realloc` path was
incomplete and buggy. I:

1. **Colorized realloc's returned pointer** — it was printed as a plain
   `{ret_ptr:#x}` while malloc/free were colored, so a `realloc` result could
   not be visually matched with its later `free`. Routed it through the same
   `colorize_ptr()` helper.
2. **Fixed an `AttributeError` crash** on `realloc(ptr, 0)` — the code
   referenced `self.freed_pointer`, an attribute that is never set (the value
   lives in the local variable `freed_pointer`).
3. **Fixed a tracker state leak** in the same branch — it called
   `enter_memory_management()` without the matching `exit_memory_management()`,
   leaving the tracker permanently flagged as "in progress" and breaking the
   next allocation.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:**
- `pwndbg/gdblib/ptmalloc2_tracking.py` — the three fixes above (the heap
  tracker is GDB-only; this is its logic module).
- `tests/binaries/host/heap_tracker.native.c` — new C test fixture exercising
  malloc/realloc/free and `realloc(ptr, 0)`.
- `tests/library/gdb/tests/heap/test_track_heap.py` — two new regression tests.
- **Key commits:** `2f9769d4` — `fix(heap): colorize realloc pointer and fix realloc(ptr, 0) in
  heap tracker` (single squashed commit; an earlier `542c00c7` was amended to
  fold in a CI lint fix).

- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:**  https://github.com/pwndbg/pwndbg/pull/3984

**PR Description:** Completes the heap-tracker pointer coloring (#3090): the `realloc()` return pointer is now colorized via `colorize_ptr()` like malloc/free, so an allocation can be visually matched with its free. Also fixes two bugs on the `realloc(ptr, 0)` path that the original coloring change left behind — an `AttributeError` from a non-existent `self.freed_pointer`, and a missing `exit_memory_management()` that left the tracker flagged in-progress and broke the next allocation. Adds GDB tests covering the colorized realloc output and the `realloc(ptr, 0)` path. `Closes #3090`.

**Maintainer Feedback:**
- 2026-06-26: Automated Copilot AI review left inline comments on the PR.
- 2026-06-26: All CI checks passing after addressing a `mypy --strict` typing requirement (annotated the new test functions).

**Status:** Awaiting review (all CI checks passing)

---

## Learnings & Reflections

### Technical Skills Gained

- How GDB plugins hook libc functions with enter/finish breakpoints and watchpoints, and how pwndbg layers a debugger-agnostic library over a thin debugger abstraction.

### Challenges Overcome

- **The feature already partly existed** — the real task was finding the gaps (all in `realloc`), not building from scratch. A pickaxe search revealed PR #3176 had added the coloring.

### What I'd Do Differently Next Time

- Run `mypy --strict` on new files *before* opening the PR, not just `./lint.sh`, to catch the CI typing gate locally.

---

## Resources Used

- pwndbg Contributing Guide — https://pwndbg.re/dev/contributing/
- Issue #3090 (this contribution) and prior PR #3176 (which added the original malloc/free coloring)
- pwndbg developer docs (`docs/contributing/`: writing-tests, common-pitfalls, adding-a-command) and the repo's `CLAUDE.md`

