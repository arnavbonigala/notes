<!-- Mon, Nov 02, 2026 | sources: slides (no transcript available) -->
# Lecture 29: Immutable Data

## Overview

This lecture closes out the Gleam portion of the course by pulling together three ideas about programming with data that can never change. First, a review of pattern matching on custom types (including a debugging exercise where a plausible-looking `increasing` function silently gives the wrong answer because one shape of input is never matched) plus the `case x, y { ... }` multiple-subject form, which is just sugar for matching on a tuple. Second, errors as values: Gleam has no exceptions in ordinary code, so a function that might fail returns a built-in `Result(a, e)` (either `Ok(value)` or `Error(reason)`), and the compiler refuses to build the program until every possible result is handled. Third, the escape hatch and the payoff: because Gleam compiles to JavaScript, you can declare `@external` functions that call JavaScript, and JavaScript *does* allow mutation, so you can smuggle a mutable cache or call-counter into a Gleam program; but when you stay inside Gleam, immutability buys you **structural sharing**, as in the `remove_leaf` exercise, where building a "new" tree without its leftmost leaf copies only the path down to that leaf and shares every untouched subtree instead of copying it.

> **Note on language:** every example in this lecture is written in **Gleam**, not Python. The notes below keep the lecture's own Gleam code as the primary source of truth, and add Python analogues separately, marked **(extra context)**, since Python is the language you can run reasoning experiments in.

---

## Key Concepts

### 1. Custom types with multiple constructors, and what field access you get

```gleam
pub type NotEmpty(a) {
  Last(first: a)
  Link(first: a, rest: NotEmpty(a))
}
```

This is a linked list that *cannot* be empty: it either ends with `Last(x)` or is a `Link(x, rest)` whose `rest` is itself a `NotEmpty`. Compare this to the Python `Link` class you've used, where emptiness is represented by a sentinel `Link.empty`. Here, emptiness is excluded by the type itself, so any function taking a `NotEmpty(a)` is statically guaranteed a first element.

That guarantee is exactly why `starts_with` can be a one-liner:

```gleam
pub fn starts_with(s: NotEmpty(a), v: a) -> Bool {
  s.first == v
}
```

The important rule to internalize: **you may use `.field` on a value of a multi-constructor type only when *every* constructor has a field with that name and that type.** `first` is present in both `Last` and `Link`, so `s.first` compiles. `s.rest` would *not* compile, because `Last` has no `rest`. To get at `rest` you must pattern match, which forces you to say what happens in the `Last` case.

### 2. Pattern matching is checked for exhaustiveness, not for correctness

The debugging exercise:

```gleam
pub fn increasing(s: NotEmpty(Int)) -> Bool {
  case s {
    Last(_) -> True
    Link(first, Link(second, _)) if second <= first -> False
    Link(_, rest) -> increasing(rest)
  }
}
```

This compiles, because the three clauses together cover every possible value, but it is wrong. The second clause can only fire when `rest` is itself a `Link`. If `rest` is a `Last`, the clause does not match and control falls through to `Link(_, rest) -> increasing(rest)`, which recurses into a `Last` and immediately returns `True`. **The final adjacent pair is never compared.** So `increasing(Link(5, Last(3)))` returns `True`.

Two lessons generalize well beyond this function:

- A **guard** (`if second <= first`) is part of the pattern. If the guard is false, Gleam moves on to the next clause rather than taking that branch. So "pattern matched but guard failed" and "pattern did not match" both fall through, and a later catch-all absorbs both.
- Exhaustiveness checking only proves that *some* branch runs for every input. It cannot tell you that the branch which runs is the branch you meant. A broad catch-all clause (`Link(_, rest) -> ...`) is precisely what hides the hole here.

Two fixes (**extra context**, the slide only asks you to spot the bug):

```gleam
// Fix A: handle the two-element-tail case explicitly
case s {
  Last(_) -> True
  Link(first, Last(second)) -> first < second
  Link(first, Link(second, _)) if second <= first -> False
  Link(_, rest) -> increasing(rest)
}

// Fix B: use field access, which is legal because both variants have `first`
case s {
  Last(_) -> True
  Link(first, rest) -> first < rest.first && increasing(rest)
}
```

Fix B is shorter and is the reason `starts_with` appears on the same slide: `rest.first` works for both a `Last` tail and a `Link` tail.

Note also that the comparison `second <= first -> False` makes this **strictly** increasing: equal neighbors are rejected.

### 3. Multiple subjects in a `case`

A `case` can take several comma-separated subjects and match combinations:

```gleam
let result = case x, y {
  0, 0 -> "Both are zero"
  0, _ -> "First is zero"
  _, 0 -> "Second is zero"
  _, _ -> "Neither are zero"
}
```

This is **exactly equivalent** to building a tuple and matching patterns against it:

```gleam
let result = case #(x, y) {
  #(0, 0) -> "Both are zero"
  #(0, _) -> "First is zero"
  #(_, 0) -> "Second is zero"
  #(_, _) -> "Neither are zero"
}
```

Same semantics, less punctuation. The ordering rule is the usual one: clauses are tried top to bottom, first match wins, so the specific combinations must precede the catch-alls. The final `_, _` clause makes the `case` exhaustive.

### 4. Errors as values: the `Result` type

In Python, "most code is written assuming no errors occur, and error handling is optional":

```python
>>> s = [3, 4]
>>> s[5]
IndexError: list index out of range
```

Nothing in the source of `s[5]` tells you it can fail, and nothing forces you to deal with it. Gleam takes the opposite stance: a function that can fail says so in its return type, and **the program does not run until the compiler confirms all errors are handled.**

`Result(a, e)` is built in with two constructors: `Ok(value)` and `Error(reason)`. Compare:

```gleam
fn first_int(s: List(Int)) -> Int {        fn nth_int(s: List(Int), n: Int) -> Int {
  case s {                                   case list.first(list.drop(s, n)) {
    [x, ..] -> x                               Ok(k) -> k
    [] -> 0                                    Error(Nil) -> 0
  }                                          }
}                                          }
```

On the left, pattern matching on the list itself forces the `[]` case to be written. On the right, `list.first(list.drop(s, n))` evaluates to a `Result(Int, Nil)`, and the `case` must handle both `Ok` and `Error`. Either way there is no path where failure is ignored. `Nil` here is Gleam's one-value "no information" type: it says "this failed" without saying why.

To **create** a result, write `Ok(...)` or `Error(...)`. To **consume** one:

```gleam
case some_expression_that_evaluates_to_a_result {
  Ok(...) -> ...
  Error(...) -> ...
}
```

The standard library has helpers so you don't write that `case` constantly. The one shown is `list.try_map`, which applies a `Result`-returning function to every element and gives you either all the successes or the first failure:

```gleam
int.parse            // pub fn parse(string: String) -> Result(Int, Nil)

list.try_map(["1", "2", "3"],   int.parse)   // Ok([1, 2, 3])
list.try_map(["1", "two", "3"], int.parse)   // Error(Nil), because of "two"
```

Notice the type-level discipline: `try_map` turns a `List(Result(Int, Nil))`-shaped computation into a single `Result(List(Int), Nil)`, so the caller handles failure once rather than per element.

### 5. External functions: the mutation escape hatch

Gleam compiles to JavaScript. JavaScript allows mutation. Gleam lets you declare that a function is implemented in JavaScript, and that function can hand back Gleam values. "Therefore, it's possible to create a mutable Gleam object."

```gleam
pub type Cache          // a Gleam type name for a JavaScript object, with no constructors

@external(javascript, "./fib_ffi.mjs", "new_cache")
pub fn new_cache() -> Cache

@external(javascript, "./fib_ffi.mjs", "lookup")
fn lookup(cache: Cache, n: Int) -> Result(Int, Nil)

@external(javascript, "./fib_ffi.mjs", "store")
fn store(cache: Cache, n: Int, value: Int) -> Int
```

```javascript
import { Ok, Error } from "./gleam.mjs";

export function new_cache() {
  return new Map();                       // JavaScript's dict
}

export function lookup(cache, n) {
  return cache.has(n)                     // JavaScript's if/else (the ?: ternary)
    ? new Ok(cache.get(n))
    : new Error(undefined);
}

export function store(cache, n, value) {
  cache.set(n, value);                    // Mutation!
  return value;
}
```

Three things to see here:

- `pub type Cache` with **no constructors** declares an opaque type. Gleam knows the name and can type-check it, but there is no Gleam code that can build or inspect one; only the external functions can.
- The JavaScript side imports `Ok` and `Error` from the compiled Gleam runtime so it can return genuine Gleam `Result` values. The FFI boundary is where the two value representations meet.
- Mutation happens inside `store`. From Gleam's point of view `store` is just a function returning an `Int`; the mutation is invisible to the type system. That is the point of an escape hatch, and also its danger.

Using it:

```gleam
pub fn main() {
  let cache = new_cache()
  store(cache, 5, 25)
  let result = case lookup(cache, 5) {   // result is assigned 25
    Ok(v) -> v
    Error(Nil) -> 0
  }
}
```

### 6. Immutability and structural sharing

The payoff slide. With immutable data, "modifying" a structure means constructing a new value that **shares** every part you did not touch. The `remove_leaf` exercise is annotated "Never copied!" pointing at the subtrees that survive untouched: they are not duplicated, the new tree just points at the very same subtree objects as the old one.

Why is this safe? Because nothing can ever mutate a shared subtree, no one can tell whether they are looking at a copy or at the original. Sharing is unobservable, so the implementation is free to share, and the cost of "removing" a leaf from a big tree is proportional to the **depth of the path to that leaf**, not the size of the tree. In Python, where lists and objects are mutable, sharing is observable (aliasing bugs), which is why Python code that wants to return a modified structure usually has to deep-copy defensively.

This also means the old value remains valid after the "modification." Both the before-tree and the after-tree exist at once, overlapping in memory. (**Extra context:** structures with this property are called *persistent* data structures, and this copy-the-spine-share-the-rest technique is sometimes called *path copying*.)

---

## Definitions

- **Immutable data:** data that cannot be changed after it is constructed. All Gleam values (outside of FFI objects) are immutable; an "update" is the construction of a new value.
- **Structural sharing:** reusing (pointing at) the unchanged parts of an old value inside a newly constructed value, instead of copying them. Safe precisely because the data is immutable.
- **Constructor / variant:** one of the alternative forms of a custom type, e.g. `Last(..)` and `Link(..)` for `NotEmpty(a)`, or `Ok(..)` and `Error(..)` for `Result`.
- **Labelled field:** a named component of a constructor, e.g. `label:` and `branches:` in `Tree(label: a, branches: List(Tree(a)))`. Accessible as `value.field` only if every constructor of the type declares a field with that name and type.
- **Pattern:** a template describing a value's shape that also binds names to its parts, e.g. `[x, ..rest]`, `Tree(label, [Tree(_, []), ..rest])`, `Link(first, Link(second, _))`.
- **Guard:** the `if <condition>` attached to a `case` clause. The clause is taken only if the pattern matches *and* the guard is true; otherwise matching continues with the next clause.
- **Multiple subjects:** the `case x, y { p1, p2 -> ... }` form, which matches a combination of values. Equivalent to `case #(x, y) { #(p1, p2) -> ... }`.
- **Exhaustiveness:** the compiler's requirement that a `case` have a matching clause for every possible value of its subject(s). Satisfied by a catch-all such as `_` or `_, _`.
- **Alternative pattern:** `0 | 1 -> ...`, one clause matching several patterns.
- **`Result(a, e)`:** the built-in type for value-based error handling, with values `Ok(a)` and `Error(e)`.
- **`Nil`:** Gleam's type with exactly one value, used as an `Error` payload meaning "failed, no further detail."
- **Value-based error handling:** reporting failure by returning a value (a `Result`) rather than raising an exception, so the type signature advertises failure and the compiler enforces handling it.
- **External function (FFI):** a function declared in Gleam with `@external(javascript, "<module>", "<name>")` and implemented in the target language. It has a Gleam type signature but no Gleam body.
- **Opaque external type:** `pub type Cache` declared with no constructors, naming a value that only external functions can create or use.

---

## Worked Examples

### Example 1: `matches`, two lists where one is a prefix of the other

Spec: lists `s` and `t` match if they are equal **or one is a prefix of the other**.

```gleam
assert matches([1, 2, 3], [1, 2, 3])
assert matches([1, 2, 1, 2], [1, 2])
assert matches([1, 2], [1, 2, 3, 4])
assert matches([], [1, 2])
assert !matches([1, 2, 3], [1, 3])

fn matches(s: List(a), t: List(a)) -> Bool {
  case s, t {
    [x, ..s_rest], [y, ..t_rest] -> x == y && matches(s_rest, t_rest)
    _, _ -> True
  }
}
```

Step by step:

1. The `case` has two subjects, `s` and `t`, so each clause gives a pattern for each.
2. The first clause fires only when **both** lists are non-empty. It binds `x` and `y` to the two first elements and `s_rest`, `t_rest` to the two tails. It then requires `x == y` and recurses on the tails. The `&&` short-circuits, so on a mismatch the recursion never happens and the answer is `False` immediately.
3. The second clause, `_, _`, matches "any value of `s` and any value of `t`." But because clauses are tried in order, by the time we reach it we already know at least one list is empty. If `s` ran out, `s` is a prefix of `t`; if `t` ran out, `t` is a prefix of `s`; if both ran out, they were equal. In all three situations the answer is `True`, which is why one word suffices.

Trace `matches([1, 2, 1, 2], [1, 2])`: both non-empty, `1 == 1`, recurse on `([2,1,2], [2])`; both non-empty, `2 == 2`, recurse on `([1,2], [])`; `t` is empty so clause 1 cannot match, clause 2 gives `True`.

Trace `matches([1, 2, 3], [1, 3])`: `1 == 1`, recurse on `([2,3], [3])`; `2 == 3` is `False`, short-circuit, return `False`. Correct.

**(extra context)** Python analogue, to make the control flow concrete:

```python
def matches(s, t):
    """Return whether s and t are equal or one is a prefix of the other."""
    if s and t:                       # both non-empty: the first clause
        return s[0] == t[0] and matches(s[1:], t[1:])
    return True                       # the _, _ clause
```

### Example 2: `repeating`, using `matches` plus `drop`

Spec: return whether list `s` is `n` values repeated, so for `n == 3` the list looks like `[x, y, z, x, y, z, x, y, z, x, ...]`.

```gleam
pub fn repeating(s: List(a), n: Int) -> Bool {
  matches(s, drop(s, n))
}

/// Remove up to the first n items of s
fn drop(s: List(a), n: Int) -> List(a) {
  case n {
    0 -> s
    _ ->
      case s {
        [] -> []
        [_, ..rest] -> drop(rest, n - 1)
      }
  }
}
```

The idea is worth remembering because it is clever and short. Shifting `s` left by `n` and comparing it against `s` element by element asks exactly the question "is `s[i] == s[i+n]` for every `i` where both exist?", and that condition is what it means for `s` to repeat with period `n`. `matches` is the right comparison because the shifted list is shorter: it stops as soon as one side runs out, and `drop(s, n)` is always a suffix, hence always at most as long as `s`.

`drop` itself is a double `case`, one on the counter and one on the list:

1. `n == 0`: nothing left to remove, return `s` unchanged (this is the base case, and it is why `drop` shares its result rather than rebuilding it, see Example 5).
2. Otherwise look at `s`. If `s` is empty there is nothing to remove, return `[]`; this is the "up to" in the docstring, so `drop([1, 2], 5)` is `[]` rather than an error.
3. If `s` is `[_, ..rest]`, discard the first element and remove `n - 1` more from `rest`.

Trace `repeating([1, 2, 3, 1, 2, 3, 1], 3)`: `drop` peels three elements to give `[1, 2, 3, 1]`. Then `matches([1,2,3,1,2,3,1], [1,2,3,1])` compares `1=1, 2=2, 3=3, 1=1`, the second list runs out, `True`.

Trace `repeating([1, 2, 3, 1, 2, 4], 3)`: `drop` gives `[1, 2, 4]`. `matches` compares `1=1, 2=2`, then `3 == 4` fails, `False`.

Edge behaviors worth noticing: if `n` is at least the length of `s` then `drop` returns `[]` and `matches` is vacuously `True`; and `repeating(s, 0)` returns `matches(s, s)`, which is always `True`.

### Example 3: `nth` returning a `Result`

Spec: `nth` takes a list `s` and an integer `n` and returns the `n`th item of `s`.

```gleam
fn nth(s: List(a), n: Int) -> Result(a, Nil) {
  case s, n {
    [], _ -> Error(Nil)
    _, _ if n < 0 -> Error(Nil)
    [x, ..], 0 -> Ok(x)
    [_, ..rest], _ -> nth(rest, n - 1)
  }
}
```

Clause by clause:

1. `[], _`: ran off the end of the list, for any `n`. There is no item to return, so `Error(Nil)`. This is where Python would raise `IndexError`; here the failure is an ordinary returned value.
2. `_, _ if n < 0`: an out-of-range index in the other direction. The pattern matches anything, and the guard restricts it to negative `n`. Putting this clause early means the recursion below never has to worry about negative indices.
3. `[x, ..], 0`: the list is non-empty and we have counted down to zero, so `x` is the answer, wrapped as `Ok(x)`. The `..` means "there is a tail and I don't care about it."
4. `[_, ..rest], _`: non-empty list, `n` is positive, so discard the first element and ask for item `n - 1` of the tail.

The slide's question, **"Why didn't we need to write `Ok(...)` or `Error(...)`?"** on the last clause: because `nth(rest, n - 1)` is *already* a `Result(a, Nil)`, which is exactly the declared return type. There is nothing to unwrap and rewrap; the recursive call's result is the answer, success or failure, and passing it straight through is both correct and the only type-correct option. Wrapping it as `Ok(nth(rest, n - 1))` would produce a `Result(Result(a, Nil), Nil)` and fail to compile.

Also notice that the clause order matters for clauses 1 and 3 versus 4, but the catch-all pattern in clause 4 is what makes the whole `case` exhaustive, since the compiler does not reason about guards.

**(extra context)** Python analogue using a tagged return instead of an exception:

```python
def nth(s, n):
    """Return ('ok', item) or ('error', None)."""
    if not s:
        return ('error', None)
    if n < 0:
        return ('error', None)
    if n == 0:
        return ('ok', s[0])
    return nth(s[1:], n - 1)        # pass the result straight through
```

### Example 4: counting calls to `fib` with a mutable counter

```gleam
pub type Counter
@external(javascript, "./fib_ffi.mjs", "new_counter")
pub fn new_counter() -> Counter
@external(javascript, "./fib_ffi.mjs", "increment")
fn increment(counter: Counter) -> Nil
@external(javascript, "./fib_ffi.mjs", "count")
pub fn count(counter: Counter) -> Int

pub fn fib(n: Int, counter: Counter) -> Int {
  increment(counter)
  case n {
    0 | 1 -> n
    _ -> fib(n - 2, counter) + fib(n - 1, counter)
  }
}

pub fn main() {
  let counter = new_counter()
  fib(42, counter)
  echo count(counter)   // echo number of calls in fib(42)
}
```

```javascript
export function new_counter() {
  return { calls: 0 };     // an object with an attribute named "calls" that has value 0
}

export function increment(counter) {
  counter.calls += 1;      // mutation, impossible to write in Gleam itself
}

export function count(counter) {
  return counter.calls;
}
```

How to read this:

- The original `fib` is pure: `case n { 0 | 1 -> n; _ -> fib(n - 2) + fib(n - 1) }`, with `0 | 1` as a single clause matching either base case and returning `n` itself (which is `0` or `1`, conveniently).
- The instrumented version threads a `Counter` through every recursive call, since Gleam has no global mutable state to reach for. The counter is only ever *passed*, never reassigned; all the mutation lives on the JavaScript side.
- The return type stays `Int`: `increment(counter)` is evaluated as a statement purely for its effect (it returns `Nil`), and the `case` expression is the function's value.
- `increment` is called once at the top of every invocation, so `count(counter)` after `fib(42, counter)` is the total number of calls.

**(extra context)** The resulting count: let `C(n)` be the number of calls. Then `C(0) = C(1) = 1` and `C(n) = 1 + C(n-2) + C(n-1)`, which solves to `C(n) = 2 * fib(n+1) - 1`. With `fib(43) = 433494437`, `fib(42)` makes `866988873` calls, which is why this version of `fib` is unusable without memoization (and why the `Cache` example from the previous slide exists).

### Example 5: `remove_leaf`, immutability and shared subtrees

```gleam
pub type Tree(a) {
  Tree(label: a,
       branches: List(Tree(a)))
}

pub fn leaf(label: a) -> Tree(a) {
  Tree(label, [])
}

/// A new tree without its leftmost leaf; a tree that is a leaf is unchanged
pub fn remove_leaf(tree: Tree(a)) -> Tree(a) {
  case tree {
    Tree(label, [Tree(_, []), ..rest]) -> Tree(label, rest)
    Tree(label, [first, ..rest]) -> Tree(label, [remove_leaf(first), ..rest])
    Tree(_, []) -> tree
  }
}
```

The three clauses, in order:

1. **The first branch is a leaf.** The pattern `[Tree(_, []), ..rest]` nests a tree pattern inside a list pattern: the branch list is non-empty, and its first element is a `Tree` whose own `branches` list is empty, i.e. a leaf. That leaf is the leftmost leaf of the whole tree, so the answer is a new `Tree` with the same `label` and with `rest` as its branches. **`rest` is reused as-is**, not rebuilt.
2. **The first branch exists but is not a leaf.** Then the leftmost leaf is somewhere inside `first`, so recurse into it and keep everything else: `Tree(label, [remove_leaf(first), ..rest])`. Again `rest` is the original list, untouched.
3. **The tree itself is a leaf** (`branches` is `[]`), so by the spec it is unchanged: return `tree`, literally the same value.

Order matters between clauses 1 and 2: both patterns match a non-empty branch list, and clause 1 is the more specific one, so it must come first. Clause 3 is disjoint from both (it is the only one matching an empty branch list), so its position is harmless.

Now the lecture's tree, and the "Never copied!" annotation. Start with

```gleam
Tree(1, [
  leaf(2),
  Tree(3, [leaf(2), leaf(4)]),
  Tree(2, [leaf(5)])])
```

**First call to `remove_leaf`.** The first branch is `leaf(2)`, i.e. `Tree(2, [])`, so clause 1 fires. The result is `Tree(1, rest)` where `rest = [Tree(3, [leaf(2), leaf(4)]), Tree(2, [leaf(5)])]`.

Box-and-pointer reasoning in words: exactly **one** new `Tree` node is allocated (the new root labelled `1`). Its `branches` pointer points at the *same* list cells that the original root's branch list used from its second element onward, and those cells point at the *same* `Tree(3, ...)` and `Tree(2, [leaf(5)])` objects the original tree contains. The original tree still exists and is still correct; the two trees overlap everywhere except the root.

**Second call, applied to that result.** Now the first branch is `Tree(3, [leaf(2), leaf(4)])`, whose `branches` list is *not* empty, so clause 1 does not match and clause 2 fires:

```gleam
Tree(1, [remove_leaf(Tree(3, [leaf(2), leaf(4)])), Tree(2, [leaf(5)])])
```

The inner call sees `Tree(3, [leaf(2), leaf(4)])`, whose first branch *is* a leaf, so clause 1 gives `Tree(3, [leaf(4)])`. The final answer is

```gleam
Tree(1, [Tree(3, [leaf(4)]), Tree(2, [leaf(5)])])
```

New nodes allocated by this second call: the root labelled `1` and the node labelled `3`, that is, exactly the nodes on the path from the root to the removed leaf, plus one new list cell for the rebuilt branch list of each. The `leaf(4)` and the whole `Tree(2, [leaf(5)])` subtree are **shared** with the input, which is what the arrow labelled "Never copied!" is pointing out.

The general cost: `remove_leaf` allocates O(depth of the leftmost leaf) nodes, independent of how big the untouched subtrees are. Because the data is immutable, nobody can ever observe the sharing, so this is a correct implementation of "a new tree without its leftmost leaf."

**(extra context)** Why Python cannot be this casual. If you wrote the same code with a mutable Python tree class and shared `rest` into a new tree, a later in-place change to a shared subtree (`t.branches[0].label = 99`) would be visible through both trees. Immutability is what converts aliasing from a bug into an optimization. The equivalent pure-Python version, where sharing is also safe because nothing is ever mutated:

```python
class Tree:
    def __init__(self, label, branches=()):
        self.label = label
        self.branches = tuple(branches)   # immutable by convention

def remove_leaf(t):
    """Return a new tree without its leftmost leaf; a leaf is unchanged."""
    if not t.branches:                                    # Tree(_, [])
        return t
    first, rest = t.branches[0], t.branches[1:]
    if not first.branches:                                # [Tree(_, []), ..rest]
        return Tree(t.label, rest)                        # rest shared, not copied
    return Tree(t.label, (remove_leaf(first),) + rest)    # rest shared, not copied
```

---

## Common Pitfalls

- **Assuming a compiling `case` is a correct `case`.** The `increasing` bug compiles and passes exhaustiveness checking. Whenever you write a nested pattern like `Link(first, Link(second, _))`, ask explicitly which shapes of the inner value it *excludes*, and check that the clause catching the leftovers does the right thing for them.
- **Forgetting that a failed guard falls through.** `Link(first, Link(second, _)) if second <= first -> False` does not mean "when the pattern matches, decide here." If the guard is false, a later catch-all runs instead.
- **Using `.field` on a type where not all constructors have that field.** `s.first` is fine for `NotEmpty(a)`; `s.rest` is a compile error because `Last` has no `rest`.
- **Getting clause order backwards.** In `remove_leaf`, putting `Tree(label, [first, ..rest])` before `Tree(label, [Tree(_, []), ..rest])` makes the leaf case unreachable and recurses into a leaf. In `nth`, putting `[_, ..rest], _` before `[x, ..], 0` would recurse forever past the target.
- **Mismatching the number of subjects and patterns.** With `case s, t`, every clause needs two comma-separated patterns. One pattern (or three) is an error, and `case #(s, t)` needs `#(..., ...)` patterns instead.
- **Double-wrapping a `Result`.** When a recursive call already returns `Result(a, Nil)`, return it directly. Writing `Ok(nth(rest, n - 1))` is a type error.
- **Expecting Gleam to raise on bad input.** There is no `IndexError` to catch. If a function can fail, its type says `Result(...)` and the compiler makes you handle both constructors; if the type says `Int`, it does not fail.
- **In `remove_leaf`, recursing into every branch.** Mapping `remove_leaf` over all of `branches` would remove a leaf from each branch. The spec removes exactly one leaf, so only `first` is touched and `rest` is passed through unchanged.
- **Thinking structural sharing is an optimization you must request.** It is the natural consequence of constructing new values out of old pieces. The mistake is the opposite one: defensively copying subtrees that nobody can mutate.
- **Assuming FFI mutation is "normal Gleam."** It is deliberately outside the type system's guarantees. `increment` looks like a pure call returning `Nil` and the compiler cannot warn you that it changes the world.

---

## Likely Exam Points

### 1. Spot the bug in a pattern-matching function

**Q.** For the `NotEmpty` type above, what does `increasing(Link(1, Link(2, Last(0))))` return, and is that correct?

**A.** It returns `True`, which is wrong (`0` is less than `2`). Clause 2 is tried with `first = 1` and `rest = Link(2, Last(0))`: the pattern matches with `second = 2`, but the guard `2 <= 1` is false, so matching continues. Clause 3 recurses with `Link(2, Last(0))`. Now clause 2's pattern needs the tail to be a `Link`, but it is `Last(0)`, so clause 3 fires again and calls `increasing(Last(0))`, which returns `True` by clause 1. The comparison between `2` and `0` never happens.

### 2. Translate between multiple subjects and tuple patterns

**Q.** Rewrite this with a single subject.

```gleam
case s, n {
  [], _ -> Error(Nil)
  [x, ..], 0 -> Ok(x)
  [_, ..rest], _ -> nth(rest, n - 1)
}
```

**A.**

```gleam
case #(s, n) {
  #([], _) -> Error(Nil)
  #([x, ..], 0) -> Ok(x)
  #([_, ..rest], _) -> nth(rest, n - 1)
}
```

The two forms are exactly equivalent; multiple subjects are shorthand for matching a tuple.

### 3. Fill in a blank in a list recursion

**Q.** Complete `matches`, which returns whether `s` and `t` are equal or one is a prefix of the other, given that the second clause is `_, _ -> True`.

**A.** `[x, ..s_rest], [y, ..t_rest] -> x == y && matches(s_rest, t_rest)`. Both lists non-empty is the only case that needs work; once either is exhausted, the prefix condition already holds, so `True` is correct.

### 4. Reason about a helper used in a clever way

**Q.** Using `matches`, why does `matches(s, drop(s, n))` correctly test whether `s` is `n` values repeated? What does `repeating(s, 0)` return?

**A.** `drop(s, n)` is `s` shifted left by `n`, so comparing it element by element against `s` checks `s[i] == s[i+n]` for every index where both exist, which is exactly `n`-periodicity. `matches` is the right comparison because the shifted list is a suffix and therefore never longer, so running out is a success rather than a failure. `repeating(s, 0)` is `matches(s, s)`, which is always `True`.

### 5. `Result` handling and why no wrapping is needed

**Q.** In `nth`, the last clause is `[_, ..rest], _ -> nth(rest, n - 1)`. Why is no `Ok` or `Error` written there, and what happens if you write `Ok(nth(rest, n - 1))`?

**A.** The recursive call's type is already `Result(a, Nil)`, matching the declared return type, so it is returned unchanged and carries whichever of `Ok`/`Error` the deeper call produced. Wrapping it would give `Result(Result(a, Nil), Nil)` and fail to compile.

### 6. Writing a function with a `Result` return type

**Q.** What does `list.try_map(["4", "x", "6"], int.parse)` evaluate to, and what is its type?

**A.** `Error(Nil)`, of type `Result(List(Int), Nil)`. `try_map` stops at the first element whose mapping fails and returns that error; `"x"` is not parseable as an `Int`. With `["4", "5", "6"]` it would be `Ok([4, 5, 6])`.

### 7. Counting newly allocated nodes under structural sharing

**Q.** Let `t = Tree(1, [leaf(2), Tree(3, [leaf(2), leaf(4)]), Tree(2, [leaf(5)])])`. How many new `Tree` nodes does `remove_leaf(remove_leaf(t))` allocate, and which parts of the original are shared?

**A.** Three new `Tree` nodes in total. The first `remove_leaf(t)` matches clause 1 (its first branch is a leaf) and allocates one new node, the root labelled `1`, sharing `Tree(3, [leaf(2), leaf(4)])` and `Tree(2, [leaf(5))]`. The second call matches clause 2 and allocates a new root labelled `1` and, via the recursive call, a new node labelled `3`; `leaf(4)` and all of `Tree(2, [leaf(5)])` are shared with the original. In general only the nodes on the path from the root to the removed leaf are rebuilt.

### 8. Why sharing is safe

**Q.** Why can `remove_leaf` return a tree that points at subtrees of its argument instead of copying them?

**A.** Because Gleam data is immutable: no code can later change a shared subtree, so a program cannot distinguish a shared subtree from a copy. The old tree also remains valid and correct, so both versions can coexist. In a language with mutation, sharing would be observable and a later in-place change would corrupt both trees.

### 9. External functions and mutation

**Q.** Gleam data is immutable, yet the lecture builds a mutable cache. How, and what does `pub type Cache` with no constructors mean?

**A.** Gleam compiles to JavaScript, and `@external(javascript, "./fib_ffi.mjs", "store")` declares that `store`'s implementation is a JavaScript function, which can mutate a JavaScript `Map` and return Gleam values (the JS side imports `Ok` and `Error` from the compiled Gleam runtime). `pub type Cache` with no constructors is an opaque external type: Gleam type-checks values of that type but no Gleam code can construct or inspect one, only the external functions can.

### 10. Threading state without mutable globals

**Q.** Why does the instrumented `fib` take `counter` as an extra parameter and pass it to both recursive calls?

**A.** Gleam has no mutable global variables and no reassignment, so the only way to reach the shared counter is to pass it explicitly down the recursion. Every invocation calls `increment(counter)` once, which mutates the JavaScript object, so after `fib(42, counter)` the value of `count(counter)` is the total number of `fib` calls.

---

## Summary

- `NotEmpty(a)` with `Last(first)` and `Link(first, rest)` is a linked list that cannot be empty; `s.first` compiles because *both* constructors have a `first` field, while `s.rest` would not.
- The `increasing` bug: `Link(first, Link(second, _))` never matches when the tail is a `Last`, so the final adjacent pair is never compared and the catch-all clause silently returns `True`. Exhaustiveness checking cannot catch this; a failed guard also falls through to later clauses.
- `case x, y { p, q -> ... }` matches combinations of values and is exactly equivalent to `case #(x, y) { #(p, q) -> ... }`. Clauses are tried top to bottom, so specific patterns precede catch-alls.
- `matches(s, t)`: compare heads and recurse while both lists are non-empty; once either runs out, one is a prefix of the other, so `_, _ -> True`.
- `repeating(s, n)` is `matches(s, drop(s, n))`: comparing `s` against itself shifted left by `n` is exactly the test for `n`-periodicity. `drop` removes *up to* the first `n` items, returning `[]` if the list runs out first.
- `Result(a, e)`, with `Ok(..)` and `Error(..)`, is Gleam's value-based error handling. Unlike Python's optional exception handling, a Gleam program will not run until the compiler confirms every error case is handled. `Nil` is the "failed, no detail" payload.
- Create results with `Ok(...)`/`Error(...)`; consume them with a `case` covering both. Standard library helpers like `list.try_map` (with `int.parse`) return all successes as `Ok(list)` or the first failure as `Error`.
- `nth` returns `Result(a, Nil)`: `Error(Nil)` for an empty list or negative `n`, `Ok(x)` at `n == 0`, and the recursive call passed straight through, since it is already a `Result` and needs no rewrapping.
- `@external(javascript, module, name)` declares a Gleam function implemented in JavaScript; with `pub type Cache` (an opaque type with no constructors) this yields genuinely mutable objects, e.g. a `Map` cache or a `{ calls: 0 }` counter threaded through `fib` to count its calls.
- Immutability makes **structural sharing** safe and automatic: `remove_leaf` rebuilds only the nodes on the path to the leftmost leaf (`Tree(label, rest)` or `Tree(label, [remove_leaf(first), ..rest])`) and shares every other subtree, so the untouched parts are "never copied" and the original value stays valid.
