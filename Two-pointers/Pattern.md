# Two Pointer — Pattern Intro + Two Sum (II)

**Pattern:** Two Pointer · **Difficulty:** Easy · **LeetCode:** Two Sum II (sorted input)

---

## Problem
Given an array and a `target`, find two elements whose sum equals `target`.
Return either their **values** or their **indices**, without extra space.

---

## Recognition Signals
- [x] Array or Linked List? → Array
- [x] Sorted / sortable? → Sortable (if returning values, not indices)
- [x] Keyword match? → "find a pair" (2 elements)
- [x] Space constraint? → "no extra space" mentioned

⚠️ **Value vs Index rule:** if the answer must be **indices** on an **unsorted** array → Two Pointer does NOT apply (sorting destroys index mapping). Only sort if returning **values**, or if array is already sorted.

---

## Core Idea
Two pointers start at opposite ends and move toward each other, using the sorted order to eliminate half the search space every step — like two people climbing a building from opposite ends to meet in the middle instead of one person doing the full round trip.

---

## Key Notes
- **Complexity of independent steps = MAX, not sum.** If step A takes O(n) and step B takes O(n log n), total is O(n log n) — not O(n + n log n). This is why Two Pointer here is O(n log n): sorting (n log n) dominates the O(n) pointer walk.
- **Hash Map space cost is real at scale.** Storing `{value: index}` roughly doubles storage per element. For 1 billion elements, that's ~16GB of RAM just for the map — often more than the machine has. This is *why* interviewers ask for a no-extra-space version.
- Brute force isn't "wrong," it's a required first step — always know the O(n²) solution before optimizing, even if you don't write it out.

---

## Approach Comparison

| Approach | Time | Space | Verdict |
|---|---|---|---|
| Brute Force (nested loop) | O(n²) | O(1) | Too slow |
| Hash Map | O(n) | O(n) | Fast, but uses space |
| Two Pointer ⭐ | O(n log n) | O(1) | Best when "no extra space" required |

---

## Algorithm Steps
1. Sort the array (skip if already sorted).
2. `i = 0`, `j = n - 1`.
3. While `i < j`:
   - `sum = arr[i] + arr[j]`
   - `sum == target` → return answer
   - `sum < target` → `i++`
   - `sum > target` → `j--`
4. If loop ends without match → return "not found".

---

## Pseudocode
```
function twoSum(arr, target):
    sort(arr)
    i, j = 0, n - 1
    while i < j:
        sum = arr[i] + arr[j]
        if sum == target: return (i, j)
        elif sum < target: i++
        else: j--
    return NOT_FOUND
```

---

## Dry Run
`arr = [2, 7, 11, 15]` (sorted), `target = 9`

| Step | i | j | arr[i]+arr[j] | Action |
|---|---|---|---|---|
| 1 | 0 | 3 | 2+15=17 | 17>9 → j-- |
| 2 | 0 | 2 | 2+11=13 | 13>9 → j-- |
| 3 | 0 | 1 | 2+7=9 | Match → return (2,7) |

---

## Gotchas / Edge Cases
- Moving the "wrong" pointer never helps — always move the one that corrects the sum direction.
- Loop must terminate: at least one pointer moves every iteration.
- Stop condition: pointers **meet** or **cross**.
- Sorting costs O(n log n) — that's why total time isn't O(n).

---

## One-Line Recall
**Sort (if allowed) → two pointers from both ends → move the one that fixes the sum → stop when they meet/cross.**