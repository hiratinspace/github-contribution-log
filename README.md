# Contribution 1: heap tracker: color or mark pointers?

**Contribution Number:** 1  
**Student:** Hirat Rahman Rahi   
**Issue:** https://github.com/pwndbg/pwndbg/issues/3090  
**Status:** Phase I Complete, Phase II Complete

---

## Why I Chose This Issue

As someone passionate about cybersecurity, I felt it's more meaningful to contribute on a tool that is used by real exploit developers. Also it would be a great chance for to deepen my understanding of low-level memory architecture.

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce



### Reproduction Evidence

- **Commit showing reproduction:** (https://github.com/hiratinspace/pwndbg)
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

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



**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

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
