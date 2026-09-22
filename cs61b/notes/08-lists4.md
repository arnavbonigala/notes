<!-- Mon, Sep 14, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 8: Resizing and Circular Arrays

## Overview

This lecture completes the `AList` (array-based list) story that began in Lecture 7, fixing three problems in sequence. First, arrays in Java have a fixed length, so to support unlimited elements we "resize": allocate a brand new, larger array, copy the old contents over, and repoint `items` at the new array. Second, we discover through a computational experiment that resizing by a constant additive amount (`resize(size + 1)`) is catastrophically slow, taking roughly 500,000 memory-box writes just to reach 1,000 items, because the total work grows like a parabola; the fix is geometric resizing (`resize(size * RFACTOR)`), which is what Python's `list` actually does under the hood. We also add downsizing based on a "usage ratio" so we do not waste memory after mass removals. Third, we generify `AList` (with the awkward `(Glorp[]) new Object[8]` cast, plus nulling out deleted items to avoid loitering) and finally speed up front operations by letting the list float inside the array with `nextFirst` / `nextLast` pointers that wrap around, the circular array design required for Project 2 (ArrayDeque).

---

## Key Concepts

### 1. Why arrays need "resizing" at all

The key idea of an `AList` is: store items in the **front part** of the array, leaving empty space at the end for later items. `size` tracks how many real items there are; `items.length` is the physical capacity of the array.

An array's length is fixed at creation. Java gives us a fixed-size tool, and our job at the `AList` level of abstraction is to *cheat*: pretend to the user that the list can grow forever, and hide the fact that behind the scenes we occasionally throw out one array and build a bigger one.

Why not just allocate an array of four billion up front? Because it wastes enormous memory for a list that may only ever hold ten items. Why not tell the user "you're full"? Some data structures do exactly that, but a list is supposed to be unbounded.

"Resizing" is a **misnomer**: the array never changes size. Josh's analogy from lecture: resizing a shirt would mean altering the shirt you're wearing; what we actually do is sew a bigger shirt and put it on.

### 2. The three steps of a resize

When `size == items.length` and someone calls `addLast(11)`:

1. `int[] resized = new int[size + 1];` - a new array, all slots filled with Java's default value (`0` for `int`, `null` for references). They are zeros because we haven't assigned anything yet.
2. Copy every existing item from `items` into `resized`. This genuinely costs time: the machine physically reads each value and writes it into a new location.
3. `items = resized;` - this copies the 64-bit **address** of the new array into the `items` box. `items` now points to the bigger array.

Then `items[size] = 11; size += 1;`.

Note on step 3 (box-and-pointer reasoning): `items` never holds the address of element 0 in any meaningful sense you can manipulate; it holds the address of the **entire array object**. After the reassignment, nothing references the old 100-element array, so Java's garbage collector reclaims it automatically. There is no `free` or `delete` in Java.

A consequence raised in lecture: during the copy, **both** arrays are alive simultaneously, so a resize temporarily needs roughly double the memory. If your list already occupies half your RAM, resizing can run you out of memory.

Also asked in lecture: why can't Java just tack extra boxes onto the end of an existing array? Because the memory immediately after the array may already be occupied by something else. Arrays are fast precisely because they are **contiguous** in memory, so indexing is a simple address computation. (Covered in more depth in CS 61C.)

### 3. Factoring out `resize` as a private helper

Both the inline version and the helper version work, but the helper is "much better":
- Easier to read: "if the array is too full, resize the array."
- Easier to test each function's correctness independently.
- Reusable from other methods (e.g. downsizing, `removeLast`).

It is **private** because the capacity of the backing array is an implementation detail. A user of `AList` should not be thinking about capacity; ideally they shouldn't even know it exists.

### 4. Why naive (additive) resizing is unusably slow

Counting memory boxes created and filled:

- Starting full at 100, one `addLast`: create 101 boxes, fill 101 boxes.
- The next `addLast`: create 102, fill 102.
- Two `addLast` calls: **203** boxes total.
- Going from capacity 100 up to size 1000: `101 + 102 + ... + 1000`.

Using `1 + 2 + ... + N = N(N+1)/2`, this sum is close to **500,000** (exact answer from lecture's Wolfram Alpha check: 495,450). So a mere 1,000 inserts costs about half a million units of work.

The intuition Josh emphasized: picture the computer saying "okay, copy all 100 things... oh, copy all 101 things... oh, copy all 102 things..." forever. Each individual `addLast` does linear work, so the **total** is quadratic, a parabola.

Since the integral of a constant is a line, a straight total-time graph means each operation is constant time (that is `SLList`). Since the integral of a line is a parabola, a parabolic total-time graph means each operation is linear time (that is the naive `AList`).

Empirical numbers from lecture (machine-dependent, and the live demo values differed slightly from the slide values):

| N | SLList (ms) | naive AList (ms) |
|---|---|---|
| 10,000 | 1 | 19 |
| 100,000 | 3 | 903 |
| 1,000,000 | 71 | 70,227 |
| 10,000,000 | 875 | (too slow to finish) |

Sanity check on magnitudes: inserting 100,000 items requires roughly 5,000,000,000 new memory boxes; computers do on the order of a billion things per second (GHz), so seconds of runtime is exactly what you'd expect.

### 5. Geometric resizing

Larger additive constants help, but only shift the parabola:

- `resize(size + 1000)`: N=100,000 drops from 602 ms to 6 ms, but N=10,000,000 still takes 6,097 ms.
- `resize(size * 2)`: N=100,000 takes 2 ms, N=10,000,000 takes 25 ms.

The rough intuition given: **as the array grows larger, we resize exponentially less often**. You still do an expensive thing occasionally, but "occasionally" becomes exponentially rarer, so the per-operation cost is amortized away. A full proof is deferred (Josh said roughly three lectures after the midterm; the textbook defers it to the final chapter).

Key results stated in lecture:
- Additive (`resize(size + RFACTOR)`): **unusably bad** for large N, no matter the constant.
- Multiplicative (`resize(size * RFACTOR)`): **great performance**, even for a small factor like 1.1 (the lecture code uses `resize((int) (size * 1.1))`). This is how the Python list is implemented.
- Josh noted in Q&A that the amortized per-operation cost is *constant*, "not even log N" (details after the midterm).

### 6. Memory efficiency and the usage ratio

Problem #2: insert 1,000,000,000 items, then remove 990,000,000. The operations run fast, but afterwards 99% of the array sits unused.

Define the **usage ratio** `R = size / items.length`. Typical solution: **halve the array when R < 0.25**.

Why 0.25 and not 0.5? If you halve at R = 0.5, the array is immediately full (R = 1) after shrinking, so a single `addLast` forces a grow. An adversarial `add, remove, add, remove, ...` pattern would then resize on every single operation. You want buffer room on both sides so you are never "right on the edge."

An `AList` should be efficient in **time and space**; the course will revisit this tradeoff repeatedly.

### 7. Generic ALists

Parameterize the class and replace `int` with the type parameter everywhere:

```java
public class AList<Glorp> {
    private Glorp[] items;
    private int size;
    ...
}
```

The one syntactic hitch: **Java does not allow generic array creation.** `new Glorp[cap]` produces a "generic array creation" compilation error. Instead:

```java
Glorp[] items = (Glorp[]) new Object[8];
```

This is a **cast** (the only cast used in this course) and produces an "unchecked cast" compiler *warning*, which you should ignore. Java's own `ArrayList` does something different internally; this is the cleaner-but-dirty approach for us. The reason is an obscure consequence of how generics are implemented (see Angelika Langer's 356-page Java Generics FAQ if curious).

### 8. Loitering: null out deleted items

With `int[]`, leaving a stale value behind after `removeLast` costs nothing. With generics, it costs **memory**: Java only destroys an object once the last reference to it is lost. If `items[2]` still points to a multi-megabyte image, that image cannot be garbage collected even though the list logically no longer contains it.

Keeping references to unneeded objects is called **loitering**. Save memory: don't loiter.

```java
public Glorp removeLast() {
    Glorp returnItem = getLast();
    items[size - 1] = null;
    size -= 1;
    return returnItem;
}
```

Decrementing `size` alone yields a *correct* list; nulling makes it a *memory-efficient* one.

### 9. Circular arrays: making front operations fast

`AList` was built to make `get(i)` fast, which fixed the `DLList`'s weakness. But is `ArrayList` strictly better? No:
- `addFirst` and `removeFirst` are faster for linked lists (constant time, versus shifting every element for a naive `AList`).
- Less important: linked list end-operations are *always* fast, whereas `ArrayList` operations occasionally trigger a linear-time resize (negligible in practice).

**The fix:** don't force the list to start at index 0. Leave empty space at the *front* too, and track where the list begins. Now `addFirst` just writes to the slot before the front and decrements the front pointer, no shifting.

**Wraparound:** when the front pointer would go below index 0, wrap it to the end of the array. Java has no negative indices (unlike Python), so you need modular arithmetic. Conceptually the array is a **circle**: imagine gluing the last slot to the first.

**The representation is not unique.** The logical list `[11, 13, 6, 5, 3, 1, 7, 81]` can sit in the array starting at any offset; every rotation is equally valid. Think of it as snipping the conceptual circle at different points and straightening it out. On Project 2 you may put your first item anywhere you want, including after a resize.

**Recommended design for Project 2:** four instance variables, `items`, `nextFirst`, `nextLast`, `size`. Technically you don't need both pointers, but it's a nice idea. `nextFirst` is the index where the next `addFirst` will write; `nextLast` is where the next `addLast` will write.

### 10. Note on Python lists

Python lists do **not** use the circular abstraction, which is why `x.insert(0, "hi")` is slow. Josh's answer for why: front insertion is too rare to justify the extra code complexity and the performance penalty it would impose on all the other operations.

Users who need fast front operations should pick the right data structure: `LinkedList` instead of `ArrayList` in Java, `collections.deque` instead of `list` in Python. (Fun fact from the slides: CPython's `deque` is actually a linked list of blocks, not a circular array.)

### 11. A note on `System.arraycopy`

Two ways to copy arrays: item-by-item in a loop, or `System.arraycopy`, which takes five parameters: source array, start position in source, target array, start position in target, number to copy.

```java
System.arraycopy(items, 0, a, 3, 2);   // in Python: a[3:5] = items[0:2]
```

`arraycopy` is likely faster (especially for large arrays) and more compact, but arguably harder to read. **Recommendation from lecture: do not use it on Project 2**, because with a circular array the items you need to copy are not one contiguous run, so the index arithmetic becomes a trap.

---

## Definitions

- **AList / ArrayList:** a list implementation that stores its items in the front part (or, once circular, some contiguous-modulo-length run) of a backing array, with unused capacity left over for future items.
- **`size`:** the number of items currently in the list. In a non-circular `AList`, also the index where the next `addLast` writes, and `size - 1` is the index of the last item.
- **`items.length`:** the **capacity**, i.e. the physical length of the backing array. Distinct from `size`.
- **Resizing:** creating a new array of a different capacity, copying the existing items into it, and reassigning the instance variable to point at the new array. A misnomer: the original array is never altered.
- **Additive resizing:** growing by a fixed constant, `resize(size + RFACTOR)`. Yields quadratic total cost; unusably bad for large N.
- **Geometric / multiplicative resizing:** growing by a multiplicative factor, `resize(size * RFACTOR)`. Yields linear total cost; used by Python's list.
- **`RFACTOR`:** the resizing factor, the constant added or multiplied when growing the array.
- **Usage ratio (R):** `size / items.length`. A measure of how much of the allocated array is actually in use.
- **Downsizing:** halving the array capacity when the usage ratio falls below a threshold (typically R < 0.25) to avoid wasting memory.
- **Generic array creation error:** the compile error Java emits for `new Glorp[n]`; worked around with `(Glorp[]) new Object[n]`, which produces an unchecked-cast warning.
- **Loitering:** retaining a reference to an object that is no longer needed, preventing garbage collection and wasting memory.
- **Garbage collection:** Java's automatic reclamation of objects once the last reference to them is lost. No manual `free`/`delete` is required.
- **Circular array:** an array treated as if its last index were adjacent to its first, so the logical list can wrap around the end. Requires modular index arithmetic.
- **`nextFirst` / `nextLast`:** indices of the empty slots immediately before the front and immediately after the back of the logical list; the destinations for the next `addFirst` and `addLast` respectively.
- **Contiguous:** occupying consecutive memory locations. Arrays are contiguous, which is what makes indexing fast and why arrays cannot simply be extended in place.

---

## Worked Examples

### Example 1: Naive resizing, inline

This is the version written live in lecture, starting from the non-resizing `addLast`.

```java
public void addLast(int x) {
    if (size == items.length) {                   // (1) is the array full?
        int[] resized = new int[size + 1];        // (2) new, one-bigger array
        for (int i = 0; i < size; i++) {          // (3) copy everything over
            resized[i] = items[i];
        }
        items = resized;                          // (4) repoint items
    }
    items[size] = x;                              // (5) write the new item
    size += 1;                                    // (6) update size
}
```

Step by step, with `items` of length 100 and `size == 100`:

1. `size == items.length` is `100 == 100`, true, so we resize.
2. `new int[101]` allocates 101 boxes, every one initialized to Java's default `0`. In the environment, `resized` is a local variable holding the address of this new array object.
3. The loop runs `i = 0 .. 99`, reading from the old array and writing into the new one. This is 100 real memory reads and 100 real memory writes; it is not free.
4. `items = resized` copies the 64-bit address out of `resized` into the instance variable `items`. Draw two arrows: before, `items` points at the 100-array; after, both `items` and `resized` point at the 101-array, and nothing points at the 100-array. When `addLast` returns, `resized` goes out of scope; the old array, now unreferenced, becomes eligible for garbage collection.
5. `items[100] = 11` fills the one empty slot. This is legal now and would have been an `ArrayIndexOutOfBoundsException` before.
6. `size` becomes 101.

**Common bug caught live in the demo:** Josh initially wrote `resize(size)` instead of `resize(size + 1)`, which created a new array of the *same* capacity. The array was still full, `items[size] = x` ran off the end, and Java threw `ArrayIndexOutOfBoundsException`.

### Example 2: Refactored into a private helper (recommended for the project)

```java
public class AList {
    private int[] items;
    private int size;

    /** Resizes the underlying array to the target capacity. */
    private void resize(int capacity) {
        int[] resized = new int[capacity];
        for (int i = 0; i < size; i++) {
            resized[i] = items[i];
        }
        items = resized;
    }

    /** Inserts x into the back of the list. */
    public void addLast(int x) {
        if (size == items.length) {
            resize(size + 1);
        }
        items[size] = x;
        size += 1;
    }
}
```

Identical behavior, but `addLast` now reads like English. Note the loop bound is `i < size`, not `i < items.length`: we only copy real items. Note also that `resize` is `private`: capacity is nobody's business but `AList`'s.

### Example 3: Counting memory boxes (the two attendance/warmup questions)

**Q (warmup):** full array of size 100, call `addLast` twice. How many total array memory boxes must be created and filled?

Trace it:
- Start: an array of 100 boxes, full.
- First `addLast`: create an array of 101, copy 100 in, write the new item. **101 boxes created and filled.**
- Second `addLast`: the 101-array is now full, so create an array of 102, copy 101 in, write the new item. **102 boxes.**

Total: 101 + 102 = **203**.

**Bonus:** the maximum number of array boxes Java tracks at any one time, assuming garbage collection is immediate, is also **203**: during the second resize, the 101-array and the 102-array coexist (the original 100-array was already collected). When the second `addLast` finishes, only the 102-array remains.

**Q (main):** full array of size 100, call `addLast` until `size == 1000`. Roughly how many boxes?

- 100 → 101 costs 101 boxes
- 101 → 102 costs 102
- ...
- 999 → 1000 costs 1000

Total: `101 + 102 + ... + 1000`. Using `1 + 2 + ... + N = N(N+1)/2`, this is `1000 * 1001 / 2` minus the first hundred terms, which is close to **500,000** (answer choice B; exactly 495,450). The first hundred terms are negligible because they are the *smallest* ones.

*Why the formula holds* (the hidden slide's argument): pair up the terms from the outside in. `1 + N`, `2 + (N-1)`, `3 + (N-2)`, ... Each pair sums to `N + 1`, and there are `N/2` such pairs, so the total is `N(N+1)/2`.

### Example 4: Geometric resizing, the one-line fix

```java
public void addLast(int x) {
    if (size == items.length) {
        resize(size * RFACTOR);      // instead of resize(size + RFACTOR)
    }
    items[size] = x;
    size += 1;
}
```

The lecture code actually used a fractional factor, which requires a cast because array lengths must be `int`:

```java
private int size;
private int[] items;

public void addLast(int x) {
    if (size == items.length) {
        resize((int) (size * 1.1));
        // other resizing strategies:
        //   resize(size + 1000);
        //   resize(size * 2);
    }
    items[size] = x;
    size += 1;
}

private void resize(int newSize) {
    int[] a = new int[newSize];
    System.arraycopy(items, 0, a, 0, size);
    items = a;
}
```

With a factor of 1.1 and capacity 100: resize to 110, then 121, then 133, and so on. Even this small factor is fast, because the *gaps between resizes* grow geometrically. With a factor of 2, the lecture demo ran out of memory before the code ever became slow.

### Example 5: Generic AList, before and after

```java
// BEFORE                                  // AFTER
public class AList {                       public class AList<Glorp> {
    private int[] items;                       private Glorp[] items;
    private int size;                          private int size;

    public AList() {                           public AList() {
        items = new int[8];                        items = (Glorp[]) new Object[8];
        size = 0;                                  size = 0;
    }                                          }

    public void resize(int capacity) {         public void resize(int cap) {
        int[] resized = new int[capacity];         Glorp[] resized = (Glorp[]) new Object[cap];
        for (int i = 0; i < size; i++) {           for (int i = 0; i < size; i++) {
            resized[i] = items[i];                     resized[i] = items[i];
        }                                          }
        items = resized;                           items = resized;
    }                                          }

    public int get(int i) {                    public Glorp get(int i) {
        return items[i];                           return items[i];
    }                                          }
}                                          }
```

Every `int` that referred to a *stored item* becomes `Glorp`. The `int size` and `int i` stay `int`, because they are counters, not items. The only nonmechanical change is the array creation line.

The lecture code file uses `T` instead of `Glorp`; the name of the type parameter does not matter, though `T` is conventional.

### Example 6: Linear trace of a circular array

Start with `items` of length 8, `nextFirst = 4`, `nextLast = 5`, `size = 0`. (Those starting values are arbitrary; any pair of adjacent slots works.)

| Operation | Conceptual list | Array contents (indices 0..7) | size | nextFirst | nextLast |
|---|---|---|---|---|---|
| *(start)* | `[]` | `_ _ _ _ _ _ _ _` | 0 | 4 | 5 |
| `addLast("a")` | `[a]` | `_ _ _ _ _ a _ _` | 1 | 4 | 6 |
| `addLast("b")` | `[a, b]` | `_ _ _ _ _ a b _` | 2 | 4 | 7 |
| `addFirst("c")` | `[c, a, b]` | `_ _ _ _ c a b _` | 3 | 3 | 7 |
| `addLast("d")` | `[c, a, b, d]` | `_ _ _ _ c a b d` | 4 | 3 | 0 |
| `addLast("e")` | `[c, a, b, d, e]` | `e _ _ _ c a b d` | 5 | 3 | 1 |
| `addFirst("f")` | `[f, c, a, b, d, e]` | `e _ _ f c a b d` | 6 | 2 | 1 |
| `addLast("g")` | `[f, c, a, b, d, e, g]` | `e g _ f c a b d` | 7 | 2 | 2 |
| `addLast("h")` | `[f, c, a, b, d, e, g, h]` | `e g h f c a b d` | 8 | 2 | 3 |

Things to notice:

- `addLast` writes at `nextLast` and then advances `nextLast`; `addFirst` writes at `nextFirst` and then retreats `nextFirst`. **`addFirst` does not touch `nextLast`, and `addLast` does not touch `nextFirst`.** Exactly three memory boxes change per add: the array slot, one pointer, and `size`.
- Going from `nextLast = 7` to `nextLast = 0` is the wraparound: index 7 is "adjacent" to index 0.
- After `addLast("e")`, the logical list is physically split: `f c a b d` occupies 3..7 and `e g h` occupies 0..2, but read circularly starting at index 3 it is one continuous run.
- After `addLast("h")` the array is full (`size == 8 == items.length`) and `nextFirst`/`nextLast` now point at occupied cells. The next add of either kind requires a resize.

### Example 7: Non-uniqueness of the representation

The list `[11, 13, 6, 5, 3, 1, 7, 81]` in an array of length 10 can be stored as any of:

```
13  6  5  3  1  7  81  _   _  11     (front = 9, wraps around)
11 13  6  5  3  1   7  81  _   _     (front = 0, no wrap)
 1  7 81  _  _ 11  13   6  5   3     (front = 5, wraps around)
```

All three are equally valid. Picture snipping the conceptual circle at different points and straightening it out; each snip gives a different linear picture of the same circle.

This matters for **resize**: after you build the new bigger array, you can lay the items out however you like. The slides show three valid post-resize layouts for `[f, c, a, b, d, e, g, h, Z]`. The one the lecture recommended considering is putting the items at index 0 in logical order, i.e. "unrotate" the list during the copy. That makes the new `nextFirst` and `nextLast` trivial to compute.

**This is exactly why `System.arraycopy` is a trap on Project 2:** in the wrapped case the items are two separate runs, so a single `arraycopy` call copies the wrong thing. Writing a loop with `get(i)` is easier to get right, and the lecture explicitly hinted that **`get` is a useful tool inside `resize`**.

### Example 8: The speed test harness

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

The pattern: stamp the clock, do N operations, stamp the clock again, print the difference. `SpeedTestSLList` is identical but uses `L.addFirst(i)` on an `SLList<Integer>`. The `...Table` variants wrap this in a loop that multiplies N by 10 each round and stops once a single round exceeds 5000 ms, which is how the timing tables above were produced.

Note that this is a **computational experiment**, not a proof. It demonstrates conclusively that the naive `AList` is bad, and it motivates the asymptotic analysis machinery that arrives after the midterm.

---

## Common Pitfalls

1. **Confusing `size` and `items.length`.** `size` is how many items the list holds; `items.length` is how many it *could* hold. The resize condition is `size == items.length`, and `get(i)` should be valid only for `i < size`.

2. **`resize(size)` instead of `resize(size + 1)`.** Caught live in the lecture demo. Allocating the same capacity leaves the array full, and the very next `items[size] = x` throws `ArrayIndexOutOfBoundsException`. Whatever your growth strategy, make sure the new capacity is strictly greater than `size`.

3. **Copying `items.length` elements instead of `size` elements.** The loop bound must be `i < size`. Using `items.length` reads past the meaningful data (and can go out of bounds on the new array if you shrink).

4. **Forgetting `items = resized;`.** You built and filled a beautiful new array, then dropped it on the floor. The instance variable still points at the old, full array.

5. **Thinking the array literally grows.** It does not. Resizing means a *new* array object plus a pointer reassignment. Keep the box-and-pointer picture straight: `items` holds an address.

6. **Assuming a bigger additive constant fixes the problem.** `resize(size + 1000)` looks great for N = 100,000 and is still terrible for N = 10,000,000. Additive is asymptotically the same shape (a shifted parabola) no matter the constant. Only *multiplying* changes the shape.

7. **Using `new Glorp[cap]`.** Compile error ("generic array creation"). Use `(Glorp[]) new Object[cap]` and ignore the unchecked-cast warning.

8. **Panicking about the unchecked-cast warning.** It is a warning, not an error, and it is expected here.

9. **Forgetting to null out removed items in a generic list.** The list is still *correct* if you only decrement `size`, but you are loitering, and the held objects (which could be megabytes each) can never be garbage collected.

10. **Halving at R < 0.5.** This leaves the array instantly full, so alternating add/remove thrashes resize on every operation. Halve at R < 0.25.

11. **Assuming negative array indices work.** `items[-1]` is not Python; Java throws. Circular arrays need explicit wraparound logic or modular arithmetic.

12. **Assuming the circular representation is unique.** Any rotation is valid; do not write tests or reasoning that depend on the first item being at index 0.

13. **Letting `addFirst` modify `nextLast` (or vice versa).** Each add moves exactly one of the two pointers.

14. **Using `System.arraycopy` in a circular `resize`.** A wrapped list is two runs, not one. The lecture explicitly recommends against `arraycopy` on Project 2.

15. **Writing the entire Project 2 before testing any of it.** The lecture advice: build incrementally, comment out what is unnecessary if overwhelmed, get it working *without* resizing first (resizing is a performance optimization), and do not be afraid to throw the code away and start over, since there isn't that much of it.

---

## Likely Exam Points

### 1. Counting memory boxes for additive resizing

**Q:** An `AList` has a full backing array of capacity 50 and uses `resize(size + 1)`. You call `addLast` until `size == 60`. How many total array boxes are created and filled?

**A:** Each `addLast` from capacity `k` creates an array of `k + 1` and fills all `k + 1` boxes. So the calls cost 51, 52, ..., 60. That is `(51 + 60) * 10 / 2 = 555` boxes.

### 2. Max boxes alive at once

**Q:** Starting from a full array of capacity 100 with `resize(size + 1)`, you call `addLast` three times. Assuming garbage collection is immediate, what is the maximum number of array boxes Java tracks at any one moment?

**A:** During the third resize the 102-array and the new 103-array coexist, giving 205. (Check the earlier steps: during resize 1 it's 100 + 101 = 201; during resize 2 it's 101 + 102 = 203; during resize 3 it's 102 + 103 = 205.) So **205**. Generalize: during a resize both the old and new arrays are alive, so peak memory is roughly double the list size.

### 3. Additive vs. geometric asymptotics

**Q:** Someone proposes `resize(size + 1000000)` and argues it is just as good as doubling because the constant is huge. Refute this.

**A:** For N below a million it looks fine, since only a handful of resizes occur. But the total work to reach N items is `1M + 2M + 3M + ...`, still a quadratic in N, just with a large constant divisor. For sufficiently large N it degrades into the same parabola. Doubling changes the *shape*: the resizes occur exponentially less often, giving linear total work (constant amortized per operation). It also wastes up to a million slots for a tiny list.

### 4. Usage ratio and downsizing thresholds

**Q:** An `AList` has `size = 30` and `items.length = 200`. What is R? Under the standard policy, what happens, and why is the halving threshold 0.25 rather than 0.5?

**A:** `R = 30/200 = 0.15`, which is below 0.25, so the array is halved (to length 100; R becomes 0.30, still below 0.25? no: 0.30 > 0.25, so it stops). Threshold 0.25 instead of 0.5: halving at exactly R = 0.5 produces R = 1, a full array, so the very next `addLast` forces a grow. An alternating add/remove sequence at the boundary would then resize on *every* operation. Leaving slack on both sides prevents this thrashing.

### 5. Generic array creation syntax

**Q:** What is wrong with `private Glorp[] items = new Glorp[8];` and what is the fix? Is the fix's compiler complaint an error or a warning?

**A:** Java forbids generic array creation, so this is a **compilation error**. Fix: `private Glorp[] items = (Glorp[]) new Object[8];`. That produces an *unchecked cast* **warning**, which is expected and should be ignored.

### 6. Loitering

**Q:** A generic `AList`'s `removeLast` is implemented as `public Glorp removeLast() { size -= 1; return items[size]; }`. Is this correct? Is there a problem?

**A:** It is functionally correct: the item is returned and no longer visible to the user. But it loiters: `items[size]` still references the removed object, so Java cannot garbage collect it even after the caller drops it. Fix by nulling the slot before returning:
```java
public Glorp removeLast() {
    Glorp returnItem = items[size - 1];
    items[size - 1] = null;
    size -= 1;
    return returnItem;
}
```
This matters with generics but not with `int[]`, since primitives are not references.

### 7. Circular array tracing

**Q:** An array of length 6 holds `nextFirst = 4`, `nextLast = 1`, `size = 4`, with contents (indices 0..5) `y z _ _ _ x`. What is the conceptual list? Now execute `addFirst("w")`, then `addLast("v")`. Give the array, `nextFirst`, `nextLast`, and `size` after each.

**A:** The first item is at `nextFirst + 1 = 5`, so reading circularly from index 5: `x` (5), `y` (0), `z` (1)... wait, `nextLast = 1` means index 1 is the *next* empty slot, so index 0 is the last item. Reading from 5: `x`, then 0: `y`. That's only 2 items but `size = 4`, so the given state is inconsistent. Take instead `nextLast = 2`, `size = 3`, contents `y z _ _ _ x`: conceptual list is `[x, y, z]`.

- `addFirst("w")`: write at index 4, decrement `nextFirst` to 3. Array: `y z _ _ w x`, `nextFirst = 3`, `nextLast = 2`, `size = 4`. List: `[w, x, y, z]`.
- `addLast("v")`: write at index 2, advance `nextLast` to 3. Array: `y z v _ w x`, `nextFirst = 3`, `nextLast = 3`, `size = 5`. List: `[w, x, y, z, v]`.

Note the array now has one free slot and both pointers land on it, so the next add of either kind requires a resize.

*(Exam-technique note: always sanity-check that the number of occupied slots between the two pointers matches `size`. Inconsistent states are a classic trap.)*

### 8. Which operations are fast in which structure

**Q:** Fill in the blanks. Compared to a `DLList`, a non-circular `AList` is much faster at ____ and much slower at ____. Does the circular version change this?

**A:** Faster at `get(i)` (random access to the middle), slower at `addFirst`/`removeFirst` (which require shifting every element). The circular version fixes the front operations, making both ends fast. The remaining difference is that a linked list's end operations are *always* fast, while an array list occasionally pays a linear resize (negligible in practice). Random access remains an array list advantage.

### 9. Why is naive AList's total-time graph a parabola?

**Q:** The total-time-vs-N graph for `SLList.addFirst` is a straight line, and for the naive `AList.addLast` it is a parabola. What does each shape tell you about the *per-operation* cost?

**A:** The total time is the running sum (integral) of the per-operation costs. The integral of a constant is a line, so a linear total-time graph means each operation takes constant time. The integral of a line is a parabola, so a parabolic total-time graph means each operation takes time linear in the current size, which is exactly what copying the whole array on every add produces.

### 10. Why can't Java extend an array in place?

**Q:** Why does Java not provide a primitive to append extra slots to the end of an existing array?

**A:** Arrays are stored contiguously in memory, and the memory immediately after an array may already be in use by another object. Contiguity is precisely what makes array indexing fast (the address of element `i` is a simple arithmetic computation), so it cannot be given up. Building a new array elsewhere and copying is the only general option.

---

## Summary

- An `AList` stores items in part of a fixed-length backing array; `size` (items in the list) and `items.length` (capacity) are different things.
- **Resizing** = allocate a new bigger array, copy the `size` real items over, reassign `items` to point at it. The old array is garbage collected automatically. The name is a misnomer; nothing is actually resized.
- Factor resizing into a **private** `resize(int capacity)` helper: clearer, testable, reusable, and capacity is an implementation detail users should not see.
- **Additive resizing is unusably bad.** Going from capacity 100 to size 1000 costs `101 + 102 + ... + 1000 ≈ 500,000` box writes. Total work is quadratic (a parabola); each operation is linear time. Bigger constants only shift the parabola.
- **Geometric resizing** (`resize(size * RFACTOR)`) is the fix, because resizes become exponentially rarer as the array grows. Even a factor of 1.1 works. This is how Python's `list` is implemented. The formal justification comes after the midterm.
- Lecture timings: 100,000 `addLast` calls took 602 ms with `+1`, 6 ms with `+1000`, and 2 ms with `* 2`; 10,000,000 calls took 6,097 ms with `+1000` but 25 ms with `* 2`.
- Define **usage ratio** `R = size / items.length`; halve the array when `R < 0.25` to avoid wasting memory. Do not use 0.5, or add/remove at the boundary thrashes.
- **Generics:** parameterize the class, replace the item type everywhere, and use `(Glorp[]) new Object[n]` since `new Glorp[n]` is a compile error. The unchecked-cast warning is expected.
- **Null out removed items** in a generic list to avoid **loitering**; Java cannot collect an object while any reference survives.
- A plain `AList` is slow at `addFirst`/`removeFirst` because everything must shift. Fix: let the list float inside the array and **wrap around** the ends, i.e. treat the array as **circular**, tracking `nextFirst` and `nextLast`.
- The circular representation is **not unique**: every rotation of the items represents the same list, and after a resize you may lay the items out however you like.
- Each add changes exactly three boxes: one array slot, one pointer, and `size`. Java has no negative indices, so wraparound needs explicit modular logic.
- Python lists are not circular (front insertion is slow) because front insertion is too rare to justify the complexity and the cost to other operations; use `collections.deque` (or Java's `LinkedList`) when you need fast front operations.
- `System.arraycopy(src, srcPos, dst, dstPos, n)` exists and is fast, but **avoid it on Project 2**: a wrapped circular list is two runs, not one. Use `get` inside `resize` instead.
- Project 2 advice: build incrementally, get it working without resizing first, and rewrite from scratch if it gets messy.
