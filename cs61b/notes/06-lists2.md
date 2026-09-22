<!-- Wed, Sep 9, 2026 | sources: slides + code (no transcript available) -->
# Lecture 6: Lists 2: SLLists

## Overview

This lecture takes the "naked" recursive `IntList` from Lecture 4/5 and wraps it in a friendlier container class called `SLList` (Singly Linked List), then walks through six incremental improvements that make the class easier to use, safer, and faster. The core idea is that a raw recursive list forces the *user* to understand references and recursion (the user holds a pointer directly into the data structure and must write `L = new IntList(5, L)` to add to the front), whereas an `SLList` acts as a middleman: the user just calls `addFirst`, `getFirst`, `addLast`, and `size`. Along the way we rename `IntList` to `IntNode` and strip its methods (Improvement #1), introduce the `SLList` wrapper with a constructor (#2), use the `private` keyword to stop users from corrupting the internal node chain (#3), nest `IntNode` inside `SLList` (#4), cache the size in an instance variable so `size()` is fast instead of linear time (#5), and finally add a **sentinel node** so that the empty list looks structurally the same as every other list, which eliminates the `NullPointerException` special case in `addLast` (#6). The lecture closes with the notion of an **invariant**: a condition guaranteed true during execution, which is what lets us write simple code that does not check for nulls everywhere.

---

## Key Concepts

### 1. Why "naked" recursive lists are painful

Recall the Lecture 4/5 `IntList`:

```java
public class IntList {
    public int first;
    public IntList rest;

    public IntList(int f, IntList r) {
        first = f;
        rest = r;
    }
    ...
}
```

The slides describe this as "functional, but hard to use." Two specific burdens on the user:

- The user must know Java references very well.
- The user must be able to think recursively.

Compare the two usage patterns side by side (directly from the slides):

```java
SLList L = new SLList(15);      |   IntList L = new IntList(15, null);
L.addFirst(10);                 |   L = new IntList(10, L);
L.addFirst(5);                  |   L = new IntList(5, L);
int x = L.getFirst();           |   int x = L.first;
```

The `IntList` user has to reassign `L` themselves and construct nodes by hand. The `SLList` user just calls methods. The slides raise and answer the obvious objection: *why not just add an `addFirst` method to `IntList`?* Answer: "Turns out there is no efficient way to do this. Try it out and you'll see it's hard (and inefficient)." The problem is that an `IntList` variable *is* the first node, so adding to the front means the object the user is holding has to become a different object, which you cannot do; you would have to shuffle all the items down by one.

### 2. The middleman idea (the central intuition)

The slides show a box-and-pointer contrast. With `IntList`, it is "natural for [the] IntList user to have variables that point to the middle of the IntList" (the diagram shows `L1` pointing at the node containing 5 and `L2` pointing at the node containing 10, i.e. two user variables aimed into the same chain). With `SLList`, there is exactly one user variable, `L1`, pointing at an `SLList` object; the `SLList` object holds `first`, which points to the chain of `IntNode`s.

The payoff of having that extra layer of indirection:

- The user never sees `IntNode` at all, so the interface is simpler.
- The wrapper can store **meta information about the entire list**, e.g. the `size`. A naked recursive list has nowhere to put "size of the whole list," because every node is itself a whole list.
- We can change the internals freely (this is what makes adding a sentinel node possible later without breaking any user code).

### 3. Rebranding: `IntList` becomes `IntNode`

```java
public class IntNode {
    public int item;
    public IntNode next;

    public IntNode(int i, IntNode n) {
        item = i;
        next = n;
    }
}
```

Two things changed: the names (`first`/`rest` become `item`/`next`) and the methods (all removed). The slides say "IntNode is now dumb, has no methods. We will reintroduce functionality in the coming slides." The rename reflects a change in mental model: `first`/`rest` suggests "this object is a list," whereas `item`/`next` suggests "this object is one link in a chain." The slides are honest that this alone is "not much of an improvement."

### 4. Bureaucracy: building `SLList`

```java
/** An SLList is a list of integers, which hides the terrible truth
  * of the nakedness within. */
public class SLList {
    public IntNode first;

    public SLList(int x) {
        first = new IntNode(x, null);
    }

    public static void main() {
        /** Creates a list of one integer, namely 10 */
        SLList L = new SLList(10);
    }
}
```

`SLList` has one instance variable, `first`, which points at the first `IntNode`. Immediate small win: `new SLList(10)` versus `new IntList(10, null)` - the user does not have to know to pass `null`.

### 5. `addFirst` and `getFirst`

```java
/** Adds x to the front of the list. */
public void addFirst(int x) {
    first = new IntNode(x, first);
}

/** Returns the first item in the list. */
public int getFirst() {
    return first.item;
}
```

Note the slide's annotation on `addFirst`: "This is how we added to the front of an IntList in lecture 4." The mechanics are identical; the difference is *who does it*. The user used to write `L = new IntList(x, L)`; now the class writes `first = new IntNode(x, first)` on the user's behalf. Reading right to left: make a new node whose `next` is the current front, then point `first` at that new node.

### 6. Access control: `public` vs `private`

With `public IntNode first`, a mischievous or confused user can do:

```java
SLList L = new SLList(15);
L.addFirst(10);
L.first.next.next = L.first.next;   // creates a loop!
```

The slide diagram shows this making the node containing 15 point back at itself (a cycle), which would make `addLast` or `size` loop forever. The fix:

```java
public class SLList {
    private IntNode first;
    ...
}
```

and the compiler now rejects the abuse:

```
$ javac SLListUser.java
SLListUser.java:8: error: first has private access in SLList
        L.first.next.next = L.first.next;
```

**Why restrict access?** The slides give two reasons and one analogy:

- Hide implementation details from users: less for the user to understand.
- Safe for *you* to change private methods / implementation later.
- Car analogy: public = pedals, steering wheel; private = fuel line, rotary valve.

And an important caveat: "Despite the term 'access control': Nothing to do with protection against hackers, spies, LLMs, and other evil entities." `private` is a tool for managing complexity, not a security mechanism.

### 7. Nested classes

Since `IntNode` only exists to serve `SLList`, put it inside:

```java
public class SLList {
    public class IntNode {
        public int item;
        public IntNode next;
        public IntNode(int i, IntNode n) {
            item = i;
            next = n;
        }
    }

    private IntNode first;
    public SLList(int x) {
        first = new IntNode(x, null);
    }
    ...
}
```

Conventions from the slides: instance variables, constructors, and methods of `SLList` typically go *below* the nested class definition. Nested classes are useful "when a class doesn't stand on its own and is obviously subordinate to another class," and you should "make the nested class private if other classes should never use the nested class."

The lecture's opinion: `IntNode` probably should be private nested, because it is hard to imagine other classes needing to manipulate `IntNode`s. But if there were some hypothetical method like `public IntNode getFrontNode()`, then `IntNode` would have to be public (you cannot expose a private type in a public signature). The lecture code file does exactly this:

```java
private class IntNode {
    public int item;
    public IntNode next;
    ...
}
```

with the comment "nobody else would ever use IntNode except for this class (SLList), so let's make it private."

**Static nested classes (marked "(Extra)" on the slide):** if the nested class never uses any instance variables or methods of the outer class, declare it `static`.

```java
public class SLList {
    private static class IntNode {
        public int item;
        public IntNode next;
        public IntNode(int i, IntNode n) {
            item = i;
            next = n;
        }
    }
    ...
}
```

- Static classes cannot access the outer class's instance variables or methods.
- Results in a minor savings of memory.
- Analogy: static *methods* have no way to access "my" instance variables; static *classes* cannot access "my" outer class's instance variables.
- "Unimportant note: For private nested classes, access modifiers are irrelevant."

### 8. `addLast`: iteration to the end

```java
/** Adds an item to the end of the list. */
public void addLast(int x) {
    IntNode p = first;

    /* Move p until it reaches the end of the list. */
    while (p.next != null) {
        p = p.next;
    }

    p.next = new IntNode(x, null);
}
```

The pointer `p` is a local variable used to "inch along" the list (the lecture code's phrasing). The loop condition is `p.next != null`, not `p != null`: we want to *stop on* the last node, not walk past it, because we need to modify that last node's `next` field.

The slides also warn, after running a single `main` test: "Running this code doesn't actually prove that our addLast works! It's just a single test." (Relevant to Project 1A, which is entirely about writing tests.)

### 9. `size`: recursion over a non-recursive class

Here is the conceptual hurdle the slides call out explicitly:

> Writing a recursive `size` method is tricky, because `SLList` itself is not recursive. The `size` method doesn't take in any arguments, so calling `size` recursively is strange. What is the base case? How do you call `size` recursively to get closer to the base case?

The `SLList` object is a single thing; there is no "rest of the SLList" to recurse on. The **solution: write a private static helper method that takes an extra argument to help with recursion.**

```java
/** Returns the size of the list that starts at IntNode p. */
private static int size(IntNode p) {
    if (p.next == null) {
        return 1;
    }
    return 1 + size(p.next);
}

public int size() {
    return size(first);
}
```

Points to notice:

- The helper can be `static` because "we don't need to reference `first`" - it works entirely off its parameter.
- These are two methods named `size` with different parameter lists (overloading). Java picks based on arguments.
- The general recipe from the slides: **to implement a recursive method in a class that is not itself recursive, create a private recursive helper method, and have the public method call it.** The lecture code phrases it as "when you want to recurse over a non-recursive thing, use a private helper method with a variable to hold state, a.k.a. position."
- The commented-out version in the lecture code uses a cleaner base case:
  ```java
  public static int size(IntNode p) {
      if (p == null) {
          return 0;
      }
      return 1 + size(p.next);
  }
  ```
  Using `p == null → 0` handles the empty list correctly, whereas `p.next == null → 1` crashes on an empty list (`first` is `null`, so `p.next` throws). (extra context) This is a general lesson about linked-list base cases: testing the node itself for null is usually more robust than testing `node.next`.

### 10. `size` efficiency and caching (Improvement #5)

The attendance question: if `size` takes 2 seconds on a list of 1,000 items, how long on 1,000,000? The recursive `size` visits every node exactly once, so the running time grows in direct proportion to the list length. A 1,000x longer list takes about 1,000x longer: **2,000 seconds** (answer c).

The fix is to stop recomputing and instead remember:

```java
public class SLList {
    private IntNode first;
    private int size;

    public SLList(int x) {
        first = new IntNode(x, null);
        size = 1;
    }

    public void addFirst(int x) {
        first = new IntNode(x, first);
        size += 1;
    }

    /** Returns the first item in the list. */
    public int getFirst() {
        return first.item;     // no modification needed: doesn't change the size
    }

    public void addLast(int x) {
        size += 1;
        IntNode p = first;
        while (p.next != null) {
            p = p.next;
        }
        p.next = new IntNode(x, null);
    }

    public int size() {
        return size;
    }
}
```

The slides' framing: "Instead of re-calculating size on demand every time `size` is called, we'll keep track of the current size in a private variable. Then, we'll update the size variable every time the list is changed. This variable is **redundant** (we could calculate the size from the list), but will save us time."

- **Caching** is defined as "putting aside data to speed up retrieval."
- **TANSTAAFL**: "There ain't no such thing as a free lunch." We pay a tiny cost on every `add` call in exchange for a fast `size()`. "But spreading the work over each add call is a net win in almost any circumstance."
- Note that `size()` and the instance variable `size` can share a name in Java; methods and fields live in different namespaces. `return size;` returns the field.
- The box-and-pointer diagram for this stage shows the `SLList` box containing `addFirst()`, `getFirst()`, `addLast()`, `size()`, plus a `size` field holding `3` and a `first` field pointing at the chain 5 → 10 → 15. Caption: "SLList class acts as a middle man between user and the naked recursive data structure. Allows us to store meta information about entire list, e.g. size."

### 11. The empty list and the `addLast` bug

Add a second (no-argument) constructor:

```java
public SLList(int x) {
    first = new IntNode(x, null);
    size = 1;
}

/** Creates an empty SLList. */
public SLList() {
    first = null;
    size = 0;
}
```

Benefits of `SLList` over `IntList` accumulated so far, per the slides:

- Faster `size()` method than would have been convenient for `IntList`.
- The user of an `SLList` never sees the `IntList`/`IntNode` class: simpler to use, more efficient `addFirst`, and avoids errors (or malfeasance, i.e. the loop-creating abuse above).
- Easy to represent the empty list: set `first` to `null`.

But: "We'll see there is a very subtle bug in the code. It crashes when you call `addLast` on an empty list."

```java
SLList L = new SLList();
L.addLast(20);   // program crashes!
```

The `NullPointerException` traceback points at `while (p.next != null)`. Reason: on an empty list `first` is `null`, so `p` is `null`, so `p.next` dereferences null.

**One possible fix, a special case:**

```java
public void addLast(int x) {
    size += 1;

    if (first == null) {
        first = new IntNode(x, null);
        return;
    }

    IntNode p = first;
    while (p.next != null) {
        p = p.next;
    }

    p.next = new IntNode(x, null);
}
```

"But there are other ways..."

### 12. Keep code simple, and the sentinel node (Improvement #6b)

The lecture's programming-practice tip:

> As a human programmer, you only have so much working memory. You want to restrict the amount of complexity in your life! Simple code is (usually) good code. Special cases are not 'simple'.

And the diagnosis:

> **The fundamental problem:** The empty list has a null `first`. Can't access `first.next`!
> Our fix is a bit ugly: requires a special case. More complex data structures will have many more special cases (gross!!)
> **How can we avoid special cases?** Make all SLLists (even empty) the "same".

The sentinel node does exactly that: "Create a special node that is always there!"

- The empty list is **just the sentinel node**.
- A list with 3 numbers has a sentinel node **and** 3 nodes that contain real data.
- The reference is renamed from `first` to `sentinel`.
- `sentinel` is never null; it always points to the sentinel node.
- The sentinel node's `item` "needs to be some integer, but doesn't matter what value we pick." The slides use `63`; the lecture code uses `-420`. The slides draw it as `??`.
- Constructors and methods all had to be fixed to be compatible.

The rewritten class (slides version):

```java
public class SLList {
    /** The first item (if it exists) is at sentinel.next. */
    private IntNode sentinel;
    private int size;

    public SLList(int x) {
        sentinel = new IntNode(63, null);
        sentinel.next = new IntNode(x, null);
        size = 1;
    }

    public SLList() {
        sentinel = new IntNode(63, null);
        size = 0;
    }

    /** Adds x to the front of the list. */
    public void addFirst(int x) {
        sentinel.next = new IntNode(x, sentinel.next);
        size += 1;
    }

    /** Returns the first item in the list. */
    public int getFirst() {
        return sentinel.next.item;
    }

    /** Adds x to the end of the list. */
    public void addLast(int x) {
        size += 1;
        IntNode p = sentinel;

        /** Move p until it reaches the end of the list. */
        while (p.next != null) {
            p = p.next;
        }
        p.next = new IntNode(x, null);
    }

    public int size() {
        return size;
    }
}
```

Every change flows from the single documented rule **"the first item (if it exists) is at `sentinel.next`"**:

- Single-element construction is now *two* nodes: the sentinel, then the node with the value.
- `addFirst` reassigns `sentinel.next` rather than `sentinel` itself. `sentinel` never changes after construction.
- `getFirst` returns `sentinel.next.item`. The slide warns: "If we returned `sentinel.item`, we would get the placeholder 63, not the true first item of the list."
- `addLast` starts `p` at `sentinel` instead of at `first`. "Sentinel always exists, so the `addLast` bug doesn't apply anymore." The `if (sentinel == null)` special case shown crossed-off on the slide is unnecessary "since it is never null."

Test:

```java
SLList L = new SLList();
L.addLast(20);
IO.println(L.size()); // should print out 1
```

### 13. Invariants

> An **invariant** is a condition that is guaranteed to be true during code execution (assuming there are no bugs in your code).

An `SLList` with a sentinel node has at least these invariants:

1. The `sentinel` reference always points to the sentinel node.
2. The first node (if it exists) is always at `sentinel.next`.
3. The `size` variable is always the total number of items that have been added.

Why invariants matter (both directions matter equally):

- You **can assume** they are true to simplify code. This is precisely why `addLast` needs no null check: invariant 1 guarantees `p = sentinel` is a real node.
- You **must ensure** that methods preserve invariants. That is why every mutating method does `size += 1`, and why `addFirst` touches `sentinel.next` instead of `sentinel`. A method that broke invariant 3 would make `size()` silently wrong forever after.

### 14. Reflections: what is still wrong with `SLList`

**The Good:**
- Doesn't have a fixed length.
- Implementation was simple-ish.
- Usage is very simple.

**The Bad:**
- Slow to get to the end of the list (`addLast` walks the whole thing).
- Only stores `int`s.
- Slow to access items in the middle of the list.

These three complaints motivate the next lectures (and the slides note "You'll know what you need to know at the end of Friday's lecture" for Project 1B).

---

## Definitions

- **`IntNode`**: A "dumb" class with no methods holding an `int item` and a reference `IntNode next` to the next node. The rebranded, method-stripped version of `IntList`. Represents one link in a chain rather than a whole list.
- **`SLList`**: Singly Linked List. A wrapper/container class with a reference into a chain of `IntNode`s plus (after Improvement #5) a cached `size`. Provides `addFirst`, `getFirst`, `addLast`, `size` and hides the nodes entirely.
- **Naked linked list**: A linked list exposed directly to the user, so that a user variable points at a node and the user must manipulate references and recurse themselves (e.g. `IntList`).
- **`private`**: Keyword used "to prevent code in other classes from using members (or constructors) of a class." Enforced by the compiler; a complexity-management tool, not a security feature.
- **Access control**: The use of `public`/`private` to hide implementation details, so there is less for a user to understand and so the implementer is free to change the internals.
- **Nested class**: A class defined inside another class. Appropriate when the inner class "doesn't stand on its own and is obviously subordinate to another class."
- **Static nested class**: A nested class declared `static`, permitted when the nested class never uses any instance variables or methods of the outer class. It cannot access the outer class's instance variables or methods, and saves a small amount of memory.
- **Private recursive helper method**: A private (often static) method taking an extra argument (typically the current node/position) used to recurse over a data structure whose wrapper class is not itself recursive. The public method calls the helper, passing the starting node.
- **Caching**: "Putting aside data to speed up retrieval." Here: storing the list's size in an instance variable rather than recomputing it. The stored data is redundant but fast to read.
- **TANSTAAFL**: "There ain't no such thing as a free lunch." Caching buys fast reads at the cost of extra bookkeeping on every write.
- **Sentinel node**: A special node that is always present in the list, whose `item` is a meaningless placeholder, and which sits before all real items. It makes the empty list structurally identical to non-empty lists, eliminating null special cases.
- **Invariant**: "A condition that is guaranteed to be true during code execution (assuming there are no bugs in your code)."

---

## Worked Examples

### Example 1: `SLList L = new SLList(15); L.addFirst(10); L.addFirst(5); int x = L.getFirst();`

Using the pre-sentinel version of the class. Reasoning in box-and-pointer terms:

```java
SLList L = new SLList(15);
```
1. `new SLList(15)` allocates an `SLList` object. Its `first` field starts as `null` (default for reference types).
2. The constructor body runs `first = new IntNode(15, null)`: a node box is created with `item = 15`, `next = null`, and `first` is pointed at it.
3. `L` now holds a reference to the `SLList` box. Picture: `L → [SLList | first]` and `first → [15 | null]`.

```java
L.addFirst(10);
```
4. `first = new IntNode(10, first)`. Java evaluates the right-hand side first, so the argument `first` is still the reference to the node containing 15. A new node `[10 | →15]` is created.
5. *Then* the assignment happens: `first` is repointed to the new node. Chain: `first → [10] → [15] → null`.
6. Crucially the node containing 15 was never modified. Adding to the front is cheap because only one field (`first`) changes.

```java
L.addFirst(5);
```
7. Same thing again: `first → [5] → [10] → [15] → null`.

```java
int x = L.getFirst();
```
8. `getFirst` returns `first.item`, i.e. follow `first` to the node `[5]` and read its `item`: `5`. The slide's version prints `5`.

Contrast with the equivalent `IntList` code: `L = new IntList(5, L)` does the *same* pointer surgery, but the user's own variable `L` is the thing being reassigned. With `SLList`, `L` never changes; only a field inside the object `L` refers to changes. That is the whole difference in a sentence.

### Example 2: `addLast` on a 3-element list, traced

```java
SLList L = new SLList(15);
L.addFirst(10);
L.addFirst(5);
L.addLast(20);
```

Starting state: `first → [5] → [10] → [15] → null`.

```java
public void addLast(int x) {      // x = 20
    IntNode p = first;            // p → [5]
    while (p.next != null) {      // [5].next is [10], not null
        p = p.next;               // p → [10]
    }                             // [10].next is [15], not null → p → [15]
                                  // [15].next is null → loop exits, p → [15]
    p.next = new IntNode(x, null);
}
```

Iteration-by-iteration:

| Check | `p` before | `p.next` | Enter body? | `p` after |
|---|---|---|---|---|
| 1 | `[5]` | `[10]` | yes | `[10]` |
| 2 | `[10]` | `[15]` | yes | `[15]` |
| 3 | `[15]` | `null` | no | `[15]` |

After the loop, `p` points at the last node. `p.next = new IntNode(20, null)` mutates that node's `next` field, so the chain becomes `[5] → [10] → [15] → [20] → null`.

Two things worth internalizing: (a) `p = p.next` moves the local pointer and does **not** modify any node; (b) `p.next = ...` modifies a node and does **not** move `p`. Reassigning `p` itself at the end (e.g. `p = new IntNode(x, null)`) would do nothing to the list, since `p` is just a local variable.

### Example 3: `size(IntNode p)` recursion on a 3-element list

```java
private static int size(IntNode p) {
    if (p.next == null) {
        return 1;
    }
    return 1 + size(p.next);
}

public int size() {
    return size(first);
}
```

Call `L.size()` on `[5] → [10] → [15] → null`:

1. `size()` calls `size(first)` = `size([5])`.
2. `size([5])`: `[5].next` is `[10]`, not null, so return `1 + size([10])`. This call is suspended.
3. `size([10])`: `[10].next` is `[15]`, not null, so return `1 + size([15])`.
4. `size([15])`: `[15].next` is `null`, base case, return `1`.
5. Unwinding: `size([10])` returns `1 + 1 = 2`; `size([5])` returns `1 + 2 = 3`; `size()` returns `3`.

Each node is visited exactly once, one frame per node on the call stack. Hence linear time, which is the point of the attendance question. (extra context) It is also linear *stack space*, so a long enough list can overflow the stack.

The lecture code's commented alternative with base case `p == null → 0` traces the same way but adds one extra call at the end: `size([15])` returns `1 + size(null)` = `1 + 0` = `1`. It has the advantage of working on the empty list.

### Example 4: the `addLast` crash on an empty list, step by step

```java
SLList L = new SLList();   // first = null, size = 0
L.addLast(20);             // program crashes!
```

Inside `addLast`:

1. `size += 1` succeeds, so `size` is now `1`. (Note: a bug has already occurred conceptually, since the list still has 0 items.)
2. `IntNode p = first;` copies the value `null` into `p`.
3. `while (p.next != null)` evaluates `p.next`, which requires following `p` to an object. There is no object. Java throws a `NullPointerException`, and the traceback identifies this line.

Sentinel version, same call:

```java
public SLList() {
    sentinel = new IntNode(63, null);   // or -420 in lecture code
    size = 0;
}

public void addLast(int x) {
    size += 1;                          // size = 1
    IntNode p = sentinel;               // p → [63 | null], a real node
    while (p.next != null) {            // [63].next is null → loop never runs
        p = p.next;
    }
    p.next = new IntNode(x, null);      // sentinel.next → [20 | null]
}
```

No crash, no special case, and the result satisfies the invariant "the first item is at `sentinel.next`" automatically. The empty list and the 3-element list go down exactly the same code path; the only difference is how many times the loop runs (zero times versus three).

### Example 5: `addFirst` with a sentinel

```java
public void addFirst(int x) {
    sentinel.next = new IntNode(x, sentinel.next);
    size += 1;
}
```

Starting from `sentinel → [63] → [10] → [15] → null`, call `addFirst(5)`:

1. Right-hand side is evaluated: `sentinel.next` is currently the node `[10]`, so `new IntNode(5, [10])` creates `[5] → [10]`.
2. Left-hand side assignment: the sentinel node's `next` field is repointed to `[5]`.
3. Result: `sentinel → [63] → [5] → [10] → [15] → null`.
4. `size += 1`, preserving invariant 3.

Compare to the pre-sentinel `first = new IntNode(x, first)`. The pattern is identical; we simply substituted `sentinel.next` for `first` everywhere, because `sentinel.next` is now "where the first item lives." Note that the `sentinel` variable itself is untouched, as invariant 1 requires.

### Example 6: the lecture code's `main`

```java
static void main() {
    SLList L = new SLList();

    L.addLast(15);
    L.addFirst(5);
    IO.println(L.size()); // prints 2
}
```

1. `new SLList()`: `size = 0`, `sentinel = new IntNode(-420, null)`. Chain: `sentinel → [-420 | null]`.
2. `addLast(15)`: `size = 1`; `p` starts at the sentinel; `sentinel.next` is null so the loop body never runs; `sentinel.next = [15 | null]`. Chain: `[-420] → [15] → null`.
3. `addFirst(5)`: `sentinel.next = new IntNode(5, [15])`, so chain becomes `[-420] → [5] → [15] → null`; `size = 2`.
4. `size()` returns the cached field: `2`. No traversal happened.

### Example 7: the abuse that `private` prevents

```java
SLList L = new SLList(15);
L.addFirst(10);
L.first.next.next = L.first.next;
```

With `public IntNode first`: `L.first` is `[10]`, `L.first.next` is `[15]`. The statement sets `[15].next = [15]`, a self-loop. Now `addLast` would spin forever (`p.next` is never null), and the recursive `size` would recurse until the stack overflows. With `private IntNode first`, this does not compile at all: `error: first has private access in SLList`. The class's invariants are safe because only code inside `SLList` can touch the nodes.

---

## Common Pitfalls

1. **Writing `while (p != null)` in `addLast`.** The loop would run one step too far and `p` would end up `null`; then `p.next = ...` throws a `NullPointerException`. You need to stop *on* the last node because you must modify its `next` field. Use `while (p.next != null)`.

2. **Assigning to `p` instead of to `p.next`.** `p = new IntNode(x, null)` at the end of `addLast` changes only the local variable and leaves the list untouched. Only field assignments (`p.next = ...`) mutate the structure.

3. **Forgetting that `sentinel` itself never gets reassigned.** After construction, every operation works through `sentinel.next` and beyond. Writing `sentinel = new IntNode(x, sentinel)` in `addFirst` would break invariants 1 and 2, and the sentinel would become a real data node.

4. **Returning `sentinel.item` from `getFirst`.** The slides call this out explicitly: you would get the meaningless placeholder (63, or -420 in the lecture code), not the first real item. The first item is at `sentinel.next`.

5. **Forgetting `size += 1` in some mutating method.** The cached size is redundant data, so any method that changes the list *must* update it or invariant 3 silently breaks and `size()` lies forever afterward. (In the buggy pre-sentinel `addLast`, `size += 1` ran even though the crash meant no item was added, so the size was wrong *and* the program crashed.)

6. **Using base case `p.next == null` when the empty list is possible.** `size(first)` on an empty list passes `null` and immediately throws on `p.next`. The `p == null → 0` base case is safer. (With a sentinel, `size(sentinel.next)` on an empty list also passes `null`, so this matters.)

7. **Trying to write `size()` recursively without a helper.** `size()` takes no arguments and `SLList` is not recursive, so there is no smaller `SLList` to hand to a recursive call. You need the private helper taking an `IntNode`.

8. **Off-by-one with the sentinel when counting or traversing.** A list with 3 items has 4 nodes. Do not count the sentinel as an item.

9. **Assuming `private` provides security.** It stops other *classes* from compiling against your fields. It is about managing complexity, not defending against adversaries.

10. **Declaring a nested class `static` when it needs the outer object.** `static` nested classes cannot touch the outer class's instance variables or methods. `IntNode` can be static precisely because it only ever uses its own fields.

11. **Exposing a private nested type in a public signature.** A method like `public IntNode getFrontNode()` forces `IntNode` to be public. If you want `IntNode` private, do not leak it out through the public API.

12. **Believing a single passing `main` proves correctness.** From the slides: "Running this code doesn't actually prove that our addLast works! It's just a single test." Test the empty list, the one-element list, and combinations of operations.

---

## Likely Exam Points

### 1. Efficiency of the recursive `size`

**Q.** The recursive `size` takes 2 seconds on a list of 1,000 items. About how long on a list of 1,000,000 items? (a) 0.002 s (b) 2 s (c) 2,000 s (d) 2,000,000 s

**A.** (c) 2,000 seconds. `size` visits each node exactly once, so the time grows in proportion to the list length. The list is 1,000 times longer, so it takes about 1,000 times as long. Answer (b) would be correct only for the *cached* `size()`, which just reads an instance variable and does not depend on the list length at all.

### 2. Why not add `addFirst` to `IntList` directly?

**Q.** Your friend says the whole `SLList` wrapper is unnecessary: just add an `addFirst(int x)` method to `IntList`. Explain why this does not work well.

**A.** An `IntList` variable *is* the front node, so the user's variable must change to point at a new node, and a method cannot reassign the caller's variable. The only way to make an instance method work is to shift every item down by one (copy the current `first` into a new node placed at `rest`, then overwrite `first` with `x`), which is possible but awkward, and more importantly there is no place to store whole-list metadata like a cached size. The slides put it as: "Turns out there is no efficient way to do this. Try it out and you'll see it's hard (and inefficient)."

### 3. Fill in `addLast`

**Q.** Fill in the blanks so that `addLast` works on a sentinel-based `SLList`, including on an empty list.

```java
public void addLast(int x) {
    size += 1;
    IntNode p = ________;
    while (________) {
        p = ________;
    }
    ________ = new IntNode(x, null);
}
```

**A.**
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
Starting at `sentinel` (not `sentinel.next`) is what makes the empty case work: the sentinel always exists, so `p.next` is always a legal dereference. On an empty list the loop body never executes and the new node is attached directly to `sentinel.next`.

### 4. Trace and draw

**Q.** Draw the state (in words) after:
```java
SLList L = new SLList();
L.addLast(3);
L.addFirst(1);
L.addLast(7);
```
What are `L.size()` and `L.getFirst()`?

**A.** Chain: `sentinel → [placeholder] → [1] → [3] → [7] → null`. Four nodes total, three of which hold data. `L.size()` returns `3` (the cached field, incremented once per add). `L.getFirst()` returns `sentinel.next.item` = `1`. Trace: `addLast(3)` gives `[ph] → [3]`; `addFirst(1)` splices `[1]` in right after the sentinel giving `[ph] → [1] → [3]`; `addLast(7)` walks `p` from `[ph]` to `[1]` to `[3]` and attaches `[7]`.

### 5. Write a recursive helper

**Q.** Add a method `public int sum()` to the sentinel-based `SLList` that returns the sum of all items, using recursion.

**A.**
```java
private static int sum(IntNode p) {
    if (p == null) {
        return 0;
    }
    return p.item + sum(p.next);
}

public int sum() {
    return sum(sentinel.next);   // skip the sentinel: its item is a placeholder
}
```
Two exam-relevant points: the public no-argument method delegates to a private recursive helper that carries the position, and the initial call must be `sentinel.next`, not `sentinel`, or you would add the meaningless placeholder value. The `p == null → 0` base case handles the empty list for free.

### 6. Identify the invariant that is broken

**Q.** A student writes this `addFirst` for the sentinel version. Which invariant does it break, and what goes wrong?
```java
public void addFirst(int x) {
    sentinel = new IntNode(x, sentinel);
    size += 1;
}
```

**A.** It breaks invariant 1 ("the `sentinel` reference always points to the sentinel node") and consequently invariant 2 ("the first node is at `sentinel.next`"). After the call, `sentinel` points at a node holding real data `x`, and the old sentinel node has been demoted to an ordinary interior node. `getFirst()` would return the *second* item, and the placeholder value would appear in the middle of the list and be included by any traversal. The correct body is `sentinel.next = new IntNode(x, sentinel.next);`.

### 7. What is an invariant, and how does a sentinel help?

**Q.** Define "invariant" and state two invariants of a sentinel-based `SLList`. Explain how one of them simplifies `addLast`.

**A.** An invariant is a condition guaranteed to be true during code execution, assuming the code is bug-free. Two invariants: `sentinel` always points to the sentinel node (so it is never null), and the first real item, if any, is at `sentinel.next`. The first one simplifies `addLast`: because `sentinel` is guaranteed non-null, `IntNode p = sentinel; while (p.next != null)` can never throw a `NullPointerException`, so the `if (first == null) { ... return; }` special case disappears. Invariants cut both ways: methods may assume them, but every method must also leave them true.

### 8. Caching tradeoff

**Q.** What is the cost of caching `size` as an instance variable? Is it worth it?

**A.** The cost is bookkeeping: every method that changes the list must update `size`, which is extra code and an extra opportunity for bugs (if any one method forgets, `size()` is wrong forever). There is also a tiny per-add time cost and a few bytes of memory for redundant data. TANSTAAFL. It is essentially always worth it: "spreading the work over each add call is a net win in almost any circumstance," since it turns a linear-time `size()` into a constant-time one.

### 9. `private`, nested, and `static`

**Q.** (a) Why make `IntNode` a private nested class? (b) When may it additionally be declared `static`? (c) What would force it to be public?

**A.** (a) `IntNode` does not stand on its own; it is obviously subordinate to `SLList`, and no other class has any need to manipulate `IntNode`s. Nesting keeps the two in one file, and `private` prevents outside code from corrupting the node chain (e.g. `L.first.next.next = L.first.next`, which creates a cycle). (b) It may be `static` because it never uses any of `SLList`'s instance variables or methods; a static nested class cannot access them, and using `static` saves a small amount of memory. (c) A public method whose signature mentions `IntNode`, such as the slides' hypothetical `public IntNode getFrontNode()`, would require `IntNode` to be public.

### 10. Spot the crash

**Q.** Given the pre-sentinel version where `SLList()` sets `first = null`, which of the following crash, and on which line?
```java
SLList A = new SLList();  A.addFirst(1);  A.addLast(2);
SLList B = new SLList();  B.addLast(1);
```

**A.** `A` is fine: `addFirst(1)` sets `first` to a real node, so by the time `addLast` runs, `p` is non-null and the loop is safe. `B` crashes inside `addLast` at `while (p.next != null)` with a `NullPointerException`, because `first` (and therefore `p`) is `null`. This is exactly the bug that motivates the sentinel: the failure depends on the order of calls, which is why a single ad hoc `main` test can easily miss it.

---

## Summary

- **Problem with naked recursive lists (`IntList`)**: the user's variable points directly into the data structure, so the user must understand references and recursion, and there is nowhere to store whole-list metadata.
- **`SLList` is a middleman**: it holds a reference into a chain of `IntNode`s and exposes simple methods (`addFirst`, `getFirst`, `addLast`, `size`). The user never sees a node.
- **The six improvements**:
  1. **Rebranding**: `IntList` → `IntNode` (`first`/`rest` → `item`/`next`), methods removed; nodes are now "dumb."
  2. **Bureaucracy**: the `SLList` wrapper class with a `first` field and a constructor, so `new SLList(10)` replaces `new IntList(10, null)`.
  3. **Access control**: `private IntNode first` stops outside code from corrupting the chain (e.g. creating a cycle). Hides implementation details and frees you to change them. Not a security mechanism.
  4. **Nested class**: `IntNode` defined inside `SLList`, ideally `private` (and can be `static`, since it never uses `SLList`'s instance state).
  5. **Caching**: `private int size`, incremented by every mutating method, makes `size()` a single field read instead of a full traversal. Redundant data, updated on write. TANSTAAFL.
  6. **Generalizing with a sentinel node**: a node that is always present, with a placeholder `item` (63 on the slides, -420 in the lecture code), sitting before all real data. The empty list is just the sentinel; an *n*-item list has *n* + 1 nodes.
- **`addLast` pattern**: start `p` at `sentinel`, loop `while (p.next != null) p = p.next;`, then `p.next = new IntNode(x, null)`. Stop *on* the last node, since you must modify its `next` field.
- **Recursion over a non-recursive class**: write a `private static` helper that takes an `IntNode` parameter to carry the position, and have the public no-argument method call it (`return size(first);`). Prefer the `p == null → 0` base case so the empty list works.
- **The empty-list bug**: with `first = null`, `addLast` dereferences `p.next` on `null` and throws a `NullPointerException`. You can patch it with an `if (first == null)` special case, but special cases are not simple, and complex data structures accumulate many of them. The sentinel removes the special case entirely by making all lists structurally the same.
- **Invariants** are conditions guaranteed true during execution: `sentinel` always points to the sentinel node; the first item (if any) is at `sentinel.next`; `size` is always the number of items added. You may assume them to simplify code, and you must preserve them in every method.
- **Still unsatisfying about `SLList`**: slow to reach the end of the list, slow to reach the middle, and it only stores `int`s. These motivate the following lectures.
- **Course logistics**: Project 1A (write tests for `LinkedListDeque.java`) due Friday Sep 11; Project 1B (write `LinkedListDeque.java`) due Wednesday Sep 16. Lab 3 this week gives more practice with recursive lists.
