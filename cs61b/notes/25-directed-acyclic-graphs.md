<!-- Fri, Oct 23, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 25: Directed Acyclic Graphs

This lecture closes out the graph unit by using tools already in hand (DFS, relaxation, Dijkstra's) to attack harder problems on a restricted but extremely common class of graphs: directed acyclic graphs (DAGs). It opens with a warmup showing exactly how Dijkstra's algorithm breaks on negative edge weights (it commits to a vertex the first time it is popped, and a later negative edge can beat that committed distance), then develops topological sort via DFS reverse postorder, then uses topological order as a replacement for Dijkstra's priority queue to get an O(V+E) shortest paths algorithm on DAGs that tolerates negative weights. From there the lecture poses the longest paths problem, reveals that no efficient general algorithm is known for it (it is tied to the biggest open question in the field), and then solves the DAG case with a one-trick transformation: flip the sign of every edge, run DAG shortest paths, flip the distances back. That trick is the real punchline, introducing reduction: solving problem P by preprocessing its input, calling a solver for a different problem Q, and postprocessing the answer. The lecture closes with an optional preview of a much richer reduction (3SAT to Independent Set) to show that reductions are not just sign flipping and are not limited to graph problems.

## Key Concepts

### Why Dijkstra's breaks on negative edges

Dijkstra's algorithm is greedy, and its correctness rests on one assumption: when a vertex is popped from the priority queue and visited, its `distTo` value is final and can never improve. Intuitively, Dijkstra's is an expanding radius around the source. On the warmup graph (all positive weights), the radius expands to `A` (0), `B` (1), `D` (2), `E` (5), `F` (6), `C` (7), so the visit order is A, B, D, E, F, C. Note that distances are always measured from the source, not from the most recently visited vertex, so `E` is at distance 5 (2 + 3 via `D`), not 3.

That "once seen, never improved" guarantee holds only because all weights are nonnegative. Any alternative path to an already-visited vertex must go through more edges, and more nonnegative edges can only add to the total, so a longer detour can never undercut the committed distance.

Negative edges destroy this. In the second warmup graph, `C` is the last vertex visited, and `C` has an edge to `E` of weight -20. By the time Dijkstra's visits `C`, it has already committed `distTo[E] = 2` (via `D`) and `distTo[F] = 3`. Relaxing the `C -> E` edge gives 7 + (-20) = -13, which does improve `distTo[E]`, but `F` was already finalized at 3 and will never be reconsidered. The correct `distTo[F]` is -13 + 1 = -12, so Dijkstra's leaves a wrong answer in the table. The lecture's phrase for this is that `E` was "prematurely finalized": Dijkstra's used `E` before all the information about `E` was available.

A real-world reason negative weights come up (lecture's example): an influencer needs a guitar pedal to make a video. Buying it at a store costs money (positive edge), but talking to the manufacturer first may mean they pay the influencer to feature it (negative edge). Costs along a dependency chain are not always positive.

### Directed acyclic graphs

A directed graph's edges have direction, so a sequence of vertices that looks like a loop on paper may not be traversable as one. In the lecture's `A` through `H` task graph, `C -> D -> E -> F` is not a cycle, because the `E -> F` arrow points the wrong way: you can walk `C` to `D` to `E`, and then you are stuck. Acyclic means no directed cycle exists anywhere.

DAGs are the natural model for dependency graphs. If `v -> w` means "v must happen before w," then a cycle is a logical contradiction. The lecture gives the course-prerequisite version: if 61A were a prerequisite for 61B, 61B for 70, 70 for 170, and 170 for 61A, there would be no legal first course. The laundry version is the same idea: you cannot run the washing machine until you have both detergent and coins.

### Topological sort

A topological sort (also called a topological ordering) of a DAG is an ordering of all vertices such that every edge points forward in the ordering. If you lay out the vertices along a line in topological order, every arrow points rightwards, and nothing depends on anything to its right.

A topological ordering exists if and only if the graph is a DAG. The forward direction is the useful one in practice (every DAG has at least one topological ordering, and often many). The reverse direction is the explanation for why cycles are fatal: in a cycle `A -> B -> E -> H -> A`, whichever of those vertices you place first in your ordering, some edge in the cycle must eventually point backwards to it.

The natural first guess is BFS by levels, and on the lecture's example graph that happens to work. But the lecture shows it breaks: add an edge `A -> F` while `D -> F` already exists, and plain level-order BFS enqueues `B`, `D`, and `F` together, possibly emitting `F` before `D`, violating `D -> F`. (The lecture notes that a careful BFS-based algorithm, Kahn's algorithm, does work, but is not covered today.)

The algorithm this lecture uses instead:

> Perform a DFS traversal from every vertex with indegree 0, NOT clearing markings in between traversals. Record DFS postorder in a list. The topological ordering is the reverse of that list (reverse postorder).

Three details matter. First, you start only from indegree-0 vertices (vertices with no incoming edges, so no unmet dependencies). Second, you do not reset the marked array between those restarts, so a vertex already visited from one root is never revisited from another. Third, you record each vertex in *postorder*, meaning after all of its outgoing edges have been fully explored, and then reverse the whole list.

Postorder alone is not a valid ordering. On the lecture's graph, postorder is `[H, E, B, D, A, G, F, C]`, which starts with `H`, a vertex that depends on several others. The reverse, `[C, F, G, A, D, B, E, H]`, is valid. The intuition is that a vertex is appended to the postorder list only after everything reachable from it is already in the list, so reversing puts every vertex ahead of everything it points to.

The lecture also notes a slicker variant (marked as an extra slide): you do not actually need to find the indegree-0 vertices. Run DFS from an arbitrary vertex, and if not everything is marked, pick any unmarked vertex and run again, repeating until done. This still produces a valid reverse postorder on a DAG.

A terminology warning the lecture calls out explicitly: "depth first search" sometimes means with restarts and sometimes without. `DepthFirstPaths` for reachability from a single source does not restart. Topological sort does restart.

Runtime: O(V + E) time, Θ(V) space, because it is a DFS traversal of the whole graph.

### Shortest paths on DAGs

The fix for Dijkstra's failure is to replace the priority queue's notion of "which vertex next" with topological order. The algorithm:

1. Compute a topological ordering of the DAG. O(V + E).
2. Visit vertices in that order. On each visit, relax all outgoing edges of that vertex. O(V + E).

The correctness intuition is exactly the complement of Dijkstra's failure mode: in topological order, every vertex is visited only after every vertex that can possibly reach it has already been visited and relaxed. So when you visit `v`, all possible information about `distTo[v]` has already been used, and `distTo[v]` is final. There is no premature finalization, which is why negative weights are harmless here. As one student put it in lecture, topological sort "requires that all previous dependencies be resolved."

Important differences from Dijkstra's, worth internalizing because exams probe them:

- **No priority queue.** The visit order is a fixed list computed once up front. Relaxing an edge updates `distTo` and `edgeTo` but never reorders anything.
- **No "already visited so skip" subtlety about distances.** Each vertex is visited exactly once, and its `distTo` is correct at the moment of visit.
- **Negative edges are fine.** Negative *cycles* are impossible by assumption, since a DAG has no cycles at all.

Runtime: O(V + E) total, Θ(V) space. A frequent student question the lecture answers directly: why is step 2 not O(V * E)? Because `E` is the total number of edges in the entire graph, not the number of edges per vertex. Summing the outdegrees over all vertices gives `E`, not `V * E`.

Why keep Dijkstra's at all, if this is O(V + E) and handles negative weights? Because the two algorithms solve different problems. Dijkstra's works on any weighted digraph, including those with cycles, as long as weights are nonnegative. DAG shortest paths requires acyclicity and directedness, but tolerates negative weights. Neither dominates the other. (Dijkstra's O(E log V) comes from the priority queue: in the worst case every edge triggers a priority update, each costing log V.)

### Longest paths

The longest paths tree (LPT) problem: find, from source `s`, the longest *simple* path to every other vertex. Simple means no repeated vertices. The simplicity requirement is essential, because with a cycle present you could loop forever and accumulate unbounded weight.

The lecture asked the class to guess the runtime, offering log V, V, E, E log V, E², E³, E⁴, E¹⁰, E¹⁰⁰, and then revealed the answer: the best known algorithm is **exponential**. There is no known efficient solution. The lecture frames this as "perhaps the most important unsolved problem in mathematics," with the caveat that this is both true and misleading, which the reductions section then unpacks.

Two exercises the lecture suggests doing afterward:
- Find an example where the obvious greedy algorithm (Dijkstra's but always pick the biggest edge first) fails.
- Figure out whether the longest path to every other vertex is always a tree, i.e. whether an LPT always exists for all graphs.
- Separately recommended: search for "why longest paths hard" and try to articulate in English why it is hard.

### Longest paths on DAGs, and the sign-flip trick

On a DAG, longest paths becomes easy, in O(V + E):

1. Form a new graph `G'` that is a copy of `G` with the sign of every edge weight flipped.
2. Run DAG shortest paths (DAG-SPT) on `G'`, yielding a result `X`.
3. Flip the signs of all values in `X.distTo`. `X.edgeTo` is already correct.

Why this works: the most negative total in `G'` corresponds to the most positive total in `G`. The lecture presents this under the heading "A Note on Mathematical Maturity": if you have enough of it, the algorithm should look plainly correct with no proof or demo needed, because you already know DAG-SPT handles negative edge weights, and you already know that -(-a + -b + -c + -d) = a + b + c + d.

Note the asymmetry in step 3: distances need un-flipping because they are sums of weights, but `edgeTo` just records which edge was used to reach each vertex, and the sign flip does not change the identity of any edge.

Also note why the DAG restriction helps even though simple paths already forbid cycles: a DAG *guarantees* no cycles up front, so the algorithm never has to track whether the path it is building has revisited a vertex. That bookkeeping is the expensive part in the general case.

### Reductions

The sign-flip solution has a distinctive shape. Given a graph `G`, you did not solve the problem directly. Instead you:

```
G --[preprocess]--> G' --[DAG-SPT]--> SPT of G' --[postprocess]--> LPT of G
```

This is a **reduction**. Because DAG-SPT can be used to solve DAG-LPT, we say "DAG-LPT reduces to DAG-SPT." The informal definition from *Algorithms* by Dasgupta, Papadimitriou, and Vazirani, as quoted in lecture: "If any subroutine for task Q can be used to solve P, we say P reduces to Q."

The lecture's analogies:
- You know how to make tomato soup but not pizza sauce. Boil the soup down until it thickens and you have pizza sauce. Making pizza sauce reduces to making tomato soup (with boiling as the postprocessing step).
- Finding the median in a list reduces to sorting the list (sort, then look at the middle).
- Reading a German book reduces to using a German-to-English translator.

A reduction can be formalized (Karp reductions, Cook reductions) but that is well beyond the scope of 61B.

The crucial follow-up: the claim that longest paths is "the most important unsolved problem in mathematics" is misleading precisely *because* of reductions. Vast numbers of important problems reduce to longest paths, and longest paths also reduces back to those. So solving any one of them solves all of them. This is the circle of NP-complete problems, which the lecture says may be revisited in the last week of class. The lecturer notes they had never heard of the longest paths problem by that name while prepping, yet had encountered the same problem many times in different disguises, which is the whole point: reductions reveal that apparently different problems are one problem wearing different costumes.

Christos Papadimitriou's line, quoted in lecture: in complexity theory "we don't know how to create more answers, but we know how to create fewer questions."

Finally, the lecture points out that reductions are just a formal version of what the course has been doing all along: percolation solved with disjoint sets, finding hyponyms of a word solved with DFS or BFS. At its heart, computer science is breaking a complex task into smaller parts, and good abstractions make problem solving vastly easier.

### Reduction to 3SAT (optional CS170 preview)

This section is explicitly marked optional and bonus. Its purpose is to show that reductions are richer than sign flipping and are not limited to graph problems.

**Independent set:** a set of vertices in which no two vertices are adjacent. The Independent-Set problem asks: does there exist an independent set of size `k`? Equivalently, can you color `k` vertices red such that no two red vertices touch?

**3SAT:** given a boolean formula expressed as a conjunction of 3-variable disjunctive clauses, does there exist an assignment of truth values satisfying all constraints? Example: `(x1 || x2 || !x3) && (x1 || !x1 || x1) && (x2 || x3 || x4)`, satisfied by x1 = true, x2 = true, x3 = true, x4 = false.

**Proposition: 3SAT reduces to Independent-Set.** Given a 3SAT instance φ, construct a graph `G`:
- For each clause in φ, create 3 vertices (one per literal) joined in a triangle. The triangle means at most one vertex per clause can be in an independent set, which encodes "pick one literal from this clause to satisfy it."
- Add an edge between each literal and its negation. Since `x` and `!x` cannot both be true in 3SAT, they must not both be chosen in the independent set.

Then set `k` to the number of clauses, and an independent set of size `k` corresponds to a consistent choice of one true literal per clause, which is a satisfying assignment. (The slides' worked example uses φ = `(x1 or x2 or x3) and (!x1 or !x2 or x4) and (!x1 or x3 or !x4) and (x1 or x3 or x4)` with k = 4 and 4 clauses, and the postprocessing step reads off x1: true, x2: false, x3: false, x4: true.)

The lecture draws the same reduction diagram as before, and stresses two things: 3SAT is not a graph problem, and reductions do not always involve creating graphs.

## Definitions

**Directed graph (digraph):** a graph whose edges have a direction, written `v -> w`, meaning the edge can be traversed from `v` to `w` but not the reverse.

**Directed cycle:** a sequence of vertices `v1 -> v2 -> ... -> vk -> v1` in which every consecutive pair is connected by a directed edge in that direction, returning to the start.

**Directed acyclic graph (DAG):** a directed graph containing no directed cycle.

**Dependency graph:** a DAG in which an edge `v -> w` means "task v must happen before task w." Cycles are logically impossible in such a graph.

**Indegree of a vertex:** the number of edges pointing into that vertex. A vertex with indegree 0 has no unmet dependencies.

**Topological sort (topological ordering):** an ordering of all vertices of a DAG such that for every edge `v -> w`, `v` appears before `w`. Equivalently, if vertices are laid out left to right in this order, every edge points rightwards.

**Postorder (of a DFS traversal):** the order in which vertices are finished, i.e. each vertex is appended to the list after all of its outgoing edges have been fully explored.

**Reverse postorder:** the postorder list reversed. For a DAG explored by DFS with restarts, this is a valid topological ordering.

**DFS with restarts:** running DFS repeatedly from new starting vertices without clearing the marked array between runs, so the union of all traversals covers the whole graph and no vertex is visited twice.

**Relaxation (of edge v -> w with weight x):** if `distTo[v] + x < distTo[w]`, set `distTo[w] = distTo[v] + x` and `edgeTo[w] = v -> w`.

**Shortest paths tree (SPT):** a tree rooted at the source `s` containing, for each reachable vertex, a shortest path from `s` to that vertex.

**Simple path:** a path that does not repeat any vertex.

**Longest paths tree (LPT):** a structure giving, from source `s`, the longest simple path to every other vertex.

**Reduction:** "If any subroutine for task Q can be used to solve P, we say P reduces to Q." (Dasgupta, Papadimitriou, Vazirani) Operationally: preprocess P's input into an input for Q, call Q's solver, postprocess Q's output into an answer for P.

**Independent set:** a set of vertices in which no two vertices are adjacent.

**Independent-Set problem:** given a graph and an integer `k`, does there exist an independent set of size `k`?

**3SAT:** given a boolean formula as a conjunction of 3-variable disjunctive clauses, does there exist a truth assignment satisfying all of them?

## Worked Examples

### Example 1: Topological sort by DFS reverse postorder

The lecture's graph has vertices `A` through `H` with edges: `A -> B`, `A -> D`, `B -> E`, `C -> D`, `C -> F`, `D -> E`, `E -> H`, `F -> G`. (Reconstructed from the demo's call stacks and postorder; the slide diagrams are missing.) Vertices `A` and `C` have indegree 0.

```java
public class Topological {
    private boolean[] marked;
    private List<Integer> postorder;
    private Iterable<Integer> order;

    public Topological(Digraph G) {
        marked = new boolean[G.V()];
        postorder = new ArrayList<>();

        for (int v = 0; v < G.V(); v += 1) {
            if (indegreeZero(G, v) && !marked[v]) {
                dfs(G, v);
            }
        }

        Collections.reverse(postorder);
        order = postorder;
    }

    private void dfs(Digraph G, int v) {
        marked[v] = true;
        for (int w : G.adj(v)) {
            if (!marked[w]) {
                dfs(G, w);
            }
        }
        postorder.add(v);
    }

    public Iterable<Integer> order() {
        return order;
    }
}
```

Step by step through the lecture's demo, tracking the call stack and the postorder list:

| Step | Call stack | Postorder | What happened |
|---|---|---|---|
| 1 | `A` | `[]` | Start DFS at `A` (indegree 0). Mark `A`. |
| 2 | `A -> B` | `[]` | Recurse into `B`. Mark `B`. |
| 3 | `A -> B -> E` | `[]` | Recurse into `E`. Mark `E`. |
| 4 | `A -> B -> E -> H` | `[]` | Recurse into `H`. Mark `H`. |
| 5 | `A -> B -> E` | `[H]` | `H` has no outgoing edges, so it is finished. Append `H`, return. |
| 6 | `A -> B` | `[H, E]` | `E`'s only neighbor `H` is marked, so `E` is finished. Append `E`. |
| 7 | `A` | `[H, E, B]` | `B`'s only neighbor `E` is marked. Append `B`. |
| 8 | `A -> D` | `[H, E, B]` | Back at `A`, next neighbor is `D`. Recurse. Mark `D`. |
| 9 | `A` | `[H, E, B, D]` | `D`'s neighbor `E` is already marked, so nothing to explore. Append `D`. |
| 10 | (empty) | `[H, E, B, D, A]` | `A` is out of neighbors. Append `A`. First DFS done. |
| 11 | `C` | `[H, E, B, D, A]` | Restart from `C`, the other indegree-0 vertex. Marks are NOT cleared. Mark `C`. |
| 12 | `C -> F` | `[H, E, B, D, A]` | `C`'s neighbor `D` is already marked (crucially, from the first traversal), so skip it. Recurse into `F`. |
| 13 | `C -> F -> G` | `[H, E, B, D, A]` | Recurse into `G`. Mark `G`. |
| 14 | `C -> F` | `[H, E, B, D, A, G]` | `G` is finished. Append `G`. |
| 15 | `C` | `[H, E, B, D, A, G, F]` | `F` is finished. Append `F`. |
| 16 | (empty) | `[H, E, B, D, A, G, F, C]` | `C` is finished. Append `C`. |

Reversing gives the topological ordering `[C, F, G, A, D, B, E, H]`.

Environment reasoning in words: each nested `dfs` call pushes a new frame onto the call stack, and each frame holds its own `v` and its own position in the iteration over `G.adj(v)`. The `marked` array and `postorder` list, in contrast, live on the heap and are shared by every frame: `marked` is a single array object that all frames mutate, which is exactly why step 12 can see that `D` was marked back in step 8, and why step 9 can see that `E` was marked in step 3. The `postorder.add(v)` line sits *after* the loop in each frame, so a vertex is only appended once its frame is about to pop, which is what makes the list a postorder rather than a preorder. If `postorder.add(v)` were placed before the loop instead, you would get preorder (`A, B, E, H, D, C, F, G` here), and reversing preorder is not a topological sort in general.

Why the ordering is not unique: the lecture gives two valid orderings for this graph, `[A, C, B, D, F, E, H, G]` and `[C, A, D, F, B, E, G, H]`, plus the reverse-postorder answer above. Any ordering respecting all edges is correct.

### Example 2: A graph whose topological sort is unique

The warmup graph, re-read as a DAG: vertices `A, B, C, D, E, F` with directed edges `A -> B` (1), `A -> D` (2), `D -> B` (4), `B -> C` (6), `C -> E` (1), `D -> E` (3), `E -> F` (1), `C -> F` (2).

The lecture asks: starting at `A`, can `B` come next? No, because `B` depends on `D` (there is a `D -> B` edge), and `D` has not been emitted. Can `E` come next after `A, D, B`? No, because `E` depends on `C`. Working through:

- `A` first (only indegree-0 vertex).
- Then `D`, since `B` still needs `D`.
- Then `B`.
- Then `C`, since `E` still needs `C`.
- Then `E`.
- Then `F`.

Result: `A, D, B, C, E, F`, and it is the *only* topological ordering of this graph, because at each step exactly one vertex has all its dependencies satisfied. Laid out on a line, every arrow points rightwards:

```
s -> A --1--> D --4--> B --6--> C --1--> E --1--> F
       \_______2_______/   \____3____/      \__2__/   (all forward)
```

The lecture notes one way to find this ordering is DFS postorder `F, E, C, B, D, A`, reversed.

### Example 3: DAG shortest paths with a negative edge

This is the lecture's full demo. Topological order is `A, D, B, C, E, F`. Edges and weights: `A -> D` (1), `A -> B` (1), `D -> B` (25), `D -> E` (1), `B -> C` (6), `C -> E` (-20), `C -> F` (1), `E -> F` (1).

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

        Topological top = new Topological(G);
        for (int v : top.order()) {
            for (DirectedEdge e : G.adj(v)) {
                relax(e);
            }
        }
    }

    private void relax(DirectedEdge e) {
        int v = e.from();
        int w = e.to();
        if (distTo[w] > distTo[v] + e.weight()) {
            distTo[w] = distTo[v] + e.weight();
            edgeTo[w] = e;
        }
    }
}
```

Trace, following the lecture's table:

**Initialize.** `distTo = [A:0, B:inf, C:inf, D:inf, E:inf, F:inf]`, all `edgeTo` null. Order list: `[A, D, B, C, E, F]`.

**Visit A.** Relax `A -> D` (weight 1): `inf > 0 + 1`, so `distTo[D] = 1`, `edgeTo[D] = A`. Relax `A -> B` (weight 1): `distTo[B] = 1`, `edgeTo[B] = A`.

The lecture pauses here to ask what Dijkstra's would do differently at this moment. Answer: Dijkstra's would also push `D` and `B` into a priority queue with priority 1, i.e. it would reorder the pending work. DAG-SPT does not. The visit order is the fixed topological list, immutable once computed.

**Visit D.** Relax `D -> B` (weight 25): `distTo[B] + ...` check is `1 > 1 + 25`? No. 26 is worse than the direct edge's 1, so no update. This is a useful check that relaxation only improves. Relax `D -> E` (weight 1): `inf > 1 + 1`, so `distTo[E] = 2`, `edgeTo[E] = D`. Note 2, not 1, because distances are from the source.

**Visit B.** Relax `B -> C` (weight 6): `inf > 1 + 6`, so `distTo[C] = 7`, `edgeTo[C] = B`. Again 7, not 6: one hop `A -> B` costing 1, plus 6.

**Visit C.** This is the step that broke Dijkstra's. Here `E` has *not* been finalized, because `E` comes after `C` in topological order. Relax `C -> F` (weight 1): `inf > 7 + 1`, so `distTo[F] = 8`, `edgeTo[F] = C`. Relax `C -> E` (weight -20): `2 > 7 + (-20) = -13`? Yes, so `distTo[E] = -13`, `edgeTo[E] = C`. The negative edge is handled correctly because `E` was still open.

**Visit E.** Relax `E -> F` (weight 1): `8 > -13 + 1 = -12`? Yes, so `distTo[F] = -12`, `edgeTo[F] = E`.

**Visit F.** No outgoing edges. Done.

Final: `distTo = [A:0, B:1, C:7, D:1, E:-13, F:-12]`, `edgeTo = [A:-, B:A, C:B, D:A, E:C, F:E]`. Compare with Dijkstra's on the same graph, which left `distTo[F] = 3`, a wrong answer.

Environment reasoning: `distTo` and `edgeTo` are two parallel heap arrays indexed by vertex, and `relax` mutates them in place. Nothing about the iteration order is stored in those arrays; the order lives entirely in the `Iterable<Integer>` returned by `top.order()`, which was fully materialized before the loop started. Compare with Dijkstra's, where a `MinPQ` object on the heap is mutated during the loop and determines the next vertex dynamically. Removing that mutable scheduling structure is the entire structural difference.

### Example 4: DAG longest paths by sign flipping

The lecture's DAG-LPT algorithm as code:

```java
public class AcyclicLP {
    private double[] distTo;
    private DirectedEdge[] edgeTo;

    public AcyclicLP(EdgeWeightedDigraph G, int s) {
        EdgeWeightedDigraph negated = new EdgeWeightedDigraph(G.V());
        for (DirectedEdge e : G.edges()) {
            negated.addEdge(new DirectedEdge(e.from(), e.to(), -e.weight()));
        }

        AcyclicSP sp = new AcyclicSP(negated, s);

        distTo = new double[G.V()];
        for (int v = 0; v < G.V(); v += 1) {
            distTo[v] = -sp.distTo(v);
        }
        edgeTo = sp.edgeToArray();
    }
}
```

Three steps, matching the slides exactly:

1. **Preprocess.** Build `G'`, a fresh graph with every edge weight negated. On the lecture's example, `B -> C` weight 6 becomes -6, `C -> F` weight 2 becomes -2, and so on. The vertex set and edge *directions* are unchanged, so `G'` is still a DAG and still has the same topological order as `G`.
2. **Solve.** Run `AcyclicSP` on `G'`. Because the weights are now mostly negative, the shortest path in `G'` is the one with the most negative total, which corresponds to the heaviest total in `G`. This is the step that *requires* DAG-SPT rather than Dijkstra's: Dijkstra's would be wrong on `G'` by construction.
3. **Postprocess.** Negate every entry of `distTo`. Do *not* touch `edgeTo`: negating weights does not change which edge was used, so `edgeTo` is already the correct LPT structure. On the lecture's example the result is `distTo = [A:0, B:6, C:12, D:2, E:13, F:14]`.

Why `distTo` needs flipping but `edgeTo` does not, stated carefully: `distTo[v]` is a numerical sum of the weights along the chosen path, so negating all the weights negates the sum, and you must negate it back. `edgeTo[v]` is a reference to a `DirectedEdge` object identifying *which* edge arrives at `v`. The choice of path is preserved under the transformation (shortest in `G'` means longest in `G`), so the identity of the arriving edge is already right.

Note that `edgeTo` in `G'` holds edges with negated weights. If you care about reporting the edge weights themselves, you need to look them up in `G`, not in the returned `edgeTo` entries. (extra context, the slides only state that `X.edgeTo` is already correct for path structure.)

Runtime: O(V + E), dominated by the single DAG-SPT call. The negation pass is O(E) and the distance flip is O(V).

### Example 5: reading the reduction diagram

The shape to memorize, from the slides:

```
            DAG-LPT
   G -----------------------> G'
       preprocess (flip signs)
                                 |
                                 | DAG-SPT
                                 v
   LPT of G <---------------- SPT of G'
       postprocess (flip distTo back)
```

The outer box is the problem you were asked to solve (DAG-LPT). The inner arrow is the solver you already had (DAG-SPT). "DAG-LPT reduces to DAG-SPT" names this direction: the problem being solved reduces to the problem whose solver you borrow. Getting this direction backwards is a common exam error.

The 3SAT example has the same shape with different labels: φ (a boolean formula) preprocesses into a graph `G`, you call an Independent-Set solver, and you postprocess an independent set of size `k` into a satisfying truth assignment. Same diagram, and this time the input is not a graph at all.

## Common Pitfalls

**Thinking a "loop" in a directed graph is a cycle.** `C -> D -> E` plus `F -> C` is not a cycle if the `E`/`F` edge points `F -> E` or `E -> F` in the wrong direction. Always check arrow directions before declaring a cycle. The lecture opened the topological sort section with exactly this trap.

**Using postorder instead of reverse postorder.** Postorder starts with the deepest, most-dependent vertex (`H` in the demo). It is the opposite of what you want. Always reverse.

**Using preorder and reversing it.** Preorder does not become a topological order when reversed. The postorder-then-reverse combination is specific; placing the `add` before the recursion instead of after breaks the algorithm.

**Clearing the marked array between DFS restarts.** If you reset marks, a vertex reachable from two indegree-0 roots will appear twice in the postorder list, and the resulting "ordering" is garbage. The slides emphasize "NOT clearing markings in between traversals."

**Assuming BFS level order is a topological sort.** It happens to work on some graphs but is not correct in general. The lecture's counterexample adds an edge so that BFS enqueues `F` alongside `D` and can emit `F` before `D`, violating `D -> F`.

**Assuming the topological ordering is unique.** Usually it is not. The demo graph has multiple valid orderings. The warmup graph happens to have exactly one. Both situations are normal, and an exam answer should accept any valid ordering unless uniqueness is the point of the question.

**Measuring distances from the previous vertex instead of from the source.** In the warmup, `distTo[E]` is 5 (via `A -> D -> E`, 2 + 3), not 3 (just the `D -> E` edge). The lecture flagged this explicitly because minimum spanning tree reasoning from the previous lecture encourages the wrong habit: MST cares about individual edge weights, shortest paths cares about cumulative distance from the source.

**Believing relaxation always updates.** Relaxing `D -> B` (weight 25) in Example 3 does nothing, because `distTo[B]` is already 1 and the alternative is 26. Relaxation is a conditional improvement, not an assignment.

**Using a priority queue in DAG-SPT.** There is none. The lecture makes this the key contrast with Dijkstra's. If your trace involves reordering pending vertices, you have written Dijkstra's, not DAG-SPT.

**Applying DAG-SPT to a graph with a cycle, or to an undirected graph.** The algorithm's correctness depends entirely on a valid topological order existing, which requires a DAG. On a cyclic graph, no topological order exists and the algorithm has nothing to iterate over.

**Concluding that DAG-SPT makes Dijkstra's obsolete.** It does not. They have different preconditions: DAG-SPT needs acyclic and directed but tolerates negative weights; Dijkstra's handles cycles and undirected graphs but needs nonnegative weights. Neither is strictly better.

**Thinking longest paths is "just shortest paths with max instead of min."** The greedy swap (Dijkstra's but take the biggest edge first) does not work, and the lecture assigns finding a counterexample as an exercise. General longest simple paths has no known efficient algorithm.

**Flipping `edgeTo` in the DAG-LPT reduction.** Only `distTo` needs un-negating. `edgeTo` records which edge was taken and is already correct.

**Forgetting to flip the signs back at all.** Then you have reported the *negated* longest-path distances, which will all be nonpositive and look obviously wrong.

**Stating the reduction direction backwards.** "DAG-LPT reduces to DAG-SPT" because the SPT solver is the subroutine being used. Not the other way around.

**Thinking reductions require graphs or sign flipping.** The 3SAT to Independent-Set reduction starts from a boolean formula, and the slides call out both points explicitly.

## Likely Exam Points

### 1. Compute a topological sort via DFS reverse postorder

This is the single most mechanical thing on this material, and the trace is easy to get wrong under time pressure. Expect to be given a small DAG, asked to run DFS from indegree-0 vertices in a specified tie-breaking order (often alphabetical), record postorder, and reverse it.

**Practice:** Given a DAG with edges `A -> B`, `A -> D`, `B -> E`, `C -> D`, `C -> F`, `D -> E`, `E -> H`, `F -> G`, give the topological ordering produced by DFS-with-restarts from indegree-0 vertices in alphabetical order, exploring neighbors in alphabetical order.

**Answer:** Postorder is `[H, E, B, D, A, G, F, C]`, so the ordering is `[C, F, G, A, D, B, E, H]`. Start at `A` (indegree 0), go `A -> B -> E -> H`, finish `H, E, B`, back up to `A` and take `A -> D`, `D`'s neighbor `E` is already marked so finish `D`, then finish `A`. Restart at `C`, skip the already-marked `D`, go `C -> F -> G`, finish `G, F, C`. Reverse.

### 2. Determine whether a topological ordering exists, or is unique

**Practice:** Does a graph with edges `A -> B`, `B -> E`, `E -> H`, `H -> A` have a topological ordering? Why or why not?

**Answer:** No. Those edges form a directed cycle `A -> B -> E -> H -> A`, so the graph is not a DAG. Whichever of those four vertices you place earliest in a proposed ordering, the edge entering it from within the cycle must point backwards, violating the ordering property. A topological ordering exists if and only if the graph is acyclic.

**Practice (uniqueness):** For the graph with edges `A -> B`, `A -> D`, `D -> B`, `B -> C`, `C -> E`, `D -> E`, `E -> F`, `C -> F`, how many distinct topological orderings exist?

**Answer:** Exactly one: `A, D, B, C, E, F`. At every step precisely one vertex has all incoming edges already satisfied: only `A` has indegree 0; `B` must wait for `D`; `E` must wait for `C`; and so on.

### 3. Explain why Dijkstra's fails on negative edges, with a concrete counterexample

**Practice:** Give a graph with a negative edge on which Dijkstra's produces an incorrect `distTo`, and explain which specific assumption is violated.

**Answer:** Use the lecture's graph: `A -> B` (1), `A -> D` (1), `D -> E` (1), `B -> C` (6), `C -> E` (-20), `E -> F` (1), `C -> F` (1). Dijkstra's visits `A` (0), `B` (1), `D` (1), `E` (2 via `D`), `F` (3 via `E`), `C` (7) in that order. When `C` is finally visited and `C -> E` is relaxed, `distTo[E]` improves to -13, but `F` was already finalized at 3 and is never reconsidered, so Dijkstra's reports `distTo[F] = 3` when the true answer is -12. The violated assumption: Dijkstra's requires that a vertex's distance be final at the moment it is popped, which holds only when all weights are nonnegative (any detour through more nonnegative edges can only add weight).

### 4. Trace DAG shortest paths

**Practice:** On the graph from question 3 with source `A`, trace DAG-SPT and give the final `distTo` and `edgeTo` tables. (Also include `D -> B` with weight 25.) Topological order: `A, D, B, C, E, F`.

**Answer:** `distTo` = A: 0, B: 1, C: 7, D: 1, E: -13, F: -12. `edgeTo` = A: none, B: `A -> B`, C: `B -> C`, D: `A -> D`, E: `C -> E`, F: `E -> F`. The key moments: `D -> B` fails to relax (26 is worse than 1), `C -> E` succeeds with 7 + (-20) = -13 because `E` has not yet been visited, and `E -> F` then improves `F` from 8 to -12.

### 5. State and justify the runtime of DAG-SPT

**Practice:** DAG-SPT relaxes all outgoing edges of every vertex. Why is the runtime O(V + E) rather than O(V · E)?

**Answer:** `E` is the total number of edges in the whole graph, not the number of edges per vertex. Summing outdegrees over all `V` vertices totals exactly `E`, so the relaxation work across the entire run is O(E). Add O(V) for visiting each vertex and O(V + E) for the initial topological sort, giving O(V + E) overall with Θ(V) space.

### 6. Compare DAG-SPT and Dijkstra's

**Practice:** DAG-SPT is O(V + E), beating Dijkstra's O(E log V), and it handles negative weights. Why does the course still teach Dijkstra's?

**Answer:** They solve different problems. Dijkstra's works on any weighted graph, directed or undirected, with or without cycles, provided all weights are nonnegative. DAG-SPT requires a directed acyclic graph, so it cannot be used when cycles are present or the graph is undirected, but it tolerates negative weights. Neither strictly dominates. (Dijkstra's extra log V factor comes from priority-queue updates: in the worst case each of the `E` edges triggers an update costing log V.)

### 7. The DAG longest paths reduction

**Practice:** Describe an O(V + E) algorithm for finding the longest paths tree on a DAG, and explain why it is correct.

**Answer:** Build `G'`, a copy of `G` with every edge weight negated. Run DAG-SPT on `G'`, obtaining result `X`. Negate every value in `X.distTo`; `X.edgeTo` needs no change. Correctness: DAG-SPT is valid on negative weights, and for any path, the sum of negated weights is the negation of the sum, so the path minimizing the total in `G'` is exactly the path maximizing it in `G`. Negating the resulting distances recovers the true longest-path lengths. Runtime: O(E) to negate, O(V + E) for DAG-SPT, O(V) to un-negate.

### 8. Identify and state a reduction

**Practice:** In the DAG-LPT algorithm above, which problem reduces to which, and what are the preprocessing and postprocessing steps?

**Answer:** DAG-LPT reduces to DAG-SPT, because a DAG-SPT subroutine is being used to solve DAG-LPT. Preprocessing: negate every edge weight to produce `G'`. Postprocessing: negate every entry of the returned `distTo`. Per Dasgupta, Papadimitriou, and Vazirani: "If any subroutine for task Q can be used to solve P, we say P reduces to Q," so here P is DAG-LPT and Q is DAG-SPT.

### 9. Why the general longest paths problem is hard

**Practice:** Why can we not simply run Dijkstra's with `max` substituted for `min` to solve longest paths on a general graph? What is the best known runtime?

**Answer:** The greedy exchange argument underlying Dijkstra's does not survive the swap: with a maximization objective, extending a path can always improve it, so there is no point at which a vertex's value can safely be finalized, and the simple-path requirement means you must also track which vertices the current path has already used. The best known algorithm is exponential, and no efficient algorithm is known. On a DAG specifically the problem becomes O(V + E) via the sign-flip reduction, because acyclicity removes the need to track revisited vertices.

### 10. (Optional / bonus) The 3SAT to Independent-Set reduction

**Practice:** In the reduction from 3SAT to Independent-Set, what graph is constructed from a formula φ, and what role does each kind of edge play?

**Answer:** For each clause of φ, create three vertices (one per literal) joined pairwise into a triangle. Additionally, add an edge between every literal vertex and every vertex for its negation. The triangles ensure an independent set picks at most one literal per clause (choosing which literal satisfies that clause). The literal-to-negation edges ensure the choices are consistent, since `x` and `!x` cannot both be true in 3SAT and therefore must not both be in the independent set. Setting `k` to the number of clauses, an independent set of size `k` yields a satisfying assignment. Note that 3SAT is not a graph problem, which illustrates that reductions need not preserve problem domain.

## Summary

- A **directed acyclic graph (DAG)** is a directed graph with no directed cycle. Edge direction matters: `C -> D -> E` with `E -> F` is not a cycle even though it looks like one on paper.
- DAGs model **dependency graphs** naturally, where `v -> w` means "v before w." Cycles in a dependency graph are logically impossible (no legal first task).
- A **topological ordering** places all vertices on a line with every edge pointing rightwards. It exists **if and only if** the graph is a DAG, and it is often not unique.
- **Topological sort algorithm:** DFS from every indegree-0 vertex, not clearing marks between restarts, record **postorder**, then **reverse** it. O(V + E) time, Θ(V) space. (A variant: restart from any arbitrary unmarked vertex instead of computing indegrees.)
- Postorder alone is the wrong answer; so is reversed preorder. The reversal is specifically of postorder.
- BFS level order is not a correct topological sort in general (Kahn's algorithm is the careful BFS version, not covered).
- **Dijkstra's fails on negative edges** because it finalizes a vertex's distance the moment it is popped. A later negative edge can beat that finalized value, and downstream vertices that were already finalized are never revisited. The lecture's graph leaves `distTo[F] = 3` instead of -12.
- **DAG shortest paths:** compute a topological order, then visit vertices in that order and relax all outgoing edges. O(V + E) time, Θ(V) space. Correct with negative weights because every vertex is visited only after all vertices that can reach it, so all available information about it has already been used.
- DAG-SPT has **no priority queue**. The visit order is a fixed list computed once.
- The runtime is O(V + E), not O(V · E), because `E` is the total edge count across the whole graph, and the outdegrees sum to `E`.
- DAG-SPT and Dijkstra's **do not dominate each other**: Dijkstra's handles cycles and undirected graphs but needs nonnegative weights; DAG-SPT needs a DAG but tolerates negative weights.
- **Longest simple paths** on a general graph has **no known efficient algorithm**; the best known is exponential. It is tied to the most important open problem in the field.
- **DAG longest paths** is O(V + E): negate all edge weights, run DAG-SPT, negate the resulting `distTo`. Leave `edgeTo` alone, since negating weights does not change which edge was taken.
- A **reduction** solves P by preprocessing its input, invoking a solver for Q, and postprocessing the output: "If any subroutine for task Q can be used to solve P, we say P reduces to Q." Here, **DAG-LPT reduces to DAG-SPT**.
- Reductions explain why longest paths is both obscure and enormously important: vast numbers of problems reduce to it and it reduces back to them, so solving any one solves all (NP-completeness, possibly revisited in the last week).
- Reductions are not limited to graphs or sign flips: **3SAT reduces to Independent-Set** by building a triangle per clause plus literal-to-negation edges, and 3SAT is not a graph problem at all.
- Reductions formalize what the course has done all along (percolation via disjoint sets, hyponyms via DFS/BFS): decompose a hard task into parts solvable with abstractions you already have.
