<!-- Wed, Sep 02, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 4: Packages, Recursion, IntLists

## Overview

This lecture closes out the 2D array material from Lecture 3 and then makes the leap into the real heart of CS 61B: building a list from scratch. Java, unlike Python, does not have lists as a core language feature (they were added roughly three years into the language's existence), so we ask what the Java designers had to do to make lists exist at all. The answer, surprisingly, is a two-field class: `public int first;` and `public IntList rest;`. That is an infinitely extendable list. Everything else in this lecture (and the next three) is about making that bare structure pleasant and safe to use: adding a constructor so we can prepend instead of writing `L.rest.rest.rest`, writing `size()` recursively and `iterativeSize()` with a walking pointer `p`, writing `get(i)` recursively via the insight "my ith item is my rest's (i-1)th item," and finally writing client code in a separate `IntListTools` class that distinguishes destructive from non-destructive operations. Packages were on the slides but explicitly labeled extra content and skipped in lecture; they explain how Java avoids duplicate class name collisions and what `public` versus package-private actually means. Throughout, the "Mystery of the Walrus" (reference semantics and the Golden Rule of Equals from Lecture 3) is the tool that makes all of this non-mysterious.

---

## Key Concepts

### 1. Finishing 2D Arrays (carried over from Lecture 3)

A 2D array in Java is **not** a matrix. It is an array of addresses of arrays.

```java
int[][] theMatrix = new int[4][];
```

Reason about this in bits:

- The variable `theMatrix` is a **64-bit** box. It holds an address, not data. In our simplified model, every address in Java is 64 bits.
- `new int[4][]` creates an array of **four 64-bit boxes**. Each of those is itself an address, specifically the address of an `int[]`.
- Once you do `theMatrix[0] = new int[3];`, that creates an array of **three 32-bit boxes**, because those actually hold `int`s.

So there are levels of indirection: a 64-bit box pointing at an array of 64-bit boxes, each of which points at an array of 32-bit `int` boxes.

Two consequences Josh emphasized:

1. **Rows can have different lengths.** Nothing forces `theMatrix[2]` to point at an array of length 3; it could be length 1000. This is why it is not really a matrix.
2. **Aliasing across rows.** Consider:
   ```java
   int[] row0 = theMatrix[0];
   row0[1] = -5;
   ```
   `row0` is a new 64-bit box, and `= theMatrix[0]` copies the *address* of the row array into it (Golden Rule of Equals). So `row0` and `theMatrix[0]` point at the **same** array, and `row0[1] = -5` genuinely changes `theMatrix`.

There is also syntax for creating a 2D array with literal values in advance, but the lecture noted you will never need it in this class. Java also type-checks these: assigning a `double` where an `int[]` element is expected is a compile error ("double cannot be converted to int").

*(Extra context)* Memory is reclaimed automatically in Java by the **garbage collector**; you never manually free objects. Josh cited this as one reason Java suits this course.

### 2. Packages (Extra Content, skipped in lecture but on the slides)

Josh explicitly called this "extra content" and said that for 61B purposes, whether you write `public` or not is basically irrelevant. Still, here is what the slides said.

**The problem: duplicate classes.** If you have `Dog.java` in a `lec2_intro2` folder and another `Dog.java` in a `lec5_lists1` folder, Java gets upset because it sees two `Dog` classes.

**The fix: declare a package.** Putting `package lec5_lists1;` as the first line of the file gives the class a new **canonical name**:

- The one with no package declaration is just named `Dog`.
- The one in the package is named `lec5_lists1.Dog`.

**Importing is not strictly necessary.** You can always use the full canonical name:

```java
lec4_testing.Sort.sort(someArray);   // valid without any import
```

An `import` statement is purely shorthand. When you write `import lec4_testing.Sort;`, you are telling Java: "whenever I write `Sort`, that is shorthand for `lec4_testing.Sort`."

You can also import **static members**. In `import static com.google.common.truth.Truth.assertThat;`:
- `com.google.common.truth` is the package name,
- `Truth` is a class in that package,
- `assertThat` is a static method in `Truth`.

**The CLASSPATH.** Java will not scan every folder on your computer looking for classes: too slow, and it might pick up things you did not want. Instead it looks only along the **CLASSPATH**. (In IntelliJ you can see yours by clicking the `…` in the terminal after running your code.)

**`public` vs. package-private.**
- Declared `public`: usable by code in any package.
- `public` omitted: usable only by code in the **same package**.

### 3. Why We Are Building a List at All

In Java, lists come from the library, not the language:

```java
import java.util.List;
import java.util.LinkedList;
List<String> L = new LinkedList<>();
L.add("a");
L.add("b");
```

This feels clumsy compared to Python. The lecture kicks off a **four-lecture journey**: first a linked-list-based implementation, later an array-based one (which Josh described as clunkier but the more common approach for most purposes). Friday's lecture takes a break to cover automated testing, then lists resume the following week.

The philosophical framing: it feels absurd to "build a list," like being told you will build a quark. But a list turns out to be constructible from abstractions we already have.

### 4. The IntList Definition

A list should be able to grow to arbitrary size, unlike an array, which has a fixed size. Here is the entire definition:

```java
public class IntList {
    public int first;      // this is the first item in the list
    public IntList rest;   // this is the rest of the list
}
```

That's it. This is recursive in its very definition: an `IntList` contains an `int` and another `IntList`. The `rest` field holds a *reference* to another `IntList` object, not a copy of one, which is why this does not infinitely regress. The chain terminates when `rest` is `null`.

**Size in bits:** an `IntList` object's instance variables take 96 bits total: 32 for `first` (an `int`) and 64 for `rest` (an address).

### 5. Two Ways to Build a List

**Adding to the end (ugly).** Without a constructor, you must poke fields one at a time:

```java
IntList L = new IntList();
L.first = 5;
L.rest = null;

L.rest = new IntList();
L.rest.first = 10;

L.rest.rest = new IntList();
L.rest.rest.first = 15;
```

This produces "lots of `.rest.rest.rest` nonsense." Imagine adding the 11th item.

**Adding a constructor helps a little:**

```java
public IntList(int f, IntList r) {
    first = f;
    rest = r;
}
```

Now each node comes into existence fully populated (`new IntList(5, null)`), so the debugger never even shows the temporary default values of `0` and `null`. But you still write `L.rest.rest = new IntList(15, null);` if you insist on appending.

**Adding to the front (beautiful).** Build the list *backwards*:

```java
IntList L = new IntList(15, null);
L = new IntList(10, L);
L = new IntList(5, L);
```

Josh's framing: "whatever `L` used to be, that's going to be the new end, and the way you write that in code is just `L`." Each line creates a new node whose `rest` is the address currently in `L`, then reassigns `L` to point at the new node. No `.rest` chains at all. The lecture's own code used `5 -> 10 -> 3`; the textbook and slide version uses `5 -> 10 -> 15`.

### 6. Recursion: `size()`

The classroom demo: Josh asked a student in the front row what row they were in ("zero"), and the student behind said "that means I'm in row one," and so on, because "I'm one row further back than he is." That is exactly the recursion.

```java
/** Return the size of the list using... recursion! */
public int size() {
    if (rest == null) {
        return 1;
    }
    return 1 + this.rest.size();
}
```

- **Base case:** how do I know I'm at the end? `rest == null`. Then my size is 1 (me, myself).
- **Recursive case:** ask the rest of the list how big it is, then add 1 for myself.

Note the method takes **no arguments**. In Java it is idiomatic to write `L.size()` rather than `size(L)`; the list you are measuring is `this`.

**Why not `if (this == null) return 0;`?** This is the textbook's exercise. The answer: you always call `size()` *on an object*, e.g. `L.size()`. If `L` were `null`, you'd get a `NullPointerException` before the method body ever ran. A method cannot detect that the object it was invoked on is null, because there is no object.

A consequence of this design: `size()` as written cannot handle an empty list, since a "list" is always at least one node.

### 7. Iteration: `iterativeSize()`

```java
/** Return the size of the list using no recursion! */
public int iterativeSize() {
    IntList p = this;
    int totalSize = 0;
    while (p != null) {
        totalSize += 1;
        p = p.rest;
    }
    return totalSize;
}
```

(In lecture Josh spelled the variable `currentLocation`; the textbook and slides use `p`. The textbook recommends the name `p` to remind yourself that the variable holds a **pointer**.)

**Why do we need `p` at all, if we already have `this`?** Because we need something that changes as we walk down the list, and **you cannot reassign `this` in Java**. `this` is a fixed reference to yourself. So we make a separate 64-bit box, initialize it to the same address, and inch it along.

**Box-and-pointer reasoning for `p = p.rest;`:** `p.rest` is a 64-bit address sitting inside the node `p` currently points at. The Golden Rule of Equals says those 64 bits get **copied** into the box `p`. Nothing inside any node is modified. So `p` steps forward, node by node, and the list itself is untouched. Students often fear this will "break the list" because `p` started out equal to `this`; the reference model shows it cannot.

The loop condition `p != null` is what lets this version, unlike `size()`, handle walking off the end cleanly.

### 8. `get(int i)`

The class challenge. Front item is the 0th item; assume the item exists.

```java
/** Return the ith item of this IntList. */
public int get(int i) {
    if (i == 0) {
        return first;        // or this.first
    }
    return rest.get(i - 1);  // or this.rest.get(i - 1)
}
```

The key insight, in Josh's words: **"my 5th item is my rest's 4th item."** Someone asks me for my 5th item; I ask the rest of the list for its 4th; that node asks for the 3rd; and so on until someone is asked for their 0th, which is just their `first`.

**Performance:** `get` takes **linear time**. Getting the last item of a 1,000,000-item list takes far longer than getting the last item of a small one. A future lecture introduces an array-based list that avoids this.

**On bounds checking:** you *could* guard with something like `if (i >= size()) throw new ...`, but as Josh noted, calling `size()` is itself linear, which would make `get` unnecessarily slow. The problem statement said not to worry about invalid `i`.

### 9. Destructive vs. Non-Destructive Methods

Definitions from the slides:

- **Non-destructive:** the method does **not** modify the object it operates on. It returns a new, changed version while leaving the original intact.
- **Destructive:** the method **is allowed to** modify the object, as a side effect. A destructive method might also return `void`.

Josh's mnemonic: someone hands you their list and says "please take care of my baby." Writing `L.first = L.first + x;` is harmful to baby. Not allowed, if you promised to be non-destructive.

This is also our first **client class**: `IntListTools` is a separate class containing `static` methods that operate on `IntList`s, as though someone else had already written `IntList` and we wanted to extend its abilities from outside.

---

## Definitions

- **IntList:** a class with two instance variables, `int first` (the first item) and `IntList rest` (a reference to the rest of the list), forming a recursively defined, arbitrarily extendable list of integers.
- **Linked list:** a list built as a chain of objects, each holding one value and a reference to the next. Terminated by `null`. (Known from CS 61A by this name.)
- **Base case:** the case in a recursive method that returns an answer directly, without recursing. Required; without one, recursion never terminates and you get a `StackOverflowError`.
- **Recursive case:** the case that computes the answer by calling the method on a smaller subproblem (here, `rest`) and combining the result.
- **`this`:** an implicit reference to the object the method was invoked on. It cannot be reassigned in Java.
- **Pointer / reference:** a 64-bit address of an object. In our box-and-pointer model, drawn as an arrow.
- **`null`:** the absence of a reference; used here to mark the end of a list. Josh described it as "shorthand for when the address is actually zero," while noting Java does not formally treat `null` as the integer 0.
- **Destructive method:** a method permitted to modify the object or structure passed to it, as a side effect.
- **Non-destructive method:** a method that leaves its input unmodified, returning a new structure instead.
- **Client class:** a class (like `IntListTools`) that uses another class's public API to provide additional functionality, rather than modifying that class.
- **Package:** a named grouping of classes, declared with `package name;` at the top of a file. Prevents name collisions by giving classes canonical names like `lec5_lists1.Dog`. *(Extra content)*
- **Canonical name:** a class's fully qualified name, `packageName.ClassName`. *(Extra content)*
- **Import:** a shorthand declaration; `import p.C;` means "whenever I write `C`, I mean `p.C`." Never strictly required. *(Extra content)*
- **CLASSPATH:** the set of locations Java searches for classes. Java does not search your whole filesystem. *(Extra content)*
- **Package-private:** the default access level when `public` is omitted; visible only to code in the same package. *(Extra content)*
- **Garbage collector:** the Java mechanism that automatically reclaims memory for objects nothing references anymore. *(Extra context, mentioned in Q&A)*

---

## Worked Examples

### Example 1: Building a list by appending (and why it's painful)

```java
public class IntList {
    public int first;
    public IntList rest;

    public static void main(String[] args) {
        IntList L = new IntList();
        L.first = 5;
        L.rest = null;

        L.rest = new IntList();
        L.rest.first = 10;

        L.rest.rest = new IntList();
        L.rest.rest.first = 15;
    }
}
```

**Step by step, in boxes and pointers:**

1. `new IntList()` builds an object with two boxes: `first` (32 bits, default 0) and `rest` (64 bits, default `null`). `new` returns the object's 64-bit address; the Golden Rule of Equals copies those 64 bits into the variable `L`, which is itself a 64-bit box. Draw an arrow from `L` to the object.
2. `L.first = 5;` copies the 32 bits representing 5 into the `first` box of the object `L` points at.
3. `L.rest = new IntList();` creates a second object (again `0`/`null`), and copies its 64-bit address into the `rest` box of the first object. Arrow from node 1 to node 2.
4. `L.rest.first = 10;` follows `L`'s arrow, then follows that node's `rest` arrow, then writes 10 into `first`.
5. Same again one level deeper for 15.

Result: `L -> [5|•] -> [10|•] -> [15|null]`. It works, but accessing item *n* requires typing `.rest` *n* times.

### Example 2: Building by prepending (the clean way)

```java
public class IntList {
    public int first;
    public IntList rest;

    public IntList(int f, IntList r) {
        first = f;
        rest = r;
    }

    public static void main(String[] args) {
        IntList L = new IntList(15, null);
        L = new IntList(10, L);
        L = new IntList(5, L);
    }
}
```

**Step by step:**

1. `new IntList(15, null)` builds a node with `first = 15`, `rest = null`. `L` points at it. The list is `15`.
2. `new IntList(10, L)`: the argument `L` is evaluated **first**, yielding the 64-bit address of the `15` node. A new node is created with `first = 10` and `rest` set to that address. Then, and only then, is `L` reassigned to the new node's address. The list is now `10 -> 15`. The `15` node was never touched; it just gained an incoming arrow.
3. `new IntList(5, L)`: same again. The list is `5 -> 10 -> 15`.

The subtlety worth internalizing: `L = new IntList(5, L);` looks circular but is not, because the right side is fully evaluated (using the *old* value of `L`) before the assignment copies the result into `L`.

### Example 3: `size()` traced

```java
public int size() {
    if (rest == null) {
        return 1;
    }
    return 1 + this.rest.size();
}
```

With `L` being `5 -> 10 -> 15`, calling `L.size()`:

| Call | `this.first` | `rest == null`? | Action |
|---|---|---|---|
| `L.size()` | 5 | no | returns `1 + (10-node).size()` |
| `(10-node).size()` | 10 | no | returns `1 + (15-node).size()` |
| `(15-node).size()` | 15 | **yes** | returns `1` |

Unwinding: the 15-node returns 1, so the 10-node returns 2, so `L` returns 3. Prints `3`.

This is the "recursive leap of faith": when writing the recursive case, you trust that `rest.size()` returns the right answer for the shorter list, and you only handle your own contribution (the `+ 1`).

### Example 4: `iterativeSize()` traced

```java
public int iterativeSize() {
    IntList p = this;
    int totalSize = 0;
    while (p != null) {
        totalSize += 1;
        p = p.rest;
    }
    return totalSize;
}
```

With `L` being `5 -> 10 -> 15`:

| Iteration | `p` points at | `totalSize` after increment | `p` after step |
|---|---|---|---|
| start | node 5 | 0 | - |
| 1 | node 5 | 1 | node 10 |
| 2 | node 10 | 2 | node 15 |
| 3 | node 15 | 3 | `null` |
| exit | - | 3 | - |

Returns 3.

**Box-and-pointer narration:** `IntList p = this;` creates a brand new 64-bit box holding the same address as `this`, so two arrows point at node 5. Each `p = p.rest;` reads the 64 bits out of the current node's `rest` field and copies them into `p`'s box. Nothing inside any node changes; only `p`'s box changes. In the visualizer you literally watch the `p` arrow inch rightward along the chain until it becomes `null`.

### Example 5: `get(int i)` traced

```java
public int get(int i) {
    if (i == 0) {
        return first;
    }
    return rest.get(i - 1);
}
```

`L` is `5 -> 10 -> 15`. Call `L.get(2)`:

1. `L.get(2)`: `i` is 2, not 0, so return `(10-node).get(1)`.
2. `(10-node).get(1)`: `i` is 1, not 0, so return `(15-node).get(0)`.
3. `(15-node).get(0)`: `i` is 0, so return `first`, which is 15.

Each frame passes 15 back up. Result: 15. (`L.get(0)` is 5, `L.get(1)` is 10.)

Notice the two things shrinking in lockstep: the index `i` counts down toward 0 while the list pointer walks forward toward the end. The base case fires when the index hits 0, not when the list ends.

### Example 6: `incrementRecursiveNonDestructive`

Lecture's client class:

```java
package lec4_lists1;

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

**Why it is written this way:**

- It is `static` and takes `L` as a parameter, because it lives in a *different class* than `IntList`. (In lecture Josh initially forgot `static` and had to add it so he could call `IntListTools.incrementRecursiveNonDestructive(L, 10)`.)
- The illegal move would be `L.first = L.first + x;`. That mutates the caller's list: destructive, and forbidden by this method's contract.
- Instead, we allocate a **brand new node** for each node of the original. The original nodes are read from but never written to.

**Tracing with `L` being `5 -> 10 -> 3` and `x = 10`:**

1. `L` is not null. Build new node `[15|null]`. Recurse on `10 -> 3`.
2. Not null. Build new node `[20|null]`. Recurse on `3`.
3. Not null. Build new node `[13|null]`. Recurse on `null`.
4. `L == null`, return `null`. So node `[13]`'s `rest` becomes `null`; return the `13` node.
5. Node `[20]`'s `rest` becomes the `13` node; return the `20` node.
6. Node `[15]`'s `rest` becomes the `20` node; return the `15` node.

`L2` is `15 -> 20 -> 13`, and `L` is still `5 -> 10 -> 3`.

**The bug hunt from lecture (worth studying):** Josh first wrote the method with *no* base case and got a `StackOverflowError`, because the recursion eventually calls the method on `null` and dereferences `null.first`... or rather, recurses forever. A student suggested the base case `if (L.rest == null) return null;`. Running it produced `15 -> 20` : the last element, 3, was silently dropped, because that base case throws away the final node instead of copying it. The correct base case is `if (L == null) return null;`, which produces `15 -> 20 -> 13`. The lesson: pick the base case that corresponds to "there is nothing left to copy," not "there is one thing left."

**Second goal (stated on the slides, left as an exercise):** `incrementRecursiveDestructive(IntList L, int x)` returns an incremented version of `L` and **also** changes `L` as a side effect. The destructive version would mutate `L.first` in place and recurse on `L.rest`, with no new nodes allocated.

---

## Common Pitfalls

1. **Forgetting a base case.** Any recursive method without one eventually blows the call stack: `StackOverflowError`. This happened live in lecture.

2. **Picking the wrong base case.** `if (L.rest == null) return null;` in the increment method looks plausible but drops the last element. Always ask: for the smallest possible input, does my base case return the right thing?

3. **Writing `if (this == null) return 0;`.** You cannot test whether `this` is null. You invoked the method *on* an object; if the reference were null you would already have a `NullPointerException`. This is the textbook's explicit exercise.

4. **Trying to reassign `this`.** Java forbids it. That's exactly why `iterativeSize` needs a separate pointer variable `p`.

5. **Fearing that `p = p.rest;` mutates the list.** It does not. It copies 64 bits into `p`'s own box. Only assignments of the form `someNode.rest = ...` or `someNode.first = ...` change the list.

6. **Confusing `L = new IntList(5, L);` with something circular.** The right-hand side is evaluated with the old `L` before assignment.

7. **Mutating a list inside a method documented as non-destructive.** Writing `L.first = L.first + x;` changes the caller's data. The caller entrusted you with their list.

8. **Assuming `get(i)` is fast.** It is linear. Getting the last item of a million-element IntList walks a million nodes.

9. **Adding a bounds check with `size()`.** Correct but expensive: `size()` is itself linear, so `get` would become two linear passes for no good reason.

10. **Forgetting `static` on a method in a client class** that you intend to call as `IntListTools.method(...)` without constructing an `IntListTools` object.

11. **Treating a 2D array as a rectangle.** Rows are independent arrays and may have different lengths, or be aliased by other variables.

12. **Duplicate class names across folders.** Java complains if two `Dog` classes are visible; declare packages to give them distinct canonical names. *(Extra content)*

---

## Likely Exam Points

### 1. Box-and-pointer diagrams for IntList construction

**Q:** After running the code below, how many `IntList` objects exist, and what does `L` print as a sequence?
```java
IntList L = new IntList(15, null);
IntList M = new IntList(10, L);
L = new IntList(5, M);
```
**A:** Three objects. `M` points at `10 -> 15`, and `L` points at `5 -> 10 -> 15`. No object is orphaned: the `15` node has two incoming references paths (from `M.rest` and from `L.rest.rest`, which are the same node). Note that reassigning `L` did not change what `M` sees; `M` still refers to `10 -> 15`.

### 2. Bit-size questions about memory boxes

**Q:** In `int[][] x = new int[4][6];`, how many bits is the box named `x`, how many bits is each box in the length-4 array, and how many bits is each box in a length-6 array?
**A:** 64 bits (an address), 64 bits each (each is an address of an `int[]`), and 32 bits each (each is an `int`). An `IntList` object's fields total 96 bits: 32 for `first`, 64 for `rest`.

### 3. Writing `size()` recursively / spotting a broken base case

**Q:** Why does the following fail, and on what input?
```java
public int size() {
    return 1 + rest.size();
}
```
**A:** No base case. On any list, it eventually calls `size()` on `null` via `rest.size()` where `rest` is `null`, throwing a `NullPointerException`. Fix: `if (rest == null) return 1;` first.

### 4. Iterative traversal and the role of `p`

**Q:** Why can't `iterativeSize` be written as `while (this != null) { totalSize += 1; this = this.rest; }`?
**A:** Because `this` cannot be reassigned in Java, so `this = this.rest;` does not compile. You need a separate local reference variable, conventionally named `p`, initialized to `this`.

### 5. Writing `get(i)` and reasoning about the base case

**Q:** Write `get(int i)` recursively, then explain in one sentence what the recursive call means.
**A:**
```java
public int get(int i) {
    if (i == 0) {
        return first;
    }
    return rest.get(i - 1);
}
```
The recursive call means: my ith item is my rest's (i-1)th item.

### 6. Runtime of `get`

**Q:** If an IntList has N elements, how long does `L.get(N - 1)` take, roughly?
**A:** Linear in N: the call chain visits every node. This is a known weakness of linked lists and motivates array-based lists later in the course.

### 7. Destructive vs. non-destructive

**Q:** Which of these is destructive? For the non-destructive one, how many new `IntList` objects does it allocate on a list of length N?
```java
// (a)
public static IntList incA(IntList L, int x) {
    if (L == null) return null;
    L.first += x;
    incA(L.rest, x);
    return L;
}
// (b)
public static IntList incB(IntList L, int x) {
    if (L == null) return null;
    return new IntList(L.first + x, incB(L.rest, x));
}
```
**A:** (a) is destructive: it mutates `L.first` in place, so the caller's list changes and it allocates 0 new objects. (b) is non-destructive and allocates exactly N new `IntList` objects, one per element of the original. Note (b) is a more compact but equivalent form of the lecture's `incrementRecursiveNonDestructive`.

### 8. Finding the base-case bug

**Q:** A student writes a non-destructive increment with the base case `if (L.rest == null) return null;`. On the input `5 -> 10 -> 3` with `x = 10`, what is the output, and what is wrong?
**A:** Output is `15 -> 20`. The base case fires while there is still a node (`3`) left to copy, and returns `null` instead of copying it, silently dropping the last element. Correct base case: `if (L == null) return null;`.

### 9. Why not check `this == null`

**Q:** Explain why `if (this == null) return 0;` is not a valid way to handle empty lists.
**A:** `size()` is invoked on an object, e.g. `L.size()`. If `L` were `null`, the JVM throws a `NullPointerException` at the call site, before any method body executes. A method can never observe that `this` is null.

### 10. Packages and imports *(Extra content, less likely to be tested given it was skipped)*

**Q:** True or false: you must `import` a class to use it.
**A:** False. Importing is purely shorthand; you can always write the full canonical name, e.g. `lec4_testing.Sort`. Separately: omitting `public` makes a class or member visible only within its own package.

---

## Summary

- Lecture opened by finishing 2D arrays: `int[][]` is a 64-bit reference to an array of 64-bit references to `int[]`s, rows may differ in length, and aliasing a row (`int[] row0 = theMatrix[0];`) lets you mutate the original.
- **Packages were explicitly extra content and skipped in lecture.** Key points from slides: `package p;` gives a class the canonical name `p.ClassName` and resolves duplicate-class conflicts; `import` is just shorthand and never required; Java searches only the CLASSPATH; `public` means visible from any package, omitting it means package-private only.
- Java lists are a library feature, not a language feature. This is lecture 1 of 4 building lists ourselves: linked-list-based first, array-based later.
- An `IntList` is just `int first` plus `IntList rest`, a recursively defined, arbitrarily extendable list. `null` terminates it. Object fields total 96 bits.
- Appending without a constructor produces `.rest.rest.rest` chains. Adding the constructor `IntList(int f, IntList r)` enables the clean prepend idiom: `L = new IntList(5, L);`, building the list backwards.
- `size()` recursive: base case `rest == null` returns 1; recursive case returns `1 + rest.size()`. The "recursive leap of faith" is trusting the subproblem's answer.
- `iterativeSize()` uses a separate pointer `p`, initialized to `this`, advanced by `p = p.rest;` until `null`. The separate variable is needed because `this` cannot be reassigned. Advancing `p` never modifies the list.
- `get(int i)`: base case `i == 0` returns `first`; otherwise `rest.get(i - 1)`. "My ith item is my rest's (i-1)th item." Runtime is linear.
- `IntListTools` demonstrates a client class with `static` methods operating on `IntList`s.
- **Non-destructive** means leaving the input untouched (allocate new nodes); **destructive** means mutating it as a side effect (and may return `void`).
- The live-coded bug: missing base case gives `StackOverflowError`; the base case `L.rest == null` silently drops the last element; the correct one is `L == null`.
- Every step is explained by the Mystery of the Walrus / Golden Rule of Equals: assignment copies bits, and for objects those bits are a 64-bit address.
