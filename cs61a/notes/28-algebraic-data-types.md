<!-- Fri, Oct 30, 2026 | sources: slides (no transcript available) -->
# Lecture 28: Algebraic Data Types

## Overview

This lecture finishes the introduction to Gleam's list data type and then introduces **algebraic data types** (ADTs): custom types defined as a set of named **variants**, each carrying its own **fields**. Gleam lists are linked lists built from immutable cells, so there is no `.first` or `.rest` accessor and no mutation; instead you take lists apart with `case` **patterns**, which both test the shape of a list and give local names to its parts. The list-processing examples (`alternating`, `repeating`/`nth_equals`, `matches`, `drop`) show the pattern-matching style of recursion: your base cases come from the patterns themselves rather than from `if` statements. The ADT half of the lecture shows how to define your own types with multiple variants (`Contact` with `EmailOnly`, `PhoneOnly`, `Both`), how patterns name variant fields, when a dot expression like `c.email` is legal, and how recursive, parameterized ADTs let you express data that a homogeneous list cannot express on its own (`NestedList(a)` for nested lists) or that rules out bad values by construction (`NotEmpty(a)` for non-empty lists). It closes with a debugging exercise on `increasing` over `NotEmpty(Int)`.

> Note on code: all of this lecture's code is in **Gleam**, so the worked examples below are the lecture's Gleam code. Where a Python translation helps build intuition (using the CS 61A `Link` class or Python lists), it is included and marked **(extra context)**.

---

## Key Concepts

### 1. Gleam lists are linked lists

A list literal looks like a Python list:

```gleam
[3, 7, 5, 9]
```

but it behaves like the `Link` linked list from earlier in the course: a chain of cells, each holding one item and a reference to the rest of the list. All values in Gleam are immutable, so a list can never be changed after it is built.

A list literal can also contain an expression for *the rest of the list*, using `..`:

```gleam
let s = [5, 7, 9]
let odds = [1, 3, ..s]   // [1, 3, 5, 7, 9], sharing memory with s
```

The `..` spread has three restrictions stated on the slide: it can appear **only once**, **only at the end** of the literal, and **only before a list** (the thing after `..` must itself be a list).

**Box-and-pointer reasoning in words.** `s` is a chain of three cells: `5 -> 7 -> 9 -> (end)`. Evaluating `[1, 3, ..s]` allocates exactly **two** new cells: a cell holding `1` whose rest points at a cell holding `3`, whose rest points at *the very same first cell of `s`*. Nothing is copied. `odds` and `s` share their last three cells. This sharing is completely safe precisely because values are immutable: no one can later change `s`'s cells and corrupt `odds`.

This also explains the "only at the end" rule **(extra context)**: adding to the front of a linked list is cheap (one new cell), while adding to the end would require rebuilding the whole chain, so Gleam's literal syntax only supports the cheap operation.

### 2. Lists are homogeneous, and the type `List(a)` says so

Every item in a Gleam list must have the same type. All of these are type mismatch errors:

```gleam
[1, "two"]        // Int and String in the same list
["one", ..s]      // s is a List(Int), so its items are Int, not String
[1, 3, s]         // s is a list; the other items are Ints
```

The last one is the subtle one: `[1, 3, s]` tries to put a *list* in as a third item next to two integers. `[1, 3, ..s]` is what you meant.

A list type names the type of the items it contains: `List(Int)` is a list of integers, `List(String)` a list of strings, `List(List(Int))` a list of lists of integers. A lowercase name like `a` in `List(a)` is a **type variable**: the function works for lists of any single item type (parametric polymorphism).

```gleam
import gleam/list

fn double_each(s: List(Int)) -> List(Int) {
  list.map(s, fn(n) { 2 * n })
}
```

The `gleam/list` module (the lecture points at the stdlib docs) contains `map`, `filter`, `all`, `any`, `append`, `zip`, and more, so many list problems do not need hand-written recursion.

### 3. There is no `.first` or `.rest`: use `case` patterns

Because there are no list accessors, a `case` expression does double duty: it **tests the shape** of the list and **binds names** to its parts.

```gleam
fn describe(s: List(Int)) -> String {
  case s {
    [] -> "Empty list"
    [1] -> "List of just 1"
    [4, ..] -> "List starting with 4"
    [a, b] if a < b -> "List of " <> int.to_string(a) <> " then " <> int.to_string(b)
    [_, _] -> "List of two elements"
    [a, b, ..rest] if a < b -> "List ending in " <> string.inspect(rest)
    _ -> "Some other list"
  }
}
```

Reading the pattern vocabulary off this example:

- `[]` matches only the empty list.
- `[1]` matches a list of exactly one item whose value is `1`. A literal inside a pattern is a *test*, not a binding.
- `[a, b]` matches a list of exactly two items and names them `a` and `b`. A lowercase name inside a pattern is a *binding*: `a` is now a name for the first item of `s`.
- `_` is a wildcard: matches anything, binds nothing. `[_, _]` means "exactly two items, I don't care what."
- `[4, ..]` matches any list whose first item is `4`, with any (possibly empty) rest that is discarded.
- `[a, b, ..rest]` matches a list of **two or more** items, naming the first two and naming `rest` for everything after them.
- `if <condition>` after a pattern is a **guard**: the branch is taken only if the pattern matches *and* the guard is true.
- `_` alone at the end is the catch-all.

**Branch order matters: the first matching branch wins.** `describe([4, 9])` returns `"List starting with 4"`, not `"List of 4 then 9"`, even though the `[a, b] if a < b` pattern would also have matched, because `[4, ..]` comes first.

**Exhaustiveness (extra context).** Gleam's compiler requires a `case` to cover every possible value of the subject; a missing case is a compile-time error, not a runtime surprise like Python returning `None`. That is why these examples end in a catch-all, and it is also why guards alone are never enough to satisfy the compiler (a guard can fail, so a guarded branch does not count as covering its pattern).

Two more pattern features used later in the lecture:

- **Alternation** with `|`: `[] | [_] | [_, _] -> True` takes the branch if *any* of the alternatives matches. If the alternatives bind names, every alternative must bind the same names with the same types (as in `EmailOnly(e) | Both(e, _)`).
- **Multiple subjects**: `case s, t { ... }` matches two values at once, with each branch giving one pattern per subject, separated by commas.

### 4. Comparison and boolean operators in Gleam

- Two values of the same type can always be compared with `==` and `!=`, and this is **structural** equality: `[1, 2] == [1, 2]` is true regardless of whether the two lists share memory.
- There is **no equivalent of Python's `is`**, because all values are immutable, so identity is not observable the way it is for mutable Python lists.
- `&&` is `and`, `||` is `or`, `!` is `not`. Like Python's `and`/`or`, `&&` and `||` short-circuit **(extra context, and relied on by `repeating` and `matches` to avoid work after a failure)**.
- `<>` concatenates strings; `int.to_string` and `string.inspect` convert values to strings.

### 5. Algebraic data types: variants and fields

A custom type is declared with a name and a list of **variants**, each variant being a constructor with named fields:

```gleam
pub type Contact {
  EmailOnly(email: String)
  PhoneOnly(phone: Int)
  Both(email: String, phone: Int)
}

pub type User {
  User(name: String, contact: Contact)
}
```

A value of type `Contact` is *one of* these three shapes. A value of type `User` always has the single shape `User(name, contact)`. Type names and variant names are capitalized; function and variable names are lowercase.

Why "algebraic" **(extra context)**: a type like `Contact` is a **sum** ("or") of its variants, and each variant is a **product** ("and") of its fields. The number of possible values is the sum over variants of the product of the field possibilities. This is just terminology, but it explains the lecture title: you build types by combining "or" and "and".

The payoff is that a `case` over an ADT handles each shape separately and gives local names to that shape's fields:

```gleam
pub fn contact(user: User) -> String {
  case user.contact {
    EmailOnly(e) | Both(e, _) -> "mailto:" <> e
    PhoneOnly(n) -> "tel:" <> int.to_string(n)
  }
}
```

`EmailOnly(e) | Both(e, _)` is the alternation pattern: both alternatives bind a `String` named `e`, so the branch can use `e` whichever variant arrived. `Both`'s phone field is discarded with `_`.

### 6. When a dot expression is allowed

`user.contact` is fine: `User` has exactly one variant, and that variant has a `contact` field, so the field is guaranteed to exist.

`user.contact.email` is **not allowed**, because `user.contact` has type `Contact` and a `Contact` may not have an `email` attribute (it could be `PhoneOnly`). The general rule: you may access a field through a dot expression only when the compiler is sure the field is there.

A pattern match can supply that certainty. After matching a specific variant, you can bind the whole value with `as` and then use dot expressions on it:

```gleam
pub fn contact(user: User) -> String {
  case user.contact {
    EmailOnly(..) as c -> "mailto:" <> c.email
    Both(..) as c -> "mailto:" <> c.email
    PhoneOnly(..) as c -> "tel:" <> int.to_string(c.phone)
  }
}
```

Here `(..)` inside a record pattern means "I don't care about any of the fields," and `as c` names the matched value. Because the branch is reached only when the value is an `EmailOnly`, `c.email` is known to exist. Note that `EmailOnly` and `Both` have to be written as two separate branches in this version even though the bodies are identical: combining them with `|` would leave the compiler unsure which variant `c` is **(extra context on the reason; the slide just shows them split)**.

The other direction of this rule shows up in `NotEmpty` below: `s.first` *is* allowed for `s: NotEmpty(a)` without any pattern match, because **every** variant of `NotEmpty` has a `first` field of type `a`.

### 7. Type aliases, parameterized types, and recursive types

```gleam
pub type NestedList(a) =
  List(Nested(a))

pub type Nested(a) {
  Sublist(NestedList(a))
  Item(a)
}
```

Two different declarations are at work here:

- `pub type NestedList(a) = List(Nested(a))` with an `=` is a **type alias**. It introduces no new constructors and no new values; `NestedList(Int)` is just another way to write `List(Nested(Int))`.
- `pub type Nested(a) { ... }` with a block is a **new custom type** with two constructors, `Sublist` and `Item`.

Both are **parameterized** by a type variable `a`, and the definitions are **mutually recursive**: a `NestedList(a)` is a list of `Nested(a)`, and a `Sublist` holds a `NestedList(a)`. That recursion is exactly the English definition on the slide: "a nested list of integers is a list containing integers and nested lists of integers."

**Why an ADT is needed at all here.** In Python you can simply write `[1, [2, 3], 4]`, because a Python list can hold anything, and you distinguish the cases at runtime with `isinstance(x, list)`. A Gleam list must be homogeneous, so `[Item(1), Sublist([Item(2), Item(3)]), Item(4)]` is how you say it: every element has the same type, `Nested(Int)`, and the *tag* (`Item` vs `Sublist`) carries the information `isinstance` would have recovered. Tagging replaces runtime type inspection.

The `NotEmpty` type illustrates the other classic use of ADTs, making bad values impossible to construct:

```gleam
pub type NotEmpty(a) {
  Last(first: a)
  Link(first: a, rest: NotEmpty(a))
}
```

A `NotEmpty(a)` is either a final item (`Last`) or an item followed by another non-empty list (`Link`). There is no empty case, so

```gleam
pub fn starts_with(s: NotEmpty(a), v: a) -> Bool {
  s.first == v
}
```

needs no empty check at all, and `s.first` type-checks without a `case` because both variants have `first`.

---

## Definitions

- **Algebraic data type (ADT)**: a type defined as a fixed set of named variants, each variant holding a fixed set of typed fields. A value of the type is exactly one variant.
- **Variant (constructor)**: one of the alternative shapes of an ADT, written with a capitalized name, for example `EmailOnly(email: String)`. Used both to construct values and inside patterns to match them.
- **Field (attribute)**: a named, typed component of a variant, for example `email` in `EmailOnly(email: String)`.
- **Sum type (extra context)**: a type formed by an "or" of variants (`Contact` is `EmailOnly` or `PhoneOnly` or `Both`).
- **Product type (extra context)**: a shape formed by an "and" of fields (`Both` is an email *and* a phone).
- **Type alias**: a declaration of the form `pub type Name(a) = SomeExistingType`, giving a new name to an existing type without creating new constructors.
- **Type variable**: a lowercase name in a type, such as the `a` in `List(a)` or `NotEmpty(a)`, standing for any one type chosen by the caller.
- **Linked list**: a list represented as a chain of cells, each holding one item and a reference to the rest. Gleam's `List` is a linked list.
- **List literal**: `[e1, e2, ...]`, optionally ending with `..rest_expression`, which supplies the rest of the list and shares its memory.
- **Spread (`..`) in a literal**: the rest-of-list form; allowed only once, only at the end of the literal, and only in front of a value that is itself a list.
- **Pattern**: a shape written in a `case` branch that matches certain values and binds names to their parts. Literals in a pattern test for equality; lowercase names bind; `_` matches and discards; `..` or `..name` in a list pattern stands for the remaining items.
- **Guard**: an `if condition` attached to a pattern; the branch is taken only when the pattern matches and the condition is true.
- **Alternation pattern**: `p1 | p2 | p3`, matching if any alternative matches; all alternatives must bind the same names with the same types.
- **`as` pattern**: `Variant(..) as c`, which matches and also binds the whole matched value to `c`, enabling dot access to that variant's fields.
- **Multiple subjects**: `case s, t { pa, pb -> ... }`, matching several values simultaneously with one pattern per subject.
- **Exhaustiveness (extra context)**: the requirement that a `case` cover all possible values of its subjects; Gleam enforces it at compile time.
- **Structural equality**: `==` compares values by their contents and shape. Gleam has no identity comparison (no `is`) because all values are immutable.
- **Prefix**: list `t` is a prefix of `s` if `s` begins with all of `t`'s items in order.
- **Matching lists** (this lecture's definition): `s` and `t` *match* if they are equal or one is a prefix of the other.
- **Alternating list**: a list of the form `[x, y, x, y, x, ...]`, that is, every item equals the item two positions earlier.
- **Repeating list with period `n`**: a list of the form `[x1, ..., xn, x1, ..., xn, x1, ...]`, that is, every item equals the item `n` positions earlier.
- **Nested list**: a list whose elements are either items or nested lists; in Gleam, `NestedList(a) = List(Nested(a))` with `Nested(a)` being `Item(a)` or `Sublist(NestedList(a))`.

---

## Worked Examples

### Example 1: Alternating lists

Goal: return whether `s` has the form `[x, y, x, y, x, ...]`.

```gleam
fn alternating(s: List(a)) -> Bool {
  case s {
    [] | [_] | [_, _] -> True
    [x, _, z, ..] if x != z -> False
    [_, ..rest] -> alternating(rest)
  }
}
```

Tests from the slide:

```gleam
assert alternating([])
assert alternating([1, 2, 1, 2, 1])
assert alternating(["a", "b", "a", "b"])
assert !alternating([1, 2, 1, 3])
assert !alternating([1, 2, 2, 1])
```

How each branch earns its place:

1. **Base case `[] | [_] | [_, _] -> True`.** The condition "every item equals the item two before it" is vacuously true for lists of length 0, 1, and 2, since no item has an item two positions earlier. Alternation `|` collapses three base cases into one branch.
2. **Failure case `[x, _, z, ..] if x != z -> False`.** This pattern requires at least three items and names the first (`x`) and the third (`z`), discarding the second and any tail. If `x != z`, the alternating property is already broken, so return `False` immediately.
3. **Recursive case `[_, ..rest] -> alternating(rest)`.** Reached only when the list has three or more items and `x == z`. Having verified the first constraint, drop the first item and check the rest. Each recursive call checks one more adjacent-but-one pair, and the list shrinks by one, so it terminates in a base case.

Note the type: `List(a)`, not `List(Int)`. The function works on lists of any type because Gleam lets you compare any two values of the same type with `!=`. That is why `alternating(["a", "b", "a", "b"])` type-checks.

**Trace `alternating([1, 2, 1, 3])`:**

| call | branch 1? | branch 2? | action |
|---|---|---|---|
| `[1, 2, 1, 3]` | no (length 4) | `x=1, z=1`, guard `1 != 1` false, no | recurse on `[2, 1, 3]` |
| `[2, 1, 3]` | no (length 3) | `x=2, z=3`, guard `2 != 3` true | return `False` |

**Trace `alternating([1, 2, 1, 2, 1])`:** `x=1,z=1` so recurse on `[2,1,2,1]`; `x=2,z=2` so recurse on `[1,2,1]`; `x=1,z=1` so recurse on `[2,1]`, which matches `[_, _]` and returns `True`.

**Trace `alternating([1, 2, 2, 1])`:** `x=1, z=2`, guard `1 != 2` is true, so `False` right away.

**Python translation (extra context)**, with the 61A `Link` class, to show what the patterns are doing:

```python
class Link:
    empty = ()
    def __init__(self, first, rest=empty):
        self.first = first
        self.rest = rest

def alternating(s):
    if s is Link.empty or s.rest is Link.empty or s.rest.rest is Link.empty:
        return True          # the [] | [_] | [_, _] branch
    if s.first != s.rest.rest.first:
        return False         # the [x, _, z, ..] if x != z branch
    return alternating(s.rest)
```

The Gleam version needs no `s.rest.rest.first` chain and cannot raise an attribute error on a short list, because the pattern `[x, _, z, ..]` already guarantees three items exist before `x` and `z` are ever evaluated. That safety is the main practical argument for pattern matching.

### Example 2: Repeating lists (direct recursion)

Goal: return whether `s` is `n` values repeated, for example for `n == 3`, `[x, y, z, x, y, z, x, y, z, x, ...]`. Equivalently: every item equals the item `n` positions earlier.

```gleam
pub fn repeating(s: List(a), n: Int) -> Bool {
  case s {
    [] -> True
    [first, ..rest] -> nth_equals(first, s, n) && repeating(rest, n)
  }
}

/// Return True if s is shorter than n or s[n] == expected
pub fn nth_equals(expected: a, s: List(a), n: Int) -> Bool {
  case s {
    [] -> True
    [first, ..rest] ->
      case n {
        0 -> first == expected
        _ -> nth_equals(expected, rest, n - 1)
      }
  }
}
```

Reading `repeating`:

- `[] -> True`: an empty list is trivially periodic.
- `[first, ..rest]`: name the first item and the rest. The constraint for this position is "the item `n` later equals `first`," which is exactly `nth_equals(first, s, n)`. Then the same must hold for every later position, which is `repeating(rest, n)`.
- Note that `nth_equals` is given **`s`**, not `rest`. Index `n` of `s` is index `n - 1` of `rest`, so passing the whole list with `n` keeps the indexing honest: `nth_equals(first, s, n)` asks about `s[n]` compared to `s[0]`.
- `&&` short-circuits, so once one position fails, no further recursion happens.

Reading `nth_equals(expected, s, n)`:

- Two nested `case` expressions: the outer one on the list, the inner one on the counter.
- `[] -> True`: we ran off the end of the list before reaching index `n`. There is no item at that index, so there is no constraint to violate; returning `True` is what makes the function total. This is why the doc comment says "return True if `s` is shorter than `n`."
- `n == 0`: we have arrived; compare `first == expected`.
- otherwise: walk one step forward and decrement, `nth_equals(expected, rest, n - 1)`.

**Trace `repeating([1, 2, 1, 2, 1], 2)`:**

| `s` in `repeating` | check | result |
|---|---|---|
| `[1,2,1,2,1]` | `nth_equals(1, [1,2,1,2,1], 2)` walks to `[1,2,1]` with `n=0`, compares `1 == 1` | `True`, continue |
| `[2,1,2,1]` | `nth_equals(2, [2,1,2,1], 2)` reaches `2 == 2` | `True`, continue |
| `[1,2,1]` | `nth_equals(1, [1,2,1], 2)` reaches `1 == 1` | `True`, continue |
| `[2,1]` | `nth_equals(2, [2,1], 2)` runs off the end (`[]`) | `True`, continue |
| `[1]` | `nth_equals(1, [1], 2)` runs off the end | `True`, continue |
| `[]` | base case | `True` |

So the answer is `True`. If instead the list were `[1, 2, 1, 3, 1]`, the second position would compare `s[1] = 2` with `s[3] = 3` and return `False`, which short-circuits the whole thing.

**Edge cases worth knowing (extra context).** With `n == 0`, `nth_equals(first, s, 0)` compares `first == first`, so `repeating(s, 0)` is `True` for every list. With a negative `n`, the counter never reaches `0`, so `nth_equals` walks off the end and returns `True`, again making `repeating` vacuously `True`. Neither is in the slide's test cases, but it is worth knowing that the base case `[] -> True` is what absorbs these.

### Example 3: Matching lists

Definition from the slide: lists `s` and `t` *match* if they are equal or one is a prefix of the other.

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

This is the lecture's demonstration of **multiple subjects**: `case s, t` with two patterns per branch.

- First branch: both lists are non-empty. Name both first items and both rests. The lists match iff the first items are equal and the rests match. `&&` short-circuits, so a mismatch stops the recursion immediately.
- Second branch, `_, _ -> True`: reached only when the first branch failed to match, which happens exactly when `s` is empty or `t` is empty. If either has run out, everything compared so far was equal, so the shorter one is a prefix of the longer one, and the answer is `True`. The slide's annotation is the key point: "this pattern matches any value of `s` and any value of `t`."

Branch **order** is what makes the catch-all correct. If you wrote `_, _ -> True` first, it would match everything and `matches` would always return `True`.

**Trace `matches([1, 2, 1, 2], [1, 2])`:** compare `1 == 1`, recurse on `([2, 1, 2], [2])`; compare `2 == 2`, recurse on `([1, 2], [])`; the first branch cannot match because `t` is empty, so `_, _ -> True`. Result: `True` (`[1, 2]` is a prefix of `[1, 2, 1, 2]`).

**Trace `matches([1, 2, 3], [1, 3])`:** compare `1 == 1`, recurse on `([2, 3], [3])`; compare `2 == 3`, which is `False`, so `&&` short-circuits and the result is `False`.

**Python translation (extra context):**

```python
def matches(s, t):
    if s is Link.empty or t is Link.empty:
        return True
    return s.first == t.first and matches(s.rest, t.rest)
```

### Example 4: Repeating lists using matching, plus `drop`

The elegant reformulation: a list has period `n` exactly when it matches itself with the first `n` items removed.

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

Why this works: line up `s` against `drop(s, n)`.

```
s            : [x, y, z, x, y, z, x, ...]
drop(s, 3)   : [x, y, z, x, ...]
```

Position `i` of `drop(s, n)` is position `i + n` of `s`. So `matches` comparing them item by item is exactly the test "`s[i] == s[i+n]` for every `i` where both exist," and `matches` stops with `True` when the shorter list (always `drop(s, n)`) runs out, which is exactly the right place to stop. That is why `matches` was defined to accept prefixes rather than requiring equal lengths: a strict equality check would wrongly reject every finite periodic list.

Reading `drop`:

- Outer `case` is on the **counter**, not the list: `0 -> s` means "done, nothing more to remove."
- Inner `case` on the list: `[] -> []` makes `drop` total, handling `n` larger than the length ("remove **up to** the first `n` items," as the doc comment carefully says). `[_, ..rest] -> drop(rest, n - 1)` discards one item and continues.
- `drop` allocates nothing: it returns an existing tail of `s`, sharing memory. Walking down a linked list and returning a suffix is free.

**Trace `repeating([1, 2, 3, 1, 2, 3, 1], 3)`:** `drop(s, 3)` walks past `1, 2, 3` and returns the suffix `[1, 2, 3, 1]`. Then `matches([1,2,3,1,2,3,1], [1,2,3,1])` compares `1=1, 2=2, 3=3, 1=1`, then `t` is empty, so `True`.

**Trace `repeating([1, 2, 3, 1, 2, 4], 3)`:** `drop` gives `[1, 2, 4]`; `matches` compares `1=1`, `2=2`, then `3` against `4`, returning `False`.

### Example 5: Contacts, an ADT with three variants

```gleam
pub type Contact {
  EmailOnly(email: String)
  PhoneOnly(phone: Int)
  Both(email: String, phone: Int)
}

pub type User {
  User(name: String, contact: Contact)
}

pub fn contact(user: User) -> String {
  case user.contact {
    EmailOnly(e) | Both(e, _) -> "mailto:" <> e
    PhoneOnly(n) -> "tel:" <> int.to_string(n)
  }
}
```

Step by step on the slide's example, `contact(User("Oski", EmailOnly("oski@berkeley.edu")))`:

1. `User("Oski", EmailOnly("oski@berkeley.edu"))` constructs a `User` value with `name = "Oski"` and `contact = EmailOnly("oski@berkeley.edu")`. Think of it as two nested boxes: a `User` box with two slots, whose second slot holds an `EmailOnly` box with one slot holding the string.
2. Inside `contact`, `user.contact` is a legal dot expression because `User` has one variant and that variant has a `contact` field. It evaluates to the `EmailOnly(...)` value.
3. The `case` tries the first branch. The alternation `EmailOnly(e) | Both(e, _)` matches on its first alternative, binding `e` to `"oski@berkeley.edu"`.
4. The body evaluates `"mailto:" <> e`, giving `"mailto:oski@berkeley.edu"`.

No branch for `PhoneOnly` was needed in step 3, but it must exist in the `case` for the code to compile, since the `case` has to be exhaustive over all three variants.

The alternative formulation with `as`:

```gleam
pub fn contact(user: User) -> String {
  case user.contact {
    EmailOnly(..) as c -> "mailto:" <> c.email
    Both(..) as c -> "mailto:" <> c.email
    PhoneOnly(..) as c -> "tel:" <> int.to_string(c.phone)
  }
}
```

Here each pattern ignores the fields with `(..)` and names the whole value `c`, then reads fields with dot expressions. Both styles are fine; the first is shorter, the second is handy when a variant has many fields and you want only one or two of them.

What you cannot write is `user.contact.email`, because `user.contact` has type `Contact` and "user.contact may not have an email attribute." The dot expression is rejected at compile time, not at runtime.

### Example 6: Nested lists

```gleam
pub type NestedList(a) =
  List(Nested(a))

pub type Nested(a) {
  Sublist(NestedList(a))
  Item(a)
}
```

The nested list `[1, [2, 3], 4]` is written in Gleam as

```gleam
[Item(1), Sublist([Item(2), Item(3)]), Item(4)]
```

Every element has the same type, `Nested(Int)`, which satisfies list homogeneity, and the tag says whether the element is a bare item or a sublist.

Direct recursion:

```gleam
// Sum a nested list of integers
pub fn sum_nested(s: NestedList(Int)) -> Int {
  case s {
    [] -> 0
    [first, ..rest] ->
      case first {
        Sublist(t) -> sum_nested(t)
        Item(a) -> a
      }
      + sum_nested(rest)
  }
}
```

This is a **tree recursion** with two distinct recursive calls:

- `[] -> 0`: an empty nested list sums to 0.
- Otherwise split into `first` and `rest`. An inner `case` on `first` decides how much the first element contributes: if it is a `Sublist(t)`, recurse *into* the sublist with `sum_nested(t)`; if it is an `Item(a)`, the contribution is just `a`. Then add `sum_nested(rest)` to recurse *along* the list.
- Note that a `case` expression is a value in Gleam, which is why the whole `case first { ... }` can sit directly to the left of `+`.
- Both recursive calls shrink the data (one goes down into a sublist, one goes right along the list), so the recursion terminates on finite nested lists.

Higher-order version:

```gleam
// Sum a nested list of integers with list.map
pub fn sum_nested(s: NestedList(Int)) -> Int {
  int.sum(
    list.map(s, fn(x) {
      case x {
        Sublist(t) -> sum_nested(t)
        Item(a) -> a
      }
    })
  )
}
```

Same idea with the "along the list" recursion delegated to the library: `list.map` turns each `Nested(Int)` element into the `Int` it contributes (recursing into sublists as needed), producing a `List(Int)`, and `int.sum` adds them up. The explicit base case disappears because `list.map` of `[]` is `[]` and `int.sum([])` is `0`. Only the "down into a sublist" recursion remains written by hand.

**Python comparison (extra context):**

```python
def sum_nested(s):
    total = 0
    for x in s:
        if isinstance(x, list):
            total += sum_nested(x)
        else:
            total += x
    return total
```

The structure is the same, but Python asks the runtime "is this a list?" with `isinstance`, while Gleam encodes the answer in the data with `Item` and `Sublist`, and the compiler then forces you to handle both.

### Example 7: Debugging `increasing` on non-empty lists

```gleam
pub type NotEmpty(a) {
  Last(first: a)
  Link(first: a, rest: NotEmpty(a))
}

pub fn starts_with(s: NotEmpty(a), v: a) -> Bool {
  s.first == v
}

pub fn increasing(s: NotEmpty(Int)) -> Bool {
  case s {
    Last(_) -> True
    Link(first, Link(second, _)) if second <= first -> False
    Link(_, rest) -> increasing(rest)
  }
}
```

First, the two things that are right:

- `starts_with` uses `s.first` with no `case` at all, and that is legal because *both* variants of `NotEmpty` have a field named `first` of type `a`. Contrast with `Contact`, where no field is common to all variants.
- The `Last(_) -> True` base case is correct: a one-item list is increasing.

**The bug: the final adjacent pair is never compared.** The second pattern is `Link(first, Link(second, _))`, which requires the rest of `s` to be a `Link`, that is, to have at least two items after `first`... more precisely, it requires `s` to have at least three items total, since a `Link(second, _)` itself has at least two items. So whenever the rest is a `Last`, the guard is skipped entirely and control falls through to `Link(_, rest) -> increasing(rest)`, which eventually hits `Last(_) -> True`.

**Counterexample trace, `increasing(Link(3, Last(1)))`** (the list `[3, 1]`, which is not increasing):

1. `Last(_)`? No, `s` is a `Link`.
2. `Link(first, Link(second, _))`? No. `first` would be `3`, but the rest is `Last(1)`, not a `Link`, so the pattern fails before the guard is even considered.
3. `Link(_, rest)`? Yes, with `rest = Last(1)`. Recurse: `increasing(Last(1))` matches `Last(_)` and returns `True`.

So `increasing([3, 1])` returns `True`. In general, the last pair of neighbors in the list is never checked, so any list that is increasing except for its final step is misreported, for example `[1, 5, 2]` also returns `True`.

**Fix A (minimal patch): add the missing pattern** for the case where the rest is a `Last`, before the general recursive branch.

```gleam
pub fn increasing(s: NotEmpty(Int)) -> Bool {
  case s {
    Last(_) -> True
    Link(first, Link(second, _)) if second <= first -> False
    Link(first, Last(second)) if second <= first -> False
    Link(_, rest) -> increasing(rest)
  }
}
```

**Fix B (cleaner): compare through the common field** `first`, which exists in both variants, so the two "rest" shapes do not need separate patterns at all.

```gleam
pub fn increasing(s: NotEmpty(Int)) -> Bool {
  case s {
    Last(_) -> True
    Link(first, rest) -> first < rest.first && increasing(rest)
  }
}
```

Reading Fix B: a `Last` is increasing; a `Link` is increasing iff its own item is less than the first item of its rest *and* its rest is increasing. `rest.first` is legal for the reason `starts_with` works. Note that `increasing(rest)` re-examines the pair starting at `rest`, so every adjacent pair gets checked exactly once, including the final one.

---

## Common Pitfalls

1. **Reaching for `.first` or `.rest` on a Gleam `List`.** They do not exist. Use a `case` pattern like `[first, ..rest]`. Dot access is only for records (your own ADT variants), and only when the field is known to exist.
2. **Putting `..` in the wrong place.** `..` may appear only once, only at the very end of a literal, and only before a list. `[..s, 9]` and `[1, ..s, 9]` are not valid.
3. **Writing `[1, 3, s]` when you meant `[1, 3, ..s]`.** The first tries to make a list whose third item is a list, which is a type error (or, with homogeneous lists of lists, silently a different structure than you intended).
4. **Mixing item types.** `[1, "two"]` and `["one", ..s]` (where `s: List(Int)`) are type mismatch errors. Gleam lists are homogeneous, full stop. If you really need "integers or lists of integers," define an ADT like `Nested(a)`.
5. **Getting branch order wrong.** The first matching branch wins. A catch-all (`_` or `_, _`) placed early swallows everything after it; in `matches`, moving `_, _ -> True` to the top makes the function always return `True`. Conversely, in `describe`, `[4, ..]` preempts the later `[a, b] if a < b` for the list `[4, 9]`.
6. **Expecting guards to make a case exhaustive.** A guarded branch can fail its guard, so you still need an unguarded branch covering that shape. This is also the trap behind the `increasing` bug: a pattern that is too specific (`Link(second, _)` where a `Last` can appear) silently sends values to a later branch instead of producing an error.
7. **Forgetting that a nested pattern constrains the *shape* of the inner value, not just its contents.** `Link(first, Link(second, _))` does not mean "the next item is `second`"; it means "the rest is a `Link`, and its item is `second`."
8. **Writing `user.contact.email`.** Illegal, because a `Contact` may be a `PhoneOnly`. Match the variant first, then access the field (optionally with `Variant(..) as c`).
9. **Confusing `pub type X = ...` with `pub type X { ... }`.** The first is an alias introducing no constructors; the second defines variants. You cannot "construct a `NestedList`" with a constructor, because `NestedList(a)` is just `List(Nested(a))`; you build it with a list literal of `Item`/`Sublist` values.
10. **Python operator and comparison habits.** Use `&&`, `||`, `!`, not `and`, `or`, `not`. Use `<>` for string concatenation, not `+`. There is no `is`, and you do not need it, since all values are immutable and `==` is structural.
11. **Trying to mutate.** Nothing is mutable, so a "modified" list is always a new list that shares structure with the old one. `drop(s, n)` does not change `s`.
12. **Dropping a base case because "it can't happen."** `nth_equals`'s `[] -> True`, `drop`'s `[] -> []`, and `matches`'s `_, _ -> True` all exist to handle running off the end of a list, and in each case the chosen return value is deliberate. The compiler will reject the function without them.

---

## Likely Exam Points

### 1. Evaluating a `case` with ordered patterns and guards

**Q.** Using the `describe` function from lecture, give the result of `describe([2, 7, 9])` and `describe([7, 2])`, and explain why `describe([4, 9])` does not return `"List of 4 then 9"`.

**A.** `describe([2, 7, 9])`: `[]` no; `[1]` no; `[4, ..]` no (first item is 2); `[a, b]` no (three items); `[_, _]` no; `[a, b, ..rest] if a < b` matches with `a = 2`, `b = 7`, `rest = [9]`, and `2 < 9`... careful, the guard is `a < b`, that is `2 < 7`, which is true, so the result is `"List ending in [9]"`. `describe([7, 2])`: `[a, b] if a < b` fails its guard (`7 < 2` is false), so the next branch `[_, _]` matches, giving `"List of two elements"`. `describe([4, 9])` returns `"List starting with 4"` because `[4, ..]` appears earlier than `[a, b] if a < b`, and the first matching branch wins.

### 2. Which list literals type-check

**Q.** Given `let s = [5, 7, 9]`, which of these are errors and why: (a) `[1, 3, ..s]`, (b) `["one", ..s]`, (c) `[1, 3, s]`, (d) `[s, [1]]`?

**A.** (a) Fine: `List(Int)` with value `[1, 3, 5, 7, 9]`, sharing `s`'s cells. (b) Error: the items of `s` are `Int`, so the literal's items cannot be `String`. (c) Error: `s` is a `List(Int)` used as an item alongside `Int`s. (d) Fine: a `List(List(Int))` with two items, both lists of integers.

### 3. Completing a list-pattern recursion (fill-in-the-blank, the lecture's own format)

**Q.** Fill in `alternating`, which returns whether `s` is of the form `[x, y, x, y, x, ...]`, using three branches, and say why `List(a)` rather than `List(Int)` is an acceptable parameter type.

**A.**

```gleam
fn alternating(s: List(a)) -> Bool {
  case s {
    [] | [_] | [_, _] -> True
    [x, _, z, ..] if x != z -> False
    [_, ..rest] -> alternating(rest)
  }
}
```

Lists of length 2 or less are vacuously alternating; if the first and third items differ, the list fails; otherwise drop the first item and continue. `List(a)` works because any two values of the same type can be compared with `==`/`!=` in Gleam, so no integer-specific operation is used.

### 4. Why a base case returns what it returns

**Q.** In `matches`, why is the second branch `_, _ -> True` instead of `_, _ -> False`? And why must it come second?

**A.** The second branch is reached only when at least one of the lists is empty, which means every pair compared so far was equal and the shorter list is a prefix of the longer one. By the lecture's definition ("equal, or one is a prefix of the other") that is a match, so `True` is correct, and `assert matches([], [1, 2])` confirms it. It must come second because `_, _` matches every pair of lists; placed first, it would make `matches` return `True` unconditionally.

### 5. Multi-subject `case`

**Q.** Rewrite the first branch of `matches` as nested single-subject `case` expressions, and state the rule about how many patterns a branch needs when a `case` has multiple subjects.

**A.**

```gleam
case s {
  [x, ..s_rest] ->
    case t {
      [y, ..t_rest] -> x == y && matches(s_rest, t_rest)
      [] -> True
    }
  [] -> True
}
```

With multiple subjects, each branch must supply exactly one pattern per subject, separated by commas (`case s, t { pa, pb -> ... }`), and all patterns must match for the branch to be taken.

### 6. The `matches` plus `drop` characterization of periodicity

**Q.** Explain in one or two sentences why `matches(s, drop(s, n))` correctly decides whether `s` has period `n`, and state what `repeating(s, n)` returns when `s` is shorter than `n`.

**A.** Position `i` of `drop(s, n)` is position `i + n` of `s`, so comparing the two lists item by item checks exactly `s[i] == s[i+n]` for every `i` where both items exist, which is the definition of period `n`. When `s` is shorter than `n`, `drop(s, n)` is `[]` (it removes "up to" `n` items), and `matches(s, [])` takes the `_, _ -> True` branch, so `repeating` returns `True`, which is right since no item has an item `n` positions earlier.

### 7. Writing and tracing `drop`

**Q.** What does `drop([1, 2], 5)` return, and how many new list cells does `drop([1, 2, 3, 4], 2)` allocate?

**A.** `drop([1, 2], 5)` returns `[]`: the counter never reaches 0, so the recursion runs off the end and hits the `[] -> []` branch. `drop([1, 2, 3, 4], 2)` allocates **zero** new cells; it returns the existing tail `[3, 4]`, which is a suffix of the original chain. This is safe because lists are immutable.

### 8. Legality of dot expressions on ADTs

**Q.** For the lecture's `Contact`/`User` types and the `NotEmpty(a)` type, say whether each is allowed and why: (a) `user.name`, (b) `user.contact`, (c) `user.contact.email`, (d) `s.first` for `s: NotEmpty(a)`, (e) `s.rest` for `s: NotEmpty(a)`.

**A.** (a) and (b) are allowed: `User` has a single variant containing both fields, so they always exist. (c) is not allowed: `user.contact` has type `Contact`, and a `Contact` may be `PhoneOnly`, which has no `email` field. (d) is allowed: every variant of `NotEmpty` (`Last` and `Link`) has a `first` field of type `a`, which is exactly why `starts_with` can be written as `s.first == v` with no `case`. (e) is not allowed: `Last` has no `rest` field, so you must match `Link(_, rest)` first (or match the variant and use `Link(..) as c` before writing `c.rest`).

### 9. Alternation patterns that bind names

**Q.** Why can `EmailOnly(e) | Both(e, _)` share one branch body, and why does the `as`-based version of `contact` write `EmailOnly(..) as c` and `Both(..) as c` as two separate branches with identical bodies?

**A.** Alternation is allowed when every alternative binds the same names with the same types: both `EmailOnly(e)` and `Both(e, _)` bind a single `String` named `e`, so the body `"mailto:" <> e` is well-typed either way. In the `as` version, the body reads `c.email` through a dot expression, which is only permitted when the compiler knows which variant `c` is; combining the two variants into one branch would leave that unknown, so they are written separately.

### 10. Recursion over a nested ADT

**Q.** Write `count_items`, returning the number of `Item` values anywhere in a `NestedList(a)` (sublists counted recursively). Then give the result for `[Item(1), Sublist([Item(2), Sublist([]), Item(3)]), Item(4)]`.

**A.**

```gleam
pub fn count_items(s: NestedList(a)) -> Int {
  case s {
    [] -> 0
    [first, ..rest] ->
      case first {
        Sublist(t) -> count_items(t)
        Item(_) -> 1
      }
      + count_items(rest)
  }
}
```

Same two-way (tree) recursion as `sum_nested`: recurse down into each `Sublist` and along the rest of the list. For the given value: `Item(1)` contributes 1; the `Sublist` contributes `count_items([Item(2), Sublist([]), Item(3)]) = 1 + 0 + 1 = 2`; `Item(4)` contributes 1. Total **4**.

### 11. Why the ADT is necessary for nested lists

**Q.** Why can you not simply write `[1, [2, 3], 4]` as a Gleam value, and what does the `Nested(a)` type do about it?

**A.** Gleam lists are homogeneous: every item must have the same type, and `1` (an `Int`) does not have the same type as `[2, 3]` (a `List(Int)`). `Nested(a)` wraps both possibilities into one type with two variants, `Item(a)` and `Sublist(NestedList(a))`, so the list `[Item(1), Sublist([Item(2), Item(3)]), Item(4)]` is homogeneous (all items are `Nested(Int)`) while the constructor tag records which case each element actually is. The tag plays the role that `isinstance(x, list)` plays in Python.

### 12. Spotting the bug in `increasing`

**Q.** The lecture's `increasing` for `NotEmpty(Int)` has a bug. Give an input it gets wrong, explain the cause, and fix it.

**A.** `increasing(Link(3, Last(1)))`, the list `[3, 1]`, wrongly returns `True`. The branch `Link(first, Link(second, _)) if second <= first` requires the *rest* to be a `Link`, so when the rest is a `Last` the comparison is skipped and control falls to `Link(_, rest) -> increasing(rest)`, which returns `True` from the `Last(_)` base case. The result: the final adjacent pair is never compared, so `[1, 5, 2]` also returns `True`. Fix by comparing through the common `first` field:

```gleam
pub fn increasing(s: NotEmpty(Int)) -> Bool {
  case s {
    Last(_) -> True
    Link(first, rest) -> first < rest.first && increasing(rest)
  }
}
```

(Equivalently, add the missing branch `Link(first, Last(second)) if second <= first -> False` before the recursive branch.)

### 13. Type alias versus custom type

**Q.** What is the difference between `pub type NestedList(a) = List(Nested(a))` and `pub type Nested(a) { Sublist(NestedList(a)) Item(a) }`?

**A.** The first, with `=`, is a **type alias**: it introduces a new *name* for the existing type `List(Nested(a))` and no new constructors or values. The second, with a block, declares a **new custom type** with two variants, `Sublist` and `Item`, which are the constructors you use to build values and the patterns you use to take them apart. The two definitions refer to each other, which is how the recursive "a nested list contains items and nested lists" shape is expressed.

---

## Summary

- Gleam `List` values are **immutable linked lists**. A literal may end with `..rest`, which supplies the rest of the list and **shares memory** with it; `..` appears at most once, only at the end, only before a list.
- Lists are **homogeneous**: `List(Int)`, `List(String)`, `List(List(Int))`. `[1, "two"]`, `["one", ..s]` (with `s: List(Int)`), and `[1, 3, s]` are all type errors.
- `gleam/list` provides `map`, `filter`, `all`, `any`, `append`, `zip`, and more, so not every list problem needs hand-written recursion.
- There is **no `.first` or `.rest`** for lists. A `case` expression with list patterns both tests the shape and binds names: `[]`, `[1]`, `[4, ..]`, `[a, b]`, `[_, _]`, `[a, b, ..rest]`, `_`.
- Pattern extras: guards (`if a < b`), alternation (`[] | [_] | [_, _]`), wildcards (`_`), rest patterns (`..` and `..rest`), `as` bindings, and **multiple subjects** (`case s, t { pa, pb -> ... }`).
- **The first matching branch wins**, and Gleam requires the `case` to be exhaustive, so guarded branches always need an unguarded fallback.
- Gleam has no `is`: `==`/`!=` are structural and work on any two values of the same type. Boolean operators are `&&`, `||`, `!`; string concatenation is `<>`.
- `alternating`: base case for length 0, 1, 2; fail on `[x, _, z, ..] if x != z`; otherwise recurse on `[_, ..rest]`.
- `repeating` directly: for each suffix, check `nth_equals(first, s, n)` and recurse; `nth_equals` returns `True` when it runs off the end, which makes short lists pass.
- `matches s t`: equal first items plus matching rests, with `_, _ -> True` when either list runs out (prefixes count as matches). `drop(s, n)` returns the suffix after up to `n` items, allocating nothing.
- `repeating(s, n) = matches(s, drop(s, n))`: a list has period `n` exactly when it matches itself shifted left by `n`.
- An **algebraic data type** is a set of named **variants**, each with typed **fields**: a "sum" of variants, each a "product" of fields. Constructors are capitalized and serve as both builders and patterns.
- A pattern gives local names to a variant's fields (`EmailOnly(e) | Both(e, _)`); alternatives in a `|` pattern must bind the same names and types.
- **Dot access rule**: `x.f` is allowed only when `f` is guaranteed present, that is, when every variant of `x`'s type has it (`s.first` on `NotEmpty`, `user.contact` on `User`) or when a pattern has already pinned down the variant (`EmailOnly(..) as c -> c.email`). `user.contact.email` is rejected at compile time.
- `pub type X = ...` is a **type alias** (no new constructors); `pub type X { ... }` defines variants. Types can be **parameterized** (`NestedList(a)`) and **recursive** (`Nested`, `NotEmpty`).
- ADTs buy you two things: expressing data that a homogeneous list cannot hold on its own (`Item` vs `Sublist` tags replacing Python's `isinstance`), and ruling out bad values by construction (`NotEmpty` has no empty case, so `s.first` always exists).
- `sum_nested` is a tree recursion: recurse **into** each `Sublist` and **along** the rest of the list, either by hand or with `int.sum(list.map(s, ...))`.
- Debugging lesson from `increasing`: an over-specific nested pattern (`Link(first, Link(second, _))`) silently skips cases where the inner value is a different variant (`Last`), so the final adjacent pair is never checked. Compare through a field common to all variants (`rest.first`) or add the missing pattern.
