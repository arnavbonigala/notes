<!-- Wed, Sep 30, 2026 | sources: slides + code (no transcript available) -->
# Lecture 15: Trees

A tree is the course's first branching recursive data structure: a value (the **label**) together with a list of **branches**, each of which is itself a tree. This lecture introduces two complementary vocabularies for talking about trees (the recursive "wooden tree" view: root label plus branches; and the relative "family tree" view: nodes, parents, children, paths), defines the `Tree` data class used for the rest of the course, and then establishes a standard recipe for writing tree-processing functions: make one small choice (which branch to explore), recurse once per branch, and combine the results. Every example in the lecture (`count_leaves`, `largest_label`, `largest_path_sum`, `above_root`) is an instance of that recipe, and the only thing that changes between them is how the per-branch results get combined (`sum`, `max`, `t.label + max(...)`, or no combination at all when you are printing rather than returning). The lecture closes with a short `min` exercise that sets up the "pick the element that minimizes something" pattern.

---

## Key Concepts

### Two ways to describe the same structure

The slides deliberately give two descriptions, because problems are phrased in both languages.

**Recursive description (wooden trees):**
- A tree has a **root label** and a **list of branches**.
- Each branch is itself a tree.
- A tree with zero branches is a **leaf**.
- A tree starts at the root.

**Relative description (family trees):**
- Each location in a tree is a **node**.
- Each node has a **label**, which can be any value.
- One node can be the **parent** or **child** of another.
- The top node is the **root node**.

These are the same structure seen from different angles. The recursive view is what you write code with (a branch *is* a tree, so you can hand it straight to a recursive call). The relative view is what problem statements use: "each parent is the sum of its children" is really a statement about labels at locations, not about separate objects. The slides point this out explicitly: people routinely refer to labels by their locations.

A key consequence of the recursive view: **a branch is a tree**, and **a leaf is a tree**. There is no separate "subtree" type and no separate "leaf" type. In the slide diagram, the node labeled `1` with children `0` and `1` is simultaneously a branch of the whole tree and a perfectly good tree in its own right. This is what makes recursion work: the recursive call `f(b)` on a branch `b` is the *same kind of problem* as the original.

### The tree abstraction: what you are allowed to do

For a tree `t`, the lecture says you may only:

- Get the root label: `t.label`
- Get the list of branches: `t.branches`
- Get the branch at index `i`, which is a tree: `t.branches[i]`
- Determine whether it is a leaf: `is_leaf(t)`
- Treat `t` as a value: `return t`, `f(t)`, `[t]`, `s = t`, and so on

That last bullet matters more than it looks. A `Tree` is an ordinary Python object, so it can be stored in lists, passed to functions, returned, and aliased. `print(Tree(1, [t, t]))` in the slides builds a tree whose two branches are *the same object* `t`, and it prints the identical subtree twice.

### The `Tree` data class

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

Details worth understanding rather than memorizing:

- `Tree[T]` declares a **type variable** `T` that describes the type of the labels. `Tree(2, [Tree(4), Tree(6, [...])])` is a `Tree[int]`, or just "a Tree".
- `branches` defaults to the empty list, so `Tree(4)` is a leaf and you do not have to write `Tree(4, [])`.
- The default is written `field(default_factory=list)`, not `= []`. A single shared list as a default would be the same list object for every `Tree` ever constructed, which is exactly the mutable-default bug. `default_factory=list` calls `list()` fresh for each new tree. (Dataclasses actually refuse a mutable default outright, so you get an error rather than a silent bug.) (extra context)
- `is_leaf` is `not t.branches`, relying on the empty list being falsy. A leaf is a tree with zero branches, not a tree with `None` branches.
- `from __future__ import annotations` is what lets the annotation `list[Tree[T]]` refer to `Tree` inside the definition of `Tree` itself. (extra context)

### Printing trees

`Tree.__str__` calls `format_tree`, which indents each branch two spaces further than its label:

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

So for `t = Tree(2, [Tree(4), Tree(6, [Tree(8), Tree(10)])])`:

```
2
  4
  6
    8
    10
```

Reading this backwards is a common exam skill: indentation depth is depth in the tree, and consecutive lines at the same indentation under a shared parent are siblings.

Labels do not have to be numbers. The code's expression-tree example mixes ints and strings:

```python
expr = Tree(10, [Tree(6, [Tree(2), Tree('*'), Tree(3)]), Tree('+'), Tree(4)])
```

which prints

```
10
  6
    2
    *
    3
  +
  4
```

### Trees vs linked lists

The lecture puts the two data classes side by side:

```python
type LinkedList[T] = Link[T] | tuple[()]

@dataclass
class Link[T]:
    first: T
    rest: LinkedList[T] = ()   # rest defaults to ()
```

**What's similar:** a tree in which each node has at most one child is just like a linked list. `label`/`branches` plays the role of `first`/`rest`, and recursion on `rest` is the one-branch case of recursion on `branches`.

**What's different:**
- Trees can have many branches, so recursion must loop over or map across all of them rather than making one recursive call.
- **Trees can't be empty.** The slides flag this as a course-specific decision. A linked list has an empty case (`()`), so linked-list recursion usually bottoms out on "empty". Tree recursion bottoms out on **leaf**, not on empty, because there is no empty tree to bottom out on.

### The recipe for writing recursive tree functions

The slides give a checklist to answer *before* writing code:

1. **What small initial choice can I make?** For trees, this is usually: which branch to explore?
2. **What recursive call for each option?** Usually `f(b)` for each `b` in `t.branches`.
3. **How can you combine the results of those recursive calls?** To answer this, answer three sub-questions:
   - What type of values do they return?
   - What do the possible return values mean?
   - How can you use those return values to complete your implementation? For example: look to see if any option evaluated to true; add up the results from each option.

The slides note that **processing a leaf is often the base case**. That is the structural consequence of "trees can't be empty".

The shape that falls out of this recipe, over and over:

```python
def f(t):
    if is_leaf(t):
        return <something about t.label>
    else:
        return <combine>([f(b) for b in t.branches], <and maybe t.label>)
```

The list comprehension `[f(b) for b in t.branches]` is the idiom: it makes exactly one recursive call per branch and collects the results in a list, which you then feed to `sum`, `max`, `any`, `all`, and so on.

### When you do not need to combine

`above_root` is the counterexample in the lecture. It prints rather than returns, so there is nothing to combine: "Don't need to combine the recursive return call! Do need to print this label, if it's larger than the root." When a function's job is a side effect, the recursive calls are statements in a `for` loop, not expressions inside a combiner.

`above_root` also demonstrates the **helper-function-with-a-closure** pattern. The comparison is always against *the original root's* label, which changes meaning if you recurse on `t` directly (at depth 2, `t` would be a subtree and `t.label` would be the wrong root). Defining an inner `process(u)` keeps `t` fixed in the enclosing frame while `u` moves down the tree.

---

## Definitions

- **Tree:** a data structure with a root label and a list of branches, each of which is a tree. In this course a tree always has at least one node; there is no empty tree.
- **Label:** the value stored at a node. Can be any value (`int`, `str`, etc.). Accessed as `t.label`.
- **Branch:** one of the trees in `t.branches`. A branch is itself a tree, so it has its own label and branches.
- **Node:** a location in a tree. Every node has a label and is the root of some subtree.
- **Root / root node:** the top node of a tree. "Root label" means the label of that node, `t.label`.
- **Root of a branch:** the top node of a branch, that is, the branch's own root. A branch is also a tree, so this is just `b.label` for a branch `b`.
- **Leaf:** a tree with zero branches, that is, a node with no children. `is_leaf(t)` returns `not t.branches`. A leaf is also a tree.
- **Parent / child:** if node `p`'s branches include node `c`, then `p` is the parent of `c` and `c` is a child of `p`.
- **Path:** a sequence of nodes from one node down through children to another. In this lecture, "a path from the root to a leaf" means the sequence of nodes starting at the root, moving to a child each step, and ending at a leaf.
- **Path sum:** the sum of the labels along a path.
- **`is_leaf(t)`:** `not t.branches`, a function (not a method) that reports whether `t` has zero branches.
- **`field(default_factory=list)`:** dataclass mechanism giving each new instance a fresh empty list as the default value for `branches`.
- **Type variable `T` in `Tree[T]`:** describes the type of the labels in the tree. `Tree[int]` is a tree whose labels are integers.
- **`LinkedList[T]`:** the type alias `Link[T] | tuple[()]`, that is, either a `Link` or the empty tuple.

---

## Worked Examples

All examples use the lecture's tree:

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

which prints as

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

**Box-and-pointer / object reasoning in words:** there are nine `Tree` objects here. The outermost object has `label` bound to `1` and `branches` bound to a one-element list whose single element is a pointer to the `9` object. The `9` object's `branches` is a two-element list pointing at the `2` object and the `4` object. The `2` object's `branches` is a three-element list pointing at `5`, `8`, `3`. The `5` object's `branches` points at `6` and `7`. The objects for `6`, `7`, `8`, `3`, `4` each have `branches` bound to their own fresh empty list (fresh because of `default_factory=list`, so no two leaves share a list). Nothing points upward: there are no parent pointers, so a recursive function can never look back up at an ancestor unless you pass that information down yourself (which is exactly what `above_root` does with its closure).

---

### 1. `print_leaves`: recursion with side effects

```python
def print_leaves(t):
    """print the leaf labels of Tree t."""
    if is_leaf(t):
        print(t.label)
    else:
        for b in t.branches:
            print_leaves(b)
```

Why it works: there are exactly two cases for a tree, leaf or not-leaf, and they are exhaustive because `t.branches` is either empty or not. If `t` is a leaf, its only leaf is itself, so print its label. Otherwise `t` contributes no leaf of its own, and its leaves are exactly the union of the leaves of its branches, so recurse on each branch in order.

Trace on `t`: `print_leaves(t)` is not a leaf, so it calls `print_leaves` on the `9` tree, which calls on `2` then `4`. The `2` call recurses into `5`, `8`, `3`; the `5` call recurses into `6` and `7`, which are leaves and print. Then `8` prints, then `3` prints, then control returns up to the `9` frame, which moves to its second branch `4` and prints it.

Output:

```
6
7
8
3
4
```

Note that the left-to-right order of `t.branches` determines the print order, and that the deeper calls finish before their later siblings start.

---

### 2. `count_leaves`: combine with `sum`

Applying the recipe from the slides:

- Small, initial choice: which branch's leaves to count?
- Recursive call for each option: for each branch `b`, `count_leaves(b)`, meaning the number of leaves on branch `b`.
- Combine results: add up all of the counts.
- Type of the returned values: `int`. Meaning: a count. How to use them: sum them.

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

Step by step on `t`:

| Call | Branch results | Returns |
|---|---|---|
| `count_leaves(Tree(6))` | leaf | `1` |
| `count_leaves(Tree(7))` | leaf | `1` |
| `count_leaves(5-tree)` | `[1, 1]` | `sum([1,1]) = 2` |
| `count_leaves(Tree(8))` | leaf | `1` |
| `count_leaves(Tree(3))` | leaf | `1` |
| `count_leaves(2-tree)` | `[2, 1, 1]` | `4` |
| `count_leaves(Tree(4))` | leaf | `1` |
| `count_leaves(9-tree)` | `[4, 1]` | `5` |
| `count_leaves(t)` | `[5]` | `5` |

Environment reasoning: each of these is a separate frame of `count_leaves`, each with its own `t` bound to a different `Tree` object. The frames nest: the frame for the `5` tree is still open (waiting inside the list comprehension) while the frames for `6` and `7` run and return. At the deepest point, five `count_leaves` frames are open at once (for `1`, `9`, `2`, `5`, `6`). Nothing is mutated; each frame just returns a number up to its caller.

Note that the root's own label never appears in the answer. `count_leaves` cares only about structure.

---

### 3. `largest_label`: combine with `max`, including the root

- Small, initial choice: which branch to look for the largest label on?
- Recursive call for each option: for each branch `b`, `largest_label(b)`, meaning a label that is the largest one from branch `b`.
- Combine results: return the largest of these **and the root label**.

The slides build it as a fill-in-the-blank:

```python
def largest_label(t):
    """Return the largest label in tree t.

    >>> largest_label(Tree(6, [Tree(4), Tree(5), Tree(3)]))
    6
    """
    if is_leaf(t):
        return t.label
    else:
        return max([largest_label(b) for b in t.branches] + [t.label])
```

Two things to notice.

**Why `+ [t.label]`.** The doctest is the whole point: the root label `6` is bigger than any of the branch labels `4, 5, 3`, so if you only took `max` over the branch results you would return `5`, which is wrong. The largest label in a tree is the largest among (the root label, the largest label in each branch). `[... for b in t.branches] + [t.label]` builds a list containing all the candidates and takes the `max` of that one list.

**The slide's question: "What would happen if we got rid of this?"** (the `return t.label` base case). The answer is that the function still works. For a leaf, `t.branches` is empty, so the list comprehension evaluates to `[]`, and `[] + [t.label]` is `[t.label]`, so `max([t.label])` is `t.label`, exactly what the base case returned. The base case is **redundant** here because the `+ [t.label]` term guarantees the list passed to `max` is never empty. Contrast this with a version written as `max([largest_label(b) for b in t.branches] + [t.label])` turned into `max([largest_label(b) for b in t.branches])`: that one *does* need the base case, because `max([])` raises `ValueError: max() arg is an empty sequence`.

On the lecture's `t`, `largest_label(t)` returns `9`: the `5`-subtree yields `max(6, 7, 5) = 7`, the `2`-subtree yields `max(7, 8, 3, 2) = 8`, the `9`-subtree yields `max(8, 4, 9) = 9`, and the root yields `max(9, 1) = 9`.

(The docstring in `15.py` says "largest leaf label", but the slides and the implementation both compute the largest label anywhere in the tree, which is what the `Tree(6, [...])` doctest checks. Go by the implementation and the slides.)

---

### 4. `largest_path_sum`: combine with `max`, then add the root

- Small, initial choice: which branch contains the rest of the largest path sum?
- Recursive call for each option: for each branch `b`, `largest_path_sum(b)`, meaning the largest root-to-leaf path sum **within branch `b`**.
- Combine results: every root-to-leaf path of `t` is the root plus a root-to-leaf path of exactly one branch, so take the best branch result and add the root label.

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

Here the base case is **not** redundant: without it, a leaf would evaluate `max([])` and raise `ValueError`. The structural reason is that `t.label` is added *outside* the `max`, so nothing keeps the list non-empty. This contrast with `largest_label` is a favorite exam target.

Step by step on `t`:

| Call | Branch results | Returns |
|---|---|---|
| `lps(Tree(6))` | leaf | `6` |
| `lps(Tree(7))` | leaf | `7` |
| `lps(5-tree)` | `[6, 7]` | `5 + 7 = 12` |
| `lps(Tree(8))` | leaf | `8` |
| `lps(Tree(3))` | leaf | `3` |
| `lps(2-tree)` | `[12, 8, 3]` | `2 + 12 = 14` |
| `lps(Tree(4))` | leaf | `4` |
| `lps(9-tree)` | `[14, 4]` | `9 + 14 = 23` |
| `lps(t)` | `[23]` | `1 + 23 = 24` |

Checking by hand, the five root-to-leaf paths have sums `1+9+2+5+6 = 23`, `1+9+2+5+7 = 24`, `1+9+2+8 = 20`, `1+9+2+3 = 15`, `1+9+4 = 14`. The max is `24`, matching the doctest.

Note the important correctness argument: it is valid to pick the best branch *independently* and then add `t.label`, because `t.label` is added to every path equally. That kind of reasoning ("the choice decomposes") is what makes the one-line combination legitimate.

---

### 5. `above_root`: a helper that closes over the root

- Small, initial choice: which branch to look at for labels to print?
- Recursive call for each option: for each branch `b`, `process(b)`.
- Combine results: nothing to combine. Just print this label if it is larger than the root.

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

Environment reasoning: `process` is defined in the frame of `above_root`, so its parent frame is that `above_root` frame, where the name `t` is bound to the original tree. Every recursive call `process(b)` creates a new frame whose `u` is a different subtree, but every one of those frames looks up `t` in the *same* `above_root` frame, so `t.label` always means the original root label no matter how deep the recursion goes. This is why the comparison stays correct at depth 4.

Why a helper is needed: if you tried to write a single recursive `above_root(t)` that compares `u.label > t.label`, the parameter would be rebound on each recursive call and you would end up comparing each node against its own subtree root instead of the global root. The alternative without a closure is to pass the root label down as a second parameter, for example `def process(u, root_label)`. (extra context)

Also note the structure: the `if` and the `for` are siblings, not nested. The root itself is tested (`1 > 1` is false, so `1` is not printed), and every node is visited exactly once in preorder.

Running `above_root(t)` prints, in order:

```
9
2
5
6
7
8
3
4
```

Every label except the root `1` is greater than `1`, and the order is the preorder traversal: visit a node, then recurse into its branches left to right.

---

### 6. Min practice

Given

```python
xs = list(range(-10, 11))
```

write an expression that evaluates to the `x` in `xs` for which `x*x - 2*x + 1` is smallest.

```
>>> xs
[-10, -9, -8, -7, -6, -5, -4, -3, -2, -1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
>>> [x*x - 2*x + 1 for x in xs]
[121, 100, 81, 64, 49, 36, 25, 16, 9, 4, 1, 0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

The answer is `1`, and the crucial reading-comprehension point is that the answer is the **`x`**, not the smallest value of the expression. The smallest value of `x*x - 2*x + 1` is `0`; the `x` that achieves it is `1` (the expression is `(x - 1)**2`).

The intended expression uses `min` with a key function:

```python
>>> min(xs, key=lambda x: x*x - 2*x + 1)
1
```

`min(iterable, key=f)` returns the element of the iterable for which `f(element)` is smallest, rather than the smallest `f(element)` itself. If several elements tie, `min` returns the first one. (extra context)

An equivalent without `key`, using the fact that tuples compare lexicographically:

```python
>>> min([(x*x - 2*x + 1, x) for x in xs])[1]
1
```

This pairs each score with its `x`, takes the minimum pair (which compares scores first), and then extracts the `x`. (extra context: the tuple trick, which breaks ties by smallest `x` rather than by position.)

Connection to the tree material: this is the same "argmin/argmax" question as "which branch has the largest path sum?" The lecture's tree functions only need the *value* of the best branch, so plain `max` suffices. If a problem asked you to return the best *branch* or the best *path*, you would need the `key` or tuple technique.

---

## Common Pitfalls

1. **Confusing a branch with a label.** `t.branches[0]` is a `Tree`, not a number. `t.branches[0].label` is the number. Writing `sum(t.branches)` or `max(t.branches)` is a type error waiting to happen.
2. **Calling `is_leaf` on the wrong thing.** `is_leaf(t.branches)` is wrong; `is_leaf(t)` and `is_leaf(b)` for a branch `b` are right. `is_leaf` takes a tree.
3. **Treating a tree as possibly empty.** Trees in CS 61A cannot be empty, so `if t is None` or `if t == []` is never the right base case. The base case is `if is_leaf(t)`.
4. **`max()` on an empty sequence.** If the else-branch is `max([f(b) for b in t.branches])` with no extra element appended, a leaf would make that list empty and raise `ValueError`. Either keep a leaf base case (as in `largest_path_sum`) or append a guaranteed element (as in `largest_label`'s `+ [t.label]`).
5. **Forgetting to include the root label when the problem says "in the tree".** `largest_label` must consider `t.label`, which is why the doctest uses `Tree(6, [Tree(4), Tree(5), Tree(3)])`. Compare `count_leaves`, where the root label is irrelevant.
6. **Adding the root label in the wrong place.** `t.label + max([lps(b) ...])` is not the same as `max([t.label + lps(b) ...])` in general... actually these happen to agree for `largest_path_sum`, but `max([lps(b) ...] + [t.label])` (the `largest_label` shape) is definitely wrong for path sums. Get clear on whether the root label is a *candidate* answer or a *contribution* to every answer.
7. **Returning inside the loop.** `for b in t.branches: return f(b)` only ever examines the first branch. Collect results in a comprehension, or accumulate in a variable, before returning.
8. **Losing track of the original root in problems that compare against it.** `above_root` needs a helper closing over `t` (or an extra parameter). Recursing on the outer function rebinds `t` and silently changes the meaning of the comparison.
9. **Printing when you should return, or returning when you should print.** `print_leaves` and `above_root` return `None`; their output is a side effect. `count_leaves`, `largest_label`, `largest_path_sum` return values and print nothing. A function that prints cannot have its output combined by `sum` or `max`.
10. **Writing `branches: list[Tree[T]] = []` instead of `field(default_factory=list)`.** A single mutable default would be shared by every tree. Dataclasses reject it outright.
11. **Misreading the indented print format.** Two spaces of indentation is one level of depth. Lines at the same indentation are siblings only if no shallower line separates them from a common parent.
12. **Assuming `min`/`max` return the input element.** By default they return the extreme *value*. To get the element that minimizes some score, use `key=`.

---

## Likely Exam Points

### 1. Writing a `sum`-combining tree function

**Q.** Write `count_nodes(t)` that returns the total number of nodes in `t` (not just leaves).

**A.**
```python
def count_nodes(t):
    return 1 + sum([count_nodes(b) for b in t.branches])
```
No explicit leaf base case is needed: for a leaf the comprehension is empty, `sum([])` is `0`, and the function returns `1`. On the lecture's `t` this returns `9`. (The guaranteed `1 +` plays the same "keeps it safe" role that `+ [t.label]` plays in `largest_label`.)

### 2. The redundant-base-case question

**Q.** The slides ask: in `largest_label`, what would happen if we removed `if is_leaf(t): return t.label`? Does the same removal work for `largest_path_sum`?

**A.** For `largest_label`, nothing changes: a leaf has no branches, so the list is `[] + [t.label]` and `max([t.label])` is `t.label`, exactly the base case's answer. The base case is redundant. For `largest_path_sum`, removal breaks the function: a leaf would evaluate `max([])`, which raises `ValueError: max() arg is an empty sequence`, because `t.label` is added outside the `max` and so does not keep the list non-empty.

### 3. Tracing a tree function by hand

**Q.** Using the lecture's `t`, what does `largest_path_sum(t)` return, and which path achieves it?

**A.** `24`, via the path `1 -> 9 -> 2 -> 5 -> 7`. The branch values feeding each `max` are: `max(6, 7) = 7` at node `5` giving `12`; `max(12, 8, 3) = 12` at node `2` giving `14`; `max(14, 4) = 14` at node `9` giving `23`; then `1 + 23 = 24`.

### 4. Reading and writing the printed format

**Q.** Write the Python expression that constructs the tree printed below, then state how many leaves it has.

```
2
  4
  6
    8
    10
```

**A.** `Tree(2, [Tree(4), Tree(6, [Tree(8), Tree(10)])])`. It has three leaves (`4`, `8`, `10`), so `count_leaves` returns `3`.

### 5. Helper functions and the closure over the root

**Q.** Fill in `above_root` so that it prints all labels of `t` that are larger than the root label. Why must the body use an inner function?

**A.**
```python
def above_root(t):
    def process(u):
        if u.label > t.label:
            print(u.label)
        for b in u.branches:
            process(b)
    process(t)
```
The inner `process` has the `above_root` frame as its parent, so `t` always refers to the original tree no matter how deep the recursion. A direct recursion on `above_root` would rebind `t` to a subtree, and each node would be compared against its own subtree's root instead of the global root.

### 6. Side-effect recursion and output order

**Q.** What does `print_leaves(t)` print for the lecture's `t`?

**A.**
```
6
7
8
3
4
```
Branches are visited left to right, and a branch is fully explored before the next sibling begins.

### 7. Trees vs linked lists

**Q.** Give one structural way `Tree` and `Link` are alike and two ways they differ.

**A.** Alike: a tree where every node has at most one child behaves exactly like a linked list, with `label`/`branches` playing the role of `first`/`rest`. Different: (1) trees can have many branches, so recursion loops over all of them rather than making a single recursive call on `rest`; (2) trees cannot be empty in this course, so the base case is "leaf", whereas a linked list has an empty case `()` and its base case is "empty".

### 8. `min`/`max` with a key

**Q.** With `xs = list(range(-10, 11))`, write an expression for the `x` in `xs` minimizing `x*x - 2*x + 1`, and say what it evaluates to.

**A.** `min(xs, key=lambda x: x*x - 2*x + 1)`, which evaluates to `1`. Note the answer is the `x`, not the minimum value `0`.

### 9. Applying the recipe to a new problem

**Q.** Write `height(t)` returning the number of edges on the longest root-to-leaf path (a leaf has height `0`). (extra context: `height` was not in this lecture, but it is the same pattern as `largest_path_sum` and is standard on 61A exams.)

**A.**
```python
def height(t):
    if is_leaf(t):
        return 0
    return 1 + max([height(b) for b in t.branches])
```
For the lecture's `t`, this returns `4`. As in `largest_path_sum`, the leaf base case is required because `max([])` would otherwise error.

---

## Summary

- A **tree** has a **root label** (`t.label`) and a **list of branches** (`t.branches`); **each branch is itself a tree**, and a tree with zero branches is a **leaf**.
- The same structure is described two ways: recursively (root label plus branches) and relatively (nodes, labels, parent/child, root, path). Problem statements use both.
- The abstraction allows exactly: `t.label`, `t.branches`, `t.branches[i]`, `is_leaf(t)`, and treating `t` as an ordinary value.
- `Tree` is a dataclass with `label: T` and `branches: list[Tree[T]] = field(default_factory=list)`, so `Tree(4)` is a leaf; `is_leaf(t)` is `not t.branches`.
- `print(t)` uses `format_tree`, indenting each branch two spaces below its label; indentation depth equals tree depth.
- Trees resemble linked lists (a one-child-per-node tree is a linked list) but differ: trees can have many branches, and **trees cannot be empty**, so the base case is a leaf rather than an empty structure.
- The recipe: pick the small initial choice (which branch), make one recursive call per branch (`[f(b) for b in t.branches]`), then combine, after asking what type the results are, what they mean, and how to use them.
- `count_leaves`: base case returns `1`, else `sum` of the branch counts. On the lecture tree: `5`.
- `largest_label`: `max([largest_label(b) for b in t.branches] + [t.label])`; including `t.label` is essential for correctness and also makes the leaf base case redundant.
- `largest_path_sum`: `t.label + max([largest_path_sum(b) for b in t.branches])`; here the leaf base case is required, since `max([])` raises `ValueError`. On the lecture tree: `24`.
- `above_root`: no combining needed because it prints; uses an inner `process(u)` that closes over the outer `t` so comparisons always use the original root label.
- `min(xs, key=...)` returns the **element** that minimizes the key, not the minimum key value; for `x*x - 2*x + 1` over `range(-10, 11)` the answer is `x = 1`.
