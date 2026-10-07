<!-- Mon, Oct 05, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 17: Sets, Extends, and BSTs

This lecture is really two lectures stapled together. The first half introduces Java's `extends` keyword, the tool for making one *class* a subclass of another *class* (as opposed to `implements`, which crosses from an interface to a concrete class). Using `extends`, you get every member of the superclass for free and only have to write the new behavior you want: `RotatingLL extends LinkedList` needs only a four-line `rotateLeft`, and Project 4A's `TimeSeries extends TreeMap<Integer, Double>` gets `put` and `get` without you knowing anything about how they work. The second half asks a deeper question: how do you build a *fast* Set? Starting from an `OrderedLinkedListSet` with Θ(N) `contains` and `add`, we repeatedly move the entry pointer to the middle and flip the left-pointing links, and that recursive process *derives* the binary search tree. We then formalize trees, rooted trees, rooted binary trees, and the BST property, and implement `contains` (search), `insert`, and deletion (including the two-child case, Hibbard deletion). The payoff: a "bushy" BST does these operations in Θ(log N) instead of Θ(N). The lecture closes by noting that a Map is the same data structure as a Set, just storing a key *and* a value in each node.

---

## Key Concepts

### 1. `extends`: giving an existing class new capabilities

Up to now you have used `implements` to say "this concrete class fulfills this interface" (e.g. `ArrayList<T> implements List<T>`). But what if you want one *class* to be a hyponym (subtype) of another *class*? That is what `extends` is for.

The motivating problem: build a `RotatingLL` that can do *everything* a `LinkedList` can do, plus one new operation `rotateLeft()` that moves the front item to the back. Given `[5, 9, 15, 22]`, `rotateLeft()` yields `[9, 15, 22, 5]`.

Without `extends`, `RotatingLL` is a brand new class with no methods: calling `rsl.addLast(10)` does not compile because `addLast`, `rotateLeft`, and `getFirst` are all missing. Writing `addLast`, `getFirst`, `removeFirst`, etc. yourself would be enormous duplicated work. Adding `extends LinkedList<Item>` to the class declaration tells the compiler "a `RotatingLL` **is a** `LinkedList`," and all those methods appear for free. Now the only thing left to write is `rotateLeft`.

**What exactly is inherited.** Because of `extends`, `RotatingLL` inherits *all members* of `LinkedList`:

- all instance and static variables,
- all methods,
- all nested classes.

Crucially: inherited does not mean *accessible*. Members may be `private`, in which case the subclass cannot touch them. Josh demonstrated this live: a `TreeMap` has a `root` variable, and a `TimeSeries extends TreeMap` technically inherits it, but IntelliJ reports "private access" if you try `usGDP.root`. The root exists; it is simply below your level of abstraction.

**Constructors are the exception.** Constructors are *not* inherited in the normal sense. If you want a custom `RotatingLL` constructor, you write your own. But if you do not manually invoke the superclass constructor with `super()`, Java calls it implicitly for you. This matters because the superclass constructor might need to set up something essential (e.g. a sentinel node); the implicit call guarantees that happens.

**`super`.** You can explicitly invoke superclass methods, variables, or constructors using `super`. Josh notes 61B will not really use `super`, though it may show up on discussion worksheets.

**Overriding.** A subclass can *override* a superclass method: if you do not like how `LinkedList` implements `addFirst`, you can provide your own version in `RotatingLL`.

### 2. `extends` vs. `implements`

Josh's view: "extends and implements are nearly exact synonyms." Both establish hypernym/hyponym (is-a) relationships, and in both cases the subtype inherits all members of the supertype. The difference is purely about *what kinds of types* are on each side:

| Keyword | Used between |
|---|---|
| `implements` | class → interface (concrete implementing abstract) |
| `extends` | class → class, **and** interface → interface |

So the Java Collections hierarchy looks like:

```
            Collection          (interface)
                | extends
               List             (interface)
      implements  /  \  implements
         ArrayList   LinkedList (classes)
                         | extends
                     RotatingLL (class)
```

Why two words? Josh's framing: `implements` marks something special, the moment you "cross the veil from the abstract to the concrete." `extends` stays within one world (abstract→abstract or concrete→concrete); `implements` is the single crossing point. He concedes a different language designer might have picked one word for both.

### 3. `extends` (is-a) vs. nested classes (has-a)

A student asked about this distinction. The answer:

- A **nested class** gives a class a *capability* or component it needs: `LinkedListDeque` has a `Node` class to store things; `ArrayDeque` has an `Iterator` class. This is a **has-a** relationship. Analogy: "a hand is a thing I have."
- **`extends`** expresses **is-a**. Analogy: "I am a human; I'm Josh Hug, a specific human."

Josh notes this distinction is subtle and people designing classes often get it wrong, hinting that Java's own `Stack` class contains such a mistake.

### 4. Project 4A: `TimeSeries extends TreeMap<Integer, Double>`

A `TimeSeries` tracks the value of some quantity over time, e.g. US GDP: 1990 → \$5.963 trillion, 1991 → \$6.158 trillion, and so on. The desired behavior is `put(year, value)` and `get(year)`.

You *could* design the whole data structure from scratch and implement `put` and `get`. But a map already associates values with keys, so instead:

```java
public class TimeSeries extends TreeMap<Integer, Double> { ... }
```

and `put`/`get` just work. (`HashMap` would also work; 4A uses `TreeMap`.)

Your actual job on 4A is to add the *new* operations that make a time series more than a map: `plus`, `dividedBy`, etc. Those make sense for time series specifically (adding US GDP to China GDP year by year) but would be awkward for a general map, since a general map could be from anything to anything.

This is the usual **layers of abstraction** story: you write `TimeSeries`, relying on Josh Bloch and Doug Lea (the Java Collections authors) to have implemented the underlying `TreeMap` operations correctly. You do not need to know anything about how `put` and `get` work. You can just trust them.

**Peeking under the hood.** Java's reflections library lets you bypass access modifiers at runtime, so you *can* get at the private `root` and see what a `TreeMap` actually contains. Josh showed the output: a structure with entries like `1993 : 6.858` plus mysterious `B`s and `R`s, because a Java `TreeMap` is a **red-black tree**. Understanding that structure is the goal of the next ~2.5 lectures of lecture time.

### 5. ADTs review, and the Set as today's target

An **Abstract Data Type (ADT)** is defined only by its *operations*, not by its implementation. The Deque ADT is a list of method signatures (`addFirst`, `addLast`, `isEmpty`, `size`, `printDeque`, `removeFirst`, `removeLast`, `get`) with specified behavior and nothing said about *how*. Slides write them with semicolons and no bodies to emphasize this.

You have already seen one ADT with multiple implementations several times over:

- `List61B` implemented by `AList` and `SLList`,
- `Deque61B` implemented by `ArrayDeque61B` and `LinkedListDeque61B`,
- `DisjointSets` implemented by `ListOfSetsDS`, `QuickFindDS`, `QuickUnionDS`, `WeightedQuickUnionDS` (a "richer set of possible implementations").

An aside Josh raised: `get` is arguably *bad form* in a Deque ADT, because a double-ended queue is supposed to give access only at the ends, not the middle. He included it purely as pedagogical practice at writing recursive methods, and disallowed it on the midterm because it opens the solution space too wide.

`java.util` provides both interfaces (ADTs: `List`, `Set`, `Map`, priority queues) and implementations (`LinkedList`, `ArrayList`, `HashSet`, `TreeSet`, `HashMap`, `TreeMap`). Today's target: the core idea behind **`TreeSet`** and **`TreeMap`**.

### 6. Deriving the BST from an ordered linked list

We already have one Set implementation, `ArraySet`, with a terrible runtime: `contains` is Θ(N) in the worst case, `add` is Θ(N) in the worst case. Why is this unavoidable with a plain array? To `add(7)` you must first scan the array to confirm 7 is not already present (sets have no duplicates), which is Θ(N). You might try keeping the array *sorted* so you could binary search, but then inserting is Θ(N) in the worst case, because a new smallest element must go at the front and everything must shift.

So Josh abandons arrays entirely for today (we will return to a clever non-ordering array approach, hashing, in a later lecture) and starts over from an **`OrderedLinkedListSet`**: a sentinel-headed linked list holding `A B C D E F G` in sorted order plus a `size` field.

- `contains`: Θ(N). To reach `F` you must walk `A, B, C, D, E, F`.
- `add`: Θ(N). You must walk the whole list to check for duplicates.

No improvement over `ArraySet` yet. **The fundamental problem is slow search even though the data is in order**, because a linked list only lets you take one step at a time.

**Optimization Idea #1: extra links ("express lanes").** Give `A` two outgoing links, one to `B` and one to `D`. During search, inspect both and decide which to follow. If you want something far ahead, jump. This is a real data structure, the **skip list**, and it has genuinely good runtime, but 61B will not discuss it.

**Optimization Idea #2: change the entry point, flip links, repeat recursively.** This is the path we take.

1. *Move the entry pointer to the middle* (to `D`). Now `A`, `B`, `C` are unreachable. Students suggested doubly-linking or wrapping around; Josh instead:
2. *Flip the left links*, so from `D` the chain goes `C → B → A` leftward and `E → F → G` rightward. Search time is **halved**: previously you walked from the start to the end; now you go at most halfway in one direction. Still Θ(N), since one direction is still a long chain.
3. *Do it again, recursively.* Make `D` point left at `B` (the middle of the left half) and right at `F` (the middle of the right half), flipping arrows again inside each half. Repeat within each sub-chain.

Drag the nodes into a nicer layout and you get:

```
            D
         /     \
        B       F
       / \     / \
      A   C   E   G
```

"What we've just invented is the binary search tree." You can think of a BST as an optimized ordered linked list where, knowing the structure in advance, you reorient the nodes so the things you want are fast to find.

### 7. Formal definitions: tree → rooted tree → rooted binary tree → BST

You earn the title "binary search tree" by satisfying four things in sequence: (1) you are a tree, (2) you are rooted, (3) you are binary, (4) you have the BST property.

**Tree.** A set of nodes plus a set of edges connecting them, with the constraint that **there is exactly one path between any two nodes**. (In CS70 you would get a formal graph definition; a tree is a kind of graph.) The lecture's non-examples were shapes with cycles: if from one node you can reach another either by going down *or* by going up and around, there are two paths, so it is not a tree.

**Rooted tree.** We *choose* one node to call the root. Any node in a tree is a valid choice. Having chosen it:
- every node N *except* the root has exactly one **parent**, defined as the first node on the path from N to the root;
- the root is the unique node with no parent;
- a node with no child is a **leaf** (there can be many);
- by convention we draw the root at the top, unlike (most) real trees.

A nice exercise Josh suggested: pick a different node as the root, redraw, and notice all parent/child relationships are *uniquely determined* by that choice.

**Rooted binary tree.** Every node has 0, 1, or 2 children (subtrees). The slide's third example, a node with three children, is "Not binary!"

**BST property.** For every node X in the tree:
- every key in X's **left** subtree is **less than** X's key,
- every key in X's **right** subtree is **greater than** X's key.

This is a *recursive* property: it must hold at every node, not just the root. It is not enough that a node's immediate children are correctly ordered.

Example from the slides. This **is** a BST:

```
            dog
        /         \
      bag         flat
     /   \       /    \
   alf   cat   elf   glut
```

This is a binary tree but **not** a BST:

```
            debt
        /          \
      bus          ears
     /   \        /    \
   axe   cow   fish    gut
```

The offending nodes are highlighted in red on the slide: `fish` sits in `ears`'s left subtree but `fish` > `ears` alphabetically. (`bus`/`axe`/`cow` and `gut` are consistent locally; the violation is on the right side.)

**Properties of the ordering.** The ordering must be complete, transitive, and antisymmetric. For keys p and q:
- exactly one of p ≺ q and q ≺ p is true (complete + antisymmetric),
- p ≺ q and q ≺ r imply p ≺ r (transitive).

The squiggly `≺` is used instead of `<` to remind us the ordering is *arbitrary*: you could define `≺` by string length instead of alphabetical order. (Josh noted that if you did, you could not then store two strings of equal length, which is a little weird.) "Complete" matters because you can imagine objects with no sensible relation ("how am I related to this rock?"); everything in your BST must be mutually comparable.

**Consequence: no duplicate keys allowed.** If exactly one of p ≺ q and q ≺ p must hold, p and q can never be "equal but distinct." This keeps things simple, matches Set and Map semantics (neither allows duplicate keys), and is what most real-world implementations do. It is not hard to extend BSTs to allow duplicates, but 61B will not.

### 8. Search (`contains`) and why bushy BSTs are fast

To find `searchKey` in tree `T`: if `searchKey` equals `T.key`, return. If `searchKey ≺ T.key`, search `T.left`. If `searchKey ≻ T.key`, search `T.right`. Falling off the bottom (`T == null`) means not found.

**Runtime on a "bushy" BST: Θ(log N) in the worst case.** The reasoning: you examine exactly one node per level, and a bushy tree of N nodes has about log₂(N) levels. A 15-node bushy tree needs at most 4 node inspections. "Bushy" is used here as an *intuitive* concept (the tree is wide, not a long spindly chain); it is not yet formally defined, and effectively means "logarithmically many levels." A non-bushy tree, like one that degenerates into a single long chain, does *not* get this runtime.

How fast is this really? At 1 microsecond per level, you could find something in a bushy tree of 10^300000 items in **one second**, a size far beyond the number of atoms in the universe. Since much (perhaps most) of computation is dedicated to finding things in response to queries, it is a very good thing that such queries are essentially free.

### 9. Insert

Search for the key.
- If found, do nothing (no duplicates).
- If not found: create a new node and set the appropriate link.

The key insight, which a student supplied: a given key has **exactly one** natural home in the tree. That is the same fact search relies on. Inserting `eyes` into the `dog` tree: `eyes` ≻ `dog` → right to `flat`; `eyes` ≺ `flat` → left to `elf`; `eyes` ≻ `elf` → right of `elf`, which is empty. Done.

```
            dog
        /         \
      bag         flat
     /   \       /    \
   alf   cat   elf   glut
                 \
                eyes
```

**Avoid arms-length recursion.** This is flagged as "a common rookie bad habit." Arms-length recursion means handling the base case one step *too early*, inspecting children (or grandchildren) instead of letting the recursion carry you to the null pointer. The slides show a progression:

- Worst: checking `T.left.left == null`, `T.left.right == null`, `T.right.left == null`, `T.right.right == null`. "This base case is too complicated. The recursion can take us further."
- Better but still not best: `if (T.left == null) T.left = new BST(ik); else if (T.right == null) ...`
- **The best base case:** `if (T == null) return new BST(ik);`

### 10. Deletion: three cases, and Hibbard deletion

Deletion is "a really rich and interesting puzzle." Three cases, by number of children of the node being deleted.

**Case 1: no children** (e.g. `glut`, `cat`, `eyes` in the tree above). Just sever the parent's link (set it to `null`). The orphaned node is **garbage collected** automatically, since nothing points at it anymore.

**Case 2: one child** (e.g. `flat`, whose only child is `elf`). Move `flat`'s *parent's* pointer to point to `flat`'s *child*. Why is this safe? Because `flat` was in `dog`'s right subtree, everything in `flat`'s subtree, including `elf`, is definitely greater than `dog`, so hanging `elf` directly off `dog`'s right link preserves the BST property. Concretely, the only reassignment needed is `dog.right = elf`. You do *not* need to clear `flat`'s own left/right pointers; the whole `flat` object will be garbage collected regardless, since nothing points at it.

Note (from a student question): if `flat` had *both* a left and a right child, this procedure does not apply and you must use the two-child procedure.

**Case 3: two children (Hibbard deletion).** Example: `delete("dog")` from

```
            dog
        /         \
      bag         flat
     /   \       /    \
   alf   cat   elf   glut
                 \
                eyes
```

We need a new root node, and it must be **greater than everything in the left subtree** and **less than everything in the right subtree**.

Would `bag` work? No, because `bag` is *not* greater than everything in the left subtree: `cat` is in there, and `cat` ≻ `bag`. Promoting `bag` or `flat` simply does not satisfy the requirement.

The two keys that *do* satisfy it are:
- the **predecessor**: the largest key in the left subtree (here `cat`),
- the **successor**: the smallest key in the right subtree (here `elf`).

`cat` is bigger than everything in the left subtree and smaller than everything in the right subtree, so it is a legal root. Same for `elf`.

**Hibbard deletion**: copy the predecessor's (or successor's) key into the node being deleted, then delete the predecessor (or successor) node from its old position.

**Why is that recursive deletion easy?** Because the predecessor/successor is *guaranteed* to be Case 1 or Case 2, never Case 3. Reasoning from lecture: if the predecessor had two children, it would have a child larger than itself in the left subtree, contradicting that it is the *largest* key in the left subtree. (Symmetrically, the successor cannot have a left child.) So it has at most one child, and we already know how to handle that.

---

## Definitions

- **`extends`**: the Java keyword declaring that one class is a subclass (hyponym) of another class, or that one interface is a subinterface of another. The subclass inherits all members of the superclass.
- **`implements`**: the Java keyword declaring that a concrete class fulfills an interface. Used only at the boundary from abstract to concrete.
- **Member (of a class)**: in this lecture's usage, all instance and static variables, all methods, and all nested classes.
- **Inheritance**: a subclass automatically receives all members of its superclass. Inherited members that are `private` are still inaccessible from the subclass.
- **Override**: define a method in a subclass with the same signature as one in the superclass, replacing the superclass's behavior.
- **`super`**: keyword used to explicitly invoke a superclass's methods, variables, or constructors. Not really used in 61B.
- **Implicit superclass constructor call**: if a subclass constructor does not manually call `super(...)`, Java inserts a call to the superclass's no-argument constructor, ensuring superclass setup (e.g. sentinel creation) happens.
- **Hypernym / hyponym**: supertype / subtype. Both `extends` and `implements` establish such a relationship.
- **Is-a vs. has-a**: `extends` expresses is-a (a `RotatingLL` *is a* `LinkedList`); a nested class expresses has-a (a `LinkedListDeque` *has a* `Node` class).
- **Abstract Data Type (ADT)**: a data type defined only by its operations and their specified behavior, not by any implementation.
- **`TimeSeries` (Project 4A)**: a class tracking values of some quantity over time, implemented as `extends TreeMap<Integer, Double>` with extra operations such as `plus`.
- **Tree**: a set of nodes plus a set of edges connecting them, such that there is exactly one path between any two nodes.
- **Rooted tree**: a tree in which one node has been designated the root.
- **Parent (of node N)**: the first node on the path from N to the root. Every node except the root has exactly one parent.
- **Root**: the unique node with no parent; conventionally drawn at the top.
- **Leaf**: a node with no children.
- **Rooted binary tree**: a rooted tree in which every node has 0, 1, or 2 children (subtrees).
- **BST property**: for every node X, every key in X's left subtree is less than X's key and every key in X's right subtree is greater than X's key.
- **Binary search tree (BST)**: a rooted binary tree with the BST property.
- **`≺`**: notation for the ordering relation, written with a squiggle to emphasize the order is arbitrary (it need not be numeric or alphabetical). Must be complete, transitive, and antisymmetric.
- **Complete + antisymmetric ordering**: for all keys p, q, exactly one of p ≺ q and q ≺ p holds. Implies no duplicate keys.
- **Transitive ordering**: p ≺ q and q ≺ r imply p ≺ r.
- **Bushy**: an intuitive (not yet formally defined in this lecture) term for a BST that is wide rather than spindly, i.e. has about log N levels.
- **Predecessor (of a node in a BST)**: the largest key in that node's left subtree.
- **Successor (of a node in a BST)**: the smallest key in that node's right subtree.
- **Hibbard deletion**: the strategy for deleting a two-child node: replace its key with its predecessor's or successor's key, then delete that predecessor/successor node (guaranteed to be a 0-child or 1-child deletion).
- **Arms-length recursion**: the bad habit of writing a base case that handles the situation one step too early (inspecting children rather than recursing into them until you hit `null`).
- **Skip list**: a linked structure augmented with "express lane" links allowing jumps; mentioned as a real, fast alternative but not covered in 61B.
- **Reflections library**: a Java library allowing access modifiers to be ignored at runtime, letting you reach private members such as a `TreeMap`'s `root`. Not recommended.
- **Red-black tree**: the structure Java's `TreeMap` actually uses. Named only as a preview of upcoming lectures.

---

## Worked Examples

### Example 1: `RotatingLL`, step by step

**Attempt 1, the version that does not compile:**

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

This does not compile. `RotatingLL` is a fresh class with no methods at all, so `addLast`, `rotateLeft`, and `getFirst` are undefined. IntelliJ flags all three.

**Attempt 2, add `extends`:**

```java
public class RotatingLL<Item> extends LinkedList<Item> {
    public static void main(String[] args) {
        RotatingLL<Integer> rsl = new RotatingLL<>();
        rsl.addLast(10);
        rsl.addLast(11);
        rsl.addLast(12);
        rsl.addLast(13);

        rsl.rotateLeft();
        System.out.println(rsl.getFirst()); // print 11
    }
}
```

Now the compiler knows a `RotatingLL` **is a** `LinkedList`, so `RotatingLL` inherits `addLast` and `getFirst` from `LinkedList`. Those two calls compile. `rotateLeft` is still missing, because `LinkedList` has no such method.

**Attempt 3, write `rotateLeft` using only inherited methods:**

```java
public class RotatingLL<Item> extends LinkedList<Item> {
    /** Rotates list to the left. */
    public void rotateLeft() {
        Item oldFirst = removeFirst();
        addLast(oldFirst);
    }
}
```

Why this shape? Inside `rotateLeft` you **cannot access `LinkedList`'s instance variables**: they are private, and you do not even know what they are. This is exactly the constraint you have practiced with default methods in `List61B` and with the `replaceWithLast` problem on the midterm: you can only use the *other public methods*. Luckily "rotate left" has a clean formulation in terms of those methods: remove the front item, then append it to the back. (It could be written as a single line; the slides split it into two for clarity.)

**Box-and-pointer / environment reasoning in words.** `rsl` is a reference to a single heap object. Because of `extends`, that object physically contains the `LinkedList` machinery (whatever sentinel nodes and `size` field `LinkedList` uses) *plus* whatever `RotatingLL` adds. When `rotateLeft` calls `removeFirst()`, there is no separate `LinkedList` object being consulted; the call runs `LinkedList`'s code against *this same object's* fields, splicing out the node holding 10 and decrementing the size. `oldFirst` is a local reference to the boxed `Integer` 10. `addLast(oldFirst)` then runs `LinkedList`'s code again on the same object, creating a new node at the back pointing at that same `Integer` 10. The list goes `[10, 11, 12, 13]` → (after `removeFirst`) `[11, 12, 13]` → (after `addLast`) `[11, 12, 13, 10]`. Then `getFirst()` returns 11.

Note on constructors: `new RotatingLL<>()` runs `RotatingLL`'s default constructor, which implicitly calls `LinkedList`'s constructor. That is what guarantees any sentinel nodes `LinkedList` needs actually exist before `addLast` is ever called.

### Example 2: `TimeSeries` for Project 4A

**The target behavior:**

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

As written this fails: IntelliJ reports "cannot resolve method `put` in `TimeSeries`." You could stop and write `void put(int year, double v)` and a matching `get` yourself, maintaining your own storage. But a map already does precisely this job: associate values with keys. So:

```java
public class TimeSeries extends TreeMap<Integer, Double> {
    public static void main(String[] args) {
        TimeSeries usGDP = new TimeSeries();
        usGDP.put(1990, 5.963);
        // ... etc ...
        System.out.println(usGDP.get(1990));
    }
}
```

The key type is `Integer` (the year) and the value type is `Double` (the quantity). Now `put` and `get` are inherited and everything compiles. (`HashMap` would work too; 4A uses `TreeMap`.)

**What you actually add on 4A.** At this point `TimeSeries` is "silly," just a map with a new name. The value comes from the methods you add, such as:

```java
TimeSeries usGDP = new TimeSeries();
TimeSeries chinaGDP = new TimeSeries();
...
System.out.println(usGDP.plus(chinaGDP));
```

`plus` makes sense for time series (combine two series year by year) but would be awkward on a general `Map`, whose keys and values could be anything. The nice part is you write `plus` *without knowing anything about how `put` and `get` work*. You just trust them. That is the layers-of-abstraction picture: `TimeSeries` sits on top of `TreeMap`, and `TreeMap`'s authors handle the inside.

**The private-member demonstration.** `TimeSeries` technically inherits every member of `TreeMap`, including a variable called `root`. But:

```java
System.out.println(usGDP.root);   // compile error: root has private access
```

The root is there, holding the whole tree, but it is below the level of abstraction and you are not allowed to touch it. (You can discover the *public* members either by typing `usGDP.` and reading IntelliJ's autocomplete list, or by reading the documentation. `root` does not appear in that list.) Using the reflections library you can strip the `private` modifier at runtime and print the tree; what emerges is a structure with entries like `1993 : 6.858` annotated with `B`s and `R`s, because a `TreeMap` is a red-black tree.

### Example 3: BST search (`find`)

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

(This is pseudo-Java: `≺` stands for whatever comparison you are using. Your lab code will differ since you will not have "naked recursive trees.")

What it does: returns the *node* containing the key you are looking for, or `null` if the key is absent. Falling off the tree (`T == null`) means not found.

**Trace: `find(dog-tree, "cat")`** on

```
            dog
        /         \
      bag         flat
     /   \       /    \
   alf   cat   elf   glut
```

1. `T` = `dog`. Not null. `"cat".equals("dog")`? No. Is `cat ≺ dog`? Yes (alphabetically). Recurse on `T.left`, the `bag` node.
2. `T` = `bag`. Not null. `"cat".equals("bag")`? No. Is `cat ≺ bag`? No, `cat ≻ bag`. Recurse on `T.right`, the `cat` node.
3. `T` = `cat`. `"cat".equals("cat")`? Yes. Return this node.

Three node visits in a 7-node tree. Note the recursion is elegantly simple here; the lecture observes this is how most search tree implementations are written, and is much easier than an iterative version.

**Why Θ(log N) on a bushy tree.** Each recursive call descends exactly one level and inspects exactly one node. A bushy tree with N nodes has ~log₂(N) levels. For N = 15, at most 4 inspections. The constant work per level times log N levels gives Θ(log N).

### Example 4: BST insert

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

The slides call this out as "much cleaner than what you are likely to produce"; HW6 discusses it in more detail.

**How to read it.** `insert` returns *the root of the (possibly modified) subtree it was given*. That is the trick that makes it clean:

- If handed `null`, it returns a brand new one-node tree. That new node is the return value.
- Otherwise it recursively inserts into the correct side and **reassigns** `T.left` or `T.right` to the result. In the common case the result is the same node that was already there, so the assignment is a harmless no-op; in the one case where recursion bottomed out at `null`, the assignment is exactly what wires the new node into the tree.
- If `ik` equals `T.key` (neither `≺` nor `≻`), both branches are skipped and `T` is returned unchanged: duplicates are silently ignored, as required for a Set.
- Finally it returns `T` so the caller one level up can reattach it.

**Trace: `insert(dog-tree, "eyes")`**

1. `T` = `dog`. Not null. `eyes ≺ dog`? No. `eyes ≻ dog`? Yes. So `dog.right = insert(flat-node, "eyes")`.
2. `T` = `flat`. `eyes ≺ flat`? Yes. So `flat.left = insert(elf-node, "eyes")`.
3. `T` = `elf`. `eyes ≺ elf`? No. `eyes ≻ elf`? Yes. So `elf.right = insert(null, "eyes")`.
4. `T` = `null`. Return `new BST("eyes")`.
5. Unwinding: step 3 sets `elf.right` to the new `eyes` node (previously `null`): the actual structural change. Returns `elf`.
6. Step 2 sets `flat.left = elf`, which it already was. Returns `flat`.
7. Step 1 sets `dog.right = flat`, which it already was. Returns `dog`.

Result:

```
            dog
        /         \
      bag         flat
     /   \       /    \
   alf   cat   elf   glut
                 \
                eyes
```

**In box-and-pointer terms**, exactly one new heap object (the `eyes` node, with `null` left and right) was allocated, and exactly one existing pointer (`elf.right`) was changed from `null` to point at it. Steps 6 and 7 rewrote pointers with their own current values, which is why the "redundant" assignments are safe.

**The arms-length anti-pattern to avoid:**

```java
// BAD: arms-length recursion
if (T.left == null)
    T.left = new BST(ik);
else if (T.right == null)
    T.right = new BST(ik);
```

This checks the children for emptiness instead of letting the recursion walk off the end, and it is not even correct (it ignores ordering). Worse still is checking grandchildren (`T.left.left == null`, etc.), which the slides label "too complicated: the recursion can take us further." The best base case is simply `if (T == null) return new BST(ik);`.

### Example 5: Deletion, all three cases

Working tree:

```
            dog
        /         \
      bag         flat
     /   \       /    \
   alf   cat   elf   glut
                 \
                eyes
```

**Case 1, `delete("glut")` (no children).** Sever the parent's link: `flat.right = null`. Nothing now references the `glut` node, so it is garbage collected.

**Case 2, `delete("flat")` (one child).** First suppose `glut` has already been removed, so `flat`'s only child is `elf`. Why is the fix safe? `flat` sits in `dog`'s right subtree, so *everything* under `flat`, including `elf` and `eyes`, is greater than `dog`. Reattaching that subtree directly to `dog.right` cannot violate the BST property. So the single reassignment is:

```
dog.right = elf
```

You do **not** need to null out `flat.left` or `flat.right`. Those are just 64-bit addresses inside an object that nothing points to anymore; the garbage collector will reclaim the `flat` node (along with its instance variables) regardless. Result:

```
            dog
        /         \
      bag         elf
     /   \            \
   alf   cat         eyes
```

**Case 3, `delete("dog")` (two children), Hibbard deletion.** Back to the original tree. We need a new root that is greater than everything in the left subtree (`bag`, `alf`, `cat`) and less than everything in the right subtree (`flat`, `elf`, `eyes`, `glut`).

- `bag` fails: `cat` is in the left subtree and `cat ≻ bag`.
- `flat` fails symmetrically.
- The **predecessor** `cat` (largest in the left subtree) works: it is greater than `alf` and `bag`, and less than every key on the right.
- The **successor** `elf` (smallest in the right subtree) also works.

Pick one, say `cat`. Copy `cat` into the root position, then delete the old `cat` node. That second deletion is guaranteed to be Case 1 or Case 2, so we already know how to do it. Here `cat` is a leaf, so Case 1: `bag.right = null`.

```
            cat
        /         \
      bag         flat
     /           /    \
   alf         elf   glut
                 \
                eyes
```

**Why the predecessor/successor can never be a two-child node.** Suppose the predecessor (largest key in the left subtree) had a right child. That child would be larger than the predecessor and still in the left subtree, contradicting "largest in the left subtree." So the predecessor has no right child, hence at most one child. Symmetrically, the successor has no left child.

### Example 6: The "hard challenge," `delete(k)`

```
                           k
                  /                 \
             e                           v
          /     \                    /       \
        b         g               p             y
       / \       /               /             / \
      a   d     f               m             x   z
                                  \
                                   r
```

(The slide's layout: root `k`; left child `e` with children `b` and `g`; `b` has children `a`, `d`; `g` has child `f`; right child `v` with children `p` and `y`; `p` has children `m` and `r`; `y` has children `x`, `z`.)

`k` has two children, so Hibbard applies. There are exactly **two** good choices:

- the predecessor **`g`**, the largest key in `k`'s left subtree,
- the successor **`m`**, the smallest key in `k`'s right subtree.

Taking `g`: promote `g` to the root, then delete the old `g` node. `g` had one child (`f`), so this is Case 2: point `g`'s former parent `e` at `f`.

Result:

```
                           g
                  /                 \
             e                           v
          /     \                    /       \
        b         f               p             y
       / \                       /             / \
      a   d                     m             x   z
                                  \
                                   r
```

Promoting `m` instead would be equally valid.

### Example 7: Sets and Maps are the same data structure

Consider the BST

```
          sumomo
        /        \
    momo         uchi
    /    \
  mo      no
```

As a **Set**, this represents `{mo, no, sumomo, uchi, momo}`.

Now suppose you want word *counts*: `sumomo → 1, mo → 2, momo → 2, no → 1, uchi → 1`. How do you change the data structure? **You do not touch the tree structure at all.** Each node simply stores a key/value *pair* instead of just a key:

```
          sumomo 1
         /         \
    momo 2         uchi 1
    /      \
  mo 2      no 1
```

Search, insert, and delete all still navigate using the key only; the value just rides along. So a Map is a Set whose nodes carry an extra field. When you build a node, you supply both the key type and the value type.

**One limitation:** there is no efficient way to look up by *value*. You cannot find all keys with value 1 without iterating over **all** nodes, since the tree is organized by key. The lecture notes this is fine, it is simply not what a map is for.

---

## Common Pitfalls

1. **Thinking "inherited" means "accessible."** `RotatingLL` inherits all of `LinkedList`'s instance variables, and `TimeSeries` inherits `TreeMap`'s `root`, but private members cannot be touched from the subclass. Inside `rotateLeft` you must build behavior out of *public methods only*.
2. **Expecting constructors to be inherited.** They are not. If you want a `RotatingLL` constructor, write it. The subclass's default constructor does *implicitly* call the superclass constructor, which is why sentinel setup still happens.
3. **Using `extends` where you mean `implements` (or vice versa).** `implements` goes class → interface. `extends` goes class → class or interface → interface.
4. **Confusing is-a with has-a.** Using `extends` to grab a capability (has-a) rather than to express a genuine subtype relationship. Josh notes Java's own `Stack` class got this wrong.
5. **Thinking a sorted array fixes `ArraySet`.** Sorting makes search faster but `add` is still Θ(N) worst case, because inserting a new minimum forces everything to shift.
6. **Checking only immediate children for the BST property.** The property is recursive: *every* key in the left subtree must be less than X, not just X's left child. The `debt` tree is the counterexample: `fish` ≻ `ears` yet sits under `ears` on the left.
7. **Forgetting that the BST property forbids duplicates.** It follows from "exactly one of p ≺ q and q ≺ p." `insert` of an existing key must do nothing.
8. **Assuming all BSTs are Θ(log N).** The Θ(log N) result requires the tree to be **bushy**. A spindly tree, in the extreme a single chain, is not, and gives Θ(N). (Making trees stay bushy is the subject of the next lectures.)
9. **Arms-length recursion in `insert`.** Checking `T.left == null` or worse `T.left.left == null` instead of letting recursion reach `T == null`. Flagged explicitly as a rookie habit.
10. **Forgetting to reassign in `insert`.** `insert(T.left, ik)` alone does nothing visible; you need `T.left = insert(T.left, ik)` and `return T` so the structure gets wired up.
11. **Promoting the wrong node in two-child deletion.** The new root must be the predecessor or successor, *not* the deleted node's left or right child. Promoting `bag` in the `dog` tree breaks the BST property because `cat ≻ bag` is in `bag`'s own subtree.
12. **Applying the one-child procedure to a two-child node.** If `flat` had both a left and a right child, you cannot just hand your position to one child; you must use Hibbard.
13. **Worrying about cleaning up the deleted node's pointers.** Unnecessary. Once nothing references the node, the garbage collector reclaims it and its instance variables.
14. **Expecting value-based lookup in a BST-backed Map.** The tree is keyed; finding all keys with a given value requires visiting every node.

---

## Likely Exam Points

**1. `extends` vs. `implements`, and what is inherited.**

*Q:* You write `public class MyList<T> extends ArrayList<T>`. `ArrayList` has a private field `size` and a public method `size()`. Inside a new method in `MyList`, can you write `size = size + 1`? Can you write `int n = size();`? Does `MyList` inherit `ArrayList`'s constructors?

*A:* `size = size + 1` does **not** compile: the field is inherited but `private`, so inaccessible. `int n = size();` compiles fine: the public method is inherited and accessible. Constructors are not inherited; however, `MyList`'s implicit default constructor calls `ArrayList`'s constructor automatically, so you do not need to call `super()` yourself.

**2. Writing a method using only inherited public methods.**

*Q:* Add `rotateRight()` to `RotatingLL` (moves the *last* item to the front), using only `LinkedList` methods.

*A:*
```java
public void rotateRight() {
    Item oldLast = removeLast();
    addFirst(oldLast);
}
```
Exactly the mirror image of `rotateLeft`, and again it must go through public methods because `LinkedList`'s internals are private. *(extra context: this exact method was not in the lecture; `rotateLeft` was.)*

**3. Identifying whether a structure is a tree / binary tree / BST.**

*Q:* Is the following a BST? If not, name the violating node(s).
```
            debt
        /          \
      bus          ears
     /   \        /    \
   axe   cow   fish    gut
```
*A:* No. It is a rooted binary tree, but `fish` violates the BST property: `fish` lies in `ears`'s left subtree yet `fish ≻ ears` alphabetically. (The left half, `bus` with `axe` and `cow`, and the right child `gut`, are all fine.)

**4. BST search runtime.**

*Q:* What is the worst-case runtime of a single `contains` on a bushy BST with N nodes? What if the BST is a single long chain?

*A:* Θ(log N) on a bushy BST: you examine one node per level and there are ~log₂(N) levels. On a chain it degenerates to Θ(N), since there are N levels. The Θ(log N) claim depends entirely on bushiness.

**5. Number of comparisons / path to a key.**

*Q:* In the tree below, list the nodes visited by `find(T, "cat")` and state how many node inspections occur.
```
            dog
        /         \
      bag         flat
     /   \       /    \
   alf   cat   elf   glut
```
*A:* `dog`, then `bag` (since `cat ≺ dog`), then `cat` (since `cat ≻ bag`). Three inspections.

**6. Insertion position.**

*Q:* Insert `eyes` into the tree above. Where does it go, and how many places could it legally go?

*A:* Exactly one place: as `elf`'s right child. Trace: `eyes ≻ dog` → right to `flat`; `eyes ≺ flat` → left to `elf`; `eyes ≻ elf` → right of `elf`, which is empty. The uniqueness of this position is precisely the fact that makes search work.

**7. Recognizing and fixing arms-length recursion.**

*Q:* What is wrong with this base case in a BST `insert`, and what is the correct base case?
```java
if (T.left == null)
    T.left = new BST(ik);
else if (T.right == null)
    T.right = new BST(ik);
```
*A:* This is arms-length recursion: it stops one level too early by inspecting the children rather than letting the recursion walk off the bottom of the tree (and it also ignores key ordering). The correct base case is `if (T == null) return new BST(ik);`.

**8. Deletion, all three cases.** The slides note deletion setup-and-finish problems have appeared on past exams.

*Q:* Given
```
            dog
        /         \
      bag         flat
     /   \       /    \
   alf   cat   elf   glut
                 \
                eyes
```
describe the pointer changes for (a) `delete("glut")`, (b) `delete("alf")`, (c) `delete("elf")`.

*A:* (a) `glut` has no children: set `flat.right = null`; the node is garbage collected. (b) `alf` has no children: set `bag.left = null`. (c) `elf` has one child (`eyes`): set `flat.left = eyes`. This is safe because everything under `elf` is already known to be greater than `dog` and less than `flat`.

**9. Hibbard deletion: naming the valid replacements.**

*Q:* In the tree of question 8, `delete("dog")`. What are the two legal replacement keys, and why are `bag` and `flat` not among them? Show the result for one choice.

*A:* The legal choices are the predecessor `cat` (largest key in the left subtree) and the successor `elf` (smallest key in the right subtree). `bag` fails because `cat` lies in `bag`'s own subtree and `cat ≻ bag`, so promoting `bag` would violate "everything in the left subtree is less than the root"; `flat` fails symmetrically. Promoting `cat`: copy `cat` into the root, then delete the old `cat` node, which is a leaf, so `bag.right = null`. Result: root `cat`, left child `bag` (with left child `alf`), right child `flat` (with children `elf`, whose right child is `eyes`, and `glut`).

**10. Why Hibbard's recursive deletion is easy.**

*Q:* Prove that the predecessor of a node can never have two children.

*A:* The predecessor is the largest key in the node's left subtree. If it had a right child, that child would be greater than the predecessor and still inside that same left subtree, contradicting maximality. So the predecessor has no right child, hence at most one child, so deleting it is always Case 1 or Case 2. (Symmetric argument: the successor has no left child.)

**11. Sets vs. Maps.**

*Q:* You have a BST-based Set. What minimal change turns it into a Map? Can you then efficiently find all keys mapping to a given value?

*A:* Store a key *and* a value in each node; the tree shape and all algorithms are unchanged, since navigation uses only keys. You cannot efficiently search by value: finding all keys with a given value requires iterating over every node, because the tree is organized by key.

**12. ADT vs. implementation.**

*Q:* Name two implementations of the Set ADT seen so far and give their worst-case runtimes.

*A:* `ArraySet`: Θ(N) for `contains` and `add` in the worst case. BST-based set (`TreeSet`-style): Θ(log N) per operation if the tree is balanced/bushy.

**13. Deriving the BST.**

*Q:* Starting from an ordered linked list, what two transformations, repeated recursively, produce a BST? What does one application of them buy you?

*A:* (i) move the entry pointer to the middle element, and (ii) flip the links in the left portion so they point leftward from the middle. One application halves the search time (you traverse at most half the list), though it is still Θ(N); applying it recursively within each half yields the binary search tree, with Θ(log N) search when bushy.

---

## Summary

- **`extends`** makes one class a subclass of another (or one interface a subinterface of another); **`implements`** is reserved for the crossing from abstract (interface) to concrete (class). Both establish is-a / hypernym-hyponym relationships and both cause the subtype to inherit all members.
- A subclass inherits **all instance and static variables, all methods, and all nested classes**, but `private` members remain inaccessible. Constructors are not inherited, though the superclass constructor is called implicitly (ensuring e.g. sentinels get set up). `super` can invoke superclass members explicitly; 61B will not really use it. Methods can be overridden.
- `extends` is **is-a**; a nested class is **has-a**. This distinction is subtle and often gotten wrong in real class design.
- `RotatingLL extends LinkedList` needs only `rotateLeft()`, implemented as `Item oldFirst = removeFirst(); addLast(oldFirst);`, using public methods because the internals are private.
- Project 4A: `TimeSeries extends TreeMap<Integer, Double>` inherits `put` and `get` for free; your job is to add operations like `plus` and `dividedBy`. You trust `TreeMap` completely, a standard layers-of-abstraction arrangement. `TreeMap`'s private `root` is inaccessible (reachable only via the reflections library), and the underlying structure is a **red-black tree**, the topic of the next lectures.
- An **ADT** is defined by operations alone. `java.util` supplies interfaces (`List`, `Set`, `Map`) and implementations (`ArrayList`, `LinkedList`, `HashSet`, `TreeSet`, `HashMap`, `TreeMap`).
- `ArraySet` gives Θ(N) `contains` and `add` in the worst case, and sorting the array does not fix `add`. An `OrderedLinkedListSet` is also Θ(N) for both.
- The BST is **derived** from an ordered linked list by moving the entry point to the middle and flipping the left links (halving search time), then repeating recursively. The alternative "express lane" idea gives a **skip list**, real and fast but not covered.
- A **tree** is nodes plus edges with exactly one path between any two nodes. A **rooted tree** designates a root; every non-root node has exactly one parent (first node on the path to the root); childless nodes are **leaves**. A **rooted binary tree** allows 0, 1, or 2 children per node.
- The **BST property**: for every node X, all left-subtree keys ≺ X's key and all right-subtree keys ≻ X's key. It is recursive, holding at every node. A **BST** = rooted binary tree + BST property.
- Ordering must be complete, transitive, and antisymmetric, so exactly one of p ≺ q, q ≺ p holds. The ordering is arbitrary (hence `≺` rather than `<`); a direct consequence is that **no duplicate keys** are allowed.
- **Search**: compare, recurse left or right, return `null` on falling off the tree. **Θ(log N) worst case on a bushy BST** (one node per level, ~log₂N levels), but Θ(N) on a spindly one. Bushy BSTs are astonishingly fast: a 10^300000-node tree searchable in one second at 1 µs/level.
- **Insert**: search; if not found, create the node and set the link. The clean recursive form returns the (possibly new) subtree root and reassigns `T.left`/`T.right`. The best base case is `if (T == null) return new BST(ik);`. **Avoid arms-length recursion.**
- **Deletion**, three cases: (1) no children, sever the parent link and let the node be garbage collected; (2) one child, point the parent at the child (safe because the whole subtree is already on the correct side); (3) two children, **Hibbard deletion**, promote the **predecessor** (largest in the left subtree) or **successor** (smallest in the right subtree) into the node's position, then delete that node, which is guaranteed to be case 1 or 2 because a maximum cannot have a larger child.
- **Maps are the same structure as Sets**: just store a key/value pair per node. Lookup by value is not efficient and requires scanning all nodes; that is fine.
