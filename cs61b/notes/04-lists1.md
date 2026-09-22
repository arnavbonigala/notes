<!-- Wed, Sep 2, 2026 | sources: code (no transcript available) -->
# Lecture 4: Lists 1

This lecture builds the simplest possible linked list in Java, the `IntList`, from nothing but a single class with two instance variables: an `int first` holding this node's value and an `IntList rest` pointing at the remainder of the list. The whole point is to see that a recursive data structure falls directly out of the reference semantics established in Lecture 3 (variables hold addresses, `null` is the "points at nothing" address, and `new` creates a box somewhere in memory). Once the bare class exists, we discover it is painful to use directly (`L.rest.rest.rest = ...`), so we adopt the standard object-oriented move of adding helper methods: `size` (recursive), `iterativeSize` (loop-based, using a pointer variable because you cannot reassign `this` in Java), and `get(int i)` (recursive index lookup, which is linear time and hints at why we will later want arrays). The lecture closes with a first taste of list *transformation* via `IntListTools.incrementRecursiveNonDestructive`, which introduces the destructive vs. non-destructive distinction that will dominate the next lecture.

---

## Key Concepts

### 1. A list is a recursive data structure

The entire `IntList` class is this:

```java
public class IntList {
    public int first; // this is the first item in the list
    public IntList rest; // this is the rest of the list

    public IntList(int f, IntList r) {
        first = f;
        rest = r;
    }
}
```

The crucial and initially disorienting line is `public IntList rest;`. A class is defining a variable whose type is *the class itself*. This is legal precisely because of what we learned in Lecture 3: `rest` does not contain an `IntList`, it contains a **reference** to one (64 bits of address, or the special all-zeros `null`). If Java tried to physically nest an `IntList` inside an `IntList` you would need infinite memory. Because it only stores an address, the box is a fixed, finite size.

The definition reads naturally in English: *a list is a first item, followed by the rest of the list.* The base case, "the rest of the list is nothing," is represented by `rest == null`.

If you took CS 61A, this is the same idea as the `Link` class there; the textbook explicitly notes "You may remember something like this from 61a called a 'Linked List'."

### 2. Box-and-pointer intuition

Since the slides' diagrams did not survive PDF extraction, here is the picture in words for `IntList L = new IntList(5, new IntList(10, new IntList(3, null)))`:

- `L` is a variable in the current stack frame holding an address.
- That address points at an `IntList` object in the heap containing two fields: `first = 5` and `rest = <address2>`.
- `<address2>` points at another `IntList` object with `first = 10`, `rest = <address3>`.
- `<address3>` points at a third `IntList` with `first = 3`, `rest = null`.

Drawn as arrows: `L -> [5 | *] -> [10 | *] -> [3 | X]` where `X` denotes `null`.

Every node is an independent object in the heap. Nothing about the objects knows it is "the whole list": each node is simultaneously a node *and* the head of a perfectly valid list of its own suffix. This is why `rest.size()` makes sense: `rest` *is* a list.

### 3. Building lists by hand is ugly (the motivation for everything after)

The lecture demonstrates two ways to construct `5 -> 10 -> 3`.

**Appending to the end** (the commented-out block in the lecture's `main`):

```java
IntList L = new IntList(5, null);
// let's make some space for the next item
L.rest = new IntList(10, null);
// let's make some space for the next next item
L.rest.rest = new IntList(3, null);
```

This reads left to right in the same order as the list, which is nice, but the lecture's own comment names the problem: "we get lots of `.rest.rest.rest` nonsense." Adding the 100th item requires 99 `.rest`s.

**Prepending to the front** (what the lecture actually runs):

```java
IntList L = new IntList(3, null);
// let's put 10 at the front
L = new IntList(10, L);
// then let's put 5 at the front
L = new IntList(5, L);
// 5 -> 10 -> 3
```

The textbook calls this "slightly nicer but harder to understand code." Each line creates a brand new node whose `rest` is the old list, then reassigns `L` to point at the new node. Nothing is ever mutated; only `L` is repointed. You have to build the list backwards, which is the cognitive cost.

Both are unpleasant enough that we do what object-oriented programmers do: add methods.

### 4. Recursion on lists: the shape of the code follows the shape of the data

Because the data is defined recursively ("a first, plus a rest"), the natural algorithms are recursive too.

```java
/** returns he size of this list. Me, that is. */
public int size() {
    if (rest == null) {
        return 1;
    }
    return 1 + rest.size();
}
```

Read it as a claim about `this`: "If I have no rest, I am a list of size 1. Otherwise my size is 1 plus my rest's size."

**Why the base case is `rest == null` and not `this == null`.** This is the lecture/textbook's favorite question. You might want to write:

```java
if (this == null) { return 0; }   // WRONG
```

It cannot work. To *call* `size()` at all you must call it on an object: `L.size()`. If `L` is `null`, Java dereferences a null reference to find the method and you get a `NullPointerException` before a single line of your method body runs. The check is never reached. So the test has to be one level up, on `rest`, which you can safely compare to `null` without dereferencing it.

A consequence worth noticing: `size()` cannot return 0. There is no way to represent an empty list with this design (an empty list would be `null`, and you cannot call a method on `null`). This is a real limitation of the "naked recursive" list and is exactly why later lectures will wrap `IntList` inside an `SLList` class.

### 5. Iteration on lists: you need a pointer variable because you cannot reassign `this`

```java
// and not using recursion this time
// we'll use a loop instead, less beautiful, but...
// let's sully ourselves and write the code
public int iterativeSize() {
    int totalSize = 0;
    IntList currentLocation = this;

    while (currentLocation != null) {
        totalSize += 1;
        currentLocation = currentLocation.rest;
    }

    return totalSize;
}
```

The textbook's version names the variable `p` and recommends that convention: "when you write iterative data structure code ... use the name `p` to remind yourself that the variable is holding a pointer. You need that pointer because you can't reassign `this` in Java." Writing `this = this.rest;` is a compile error. `this` is effectively final.

Trace it on `5 -> 10 -> 3`:

| iteration | `currentLocation` points at | `totalSize` after body |
|---|---|---|
| start | node(5) | 0 |
| 1 | node(5) -> then node(10) | 1 |
| 2 | node(10) -> then node(3) | 2 |
| 3 | node(3) -> then `null` | 3 |
| check | `null`, loop exits | 3 |

Note the loop condition is `!= null`, not `rest != null`. That is what lets this version correctly count and terminate, and it is the asymmetry with the recursive version: the recursive method stops *one node early* (at the last node) and returns 1, whereas the loop walks all the way off the end.

Also note the assignment `currentLocation = currentLocation.rest;` does not modify the list at all. It only repoints a local variable. This is the key reading skill: `p.rest = q` mutates the heap; `p = p.rest` moves a local pointer.

### 6. `get(int i)`: indexing a linked list

```java
// i'll do it recursively in class
public int get(int i) {
    if (i == 0) {
        return this.first; //return first;
    }
    // my 5th item
    // is my rest's 4th item
    return this.rest.get(i - 1);
}
```

The comment in the code is the entire insight: **my ith item is my rest's (i-1)th item.** Each recursive call strips one node off the front and decrements the index in lockstep, so they hit zero together at exactly the right node.

The lecture explicitly does not handle invalid indices. The textbook says "It doesn't matter how your code behaves for invalid `i`, either too big or too small." In practice, `i` too large walks off the end and throws a `NullPointerException` when `this.rest` is `null`; negative `i` recurses until it falls off the end similarly.

**Cost.** The textbook flags this immediately: "the method we've written takes linear time! That is, if you have a list that is 1,000,000 items long, then getting the last item is going to take much longer than it would if we had a small list. We'll see an alternate way to implement a list that will avoid this problem in a future lecture." There is no way to jump to index 500,000; you must follow 500,000 arrows. This is the single biggest weakness of linked lists and the motivation for array-based lists later in the course.

### 7. Destructive vs. non-destructive (first look)

```java
public class IntListTools {
    /** Returns a copy of L, with each value incremented by x.
     *  Because this is "non-destructive", the list at L should
     *  not be modified.
     */
    public static IntList incrementRecursiveNonDestructive(IntList L, int x) {
        if (L == null) {
            return null;
        }
        IntList incrementedList = new IntList(L.first + x, null);
        incrementedList.rest = incrementRecursiveNonDestructive(L.rest, x);
        return incrementedList;
    }
}
```

Two structural things changed compared to `size` and `get`:

1. **This is a `static` method taking `L` as a parameter**, not an instance method. That is what lets the base case be `if (L == null) return null;`. Because we never call a method *on* `L`, checking `L == null` is safe. This is the workaround for the "you can't check `this == null`" problem from §4, and it means this version correctly handles the empty list.
2. **Every node is freshly allocated with `new`.** The original list is read (`L.first`, `L.rest`) but never written to. No line in this method has the form `L.first = ...` or `L.rest = ...`. That is precisely what "non-destructive" means.

Trace on `L = 5 -> 10 -> 3` with `x = 10`:

- `incRND(node(5), 10)`: `L != null`, build `new IntList(15, null)`. Call `incRND(node(10), 10)` for its rest.
  - `incRND(node(10), 10)`: build `new IntList(20, null)`. Call `incRND(node(3), 10)`.
    - `incRND(node(3), 10)`: build `new IntList(13, null)`. Call `incRND(null, 10)`.
      - `incRND(null, 10)`: returns `null`. (This is the base case doing real work: it supplies the terminating `null`.)
    - sets `13`'s rest to `null`, returns node(13).
  - sets `20`'s rest to node(13), returns node(20).
- sets `15`'s rest to node(20), returns node(15).

Result: a brand-new chain `15 -> 20 -> 13`, with `L` still pointing at the untouched `5 -> 10 -> 3`. Six nodes now exist in the heap.

*(extra context)* A common stylistic compression of this method writes it as a single line, `return new IntList(L.first + x, incrementRecursiveNonDestructive(L.rest, x));`, which behaves identically. The lecture's two-step version (allocate, then fill in `rest`) is easier to trace, which is presumably why it was written that way.

---

## Definitions

- **IntList:** A class representing a non-empty list of `int`s, consisting of a public `int first` (this node's value) and a public `IntList rest` (a reference to the remainder of the list, or `null` if there is no remainder). Sometimes called a "naked recursive data structure" because users manipulate the nodes directly.

- **Linked list:** A list built out of nodes where each node stores a value and a reference to the next node. The same structure called `Link` in CS 61A.

- **Recursive data structure:** A data structure whose definition refers to itself. `IntList` is recursive because one of its fields has type `IntList`. Legal in Java only because object variables store references, not the objects themselves.

- **`first`:** The `int` value stored in this particular node.

- **`rest`:** A reference to the `IntList` containing everything after this node. `null` means this node is the last.

- **`null`:** The reference value meaning "points at no object." Used here as the terminator of a list and as the base-case marker. Dereferencing it (calling a method on it or reading a field of it) throws a `NullPointerException`.

- **Base case:** The non-recursive branch of a recursive method, which returns an answer without calling itself. For `size()` it is `rest == null`; for `get` it is `i == 0`; for `incrementRecursiveNonDestructive` it is `L == null`.

- **`this`:** The implicit reference to the object on which an instance method was invoked. It cannot be reassigned in Java, which is why iterative traversal requires a separate local pointer variable.

- **Pointer variable (conventionally `p`):** A local variable of reference type used to walk a data structure, e.g. `IntList p = this;` followed by `p = p.rest;`. Reassigning it moves the traversal; it does not modify the list.

- **`iterativeSize`:** A method computing the list's length using a `while` loop rather than recursion.

- **`get(int i)`:** Returns the value at index `i`, zero-indexed, so `L.get(0)` is the first item. Behavior on out-of-range `i` is unspecified in this lecture.

- **Non-destructive method:** A method that produces its result without modifying the input data structure; it allocates new objects instead of writing to existing ones.

- **Destructive method:** *(extra context, named in the lecture's method name by contrast but developed in Lists 2)* A method that modifies the input data structure in place, so the caller's list is changed after the call.

- **Linear time:** Runtime proportional to the number of items. `size`, `iterativeSize`, `get` on a late index, and `incrementRecursiveNonDestructive` are all linear in the list's length.

---

## Worked Examples

### Example 1: Building `5 -> 10 -> 3` two ways

**Backwards / prepending (the version the lecture runs):**

```java
IntList L = new IntList(3, null);
L = new IntList(10, L);
L = new IntList(5, L);
```

Step by step, in terms of boxes and arrows:

1. `new IntList(3, null)` allocates a node in the heap with `first = 3`, `rest = null`. `L` is assigned that node's address. Picture: `L -> [3 | X]`.
2. `new IntList(10, L)` is evaluated **before** the assignment. Java evaluates the right-hand side first, so the argument `L` is the *old* value, the address of node(3). A new node is allocated with `first = 10` and `rest = <address of node(3)>`. Only then is `L` reassigned to the new node. Picture: `L -> [10 | *] -> [3 | X]`. Node(3) itself was never touched.
3. Same thing again with 5: `L -> [5 | *] -> [10 | *] -> [3 | X]`.

This right-hand-side-first evaluation order is what makes the idiom work and is worth stating explicitly, because it looks circular at first glance ("L is defined in terms of L").

**Forwards / appending (the commented-out version):**

```java
IntList L = new IntList(5, null);
L.rest = new IntList(10, null);
L.rest.rest = new IntList(3, null);
```

1. `L -> [5 | X]`.
2. `L.rest = new IntList(10, null)` **mutates** node(5), overwriting its `rest` field from `null` to the address of the new node(10). Now `L -> [5 | *] -> [10 | X]`.
3. `L.rest.rest = ...` requires two dereferences to reach node(10), then overwrites its `rest`. Now `L -> [5 | *] -> [10 | *] -> [3 | X]`.

Both produce the same picture, but the second requires one more `.rest` for each additional element. Hence the lecture comment: "we get lots of `.rest.rest.rest` nonsense."

### Example 2: The lecture's `main`

```java
static void main() {
    IntList L = new IntList(3, null);
    L = new IntList(10, L);
    L = new IntList(5, L);

    // 5 -> 10 -> 3

    // should print 3
    IO.println(L.iterativeSize());
    IO.println(L.get(2));

    // L is 5 -> 10 -> 3
    IntList L2 = IntListTools.incrementRecursiveNonDestructive(L, 10);

    // L2 should be 15 -> 20 -> 13
}
```

Output line by line:

- `L.iterativeSize()` walks three nodes and prints `3`.
- `L.get(2)` prints `3` as well, but for a completely different reason: it is the *value* at index 2, which happens to also be 3. The lecture's comment "should print 3" applies to the size; the coincidence is worth noticing so you do not confuse the two. (In the `get_exercise` variant of the file, the list is `5 -> 10 -> 15` and `L.get(2)` prints `15`, which makes the distinction clearer.)
- `incrementRecursiveNonDestructive(L, 10)` builds a separate three-node list `15 -> 20 -> 13` and binds `L2` to it. After this line, `L` still shows `5 -> 10 -> 3`.

*(extra context)* `static void main()` with no `String[] args` and the `IO.println` helper are features of recent Java preview/simplified-launch syntax used in the course's 2026 code; the `get_exercise` file uses the traditional `public static void main(String[] args)` with `System.out.println`. Either is fine for your own code; match whatever your skeleton uses.

### Example 3: Tracing `size()` on `5 -> 10 -> 3`

Call `L.size()` where `L` points at node(5).

1. Frame 1: `this` = node(5). `rest` is node(10), not null, so evaluate `1 + rest.size()`. Suspend and call.
2. Frame 2: `this` = node(10). `rest` is node(3), not null, so evaluate `1 + rest.size()`. Suspend and call.
3. Frame 3: `this` = node(3). `rest` is `null`, base case hit, **return 1**.
4. Back in frame 2: `1 + 1 = 2`, return 2.
5. Back in frame 1: `1 + 2 = 3`, return 3.

Three stack frames were live at the deepest point. Contrast with `iterativeSize`, which uses one frame and one extra local variable regardless of list length. *(extra context)* For very long lists the recursive version can overflow the stack while the iterative one will not; this is the practical argument for "sullying ourselves" with a loop.

### Example 4: Tracing `get(2)` on `5 -> 10 -> 3`

1. `L.get(2)`: `this` = node(5), `i = 2`, not 0, so return `this.rest.get(1)`.
2. `node(10).get(1)`: `i = 1`, not 0, so return `this.rest.get(0)`.
3. `node(3).get(0)`: `i == 0`, base case, return `this.first`, which is `3`.
4. The 3 propagates back up unchanged through both frames. Final answer: `3`.

Notice the index and the position advance together: we moved forward 2 nodes and decremented `i` by 2.

**What happens on `L.get(3)`?** Frames for `i = 3, 2, 1` bring us to node(3) with `i = 0`... no: `L.get(3)` gives node(10) `i=2`, node(3) `i=1`, and node(3) is not the base case, so it evaluates `this.rest.get(0)` where `this.rest` is `null`. Calling `get` on `null` throws a `NullPointerException`. The lecture does not require you to guard against this.

### Example 5: Writing `get` iteratively *(extra context, the natural companion exercise)*

The lecture only does `get` recursively ("i'll do it recursively in class"), but the iterative version uses exactly the `p` pattern from `iterativeSize`:

```java
public int iterativeGet(int i) {
    IntList p = this;
    while (i > 0) {
        p = p.rest;
        i -= 1;
    }
    return p.first;
}
```

Each loop iteration advances the pointer one node and burns one unit of index, mirroring the recursive call. When `i` reaches 0 we are standing on the right node.

### Example 6: The `get_exercise` skeleton

The repository includes a stripped version for you to fill in:

```java
public int get(int i) {
    // TODO: Make this work
    return 0;
}

public static void main(String[] args) {
    IntList L = new IntList(15, null);
    L = new IntList(10, L);
    L = new IntList(5, L);

    System.out.println(L.iterativeSize()); // 3
    System.out.println(L.get(2));          // should print 15
}
```

Here the list is `5 -> 10 -> 15`, so `get(0) == 5`, `get(1) == 10`, `get(2) == 15`, matching the textbook's example exactly. Fill in the body with either the recursive or iterative version above.

---

## Common Pitfalls

1. **Writing `if (this == null) return 0;` as your base case.** It is unreachable. You already had to dereference a reference to get into the method, so if that reference were `null` you would have thrown a `NullPointerException` at the call site. Check `rest == null` instead, or make the method `static` and take the list as a parameter (as `IntListTools` does).

2. **Forgetting that `size()` can never return 0.** With this design there is no empty `IntList`; the empty list is represented by `null`, on which you cannot call methods. Do not write test cases expecting `emptyList.size() == 0`.

3. **Confusing `p = p.rest` with `p.rest = q`.** The first moves a local pointer and changes nothing in the heap. The second overwrites a field inside an object and changes the list for everyone holding a reference to it. Mixing these up is the number one source of linked-list bugs.

4. **Trying to write `this = this.rest;`.** Compile error. `this` cannot be reassigned. You must introduce a local variable.

5. **Off-by-one in the loop condition.** `iterativeSize` must test `p != null`, not `p.rest != null`. Testing `p.rest != null` would undercount by one. Conversely the recursive `size` tests `rest == null`, not `this == null`. The two methods use structurally different tests for good reasons; do not copy one into the other.

6. **Assuming `get` is fast.** It is linear. Writing a loop like `for (int i = 0; i < L.size(); i++) { ... L.get(i) ... }` walks the list from the front every single iteration, and also recomputes `size()` every iteration. That innocent-looking loop is quadratic.

7. **Getting the prepend idiom backwards.** `L = new IntList(5, L);` puts 5 at the **front**. If you want `5 -> 10 -> 15` you must create it in the order 15, 10, 5. Many students write the lines in list order and get a reversed list.

8. **Thinking a non-destructive method can just edit `L.first`.** Writing `L.first += x;` inside `incrementRecursiveNonDestructive` would mutate the caller's list, which is exactly what "non-destructive" forbids. Non-destructive means every node in the result is created with `new`.

9. **Forgetting the `null` base case in a static list helper.** Without `if (L == null) return null;`, `incrementRecursiveNonDestructive` would try to read `L.first` off `null` and crash at the end of the list. That base case is also what supplies the result list's terminating `null`.

10. **Assuming `L.get(2)` prints the size.** In the lecture's `main` the size and `get(2)` both print `3` purely by coincidence, since the last element happens to be the value 3.

---

## Likely Exam Points

### 1. Box-and-pointer diagrams for list construction

Given a sequence of `IntList` statements, draw the heap or answer what a variable points at.

**Practice.** After the following code, what does `A.get(1)` return, and how many `IntList` objects exist in the heap?

```java
IntList A = new IntList(1, null);
IntList B = new IntList(2, A);
A = new IntList(3, B);
```

**Answer.** Build it up: `A -> [1|X]`. Then `B -> [2|*] -> [1|X]`. Then `new IntList(3, B)` creates a node with `first = 3`, `rest = B`, and reassigns `A` to it, so `A -> [3|*] -> [2|*] -> [1|X]`. `A.get(1)` walks one node and returns `2`. Three `IntList` objects exist; note that `B` still points into the middle of `A`'s list, and the old binding of `A` to node(1) was replaced but node(1) is still reachable as the last node.

### 2. The `this == null` question

This is asked almost verbatim in the textbook, so it is fair game.

**Practice.** Why can't `size()` be written as `if (this == null) return 0; return 1 + rest.size();`?

**Answer.** Because `size()` is an instance method, calling it requires a receiver object: `L.size()`. If `L` is `null`, the JVM throws a `NullPointerException` at the call, before the method body executes, so the `this == null` test never runs. `this` is never `null` inside a method. The fix is to check `rest == null` (base case is a one-element list) or to use a static helper that takes the list as an argument so the `null` check happens on a parameter.

### 3. Convert recursive to iterative, or iterative to recursive

**Practice.** Write `iterativeGet(int i)` for `IntList` without recursion.

**Answer.**
```java
public int iterativeGet(int i) {
    IntList p = this;
    while (i > 0) {
        p = p.rest;
        i -= 1;
    }
    return p.first;
}
```
Each iteration advances one node and decrements the remaining index; when `i` hits 0, `p` is on the target node. (Equivalently, a `for` loop running `i` times.)

### 4. Runtime of `get` and `size`

**Practice.** For an `IntList` of length N, what is the runtime of `L.get(N - 1)`? What about `L.get(0)`? Why is this a problem?

**Answer.** `L.get(N - 1)` is linear in N: you must follow N-1 `rest` pointers because there is no way to jump directly to a position. `L.get(0)` is constant time. The problem is that code which indexes into a list in a loop becomes quadratic; the textbook promises "an alternate way to implement a list that will avoid this problem in a future lecture" (array-based lists, where indexing is constant time).

### 5. Destructive vs. non-destructive

**Practice.** After running the lecture's `main`, what does `L` print as, and what does `L2` print as? Justify.

**Answer.** `L` is still `5 -> 10 -> 3` and `L2` is `15 -> 20 -> 13`. `incrementRecursiveNonDestructive` only *reads* `L.first` and `L.rest`; every node in the returned list is freshly built with `new IntList(...)`. No field of any node of `L` is ever assigned to, so `L` is unchanged. Six `IntList` objects exist in total, and they share no nodes.

### 6. Write a recursive list method from scratch

**Practice.** Write a static, non-destructive method `IntList square(IntList L)` returning a new list with each element squared, leaving `L` unmodified.

**Answer.**
```java
public static IntList square(IntList L) {
    if (L == null) {
        return null;
    }
    return new IntList(L.first * L.first, square(L.rest));
}
```
The `null` base case both terminates the recursion and supplies the new list's terminating `null`. Using `new` for every node guarantees non-destructiveness.

### 7. Reading a trace / counting recursive calls

**Practice.** How many times is `size()` invoked (counting the initial call) when you run `L.size()` on a list of length N? How deep does the call stack get?

**Answer.** N invocations and N frames deep at the deepest point: one per node, with the last node hitting the base case. `iterativeSize` by contrast uses a single frame and one extra pointer variable regardless of N.

### 8. Spot the bug

**Practice.** What is wrong with this `iterativeSize`?

```java
public int iterativeSize() {
    IntList p = this;
    int totalSize = 0;
    while (p.rest != null) {
        totalSize += 1;
        p = p.rest;
    }
    return totalSize;
}
```

**Answer.** It undercounts by one. The loop stops when `p` is the last node, so the last node is never counted; on `5 -> 10 -> 3` it returns 2. The condition must be `p != null`. (It would also throw a `NullPointerException` if `this` could somehow be `null`, but as established, it cannot.)

---

## Summary

- `IntList` is a minimal linked list: `public int first` plus `public IntList rest`, with a two-argument constructor. It is legal for a class to have a field of its own type because object variables hold **references**, not objects.
- A list is defined recursively: "a first item, plus the rest of the list." `rest == null` marks the end.
- Building lists by hand is painful: appending gives `.rest.rest.rest` chains; prepending (`L = new IntList(5, L);`) is terser but forces you to build the list back to front. This ugliness motivates adding helper methods.
- `size()` is recursive with base case `rest == null` returning 1. You **cannot** use `this == null` as a base case, because calling a method on `null` throws a `NullPointerException` before the body runs.
- Consequently there is no empty `IntList`; `size()` never returns 0.
- `iterativeSize()` uses a local pointer (`IntList p = this;`, conventionally named `p`) and loops while `p != null`, doing `p = p.rest`. A local pointer is required because `this` cannot be reassigned in Java.
- `p = p.rest` moves a pointer and changes nothing; `p.rest = q` mutates the heap. Keep them straight.
- `get(int i)` is recursive: base case `i == 0` returns `first`, otherwise "my ith item is my rest's (i-1)th item." Zero-indexed. Invalid indices are unspecified and generally throw `NullPointerException`.
- `get` is **linear time**, so linked lists are bad at random access. A future lecture introduces an array-based alternative with constant-time indexing.
- `IntListTools.incrementRecursiveNonDestructive(L, x)` is **static** (so `L == null` is a safe base case) and **non-destructive**: it allocates a new node for every element and never writes to `L`. On `5 -> 10 -> 3` with `x = 10` it returns a separate list `15 -> 20 -> 13`, leaving `L` intact.
- The destructive/non-destructive distinction introduced here is the main thread picked up in Lists 2.
