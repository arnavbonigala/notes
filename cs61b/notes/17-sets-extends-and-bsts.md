<!-- Mon, Oct 05, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 17: Sets, Extends, and BSTs

## Overview

This lecture has two halves that are joined by a single practical goal: building a `Set` (or `Map`) that is faster than the `Θ(N)` `ArraySet` we saw earlier in the course. The first half introduces the `extends` keyword, which lets one class become a hyponym (subtype) of another *class* rather than of an interface, inheriting all of its members for free. We see this twice: `RotatingLL extends LinkedList` to add a `rotateLeft()` method, and `TimeSeries extends TreeMap<Integer, Double>` to get `put` and `get` for free on Project 4A. That second example raises the obvious question: what is actually inside a `TreeMap`? The second half answers it by *deriving* the binary search tree from an ordered linked list, step by step (add express lanes, move the entry point to the middle, flip the left links, recurse), then formalizing the BST property and implementing `contains`/`find`, `insert`, and deletion (including the three-case "Hibbard deletion"). The lecture closes by observing that a Set and a Map are structurally the same thing: a Map is just a BST whose nodes store key/value pairs instead of keys alone.

---

## Key Concepts

### 1. `extends`: inheritance between classes

So far we have used `implements` to declare that a class is a hyponym of an *interface*: `ArrayList<T> implements List<T>`. If you want one **class** to be a hyponym of another **class**, you use `extends` instead.

```
                   List
        implements  |  implements
      ArrayList        LinkedList
                           | extends
                       RotatingLL
```

Because of `extends`, the subclass **inherits all members** of the superclass:

- All instance and static variables.
- All methods.
- All nested classes.

The critical caveat from the slides: *members may be private and thus inaccessible*. Inheritance is about the type relationship and the member set, not about defeating access control. `RotatingLL` "has" everything `LinkedList` has, but code in `RotatingLL` can only touch the parts that `LinkedList` made visible to it.

A subclass can also:

- **Override** superclass methods (provide its own implementation of an inherited method).
- **Invoke** superclass methods, variables, or constructors using `super`. Josh notes we won't really use `super` in 61B, though it may appear on discussion worksheets.

### 2. Constructors are called implicitly up the chain

When you write `new RotatingLL()`, the superclass (`LinkedList`) constructor is **implicitly called** for you. You do not have to write `super()` yourself. This matters because the superclass constructor is what sets up the superclass's private state: the slides specifically call out that this "ensures that, e.g. any required sentinel variable is defined." If the `LinkedList` constructor did not run, the inherited `addLast` would be operating on a half-built object.

### 3. `extends` vs. `implements`: nearly synonyms, with one distinction

Josh's framing: "to me, `extends` and `implements` are nearly exact synonyms."

- Both establish hypernym/hyponym relationships, and in both cases the subtype inherits all members of the supertype.
- `implements` is used between a **class and an interface**.
- `extends` is used between **class and class**, and also between **interface and interface** (e.g. `List extends Collection`).

So why does Java have two keywords instead of one? The slide's answer: `implements` is something special, because it marks **crossing the boundary from abstract to concrete**. An interface is a specification; a class is an implementation. `implements` is the word for "here is a concrete realization of that abstract thing."

### 4. Inheritance as a labor-saving device (and layers of abstraction)

The `TimeSeries` example is the pedagogical point of the first half. You *could* design a `TimeSeries` data structure from scratch and implement `put` and `get` yourself. Or you can write `extends TreeMap<Integer, Double>` and get both methods instantly, correct and efficient.

This is the usual story of layered abstraction:

```
   TimeSeries  ───uses───▶  TreeMap
   (you write)              (Josh Bloch, Doug Lea wrote)
```

On Project 4A you add *new* methods such as `plus`, and the nice thing is **you don't have to know anything at all about how `put` and `get` work.** You just trust them. The lecture then deliberately breaks that trust for teaching purposes: it notes that technically `TimeSeries` inherits all members of `TreeMap`, including the private tree starting at `root`, and that IntelliJ "yells at us" if we try to access `root`. Using Java's reflection library (which lets you ignore access modifiers) you can actually peek at the underlying tree. The slide showed that tree in a diagram (not recoverable from the PDF text), and the point was: *over the next three lectures we'll work to understand how this TreeMap works.*

### 5. Abstract Data Types, restated

An **Abstract Data Type (ADT)** is defined only by its **operations**, not by its implementation. The lecture reviews the course's running examples:

- **Deque ADT**: `addFirst`, `addLast`, `isEmpty`, `size`, `printDeque`, `removeFirst`, `removeLast`, `get(int index)`. `ArrayDeque` and `LinkedListDeque` are two implementations.
- **List61B**: implemented by `AList` and `SLList`; the interface provided default methods.
- **DisjointSets**: a richer family of implementations, `ListOfSetsDS`, `QuickFindDS`, `QuickUnionDS`, `WeightedQuickUnionDS`.

And `java.util` provides both interfaces (ADTs) and implementations:

```
                    Collection
          ┌──────────────┼──────────────┐
        List            Set            Map
     ┌────┴────┐     ┌───┴────┐     ┌───┴────┐
LinkedList ArrayList HashSet TreeSet HashMap TreeMap
```

Today's goal is the basic idea behind **`TreeSet` and `TreeMap`**.

### 6. Deriving the BST from an ordered linked list

This derivation is the heart of the lecture, and it is the thing most worth being able to retell.

**Start:** `ArraySet`. Easy to implement, terrible runtime: `contains` is `Θ(N)`, `add` is `Θ(N)`.

**Step 1: sort it.** Build an `OrderedLinkedListSet<Character>` holding `A B C D E F G` in order behind a sentinel, with a `size` field. Runtime of `contains`? `Θ(N)`. Runtime of `add`? `Θ(N)`. **No improvement over `ArraySet` yet.** The fundamental problem: search is slow *even though the data is in order*, because a linked list only lets you walk one step at a time.

**Optimization idea #1: extra links.** Add (random) "express lanes" that skip ahead over several nodes, so you can travel long distances quickly and drop down to the slow lane near your target. This is a **skip list**. The slide explicitly says we won't discuss skip lists in 61B.

**Optimization idea #2, part a: change the entry point.** Instead of entering the list at `A`, point into the **middle** (`D`). Now you can reach `E, F, G` quickly. But you can no longer reach `A, B, C` at all, since all the links point rightward.

**Optimization idea #2, part b: flip the left links.** Reverse the direction of the links to the left of `D`, so from `D` you can walk leftward through `C, B, A`. Now a search starts at `D`, decides left or right once, and only has to scan one half. **Search time halved.**

**Optimization idea #2, part c: repeat recursively.** "Dream big." Apply the same trick within each half: from `D`, the left chain's entry point becomes its own middle (`B`), with `A` to its left and `C` to its right; the right chain's entry point becomes `F`, with `E` left and `G` right. Carrying this all the way down turns the flat list into:

```
             D
          /     \
        B         F
       / \       / \
      A   C     E   G
```

"What we've just invented is the binary search tree." Each node is just a former list node with two links, and the left/right choice at each step throws away half of the remaining candidates instead of one.

### 7. Trees, rooted trees, rooted binary trees, BSTs

The lecture builds the definition in four layers, each adding one constraint:

1. **Tree**: a set of nodes plus a set of edges connecting them, with the constraint that **there is exactly one path between any two nodes**. (This rules out both disconnected structures and structures with cycles, which is why some of the shapes on the slide were marked "not a tree.")
2. **Rooted tree**: one node is designated the **root**. Every node except the root has exactly one **parent**, defined as the first node on the path from that node to the root. Unlike (most) real trees, the root is drawn at the **top**. A node with no child is a **leaf**.
3. **Rooted binary tree**: every node has 0, 1, or 2 children (subtrees). The slide's third example, with a node having three children labeled `C`, was marked "Not binary!"
4. **Binary search tree**: a rooted binary tree with the **BST property**.

**BST property.** For every node `X` in the tree:
- Every key in the **left** subtree is **less than** `X`'s key.
- Every key in the **right** subtree is **greater than** `X`'s key.

Note carefully: the condition is about *every key in the entire subtree*, not just the immediate children. The lecture's contrast pair makes this concrete:

```
        dog                          debt
      /     \                      /      \
   bag       flat               bus        ears
   / \       /  \               / \        /   \
 alf  cat  elf  glut          axe  cow  fish   gut

 Binary Search Tree        Binary Tree, but NOT a BST
```

The right tree fails: `bus` sits in `debt`'s left subtree but `bus ≻ debt`... and more clearly, `axe, cow` under `bus` and `fish, gut` under `ears` do not satisfy an ordering relative to the ancestors. It is a legal binary tree; it just isn't *search*-ordered.

### 8. What the ordering must satisfy

The BST property only makes sense if `≺` is a real total ordering. The lecture requires it be **complete, transitive, and antisymmetric**. Given keys `p` and `q`:

- Exactly one of `p ≺ q` and `q ≺ p` is true.
- `p ≺ q` and `q ≺ r` imply `p ≺ r`.

One direct consequence of "exactly one of ... is true": **no duplicate keys allowed**, since for a duplicate pair neither `p ≺ q` nor `q ≺ p` would hold. This keeps things simple, and most real-world implementations follow the same rule.

The lecture writes `≺` rather than `<` deliberately: it reminds us that the ordering is **arbitrary**, i.e. our choice. For example, we could have used string length to define `≺` instead of alphabetical order, and we'd get a perfectly valid (but differently shaped) BST.

### 9. Why BSTs are fast

Search on a **"bushy"** BST is `Θ(log N)` in the worst case, because the height of a bushy tree is about `log₂(N)`, and search follows one root-to-leaf path doing constant work per node. The lecture is candid that *"bushiness" is an intuitive concept that we haven't defined* yet, so treat it informally as "not stringy, roughly balanced, depth about `log N` everywhere."

The slides emphasize how dramatic `log N` is: at one microsecond per operation, you could find something in a tree of size `10^300000` in one second. And "much (perhaps most?) computation is dedicated towards finding things in response to queries," so it is a very good thing that such queries are nearly free.

### 10. Avoid arms-length recursion

Introduced alongside `insert`, this is a general code-quality principle the course cares about, not just a BST detail. **Arms-length recursion** is checking for a condition one level *above* where the recursion would naturally handle it, forcing you to write a complicated base case. The slide shows an escalating ladder of badness:

```java
// Worst: inspecting two levels down
if (T.left.left == null)  T.left.left  = new BST(ik);
else if (T.left.right == null)  T.left.right  = new BST(ik);
else if (T.right.left == null)  T.right.left  = new BST(ik);
else if (T.right.right == null) T.right.right = new BST(ik);
// "This base case is too complicated. The recursion can take us further."

// Better, but still not best:
if (T.left == null)       T.left  = new BST(ik);
else if (T.right == null) T.right = new BST(ik);

// The best base case:
if (T == null) return new BST(ik);
```

Let the recursion walk all the way off the end of the tree and handle `null` itself. The base case becomes one line, and all the link-setting happens uniformly in the recursive case.

### 11. Sets and Maps are the same thing

A BST of keys represents a **Set**: the tree

```
          sumomo
        /        \
     momo        uchi
     /   \
   mo     no
```

represents `{mo, no, sumomo, uchi, momo}`.

To represent a **Map** instead (say, word counts), **just have each BST node store a key/value pair**:

```
          sumomo 1
         /        \
     momo 2       uchi 1
     /     \
  mo 2     no 1
```

The structure, the search logic, the insert logic, and the delete logic are all unchanged; the ordering is still done purely on the key. This is why `TreeSet` and `TreeMap` are essentially the same data structure with a different public face.

One asymmetry the lecture flags: there is **no efficient way to look up by value**. You cannot find all keys with value `= 1` without iterating over *all* nodes. The slide's verdict: "This is fine." Maps are designed for key lookup.

---

## Definitions

- **`extends`**: Java keyword declaring that one class is a subclass (hyponym) of another class, or that one interface is a subinterface of another interface. The subclass inherits all members of the superclass.
- **`implements`**: Java keyword declaring that a class is a hyponym of an interface, i.e. provides a concrete realization of an abstract specification. Marks the crossing from abstract to concrete.
- **Inherits**: a subclass automatically possesses all members of its superclass: all instance and static variables, all methods, and all nested classes. Private members are inherited but inaccessible.
- **Override**: to provide, in a subclass, a new implementation of a method already defined in the superclass.
- **`super`**: keyword used to invoke a superclass method, variable, or constructor from a subclass. Not really used in 61B.
- **Abstract Data Type (ADT)**: a type defined only by its operations, not by its implementation. Examples: Deque, List, Set, Map, Disjoint Sets.
- **Tree**: a set of nodes plus a set of edges connecting those nodes, subject to the constraint that there is exactly one path between any two nodes.
- **Rooted tree**: a tree in which one node is designated the root. Every node except the root has exactly one parent.
- **Parent** (of node `N`): the first node on the path from `N` to the root.
- **Child**: `B` is a child of `A` if `A` is the parent of `B`.
- **Leaf**: a node with no children.
- **Rooted binary tree**: a rooted tree in which every node has 0, 1, or 2 children (subtrees).
- **BST property**: for every node `X`, every key in `X`'s left subtree is less than `X`'s key, and every key in `X`'s right subtree is greater than `X`'s key.
- **Binary search tree (BST)**: a rooted binary tree with the BST property.
- **`≺`**: the lecture's symbol for the (arbitrary but fixed) ordering on keys, written instead of `<` to stress that we choose the ordering. Must be complete, transitive, and antisymmetric.
- **Complete ordering**: for any keys `p`, `q`, exactly one of `p ≺ q` and `q ≺ p` holds.
- **Transitive ordering**: `p ≺ q` and `q ≺ r` imply `p ≺ r`.
- **Bushy**: informal term for a BST whose height is roughly `log₂ N`, i.e. roughly balanced. Not formally defined in this lecture.
- **Predecessor** (of a key `k` in a BST): the largest key in the tree that is `≺ k`, i.e. the rightmost node of `k`'s left subtree.
- **Successor** (of a key `k` in a BST): the smallest key in the tree that is `≻ k`, i.e. the leftmost node of `k`'s right subtree.
- **Hibbard deletion**: the strategy for deleting a node with two children: replace it with a copy of its predecessor or successor, then delete that predecessor/successor from its original position (a deletion guaranteed to fall into the 0-child or 1-child case).
- **Arms-length recursion**: the bad habit of testing for a condition one or more levels above where the recursion would naturally handle it, producing an unnecessarily complicated base case.
- **Skip list**: a sorted linked list augmented with randomized "express lane" links for faster search. Mentioned as optimization idea #1, not covered in 61B.
- **Reflection library**: a Java library that lets you ignore access modifiers and inspect private members. Used in lecture only to peek inside `TreeMap`.

---

## Worked Examples

### Example 1: `RotatingLL` without `extends` (it does not compile)

```java
public class RotatingLL<Item> {
    public static void main(String[] args) {
        RotatingLL<Integer> rsl = new RotatingLL<>();
        /* Creates List: [10, 11, 12, 13] */
        rsl.addLast(10);
        rsl.addLast(11);
        rsl.addLast(12);
        rsl.addLast(13);

        /* Should be: [11, 12, 13, 10] */
        rsl.rotateLeft();
        System.out.println(rsl.getFirst()); // print 11
    }
}
```

**What happens:** this **does not compile**. `RotatingLL` is a brand-new class with no members at all, so it is missing `addLast`, `rotateLeft`, and `getFirst`. Java's compiler checks method calls against the **static type** of the receiver, and the static type `RotatingLL<Integer>` declares none of these methods. Note this is a *compile-time* failure, so nothing runs at all.

### Example 2: adding `extends LinkedList<Item>`

```java
public class RotatingLL<Item> extends LinkedList<Item> {
    public static void main(String[] args) {
        RotatingLL<Integer> rsl = new RotatingLL<>();
        /* Creates List: [10, 11, 12, 13] */
        rsl.addLast(10);
        rsl.addLast(11);
        rsl.addLast(12);
        rsl.addLast(13);

        /* Should be: [11, 12, 13, 10] */
        rsl.rotateLeft();
        System.out.println(rsl.getFirst()); // print 11
    }
}
```

**What changed:** now the compiler knows that a `RotatingLL` **is a** `LinkedList`, so `RotatingLL` inherits `addLast` and `getFirst` from `LinkedList`. Those four `addLast` calls now resolve and work.

**Box-and-pointer reasoning in words:** `new RotatingLL<>()` allocates one object. That object's "box" contains the instance variables declared in `RotatingLL` (none yet) *plus* all the instance variables `LinkedList` declares (its header/sentinel node, its size counter, and so on). There is exactly one object, not two: inheritance widens the object, it does not create a separate `LinkedList` object alongside it. The `LinkedList` constructor runs implicitly to initialize that inherited part, which is why the sentinel is set up and `addLast` has something valid to operate on. The variable `rsl` holds one arrow pointing at that single widened box.

**Still broken:** `rotateLeft` does not exist anywhere in `LinkedList`, so the call to `rsl.rotateLeft()` still fails to compile. That's the method we have to write.

### Example 3: implementing `rotateLeft`, built up in steps

The lecture builds the method body incrementally.

**Step 1: empty stub.**
```java
/** Rotates list to the left. */
public void rotateLeft() {

}
```
Compiles now, but does nothing.

**Step 2: remove the front item.**
```java
/** Rotates list to the left. */
public void rotateLeft() {
    Item oldFirst = removeFirst();
}
```
`removeFirst()` is an unqualified call with no receiver, so it is an **implicit `this.removeFirst()`**, resolved to the inherited `LinkedList.removeFirst`. It detaches and returns the front item. For `[10, 11, 12, 13]`, `oldFirst` is now the `Integer` `10` and the list is `[11, 12, 13]`. Note the declared type is `Item`, the class's own type parameter, which is `Integer` for this instance.

**Step 3: put it on the back.**
```java
/** Rotates list to the left. */
public void rotateLeft() {
    Item oldFirst = removeFirst();
    addLast(oldFirst);
}
```
`addLast(oldFirst)` appends the saved item, giving `[11, 12, 13, 10]`. Then `rsl.getFirst()` returns `11`, matching the comment.

**Final class:**
```java
public class RotatingLL<Item> extends LinkedList<Item> {
    public void rotateLeft() {
        Item oldFirst = removeFirst();
        addLast(oldFirst);
    }
}
```

That is the whole class. Two lines of logic buy us a fully functional list with every `LinkedList` operation plus rotation, because everything else was inherited.

**A caution about the slides themselves:** the slide titled "Demo: Rotating SLList" has a comment reading `/* Rotates from [10, 11, 12, 13] to [13, 10, 11, 12] */` while still printing `11`. That comment describes a rotate-*right*, and is inconsistent with both the method name and the printed output. The correct result for `rotateLeft` on `[10, 11, 12, 13]` is `[11, 12, 13, 10]`, with `getFirst()` returning `11`, as the earlier slides say. Also, a later slide calls `rsl.print()`, which is not a `LinkedList` method, so take that line as pseudocode for "show the list." (Slide inconsistency noted; the logic above is what matters.)

### Example 4: `TimeSeries extends TreeMap<Integer, Double>` (Project 4A)

**Before** (does not work):
```java
public class TimeSeries {
    public static void main(String[] args) {
        TimeSeries usGDP = new TimeSeries();
        usGDP.put(1990, 5.963);
        usGDP.put(1991, 6.158);
        usGDP.put(1992, 6.520);
        usGDP.put(1993, 6.858);
        usGDP.put(1994, 7.287);
        usGDP.put(1995, 7.639);
        usGDP.put(1996, 8.073);
        usGDP.put(1997, 8.577);
        System.out.println(usGDP.get(1990));
    }
}
```
No `put`, no `get`: nothing to call.

**After** (works, with a one-word change):
```java
public class TimeSeries extends TreeMap<Integer, Double> {
    public static void main(String[] args) {
        TimeSeries usGDP = new TimeSeries();
        usGDP.put(1990, 5.963);
        usGDP.put(1991, 6.158);
        // ... etc ...
        System.out.println(usGDP.get(1990));
    }
}
```

**Step by step:**
1. `extends TreeMap<Integer, Double>` fixes the type arguments: keys are years (`Integer`), values are GDP figures (`Double`). The data is `GDP of the US in 1990 is $5.963 trillion`, so key `1990` maps to value `5.963`.
2. `usGDP.put(1990, 5.963)` calls the inherited `TreeMap.put`. The literal `1990` is an `int` and `5.963` is a `double`; Java **autoboxes** them to `Integer` and `Double` to match the type parameters.
3. Internally `TreeMap` walks its (private) search tree comparing years and inserts a node, exactly the `insert` algorithm developed later in this lecture. We don't need to know that to use it.
4. `usGDP.get(1990)` searches by key and returns `5.963`.

**The point of 4A:** your job is to add *new* methods such as `plus`:

```java
TimeSeries usGDP = new TimeSeries();
TimeSeries chinaGDP = new TimeSeries();
...
System.out.println(usGDP.plus(chinaGDP));
```

You implement `plus` in terms of `put`/`get`/iteration, treating the inherited map operations as a trusted black box. The slide's framing: "you don't have to know anything at all about how `put` and `get` work. You can just trust them."

### Example 5: `find` (BST search)

```java
static BST find(BST T, Key sk) {
    if (T == null)
        return null;
    if (sk.equals(T.key))
        return T;
    else if (sk ≺ T.key)
        return find(T.left, sk);
    else
        return find(T.right, sk);
}
```

(The `≺` is lecture pseudocode; in real Java you would write something like `sk.compareTo(T.key) < 0`. *(extra context)*)

**The rule in words:** if `searchKey` equals `T.key`, return. If `searchKey ≺ T.key`, search `T.left`. If `searchKey ≻ T.key`, search `T.right`.

**Trace on the lecture's tree, searching for `elf`:**

```
        dog
      /     \
   bag       flat
   / \       /  \
 alf  cat  elf  glut
```

1. `find(dog-node, "elf")`. `T` is not null. `"elf".equals("dog")` is false. Is `elf ≺ dog`? No, `e` comes after `d`. So recurse right: `find(flat-node, "elf")`.
2. `T` is not null. Not equal to `flat`. Is `elf ≺ flat`? Yes, `el` comes before `fl`. Recurse left: `find(elf-node, "elf")`.
3. `T` is not null. `"elf".equals("elf")` is true. **Return the `elf` node.**

Three node visits in a 7-node tree. Searching for a missing key, say `dot`: at `dog`, `dot ≻ dog`, go right to `flat`; `dot ≺ flat`, go left to `elf`; `dot ≺ elf`, go left to `elf.left`, which is `null`; the base case returns `null`. Notice the base case is what makes "not found" work, and notice we never had to peek ahead to check whether a child was null before recursing: that is the anti-arms-length-recursion discipline in action.

**Why this is `Θ(log N)` on a bushy tree:** each call does `O(1)` comparison work and descends one level. The number of levels in a bushy tree is about `log₂ N`, so the worst case (walking root to leaf) is `Θ(log N)`.

### Example 6: `insert`

```java
static BST insert(BST T, Key ik) {
    if (T == null)
        return new BST(ik);
    if (ik ≺ T.key)
        T.left = insert(T.left, ik);
    else if (ik ≻ T.key)
        T.right = insert(T.right, ik);
    return T;
}
```

**The rule in words:** search for the key. If found, do nothing. If not found, create a new node and set the appropriate link.

**Trace: `insert("eyes")` into the tree above.**

```
        dog                            dog
      /     \                        /     \
   bag       flat        ──▶      bag       flat
   / \       /  \                 / \       /  \
 alf  cat  elf  glut            alf  cat  elf  glut
                                            \
                                            eyes
```

1. `insert(dog-node, "eyes")`. Not null. `eyes ≺ dog`? No. `eyes ≻ dog`? Yes. So `dog.right = insert(flat-node, "eyes")`.
2. `insert(flat-node, "eyes")`. `eyes ≺ flat`? Yes (`ey` before `fl`). So `flat.left = insert(elf-node, "eyes")`.
3. `insert(elf-node, "eyes")`. `eyes ≺ elf`? No (`ey` after `el`). `eyes ≻ elf`? Yes. So `elf.right = insert(null, "eyes")`.
4. `insert(null, "eyes")` hits the base case and returns a brand-new `BST` node holding `eyes`.
5. Unwinding: step 3 assigns that new node to `elf.right` and returns `elf`. Step 2 assigns the returned `elf` to `flat.left` (a harmless self-assignment, the link is unchanged) and returns `flat`. Step 1 assigns the returned `flat` to `dog.right` (again unchanged) and returns `dog`.

**Box-and-pointer reasoning in words:** only one pointer in the whole structure actually changes, `elf.right`, which went from `null` to an arrow pointing at the new `eyes` box. Every other assignment on the way back up writes the same arrow that was already there. This "reassign the link on the way back up even when nothing changed" pattern is exactly what makes the code so short: the caller doesn't have to know whether the recursive call created a new node, moved a node, or left things alone. It just stores whatever subtree root comes back.

**Why the "do nothing if found" case works:** if `ik` is neither `≺` nor `≻` `T.key`, both branches are skipped and we fall through to `return T` unchanged. No duplicate is created, consistent with the no-duplicate-keys rule.

The slide warns: "the code on the left is much cleaner than what you are likely to produce. HW6 will discuss in more detail." The reason it is clean is the `if (T == null) return new BST(ik);` base case, i.e. no arms-length recursion.

### Example 7: Deletion, all three cases

There are 3 cases, by number of children of the node being deleted.

Working tree:
```
        dog
      /     \
   bag       flat
   / \       /  \
 alf  cat  elf  glut
             \
             eyes
```

**Case 1: deletion key has no children.** Example: `delete("glut")`.

`glut` is a leaf. **Just sever the parent's link**: set `flat.right = null`. What happens to the `glut` node itself? It is **garbage collected**, since nothing references it any more.

**Case 2: deletion key has one child.** Example: `delete("flat")` from

```
        dog
      /     \
   bag       flat
   / \       /
 alf  cat  elf
             \
             eyes
```
(after `glut` was removed, `flat` has only a left child.)

Goal: maintain the BST property. The reasoning: `flat`'s child subtree is definitely entirely larger than `dog` (it sat in `dog`'s right subtree), so it is **safe to just move that child into `flat`'s spot**. Thus: **move `flat`'s parent's pointer to `flat`'s child**, i.e. `dog.right = elf-subtree`. `flat` will be garbage collected, along with its instance variables.

```
        dog
      /     \
   bag       elf
   / \         \
 alf  cat      eyes
```

**Case 3: deletion key has two children (Hibbard deletion).** Example: `delete("dog")` on the original tree.

Goal: find a new root node for this subtree. The replacement must be:
- `>` than everything in the left subtree, and
- `<` than everything in the right subtree.

Would `bag` work? **No.** `bag` is not greater than everything in the left subtree: `cat` lives in the left subtree and `cat ≻ bag`. (This is the slide's rhetorical question, and the answer is the point.)

The only two keys that satisfy both conditions are the **predecessor** and the **successor** of `dog`:
- predecessor = largest key in the left subtree = `cat`
- successor = smallest key in the right subtree = `elf`

**Hibbard deletion:** choose either one, delete that node from its current position, and stick a new copy of it in the root position.

Why is this guaranteed to terminate cleanly? Because **that second deletion is guaranteed to be case 1 or case 2**. The predecessor is the rightmost node of the left subtree, so by definition it has **no right child**; the successor is the leftmost node of the right subtree, so it has **no left child**. Either way, at most one child, so no recursive two-child case. (The slide poses "Why?" as a question; this is the answer.)

Promoting `cat`:
```
        cat
      /     \
   bag       flat
   /         /  \
 alf       elf  glut
             \
             eyes
```

### Example 8: the "Hard Challenge", delete `k`

```
                       k
                 /           \
              e                 v
            /   \             /   \
          b      g          p       y
         / \    /          /  \    /  \
        a   d  f          m    r  x    z
```

The challenge is posed before Hibbard deletion is explained ("Hard Challenge"), then revisited after ("Hopefully Now Easy"). `k` has two children, so it is case 3.

- **Predecessor of `k`** = largest key in `k`'s left subtree (rooted at `e`) = walk right as far as possible from `e`: `e → g → f`... careful, `g`'s only child is `f` on its **left**, so the rightmost node reachable from `e` is `g` itself. Predecessor = **`g`**.
- **Successor of `k`** = smallest key in `k`'s right subtree (rooted at `v`) = walk left as far as possible: `v → p → m`. Successor = **`m`**.

So there are **two solutions: either promote `g` or `m` to be in the root.** The slide shows the `g` solution:

```
                       g
                 /           \
              e                 v
            /   \             /   \
          b      f          p       y
         / \               /  \    /  \
        a   d             m    r  x    z
```

Walk through it: `g` moves to the root. `g` had one child, `f`, which must fill `g`'s vacated slot, so `f` becomes `e`'s right child. Everything else is untouched. Check the BST property at the new root: the left subtree is `{a, b, d, e, f}`, all `≺ g`. The right subtree is `{m, p, r, v, x, y, z}`, all `≻ g`. Valid.

(For the `m` solution: `m` goes to the root, and since `m` is a leaf its old slot simply becomes `null`, leaving `p` with only the right child `r`.)

### Example 9: the same tree as a Set and as a Map

As a **Set**:
```
          sumomo
        /        \
     momo        uchi
     /   \
   mo     no
```
Represents `{mo, no, sumomo, uchi, momo}`.

As a **Map** of word counts, each node stores a key/value pair:
```
          sumomo 1
         /        \
     momo 2       uchi 1
     /     \
  mo 2     no 1
```
Represents `sumomo→1, mo→2, momo→2, no→1, uchi→1`.

**What changed in the algorithms:** nothing. `find`, `insert`, and all three delete cases compare only on `.key`. The value is inert payload carried along in the node. `get(k)` is `find(k)` followed by reading `.value`; `put(k, v)` is `insert` with the extra rule that if the key is found you overwrite the value instead of doing nothing. *(the `get`/`put` phrasing in this paragraph is extra context; the lecture states only that nodes store key/value pairs)*

**What you cannot do efficiently:** look up by value. Finding all keys with value `= 1` requires iterating over **all** nodes, because the tree is only ordered by key. The lecture's position: this is fine.

---

## Common Pitfalls

1. **Forgetting that `extends` only gives you *accessible* members.** `RotatingLL` inherits all of `LinkedList`'s instance variables, but private ones cannot be touched from subclass code. The lecture's `TimeSeries` example makes this vivid: you inherit `TreeMap`'s `root`, but it is private, and IntelliJ yells at you if you try to use it.

2. **Thinking you must call the superclass constructor yourself.** You don't. If you do not manually invoke the superclass constructor with `super()`, it is called for you implicitly. This is what guarantees inherited state (like a sentinel node) is initialized.

3. **Using `implements` where `extends` is needed, or vice versa.** `implements` goes class → interface. `extends` goes class → class and interface → interface. `class RotatingLL implements LinkedList` is wrong because `LinkedList` is a class.

4. **Checking only the immediate children for the BST property.** The property is about **every key in the subtree**, not just the children. A node can satisfy "my left child is smaller and my right child is bigger" while some deeper descendant violates the property relative to an ancestor. Always check descendants against *all* ancestors.

5. **Assuming any binary tree with sorted-looking children is a BST.** The lecture's `debt` tree is a perfectly good binary tree and not a BST.

6. **Allowing duplicate keys.** The completeness requirement ("exactly one of `p ≺ q` and `q ≺ p` is true") forbids duplicates. In `insert`, the structure `if (ik ≺ ...) else if (ik ≻ ...)` with no `else` branch is precisely what makes a duplicate insert a no-op. If you write `else` instead of `else if (ik ≻ T.key)`, you will insert duplicates on the right.

7. **Arms-length recursion.** Do not write `if (T.left == null) T.left = new BST(ik);`. Let the recursion run off the end and use `if (T == null) return new BST(ik);`. The slides rank three versions of this mistake explicitly.

8. **Forgetting to reassign the link in `insert`.** Writing `insert(T.left, ik);` instead of `T.left = insert(T.left, ik);` loses the newly created node, since the base case communicates the new node only through its return value.

9. **In Hibbard deletion, picking the wrong replacement.** It must be the predecessor or successor. Any other node (like `bag` in the `delete("dog")` example) will violate the BST property. Also do not forget the second half of the operation: after copying the predecessor/successor's key up, you must still delete it from its original spot.

10. **Thinking the `log N` guarantee is unconditional.** `Θ(log N)` holds **if the tree is balanced/bushy**. The lecture is explicit that "bushiness" isn't formally defined yet, and the summary says `Θ(log N)` "in the worst case **if tree is balanced**." An unbalanced (stringy) BST degrades. *(the degenerate `Θ(N)` case and how to avoid it is the subject of later lectures: extra context)*

11. **Expecting value-based lookup from a Map.** There is no efficient reverse lookup. Scanning every node is the only way.

12. **Thinking a subclass object is two objects.** `new RotatingLL<>()` makes one object that contains both `RotatingLL`'s and `LinkedList`'s instance variables. There is no separate inner `LinkedList` object.

---

## Likely Exam Points

### 1. `extends` vs `implements`, and what gets inherited

**Q:** You are given `interface List`, `class LinkedList implements List`, and you want `RotatingLL` to support every `LinkedList` operation plus `rotateLeft()`. Write the class header, and state which of the following `RotatingLL` inherits: `LinkedList`'s private instance variables, its public methods, its nested classes, its constructors.

**A:** Header: `public class RotatingLL<Item> extends LinkedList<Item> { ... }`. It inherits all instance and static variables (including private ones, though those are inaccessible), all methods, and all nested classes. Constructors are **not** members you inherit, but the superclass constructor is invoked implicitly when you call `new RotatingLL()`.

### 2. Writing a small method on top of inherited ones

**Q:** Implement `rotateRight()` in `RotatingLL<Item> extends LinkedList<Item>`, moving the last item to the front. What is printed by `rsl.getFirst()` after calling it on `[10, 11, 12, 13]`?

**A:**
```java
public void rotateRight() {
    Item oldLast = removeLast();
    addFirst(oldLast);
}
```
List becomes `[13, 10, 11, 12]`, so `getFirst()` prints `13`. (Mirror image of the lecture's `rotateLeft`, which uses `removeFirst` then `addLast`.)

### 3. Is this a BST?

**Q:** Is the following a binary search tree? If not, name a violated node pair.

```
        debt
      /      \
    bus      ears
    / \      /   \
  axe  cow fish  gut
```

**A:** No. It is a valid binary tree but not a BST. `bus` is in `debt`'s left subtree but `bus ≻ debt` (`bu` after `de`). Also `axe` and `cow` under `bus`: `cow ≻ bus` is fine, but both sit in `debt`'s left subtree while `cow ≻ debt` is... `cow` vs `debt`: `c` before `d`, so that one is fine; the clear violation is `bus` vs `debt`. Similarly `fish` and `gut` in `ears`'s subtrees are not ordered relative to `ears` correctly (`fish ≻ ears`, yet it is `ears`'s left child).

### 4. Searching: trace and runtime

**Q:** In the tree below, list the nodes visited by `find(T, "glut")`, and state the worst-case runtime of `find` on a bushy BST with `N` nodes.

```
        dog
      /     \
   bag       flat
   / \       /  \
 alf  cat  elf  glut
```

**A:** Visits `dog` (`glut ≻ dog`, go right), `flat` (`glut ≻ flat`, go right), `glut` (equal, return). Three nodes. Worst case on a bushy BST: **`Θ(log N)`**, because the height is about `log₂(N)` and each level costs constant time. This was the clicker question, answer **A**.

### 5. Insert: trace and result

**Q:** Starting from the tree in Q4, show the result of `insert(T, "eyes")` and state exactly which pointer(s) in the structure change.

**A:** `eyes ≻ dog` → right to `flat`; `eyes ≺ flat` → left to `elf`; `eyes ≻ elf` → right to `null` → create node. Result: `eyes` becomes the right child of `elf`. Exactly one pointer changes value: `elf.right`, from `null` to the new node. The assignments `flat.left = ...` and `dog.right = ...` on the way back up rewrite the same arrows that were already there.

### 6. Arms-length recursion

**Q:** Identify the flaw and fix it:
```java
static BST insert(BST T, Key ik) {
    if (T.left == null)  { T.left  = new BST(ik); return T; }
    if (T.right == null) { T.right = new BST(ik); return T; }
    ...
}
```

**A:** This is **arms-length recursion**: it inspects one level below `T` instead of letting the recursion reach `null` itself. The base case is needlessly complicated (and the code is also plain wrong, since it ignores the ordering). Fix: `if (T == null) return new BST(ik);` as the base case, then `T.left = insert(T.left, ik);` / `T.right = insert(T.right, ik);` guarded by `≺` / `≻`.

### 7. Deletion, all three cases

**Q:** In the tree below, perform (a) `delete("glut")`, (b) then `delete("flat")`, (c) then `delete("dog")`. Draw each result and name the case.

```
        dog
      /     \
   bag       flat
   / \       /  \
 alf  cat  elf  glut
             \
             eyes
```

**A:**
(a) **Case 1 (no children).** `glut` is a leaf: sever `flat.right`. `glut` is garbage collected.
```
        dog
      /     \
   bag       flat
   / \       /
 alf  cat  elf
             \
             eyes
```
(b) **Case 2 (one child).** `flat` has only a left child, so point `flat`'s parent at `flat`'s child: `dog.right = elf`. `flat` is garbage collected.
```
        dog
      /     \
   bag       elf
   / \         \
 alf  cat      eyes
```
(c) **Case 3 (two children), Hibbard deletion.** Predecessor of `dog` = rightmost of left subtree = `cat`. Successor = leftmost of right subtree = `elf`. Promote either. Using the predecessor `cat`:
```
        cat
      /     \
   bag       elf
   /           \
 alf           eyes
```

### 8. Why Hibbard's second deletion is easy

**Q:** In Hibbard deletion, after you choose the predecessor or successor as the replacement, you must delete it from its original position. Why is that second deletion guaranteed never to be another two-child case?

**A:** The predecessor is the **rightmost** node of the left subtree, so it has no right child. The successor is the **leftmost** node of the right subtree, so it has no left child. Either way the node has at most one child, so the second deletion is always case 1 or case 2. No recursion into case 3 is possible.

### 9. The "promote g or m" challenge

**Q:** Delete `k` from the tree below. Give both valid replacement keys and draw the result for one of them.

```
                       k
                 /           \
              e                 v
            /   \             /   \
          b      g          p       y
         / \    /          /  \    /  \
        a   d  f          m    r  x    z
```

**A:** Two solutions: promote the predecessor **`g`** or the successor **`m`**. Promoting `g`: `g` becomes the root, and `g`'s single child `f` takes `g`'s old slot as `e`'s right child.
```
                       g
                 /           \
              e                 v
            /   \             /   \
          b      f          p       y
         / \               /  \    /  \
        a   d             m    r  x    z
```

### 10. The derivation from linked list to BST

**Q:** An `OrderedLinkedListSet` stores its items in sorted order, yet `contains` is still `Θ(N)`. Why doesn't sorting help, and what two modifications turn the list into a BST?

**A:** Sorting doesn't help because a linked list only permits one-step-at-a-time traversal, so you cannot exploit the order to skip ahead. Fixes: (1) move the entry point to the middle element, and (2) flip the links on the left half so the middle can reach leftward. That halves search time. Repeating this recursively within each half yields the binary search tree. (The alternative, optimization idea #1, is to add random express lanes, giving a **skip list**, not covered in 61B.)

### 11. Ordering requirements and duplicates

**Q:** The lecture requires the key ordering `≺` be complete, transitive, and antisymmetric. Which property rules out duplicate keys, and why?

**A:** Completeness: "exactly one of `p ≺ q` and `q ≺ p` is true." For two equal keys neither would hold, so equal keys cannot both exist in the tree. Hence no duplicates. The lecture notes most real-world implementations follow this rule too. It also notes `≺` is written instead of `<` because the ordering is arbitrary (you could order strings by length, for instance).

### 12. Sets vs. Maps

**Q:** You have a working BST-based `Set`. What is the minimum change needed to make it a `Map`, and what operation remains inefficient?

**A:** Have each node store a **key/value pair** instead of just a key; order and search on the key only. All the tree algorithms are unchanged. Looking up **by value** remains inefficient: finding all keys with a given value requires iterating over every node.

### 13. Runtime comparison table

**Q:** Fill in worst-case runtimes for `contains` and `add`: `ArraySet`, `OrderedLinkedListSet`, balanced BST.

**A:**

| Implementation | `contains` | `add` |
|---|---|---|
| `ArraySet` | `Θ(N)` | `Θ(N)` |
| `OrderedLinkedListSet` | `Θ(N)` | `Θ(N)` |
| BST (balanced/bushy) | `Θ(log N)` | `Θ(log N)` |

---

## Summary

- **`extends`** makes one class a hyponym of another class (`implements` is for class → interface; `extends` also works interface → interface). Both establish hypernym/hyponym relationships; `implements` is special because it crosses from abstract to concrete.
- A subclass **inherits all members** of its superclass: all instance and static variables, all methods, all nested classes. Private members are inherited but inaccessible.
- A subclass can **override** superclass methods and can use **`super`** to reach superclass methods/variables/constructors (not really used in 61B). The superclass constructor is called **implicitly** by the subclass constructor, ensuring inherited state (like a sentinel) is initialized.
- `RotatingLL<Item> extends LinkedList<Item>` needs only `rotateLeft() { Item oldFirst = removeFirst(); addLast(oldFirst); }`; everything else is inherited.
- `TimeSeries extends TreeMap<Integer, Double>` gets `put` and `get` for free (Project 4A), letting you build `plus` on top without knowing how the map works. Layers of abstraction: you write `TimeSeries`, Josh Bloch and Doug Lea wrote `TreeMap`.
- **ADTs** are defined by operations, not implementation: Disjoint Sets, Map, Set, List. Java provides these interfaces plus implementations (`ArrayList`, `LinkedList`, `HashSet`, `TreeSet`, `HashMap`, `TreeMap`).
- **BST derivation:** `ArraySet` (`Θ(N)`) → sorted linked list (still `Θ(N)`, since order alone doesn't help) → add express lanes (**skip list**, not covered) *or* move the entry point to the middle and flip the left links (halves search) → repeat recursively = **binary search tree**.
- **Tree** = nodes + edges with exactly one path between any two nodes. **Rooted** = one node designated root, every other node has exactly one parent (first node on the path to the root); childless nodes are **leaves**. **Rooted binary** = every node has 0, 1, or 2 children.
- **BST property:** for every node `X`, every key in the left subtree `≺ X.key` and every key in the right subtree `≻ X.key`. Applies to entire subtrees, not just children.
- The ordering `≺` must be complete, transitive, and antisymmetric, which implies **no duplicate keys**. `≺` is written instead of `<` to stress that the ordering is our arbitrary choice.
- **`find`:** equal → return; `≺` → go left; `≻` → go right; `null` → not found. `Θ(log N)` worst case on a bushy BST since height is about `log₂ N`. Bushy BSTs are astonishingly fast: `10^300000` items searchable in a second at 1 µs/op.
- **`insert`:** same search, with base case `if (T == null) return new BST(ik);` and reassignment `T.left = insert(T.left, ik)` on the way back up. Duplicates fall through and change nothing.
- **Avoid arms-length recursion.** The best base case is `T == null`, not `T.left == null` and certainly not `T.left.left == null`.
- **Deletion, 3 cases:** (1) no children → sever the parent's link, node is garbage collected; (2) one child → move the parent's pointer to the child; (3) two children → **Hibbard deletion**: replace with the **predecessor** (rightmost of left subtree) or **successor** (leftmost of right subtree), then delete that node, which is guaranteed to be case 1 or 2 because it has at most one child.
- **Sets and Maps are the same thing:** store key/value pairs in the nodes, order by key. No efficient lookup by value, and that's fine.
- **Big picture:** `ArraySet` is `Θ(N)` worst case; a BST is `Θ(log N)` worst case **if the tree is balanced**. Search and insert are straightforward (insert a little tricky); deletion is the challenging one.
