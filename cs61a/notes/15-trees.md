<!-- Wed, Sep 30, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 15: Trees

Trees are the second major recursive data structure in this course, and they exist to represent hierarchical relationships: things that have parts, which have their own parts, and so on down. A tree has a root label and a list of branches, where every branch is itself a tree, and a tree with zero branches is called a leaf. Because the definition is recursive, almost every function that processes a tree is recursive too, and it follows one of a small number of shapes: make a recursive call on each branch, then aggregate the results (count, sum, max), or walk downward from the root toward the leaves carrying information with you and act when you arrive. This lecture introduces the `Tree` data class, the `is_leaf` helper, the printed indented representation, how trees compare to linked lists, and then works through `count_leaves`, `largest_label`, `largest_path_sum`, and `above_root`, plus tree-building functions and the downward-accumulating pattern.

## Key Concepts

### Why trees exist

The purpose of a tree is to store a set of labels along with the relationships between those labels. The relationships are hierarchical: parts have subparts, and those have their own subparts. In computer science we draw trees with the root at the top, growing downward, which is upside down compared to a real tree but is the universal convention.

### Two vocabularies for the same structure

The lecture is explicit that there are two metaphors in use, and both describe the exact same object.

The **recursive description (wooden trees)**:
- A tree has a root label and a list of branches.
- Each branch is a tree.
- A tree with zero branches is called a leaf.
- A tree starts at the root.

The recursion is the key sentence: a tree has a branch that is itself a tree. Because each branch is a tree, each branch also has its own root label, its own branches, and its own leaves.

The **relative description (family trees)**:
- Each location in a tree is called a node.
- Each node has a label, which can be any value.
- One node can be the parent or the child of another.
- The top node is the root node.

Both vocabularies get used in the same breath, and you should be fluent in switching. The thing to internalize: labels are values, nodes are trees. A label does not have branches; a tree does. This confusion is the source of a specific bug discussed later.

### People say "node" when they mean "label"

A very common abbreviation in CS 61A and beyond is a phrase like "each parent is the sum of its children." The strictly correct version would be "the label of each parent node is equal to the sum of the labels of its children," but that is too long to say every time. So people refer to labels by their locations. Read such sentences charitably and know what they mean.

### Paths

A path goes from the root to a leaf (or, in some problems, from the root to any node). The defining property of a path is that it always connects a parent node to one of its children and continues on that way. You cannot jump sideways between branches.

### The Tree data class

```python
from __future__ import annotations
from dataclasses import dataclass, field

@dataclass
class Tree[T]:
    label: T
    branches: list[Tree[T]] = field(default_factory=list)  # branches defaults to []

def is_leaf(t: Tree) -> bool:
    return not t.branches
```

Things worth noticing:

- `T` is a **type variable** describing the type of the labels in the tree. It is optional. `Tree[int]` says the labels are all integers; plain `Tree` says the labels can be anything.
- `field(default_factory=list)` looks complicated but is just Python's way of saying "branches defaults to an empty list." This is why `Tree(4)` is legal and builds a leaf.
- `is_leaf` is a helper function (not a method) that checks whether the branch list is empty. It exists purely to make programs cleaner. `not []` is `True`, so `is_leaf(Tree(4))` is `True`.

(extra context) A mutable default like `[]` cannot be written directly as a dataclass default, because all instances would share one list; `default_factory=list` calls `list()` fresh for each instance. The lecture only says it means "defaults to empty list," which is the part you need.

### Constructing a tree

```python
t = Tree(2, [Tree(4), Tree(6, [Tree(8), Tree(10)])])
```

You supply a root label and a list of branches, each of which must be a tree. `Tree(4)` is a leaf. `Tree(6, [Tree(8), Tree(10)])` is a tree with two leaves. The whole thing is a `Tree[int]`, or just a `Tree`.

Trees are values like any other. For a tree `t` you can `return t`, call `f(t)`, build `[t]`, write `s = t`, and so on.

### The printed representation

Trees have a `__str__` method (which calls `format_tree`) controlling how they display. The rule: the root label appears first, then each branch below it, indented one more level. Labels at the same depth get the same indentation, so **depth in the tree determines indentation in the printout**.

```python
>>> print(t)
2
  4
  6
    8
    10
```

And nesting the same tree twice:

```python
>>> print(Tree(1, [t, t]))
1
  2
    4
    6
      8
      10
  2
    4
    6
      8
      10
```

Both `2`s have the same depth, so both get the same indentation. The lecture is candid that it takes practice to go from the printed form back to the structure. The trick it suggests: horizontally in the drawing you read 1, then 2, then 4, 6; vertically in the printout you read 1, some 2s, then 4, 6. Same structure, different display.

Here is `format_tree` itself, which is a tree-recursive function in its own right:

```python
def format_tree(t: Tree, indent='') -> str:
    "Format a tree with each branch indented below its label."
    assert isinstance(t, Tree), f'{t!r} is not a Tree'
    assert isinstance(t.branches, list), f'branches of {t!r} is not a list'
    string = indent + str(t.label)
    for b in t.branches:
        string += '\n' + format_tree(b, indent + '  ')
    return string
```

Note that the recursive call adds two spaces to `indent`, which is exactly how depth becomes indentation.

### The tree abstraction: what you are allowed to do

For a tree `t`, you can only:

- Get the label for the root of the tree: `t.label`
- Get the list of branches for the tree: `t.branches`
- Get the branch at index `i`, which is a tree: `t.branches[i]`
- Determine whether the tree is a leaf: `is_leaf(t)`
- Treat `t` as a value: `return t`, `f(t)`, `[t]`, `s = t`, etc.

That is the whole interface. There is no `t.leaves`, no `t.parent`, no `t.children`, no `t.size`. If you need any of those, you write a function.

### Trees versus linked lists

```python
type LinkedList[T] = Link[T] | tuple[()]

@dataclass
class Link[T]:
    first: T
    rest: LinkedList[T] = ()   # rest defaults to ()
```

What's similar: a tree in which each node has at most one child is just like a linked list. Both are recursive: `rest` is a linked list, each branch is a tree.

What's different:
- Trees can have many branches, while a linked list has exactly one `rest`.
- Trees **cannot be empty**. This is a course-specific decision. A linked list can be empty (the `()` case), and that empty case is the natural base case for linked list recursion. Trees have no empty value, so the base case for tree recursion is usually the **leaf**, not emptiness.

That last point matters a lot for how you write base cases. You never write `if t is None` or `if t == ()` for a tree in this course.

### Writing recursive tree functions: the three questions

Before writing any code, answer these:

1. **What small initial choice can I make?** For trees, this is often: which branch to explore?
2. **What recursive call for each option?** Usually `f(b)` for each `b in t.branches`.
3. **How can I combine the results of those recursive calls?** Which breaks into three sub-questions:
   - What type of values do they return?
   - What do the possible return values mean?
   - How can I use those return values to complete my implementation? For example: look to see if any option evaluated to true, or add up the results from each option.

The lecture repeats this framework for every example, and that repetition is the point. The hard part of tree recursion is almost never the recursive call; it is knowing precisely what the recursive call gives you back so you know how to combine the pieces.

### The two major patterns

**Pattern A: recurse on all branches, then aggregate.** This is `count_leaves`, `largest_label`, `largest_path_sum`, `sum_leaves`. You get one value back per branch, you build a list of them, and you combine with `sum`, `max`, or similar.

**Pattern B: walk downward from root to leaves, carrying accumulated information, and act when you get there.** This is `above_root` (uses the root's label while descending) and `print_sums` (carries the running path sum down).

The lecture explicitly draws the analogy to the two versions of factorial. In one version you compute the factorial of `n-1` and multiply by `n` at the very end, so `n` is the *last* number folded in. In the other version you start with 1, pass an accumulator down through a two-argument recursion, and `n` is the *first* number folded in. Different order of operations, same answer. Both strategies apply to trees.

### Building trees is also recursive

A function that creates a tree from another tree is usually recursive as well. The structure is: build a `Tree(new_label, new_branches)` where `new_branches` is a list comprehension of recursive calls.

## Definitions

- **Tree**: a data structure with a root label and a list of branches, where each branch is itself a tree. In this course, a tree is never empty.
- **Label**: the value stored at a node. Can be any value (int, str, anything). The label itself has no branches.
- **Branch**: one of the subtrees of a tree, stored in `t.branches`. Each branch is a tree, so it has its own label and branches.
- **Leaf**: a tree with zero branches. A leaf is still a tree. Tested with `is_leaf(t)`, i.e. `not t.branches`.
- **Node**: a location in a tree. Every node is a tree (the node together with everything below it). The nodes of `t` are `t` itself plus all nodes of all branches of `t`.
- **Root**: the top of a tree. The root of the whole tree is the root node; each branch also has its own root, because each branch is a tree.
- **Root label**: `t.label`, the label at the top of tree `t`.
- **Parent / child**: if node `b` is a branch of node `t`, then `t` is the parent of `b` and `b` is a child of `t`.
- **Path**: a sequence of nodes starting at the root where each next node is a child of the previous. Usually root-to-leaf, but some problems use root-to-any-node.
- **Depth**: how far a label is from the root; determines indentation in the printed representation. (The lecture uses the word "depth" in exactly this sense.)
- **`is_leaf(t)`**: helper function returning `not t.branches`, i.e. `True` when the branch list is empty.
- **Type variable (`T` in `Tree[T]`)**: an optional annotation describing the type of the labels in the tree. `Tree[int]` means integer labels; plain `Tree` means any labels.
- **`field(default_factory=list)`**: dataclass syntax meaning "this field defaults to an empty list," which is what makes `Tree(4)` construct a leaf.
- **Tree recursion**: recursion in which a function makes more than one recursive call per frame, which is what happens naturally when you recurse on every branch.

## Worked Examples

### The example tree used throughout the code file

```python
t = Tree(1,
         [Tree(9, [
             Tree(2, [
                 Tree(5, [
                     Tree(6),
                     Tree(7)]),
                 Tree(8),
                 Tree(3)]),
             Tree(4)])])
```

Read this carefully, because several doctests depend on it. Printed, it is:

```
1
  9
    2
      5
        6
        7
      8
      3
    4
```

Structurally: the root label is 1 with a single branch rooted at 9. The 9 node has two branches: one rooted at 2 and one leaf 4. The 2 node has three branches: a tree rooted at 5, a leaf 8, and a leaf 3. The 5 node has two leaves, 6 and 7.

The leaves are 6, 7, 8, 3, and 4, which is five leaves.

### `print_leaves`

```python
def print_leaves(t):
    """print the leaf labels of Tree t."""
    if is_leaf(t):
        print(t.label)
    else:
        for b in t.branches:
            print_leaves(b)
```

This is Pattern B in its simplest form. There is nothing to combine, because nothing is returned: the base case *does* the work (prints), and the recursive case just forwards the job to each branch. Note the `for` loop rather than a list comprehension, because we do not need the results.

Environment reasoning: calling `print_leaves(t)` on the tree above opens a frame where `t` is bound to the whole tree. `is_leaf(t)` is false (it has one branch), so we loop over `t.branches`, which has one element, and call `print_leaves` on the tree rooted at 9. That opens another frame. Each frame either prints once and returns, or opens one frame per branch. Because frames are opened left to right and each finishes before the next starts, the leaves print in left-to-right order: 6, 7, 8, 3, 4.

### `count_leaves`

The three questions, answered as in lecture:

- Small, initial choice: which branch's leaves to count?
- Recursive call for each option: for each branch `b`, `count_leaves(b)`, which gives the number of leaves on branch `b`.
- Combine results: add up all of the counts.

```python
def count_leaves(t: Tree) -> int:
    """The number of leaves in tree.

    >>> count_leaves(t)
    5
    """
    if is_leaf(t):
        return 1
    else:
        return sum([count_leaves(b) for b in t.branches])
```

The lecture is explicit that **processing a leaf is often the base case**. How many leaves are in a tree that is just a leaf? One.

In the recursive case, each `count_leaves(b)` returns an integer, so the list comprehension produces a list of integers, one per branch, and `sum` collapses it to the total. Tracing on the big tree: the 5 node returns `sum([1, 1]) = 2`; the 2 node returns `sum([2, 1, 1]) = 4`; the 9 node returns `sum([4, 1]) = 5`; the root returns `sum([5]) = 5`.

Why no explicit base case for "no branches" beyond `is_leaf`? Because there is no empty tree. The leaf case *is* the bottom.

### `sum_leaves` (developed live in lecture)

```python
def sum_leaves(t: Tree):
    """Sum the leaf labels of Tree t."""
    if is_leaf(t):
        return t.label
    else:
        return sum([sum_leaves(b) for b in t.branches])
```

If you are already looking at a leaf, the sum of all leaf labels in that tree is just the tree's own label, because that is the only leaf. Otherwise every leaf lives on some branch, so sum the per-branch sums.

The lecture pairs this with a Fibonacci tree, built live:

```python
def fib_tree(n: int) -> Tree[int]:
    if n == 0 or n == 1:
        return Tree(n)
    else:
        left = fib_tree(n - 2)
        right = fib_tree(n - 1)
        return Tree(left.label + right.label, [left, right])
```

`fib_tree(5)` has root label 5 (the fifth Fibonacci number). `fib_tree(5).branches[0]` is `fib_tree(3)`, because the left branch is `n - 2`. The lecture then observes a fact: for a Fibonacci tree, the sum of the leaf labels equals the root label. `sum_leaves(fib_tree(5))` is 5; `sum_leaves(fib_tree(6))` is 8. This holds for Fibonacci trees specifically because the tree is built so each parent label is the sum of its two children's labels, and it does not hold for trees in general.

Accessing a nested value: to reach a specific label deep in `fib_tree(5)`, you chain branch indices and then take a label, for example `fib_tree(5).branches[1].label` is 3 (the right branch is `fib_tree(4)`, whose root label is 3), and going one level further, `fib_tree(5).branches[1].branches[1].label` is 2 (`fib_tree(3)`'s root label).

### `largest_label`

The three questions:

- Small, initial choice: which branch to look for the largest label on?
- Recursive call for each option: for each branch `b`, `largest_label(b)`, which returns a label that is the largest one from branch `b`.
- Combine results: return the largest of those and the root label.

```python
def largest_label(t: Tree):
    """Return the largest label in tree t.

    >>> largest_label(Tree(6, [Tree(4), Tree(5), Tree(3)]))
    6
    """
    if is_leaf(t):
        return t.label
    else:
        return max([largest_label(b) for b in t.branches] + [t.label])
```

The slide asks: **what would happen if we got rid of the base case return `t.label`?** The answer is that the function would break, because the recursive case's list comprehension iterates over `t.branches`, which is empty for a leaf, giving `max([] + [t.label])`, which is actually still `t.label`. So in this specific formulation the base case is redundant, since `[t.label]` is appended unconditionally. The point of the question on the slide is to make you check whether your recursive case already handles leaves. Keeping the base case makes the intent explicit and is the safer habit.

Note the `+ [t.label]`: without it the function would ignore the root of every subtree and return only the largest *leaf* label. The doctest `largest_label(Tree(6, [Tree(4), Tree(5), Tree(3)]))` returning 6 is exactly the test that catches this: 6 is the root, not a leaf, and it is the answer.

The lecture also walks a **largest leaf label** variant (the maximum over leaves only, where the root does not count). On a tree with root 100, an inner 90, and leaves 7, 8, 6, 9, 2, the answer is 9: 100 and 90 do not count because they are not leaves. The correct recursive case there is a list comprehension of recursive calls over the branches, then `max` of that list, with no `+ [t.label]`.

The lecture explicitly rejects several wrong options for that variant, and each wrong answer teaches something:

- `max([b for b in t.branches])` just gives you the branches (trees), never computing the largest leaf label in each.
- `max([largest_leaf(t) for b in t.branches])` recurses on `t`, not `b`. This is not a smaller version of the same problem, so it never advances toward the leaves and runs forever.
- `max([largest_leaf(b.branches) for b in t.branches])` or anything that takes `.branches` of a returned label: when you call `largest_leaf` you get back a **label**, not a tree, and labels have no branches. Only trees have branches. Getting the branches of branches is what recursion does for you; you do not do it by hand.
- Anything mentioning `t.leaves`: there is no such attribute.

### `largest_path_sum`

The three questions:

- Small, initial choice: which branch contains the rest of the largest path sum?
- Recursive call for each option: for each branch `b`, `largest_path_sum(b)`, which returns the largest root-to-leaf sum within branch `b`.
- Combine results: the best path through `t` is `t.label` plus the best path through whichever branch is best.

```python
def largest_path_sum(t: Tree):
    """Return the largest sum of the labels along a path from the root to a leaf.

    >>> largest_path_sum(t)
    24
    """
    if is_leaf(t):
        return t.label
    else:
        return t.label + max([largest_path_sum(b) for b in t.branches])
```

Here the base case is genuinely required, because the recursive case would compute `max([])` on a leaf, and `max` of an empty list raises a `ValueError`.

Tracing the doctest on the big tree: at the 5 node, `max([6, 7]) = 7`, so it returns `5 + 7 = 12`. At the 2 node, the branch results are 12 (through 5), 8, and 3, so `max` is 12 and it returns `2 + 12 = 14`. At the 9 node, branch results are 14 and 4, so it returns `9 + 14 = 23`. At the root, the only branch result is 23, so it returns `1 + 23 = 24`. The winning path is 1, 9, 2, 5, 7.

Compare this to `largest_label`: there we combined with `max(... + [t.label])`, here with `t.label + max(...)`. The structural difference is that a path sum must include the root of the current subtree, so the root's contribution is added rather than being one of the candidates.

### `above_root`

The three questions:

- Small, initial choice: which branch to look at for labels to print?
- Recursive call for each option: for each branch `b`, `process(b)`.
- Combine results: no need to combine any recursive return value. We do need to print this label, if it is larger than the root.

```python
def above_root(t):
    """Print all the labels of t that are larger than the root label."""
    def process(u):
        if u.label > t.label:
            print(u.label)
        for b in u.branches:
            process(b)
    process(t)
```

This is the key new idea of this example: the comparison needs the **original root's label**, but a plain recursive function on a subtree has no way to see the top of the tree. The fix is a **nested helper function**. `process` is defined inside `above_root`, so by lexical scoping its parent frame is `above_root`'s frame, where `t` is bound to the whole original tree. As `process` recurses, `u` changes to each subtree in turn, but the name `t` inside `process` always resolves, through the enclosing environment, to the one original tree.

Environment reasoning: calling `above_root(t)` creates a frame for `above_root` with `t` bound to the tree, and defines `process` whose parent is that frame. Each call to `process(u)` creates a new frame whose parent is the `above_root` frame (not the calling `process` frame). So every `process` frame looks up `t` in the same place and gets the same value, while `u` is local and different in each frame. This is exactly the higher-order-function/environment machinery from earlier lectures applied to a tree problem.

Note also that `process` returns `None`; the work is the printing. And note the root itself is never printed, because `t.label > t.label` is false.

### Building trees: increment the leaves, or increment everything

From the lecture, a function that returns a tree with the same structure as `t` but with every **leaf** label increased by one:

```python
def increment_leaves(t: Tree) -> Tree:
    """Return a tree like t but with leaf labels increased by 1."""
    if is_leaf(t):
        return Tree(t.label + 1)
    else:
        return Tree(t.label, [increment_leaves(b) for b in t.branches])
```

Read the recursive case as: the result is a tree with the same label as `t`, but with different branches, because those branches have had their leaves incremented.

Now the version that increments **every** label, which the lecture points out is easier because you no longer need to distinguish leaves from non-leaves:

```python
def increment(t: Tree) -> Tree:
    """Return a tree like t but with every label increased by 1."""
    return Tree(t.label + 1, [increment(b) for b in t.branches])
```

Where is the base case? There is no explicit one. If `t.branches` is empty, the list comprehension produces `[]` and no recursive call is ever made, so the recursion terminates on its own. This is a genuinely important pattern: a list comprehension over `t.branches` is self-terminating at leaves. (The same reason `largest_label`'s base case turned out to be redundant.)

### The downward-accumulating pattern: `print_sums`

The second pattern, where you carry information down and act at the leaf. The lecture's example prints the concatenation (or sum) of labels along each root-to-leaf path:

```python
def print_sums(t, so_far):
    """Print the sum of labels along each path from the root of t to a leaf."""
    so_far = so_far + t.label
    if is_leaf(t):
        print(so_far)
    else:
        for b in t.branches:
            print_sums(b, so_far)
```

The lecture's demonstration tree has labels `'h'`, `'a'`, `'s'`, `'t'`, `'e'` arranged so that the root-to-leaf paths spell `has`, `hat`, and `he`, and `+` concatenates strings rather than adding numbers. The structure is:

```python
Tree('h', [Tree('a', [Tree('s'), Tree('t')]), Tree('e')])
```

Calling `print_sums(t, '')` starts with the empty string. At the root, `so_far` becomes `'h'`. The root is not a leaf, so we do not print `'h'`; instead we recurse into each branch **passing `'h'` along**. At the `'a'` node, `so_far` becomes `'ha'`. At the `'s'` leaf, `so_far` becomes `'has'` and we print it. Back up and over to `'t'`: `so_far` is `'hat'`, printed. Then `'e'`: `so_far` is `'he'`, printed.

The lecture's framing of why this works: "the sums we want to print for the entire tree are the sums for each branch after I've already added the root label." The root label is folded in *before* the recursive call, which is the mirror image of the aggregate pattern where the root's contribution is folded in *after* the calls return.

### Counting paths with a given sum

This example combines both ideas. Return the number of paths from the root to **any node** (not just a leaf) in tree `t` whose labels sum to a given value.

The lecture's examples use a tree whose structure makes the following true:
- Paths summing to 7: two of them (one is `3 + 1 + 2 + 1`).
- Paths summing to 4: two of them (`3 + 1` and `3 + 1` down different branches). Note these do not reach leaves, which is why the count must consider every node visited, not only leaves.
- Paths summing to 3: two of them, the path from the root to itself (just `3`), and a path that goes 3, then 4, then back down to a -3 style label summing to 3 again. The point is that the zero-length path consisting of the root alone counts.

```python
def count_paths(t, total):
    """Return the number of paths from the root of t to any node
    whose labels sum to total."""
    if t.label == total:
        found = 1
    else:
        found = 0
    return found + sum([count_paths(b, total - t.label) for b in t.branches])
```

The reasoning, in the lecture's own terms:

- If you have reached the desired total using only the current label, you have found one path.
- Otherwise you have not found it *yet*, but you may find it by exploring the branches.
- To explore a branch, you must factor the current label into the sum, which means the total you are now looking for is `total - t.label`. That is the amount that needs to be found within the branch in order to produce the desired total for the whole tree.
- `count_paths` returns a count, so each branch gives you a number, and you add them all up for the total across all branches.

Two subtleties: the `found` contribution and the branch contributions are **added**, not chosen between, because a path that ends here and a longer path through a branch can both be valid. And notice the hybrid nature: `total` accumulates downward (Pattern B) while the counts aggregate upward (Pattern A).

### Min practice (the closing exercise)

Given:

```python
xs = list(range(-10, 11))
```

Write an expression that evaluates to the `x` in `xs` for which `x*x - 2*x + 1` is smallest. The lecture shows:

```python
>>> xs
[-10, -9, -8, -7, -6, -5, -4, -3, -2, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
>>> [x*x - 2*x + 1 for x in xs]
[121, 100, 81, 64, 49, 36, 25, 16, 9, 4, 1, 0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
>>> ... some expression involving min ...
1
```

The smallest value of the expression is 0, which occurs at `x = 1`, so the answer is 1. The expression is:

```python
min(xs, key=lambda x: x*x - 2*x + 1)
```

`min` with a `key` function returns the element of the sequence that minimizes the key, not the minimum key value itself. This is why the answer is 1 (the `x`) rather than 0 (the smallest `x*x - 2*x + 1`).

This matters for trees because `max`/`min` with a `key` is often how you pick the *best branch* rather than the *best value*. (extra context) For example, `max(t.branches, key=largest_path_sum)` gives you the branch containing the heaviest path, which you would need if a problem asked you to return the path itself and not just its sum.

## Common Pitfalls

- **Calling `.branches` on a label.** Only trees have branches. `largest_label(b)` returns a *label*, so `largest_label(b).branches` is an error. The lecture calls this out directly: "Only a tree has branches, not its labels."
- **Recursing on `t` instead of `b`.** `[f(t) for b in t.branches]` is not a smaller version of the same problem. It never advances toward the leaves, so it recurses forever. Always check that the argument to your recursive call is a *branch*.
- **Inventing attributes.** There is no `t.leaves`, no `t.parent`, no `t.children`, no `t.size`. The abstraction is `t.label`, `t.branches`, `t.branches[i]`, and `is_leaf(t)`.
- **Forgetting `+ [t.label]` in `largest_label`.** Without it you compute the largest *leaf* label, not the largest label. The doctest with root 6 and leaves 4, 5, 3 exists to catch this.
- **`max([])` on a leaf.** In `largest_path_sum`, dropping the base case makes the recursive case evaluate `max([])` on a leaf, which raises `ValueError`. Compare `largest_label`, where the appended `[t.label]` keeps the list nonempty and makes the base case redundant.
- **Looking for an empty-tree base case.** Trees cannot be empty in this course, so `if t is None` or `if t == ()` is wrong. The base case is the leaf, or there is no explicit base case at all (because a comprehension over an empty `t.branches` makes zero recursive calls).
- **Using `t.branches` instead of `u.branches` in a nested helper.** In `above_root`, the loop must be `for b in u.branches`, because `u` is the subtree being processed. Using `t.branches` would loop over the original root's branches at every level, which is wrong. Conversely, the comparison must be against `t.label`, the original root. Mixing those up is the single most common bug in this style of problem.
- **Returning inside a loop that should keep going.** In `print_leaves` and `above_root` the recursive calls are statements in a `for` loop, not returns. Adding `return process(b)` would stop after the first branch.
- **Misreading the printed representation.** Indentation encodes depth, not order. Two labels at the same indentation are siblings-in-depth, which does not necessarily mean they share a parent. You have to count indentation levels carefully.
- **Confusing `min(xs, key=f)` with `min([f(x) for x in xs])`.** The first returns the input element; the second returns the key value. The practice problem's answer is 1, not 0.
- **Assuming facts that only hold for a special tree.** The sum of the leaves equals the root label for *Fibonacci trees*, because of how they are constructed. The lecture says explicitly that this does not apply to every tree.

## Likely Exam Points

### 1. Reading and writing the printed representation

Indentation equals depth. You will be given a printout and asked for the structure, or given a construction and asked for the printout.

**Practice:** What does `print(Tree(1, [Tree(2, [Tree(3)]), Tree(4)]))` display?

**Answer:**
```
1
  2
    3
  4
```
The 2 and the 4 are both branches of the root, so both are indented two spaces. The 3 is a branch of the 2, so it is indented four spaces.

### 2. Filling in a blank in an aggregate-pattern function

The classic exam shape: a skeleton with `if is_leaf(t): return ____` and a recursive case with one or two blanks.

**Practice:** Fill in the blanks so `count_nodes` returns the total number of nodes in `t`.

```python
def count_nodes(t):
    if is_leaf(t):
        return ______
    else:
        return ______ + sum([______ for b in t.branches])
```

**Answer:** `1`, then `1`, then `count_nodes(b)`. Every node contributes 1 for itself plus the counts of its branches. (The base case is redundant here, since `sum([])` is 0, so `1 + sum([])` already equals 1 at a leaf.)

### 3. The `largest_label` / `largest_path_sum` distinction

Knowing when the root's label is a *candidate* for `max` versus a *mandatory addend*.

**Practice:** On `Tree(1, [Tree(10, [Tree(2)]), Tree(5)])`, what do `largest_label` and `largest_path_sum` return?

**Answer:** `largest_label` returns 10, the biggest label anywhere. `largest_path_sum` returns 13: the paths are 1, 10, 2 (sum 13) and 1, 5 (sum 6), so the maximum is 13. Note 10 is not on the best path as a standalone value; it is part of a sum.

### 4. Nested helper functions to access the original root

The `above_root` pattern, where a comparison needs the top of the tree.

**Practice:** Why does `above_root` use a nested `process(u)` rather than just recursing with `above_root(b)`?

**Answer:** Because the comparison is against the *original* root label. If you recursed with `above_root(b)`, then inside that call `t` would be bound to the branch, and the comparison would be against the branch's own label, which is a different question. The nested `process` has the `above_root` frame as its parent environment, so the name `t` inside `process` always resolves to the one original tree, while `u` changes on each recursive call.

### 5. Accumulating a value downward

The `print_sums` / `count_paths` pattern, where a second parameter carries information from the root toward the leaves.

**Practice:** Complete `print_depths` so it prints each leaf label together with its depth (root at depth 0).

```python
def print_depths(t, d=0):
    if is_leaf(t):
        print(t.label, ______)
    else:
        for b in t.branches:
            print_depths(b, ______)
```

**Answer:** `d`, then `d + 1`. The depth is accumulated downward in the parameter, and the action happens at the leaf, which is the same shape as `print_sums`.

### 6. Why a base case is or is not needed

**Practice:** `increment` has no `if is_leaf(t)` clause. Why does it terminate?

```python
def increment(t):
    return Tree(t.label + 1, [increment(b) for b in t.branches])
```

**Answer:** At a leaf, `t.branches` is `[]`, so the list comprehension makes zero recursive calls and evaluates to `[]`. The function returns `Tree(t.label + 1, [])`, a leaf, and the recursion bottoms out. A comprehension over `t.branches` is self-terminating at leaves. The same reasoning explains why `largest_label`'s base case is redundant, while `largest_path_sum`'s is not (`max([])` raises an error, whereas `sum([])` and `[... for b in []]` do not).

### 7. Spotting a broken recursive call

**Practice:** What is wrong with each of these as a recursive case for "the largest leaf label in `t`"?

```python
# (a)
return max([largest_leaf(t) for b in t.branches])
# (b)
return max([largest_leaf(b).branches for b in t.branches])
# (c)
return max([b for b in t.branches])
```

**Answer:** (a) recurses on `t`, not `b`, so it never gets closer to a leaf and runs forever. (b) calls `.branches` on the *label* returned by the recursive call; labels have no branches, only trees do. (c) never recurses at all, so it tries to take the max of a list of trees rather than computing anything about leaves.

### 8. Trees versus linked lists

**Practice:** Give one way trees and linked lists are alike and two ways they differ.

**Answer:** Alike: both are recursive, and a tree in which every node has at most one child is essentially a linked list. Different: a tree can have many branches while a linked list has exactly one `rest`; and in this course a tree cannot be empty (so the base case is the leaf), while a linked list can be empty (so emptiness is the natural base case).

### 9. `min` / `max` with a `key`

**Practice:** For `xs = list(range(-10, 11))`, write an expression evaluating to the `x` minimizing `x*x - 2*x + 1`, and say what it evaluates to.

**Answer:** `min(xs, key=lambda x: x*x - 2*x + 1)`, which evaluates to `1`. The expression factors as `(x - 1)**2`, minimized at `x = 1` where it equals 0. `min` with a key returns the *element*, not the key value, so the answer is 1 and not 0.

## Summary

- A tree has a **root label** and a **list of branches**; each branch is itself a tree; a tree with zero branches is a **leaf**. Trees in this course are never empty.
- Two vocabularies describe the same object: recursive (root label, branches, leaf) and relative (node, label, parent, child, root node). People say "node" when they mean "label"; read charitably.
- The data class is `Tree[T]` with fields `label: T` and `branches: list[Tree[T]] = field(default_factory=list)`, plus the helper `is_leaf(t): return not t.branches`. `T` is an optional type variable for the label type.
- The abstraction permits exactly: `t.label`, `t.branches`, `t.branches[i]`, `is_leaf(t)`, and treating `t` as a value. Nothing else exists (no `t.leaves`, no `t.parent`).
- The printed representation puts the root first and indents each branch one more level, so **depth determines indentation**.
- Versus linked lists: a one-child-per-node tree is basically a linked list; trees differ in having many branches and in not having an empty case.
- Before coding, answer three questions: what small initial choice (usually which branch), what recursive call per option, and how to combine the results (what type they are, what they mean, how to use them).
- **Pattern A, aggregate upward:** recurse on every branch, build a list of results, combine with `sum` or `max`. Seen in `count_leaves`, `sum_leaves`, `largest_label`, `largest_path_sum`.
- **Pattern B, accumulate downward:** carry information from the root toward the leaves in an extra parameter (or via a nested helper closing over the root) and act when you arrive. Seen in `print_leaves`, `above_root`, `print_sums`. Mirrors the two versions of factorial.
- `count_leaves`: base case returns 1, recursive case returns `sum([count_leaves(b) for b in t.branches])`.
- `largest_label`: `max([largest_label(b) for b in t.branches] + [t.label])`. The `+ [t.label]` makes the root a candidate and also makes the base case redundant.
- `largest_path_sum`: `t.label + max([largest_path_sum(b) for b in t.branches])`. The root is always added, and the base case is required because `max([])` errors.
- `above_root` uses a nested `process(u)` so that `t` (the original root) stays visible through the enclosing environment while `u` walks the tree.
- Tree-building functions are recursive too: `Tree(new_label, [f(b) for b in t.branches])`. A comprehension over `t.branches` terminates by itself at leaves, so an explicit base case is sometimes unnecessary.
- `count_paths` is a hybrid: the target total decreases downward (`total - t.label`) while counts add upward, and a path ending at the current node plus paths through branches are summed, not chosen between.
- `min(xs, key=f)` returns the *element* that minimizes `f`, not the minimum value of `f`. For `x*x - 2*x + 1` over `range(-10, 11)` the answer is `1`.
