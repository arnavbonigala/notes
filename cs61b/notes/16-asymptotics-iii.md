<!-- Fri, Oct 02, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 16: Asymptotics III

## Overview

This is the third and final lecture of the asymptotics unit, and it is almost entirely worked examples: the goal is pattern recognition, not new theory. The lecture opens by repeating last time's thesis (there is no magic shortcut for runtime analysis, and essentially every problem reduces to turning code into a sum and then collapsing that sum with one of two known closed forms), then applies that machinery to recursive code: a toy doubling recursion `f3` (Θ(2^N)), binary search (Θ(log N), done both intuitively and by exact counting, which motivates three handy Big Theta simplification rules about floors, ceilings, and log bases), and then the big one, mergesort. Mergesort is developed incrementally from selection sort: selection sort is Θ(N²), the `merge` operation is Θ(N), so selection-sorting two halves and merging beats selection-sorting the whole thing, and pushing that idea all the way down gives Θ(N log N) via the "work per level × number of levels" argument. The lecture then adds a short new topic, multi-variable asymptotics (Θ(f + g) = Θ(max(f, g)), runtimes in terms of both N and M), and closes with a careful review of Big O versus Big Theta using quick union tree heights, hammering home that Big O means "less than or equal to" and is *not* mathematically a synonym for "worst case", even though it is constantly used that way. (Logistics mentioned in lecture: Josh's office hours right after class, moved to Wheeler 126; Midterm 1 scores landed in the mid-to-high 60s, below the high-70s target, with the LinkedListDeque constructor question being unexpectedly hard.)

---

## Key Concepts

### 1. The general recipe, restated

Every analysis in this lecture follows the same arc:

1. **Pick a cost model**: choose one operation to count. It must be something that happens a constant number of times per "unit" of real work, so that counting it is proportional to the true runtime.
2. **Count it as a sum**: write out small cases (N = 1, 2, 3, 4), look for the pattern, and express the count as a sum.
3. **Collapse the sum** using one of the two closed forms you are required to know:
   - 1 + 2 + 3 + ... + Q = Q(Q+1)/2 = Θ(Q²)  (sum of first natural numbers)
   - 1 + 2 + 4 + 8 + ... + Q = 2Q − 1 = Θ(Q)  (sum of first powers of 2)
4. **Convert the count to a runtime**: if each counted operation costs constant time, R(N) = Θ(C(N)).

Three legal strategies for step 2 were named: find the exact sum, write out examples, draw pictures. In this class exact counting *or* intuitive analysis are both acceptable; you will not be asked to write mathematical proofs.

### 2. Choosing a cost model is a real decision, and you can get it wrong

This came up repeatedly in Q&A. For `f3`, almost anything works (calls to `f3`, executions of `n <= 1`, additions, or an imagined `print` at the top of the method) because the body is constant-time, so all of these are within a constant factor of each other. Josh's suggested mental trick: **imagine a `System.out.println("hi")` at the top of the method and count how many times "hi" is printed.** That turns a vague "count calls" into something concrete.

For selection sort, the choice matters enormously. Counting **swaps** gives N swaps and would wrongly suggest Θ(N). Counting **comparisons** or **array accesses / items under consideration** gives N + (N−1) + ... + 2 = Θ(N²), which is correct. The lesson: a cost model is only valid if it is proportional to the total work. Swaps undercount because each swap is preceded by a full scan whose cost is invisible to the swap counter.

### 3. The doubling recursion: increase N by 1, double the work

For `f3`, the intuition is structural: the call tree for `f3(N)` contains two complete copies of the call tree for `f3(N-1)`, plus the root. So C(N) = 2·C(N−1) + 1, and the work doubles for each +1 in N. That is exactly the signature of exponential growth, Θ(2^N). The exact count confirms it: level sizes are 1, 2, 4, 8, ..., so C(N) = 1 + 2 + 4 + ... + 2^(N−1), which is the powers-of-2 sum with Q = 2^(N−1), giving 2·2^(N−1) − 1 = 2^N − 1 = Θ(2^N).

A student asked what happens if the calls were `f3(n-1) + f3(n-2)`. Josh noted that is Fibonacci and is explicitly not analyzed in this course.

### 4. Halving arguments give logarithms

Binary search repeatedly discards half the remaining range. The intuition Josh used: imagine a Play-Doh snake of length N; cut it in half, cut that half in half, and so on; how many cuts until you are down to a single molecule? Formally, if C is the number of calls, the final subproblem has size 1 = N / 2^C, so 2^C = N, so C = log₂(N).

Important framing detail: the runtime is expressed in terms of **N = hi − lo + 1**, the number of items *currently under consideration*, not the physical length of the array. You might have a gigantic array but be searching only within a window of a thousand elements.

**Logarithmic time is practically constant.** The lecture's table:

| N | log₂ N | typical runtime |
|---|---|---|
| 100 | 6.6 | 1 ns |
| 100,000 | 16.6 | 2.5 ns |
| 100,000,000 | 26.5 | 4 ns |
| 100,000,000,000 | 36.5 | 5.5 ns |
| 100,000,000,000,000 | 46.5 | 7 ns |

A thousandfold increase in data costs you about 1.5 nanoseconds. As Josh put it, you will hit memory limits long before compute limits.

### 5. Three Big Theta simplification rules

Exact counting for binary search produces the ugly C(N) = ⌊log₂(N)⌋ + 1. Three properties let you clean that up (proofs were on a skipped slide):

- ⌊f(N)⌋ = Θ(f(N))  (floor does not change order of growth)
- ⌈f(N)⌉ = Θ(f(N))  (ceiling does not change order of growth)
- log_P(N) = Θ(log_Q(N))  (**log base is irrelevant** to order of growth, because changing base multiplies by a constant)

Because the base is irrelevant, we write simply **Θ(log N)**, dropping both the base and (for aesthetics) the parentheses around N.

### 6. Building mergesort out of selection sort: the "free lunch"

This was the centerpiece. Using **arbitrary units (AU)** where selection-sorting K items costs ~K² AU, and merging K total items costs ~K AU, for N = 64:

| Strategy | Cost in AU |
|---|---|
| Selection sort the whole array | 64² = ~4096 |
| Split in 2, selection sort each half, merge | 2·32² + 64 = 2048 + 64 = ~2112 |
| Split into 4, selection sort each, two layers of merges | 4·16² + (2·32 + 64) = 1024 + 128 = ~1152 |
| Merge all the way down (mergesort) | 64 per level × 6 levels = ~384 |

The one-layer and two-layer versions are **still Θ(N²)** (the dominant term is still quadratic: N + 2·(N/2)² < N², but it is a constant-factor improvement, not an asymptotic one). Only when you push the recursion all the way to base case 1 do you escape quadratic entirely.

*Why* it helps, in Josh's words: selection sort wastes information. In each pass it compares every remaining item and then throws away everything it learned except for a single answer ("which was smallest"). Merging preserves and reuses the ordering work already done by the subproblems.

### 7. The level-by-level argument for Θ(N log N)

This is the single most reusable technique in the lecture, and it recurs for quicksort, for exam toy problems, and for the discussion worksheet problems named "forestry" and "assisted forestry".

- **Work per level is Θ(N).** Top level: one merge of N items = N. Second level: two merges of N/2 items each = N total. Third level: four merges of N/4 items = N total. Every level does ~N work.
- **Number of levels is log₂ N.** The subproblem size halves each level until it reaches 1, which is the same halving count as binary search.
- **Total = (work per level) × (number of levels) = Θ(N log N).**

The exact counting version is tedious and was omitted (it is in the textbook). The recurrence-relation version was shown as "extra" and only works cleanly when N = 2^k; generalizing requires separately finding Big O and Big Omega bounds.

### 8. Linearithmic versus quadratic is an enormous practical gap

For N = 1,000,000, log N is only about 20. Josh's illustration: on an old slow machine, selection sorting a million items might take ~12 days, while mergesort takes ~20 seconds, with no downside. The summary slide's framing: **N² vs N log N is an enormous difference; going from N log N to N is nice but not a radical change.**

### 9. Multi-variable asymptotics (new topic, brief)

Key fact: **Θ(f(N) + g(N)) = Θ(max(f(N), g(N)))**. Only the dominant term survives, which is just the "drop lower-order terms" rule in another costume. Examples from lecture:

- Θ(N³ + 3N⁴) = Θ(N⁴)
- Θ(N + √N) = Θ(N)
- Θ(N log N + N) = Θ(N log N)

Where this matters: when a runtime depends on two independent sizes. Merging an array of size N with an array of size M is **Θ(N + M)**. You might be tempted to write Θ(max(N, M)); that is the same thing, so the sum form is fine and is preferred. Further examples:

- Θ(NM + M) = Θ(NM) (if N is a positive size, NM dominates M)
- Θ(E log V) stays Θ(E log V) (V grows faster than log V, so you cannot drop things carelessly here; this notation reappears later in the course for graph algorithms)

Honest caveat Josh gave: he did **not** formally define multi-variable asymptotics, and neither does almost anyone else, because doing it rigorously is genuinely hard. Computer scientists here behave like physicists and do math without a rigorous foundation underneath.

### 10. Big O versus Big Theta, revisited

The analogy: **Big O is like ≤, Big Theta is like =.** If we have `=`, why do we also have `≤`? Same reason.

Using quick union trees of N connected items (height ranges from 1 to N−1, dramatically different depending on shape):

- A **flat** tree (all N−1 items children of one root): H = Θ(1).
- A **spindly** tree (a single chain): H = Θ(N). Note a spindly tree is reachable with plain quick union (arbitrary tie-breaking) but *not* with weighted quick union.

All of these statements are simultaneously **true**:

- A. Worst case quick union tree height is Θ(N). ("is equal to linear")
- B. Quick union tree height is O(N). ("is less than or equal to linear")
- C. Quick union tree height is O(N²). ("is less than or equal to quadratic")

B and C are blanket statements over all trees; they are true but weak. The **hotel analogy**: "the most expensive room is $639/night" is more informative than "every room is ≤ $639/night", because the first pins down the actual maximum while the second only bounds it.

So quick union tree height is all four of: O(N), Θ(1) in the best case, Θ(N) in the worst case, O(N²). The middle two are the informative ones.

**Big O is NOT mathematically the same thing as "worst case"**, although it is very often (and incorrectly) used as an exact synonym for it. It is more precise to say "Θ(N) in the worst case" than "O(N)".

### 11. Why Big O is still useful

- It allows simple blanket statements: "binary search is O(log N)" instead of "binary search is Θ(log N) in the worst case".
- Sometimes nobody knows the exact runtime, so O gives an upper bound. Example: finding the shortest route visiting all world cities is O(2^N) (under unlisted assumptions); there might be something faster, but nobody knows one.
- Big O proofs are easier to write than Big Theta proofs. For mergesort you can round N up to the next power of 2, which is a little beyond the scope of this course.
- If you show a function is both O(N²) and Ω(N²), then it is Θ(N²), exactly as a ≤ b and b ≤ a give a = b.

---

## Definitions

- **Cost model**: a chosen operation whose count is used as a proxy for runtime. Valid only if the total work is proportional to that count.
- **C(N)**: in this lecture's notation, the count of the chosen cost-model operation on input size N (an exact function, not asymptotic).
- **R(N)**: the runtime on input size N. If each counted operation is constant time, R(N) = Θ(C(N)).
- **Sum of first natural numbers**: 1 + 2 + 3 + ... + Q = Q(Q+1)/2 = Θ(Q²).
- **Sum of first powers of 2**: 1 + 2 + 4 + 8 + ... + Q = 2Q − 1 = Θ(Q), valid when Q is a power of 2. Read as "twice the last term, minus one".
- **Recurrence relation**: a definition of C(N) in terms of smaller inputs plus a base case, e.g. C(1) = 1, C(N) = 2C(N−1) + 1. **Explicitly out of scope for 61B**; shown only for interest.
- **Binary search**: search of a sorted array that repeatedly examines the middle element of the live range and discards the half that cannot contain the target. Idea published 1946; first correct implementation 1962; a bug in Java's own binary search was discovered in 2006 (Josh Bloch's post, "nearly all binary searches and mergesorts are broken", an overflow when `lo + hi` is large).
- **N for binary search**: N = hi − lo + 1, the number of items under consideration.
- **Selection sort**: find the smallest unfixed item, swap it to the front, fix it in place, then selection sort the remaining unfixed items. Θ(N²).
- **Merge**: given two sorted arrays, produce one sorted array containing all their items by repeatedly copying the smaller of the two front items. Θ(N) in the total number of items, using array writes as the cost model (exactly N writes).
- **Mergesort**: if the array has size 1, return; otherwise mergesort the left half, mergesort the right half, and merge the results. Worst case Θ(N log N).
- **Arbitrary unit (AU)**: a made-up unit of time used in lecture for comparing idealized costs (selection sort of K items ≈ K² AU, merge of K items ≈ K AU).
- **Linearithmic**: Θ(N log N).
- **Flat tree**: a tree of height Θ(1) (here, N−1 items all directly under one root).
- **Spindly tree**: a tree of height Θ(N) (here, a single chain).
- **Big O (informally, as used here)**: an upper bound on order of growth; "less than or equal to". A blanket statement over all cases unless a case is specified.
- **Big Theta (informally, as used here)**: an exact order of growth; "equal to".
- **Big Omega (informally)**: a lower bound on order of growth; "greater than or equal to". O plus Ω with the same f gives Θ.
- **Θ(f + g) = Θ(max(f, g))**: when summing two functions, only the faster-growing one affects the order of growth.

---

## Worked Examples

### Example 1: The toy doubling recursion `f3`

```java
public static int f3(int n) {
    if (n <= 1)
        return 1;
    return f3(n - 1) + f3(n - 1);
}
```

**What it does.** For n ≤ 1 it returns 1. Otherwise it calls itself twice on n−1 and adds the results. (It computes 2^(n−1) for n ≥ 1, but the return value is beside the point; we care about how many calls happen.)

**Approach 1: intuition.** Draw the call tree.

- n = 1: just `f3(1)`. 1 call.
- n = 2: `f3(2)` at the root with two children `f3(1)`, `f3(1)`. 1 + 2 = 3 calls.
- n = 3: `f3(3)` with two `f3(2)` children, each of which has two `f3(1)` children. 1 + 2 + 4 = 7 calls.
- n = 4: `f3(4)` with two `f3(3)` subtrees, each of which is the 7-call tree above. 1 + 2 + 4 + 8 = 15 calls.

The pattern: the tree for N contains two complete copies of the tree for N−1. **Every time we increase N by 1, we double the work.** Therefore R(N) ∈ Θ(2^N). (In the live poll, 45% of students picked 2^N and were correct; the other popular answer was N.)

**Approach 2: exact counting.** Cost model: number of calls to `f3`, written C(N). (Equivalently: count executions of `n <= 1`, or count an imaginary `print` at the top of the method. All give the same answer up to a constant.)

Level k of the call tree (0-indexed from the root) has 2^k nodes, and the tree has N levels (from `f3(N)` down to `f3(1)`). So:

C(N) = 1 + 2 + 4 + ... + 2^(N−1)

This is literally the powers-of-2 sum. The final term is Q = 2^(N−1). Apply 1 + 2 + 4 + ... + Q = 2Q − 1:

C(N) = 2 · 2^(N−1) − 1 = 2^N − 1

Drop the lower-order −1: C(N) = Θ(2^N). Since the work done inside each call is constant (one comparison, one addition, one return), **R(N) = Θ(2^N)**.

Sanity check against the hand counts: N = 4 gives 2⁴ − 1 = 15. Matches.

**Approach 3: recurrence relations (explicitly out of scope).** C(1) = 1, C(N) = 2C(N−1) + 1. More technical to solve; not done in this course.

**Call-stack / frame reasoning in words (extra context).** It is worth separating *total calls* from *simultaneous frames*. Each invocation of `f3` gets its own frame holding its own local `n`, and because `n` is an `int` (a primitive), each frame holds its own independent copy of the value, not a reference to a shared box. The left `f3(n-1)` fully completes and its frames are popped before the right `f3(n-1)` is even started, so at any instant the stack is at most N frames deep even though 2^N − 1 frames are created over the program's lifetime. This is why the *time* is exponential but the *space* is only linear.

### Example 2: Binary search

```java
static int binarySearch(String[] sorted, String x, int lo, int hi) {
    if (lo > hi) return -1;
    int m = (lo + hi) / 2;
    int cmp = x.compareTo(sorted[m]);
    if (cmp < 0) return binarySearch(sorted, x, lo, m - 1);
    else if (cmp > 0) return binarySearch(sorted, x, m + 1, hi);
    else return m;
}
```

**What it does.** `lo` and `hi` bracket the live range, inclusive. If the range is empty (`lo > hi`), the item is absent and we return −1. Otherwise compute the midpoint `m` with integer division (which **rounds down**, so with `lo = 0, hi = 9`, `m = 4` rather than 4.5). Compare the target to `sorted[m]`: if the target is smaller, recurse on `[lo, m-1]`; if larger, recurse on `[m+1, hi]`; if equal, return `m`.

Josh noted this is harder to get right than it looks: the idea was published in 1946, the first correct implementation came in 1962, and Java's built-in version had a bug found in 2006. This very code is believed to contain that bug: if `lo + hi` is large enough it overflows `int`, so `(lo + hi) / 2` goes wrong. (The standard fix, not stated in lecture: compute `lo + (hi - lo) / 2`. *(extra context)*)

**Trace from the demo, searching for 33.** Range is indices 0..14.
- m = (0 + 14)/2 = 7. `sorted[7]` is 53. 33 < 53, so recurse left on 0..6.
- m = (0 + 6)/2 = 3. `sorted[3]` is 25. 33 > 25, so recurse right on 4..6.
- m = (4 + 6)/2 = 5. `sorted[5]` is 43. 33 < 43, so recurse left on 4..4.
- m = 4. `sorted[4]` is 33. Found, return 4.

Four items examined instead of fifteen.

**Trace of an unsuccessful search for 49**, showing the rounding-down rule on an even-length range 0..9:
- m = (0+9)/2 = 4 (4.5 rounded down). `sorted[4]` is 50. 49 < 50, recurse on 0..3.
- m = (0+3)/2 = 1 (1.5 rounded down). `sorted[1]` is 20. 49 > 20, recurse on 2..3.
- m = 2. Value is 30. 49 > 30, recurse on 3..3.
- m = 3. Value is 40. 49 > 40, recurse on 4..3, i.e. `lo > hi`. Return −1.

Two Q&A points from the demo: duplicates are fine (if you land on a matching item you are done, and if not, the sorted property still tells you which side to discard), and if the array were *not* sorted you would have no information at all and would be forced to examine everything in the worst case.

**Approach 1: intuitive.** Goal: worst case runtime in terms of N = hi − lo + 1. The live range sizes are N, ≈N/2, ≈N/4, ≈N/8, ..., halving until size 1. If C is the number of calls, solve 1 = N / 2^C, giving 2^C = N, so C = log₂(N). Hence **Θ(log N)**. (75-80% of the class got this.)

**Approach 2: exact counting (presented as a pre-recorded bonus, still in scope since it uses the same techniques).** Cost model: number of calls to `binarySearch`. For N = 6, the worst case path visits ranges of size 6, then 3, then 1: **3 calls**. Tabulating:

| N | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
|---|---|---|---|---|---|---|---|---|---|----|----|----|----|
| C(N) | 1 | 2 | 2 | 3 | 3 | 3 | 3 | 4 | 4 | 4 | 4 | 4 | 4 |

The value jumps exactly at N = 1, 2, 4, 8, which gives the closed form:

**C(N) = ⌊log₂(N)⌋ + 1**

Each call does constant work, so R(N) = Θ(⌊log₂(N)⌋ + 1). That f(N) is far too complicated, so apply the simplification rules: floors do not matter, log base does not matter, and the +1 is lower order. Result: **R(N) = Θ(log N)**.

**Recurrence version (out of scope).** Counting string comparisons: C(0) = 0, C(1) = 1, C(N) = 1 + C((N−1)/2). This can be shown to be Θ(log N), but solving it is beyond the scope of the class.

### Example 3: Selection sort, Θ(N²)

Reconstructed from the lecture's description (the lecture showed the trace, not the code):

```java
public static void selectionSort(int[] a) {
    for (int i = 0; i < a.length - 1; i += 1) {
        int smallestIndex = i;
        for (int j = i + 1; j < a.length; j += 1) {   // scan all unfixed items
            if (a[j] < a[smallestIndex]) {
                smallestIndex = j;
            }
        }
        int temp = a[i];                               // swap, not shift
        a[i] = a[smallestIndex];
        a[smallestIndex] = temp;
    }
}
```

**Trace from lecture**, starting with `6 3 7 2 8 1` (N = 6), where the prefix in bold is "fixed":
- Smallest unfixed is 1 at the end. **Swap** it with the 6: `1 3 7 2 8 6`.
- Smallest unfixed is 2. Swap with the 3: `1 2 7 3 8 6`.
- Smallest unfixed is 3. Swap with the 7: `1 2 3 7 8 6`.
- Smallest unfixed is 6. Swap with the 7: `1 2 3 6 8 7`.
- ...and so on.

**Why swap instead of shifting?** You could remove the minimum and scoot everything right of it over by one, but shifting costs Θ(N) per step in real memory, while a swap is constant time. You can physically move a card to the front of your hand; you cannot physically slide bits. And the displaced element's old position does not matter, since that slot is still unfixed and will be sorted later.

**Runtime analysis.** Choosing **swaps** as the cost model gives N swaps and would wrongly suggest Θ(N); this is the lecture's worked example of a *bad* cost model, since each swap hides a full scan. Choosing **items examined** (or comparisons) gives:

N + (N−1) + (N−2) + ... + 2

which is the first-natural-numbers sum minus a constant, so **Θ(N²)**. (The slide wrote the same sum in the other direction, 2 + 3 + 4 + ... + N.) Counting comparisons instead gives (N−1) + (N−2) + ... + 1, also Θ(N²).

In AU terms: N = 6 costs ~36 AU, N = 64 costs ~4096 AU.

### Example 4: The merge operation, Θ(N)

```java
/** Merges sorted arrays a and b into a new sorted array. */
public static int[] merge(int[] a, int[] b) {
    int[] result = new int[a.length + b.length];
    int i = 0, j = 0, k = 0;
    while (i < a.length && j < b.length) {
        if (a[i] < b[j]) {
            result[k] = a[i];
            i += 1;
        } else {
            result[k] = b[j];
            j += 1;
        }
        k += 1;
    }
    while (i < a.length) { result[k] = a[i]; i += 1; k += 1; }
    while (j < b.length) { result[k] = b[j]; j += 1; k += 1; }
    return result;
}
```

**Trace from the lecture demo.** `a = [2, 3, 6, 10, 11]`, `b = [4, 5, 7, 8]`.

Think of two pointers, one into each array, each sitting on that array's smallest not-yet-copied item. At each step you compare the two pointed-at values, copy the smaller into the output, and advance only that pointer.

- Front items are 2 and 4. Copy **2**, advance `i`.
- Front items are 3 and 4. Copy **3**, advance `i`.
- Front items are 6 and 4. Copy **4**, advance `j`.
- Front items are 6 and 5. Copy **5**, advance `j`.
- Front items are 6 and 7. Copy **6**, advance `i`.
- Front items are 10 and 7. Copy **7**, then **8**, exhausting `b`.
- `b` is empty, so copy the rest of `a`: **10, 11**. This is safe without any further comparisons, because `a` is sorted and everything left in it is at least as large as everything already written.

Result: `2 3 4 5 6 7 8 10 11`. Two things to observe about it, as drawn out in lecture: it is **sorted**, and it contains **all the original numbers**.

**Runtime.** Cost model: array writes into the target. Merge performs **exactly N writes**, where N is the total number of items. So merge is **Θ(N)**. (Comparisons also work as a cost model and give the same answer, but writes are cleaner because they also capture the tail-copy loops, which involve no comparisons at all.)

### Example 5: Mergesort, Θ(N log N)

```java
public static int[] mergeSort(int[] a) {
    if (a.length <= 1) {                 // base case: already sorted
        return a;
    }
    int mid = a.length / 2;
    int[] left  = Arrays.copyOfRange(a, 0, mid);
    int[] right = Arrays.copyOfRange(a, mid, a.length);
    return merge(mergeSort(left), mergeSort(right));   // sort halves, then merge
}
```

**The incremental derivation (N = 64, in AU).**

1. Plain selection sort: 64² = ~4096 AU.
2. **One layer of merges**: selection sort each half (2 × 32² = ~2048) then merge (~64). Total ~2112 AU. Faster, because N + 2(N/2)² < N², but still Θ(N²).
3. **Two layers of merges**: selection sort four pieces of 16 (4 × 256 = ~1024), merge pairs (2 × ~32 = ~64), merge the two halves (~64). Total ~1152 AU. Still Θ(N²).
4. **Merge all the way down**: there is no selection sort left at all, because the base case is a single item, which is trivially sorted. Every level does ~64 AU of merging, and there are log₂(64) = 6 levels, so ~384 AU.

Note on step 3: our `merge` takes exactly two arrays, so four sorted pieces require two pairwise merges and then one more merge, not a single four-way merge. A four-way merge is possible but is a different algorithm that we did not design.

**The level-by-level argument in general.** Picture the recursion tree with the root labeled N, two children labeled N/2, four grandchildren labeled N/4, eight labeled N/8, and so on.

- Root level: one merge of N items = **N** work.
- Next level: 2 merges of N/2 items each = **N** work total.
- Next level: 4 merges of N/4 items each = **N** work total.
- In general, every level merges N items in total, so every level is **Θ(N)**.
- Number of levels = how many times you can halve N before reaching 1 = **log₂ N**.

Total runtime = Θ(N) per level × log₂ N levels = **Θ(N log N)**.

The exact-count derivation is tedious and was omitted (see textbook). As "extra", a recurrence formulation counting calls to mergesort plus array writes was shown, with the caveat that it only works cleanly for N = 2^k and generalizes only at the expense of separately finding Big O and Big Omega bounds.

**Why this matters.** Josh's framing: on an old slow computer, selection sorting a million items might take ~12 days, while mergesort takes ~20 seconds, with no downside. For N = 1,000,000, log N is only ~20, so N log N is nearly as good as N and vastly better than N².

### Example 6: Two-variable asymptotics

```java
// a has length N, b has length M, both sorted
int[] merged = merge(a, b);
```

**Runtime: Θ(N + M).** The merge does exactly N + M array writes, and N and M are independent (the arrays may be radically different sizes), so you cannot collapse them into a single variable.

Someone might answer Θ(max(N, M)) on the reasoning that only the larger survives. That is **the same answer**, by the rule Θ(f + g) = Θ(max(f, g)). The lecture's guidance: if you are ever tempted to write a max, writing the sum is fine.

More applications of the same rule:

```java
// Suppose a nested loop over two collections, plus a single pass over one of them:
for (int i = 0; i < N; i += 1)
    for (int j = 0; j < M; j += 1)
        doConstantWork();
for (int j = 0; j < M; j += 1)
    doConstantWork();
```

Runtime is Θ(NM + M) = **Θ(NM)**, because for positive N the NM term always dominates M.

The lecture also previewed Θ(E log V), which appears later for graph algorithms, noting that V grows faster than log V (so, for instance, you would not be able to simplify an E·log V term by pretending log V is comparable to V).

### Example 7: Quick union tree heights and Big O versus Big Theta

Not code, but the lecture's vehicle for the Big O discussion. With N = 5 connected items:

- **Flat**: node 0 is the root with 1, 2, 3, 4 all as direct children. H = 1, so H(N) = **Θ(1)**.
- **Spindly**: 0 → 1 → 2 → 3 → 4, a single chain. H = 4, so H(N) = **Θ(N)**.

Heights across all quick union trees of N items range from 1 to N−1. The difference is dramatic in practice: on a spindly tree, `isConnected(0, N-1)` takes linear time, because it must walk the entire chain. Note that plain quick union (arbitrary choice of which root to attach) can produce the spindly tree, but **weighted** quick union cannot.

Now evaluate the three statements:

- "Worst case quick union tree height is Θ(N)." **True.** A worst case (spindly) tree's height grows *exactly* linearly.
- "Quick union tree height is O(N)." **True.** This is a blanket statement over all quick union trees, and they all have height linear or better.
- "Quick union tree height is O(N²)." **True.** All of them have height quadratic or better. Annoying to say, but correct.

All three hold. The informative ones are "Θ(1) in the best case" and "Θ(N) in the worst case"; the O statements are correct but weak, exactly as "every hotel room is ≤ $639/night" is weaker than "the most expensive room is $639/night".

---

## Common Pitfalls

1. **Picking a cost model that is not proportional to the work.** Counting swaps in selection sort gives N and the wrong answer Θ(N). A cost model must be "deeply embedded" in the work, not a side effect of it.
2. **Forgetting to identify the final term Q of the sum.** The whole exact-count for `f3` hinges on recognizing that the last term of 1 + 2 + 4 + ... is **2^(N−1)**, not 2^N. (The in-lecture poll for this was broken, so the slide answer 2^(N−1) is the one to remember.)
3. **Misapplying the powers-of-2 formula.** 1 + 2 + 4 + ... + Q = **2Q − 1**, that is, twice the *last term* minus one. It is not 2^Q and it is not Q².
4. **Writing Θ(2^N − 1) or Θ(⌊log₂ N⌋ + 1) as a final answer.** Simplify: drop lower-order terms, drop floors and ceilings, drop the log base. Final answers should be clean: Θ(2^N), Θ(log N).
5. **Thinking the log base matters.** log₂ N, log₁₀ N, and ln N all have the same order of growth. Write Θ(log N).
6. **Confusing "N" with the array length in binary search.** Here N = hi − lo + 1, the number of items under consideration.
7. **Believing a constant-factor improvement changes the order of growth.** The one-layer and two-layer merge hybrids are genuinely faster (2112 and 1152 AU vs 4096) but remain Θ(N²). Only recursing all the way down gives Θ(N log N).
8. **Counting the number of merge levels as N instead of log N**, or counting the work per level as log N instead of N. Keep the two factors straight: ~N work per level, log N levels.
9. **Treating Big O as a synonym for worst case.** Big O is an upper bound, full stop. "Θ(N) in the worst case" is the precise statement; "O(N)" is a weaker blanket claim that happens to be true. Many sources, and informal speech in this very course, blur this.
10. **Thinking a weaker Big O bound is false.** "Quick union height is O(N²)" is true, just uninformative. On a "which are true" question, do not reject a bound merely for being loose.
11. **Trying to use recurrence relations on an exam.** Recurrence relations are explicitly out of scope for 61B. Use intuition or exact counting.
12. **Collapsing two independent variables.** With arrays of size N and M, the answer is Θ(N + M); you cannot assume one dominates unless you are told so.
13. **Assuming a correct idea yields correct code.** Binary search is the cautionary tale: intuitive in 1946, correctly implemented in 1962, still buggy in Java's standard library in 2006 because of an integer overflow in `(lo + hi) / 2`.

---

## Likely Exam Points

**1. Analyzing a branching recursion.**

*Q:* Give the order of growth of the runtime of the following, in terms of N.
```java
public static int g(int n) {
    if (n <= 1) return 1;
    return g(n - 1) + g(n - 1) + g(n - 1);
}
```
*A:* Each call spawns three children, so level k has 3^k calls and there are N levels. C(N) = 1 + 3 + 9 + ... + 3^(N−1) = (3^N − 1)/2, so **Θ(3^N)**. Intuitively: increasing N by 1 triples the work. (The lecture's own `f3`, with two calls, is Θ(2^N).)

**2. Recognizing the halving pattern.**

*Q:* What is the worst case order of growth of the runtime of this method, in terms of N = the length of `arr`?
```java
public static int count(int[] arr, int lo, int hi) {
    if (lo > hi) return 0;
    int m = (lo + hi) / 2;
    return 1 + count(arr, lo, m - 1);
}
```
*A:* Each call discards everything from `m` onward, so the range size roughly halves every call. Number of calls C satisfies 1 = N/2^C, so C = log₂ N, and each call does constant work: **Θ(log N)**.

**3. Applying the two required sums.**

*Q:* State both sums you are expected to know, with closed forms and orders of growth.
*A:* 1 + 2 + 3 + ... + Q = Q(Q+1)/2 = Θ(Q²). And 1 + 2 + 4 + 8 + ... + Q = 2Q − 1 = Θ(Q).

**4. Simplifying an exact count into Big Theta.**

*Q:* Simplify Θ(⌈log₇(N)⌉ + 5).
*A:* **Θ(log N)**. Ceilings do not change order of growth (⌈f(N)⌉ = Θ(f(N))), log base does not change order of growth (log_P N = Θ(log_Q N)), and the +5 is a lower-order term.

**5. Mergesort's level-by-level argument.**

*Q:* Explain, without solving a recurrence, why mergesort is Θ(N log N).
*A:* At the top level, one merge handles N items: Θ(N) work. At the next level, two merges handle N/2 items each, again N items total: Θ(N). This holds at every level, since the subproblems at any level partition the original array. The recursion halves the subproblem size each level and stops at size 1, so there are log₂ N levels. Total = Θ(N) × log₂ N = **Θ(N log N)**.

**6. Merge's runtime and cost model.**

*Q:* Two sorted arrays of total length N are merged. What is the runtime, and what cost model justifies it?
*A:* **Θ(N)**, using array writes into the target as the cost model: merge performs exactly N writes. (Comparisons also work and give the same order.)

**7. Why hybrid schemes stay quadratic.**

*Q:* You split an array in half, selection sort each half, and merge. Is the result asymptotically faster than plain selection sort?
*A:* No. The cost is N + 2(N/2)² = N + N²/2, which is still **Θ(N²)**. It is faster by a constant factor (~2112 vs ~4096 AU at N = 64) but the order of growth is unchanged. You need merges all the way down to reach Θ(N log N).

**8. Selection sort with a trap cost model.**

*Q:* A student claims selection sort is Θ(N) because it performs only N swaps. What is wrong, and what is the real answer?
*A:* Swaps are an invalid cost model: each swap is preceded by a scan of all remaining unfixed items, and that scan's cost is invisible to a swap counter. Counting items examined gives N + (N−1) + ... + 2 = **Θ(N²)**.

**9. Big O versus Big Theta truth values.**

*Q:* A quick union tree of N items has height between 1 and N−1. Which of the following are true? (i) height is Θ(N), (ii) worst case height is Θ(N), (iii) height is O(N), (iv) height is O(N²), (v) best case height is Θ(1).
*A:* (ii), (iii), (iv), (v) are **true**. (i) is **false** as a blanket statement, because a flat tree has height Θ(1), so the height does not grow exactly linearly in all cases. Statements (ii) and (v) are the most informative; (iii) and (iv) are true but weak upper bounds.

**10. Big O versus "worst case" as concepts.**

*Q:* Is Big O the same thing as worst case? Why is "Θ(N) in the worst case" preferred over "O(N)"?
*A:* No. Big O is an upper bound on order of growth, like ≤; worst case refers to a particular input family. "Θ(N) in the worst case" pins down the exact growth of the worst case, whereas "O(N)" only bounds all cases from above and would still be true if the real answer were Θ(log N). Compare: "the most expensive room is $639" is more informative than "every room is ≤ $639".

**11. Why Big O is still worth having.**

*Q:* Give two reasons Big O is useful despite being less informative.
*A:* It permits simple blanket statements ("binary search is O(log N)") without case qualifiers; it lets us state upper bounds when the true runtime is unknown (shortest route through all world cities is O(2^N), but a faster algorithm may exist); and Big O proofs are easier than Big Theta proofs. Also, O(f) plus Ω(f) together give Θ(f).

**12. Two-variable runtime.**

*Q:* `merge` is called on a sorted array of length N and a sorted array of length M. Give the runtime. Also simplify Θ(NM + M + N).
*A:* **Θ(N + M)** (equivalently Θ(max(N, M)); the sum form is preferred). And Θ(NM + M + N) = **Θ(NM)**, since NM dominates both M and N.

**13. Conceptual: why mergesort beats selection sort.**

*Q:* In one or two sentences, why is mergesort so much faster than selection sort?
*A:* Selection sort wastes information: each pass compares all remaining items and then discards everything learned except which single item was smallest. Mergesort preserves the ordering work done on subproblems and combines two already-sorted results in linear time, so no comparison's result is thrown away.

---

## Summary

- There is no magic shortcut for asymptotic analysis. The workflow is: pick a cost model, write out small cases, express the count as a sum, collapse the sum, convert to a runtime.
- The two sums you must know: 1 + 2 + ... + Q = Q(Q+1)/2 = Θ(Q²), and 1 + 2 + 4 + ... + Q = 2Q − 1 = Θ(Q).
- Cost models are a real choice. For selection sort, counting swaps gives the wrong answer (Θ(N)); counting items examined gives the right one (Θ(N²)). A useful trick is to imagine a `print` at the top of the method and count printouts.
- `f3`, which calls itself twice on n−1, is **Θ(2^N)**: each +1 in N doubles the work, and exact counting gives C(N) = 1 + 2 + ... + 2^(N−1) = 2^N − 1.
- Recurrence relations (C(N) = 2C(N−1) + 1 and friends) are **out of scope**; shown only for interest.
- Binary search is **Θ(log N)** where N = hi − lo + 1. Intuition: repeated halving, 1 = N/2^C gives C = log₂ N. Exact count: C(N) = ⌊log₂ N⌋ + 1.
- Simplification rules: ⌊f⌋ = Θ(f), ⌈f⌉ = Θ(f), and log_P N = Θ(log_Q N), so the log base is dropped. Write Θ(log N).
- Logarithmic runtimes are effectively constant in practice: going from 100 items to 100 trillion moves the runtime from ~1 ns to ~7 ns.
- Binary search is deceptively hard to implement: idea 1946, first correct implementation 1962, bug found in Java's own version in 2006 (overflow in `(lo + hi) / 2`).
- Selection sort is **Θ(N²)**; `merge` is **Θ(N)** (exactly N array writes).
- Hybrids (selection sort the halves or quarters, then merge) give real constant-factor wins (4096 → 2112 → 1152 AU at N = 64) but remain Θ(N²).
- Mergesort merges all the way down and is **Θ(N log N)**: ~N work per level, log₂ N levels. At N = 64 that is ~64 × 6 = ~384 AU versus ~4096 for selection sort.
- The level-by-level technique (work per level × number of levels) is the most reusable idea in the lecture; it reappears for quicksort, for exam toy problems, and for the discussion problems "forestry" and "assisted forestry".
- N² vs N log N is an enormous difference; N log N vs N is nice but not radical. For N = 1,000,000, log N is only ~20.
- Multi-variable asymptotics: Θ(f + g) = Θ(max(f, g)). Merging arrays of sizes N and M is Θ(N + M); Θ(NM + M) = Θ(NM). The notion was not formally defined, and defining it rigorously is genuinely hard.
- Big O is like ≤; Big Theta is like =. Quick union tree height is simultaneously Θ(1) in the best case, Θ(N) in the worst case, O(N), and O(N²); the two Theta statements are the informative ones.
- **Big O is not mathematically the same as "worst case"**, even though it is routinely used that way. Prefer "Θ(N) in the worst case" over "O(N)".
- Big O remains useful for blanket statements, for bounding runtimes nobody knows exactly, and because O(f) together with Ω(f) yields Θ(f).
