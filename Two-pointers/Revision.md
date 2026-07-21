# Two Pointer — Master Revision Sheet (Series Recap)

**Covers:** Full pattern recap across all problems · **Use this as:** the index/flashcard before diving into individual problem sheets (01–08)

---

## The Flowchart (Memorize This)

```
Is it an Array or Linked List question?
   NO  → Two Pointer unlikely, stop here
   YES ↓
Is it ALREADY SORTED, or would sorting help?
   ↓ (either)
Does the question say any of:
   • "rearrange"
   • "remove duplicates"
   • "merge (in-place)"
   • "subarray" / "subarray product or sum"
   ↓ (any one matches)
        OR
Does it ask you to find MORE THAN ONE thing?
   • a pair
   • a triplet
   • a quadruplet
   ↓ (yes)
        → 90%+ CHANCE: Two Pointer applies
```

**The single biggest combined signal:** `Array + Sorting` (already sorted OR sorting would help). This pairing shows up in every problem in the series.
**The second biggest combined signal:** `rearrange / remove / merge` + `"in-place"` (i.e., no extra space allowed).

---

## Every Problem, Checked Against the Flowchart

| Problem | Array? | Sorted/sortable? | Keyword match | Find >1 thing? | Sheet # |
|---|---|---|---|---|---|
| Two Sum (II) | ✅ | ✅ (sortable) | — | ✅ pair | 01 |
| Two Sum — all unique pairs | ✅ | ✅ | "unique pairs" | ✅ pair | 04 |
| 3Sum | ✅ | ✅ (sortable) | — | ✅ triplet | 05 |
| 3Sum Closest | ✅ | ✅ | — | ✅ triplet | 06 |
| Triplets with Smaller Sum | ✅ | ✅ | — | ✅ triplet | 07 |
| Remove Duplicates (Sorted Array) | ✅ | ✅ (given) | "remove duplicates" | — | 02 |
| Squares of Sorted Array | ✅ | ✅ (given) | *hidden:* "merge two sorted arrays" | — | 03 |
| Dutch National Flag (Sort Colors) | ✅ | ❌ (can't sort — no sort fn allowed) | "rearrange" + "in-place" | — | 08 |

**Takeaway:** even Dutch National Flag — the one problem *without* a sorted-array signal — still hits Two Pointer because "rearrange + in-place" alone is a strong enough signal on its own.

---

## Every Pointer-Movement Style Seen So Far

| Movement Type | Behavior | Problems |
|---|---|---|
| **Converge** (`left++`, `right--`) | Pointers start at opposite ends, move toward each other | Two Sum, 3Sum, 3Sum Closest, Triplets Smaller Sum |
| **Co-advance** (`slow++`, `fast++`, both forward) | Both pointers move in the same direction, fast scans ahead of slow | Remove Duplicates, Merge Sorted Arrays (used inside Squares of Sorted Array) |
| **Three-pointer** (`low`, `mid`, `high`) | Three-way partition instead of two-way | Dutch National Flag |

**Rule of thumb:** if the problem is about finding a combination that satisfies a sum/target condition → **converge**. If it's about compacting/merging elements in a single sweep → **co-advance**. If there are exactly 3 categories to sort into → **three-pointer**.

---

## Key Notes (Recurring Across the Whole Series)

- **Time complexity of independent steps = MAX, not sum.** (e.g., sort O(n log n) + scan O(n) → O(n log n) overall, not added.)
- **Nested Two Pointer (e.g., 3Sum) multiplies, not maxes:** outer loop O(n) × inner Two Pointer O(n) = O(n²), because the inner pass runs *once per outer iteration*, not independently alongside it.
- **Sorting is either the enabler or the constraint.** Either the problem gives you sorted data because Two Pointer relies on it (Two Sum, 3Sum family), or it explicitly forbids sorting/extra space and expects an equivalent-effect trick instead (Dutch National Flag's 3-pointer partition, Squares' split-then-merge).
- **Duplicate-skipping is problem-dependent, not automatic.** 3Sum needs it (3 separate skip-checks: `i`, `left`, `right`); 3Sum Closest and Triplets Smaller Sum don't (input guarantees uniqueness or doesn't require distinct outputs).

---

## What's NOT Two Pointer (Important Distinction)

**Sliding Window** looks similar but is a *specialized subset*, not the same pattern:
- Every Sliding Window problem is technically Two Pointer.
- **Not** every Two Pointer problem is Sliding Window.
- Sliding Window = a `low`/`high` pair that defines a **contiguous window** which **expands and shrinks** over the array (both pointers generally move in the *same* direction, growing/shrinking a range) — rather than probing for a target combination or partitioning values.
- Series order going forward: **Two Pointer (done) → Sliding Window (next) → ...**

---

## One-Line Recall (The Whole Series in One Sentence)
**Array + (sorted or sortable) → converge inward for target-sum problems, co-advance for in-place compaction/merging, or use three pointers when partitioning into exactly three categories.**