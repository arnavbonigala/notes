<!-- Mon, Sep 14, 2026 | sources: code (no transcript available) -->
# Lecture 8: Lists 4

This lecture finishes the array-based list (`AList`) and turns it into something that is actually fast. We start from the naive `AList` that resizes by exactly one slot every time it fills up (`resize(size + 1)`), observe experimentally that this is catastrophically slow (inserting 100,000 items takes seconds, while an `SLList` doing the same work finishes instantly), and diagnose why: additive resizing forces us to copy the entire array on every single `addLast`, so N inserts cost roughly 1 + 2 + 3 + ... + N ≈ N²/2 array-box copies. The fix is **geometric resizing**: multiply the capacity instead of adding to it (`resize(size * 2)` or `resize((int) (size * 1.1))`), which makes copying rare enough that the average cost per `addLast` becomes constant. Along the way we make `AList` generic (with the awkward `(T[]) new Object[n]` cast that Java forces on us), discuss the "usage ratio" and downsizing so a list that shrinks does not hoard memory, and discuss **loitering**: nulling out removed references so the garbage collector can actually reclaim them. The lecture also introduces the experimental methodology used throughout: timing with `System.currentTimeMillis()`, and printing a table of N vs. time where N grows by factors of 10, which lets you *see* linear vs. quadratic growth in the numbers.

## Key Concepts

### 1. The array-based list, and the meaning of `size` vs. `length`

An `AList` stores items in a Java array `items` plus an `int size`. These are two very different numbers, and confusing them is the single most common source of `AList` bugs:

- `items.length` is the **capacity**: how many boxes the underlying array physically has. It is fixed at array-creation time and can never change.
- `size` is the **number of items the list logically contains**. Everything from `items[0]` to `items[size - 1]` is real data; everything from `items[size]` to `items[items.length - 1]` is junk (zeros or `null`s) that the user must never see.

The lecture code states the two key invariants in a comment:

```java
// [3, 4, 2, 0, 0, 0, ....]
//           ^ (size = 3)
// size is the location of the next add
// size - 1 location of the last item
```

That is the whole design in two lines. `addLast` writes at `items[size]`, then increments; `getLast` reads `items[size - 1]`; `removeLast` decrements `size`.

### 2. Why `removeLast` is (almost) free

```java
public int removeLast() {
    int itemToReturn = getLast();
    size -= 1;
    return itemToReturn;
}
```

Nothing is erased. The item is still physically sitting in the array, but because `size` shrank, it is now in the "junk" region past the end of the list, so no legal operation can observe it, and the next `addLast` will overwrite it. This is why array lists are so fast at the back end: the entire deletion is one decrement. (For a *generic* `AList` this exact code is a memory bug, see loitering below.)

### 3. Arrays cannot grow, so we fake it

Java arrays have a fixed length. There is no way to make `new int[100]` become `new int[101]`. So "resizing" is a misnomer, what we really do is:

1. Allocate a brand new, bigger array.
2. Copy the existing items over.
3. Reassign `items` to point at the new array.
4. Let the old array become garbage.

Both versions of the copy appear in this lecture's code. The generic `AList` copies by hand:

```java
private void resize(int capacity) {
    T[] resized = (T[]) new Object[capacity];
    for (int i = 0; i < size; i++) {
        resized[i] = items[i];
    }
    items = resized;
}
```

and the speedtest `AList` uses the built-in library call, which does exactly the same thing but faster:

```java
private void resize(int newSize) {
    int[] a = new int[newSize];
    System.arraycopy(items, 0, a, 0, size);
    items = a;
}
```

`System.arraycopy(src, srcPos, dest, destPos, length)` copies `length` items starting at `src[srcPos]` into `dest` starting at `destPos`. Note we copy `size` items, not `items.length` items: copying the junk region would be a waste.

In box-and-pointer terms: before `resize`, the `AList` object's `items` box holds an arrow into an old array object. After `resize`, that box holds an arrow into a new, longer array object whose first `size` boxes hold copies of the old contents (for a generic list, copies of the *references*, not of the objects themselves). The old array now has no arrows pointing to it and is eligible for garbage collection.

### 4. The naive strategy and why it is quadratic

The `AList.java` in `lec8_lists4` (the generic one) does the naive thing:

```java
if (size == items.length) {
    resize(size + 1);
}
```

This grows the array by exactly one box each time it is full. Once the array is full, it is full again immediately after the next insert, so *every subsequent* `addLast` triggers a full copy of the whole array.

Count the work for N inserts starting from a full array: the 1st insert copies 1 item, the 2nd copies 2, ..., the Nth copies N. Total ≈ N(N+1)/2, which is quadratic in N. Each individual `addLast` therefore costs time proportional to the current size (linear), not constant.

The same problem occurs with `resize(size + RFACTOR)` for any constant like 1000: you only amortize the cost over 1000 inserts, so the total is still ≈ N²/(2·1000), still a parabola, just with a smaller constant. **Adding a constant is never enough; you must multiply.**

The textbook's graph makes this visual: plotting total time vs. number of operations gives a *straight line* for `SLList.addFirst` (constant time per op, since the integral of a constant is a line) and a *parabola* for the naive array list (linear time per op, since the integral of a line is a parabola). For 100,000 items the naive list does on the order of 100,000 times more work.

### 5. Geometric resizing: the fix

```java
if (size == items.length) {
    resize(size * 2);        // or (int) (size * 1.1)
}
```

Now the capacity doubles: 100, 200, 400, 800, ... Resizes become exponentially rarer as the list grows. To reach size N you perform about log₂(N) resizes, and the copies cost N/2 + N/4 + N/8 + ... < N boxes *in total*. So N inserts cost O(N) work overall, meaning an **average** of constant work per insert, even though individual inserts occasionally do a lot of work. This "expensive rarely, cheap usually, constant on average" idea is called amortized constant time; the lecture uses the geometric strategy and defers the full formal analysis to later in the course.

The lecture code enumerates the candidate strategies side by side so you can compare them:

```java
// other resizing strategies:
//  resize(size + 1000);          // additive: still quadratic overall
//  resize(size * 2);             // geometric: linear overall
//  resize((int) (size * 1.1));   // geometric with a smaller factor
```

`size * 1.1` is still geometric (still multiplicative), so it is still amortized constant time; the tradeoff is that it does about 7x more resize events than doubling (log base 1.1 instead of log base 2) but wastes at most ~10% of memory instead of up to ~50%.

> Caution (extra context): `resize((int) (size * 1.1))` only works because the starting capacity is 100. If `size` were small, `(int)(size * 1.1)` could round back down to `size`, producing a "new" array of the same length and an infinite loop or an out-of-bounds crash. Robust implementations use something like `Math.max(size + 1, (int)(size * 1.1))`.

### 6. Memory performance: the usage ratio and downsizing

Geometric growth solves time but creates a space problem in the other direction. Insert 1,000,000,000 items, then remove 990,000,000 of them: `size` is now 10,000,000 but `items.length` is still around a billion, so 99% of the memory is wasted.

Define the **usage ratio** R = size / items.length. A typical implementation halves the array's capacity when R drops below 0.25. Why 0.25 rather than 0.5? (extra context) If you halved at exactly R = 0.5, a workload that alternates `addLast`/`removeLast` at the boundary would resize on every single operation, destroying the amortized guarantee. Leaving a gap between the grow threshold and the shrink threshold prevents this thrashing.

### 7. Generics in an array-backed list

Making `AList` generic is mostly the same as for `SLList`: put `<T>` after the class name and replace `int` with `T`:

```java
public class AList<T> {
    public T[] items;
    public int size;
```

But Java forbids creating an array of a generic type. You cannot write `new T[11]`. Instead you must write:

```java
items = (T[]) new Object[11];
```

This creates an array of `Object` and lies to the compiler about its type. It produces an unchecked-cast compilation **warning**, not an error, and the course's position for now is that we simply live with it (the underlying reason, type erasure, is covered in a later chapter).

Also note the generic `SLList` in this lecture uses `Mustard` as the type parameter name:

```java
public class SLList<Mustard> {
    private class MustardNode {
        public Mustard item;
        public MustardNode next;
```

The point of choosing a silly name is pedagogical: the type parameter is just a placeholder identifier, there is nothing magic about `T` or `E`. (`T` is the conventional choice in real code.)

### 8. Loitering

For an `int[]`, `removeLast` can leave the stale value in the array harmlessly. For a `T[]` holding object references, it cannot. Java's garbage collector reclaims an object only when the **last reference to it is lost**. If a deleted item's reference is still sitting in `items[size]`, the array keeps that object alive forever even though the list "no longer contains" it. That is **loitering**: memory held by a reference we will never use again.

The fix is to null out the slot when deleting:

```java
public T removeLast() {
    T itemToReturn = items[size - 1];
    items[size - 1] = null;   // prevents loitering
    size -= 1;
    return itemToReturn;
}
```

The textbook calls this a subtle bug you are unlikely to notice unless you look for it, but one that can waste significant memory. (Note the `AList<T>` in this lecture's code has no `removeLast` yet, so there is nothing to null out there; the point applies as soon as you add one.)

### 9. Measuring performance experimentally

The lecture's speedtest classes establish the methodology used for the rest of the course:

- Record `System.currentTimeMillis()` before and after, subtract to get elapsed milliseconds.
- Run one fixed N (100,000) to get a single number, as in `SpeedTestAList` / `SpeedTestSLList`.
- Better: run a *table* of increasing N, multiplying N by 10 each round and stopping once a round exceeds 5000 ms, as in `SpeedTestAListTable` / `SpeedTestSLListTable`.

The table is the important idea. If time is linear in N, multiplying N by 10 multiplies the time by about 10. If time is quadratic, multiplying N by 10 multiplies the time by about 100. You can read the asymptotic behavior straight off the column of numbers without ever plotting anything.

Note the comparison is deliberately apples-to-apples on cost, not on operation name: `SpeedTestSLList` uses `addFirst` (which is genuinely constant time for a singly linked list) while `SpeedTestAList` uses `addLast`. The `SLList` in this package has no sentinel-tail/`last` pointer and no caching, so its `addLast` walks the entire list with `while (p.next != null) p = p.next;`, which is itself linear and would give a quadratic total. `addFirst` is the fair representative of "fast linked list operation."

## Definitions

- **AList (array list):** A list implementation that stores items in a contiguous backing array `items` together with an `int size`, where the list's contents are exactly `items[0]` through `items[size - 1]`.
- **size:** The number of items currently in the list. Also the index at which the next `addLast` will write.
- **length / capacity:** `items.length`, the physical number of boxes in the backing array. Always ≥ `size`.
- **Resizing:** Creating a new array of a different length, copying the existing `size` items into it, and reassigning the `items` reference to the new array. The old array is not changed, it is discarded.
- **Additive (naive) resizing:** Growing the array by a fixed constant number of boxes, e.g. `resize(size + 1)` or `resize(size + 1000)`. Leads to Θ(N²) total work for N inserts.
- **Geometric (multiplicative) resizing:** Growing the array by a multiplicative factor RFACTOR > 1, e.g. `resize(size * 2)` or `resize((int)(size * 1.1))`. Leads to Θ(N) total work for N inserts, i.e. constant time per insert on average.
- **RFACTOR:** The resizing factor, the constant used in the resizing rule (added in the naive scheme, multiplied in the geometric scheme).
- **Usage ratio (R):** R = size / items.length, the fraction of the backing array actually in use. A typical implementation halves the capacity when R < 0.25.
- **Loitering:** Retaining a reference to an object that the program will never use again, preventing the garbage collector from reclaiming it. Avoided by setting deleted array slots to `null`.
- **Garbage collection:** Java's automatic reclamation of objects once the last reference to them is lost.
- **`System.arraycopy(src, srcPos, dest, destPos, length)`:** Library method that copies `length` elements from `src` beginning at `srcPos` into `dest` beginning at `destPos`.
- **Type parameter:** The placeholder type name in angle brackets in a generic class declaration (`<T>`, or `<Mustard>` in this lecture's `SLList`), substituted with a real reference type at instantiation.
- **Amortized constant time (extra context, named informally here):** A cost guarantee where any individual operation may be expensive but the average cost over a long sequence of operations is constant.

## Worked Examples

### Example 1: Tracing the naive generic `AList`

```java
public class AList<T> {
    public T[] items;
    public int size;

    public AList() {
        items = (T[]) new Object[11];
        size = 0;
    }

    private void resize(int capacity) {
        T[] resized = (T[]) new Object[capacity];
        for (int i = 0; i < size; i++) {
            resized[i] = items[i];
        }
        items = resized;
    }

    public void addLast(T x) {
        if (size == items.length) {
            resize(size + 1);
        }
        items[size] = x;
        size += 1;
    }

    public T get(int i) {
        return items[i];
    }
}
```

Step by step for `AList<String> L = new AList<>();` followed by 12 `addLast` calls:

1. **Construction.** `new Object[11]` allocates an 11-box array, every box holding `null`. The cast `(T[])` does not change the object at all, it only changes what the compiler believes about the type of the expression. `items` points at this array; `size = 0`.
2. **`addLast("a")`.** `size` (0) `!= items.length` (11), so no resize. Write `items[0] = "a"` (the box now holds a reference to the string object, not the characters themselves). `size` becomes 1.
3. **`addLast("b")` through `addLast("k")`.** Same thing each time, filling `items[1]` through `items[10]`. After the 11th call, `size == 11` and every box is occupied.
4. **`addLast("l")` (the 12th).** Now `size == items.length == 11`, so `resize(12)` runs: allocate a 12-box `Object[]`, loop `i` from 0 to 10 copying each reference over, then point `items` at the new array. The old 11-box array is now unreachable and will be garbage collected. Back in `addLast`, write `items[11] = "l"` and set `size = 12`.
5. **`addLast("m")` (the 13th).** `size == items.length == 12` again immediately, so we copy all 12 items into a 13-box array. And so on, forever. **Every insert past 11 copies the whole list.** This is the quadratic behavior.

Notice what `get` does *not* do: it never checks `i < size`. `L.get(11)` right after step 3 would return `null` (a junk box), not throw. Real implementations add bounds checking.

### Example 2: Counting boxes (textbook Exercises 2.5.5 and 2.5.6)

*Suppose we have a full array of size 100 and we call `addLast` twice under naive `resize(size + 1)` resizing.*

- First `addLast`: create an array of 101 boxes, fill 100 of them by copying, then fill the 101st with the new item. Boxes created: 101.
- Second `addLast`: `size == 101 == items.length`, so create an array of 102 boxes, copy 101, fill the last. Boxes created: 102.
- **Total boxes created and filled across the process: 100 + 101 + 102 = 303** counting the original array, or 203 newly created.
- **Boxes in existence at any one time:** at the peak of the first resize, both the 100-box array and the 101-box array exist simultaneously (201 boxes), because the copy loop needs both. As soon as `items = a` executes and the old reference is lost, the old array becomes garbage, so we drop back to 101.

*Starting from an array of size 100, how many boxes get created and filled over 1,000 `addLast` calls?*

Roughly 101 + 102 + ... + 1100, which is about (1100 + 101) × 1000 / 2 ≈ 600,000 boxes, to store 1,000 items. That is the parabola.

Now redo it with doubling from capacity 100: resizes happen at capacity 100 → 200 → 400 → 800 → 1600, copying 100 + 200 + 400 + 800 = 1,500 items total across 4 resizes. Roughly 400x less copying, and the gap widens as N grows.

### Example 3: The speedtest `AList` with geometric resizing

```java
public class AList {
    private int size;
    private int[] items;

    public AList() {
        size = 0;
        items = new int[100];
    }

    public void addLast(int x) {
        if (size == items.length) {
            resize((int) (size * 1.1));
        }
        items[size] = x;
        size += 1;
    }

    private void resize(int newSize) {
        int[] a = new int[newSize];
        System.arraycopy(items, 0, a, 0, size);
        items = a;
    }

    public int getLast() { return items[size - 1]; }
    public int get(int i) { return items[i]; }

    public int removeLast() {
        int itemToReturn = getLast();
        size -= 1;
        return itemToReturn;
    }
}
```

What happens as we insert:

- Inserts 1 through 100 are pure writes, no resizing at all.
- At insert 101, `size == 100 == items.length`, so `resize(110)`: allocate 110 ints, `System.arraycopy` moves the 100 existing values, `items` is repointed. Then the write proceeds.
- The next resize is not until insert 111, then 122, then 135, and so on. The gap between resizes *grows by 10% each time*, so resizes become steadily rarer relative to the number of inserts.
- Total copying to reach N items is 100 + 110 + 121 + ... , a geometric series whose sum is bounded by about 11N, i.e. linear in N. Compare with the naive version's N²/2.

Note this is an `int[]`, not a generic array, so no cast is needed and `removeLast` can safely leave the stale int behind. Also note this class is *not* generic while the other `AList` in the lecture is; the speedtest version deliberately uses primitive `int`s to keep the timing focused on the resizing cost.

### Example 4: The single-N speed tests

```java
public class SpeedTestAList {
    public static void main(String[] args) {
        long startTime = System.currentTimeMillis();

        AList L = new AList();
        int i = 0;
        while (i < 100000) {
            L.addLast(i);
            i = i + 1;
        }

        long endTime = System.currentTimeMillis();
        System.out.println("Total runtime: " + (endTime - startTime) + " ms");
    }
}
```

and its linked-list counterpart:

```java
public class SpeedTestSLList {
    public static void main(String[] args) {
        long startTime = System.currentTimeMillis();

        SLList<Integer> L = new SLList<>();
        int i = 0;
        while (i < 100000) {
            L.addFirst(i);
            i = i + 1;
        }

        long endTime = System.currentTimeMillis();
        System.out.println("Total runtime: " + (endTime - startTime) + " ms");
    }
}
```

Both do the same number of logical insertions. With the **naive** `AList`, the array version takes several seconds while the `SLList` finishes essentially instantly, which is the observation that motivates the whole lecture. With the **geometric** `AList` (the code as shipped, using `size * 1.1`), the array version also finishes so fast you can barely measure it.

Two things to notice about `SpeedTestSLList` (extra context): `L.addFirst(i)` autoboxes each `int i` into an `Integer` object, since generics only work with reference types. And single-run millisecond timings on the JVM are noisy (JIT warm-up, garbage collection), which is exactly why the table version below is more informative.

### Example 5: The table-based speed test, and how to read it

```java
public class SpeedTestAListTable {
    public static void main(String[] args) {
        System.out.printf("%-15s %-15s\n", "N", "Time (ms)");

        int currentN = 10_000;

        while (true) {
            long timeForCurrentN = measureAndPrintForN(currentN);

            if (timeForCurrentN > 5000) {
                break; // Stop if the test took more than 5000 ms
            }

            currentN *= 10; // Increase N by a factor of 10
        }
    }

    private static long measureAndPrintForN(int N) {
        AList L = new AList();

        long startTime = System.currentTimeMillis();
        for (int i = 0; i < N; i++) {
            L.addLast(i);
        }
        long endTime = System.currentTimeMillis();
        long totalTime = endTime - startTime;

        System.out.printf("%-15d %-15d\n", N, totalTime);
        return totalTime;
    }
}
```

The structure to learn: build a **fresh** list for each N (reusing one list would contaminate the measurement), time only the insertion loop, print N and the time, and multiply N by 10 until a run exceeds 5 seconds.

How to interpret the output. Suppose N = 10,000 takes about 1 ms. Then:

- If the implementation is **linear overall** (geometric resizing, or `SLList.addFirst`), N = 100,000 is about 10 ms, N = 1,000,000 about 100 ms, N = 10,000,000 about 1000 ms. Each row is roughly **10x** the previous. You get many rows before hitting 5 seconds.
- If the implementation is **quadratic overall** (naive `resize(size + 1)`), each row is roughly **100x** the previous: 1 ms, 100 ms, 10,000 ms. The loop terminates after only two or three rows.

So the diagnostic is: *look at the ratio between consecutive rows.* A ratio near 10 means linear, a ratio near 100 means quadratic. `SpeedTestSLListTable` is identical in structure but builds an `SLList<Integer>` and calls `addFirst`, giving you the linear baseline to compare against.

One caveat visible in the source: `SpeedTestSLListTable`'s comment says "more than 500 ms" while the code checks `> 5000`. The code is what runs; the comment is stale.

### Example 6: Box-and-pointer reasoning through a resize

Consider `AList<String> L` with `items` pointing to a 2-box array `["a", "b"]` and `size = 2`, and we call `L.addLast("c")` under `resize(size + 1)`.

- `size == items.length`, so `resize(3)` is called.
- A new 3-box `Object[]` is allocated somewhere else in the heap: `[null, null, null]`.
- The loop copies **references**: `resized[0]` now points to the same `"a"` object that `items[0]` points to, and likewise for `"b"`. The strings themselves are never duplicated. If these were mutable objects, both arrays would be aliasing the same objects during the copy.
- `items = resized` redirects the `AList`'s `items` arrow to the new array. The old 2-box array now has zero incoming arrows: it is garbage.
- Back in `addLast`, `items[2] = "c"` and `size = 3`.

The key takeaway: a resize changes *which array the list points at*, not the array itself, and it copies references, not objects.

## Common Pitfalls

1. **Confusing `size` with `items.length`.** Writing `for (int i = 0; i < items.length; i++)` when iterating the list walks into the junk region. Writing `resize` with `items.length` instead of `size` in the copy loop copies garbage (and crashes when shrinking). The copy loop bound must be `size`.
2. **Believing an array can grow.** `items.length` is immutable. Every "resize" is a fresh allocation plus a copy plus a reassignment. Forgetting the reassignment (`items = resized;`) means you built a new array and threw it away, leaving the list unchanged.
3. **Thinking `resize(size + 1000)` is fast.** Any *additive* growth is asymptotically quadratic. It only changes the constant factor. Only *multiplicative* growth gives constant amortized cost.
4. **Checking `size > items.length` instead of `size == items.length`.** By the time `size` exceeds `length`, you have already written out of bounds. The check must happen *before* the write, and the correct condition is equality (the invariant guarantees `size` never exceeds `length`).
5. **`new T[n]`.** Illegal in Java. You must write `(T[]) new Object[n]`, and you must accept the resulting unchecked-cast warning.
6. **Forgetting to null out on delete in a generic list.** `size -= 1` alone is correct for `int[]` but causes loitering for `T[]`: the removed object stays reachable through the array and is never garbage collected.
7. **Growing but never shrinking.** Without a downsizing rule, a list that peaks at a billion items and then drops to ten million permanently holds the billion-box array. Halve the capacity when the usage ratio falls below 0.25.
8. **Halving as soon as R < 0.5.** (extra context) Too aggressive: alternating add/remove at the threshold triggers a resize on every operation. The gap between the grow and shrink thresholds is what prevents thrashing.
9. **Assuming `SLList.addLast` is constant time.** The `SLList` in this lecture has no tail pointer, so `addLast` walks the list: linear per call, quadratic for N calls. That is why the speed tests use `addFirst`.
10. **Assuming the multiplicative factor must be 2.** 1.1 is also geometric and also amortized constant. The factor trades resize frequency against wasted memory.
11. **No bounds checking in `get`.** The lecture's `get(int i)` returns `items[i]` with no check against `size`, so `get` of an index in the junk region silently returns `null` or a stale value instead of erroring.
12. **Timing a reused list.** In a table-based experiment, each N must get a freshly constructed list, or earlier work skews later rows.

## Likely Exam Points

### 1. Count the copies under a given resizing strategy

**Q:** An `AList` starts with a backing array of length 4 and uses `resize(size + 1)`. Starting from an empty list, how many total item-copies are performed by `resize` over the first 10 `addLast` calls?

**A:** No resize happens for the first 4 adds (capacity 4). The 5th add triggers `resize(5)`, copying 4 items. The 6th triggers `resize(6)`, copying 5. And so on through the 10th, which copies 9. Total = 4 + 5 + 6 + 7 + 8 + 9 = **39 copies** for 10 inserts. Compare with doubling (4 → 8 → 16), which would copy 4 + 8 = 12.

### 2. Identify the asymptotic behavior from a timing table

**Q:** You run a table-based speed test and get: N = 10,000 → 2 ms; N = 100,000 → 210 ms; N = 1,000,000 → 21,300 ms. What resizing strategy is the implementation most likely using, and why?

**A:** Each 10x increase in N multiplies the time by about 100, which is the signature of quadratic total runtime, i.e. linear time per `addLast`. That means **additive resizing** (something like `resize(size + c)`). A geometric strategy would give ratios near 10 per row.

### 3. Fix the naive `addLast`

**Q:** Rewrite this `addLast` so that N inserts take time linear in N overall, and explain the change in one sentence.

```java
public void addLast(T x) {
    if (size == items.length) {
        resize(size + 1);
    }
    items[size] = x;
    size += 1;
}
```

**A:**

```java
public void addLast(T x) {
    if (size == items.length) {
        resize(size * 2);
    }
    items[size] = x;
    size += 1;
}
```

Multiplying the capacity means resizes happen only about log₂(N) times and the copy costs form a geometric series summing to less than 2N, so the total work is linear and the per-insert cost is constant on average. (Caveat: if the initial capacity could be 0, use `Math.max(1, size * 2)`, since `0 * 2 == 0`.)

### 4. size vs. length

**Q:** An `AList<String>` has `size == 3` and `items.length == 8`. What does `items[5]` contain, and what should `get(5)` do?

**A:** `items[5]` is in the junk region past the end of the list. It holds either `null` (never written) or a stale reference from a previously removed item. `get(5)` should throw an exception or otherwise signal an error, because index 5 is not a valid position in a 3-item list. The lecture's `get` does not check, so it would silently return whatever is in the box.

### 5. Generic array creation

**Q:** Why does `T[] items = new T[10];` fail to compile, and what is written instead? What is the consequence?

**A:** Java does not permit instantiating an array of a generic type parameter. You write `T[] items = (T[]) new Object[10];`, which allocates an `Object[]` and casts it. The consequence is an unchecked-cast compiler **warning** (not an error), which we accept for now; the deeper reason relates to how generics are implemented and is covered later.

### 6. Loitering

**Q:** Here is `removeLast` for a generic `AList`. What memory problem does it have, and how do you fix it?

```java
public T removeLast() {
    T item = items[size - 1];
    size -= 1;
    return item;
}
```

**A:** The reference at `items[size - 1]` (now past the end of the list) still points at the removed object, so the garbage collector cannot reclaim that object even though the list has logically discarded it. This is loitering. Fix: set `items[size - 1] = null;` before decrementing `size` (or `items[size] = null;` after decrementing).

### 7. Usage ratio and downsizing

**Q:** An `AList` has `size == 30` and `items.length == 1000`. What is the usage ratio, and what would a typical implementation do?

**A:** R = 30 / 1000 = 0.03. Since R < 0.25, a typical implementation halves the backing array. (To actually reach a reasonable usage ratio it would keep halving on subsequent removals: 1000 → 500 → 250 → 125, and so on.)

### 8. Array list vs. linked list tradeoff

**Q:** In this lecture's code, `AList.addLast` is amortized constant time but `SLList.addLast` is linear. Why, and how could `SLList.addLast` be made constant time?

**A:** `AList` knows exactly where the end is (`items[size]`), so adding at the back is a single write plus an occasional amortized-constant resize. This lecture's `SLList` has only a `sentinel` reference, so `addLast` must traverse with `while (p.next != null) p = p.next;` to find the final node, which takes time proportional to the list length. It can be made constant time by maintaining a `last` pointer to the final node (and, more generally, by using a doubly linked list with a sentinel).

### 9. Why the speed test compares `addLast` to `addFirst`

**Q:** `SpeedTestAList` calls `addLast` but `SpeedTestSLList` calls `addFirst`. Is this a fair comparison?

**A:** Yes, for the question being asked. The goal is to compare the *fast back-end insertion* of each data structure. For `SLList` (no tail pointer) the fast insertion is `addFirst`, which is constant time; using `addLast` would measure the traversal cost, not the insertion cost, and would make the linked list look quadratic for unrelated reasons. Both tests insert the same number of items.

## Summary

- An `AList` is a backing array `items` plus an `int size`; the list is exactly `items[0]` through `items[size - 1]`. `size` is where the next `addLast` writes, `size - 1` is the last item, `items.length` is the physical capacity.
- Java arrays cannot grow. "Resizing" means allocating a new array, copying `size` items over (by hand or with `System.arraycopy`), and reassigning `items`; the old array becomes garbage.
- Naive additive resizing (`resize(size + 1)`) copies the whole array on nearly every insert: about N²/2 copies for N inserts, quadratic total time, unusable at N = 100,000. Any constant additive factor (even `+1000`) is still quadratic.
- Geometric resizing (`resize(size * 2)` or `resize((int)(size * 1.1))`) makes resizes exponentially rarer; total copying is linear in N, so `addLast` costs constant time on average. The factor trades resize frequency against wasted memory.
- Timing method: `System.currentTimeMillis()` around the insertion loop; better, a table of N vs. time with N multiplied by 10 each round. Row-to-row ratio ≈ 10 means linear, ≈ 100 means quadratic. Build a fresh list per row.
- `SLList.addFirst` is constant time (straight-line graph); the naive `AList.addLast` is linear time per op (parabola).
- Space matters too: usage ratio R = size / items.length; halve the array when R < 0.25 so a shrunken list does not hoard memory.
- Generic `AList`: `class AList<T>`, and `items = (T[]) new Object[n]` because `new T[n]` is illegal. Expect an unchecked-cast warning. The type parameter name is arbitrary (this lecture's `SLList<Mustard>`).
- Null out removed slots in a generic list to avoid loitering, since Java frees an object only when the last reference to it is lost.
- The lecture's `SLList` has no tail pointer, so its `addLast` traverses the whole list and is linear; that is why the speed tests use `addFirst`.
