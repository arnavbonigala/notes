<!-- Fri, Oct 23, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 25: Directed Acyclic Graphs

## Overview

This lecture is the fifth and last of the graph sequence, and its stated goal is practice at A-level problem solving using only tools we already have. It introduces the **directed acyclic graph (DAG)**, a digraph with no directed cycles, and then develops three results about DAGs plus one big idea. First, **topological sort**: a DAG's vertices can be linearly ordered so that every edge points "rightwards", and you compute such an order by running DFS with restarts and reversing the postorder, in O(V+E) time. Second, **DAG shortest paths**: Dijkstra's algorithm breaks when edge weights can be negative (a vertex can be finalized before a later negative edge improves its distance), but on a DAG you can simply visit vertices in topological order and relax every outgoing edge, which gives correct shortest paths even with negative weights, again in O(V+E). Third, **DAG longest paths**: the general longest simple path problem has no known efficient algorithm (the lecture calls it perhaps the most important unsolved problem in mathematics, in the sense that a huge family of problems is interconvertible with it), but on a DAG you negate all weights, run DAG-SPT, and negate the resulting distances. That last trick is the lecture's entry point into **reductions**: using a subroutine for problem Q to solve problem P means "P reduces to Q", and reductions are just the formal version of the preprocess/solve/postprocess decomposition we have used all semester (percolation reduces to disjoint sets, hyponyms reduces to DFS). An optional CS 170 preview shows a richer, non-sign-flipping reduction: 3SAT reduces to Independent Set.

> Note on sources: the lecture slides label themselves "Spring 2026" and carry a warning that they have not yet been updated for Fall 2026. The linked textbook chapter ("24. Reductions and Decomposition") was a stub with no body text, and no transcript was available, so everything below comes from the slide deck. Slide diagrams did not survive PDF extraction; where I reconstruct an edge set I say so and show how the reconstruction is forced by the demo traces. Java code is written in the algs4 style used by the course files the slides name (`Topological.java`, `AcyclicSP.java`); the slides list the filenames but do not show the source, so the code bodies are my faithful reconstruction (extra context).

---

## Key Concepts

### 1. Where we are: the graph problems table

The lecture opens by recapping the running table of graph problems. Everything so far:

| Problem | Description | Solution | Efficiency |
|---|---|---|---|
| paths | Find a path from `s` to every reachable vertex | `DepthFirstPaths.java` | O(V+E) time, Θ(V) space |
| shortest paths (unweighted) | Fewest-edge path from `s` to every reachable vertex | `BreadthFirstPaths.java` | O(V+E) time, Θ(V) space |
| shortest weighted paths | Lowest-weight path from `s` to every reachable vertex | `DijkstrasSP.java` | O(E log V) time, Θ(V) space |
| shortest weighted path to one target (out of scope) | Lowest-weight path from `s` to a target | A\*: Dijkstra's plus heuristic `h(v, goal)` added to priority | depends on heuristic, Θ(V) space |
| MST | Minimum spanning tree | `LazyPrimMST.java` | left as `O(???)` on the slide |
| MST | Minimum spanning tree | `PrimMST.java` | O(E log V) time, Θ(V) space |
| MST | Minimum spanning tree | `KruskalMST.java` | O(E log E) time, Θ(E) space |

By the end of today the table gains four rows: topological sort, DAG shortest paths, longest paths (no known efficient solution), and DAG longest paths.

### 2. The task-ordering problem and why DFS postorder solves it

Model a set of tasks A through H as a digraph where an edge `v -> w` means "v must happen before w". A valid schedule is a linear order of all vertices in which every edge points forward. The lecture's question is: which algorithm finds one?

Answer: **DFS with restarts, then reverse the postorder.**

> Perform a DFS traversal from every vertex with indegree 0, **not** clearing markings in between traversals. Record the DFS postorder in a list. The topological ordering is the reverse of that list (reverse postorder).

The intuition for why reverse postorder works: a vertex is added to the postorder list only after every vertex reachable from it has already been added. So in the postorder list, every vertex appears *after* all of its descendants. Reverse the list and every vertex appears *before* all of its descendants, which is exactly the topological property. Two parts of the recipe matter:

- **Postorder, not preorder.** Preorder records a vertex when you first touch it, which says nothing about where its successors land. Postorder records it when it is finished.
- **Do not clear the markings between restarts.** The `marked` array must persist across all the restarts, because the whole point is to build one global postorder list. Clearing marks would revisit vertices and corrupt the order (and could blow up the runtime).

The name "topological sort" comes from the picture: if you lay the sorted vertices out left to right on a line, every arrow in the diagram points rightwards.

### 3. DFS "with restarts" versus "without"

The lecture explicitly flags a vocabulary hazard:

> Be aware, that when people say "Depth First Search", they sometimes mean with restarts, and they sometimes mean without. For example, when we did `DepthFirstPaths` for reachability, we did not restart. For topological sort, we restarted from every vertex with indegree 0.

So "DFS" is ambiguous in general, and on an exam you should read the problem for which variant is intended. Reachability from a source: no restarts. Topological sort: restarts, because a DAG need not have a single vertex that reaches everything.

### 4. You do not actually need the indegree-0 vertices

An "extra slide" asks: what is the runtime to find all vertices of indegree 0? The punchline is that you do not have to.

> Another better topological sort algorithm: run DFS from an arbitrary vertex. If not all marked, pick an unmarked vertex and do it again. Repeat until done.

Reverse postorder of a complete DFS forest is a valid topological order no matter which unmarked vertex you restart from. (Starting at a vertex with incoming edges is harmless: the vertices that should precede it simply finish later and therefore land earlier in the reversed list.)

(Extra context, since the slide leaves the question dangling: computing all indegrees is O(V+E) anyway, since you scan every adjacency list once and increment a counter per edge, so it would not change the asymptotic runtime. It is just unnecessary work.)

### 5. Topological order exists exactly when the graph is a DAG

> A topological sort only exists if the graph is a directed acyclic graph (DAG).

If the graph has a directed cycle `v1 -> v2 -> ... -> v1`, then any linear order would need `v1` before `v2` before ... before `v1`, which is impossible. The slide reuses the A-through-H picture with an added back edge and notes there is **no** possible ordering where all arrows are respected. The converse also holds: every finite DAG has at least one topological order, and the DFS algorithm above produces one. The lecture notes that DAGs show up in many real-world applications (build systems, course prerequisites, spreadsheet recalculation, and so on are the usual examples) and that there are many graph algorithms that only work on DAGs, which is exactly the point of the next two sections.

New table row:

| Problem | Description | Solution | Efficiency |
|---|---|---|---|
| topological sort | Find an ordering of vertices that respects edges of our DAG | `Topological.java` | O(V+E) time, Θ(V) space |

### 6. Dijkstra's algorithm fails with negative edges

The lecture warms up by running Dijkstra's on a small weighted DAG, then changes two weights to show the failure. The mechanism of failure is worth memorizing, because it is the standard exam question.

Dijkstra's correctness rests on this invariant: when a vertex is popped from the priority queue and visited, its `distTo` is final. That is justified only when weights are non-negative, because then any path that leaves the visited set can never come back cheaper. With a negative edge, a vertex you already finalized can later be improved, and the improvement arrives too late to propagate.

In the lecture's example (reconstructed edge set below), Dijkstra's is shown just before vertex C is visited. Relaxation along `C -> E` with weight -20 succeeds, so `distTo[E]` drops from 2 to -13. But F was already visited and finalized at 3, so **`distTo[F]` is never updated** and the final answer is wrong: the true shortest distance to F is -12, via `A -> B -> C -> E -> F`.

The slide's annotation is blunt: "distTo is wrong!"

### 7. The DAG-SPT algorithm: relax in topological order

The challenge posed is: find an algorithm for shortest paths on a DAG that works even with negative edges, still using `relax` as the basic building block.

> One simple idea: visit vertices in topological order. On each visit, relax all outgoing edges. Each vertex is visited only when all possible info about it has been used!

That last bullet is the correctness argument in one line. In a topological order, every edge into vertex `v` comes from a vertex that appears earlier. So by the time you visit `v`, every edge that could possibly improve `distTo[v]` has already been relaxed, and `distTo[v]` is final. Negative weights are irrelevant to this argument: it never appeals to non-negativity, only to the order of relaxations. There is no priority queue and no notion of "closest unvisited vertex", so there is nothing for a negative edge to break.

The two-step algorithm:

1. Find a topological order. O(V+E).
2. Visit vertices in that order, relaxing all outgoing edges of each. O(V+E).

Total: O(V+E) time, Θ(V) space. Note this is **asymptotically faster than Dijkstra's** O(E log V) and handles negative weights, at the cost of requiring acyclicity.

The slide anticipates a recurring student confusion:

> Occasional question asked by students: why isn't it O(V\*E)? We're relaxing all edges from each vertex. Keep in mind that E is the total number of edges in the entire graph, not the number of edges per vertex.

Across the whole run, each edge is relaxed exactly once, so the edge work sums to Θ(E), not Θ(V·E). (This is the same "sum of adjacency list lengths equals E" accounting as in DFS and BFS.)

New table row, with the note that **DAG shortest paths uses topological sort as a subroutine**, which is the first hint of the reduction/decomposition theme:

| Problem | Description | Solution | Efficiency |
|---|---|---|---|
| DAG shortest paths | Find a shortest paths tree on a DAG | `AcyclicSP.java` | O(V+E) time, Θ(V) space |

### 8. What if it is not a DAG?

> The DAG algorithm only works on graphs that contain no cycles. If a graph has negative edges, and includes some cycles, can't use DAG algorithm. Finding a fast algorithm on such graphs was an open problem for 40 years, solved in 2022 by Bernstein, Nanongkai and Wulff-Nilsen.

The slide links a Quanta article (`quantamagazine.org/finally-a-fast-algorithm-for-shortest-paths-on-negative-graphs-20230118/`). This is cultural context, not examinable material. (Extra context: the classical in-scope-elsewhere answer for negative weights with cycles is Bellman-Ford, O(V·E), which also detects negative cycles; 61B does not cover it in this lecture, and note that with a reachable negative cycle "shortest path" is not even well defined.)

### 9. Longest paths: hard in general, easy on a DAG

The **longest paths tree (LPT)** problem: find the longest path from `s` to every other vertex, where the path must be **simple** (no repeated vertices, hence no cycles). The simplicity requirement is what makes the problem hard: without it, any positive cycle lets you loop forever and the answer is unbounded.

The lecture's "surprising facts" about general longest paths:

- The best known algorithm is exponential (described as "extremely bad").
- It is "perhaps the most important unsolved problem in mathematics." The later reductions section explains what that claim really means (see below).

Two exercises the lecture suggests after class:

- Find an example where the obvious algorithm (Dijkstra's, but pick the biggest edge first) fails.
- Figure out whether the longest path to every other vertex always forms a tree, that is, whether an LPT exists for all graphs.

And a "highly recommended exercise": play with the problem until you can explain in English *why* it is hard, searching for "why longest paths hard" if you get stuck.

(Extra context on the greedy-fails exercise: a one-line counterexample is `s -> a` with weight 1, `s -> b` with weight 10, `b -> t` with weight 1, `a -> t` with weight 100. Greedy "always grab the biggest edge" takes `s -> b` and reports 11 to t, while the true longest simple path is `s -> a -> t` with 101. The deeper reason the problem is hard: shortest paths has the property that a prefix of an optimal path is itself optimal, so local decisions compose. For longest *simple* paths, the best route to an intermediate vertex may burn vertices you needed later, so subproblems are not independent and relaxation-style reasoning collapses.)

**On a DAG, though, longest paths is as easy as shortest paths**, and the lecture asks for an O(V+E) algorithm. The solution:

> DAG LPT solution for graph G:
> - Form a new copy of the graph G' with signs of all edge weights flipped.
> - Run DAGSPT on G' yielding result X.
> - Flip signs of all values in `X.distTo`. `X.edgeTo` is already correct.

Why acyclicity rescues the problem: in a DAG every path is automatically simple, so "longest path" and "longest walk" coincide and there is no cycle to exploit. And negating weights turns minimization into maximization because minimizing a sum of negated weights maximizes the original sum.

### 10. Mathematical maturity

The lecture pauses on a meta-point:

> If you have a very high degree of so-called "mathematical maturity", this algorithm should seem plainly correct. There's no real need to prove anything or show demos. We know DAG SPT works on graphs with negative edge weights. We also know that -(-a + -b + -c + -d) = a + b + c + d.

Those two facts compose into a complete correctness argument: the set of paths from `s` to `v` is identical in G and G', the weight of each path is exactly negated, so the minimum-weight path in G' is the maximum-weight path in G, and negating its total recovers the true longest distance. `edgeTo` needs no fixing because the chosen *path* is the same path; only its numeric label changed. The slide notes that building this kind of maturity is part of the Berkeley technical education and will be emphasized in EECS 16A/16B, CS 126/127, CS 70, CS 170, and Math 110.

### 11. Reductions

The lecture observes that the DAG-LPT solution *felt* different: given a graph G we built a different graph G', fed it to a related but different algorithm, and interpreted the result. The slide draws this as a box labeled DAG-LPT containing:

```
G --preprocess--> G' --DAG-SPT--> SPT of G' --postprocess--> LPT of G
```

The outer box is a complete algorithm for DAG-LPT, built out of a call to DAG-SPT plus cheap wrapper code.

> This process is known as reduction. Since DAG-SPT can be used to solve DAG-LPT, we say that "DAG-LPT reduces to DAG-SPT".

Get the direction right: **the problem you are solving reduces to the tool you are calling.** The informal definition from *Algorithms* by Dasgupta, Papadimitriou, and Vazirani:

> "If any subroutine for task Q can be used to solve P, we say P reduces to Q."

The lecture's real-world analogy, which usefully stresses that reductions are many-to-one and not unique:

- "Climbing a hill" reduces to "riding a ski lift."
- "Climbing a hill" reduces to "being shot out of a cannon."
- "Climbing a hill" reduces to "riding a bike up the hill."

The formal versions (Karp reductions, Cook reductions) are named as "way beyond the scope of our class" but offered as further reading. And the slides warn that reduction is more than sign flipping, which is why the bonus 3SAT example exists.

### 12. What "most important unsolved problem" really meant

> Earlier I claimed that the longest paths problem is potentially the most important unsolved question in mathematics. This is true, but also a bit misleading. The reason: vast numbers of important problems reduce to the longest paths problem! But also: the longest path problem also reduces back to those! In other words: solving any of these problems solves them all.

That mutual reducibility is the shape of NP-completeness, which the lecture says may be revisited in the last week of class. The important takeaway for 61B: a reduction transports difficulty and efficiency in a specific direction. If P reduces to Q and Q is easy, then P is easy. If P reduces to Q and P is known hard, then Q is hard.

### 13. Reductions and decomposition

> Reductions are just a more formal version of what we've been doing throughout the course. Percolation can be solved with Disjoint Sets. Finding the hyponyms of a word can be solved with DFS (or BFS). At its heart, computer science is about taking a complex task and breaking it into smaller parts. Using appropriate abstractions makes problem solving vastly easier.

This is the thesis of the lecture: today's three algorithms are all compositions. DAG-SPT calls topological sort. DAG-LPT calls DAG-SPT, which calls topological sort. Nothing new was invented, only recombined.

### 14. Bonus: Independent Set and 3SAT (optional CS 170 preview)

**Independent set**: a set of vertices in which no two vertices are adjacent. **The Independent-set problem**: does there exist an independent set of size k, that is, can you color k vertices red so that none touch? (The slide's example graph has N = 24 vertices with k = 9.)

**3SAT**: given a boolean formula, does there exist an assignment of truth values to the boolean variables that obeys a set of 3-variable disjunctive constraints? The slide's example:

```
(x1 || x2 || !x3) && (x1 || !x1 || x1) && (x2 || x3 || x4)
Solution: x1 = true, x2 = true, x3 = true, x4 = false
```

**Proposition: 3SAT reduces to Independent-set.** Given an instance phi of 3SAT, construct a graph G:

- For each clause in phi, create 3 vertices (one per literal) joined in a triangle.
- Add an edge between each literal and its negation. (Rationale on the slide: "can't both be true in 3SAT means can't be in same set in Independent-set".)

Then ask for an independent set of size k, where k is the number of clauses (the slide for the four-clause, four-variable example writes "k = number of variables = 4"; see the pitfall note below). The triangle edges force you to pick at most one literal per clause, so an independent set of size k picks exactly one literal from every clause. The negation edges guarantee the picked literals are mutually consistent. Set every picked literal true (and anything unpicked arbitrarily) and every clause has a true literal, so phi is satisfied.

The slide's formula is:

```
phi = (x1 or x2 or x3) and (!x1 or !x2 or x4) and (!x1 or x3 or !x4) and (x1 or x3 or x4)
```

Final diagram:

```
            3SAT
phi --preprocess--> G --IND-SET--> IND-SET for G --postprocess--> satisfying assignment
```

with the lecture's two emphasized notes: **3SAT is not a graph problem**, and **reductions don't always involve creating graphs.** The point is that the preprocess step can be a genuinely creative encoding, not just a sign flip.

---

## Definitions

- **Directed acyclic graph (DAG)**: a directed graph containing no directed cycles.
- **Topological order (topological sort)**: an ordering of all the vertices of a DAG such that for every edge `v -> w`, `v` appears before `w`. Equivalently, when the vertices are laid out left to right in this order, every arrow points rightwards. Such an ordering exists if and only if the graph is a DAG.
- **Postorder (DFS)**: the sequence in which vertices are *finished* during DFS, that is, each vertex is appended to the list after all of its DFS recursive calls return.
- **Reverse postorder**: the postorder list reversed. For a DFS with restarts over a DAG, this is a topological order.
- **DFS with restarts**: run DFS, and when it terminates with unmarked vertices remaining, start a new DFS from an unmarked vertex, **without** clearing the `marked` array. Repeat until all vertices are marked.
- **Indegree of v**: the number of edges pointing into `v`. A vertex of indegree 0 has no prerequisites and is a safe DFS starting point for topological sort (though not required).
- **Relax (edge `v -> w` with weight `wt`)**: if `distTo[v] + wt < distTo[w]`, set `distTo[w] = distTo[v] + wt` and `edgeTo[w] = v -> w`. The basic building block of all shortest paths algorithms in this course.
- **Shortest paths tree (SPT)**: the tree formed by the `edgeTo` edges, giving a shortest path from the source to every reachable vertex.
- **DAG-SPT**: the algorithm that computes an SPT on a DAG by visiting vertices in topological order and relaxing all outgoing edges of each. Works with negative edge weights. O(V+E) time, Θ(V) space. Implemented by `AcyclicSP.java`.
- **Longest paths tree (LPT)**: the analogous structure for longest **simple** paths (no repeated vertices) from the source to every other vertex.
- **Longest paths problem (general graphs)**: no known efficient solution; best known algorithm is exponential.
- **DAG-LPT**: negate all edge weights into a copy G', run DAG-SPT on G', negate all `distTo` values; `edgeTo` is already correct. O(V+E) time, Θ(V) space.
- **Reduction**: "If any subroutine for task Q can be used to solve P, we say P reduces to Q." (Dasgupta, Papadimitriou, Vazirani.) In practice: preprocess the input of P into an input for Q, call Q, then postprocess Q's answer into an answer for P.
- **Independent set**: a set of vertices no two of which are adjacent.
- **Independent-set problem**: does a given graph contain an independent set of size k?
- **3SAT**: given a boolean formula that is an AND of 3-literal disjunctive clauses, is there a truth assignment making it true?
- **3-variable disjunctive constraint**: a clause like `(x1 || x2 || !x3)`, an OR of three literals.

---

## Worked Examples

### Example 1: Topological sort of the A-through-H task graph (the lecture's own demo)

**Reconstructing the graph.** The diagram is lost, but the demo's call stacks and postorder snapshots pin the edge set down. The slides show these states in order:

```
Postorder: []               Call stack: A
Postorder: []               Call stack: A->B
Postorder: []               Call stack: A->B->E
Postorder: [H]              Call stack: A->B->E->H
Postorder: [H, E]           Call stack: A->B->E
Postorder: [H, E, B]        Call stack: A->B
Postorder: [H, E, B]        Call stack: A
Postorder: [H, E, B, D]     Call stack: A->D
Postorder: [H, E, B, D, A]  Call stack: A
Postorder: [H, E, B, D, A]  Call stack: C
Postorder: [H, E, B, D, A]  Call stack: C->F
Postorder: [H,E,B,D,A,G]    Call stack: C->F->G
Postorder: [H,E,B,D,A,G,F]  Call stack: C->F
Postorder: [H,E,B,D,A,G,F,C] Call stack: C
```

An edge set consistent with every one of those frames, with the stated indegree-0 starts (A and C) and the stated valid orderings `[A, C, B, D, F, E, H, G]` and `[C, A, D, F, B, E, G, H]`:

```
A -> B,  A -> D,  B -> E,  C -> D,  C -> F,  D -> E,  E -> H,  F -> G
```

Sanity checks that force it: `D` is finished immediately when reached from A even though `E` lies below it, so `D -> E` must exist but `E` must already be marked. `G` is reached only from `F` under `C`, so there is no `E -> G`. (Reconstruction, flagged.)

**Step-by-step trace.**

1. Start DFS at A (indegree 0). Mark A. Call stack: `A`.
2. Visit A's first unmarked neighbor B. Mark B. Stack: `A -> B`.
3. From B go to E. Mark E. Stack: `A -> B -> E`.
4. From E go to H. Mark H. Stack: `A -> B -> E -> H`.
5. H has no outgoing edges, so H finishes. **Postorder: [H].** Return to E.
6. E's only successor H is marked, so E finishes. **Postorder: [H, E].** Return to B.
7. B is done. **Postorder: [H, E, B].** Return to A.
8. A's next neighbor is D, unmarked. Mark D. Stack: `A -> D`. D's only successor E is already marked, so D finishes immediately. **Postorder: [H, E, B, D].**
9. A has no more neighbors. A finishes. **Postorder: [H, E, B, D, A].** The first DFS call returns.
10. **Restart** at C, the other indegree-0 vertex. Critically, we do *not* clear markings, so A, B, D, E, H stay marked. C's neighbor D is already marked and is skipped.
11. From C go to F. Mark F. Stack: `C -> F`. From F go to G. Mark G. Stack: `C -> F -> G`.
12. G finishes. **Postorder: [H, E, B, D, A, G].** Then F finishes: **[H, E, B, D, A, G, F]**. Then C finishes: **[H, E, B, D, A, G, F, C]**.

**Reverse the postorder:** `[C, F, G, A, D, B, E, H]`. Every edge points rightwards in this list: C before F and D, F before G, A before B and D, D before E, B before E, E before H. Done.

**Environment / memory reasoning in words.** There is one `boolean[] marked` array and one shared result list living in the `Topological`/`DepthFirstOrder` object on the heap; every recursive `dfs` frame on the call stack holds only its own `v` and its own position in `v`'s adjacency iterator. The "call stack" line on each slide is literally the chain of live `dfs` frames. The reason step 10 works is that `marked` is reachable from the object, not from any one frame, so it survives the return of the outermost `dfs(A)` call and is still fully populated when `dfs(C)` begins. If instead you allocated a fresh `marked` array per restart, `dfs(C)` would re-walk D, E, H and append them again, producing garbage.

**The code** (reconstruction of `Topological.java` in algs4 style, extra context):

```java
public class Topological {
    private boolean[] marked;
    private Stack<Integer> reversePostorder; // or an ArrayList we reverse at the end

    public Topological(Digraph G) {
        marked = new boolean[G.V()];
        reversePostorder = new Stack<>();
        // DFS with restarts. The lecture restarts from indegree-0 vertices,
        // but an arbitrary unmarked vertex works just as well.
        for (int v = 0; v < G.V(); v += 1) {
            if (!marked[v]) {
                dfs(G, v);
            }
        }
    }

    private void dfs(Digraph G, int v) {
        marked[v] = true;                 // mark on the way down
        for (int w : G.adj(v)) {
            if (!marked[w]) {
                dfs(G, w);
            }
        }
        reversePostorder.push(v);         // POSTorder: after all children finish
    }

    /** Vertices in a valid topological order. */
    public Iterable<Integer> order() {
        return reversePostorder;          // popping a Stack yields reverse insertion order
    }
}
```

Why a `Stack` and not a list: pushing in postorder and then iterating the stack pops in LIFO order, which is exactly reverse postorder, so the reversal is free. Runtime: each vertex is marked once and each adjacency list is scanned once, so O(V+E). Space: `marked` is Θ(V), the stack is Θ(V), and the recursion depth is O(V) in the worst case (a long chain), so Θ(V) overall.

(Extra context: because the recursion depth can be Θ(V), a DFS-based topological sort can overflow the Java call stack on very large graphs, which is exactly the lesson from earlier in the course about recursive DFS. The standard alternative is Kahn's algorithm: repeatedly remove an indegree-0 vertex and decrement its neighbors' indegrees, maintaining a queue of indegree-0 vertices. It is also O(V+E) and iterative, and it additionally detects cycles, since it ends early with vertices remaining if and only if the graph has a cycle. Kahn's algorithm was not presented in this lecture.)

### Example 2: The "Test Your Understanding" topological sort

The graph (from *Algorithms* by Vazirani and Papadimitriou) has source `s = A` and vertices A, B, C, D, E, F. Reconstructing the edge set from the weights in the slide plus the linearized chain the slides draw later:

```
A -> B (1)   A -> D (2)   D -> B (4)   D -> E (3)
B -> C (6)   C -> E (1)   C -> F (2)   E -> F (1)
```

That is 8 edges, which matches the later slide's statement "for the graph below, E = 8".

**Question:** give a topological ordering.

**Answer from the slide:** `A, D, B, C, E, F`, found via DFS postorder `F, E, C, B, D, A` (reverse that and you get A, D, B, C, E, F).

**Verification edge by edge:** A(0) before B(2) and D(1); D(1) before B(2) and E(4); B(2) before C(3); C(3) before E(4) and F(5); E(4) before F(5). All forward. The slide also draws the vertices on a line in this order, with the "task ordering" reading: `s=A -> D -> B -> C -> E -> F`.

Note that the orderings used by the warmups are not unique in general; this graph happens to have a unique one (every consecutive pair is joined by an edge, which chains the whole order).

### Example 3: Dijkstra's failing on negative edges (the lecture's counterexample)

Same vertex set, with the lecture's modified weights:

```
A -> B (1)   A -> D (1)   D -> B (25)   D -> E (1)
B -> C (6)   C -> E (-20)  C -> F (1)   E -> F (1)
```

Trace Dijkstra's from A, following the slide's annotated `distTo` labels (B=1, C=7, D=1, E=2, F=3 at the moment just before C is visited):

1. Visit A (`distTo[A] = 0`). Relax `A -> B`: `distTo[B] = 1`. Relax `A -> D`: `distTo[D] = 1`.
2. Visit B or D (both at 1). Say B. Relax `B -> C`: `distTo[C] = 7`.
3. Visit D. Relax `D -> B`: 1 + 25 = 26, no improvement. Relax `D -> E`: `distTo[E] = 2`.
4. Visit E (priority 2, the smallest remaining). Relax `E -> F`: `distTo[F] = 3`.
5. Visit F (priority 3). **F is now finalized at 3.** F has no outgoing edges.
6. Visit C (priority 7). Relax `C -> E`: 7 + (-20) = -13 < 2, so `distTo[E] = -13`, `edgeTo[E] = C -> E`. The slide says: "Relaxation on E succeeds, but distance to F will never be updated."
7. Done. `distTo[F]` is still 3.

The correct answer is `distTo[F] = -12` via `A -> B -> C -> E -> F` (1 + 6 - 20 + 1). Slide verdict: **"distTo is wrong!"** Worse, the final `distTo` is internally inconsistent: `distTo[E] = -13` with the edge `E -> F` of weight 1 present, yet `distTo[F] = 3`, which violates the basic invariant `distTo[w] <= distTo[v] + weight(v,w)`. The structural cause: Dijkstra's visits E before C because E looked cheap, and it never revisits a finalized vertex, so the late improvement to E cannot propagate to F.

### Example 4: DAG-SPT on the same graph (the lecture's full demo)

**Step 1: topological order.** `A, D, B, C, E, F`, found in O(V+E). The slides redraw the graph as a line in this order:

```
s=A --1--> D --25--> B --6--> C --(-20)--> E --1--> F
    \__1__> B (the A->B edge, drawn below)      ^
                 D --1--> E (drawn below) ______|
```

**Step 2: visit in that order, relaxing all outgoing edges.** The slides show the "fringe" as the fixed list `[A, D, B, C, E, F]`: there is no priority queue, just a predetermined visiting order.

Initial state:

| v | distTo | edgeTo |
|---|---|---|
| A | 0 | - |
| B | ∞ | - |
| C | ∞ | - |
| D | ∞ | - |
| E | ∞ | - |
| F | ∞ | - |

1. **Visit A** (distTo 0). Relax `A -> B`: 0 + 1 = 1 < ∞, so `distTo[B] = 1`, `edgeTo[B] = A`. Relax `A -> D`: 0 + 1 = 1 < ∞, so `distTo[D] = 1`, `edgeTo[D] = A`.
2. **Visit D** (distTo 1). Relax `D -> B`: 1 + 25 = 26, not less than 1, so **no change** (this is the step where B's earlier, better value survives). Relax `D -> E`: 1 + 1 = 2 < ∞, so `distTo[E] = 2`, `edgeTo[E] = D`.
3. **Visit B** (distTo 1). Relax `B -> C`: 1 + 6 = 7 < ∞, so `distTo[C] = 7`, `edgeTo[C] = B`.
4. **Visit C** (distTo 7). Relax `C -> E`: 7 + (-20) = -13 < 2, so `distTo[E] = -13`, `edgeTo[E] = C`. This is exactly the relaxation Dijkstra's performed too late. Here it happens **before** E and F are ever visited.
5. **Visit E** (distTo -13). Relax `E -> F`: -13 + 1 = -12 < ∞, so `distTo[F] = -12`, `edgeTo[F] = E`.
6. **Visit F** (distTo -12). No outgoing edges. Done.

Final state, matching the last slide of the demo:

| v | distTo | edgeTo |
|---|---|---|
| A | 0 | - |
| B | 1 | A |
| C | 7 | B |
| D | 1 | A |
| E | -13 | C |
| F | -12 | E |

The SPT is `A -> B`, `A -> D`, `B -> C`, `C -> E`, `E -> F`, and `distTo[F] = -12` is the correct answer Dijkstra's missed.

**Why the order mattered at step 4.** Because C precedes E in topological order, and because *every* edge into E (`D -> E` and `C -> E`) comes from a vertex earlier in the order, `distTo[E]` has absorbed all possible improvements by the time we visit E. The same argument applies at every vertex, by induction along the order. That is the whole proof.

**The code** (reconstruction of `AcyclicSP.java`, extra context):

```java
public class AcyclicSP {
    private double[] distTo;
    private DirectedEdge[] edgeTo;

    public AcyclicSP(EdgeWeightedDigraph G, int s) {
        distTo = new double[G.V()];
        edgeTo = new DirectedEdge[G.V()];
        for (int v = 0; v < G.V(); v += 1) {
            distTo[v] = Double.POSITIVE_INFINITY;
        }
        distTo[s] = 0.0;

        // Step 1: reduce to topological sort (O(V + E)).
        Topological topological = new Topological(G);

        // Step 2: relax every edge once, in topological order (O(V + E)).
        for (int v : topological.order()) {
            for (DirectedEdge e : G.adj(v)) {
                relax(e);
            }
        }
    }

    private void relax(DirectedEdge e) {
        int v = e.from(), w = e.to();
        if (distTo[w] > distTo[v] + e.weight()) {
            distTo[w] = distTo[v] + e.weight();
            edgeTo[w] = e;
        }
    }

    public double distTo(int v) { return distTo[v]; }
    public boolean hasPathTo(int v) { return distTo[v] < Double.POSITIVE_INFINITY; }
}
```

Two things to notice. There is **no `PriorityQueue` and no `marked` array** in the main loop: the topological order replaces both, which is why the runtime drops the `log V` factor. And relaxing a vertex whose `distTo` is still infinity is harmless: `infinity + w` stays infinity and never wins a comparison, so unreachable vertices simply never improve anything. (In exact floating point, `Double.POSITIVE_INFINITY + w` is infinity for finite `w`, so the guard `distTo[w] > distTo[v] + e.weight()` is false.)

**Memory reasoning in words.** `distTo` and `edgeTo` are two parallel arrays of length V on the heap, owned by the `AcyclicSP` object. `edgeTo[w]` is a *reference* to a `DirectedEdge` object stored in G's adjacency lists, not a copy, so the SPT is represented with V references into the existing edge objects rather than a new graph. Reconstructing a path to `v` means following `edgeTo[v].from()` back to `s`, which is why `edgeTo` alone encodes the whole tree in Θ(V) space.

### Example 5: DAG longest paths by sign flipping (the lecture's own example)

Take the Example 2 graph with its original positive weights:

```
A -> B (1)   A -> D (2)   D -> B (4)   D -> E (3)
B -> C (6)   C -> E (1)   C -> F (2)   E -> F (1)
```

**Step 1: build G' with all signs flipped.** The slide shows exactly this picture with weights -1, -2, -4, -3, -6, -1, -2, -1.

**Step 2: run DAG-SPT on G' from A.** Topological order is unchanged by reweighting (it depends only on edge directions), so it is still `A, D, B, C, E, F`.

1. Visit A: `distTo[B] = -1` (via `A -> B`), `distTo[D] = -2`.
2. Visit D (-2): `D -> B` gives -2 + (-4) = -6 < -1, so `distTo[B] = -6`, `edgeTo[B] = D -> B`. `D -> E` gives -5, so `distTo[E] = -5`, `edgeTo[E] = D -> E`.
3. Visit B (-6): `B -> C` gives -6 + (-6) = -12, so `distTo[C] = -12`, `edgeTo[C] = B -> C`.
4. Visit C (-12): `C -> E` gives -13 < -5, so `distTo[E] = -13`, `edgeTo[E] = C -> E`. `C -> F` gives -14, so `distTo[F] = -14`.
5. Visit E (-13): `E -> F` gives -13 + (-1) = -14, which is **not** less than -14, so no change (tie; either predecessor is a valid longest path).
6. Visit F (-14). Done.

**Step 3: flip the signs of `distTo`.** Result: A = 0, B = 6, C = 12, D = 2, E = 13, F = 14. These are exactly the longest-path labels the slide prints on the DAG-LPT graph (`A 0, B 6, C 12, D 2, E 13, F 14`). Spot check: the longest simple path to F is `A -> D -> B -> C -> F` with 2 + 4 + 6 + 2 = 14, and `A -> D -> B -> C -> E -> F` also gives 2 + 4 + 6 + 1 + 1 = 14. Both are 14, which is the tie we saw at step 5.

`edgeTo` needs no post-processing, as the slide says, because the tree of chosen edges is the same tree; only the numbers attached to the paths were negated.

**The code** (the reduction written out, extra context):

```java
public class AcyclicLP {
    private double[] distTo;
    private DirectedEdge[] edgeTo;

    public AcyclicLP(EdgeWeightedDigraph G, int s) {
        // PREPROCESS: a NEW graph with every weight negated.
        // This must be a copy; do not mutate G, and do not alias its edge objects.
        EdgeWeightedDigraph negated = new EdgeWeightedDigraph(G.V());
        for (DirectedEdge e : G.edges()) {
            negated.addEdge(new DirectedEdge(e.from(), e.to(), -e.weight()));
        }

        // SOLVE: call the existing subroutine.
        AcyclicSP sp = new AcyclicSP(negated, s);

        // POSTPROCESS: flip the distances back.
        distTo = new double[G.V()];
        for (int v = 0; v < G.V(); v += 1) {
            distTo[v] = -sp.distTo(v);   // -(-a + -b + -c) == a + b + c
        }
        // The tree structure (which edge enters each vertex) is already correct.
    }

    public double distTo(int v) { return distTo[v]; }
}
```

**Box-and-pointer caution (extra context, a real implementation gotcha the slide glosses over).** The slide says "`X.edgeTo` is already correct," which is true about *which edge* enters each vertex. But if you implement this by handing `AcyclicSP` a copied graph of freshly constructed negated `DirectedEdge` objects, then `X.edgeTo[v]` points at an object in the *negated* graph, so calling `.weight()` on it returns the negated weight. The `from()`/`to()` endpoints, and therefore the path, are correct. If you need the original edge objects back, keep a parallel mapping, or note that `-e.weight()` recovers the original. Also note that negating is Θ(V+E) work and allocates a second graph, so the total is still O(V+E) time but roughly doubles the edge storage.

Final table from the lecture:

| Problem | Description | Solution | Efficiency |
|---|---|---|---|
| topological sort | Ordering of vertices respecting DAG edges | `Topological.java` | O(V+E) time, Θ(V) space |
| DAG shortest paths | Shortest paths tree on a DAG | `AcyclicSP.java` | O(V+E) time, Θ(V) space |
| longest paths | Longest paths tree on a general graph | **no known efficient solution** | `O(???)` |
| DAG longest paths | Longest paths tree on a DAG | flip signs, run DAG-SPT, flip signs again | O(V+E) time, Θ(V) space |

### Example 6: The 3SAT to Independent-Set reduction, walked through

Lecture formula:

```
phi = (x1 or x2 or x3) and (!x1 or !x2 or x4) and (!x1 or x3 or !x4) and (x1 or x3 or x4)
```

**Preprocess.** Four clauses, so four triangles, 12 vertices total:

```
T1: x1  --  x2  --  x3  (triangle)
T2: !x1 -- !x2 --  x4   (triangle)
T3: !x1 --  x3 -- !x4   (triangle)
T4: x1  --  x3 --  x4   (triangle)
```

Then add a "conflict" edge between every literal and its negation across triangles: every `x1` vertex (in T1 and T4) connects to every `!x1` vertex (in T2 and T3), every `x2` to every `!x2`, every `x4` to every `!x4`, and so on.

**Call Independent-Set with k = 4** (one per clause).

**Postprocess.** Read off the four chosen vertices and set each chosen literal to true. Because the triangles force exactly one pick per clause, every clause gets a true literal; because the conflict edges forbid picking both `xi` and `!xi`, the assignment is well defined.

Worked choice: pick `x3` from T1, `!x1` from T2, `!x1`... no, each triangle contributes a distinct vertex, so pick `x3` from T1, `!x1` from T2, `x3` from T3, `x3` from T4. Those four vertices are pairwise non-adjacent (no two are in the same triangle, and `x3` never conflicts with `!x1` or with another `x3`). Postprocessing gives `x3 = true, x1 = false`, with `x2` and `x4` free. Check: clause 1 has `x3` true; clause 2 has `!x1` true; clause 3 has `x3` true; clause 4 has `x3` true. Satisfied.

> (Extra context, two notes where the slides are loose.) First, the slide labels the target size as "k = number of variables = 4"; the correct reduction uses **k = number of clauses**, which also happens to be 4 in this example, so the demo still works. If you ever reproduce the reduction with a formula where the two counts differ, use the clause count. Second, the assignment printed on the final slide (`x1: true, x2: false, x3: false, x4: true`) does not satisfy clause 3 of the formula as transcribed, since `!x1`, `x3`, and `!x4` are all false under it; `x1 = false, x3 = true` does satisfy the formula. Treat the printed assignment as a slide artifact rather than a fact to memorize, and remember the structural lesson instead: triangles enforce "one literal per clause," conflict edges enforce "consistency."

The direction, once more: since IND-SET can be used to solve 3SAT, **3SAT reduces to IND-SET**. The lecture stresses that 3SAT is not a graph problem at all, and that reductions need not produce graphs.

---

## Common Pitfalls

1. **Using preorder instead of postorder.** Reverse *postorder* is a topological order. Reversing a preorder, or using postorder un-reversed, is wrong in general. (Un-reversed postorder is a topological order of the *reverse* graph.)
2. **Clearing the `marked` array between restarts.** The whole algorithm depends on one global postorder list built over one persistent `marked` array. Resetting marks re-adds vertices and destroys the order.
3. **Thinking you must start at indegree-0 vertices.** The lecture's extra slide says you do not: restarting from any unmarked vertex also works. Starting at indegree-0 vertices is intuitive but not required.
4. **Believing a topological order is unique.** The lecture lists multiple valid orderings for the A-through-H graph (`[A, C, B, D, F, E, H, G]`, `[C, A, D, F, B, E, G, H]`, and the DFS output `[C, F, G, A, D, B, E, H]`). "Give a topological order" means produce one, and your correct answer may differ from the posted one.
5. **Running topological sort on a graph with a cycle.** No ordering exists. The lecture shows the A-through-H picture with a cycle added and states there is no possible ordering where all arrows are respected. Also, the DFS code above will happily return *something* on a cyclic digraph, and that something is not a topological order, so cycle detection is a separate concern.
6. **Using Dijkstra's with negative edges "because it usually works."** It can silently produce wrong answers, as the demo shows, and the wrong answer may be only on some vertices. A finalized vertex is never revisited.
7. **Trying to fix negative weights by adding a constant to every edge.** (Extra context, a classic trap.) This changes the problem: adding `c` to every edge adds `c * (number of edges on the path)`, which penalizes long paths and can change which path is shortest. Do not do it.
8. **Using DAG-SPT on a graph with a cycle.** There is no topological order to iterate, so step 1 fails. The lecture is explicit: if a graph has negative edges and cycles, you cannot use the DAG algorithm.
9. **Thinking DAG-SPT needs a priority queue.** It does not. The topological order *is* the visiting order. Adding a PQ would only slow it to O(E log V).
10. **Miscounting the runtime as O(V·E).** The lecture calls this out directly: E is the total edge count in the whole graph, not edges per vertex, so relaxing all outgoing edges at every vertex costs Θ(E) total.
11. **Negating the graph in place.** Mutating the caller's graph is a side effect that corrupts the input. Build a copy G', as the slide says ("form a new copy of the graph G'").
12. **Forgetting to negate `distTo` back, or needlessly "fixing" `edgeTo`.** `distTo` must be flipped; `edgeTo` must not be touched.
13. **Thinking sign flipping solves longest paths in general.** It only moves you from "longest on a DAG" to "shortest with negative weights on a DAG", which you can solve. On a general graph, flipping gives you negative weights plus cycles, which is precisely the case DAG-SPT cannot handle, and general longest simple paths has no known efficient algorithm.
14. **Reversing the direction of a reduction.** "P reduces to Q" means a subroutine for **Q** solves **P**. DAG-LPT reduces to DAG-SPT (SPT is the tool). 3SAT reduces to IND-SET (IND-SET is the tool). Saying it backwards inverts the entire meaning, including what the reduction implies about difficulty.
15. **Confusing "longest path" with "longest walk."** The problem requires a **simple** path. Without simplicity, a positive cycle makes the answer unbounded. The requirement is automatically satisfied on a DAG, which is why the DAG version is easy.

---

## Likely Exam Points

### 1. Produce a topological order from a drawn DAG

**Q.** For the digraph with edges `A -> B`, `A -> D`, `D -> B`, `D -> E`, `B -> C`, `C -> E`, `C -> F`, `E -> F`, give a topological ordering.

**A.** `A, D, B, C, E, F`. (Check each edge points forward.) Via the algorithm: DFS postorder is `F, E, C, B, D, A`, and reversing gives the answer. This is the lecture's own "Test Your Understanding" question and answer.

### 2. Execute DFS reverse postorder precisely

**Q.** Run topological sort on `A -> B`, `A -> D`, `B -> E`, `C -> D`, `C -> F`, `D -> E`, `E -> H`, `F -> G`, starting DFS at A and then restarting at C, visiting neighbors in alphabetical order. Give the postorder list and the final ordering.

**A.** Postorder `[H, E, B, D, A, G, F, C]`; topological order is its reverse, `[C, F, G, A, D, B, E, H]`. The key moments: H is finished first because it is the deepest vertex on the first descent, D finishes immediately when reached from A because its only successor E is already marked, and the restart at C does not clear marks so D is skipped.

### 3. Count the topological orderings of a small DAG

**Q.** How many distinct topological orderings does the DAG with edges `A -> B`, `A -> C`, `B -> D`, `C -> D` have?

**A.** 2: `A, B, C, D` and `A, C, B, D`. A must be first (it is the only indegree-0 vertex) and D must be last (only vertex with no outgoing edges); B and C are incomparable, so either order works. (Technique: identify forced positions, then count the free interleavings of mutually unreachable vertices.)

### 4. Does a topological order exist?

**Q.** True or false: every directed graph has a topological ordering. If false, state the exact condition.

**A.** False. A topological ordering exists if and only if the graph is a DAG, that is, has no directed cycle. A cycle `v1 -> ... -> v1` would require `v1` to precede itself.

### 5. Why Dijkstra's fails on negative edges, mechanically

**Q.** On the digraph `A -> B (1)`, `A -> D (1)`, `D -> B (25)`, `D -> E (1)`, `B -> C (6)`, `C -> E (-20)`, `C -> F (1)`, `E -> F (1)`, run Dijkstra's from A. What `distTo[F]` does it report, what is the true answer, and why do they differ?

**A.** Dijkstra's reports `distTo[F] = 3` (via `A -> D -> E -> F`). The true shortest distance is `-12` via `A -> B -> C -> E -> F` (1 + 6 - 20 + 1). They differ because Dijkstra's visits in increasing `distTo` order, so it finalizes E at 2 and F at 3 before ever visiting C at 7. When `C -> E` is finally relaxed and `distTo[E]` drops to -13, F has already been removed from the fringe and is never reconsidered. The resulting `distTo` even violates `distTo[F] <= distTo[E] + 1`.

### 6. State and justify the DAG-SPT algorithm

**Q.** Give an O(V+E) algorithm for single-source shortest paths on a DAG with possibly negative edge weights, and argue correctness in two sentences.

**A.** Compute a topological order via DFS reverse postorder (O(V+E)), then visit the vertices in that order, relaxing every outgoing edge of each vertex (O(V+E) since each edge is relaxed exactly once). Correctness: in a topological order, every edge into `v` originates at a vertex earlier in the order, so by the time `v` is visited every edge that could improve `distTo[v]` has already been relaxed and `distTo[v]` is final; this argument never uses non-negativity, so negative weights are fine.

### 7. Fill in the runtime table

**Q.** Give time and space for topological sort, DAG shortest paths, and DAG longest paths.

**A.** All three: O(V+E) time, Θ(V) space. Topological sort uses `marked` plus the postorder stack plus O(V) recursion depth. DAG-SPT adds `distTo` and `edgeTo`, both Θ(V). DAG-LPT is DAG-SPT plus Θ(V+E) preprocessing and Θ(V) postprocessing, which does not change the asymptotics. (Compare: Dijkstra's is O(E log V).)

### 8. The DAG longest paths trick

**Q.** You are given an O(V+E) subroutine for DAG shortest paths. Using it as a black box, give an O(V+E) algorithm for the longest paths tree on a DAG. Which outputs need post-processing?

**A.** Build a copy G' of G with every edge weight negated. Run DAG-SPT on G' from `s`, getting result X. Negate every value in `X.distTo` to get the longest-path distances. `X.edgeTo` needs no change, since the tree of chosen edges is identical and only the path totals were negated. Correctness: the paths from `s` to `v` are the same set in G and G', each path's weight is exactly negated, and minimizing the negated total maximizes the original total, because `-(-a + -b + -c + -d) = a + b + c + d`.

### 9. Why the trick does not generalize

**Q.** Why can you not solve longest paths on an arbitrary weighted digraph by negating weights and running your favorite shortest paths algorithm?

**A.** Negating produces negative weights, so Dijkstra's is invalid, and if the graph has cycles then DAG-SPT is also inapplicable (no topological order exists). Longest *simple* paths on a general graph has no known efficient algorithm; the best known is exponential. Acyclicity is what makes the reduction work, both because it supplies a topological order and because in a DAG every path is automatically simple.

### 10. Direction of a reduction

**Q.** The lecture says "DAG-LPT reduces to DAG-SPT." Which problem's solver is used as a subroutine? Also, does DAG-SPT reduce to DAG-LPT?

**A.** DAG-SPT's solver is the subroutine; DAG-LPT is the problem being solved. Using the definition "if any subroutine for task Q can be used to solve P, we say P reduces to Q", here P = DAG-LPT and Q = DAG-SPT. And yes, DAG-SPT also reduces to DAG-LPT by the same sign flip in reverse, so the two problems are interreducible and equally hard up to O(V+E) overhead.

### 11. Identify a reduction in earlier course material

**Q.** Name two reductions from earlier in CS 61B, as the lecture does.

**A.** Percolation reduces to disjoint sets (union-find is the subroutine). Finding the hyponyms of a word reduces to DFS, or equivalently BFS (graph traversal is the subroutine). More generally, DAG shortest paths reduces to topological sort, since `AcyclicSP` calls `Topological` as a subroutine.

### 12. Identify the right tool for a "tasks with prerequisites" word problem

**Q.** You have 500 build targets with "must build X before Y" constraints and want a build order, or a report that the constraints are contradictory. What do you run, and what is the runtime?

**A.** Model targets as vertices and each constraint as a directed edge `X -> Y`, then topologically sort with DFS reverse postorder, O(V+E). The constraints are contradictory exactly when the digraph has a directed cycle, so pair the sort with cycle detection (for example, Kahn's algorithm, which ends with vertices remaining if and only if a cycle exists, or a DFS that detects a back edge to a vertex currently on the call stack). (The cycle-detection mechanics are extra context; the lecture states only that a topological order exists if and only if the graph is a DAG.)

### 13. Independent set and 3SAT definitions (bonus material)

**Q.** Define an independent set, and describe the first step of the reduction from 3SAT to Independent-Set.

**A.** An independent set is a set of vertices in which no two vertices are adjacent; the Independent-set problem asks whether one of size k exists. In the reduction, for each clause of the 3SAT formula you create 3 vertices (one per literal) joined into a triangle, and then you add an edge between each literal and its negation. The triangles force at most one literal per clause into the independent set, and the negation edges prevent selecting both `xi` and `!xi`, so an independent set of size k = (number of clauses) corresponds exactly to a satisfying assignment. (The lecture labels this optional CS 170 preview material.)

---

## Summary

- A **DAG** is a directed graph with no directed cycles. A **topological order** (every edge pointing forward) exists if and only if the graph is a DAG.
- **Topological sort algorithm:** run DFS **with restarts** (never clearing `marked`), record **postorder**, and output the **reverse postorder**. O(V+E) time, Θ(V) space, implemented in `Topological.java`.
- Restarting from indegree-0 vertices is intuitive but unnecessary: restarting from any unmarked vertex also yields a valid topological order.
- "DFS" is ambiguous: `DepthFirstPaths` does not restart, topological sort does. Read the problem.
- **Dijkstra's fails with negative edges.** A vertex finalized early (F at 3 in the demo) is never revisited, so a later negative relaxation (`C -> E` at -20) cannot propagate, and `distTo` ends up wrong and even self-inconsistent.
- **DAG-SPT:** find a topological order, then visit vertices in that order relaxing all outgoing edges. Correct even with negative weights, because every edge into `v` is relaxed before `v` is visited. O(V+E) time, Θ(V) space, no priority queue, implemented in `AcyclicSP.java`, which calls `Topological` as a subroutine.
- The runtime is O(V+E), not O(V·E): E counts all edges in the graph, and each edge is relaxed exactly once.
- Negative weights **plus** cycles is a different and much harder story; it was open for 40 years and solved in 2022 by Bernstein, Nanongkai, and Wulff-Nilsen.
- **Longest simple paths** on a general graph has no known efficient algorithm (best known is exponential), and the lecture frames it as perhaps the most important unsolved problem in mathematics, in the precise sense that vast numbers of important problems reduce to it and it reduces back to them, so solving any solves all (NP-completeness, possibly revisited in the last week).
- **DAG-LPT:** copy the graph with all weights negated, run DAG-SPT, negate all `distTo` values; `edgeTo` is already correct. O(V+E). Justified by two facts: DAG-SPT tolerates negative weights, and `-(-a + -b + -c + -d) = a + b + c + d`.
- **Reduction:** "If any subroutine for task Q can be used to solve P, we say P reduces to Q." The pattern is preprocess, call the subroutine, postprocess. Mind the direction: DAG-LPT reduces to DAG-SPT; 3SAT reduces to IND-SET.
- Reductions are just the formal version of the decomposition we have done all semester (percolation via disjoint sets, hyponyms via DFS/BFS). Reductions need not involve graphs, and the preprocess step can be genuinely creative, as in 3SAT to Independent Set, where each clause becomes a triangle and each literal is joined to its negation.
