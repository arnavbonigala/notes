<!-- Wed, Oct 21, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 24: Minimum Spanning Trees

## Overview

This lecture moves from shortest paths to a different graph optimization problem: given a connected, undirected, weighted graph, find the cheapest set of edges that keeps everything connected. That object is a **minimum spanning tree (MST)**: a subgraph that is connected, acyclic, and touches every vertex, with minimum total edge weight. The entire lecture is built on one idea, the **cut property**: for any split of the vertices into two non-empty sets, the minimum-weight edge crossing that split must be in the MST. From the cut property we get a generic (but impractical) MST algorithm, and then two efficient specializations of it: **Prim's algorithm**, which grows a single tree outward and is essentially Dijkstra's with "distance from the tree" replacing "distance from the source" (O(E log V) with a priority queue), and **Kruskal's algorithm**, which sorts all edges and greedily adds any edge that does not create a cycle, using a weighted quick union with path compression to detect cycles (O(E log E), or O(E log* V) if the edges arrive pre-sorted). The lecture opens with a warm-up on cycle detection, which is exactly the subroutine Kruskal's needs.

---

## Key Concepts

### 1. Warm-up: detecting a cycle in an undirected graph

Before MSTs, the lecture asks: given a connected undirected graph, how do you tell whether it contains a cycle? Two answers, both using tools already covered in the course.

**Approach 1: DFS with marking.** Do DFS from an arbitrary vertex. If you ever reach a vertex that is already marked, you have found a cycle. There is one subtlety the lecture flags explicitly: when DFS is at `B` and looks back at its parent `A`, `A` is marked, but that is not a cycle. The fix is to not count the node you came from.

Runtime: DFS is O(V + E) in general, but here we can tighten it to **O(V)**. The reasoning given in lecture: a tree (connected, acyclic) has exactly V - 1 edges, and there is no graph with V or more edges that is acyclic. So we never need to examine more than V edges before either finishing or finding a cycle. E is bounded by V for this procedure, so O(V + E) collapses to O(V).

**Approach 2: WeightedQuickUnionUF (disjoint sets).** Start with every vertex in its own set. For each edge `v-w`:
- If `connected(v, w)` is already true, then `v` and `w` are already joined through other edges, so adding this edge closes a cycle: return true.
- Otherwise, `union(v, w)`.

The crucial distinction the lecture hammers on: "connected" here does **not** mean "there is an edge between them in the graph." It means "we have already unioned them into the same set." An edge can exist in the graph while its endpoints are still separate islands in the disjoint-set structure.

Walking the lecture's example graph (vertices A, B, C, D, E, F, G): check A-C (not yet unioned, so union them); check E-F (not unioned, union); check F-G (G is new to the component, union, so now {E, F, G} are one set); check E-G (already in the same set, so return true: cycle).

Runtime: O(V + E·α(V)) with path compression, where α is the inverse Ackermann function from the Disjoint Sets lecture, simplifying to O(V·α(V)) by the same V - 1 edge argument.

**Why bother with the slower one?** The lecture answers this at the very end. DFS is faster for a one-shot cycle check on a single fixed graph. But Kruskal's algorithm needs to repeatedly ask "would adding this edge create a cycle?" as a growing collection of components merges. Rerunning DFS from scratch every time is wasteful; the disjoint-set structure *remembers* the component structure across queries. That is why WQU, not DFS, is the subroutine inside Kruskal's.

### 2. Spanning trees and minimum spanning trees

Informally: a spanning tree is a subgraph that is a tree and that touches every single vertex. A minimum spanning tree is the spanning tree whose edge weights sum to the smallest possible total.

The lecture builds this up by rejecting candidates. Three properties must all hold, and each failure mode was demonstrated:

- **Includes all vertices (spanning).** A candidate that left out vertex F was rejected: "it's not even spanning." How close to minimum it is becomes irrelevant.
- **Acyclic.** A candidate containing the cycle A-B-E-C was rejected: trees do not have cycles.
- **Connected.** A candidate that split into pieces was rejected.
- **Minimum weight.** A candidate that was a genuine spanning tree but used the weight-15 and weight-11 edges was rejected for not being minimum. The lecture noted these were "pretty serious options," meaning the tree was valid but expensive.

The clicker question (three candidate subgraphs, answer: only #3) tested exactly these three rejections: #1 fails connectedness, #2 contains a cycle, #3 is a spanning tree.

### 3. Intuition for greed: the lightest edge at a vertex

Before the formal cut property, the lecture asked: looking at vertex F, which edge do we know belongs in the MST? Answer: the weight-1 edge F-G, the lightest edge touching F.

The informal argument given: F has to be connected somehow. If a spanning tree reached F only through the heavier edges, you could delete one of those heavier edges and substitute the weight-1 edge, which still connects F, producing a cheaper spanning tree. So greed works here. The cut property generalizes this from "edges at a single vertex" to "edges crossing any split at all."

### 4. The cut property

**A cut** is an assignment of a graph's nodes to two non-empty sets. The lecture stressed that a cut is *just a coloring*: the two sets do **not** have to be geometrically grouped, left/right, or contiguous in any way. Any partition into two non-empty groups is a legal cut.

**A crossing edge** is an edge connecting a node in one set to a node in the other.

**Cut property:** given any cut, the minimum-weight crossing edge is in the MST. The lecture added the stronger statement: if there are multiple MSTs, that minimum crossing edge is in *every* one of them. For the rest of the lecture, assume edge weights are unique (this makes the MST unique and the argument clean; the lecture noted non-unique weights appear in practice and are fine, just messier to reason about).

### 5. Cut property in action

The lecture's clicker example: a graph on vertices 0-7 with a sorted edge list, cut = {2, 3, 5, 6} versus {0, 1, 4, 7}.

```
0-7  0.16     5-7  0.28     4-5  0.35     6-2  0.40
2-3  0.17     1-3  0.29     1-2  0.36     3-6  0.52
1-7  0.19     1-5  0.32     4-7  0.37     6-0  0.58
0-2  0.26     2-7  0.34     0-4  0.38     6-4  0.93
```

Scan the sorted list top to bottom and take the first edge that actually crosses:
- `0-7` (0.16): both 0 and 7 are in the second set. Does not cross. (The lecture used this to make the point that the globally lightest edge need not cross a given cut.)
- `2-3` (0.17): both in {2,3,5,6}. Does not cross.
- `1-7` (0.19): both in the second set. Does not cross.
- `0-2` (0.26): 0 is outside, 2 is inside. **Crosses.**

Answer: **0-2**, and because the list is sorted, 0-2 is the minimum-weight crossing edge, so it must be in the MST.

### 6. Why the cut property is true (proof by contradiction)

Let X be an MST. Suppose the minimum-weight crossing edge `e` for some cut is **not** in X.

1. Adding `e` to X creates a cycle (X is spanning and connected, so there is already a path between `e`'s endpoints).
2. Since `e`'s endpoints are on opposite sides of the cut, that cycle must cross the cut again. So some other edge `f` in X is also a crossing edge and lies on the cycle.
3. Because `e` is the *minimum* crossing edge and weights are unique, weight(`e`) < weight(`f`).
4. Remove `f`, add `e`. The result is still spanning (removing `f` from the cycle cannot disconnect anything; `e` still bridges the two sides) and has strictly lower total weight.
5. That contradicts X being minimum.

The lecture's informal gloss: "if F costs more than E, then remove F, and E still connects the two sides." Same substitution argument as the single-vertex intuition, generalized.

### 7. Generic MST algorithm (and why it is impractical)

```
Start with no edges in the MST.
Repeat until V - 1 edges:
    Find a cut that has no crossing edges currently in the MST.
    Add the smallest crossing edge of that cut to the MST.
```

This is correct, but it needs a way to *find* a useful cut. Picking cuts randomly is a bad idea: with V vertices there are on the order of 2^V cuts, exponentially many, and you would keep rediscovering edges you already have. Prim's and Kruskal's are both this generic algorithm with a clever, cheap rule for choosing the cut.

### 8. Prim's algorithm (conceptual version)

```
Start from some arbitrary start node.
Repeatedly add the shortest edge that has exactly one endpoint inside
    the MST under construction.
Repeat until V - 1 edges.
```

**Why it is correct:** it is the generic algorithm with the cut always chosen as *(vertices already in the tree) vs. (everything else)*. In the lecture's coloring, white vertices are everything connected to the start so far and green vertices are everything not yet connected. Every step adds the minimum-weight white-to-green edge, which is the minimum crossing edge of that cut, so the cut property guarantees it belongs in the MST. No crossing edge is already in the tree, by construction, so the cut is always a legal choice for the generic algorithm.

**The lecture's trace** on the graph with start A (edges: A-B 2, A-C 1, B-C 3, B-D 11, B-E 5, C-E 1, C-F 15, D-E 2, D-G 1, E-F 4, E-G 3, F-G 1):

| Step | Tree so far | Candidate crossing edges | Chosen |
|---|---|---|---|
| 1 | {A} | A-B (2), A-C (1) | **A-C (1)** |
| 2 | {A, C} | A-B (2), B-C (3), C-E (1), C-F (15) | **C-E (1)** |
| 3 | {A, C, E} | A-B (2), B-C (3), B-E (5), D-E (2), E-F (4), E-G (3), C-F (15) | tie at 2: **A-B** or **D-E** (lecture took A-B; "it doesn't matter, any of them could work") |
| 4 | {A, B, C, E} | includes B-D (11), D-E (2), E-G (3), ... | **D-E (2)** |
| 5 | {A, B, C, D, E} | D-G (1), E-G (3), E-F (4), F-G (1), ... | **D-G (1)** |
| 6 | {A, B, C, D, E, G} | F-G (1), E-F (4), C-F (15) | **F-G (1)** |

Six edges for seven vertices: V - 1, done. The lecture explicitly flagged the tie in step 3 as a real tie with two valid choices (a consequence of weights not being unique in this particular example).

**Why the naive version is slow:** at each step you iterate over *all* edges leaving the tree (the "magenta" edges in the slides). As the tree grows, that set approaches the whole edge set, and you re-examine the same edges over and over. The lecture: "at some point we almost get the entire graph... that's a lot of edges to just loop through."

### 9. Optimized Prim's: Prim's is Dijkstra's with a different distance

This is the central comparison of the lecture.

> **Prim's and Dijkstra's are exactly the same algorithm, except Dijkstra's considers "distance from the source" and Prim's considers "distance from the tree."**

- **Visit order.** Dijkstra's visits vertices in order of distance from the source. Prim's visits vertices in order of distance from the MST under construction.
- **Relaxation.** Dijkstra's relaxation judges an edge by the resulting distance *to the source*. Prim's judges an edge by its own weight, that is, the distance *to the tree*.

Concretely: `distTo[w]` in Dijkstra's is `distTo[v] + e.weight()`. In Prim's it is just `e.weight()`. The lecture made this vivid with the C-F edge of weight 15, reached from C which is at distance 1 from A: Dijkstra's would record **16** for F (1 + 15), while Prim's records **15**, because Prim's does not care about any particular source, only "how far is this vertex from the tree."

**The lecture's optimized trace** (same graph, start A):

1. Set `distTo[A] = 0`, every other `distTo = infinity`, insert all vertices into the PQ. A is at the front.
2. **delMin A.** Scan A's edges: `distTo[B] = 2`, `edgeTo[B] = A-B`; `distTo[C] = 1`, `edgeTo[C] = A-C`. C jumps the queue. Nothing is committed yet; commitment happens only on delMin.
3. **delMin C.** Commit A-C. Scan C's edges: B-C has weight 3, but `distTo[B]` is already 2, so no update ("B is in the queue, so we don't need to update its priority": this is exactly where the optimized version saves work over naive Prim's, which would re-examine that edge every round). Then `distTo[E] = 1` via C-E, and `distTo[F] = 15` via C-F.
4. **delMin E.** Commit C-E. Scan E's edges: `distTo[D] = 2` (D-E), `distTo[G] = 3` (E-G), `distTo[F] = 4` (E-F, which beats the previous 15, so update). B-E has weight 5, worse than B's current 2, so no update.
5. Continue delMin-ing. The next minimum is a 2 (either A-B or D-E). The lecture noted that the PQ version happened to pop a different one of the tied edges than the hand-run version did, which is fine: both produce a valid MST.
6. Keep going until the fringe is empty. `edgeTo[]` holds the MST.

The payoff: instead of re-scanning a growing cloud of crossing edges, each vertex keeps a single best-known distance to the tree, and each edge is only considered when one of its endpoints is popped.

### 10. Prim's runtime

Assuming a binary-heap PQ where all operations are O(log V):

| Operation | # Operations | Cost per operation | Total |
|---|---|---|---|
| PQ add | V | O(log V) | O(V log V) |
| PQ delMin | V | O(log V) | O(V log V) |
| PQ decreasePriority | O(E) | O(log V) | O(E log V) |

Overall: O(V log V + V log V + E log V). Assuming E > V (which holds for connected graphs), this is **O(E log V)**, the same as Dijkstra's. Every vertex is inserted once and deleted once; every edge can in the worst case trigger one decreasePriority, and edges dominate.

The lecture noted that `decreasePriority` was not itself implemented in lecture, but can be made to run in O(log V).

### 11. Kruskal's algorithm

```
Initially mark all edges gray.
Consider edges in increasing order of weight.
Add edge to MST (mark black) unless doing so creates a cycle.
Repeat until V - 1 edges.
```

Pure greed over a sorted edge list, with a cycle check as the only veto.

**Why it is correct:** also a special case of the generic algorithm. Suppose we are about to add edge `e = v-w`. Define the cut as *(all vertices currently connected to v)* versus *(everything else)*. Then:
- No crossing edge of this cut is already in the MST, because if one were, `w`'s side would already be joined to `v`'s side, and `v`'s side is by definition everything connected to `v`.
- No crossing edge has lower weight than `e`, because we are processing edges in increasing weight order, so every lighter edge has already been considered.

So `e` is the minimum crossing edge of a cut with no crossing edges in the tree: exactly what the generic algorithm asks for.

**The lecture's basic trace** (same graph): sorted, the weight-1 edges come first.
- A-C: no cycle, add.
- C-E: no cycle, add.
- D-G: no cycle, add.
- F-G: no cycle, add.
- A-B (weight 2): no cycle, add.
- B-E (weight 5, reached later in sorted order): A, C, E, B are all already connected, so this **would** create a cycle. Skip ("leave that on the table").
- Continue until V - 1 = 6 edges are collected.

Notice the components grow in several places at once, not from a single seed.

### 12. Optimized Kruskal's: getting the cycle check and the sort right

Two implementation choices, both from material already covered:

**Sorting the edges.** Either sort them up front, or dump them all into a `MinPQ` and `delMin` repeatedly, which yields them in increasing weight order. A student asked about this directly; the lecture gave both options.

**Cycle detection.** Use the warm-up's Approach 2, `WeightedQuickUnionPC` (weighted quick union with path compression). For each candidate edge `v-w`:
- If `uf.connected(v, w)`, then `v` and `w` are already in the same component, so this edge closes a cycle: skip it.
- Otherwise `uf.union(v, w)` and add the edge to the MST.

The lecture re-emphasized the distinction from the warm-up: `connected(v, w)` asks "have we already unioned these?", not "is there an edge between them?" There is obviously an edge between them, that is why we are looking at this edge at all.

Tracing the disjoint-set state alongside the MST:
- A-C: not connected, union. Sets: {A, C}.
- C-E: not connected, union. Sets: {A, C, E}. ("Now realize that these are not edges... this is the combined set of elements A, C, and E.")
- D-G: not connected, union. Sets: {A, C, E}, {D, G}.
- F-G: F is not in {D, G}, union. Sets: {A, C, E}, {D, G, F}.
- A-B: B is not yet in any union with A, union. Sets: {A, C, E, B}, {D, G, F}.
- B-E: **connected already** (both in {A, B, C, E}), so skip: this is the cycle.
- Continue; the next edge joining the two big components is accepted, and once V - 1 edges are in hand we stop.

### 13. Kruskal's vs. Prim's (visual character)

The lecture contrasted the animations:

- **Prim's** grows exactly one tree. It starts from a single seed and expands, always attaching the vertex nearest to the current tree. That nearest vertex can be anywhere along the tree's frontier, so the growth looks like a sprawling transit network spreading out from one place. (Compared against Dijkstra's, which grows in neat expanding radial rings from the source.)
- **Kruskal's** looks completely different. It scatters tiny edges all over the graph at once, creating many small connected components (so small they look like dots at first). As it proceeds, these components merge, and merges that would close a cycle get rejected. Eventually everything coalesces into one tree.

Same output (the MST, unique if weights are unique), very different process.

### 14. Kruskal's runtime

| Operation | Number of times | Time per operation | Total |
|---|---|---|---|
| Insert (into PQ) | E | O(log E) | O(E log E) |
| Delete minimum | O(E) | O(log E) | O(E log E) |
| `union` | O(V) | O(log* V) | O(V log* V) |
| `isConnected` | O(E) | O(log* V) | O(E log* V) |

Overall: **O(E log E)**, dominated by the priority queue work.

Two important notes from the slides:

- **Note 1:** with a **pre-sorted** list of edges instead of a PQ, you just iterate through the list in O(E) time, so the total becomes O(E + V log* V + E log* V) = **O(E log\* V)**, which is very nearly linear.
- **Note 2:** E < V², so log E < log V² = 2 log V, hence **O(E log E) = O(E log V)**. So Kruskal's on unsorted edges is slower than Prim's in practice on worst-case input, but **not asymptotically slower**.

(The slides also mention, as a forward reference to the HeapSort lecture, that building the heap of E edges can be done in O(E) via bottom-up heapification.)

### 15. Summary table of shortest-path and MST algorithms

| Problem | Algorithm | Runtime (if E > V) | Notes |
|---|---|---|---|
| Shortest Paths | Dijkstra's | O(E log V) | Fails for negative weight edges |
| MST | Prim's | O(E log V) | Analogous to Dijkstra's |
| MST | Kruskal's | O(E log E) | Uses WQUPC |
| MST | Kruskal's with pre-sorted edges | O(E log* V) | Uses WQUPC |

### 16. MST applications

The lecture and slides mention: networks of power lines connecting buildings, telephone lines, transit systems like BART, old-school handwriting recognition, and medical imaging (for example, the arrangement of nuclei in cancer cells).

The lecture also spent time on a slime mold that reproduces the Tokyo rail map when sugar cubes are placed at station locations: it expands radially in all directions, then prunes itself back to efficient paths. Important caveat the lecturer gave: that result is technically a **Steiner tree**, a generalization of the MST where the structure is allowed to create its own extra junction points/vertices. If instead you confine the mold to a fixed graph (a maze with edges) with food at the vertices, it will find the MST itself.

### 17. Extra (covered in the web videos, not covered live): MST vs. SPT

The slides include a section the lecture explicitly skipped in person but which exists in the web videos.

Graph: A-B 2, B-C 2, C-D 2, A-D 3.

The MST is {A-B, B-C, C-D} (total 6), dropping A-D (3) rather than one of the 2s.

Is that MST also a shortest paths tree? It depends on the source:
- From **D**: no. The SPT from D would use D-A (3) directly rather than the length-6 path D-C-B-A.
- From **B**: yes. The MST is exactly the SPT from B.
- From A: the SPT uses A-B and A-D, which is not the MST.
- From C: not the MST either.

The takeaways stated on the slides:
- A shortest paths tree **depends on the start vertex**, because it tells you how to get from a source to EVERYTHING.
- There is **no source** for an MST.
- Nonetheless, the MST sometimes happens to be an SPT for a specific vertex.

The slides also give a graph where **no** vertex's SPT equals the MST: a ladder-shaped graph with two weight-2 rungs in the middle and weight-1 edges elsewhere. The MST includes only 2 of the weight-2 edges, but any SPT must include at least 3 of them, so no source works. Answer to "Is there a node whose SPT is also the MST?": **No.**

### 18. Extra (web videos only): state of the art

Can we beat O(E log V)? History of compare-based MST algorithms:

| Year | Worst case | Discovered by |
|---|---|---|
| 1975 | E log log V | Yao |
| 1984 | E log* V | Fredman-Tarjan |
| 1986 | E log(log* V) | Gabow-Galil-Spencer-Tarjan |
| 1997 | E α(V) log α(V) | Chazelle |
| 2000 | E α(V) | Chazelle |
| 2002 | optimal (but the complexity of that optimum is unknown) | Pettie-Ramachandran |

---

## Definitions

**Spanning tree:** given an undirected graph G, a subgraph T of G that (1) is connected, (2) is acyclic, and (3) includes all of the vertices of G. Properties (1) and (2) make it a tree; property (3) makes it spanning.

**Minimum spanning tree (MST):** a spanning tree of minimum total edge weight, that is, the spanning tree minimizing the sum of its edge weights.

**Tree (as used here):** a connected, acyclic graph. (One of several equivalent definitions; the equivalences are proved in CS 70.) A tree on V vertices has exactly V - 1 edges.

**Cut:** an assignment of a graph's nodes to two non-empty sets. The two sets need not be contiguous or geometrically grouped; any 2-coloring of the vertices with both colors used is a cut.

**Crossing edge:** an edge that connects a node in one set of a cut to a node in the other set.

**Cut property:** given any cut of a graph, the minimum-weight crossing edge is in the MST. (With unique edge weights; and such an edge is in every MST if there are several.)

**Prim's algorithm:** starting from an arbitrary node, repeatedly add the minimum-weight edge with exactly one endpoint in the tree built so far, until V - 1 edges are added.

**Kruskal's algorithm:** consider all edges in increasing order of weight; add each edge to the MST unless doing so creates a cycle; stop at V - 1 edges.

**Scan (in Prim's):** to consider all of a vertex's outgoing edges, relaxing them against the current best-known distances to the tree, and marking the vertex as visited.

**Relaxation (Prim's version):** given edge `e` to vertex `w`, if `e.weight() < distTo[w]`, update `distTo[w] = e.weight()`, `edgeTo[w] = e`, and decrease `w`'s priority in the fringe.

**α(V):** the inverse Ackermann function, the (effectively constant) cost factor for weighted quick union with path compression, from the Disjoint Sets lecture.

**log\* V:** iterated logarithm, the number of times you must apply log before reaching 1; also an effectively constant factor, used in the slides' WQUPC bound.

**Steiner tree (extra context, mentioned in lecture):** a generalization of the MST in which new vertices (junction points) not present in the original vertex set may be introduced to reduce total weight.

---

## Worked Examples

### Example 1: Cycle detection via DFS (the warm-up, Approach 1)

```java
public class CycleDetector {
    private boolean[] marked;
    private boolean hasCycle;
    private Graph G;

    public CycleDetector(Graph G) {
        this.G = G;
        marked = new boolean[G.V()];
        hasCycle = false;
        dfs(0, -1);   // start at vertex 0; -1 means "no parent"
    }

    /* v is the vertex we're exploring; parent is where we came from. */
    private void dfs(int v, int parent) {
        marked[v] = true;
        for (int w : G.adj(v)) {
            if (w == parent) {
                continue;        // don't count the node we came from
            }
            if (marked[w]) {
                hasCycle = true; // reached an already-seen vertex: cycle
                return;
            }
            dfs(w, v);
        }
    }

    public boolean hasCycle() {
        return hasCycle;
    }
}
```

**What it does and why.** `marked` records which vertices the search has already reached. The recursion dives as deep as possible, backtracks, and dives again. In a tree, this always bottoms out at leaves and never revisits anything, so `hasCycle` stays false. If a cycle exists, some edge eventually leads to a vertex already on the marked set, and we detect it.

**The parent check is the whole subtlety.** In an undirected graph, every edge appears in both adjacency lists. When DFS moves from A to B, B's adjacency list contains A, and A is marked. Without `if (w == parent) continue;`, this two-vertex back-glance would be reported as a cycle in every graph with at least one edge. The lecture: "we can just make sure that the parent object, we don't think of as something we've already seen... and we ignore it."

**Environment reasoning (in words).** Each nested `dfs` call creates its own frame holding its own `v` and `parent`; `marked` and `hasCycle` live on the `CycleDetector` instance, shared by all frames. So the frames form a stack tracking the current root-to-current-vertex path, while the shared `marked` array accumulates knowledge across all branches, including branches that have already returned. That split (per-frame path, shared global seen-set) is what makes "already marked" mean "seen anywhere before," not "seen on this path."

**Runtime.** O(V + E) in general. Here, because an acyclic graph has at most V - 1 edges, we examine at most V edges before concluding, so O(V).

### Example 2: Cycle detection via disjoint sets (the warm-up, Approach 2)

```java
public static boolean hasCycle(EdgeWeightedGraph G) {
    WeightedQuickUnionPC uf = new WeightedQuickUnionPC(G.V());
    for (Edge e : G.edges()) {
        int v = e.from();
        int w = e.to();
        if (uf.connected(v, w)) {
            return true;      // already joined: this edge closes a cycle
        }
        uf.union(v, w);
    }
    return false;
}
```

**Step by step on the lecture's graph.**

| Edge considered | `connected(v, w)`? | Action | Sets afterward |
|---|---|---|---|
| A-C | no | union | {A, C} |
| E-F | no | union | {A, C}, {E, F} |
| F-G | no (G is alone) | union | {A, C}, {E, F, G} |
| E-G | **yes** (both in {E, F, G}) | return true | cycle found |

**Why `connected` is not "is there an edge".** The lecture leaned on this repeatedly. When we ask `connected(A, C)`, there obviously *is* an edge A-C, that is why we are examining it. The question is whether A and C were already joined *through other, previously processed edges*. If yes, this edge adds a second route between them, hence a cycle.

**Runtime.** O(V + E·α(V)) with path compression, simplifying to O(V·α(V)) by the same at-most-V-edges argument. Slightly slower than DFS for a one-shot check, but it persists component information across many queries, which is exactly what Kruskal's needs.

### Example 3: Prim's algorithm (lecture pseudocode)

```java
public class PrimMST {
    private Edge[] edgeTo;
    private double[] distTo;
    private boolean[] marked;
    private SpecialPQ<Double> fringe;

    public PrimMST(EdgeWeightedGraph G) {
        edgeTo = new Edge[G.V()];
        distTo = new double[G.V()];
        marked = new boolean[G.V()];
        fringe = new SpecialPQ<Double>(G.V());

        distTo[s] = 0.0;
        setDistancesToInfinityExceptS(s);
        insertAllVertices(fringe);

        /* Get vertices in order of distance from tree. */
        while (!fringe.isEmpty()) {
            int v = fringe.delMin();
            scan(G, v);
        }
    }

    private void scan(EdgeWeightedGraph G, int v) {
        marked[v] = true;                       // v is closest, add to MST
        for (Edge e : G.adj(v)) {
            int w = e.other(v);
            if (marked[w]) { continue; }        // already in MST, skip
            if (e.weight() < distTo[w]) {       // better edge to tree found
                distTo[w] = e.weight();
                edgeTo[w] = e;
                fringe.decreasePriority(w, distTo[w]);
            }
        }
    }
}
```

**Line-by-line reasoning.**

- `distTo[w]` means **"the weight of the cheapest known edge connecting w to the current tree,"** not "distance from the source." That single reinterpretation is the entire difference from Dijkstra's.
- `edgeTo[w]` remembers *which* edge achieves that, so that when `w` is finally popped, we know which edge to commit to the MST. At the end, `edgeTo[]` holds the MST's edges.
- `distTo[s] = 0.0` and everything else infinity, then all V vertices go into the fringe. The source has top priority, so it pops first.
- The fringe is a `SpecialPQ`, the same kind as Dijkstra's needs: one supporting `decreasePriority`, since priorities drop as better edges are discovered.
- `while (!fringe.isEmpty())`: the important invariant is that the fringe is ordered by *current best known distance from the tree*. Popping the min gives the unvisited vertex closest to the tree, which is precisely the generic algorithm's minimum crossing edge for the cut (tree) vs. (not tree).
- Inside `scan`: `marked[v] = true` commits v (and `edgeTo[v]`) to the MST. **Nothing is committed until delMin.** Until then, an entry in the fringe is only a tentative best candidate.
- `if (marked[w]) continue;` skips edges leading back into the tree. These are the edges that would create cycles.
- `if (e.weight() < distTo[w])`: relaxation. Note the right-hand side is just `e.weight()`, **not** `distTo[v] + e.weight()`. That is the Prim's/Dijkstra's fork in the road.

**Where the savings come from.** In naive Prim's, every round re-enumerates all edges crossing the frontier. Here, each vertex stores one number (its best distance to the tree), and each edge is examined only when one of its endpoints is popped. The lecture's step 3 illustrated this: when C was popped, edge B-C (weight 3) was examined and immediately discarded because `distTo[B]` was already 2; B stayed in the queue at priority 2, and B-C never needs to be looked at again.

**The distance-semantics checkpoint.** When C (at `distTo` 1 from A) is scanned and we look at edge C-F of weight 15:
- Dijkstra's would set `distTo[F] = distTo[C] + 15 = 16`.
- Prim's sets `distTo[F] = 15`, since F is 15 away from the *tree*.

Later, scanning E finds edge E-F of weight 4. Since 4 < 15, F is updated to 4 with `edgeTo[F] = E-F`.

**Runtime.** V inserts, V delMins, O(E) decreasePriority calls, each O(log V): O(V log V + V log V + E log V) = **O(E log V)** when E > V.

### Example 4: Kruskal's algorithm (lecture pseudocode)

```java
public class KruskalMST {
    private List<Edge> mst = new ArrayList<Edge>();

    public KruskalMST(EdgeWeightedGraph G) {
        MinPQ<Edge> pq = new MinPQ<Edge>();
        for (Edge e : G.edges()) {
            pq.insert(e);
        }
        WeightedQuickUnionPC uf = new WeightedQuickUnionPC(G.V());
        while (!pq.isEmpty() && mst.size() < G.V() - 1) {
            Edge e = pq.delMin();
            int v = e.from();
            int w = e.to();
            if (!uf.connected(v, w)) {
                uf.union(v, w);
                mst.add(e);
            }
        }
    }
}
```

**Line-by-line reasoning.**

- Insert all E edges into a `MinPQ`. Repeated `delMin` then hands them back in increasing weight order; the PQ is being used purely as a sorter. (Equivalently, quicksort the edge array up front, as the lecture mentioned when asked.)
- `WeightedQuickUnionPC uf = new WeightedQuickUnionPC(G.V())`: every vertex starts as its own singleton component. The disjoint-set structure tracks which vertices are joined *so far by accepted MST edges*, which is exactly the cycle test from the warm-up.
- The loop condition has two parts. `!pq.isEmpty()` guards against running out of edges; `mst.size() < G.V() - 1` stops as soon as we have V - 1 edges, since a spanning tree on V vertices has exactly V - 1 edges and there is nothing left to do.
- `if (!uf.connected(v, w))`: if v and w are in different components, this edge joins two components and cannot create a cycle, so accept it: `union` them and add the edge. If they are already in the same component, adding the edge would close a cycle, so skip it entirely (note there is no `else`: the edge is simply dropped).

**Box-and-pointer-style reasoning in words.** It is tempting to picture the disjoint-set structure as holding edges. It does not. After accepting A-C and C-E, `uf` does not know "A-C and C-E are edges"; it knows "A, C, E are one set." The lecture made this correction explicitly: "realize that these are not edges... this is the combined set of elements A, C, and E." Internally, WQU stores a parent array, so each set is a tree of parent pointers with one root; `connected(v, w)` walks both to their roots and compares, and path compression flattens those walks so future queries are nearly constant. The MST edges themselves live only in the separate `mst` list.

**The cycle rejection, concretely.** When B-E comes up, the accepted edges so far have already built the component {A, B, C, E} (via A-C, C-E, A-B). So `uf.connected(B, E)` is true, and B-E is discarded: B and E are already reachable from each other, so an edge between them would be a second route, hence a cycle.

**Runtime.** E inserts and O(E) delMins at O(log E) each dominate: **O(E log E)**. With pre-sorted edges the PQ work becomes O(E) and the total drops to O(E + V log* V + E log* V) = **O(E log\* V)**. And since E < V², O(E log E) = O(E log V), so Kruskal's is not asymptotically worse than Prim's.

---

## Common Pitfalls

**1. Forgetting the parent check in DFS cycle detection.** In an undirected graph, the parent is always already marked. Without skipping it, you report a cycle on every edge.

**2. Reading `connected(v, w)` as "is there an edge between v and w."** The lecture corrected this several times. In the disjoint-set cycle test, `connected` means "have these already been unioned by previously accepted edges." Of course there is an edge, that is the edge you are testing.

**3. Thinking a cut must be a contiguous or left/right split.** A cut is just a 2-coloring of the vertices with both colors non-empty. The lecture's clicker on this had many students unsure, and the answer was that the scattered-looking coloring was a perfectly valid cut.

**4. Assuming the globally lightest edge crosses every cut.** In the 0-7 example, `0-7` (0.16) is the lightest edge in the whole graph but does not cross the cut {2,3,5,6}, because both endpoints are on the same side. You must scan down the sorted list for the first edge that actually crosses.

**5. Using `distTo[v] + e.weight()` in Prim's.** That is Dijkstra's relaxation. Prim's compares `e.weight()` alone against `distTo[w]`, because "distance" in Prim's means distance to the tree, not to a source.

**6. Expecting an MST to be a shortest paths tree.** (Extra material.) There is no source for an MST; SPTs depend on the source. The MST may coincide with the SPT from some particular vertex, but it need not, and in the ladder graph *no* vertex's SPT equals the MST.

**7. Committing to a vertex in Prim's when you insert/update it rather than when you pop it.** Fringe entries are tentative. A vertex's `distTo` and `edgeTo` can improve several times before it is popped; only `delMin` (which triggers `marked[v] = true`) commits it.

**8. Forgetting the stopping condition.** Both algorithms finish at V - 1 edges, because that is exactly how many edges a tree on V vertices has. Kruskal's needs this in the loop condition so it does not keep churning through heavy edges after the tree is complete.

**9. Rerunning DFS inside Kruskal's for every cycle check.** Technically correct but wasteful; the whole reason the course covers weighted quick union is that it remembers component structure across the many queries Kruskal's makes.

**10. Claiming Kruskal's is asymptotically worse than Prim's.** It is not. Because E < V², O(E log E) = O(E log V). It may be slower in practice on unsorted edges, but not asymptotically.

**11. Dropping the unique-weights assumption without noticing.** With ties, there can be several MSTs, and a given minimum crossing edge is in *some* MST rather than provably in a unique one. The lecture's own Prim's trace hit a genuine tie at weight 2 and noted that either choice works.

---

## Likely Exam Points

### 1. Identify valid spanning trees / MSTs

Commonly tested as "which of these subgraphs is a spanning tree?" with distractors that fail connectedness, acyclicity, or coverage.

**Practice:** A connected undirected graph has 9 vertices. A candidate subgraph uses 9 edges, touches all 9 vertices, and is connected. Can it be a spanning tree?

**Answer:** No. A spanning tree on 9 vertices has exactly 9 - 1 = 8 edges. With 9 edges and 9 vertices it must contain a cycle, so it is not a tree.

### 2. Apply the cut property to a sorted edge list

**Practice:** Using the lecture's edge list, which edge is the minimum-weight edge crossing the cut {1, 3} vs. {0, 2, 4, 5, 6, 7}, and what does that tell you?

**Answer:** Scan the sorted list for the first crossing edge. `0-7` (both outside), `2-3` crosses (2 outside, 3 inside) at weight **0.17**. Earlier edge `0-7` (0.16) does not cross. So `2-3` is the minimum crossing edge and must therefore be in the MST, by the cut property.

### 3. Run Prim's by hand and give the visit order

**Practice:** On the lecture graph (A-B 2, A-C 1, B-C 3, B-E 5, B-D 11, C-E 1, C-F 15, D-E 2, D-G 1, E-F 4, E-G 3, F-G 1), run Prim's from A. List the MST edges in the order added and the total weight.

**Answer:** A-C (1), C-E (1), then a tie at 2 between A-B and D-E (either order is valid), then D-G (1), then F-G (1). One valid sequence: A-C, C-E, A-B, D-E, D-G, F-G. That is 6 edges for 7 vertices, total weight 1 + 1 + 2 + 2 + 1 + 1 = **8**.

### 4. Run Kruskal's by hand, including identifying the rejected edges

**Practice:** On the same graph, run Kruskal's. Which edge does the lecture point to as the first one rejected for creating a cycle, and why?

**Answer:** The weight-1 edges go in first (A-C, C-E, D-G, F-G), then A-B at weight 2. When B-E (weight 5) is reached, the accepted edges have already placed A, B, C, E in one component, so `connected(B, E)` is true and B-E is skipped: adding it would close the cycle through A-B, A-C, C-E.

### 5. Prim's vs. Dijkstra's: state the one difference

A classic short-answer question.

**Practice:** In one sentence, how does Prim's relaxation differ from Dijkstra's? Then: scanning vertex C (where `distTo[C] = 1`) across edge C-F of weight 15, what value does each algorithm store for F?

**Answer:** Dijkstra's judges an edge by the resulting distance to the **source**, so it stores `distTo[C] + 15 = 16`; Prim's judges an edge by the distance to the **tree**, so it stores just `15`. Equivalently, Dijkstra's relaxes with `distTo[v] + e.weight()` and Prim's with `e.weight()`.

### 6. Runtime derivation with an operation-count table

**Practice:** Derive Prim's runtime assuming a binary-heap PQ with O(log V) operations, and explain which term dominates.

**Answer:** V inserts at O(log V) gives O(V log V); V delMins at O(log V) gives O(V log V); up to O(E) `decreasePriority` calls at O(log V) gives O(E log V). Total O(V log V + V log V + E log V). For a connected graph E >= V - 1, and assuming E > V, the **E log V** term dominates: **O(E log V)**, the same as Dijkstra's.

### 7. Kruskal's runtime and the pre-sorted variant

**Practice:** Kruskal's with a MinPQ is O(E log E). Why does it drop to O(E log* V) with pre-sorted edges, and why is O(E log E) not asymptotically worse than O(E log V)?

**Answer:** With pre-sorted edges you skip the PQ entirely and just iterate the list in O(E), leaving O(E) + O(V log* V) for the unions + O(E log* V) for the connected-checks = O(E log* V). And since E < V², log E < log V² = 2 log V, so O(E log E) = O(E log V): the same asymptotic class as Prim's, even if slower in practice.

### 8. Why does Prim's / Kruskal's work?

Proof-sketch questions asking you to connect each algorithm to the generic cut-property algorithm.

**Practice:** State the cut used by Kruskal's when it adds edge `e = v-w`, and verify the two conditions the generic algorithm requires.

**Answer:** The cut is (all vertices currently connected to v) vs. (everything else). (i) No crossing edge is already in the MST, since v's side is by definition everything already connected to v. (ii) No crossing edge is lighter than `e`, since edges are processed in increasing weight order, so every lighter edge has already been considered. Hence `e` is the minimum crossing edge of a cut with no crossing edges in the tree, and the cut property guarantees it is in the MST.

### 9. Prove the cut property

**Practice:** Prove that for any cut, the minimum-weight crossing edge `e` is in the MST (assume unique weights).

**Answer:** By contradiction. Let X be an MST and suppose `e` is not in X. Adding `e` to X creates a cycle; because `e`'s endpoints lie on opposite sides of the cut, that cycle must cross the cut again at some edge `f` in X. Since `e` is the minimum crossing edge, weight(`e`) < weight(`f`). Removing `f` and adding `e` yields a spanning tree (still connected, still V - 1 edges) of strictly lower weight, contradicting the minimality of X.

### 10. MST vs. SPT (extra material, from the web videos)

**Practice:** For the graph A-B 2, B-C 2, C-D 2, A-D 3, the MST is {A-B, B-C, C-D}. Is it a shortest paths tree, and if so from which source?

**Answer:** Yes, from **B**. It is not the SPT from D (the SPT from D would use the direct D-A edge of weight 3 instead of the weight-6 path D-C-B-A), nor from A or C. In general, an SPT depends on its source while an MST has no source, so they need not coincide, though sometimes they do.

### 11. Cycle detection runtimes

**Practice:** Why is DFS cycle detection on a connected undirected graph O(V) rather than O(V + E)?

**Answer:** An acyclic graph has at most V - 1 edges, so after examining V edges we are guaranteed to have found a cycle. The number of edges we actually check is bounded by V, so O(V + E) collapses to O(V).

---

## Summary

- A **spanning tree** is a connected, acyclic subgraph containing all vertices; an **MST** is one of minimum total edge weight. On V vertices it has exactly **V - 1** edges.
- **Cut:** any assignment of vertices to two non-empty sets (no contiguity required). **Crossing edge:** one with endpoints in different sets.
- **Cut property:** for any cut, the minimum-weight crossing edge is in the MST (and in every MST if several exist). Proof: adding that edge to a supposed MST that lacks it creates a cycle that re-crosses the cut at a heavier edge; swap them for a cheaper spanning tree, a contradiction.
- **Generic MST algorithm:** repeatedly find a cut with no crossing edges in the tree, add its minimum crossing edge, until V - 1 edges. Correct but impractical: there are ~2^V cuts.
- **Prim's:** start anywhere, repeatedly add the lightest edge with exactly one endpoint in the tree. The implicit cut is (in tree) vs. (not in tree).
- **Prim's = Dijkstra's with "distance from the tree" instead of "distance from the source."** Relaxation compares `e.weight()` to `distTo[w]`, not `distTo[v] + e.weight()`. Visit order is by distance from the growing tree.
- **Prim's runtime:** V inserts + V delMins + O(E) decreasePriority, all O(log V), giving **O(E log V)** when E > V.
- **Kruskal's:** process edges in increasing weight order, adding each unless it creates a cycle, until V - 1 edges. The implicit cut is (everything connected to v) vs. (the rest).
- Cycle detection in Kruskal's uses **weighted quick union with path compression**; `connected(v, w)` means "already unioned," not "there is an edge." WQU is used instead of DFS because it remembers component structure across many queries.
- **Kruskal's runtime:** **O(E log E)** with a MinPQ, **O(E log\* V)** with pre-sorted edges. Since E < V², O(E log E) = O(E log V), so it is not asymptotically worse than Prim's.
- **Visual character:** Prim's grows one tree outward from a seed; Kruskal's scatters many tiny components across the graph that gradually merge.
- Warm-up takeaway: cycles in an undirected graph can be found by DFS with marking (O(V) here, remembering to skip the parent) or by disjoint sets (O(V·α(V))).
- **Extra:** MST has no source; SPT does. The MST may coincide with the SPT from some vertex, but for some graphs no vertex works. State of the art for compare-based MST is Pettie-Ramachandran (2002), optimal but of unknown complexity.
