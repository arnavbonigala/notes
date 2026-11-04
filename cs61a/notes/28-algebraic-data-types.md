<!-- Fri, Oct 30, 2026 | sources: slides + YouTube auto-transcript -->
# Lecture 28: Algebraic Data Types

## Overview

This lecture finishes the tour of Gleam's data: first linked lists (Gleam's `List` is a linked list, built and taken apart with list literals, the spread `..` syntax, and `case` patterns rather than `.first`/`.rest`), then tuples, then the main idea: **algebraic data types (ADTs)**. An ADT is built from just two ways of combining data: **product types** (combine several fields into one value, like a Python `@dataclass`) and **sum types** (choose between named variants, like a Python union `int | None`). "Algebraic" just means using both together. The payoff is that a well-designed ADT makes illegal states impossible to even write down: the lecture's running example is a `User` whose `Contact` is `EmailOnly`, `PhoneOnly`, or `Both`, so a user with no way to be contacted is a *type error caught before the program runs*, not a bug discovered later. Along the way the lecture works a pile of list-recursion examples (`describe`, `sorted`, `alternating`, `repeating`/`nth_equals`, `matches`, `drop`) and then ADT examples (`Contact`/`User`, `Tree`, nested lists, and a buggy non-empty-list `increasing` to debug).

Note on language: this lecture's code is **Gleam**, not Python. The Gleam code below is the lecture's own code; Python versions are given alongside as translations to build intuition (the Python/dataclass product-and-sum examples were also shown in lecture).

---

## Key Concepts

### 1. A Gleam list is a linked list

`[3, 7, 5, 9]` *looks* like a Python list but is a linked list: a chain of pairs, each holding a first item and a pointer to the rest.

Because of that, a list literal can end with an expression for **the rest of the list**:

```gleam
let s = [5, 7, 9]
let odds = [1, 3, ..s]   // [1, 3, 5, 7, 9], sharing memory with s
```

Box-and-pointer reasoning in words: `s` points to a chain `5 -> 7 -> 9 -> end`. Evaluating `[1, 3, ..s]` allocates exactly **two** new links, one holding `1` and one holding `3`, and the `3` link's rest pointer points **at the very same `5` link that `s` points at**. Nothing is copied. Both `s` and `odds` now share the tail `5 -> 7 -> 9`. This is safe precisely because all Gleam values are immutable, so no one can mutate the shared tail out from under either name.

Rules for `..` inside a list literal:
- it can appear **only once**,
- **only at the end**,
- and the expression after it must be **a list**.

### 2. Lists are homogeneous, and the type says what is inside

Every item of a Gleam list must have the same type. All of these are type mismatch errors:

```gleam
[1, "two"]        // Int and String
["one", ..s]      // s is a List(Int), so its items are Ints
[1, 3, s]         // 1 and 3 are Ints; s is a List(Int), not an Int
```

The type `List(Int)` means "list whose items are `Int`s". Combined with the `gleam/list` module:

```gleam
import gleam/list

fn double_each(s: List(Int)) -> List(Int) {
  list.map(s, fn(n) { 2 * n })
}
// double_each([5, 7, 9]) == [10, 14, 18]
```

The `list` module has the higher-order functions you already know from Python: `map`, `filter`, `all`, `any`, `append`, `zip`, and more.

### 3. There is no `.first` or `.rest`: use `case` patterns

Gleam does not let you reach into a list with attribute access. The *only* way to look inside is a `case` expression whose patterns describe the shape of the list and give **local names** to the pieces. This is the single most important mechanical skill from this lecture.

Pattern vocabulary for lists:

| Pattern | Matches |
|---|---|
| `[]` | the empty list only |
| `[1]` | a one-item list whose item equals `1` |
| `[_]` | any one-item list (item ignored) |
| `[_, _]` | any two-item list, exactly |
| `[4, ..]` | any list of length >= 1 starting with `4` (the `..` may match zero items) |
| `[a, b]` | exactly two items, named `a` and `b` |
| `[a, b, ..rest]` | at least two items; `rest` names the **list** of everything after them |
| `p1 \| p2` | either pattern (both alternatives must bind the same names) |
| `_` | anything |
| `pattern if cond` | a **guard**: match only if `cond` is also true |

Branches are tried **top to bottom**, and the first match wins. Guards can use the names bound by that pattern.

### 4. `case` can have multiple subjects

```gleam
case s, t {
  [x, ..s_rest], [y, ..t_rest] -> ...
  _, _ -> ...
}
```
You write one pattern per subject, separated by commas. `_, _` matches any `s` and any `t`.

### 5. Equality works on any type; there is no `is`

Two values of the same type can always be compared with `==` and `!=`, structurally. There is no equivalent of Python's `is`, because all values are immutable, so identity never matters observably. Logical operators: `&&` for `and`, `||` for `or`, `!` for `not`.

### 6. Tuples

- Construct with `#(...)`: `#(3, 4)`, `#("oski", 61)`.
- Unlike lists, a tuple may hold **different types**, and its type records each one: `#(Int, Int)`, `#(String, Int)`. The type therefore fixes the length.
- Access by position with `.0`, `.1`, `.2`, ..., which is efficient regardless of tuple length.
- Destructure with `let`: `let #(x2, y2) = #(x * x, y * y)`.
- Tuple syntax doubles as a **pattern** in `case`, naming the components.

### 7. Two ways to build complex data: product and sum

**Product type (combination).** Bundle several fields into one value. Python's `@dataclass`; Gleam's single-variant type. The fields may have different types, and may themselves be user-defined types.

```python
@dataclass
class Position:
    latitude: float
    longitude: float

@dataclass
class Student:
    email: str
    sid: int
    position: Position
```

Why "product"? The set of possible values is the set of possible latitudes **times** the set of possible longitudes: every combination is allowed, so cardinalities multiply.

**Sum type (choice).** One name for several alternative variants. In Python, a union:

```python
OptionalInt = int | None          # an int, or nothing
LinkedList = Link | tuple[()]     # a Link, or the empty tuple
```

Why "sum"? The possible values are all the `int`s **plus** the one `None` value: cardinalities add.

The distinction in one line: a product type is "a `String` **and** an `Int` combined into one object"; a sum type is "something that is **either** a `String` **or** an `Int`".

**Algebraic data types are just these two used together**: a named type lists its variants (sum), and each variant carries fields (product).

### 8. The point of ADTs: make illegal states unrepresentable

Suppose every user must have an email, a phone, or both, and must never have neither (otherwise you cannot contact them). The naive product type does not express this:

```python
@dataclass
class User:                 # BAD
    name: str
    email: str | None
    phone: int | None       # allows email=None, phone=None
```

You could add runtime checks, but then you have to decide what to do with a contactless user, and the bad state is still *constructible*. The ADT version removes the possibility entirely:

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

There is no "neither" variant, so there is no way to build a `User` without contact information. In Gleam a program that violates the types **cannot be run at all**, so the bad case is eliminated before execution rather than defended against during it. (The same shapes can be written in Python with three dataclasses and `Contact = EmailOnly | PhoneOnly | Both`, but Python only *hints*: nothing forces `user.contact` to actually hold one of them.)

Two further Gleam design points from lecture:
- A sum type's variants are **not themselves types**. `EmailOnly` is a constructor of `Contact`; the field in `User` is declared as `Contact`. You cannot splice ad-hoc unions together on the fly, which pushes you to name the concept ("these three variants exist because they describe a contact option").
- Cost of this: a type with one variant makes you write the name twice (`pub type User { User(...) }`). Benefit: you habitually ask "what are the possible values of this type?"

### 9. Patterns over ADT values, and when dot access is allowed

```gleam
pub fn contact(user: User) -> String {
  case user.contact {
    EmailOnly(e) | Both(e, _) -> "mailto:" <> e
    PhoneOnly(n) -> "tel:" <> int.to_string(n)
  }
}
```

- A constructor pattern gives local names to that variant's fields, so `e` is the email in whichever of the two variants matched. The `|` alternative is legal here because **both alternatives bind `e`**.
- The compiler checks **exhaustiveness**: if some variant of `Contact` were unhandled, the program would not run. So after the `case`, you know every possibility was considered, and the string concatenation is type-correct.

Dot access on ADT fields is allowed only when the field is guaranteed to exist:
- `user.contact` is fine: every variant of `User` (there is one) has a `contact` field.
- `user.contact.email` is **not allowed**: `user.contact` might be a `PhoneOnly`, which has no `email`.
- After matching a variant, dot access on that value is fine. The `Variant(..)` pattern ignores the fields and `as c` names the whole matched value:

```gleam
pub fn contact(user: User) -> String {
  case user.contact {
    EmailOnly(..) as c -> "mailto:" <> c.email
    Both(..) as c -> "mailto:" <> c.email
    PhoneOnly(..) as c -> "tel:" <> int.to_string(c.phone)
  }
}
```

So the rule is: use `case` to establish which variant you have, then use dots inside that branch.

### 10. Type aliases vs new types

```gleam
pub type NestedList(a) =            // alias: just another name for an existing type
  List(Nested(a))

pub type Nested(a) {                // new type with two variants
  Sublist(NestedList(a))
  Item(a)
}
```

`=` introduces an **alias** (no new variants, no constructors). Braces with variant names introduce a **new** sum type. Here `a` is a **type variable**, so `NestedList(Int)`, `NestedList(String)`, etc. are all available from one definition. The two definitions are mutually recursive, which is exactly what "a nested list contains items and nested lists" means.

---

## Definitions

- **Linked list (Gleam `List`)**: a sequence represented as a chain of links, each holding one item and a reference to the rest; Gleam's built-in list type.
- **List literal**: `[e1, e2, ...]`, optionally ending in `..rest_expression` to reuse an existing list as the tail.
- **Spread / tail syntax (`..`)**: in a list literal, prepends items onto an existing list, sharing its memory; allowed once, only at the end, only before a list. In a pattern, `..` means "and any remaining items", optionally named (`..rest`), where the name refers to a list.
- **Homogeneous list**: a list all of whose items have the same type; `List(T)` is the type of lists of `T`.
- **Type variable (e.g. `a` in `List(a)`)**: a placeholder in a type that may be instantiated with any type, letting one function or type definition work for many element types.
- **Pattern**: a syntactic description of a value's shape used in `case` (and `let`) that both tests the value and binds local names to its parts.
- **Guard (`if`)**: an extra boolean condition attached to a pattern; the branch is taken only if the pattern matches **and** the guard is true.
- **Or-pattern (`|`)**: a pattern matching any of several alternatives; all alternatives must bind the same names.
- **Wildcard (`_`)**: a pattern matching any value without binding a name.
- **Multiple subjects**: `case s, t { ... }`, where each branch lists one pattern per subject, comma-separated.
- **Tuple**: a fixed-length, heterogeneous combination written `#(...)`, with positional access `.0`, `.1`, ...; its type lists each component's type.
- **Product type**: a type whose values combine several fields, all present at once; the number of possible values is the product of the fields' possibilities. (Python `@dataclass`; Gleam single-variant type.)
- **Sum type (union type)**: a named type whose values are one of several variants; the number of possible values is the sum over variants. (Python `A | B`; Gleam multi-variant type.)
- **Variant / constructor**: one named alternative of a sum type, with its own fields; used to build values and to pattern match. Not itself a type in Gleam.
- **Field (attribute)**: a named component of a variant, e.g. `email` in `EmailOnly(email: String)`.
- **Algebraic data type**: a type built by combining products and sums, i.e. a named set of variants each carrying fields.
- **Exhaustiveness checking**: the compiler's verification that a `case` covers every possible value of the subject's type; a non-exhaustive `case` prevents the program from running.
- **Type alias (`pub type X(a) = ...`)**: a new name for an existing type, introducing no constructors.
- **`as` pattern (`P as c`)**: matches `P` and also binds `c` to the whole matched value.
- **`Variant(..)` pattern**: matches that variant while ignoring all of its fields.

---

## Worked Examples

### A. `describe`: reading pattern order off the page

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

`case s` matches the patterns against the *contents* of `s`, top to bottom, first match wins. The lecture's live traces:

| Call | Which branch, and why | Result |
|---|---|---|
| `describe([])` | `[]` matches immediately | `"Empty list"` |
| `describe([1])` | not `[]`; `[1]` requires one item equal to 1 | `"List of just 1"` |
| `describe([2])` | `[]` no; `[1]` no (item is not 1); `[4, ..]` no; all two-item patterns no; falls through to `_` | `"Some other list"` |
| `describe([4, 5, 6])` | `[4, ..]` matches (`..` covers `5, 6`) | `"List starting with 4"` |
| `describe([5, 6])` | `[4, ..]` no; `[a, b]` with `a = 5, b = 6`, guard `5 < 6` true | `"List of 5 then 6"` |
| `describe([6, 5])` | `[a, b]` matches but guard `6 < 5` fails, so continue; `[_, _]` matches | `"List of two elements"` |
| `describe([1, 5, 7, 9, 3])` | `[4, ..]` no; `[a, b]`/`[_, _]` need exactly two items; `[a, b, ..rest]` with `a = 1, b = 5, rest = [7, 9, 3]`, guard `1 < 5` true | `"List ending in [7, 9, 3]"` |

Two lessons: (i) `[a, b]` is checked before `[a, b, ..rest]`, so the exactly-two case never reaches the longer pattern; (ii) the branch body may ignore some bound names, as here where only `rest` is used.

### B. `sorted`, two ways (reconstructed from the transcript; slides for this part were not included)

Recursive version: a list is sorted if the first two items are in order and the list from the second item onward is sorted.

```gleam
fn sorted(s: List(Int)) -> Bool {
  case s {
    [] | [_] -> True
    [first, second, ..rest] -> first <= second && sorted([second, ..rest])
  }
}
```
Progress: the recursive call's argument `[second, ..rest]` is one shorter than `s` (we dropped `first`), so recursion terminates. Note the rebuilt literal `[second, ..rest]` shares `rest` rather than copying it.

Shorter and tail-recursive version:

```gleam
fn sorted(s: List(Int)) -> Bool {
  case s {
    [] | [_] -> True
    [first, second, ..] if first > second -> False
    [_, ..rest] -> sorted(rest)
  }
}
```
Here the "found a violation" case returns directly, and otherwise we drop the already-checked first item and recurse on `rest`. Because the recursive call is the entire body of that branch, it is a **tail call**, so it runs in constant space in Gleam.

### C. `alternating`: does the list look like `[x, y, x, y, x, ...]`?

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
How it works: a list alternates exactly when **every item equals the item two positions later**. Lists of length 0, 1, or 2 trivially satisfy that, hence the first branch. Otherwise, look at positions 0 and 2 (`x` and `z`, skipping position 1 with `_`); if they differ, the answer is `False` immediately. If they agree, drop one item and check the rest, which slides the window by one and eventually checks every such pair.

Trace `alternating([1, 2, 1, 3])`: length 4, so branch 1 fails. `x = 1, z = 1`, guard `1 != 1` is false, so branch 2 fails. Branch 3 recurses on `[2, 1, 3]`: length 3, `x = 2, z = 3`, guard true, so `False`. Propagates out as `False`.

Trace `alternating([1, 2, 2, 1])`: `x = 1, z = 2`, differ, so `False` right away.

Note the signature `List(a) -> Bool`: it works for any item type because `!=` works on any type.

### D. `repeating` with `nth_equals`: is `s` a block of `n` values repeated?

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

The idea generalizes the alternating case: `s` is `n` values repeated exactly when `s[i] == s[i + n]` for every `i` where both exist. `repeating` walks `i` from the front, and for each position checks that item against the item `n` later.

Subtle but important detail: `nth_equals(first, s, n)` is called with **`s`, the whole list**, not `rest`. Since `s[0]` is `first`, counting `n` steps from `s` lands on `s[n]`, which is exactly the item we want to compare with `first`.

Inside `nth_equals`, the two base cases are: the list ran out (`[]` gives `True`, because "shorter than `n`" means there is nothing to contradict) and the countdown hit zero (`0 -> first == expected`). Otherwise step one item forward and decrement `n`. Notice the **nested `case`**: the outer one inspects the list, the inner one inspects the number.

### E. `matches`: equal, or one a prefix of the other

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
This is the multiple-subject `case`. The first branch requires **both** lists to be non-empty: compare heads, then recurse on both tails. The second branch, `_, _`, is reached only when at least one list is empty, which is precisely the situation "we ran out before finding a mismatch", i.e. one is a prefix of the other, so `True`.

Trace `matches([1, 2, 3], [1, 3])`: `1 == 1`, recurse on `[2, 3]` and `[3]`; `2 == 3` is false, so `&&` short-circuits to `False` without recursing further.

Trace `matches([], [1, 2])`: the first branch needs `s` non-empty, so it fails; `_, _` gives `True`.

### F. `repeating` again, built from `matches` and `drop`

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
Why this is the same property: `drop(s, n)` is `s` shifted left by `n`, so comparing `s` against it item by item compares `s[i]` with `s[i + n]`. `matches` stops happily when the shorter list (the dropped one) runs out, which is exactly the "up to the end" behavior we want. "Up to the first `n`" in the docstring matters: if `s` is shorter than `n`, `drop` returns `[]` and `matches(s, [])` is `True`, which is correct since a short list cannot violate the pattern.

`drop` itself is two nested `case`s again: count down on `n`, and separately guard against the list running out. `drop` also illustrates sharing: it returns a **suffix of the original chain**, allocating nothing.

### G. Tuples: formatting a fraction (reconstructed from the transcript)

```gleam
fn fraction_to_string(f: #(Int, Int)) -> String {
  case f {
    #(_, 0) -> "undefined"
    #(n, 1) -> int.to_string(n)
    #(n, d) -> int.to_string(n) <> "/" <> int.to_string(d)
  }
}
```
`#(Int, Int)` is the tuple type (numerator, denominator); the patterns describe the tuple and name its parts. The lecture also showed the `let` and positional forms:

```gleam
let t = #(x * x, y * y)
let #(x2, y2) = t
x2 + y2          // same as t.0 + t.1
```
(and noted you would really just write `x * x + y * y`; the tuple is only there to demonstrate the syntax).

### H. The `Contact` / `User` ADT in use

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
Evaluating the lecture's example:
```gleam
contact(User("Oski", EmailOnly("oski@berkeley.edu")))   // "mailto:oski@berkeley.edu"
```
Step by step: `User("Oski", EmailOnly(...))` builds a `User` value whose `contact` field points to a `Contact` value built by the `EmailOnly` constructor. In `contact`, `user.contact` is legal because every `User` has a `contact` field. The subject is that `Contact` value; the first pattern `EmailOnly(e)` matches and binds `e` to `"oski@berkeley.edu"`; `<>` concatenates. If the user had been built with `Both("a@b.c", 5551234)`, the same first branch would match via its second alternative with `e = "a@b.c"` and the phone ignored.

### I. Trees as a single-variant ADT (from the transcript)

```gleam
pub type Tree(a) {
  Tree(label: a, branches: List(Tree(a)))
}

pub fn leaf(label: a) -> Tree(a) {
  Tree(label, [])
}
```
This is the familiar 61A tree ADT: a tree is a label plus a list of branches, and a leaf is a tree with no branches. `leaf` is an ordinary **function**, not a variant. `a` is a type variable, so you can have `Tree(Int)` or `Tree(String)`.

The lecture's tree: root `1`, branches `2` and `5`; under `2`, the leaves `3` and `4`; under `5`, the leaf `6`.

```gleam
let t = Tree(1, [Tree(2, [leaf(3), leaf(4)]), Tree(5, [leaf(6)])])
```
(`leaf(3)` and `Tree(3, [])` are the same thing; `leaf` is just shorter.)

Summing labels with tree recursion plus `list.map`:

```gleam
pub fn sum_labels(t: Tree(Int)) -> Int {
  t.label + int.sum(list.map(t.branches, sum_labels))
}
```
`t.label` is legal dot access because `Tree` has exactly one variant, so every value has those fields. `list.map(t.branches, sum_labels)` recursively sums each branch, producing a list of numbers, and `int.sum` adds them.

On the example: branch rooted at `2` gives `2 + 3 + 4 = 9`; branch rooted at `5` gives `5 + 6 = 11`; `int.sum([9, 11]) = 20`; plus the root label `1` gives `21`.

Where is the base case? There is no explicit leaf branch: for a leaf, `t.branches` is `[]`, `list.map` returns `[]`, and `int.sum([]) = 0` (like Python's `sum([])`), so the recursion bottoms out at `t.label + 0`. You *could* write an explicit leaf case and it would do no harm.

### J. Nested lists: an alias plus a two-variant type

```gleam
pub type NestedList(a) =
  List(Nested(a))

pub type Nested(a) {
  Sublist(NestedList(a))
  Item(a)
}
```
"A nested list of integers is a list containing integers and nested lists of integers." The sum type `Nested` is what makes the heterogeneity type-safe: a Gleam list must be homogeneous, so instead of mixing `Int`s and lists, every element is a `Nested(Int)`, which is *either* an `Item` wrapping an integer *or* a `Sublist` wrapping another nested list.

Version 1, explicit recursion over the list:

```gleam
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
Two kinds of recursion are interleaved: `sum_nested(rest)` walks along the list (linear recursion), and `sum_nested(t)` descends into a sublist (deep recursion). The inner `case` is the ADT dispatch: unwrap an `Item` to get the integer, or recurse on a `Sublist`. Note that a `case` expression *has a value*, so it can sit directly to the left of `+`.

Version 2, the same thing with `list.map`:

```gleam
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
Map each element to its own total, then add them all. The `[] -> 0` base case disappears because `int.sum([]) = 0`, exactly as in `sum_labels`.

### K. Debugging exercise: `increasing` for non-empty lists

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

First, note *why* `starts_with` can write `s.first` without a `case`: **both** variants of `NotEmpty` have a field named `first` of the same type, so the field is guaranteed to exist. (Contrast `user.contact.email`, which is rejected because `email` is missing from `PhoneOnly`.) This type also structurally guarantees non-emptiness: there is no empty variant, so "a `NotEmpty` list with zero items" cannot be written.

**The bug:** the middle pattern is `Link(first, Link(second, _))`, which requires the rest to be a `Link`. The **final pair** of the list has shape `Link(first, Last(second))`, which that pattern does **not** match. So the last pair is never compared: control falls to `Link(_, rest) -> increasing(rest)`, then `Last(_) -> True`.

Concretely, for the list `5, 3` built as `Link(5, Last(3))`: `Last(_)` does not match; `Link(first, Link(...))` does not match because the rest is a `Last`; the third branch recurses on `Last(3)`, which returns `True`. So `increasing` wrongly reports `True`. In general, any list whose only decrease is between the last two items is misclassified.

**Fixes.** Either add the missing pattern:

```gleam
pub fn increasing(s: NotEmpty(Int)) -> Bool {
  case s {
    Last(_) -> True
    Link(first, Last(second)) if second <= first -> False
    Link(first, Link(second, _)) if second <= first -> False
    Link(_, rest) -> increasing(rest)
  }
}
```

or, more cleanly, exploit the fact that **every** variant has a `first` field (the reason `starts_with` was put on the slide as a hint):

```gleam
pub fn increasing(s: NotEmpty(Int)) -> Bool {
  case s {
    Last(_) -> True
    Link(first, rest) -> first < rest.first && increasing(rest)
  }
}
```
Here `rest.first` is the next item regardless of whether `rest` is a `Last` or a `Link`, so both pairs-at-the-end and pairs-in-the-middle are checked uniformly. (extra context: the second form is my preferred fix rather than one written on the slide; the slide only supplied `starts_with` and the buggy function.)

---

## Common Pitfalls

1. **Trying `s.first` / `s.rest` on a `List`.** Gleam's `List` has no such accessors. Use `case s { [] -> ...; [first, ..rest] -> ... }`. (Your *own* ADTs, like `Tree` or `NotEmpty`, do support dot access on fields present in every variant.)
2. **Misplacing `..` in a literal.** `[1, ..s, 3]` and `[..s, 1]` and `[1, ..s, ..t]` are all illegal: once, at the end, and the thing after it must be a list.
3. **Mixing item types.** `[1, "two"]`, `["one", ..s]` where `s : List(Int)`, and `[1, 3, s]` are type mismatches. If you genuinely need a mixed sequence, that is a signal to define a sum type (as `Nested` does) or to use a tuple.
4. **Forgetting that branch order decides everything.** A broad pattern placed early shadows later ones. In `describe`, `[a, b]` precedes `[a, b, ..rest]`, so two-item lists never reach the longer pattern; a leading `_` would swallow every case.
5. **Forgetting that a guard failure continues to later branches.** `[a, b] if a < b` not matching does not end the `case`; it falls through to `[_, _]`.
6. **Assuming `..` requires at least one remaining item.** `[4, ..]` matches the single-item list `[4]`; `..rest` can bind `[]`.
7. **Writing `user.contact.email`.** Illegal: `user.contact` may be a `PhoneOnly`. Pattern match on the variant first, then use dots inside that branch (optionally with `Variant(..) as c`).
8. **Using a variant name where a type is expected.** In `User(name: String, contact: Contact)` the field type is `Contact`, not `EmailOnly`. Variants are constructors, not types.
9. **Writing a `case` that misses a variant.** Non-exhaustive `case` means the program will not run. Equally, leaving out the "list ran out" case (`[]`, or `Last` for `NotEmpty`) is the classic source of both type errors and logic bugs, as in `increasing`.
10. **Or-patterns that bind different names.** `EmailOnly(e) | Both(e, _)` is fine because both bind `e`; `EmailOnly(e) | PhoneOnly(n)` would not be, since the body cannot know which name exists.
11. **Reaching for `is`.** There is no `is` in Gleam. Use `==` and `!=`; values are immutable, so structural equality is all you need.
12. **Mixing up Python and Gleam operators.** `&&`, `||`, `!`, not `and`, `or`, `not`. Concatenate strings with `<>`.
13. **In `repeating`, passing `rest` instead of `s` to `nth_equals`.** Passing `s` is what makes the countdown land on `s[n]`; passing `rest` would compare against `s[n + 1]` and be off by one.
14. **Confusing `type X = ...` with `type X { ... }`.** The first is an alias (no constructors); the second defines a new type with variants.
15. **Forgetting the double name for single-variant types.** `pub type User { User(...) }`: the type and its one constructor each need the name.

---

## Likely Exam Points

**1. Which branch does a `case` take?**
Given `describe` from part A, what does `describe([4, 3])` return?
*Answer:* `"List starting with 4"`. `[]` fails, `[1]` fails, and `[4, ..]` matches with `..` binding the single remaining item `3`, so the later two-item patterns are never reached.

**2. Fill in list patterns to complete a recursive function.**
Complete `alternating` so that `[x, y, x, y, ...]` returns `True`.
*Answer:*
```gleam
case s {
  [] | [_] | [_, _] -> True
  [x, _, z, ..] if x != z -> False
  [_, ..rest] -> alternating(rest)
}
```
Lists of length <= 2 always alternate; otherwise compare positions 0 and 2 and slide the window by one.

**3. Spot the type error in a list literal.**
Let `s : List(Int)`. Which of `[1, 3, ..s]`, `["one", ..s]`, `[1, 3, s]` type check?
*Answer:* only `[1, 3, ..s]`. `["one", ..s]` mixes `String` items with an `Int` tail. `[1, 3, s]` tries to put a `List(Int)` in as if it were an `Int` item (it lacks `..`).

**4. Memory sharing.**
After `let s = [5, 7, 9]` and `let odds = [1, 3, ..s]`, how many new links are allocated, and what do `s` and `odds` share?
*Answer:* two new links (for `1` and `3`); the `3` link's rest pointer points at the existing `5` link, so `odds` shares the entire tail `5 -> 7 -> 9` with `s`. Nothing is copied, and this is safe because values are immutable.

**5. Product vs sum, and counting values.**
(extra context: the counting arithmetic below is a standard way to test the intuition the lecture gave.) Given
```gleam
pub type Light { Red Yellow Green }
pub type Pair { Pair(Light, Bool) }
pub type Either { L(Light) R(Bool) }
```
how many distinct values does each type have, and which is a product and which a sum?
*Answer:* `Light` has 3. `Pair` is a product: 3 * 2 = 6. `Either` is a sum: 3 + 2 = 5. Products multiply possibilities (all fields present at once), sums add them (one variant at a time).

**6. Design an ADT that forbids an illegal state.**
Define a type for a user who must have an email, a phone, or both, but never neither.
*Answer:*
```gleam
pub type Contact {
  EmailOnly(email: String)
  PhoneOnly(phone: Int)
  Both(email: String, phone: Int)
}
pub type User { User(name: String, contact: Contact) }
```
There is no "neither" variant, so an uncontactable user is a type error caught before the program runs, rather than a runtime check. A single record with two optional fields would permit the bad state.

**7. When is dot access legal?**
Why is `user.contact` allowed but `user.contact.email` rejected? Rewrite the rejected expression correctly.
*Answer:* `contact` exists in every variant of `User`, so it is guaranteed present; `email` does not exist in `PhoneOnly`, so the compiler cannot promise it. Match first:
```gleam
case user.contact {
  EmailOnly(..) as c | Both(..) as c -> c.email
  PhoneOnly(..) -> "none"
}
```
(or bind the field directly with `EmailOnly(e) | Both(e, _) -> e`).

**8. Exhaustiveness.**
What goes wrong with `case user.contact { EmailOnly(e) -> e }`?
*Answer:* it is non-exhaustive: `PhoneOnly` and `Both` are unhandled, so the program will not compile or run. Exhaustiveness checking is the mechanism that makes "all cases considered" a compile-time guarantee.

**9. Multiple-subject `case`.**
Write `matches(s, t)`, true when the lists are equal or one is a prefix of the other, using one `case` with two subjects.
*Answer:*
```gleam
case s, t {
  [x, ..s_rest], [y, ..t_rest] -> x == y && matches(s_rest, t_rest)
  _, _ -> True
}
```
The fallthrough is reached only when one list is exhausted, which is exactly the prefix case.

**10. Composing helpers.**
Explain why `matches(s, drop(s, n))` decides whether `s` is `n` values repeated.
*Answer:* `drop(s, n)` is `s` shifted left by `n` positions, so matching `s` against it compares `s[i]` with `s[i + n]` for every `i` where both exist, which is precisely the repeating condition; `matches` returns `True` once the shifted (shorter) list runs out, so the comparison stops at the right place, and if `s` is shorter than `n`, `drop` yields `[]` and the answer is `True`.

**11. Find the bug in a recursion over a custom ADT.**
Why does the lecture's `increasing` return `True` for `Link(5, Last(3))`?
*Answer:* the guarded pattern is `Link(first, Link(second, _))`, which never matches when the rest is a `Last`, so the final pair is never compared; control falls to `Link(_, rest) -> increasing(rest)` and then to `Last(_) -> True`. Fix by adding `Link(first, Last(second)) if second <= first -> False`, or by rewriting as `Link(first, rest) -> first < rest.first && increasing(rest)`, which works because every variant of `NotEmpty` has a `first` field.

**12. Why a sum type is needed for nested lists.**
Gleam lists are homogeneous, so how can a nested list hold both integers and sublists?
*Answer:* every element has the single type `Nested(a)`, a sum type with variants `Item(a)` and `Sublist(NestedList(a))`. Homogeneity is preserved at the list level, and the variety lives inside the element type, recovered by a `case`.

**13. Base cases via `int.sum`.**
In `sum_labels` and the `list.map` version of `sum_nested` there is no explicit leaf or empty branch. Why is that correct?
*Answer:* `list.map` of the empty list is `[]` and `int.sum([]) = 0`, so a leaf (no branches) or empty nested list contributes 0 automatically. Writing the base case explicitly is allowed and harmless.

**14. Tuple vs list vs ADT.**
You need a value holding a name (`String`) and an age (`Int`). Which of `List`, tuple, or ADT applies, and why?
*Answer:* not a `List` (items must share a type); a tuple `#(String, Int)` works and gives `.0`/`.1` access; a one-variant ADT like `Person(name: String, age: Int)` is usually better because the fields are named and the type documents its meaning. All three can be pattern matched.

---

## Summary

- Gleam's `List` is a **linked list**. `[1, 3, ..s]` prepends onto `s` and **shares** its memory; `..` may appear only once, only at the end, only before a list.
- Lists are **homogeneous**: `List(Int)` is the type of lists of integers. `gleam/list` provides `map`, `filter`, `all`, `any`, `append`, `zip`, and more.
- There is no `.first`/`.rest` on lists. Inspect lists with `case` **patterns**: `[]`, `[1]`, `[_]`, `[_, _]`, `[4, ..]`, `[a, b, ..rest]`, with `|` alternatives, `_` wildcards, and `if` **guards**. Branches are tried in order; first match wins; a failed guard falls through.
- `case` can take **multiple subjects** (`case s, t { ... }`), one pattern per subject.
- `==`/`!=` work on any type; there is no `is` because all values are immutable. Operators are `&&`, `||`, `!`, and `<>` for string concatenation.
- **Tuples** `#(a, b)` are fixed-length and heterogeneous, accessed by `.0`, `.1`, destructurable with `let`, and usable as patterns; their type lists each component type.
- List-recursion patterns practiced: `describe` (pattern order), `sorted` (including a tail-recursive version), `alternating` (`s[i] == s[i+2]`), `repeating` + `nth_equals` (`s[i] == s[i+n]`), `matches` (prefix equality), `drop`, and `repeating` rebuilt as `matches(s, drop(s, n))`.
- **Product types** combine fields that are all present (Python `@dataclass`, Gleam single-variant type); possibilities **multiply**. **Sum types** choose among named variants (Python `A | B`, Gleam multi-variant type); possibilities **add**. **Algebraic data types** use both together.
- The payoff: ADTs make illegal states unrepresentable. `Contact` as `EmailOnly | PhoneOnly | Both` means a `User` with no contact information is a compile-time impossibility, not a runtime check.
- In Gleam, variants are constructors, not types: a sum type must be **named**, and single-variant types repeat the name (`type User { User(...) }`).
- `case` over an ADT binds each variant's fields to local names, supports `Variant(..) as c` plus dot access, and is **exhaustiveness checked**, so unhandled variants stop the program from running.
- Dot access on a field is legal only if **every** variant has it: `user.contact` yes, `user.contact.email` no, `s.first` on `NotEmpty` yes.
- `type X(a) = ...` is an **alias**; `type X(a) { ... }` defines a new type. Mutual recursion between them models nested data (`NestedList`/`Nested`).
- Tree ADT: `Tree(label: a, branches: List(Tree(a)))`, with `leaf` as an ordinary function, and `sum_labels(t) = t.label + int.sum(list.map(t.branches, sum_labels))`, whose base case is implicit in `int.sum([]) == 0`.
- Classic bug to watch for: a pattern that requires the *rest* to be the recursive variant (`Link(first, Link(second, _))`) silently skips the last pair.
