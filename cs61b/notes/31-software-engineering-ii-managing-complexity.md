<!-- Fri, Nov 06, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 31: Software Engineering II: Managing Complexity

**Overview.** This lecture is the second of the course's software engineering lectures, and its subject is the single resource that every large program eventually runs out of: human attention. Where the data structures half of CS 61B asks "how fast is this?", the software engineering half asks "can a person still understand this after it has been edited fifty times by five people?" The organizing claim is that complexity, meaning anything about the structure of a system that makes it hard to understand or hard to modify, is the dominant cost of real software, that it accumulates incrementally rather than arriving all at once, and that the main tool for fighting it is modular design: splitting a program into pieces whose interfaces are much simpler than their implementations, so that a programmer working on one piece can ignore almost everything about the others. In 61B terms, this is the principle behind every abstract data type you have written: a `Deque` user should need to know four method signatures, not whether the backing array is circular.

**Important note on source material.** The slide deck supplied for this lecture is an unfinished placeholder. It contains only the title ("Managing Software Complexity", CS61B, Josh Hug and Manuel Sabin), a banner reading "WARNING, THESE SLIDES HAVE NOT YET BEEN UPDATED FOR FALL 2026", and a slide literally titled "Stuff". No transcript was available. Separately, the "textbook chapter for this lecture" that was provided is **Chapter 31: Quicksort**, which is a different topic from the lecture title: the chapter numbering and the lecture numbering do not line up here. I have therefore done two things, both clearly labeled: (1) reconstructed the managing-complexity content from the standard source this CS 61B lecture is built on, John Ousterhout's *A Philosophy of Software Design*, with everything marked **(extra context)** because it is not in the provided slides; and (2) covered the provided Quicksort chapter faithfully and in full, since it is the assigned reading you were given and is squarely exam material. Treat part (1) as a reliable guide to the *themes* of this lecture but verify specific terminology against the final posted slides.

---

## Key Concepts

### Part A: Managing software complexity (extra context, reconstructed from Ousterhout, the standard basis for this CS 61B lecture)

#### 1. Complexity is about minds, not machines

When you analyze a data structure you count operations. When you analyze a *design* you count the things a programmer must hold in their head to make a safe change. Ousterhout's working definition: complexity is anything about the structure of a software system that makes it hard to understand or modify. Note the consequences of this definition. Complexity is not measured in lines of code, and it is not intrinsic to the problem being solved; it is a property of the *structure* you chose. Two programs that compute the same function can differ wildly in complexity. Complexity is also observer-relative in an important practical sense: if a system is easy for its original author and hard for everyone else, it is complex, because the author will not be around forever.

The economic argument for caring: in a long-lived system, almost all of the cost is paid after the first version works. Every bug fix, every feature, every onboarding of a new teammate pays a tax proportional to how complex the system is. Speed of initial writing is a small part of total cost.

#### 2. The three symptoms

You diagnose complexity by its symptoms, because "this feels messy" is not actionable.

- **Change amplification.** A conceptually tiny change requires edits in many places. The classic example is a color or a constant hard-coded in dozens of files: changing "the" color means changing it thirty times, and you will miss one. In 61B terms, if adding one new kind of `Gitlet` command means touching the serializer, the command parser, three helper classes, and a switch statement in `Main`, you have change amplification.
- **Cognitive load.** How much a developer must know in order to complete a task. This is the symptom people most often get wrong: it is *not* the same as line count. An API with six parameters and three documented invariants about calling order can be far more expensive than a longer implementation with a one-argument method. A method that silently returns `null` on failure adds cognitive load to every call site, forever.
- **Unknown unknowns.** The worst of the three: it is not obvious which code must be changed to complete a task, or what information you would need in order to do it correctly. With change amplification you at least know the work is large. With unknown unknowns you make a change, it compiles, the tests pass, and something breaks three months later, because there was an invariant you had no way to discover. Good design aims to make the *obvious* change the *correct* change.

#### 3. The two causes: dependencies and obscurity

- A **dependency** exists when a piece of code cannot be understood or modified in isolation: it only makes sense in relation to some other code. Dependencies are not evil and cannot be eliminated (a program with no dependencies does nothing coherent), but they can be reduced in number and made explicit and obvious. A method signature is a good dependency: it is visible, documented, and checked by the compiler. "Caller must call `init()` before `read()`" is a bad dependency: invisible, unchecked, and discoverable only by crashing.
- **Obscurity** is when important information is not obvious: a name like `data` or `tmp`, inconsistent conventions, a return value whose meaning you must read the implementation to learn, documentation that exists but does not say the one thing you need.

Dependencies mostly cause change amplification and cognitive load; obscurity mostly causes unknown unknowns.

#### 4. Complexity is incremental, so the discipline must be too

No one adds complexity in one catastrophic commit. It arrives as a thousand small, individually defensible decisions: one hard-coded constant here, one special case there, one method that is "temporarily" doing two jobs. Each one is too small to be worth a fight, and collectively they are fatal. The practical implication is a zero-tolerance mindset: you cannot fix complexity by scheduling a cleanup sprint later, because by then there is no single thing to fix. You fix it by not letting each small piece in.

#### 5. Tactical versus strategic programming

- **Tactical programming**: the goal is to get this feature working, right now. Design quality is whatever survives that goal. This is a local optimum and a global disaster: it is how you get a codebase where every change is scary.
- **Strategic programming**: the goal is a great design that happens to also make this feature work. You accept a small up-front cost (Ousterhout suggests something like 10 to 20 percent of development time spent on design, writing documentation, and refactoring) as an *investment* that pays back within months.
- The **tactical tornado** is the prolific developer who produces features at impressive speed and leaves a wake of complexity that others clean up. Management often rewards them, which is part of why this problem persists. (This framing is Ousterhout's and is the kind of thing a 61B lecture quotes approvingly; it is also a warning about how to behave on a team project.)

#### 6. Modules should be deep

Think of every module (a class, a package, a method, a service) as having two parts:

- its **interface**: everything a caller must know to use it, including the signatures *and* the informal contract (what it does, what it requires, what it guarantees, what it may throw, whether it mutates its arguments);
- its **implementation**: the code that fulfills the contract.

The cost of a module to the rest of the system is the complexity of its interface. The benefit is the implementation complexity it hides. Therefore:

- A **deep module** has a simple interface and a substantial implementation. The best 61B examples: `Collections.sort` (one line to call, a highly tuned sort inside), a garbage collector (interface: none, you just allocate), a file system (`open`, `read`, `write`, `close` hiding block allocation and caching), or your own `HashMap` implementation (interface: `get`, `put`, `containsKey`; implementation: buckets, resizing, hashing).
- A **shallow module** has an interface nearly as complicated as its implementation. It costs about as much to learn as to not have at all.

**Classitis** is the disease of believing classes should be small, as an end in itself, producing many tiny shallow classes. More classes means more interfaces, and interfaces are the expensive part. The right target is not small classes; it is *simple interfaces relative to the work hidden*. Sometimes the complexity-reducing move is to *merge* two classes.

#### 7. Information hiding and information leakage

**Information hiding** is the main technique for achieving depth: each module encapsulates design decisions that nothing else needs to know. In 61B this is `private`, and it is also the abstraction barrier you draw when you say "`ArrayDeque` users do not know about `nextFirst` and `nextLast`".

**Information leakage** is the opposite: a design decision is reflected in multiple modules, so changing it requires changing all of them. Leakage is the mechanism behind change amplification. Two signs of leakage:

- The same knowledge appears in two places (both the writer and the reader of a file know its byte layout, in separate hand-written code).
- A private field is "hidden" but a getter exposes it so faithfully that callers depend on its representation anyway. Making a field `private` and then adding `getInternalList()` that returns the live list hides nothing.

**Temporal decomposition** is the most common cause of accidental leakage: you structure the system by the *order in which operations happen* (first read the file, then process it, then write it) rather than by *knowledge*. The reader and writer both need the file format, so the format leaks across modules. The fix is to decompose by knowledge: one class owns the format and provides both read and write.

#### 8. General-purpose interfaces, pulling complexity downward, and different layers

- Prefer interfaces that are **somewhat general-purpose**: general enough that they are not entangled with one particular caller's needs, specific enough to still be easy to use. A text editor whose buffer offers `insert(text, position)` and `delete(start, end)` is better than one offering `backspace()`, `deleteSelection()`, and `deleteWordLeft()`, because the special-purpose version pushes editor-specific policy into the buffer and will grow a new method per UI gesture. A good test: "what is the simplest interface that covers all my current needs?" and "in how many situations will this be used?"
- **Pull complexity downward.** Given a choice between a module handling an ugly case internally and exposing a configuration option or error for the caller to handle, prefer handling it internally. One implementer suffers once; every caller suffers forever. Corollary: a method that can transparently do the right thing should not ask the caller to decide.
- **Different layer, different abstraction.** If a class's method has the same signature and meaning as a method it calls one layer down, the layer is adding nothing. These are **pass-through methods**, and they are a sign that responsibility is split badly. (Exception: decorators and interfaces designed for substitutability, where sameness is the point, but even there, be suspicious of a decorator that adds little.)
- **Define errors out of existence.** The best handling of an exceptional case is a design in which the case is not exceptional. The canonical example: Java's `substring` throws on out-of-range indices, forcing every caller to bounds-check, whereas a clipping definition ("return the overlap of the requested range with the actual string") makes an entire class of bugs and an entire class of `try/catch` blocks disappear. In your own 61B code, `Map.get` returning `null` for a missing key rather than throwing is a (partial) example of this move.

#### 9. Comments, names, and consistency

Comments exist because code cannot express everything: not the *why*, not the invariants, not the units, not the abstraction a reader should have in mind. The useful test for a comment is whether it says something **not deducible from the code next to it**. `i += 1; // increment i` is noise. `// invariant: items[nextFirst] is always empty` is load-bearing. Writing the interface comment *before* the implementation is a design tool: if the comment is long, awkward, or full of conditions, the interface is bad, and you have learned that before writing the code. Good naming is the cheapest cure for obscurity: a name should create an accurate image in the reader's mind, and if a thing is hard to name, it probably does too many things. Consistency is a complexity reducer because it lets a reader transfer conclusions from one part of the system to another without re-reading it.

#### 10. How this connects to the rest of CS 61B

Everything above is a generalization of practices you have already been graded on: abstract data types and abstraction barriers (information hiding), interfaces like `Comparator` and `Iterable` (general-purpose interfaces), `private` fields and invariants documented in comments (dependencies made explicit), and the design of a project like Gitlet, where the real difficulty is not any single algorithm but deciding which class owns which knowledge (the commit graph, the serialization format, the staging area, the command-line parsing).

---

### Part B: The provided textbook chapter, Chapter 31: Quicksort

This is the reading supplied with this lecture slot, so it is covered here in full. It is a different topic from the lecture title.

#### 1. Partitioning is the core idea

Each sort you have seen is powered by one idea:

| Sort | Core idea |
| --- | --- |
| Selection sort | Find the smallest item, put it at the front |
| Heapsort | Use a MaxPQ to find the maximum, put it at the back/front |
| Mergesort | Merge two sorted halves into one sorted whole |
| Insertion sort | Determine where the current item belongs and insert it |
| **Quicksort** | **Partition around a pivot** |

To partition an array `a[]`, pick an element `x = a[i]` as the **pivot**, and rearrange `a[]` so that:

1. `x` ends at some position `j` (possibly its original position `i`),
2. every element left of position `j` is `<= x`,
3. every element right of position `j` is `>= x`.

Note what you are *not* promised: nothing about the order *within* the left part or *within* the right part, and nothing about where equal elements go.

#### 2. Why one partition is already enormous progress

After partitioning, the pivot is in its **final sorted position**. The chapter makes this point with a partition on the pivot 5: 5 ends up exactly where it would be in the fully sorted array. Reason: every element less than or equal to it is to its left and every element greater than or equal to it is to its right, which is precisely the definition of being in the right slot. Since the pivot never needs to move again, and since nothing on the left will ever need to swap with anything on the right, the two sides are independent subproblems. That observation is the whole algorithm.

#### 3. The algorithm (Tony Hoare, 1960)

To quicksort N items:

1. Partition on the leftmost item as the pivot.
2. Recursively quicksort the left half (everything left of the pivot's final position).
3. Recursively quicksort the right half.

Base case: zero or one element is already sorted. Unlike mergesort, there is no combine step; the work happens on the way down, in the partition.

#### 4. Runtime analysis

Partitioning a region of K elements costs Theta(K), because you compare each of those elements against the pivot.

**Best case.** The pivot always lands in the middle (the pivot is the median). Level 0 partitions N elements. Level 1 partitions two regions of about N/2, which is about N total. In general each level does Theta(N) work. The number of levels H is Theta(log N) because the region size halves each time. Total: Theta(N log N).

**Worst case.** The pivot always lands at the very beginning (or very end) of its region, so one side is empty and the other has K-1 elements. Each level still does Theta(N) work in the aggregate, but now there are H = N levels, since you pivot on essentially every element one at a time. Total: Theta(N^2). Counting compares directly: (N-1) + (N-2) + ... + 1 = N(N-1)/2.

| Theoretical analysis | Quicksort | Mergesort |
| --- | --- | --- |
| Best case | Theta(N log N) | Theta(N log N) |
| Worst case | Theta(N^2) | Theta(N log N) |

On paper mergesort looks strictly better, yet Quicksort is empirically the fastest sort in most common situations. The resolution is that Quicksort's *average* case is Theta(N log N) with excellent constants.

**Why the average is good, argument 1 (the 10% case).** Suppose the pivot is never great, but merely always lands at least 10% of the way in from either edge. Then the larger side has at most 9/10 of the elements, so the number of levels is about log base 10/9 of N, which is still O(log N). Work per level is O(N), so total is O(N log N). The point: you do not need lucky medians, you only need to avoid the extremes, and "at least 10% from the edge" happens for the overwhelming majority of random pivots.

**Why the average is good, argument 2 (Quicksort is BST sort).** The sequence of `compareTo` calls Quicksort makes (each element against the pivot of its region) is exactly the sequence of comparisons made when inserting those elements into a BST in that order: the first pivot is the root, elements less than it go left, greater go right, and each subregion's pivot is that subtree's root. Random BST insertion of N items takes Theta(N log N) comparisons, so random-pivot Quicksort does too. This also explains the worst case: inserting a sorted sequence into a BST produces a spindly height-N tree, and quicksorting a sorted array with a leftmost pivot produces the same degenerate structure.

**Empirically.** For N items Quicksort uses an average of about 2N ln N compares (measured over 10,000 trials at N = 1000). Note 2 ln N is about 1.39 log2 N, so this is roughly 39% more compares than the theoretical minimum, yet Quicksort still wins on wall clock because its inner loop is simple and cache-friendly.

#### 5. Avoiding the worst case

Performance, both the order of growth and the constant factors, depends critically on how you select the pivot, how you partition around it, and what extra optimizations you add. Two failure modes for a leftmost pivot:

- **Bad ordering.** The array is already sorted, or reverse sorted, or nearly so. This is catastrophic in practice because real-world data is very often already sorted.
- **Bad elements.** Lots of duplicates, in the extreme all elements equal. With a two-way "less than or equal / greater than" partition, everything piles into one side.

Two pivot selection strategies discussed in the chapter and summary: **use the leftmost item** (simple, but dangerous on real data) and **pick a random pivot**. Randomizing the pivot, or shuffling the array before sorting, means the adversarial inputs no longer exist: no fixed input is reliably bad, so you are in the average case, Theta(N log N). You should accept the average-case Theta(N log N) result without proof.

#### 6. Related summary points from the chapter

**Insertion sort's sweet spots.** Insertion sort is very fast on arrays that are almost sorted (Theta(N) inversions) and is the fastest sort for small N (roughly N <= 15). This is why real library sorts switch to insertion sort on small subarrays. **Partitioning strategies.** You are expected to know how Quicksort partitioning works.

---

## Definitions

Terms from Part A are marked (extra context) because they are reconstructed, not taken from the provided slides.

**Complexity** (extra context): anything about the structure of a software system that makes it hard to understand or modify.

**Change amplification** (extra context): a symptom of complexity in which a conceptually small change requires modifications in many different places.

**Cognitive load** (extra context): a symptom of complexity measured by how much a developer must know in order to complete a task correctly. Not the same as amount of code.

**Unknown unknowns** (extra context): a symptom of complexity in which it is not obvious which code must change, or what knowledge is required, to complete a task correctly. The most damaging symptom, because there is no cue that you are about to be wrong.

**Dependency** (extra context): a relationship in which a piece of code cannot be understood or modified in isolation because its correctness depends on other code. Good dependencies are explicit, documented, and compiler-checked.

**Obscurity** (extra context): the condition of important information not being obvious, for example through poor names, undocumented invariants, or inconsistent conventions.

**Interface of a module** (extra context): everything a user of the module must know in order to use it, comprising the formal part (signatures, types, exceptions) and the informal contract (what it does, preconditions, guarantees, side effects).

**Implementation of a module** (extra context): the code that fulfills the interface's promises, which callers should not need to read.

**Deep module** (extra context): a module whose interface is much simpler than its implementation, so it hides a lot of complexity at a low cost.

**Shallow module** (extra context): a module whose interface is nearly as complex as its implementation, providing little net benefit.

**Classitis** (extra context): the belief that classes should be small for its own sake, which multiplies interfaces and so multiplies complexity.

**Information hiding** (extra context): encapsulating design decisions inside a module so that no other module depends on them.

**Information leakage** (extra context): the condition in which one design decision is reflected in multiple modules, so changing it requires changing all of them.

**Temporal decomposition** (extra context): structuring a system according to the time order of operations rather than according to what knowledge each part needs, a frequent cause of information leakage.

**Pass-through method** (extra context): a method that does little except invoke another method with the same or a very similar signature, indicating a layer that adds no abstraction.

**Tactical programming** (extra context): working with "make this feature work now" as the primary goal, accepting incremental complexity.

**Strategic programming** (extra context): working with "produce a good design" as a primary goal, investing a fraction of development time in design, documentation, and refactoring.

**Pivot**: an element `x = a[i]` of an array chosen as the reference point for a partition.

**Partitioning**: rearranging an array around a pivot so that the pivot sits at some index `j`, all elements left of `j` are `<=` the pivot, and all elements right of `j` are `>=` the pivot. The pivot may move during this process.

**Quicksort**: the sort that partitions on some pivot, then recursively quicksorts the region left of the pivot and the region right of the pivot. Named by Tony Hoare.

**Best case of Quicksort**: the pivot always lands at the median of its region, giving Theta(log N) levels and Theta(N log N) total time.

**Worst case of Quicksort**: the pivot always lands at an end of its region, giving N levels and Theta(N^2) total time.

**Inversion**: a pair of elements that are out of order relative to each other. Insertion sort runs in Theta(N) time on arrays with Theta(N) inversions.

---

## Worked Examples

### Example 1 (extra context): a shallow module versus a deep one

Suppose you are writing the disjoint-sets structure from earlier in the course. Here is a shallow design.

```java
/** Shallow: the interface exposes the implementation. */
public class ShallowDisjointSets {
    public int[] parent;          // public! callers can and will touch this
    public int[] size;

    public ShallowDisjointSets(int n) {
        parent = new int[n];
        size = new int[n];
        for (int i = 0; i < n; i += 1) {
            parent[i] = i;
            size[i] = 1;
        }
    }

    public int getParent(int i) { return parent[i]; }
    public void setParent(int i, int p) { parent[i] = p; }
    public int getSize(int i) { return size[i]; }
    public void setSize(int i, int s) { size[i] = s; }
}
```

Step by step, what goes wrong. The interface is six members wide and every one of them is a direct window onto the representation. To answer "are 3 and 7 connected?", a caller must itself walk parent pointers to the roots and compare them. To union two sets, a caller must itself find both roots, compare sizes, point the smaller root at the larger, and update the size array. So the algorithm (weighted quick union) lives in the *callers*, in as many copies as there are call sites. That is information leakage, and it produces all three symptoms at once: switching to path compression means editing every caller (change amplification); every caller must know the parent/size invariants (cognitive load); and a new teammate who writes `setParent(3, 7)` without updating `size` corrupts the structure silently (unknown unknowns). Note also that the getters and setters are pass-through methods: `getParent(i)` has exactly the meaning of `parent[i]` one layer down, so this layer adds nothing.

Now the deep version.

```java
/** Deep: two-method interface, nontrivial implementation hidden. */
public class DisjointSets {
    private final int[] parent;   // parent[i] < 0 means i is a root,
                                  // and -parent[i] is that tree's size

    public DisjointSets(int n) {
        parent = new int[n];
        for (int i = 0; i < n; i += 1) {
            parent[i] = -1;
        }
    }

    /** Returns true iff p and q are in the same set. */
    public boolean isConnected(int p, int q) {
        return find(p) == find(q);
    }

    /** Merges the sets containing p and q. No effect if already merged. */
    public void union(int p, int q) {
        int rootP = find(p);
        int rootQ = find(q);
        if (rootP == rootQ) {
            return;
        }
        if (parent[rootP] <= parent[rootQ]) {      // rootP's tree is larger
            parent[rootP] += parent[rootQ];
            parent[rootQ] = rootP;
        } else {
            parent[rootQ] += parent[rootP];
            parent[rootP] = rootQ;
        }
    }

    private int find(int i) {
        if (parent[i] < 0) {
            return i;
        }
        parent[i] = find(parent[i]);               // path compression
        return parent[i];
    }
}
```

The interface is now two methods whose contracts can be stated in one line each, and it says nothing at all about arrays, roots, sizes, negative sentinels, or compression. Three design moves are visible. First, information hiding: `parent` is private and `find` is private, so the weighting-plus-compression decision is a secret. Second, pulling complexity downward: `union` silently does the right thing when the two items are already connected instead of throwing or asking the caller to check, which is defining an error out of existence. Third, depth: the clever encoding "a negative entry is a root whose magnitude is the size" lets one array do two jobs, and no caller is burdened by it. You could replace the entire implementation with a completely different algorithm tomorrow and not a single caller would change. Note that the *implementation* got more intricate while the *interface* got simpler; that is exactly the trade you want, and it is why "smaller classes" is the wrong goal.

### Example 2 (extra context): temporal decomposition causing information leakage

A tempting structure for a program that loads and saves a Gitlet-style commit:

```java
class CommitReader {
    Commit read(File f) {
        // knows: byte 0..7 = timestamp, 8..47 = parent hash, 48.. = message
        ...
    }
}

class CommitWriter {
    void write(Commit c, File f) {
        // ALSO knows: byte 0..7 = timestamp, 8..47 = parent hash, 48.. = message
        ...
    }
}
```

Reasoning about why this hurts: the classes were split by *when* things happen (reading happens at load time, writing happens at save time), but the knowledge they need is identical, namely the on-disk format. That knowledge now lives in two modules, so adding a second parent hash for merge commits is a two-file change where it should be a one-file change, and a change that is easy to make in one place and forget in the other, producing files that write fine and read back as garbage. The fix is to decompose by knowledge instead:

```java
class CommitSerializer {
    byte[] toBytes(Commit c) { ... }
    Commit fromBytes(byte[] b) { ... }   // the format is a secret of this class
}
```

One module owns the format, the symmetry is enforced by proximity, and the format never appears anywhere else in the program.

### Example 3: Quicksort partitioning in Java, with a full trace

The chapter says you are expected to know how Quicksort partitioning works, so here is a complete, simple, in-place scheme using the leftmost element as pivot. (Many partitioning schemes exist and the chapter links to an illustration of Hoare's two-pointer version; the one below is the easiest to trace and satisfies the same specification.)

```java
public class Quicksort {

    /** Sorts a[lo..hi] inclusive. */
    public static void sort(int[] a, int lo, int hi) {
        if (lo >= hi) {                  // base case: 0 or 1 element
            return;
        }
        int j = partition(a, lo, hi);    // a[j] is now in its final place
        sort(a, lo, j - 1);              // quicksort the left
        sort(a, j + 1, hi);              // quicksort the right
    }

    /**
     * Partitions a[lo..hi] on pivot a[lo].
     * Returns the final index j of the pivot, with
     * a[lo..j-1] <= a[j] <= a[j+1..hi].
     */
    private static int partition(int[] a, int lo, int hi) {
        int pivot = a[lo];
        int i = lo;            // invariant: a[lo+1..i] < pivot,
                               //            a[i+1..j-1] >= pivot
        for (int j = lo + 1; j <= hi; j += 1) {
            if (a[j] < pivot) {
                i += 1;
                swap(a, i, j);
            }
        }
        swap(a, lo, i);        // drop the pivot into the boundary slot
        return i;
    }

    private static void swap(int[] a, int p, int q) {
        int t = a[p];
        a[p] = a[q];
        a[q] = t;
    }
}
```

**Trace on `[5, 3, 8, 1, 9, 2]`, call `sort(a, 0, 5)`.**

Top-level partition, `pivot = a[0] = 5`, `i = 0`:

| `j` | `a[j]` | `a[j] < 5`? | action | array after |
| --- | --- | --- | --- | --- |
| 1 | 3 | yes | `i` becomes 1, swap(1,1) | `[5, 3, 8, 1, 9, 2]` |
| 2 | 8 | no | none | `[5, 3, 8, 1, 9, 2]` |
| 3 | 1 | yes | `i` becomes 2, swap(2,3) | `[5, 3, 1, 8, 9, 2]` |
| 4 | 9 | no | none | `[5, 3, 1, 8, 9, 2]` |
| 5 | 2 | yes | `i` becomes 3, swap(3,5) | `[5, 3, 1, 2, 9, 8]` |

Final `swap(a, lo = 0, i = 3)` gives `[2, 3, 1, 5, 9, 8]` and returns `3`. Check the three partition properties: the pivot 5 sits at index 3; `{2, 3, 1}` to its left are all `<= 5`; `{9, 8}` to its right are all `>= 5`. And 5 is in its final sorted position, since exactly three elements of the array are smaller than it.

Memory reasoning in words (no box and pointer diagram needed here, but the aliasing matters): `a` is a reference to a single array object on the heap. Both `sort` and `partition` receive copies of that reference, not copies of the array, so every `swap` mutates the one shared object. The recursive calls therefore do not need to return anything or merge anything, which is the structural difference from mergesort: mergesort allocates and combines, Quicksort mutates in place and never looks back. The only per-call stack data is `lo`, `hi`, `pivot`, `i`, and `j`, so the extra space is proportional to the recursion depth.

Recursion continues:

- `sort(a, 0, 2)` on `[2, 3, 1]`: pivot 2, `i = 0`; `j = 1` (3 is not `< 2`, nothing); `j = 2` (1 `< 2`, so `i` becomes 1, swap(1,2) giving `[2, 1, 3]`); final swap(0,1) gives `[1, 2, 3]`, return 1. Then `sort(a, 0, 0)` and `sort(a, 2, 2)` are base cases.
- `sort(a, 4, 5)` on `[9, 8]`: pivot 9, `j = 5` (8 `< 9`, `i` becomes 5, swap(5,5)); final swap(4,5) gives `[8, 9]`, return 5. Both halves are base cases.

Final array: `[1, 2, 3, 5, 8, 9]`.

### Example 4: the worst case, traced structurally

Run the same code on the already-sorted `[1, 2, 3, 4, 5]`. The pivot is 1, and no element is `< 1`, so `i` stays at `lo = 0`, the final swap is a no-op, and `partition` returns 0 after 4 comparisons. The left recursive call is `sort(a, 0, -1)`, which is empty; the right is `sort(a, 1, 4)` on 4 elements. The next level does 3 comparisons and again splits 0 and 3. Total comparisons: 4 + 3 + 2 + 1 = 10 = N(N-1)/2, and the recursion depth is N. This is Theta(N^2) on the single most common shape of real-world input, which is the entire argument against the leftmost-pivot strategy.

Viewed through the BST lens: inserting 1, 2, 3, 4, 5 into a BST in that order yields a spindly right-leaning chain of height 5, and the comparison sequence is identical. The reverse-sorted array fails the same way with the pivot always the largest. An all-duplicates array `[7, 7, 7, 7, 7]` also degenerates with a two-way partition, since no element is strictly less than the pivot.

The fix, per the chapter: choose a random pivot (swap a random element into position `lo` before partitioning), or shuffle the array first.

```java
public static void sort(int[] a, int lo, int hi) {
    if (lo >= hi) {
        return;
    }
    swap(a, lo, lo + new java.util.Random().nextInt(hi - lo + 1));  // random pivot
    int j = partition(a, lo, hi);
    sort(a, lo, j - 1);
    sort(a, j + 1, hi);
}
```

(In real code you would hoist the `Random` out rather than allocating one per call; that detail is extra context.) Note that this does not make Theta(N^2) impossible, it makes it astronomically unlikely and, crucially, makes it independent of the input, so no particular dataset is reliably bad.

---

## Common Pitfalls

**On managing complexity (extra context):**

1. **Equating complexity with size.** A 200-line method with a one-argument signature and no hidden preconditions may be far simpler, in the sense that matters, than five 20-line classes that must be used together in a specific order.
2. **Believing more classes means better design.** Classitis. Each new class is a new interface to learn. Split a class when it is hiding two unrelated secrets, not because it crossed a line count.
3. **Calling a field hidden because it is `private`.** If a getter returns the live internal list, or if the field's structure is visible through the API's behavior, the information still leaked. Ask "could I change this decision without touching any other file?"
4. **Decomposing by time instead of by knowledge.** "Read phase class, process phase class, write phase class" feels natural and reliably duplicates knowledge across modules.
5. **Pushing complexity upward for the implementer's convenience.** Adding a configuration flag or throwing an exception so the caller decides is cheap for you and expensive for everyone, multiplied by the number of call sites.
6. **Writing comments that restate the code.** The test is whether the comment adds information a reader cannot get from the code beside it. Invariants, units, ownership, and "why" are worth comments; `// loop over items` is not.
7. **Assuming you will clean it up later.** Complexity accrues in increments too small to schedule a fix for. The only effective policy is to refuse each increment.
8. **Confusing cognitive load with unknown unknowns on exams.** Cognitive load is "there is a lot I must learn." Unknown unknowns is "I do not know what I must learn, and nothing will tell me."

**On Quicksort:**

9. **Claiming Quicksort is Theta(N log N).** It is Theta(N log N) in the best and average cases, and Theta(N^2) in the worst case. A single unqualified Theta bound for all inputs does not exist for Quicksort.
10. **Thinking partitioning sorts each side.** Partitioning guarantees only "left is all `<=`, right is all `>=`". The two sides are typically unsorted, and the relative order inside them is whatever the partition scheme happened to produce.
11. **Forgetting that the pivot is done.** After `partition` returns `j`, recursing on `lo..j` instead of `lo..j-1` includes the pivot again, which can cause infinite recursion.
12. **Thinking randomization removes the worst case.** It removes *predictable* worst cases. The Theta(N^2) behavior remains possible, just vanishingly improbable and no longer tied to any particular input.
13. **Forgetting that already-sorted input is the common case in practice.** This is the specific reason leftmost-pivot Quicksort is dangerous on real data, and the reason the chapter emphasizes pivot selection.
14. **Overlooking duplicates.** All-equal arrays degenerate with a two-way partition, which is a separate failure mode from bad ordering.
15. **Assuming Quicksort is stable or in-place-with-no-extra-space.** The usual in-place schemes are not stable, and the recursion uses stack space proportional to the depth, which is Theta(N) in the worst case. (The stability and space observations are extra context beyond the provided chapter text.)

---

## Likely Exam Points

Because the slides for this lecture are a placeholder, the complexity items below are flagged (extra context) and reflect how this material is normally assessed in CS 61B: as short conceptual questions and as design critique, not as code tracing. The Quicksort items come directly from the provided chapter and its exercises.

**1. Name the symptom (extra context).**
*Q: A team's app hard-codes the string "blue" in 27 files. Marketing asks for green. Which symptom of complexity is this, and what is its underlying cause?*
A: Change amplification, caused by information leakage: the design decision "the theme color" is reflected in 27 modules instead of being encapsulated in one. The fix is a single constant or theme module owning that decision.

**2. Cognitive load versus unknown unknowns (extra context).**
*Q: Distinguish them with one example each.*
A: Cognitive load is the amount a developer must learn to do a task, for example an API with eight parameters and a required call order: the work is visible but expensive. Unknown unknowns is not knowing what you need to know, for example an undocumented invariant that `items[nextFirst]` must always be empty: you change the code, it compiles, tests pass, and it breaks later. Unknown unknowns is worse because nothing warns you.

**3. Deep versus shallow (extra context).**
*Q: Is a class with 2 public methods and 300 lines of implementation deep or shallow? What about a class with 15 public methods and 40 lines?*
A: The first is deep (simple interface hiding substantial implementation, high benefit-to-cost). The second is almost certainly shallow (its interface is nearly as complex as what it does, so it costs about as much to learn as to do yourself). Depth is the ratio of hidden complexity to interface complexity, not an absolute size.

**4. Critique a design (extra context).**
*Q: A `Deque` exposes `public int[] items; public int nextFirst;` plus `addFirst`/`removeFirst`. What is wrong?*
A: Information leakage: the circular-array representation is part of the interface, so callers can depend on it and changing to a linked implementation becomes a cross-codebase change. The public fields also let callers break invariants, creating unknown unknowns. Make the fields private and let the only contract be the deque operations.

**5. Tactical versus strategic (extra context).**
*Q: Why is the "tactical tornado" a net negative even though they ship features fastest?*
A: Their speed comes from accepting incremental complexity that everyone else pays for in change amplification, cognitive load, and unknown unknowns. Because the cost is paid later and by others, it is undercounted. Strategic programming spends roughly 10 to 20 percent of time on design and refactoring and recovers it within months.

**6. Define the partition specification.**
*Q: State precisely what partitioning `a[]` on pivot `x = a[i]` guarantees.*
A: `x` moves to some index `j` (possibly still `i`), every element at indices `< j` is `<= x`, and every element at indices `> j` is `>= x`. Nothing is guaranteed about the order within either side.

**7. One-partition consequence.**
*Q: After a single partition, which element(s) are definitely in final sorted position, and why?*
A: The pivot. Everything `<=` it is to its left and everything `>=` it is to its right, so its index equals the number of elements that must precede it in sorted order. This is why the two sides can be sorted independently with no merge step.

**8. Trace a partition.**
*Q: Partition `[7, 2, 9, 4, 1]` on the leftmost pivot using the scheme above. Give the array and the returned index.*
A: pivot 7, `i = 0`. `j=1`: 2 < 7, `i=1`, swap(1,1) gives `[7,2,9,4,1]`. `j=2`: 9 no. `j=3`: 4 < 7, `i=2`, swap(2,3) gives `[7,2,4,9,1]`. `j=4`: 1 < 7, `i=3`, swap(3,4) gives `[7,2,4,1,9]`. Final swap(0,3) gives `[1,2,4,7,9]` and returns 3. (Here the result happens to be fully sorted, which partitioning does not promise in general.)

**9. Best and worst case runtimes with justification.**
*Q: Derive both.*
A: Partitioning K elements costs Theta(K), so each full level of the recursion costs Theta(N) in total. Best case the pivot is the median, region sizes halve, so the number of levels is Theta(log N) and the total is Theta(N log N). Worst case the pivot lands at an end, so one side is empty and the other loses only one element, giving N levels and Theta(N^2) (compare count N(N-1)/2).

**10. Exact recursion depth (from the chapter exercises).**
*Q: Give the exact depth of the Quicksort recursion tree in the best and worst cases, not a Theta bound.*
A: Worst case N (one recursive call per element, always pivoting on the smallest or largest). Best case log base 2 of N (the median halves the region each time).

**11. Which arrays are worst case with a leftmost pivot (from the chapter exercises).**
*Q: With a leftmost pivot and a partition into "items `<=` pivot" and "items `>` pivot", name three input families that give worst-case runtime.*
A: (1) Already-sorted arrays, since the pivot is always the minimum and nothing lands to its left. (2) Reverse-sorted arrays, same problem with the pivot always the maximum. (3) Arrays where all items are equal, since everything falls into the `<=` side and the `>` side is always empty.

**12. Choose the best pivot (from the chapter exercises).**
*Q: For `[90, 50, 20, 30, 40, 10, 60, 50, 30]`, what is the best pivot?*
A: 40, because it is the median and will split the array as evenly as possible.

**13. Why is Quicksort used if mergesort has a better worst case?**
A: Because Quicksort's average case is Theta(N log N) with small constants and good locality, and randomizing the pivot or shuffling the input puts you in the average case for *any* input. Empirically it uses about 2N ln N compares and is the fastest sort for most real-world situations. Supporting arguments: a pivot merely 10% in from either edge still gives log base 10/9 of N levels, hence O(N log N); and Quicksort's comparisons mirror random BST insertion, which is Theta(N log N).

**14. Quicksort as BST sort.**
*Q: Explain the correspondence and use it to explain Quicksort's worst case.*
A: The first pivot acts as a BST root: every other element is compared against it and routed left or right exactly as in BST insert, and each subregion's pivot is that subtree's root. So Quicksort's comparison sequence equals the comparison sequence of inserting the items in pivot order. Sorted input with a leftmost pivot corresponds to inserting a sorted sequence, producing a height-N spindly tree, which is the Theta(N^2) case.

**15. Insertion sort sweet spots.**
*Q: When is insertion sort the right choice?*
A: When the array is almost sorted (Theta(N) inversions), where it runs in Theta(N) time, and when N is small (roughly N <= 15), where its tiny constants beat asymptotically better sorts.

**16. Selection sort compare count (from the chapter exercises).**
*Q: How many compares does selection sort use on 10 items?*
A: 45. Nine compares to find the smallest, then 8, then 7, down to 1: 9 + 8 + ... + 1 = 45.

**17. Binary search versus linear scan (from the chapter exercises, a reminder that constants and hardware matter).**
*Q: On a sorted array in memory, and on a sorted array stored on physical tape, which is better?*
A: In memory, binary search, log N time. On tape, linear scanning, because runtime is dominated by physically accelerating the tape; binary search jumps to the middle, then a quarter back, then an eighth forward, covering O(N) of tape anyway while paying enormous acceleration costs in both directions. The moral generalizes: asymptotics do not capture the cost model of the hardware.

---

## Summary

- The provided slide deck for this lecture is an unfinished placeholder (title plus a slide called "Stuff"), there was no transcript, and the provided textbook chapter is Chapter 31 on Quicksort, a different topic from the lecture title. Part A below is reconstructed context, Part B is the provided reading.

**Managing complexity (extra context):**

- Complexity is anything about a system's structure that makes it hard to understand or modify. It is the dominant long-run cost of software, and it is about human attention, not machine time.
- Three symptoms: change amplification (small change, many edits), cognitive load (much to know), unknown unknowns (you do not know what you must know). The third is the worst.
- Two causes: dependencies (code that cannot be understood alone) and obscurity (important information not obvious).
- Complexity accumulates incrementally, so it must be resisted incrementally. Strategic programming invests a slice of every task in design and refactoring; tactical programming borrows against the future.
- A module's cost is its interface, its benefit is the implementation it hides. Prefer deep modules (simple interface, substantial implementation). Beware classitis: more classes means more interfaces.
- Information hiding is the main tool; information leakage is the main failure, and temporal decomposition (structuring by order of operations rather than by knowledge) is its most common source.
- Prefer somewhat general-purpose interfaces, pull complexity downward into the implementation rather than onto callers, define errors out of existence where you can, give each layer a distinct abstraction (no pass-through methods), and write comments that say what the code cannot.

**Quicksort (provided chapter):**

- Partitioning on pivot `x`: `x` moves to index `j`, everything left of `j` is `<= x`, everything right is `>= x`. The pivot is then in its final sorted position.
- Quicksort: partition, quicksort the left, quicksort the right. No merge step. You are expected to know how partitioning works.
- Partitioning K elements costs Theta(K), so each level costs Theta(N). Best case (median pivot): Theta(log N) levels, Theta(N log N). Worst case (pivot at an end): N levels, Theta(N^2).
- Mergesort is Theta(N log N) in both best and worst cases, yet Quicksort is empirically the fastest sort for most real situations, because its average case is Theta(N log N) with small constants (about 2N ln N compares).
- Two arguments for the average case: a pivot at least 10% in from either edge still gives O(log N) levels, and Quicksort's comparisons are exactly those of random BST insertion.
- Worst cases with a leftmost pivot come from bad ordering (sorted or reverse-sorted input) and bad elements (all duplicates). A random pivot or an up-front shuffle puts you in the average case regardless of the input.
- Insertion sort remains the right tool for almost-sorted arrays (Theta(N) inversions) and for small N (roughly N <= 15).
