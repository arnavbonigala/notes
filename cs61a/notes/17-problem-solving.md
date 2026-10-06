<!-- Mon, Oct 05, 2026 | sources: slides + code (no transcript available) -->
# Lecture 17: Problem Solving

This lecture is about *technique* rather than new language features: given a `Tree` abstract data type and nothing but recursion, how do you reliably get from an English problem statement to working code? The unifying theme is **which direction information flows** in a tree recursion. Return values flow *up* from branches to their parent, which is what the standard `sum([f(b) for b in t.branches])` template handles. But many problems ask about a node's *ancestors* (the nodes above it), and that information can only flow *down*, as an extra argument. The lecture develops three versions of that idea: an inner helper that carries "largest ancestor so far" (`bigs`), repurposing an existing parameter as a shrinking budget (`count_big`), and carrying the whole partial path as a list (`print_big`, `only_paths`). Along the way it introduces the *How to Design Programs* (HtDP) design process (data definitions, signature/purpose/header, functional examples, template, definition, testing) and applies it to two real exam problems from Fall 2015 and Fall 2023.

## Key Concepts

### 1. The `Tree` abstraction and its interface

`Tree` is defined in `17.py` as a dataclass with two attributes: a `label` and a list of `branches`, each of which is itself a `Tree`.

```python
@dataclass
class Tree[T]:
    label: T
    branches: list[Tree[T]] = field(default_factory=list)  # branches defaults to []
```

The slides state explicitly what you are allowed to do with a tree `t`:

- `t.label`: the label of the root node
- `t.branches`: the list of branches (each one a `Tree`)
- `t.branches[i]`: the branch at index `i`, which is a `Tree`
- `is_leaf(t)`: whether `t` has no branches
- treat `t` as a value: `return t`, `f(t)`, `[t]`, `s = t`, etc.

You build a tree by calling the constructor on a label and a list of branches:

```python
t = Tree(2, [Tree(1), Tree(1, [Tree(0), Tree(1)])])
isinstance(t, Tree)   # True
```

`is_leaf` is just `not t.branches`: a leaf is a tree whose branch list is empty. Note the vocabulary trap built into this ADT: **every branch is itself a full tree**, so "the tree at this node" and "the node" are the same object. There is no separate node type.

(extra context) Three Python details in the code that are not the point of the lecture but explain the syntax: `from __future__ import annotations` lets the annotation `list[Tree[T]]` mention `Tree` while `Tree` is still being defined; `class Tree[T]` is the Python 3.12+ generic syntax, so `Tree[int]` in a signature means "a tree whose labels are `int`s"; and `field(default_factory=list)` is how a dataclass gives a *fresh* empty list to each instance, which is necessary because a plain `= []` default would be shared by every tree ever constructed.

### 2. The basic tree recursion template

Almost every function in this lecture has the same shape:

```python
def f(t):
    if is_leaf(t):
        return <something about t.label>
    else:
        return <combine>([f(b) for b in t.branches])
```

The list comprehension handles "do the same thing to each branch" and the combiner (`sum`, `max`, `any`, list concatenation) handles "put the branch answers together". Writing the comprehension first and then asking "what combines these?" is the fastest way to fill in a blank on an exam.

A detail worth memorizing: **`sum([])` is `0`**, and `any([])` is `False`. Because of this, a recursive case written with `sum([... for b in t.branches])` often already does the right thing at a leaf, so you can sometimes delete the leaf base case entirely. That is exactly what happens in `bigs`: the first slide sketch had an `if is_leaf(t): return ___` branch, and the final solution does not need it.

### 3. Information that must flow downward

The slide for `bigs` shows the thought process in stages:

1. "Somehow track a list of ancestors", with the test `if node.label > max(ancestors)`.
2. "Somehow track the largest ancestor", with the test `if node.label > max_ancestor`.

The refinement from step 1 to step 2 is the real lesson. The question "is this label bigger than *all* its ancestors?" depends on the ancestors only through their maximum, so you do not need the list: a single number is enough. **Find the smallest summary of the ancestors that answers the question, and pass that down.** A list of ancestors would also work, but it costs more space and makes the code longer.

Since `bigs` only takes `t`, the extra parameter needs somewhere to live, so the solution defines an inner helper `f(a, x)` where `a` is the current node and `x` is the largest ancestor label so far. The outer function's only job is to pick the right *initial* value for `x`.

### 4. Choosing the initial value so the base case is free

The problem says to assume the root label is larger than all of its (nonexistent) ancestors. The solution encodes that with `f(t, t.label - 1)`: a value guaranteed to be less than `t.label`, so the root always counts. This is a recurring trick: instead of special-casing the root with an `if`, choose a starting argument that makes the general rule produce the right answer at the root.

(extra context) `float('-inf')` would work equally well here and does not depend on the labels being integers; `t.label - 1` is specific to integer labels.

### 5. Repurposing a parameter instead of adding one

`count_big(t, n)` counts root-to-leaf paths whose labels sum to at least `n`. The obvious approach is to carry the sum accumulated so far downward, which would need a helper. The lecture's solution instead *decrements the parameter it already has*:

```python
return sum([count_big(b, n - t.label) for b in t.branches])
```

`n` stops meaning "the original threshold" and starts meaning "how much is still needed from here down". The algebra justifying this: `label + rest >= n` exactly when `rest >= n - label`. Because the recursive meaning of `n` matches the original meaning of `n`, no helper function is required at all. Recognizing when a parameter can be reinterpreted as a *remaining budget* saves you from writing a helper under exam pressure.

### 6. Carrying the path itself, without mutation

When the answer needs the actual labels and not just a number, you have to carry the list. `print_big`'s helper does:

```python
def helper(t, p):
    p = p + [t.label]
```

`p` is the partial path of labels *above* `t`; the first line extends it with the current label. The use of `+` and not `append` is essential. `p + [t.label]` builds a brand new list and rebinds the local name `p` in this frame only. In box-and-pointer terms, each frame of `helper` points at its own list object, and the parent's list is untouched, so sibling branches each start from the same unmodified prefix. With `p.append(t.label)` every frame would be pointing at one shared list, and after visiting the first branch the second branch would see the first branch's labels still stuck on the end.

Notice also that `helper` only takes `t` and `p`: it reads `n` from the enclosing `print_big` frame by lexical scoping. Contrast with `count_big`, which changes `n` on the way down. Decide for each piece of downward information whether it is *constant* (close over it) or *changing* (pass it as a parameter).

The same pattern appears in the lecture's own `format_tree`, which passes `indent + '  '` down to each branch: a new string per level, never mutated.

### 7. Returning a new tree, and using `None` for "no answer"

`only_paths` returns a `Tree` when some root-to-leaf path sums to `n`, and `None` when none does. Two ideas combine here:

- **Building, not mutating.** The result is constructed with `Tree(t.label, [...])`, leaving the input tree alone.
- **A sentinel return value.** Each recursive call returns either a pruned tree or `None`, so the parent must filter `None`s out before building its own tree, and must itself return `None` if *all* its branches returned `None`. The `if any(new_branches)` test is what detects "at least one branch survived", and the function falls off the end (implicitly returning `None`) when nothing survived.

### 8. The HtDP design process

The lecture quotes the systematic design recipe from *How to Design Programs*:

1. **From Problem Analysis to Data Definitions.** Identify the information to represent and how to represent it in the language; write data definitions and illustrate them with examples.
2. **Signature, Purpose Statement, Header.** State what kinds of data the function consumes and produces, answer concisely what it computes, and define a stub matching the signature.
3. **Functional Examples.** Work through examples illustrating the purpose.
4. **Function Template.** Translate the data definitions into an outline of the function.
5. **Function Definition.** Fill in the template, exploiting the purpose statement and examples.
6. **Testing.** Turn the examples into tests and make them pass; tests find mistakes and also document the function for later readers.

The exam problems in the lecture are annotated in exactly this style. `count_big` is labeled with its signature `(Tree, minimum path sum) -> number of paths`; the helper in `print_big` is given both a signature `(Tree, list[int]) -> None` and a one-line purpose statement, "For a node and partial path, extend the path and maybe print it". Writing those two lines for a helper *before* writing its body is the practical takeaway: most tree-recursion bugs come from being unsure what the helper's extra parameter means.

## Definitions

- **Tree**: a data type with a `label` and a list of `branches`, each of which is a `Tree`. There is no separate node type: every branch is a tree.
- **Label**: the value stored at the root of a tree, accessed as `t.label`.
- **Branches**: the list of subtrees directly below the root, accessed as `t.branches`. `t.branches[i]` is a `Tree`.
- **Leaf**: a tree with no branches. `is_leaf(t)` returns `not t.branches`.
- **Ancestor** (of a node): any node on the path from the root down to, but not including, that node. The root has no ancestors.
- **Path through a tree** (lecture's definition): a list of adjacent node labels that starts with the root label and ends with a leaf label. Paths always run root-to-leaf; partial top-down segments are not paths.
- **`bigs(t)`**: the number of nodes in `t` whose labels are larger than all labels of their ancestors (the root always counts).
- **`count_big(t, n)`**: the number of paths in `t` whose label sum is greater than or equal to `n`.
- **`print_big(t, n)`**: prints (returns `None`) each path in `t` whose label sum is greater than or equal to `n`, one list per line.
- **`only_paths(t, n)`**: a tree containing only the nodes of `t` that lie on some root-to-leaf path summing to `n`, or `None` if no such path exists.
- **`one(b)`**: helper given by the exam problem; returns `1` if `b` is a true value and `0` otherwise.
- **`format_tree(t, indent='')`**: returns a string with each branch indented two spaces further than its parent's label; used by `Tree.__str__`.
- **Signature** (HtDP): a statement of what kinds of data a function consumes and produces.
- **Purpose statement** (HtDP): a concise answer to "what does this function compute?"
- **Stub / header** (HtDP): a minimal definition that satisfies the signature before the body is written.
- **Template** (HtDP): an outline of the function derived from the shape of the data (for trees: a leaf case plus a loop or comprehension over `t.branches`).

## Worked Examples

### Example 1: `bigs`, with a helper carrying the largest ancestor

```python
def bigs(t: Tree[int]):
    """Return the number of nodes in t that are larger than all their ancestors.

    >>> a = Tree(1, [Tree(4, [Tree(4), Tree(5)]), Tree(3, [Tree(0, [Tree(2)])])])
    >>> bigs(a)
    4
    """
    def f(a, x):
        if a.label > x:
            return 1 + sum([f(b, a.label) for b in a.branches])
        else:
            return sum([f(b, x) for b in a.branches])
    return f(t, t.label - 1)
```

Reading the body: `f(a, x)` means "how many nodes in the tree `a` are larger than all their ancestors, given that the largest label strictly above `a` is `x`". There are two cases, and the only difference between them is *what gets passed down*:

- If `a.label > x`, this node counts (hence `1 +`), and `a.label` is now the new maximum for everything below it, so the recursive calls pass `a.label`.
- Otherwise this node does not count, and the maximum above the branches is still `x`, so the recursive calls pass `x` unchanged.

Both cases use `sum([... for b in a.branches])`, which returns `0` on a leaf because `sum([]) == 0`. That is why no `is_leaf` check is needed.

Full trace on `a = Tree(1, [Tree(4, [Tree(4), Tree(5)]), Tree(3, [Tree(0, [Tree(2)])])])`. The outer call is `f(a, 0)` since `t.label - 1 == 0`:

| Call | `a.label > x`? | Contribution |
|---|---|---|
| `f(1-node, 0)` | `1 > 0` yes | `1 + f(4-node, 1) + f(3-node, 1)` |
| `f(4-node, 1)` | `4 > 1` yes | `1 + f(4-leaf, 4) + f(5-leaf, 4)` |
| `f(4-leaf, 4)` | `4 > 4` no | `sum([])` = `0` |
| `f(5-leaf, 4)` | `5 > 4` yes | `1 + sum([])` = `1` |
| `f(3-node, 1)` | `3 > 1` yes | `1 + f(0-node, 3)` |
| `f(0-node, 3)` | `0 > 3` no | `f(2-leaf, 3)` |
| `f(2-leaf, 3)` | `2 > 3` no | `0` |

So `f(4-node, 1) = 1 + 0 + 1 = 2`, `f(3-node, 1) = 1 + 0 = 1`, and the total is `1 + 2 + 1 = 4`. The counted nodes are `1`, `4`, `5`, `3`, which matches the check marks on the slide diagram.

Environment reasoning in words: `f` is defined inside `bigs`, so its parent frame is the `bigs` frame. Each recursive call creates its own frame with its own bindings for `a` and `x`, so the "largest ancestor" value is naturally per-subtree with no bookkeeping needed to restore it on the way back up. Note that `f` does not actually need anything from the `bigs` frame; the nesting here is just to keep the extra parameter out of `bigs`'s public signature. The node `2` is a good illustration of why `x` must be the max of *all* ancestors and not the parent's label: `2 > 0` (its parent), but `2 < 3` (its grandparent), so it must not count, and passing `x` unchanged through the `0` node is what preserves that.

### Example 2: `count_big`, repurposing `n` as a remaining budget

```python
def one(b):
    if b:
        return 1
    else:
        return 0

def count_big(t, n):
    """Return the number of paths in t that have a sum larger or equal to n.

    >>> t = Tree(1, [Tree(2), Tree(3, [Tree(4), Tree(5)])])
    >>> count_big(t, 3)
    3
    >>> count_big(t, 6)
    2
    >>> count_big(t, 9)
    1
    """
    if is_leaf(t):
        return one(t.label >= n)
    else:
        return sum([count_big(b, n - t.label) for b in t.branches])
```

The example tree has three paths: `[1, 2]` summing to 3, `[1, 3, 4]` summing to 8, and `[1, 3, 5]` summing to 9.

Trace of `count_big(t, 6)`:

- `t` is not a leaf, so compute `count_big(Tree(2), 6 - 1)` and `count_big(Tree(3, [...]), 6 - 1)`. The `5` means "the rest of this path still needs to total at least 5".
- `count_big(Tree(2), 5)`: leaf, `one(2 >= 5)` is `one(False)` = `0`. Correct: path `[1, 2]` sums to 3, below 6.
- `count_big(Tree(3, [Tree(4), Tree(5)]), 5)`: not a leaf, so `count_big(Tree(4), 5 - 3)` and `count_big(Tree(5), 5 - 3)`, i.e. budget 2 each.
  - `one(4 >= 2)` = `1`, `one(5 >= 2)` = `1`, so this subtree contributes `2`.
- Total: `0 + 2 = 2`. Matches the docstring.

Why the base case must be `t.label >= n` and not `t.label >= n - something`: by the time you reach a leaf, `n` has already had every ancestor's label subtracted from it, so `n` is exactly what this last label must cover on its own.

(extra context) `one` is only needed because the skeleton supplied it. In Python `True == 1` and `False == 0`, so `return t.label >= n` or `sum(...)` over booleans would also work; exams sometimes supply helpers like `one` to tell you the intended shape of the answer.

### Example 3: `print_big`, carrying the partial path

```python
def print_big(t, n):
    """Print the paths in t that have a sum larger or equal to n.

    >>> t = Tree(1, [Tree(2), Tree(3, [Tree(4), Tree(5)])])
    >>> print_big(t, 3)
    [1, 2]
    [1, 3, 4]
    [1, 3, 5]
    """
    def helper(t, p):
        p = p + [t.label]
        if is_leaf(t):
            if sum(p) >= n:
                print(p)
        else:
            for b in t.branches:
                helper(b, p)
    helper(t, [])
```

The helper's signature is `(Tree, list[int]) -> None` and its purpose is "for a node and partial path, extend the path and maybe print it". `p` arrives holding the labels strictly above `t`; the first line makes it hold the labels from the root down to and including `t`.

Trace of `print_big(t, 6)`:

1. `helper(t, [])` sets `p = [1]`. Not a leaf, so loop over branches with `p = [1]`.
2. `helper(Tree(2), [1])` sets its own `p = [1, 2]`. Leaf; `sum([1, 2]) == 3`, not `>= 6`, so nothing is printed.
3. Back in frame 1, `p` is still `[1]` (frame 2 rebound only its own `p`, and `+` did not touch the `[1]` object). `helper(Tree(3, [...]), [1])` sets `p = [1, 3]`. Not a leaf.
4. `helper(Tree(4), [1, 3])` sets `p = [1, 3, 4]`, a leaf, `sum == 8 >= 6`, prints `[1, 3, 4]`.
5. `helper(Tree(5), [1, 3])` sets `p = [1, 3, 5]`, `sum == 9 >= 6`, prints `[1, 3, 5]`.

Box-and-pointer reasoning: after step 4, there are three distinct list objects alive, `[1]`, `[1, 3]`, and `[1, 3, 4]`, each pointed at by exactly one frame. If line 1 had been `p.append(t.label)`, every frame would point at the single list created by `helper(t, [])`, and step 5 would see `[1, 3, 4, 5]` instead of `[1, 3, 5]`.

Two further observations: the comparison uses `n`, which `helper` finds in the parent frame (`print_big`'s frame) because `n` never changes as you descend; and `print_big` returns `None`, so in an interactive session only the printed lines appear, with no `>>>` result value.

### Example 4: `only_paths`, pruning and returning a new tree

```python
def only_paths(t: Tree[int], n: int) -> Tree[int] | None:
    """Return a Tree with only the nodes of t that are on a path from the root
    to a leaf with labels that sum to n, or None if no path sums to n.

    >>> t = Tree(3, [Tree(4), Tree(1, [Tree(3, [Tree(2)]), Tree(2, [Tree(1)]), Tree(5), Tree(3)])])
    >>> print(only_paths(t, 7))
    3
      4
      1
        2
          1
        3
    >>> u = Tree(3, [Tree(4), Tree(5, [Tree(6)]), Tree(7), Tree(8, [Tree(9), Tree(10)])])
    >>> print(only_paths(u, 2))
    None
    """
    if is_leaf(t) and t.label == n:
        return t
    new_branches = [only_paths(b, n - t.label) for b in t.branches]
    if any(new_branches):
        return Tree(t.label, [b for b in new_branches if b != None])
```

Line by line:

- `if is_leaf(t) and t.label == n: return t` is the success base case. A path must end at a leaf, so a non-leaf whose label happens to equal `n` is *not* a success and must keep recursing. This is why `is_leaf(t)` is conjoined with the label test.
- `new_branches` applies the same budget-decrement idea as `count_big`: each branch is asked "is there a path through you summing to `n - t.label`?" The entries are a mix of pruned `Tree`s and `None`s.
- `any(new_branches)` is `True` exactly when at least one branch survived. This works because a `Tree` instance is always truthy (the dataclass defines no `__bool__` or `__len__`), so the only falsy entries are the `None`s. Note the two convenient degenerate cases: for a leaf whose label is not `n`, `t.branches` is empty, so `new_branches == []`, `any([])` is `False`, and the function falls off the end returning `None`.
- The final `Tree(t.label, [...])` builds a fresh node whose branches are the surviving pruned branches, in their original relative order.
- There is no explicit `else`, so when `any(new_branches)` is false the function implicitly returns `None`, which is exactly the specified "no path sums to `n`" answer.

Trace of the first doctest, `only_paths(t, 7)`:

- Root `3`: not a leaf, so each branch gets budget `7 - 3 = 4`.
- `only_paths(Tree(4), 4)`: leaf and `4 == 4`, returns the leaf `Tree(4)`.
- `only_paths(Tree(1, [...]), 4)`: not a leaf, so each of its four branches gets budget `4 - 1 = 3`.
  - `only_paths(Tree(3, [Tree(2)]), 3)`: not a leaf despite `3 == 3`, so its branch gets budget `0`; `only_paths(Tree(2), 0)` is a leaf with `2 != 0`, so `None`. `any([None])` is `False`, so this branch is pruned to `None`. (Path `3, 1, 3, 2` sums to 9, not 7, so pruning is right.)
  - `only_paths(Tree(2, [Tree(1)]), 3)`: branch budget `3 - 2 = 1`; `only_paths(Tree(1), 1)` is a matching leaf, returns `Tree(1)`. So this yields `Tree(2, [Tree(1)])`. (Path `3, 1, 2, 1` sums to 7.)
  - `only_paths(Tree(5), 3)`: leaf, `5 != 3`, `None`.
  - `only_paths(Tree(3), 3)`: leaf, matches, returns `Tree(3)`. (Path `3, 1, 3` sums to 7.)
  - `new_branches == [None, Tree(2, [Tree(1)]), None, Tree(3)]`, `any` is `True`, so returns `Tree(1, [Tree(2, [Tree(1)]), Tree(3)])`.
- Top level: `new_branches == [Tree(4), Tree(1, [...])]`, so the result is `Tree(3, [Tree(4), Tree(1, [Tree(2, [Tree(1)]), Tree(3)])])`, which `format_tree` prints exactly as the docstring shows.

Trace of the second doctest, `only_paths(u, 2)`: the root label `3` already exceeds `2`, so every branch is called with budget `-1` and no positive leaf label can match; every entry of `new_branches` is `None`, `any` is `False`, and the function returns `None`. `print(None)` displays `None`.

(extra context) Two notes on the result object. For a matching leaf the function returns `t` itself, not a copy, so the returned tree *shares* leaf nodes with the input: mutating `result.branches[0].label` would change the original. Also, `b != None` works here because dataclass `__eq__` with a non-`Tree` returns `NotImplemented` and Python falls back to identity, but `b is not None` is the idiomatic and safer form.

## Common Pitfalls

- **Mutating the accumulated path.** `p.append(t.label)` instead of `p = p + [t.label]` shares one list across all frames, and sibling subtrees contaminate each other. Always build a new list when passing a path downward.
- **Forgetting that the extra parameter is per-subtree.** In `bigs`, writing `f(b, a.label)` in the *else* branch (where the node did not count) would wrongly lower the running maximum; the `else` case must pass `x` through unchanged.
- **Using the parent's label instead of the max ancestor.** `bigs` compares against the largest label anywhere above, not the immediate parent. The node `2` under `0` under `3` in the lecture example is exactly the case that distinguishes these.
- **Adding an unnecessary leaf base case, or needing one and omitting it.** `sum([])` is `0`, so a `sum`-over-branches recursive case usually handles leaves correctly (as in `bigs`). But if the leaf answer is not the identity element of the combiner, as in `count_big` where a leaf must test `t.label >= n`, you need the explicit case.
- **Mixing up the two meanings of `n`.** In `count_big`, `n` is the *remaining* requirement, not the original. Subtracting `t.label` at the leaf a second time, or forgetting to subtract at an internal node, both give wrong answers.
- **Testing a label match before checking `is_leaf` in `only_paths`.** `t.label == n` alone would accept an internal node and return its whole subtree, violating the "path ends at a leaf" definition.
- **Forgetting to filter `None` before constructing the new tree.** `Tree(t.label, new_branches)` would put `None` values into a branch list, which then breaks `format_tree`'s `isinstance` assertions and any later recursion.
- **Assuming `only_paths` has an explicit `else: return None`.** It relies on falling off the end of the function; if you add code after the `if`, be sure you have not changed that behavior.
- **Confusing printing with returning.** `print_big` returns `None` and produces output as a side effect; `count_big` returns a number and prints nothing. A doctest for one cannot be written in the style of the other.
- **Breaking the tree abstraction.** Reaching for `t.branches[0].branches[1].label` instead of recursing is fragile and usually a sign the recursion is not set up right. The slide's list of permitted operations is the intended toolbox.
- **Mutable default for `branches`.** (extra context) If you re-implement `Tree` yourself, `branches=[]` as a default argument creates one shared list for all trees; use `field(default_factory=list)` in a dataclass or `branches=None` plus a fresh list in a plain `__init__`.

## Likely Exam Points

**1. Filling in a tree-recursion template with an aggregator over `t.branches`.**

*Practice:* (extra context) Implement `max_path_sum(t)`, returning the largest sum of any root-to-leaf path.

*Answer:*
```python
def max_path_sum(t):
    if is_leaf(t):
        return t.label
    return t.label + max([max_path_sum(b) for b in t.branches])
```
The leaf case is required here because `max([])` raises a `ValueError`, unlike `sum([])`.

**2. Writing a helper with an extra downward parameter, and choosing its initial value.**

*Practice:* In `bigs`, what would go wrong with `return f(t, t.label)`, and what is the smallest change to `bigs` that makes it count nodes larger than *or equal to* all their ancestors? What would `bigs(a)` be for the lecture's tree `a` under that definition?

*Answer:* `f(t, t.label)` fails the test `a.label > x` at the root (`t.label > t.label` is false), so the root would never be counted and `bigs(Tree(5))` would be `0` instead of `1`. For the "or equal" version, change only the condition to `a.label >= x`; the initial `t.label - 1` still works since `t.label >= t.label - 1`. On `a`, the previously rejected `4` under the `4` now counts, giving `5`.

**3. `sum([])` and when the leaf case can be dropped.**

*Practice:* What does `bigs(Tree(5))` return, and which part of the code produces it?

*Answer:* `1`. `bigs` calls `f(Tree(5), 4)`; `5 > 4`, so it returns `1 + sum([])`, and `sum([])` is `0` because `Tree(5).branches` is empty.

**4. The "remaining budget" transformation.**

*Practice:* Evaluate `count_big(Tree(1, [Tree(2), Tree(3, [Tree(4), Tree(5)])]), 9)` by hand.

*Answer:* `1`. Branches get budget `9 - 1 = 8`. `Tree(2)` is a leaf: `one(2 >= 8) = 0`. `Tree(3, [...])` passes budget `8 - 3 = 5` down: `one(4 >= 5) = 0` and `one(5 >= 5) = 1`, total `1`. Only `[1, 3, 5]`, summing to 9, qualifies.

**5. Why the path must be copied, not mutated.**

*Practice:* If `p = p + [t.label]` in `print_big` were replaced by `p.append(t.label)`, what would `print_big(Tree(1, [Tree(2), Tree(3)]), 0)` print?

*Answer:* `[1, 2]` then `[1, 2, 3]`. The single shared list keeps the `2` from the first branch, so the second leaf sees `[1, 2]` and appends `3`. (With the correct code it prints `[1, 2]` then `[1, 3]`.)

**6. Print-versus-return behavior.**

*Practice:* What is displayed by `print_big(Tree(5), 3)` in an interactive interpreter, and what is the function's return value?

*Answer:* It displays the single line `[5]`. `helper(Tree(5), [])` sets `p = [5]`, the node is a leaf, and `sum([5]) >= 3`. The return value is `None`, so nothing else is echoed.

**7. Returning `None` as a sentinel, and the `any` test.**

*Practice:* What does `only_paths(Tree(1, [Tree(2), Tree(3)]), 4)` return, and why is `any(new_branches)` a correct test for "keep this node"?

*Answer:* `Tree(1, [Tree(3)])`. The root is not a leaf, so branches get budget `3`: `Tree(2)` is a non-matching leaf and yields `None`, while `Tree(3)` matches and yields `Tree(3)`. `any([None, Tree(3)])` is `True`, and filtering out the `None` leaves `[Tree(3)]`. `any` is correct because every `Tree` instance is truthy, so the only false entries are the `None`s, making `any` equivalent to "some branch is not `None`".

**8. Ordering of conditions in `only_paths`.**

*Practice:* Why is the base case `is_leaf(t) and t.label == n` rather than just `t.label == n`?

*Answer:* A path must end at a leaf. Without `is_leaf(t)`, an internal node whose label equals the remaining budget would be returned together with its entire unpruned subtree, even though no root-to-leaf path through it necessarily sums to `n`.

**9. Reading and writing signatures and purpose statements (HtDP).**

*Practice:* State the signature and purpose of `print_big`'s inner `helper`, and name the six steps of the HtDP design process in order.

*Answer:* Signature `(Tree, list[int]) -> None`; purpose: for a node and the partial path of labels above it, extend the path with this node's label and print it if the node is a leaf and the path sum is at least `n`. The six steps: (1) from problem analysis to data definitions, (2) signature, purpose statement, header, (3) functional examples, (4) function template, (5) function definition, (6) testing.

**10. Respecting the abstraction.**

*Practice:* Which of these are permitted by the lecture's list of tree operations: `t.branches[0]`, `len(t.branches)`, `t.label`, `is_leaf(t)`, `[t]`?

*Answer:* All of these follow from the stated interface: `t.label`, `t.branches` (and therefore indexing it and taking its length), `is_leaf(t)`, and treating `t` as a value. What the slide is warning against is bypassing the constructor/selectors or assuming anything else about the representation.

## Summary

- `Tree` has exactly two parts, `label` and `branches`, every branch is itself a `Tree`, and `is_leaf(t)` is `not t.branches`; build trees with `Tree(label, branches)`.
- The default template is a leaf case plus `<combine>([f(b) for b in t.branches])`; because `sum([]) == 0`, a `sum`-based recursive case often makes the leaf case unnecessary (as in `bigs`), while `max` and leaf-specific tests (as in `count_big`) still need one.
- Return values carry information *up* the tree; anything about *ancestors* must be passed *down* as an argument.
- `bigs` passes down the single largest ancestor label rather than a list of ancestors: track the minimal summary that answers the question. The inner helper `f(a, x)` passes `a.label` down when the node counts and `x` down when it does not, and `f(t, t.label - 1)` makes the root count without a special case.
- `count_big` needs no helper because it repurposes `n` as the *remaining* required sum, using `n - t.label` on the way down and testing `t.label >= n` at the leaf.
- `print_big` carries the partial path with `p = p + [t.label]`, which creates a fresh list per frame; `append` would share one list and corrupt sibling paths. Constant information like `n` is read from the enclosing frame instead of being passed.
- `only_paths` combines the budget idea with a `None` sentinel: match a leaf exactly (`is_leaf(t) and t.label == n`), recurse on branches with `n - t.label`, keep the node when `any(new_branches)` is true, filter out the `None`s, and build a *new* `Tree`; falling off the end returns `None` for "no path".
- The HtDP recipe (data definitions, signature/purpose/header, functional examples, template, definition, testing) is the process behind these solutions; writing a helper's signature and purpose statement first is the single most useful habit for tree problems.
