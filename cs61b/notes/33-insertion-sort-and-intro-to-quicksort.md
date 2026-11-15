<!-- Fri, Nov 13, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 33: Insertion Sort and Intro to Quicksort

This lecture finishes the "basic sorts" arc by adding insertion sort, then opens the door to quicksort. Insertion sort builds a sorted region one item at a time: take the next unexamined item (the "traveller") and swap it backwards until it sits in the right place among everything already examined. The naive version builds a brand new output array, but we can do the whole thing in place with swaps, giving Θ(1) extra space. The runtime analysis hinges on inversions (pairs out of order): insertion sort does exactly one swap per inversion, so its runtime is Θ(N + K) where K is the inversion count. That makes it Θ(N²) on random input but Θ(N) on almost-sorted input, which is why it wins the warm-up question (sorted array of 1,000,000 integers with one element changed). The second half introduces quicksort through its historical origin: Tony Hoare in 1960 needed to sort a sentence so he could scan a dictionary stored on tape in one pass, since binary search on tape is not logarithmic. The key operation he invented is partitioning: pick a pivot, move all smaller items left and all larger items right, and the pivot lands in its final sorted position. Recursing on the two halves gives quicksort, and the lecture closes by flagging that the runtime depends crucially on where the pivot ends up.

## Key Concepts

### The warm-up that frames the whole lecture

Start with a sorted array of 1,000,000 integers, change one randomly chosen element, then sort. Which algorithm is fastest?

Working through the known candidates:

- **Selection sort**: Θ(N²). It finds the smallest remaining item N times. It has no way to notice that the array is nearly sorted, so it does the same full amount of work regardless.
- **Heapsort**: O(N log N). Again, mostly blind to existing order (its only listed best case, Θ(N), is the all-duplicates case).
- **Mergesort**: Θ(N log N). It always does the same recursive splitting and merging.

The lecture poses this question first and only answers it after developing insertion sort and inversions. The answer is **insertion sort, at Θ(N)**, because changing one element in a sorted array creates at most N - 1 inversions, and insertion sort does one swap per inversion.

The point is bigger than one trivia answer: the asymptotic worst case is not the whole story. An algorithm with a worse worst case (Θ(N²) vs Θ(N log N)) can be dramatically faster on inputs with structure.

### Insertion sort: the general strategy

The general strategy is stated the same way for both versions:

- Start with an empty output sequence.
- Add each item from the input, inserting it into the output at the right point.

That is the whole idea. The only question is whether you need a second array to hold the output.

### Naive insertion sort

The naive version keeps the input and builds an entirely new output array. Using the lecture's running example, input `32 15 2 17 19 26 41 17 17`:

| Step | Item inserted | Output so far |
|---|---|---|
| start | | (empty) |
| 1 | 32 | `32` |
| 2 | 15 | `15 32` |
| 3 | 2 | `2 15 32` |
| 4 | 17 | `2 15 17 32` |
| 5 | 19 | `2 15 17 19 32` |
| 6 | 26 | `2 15 17 19 26 32` |
| 7 | 41 | `2 15 17 19 26 32 41` |
| 8 | 17 | `2 15 17 17 19 26 32 41` |
| 9 | 17 | `2 15 17 17 17 19 26 32 41` |

Notice that each insertion is not free. If the output sequence already contains k items, the worst cost to insert one more item is k, because you might have to shift everything over to make room. (Example: inserting `2` into `15 32` requires shifting both items right.)

The slide asks directly: "Can we eliminate this output array?" Yes.

### In-place insertion sort

The more efficient method does everything in place using swapping. The loop structure:

- Repeat for i = 0 to N - 1:
  - "Insert" the item at index i into the right place among the sorted items. The item at index i is called the **traveller**.
  - How? Swap the traveller backwards until it is in the right place among all previously examined items.

The array is conceptually split into two regions at all times:

```
[ sorted region ][ unexamined region ]
                i
```

Everything strictly left of `i` is already sorted relative to itself. Everything at `i` and right is unexamined. Note the subtlety in the invariant: the slides say "the items up through item i should be sorted," and while the traveller is mid-flight that invariant is **temporarily broken**. Once the traveller settles into its spot, the invariant is restored.

A second pointer `j` tracks the current spot of the travelling item as it moves left. So `i` marks the boundary of the examined region, and `j` marks where the traveller currently sits.

### The inner loop condition

The slides present this as a fill-in-the-blank:

```java
while (/* FILL ME IN */) {
    // Swap backwards
}
```

and then give the answer:

```java
while (j > 0 && input[j - 1] > input[j]) {
    // Swap backwards
}
```

Both halves of that condition matter:

- `j > 0` keeps you from walking off the left end of the array. Without it, `input[j - 1]` with `j == 0` would index `-1` and throw.
- `input[j - 1] > input[j]` stops the traveller as soon as the item to its left is not greater than it. Using strict `>` (rather than `>=`) means equal elements do not get swapped past each other, which is what makes insertion sort **stable** (extra context: the lecture does not use the word "stable" here, but the strictness of this comparison is exactly the stability condition).

Because of short-circuit `&&`, the bounds check runs first, so the array access is always safe.

### Walking the main example in place

Full trace of `32 15 2 17 19 26 41 17 17` in place, following the slides:

| i | Array state after the traveller settles | Swaps this iteration |
|---|---|---|
| 0 | `32 15 2 17 19 26 41 17 17` | 0 |
| 1 | `15 32 2 17 19 26 41 17 17` | 1 |
| 2 | `2 15 32 17 19 26 41 17 17` | 2 |
| 3 | `2 15 17 32 19 26 41 17 17` | 1 |
| 4 | `2 15 17 19 32 26 41 17 17` | 1 |
| 5 | `2 15 17 19 26 32 41 17 17` | 1 |
| 6 | `2 15 17 19 26 32 41 17 17` | 0 |
| 7 | `2 15 17 17 19 26 32 41 17` | 4 |
| 8 | `2 15 17 17 17 19 26 32 41` | 4 |

The `i = 7` step is the one the slides walk in detail, asking "How many swaps do we need to do to get 17 in the correct place?" The traveller `17` starts at index 7 with the sorted region `2 15 17 19 26 32 41` to its left. It swaps past `41`, then `32`, then `26`, then `19`, and stops when it reaches the `17` at index 2 (since `17 > 17` is false). That is **4 swaps**.

### Two more worked traces: POTATO and SORTEXAMPLE

The slides color-code these: **purple** is the element being moved left with swaps, **black** are the elements that got swapped with purple, **grey** is not considered this iteration.

`P O T A T O`, **7 swaps total**:

```
P O T A T O
P O T A T O     (0 swaps)
O P T A T O     (1 swap )
O P T A T O     (0 swaps)
A O P T T O     (3 swaps)
A O P T T O     (0 swaps)
A O O P T T     (3 swaps)
```

`S O R T E X A M P L E`, **36 swaps total**:

```
S O R T E X A M P L E
S O R T E X A M P L E   (0 swaps)
O S R T E X A M P L E   (1 swap )
O R S T E X A M P L E   (1 swap )
O R S T E X A M P L E   (0 swaps)
E O R S T X A M P L E   (4 swaps)
E O R S T X A M P L E   (0 swaps)
A E O R S T X M P L E   (6 swaps)
A E M O R S T X P L E   (5 swaps)
A E M O P R S T X L E   (4 swaps)
A E L M O P R S T X E   (7 swaps)
A E E L M O P R S T X   (8 swaps)
```

Two things to pull out of `SORTEXAMPLE`:

1. Some iterations cost 0 swaps (when the traveller is already larger than everything to its left, like `T` after `O R S`, or `X` after `E O R S T`).
2. Some cost a lot (the final `E` costs 8 swaps, because it has to travel past almost the entire array).

### Insertion sort runtime

The slides pose this as a multiple-choice question. The answer is **D: Best Θ(N), Worst Θ(N²)**.

Reasoning:

- **Best case Θ(N)**: If the array is already sorted, then for every `i`, the condition `input[j - 1] > input[j]` fails immediately, so each iteration does 1 comparison and 0 swaps. You still make N passes of the outer loop, so Θ(N), not Θ(1). (This is the standard trap: students pick Θ(1) because "no work happens," but you still have to look at every element to know that.)
- **Worst case Θ(N²)**: If the array is in reverse order, the traveller at index `i` has to swap all the way to index 0, costing `i` swaps. Total: `0 + 1 + 2 + ... + (N-1) = N(N-1)/2`, which is Θ(N²).

The summary table on the slides places insertion sort as:

| | Best Case Runtime | Worst Case Runtime | Space | Notes |
|---|---|---|---|---|
| Selection Sort | Θ(N²) | Θ(N²) | Θ(1) | Find smallest item in remaining items, N times |
| Heapsort (in place) | Θ(N) (all duplicates) | Θ(N log N) | Θ(1)** | Bad cache (61C) performance |
| Mergesort | Θ(N log N) | Θ(N log N) | Θ(N) | Faster than heapsort |
| Insertion Sort (in place) | Θ(N) (cN inversions) | Θ(N²) | Θ(1) | Best for small N or almost sorted |

### Inversions: the right way to measure insertion sort

An **inversion** is a pair of elements that are out of order with respect to `<`.

The lecture recalls this concept and counts the inversions in `2 15 17 19 26 17 32 41 17`. The slides build the list up incrementally:

```
19-17, 19-17, 26-17, 26-17, 32-17, 41-17  = 6 total inversions
(out of 36 possible)
```

The "out of 36" is because with N = 9 elements, the maximum number of inversions is `N(N-1)/2 = 9·8/2 = 36`. That maximum comes from the sum `(N-1) + (N-2) + ... + 1`: the first element can be inverted with the N-1 elements after it, the second with N-2, and so on.

This gives an alternative way to state the goal of sorting:

- Given a sequence of elements with Z inversions,
- perform a sequence of operations that reduces inversions to 0.

### Why each insertion-sort swap fixes exactly one inversion

This is the central argument of the lecture, and it is worth internalizing because it is how you justify the Θ(N + K) bound.

When insertion sort swaps the two adjacent elements at `j - 1` and `j`:

- **Inversions among the elements that did not move** do not change, because they did not move.
- **Inversions between a moved element and an unmoved element** do not change, because their relative positions in the array stay the same. (Swapping two adjacent items does not change whether either of them is before or after any third item.)
- **The one pair we did move** was an inversion (that is why we swapped them) and is now in order.

So the total inversion count drops by exactly 1 per swap. This is why the swap count equals the inversion count: not approximately, exactly.

Concretely, from the slides:

```
2  15  17  19  26  17  32  41  17
                j   i
becomes
2  15  17  19  17  26  32  41  17
                j   i
```

One swap, one inversion (26-17) gone.

### Insertion sort's sweet spots

Putting the inversion argument together:

- **Runtime is Θ(N + K)** where K is the number of inversions. One exchange per inversion, and the comparison count is similar. The N term is the unavoidable outer loop.
- **K is on average Θ(N²) for random lists**, which is why insertion sort is a Θ(N²) algorithm in general.
- Define an **almost sorted array** as one in which the number of inversions is ≤ cN for some constant c. Insertion sort is excellent on these: Θ(N + cN) = Θ(N).

There is also a less obvious empirical fact: **for small arrays (N < 15 or so), insertion sort is the fastest sort**, period. The slides are explicit that this is an empirical result, not a theoretical one, and that the theoretical analysis is beyond the scope of the course. The rough intuition offered: divide-and-conquer algorithms like heapsort and mergesort spend too much time dividing, while insertion sort goes straight to the conquest. The Java implementation of mergesort exploits this by switching to insertion sort on small subarrays.

### Resolving the warm-up

Now the warm-up answer falls out. Start with a sorted array of 1,000,000 integers and change one element. How many inversions does that create, worst case?

Using the shorter example `[9, 12, 25, 34, 55]`:

- Change `9` to `56`: it is now inverted with every item after it, giving 4 inversions.
- Change `55` to `8`: it is now inverted with every item before it, giving 4 inversions.

In general, changing one element in a sorted array can make it inverted with at most every other element, so worst case is **N - 1 inversions**, which is Θ(N).

So the worst-case runtimes on this specific input are:

| Algorithm | Runtime on this input |
|---|---|
| Selection sort | Θ(N²) |
| Heapsort | O(N log N) |
| Mergesort | Θ(N log N) |
| **Insertion sort** | **Θ(N)** |

Insertion sort wins, and wins by a lot (a factor of roughly log N ≈ 20 against mergesort). The slide's punchline: "Number of swaps = number of inversions!"

The slides also mention bonus material on **Shell's Sort**, an optimization of insertion sort, in linked extra slides (not covered in the main lecture).

### Sorting so far: the core ideas

Before moving on, the lecture summarizes each sort by its one-line idea:

- **Selection sort**: find the smallest item and put it at the front.
  - **Heapsort variant**: use a MaxPQ to find the max element and put it at the back.
- **Mergesort**: merge two sorted halves into one sorted whole.
- **Insertion sort**: figure out where to insert the current item.
- **Quicksort**: partitioning.

Quicksort was invented by Sir Tony Hoare in 1960, when he was at the time a novice programmer.

### Quicksort's backstory: why Hoare needed a sort at all

This is not just color. The story explains what constraint produced the algorithm.

In 1960 Hoare was working on a crude automated translation program for Russian and English. Given a sentence of N words like "The cat wore a beautiful hat," and a dictionary of D English words paired with their Russian translations, produce "Кошка носил красивая шапка."

The obvious algorithm: binary search for each word.

- Find "the" in log D time.
- Find "cat" in log D time, and so on.
- Total time: **N log D**.

But that algorithm **assumes log time binary search**, and that assumption failed on 1960 hardware:

- The dictionary was stored on a long piece of tape. The sentence was an array in RAM.
- Binary search of tape is not log time, because it requires physical movement of the tape to reach an arbitrary position.

The fix: **sort the sentence, then scan the dictionary tape exactly once**, matching words off in order as the tape rolls past. That takes **N log N + D** time: N log N to sort the sentence in RAM, plus D for one linear pass over the tape.

So Hoare needed an efficient in-RAM array sort, and had to figure out how to write one without Google. The result was quicksort.

(Extra context: the lesson generalizes. Asymptotic analysis assumes a cost model, and when the real hardware violates it (tape has no random access, or in modern terms cache misses dominate), the right algorithm changes. This is the same reason the slides note heapsort has "bad cache performance," a topic taken up in CS 61C.)

### Partitioning

The main idea, stated informally first:

> I have a list of small and large items. To sort this: separate the small items and large items into two separate piles, sort the small items, sort the large items, and put the sorted lists together.

How do you decide what is big and what is small? Pick an item from the list, called the **pivot**. Smaller items are small, bigger items are big.

Formally, a **partition** of an array, given a pivot `x`, is a rearrangement of the items so that:

- All entries to the left of `x` are `<= x`.
- All entries to the right of `x` are `>= x`.
- `x` moves between the smaller and larger items.

Three things to note carefully:

1. The comparisons are `<=` and `>=`, not strict. Items equal to the pivot may legally go on either side.
2. **The relative order of items within each side is unconstrained.** Any arrangement satisfying the two conditions is a valid partition.
3. Partitioning is *not* sorting. The two halves are generally still unsorted.

For input `6 8 3 1 2 7 4 9` with pivot `6`, both of these are valid outputs:

```
3 1 2 4 [6] 9 8 7
3 4 1 2 [6] 9 7 8
```

### The partition-validity question

The slides pose a multiple-choice check. The array is `5 550 10 4 10 9 330` with `i` marking the pivot, which is the `10` at index 2. The `j` pointer in each option marks where the pivot ended up.

| Option | Result | Pivot position (j) |
|---|---|---|
| A | `4 5 9 10 10 330 550` | index 4 |
| B | `5 4 9 10 10 550 330` | index 3 |
| C | `5 9 10 4 10 550 330` | index 2 |
| D | `5 9 10 4 10 550 330` | index 2 |

Checking each against the definition (pivot value 10):

- **A** (`4 5 9 | 10 | 10 330 550`, pivot at index 4): left side is `4 5 9 10`, all ≤ 10. Right side is `330 550`, all ≥ 10. **Valid.** (Note this one happens to be fully sorted, which is allowed but not required.)
- **B** (`5 4 9 | 10 | 10 550 330`, pivot at index 3): left is `5 4 9`, all ≤ 10. Right is `10 550 330`, all ≥ 10. **Valid.**
- **C / D** (`5 9 | 10 | 4 10 550 330`, pivot at index 2): the right side contains `4`, which is **not** ≥ 10. **Invalid.**

The lesson: validity depends only on the two inequality conditions, not on how pretty or sorted the result looks. C and D fail for exactly one reason: a small element stranded on the large side.

### The job-interview framing of partitioning

The slides restate partitioning as an interview question, which is a useful way to see it as a standalone problem:

> Given an array of colors where the 0th element is white (and maybe a few more), and the remaining elements are red (less) or blue (greater), rearrange the array so that all red squares are to the left of the white square, the white squares end up together, and all blue squares are to the right. Your algorithm must complete in Θ(N) time (no space restriction). The relative order of reds and blues does NOT need to stay the same.

Here white = equal to the pivot, red = less than, blue = greater than. The input `6 8 3 1 2 7 4 6` (two 6s, so two whites) admits outputs like `3 1 2 4 6 6 8 7` or `3 4 1 2 6 6 7 8`.

Two details matter: the **Θ(N) time requirement** (so you cannot sort as a subroutine), and the **no space restriction** (so a simple three-array approach is permitted). The lecture does not present a specific partitioning implementation; it establishes the goal and the validity conditions.

### Quicksort: partition sort

With partitioning in hand, the algorithm is short. Quick sorting N items:

1. Partition on the leftmost item.
2. Quicksort the left half.
3. Quicksort the right half.

The crucial observation that makes this work: after partitioning, **the pivot is in its place**. Exactly where it would be if the array were sorted. Nothing will ever need to move it again. And since everything left of it is ≤ it and everything right of it is ≥ it, the two sides can be sorted completely independently.

Using the slides' smaller example, input `5 3 2 1 7 8 4 6` with pivot `5`:

```
5 3 2 1 7 8 4 6     partition on 5
3 2 1 4 [5] 7 8 6
```

Now `5` is in its place. Recursively sort each side:

```
left:  3 2 1 4  ->  2 1 3 4 5   (partition on 3, recurse)
right: 7 8 6    ->  5 6 7 8     (partition on 7, recurse)
```

And on the lecture's running example `32 15 2 17 19 26 41 17 17`, partitioning on the leftmost item `32`:

```
32 15 2 17 19 26 41 17 17
         partition(32)
15 2 17 19 26 17 17 [32] 41
   <= 32             in its   >= 32
                     place
```

The recursion tree for the full sort looks like this (`x` marks a base case, a subarray too small to need work):

```
                              partition(32)
                   partition(15)
        partition(2)          partition(17)
        x         x           x
                                     partition(19)
                            partition(17)      partition(26)
                             x                  x         x
                                    partition(17)
                                    x          x
```

Final result: `2 15 17 17 17 19 26 32 41`, with every element in its place.

### The interesting twist

The slides close on this. Quicksort was the name Hoare chose for partition sort, and for most common situations it is **empirically the fastest sort**. The slides remark drily that "Tony was lucky that the name was correct."

How fast is quicksort? You need to count the number and difficulty of the partition operations. The interesting twist, which is the hook for the next lecture: **the overall runtime will depend crucially on where the pivot ends up.**

The intuition to carry forward: if the pivot lands near the middle every time, you halve the problem at each level, giving log N levels of Θ(N) partitioning work. If the pivot lands at an end every time (for example, if you always pick the leftmost item and the array is already sorted), you only peel off one element per level, giving N levels of Θ(N) work. The lecture states only that the runtime depends on pivot position; the full analysis comes later.

## Definitions

- **Insertion sort**: A sorting algorithm that starts with an empty output sequence and adds each item from the input, inserting it into the output at the right point.
- **Naive insertion sort**: The variant that builds an entirely new output array, separate from the input. If the output contains k items, the worst cost to insert a single item is k, because everything may need to move over.
- **In-place insertion sort**: The variant that does everything in place using swapping, requiring Θ(1) extra space. For i = 0 to N - 1, insert the item at index i into the right place among the sorted items by swapping it backwards.
- **Traveller**: The item at index `i` that is currently being inserted into the sorted region, moving leftward by repeated swaps.
- **`i` pointer**: Marks the boundary between the sorted (examined) region and the unexamined region in in-place insertion sort.
- **`j` pointer**: Tracks the current spot of the travelling item as it swaps leftward.
- **Inversion**: A pair of elements that are out of order with respect to `<`.
- **Maximum number of inversions**: For N elements, `(N-1) + (N-2) + ... + 1 = N(N-1)/2`.
- **Sorting, restated via inversions**: Given a sequence of elements with Z inversions, perform a sequence of operations that reduces the number of inversions to 0.
- **Almost sorted array**: An array in which the number of inversions is ≤ cN for some constant c. Insertion sort is excellent on these arrays.
- **Pivot**: An item picked from the list to serve as the dividing value in a partition. Items smaller than it are "small" and items bigger are "big."
- **Partition** (of an array, given a pivot `x`): A rearrangement of the items so that all entries to the left of `x` are `<= x`, all entries to the right of `x` are `>= x`, and `x` sits between the smaller and larger items.
- **Quicksort** (also called partition sort): Partition on the leftmost item, then quicksort the left half, then quicksort the right half. Named by Tony Hoare, who invented it in 1960.

## Worked Examples

### Example 1: In-place insertion sort, the full implementation

The slides give the inner loop condition explicitly. Here is the complete method built around it.

```java
public class InsertionSort {
    public static void sort(int[] input) {
        for (int i = 0; i < input.length; i += 1) {
            int j = i;
            while (j > 0 && input[j - 1] > input[j]) {
                swap(input, j - 1, j);
                j -= 1;
            }
        }
    }

    private static void swap(int[] arr, int a, int b) {
        int temp = arr[a];
        arr[a] = arr[b];
        arr[b] = temp;
    }
}
```

What each piece does and why:

- **`for (int i = 0; ...)`**: This is the "Repeat for i = 0 to N - 1" from the slides. After iteration `i` completes, everything in `input[0..i]` is sorted relative to itself. That is the invariant.
- **`int j = i;`**: The traveller starts at the boundary. The slides are careful to draw `i` and `j` at the same index at the start of each outer iteration, then show `j` walking left while `i` stays put.
- **`j > 0`**: The guard. When `j == 0` the traveller is at the front of the array; there is nothing further left to compare against, and evaluating `input[j - 1]` would index `-1`. Java's `&&` short-circuits, so `input[j - 1]` is never reached when `j == 0`.
- **`input[j - 1] > input[j]`**: Strict `>`. We swap only when the left neighbour is genuinely larger. If it is equal or smaller, the traveller has found its place and the loop exits.
- **`swap(...); j -= 1;`**: One swap moves the traveller one position left, and `j` follows it. These two lines together are the "swap backwards" step.

**Environment and box-and-pointer reasoning** (in words, since the array is primitive here): `input` is a local variable in `sort`'s frame holding a reference (an arrow) to a single `int[]` object on the heap. `swap` receives a copy of that arrow in its own frame, so both frames point at the same array object. Mutating `arr[a]` inside `swap` changes the one shared heap object, which is exactly why the in-place version needs no return value. The `i` and `j` variables are plain ints living in `sort`'s frame; they are indices, not pointers, so when the slides draw them as arrows under the array they are picturing index values, not Java references.

Why `i` never moves backwards even though the traveller does: `i` is the loop variable of the outer loop, so it is monotonically increasing. Only `j` walks left. Conflating them is a classic bug (see Pitfalls).

**Why this is Θ(1) space**: the only extra storage is `i`, `j`, and `temp` inside `swap`, all constant regardless of N. Contrast with mergesort's Θ(N) auxiliary array.

### Example 2: Tracing the `i = 7` step by hand

The slides spend several frames on this one step, asking "How many swaps do we need to do to get 17 in the correct place?"

State at the start of `i = 7`:

```
index:  0   1   2   3   4   5   6   7   8
array:  2  15  17  19  26  32  41  17  17
                                    i,j
        [-------- sorted --------]
```

The traveller is `input[7] == 17`. Trace the while loop:

| Iteration | j | `input[j-1]` | `input[j]` | `>` ? | Action | Array after |
|---|---|---|---|---|---|---|
| 1 | 7 | 41 | 17 | yes | swap(6,7), j→6 | `2 15 17 19 26 32 17 41 17` |
| 2 | 6 | 32 | 17 | yes | swap(5,6), j→5 | `2 15 17 19 26 17 32 41 17` |
| 3 | 5 | 26 | 17 | yes | swap(4,5), j→4 | `2 15 17 19 17 26 32 41 17` |
| 4 | 4 | 19 | 17 | yes | swap(3,4), j→3 | `2 15 17 17 19 26 32 41 17` |
| 5 | 3 | 17 | 17 | **no** | exit loop | (unchanged) |

**4 swaps total**, matching the slides' count. Note the loop exits because `17 > 17` is false, not because of the `j > 0` guard. This is the strictness of `>` doing its job: the traveller comes to rest immediately to the right of the equal element rather than continuing past it.

Also worth noticing: the sorted region was `2 15 17 19 26 32 41` (7 items) and the traveller went 4 positions. It did not cost 7 swaps. The cost per iteration is the number of items the traveller must pass, which is exactly the number of inversions involving the traveller and the sorted region.

### Example 3: Counting inversions and predicting the swap count

The slides count inversions in this array:

```java
int[] a = {2, 15, 17, 19, 26, 17, 32, 41, 17};
```

A brute-force inversion counter makes the definition concrete:

```java
public static int countInversions(int[] a) {
    int count = 0;
    for (int x = 0; x < a.length; x += 1) {
        for (int y = x + 1; y < a.length; y += 1) {
            if (a[x] > a[y]) {
                count += 1;
            }
        }
    }
    return count;
}
```

This checks every pair `(x, y)` with `x < y` and counts it when the earlier element is larger. For the array above:

- `19` at index 3 is inverted with the `17` at index 5 and the `17` at index 8: **19-17, 19-17**
- `26` at index 4 is inverted with both later `17`s: **26-17, 26-17**
- `32` at index 6 is inverted with the `17` at index 8: **32-17**
- `41` at index 7 is inverted with the `17` at index 8: **41-17**

Total: **6 inversions**, out of a maximum of `9·8/2 = 36`.

This matches the slides exactly. And by the one-swap-per-inversion argument, running the in-place insertion sort on this array will perform **exactly 6 swaps**. That is a testable prediction: instrument `swap` with a counter and the counter will read 6.

Note what `countInversions` is *not*: it is a Θ(N²) diagnostic tool for understanding the concept, not part of any sorting algorithm. You would never run it inside insertion sort.

### Example 4: Validating a partition in code

The partition-validity question from the slides can be checked mechanically. Given the partitioned array and the index where the pivot landed:

```java
public static boolean isValidPartition(int[] arr, int pivotIndex) {
    int pivot = arr[pivotIndex];
    for (int k = 0; k < pivotIndex; k += 1) {
        if (arr[k] > pivot) {
            return false;
        }
    }
    for (int k = pivotIndex + 1; k < arr.length; k += 1) {
        if (arr[k] < pivot) {
            return false;
        }
    }
    return true;
}
```

Walking the slides' four options with pivot value 10:

```java
// Option A: 4 5 9 10 10 330 550, pivot at index 4
// left: 4, 5, 9, 10 -- none > 10. right: 330, 550 -- none < 10. TRUE

// Option B: 5 4 9 10 10 550 330, pivot at index 3
// left: 5, 4, 9 -- none > 10. right: 10, 550, 330 -- none < 10. TRUE

// Option C/D: 5 9 10 4 10 550 330, pivot at index 2
// left: 5, 9 -- none > 10. right: 4 -- 4 < 10. FALSE
```

The structure of this checker mirrors the definition line for line: left side uses `> pivot` (violating `<=`), right side uses `< pivot` (violating `>=`). Notice there is no check on the *order* within each side, because the definition imposes none. That is why Option B passes despite `5 4 9` being unsorted.

### Example 5: Quicksort built on a partition routine

The slides describe quicksort in three lines but do not give a partition implementation. Here is the recursive skeleton exactly as described (partition on leftmost item, recurse on both halves), with the partition step left as a described contract:

```java
public class Quicksort {
    public static void sort(int[] arr) {
        sort(arr, 0, arr.length - 1);
    }

    private static void sort(int[] arr, int lo, int hi) {
        if (lo >= hi) {
            return;
        }
        int pivotIndex = partition(arr, lo, hi);
        sort(arr, lo, pivotIndex - 1);
        sort(arr, pivotIndex + 1, hi);
    }

    // Partitions arr[lo..hi] around arr[lo] as the pivot, returns the
    // index where the pivot ends up. Not implemented in this lecture.
    private static int partition(int[] arr, int lo, int hi) {
        throw new UnsupportedOperationException("See next lecture.");
    }
}
```

The three key design points:

- **`if (lo >= hi) return;`** is the base case. A subarray of size 0 or 1 is already sorted. These are the `x` markers in the slides' recursion tree.
- **`partition` returns `pivotIndex`**, and the recursive calls deliberately **exclude** it: `[lo, pivotIndex - 1]` and `[pivotIndex + 1, hi]`. This is the "in its place" observation made operational. The pivot is never touched again, which is also what guarantees the recursion makes progress (each level removes at least one element from consideration).
- **`arr[lo]` is the pivot**, matching the slides' "Partition on leftmost item."

Tracing the slides' example `5 3 2 1 7 8 4 6` through `sort(arr, 0, 7)`:

1. `partition` with pivot `5` yields `3 2 1 4 [5] 7 8 6` and returns `pivotIndex == 4`.
2. `sort(arr, 0, 3)` handles `3 2 1 4`: partition on `3` gives `2 1 [3] 4`, then recurse on `2 1` and on `4`.
3. `sort(arr, 5, 7)` handles `7 8 6`: partition on `7` gives `6 [7] 8`, then recurse on `6` and on `8`.
4. All subcalls bottom out at size ≤ 1. Array is `1 2 3 4 5 6 7 8`.

(Extra context: this skeleton assumes `partition` places the pivot at the returned index and satisfies the `<=` / `>=` conditions. Equal elements landing on either side is still correct here, because the recursive calls sort both sides fully.)

### Example 6: Why insertion sort beats mergesort on the warm-up input

Making the warm-up concrete in code:

```java
int N = 1000000;
int[] a = new int[N];
for (int k = 0; k < N; k += 1) {
    a[k] = k;              // sorted: 0, 1, 2, ..., 999999
}
a[0] = N;                  // worst-case single change
```

Setting `a[0] = N` makes element 0 larger than every other element, so it is inverted with all `N - 1` of them. That is the maximum damage a single-element change can do, confirming the slides' answer of **N - 1 inversions**.

Running in-place insertion sort on this:

- `i = 0`: 0 swaps.
- `i = 1`: traveller is `1`, left neighbour is `1000000`. One swap, then `1 > 0`? The array is now `1 1000000 2 3 ...`, and the traveller is at index 0 so the `j > 0` guard stops it. **1 swap.**
- `i = 2`: traveller is `2`, left neighbour is `1000000`. 1 swap, then stops against `1`. **1 swap.**
- ... and so on. Each of the remaining iterations performs exactly 1 swap, pushing the big value one step right.
- Total: `N - 1 = 999,999` swaps, plus N outer iterations. **Θ(N).**

Mergesort on the same input does its full Θ(N log N) work, roughly 20 million operations, because its recursion structure is input-independent. Insertion sort does roughly 1 million. The structural knowledge wins.

## Common Pitfalls

**Forgetting the `j > 0` guard, or writing it second.** `while (input[j - 1] > input[j] && j > 0)` compiles but throws `ArrayIndexOutOfBoundsException` when the traveller reaches the front, because `&&` evaluates left to right and `input[-1]` is reached first. The slides' answer puts `j > 0` first for exactly this reason.

**Decrementing `i` instead of `j`.** The traveller's position is `j`; the sorted-region boundary is `i`. If you write `i -= 1` inside the while loop, the outer loop never advances past the first few indices and you get an infinite loop or a scrambled array. Keeping the two roles distinct is why the slides ask "What new names would you propose for i and j?": `i` is the boundary, `j` is the current spot of the travelling item.

**Claiming the best case is Θ(1).** On an already-sorted array no swaps happen, but the outer loop still runs N times and performs N - 1 comparisons. Best case is **Θ(N)**, choice D on the slide, not choice C. You cannot know an array is sorted without looking at all of it.

**Thinking the invariant holds at every instant.** The slides explicitly call this out: mid-flight, "We've temporarily broken our invariant that the items up through item i should be sorted!" The invariant holds at the *end* of each outer iteration, once the traveller settles. Expecting it mid-swap leads to confusion when hand-tracing.

**Confusing "number of inversions" with "number of elements out of place."** In `2 15 17 19 26 17 32 41 17`, only three elements are arguably "misplaced," but there are 6 inversions. Inversions count *pairs*, and one misplaced element can generate many inversions. The swap count tracks pairs, not elements.

**Assuming a partition must be sorted, or must be unsorted.** Option A in the validity quiz (`4 5 9 10 10 330 550`) happens to be fully sorted and is valid. Option B (`5 4 9 10 10 550 330`) is unsorted on both sides and is equally valid. Validity depends only on the two inequality conditions. A fully sorted result is allowed, not required, and not a signal that something went wrong.

**Thinking partitioning sorts the array.** Partitioning places exactly *one* element (the pivot) in its final position. Everything else is merely on the correct side. The sorting comes from recursion.

**Using strict `<` and `>` in the partition definition.** The definition uses `<=` on the left and `>=` on the right, so duplicates of the pivot may go on either side. Option B puts a second `10` on the right of the pivot `10` and is still valid. Requiring strict inequality would make many legal partitions look wrong, and would be impossible when the array has duplicate pivots.

**Expecting quicksort to recurse on the pivot.** The recursive calls are `[lo, pivotIndex - 1]` and `[pivotIndex + 1, hi]`. Including `pivotIndex` in either range means the subproblem never shrinks on inputs where the pivot lands at an end, giving infinite recursion.

**Assuming quicksort is always fast because it is "empirically the fastest sort."** The slides end by flagging the opposite: "the overall runtime will depend crucially on where the pivot ends up." The empirical speed claim is for "most common situations," and the lecture deliberately leaves the bad cases for later.

**Mis-reading the Hoare story as about binary search being slow.** Binary search is log time. The problem was that *tape* does not support random access, so the log-time analysis did not apply to that storage medium. The algorithm was fine; the cost model was wrong.

## Likely Exam Points

### 1. Best and worst case runtime of insertion sort

This is asked directly on the slides as a multiple-choice question.

**Q:** What are the best- and worst-case runtimes of in-place insertion sort, and give an input achieving each?

**A:** Best Θ(N), worst Θ(N²). Best case: an already-sorted array, where each outer iteration does 1 comparison and 0 swaps, for N iterations total. Worst case: a reverse-sorted array, where the traveller at index `i` swaps `i` times, totalling `N(N-1)/2 = Θ(N²)`. The best case is Θ(N) and not Θ(1) because the outer loop must still visit every element.

### 2. Counting swaps on a given array

The POTATO and SORTEXAMPLE traces are drilled frame by frame, strongly suggesting a hand-trace question.

**Q:** Run in-place insertion sort on `D A C B` and report the array after each outer iteration plus the swap count for that iteration.

**A:**

| i | Array after | Swaps |
|---|---|---|
| 0 | `D A C B` | 0 |
| 1 | `A D C B` | 1 |
| 2 | `A C D B` | 1 |
| 3 | `A B C D` | 2 |

Total 4 swaps. Check against the inversion count of `D A C B`: D-A, D-C, D-B, C-B = 4 inversions. They match, as they must.

### 3. Counting inversions, including the maximum

**Q:** How many inversions are in `3 1 4 1 5`? What is the maximum possible number of inversions for an array of 5 elements, and how many swaps will insertion sort perform on this input?

**A:** Pairs `(x, y)` with `x < y` and `a[x] > a[y]`: 3-1 (indices 0,1), 3-1 (indices 0,3), 4-1 (indices 2,3). That is **3 inversions**. Maximum for N = 5 is `N(N-1)/2 = 10`. Insertion sort performs exactly **3 swaps**, since each swap fixes exactly one inversion.

### 4. Proving one swap removes exactly one inversion

This is the central argument of the lecture and is the kind of short proof that shows up as a written question.

**Q:** Explain why each swap in insertion sort reduces the inversion count by exactly 1.

**A:** The swap exchanges two adjacent elements at indices `j - 1` and `j`. Inversions among elements that did not move are unaffected, since those elements did not move. Inversions between a moved element and an unmoved element are unaffected, since swapping two adjacent items does not change either one's relative position with respect to any third item. The one pair that did move was an inversion (that is the swap condition) and is now in order. So the count drops by exactly 1.

### 5. Picking the best sort for a structured input

The lecture's warm-up and its resolution bracket the entire first half.

**Q:** You start with a sorted array of N integers and change one element to an arbitrary new value. Among selection sort, heapsort, mergesort, and insertion sort, which is asymptotically fastest, and why?

**A:** Insertion sort, at Θ(N). Changing one element in a sorted array creates at most N - 1 inversions (the changed element can be inverted with every other element, but no pair of unchanged elements is inverted). Insertion sort runs in Θ(N + K) with K the inversion count, so Θ(N + N) = Θ(N). Selection sort is Θ(N²) regardless of input, and heapsort and mergesort are Θ(N log N) regardless of input, since none of the three adapts to existing order.

### 6. Identifying valid partitions

A direct multiple-choice item on the slides.

**Q:** The array `7 2 9 3 8` is partitioned around pivot value 7. Which of these are valid partitions? (a) `2 3 7 9 8` (b) `3 2 7 8 9` (c) `2 9 7 3 8` (d) `2 3 7 7 9`

**A:** (a) valid: left `2 3` all ≤ 7, right `9 8` all ≥ 7. (b) valid: left `3 2` all ≤ 7 (order within a side is unconstrained), right `8 9` all ≥ 7. (c) invalid: `9` is left of the pivot but `9 > 7`, and `3` is right of the pivot but `3 < 7`. (d) invalid as a rearrangement: it contains a second 7 that was not in the input, so it is not a permutation of the original items.

### 7. What partitioning guarantees

**Q:** After one partition operation on an array of N elements, how many elements are guaranteed to be in their final sorted position?

**A:** Exactly one, the pivot. All other elements are only guaranteed to be on the correct side of the pivot, in no particular order. The rest of the sorting comes from recursively quicksorting the two sides.

### 8. The quicksort recursion structure

**Q:** Write the quicksort recursion as described in lecture, and explain why the recursive calls exclude the pivot index.

**A:**

```java
private static void sort(int[] arr, int lo, int hi) {
    if (lo >= hi) { return; }
    int p = partition(arr, lo, hi);
    sort(arr, lo, p - 1);
    sort(arr, p + 1, hi);
}
```

The pivot is excluded because after partitioning it is already in its final sorted position; everything left of it is ≤ it and everything right is ≥ it, so no later operation should move it. Excluding it also guarantees each recursive subproblem is strictly smaller than its parent, which is what makes the recursion terminate.

### 9. Why insertion sort for small N, and where this is used

**Q:** Insertion sort is Θ(N²) and mergesort is Θ(N log N), yet real mergesort implementations call insertion sort. Why?

**A:** For small arrays (N under roughly 15), insertion sort is empirically the fastest sort. The rough reason given in lecture: divide-and-conquer algorithms like mergesort and heapsort spend too much time dividing, while insertion sort goes straight to the conquest. Asymptotic notation hides the constant factors that dominate at small N. Java's mergesort implementation switches to insertion sort on small subarrays for exactly this reason. The lecture notes the theoretical analysis of this is beyond the scope of the course.

### 10. The sorts comparison table

Filling in this table is a near-certain exam item given that the slides present it as cumulative.

**Q:** Give best-case runtime, worst-case runtime, and space for selection sort, heapsort, mergesort, and insertion sort.

**A:**

| | Best | Worst | Space |
|---|---|---|---|
| Selection sort | Θ(N²) | Θ(N²) | Θ(1) |
| Heapsort (in place) | Θ(N) (all duplicates) | Θ(N log N) | Θ(1) |
| Mergesort | Θ(N log N) | Θ(N log N) | Θ(N) |
| Insertion sort (in place) | Θ(N) (cN inversions) | Θ(N²) | Θ(1) |

Insertion sort is the only one on this list with a Θ(1)-space, Θ(N)-best-case profile that adapts to input order, which is what makes it the right answer to the warm-up.

### 11. The Hoare backstory as a cost-model question

**Q:** Hoare's translator could have used N binary searches of a D-word dictionary for N log D total time. Why was sorting the sentence (N log N + D) faster in practice?

**A:** The N log D analysis assumes binary search is log time, which requires random access. The dictionary was on a long tape, and seeking an arbitrary tape position requires physical movement, so binary search on tape is not logarithmic. Sorting the sentence in RAM (N log N) and then scanning the tape exactly once in order (D) avoids all tape seeking. This is why Hoare needed a fast array sort and ended up inventing quicksort.

## Summary

- **Insertion sort's strategy**: start with an empty output, and insert each input item at the right point.
- The **naive** version builds a whole new output array; inserting into a k-item output costs up to k because everything may need to shift. The **in-place** version does it with swaps and Θ(1) extra space.
- **In-place loop**: for i = 0 to N - 1, let `j = i` (the traveller), then `while (j > 0 && input[j - 1] > input[j])` swap backwards. The `j > 0` guard must come first (short-circuit `&&`).
- `i` is the sorted/unexamined boundary; `j` is the traveller's current spot. The invariant "items up through i are sorted" is temporarily broken mid-flight and restored when the traveller settles.
- Traced examples: the lecture's `32 15 2 17 19 26 41 17 17`, plus `POTATO` (7 swaps) and `SORTEXAMPLE` (36 swaps).
- **Runtime: best Θ(N)** (already sorted: N iterations, 0 swaps), **worst Θ(N²)** (reverse sorted: `N(N-1)/2` swaps). Best is Θ(N), not Θ(1).
- An **inversion** is a pair out of order with respect to `<`. Maximum for N elements is `N(N-1)/2`. Sorting = reducing inversions to 0.
- **Each insertion-sort swap removes exactly one inversion**: unmoved pairs are unaffected, moved-vs-unmoved relative positions are unaffected, and the one swapped pair is fixed.
- So **insertion sort runs in Θ(N + K)** with K inversions. K averages Θ(N²) on random lists, but an **almost sorted array** (K ≤ cN) sorts in Θ(N).
- Empirically, insertion sort is the **fastest sort for N under roughly 15**; Java's mergesort switches to it for small subarrays.
- **Warm-up answer**: a sorted array with one element changed has at most N - 1 inversions, so insertion sort at Θ(N) beats heapsort and mergesort at Θ(N log N) and selection sort at Θ(N²).
- **Quicksort** was invented by Tony Hoare in 1960 for a Russian-English translator. Binary searching a tape-stored dictionary is not log time (tape needs physical movement), so sorting the sentence and scanning the tape once (N log N + D) beat N binary searches (N log D).
- A **partition** around pivot `x` rearranges the array so everything left of `x` is `<= x`, everything right is `>= x`, and `x` sits between them. Order within each side is unconstrained; duplicates of the pivot may go either way.
- After a partition, **the pivot is in its final sorted position** and never moves again. That is the one guarantee a partition gives.
- **Quicksort**: partition on the leftmost item, quicksort the left half, quicksort the right half. Recursive calls exclude the pivot index.
- Quicksort is empirically the fastest sort in most common situations, but the **runtime depends crucially on where the pivot ends up**, which is the hook for the next lecture.
