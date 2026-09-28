<!-- Mon, Sep 28, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 14: Disjoint Sets

This lecture opens the "Data Structures" arc of CS 61B by deriving one data structure from scratch, start to finish: the **Disjoint Sets** (a.k.a. Union-Find) structure that solves the **Dynamic Connectivity** problem. After a quick review of Big Theta versus Big O (Big O matters here because some of these operations have runtimes that depend on the shape of the data, not just on N), we define a two-method API, `connect(p, q)` and `isConnected(p, q)`, and then walk up a ladder of five implementations: `ListOfSetsDS` (intuitive but slow and complicated), `QuickFindDS` (constant-time `isConnected`, linear-time `connect`), `QuickUnionDS` (tree of parent pointers, fast in the best case but Θ(N) when the tree is a stick), `WeightedQuickUnionDS` (always hang the smaller tree under the bigger one, forcing height Θ(log N)), and finally `WeightedQuickUnionWithPathCompressionDS` (flatten every path you walk), which is the implementation actually used in practice today and runs in O(M α(N)) for M operations. The big takeaways are less about union-find trivia and more about method: represent the *connected components*, not the individual connections; the choice of underlying representation (list of sets versus array of ids versus array of parents) can doom or save your asymptotics; and a good data structure is what makes an otherwise impossible problem (30 years of compute versus 6 seconds) solvable.

---

## Key Concepts

### 1. Big Theta versus Big O, and why this lecture needs both

Formally, R(N) ∈ Θ(f(N)) means there exist positive constants k₁ and k₂ such that

```
k1 * f(N)  <=  R(N)  <=  k2 * f(N)      for all N greater than some N0
```

so Θ pins the order of growth from both sides: it is a two-sided "order of growth *is* f(N)" statement. R(N) ∈ O(f(N)) is only the **upper bound**: "order of growth is less than or equal to f(N)". So N²/2, 2N², and N² + 38N + N are all in Θ(N²) *and* in O(N²), but lg(N) is in O(N²) only.

*(extra context: the slide that visualizes Big O reuses the two-constant sentence from the Big Theta slide, which is a copy/paste artifact. The Big O definition needs only the single upper constant k₂.)*

Why it matters today: for several of these implementations, the runtime of a single call depends on the *input* and on the current shape of the structure. `QuickUnionDS.connect` can be as cheap as constant time (both nodes are already roots) or as expensive as Θ(N) (the tree is a long stick). There is no single f(N) sandwiching it from both sides, so we report an upper bound: O(N). When the runtime does not vary, as with `QuickFindDS.isConnected` (always exactly two array accesses) or `QuickFindDS.connect` (always a full array scan), we can and do say Θ.

### 2. The problem: Dynamic Connectivity

A `Set` is a collection with no duplicates, supporting `add` and `contains` (the lecture asserts Θ(1) for these and defers the reason to later in the course). Disjoint Sets is a different, more specialized ADT with exactly two operations:

- `connect(x, y)`: connect x and y.
- `isConnected(x, y)`: is x connected to y? **Connections are transitive**, so they need not be direct.

Geographical example from lecture: `connect(Russia, China)`, `connect(Russia, Mongolia)`, then `isConnected(China, Mongolia)` is true (via Russia) even though we never connected them directly. Then `connect(USA, Canada)` leaves `isConnected(USA, Mongolia)` false, but after `connect(Russia, USA)` (the Bering Strait crossing) it becomes true.

Note what is *absent* from the API: there is no "disconnect", no "list the members of x's group", no "how are x and y connected". This narrowness is what lets us be fast.

Applications mentioned: percolation theory (used in computational chemistry) and as a subroutine inside other algorithms, notably Kruskal's minimum spanning tree algorithm.

### 3. Simplifying assumptions

To avoid tangling with generics and hashing, the lecture restricts to integers:

- Items are the integers `0` through `N - 1` (so `8` instead of `USA`).
- N is declared up front in the constructor, and **everything starts disconnected** (N singleton sets).

Design constraints stated explicitly:

- N (number of elements) can be huge.
- M (number of method calls) can be huge.
- Calls are **interspersed**: you cannot assume all the `connect`s come first and all the `isConnected`s after. This is what "dynamic" means, and it rules out "just preprocess everything once" designs.

### 4. The pivotal idea: track connected components, not connections

The naive approach records every connecting line, for example a `List<Integer[]>` holding `[0,1], [1,2], [0,4], [2,4], [3,5]`. `connect` is then trivially fast, but `isConnected(3, 0)` is hard: you would have to do some search over all of the pairs to decide whether one item is reachable from another. The data you stored does not answer the question you were asked.

The fix is to store the answer instead of the history. For each item, its **connected component** is the set of all items connected to it. *How* things got connected is information we never need. The connect sequence from the slides, viewed this way:

```
start                     {0}, {1}, {2}, {3}, {4}, {5}, {6}
connect(0, 1)             {0, 1}, {2}, {3}, {4}, {5}, {6}
connect(1, 2)             {0, 1, 2}, {3}, {4}, {5}, {6}
connect(0, 4)             {0, 1, 2, 4}, {3}, {5}, {6}
connect(3, 5)             {0, 1, 2, 4}, {3, 5}, {6}
isConnected(2, 4)         true      (same set)
isConnected(3, 0)         false     (different sets)
connect(4, 2)             {0, 1, 2, 4}, {3, 5}, {6}     <- no change, already together
connect(4, 6)             {0, 1, 2, 4, 6}, {3, 5}
connect(3, 6)             {0, 1, 2, 3, 4, 5, 6}
isConnected(3, 0)         true
```

Observe the two structural facts that every later implementation exploits: the components are **disjoint** (hence the name) and they **only ever merge**, never split. `connect` is a merge of two components; `isConnected` is "same component?".

### 5. Implementation ladder, and what each step buys

**Idea #1, `ListOfSetsDS`:** literally `List<Set<Integer>>`, for example `[{0,1,2,4}, {3,5}, {6}]`. Very intuitive and actually terrible. To do anything you must iterate over the outer list to find which set an item lives in. `isConnected` checks up to N sets: O(N). `connect` must find both items' sets and then merge them: O(N). Worst case is when nothing is connected, since then there are N sets to scan. The lecture's verdict is the lesson: *the data structure choice doomed us*. We picked a representation in which the most common question ("which component is item i in?") requires a search.

**Idea #2, `QuickFindDS`:** flip the representation inside out. Keep `int[] id` where `id[i]` is the set number of item i. Now "which component is i in?" is a single array access, and `isConnected` is `id[p] == id[q]`: Θ(1), two array accesses. The price is `connect(p, q)`, which must relabel an entire component: walk the whole array and change every entry equal to `id[p]` into `id[q]`. That is always a full scan, so Θ(N), more precisely N+2 to 2N+2 array accesses. Too slow for practical use: every connection costs linear time.

**Idea #3, `QuickUnionDS`:** the radical move. Instead of giving each item an *id*, give each item a **parent**. Roots are marked with `-1`. This implicitly builds a forest: one tree per connected component, and the root of the tree serves as the component's representative.

```
parent   -1   0   1   -1   0   3   -1
          0   1   2    3   4   5    6

tree shapes:   0                 3          6
              / \                 \
             1   4                 5
             |
             2
components:  {0, 1, 2, 4}       {3, 5}     {6}
```

(The slides note that the optional textbook stores an item's parent as **itself** for roots instead of `-1`.)

Now `connect(p, q)` does not relabel anything. It finds both roots and hangs one root under the other: a **single array write**. For `connect(5, 2)`: `root(5)` is 3, `root(2)` is 0, so set `parent[root(5)] = root(2)`, that is `parent[3] = 0`. Why can't we just write `parent[5] = 2`? Because then 3 would be orphaned: 3 was 5's parent and 5's only link to the rest of its component, so we would "lose 3" out of the merged component. Reparenting the *root* is what moves the whole subtree along with it.

The cost has moved into `root`, which climbs parent pointers to the top. If the trees are tall, that climb is slow. Worst case: always attach the first item's tree below the second item's, for example `connect(4, 3); connect(3, 2); connect(2, 1); connect(1, 0)`, and you get a stick of height M after M operations. For N items, both `connect` and `isConnected` are then Θ(N) in the worst case, which can be *worse* than QuickFind (QuickFind at least had Θ(1) `isConnected`). We report O(N) in the table because the runtime ranges between constant and linear.

**Idea #4, `WeightedQuickUnionDS`:** the defect was tall trees, so refuse to build them. Track the **size** (a.k.a. **weight**, the number of items) of each tree, and always link the root of the *smaller* tree to the root of the larger. Ties are broken arbitrarily. `isConnected` needs no change at all.

Why size and not height? Comparing heights also gives Θ(log N) worst case, but tracking height becomes awkward once we add path compression later, and the code is more complicated with no asymptotic gain. Note that size and height can disagree about which choice makes the shorter tree in a given step; weighting is nevertheless asymptotically just as good.

The height bound: to make the height grow as fast as possible, you must repeatedly join two trees of equal height, which doubles the node count each time the height increases:

| N | max height |
|---|---|
| 1 | 0 |
| 2 | 1 |
| 4 | 2 |
| 8 | 3 |
| 16 | 4 |

So worst case height is Θ(log N), and both operations become O(log N).

Two common ways to store the weights:
- Overload the parent array: store `-weight` at roots instead of `-1`, for example `parent = [-2, -1, -1, -1, -1, 0, -4, 6, 6, 8]`.
- Keep a separate `size` array, for example `size = [2, 1, 1, 1, 1, 1, 4, 1, 2, 1]`.

**Idea #5, `WeightedQuickUnionWithPathCompressionDS`:** when you climb from a node to its root, you have already paid for the walk, so on the way you may as well **reparent every node you touched directly to the root**. The extra cost is insignificant (same order of growth as the climb you were doing anyway), and it permanently flattens that path. The key intuition: as N grows, trees tend to get taller; as M (number of operations) grows, trees get shorter, and with enough operations the height shrinks toward 1. Compression applies on both `connect` and `isConnected`, since both call find/root.

This is the punchline of the derivation: **weighted quick union plus path compression is the standard way disjoint sets are implemented today.**

### 6. The final performance picture

Per-operation (constructor is Θ(N) for all of them, since all must initialize N sets or an N-length array):

| Implementation | constructor | connect | isConnected |
|---|---|---|---|
| `ListOfSetsDS` | Θ(N) (make N sets) | O(N) (find each element, combine 2 sets) | O(N) (check up to N sets) |
| `QuickFindDS` | Θ(N) (size-N array) | Θ(N) (scan whole array, update ids) | Θ(1) (two array accesses) |
| `QuickUnionDS` | Θ(N) | O(N) (climb tree to find root) | O(N) (climb 2 trees, compare roots) |
| `WeightedQuickUnionDS` | Θ(N) | O(log N) | O(log N) |
| `WQU + path compression` | Θ(N) | O(α(N)) amortized | O(α(N)) amortized |

Total for M operations on N elements (an operation being a `connect` or an `isConnected`):

| Implementation | Total runtime for M ops |
|---|---|
| `ListOfSetsDS` | O(NM) |
| `QuickFindDS` | Θ(NM) |
| `QuickUnionDS` | O(NM) |
| `WeightedQuickUnionDS` | O(M log N) |
| `WQUWithPathCompressionDS` | O(M α(N)) |

The lecture's motivating number: with the naive implementation the total is O(N + MN) = O(MN); with the best it is O(N + M log N). For N = 10⁹ and M = 10⁹ that is the difference between roughly **30 years and 6 seconds**. "Good data structure unlocks solutions to problems that could otherwise not be solved."

### 7. lg* and the inverse Ackermann function

CS 170 territory, flagged in lecture as a spoiler rather than examinable analysis:

- With path compression, each operation takes on average **lg\* N** time. `lg*` is "how many times you press the log₂ button on a calculator before reaching a number that is 1 or less".

| N | lg* N |
|---|---|
| 1 | 0 |
| 2 | 1 |
| 4 | 2 |
| 16 | 3 |
| 65536 | 4 |
| 2^65536 | 5 |

  So lg* N ≤ 5 for any realistic input, and M operations cost O(M lg\* N) for large M.
- An even tighter bound is **α(N)**, the inverse Ackermann function, which also does not exceed 5 for any conceivable N. Source: Bob Tarjan, "Efficiency of a Good But Not Linear Set Union Algorithm", written while at UC Berkeley in 1975.

Neither bound is literally constant, but both are "good enough for all practical uses".

---

## Definitions

- **Set:** a collection of elements with no duplicates. `add` of an element already present does nothing. Lecture takes `add` and `contains` to be Θ(1).
- **Disjoint Sets (Union-Find):** an ADT over a fixed universe of N items, maintaining a partition of those items into non-overlapping sets, with two operations: `connect(x, y)` and `isConnected(x, y)`.
- **`connect(p, q)`:** merge the set containing p with the set containing q. Also called *union*.
- **`isConnected(p, q)`:** return true if p and q currently lie in the same set. Connectivity is **transitive**, so indirect connections count.
- **Dynamic Connectivity problem:** the problem of answering interspersed `connect` and `isConnected` queries as they arrive, with no ability to see the whole input in advance.
- **Connected component of an item:** the set of all items connected to that item (including itself).
- **Disjoint:** no item belongs to two components at once, so the components form a partition.
- **Big Theta, Θ(f(N)):** order of growth *is* f(N); there exist positive k₁, k₂ with k₁f(N) ≤ R(N) ≤ k₂f(N) for all N beyond some N₀.
- **Big O, O(f(N)):** order of growth is *less than or equal to* f(N); an upper bound only.
- **id (in QuickFind):** the set number of an item; `id[i]` is item i's component label. Two items are connected exactly when their ids match.
- **parent (in QuickUnion):** `parent[i]` is the index of item i's parent in the forest; a negative value marks a **root**.
- **root (or find):** the operation that climbs parent pointers from an item to the root of its tree. The root acts as the canonical representative of the component.
- **Weight / size of a tree:** the total number of items in that tree (the two words are used interchangeably in this lecture).
- **Weighted quick union:** quick union with the rule "always link the root of the smaller tree to the root of the larger tree", ties broken arbitrarily.
- **Path compression:** during a root/find call, reset the parent of every node encountered on the path to point directly at the root.
- **lg\* N (log-star):** the number of times you must apply log₂ to N before the result is at most 1. Effectively ≤ 5 for realistic N.
- **α(N):** the inverse Ackermann function; grows even more slowly than lg\*, also ≤ 5 for realistic N.

---

## Worked Examples

### The interface

```java
public interface DisjointSets {
    /** Connects two items p and q. */
    void connect(int p, int q);

    /** Checks to see if two items are connected. */
    boolean isConnected(int p, int q);
}
```

Design goal: an efficient implementation given huge N, huge M, and interspersed calls.

### Example 1: `QuickFindDS` (lecture code)

```java
public class QuickFindDS implements DisjointSets {
    private int[] id;

    public QuickFindDS(int N) {
        id = new int[N];
        for (int i = 0; i < N; i++) {
            id[i] = i;          // each item starts in its own set
        }
    }

    public boolean isConnected(int p, int q) {
        return id[p] == id[q];
    }

    public void connect(int p, int q) {
        int pid = id[p];
        int qid = id[q];
        for (int i = 0; i < id.length; i++) {
            if (id[i] == pid) {
                id[i] = qid;
            }
        }
    }
}
```

Step by step:

1. **Constructor.** Allocates one `int[]` of length N on the heap; the instance variable `id` holds a reference to it. Each item is its own set, so `id[i] = i`. *(The slide's constructor writes `id[i] = -1;`. Taken literally that makes every item share the id `-1`, so `isConnected` would return true for every pair immediately, which contradicts the slide's own diagrams showing `id = [4,4,4,5,4,5,6]` and "everything is disconnected at start". Read it as `id[i] = i`. (extra context))*
2. **`isConnected`.** Exactly two array accesses plus a comparison, no loops, no allocation: Θ(1) always. This is the whole point of storing labels.
3. **`connect`.** Caching `pid` and `qid` **before** the loop is essential. If you wrote `if (id[i] == id[p])` inside the loop, then once the loop reaches index p and overwrites `id[p]`, the comparison target changes mid-flight and the remaining members of p's component are never relabeled.
4. The loop touches all N entries no matter what, so `connect` is Θ(N): N+2 to 2N+2 array accesses depending on how many entries need writing.

Tracing the lecture's `connect(2, 3)` on `{0,1,2,4}, {3,5}, {6}`:

```
before:  id = [4, 4, 4, 5, 4, 5, 6]
pid = id[2] = 4,  qid = id[3] = 5
every entry equal to 4 becomes 5
after:   id = [5, 5, 5, 5, 5, 5, 6]     components: {0,1,2,3,4,5}, {6}
isConnected(3, 6):  id[3] == id[6]  ->  5 == 6  ->  false
```

Note the direction of relabeling is arbitrary: one slide instead shows `[0,0,0,3,0,3,6]` becoming `[3,3,3,3,3,3,6]` for `connect(5, 2)`, that is relabeling q's component into p's. Either is correct, because the ids are just opaque labels. Only *equality* of ids carries meaning, never the id's numeric value.

### Example 2: `QuickUnionDS` (lecture code)

```java
public class QuickUnionDS implements DisjointSets {
    private int[] parent;

    public QuickUnionDS(int N) {
        parent = new int[N];
        for (int i = 0; i < N; i++) {
            parent[i] = -1;     // every item is its own root
        }
    }

    private int find(int p) {           // called "root(p)" on the earlier slides
        int r = p;
        while (parent[r] >= 0) {
            r = parent[r];
        }
        return r;
    }

    public boolean isConnected(int p, int q) {
        return find(p) == find(q);
    }

    public void connect(int p, int q) {
        int i = find(p);
        int j = find(q);
        parent[i] = j;
    }
}
```

Step by step:

1. **`find`.** `r` starts at `p` and climbs while `parent[r] >= 0`. A negative value means "I am a root", so the loop stops there and returns the root index. Runtime is proportional to the **depth of p**, which in the worst case is the height of the tree.
2. **`isConnected`.** Two climbs, compare the two roots. Same component if and only if same root. This is correct precisely because each tree has exactly one root and each item is in exactly one tree.
3. **`connect`.** Two climbs, then **one write**: `parent[i] = j` hangs p's entire tree under q's root. That single write is what makes union "quick"; all the work has migrated into `find`.

Box-and-pointer style reasoning for `connect(5, 2)` on `parent = [-1, 0, 1, -1, 0, 3, -1]`:

- The array is one object on the heap. Think of it as seven labeled cells. Cells 0, 3, 6 hold `-1`, so they are the three roots. Cell 1 holds 0, cell 2 holds 1, cell 4 holds 0, cell 5 holds 3. Following the values as arrows: 2 -> 1 -> 0 (root), 4 -> 0 (root), 5 -> 3 (root).
- `find(5)`: r = 5, `parent[5] = 3 >= 0`, so r = 3; `parent[3] = -1 < 0`, stop. Returns 3.
- `find(2)`: r = 2 -> 1 -> 0, returns 0. Two hops.
- `parent[3] = 0`, so `parent` becomes `[-1, 0, 1, 0, 0, 3, -1]`. Node 3 is no longer a root, and **both** 3 and its child 5 are now in 0's component without 5's own entry changing at all. This is exactly why we must reparent the *root* and not the queried node: writing `parent[5] = 2` would have merged 5 while stranding 3.

The worst case, from the slides: `connect(4, 3); connect(3, 2); connect(2, 1); connect(1, 0)`. Each call takes the first argument's root and hangs it beneath the second argument's root, producing a stick 0 -> 1 -> 2 -> 3 -> 4. After M such operations the height is M, and with N items `find` costs Θ(N), so both `connect` and `isConnected` are Θ(N) worst case.

*(extra context) A subtle bug in this lecture-slide version: if p and q are already connected, then `i == j` and `parent[i] = j` writes `parent[i] = i`, a non-negative value, so `i` stops looking like a root and `find` spins forever (`r = parent[r] = r`). Production code guards with `if (i == j) { return; }` at the top of `connect`. Worth knowing, since `connect(4, 2)` on an already-connected pair appears in the lecture's own example sequence.*

### Example 3: `WeightedQuickUnionDS`

The slides describe the changes rather than giving code: keep using `parent[]`, leave `isConnected` untouched, and make `connect` track sizes. Here is the separate-size-array version. *(extra context: code is a faithful reconstruction of the stated rules, not printed on the slides.)*

```java
public class WeightedQuickUnionDS implements DisjointSets {
    private int[] parent;
    private int[] size;

    public WeightedQuickUnionDS(int N) {
        parent = new int[N];
        size = new int[N];
        for (int i = 0; i < N; i++) {
            parent[i] = -1;
            size[i] = 1;        // every singleton tree has weight 1
        }
    }

    private int find(int p) {
        int r = p;
        while (parent[r] >= 0) {
            r = parent[r];
        }
        return r;
    }

    public boolean isConnected(int p, int q) {   // unchanged from QuickUnion
        return find(p) == find(q);
    }

    public void connect(int p, int q) {
        int i = find(p);
        int j = find(q);
        if (i == j) {
            return;                              // already connected
        }
        if (size[i] < size[j]) {                 // smaller root goes under larger
            parent[i] = j;
            size[j] += size[i];
        } else {                                 // ties broken arbitrarily
            parent[j] = i;
            size[i] += size[j];
        }
    }
}
```

The only semantic difference from `QuickUnionDS` is the `if`: we no longer always hang p's root under q's root, we hang the lighter root under the heavier one. Note that `size[x]` is only meaningful when x is a root, and that the winner's size must be updated to the combined total.

**Lecture quiz, `connect(2, 5)`.** Given a tree rooted at 0 containing {0, 1, 2, 4} of height 2, and a tree rooted at 3 containing {3, 5}:

- Option A: make 5's root (3) a child of 2's root (0). Result height 2.
- Option B: make 2's root (0) a child of 5's root (3). Result height 3.

A is better, and the weight rule picks it automatically because 0's tree has 4 items versus 3's tree with 2.

**Lecture quiz, `connect(3, 8)`.** Given

```
parent   -1   0   0   0   0   0   -1   6   6   8
          0   1   2   3   4   5    6   7   8   9
```

- 0's tree is {0, 1, 2, 3, 4, 5}, weight 6. 6's tree: 7 -> 6, 8 -> 6, 9 -> 8, so {6, 7, 8, 9}, weight 4.
- `find(3)` returns 0 (one hop). `find(8)` returns 6 (one hop).
- Weight 4 < weight 6, so the *smaller* root 6 gets reparented: **`parent[6]` changes** (answer D), giving

```
parent   -1   0   0   0   0   0    0   6   6   8
```

Note carefully that neither `parent[3]` nor `parent[8]` changes, even though 3 and 8 are the arguments. That is the single most common wrong answer on this style of question.

**The weighted worst case.** To grow the height as fast as possible you must always merge two trees of *equal* height (with unequal heights, the shorter tree slides under the taller one and the height does not grow at all). Merging two height-h trees of size 2^h each yields height h+1 with size 2^(h+1). So height h requires at least 2^h nodes, giving max height Θ(log N) and O(log N) operations.

### Example 4: Path compression

The clever idea: when `find` climbs a path, tie every node it saw directly to the root.

```java
    private int find(int p) {                // extra context: reconstruction
        int r = p;
        while (parent[r] >= 0) {             // 1. climb to the root
            r = parent[r];
        }
        int curr = p;                        // 2. second pass: reparent the path
        while (curr != r) {
            int next = parent[curr];
            parent[curr] = r;
            curr = next;
        }
        return r;
    }
```

Everything else stays the same. Because both `connect` and `isConnected` call `find`, both operations compress. The saved `next` before overwriting `parent[curr]` is essential, otherwise you destroy the pointer you still need to continue walking.

**Lecture trace 1: `isConnected(15, 10)` on the WQU worst-case tree.** Reconstructing the diagram: root 0 has children 1, 2, 3, 4; 1 has children 5, 6; 2 has children 7, 8; 3 has children 9, 10; 5 has children 11, 12; 6 has child 13; 8 has child 14; 11 has child 15.

- `find(15)` climbs 15 -> 11 -> 5 -> 1 -> 0. Compression reparents 15, 11, 5, and 1 all to 0. Node 12 stays a child of 5 (it was never on the path, and 5 is now a child of the root, so 12's depth drops from 3 to 2 for free). Node 6 stays a child of 1, so 13 stays under 6.
- `find(10)` climbs 10 -> 3 -> 0. Compression reparents 10 to 0 (3 was already a root's child).
- Resulting shape: 0's children are 15, 11, 5, 1, 2, 3, 10, 4; then 12, 6, 7, 8, 9 at depth 2; then 13, 14 at depth 3.
- Return value: both roots are 0, so `true`. Note that a *query*, not a mutation, permanently restructured the data. That is fine because the partition itself is unchanged; only the representation moved.

**Lecture trace 2: `isConnected(14, 13)` on that result.** With 13 -> 6 -> 1 -> 0 and 14 -> 8 -> 2 -> 0:

- `find(13)` reparents 13 and 6 to 0.
- `find(14)` reparents 14 and 8 to 0.
- Resulting root children: 15, 11, 5, 1, 13, 6, 14, 8, 2, 3, 10, 4, with only 12 (under 5), 7 (under 2), and 9 (under 3) left at depth 2 and nothing deeper.
- Returns `true`.

Two operations turned a height-4 tree into a height-2 tree. The slides also point out that the starting tree in that exercise is **impossible to generate** if path compression has been on the whole time: certain deep structures simply cannot survive the finds required to build them. The claim is asserted, not proven, in lecture.

---

## Common Pitfalls

1. **Storing the connections instead of the components.** `List<Integer[]>` of edges makes `connect` trivial and `isConnected` a graph search. The API's questions, not the input's format, should drive the representation.
2. **Reparenting the queried node instead of its root.** In quick union, `parent[p] = q` is wrong: p's ancestors (and their other subtrees) get stranded. Always `parent[find(p)] = find(q)`.
3. **In QuickFind's `connect`, comparing against `id[p]` inside the loop.** Once index p is overwritten the comparison target mutates and the relabel silently stops halfway. Cache `pid` and `qid` first.
4. **Assuming the id value means something.** In QuickFind, the id is an arbitrary label; only equality matters. Relabeling p's component into q's or vice versa are both correct.
5. **Assuming the id array and the parent array look alike.** For `{0,1,2,4}, {3,5}, {6}`, QuickFind might store `[4,4,4,5,4,5,6]` while QuickUnion stores `[-1,0,1,-1,0,3,-1]`. Same components, entirely different meanings per slot. Always check which structure you are looking at before reading an array off an exam page.
6. **Forgetting the `-1` root sentinel convention.** The slides use `-1` for roots (or `-weight` for weighted roots); the optional textbook uses "parent of a root is itself". Under the self-parent convention, `while (parent[r] >= 0)` never terminates. Mixing conventions breaks `find`.
7. **Not handling `connect(p, q)` when p and q are already connected.** The bare slide code writes `parent[i] = i`, which destroys the root sentinel and makes `find` loop forever. Guard with `if (i == j) return;`.
8. **Weighting by "which node is deeper" or by the argument order.** The rule is about the **sizes of the two whole trees**, and the entry that changes is the **lighter tree's root**, not either argument.
9. **Forgetting to update the size of the surviving root.** `size[i] += size[j]` (or the `-weight` equivalent) must accompany every link, or the weight rule degrades to arbitrary linking.
10. **Confusing weighting with height-tracking.** Weighted quick union does not guarantee the locally shortest result at every step; it guarantees Θ(log N) height overall. Tracking height gives the same asymptotics with messier code and gets harder once path compression is added.
11. **Claiming Θ where only O holds.** `QuickUnionDS.connect` can be constant time or linear time depending on tree shape, so O(N) is the honest statement. Conversely, `QuickFindDS.connect` really does always scan the array, so Θ(N) is correct there.
12. **Saying path compression is Θ(1) per operation.** It is O(lg\* N), or more tightly O(α(N)), amortized. Constant "for all realistic inputs" is not the same as constant.
13. **Believing path compression alone fixes quick union.** The lecture's final structure is weighting *and* compression together; the table entry O(M α(N)) is for `WeightedQuickUnionWithPathCompressionDS`.
14. **Confusing N and M.** N is the number of elements (fixed at construction); M is the number of operations. Totals like O(M log N) mix both, and the constructor contributes its own Θ(N).

---

## Likely Exam Points

**1. Trace a `connect` on QuickFind and report the resulting array.**

*Q:* `QuickFindDS` with `id = [1, 1, 3, 3, 5, 5]`. Show `id` after `connect(0, 4)` and then give `isConnected(1, 2)`.

*A:* `pid = id[0] = 1`, `qid = id[4] = 5`. Every entry equal to 1 becomes 5: `id = [5, 5, 3, 3, 5, 5]`. Then `isConnected(1, 2)` compares `id[1] == id[2]`, that is `5 == 3`, so **false**. (Components: {0, 1, 4, 5} and {2, 3}.)

**2. Which single `parent[]` entry changes under weighted quick union?**

*Q:* `parent = [-1, 0, 0, 0, -1, 4, 4, 6]`. Which entry changes on `connect(5, 2)`?

*A:* `find(5)` gives 4; 4's tree is {4, 5, 6, 7}, weight 4. `find(2)` gives 0; 0's tree is {0, 1, 2, 3}, weight 4. Weights tie, so the tie is broken arbitrarily: either `parent[4] = 0` or `parent[0] = 4`. The changed entry is one of the two **roots**, `parent[4]` or `parent[0]`, and never `parent[5]` or `parent[2]`.

**3. Give the worst-case height, and the input that achieves it.**

*Q:* For N items, what is the worst-case height of a `QuickUnionDS` forest, and of a `WeightedQuickUnionDS` forest? Give a call sequence achieving the quick union worst case.

*A:* Quick union: height N-1, that is Θ(N), achieved by always hanging the first argument's root below the second's, for example `connect(4,3); connect(3,2); connect(2,1); connect(1,0)`, producing the stick 0 -> 1 -> 2 -> 3 -> 4. Weighted quick union: Θ(log N), because the height only increases when two equal-height trees merge, and that doubles the node count, so height h needs at least 2^h nodes.

**4. Fill in the performance table.**

*Q:* Give constructor / `connect` / `isConnected` runtimes for QuickFind, QuickUnion, and WeightedQuickUnion, using Θ or O as appropriate.

*A:* QuickFind: Θ(N) / Θ(N) / Θ(1). QuickUnion: Θ(N) / O(N) / O(N). WeightedQuickUnion: Θ(N) / O(log N) / O(log N). O rather than Θ for the union variants because the cost depends on tree shape and ranges from constant to the bound.

**5. Total runtime for M operations.**

*Q:* N = 10⁶ elements, M = 10⁶ interspersed calls. Give the total asymptotic runtime with `ListOfSetsDS` and with `WeightedQuickUnionDS`, in terms of N and M.

*A:* `ListOfSetsDS`: O(N + MN) = O(MN). `WeightedQuickUnionDS`: O(N + M log N). This is the lecture's 30-years-versus-6-seconds comparison at N = M = 10⁹.

**6. Why can't you just set `parent[p] = q`?**

*Q:* In quick union, explain why `connect(5, 2)` sets `parent[find(5)] = find(2)` rather than `parent[5] = 2`.

*A:* Because 5's ancestors are part of 5's component. If 3 is 5's parent and 5's only path to the rest of its tree, writing `parent[5] = 2` moves 5 into 2's component while leaving 3 (and anything else hanging off 3) behind: we "lose 3". Reparenting the root carries the entire subtree along and preserves the invariant that one tree equals one component.

**7. Draw the tree after path compression.**

*Q:* `parent` encodes 0 as root with child 1; 1 with child 2; 2 with child 3; 3 with child 4. Under WQU with path compression, draw the tree after `isConnected(4, 0)`.

*A:* `find(4)` climbs 4 -> 3 -> 2 -> 1 -> 0, then reparents 4, 3, 2, and 1 directly to 0. `find(0)` is already the root and changes nothing. Result: 0 with four children 1, 2, 3, 4, height 1. Returns `true`. Note that a read-only-looking query mutated the structure.

**8. Weight versus height, and why weight wins.**

*Q:* Why does CS 61B weight by size rather than by height?

*A:* Both give Θ(log N) worst-case height, so there is no asymptotic gain from heights, and height tracking becomes difficult to maintain once path compression starts rearranging the tree (compression changes heights but never changes sizes). Size gives the same bound with simpler code.

**9. Big O versus Big Theta as a concept question.**

*Q:* True or false: `QuickUnionDS.isConnected` is Θ(N). Justify.

*A:* False as a description of all cases. Its worst case is Θ(N), but on a flat forest it is constant time, so there is no single f(N) that bounds it above and below for all inputs. The correct blanket statement is O(N). (It is fine to say "the worst-case runtime is Θ(N)", since that phrase fixes one specific input family.)

**10. What lg\* / α buy you.**

*Q:* What is lg\*(65536), and what is the total cost of M operations on N elements with WQU plus path compression?

*A:* lg\*(65536) = 4, since 65536 -> 16 -> 4 -> 2 -> 1 is four presses of log₂. Total cost is O(M lg\* N), tightened to O(M α(N)) where α is the inverse Ackermann function. Both are at most about 5 for any realistic N, so each operation is effectively constant but not formally so.

**11. Identify the representation from an array.**

*Q:* The array `[-1, 0, 1, -1, 0, 3, -1]` is given. How many connected components are there, and is 2 connected to 4?

*A:* Negative entries mark roots, and there are three of them (indices 0, 3, 6), so **three components**. 2 -> 1 -> 0, and 4 -> 0, so both have root 0: **yes, connected**. Components are {0, 1, 2, 4}, {3, 5}, {6}.

---

## Summary

- **Problem:** Dynamic Connectivity. Support interspersed `connect(p, q)` and `isConnected(p, q)` over N items (integers 0 to N-1, all disconnected initially), where connectivity is transitive. Huge N, huge M, no lookahead.
- **Key reframing:** do not record how items are connected; record **connected components**. Components are disjoint and only ever merge.
- **`ListOfSetsDS`** (`List<Set<Integer>>`): intuitive, complicated, and slow. Θ(N) constructor, O(N) `connect`, O(N) `isConnected`, because finding anything means scanning up to N sets. Lesson: the representation doomed it.
- **`QuickFindDS`** (`int[] id`, value = set number): `isConnected` is Θ(1) (two array accesses), but `connect` must relabel a whole component with a full array scan, so Θ(N). Cache `pid`/`qid` before looping.
- **`QuickUnionDS`** (`int[] parent`, `-1` marks roots): `connect` does two `find`s and **one write**, `parent[find(p)] = find(q)`. Cost moves into `find`, whose runtime is the node's depth. Worst case a stick of height M, so both operations are O(N) and Θ(N) in the worst case, potentially worse than QuickFind.
- **`WeightedQuickUnionDS`:** track each tree's size/weight and always link the **smaller root under the larger**, ties arbitrary. `isConnected` unchanged. Heights become Θ(log N) (height h needs ≥ 2^h nodes), so both operations are O(log N). Store weights either as `-weight` at roots or in a separate `size` array. Weight beats height because it is equally good asymptotically and simpler, especially once compression is added.
- **`WeightedQuickUnionWithPathCompressionDS`:** on every `find` (so during both `connect` and `isConnected`), reparent every node on the walked path directly to the root. Cost is asymptotically free relative to the climb already performed; trees flatten toward height 1 as M grows. This is the standard modern implementation.
- **Totals for M operations on N elements:** ListOfSets O(NM), QuickFind Θ(NM), QuickUnion O(NM), WQU O(M log N), WQU+PC O(M α(N)). At N = M = 10⁹ that is roughly 30 years versus 6 seconds.
- **Analysis footnote (CS 170 territory):** amortized O(lg\* N), tightened to O(α(N)) by Tarjan (1975, at Berkeley). Both are ≤ 5 for realistic inputs, and both are "not quite constant".
- **Notation discipline:** use Θ when the cost is fixed regardless of shape (QuickFind), and O when it varies between constant and the bound (the union family).
- **Meta-lesson:** asymptotic performance and code complexity both follow from the choice of underlying abstraction, and a good data structure can turn an infeasible problem into a trivial one.
