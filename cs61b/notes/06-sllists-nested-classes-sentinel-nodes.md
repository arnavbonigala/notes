<!-- Wed, Sep 09, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 6: Lists 2: SLLists

## Overview

This lecture takes the `IntList` from Lecture 4/5, a "naked recursive" data structure whose users must understand references and recursion just to add an item to the front, and wraps it in something a normal Java programmer would actually want to use. Through six improvements (rebranding `IntList` to `IntNode`, adding an `SLList` middleman class, making fields `private`, nesting `IntNode` inside `SLList`, caching `size` in an instance variable, and introducing a sentinel node), we end up with a singly linked list class whose public interface is `addFirst`, `getFirst`, `addLast`, and `size`, with all the pointer manipulation hidden behind it. Along the way the lecture introduces several ideas that recur constantly in CS 61B: access control as a communication tool rather than a security tool, nested (and static nested) classes, the pattern of a public non-recursive method delegating to a private recursive helper that takes a node as an argument, caching as a time/space tradeoff, the value of eliminating special cases, and invariants as facts you may rely on while reasoning about code.

---

## Key Concepts

### 1. The problem with "naked recursive" data structures

`IntList` is a *naked recursive* data structure: the recursion is exposed directly to the user. To use it you must write things like `L = new IntList(5, L)` to add to the front, which requires understanding that `L` is a reference, that `new` creates a new object, and that the old list becomes the tail of the new one. Josh's framing in lecture: the recursion is "right there on the sleeve."

Two consequences:

- **Usability.** A novice cannot use `IntList` without deeply understanding references and recursion.
- **Safety.** Because users hold references directly into the list, they can (accidentally or deliberately) hold a pointer to the middle of the list, or wire nodes into cycles.

This is not a claim that recursive lists are bad in general. The lecture explicitly notes that in functional / Scheme-like languages naked recursive lists are perfectly idiomatic; it is Java convention that expects something different.

### 2. The middleman pattern

The core move of the lecture: introduce a second class, `SLList`, that holds a reference to the chain of nodes and exposes only friendly methods.

```
Naked (IntList):    user variable --> node --> node --> node
With SLList:        user variable --> SLList object --> node --> node --> node
                                       (first, size, addFirst(), getFirst(), ...)
```

The `SLList` object sits between the user and the raw nodes. From the box-and-pointer picture in the slides: with `IntList`, variables `L1` and `L2` might point at *different nodes in the same chain* (L1 at the node holding 5, L2 at the node holding 10). With `SLList`, the user's variable points at the `SLList` object, and only the `SLList` object points at the first node. Everything the user does must route through methods.

Two distinct benefits follow, and it is worth keeping them separate:

- **Simplicity/safety**: the user never writes `new IntNode(...)`, never types `null`, and cannot reach into the node chain.
- **Metadata**: the `SLList` object is a natural home for information about the list *as a whole*, such as its size. There is nowhere natural to put that in a bare chain of nodes. (You could cache size in every node, but then every mutation would have to update many variables.)

### 3. Public vs. private is about communication, not security

`private` members can only be accessed by code in the same `.java` file. Making `first` private means an external class writing `L.first.next.next = L.first.next;` fails to *compile* ("first has private access in SLList").

What `private` is for:

- **Hiding implementation details**: less for the user of the class to understand.
- **Freedom to change**: anything private is yours to rewrite at will, because nobody outside depends on it.

What `private` is *not* for: the lecture is emphatic that despite the name "access control," this has nothing to do with protection against hackers, spies, or other malicious parties. Anyone with your source can do what they want, and Java's reflection feature can bypass modifiers anyway (rarely used).

The flip side is the commitment implied by `public`: **when you make a member public, you are effectively promising it will behave exactly as it does now, forever.** Car analogy from lecture: pedals and steering wheel are public (every driver relies on them behaving the same way); the fuel line or rotary valve is private (a gas car and an electric car implement "accelerate" completely differently, and that is fine because nobody outside depends on the mechanism).

### 4. Nested classes

`IntNode` does not stand on its own; it is obviously subordinate to `SLList`. Java lets you declare a class inside another class:

```java
public class SLList {
    private static class IntNode { ... }
    private IntNode first;
    ...
}
```

Guidelines from the lecture:

- Use a nested class when the class is obviously subordinate to another and does not stand alone.
- Make it **private** if no other class should ever need to touch it. `IntNode` qualifies: nothing outside `SLList` needs an `IntNode` reference. (Counterexample given in lecture: if you had some strange method `public IntNode getFrontNode()`, then `IntNode` would need to be public, since its type appears in a public signature.)
- Convention is typically to put nested class definitions at the top and the outer class's variables/constructors/methods below, but this is style, not a rule.

### 5. Static nested classes

If the nested class never uses any instance variable or instance method of the outer class, declare it `static`.

- A static nested class **cannot** access the outer class's instance members. If `IntNode` were static, no method in `IntNode` could refer to `first`, `addFirst`, or `getFirst`.
- The payoff is a small memory savings: a non-static nested class instance keeps a reference back to its enclosing object ("a reference to its boss"), and a static one does not.
- Analogy from the slides: static *methods* had no way to access "my" instance variables; static *classes* cannot access "my" outer class's instance variables. Josh's suggested mental reading of the keyword here: "never looks outwards."
- Rule of thumb: **if you don't use any instance members of the outer class, make the nested class static.** `IntNode` uses only its own `item`, `next`, and constructor, so it qualifies.
- Minor note from the slides: for private nested classes, the inner access modifiers are largely irrelevant.

### 6. Writing recursive methods on a non-recursive class

`SLList` is not itself recursive: an `SLList` has no `SLList` field, so `size()` has no smaller `SLList` to recurse on, and `size()` takes no arguments, so there is no position to advance. This is the standard obstacle.

The standard fix, which recurs throughout 61B:

> **Write a private (often static) recursive helper method that takes a node as an extra argument, and have the public method call it with the starting node.**

The public method "speaks the language of mortals"; the private helper "speaks the secret language of the gods," i.e. operates on the naked recursive structure. The two `size` methods have the same name but different parameter lists, which Java allows: this is **overloading**.

### 7. Caching, and the cost of `size()`

The recursive `size(IntNode p)` walks the entire list. If it takes 2 seconds on a list of 1,000 items, it takes **2,000 seconds** on a list of 1,000,000 items (1,000 times the work). That is the answer to the in-lecture attendance question.

The fix is **caching**: store the size in a `private int size` instance variable, set it in the constructors, and update it (`size += 1`) in every method that changes the length of the list. Then `size()` is just `return size;` and runs in the same time regardless of list length.

Tradeoffs, honestly stated: this costs a tiny bit of memory and makes `addFirst`/`addLast` do a little extra work. **TANSTAAFL** ("there ain't no such thing as a free lunch"). But spreading one increment across each `add` call is a clear net win in essentially any realistic situation. Note the cached `size` is *redundant*: it could always be recomputed by walking the list. Redundancy for speed is exactly what caching is.

Note also that `getFirst()` needs no change, because it does not alter the length of the list.

### 8. The empty list and the `addLast` bug

Adding a no-argument constructor for an empty list seems easy:

```java
public SLList() {
    first = null;
    size = 0;
}
```

But now `addLast` crashes on an empty list. Trace it: `p = first` sets `p = null`, then the loop condition `p.next != null` dereferences `null`, giving a **`NullPointerException`**. The traceback points at the `while (p.next != null)` line.

The lecture is candid that this bug is not at all obvious from reading the code; Josh says he only knows it is there because he has taught it before. That is itself a lesson about why systematic testing matters (foreshadowing Project 1A and a later lecture on testing).

The obvious fix is a special case:

```java
public void addLast(int x) {
    size += 1;
    if (first == null) {
        first = new IntNode(x, null);
        return;
    }
    IntNode p = first;
    while (p.next != null) { p = p.next; }
    p.next = new IntNode(x, null);
}
```

This works, but it is ugly. Special cases consume your limited working memory, and this one is mild only because a list is a simple structure. For trees and structures with multiple links, the number of special cases explodes.

### 9. Sentinel nodes

The fundamental problem is that the empty list has `first == null`, so empty lists are structurally *different* from nonempty ones. Rather than patching every method with a null check, **make all lists, even empty ones, the same.**

Do this by always having one dummy node at the front, the **sentinel node**, "the faithful companion," always there, whose `item` value is never examined.

- The empty list *is* just the sentinel node.
- A three-item list is the sentinel node followed by three real nodes.
- Rename `first` to `sentinel`, because it no longer points at the first item.
- `sentinel` is never `null`; it always points at the sentinel node.
- The sentinel node's `item` must be *some* int (Java will not let you write `??`), but the value is irrelevant. The slides use 63; the lecture code uses -420; the textbook mentions -518273. Any value is fine.
- Every constructor and method must be rewritten to be consistent. Doing this retroactively feels haphazard; designing with sentinels from the start is cleaner.

The payoff: `addLast` starts `p` at the sentinel, which is guaranteed non-null, so the null check disappears entirely.

### 10. Invariants

An **invariant** is a condition guaranteed to be true during code execution, assuming your code has no bugs. For a sentinel-based `SLList`:

- `sentinel` always points to a sentinel node (and is never null).
- The first real item, if it exists, is always at `sentinel.next`.
- `size` is always the total number of items currently in the list.

Invariants are useful in two directions:

- **You may assume them** when writing a method. `addLast` need not worry about nulls at the front because the sentinel invariant guarantees `sentinel != null`.
- **You must preserve them.** Every method, when it finishes, should leave all invariants true. This gives you a concrete checklist and helps avoid bugs (e.g. remembering to update `size`).

Some programmers write invariants out explicitly as comments (the lecture code does exactly that: `/** The first item (if it exists) is at sentinel.next. */`); others hold them implicitly.

---

## Definitions

- **Naked recursive data structure**: a data structure, like `IntList`, whose recursive structure is directly exposed to its users, so that using it correctly requires the user to understand references and recursion.
- **`IntNode`**: the rebranded, method-free version of `IntList`, with fields `item` (the value) and `next` (a reference to the rest of the chain). "Dumb": it has no methods besides its constructor.
- **`SLList`**: "singly linked list"; the middleman class that holds a reference to the chain of `IntNode`s and exposes friendly methods (`addFirst`, `getFirst`, `addLast`, `size`) to the user.
- **Middleman (intermediary) class**: a class that stands between the user and a raw/naked data structure, exposing a safe, simple interface and hiding the underlying representation.
- **`private`**: an access modifier preventing code in other classes from using a member (variable, method, or constructor). Private members are accessible only from code inside the same `.java` file.
- **`public`**: an access modifier making a member usable by any other class; implicitly a promise that the member will behave as it does now, permanently.
- **Access control**: the Java language feature comprising `public`/`private` (and others). Despite the name, it is a signal to other programmers enforced by the compiler, not a security mechanism.
- **Nested class**: a class declared inside another class declaration. Purely an organizational tool; no meaningful effect on performance.
- **Static nested class**: a nested class declared `static`, which therefore cannot access the outer class's instance variables or methods, and which does not store a reference to an enclosing instance (small memory savings).
- **Private recursive helper method**: a private method that takes a node (or other position) as an argument so that recursion is possible, called by a public method that itself takes no such argument.
- **Overloading**: having two methods in the same class with the same name but different parameter lists, e.g. `size()` and `size(IntNode p)`. Legal in Java.
- **Caching**: storing (redundant) data so that it can be retrieved quickly later, instead of recomputing it on demand. Example: the `size` instance variable.
- **TANSTAAFL**: "There Ain't No Such Thing As A Free Lunch"; here, caching buys fast `size()` at the cost of slightly slower adds and slightly more memory.
- **`NullPointerException`**: the runtime error resulting from trying to follow a reference whose value is `null`, e.g. evaluating `p.next` when `p == null`.
- **Sentinel node**: a dummy node that is always present in the list, whose stored value is never used, existing so that every list (including the empty list) has the same structure and special cases are unnecessary.
- **Invariant**: a condition guaranteed to be true during code execution, assuming the code is correct.

---

## Worked Examples

### Example 1: Rebranding, `IntList` to `IntNode`

Before (Lecture 4/5):

```java
public class IntList {
    public int first;
    public IntList rest;

    public IntList(int f, IntList r) {
        first = f;
        rest = r;
    }
    // ... helper methods ...
}
```

After:

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

Two changes, neither of which improves anything by itself: all the helper methods are deleted, and the fields are renamed (`first` to `item`, `rest` to `next`). The renaming matters conceptually: in `IntList`, `first` meant "the value at the front of this list"; in `IntNode`, `item` means "the value at this node" and `next` means "the following node." The class no longer pretends to be a whole list, it is now one link.

### Example 2: The `SLList` class with `addFirst` and `getFirst`

```java
public class SLList {
    public IntNode first;

    public SLList(int x) {
        first = new IntNode(x, null);
    }

    /** Adds x to the front of the list. */
    public void addFirst(int x) {
        first = new IntNode(x, first);
    }

    /** Returns the first item in the list. */
    public int getFirst() {
        return first.item;
    }
}
```

`addFirst` is literally the same idea as `L = new IntList(5, L)` from the previous lecture, but now the reassignment happens to the `SLList`'s own `first` field rather than to the user's variable. Reading `first = new IntNode(x, first)` carefully: the right-hand side is evaluated first, constructing a new node whose `next` is the *current* front node; then that new node's address is stored into `first`. The old front node is not modified at all, it simply gains a new predecessor.

`getFirst` is a one-liner: follow `first` to a node, return that node's `item`.

Usage comparison from the slides:

```java
SLList L = new SLList(15);      |  IntList L = new IntList(15, null);
L.addFirst(10);                 |  L = new IntList(10, L);
L.addFirst(5);                  |  L = new IntList(5, L);
int x = L.getFirst();           |  int x = L.first;
```

The left column mentions `null` zero times and never reassigns `L`. Note the rhetorical question raised and answered in lecture: *why not just add `addFirst` to `IntList`?* Because there is no efficient way to do it; try it and you will find the result is both awkward and inefficient (textbook Exercise 2.2.1).

**Box-and-pointer reasoning for the left column.** `new SLList(15)` creates an `SLList` object with a `first` field pointing at a node `[15 | null]`. `L.addFirst(10)` makes `[10 | *]` whose arrow goes to the 15-node, and redirects `first` to it. `L.addFirst(5)` does the same again. Final picture: `L` -> `SLList{first}` -> `[5]` -> `[10]` -> `[15]` -> null. Crucially, `L` itself never changed value; only fields inside the object it points to changed.

### Example 3: The danger that motivates `private`

```java
SLList L = new SLList(15);
L.addFirst(10);
L.first.next.next = L.first.next;
```

Step by step on the box-and-pointer diagram:

1. After the first two lines, `L.first` is the node holding 10, and `L.first.next` is the node holding 15.
2. `L.first.next.next` is the `next` field of the 15-node (currently `null`).
3. The right-hand side `L.first.next` is the address of the 15-node itself.
4. So we set the 15-node's `next` to point at the 15-node. It points at itself.

The result is a malformed list: `10, 15, 15, 15, ...` forever. Walking it with a `while (p.next != null)` loop never terminates. Nothing in the `SLList` methods would ever produce this state; the user reached past the interface and broke an invariant.

Fix:

```java
public class SLList {
    private IntNode first;
    ...
```

Now the offending line in another file fails to compile:

```
$ javac SLListUser.java
SLListUser.java:8: error: first has private access in SLList
        L.first.next.next = L.first.next;
```

Note that the same line *inside* `SLList.java` (e.g. in a `main` method there) would still compile, since `private` restricts by class/file, not by object.

### Example 4: Nesting `IntNode` inside `SLList`

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

    private IntNode first;

    public SLList(int x) {
        first = new IntNode(x, null);
    }
    ...
}
```

Why each modifier:

- **nested**: `IntNode` exists only to serve `SLList`.
- **private**: no other class needs `IntNode` references, so no other class should be able to name the type.
- **static**: scanning the body of `IntNode`, it uses only `item`, `next`, and its own constructor; it never touches `first`, `addFirst`, or `getFirst`. Since it never looks outward, it does not need a reference to its enclosing `SLList`, and `static` removes that hidden reference.

### Example 5: `addLast`, iteratively

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

How to read this: `p` is a walking pointer. It starts at the front node. The loop condition asks "does `p` have a successor?" If yes, step forward. The loop therefore stops exactly when `p` is the **last** node (the one whose `next` is `null`), not one past the end. Then we attach a fresh node there.

The choice of loop condition matters. `while (p.next != null)` leaves `p` on the last node, which is what we need since we must modify that node's `next` field. `while (p != null)` would walk `p` off the end to `null` and lose the ability to attach anything.

Josh's note after running a quick `main`: printing `getFirst()` after an `addLast` does not actually prove `addLast` works. It is one test, and the class does not even expose a way to see the last element. Later lectures cover proper testing; Project 1A is about writing tests first.

### Example 6: `size`, recursively, via a private helper

First, why the naive attempt fails. You might try:

```java
public int size() {
    // ??? what is the base case?
    // ??? return 1 + first.next.size();  <- first.next is an IntNode, not an SLList
}
```

`SLList` has no `SLList` field, and `size()` takes no arguments, so there is nothing to make smaller on the recursive call. The solution:

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

- The helper is **private** (implementation detail), **static** (it never needs `first` or any other instance member, since the node is passed in), and takes an `IntNode` parameter so that recursion has somewhere to go.
- Base case: if `p` has no successor, `p` is the last node, so the sublist starting at `p` has exactly 1 item.
- Recursive case: the sublist starting at `p` has one more item than the sublist starting at `p.next`.
- The public `size()` kicks it off at the front.

Tracing on `5 -> 10 -> 15`: `size(node5)` needs `1 + size(node10)`, which needs `1 + size(node15)`, which hits the base case and returns 1. Unwinding: 1, then 2, then 3.

The two `size` methods are **overloaded**. Note this version assumes `p` is non-null (it would break on an empty list); the lecture code file contains a commented-out variant with the cleaner base case `if (p == null) return 0;`.

### Example 7: Caching size

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

    public void addLast(int x) {
        size += 1;
        IntNode p = first;
        while (p.next != null) {
            p = p.next;
        }
        p.next = new IntNode(x, null);
    }

    public int getFirst() {     // unchanged: doesn't change the length
        return first.item;
    }

    public int size() {
        return size;
    }
}
```

The discipline: identify every method that changes the number of items and update `size` there. The constructor sets the starting value; `addFirst` and `addLast` increment; `getFirst` does nothing. The recursive helper is deleted entirely (it is no longer needed, though the pattern will return in later work).

Note `size` is used both as a field name and a method name here, which Java permits, because it can tell from context (`size` vs `size()`) which you mean.

### Example 8: The empty-list constructor and the crash

```java
/** Creates an empty SLList. */
public SLList() {
    first = null;
    size = 0;
}
```

```java
SLList L = new SLList();
L.addLast(20);  // program crashes!
```

Trace: `size += 1` runs fine. `IntNode p = first;` sets `p = null`. The loop condition evaluates `p.next`, i.e. asks for the `next` field of *nothing*. There is no object at memory location `null`, so Java throws a `NullPointerException` at the `while (p.next != null)` line.

Interestingly, `addFirst` on an empty list works fine, since `new IntNode(x, first)` with `first == null` correctly produces a one-node list. Only `addLast` breaks, which is exactly why the bug is easy to miss.

### Example 9: The sentinel version (final form, matching the lecture code)

```java
public class SLList {
    private class IntNode {
        public int item;
        public IntNode next;

        public IntNode(int x, IntNode r) {
            item = x;
            next = r;
        }
    }

    /** The first item (if it exists) is at sentinel.next. */
    private IntNode sentinel;
    private int size;

    /** Create an SLList with x in it. */
    public SLList(int x) {
        size = 1;
        sentinel = new IntNode(-420, null);
        sentinel.next = new IntNode(x, null);
    }

    /** Creates an empty list. */
    public SLList() {
        size = 0;
        sentinel = new IntNode(-420, null);
    }

    /** Adds x to the front of the list. */
    public void addFirst(int x) {
        sentinel.next = new IntNode(x, sentinel.next);
        size += 1;
    }

    public int getFirst() {
        return sentinel.next.item;
    }

    public int size() {
        return size;
    }

    /** Adds x to the end of the list. */
    public void addLast(int x) {
        size += 1;
        IntNode p = sentinel;
        while (p.next != null) {
            p = p.next;
        }
        p.next = new IntNode(x, null);
    }
}
```

Walking through each change and *why*:

- **`first` renamed to `sentinel`.** The variable no longer points at the first item, so the old name would lie.
- **Empty constructor** builds the sentinel node instead of setting `null`. The empty list is exactly one node, the sentinel. (The slides use 63 as the placeholder value; the lecture code uses -420. It genuinely does not matter.)
- **One-item constructor** builds the sentinel *and then* hangs the real node off `sentinel.next`. A one-item list is now two nodes.
- **`addFirst`** is `sentinel.next = new IntNode(x, sentinel.next)`, not `sentinel = new IntNode(x, sentinel)`. The second version would be wrong: it would replace the sentinel and destroy the invariant that `sentinel` always points to the sentinel node. As Josh phrases it, we never reassign the sentinel; instead someone "gets in line behind" it, and whoever was first becomes second.
- **`getFirst`** returns `sentinel.next.item`, not `sentinel.item`. The latter would return the placeholder (-420 or 63), not a real item. "Never ask the sentinel for its opinions."
- **`addLast`** starts `p` at `sentinel` rather than `first`, and **the special case is gone**. On an empty list, `p` is the sentinel, `p.next` is `null`, the loop body never runs, and `p.next = new IntNode(x, null)` attaches the first real node. On a nonempty list, `p` walks to the last node as before. One code path handles both.

**Box-and-pointer reasoning for the empty case.** `new SLList()` gives `L` -> `SLList{size=0, sentinel}` -> `[-420 | null]`. After `addLast(20)`: `size` becomes 1, `p` = the sentinel node, the loop does not execute, and the sentinel's `next` is set to a new `[20 | null]`. Picture: `sentinel` -> `[-420]` -> `[20]` -> null. `size()` returns 1.

Note also that the empty list now has no way to fail in `getFirst` gracefully, it would NPE on `sentinel.next.item`, but that is a separate concern the lecture does not address here.

---

## Common Pitfalls

1. **Writing `sentinel = new IntNode(x, sentinel)` in `addFirst`.** This reassigns the sentinel reference, turning the old sentinel into a data node and the new node into the sentinel. It breaks the invariant that `sentinel` points to *the* sentinel node. Always modify `sentinel.next`.

2. **Returning `sentinel.item` from `getFirst`.** Gives you the meaningless placeholder value. The first real item is at `sentinel.next.item`.

3. **Forgetting to update the cached `size` in some method that changes the list.** The cache is only as good as the discipline maintaining it. Every add (and later, every remove) must touch it. This is precisely the kind of thing invariants are a checklist for.

4. **Using `while (p != null)` instead of `while (p.next != null)` in `addLast`.** The first walks `p` off the end to `null`, at which point you have lost the reference to the last node and `p.next = ...` throws an NPE.

5. **Trying to write `size()` recursively without a helper.** There is no smaller `SLList` and no argument to advance. You need the private helper that takes an `IntNode`.

6. **Forgetting the `null` case in the recursive `size(IntNode p)` helper.** The lecture's version uses `if (p.next == null) return 1;`, which assumes `p` is non-null and so breaks on an empty list. The variant `if (p == null) return 0;` handles that.

7. **Thinking `private` protects against malicious actors.** It does not. It is a compiler-enforced signal to other programmers. Reflection can bypass it.

8. **Casually making things `public`.** Public is a permanent promise. Prefer private; you can always widen access later, but narrowing it breaks everyone's code.

9. **Adding `static` to a nested class that does use the outer class's instance members.** It will not compile. Check the body first; `IntNode` qualifies because it only ever uses `item`, `next`, and its own constructor.

10. **Believing one passing `main` means the method works.** Running `addLast(20)` and then printing `getFirst()` never checks anything about the *last* item. Josh calls this out explicitly.

11. **Forgetting that the sentinel changes what an "empty" list looks like.** An empty sentinel-based list has one node, not zero. Do not write emptiness checks like `sentinel == null`; use `size == 0` or `sentinel.next == null`.

---

## Likely Exam Points

### 1. Box-and-pointer tracing of `addFirst` / `addLast`

**Q.** Starting from `SLList L = new SLList();` (sentinel-based, sentinel item 63), draw/describe the structure after `L.addLast(20); L.addFirst(5); L.addLast(30);` and give `L.size()`.

**A.** `L` points to an `SLList` object with `size = 3` and `sentinel` pointing at `[63 | *]`. The chain is `[63] -> [5] -> [20] -> [30] -> null`. Reasoning: `addLast(20)` starts `p` at the sentinel, loop does not run, attaches `[20]`. `addFirst(5)` sets `sentinel.next = new IntNode(5, sentinel.next)`, splicing 5 between the sentinel and 20. `addLast(30)` walks `p` from the sentinel to `[20]` (the last node) and attaches `[30]`. `size()` returns 3.

### 2. Efficiency of `size` before and after caching

**Q.** The recursive `size` takes 2 seconds on a list of 1,000 items. How long on 1,000,000 items? What if `size` is cached?

**A.** 2,000 seconds, because the method steps through 1,000 times as many nodes. With caching, `size()` just returns a stored `int`, so it takes essentially the same short time regardless of list length. The cost is a small increment on each add and a few extra bytes of memory (TANSTAAFL).

### 3. Why `addLast` crashes on the empty (non-sentinel) list

**Q.** Given the non-sentinel `SLList` with `public SLList() { first = null; size = 0; }`, exactly which line of `addLast` throws, and what exception?

**A.** `while (p.next != null)` throws a `NullPointerException`, because `p` was assigned `first`, which is `null`, and you cannot dereference `null` to read its `next` field. Note `size += 1` executed first, so the size field is even left inconsistent.

### 4. Fixing `addLast` two ways

**Q.** Give two fixes for that crash, and say which is preferable and why.

**A.** (a) Special-case it: `if (first == null) { first = new IntNode(x, null); return; }` before the loop. (b) Use a sentinel node, so `p` starts at a node that always exists and the loop handles both cases uniformly. (b) is preferable: special cases consume working memory and multiply badly in more complex structures (trees, multi-link structures), whereas the sentinel makes every list structurally identical.

### 5. Public vs. private semantics

**Q.** True or false: making `first` private in `SLList` prevents a `main` method inside `SLList.java` from writing `L.first.next = null`. Explain.

**A.** False. `private` restricts access to code in the same class/file. Code inside `SLList.java` can freely touch `first`; only *other* classes are blocked at compile time with "first has private access in SLList."

**Q (follow-up).** What is the main reason to mark things private?

**A.** To hide implementation details (less for users to learn) and to preserve your freedom to change the implementation later, since nothing outside depends on it. It is not a security feature.

### 6. Nested and static nested classes

**Q.** Can `IntNode` be declared `static` inside `SLList`? What would prevent it? What is gained?

**A.** Yes, because `IntNode` never references any instance variable or instance method of `SLList` (it uses only `item`, `next`, and its own constructor). If any `IntNode` method referred to `first` or `addFirst`, `static` would not compile. The gain is a small memory savings: a static nested class instance does not store a reference to an enclosing `SLList` object.

### 7. Recursive helper pattern

**Q.** Write a method `public int sum()` for a sentinel-based `SLList` that returns the sum of all items, using recursion.

**A.**

```java
private static int sum(IntNode p) {
    if (p == null) {
        return 0;
    }
    return p.item + sum(p.next);
}

public int sum() {
    return sum(sentinel.next);   // skip the sentinel!
}
```

Key points: private static helper taking an `IntNode`; the public method starts at `sentinel.next` so the placeholder value is never added; base case `p == null` handles the empty list correctly. The two methods are **overloaded**.

### 8. Invariants

**Q.** State three invariants of a sentinel-based `SLList` and explain how one of them simplifies `addLast`.

**A.** (i) `sentinel` always points to a sentinel node and is never null; (ii) the first real item, if any, is at `sentinel.next`; (iii) `size` is always the number of items in the list. Invariant (i) means `addLast` can set `IntNode p = sentinel;` and immediately evaluate `p.next` with no null check, since `p` is guaranteed non-null. Every method must also *preserve* the invariants, e.g. by incrementing `size`.

### 9. Spot-the-bug on sentinel `addFirst`

**Q.** What is wrong with `public void addFirst(int x) { sentinel = new IntNode(x, sentinel); size += 1; }`?

**A.** It reassigns `sentinel` so it no longer points at the sentinel node. The new node becomes the "sentinel" (and its value is then ignored by `getFirst`, which would return the old sentinel's placeholder), and the real placeholder node becomes a data item in the list. Correct version: `sentinel.next = new IntNode(x, sentinel.next);`.

### 10. Why not just add `addFirst` to `IntList`?

**Q.** Why does the lecture reject "just add an `addFirst` method to `IntList`" as a fix?

**A.** Because with a naked recursive list, the user's own variable must be reassigned to point at the new front node; a method called on the object cannot change the caller's variable. Any workaround (e.g. copying values down the list) is awkward and inefficient. The `SLList` middleman solves it cleanly because the user's variable points at the `SLList`, which never changes, while `first` inside it does.

---

## Summary

- **`IntList` is a naked recursive data structure**: its users must understand references and recursion, and they can hold pointers into (and corrupt) the middle of the list.
- **Improvement #1, Rebranding**: `IntList` becomes `IntNode` with fields `item` and `next`, and no methods.
- **Improvement #2, Bureaucracy**: a new `SLList` class holds `first` and acts as a middleman, exposing `addFirst`, `getFirst`, etc. Users never type `null` or `new IntNode`.
- **Improvement #3, Access control**: make `first` (and `IntNode`) `private` to hide implementation details and keep the freedom to change them. `public` is a permanent promise. This is about communication, not security.
- **Improvement #4, Nested classes**: put `IntNode` inside `SLList`; make it private (nobody else needs it) and `static` (it never uses the outer class's instance members, saving a little memory).
- **`addLast` iteratively**: walk `p` with `while (p.next != null)` so it stops on the last node, then attach.
- **Recursive methods on non-recursive classes**: use a **private (static) recursive helper** that takes an `IntNode` argument; the public method calls it. Two same-named, different-signature methods = **overloading**.
- **Improvement #5, Caching**: store `size` in an instance variable updated by every length-changing method. `size()` becomes fast regardless of list length. Recursive `size` on 1,000,000 items would take 2,000 seconds if 1,000 items takes 2. TANSTAAFL, but the tradeoff is clearly worth it.
- **The empty-list bug**: with `first = null`, `addLast` throws a `NullPointerException` at `while (p.next != null)`. A special case fixes it but is ugly.
- **Improvement #6, Sentinel nodes**: always keep one dummy node in front whose value is never read. Empty list = just the sentinel. Rename `first` to `sentinel`; first real item is `sentinel.next`; `addFirst` modifies `sentinel.next`; `addLast` starts `p` at `sentinel` and needs no special case.
- **Invariants**: conditions guaranteed true during execution (sentinel is never null, first item at `sentinel.next`, `size` is accurate). Assume them to simplify code; preserve them in every method.
- **Remaining weaknesses** (next lecture): `SLList` only stores `int`s, and `addLast` / accessing middle items is slow.
