<!-- Fri, Sep 11, 2026 | sources: code (no transcript available) -->
# Lecture 7: Lists 3

## Overview

This lecture closes out the linked list story and opens the array-based one. We start by noticing that `SLList.addLast` is slow because it walks the whole list, then work through a sequence of structural fixes: a `last` pointer (fast `addLast` and `getLast`, but still slow `removeLast`), back pointers giving a **doubly linked list** (`DLList`), and a **sentinel upgrade** (either two sentinels or a single circular sentinel) that removes the ugly special cases where `last` sometimes points at the sentinel. Next we make lists **generic**, so an `SLList<Cow>` can hold any reference type instead of only `int`s; the lecture code uses the deliberately silly placeholder name `Cow` to emphasize that the type parameter is just a name you invent. Finally we observe the fundamental weakness of any linked list, that `get(i)` requires walking `i` links, and pivot to **arrays**: fixed-length, numbered sequences of same-typed memory boxes with constant-time random access. The lecture's `AList` is a first, deliberately naive array-backed list with `addLast`, `get`, and `size`, whose limitations (fixed capacity 11, no resizing, no `removeLast`) set up the next lecture.

---

## Key Concepts

### 1. Why `addLast` is slow, and why a `last` pointer is only a partial fix

The naive `addLast` in `SLList` starts at the sentinel and inches a pointer `p` forward until `p.next == null`:

```java
public void addLast(Cow x) {
    size += 1;
    Node p = sentinel;
    while (p.next != null) {
        p = p.next;
    }
    p.next = new Node(x, null);
}
```

For a list of length N this touches N nodes. This is exactly the same disease we cured for `size` by caching the answer in an instance variable, so the same medicine suggests itself: cache a pointer to the last node.

```java
public class SLList {
    private IntNode sentinel;
    private IntNode last;
    private int size;

    public void addLast(int x) {
        last.next = new IntNode(x, null);
        last = last.next;
        size += 1;
    }
}
```

Now `addLast` and `getLast` are constant time. But `removeLast` is still slow, and this is the crucial insight: after deleting the final node, `last` must be updated to point at the **second-to-last** node, and there is no way to find that node from `last` alone, because the links only go forward. You have to walk from the front again.

Caching `secondToLast` does not save you either. It just pushes the problem back one step: after a removal, `secondToLast` must become the third-to-last node, and finding *that* requires another full walk. Any finite number of "last k" pointers loses to a sufficiently long sequence of removals. The problem is structural, not a matter of adding more bookkeeping.

### 2. Improvement #7: back pointers (the doubly linked list)

The real fix is to let every node know its predecessor:

```java
public class IntNode {
    public IntNode prev;
    public int item;
    public IntNode next;
}
```

A list whose nodes have both `prev` and `next` is a **doubly linked list**, abbreviated `DLList`, in contrast to the **singly linked list** `SLList`. (This is the reveal of why the previous chapter used the awkward name `SLList`: it was always anticipating a doubly linked sibling.)

With back pointers, `removeLast` becomes constant time: from `last`, follow `last.prev` to reach the second-to-last node in one step, set its `next` to null, and set `last` to it. Adding, getting, and removing at **both** ends is now fast.

The cost is code complexity. Every structural operation must now maintain twice as many links, and every link must be kept consistent in both directions. (The textbook defers the implementation to Project 1 rather than walking through it.)

### 3. Improvement #8: the sentinel upgrade

There is a subtle wart in the naive `DLList` design: for an empty list, what does `last` point to? It has to point at the sentinel, since there is no real node. But for a nonempty list, `last` points at a real node. That "sometimes sentinel, sometimes real node" ambiguity is exactly the kind of thing that breeds special-case `if` statements, which is the same problem sentinels were introduced to kill in the first place. Methods like `addLast` and `removeLast` end up needing to ask "is the list empty?" before they can proceed.

Two clean fixes:

- **Two sentinels.** Put one sentinel node at the front and a separate one at the back. Real items always live strictly between them. `last` conceptually becomes `backSentinel.prev`, which is always a well-defined node reference.
- **Circular sentinel.** Use a single sentinel and make the list circular: the sentinel's `next` points at the first real item, the sentinel's `prev` points at the last real item, and the last real item's `next` points back at the sentinel. For an empty list, the sentinel simply points to itself in both directions.

Both eliminate special cases. The textbook author's stated aesthetic preference is the circular version, calling it "cleaner and more aesthetically beautiful." Either is fine; the important property is the **invariant** that every pointer you ever dereference points at a real `Node` object, never at `null`, so no method needs an emptiness check.

Box-and-pointer reasoning in words for the **circular, size 0** case: one `DLList` object with a `sentinel` field and a `size` field (0). `sentinel` points at a single `Node` whose `item` is garbage/null, whose `next` points back at that same node, and whose `prev` also points back at that same node. It is a self-loop.

For **circular, size 2** holding items `a` then `b`: `sentinel.next` points to node A, `A.next` points to node B, `B.next` points back to the sentinel. Going the other way, `sentinel.prev` points to B, `B.prev` points to A, and `A.prev` points back to the sentinel. Walking either direction from the sentinel and taking `size` steps lands you back on the sentinel.

### 4. Generics: lists that hold anything

Our lists so far hold only `int`s. This fails immediately for anything else:

```java
DLList d2 = new DLList("hello");   // does not compile: constructor wants an int
d2.addLast("world");               // does not compile
```

Java added **generics** in 2004 to solve this. The syntax: put an arbitrary placeholder in angle brackets right after the class name in the declaration, then use that placeholder wherever the item type should go.

The lecture code does exactly this with a placeholder called `Cow`:

```java
public class SLList<Cow> {
    private class Node {
        public Cow item;
        public Node next;

        public Node(Cow x, Node r) {
            item = x;
            next = r;
        }
    }
    ...
}
```

The name is arbitrary. `Cow`, `BleepBlorp`, `GloopGlop`, `TelbudorphMulticulus` all work identically. Conventional Java style uses single capital letters like `T`, `E`, or `K`/`V`, but the lecture's joke name makes the point that the compiler attaches no meaning to it: it is a placeholder for "whatever type someone gives us, like say `String`." (extra context: the single-letter convention is standard in real Java codebases; `T` for "type", `E` for "element".)

Note also that in the lecture code the node class is renamed from `IntNode` to `Node`, because it no longer holds `int`s, and it is marked `private`: nobody outside `SLList` has any business using it.

**Rules of thumb for using generics:**

- In the `.java` file **implementing** the data structure, write the generic type name **once**, at the top, after the class name. Do not repeat `<Cow>` on every method.
- In `.java` files that **use** the data structure, write the concrete type in angle brackets at declaration, and use the empty **diamond** `<>` at instantiation.
- Generics work only with **reference types**. You cannot write `<int>`. Use the wrapper classes: `Integer`, `Double`, `Character`, `Boolean`, `Long`, `Short`, `Byte`, `Float`.

```java
SLList<String> L = new SLList<>();
L.addLast("cat");
L.addLast("machine");

DLList<Integer> d1 = new DLList<>(5);
d1.insertFront(10);
```

Minor detail: `DLList<Integer> d1 = new DLList<Integer>(5);` is also perfectly valid; the right-hand `Integer` is just redundant when you are declaring a variable on the same line.

### 5. The linked list performance puzzle

Suppose we add `int get(int i)` to `DLList`. Why is it slower than `getLast`, and for which inputs is it worst?

Because we hold references only to the front and the back. To reach item #417 in a 10,000 element list, we must follow 417 forward links, one at a time. There is no arithmetic shortcut; the nodes can be scattered anywhere in memory and the only way to find node `i` is to ask node `i-1` where it is.

The best strategy is to start from whichever end is closer, so the **worst case is an item in the middle**, costing about N/2 link traversals. That is still **linear in the size of the list**. Meanwhile `getLast` is **constant time** regardless of N. (The course will formalize this with big-O and big-Theta later; for now, informal reasoning suffices.)

This is the motivation to change the underlying storage entirely.

### 6. Arrays

To build a list differently, we need a different way to get memory boxes. Previously we got them from variable declarations and class instantiations:

- `int x;` gives a 32-bit box holding an `int`.
- `Walrus w1;` gives a 64-bit box holding a `Walrus` **reference**.
- `Walrus w2 = new Walrus(30, 5.6);` gives 3 boxes total: the 64-bit reference box, plus a 32-bit `int` box for `size` and a 64-bit `double` box for `tuskSize` inside the new object.

An **array** is a special kind of object consisting of a **numbered** sequence of memory boxes, in contrast to class instances whose boxes are **named**. An array has:

- a **fixed** integer length N, and
- N memory boxes, all of the **same type**, numbered `0` through `N - 1`.

Arrays have **no methods**. (`length` is a field, not a method: `x.length`, not `x.length()`. Contrast `String`'s `.length()`, which is a method. (extra context: this asymmetry is a classic source of typos.))

**Three creation notations:**

```java
x = new int[3];                  // length 3, filled with default value 0
y = new int[]{1, 2, 3, 4, 5};    // length inferred from the listed values
int[] z = {9, 10, 11, 12, 13};   // same as above, but only valid with a declaration
```

None is better than the others. The first fills with defaults (`0` for `int`, `null` for reference types, `false` for `boolean`).

**Copying:** `System.arraycopy(src, srcStart, dest, destStart, numItems)` copies a run of elements. `System.arraycopy(b, 0, x, 3, 2)` is Python's `x[3:5] = b[0:2]`. It is usually faster than a hand-written loop and more compact, at some cost in readability.

**Bounds checking happens at runtime, not compile time.** Code that writes past the end of an array compiles fine and then crashes with `ArrayIndexOutOfBoundsException`.

**Arrays vs. classes.** Both organize a fixed number of memory boxes (an array's length cannot change, just as fields cannot be added to or removed from a class). The differences:

- Array boxes are **numbered** and accessed with `[]`; class boxes are **named** and accessed with dot notation.
- Array boxes must all be the **same type**; class boxes may differ in type.

The practical consequence: with `[]` you can choose the index **at runtime** (`x[askUserForInteger()]`), whereas you cannot choose a field name at runtime. `p[fieldOfInterest]` is a compile error ("array required, but Planet found") and `p.fieldOfInterest` looks for a field literally named `fieldOfInterest` ("cannot find symbol"). Java does have *reflection* for runtime field access, but it is bad style for ordinary programs: **never use reflection in 61B.**

**2D arrays** in Java are really arrays of arrays. `int[][] bamboozle = new int[4][];` creates exactly four boxes, each able to point at an `int[]` of unspecified length. `new int[4][4]` additionally allocates the four inner arrays.

### 7. The naive `AList`

Array indexing is **constant time** on a modern computer, which is precisely what linked lists could not offer. So we build a list backed by an array. The lecture's version:

```java
public class AList {
    public int[] items;
    public int size;

    public AList() {
        items = new int[11];
        size = 0;
    }

    public int size() {
        return size;
    }

    public void addLast(int x) {
        items[size] = x;
        size += 1;
    }

    public int get(int i) {
        return items[i];
    }
}
```

The design rests on a two-part **invariant**:

1. The next item to be added goes in `items[size]`.
2. `size` is the number of items in the list, so the items occupy `items[0]` through `items[size - 1]`.

Given that invariant, `addLast` is three words long and `get` is one. Both run in constant time. This is the big win over the linked list.

The key observation underlying `removeLast` (which the lecture code does not yet implement): **any change to the list must be reflected in a change to one or more memory boxes.** The list is an abstract idea; `size`, `items`, and the `items[i]` boxes are its concrete representation. Whatever the user does through `addLast`/`removeLast`, our memory boxes must end up in a state consistent with the invariants. For `removeLast`, decrementing `size` is sufficient: the item at the old `items[size - 1]` is now logically outside the list, even though the box still physically contains the old value. (extra context: this "stale garbage past `size`" point matters later for loitering/memory, and for a generic `AList` you would null out the vacated slot.)

**What is wrong with this `AList`:** the array has a hard-coded capacity of 11. The twelfth `addLast` writes `items[11]`, which is out of bounds, and the program crashes with `ArrayIndexOutOfBoundsException`. There is no resizing, no `removeLast`, and the fields are `public` rather than `private`. Fixing the capacity problem (resizing) is the next lecture's job.

---

## Definitions

- **`SLList` (singly linked list):** a list class whose nodes each hold an item and a single `next` reference to the rest of the list.
- **`DLList` (doubly linked list):** a list class whose nodes each hold an item, a `next` reference, and a `prev` reference to the previous node. Supports constant-time add/get/remove at both ends.
- **Back pointer (`prev`):** the reference in a node pointing to its predecessor; the ingredient that makes `removeLast` constant time.
- **Sentinel node:** a permanently present node holding no meaningful item, whose purpose is to guarantee that list-manipulating code never has to special-case an empty list or a null reference.
- **Two-sentinel list:** a `DLList` with a front sentinel and a separate back sentinel; every real item lies strictly between them.
- **Circular sentinel list:** a `DLList` with one sentinel where the last real node's `next` points back to the sentinel and the sentinel's `prev` points to the last real node. Empty list: the sentinel points to itself both ways.
- **Invariant:** a condition guaranteed to be true of a data structure before and after every operation (for example, "`items[0]` through `items[size-1]` hold the list's items").
- **Generic (parameterized) type:** a class declared with a type placeholder in angle brackets, e.g. `public class SLList<Cow>`, allowing the class to store any reference type.
- **Type parameter:** the placeholder name inside the angle brackets of a generic class declaration (`Cow` in the lecture code). Its name is arbitrary and has no meaning to the compiler.
- **Diamond operator (`<>`):** the empty angle brackets used at instantiation, e.g. `new SLList<>()`, letting the compiler infer the type argument from the declaration.
- **Wrapper class:** the reference-type counterpart of a primitive: `Integer`, `Double`, `Character`, `Boolean`, `Long`, `Short`, `Byte`, `Float`. Required inside angle brackets since generics cannot take primitives.
- **Array:** a special object consisting of a fixed-length, numbered sequence of memory boxes, all of the same type, indexed `0` through `length - 1`, with no methods.
- **`AList`:** an array-backed list class, offering constant-time `get(i)` via bracket indexing.
- **`System.arraycopy(src, srcPos, dest, destPos, len)`:** library method copying `len` elements from `src` starting at `srcPos` into `dest` starting at `destPos`.
- **`ArrayIndexOutOfBoundsException`:** the runtime error thrown when an array index is negative or at least `length`. Java checks bounds at runtime only, never at compile time.
- **Reflection:** a Java API for inspecting and accessing fields/methods by name at runtime. Forbidden in CS 61B.

---

## Worked Examples

### Example 1: Why `removeLast` is slow with only a `last` pointer

Consider an `SLList` with a sentinel, a `last` pointer, and items `[5, 10, 15]`.

Box and pointer reasoning in words: the `SLList` object holds `sentinel`, `last`, and `size = 3`. `sentinel.next` points at node(5), node(5).`next` points at node(10), node(10).`next` points at node(15), node(15).`next` is `null`. `last` points at node(15).

Now call `removeLast()`. We must end with items `[5, 10]`, meaning:

- node(10).`next` must become `null`, and
- `last` must point at node(10).

But from `last` (node(15)) there is **no way to reach node(10)**. Every arrow points forward. The only route to node(10) is to start at `sentinel` and walk: `sentinel -> node(5) -> node(10)`, checking at each step whether `p.next.next == null`. That walk is linear in N.

```java
// slow removeLast, linear time
public void removeLast() {
    IntNode p = sentinel;
    while (p.next.next != null) {   // stop when p is second-to-last
        p = p.next;
    }
    p.next = null;
    last = p;
    size -= 1;
}
```

Adding a `secondToLast` field does not help: after this removal, `secondToLast` would need to point at node(5), and reaching node(5) from node(10) requires the same forward-only walk. The structure, not the bookkeeping, is the problem.

### Example 2: The fix, `removeLast` on a circular-sentinel `DLList`

With `prev` pointers and a circular sentinel, the last real node is `sentinel.prev`, and the second-to-last is `sentinel.prev.prev`. Both are reachable in a constant number of hops.

```java
public Cow removeLast() {
    if (size == 0) { return null; }        // only guard needed: nothing to remove
    Node lastNode = sentinel.prev;         // constant time, no walking
    Node newLast = lastNode.prev;
    newLast.next = sentinel;               // relink forward
    sentinel.prev = newLast;               // relink backward
    size -= 1;
    return lastNode.item;
}
```

(extra context: the lecture/textbook does not write this code out, deferring it to Project 1; it is included here to make the constant-time claim concrete. Note how the circular sentinel means we never test whether `newLast` is a real node or the sentinel itself: if the list had one item, `newLast` **is** the sentinel and the two relinking lines correctly restore the empty self-loop.)

Trace on a two-item list `[a, b]`:
1. `lastNode = sentinel.prev` = node(b).
2. `newLast = node(b).prev` = node(a).
3. `node(a).next = sentinel`.
4. `sentinel.prev = node(a)`.
5. `size` goes 2 to 1. Node(b) is now unreachable and will be garbage collected.

Trace on a one-item list `[a]`:
1. `lastNode = sentinel.prev` = node(a).
2. `newLast = node(a).prev` = **sentinel**.
3. `sentinel.next = sentinel`.
4. `sentinel.prev = sentinel`.
5. `size` goes 1 to 0. We are back to the empty self-loop, with no special case written.

### Example 3: The generic `SLList` from lecture

```java
public class SLList<Cow> {
    private class Node {
        public Cow item;
        public Node next;

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

    /** Adds x to the front of the list */
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

    /** Add to the end of the list. */
    public void addLast(Cow x) {
        size += 1;
        Node p = sentinel;
        while (p.next != null) {   // inch p along until it's the last node
            p = p.next;
        }
        p.next = new Node(x, null);
    }

    static void main() {
        SLList<String> L = new SLList<>();
        L.addLast("cat");
        L.addLast("machine");
    }
}
```

Things to notice, line by line:

- `<Cow>` appears **exactly once**, on the class declaration. Everywhere an item's type is needed (`Cow item;`, `Node(Cow x, ...)`, `addFirst(Cow x)`, `Cow getFirst()`), we write `Cow`.
- The node class is `private`: "nobody else would ever use `Node` except for this class."
- `Node` does **not** need its own type parameter here. Because it is a (non-static) inner class of `SLList<Cow>`, it can just use the enclosing class's `Cow` directly.
- The sentinel is constructed as `new Node(null, null)`. Its `item` is `null` and is never read: the first real item is at `sentinel.next.item`. This works cleanly with generics because `Cow` is always a reference type, so `null` is always a legal value for it.
- `addLast` still walks the list. Making the class generic changes **nothing** about performance; generics are about types, not speed.

Trace of `main`:
1. `new SLList<>()` runs the no-arg constructor: `size = 0`, `sentinel` points at a fresh `Node` with `item = null`, `next = null`. The `<>` infers `Cow = String` from the declared type `SLList<String>`.
2. `L.addLast("cat")`: `size` becomes 1. `p` starts at sentinel; `p.next` is `null`, so the loop body never runs. `sentinel.next = new Node("cat", null)`.
3. `L.addLast("machine")`: `size` becomes 2. `p` starts at sentinel, `p.next` is node("cat") which is not null, so `p` advances to node("cat"). Now `p.next` is `null`, loop exits. `node("cat").next = new Node("machine", null)`.

Final picture: `L.sentinel -> Node(null) -> Node("cat") -> Node("machine") -> null`, `L.size == 2`.

### Example 4: What breaks if you try to use a primitive

```java
SLList<int> bad = new SLList<>();       // COMPILE ERROR: unexpected type
SLList<Integer> good = new SLList<>();  // fine
good.addLast(5);                        // fine: autoboxed to Integer.valueOf(5)
```

Generics only work over reference types. `Integer` is the reference counterpart of `int`. (extra context: the automatic conversion from `int` to `Integer` is called **autoboxing**; the course covers it in more detail later.)

### Example 5: The naive `AList`, traced

```java
public class AList {
    public int[] items;
    public int size;

    public AList() {
        items = new int[11];
        size = 0;
    }

    public int size() { return size; }

    public void addLast(int x) {
        items[size] = x;
        size += 1;
    }

    public int get(int i) { return items[i]; }
}
```

Trace:

```java
AList L = new AList();
L.addLast(3);
L.addLast(4);
L.addLast(5);
System.out.println(L.get(1));   // 4
```

1. Constructor: `items` points at a new `int[11]`, every box containing the default `0`. Memory: `[0 0 0 0 0 0 0 0 0 0 0]`. `size = 0`.
2. `addLast(3)`: `items[0] = 3`, so `[3 0 0 0 0 0 0 0 0 0 0]`; `size` becomes 1.
3. `addLast(4)`: `items[1] = 4`, so `[3 4 0 0 0 0 0 0 0 0 0]`; `size` becomes 2.
4. `addLast(5)`: `items[2] = 5`, so `[3 4 5 0 0 0 0 0 0 0 0]`; `size` becomes 3.
5. `get(1)` returns `items[1]`, which is `4`, with **no walking at all**, just one indexed lookup.

Box and pointer reasoning in words: the `AList` object has two boxes, `items` (a 64-bit reference) and `size` (a 32-bit `int`). The `items` box points at a separate array object elsewhere on the heap, which itself is 11 numbered 32-bit boxes. `size = 3` tells us boxes 0, 1, 2 are meaningful and boxes 3 through 10 are garbage that happens to be `0`.

Notice the invariant at work: `size` doubles as both "how many items" and "where the next item goes." That is why `addLast` is just two lines.

Now the failure:

```java
AList L = new AList();
for (int i = 0; i < 12; i += 1) {
    L.addLast(i);
}
```

On the twelfth call, `size` is 11, and `items[11] = x` indexes past the end of an 11-element array. It **compiles fine** (Java does not bounds check at compile time) and crashes at runtime with `ArrayIndexOutOfBoundsException: Index 11 out of bounds for length 11`.

### Example 6: Array reference semantics (`System.arraycopy` vs. aliasing)

From the textbook's Exercise 2.4.1:

```java
int[][] x = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};

int[][] z = new int[3][];
z[0] = x[0];            // z[0] and x[0] are the SAME array object (aliased)
z[1] = x[1];
z[2] = x[2];
z[0][0] = -z[0][0];     // mutates the shared array

int[][] w = new int[3][3];
System.arraycopy(x[0], 0, w[0], 0, 3);   // copies VALUES into w's own arrays
System.arraycopy(x[1], 0, w[1], 0, 3);
System.arraycopy(x[2], 0, w[2], 0, 3);
w[0][0] = -w[0][0];     // mutates only w's array
```

Step by step:

- `new int[3][]` makes three reference boxes, all `null`. Assigning `z[0] = x[0]` copies a **reference**, not the contents. Now `x[0]` and `z[0]` name the same three-element array.
- `z[0][0] = -z[0][0]` writes `-1` into that shared array, so `x[0][0]` also becomes `-1`.
- `new int[3][3]` makes three reference boxes each pointing at a **distinct** new `int[3]` full of zeros. `System.arraycopy` copies the three values from `x[0]` into `w[0]`'s own array. At this moment `x[0][0]` is already `-1`, so `w[0][0]` receives `-1`.
- `w[0][0] = -w[0][0]` flips `w[0][0]` to `1`, affecting nothing else.

**Answers: `x[0][0]` is `-1`, `w[0][0]` is `1`.**

The lesson: assignment of an array copies only the arrow; `System.arraycopy` (or a loop) copies the contents.

---

## Common Pitfalls

1. **Thinking a `last` pointer makes `removeLast` fast.** It makes `addLast` and `getLast` fast. `removeLast` still needs the second-to-last node, which a forward-only list cannot supply quickly.
2. **Thinking a `secondToLast` pointer rescues you.** It just moves the problem one node back. Only `prev` pointers actually solve it.
3. **Forgetting to update *both* links in a `DLList`.** Every structural change touches a `next` and a `prev`. Half-updated lists behave correctly when walked forward and mysteriously wrong when walked backward.
4. **Repeating the type parameter on every method.** `public <Cow> void addFirst(Cow x)` is wrong (it declares a *new*, unrelated type parameter). The parameter belongs only on the class declaration.
5. **Writing `SLList<int>` or `SLList<double>`.** Generics require reference types; use `Integer`, `Double`, etc.
6. **Writing the concrete type inside the implementation file.** Inside `SLList.java` you write `Cow`, never `String`. The whole point is that the implementation does not know what type it will hold.
7. **Confusing `x.length` (array field, no parentheses) with `s.length()` (String method, with parentheses).**
8. **Expecting the compiler to catch out-of-bounds access.** Java bounds-checks at **runtime** only. Bad indices compile cleanly and blow up during execution.
9. **Assuming `y = x` copies an array.** It copies the reference. Both names now refer to one array, and mutating through either is visible through the other.
10. **Confusing the array's `length` (capacity) with the list's `size` (number of items).** In `AList`, `items.length` is 11 from the start, but `size` is the count of real items. The boxes from `size` to `items.length - 1` contain stale garbage that is **not** part of the list.
11. **Believing the naive `AList` is a finished data structure.** It crashes on the twelfth `addLast`, has no `removeLast`, and has `public` fields. It is a stepping stone, not the goal.
12. **Believing generics affect performance.** `SLList<String>.addLast` is exactly as slow as the `int` version. Generics are a compile-time typing feature.
13. **Reading `sentinel.item`.** It is meaningless (`null` in the lecture code). The first real item is at `sentinel.next.item`.

---

## Likely Exam Points

### 1. Which operations are fast for which structure?

You will be asked to fill in a table of constant vs. linear for `addFirst`, `addLast`, `getFirst`, `getLast`, `removeLast`, `get(i)` across `SLList` (with and without `last`) and `DLList`.

**Q:** For a `DLList` with a circular sentinel, classify `addLast`, `removeLast`, and `get(i)` as constant or linear time in the list length N.

**A:** `addLast` is constant (splice a new node between `sentinel.prev` and `sentinel`). `removeLast` is constant (`sentinel.prev.prev` is reachable in two hops). `get(i)` is **linear**: with only front/back references, reaching index `i` requires walking links, and the worst case (middle of the list) costs about N/2 steps.

### 2. Why doesn't `secondToLast` fix `removeLast`?

**Q:** A student proposes adding both `last` and `secondToLast` pointers to an `SLList` to make `removeLast` constant time. Explain precisely why this fails.

**A:** The first `removeLast` is fast: set `secondToLast.next = null` and `last = secondToLast`. But now `secondToLast` is stale and must be updated to the new second-to-last node, which is the old third-to-last node. With only forward pointers there is no way to reach it except by walking from the front, which is linear. So the *first* removal is fast and every subsequent one is slow. Generalizing, any fixed number of trailing pointers fails after that many removals. The cure is `prev` pointers.

### 3. Sentinel design and special cases

**Q:** Give one concrete `DLList` method that needs an ugly special case when `last` is a plain pointer that can point at the sentinel, and explain how the circular sentinel removes it.

**A:** `removeLast` on a one-item list. With a plain `last` pointer, after removing the single real node, `last` must be reassigned to the sentinel, so the code needs an `if` checking whether `last.prev` is the sentinel (or whether `size == 1`). Similarly `addLast` on an empty list must handle `last` currently being the sentinel. With a circular sentinel, `sentinel.prev` is always a legal node reference (possibly the sentinel itself), so the same relinking lines work for size 0, 1, and N without any branch.

### 4. Writing/reading generic class syntax

**Q:** Here is a broken generic class. Fix every error.

```java
public class Box {
    private T item;
    public Box(T x) { item = x; }
    public T get() { return item; }
}
// usage:
Box<int> b = new Box<int>(5);
```

**A:** Two fixes. First, declare the type parameter on the class: `public class Box<T> { ... }`. Without it, `T` is an unknown symbol. Second, generics cannot be instantiated over primitives, so use the wrapper: `Box<Integer> b = new Box<>(5);` (the diamond is preferred, and `5` autoboxes to `Integer`). The method bodies themselves are already correct: `T` is used, not a concrete type.

### 5. Generic `SLList` trace

**Q:** After running the lecture's `main`, what is the value of `L.size`, and what expression retrieves the string `"machine"`?

**A:** `L.size == 2`. Since `sentinel.item` is unused, the first real item is `sentinel.next.item` (`"cat"`), so `"machine"` is at `sentinel.next.next.item`. Externally, only `getFirst()` exists in this class, returning `"cat"`; there is no public accessor for the last item in the lecture code.

### 6. Array aliasing vs. copying

**Q:** What does this print?

```java
int[] x = {1, 2, 3};
int[] y = x;
y[0] = 99;
int[] w = new int[3];
System.arraycopy(x, 0, w, 0, 3);
w[1] = 42;
System.out.println(x[0] + " " + x[1]);
```

**A:** `99 2`. `y = x` copies the reference, so `y[0] = 99` mutates the one shared array and `x[0]` becomes 99. `System.arraycopy` copies values into `w`'s own separate array, so `w[1] = 42` leaves `x[1]` at 2.

### 7. `AList` invariant and the `size` field

**Q:** In the lecture's `AList`, explain why `addLast` writes to `items[size]` rather than `items[size - 1]` or `items[size + 1]`, and state the invariant that makes this correct.

**A:** The invariant is that the list's items occupy `items[0]` through `items[size - 1]`, so the first unused box is exactly `items[size]`. Writing to `items[size - 1]` would overwrite the current last item; `items[size + 1]` would leave a hole at index `size`, breaking the invariant that items are contiguous from 0. After the write, `size += 1` restores the invariant.

### 8. When does the naive `AList` break?

**Q:** Using the lecture's `AList`, how many successful `addLast` calls can you make on a freshly constructed list, and what exactly happens on the next one?

**A:** 11 (indices 0 through 10, since the constructor allocates `new int[11]`). The twelfth call evaluates `items[11] = x` with `size == 11`, which is out of bounds for an array of length 11. The code compiles without complaint because Java checks array bounds only at runtime; at runtime it throws `ArrayIndexOutOfBoundsException`. Fixing this requires resizing, i.e. allocating a bigger array and copying the items over.

### 9. `removeLast` for `AList`

**Q:** Write `removeLast` for the lecture's `AList` and justify its correctness using the "any change to the list must be a change to memory boxes" principle.

**A:**

```java
public int removeLast() {
    int x = items[size - 1];
    size -= 1;
    return x;
}
```

The abstract change ("the list is one shorter") is realized concretely by changing the `size` box. Because the invariant says the list is `items[0]` through `items[size - 1]`, decrementing `size` immediately makes the old last element logically absent, and `addLast` will later overwrite that box. No other memory box needs to change; leaving the stale value in `items[size]` is harmless for an `int` array. (extra context: for a generic `AList<T>` you would also set the vacated slot to `null` to avoid retaining a reference to an object that should be garbage collected.)

### 10. Arrays vs. classes

**Q:** Name two structural differences between the memory boxes of an array and those of a class instance, and give one capability that follows from them.

**A:** (1) Array boxes are numbered and accessed with `[]`; class boxes are named and accessed with dot notation. (2) All array boxes have the same type; class boxes may have different types. The consequence: the index of an array box can be computed at **runtime** (`x[askUserForInteger()]`), while a field name cannot be chosen at runtime (`p.fieldOfInterest` is a compile error unless a field is literally named that). Reflection can do it, but it is bad style and banned in 61B.

---

## Summary

- Naive `addLast` walks the whole list, so it is linear; caching a `last` pointer makes `addLast` and `getLast` constant.
- A `last` pointer does **not** fix `removeLast`, because the second-to-last node is unreachable in a forward-only list. Adding `secondToLast` just defers the problem by one node.
- **Improvement #7:** add `prev` back pointers to every node, giving a **doubly linked list** (`DLList`). Add/get/remove at both ends becomes constant time, at the cost of more complex code.
- **Improvement #8:** upgrade the sentinel, either **two sentinels** (front and back) or a **circular sentinel** (one node, pointing to itself when empty), to eliminate the case where `last` sometimes points at the sentinel and sometimes at a real node. Both kill special cases; circular is the author's aesthetic favorite.
- **Generics** (Java, 2004) let a class hold any **reference** type. Declare the placeholder once, in angle brackets after the class name (the lecture used `<Cow>`); use it wherever the item type appears; use `<>` (the diamond) at instantiation; use wrapper classes (`Integer`, `Double`, ...) for primitives.
- Generics change types, not performance, and the inner `Node` class can be `private` and reuse the enclosing class's type parameter.
- Linked lists are fundamentally bad at `get(i)`: with only front/back references you must walk the links, worst case about N/2 steps, i.e. linear. `getLast` stays constant.
- **Arrays** are objects with a fixed length N and N numbered, same-typed memory boxes indexed 0 to N-1, no methods, `length` as a field. Three creation notations, all equivalent in worth. `System.arraycopy` copies contents; plain assignment copies only the reference.
- Java checks array bounds at **runtime** only, throwing `ArrayIndexOutOfBoundsException`.
- Arrays vs. classes: numbered/`[]` vs. named/dot, and uniform type vs. mixed types. Only the array index can be chosen at runtime. Never use reflection in 61B.
- Array indexing is **constant time**, which motivates the array-backed **`AList`**. Its invariant: items live in `items[0..size-1]`, and the next item goes in `items[size]`. Hence `addLast` and `get` are both tiny and constant time.
- The lecture's `AList` is deliberately naive: capacity is hard-coded at 11, so the twelfth `addLast` crashes. Resizing comes next.
