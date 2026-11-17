<!-- Mon, Nov 16, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 34: Quicksort

Quicksort is the sort that Tony Hoare invented in 1960 while trying to build a Russian-to-English translation program: he needed to sort a short sentence so he could scan a dictionary stored on tape exactly once, because binary searching a tape is not logarithmic when the tape head has to physically move. The algorithm itself is three lines of idea: pick one item (the **pivot**), **partition** the array so that everything to the pivot's left is `<=` it and everything to its right is `>=` it (which puts the pivot in its final sorted position), then recursively quicksort the left part and the right part. Everything interesting about Quicksort lives in the two choices the idea leaves open: how you pick the pivot, and how you partition. A naive partition that copies into a new array is correct but slow and uses `Θ(N)` space; Hoare's two-pointer in-place scheme is `Θ(N)` time and `Θ(1)` space and makes Quicksort beat Mergesort empirically (0.9s vs 1.3s on the lecture's benchmark). The catch is the runtime profile: best case `Θ(N log N)`, worst case `Θ(N^2)` when the pivot repeatedly lands at the edge (for leftmost-pivot Quicksort that means already-sorted or nearly-sorted input), and `Θ(N log N)` expected on a randomly ordered array. The rest of the lecture is about why the average case dominates in practice (the 10 percent argument, the BST-sort equivalence, the `~2N ln N` empirical compare count) and the four philosophies for dodging the worst case: randomness, smarter pivot selection, introspection, and preprocessing.

## Key Concepts

### The warm-up: which sorts even have "bad" inputs

The lecture opens by asking you to construct a worst-case array for selection sort, heapsort, mergesort, and insertion sort. The useful observation is that most of these do not really have adversarial inputs:

- **Selection sort** scans the remaining items to find the minimum `N` times no matter what the data looks like, so it is `Θ(N^2)` on every input. There is no bad array because every array is the bad array.
- **Mergesort** always splits in half and always merges, so it is `Θ(N log N)` on every input.
- **Heapsort** is `Θ(N log N)` in the worst case, and its *best* case (`Θ(N)`) needs all duplicates, so again there is no input that pushes it past `N log N`.
- **Insertion sort** is the one with a real spread: `Θ(N)` when there are `cN` inversions (almost sorted), `Θ(N^2)` when reverse sorted.

(extra context) The "is there a simple fix?" half of the warm-up is a setup for Quicksort. For these four sorts there is nothing to fix, and in insertion sort's case you cannot fix it by shuffling, because a random array has about `N^2/4` inversions and shuffling would also destroy the almost-sorted case you wanted to keep fast. Quicksort is the first sort in the course where shuffling the input genuinely converts a worst case into an expected case, which is why the question is asked right before it.

### Why Hoare needed a sort at all

The translation setup is worth understanding because it explains the constraint that drove the design:

- A sentence of `N` words lives in RAM as an array. A dictionary of `D` English words lives on a long tape.
- The obvious algorithm is `N` binary searches, one per word, for `N log D` total. That *assumes log-time binary search*, which assumes random access.
- Tape is not random access. Jumping to the middle of a tape means physically spooling to it, so a "binary search" on tape costs far more than `log D`.
- The fix: sort the `N` words of the sentence, then stream the dictionary tape once from start to finish, matching words as you pass them. That costs `N log N + D`.

So Quicksort exists because sorting `N` things in RAM was cheaper than seeking `N` times on tape. Amusingly, the lecture notes that Quicksort was the wrong tool for this job twice over: the language Hoare was using did not support recursion, so he could not easily implement it, and sentences are usually under 15 words, where insertion sort is faster anyway.

### Partitioning is the whole algorithm

A partition of an array around a pivot value `x` is a rearrangement where all entries left of `x` are `<= x` and all entries right of `x` are `>= x`. Three consequences matter:

1. **The pivot is now in its final sorted position.** Nothing to its left can ever need to move right past it, and nothing to its right can ever need to move left past it. This is the key insight that makes the recursion work: one item is permanently done after one partition.
2. **The two sides are independent subproblems.** Sorting the left side never looks at the right side. So "partition, then quicksort each side" terminates with a fully sorted array.
3. **A partition is not unique.** The relative order of items on a side does not matter. On the lecture's `[6, 8, 3, 1, 2, 7, 4]` with pivot `6`, both `[3, 1, 2, 4, 6, 8, 7]` and `[3, 4, 1, 2, 6, 7, 8]` are valid outputs. This freedom is exactly what lets partitioning be done in `Θ(N)` time and `Θ(1)` space.

Note the inequalities are non-strict on both sides (`<=` on the left, `>=` on the right). That is deliberate: items equal to the pivot are allowed on either side, which is what makes arrays of duplicates manageable.

### The "white, red, blue" framing

The lecture poses partitioning as a job interview question to strip away the sorting context. You are given an array whose `0`th element is white (the pivot) and whose remaining elements are either red (less than the pivot) or blue (greater). Rearrange so all reds are left of the white and all blues are right of it, in `Θ(N)` time, with no space restriction. Relative order within reds or within blues does not need to be preserved.

Ideas raised in lecture: swap the rightmost blue with the leftmost red repeatedly and work inward, or do a pass that forks reds toward the left end and blues toward the right end. Both are pointing at the same structural idea that Hoare's scheme formalizes.

### Naive partitioning (the 3-scan method)

The straightforward answer: allocate a second array of the same length, then

1. scan the input and copy every red item into the first `R` slots,
2. scan for the white item and place it right after them,
3. scan and copy every blue item into the last `B` slots.

On `[6, 8, 3, 1, 2, 7, 4]` this produces `[3, 1, 2, 4, 6, 8, 7]`. This is `Θ(N)` time, which satisfies the stated requirement, but it costs `Θ(N)` extra space and does three separate passes over memory. The lecture's benchmark shows the damage: Quicksort with leftmost pivot, 3-scan partition, and a pre-shuffle ("Quicksort L3S") takes 3.2 seconds on 1000 arrays of 10000 ints, while Mergesort takes 1.3 seconds. Quicksort with the asymptotically identical but constant-factor-better Hoare partition ("Quicksort LTHS") takes 0.9 seconds. Same `Θ`, three and a half times the wall clock.

### Hoare partitioning: two pointers that hate different things

The in-place scheme. Set up two pointers, `L` starting just right of the pivot and `G` starting at the right end, and give each one a personality:

- **`L` is a friend to small items and hates large *or equal* items.** It walks right and stops when it finds something it hates.
- **`G` is a friend to large items and hates small *or equal* items.** It walks left and stops when it finds something it hates.
- When both pointers are stopped, **swap** the two hated items (each pointer has found an item the other side would be happy with) and then move each pointer by one.
- When the pointers cross, you are done walking.
- Finally, **swap the pivot with `G`**. (The lecture flags that this last step is not in Hoare's original scheme, but is equivalent.)

Why does "hates *equal* items" matter? Because an item equal to the pivot is legal on either side, so a pointer that stops on equals keeps moving items across the midline and keeps the two sides balanced. A pointer that skips past equals would walk all the way through an array of duplicates and produce a maximally lopsided split, giving `Θ(N^2)` on all-duplicate input. The lecture calls this out directly as a common bug "even in a well known 2010s textbook."

Hoare partitioning is `Θ(N)` time (each pointer traverses its part of the array once, and together they cover the array once) and `Θ(1)` space, since there is no array copy.

### Runtime: it all depends on where the pivot lands

**Worst case.** Suppose the pivot always ends up at the very start (or very end) of the subarray, for example because you always pick the leftmost item and the array is already sorted. Then partitioning the full array costs `N`, the one remaining subproblem has size `N-1` and costs `N-1`, then `N-2`, and so on:

```
N + (N-1) + (N-2) + ... + 2 + 1 = Θ(N^2)
```

The lecture's concrete bad input for leftmost-pivot Quicksort is `[1, 2, 3, 4, 5, 6]`.

**Best case.** Suppose the pivot is always the median. Each level splits into two halves, so the work per level is `N/2 + N/2 = N`, then `N/4 * 4 = N`, and so on. There are `Θ(log N)` levels before every subproblem has size 1:

```
Work per level: Θ(N)
Number of levels: Θ(log N)
Total: Θ(N log N)
```

**Average case.** `Θ(N log N)` expected for a randomly ordered array. The rigorous proof needs probability and calculus, but the lecture gives two convincing arguments plus data.

### Why the average case is so robust: three arguments

**The 10 percent argument.** Suppose you are unlucky and your pivot *never* lands near the middle, but it always lands at least 10 percent in from either edge. Then the subproblems are `N/10` and `9N/10`, then `N/100`, `9N/100`, `9N/100`, `81N/100`, and so on. Work per level is still `O(N)`. The height is driven by the slowest-shrinking branch, which divides by `10/9` each time, so `H ≈ log_{10/9} N = O(log N)`. Total: `O(N log N)`. The punchline is that you do not need good pivots, you only need pivots that are not pathologically extreme, and "not extreme" is the overwhelmingly common case.

**Quicksort is BST sort.** Consider inserting `5, 3, 2, 1, 7, 8, 4, 6` into a BST in that order, versus quicksorting the same array with leftmost pivot. The `compareTo` calls are identical. Every element gets compared to `5` in both (BST: because `5` is the root; Quicksort: because `5` is the first pivot). Then `3` gets compared only with `1, 2, 4, 5` in both. The recursion tree of Quicksort *is* the BST built by inserting in that order. Since random insertion into a BST takes `O(N log N)` time, Quicksort on a randomly ordered array takes `O(N log N)`.

**Empirical compare counts.** Over 10,000 trials with `N = 1000`, the mean number of compares to complete Quicksort is about `2N ln N`, with a small standard deviation (the exact formula did not survive the slide extraction). The histogram is tightly clustered: lots of arrays need around 12,000 compares, a very small number need around 15,000, and the chance of needing anything like 1,000,000 compares is effectively zero. The worst case exists but it is astronomically unlikely on random data.

### The four philosophies for avoiding the worst case

1. **Randomness.** Pick a random pivot, or shuffle the array before sorting. This is the lecture's preferred approach. Shuffling requires care in the partitioning code so that arrays of duplicates do not trigger `Θ(N^2)` (see the "hates equal" discussion above).
2. **Smarter pivot selection.** Calculate or approximate the median.
   - *Constant-time deterministic selection is never safe.* For any pivot rule that is both deterministic and constant time (leftmost, middle, median-of-three, and so on), there is a family of dangerous inputs an adversary can easily generate. See McIlroy's "A Killer Adversary for Quicksort."
   - *Linear-time exact median is safe but slow.* "Exact median Quicksort" has worst case `Θ(N log N)`, which beats Mergesort's guarantee on paper but loses badly in practice.
3. **Introspection.** Watch your recursion depth, and if it exceeds some critical value (say `10 ln N`), bail out and switch to a safer sort like Mergesort. Java does this, in addition to approximating the median.
4. **Preprocess the array.** Analyze the input to see whether Quicksort will be slow. There is no obvious way to do this. Checking whether the array is sorted is not enough, because almost-sorted arrays are almost as slow, and detecting "almost sorted" is itself work.

### Quick Select (median finding) (extra)

Marked as extra material in the lecture. The best pivot is the median, so: can you find the median in better than sort time? Sorting and taking the middle element costs `Θ(N log N)`, which defeats the purpose. The insight is that *partitioning itself* can find the median, because a partition tells you the exact final index of the pivot.

The algorithm: partition. If the pivot landed at the index you are looking for, you are done. If it landed left of your target, the target must be in the right subarray, so recurse there only. If it landed right of your target, recurse left only. Unlike Quicksort, you recurse into one side, not both.

- **Worst case `Θ(N^2)`**, on a sorted array with leftmost pivot: `[1 2 3 ... N]` shaves off one element per partition, so you do `N + (N-1) + ...` work.
- **Expected `Θ(N)`**. Intuition (not a proof): on average the pivot lands about halfway, so the work is `N + N/2 + N/4 + ... + 1 = Θ(N)`, a geometric series that sums to linear.

There is also a truly worst-case-linear median algorithm, **BFPRT** (called PICK in the original 1972 paper, whose authors include four Turing Award winners plus Vaughan Pratt). It is rarely used in practice.

Does exact-median Quicksort pay off? The benchmark says no, emphatically:

| Variant | Pivot selection | Partition | Worst case avoidance | Time (1000 arrays of 10000 ints) | Worst case |
|---|---|---|---|---|---|
| Mergesort | N/A | N/A | N/A | 1.3 s | `Θ(N log N)` |
| Quicksort L3S | Leftmost | 3-scan | Shuffle | 3.2 s | `Θ(N^2)` |
| Quicksort LTHS | Leftmost | Tony Hoare | Shuffle | 0.9 s | `Θ(N^2)` |
| Quicksort QSTH | QuickSelect | Tony Hoare | Exact median | 4.9 s | `Θ(N^2)` |
| Quicksort PickTH | Exact median (PICK) | Tony Hoare | Exact median | ~15 s | `Θ(N log N)` |

Two things to read off this table. First, QSTH still has a `Θ(N^2)` worst case despite using exact medians, because QuickSelect itself is `Θ(N^2)` in the worst case, so the pivot-finding step is the thing that blows up. Second, PickTH does fix the worst case to `Θ(N log N)` and is roughly 17 times slower than plain LTHS, because computing medians costs too much. The lecture's conclusion: you have to live with a worst case of `Θ(N^2)` if you want good practical performance. There is also something philosophically odd about doing a bunch of partitions just to identify the optimal item to partition around.

(Note on the benchmarks: these are unoptimized versions of both Mergesort and Quicksort, with no switch to insertion sort for small arrays.)

### Where Quicksort sits among the sorts

| | Memory | Time | Notes |
|---|---|---|---|
| Selection sort | `Θ(1)` | `Θ(N^2)` best and worst | Find smallest item in remaining items, `N` times |
| Heapsort (in place) | `Θ(1)` | `Θ(N log N)` worst, `Θ(N)` best (all duplicates) | Bad cache performance (covered in 61C) |
| Mergesort | `Θ(N)` | `Θ(N log N)` best and worst | Faster than heapsort |
| Insertion sort (in place) | `Θ(1)` | `Θ(N^2)` worst, `Θ(N)` with `cN` inversions | Best for small `N` or almost sorted |
| Random Quicksort (in place) | `Θ(log N)` expected (call stack) | `Θ(N log N)` expected, `Θ(N^2)` worst | Fastest sort |

By mechanism:

- **Selection sort:** find the smallest item and put it at the front.
- **Insertion sort:** figure out where to insert the current item.
- **Mergesort:** merge two sorted halves into one sorted whole.
- **Partition (quick) sort:** partition items around a pivot.

Note that Quicksort's `Θ(log N)` space is the recursion call stack, not auxiliary arrays. The partition itself is in place. (extra context) In the `Θ(N^2)` worst case the recursion depth is `Θ(N)`, so the worst-case space is `Θ(N)` on the call stack unless the implementation recurses on the smaller side and loops on the larger.

The lecture also notes that "Quicksort is the fastest" only holds once you make the right decisions about pivot selection, partition algorithm, and worst-case avoidance. More recent schemes do better than LTHS: the best known Quicksort uses a two-pivot scheme, which was introduced to the world by a previously unknown developer in a Java developers forum.

## Definitions

- **Pivot:** the single item chosen during a partition step, which ends up in its final sorted position once the partition completes.
- **Partition (of an array around pivot `x`):** a rearrangement of the array's items such that every entry to the left of `x` is `<= x` and every entry to the right of `x` is `>= x`. Not unique; the order within each side is unconstrained.
- **Quicksort:** partition the array on some pivot, then recursively quicksort the left part and the right part. Base case is a part of size 0 or 1.
- **Naive (3-scan) partitioning:** partitioning by allocating a second array and performing three scans, copying items less than the pivot into the front, the pivot next, and items greater than the pivot into the back. `Θ(N)` time, `Θ(N)` space.
- **Hoare partitioning:** in-place partitioning using two pointers that walk toward each other from the ends, each stopping on an item it "hates" (`L` hates large or equal, `G` hates small or equal), swapping when both are stopped, and finishing by swapping the pivot into `G`'s position. `Θ(N)` time, `Θ(1)` space.
- **Best case runtime (Quicksort):** `Θ(N log N)`, achieved when the pivot is always the median.
- **Worst case runtime (Quicksort):** `Θ(N^2)`, achieved when the pivot always lands at an edge of the subarray. For leftmost-pivot Quicksort, sorted or nearly sorted input triggers this.
- **Expected/average runtime (Quicksort):** `Θ(N log N)` on a randomly ordered array, with high probability.
- **In place:** uses `O(1)` auxiliary data space beyond the input array, ignoring the recursion call stack.
- **Introspection (introsort idea):** monitoring the recursion depth during Quicksort and switching to a different sort (such as Mergesort) if the depth exceeds a threshold like `10 ln N`.
- **Quick Select:** find the `k`th smallest element by partitioning and recursing into only the side that contains index `k`. Expected `Θ(N)`, worst case `Θ(N^2)`.
- **The selection problem:** given an array and an index `k`, find the element that would occupy index `k` if the array were sorted. Median finding is the case `k = N/2`.
- **BFPRT (PICK):** a 1972 algorithm that finds the exact median in worst-case `Θ(N)` time. Rarely used in practice because its constants are large.
- **Killer adversary:** an input constructed specifically to force `Θ(N^2)` behavior out of a given Quicksort implementation. Exists for every deterministic constant-time pivot rule (McIlroy).
- **Stable sort:** (extra context, not covered in this lecture) a sort that preserves the relative order of equal items. Hoare partitioning is not stable, since it swaps across long distances.

## Worked Examples

All the Java below is my implementation of the lecture's algorithms (the slides give the algorithms as pointer diagrams, not code), so treat the code itself as **(extra context)** while the arrays and traces are the lecture's own.

### Example 1: Checking whether a partition is valid

This is the lecture's clicker question. The original array is `[5, 550, 10, 4, 10, 9, 330]` and the pivot is `5` (the leftmost item). Which of the following are valid partitions?

```
A.  4   5   9  10  10  330 550
B.  5   4   9  10  10  550 330
C.  5   9  10   4  10  550 330
D.  5   9  10   4  10  550 330
```

To check a candidate, do three things in order:

1. **Confirm it is a permutation of the input.** All four options contain exactly `{4, 5, 9, 10, 10, 330, 550}`, so all four pass this test.
2. **Locate the pivot value `5`.**
3. **Check that everything left of it is `<= 5` and everything right of it is `>= 5`.**

- **A is valid.** `5` sits at index 1. To its left is `{4}`, and `4 <= 5`. To its right is `{9, 10, 10, 330, 550}`, all `>= 5`.
- **B is invalid.** `5` sits at index 0, so the left side is empty (fine), but the right side contains `4`, and `4 < 5`. A single item on the wrong side kills the partition.
- **C and D are invalid** for the same reason as B: `5` is at index 0 and `4` sits to its right at index 3.

(Both C and D came out of the PDF as the identical sequence `5 9 10 4 10 550 330`, which means the slide's diagram markers did not survive extraction. Either they were genuinely the same array with different pointer annotations, or one of them differed in a way the extraction lost. Judge each one by the three-step test above rather than by memorizing which letter was correct.)

The exam-relevant lesson: valid partitions are plural, so a question will usually say "which of these are valid" rather than "what is the partition." Scan for a single out-of-place element rather than trying to verify the whole thing holistically.

### Example 2: Naive 3-scan partitioning

This implements the lecture's "Quicksort Partitioning, take 1" on the input `[6, 8, 3, 1, 2, 7, 4]`.

```java
public class NaivePartition {
    public static int[] partition(int[] input) {
        int pivot = input[0];
        int reds = 0;
        for (int i = 1; i < input.length; i += 1) {
            if (input[i] < pivot) {
                reds += 1;
            }
        }

        int[] output = new int[input.length];
        int next = 0;
        for (int i = 1; i < input.length; i += 1) {
            if (input[i] < pivot) {
                output[next] = input[i];
                next += 1;
            }
        }

        output[reds] = pivot;

        next = reds + 1;
        for (int i = 1; i < input.length; i += 1) {
            if (input[i] > pivot) {
                output[next] = input[i];
                next += 1;
            }
        }

        return output;
    }
}
```

Step by step on `[6, 8, 3, 1, 2, 7, 4]`:

1. `pivot = input[0] = 6`.
2. **Counting pass.** Walk indices 1 through 6 and count items less than 6: `3`, `1`, `2`, `4`. So `reds = 4`. This pass exists only so we know where the pivot belongs, at index 4.
3. **Red pass.** Allocate a fresh `int[7]` (this is the `Θ(N)` space). Walk the input again and copy each item less than 6 into the next free front slot, in the order encountered: `3` goes to index 0, `1` to index 1, `2` to index 2, `4` to index 3. Output so far: `[3, 1, 2, 4, _, _, _]`.
4. **White placement.** `output[4] = 6`. Output: `[3, 1, 2, 4, 6, _, _]`.
5. **Blue pass.** Walk the input a third time and copy each item greater than 6 into the next free slot starting at index 5: `8` goes to index 5, `7` to index 6. Output: `[3, 1, 2, 4, 6, 8, 7]`.

This matches the lecture's output exactly. Notice that the blue items came out as `8, 7` rather than `7, 8`, which is fine, since a partition does not constrain order within a side.

**Environment and pointer reasoning in words.** When `partition` is called, the local variable `input` holds a reference to the caller's array object, so reading `input[i]` reaches the caller's array on the heap. `output` holds a reference to a *different*, freshly allocated array object. Because `int` is primitive, `output[next] = input[i]` copies the value, so the two arrays never share element storage. The input array is never written to, so the original is untouched, which is exactly why this version needs `Θ(N)` extra space and why it is "not in place." Three passes over the input also means three walks through memory, which is where the constant-factor penalty behind the 3.2-second benchmark comes from.

### Example 3: Hoare partitioning, traced on the lecture's array

This implements the lecture's two-pointer rules, including the final pivot swap.

```java
public class HoarePartition {
    private static void swap(int[] a, int i, int j) {
        int temp = a[i];
        a[i] = a[j];
        a[j] = temp;
    }

    public static int partition(int[] a, int lo, int hi) {
        int pivot = a[lo];
        int left = lo + 1;
        int right = hi;

        while (left <= right) {
            while (left <= hi && a[left] < pivot) {
                left += 1;
            }
            while (right > lo && a[right] > pivot) {
                right -= 1;
            }
            if (left > right) {
                break;
            }
            swap(a, left, right);
            left += 1;
            right -= 1;
        }

        swap(a, lo, right);
        return right;
    }
}
```

Mapping the code to the lecture's personalities: the condition `a[left] < pivot` means `left` keeps moving while the item is strictly small, which is the same as saying it stops on large *or equal* items. Symmetrically, `a[right] > pivot` means `right` stops on small *or equal* items. The guards `left <= hi` and `right > lo` keep the pointers inside the subarray when the pivot happens to be the maximum or minimum.

Trace on the lecture's input `[17, 15, 19, 32, 2, 26, 41, 17, 17]` with `lo = 0`, `hi = 8`, so `pivot = 17`, `left = 1`, `right = 8`:

**Round 1.**
- `left` at index 1 sees `15`. `15 < 17`, so this is a "lovely" small item and `left` advances to index 2.
- `left` at index 2 sees `19`. `19` is not `< 17`, so `left` stops ("Grrr, 19").
- `right` at index 8 sees `17`. `17` is not `> 17`, so `right` stops. This is the crucial equal-item stop ("I dislike 17").
- Both stopped and `left = 2 <= right = 8`, so swap indices 2 and 8, then move both inward.
- Array: `[17, 15, 17, 32, 2, 26, 41, 17, 19]`. Now `left = 3`, `right = 7`.

**Round 2.**
- `left` at index 3 sees `32`, immediately hated, stop ("Immediate trouble").
- `right` at index 7 sees `17`, equal, immediately hated, stop ("Trouble here, too").
- Swap indices 3 and 7, move inward.
- Array: `[17, 15, 17, 17, 2, 26, 41, 32, 19]`. Now `left = 4`, `right = 6`.

**Round 3.**
- `left` at index 4 sees `2`. Small and lovely ("2 is cool"), advance to index 5.
- `left` at index 5 sees `26`, hated, stop ("26 is grossly large").
- `right` at index 6 sees `41`. `41 > 17`, so it is a large item `right` is happy to leave alone ("41 is fine"), decrement to index 5.
- `right` at index 5 sees `26`, also fine, decrement to index 4.
- `right` at index 4 sees `2`, which is small, hated, stop ("2 is no good, also hi L!!").
- Now `left = 5 > right = 4`: the pointers have crossed, so break out of the loop.

**Final step.** Swap the pivot at `lo = 0` with index `right = 4`:
- Array: `[2, 15, 17, 17, 17, 26, 41, 32, 19]`, return `4`.

Verify the postcondition: the pivot `17` is at index 4. Left of it: `2, 15, 17, 17`, all `<= 17`. Right of it: `26, 41, 32, 19`, all `>= 17`. Valid partition, computed with zero extra arrays.

**Why `right` and not `left` for the final swap.** When the loop ends, `right` is sitting on an item that is `<= pivot` (that is why it stopped) and everything strictly right of `right` is `>= pivot`. So `right` is the last position in the "small" region, and swapping the pivot there puts the pivot exactly at the boundary. Swapping with `left` would move an item that is `>= pivot` into the left region and break the invariant.

**Why it is `Θ(N)` time.** `left` only ever increases and `right` only ever decreases, and they stop once they cross. Between them they visit each index a constant number of times, so one partition call does `Θ(hi - lo)` work.

**Why it is `Θ(1)` space.** The only extra storage is `pivot`, `left`, `right`, and `temp` inside `swap`, all fixed-size locals. `swap` receives a *reference* to the same array object the caller holds, so writes through `a` mutate the caller's array in place. This is the behavioral difference from Example 2: the naive version returns a new array and leaves the input alone, while this version returns an index and destructively rearranges the input.

**Duplicates sanity check.** On `[5, 5, 5, 5, 5]`: `pivot = 5`, `left = 1`, `right = 4`. Neither inner loop advances (nothing is strictly less or strictly greater), so we swap 1 and 4 (a no-op on equal values), then swap 2 and 3 (also a no-op), then `left = 3 > right = 2` and we break. Swapping `lo` with `right = 2` puts the pivot at index 2, the middle. The subproblems have sizes 2 and 2, which is a balanced split. That balance is precisely the payoff for stopping on equal items, and it is why all-duplicate arrays run in `Θ(N log N)` here instead of `Θ(N^2)`.

### Example 4: Full Quicksort with a shuffle

```java
import java.util.Random;

public class Quicksort {
    private static final Random RANDOM = new Random();

    public static void sort(int[] a) {
        shuffle(a);
        sort(a, 0, a.length - 1);
    }

    private static void sort(int[] a, int lo, int hi) {
        if (lo >= hi) {
            return;
        }
        int pivotIndex = HoarePartition.partition(a, lo, hi);
        sort(a, lo, pivotIndex - 1);
        sort(a, pivotIndex + 1, hi);
    }

    private static void shuffle(int[] a) {
        for (int i = a.length - 1; i > 0; i -= 1) {
            int j = RANDOM.nextInt(i + 1);
            int temp = a[i];
            a[i] = a[j];
            a[j] = temp;
        }
    }
}
```

Step by step on the lecture's array `[32, 15, 2, 17, 19, 26, 41, 17, 17]`, assuming for the demo that the shuffle happens to leave it alone:

1. `sort(a, 0, 8)`. Not a base case, so partition on the leftmost item, `32`.
2. The partition puts `32` at index 7 and arranges the array as `[15, 2, 17, 19, 26, 17, 17, 32, 41]`. Everything in indices 0 through 6 is `<= 32`, and index 8 holds `41`, which is `>= 32`. **Index 7 is now permanently correct and will never be touched again.**
3. `sort(a, 0, 6)` recursively sorts the left part. The lecture does not show these details; the result is `[2, 15, 17, 17, 17, 19, 26]` in indices 0 through 6, all in their final places.
4. `sort(a, 8, 8)` on the right part hits the base case `lo >= hi` immediately: a one-element region is already sorted.
5. Final array: `[2, 15, 17, 17, 17, 19, 26, 32, 41]`.

**Why the base case is `lo >= hi` rather than `lo == hi`.** When the pivot lands at the very left of a region, `pivotIndex - 1 < lo`, so the left recursive call gets `hi < lo`, an empty region. The `>=` catches both the empty and the single-element case.

**Call stack reasoning in words.** Each frame of `sort(a, lo, hi)` holds just `lo`, `hi`, and `pivotIndex`, all primitives, plus the reference `a` pointing at the one shared array on the heap. There is no array copying anywhere, which is why Quicksort's space cost is purely the depth of this stack. Balanced partitions give depth `Θ(log N)`; the degenerate case gives depth `Θ(N)`.

**What the shuffle buys.** Without the shuffle, calling `sort` on `[1, 2, 3, ..., N]` makes every partition put the pivot at index `lo`, so each call spawns an empty left region and a right region of size `n - 1`, giving `Θ(N^2)` time and `Θ(N)` stack depth. The shuffle makes the *input order* random, so the sorted-input worst case no longer depends on the data your caller happened to hand you. Equivalently, you can skip the shuffle and pick `pivotIndex` at random inside `partition` by swapping a random element into position `lo` first.

### Example 5: Quick Select, traced on the lecture's median example (extra)

```java
public class QuickSelect {
    public static int select(int[] a, int k) {
        int lo = 0;
        int hi = a.length - 1;
        while (lo < hi) {
            int pivotIndex = HoarePartition.partition(a, lo, hi);
            if (pivotIndex == k) {
                return a[k];
            } else if (pivotIndex < k) {
                lo = pivotIndex + 1;
            } else {
                hi = pivotIndex - 1;
            }
        }
        return a[k];
    }

    public static int median(int[] a) {
        return select(a, a.length / 2);
    }
}
```

The lecture's example: find the median of `[9, 550, 14, 6, 10, 5, 330, 817, 913]`. There are 9 items, so the median is the element at index `9 / 2 = 4`, and `k = 4`.

1. **Partition on the whole array** with pivot `9`. Items `6` and `5` are less than `9`, so `9` lands at index 2: `[6, 5, 9, 550, 14, 10, 330, 817, 913]`. Is `2 == 4`? No. Is `2 < 4`? Yes, so the median must sit somewhere to the *right* of index 2. Set `lo = 3`, keep `hi = 8`. Indices 0 through 2 are discarded from consideration forever.
2. **Partition indices 3 through 8** with pivot `550`. Items `14`, `10`, `330` are less than `550`, so they fill indices 3, 4, 5 and `550` lands at index 6: `[6, 5, 9, 14, 10, 330, 550, 817, 913]`. Is `6 == 4`? No. Is `6 < 4`? No, so the median sits to the *left* of index 6. Set `hi = 5`, keep `lo = 3`.
3. **Partition indices 3 through 5** with pivot `14`. Only `10` is less than `14`, so `14` lands at index 4: `[6, 5, 9, 10, 14, 330, 550, 817, 913]`. Is `4 == 4`? Yes. Return `a[4] = 14`.

Check by hand: sorted, the array is `5, 6, 9, 10, 14, 330, 550, 817, 913`, and the element at index 4 is indeed `14`. (The slide's final rows came out of the PDF with garbled leading values, but the index arithmetic and the "Yep, 9/2 = 4" annotation are intact and the answer is `14`.)

**The key contrast with Quicksort.** After each partition, Quicksort recurses on *both* sides; Quick Select recurses on *one*. That is the entire reason the geometric series collapses from `N log N` to `N`: on average the work is `N + N/2 + N/4 + ... = Θ(N)`, because the surviving subproblem halves each time instead of the total work staying at `N` per level.

**Worst case.** On `[1, 2, 3, ..., N]` with leftmost pivot, each partition puts the pivot at the left edge and only removes one element from consideration, so you do `N + (N-1) + (N-2) + ...` work, which is `Θ(N^2)`. That is the reason the QSTH row in the benchmark table still shows a `Θ(N^2)` worst case even though it is using "exact medians" as pivots: the median finder is the part that degrades.

## Common Pitfalls

- **Forgetting that the pivot is finished after one partition.** A frequent mistake is writing `sort(a, lo, pivotIndex)` and `sort(a, pivotIndex, hi)`, which includes the pivot in a subproblem. On a two-element region this never shrinks the problem and you get infinite recursion. The pivot is already in its final place, so the recursive calls must exclude it: `pivotIndex - 1` and `pivotIndex + 1`.
- **Skipping over items equal to the pivot.** Writing `a[left] <= pivot` in the inner loop (instead of `<`) makes `left` walk past equal items. On an array of all duplicates, `left` then runs to the end of the array and the partition splits into sizes `N-1` and `0`, which is `Θ(N^2)`. The lecture flags this as a bug that appeared in a well known 2010s textbook. `L` must hate large **or equal** items; `G` must hate small **or equal** items.
- **Dropping the bounds guards in the inner loops.** If the pivot is the largest item in the region, nothing stops `left` from walking past `hi` and off the end of the array. `left <= hi` and `right > lo` are not decoration.
- **Swapping the pivot with `left` instead of `right` at the end.** `left` is sitting on an item that is `>= pivot`; moving that into the left region breaks the partition invariant. After crossing, `right` is the last index of the small region, so `right` is the pivot's home.
- **Claiming Quicksort is `Θ(N log N)`.** It is `Θ(N log N)` in the *best case* and in *expectation on a randomly ordered array*. Its worst case is `Θ(N^2)`. Writing `Θ(N log N)` with no qualifier on an exam about worst-case behavior is wrong.
- **Claiming Quicksort uses `Θ(1)` space.** The partition is in place, but the recursion is not free. Expected call stack depth is `Θ(log N)`, and worst-case depth is `Θ(N)`.
- **Thinking the worst case requires a weird or random array.** For leftmost-pivot Quicksort the worst case is *sorted* input, which is one of the most common shapes real data arrives in. This is why worst cases "do happen in practice" despite the histogram being tightly clustered on random data.
- **Believing median-of-three or middle-element pivots make Quicksort safe.** Any deterministic, constant-time pivot rule has a family of killer inputs (McIlroy). Only randomness, linear-time median computation, or introspection actually removes the vulnerability.
- **Trying to detect the bad case by checking whether the array is sorted.** The lecture explicitly rules this out: almost-sorted arrays are almost as slow, and that check does not catch them.
- **Assuming a better asymptotic worst case means a faster sort.** `Quicksort PickTH` has a `Θ(N log N)` worst case and runs about 17 times slower than `Quicksort LTHS`, which has a `Θ(N^2)` worst case. Asymptotics describe growth, not speed at a given `N`.
- **Confusing "a partition" with "the partition."** Partitions are not unique. Checking an answer means verifying the inequalities, not comparing against one memorized output.
- **Forgetting the shuffle is part of the algorithm's correctness argument.** `sort(a)` shuffles and then partitions. If you call the private `sort(a, lo, hi)` directly on sorted data, you have opted out of the randomness and reintroduced the worst case.

## Likely Exam Points

### 1. Best, worst, and average runtime of Quicksort

**Q:** Give the best case, worst case, and expected (random array) runtimes of Quicksort, and its space complexity. For a Quicksort that always chooses the leftmost item as pivot, give a concrete 6-element array that triggers the worst case.

**A:** Best case `Θ(N log N)` (pivot is always the median: `Θ(N)` work per level, `Θ(log N)` levels). Worst case `Θ(N^2)` (pivot always lands at an edge: `N + (N-1) + ... + 1`). Expected `Θ(N log N)` on a randomly ordered array. Space `Θ(log N)` expected, on the recursion call stack, since partitioning is in place. A worst-case array for leftmost pivot: `[1, 2, 3, 4, 5, 6]` (already sorted). Reverse sorted and almost-sorted arrays are also bad.

### 2. Executing Hoare partitioning by hand

**Q:** Partition `[17, 15, 19, 32, 2, 26, 41, 17, 17]` using Hoare's two-pointer scheme with the leftmost item as pivot. Show the array after each swap and give the pivot's final index.

**A:** Pivot `17`, `L` starts at index 1, `G` at index 8.
- `L` passes `15`, stops at `19` (index 2). `G` stops immediately on `17` (index 8, equal counts as hated). Swap: `[17, 15, 17, 32, 2, 26, 41, 17, 19]`.
- `L` stops at `32` (index 3), `G` stops at `17` (index 7). Swap: `[17, 15, 17, 17, 2, 26, 41, 32, 19]`.
- `L` passes `2`, stops at `26` (index 5). `G` passes `41` and `26`, stops at `2` (index 4). `L = 5 > G = 4`, crossed, stop walking.
- Swap pivot with `G`: `[2, 15, 17, 17, 17, 26, 41, 32, 19]`. Pivot's final index is **4**.

### 3. Time and space complexity of partitioning

**Q:** State the time and space complexity of (a) the 3-scan partition and (b) Hoare partition. Both are `Θ(N)` time, so why does the choice matter?

**A:** (a) `Θ(N)` time, `Θ(N)` space (it allocates a second array). (b) `Θ(N)` time, `Θ(1)` space (in place, no copy). It matters because of constant factors and memory traffic: on the lecture's benchmark, Quicksort with 3-scan partitioning (L3S) takes 3.2 seconds while Quicksort with Hoare partitioning (LTHS) takes 0.9 seconds on the same workload, and only the Hoare version beats Mergesort's 1.3 seconds. Same asymptotics, very different sort.

### 4. Validating a partition

**Q:** The array `[5, 550, 10, 4, 10, 9, 330]` is partitioned around pivot `5`. Is `[5, 4, 9, 10, 10, 550, 330]` a valid partition? Justify.

**A:** No. The pivot `5` is at index 0, so its left side is empty (acceptable), but `4` appears at index 1, to the right of the pivot, and `4 < 5`. A valid partition requires every entry right of the pivot to be `>= 5`. By contrast `[4, 5, 9, 10, 10, 330, 550]` *is* valid: `5` is at index 1, the left side `{4}` is all `<= 5`, and the right side `{9, 10, 10, 330, 550}` is all `>= 5`.

### 5. Why Quicksort is `Θ(N log N)` in expectation

**Q:** Give two arguments for why Quicksort runs in `Θ(N log N)` on a randomly ordered array, even though its worst case is `Θ(N^2)`.

**A:** (1) *The 10 percent argument*: if the pivot always lands at least 10 percent in from either edge (far from a good pivot), the subproblem sizes shrink by a factor of `10/9` down the slowest branch, so the height is `log_{10/9} N = O(log N)` while the work per level stays `O(N)`, giving `O(N log N)`. You need only non-extreme pivots, not good ones. (2) *Quicksort is BST sort*: the sequence of `compareTo` calls made by leftmost-pivot Quicksort is identical to the sequence made by inserting the same elements into a BST in array order, so Quicksort's recursion tree is that BST. Random BST insertion is `O(N log N)`, therefore so is Quicksort on a random array. Empirically, the mean compare count is about `2N ln N` with small variance.

### 6. The four philosophies for avoiding the worst case

**Q:** Name the four philosophies for avoiding Quicksort's worst case. Which one does Java use, and why is "check if the array is already sorted" not a solution?

**A:** (1) Randomness: random pivot or shuffle before sorting. (2) Smarter pivot selection: compute or approximate the median. (3) Introspection: watch the recursion depth and switch to a safer sort (say Mergesort) past a threshold like `10 ln N`. (4) Preprocess the array to predict slowness, which has no obvious implementation. Java uses introspection plus median approximation. Checking for sortedness fails because almost-sorted arrays are almost as slow as sorted ones, and that check does not detect them.

### 7. Deterministic pivot rules are never safe

**Q:** Your friend proposes always using the middle element as the pivot, arguing it fixes the sorted-array worst case. Is Quicksort now worst-case `Θ(N log N)`?

**A:** No. Middle-element selection does avoid the specific sorted-input case, but it is deterministic and constant time, so an adversary can construct a family of inputs that forces every pivot to the edge and yields `Θ(N^2)` (see McIlroy, "A Killer Adversary for Quicksort"). This holds for leftmost, rightmost, middle, median-of-three, and any other deterministic constant-time rule. Getting a `Θ(N log N)` worst-case guarantee requires computing the exact median in linear time (safe but slower than Mergesort in practice) or using introspection to bail out to another sort.

### 8. Quick Select runtime (extra)

**Q:** What are the expected and worst-case runtimes of Quick Select, and why does it beat Quicksort asymptotically when Quicksort does strictly more work per level?

**A:** Expected `Θ(N)`, worst case `Θ(N^2)` (sorted input with leftmost pivot). Quicksort recurses into both halves, so the work per level stays at about `N` across `log N` levels, giving `N log N`. Quick Select recurses into only the side containing the target index, so the surviving subproblem roughly halves each time and the work forms a geometric series `N + N/2 + N/4 + ... = Θ(N)`.

### 9. Why exact-median Quicksort is not used (extra)

**Q:** `Quicksort PickTH` uses BFPRT to find the exact median and therefore has worst case `Θ(N log N)`, strictly better than `Quicksort LTHS`. Why does the lecture not recommend it?

**A:** Because it runs about 15 seconds on the benchmark where LTHS runs 0.9 seconds and Mergesort runs 1.3 seconds. Computing exact medians costs far too much in constant factors. The lecture's conclusion is that you have to accept a `Θ(N^2)` worst case to get good practical performance. The related `Quicksort QSTH`, which uses Quick Select for the pivot, is both slow (4.9 seconds) and *still* `Θ(N^2)` in the worst case, because Quick Select itself degrades to `Θ(N^2)`.

### 10. Comparing all sorts so far

**Q:** Fill in best case, worst case, and space for selection sort, heapsort, mergesort, insertion sort, and Quicksort. Which is the only one whose worst case is worse than its best case by more than a log factor?

**A:**

| Sort | Best | Worst | Space |
|---|---|---|---|
| Selection sort | `Θ(N^2)` | `Θ(N^2)` | `Θ(1)` |
| Heapsort (in place) | `Θ(N)` (all duplicates) | `Θ(N log N)` | `Θ(1)` |
| Mergesort | `Θ(N log N)` | `Θ(N log N)` | `Θ(N)` |
| Insertion sort (in place) | `Θ(N)` (`cN` inversions) | `Θ(N^2)` | `Θ(1)` |
| Quicksort (in place) | `Θ(N log N)` | `Θ(N^2)` | `Θ(log N)` expected, call stack |

Insertion sort and Quicksort both span `Θ(N)` or `Θ(N log N)` up to `Θ(N^2)`. Heapsort's spread is only the `log N` factor, and selection sort and mergesort have no spread at all. Heapsort's practical weakness is cache behavior rather than asymptotics (a 61C topic).

## Summary

- **The idea:** pick a pivot, partition so left items are `<= pivot` and right items are `>= pivot` (which places the pivot permanently), then recursively quicksort each side. Base case: a region of size 0 or 1.
- **A partition is not unique.** Order within each side is unconstrained, and items equal to the pivot may sit on either side. Validate a candidate by locating the pivot and checking both inequalities.
- **Naive 3-scan partition:** allocate a new array, copy smaller items to the front, the pivot next, larger items to the back. `Θ(N)` time, `Θ(N)` space, slow in practice.
- **Hoare partition:** two pointers walk toward each other; `L` hates large or equal items, `G` hates small or equal items; swap when both stop, move inward, stop when they cross, then swap the pivot into `G`'s position. `Θ(N)` time, `Θ(1)` space.
- **Stopping on *equal* items is mandatory**, not cosmetic: skipping equals makes all-duplicate arrays `Θ(N^2)`. The lecture notes this bug shipped in a well known textbook.
- **Runtimes:** best `Θ(N log N)` (median pivot), worst `Θ(N^2)` (pivot always at an edge, e.g. sorted input under leftmost-pivot), expected `Θ(N log N)` on a random array. Space `Θ(log N)` expected (recursion stack only).
- **Why average dominates:** the 10 percent argument shows even persistently off-center pivots give `O(N log N)`; Quicksort's compare sequence is identical to random BST insertion, which is `O(N log N)`; empirically the mean is `~2N ln N` with a tight histogram.
- **Benchmarks (1000 arrays of 10000 ints):** Mergesort 1.3s, Quicksort L3S 3.2s, Quicksort **LTHS 0.9s**, Quicksort QSTH 4.9s, Quicksort PickTH ~15s. The partitioning scheme, not the asymptotics, decides whether Quicksort beats Mergesort.
- **Four philosophies against the worst case:** randomness (random pivot or shuffle, the lecture's preference), smarter pivot selection (exact or approximate median), introspection (switch sorts past a depth like `10 ln N`, which Java does), and preprocessing (no good implementation known).
- **No deterministic constant-time pivot rule is safe.** Each has a killer adversary family (McIlroy). Exact-median Quicksort is `Θ(N log N)` worst case and far too slow to use.
- **Quick Select (extra):** partition and recurse into only the side holding your target index. Expected `Θ(N)` (`N + N/2 + N/4 + ...`), worst case `Θ(N^2)` on sorted input. BFPRT/PICK (1972) gets worst-case `Θ(N)` median finding but is rarely used.
- **Historical note:** Hoare invented Quicksort in 1960 to sort sentences so a tape dictionary could be scanned once (`N log N + D` instead of `N log D` with unaffordable tape seeks). It was the wrong tool twice: his language lacked recursion, and sentences are short enough that insertion sort wins.
