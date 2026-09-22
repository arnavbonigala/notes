<!-- Fri, Sep 11, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 7: DLLists, Basic ALists

## Overview

This lecture closes out the linked list arc and opens the array-based list arc. It starts by reviewing the "allegory of the cave" framing from the `SLList` lecture: the user has a mental model (a list of items) while we, the implementers, secretly control the actual memory boxes. It then walks through two more improvements to fix the remaining slowness of `SLList`: Improvement #7 (adding a `.last` pointer plus `.prev` back-pointers in every node, giving a **doubly linked list** or `DLList`, which makes `removeLast` fast) and Improvement #8 (a **sentinel upgrade**, either two sentinels or a single circular sentinel, to eliminate ugly special cases). Improvement #9 adds **generics**, so a list can hold any reference type rather than only `int`. The lecture then identifies the one problem linked lists cannot solve well: `get(int i)` requires walking the list, so it is slow for indices far from either end. That motivates a pivot to **arrays**, which support constant-time random access, and the lecture builds a naive `AList` (array-based list) supporting `addLast`, `getLast`, `get`, `size`, and `removeLast`, reasoning about it through explicit **invariants**. A bonus section contrasts arrays and classes.

---

## Key Concepts

### 1. The allegory of the cave: desire vs. implementation

The user says "I want 5, 10, 15." That is the *desire*, an abstract idea of a list. What actually exists is a pile of memory boxes: a 32-bit `size` box, a 64-bit `sentinel` box holding an address, and a chain of node objects.

The crucial point: **it is our reality**. We choose the representation. The sentinel-based `SLList` and the no-sentinel `SLList` (with a plain `first` pointer that is sometimes `null`) both satisfy the same user-facing desire. The user never sees the difference. This clean separation between desire and implementation is exactly what "naked recursion" (the `IntList`) cannot give you: to use an `IntList` you must personally understand references and recursion, and you cannot even write an `addFirst` method on it.

The sentinel's `item` field holds garbage. In lecture it was `63` (or `-420` in the live-coded version), chosen deliberately to emphasize that the value is meaningless. It can't be `null` because `int` in Java is a primitive and primitives cannot be `null`.

### 2. Why `.last` alone is not enough

`addLast` on a plain `SLList` is slow because it walks the whole list:

```java
public void addLast(int x) {
    size += 1;
    IntNode p = sentinel;
    while (p.next != null) {
        p = p.next;
    }
    p.next = new IntNode(x, null);
}
```

The natural fix is to cache a `last` pointer, exactly like caching `size`. That makes `addLast` and `getLast` fast. But `removeLast` stays slow, and the reason is subtle.

Deletion is not "make the node vanish." Deletion means **changing pointers**. To remove the last node from `[3, 9, 50]`:
- Set the `9` node's `next` to `null` (disconnect `50`).
- Set `last` to point at the `9` node.

Both require knowing where the **second-to-last** node is, and with only forward links you must walk the whole list to find it. Adding a `secondToLast` pointer does not help: after the removal you would then need the third-to-last node to restore `secondToLast`, and so on forever.

A notational point the lecture stressed: an arrow drawn from `last` points at the **entire node**, not at a specific field inside it. `last` holds the address of a node object.

### 3. Improvement #7: back-pointers (the doubly linked list)

If the problem is "I have the last node and need the one before it," the clean fix is to make every node know who comes before it:

```java
public class IntNode {
    public IntNode prev;
    public int item;
    public IntNode next;
}
```

Now `removeLast` follows `last.prev` in constant time. A list with these backwards links is a **doubly linked list** (`DLList`), as opposed to the **singly linked list** (`SLList`) from last lecture. This is the promised reveal of why the class was awkwardly named `SLList`: the `S` stands for "singly."

Note that Java does not enforce the consistency of `prev` and `next`. Nothing stops you from pointing them anywhere. Keeping them mutually consistent (if `a.next == b` then `b.prev == a`) is a *convention* your code must maintain, which is exactly the sort of thing an invariant captures.

Removal from the middle is now also possible but requires care: removing a middle node means changing two pointers (the predecessor's `next` and the successor's `prev`).

### 4. Improvement #8: the sentinel upgrade

The naive `DLList` (one front sentinel plus a `last` pointer) has an annoying special case: **`last` sometimes points at the sentinel (empty list) and sometimes at a real node.** That asymmetry forces `if (last == sentinel) { ... } else { ... }` branches into your methods. Two ways to kill it:

- **Double sentinel:** a `sentFront` and a `sentBack`. Real items live between them. The empty list is just the two sentinels pointing at each other.
- **Circular sentinel (required for Project 1B):** one sentinel that serves as both front and back bookend. An empty list is a sentinel whose `next` and `prev` both point at *itself*. A list of items forms a ring.

With the circular sentinel:
- First item: `sentinel.next.item`
- Last item: `sentinel.prev.item`
- Second item: `sentinel.next.next.item`

The lecture's honest assessment: the circular version has a *higher cognitive load* for a novice but produces *much cleaner code* with no special cases. The instructor suggested that after finishing the project you can compare your circular version against an LLM-generated `sentFront`/`sentBack` version and see the difference in messiness.

### 5. Improvement #9: generics

Our lists only store `int`. `new SLList("hi")` fails to compile: *incompatible types: String cannot be converted to int*.

Java generics (added in 2004) let you defer type selection until declaration. You put a **placeholder type parameter** in angle brackets after the class name, then use that placeholder wherever the item type appears:

```java
public class SLList<Cow> {
    private class Node {
        public Cow item;
        public Node next;
        public Node(Cow x, Node r) { item = x; next = r; }
    }
    ...
}
```

The name is arbitrary: the lecture used `Cow` live, `LochNess` and `BleepBlorp` on slides, and the textbook mentions `GloopGlop`, `Horse`, `TelbudorphMulticulus`. Inside the class, `Cow` has no meaning; it is a placeholder. When someone writes `SLList<String>`, you can think of `Cow` as being find-replaced by `String`.

Rules of thumb for 61B:
- In the file **implementing** the data structure: declare the generic type **once**, at the very top, after the class name.
- **Do not** re-declare the type parameter on the nested class (i.e. do **not** write `private class Node<Cow>`). That creates a *new, different* type parameter that happens to share a name, and will produce confusing errors. It is legal Java, just a trap.
- Make the nested class **non-static** (a static nested class cannot see the outer class's type parameter).
- In files that **use** the data structure: write the desired type during declaration, use the empty diamond `<>` during instantiation.
- Generics only work with reference types. Use the wrapper types: `Integer`, `Double`, `Character`, `Boolean`, `Long`, `Short`, `Byte`, `Float`.

```java
SLList<Integer> s1 = new SLList<>(5);
s1.addFirst(10);

SLList<String> s2 = new SLList<>("hi");
s2.addFirst("apple");

DLList<Double> d1 = new DLList<>(5.3);
double x = 9.3 + 15.2;
d1.addFirst(x);
```

One consequence inside the generic class: the sentinel's item can no longer be `63` or `-420`. It must be a value valid for the placeholder type, and every reference type admits `null`, so the sentinel item becomes `null`.

### 6. Linked lists are bad at `get`

A `DLList` handles `addFirst`, `addLast`, `getFirst`, `getLast`, `removeFirst`, `removeLast` all quickly. But `get(int i)` is unavoidably slow: you have only references to the front and back, so you must scan forward or backward to position `i`. Item #417 in a 10,000-element list costs 417 forward hops. Worst case is the middle: work proportional to list length (roughly N/2). Compare `getLast`, which is constant regardless of length.

The lecture noted two possible directions: exotic restructurings (extra links, tree shapes, "de-linearizing" the list, covered much later in the course), or abandoning links entirely and using arrays. Today we take the array route.

### 7. Arrays

To build any data structure you need memory boxes. Ways to get them in Java:
- `int x;` gives one 32-bit box for ints.
- `Walrus w1;` gives one 64-bit box for a Walrus reference.
- `Walrus w2 = new Walrus(30, 5.6);` gives a 64-bit reference box plus 96 bits inside the object (32-bit `int size`, 64-bit `double tuskSize`).

**Arrays** are a special kind of object consisting of a *numbered* sequence of memory boxes (unlike classes, which have *named* boxes). An array has:
- A **fixed** integer length `N` that can never change.
- `N` memory boxes, all the same type (and therefore the same number of bits), numbered `0` through `N - 1`.

Like class instances, you get one reference when the array is created, and if you reassign every variable holding that reference, the array is gone forever. **Unlike classes, arrays have no methods.**

Declaration vs. instantiation vs. assignment:

```java
int[] a;                                  // declaration: a 64-bit box for an int-array reference
new int[]{0, 1, 2, 95, 4};                // instantiation: creates an anonymous array object
int[] a = new int[]{0, 1, 2, 95, 4};      // all three at once
```

Three creation notations, all equally valid:
- `x = new int[3];` (length 3, filled with default value `0`)
- `y = new int[]{1, 2, 3, 4, 5};` (length inferred from the listed values)
- `int[] z = {9, 10, 11, 12, 13};` (same as above, but only legal combined with a declaration)

**Random access is fast.** `A[5000000]` is about as fast as `A[0]`. This is a consequence of every box having the same size in bits, so the machine can compute the address directly instead of following a chain. (The lecture flagged that CS 61C explains why, and also explains the caveats: recently-touched boxes, and boxes near recently-touched boxes, are actually somewhat faster. To first order, treat all accesses as equally fast.) This is precisely the opposite of a linked list, where there is no way to jump to the middle.

### 8. Invariants

An **invariant** is a condition guaranteed to be true during code execution, assuming no bugs. The lecture's `AList` invariants:

1. The position of the next item to be inserted is always `size`.
2. `size` is always the number of items in the `AList`.
3. The last item in the list is always in position `size - 1`.

Invariants make it easier to reason about code: you may **assume** they hold when writing a method (which simplifies the method), and you must **ensure** each method leaves them true when it returns. The instructor described them as "safety equipment" while building a class, and noted the next discussion worksheet asks you to write an invariant for a circular array list.

---

## Definitions

- **Singly linked list (`SLList`)**: a list whose nodes each hold an item and a single forward reference (`next`) to the following node. The `S` in `SLList` stands for "singly."
- **Doubly linked list (`DLList`)**: a list whose nodes each hold an item, a forward reference (`next`), and a backward reference (`prev`). Back-pointers make `removeLast` fast.
- **Sentinel node**: a permanent, never-removed node whose item field is meaningless garbage, existing so that every real node has a non-null predecessor (and successor), eliminating special cases for the empty list.
- **Double sentinel list**: a `DLList` with a `sentFront` at the front and a `sentBack` at the back, real items living strictly between them.
- **Circular sentinel list**: a `DLList` with exactly one sentinel that acts as both front and back bookend. In the empty list, the sentinel's `next` and `prev` both point at itself. Required topology for Project 1B.
- **Generic type / type parameter**: an arbitrary placeholder name declared in angle brackets after a class name (`public class SLList<Cow>`), standing in for a type chosen later by whoever declares a variable of that class.
- **Diamond operator (`<>`)**: the empty angle brackets used at instantiation (`new SLList<>("hi")`) when the type is already given on the left-hand side of the declaration.
- **Reference type (wrapper type)**: the object versions of primitives (`Integer`, `Double`, `Character`, `Boolean`, `Long`, `Short`, `Byte`, `Float`), required inside angle brackets because generics cannot take primitives.
- **Array**: a special object consisting of a fixed number `N` of numbered memory boxes, all of the same type, indexed `0` through `N - 1`, with no methods.
- **Declaration (of an array)**: creating a 64-bit box intended to hold a reference to an array; no array object is created.
- **Instantiation (of an array)**: creating the actual array object, e.g. `new int[]{0, 1, 2, 95, 4}`. The resulting object is anonymous until assigned.
- **Random access**: retrieving an item at an arbitrary index in time independent of the size of the collection.
- **`AList`**: an array-based list; a list whose items are stored in an array rather than in linked nodes. (This is the approach Python's built-in list uses.)
- **Invariant**: a condition guaranteed to be true during code execution, assuming no bugs. Methods may assume invariants on entry and must preserve them on exit.

---

## Worked Examples

### Example 1: Why `removeLast` is slow with only a `last` pointer

Setup: `SLList` with `size = 3`, a sentinel, nodes holding `3`, `9`, `50`, and a `last` pointer aimed at the `50` node.

Box-and-pointer reasoning in words:
- `sentinel.next` is the `3` node; `3.next` is the `9` node; `9.next` is the `50` node; `50.next` is `null`.
- `last` holds the address of the whole `50` node.

To `removeLast()` we must end in a state where `size == 2`, the list is `[3, 9]`, and `last` points at the `9` node. That means exactly two memory boxes change:
1. `9.next` must become `null`.
2. `last` must become the address of the `9` node.

Both require a reference to the `9` node. From `last` we can reach `50` but not backwards. So we must start at `sentinel` and walk: `p = sentinel`, `p = p.next` (`3`), `p = p.next` (`9`), and check whether `p.next.next == null`. For a list of length N that is roughly N steps. Slow.

Adding a `secondToLast` pointer would make *this* removal fast, but afterwards `secondToLast` would need to become the `3` node, requiring the third-to-last node, and you have merely pushed the problem one step down.

### Example 2: Navigating a circular-sentinel `DLList`

Setup: circular `DLList` with `size = 2` holding `3` and `9`.

Pointer structure in words:
- `sentinel.next` is the `3` node. `sentinel.prev` is the `9` node.
- `3.prev` is the sentinel. `3.next` is the `9` node.
- `9.prev` is the `3` node. `9.next` is the sentinel.

So the nodes form a ring: sentinel → 3 → 9 → sentinel, and the same ring traversed backwards.

Accessing items:

```java
// first item (the 3)
sentinel.next.item

// last item (the 9)
sentinel.prev.item

// second item (the 9), reached forwards
sentinel.next.next.item
```

Note that `sentinel.item` is *not* the first item: it is the sentinel's meaningless garbage value.

For the **empty** circular list (`size = 0`), `sentinel.next == sentinel` and `sentinel.prev == sentinel`. This is why there are no special cases: `sentinel.next` and `sentinel.prev` are always valid non-null node references, empty list or not.

### Example 3: Making `SLList` generic (the live-coded demo)

Before:

```java
public class SLList {
    private class IntNode {
        public int item;
        public IntNode next;
        public IntNode(int i, IntNode n) { item = i; next = n; }
    }
    private IntNode sentinel;
    private int size;

    public void addFirst(int x) {
        sentinel.next = new IntNode(x, sentinel.next);
        size += 1;
    }
    public int getFirst() { return sentinel.next.item; }
}
```

After (the lecture's version, with `Cow` as the placeholder and `IntNode` renamed to `Node` since it no longer holds ints):

```java
public class SLList<Cow> {
    private class Node {
        public Cow item;   // the first item in the list
        public Node next;  // the rest of the list

        public Node(Cow x, Node r) {
            item = x;
            next = r;
        }
    }

    private Node sentinel;
    private int size;

    /** Create an SLList with x in it. */
    public SLList(Cow x) {
        size = 1;
        sentinel = new Node(null, null);
        sentinel.next = new Node(x, null);
    }

    /** Creates an empty list. */
    public SLList() {
        size = 0;
        sentinel = new Node(null, null);
    }

    public void addFirst(Cow x) {
        sentinel.next = new Node(x, sentinel.next);
        size += 1;
    }

    public Cow getFirst() {
        return sentinel.next.item;
    }

    public int size() {
        return size;
    }

    public void addLast(Cow x) {
        size += 1;
        Node p = sentinel;
        while (p.next != null) {
            p = p.next;
        }
        p.next = new Node(x, null);
    }
}
```

Step by step, what changed and why:
1. `<Cow>` after the class name creates a landing spot for the type the caller supplies. Without it, `new SLList<String>()` fails with "does not have type parameters."
2. `IntNode` renamed to `Node` (cosmetic, but "IntNode" is now a lie).
3. `public int item` became `public Cow item`: the stored value is whatever type the caller chose.
4. The constructor parameter `int i` became `Cow x`.
5. Every method that takes or returns an item switched: `addFirst(Cow x)`, `addLast(Cow x)`, `Cow getFirst()`.
6. The sentinel's placeholder item changed from `63` / `-420` to `null`, because `Cow` is a reference type and only `null` is a legal placeholder for an unknown reference type.
7. `size` stays `int`. It counts items; it is not an item.

Using it:

```java
SLList<String> L = new SLList<>();
L.addLast("cat");
L.addLast("machine");
```

Tracing this: `new SLList<>()` sets `size = 0` and `sentinel = new Node(null, null)`. `addLast("cat")` bumps `size` to 1, walks `p` from `sentinel` (whose `next` is already `null`, so the loop body never runs), and sets `sentinel.next = new Node("cat", null)`. `addLast("machine")` bumps `size` to 2, walks `p` one step to the `"cat"` node, then sets `cat.next = new Node("machine", null)`.

### Example 4: Building the naive `AList` (the second live-coded demo)

**Constructor.** We need memory boxes for the items and a count of how many are in use.

```java
public class AList {
    private int[] items;
    private int size;

    /** Creates an empty list. */
    public AList() {
        items = new int[100];
        size = 0;
    }
}
```

The array length (the slides used `100`, the in-class code used `11`, chosen by an audience member) is arbitrary and is a limitation fixed next lecture. Critically, `size` starts at `0`, **not** at the array length: `size` is the number of items in the *list*, not the number of boxes in the *array*.

**`addLast`: deriving the pattern.** Rather than guessing, write out small examples:

```
after constructor:   [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]   size = 0
addLast(6):          [6, 0, 0, 0, 0, 0, 0, 0, 0, 0]   size = 1
addLast(9):          [6, 9, 0, 0, 0, 0, 0, 0, 0, 0]   size = 2
addLast(-1):         [6, 9, -1, 0, 0, 0, 0, 0, 0, 0]  size = 3
```

Pattern: the first item went to index 0 when `size` was 0; the second to index 1 when `size` was 1; the third to index 2 when `size` was 2. So the next item always goes into position `size`. That is invariant #1, and it directly gives the code:

```java
/** Inserts x into the back of the list. */
public void addLast(int x) {
    items[size] = x;
    size += 1;
}
```

The instructor emphasized: the way to get here in real life is to write the examples on paper and read off the pattern, not to guess between `size` and `size + 1`.

**`getLast`.** Since the next insertion goes at `size`, the most recent insertion is at `size - 1`. That is invariant #3:

```java
/** Returns the item from the back of the list. */
public int getLast() {
    return items[size - 1];
}
```

**`get`.** This is the whole payoff for switching to arrays. No walking:

```java
/** Gets the ith item in the list (0 is the front). */
public int get(int i) {
    return items[i];
}
```

**`size`.** Invariant #2 says `size` is always the item count, so:

```java
/** Returns the number of items in the list. */
public int size() {
    return size;
}
```

### Example 5: `removeLast` and the abstract vs. the concrete

Setup: `size = 6`, `items` points at a length-100 array whose first six entries are `5, 3, 1, 7, 22, -1` and whose remaining 94 entries are `0`.

The user's mental model of `removeLast()` is `[5, 3, 1, 7, 22, -1]` becoming `[5, 3, 1, 7, 22]`.

The concrete question: **which memory boxes must change?** The candidates were:

- **`items`?** No. Changing `items` means making it point at an entirely different array. Unnecessary.
- **`size`?** Yes. It must go from `6` to `5`. This is forced by invariant #2.
- **`items[i]` for some `i`?** Not required. Zeroing out `items[5]` is optional.

Check the invariants after only changing `size` to `5`:
1. Next insertion goes at position `size` = 5. Correct: index 5 currently holds the stale `-1`, which a subsequent `addLast` will simply overwrite.
2. `size` = 5 is the number of items. Correct.
3. Last item is at `size - 1` = 4, which holds `22`. Correct.

All three hold, so changing `size` alone is sufficient for correctness. The minimal implementation:

```java
/** Deletes item from back of list and returns deleted item. */
public int removeLast() {
    int x = getLast();
    size = size - 1;
    return x;
}
```

Note the ordering: `getLast()` must be called **before** decrementing `size`, since `getLast` reads `items[size - 1]`.

The equivalent version with optional zeroing, from the summary slide:

```java
public int removeLast() {
    int x = items[size - 1];
    items[size - 1] = 0;   // not necessary to preserve invariants
    size -= 1;
    return x;
}
```

Arguments raised in lecture for zeroing anyway: aesthetics; easier debugging in the visualizer (you don't see a stale value); and, most substantively, that `get(5)` would otherwise still reveal the removed `-1`. The instructor's response: there is no specified behavior for an out-of-bounds `get`, and the cleaner fix is to make out-of-bounds access impossible rather than to scrub the array:

```java
public int get(int i) {
    if (i >= items.length) {
        throw new IllegalArgumentException();
    }
    return items[i];
}
```

The instructor's stated preference is to leave the stale value alone (zeroing "wastes computation"), but called both choices reasonable.

### Example 6 (bonus section): arrays vs. classes

Both organize a bunch of memory boxes, and both have a fixed number of boxes (an array's length cannot change; a class's fields cannot be added or removed). The differences:

| | Arrays | Classes |
|---|---|---|
| Access | `[]` notation, numbered | dot notation, named |
| Box types | all boxes same type | boxes may differ |
| Methods | none | yes |

The notable consequence: **array indices can be computed at runtime, but field names cannot.**

```java
int[] x = new int[]{100, 101, 102, 103};
int indexOfInterest = askUser();
int k = x[indexOfInterest];
System.out.println(k);
```

```
$ java ArrayDemo
What index do you want? 2
102
```

Versus:

```java
String fieldOfInterest = "mass";
Planet earth = new Planet(6e24, "earth");
double mass = earth[fieldOfInterest];   // error: array required, but Planet found
double mass = earth.fieldOfInterest;    // error: cannot find symbol
```

The Java compiler does not treat the text on either side of a dot as an expression, so it is never evaluated. Only hard-coded dot notation works (`double w = p.mass;`). There is a *reflection* library that can access fields by string name, but it is not for casual use, and **you should never use reflection in any 61B program**.

(extra context) The Java arrays vs. other languages comparison from the textbook appendix: Java arrays have no slicing syntax (unlike Python), cannot be shrunk or expanded (unlike Ruby), have no member methods (unlike JavaScript), and must hold values of a single type (unlike Python).

### Example 7 (extra context, from the textbook appendix): array copying and 2D arrays

These appear in the textbook chapter but were not covered in the lecture itself.

```java
int[] b = {9, 10, 11};
System.arraycopy(b, 0, x, 3, 2);
```

`System.arraycopy` takes: source array, start index in source, destination array, start index in destination, number of items to copy. The above is Python's `x[3:5] = b[0:2]`. It is usually faster than a loop and more compact, at some cost to readability. Java arrays bounds-check at **runtime**, so writing past the end of a destination array compiles fine and then crashes with an `ArrayIndexOutOfBoundsException`.

A "2D array" in Java is really an array of arrays:

```java
int[][] pascalsTriangle = new int[4][];   // 4 boxes, each holding an int[] reference
pascalsTriangle[0] = new int[]{1};
pascalsTriangle[1] = new int[]{1, 1};
pascalsTriangle[2] = new int[]{1, 2, 1};
pascalsTriangle[3] = new int[]{1, 3, 3, 1};
```

`new int[4][]` creates exactly four reference boxes; `new int[4][4]` additionally creates four length-4 int arrays. The distinction matters for aliasing: assigning `z[0] = x[0]` shares the underlying row (so `z[0][0] = -z[0][0]` also changes `x[0][0]`), whereas `System.arraycopy(x[0], 0, w[0], 0, 3)` copies values into a separate row (so `w[0][0] = -w[0][0]` leaves `x[0][0]` untouched).

---

## Common Pitfalls

1. **Thinking deletion means "making the node disappear."** In a linked list, deletion *is* the act of changing pointers. If you don't change a pointer, nothing was deleted. The orphaned node still exists in memory until Java's garbage collector reclaims it.

2. **Believing a `last` pointer fixes everything.** It fixes `addLast` and `getLast`. It does not fix `removeLast`, because you need the second-to-last node.

3. **Chasing the problem with `secondToLast`.** This just relocates the problem to the third-to-last node. Back-pointers on every node are the actual fix.

4. **Confusing arrow targets.** An arrow from `last` or `sentinel` points at an entire node object, not at one field inside it.

5. **Forgetting the sentinel's item is garbage.** `sentinel.item` is never the first item. Use `sentinel.next.item`.

6. **Re-declaring the type parameter on the nested class.** `private class Node<Cow>` inside `class SLList<Cow>` compiles but creates a *different* type parameter with the same name, generating baffling errors. Declare the generic type once, at the top of the file only.

7. **Making the nested class `static` in a generic class.** A static nested class cannot use the outer class's type parameter.

8. **Putting a primitive in angle brackets.** `SLList<int>` is illegal. Use `SLList<Integer>`.

9. **Setting the generic sentinel's item to a number.** Once items are of type `Cow`, the placeholder must be `null`, not `63` or `-420`.

10. **Initializing `AList`'s `size` to the array length.** `size` is the number of items in the *list*. It starts at 0 even though `items.length` is 100 (or 11).

11. **Off-by-one between `size` and `size - 1`.** Insertion goes at `size`; the last item lives at `size - 1`. Derive this from examples rather than guessing.

12. **Decrementing `size` before reading the last item in `removeLast`.** `getLast()` depends on `size`, so read first, then decrement.

13. **Assuming you must zero out the removed slot.** You don't, as far as the invariants are concerned. It is a taste/debugging decision, not a correctness one.

14. **Expecting `get` on a linked list to be fast.** It is linear, worst case around N/2 for a doubly linked list, no matter how cleverly you write it.

15. **Trying to change an array's length.** Arrays have a fixed length. Handling growth is next lecture's topic.

---

## Likely Exam Points

**1. Why `.last` alone is insufficient**

> *Q:* An `SLList` has a sentinel, a `size` field, and a `last` pointer. Which of `addLast`, `getLast`, `removeLast` is slow on long lists, and why?

*A:* `removeLast`. Removing the last node requires setting the second-to-last node's `next` to `null` and re-aiming `last` at the second-to-last node. With only forward links you must walk from the sentinel to find that node, which takes time proportional to the list's length. `addLast` and `getLast` are fast because they only need `last` itself.

**2. Circular-sentinel navigation**

> *Q:* For a circular-sentinel `DLList` with `size = 3` holding `[5, 17, 38]`, write expressions for the first item, the last item, and the middle item. What is `sentinel.next` for an empty circular list?

*A:* First: `sentinel.next.item` (5). Last: `sentinel.prev.item` (38). Middle: `sentinel.next.next.item` or equivalently `sentinel.prev.prev.item` (17). For an empty circular list, `sentinel.next == sentinel` (and `sentinel.prev == sentinel`).

**3. Why the naive `DLList` needs a sentinel upgrade**

> *Q:* What is the "annoying special case" in a `DLList` with one front sentinel plus a `last` pointer, and what are the two fixes?

*A:* `last` sometimes points at the sentinel (when the list is empty) and sometimes at a real node, forcing special-case `if` branches in methods. Fix one: add a second sentinel (`sentFront` and `sentBack`). Fix two: make the list circular with a single sentinel whose `next` and `prev` point at itself when empty. Project 1B requires the circular approach.

**4. Generics syntax**

> *Q:* Which of these compile? (a) `SLList<int> a = new SLList<>();` (b) `SLList<String> b = new SLList<String>("hi");` (c) `SLList<Double> c = new SLList<>(5.3);`

*A:* (a) does not compile: generics require reference types, so use `Integer`. (b) compiles: repeating the type on the right is redundant but legal. (c) compiles: `5.3` is autoboxed to a `Double`.

**5. Generics inside the implementing file**

> *Q:* A student writes `public class DLList<Item>` and then, inside it, `private class Node<Item> { ... }`. What goes wrong?

*A:* The nested class declares a *new* type parameter that shadows the outer one despite sharing the name, so the two `Item`s are unrelated types. This produces confusing type errors. Declare the type parameter once, on the outer class only, and leave the (non-static) nested class undecorated.

**6. `get` performance in linked vs. array lists**

> *Q:* For a doubly linked list of length 1,000,000, roughly how many pointer hops does `get(500000)` take, and how does that compare to an `AList`?

*A:* About 500,000 hops (you can start from either end, so worst case is the middle, roughly N/2). An `AList` does `items[500000]`, which is a single constant-time access independent of list size.

**7. `AList` invariants and index arithmetic**

> *Q:* State the three `AList` invariants and use them to explain why `addLast` writes to `items[size]` while `getLast` reads `items[size - 1]`.

*A:* (i) The next item to be inserted goes into position `size`; (ii) `size` is always the number of items; (iii) the last item is always in position `size - 1`. Since indices start at 0 and `size` items occupy indices `0` through `size - 1`, the first free slot is exactly `size` (so `addLast` writes there, then increments `size` to restore the invariants), and the most recently added item sits at `size - 1` (so `getLast` reads there).

**8. Which memory boxes change in `removeLast`**

> *Q:* An `AList` has `size = 6`, `items` pointing at a length-100 array with first six entries `5, 3, 1, 7, 22, -1`. On `removeLast()`, which memory boxes *must* change: `size`, `items`, `items[i]` for some `i`, or several of these?

*A:* Only `size` (from 6 to 5). `items` must not change, since that would mean pointing at a different array. No `items[i]` needs changing: the stale `-1` at index 5 violates no invariant and will be overwritten by the next `addLast`. Zeroing it out is optional (defensible for debugging and to prevent `get(5)` from revealing it, but the cleaner fix for out-of-bounds `get` is to throw an exception).

**9. Trace an `AList`**

> *Q:* After `AList a = new AList(); a.addLast(6); a.addLast(9); a.addLast(-1); a.removeLast(); a.addLast(4);` what are `a.size()`, `a.getLast()`, and `a.get(2)`?

*A:* `size()` is 3. The array holds `6, 9, 4` in positions 0, 1, 2. `removeLast()` returned `-1` and dropped `size` to 2, then `addLast(4)` wrote `4` into position 2 (overwriting the stale `-1`) and raised `size` to 3. `getLast()` is `items[2]` = 4, and `get(2)` is also 4.

**10. Arrays vs. classes**

> *Q:* Why can you write `x[indexOfInterest]` where `indexOfInterest` is computed at runtime, but not `p[fieldOfInterest]` or `p.fieldOfInterest` for a `String fieldOfInterest`?

*A:* Array indices are expressions evaluated at runtime, and uniform box sizes let the machine compute the address. Class fields are named, and the Java compiler does not treat the text on either side of a dot as an expression, so a field name must be hard-coded at compile time. Reflection can do this but is banned in 61B.

**11. Array declaration vs. instantiation**

> *Q:* What does `int[] a;` create? What does `new int[]{0, 1, 2, 95, 4};` create? What does `int[] a = new int[]{0, 1, 2, 95, 4};` do?

*A:* `int[] a;` creates only a 64-bit box intended to hold a reference to an int array; no array object exists. `new int[]{...}` instantiates an anonymous int array object of length 5. The combined line does all three: declaration, instantiation, and assignment of the new object's address into `a`.

---

## Summary

- **`SLList` remaining problems** entering this lecture: slow to reach the end, stores only `int`, slow to reach the middle.
- **The allegory of the cave**: the user has a desire (a list); we control the concrete memory boxes. Naked recursion (`IntList`) cannot provide that separation.
- **Improvement #7 (Looking back)**: add `.last` *and* a `.prev` back-pointer in every node, yielding a **doubly linked list**. Back-pointers make `removeLast` fast by giving constant-time access to the second-to-last node. (`.last` alone only fixes `addLast` and `getLast`.)
- Deletion in a linked list means **changing pointers**, not making a node vanish.
- **Improvement #8 (Sentinel upgrade)**: the naive `DLList` has the special case that `last` sometimes points at the sentinel. Fix with either two sentinels (`sentFront`/`sentBack`) or a single **circular sentinel** that points at itself when the list is empty. Circular is required for Project 1B; harder to think about, much cleaner code.
- In a circular-sentinel list: first item is `sentinel.next.item`, last item is `sentinel.prev.item`.
- **Improvement #9 (Generics)**: declare a placeholder type parameter once at the top of the implementing class (`public class SLList<Cow>`), use it for every item type inside. Users write the type at declaration and `<>` at instantiation. Only reference types (`Integer`, `Double`, ...) go in the brackets. Keep nested classes non-static and do not re-declare the parameter on them. The generic sentinel's item becomes `null`.
- **Linked lists are bad at `get(int i)`**: you must scan, so it is slow for any `i` far from a sentinel, worst case about N/2 for a `DLList`. Constant-time `getLast`, linear-time `get`.
- **Arrays**: fixed length, numbered same-type boxes indexed `0` to `length - 1`, no methods. Random access is fast and essentially independent of array size, because all boxes are the same number of bits.
- **Naive `AList`**: an `int[] items` plus an `int size`. `addLast` writes `items[size]` then increments; `getLast` returns `items[size - 1]`; `get(i)` returns `items[i]`; `size()` returns `size`; `removeLast` reads the last item, decrements `size`, and returns the item.
- **Invariants** drive the code: next insertion goes at `size`; `size` is the item count; last item sits at `size - 1`. Methods may assume them and must preserve them.
- `removeLast` requires changing only `size`. Zeroing the vacated slot is optional; the principled way to stop `get` from exposing stale data is bounds checking with an exception.
- **Arrays vs. classes**: numbered vs. named boxes, uniform vs. mixed types, methods vs. none. Array indices can be computed at runtime; field names cannot. Never use reflection in 61B.
- **Still missing from `AList`** (coming Monday, Lecture 8): growing the array beyond its fixed capacity, and making `AList` generic. Front operations come in Project 2.
- Logistics: Project 1A due Fri 9/11 (tests only, no local compilation); Project 1B due Wed 9/16, limited to 4 submissions per day. Everything needed for 1B was covered here; `Deque.java` (interfaces) is covered the following Wednesday.
