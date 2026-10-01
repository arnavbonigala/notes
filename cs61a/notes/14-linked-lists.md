<!-- Mon, Sep 28, 2026 | sources: slides + code (no transcript available) -->
# Lecture 14: Linked Lists

This lecture makes the recursive view of sequences explicit. It opens with a tree-recursion warm-up (`longest`) that treats a Python list as "a first item `s[0]` and the rest `s[1:]`", then builds a data structure that *is* exactly that definition: a **linked list** is either empty (represented by the empty tuple `()`) or a `Link` object with a `first` value and a `rest` that is itself a linked list. Because the type is defined recursively, every operation on it has the same shape: check `isinstance(s, Link)`, do something with `s.first`, and recurse (or loop) on `s.rest`. We implement the standard toolkit (`len_link`, `getitem_link`, `sum_link`, `range_link`, `extend_link`, `map_link`, `filter_link`, `join_link`) in both iterative and recursive forms, see how to build a linked list from the back forward, and finish with `partitions`, which returns a *Python list* of *linked lists*, showing how the two kinds of sequence compose.

---

## Key Concepts

### 1. A sequence is a first item and the rest

The central idea of the whole lecture. For a Python list `s`, the pair `(s[0], s[1:])` contains all the information in `s`. This gives every list problem a recursive decomposition: solve the problem for `s[1:]`, then decide what to do with `s[0]`. The empty list is the base case.

This is only a *view* for Python lists (slicing `s[1:]` copies). For linked lists it is the literal definition of the data type.

### 2. Tree recursion over a list: include or exclude the first item

When you are searching for a **sublist** (a subsequence: items in order, not necessarily contiguous), there are two recursive subproblems for each item, not one:

- **include** `s[0]`: then the rest must make up `n - s[0]`, so recurse as `longest(s[1:], n - s[0])`
- **exclude** `s[0]`: then the rest must still make up `n`, so recurse as `longest(s[1:], n)`

Two recursive calls per frame means tree recursion, which is the right tool for "search for a variant" or "count the variants" problems. The two results are then combined: here, `max(..., key=len)` picks the longer one.

An important subtlety: the return type is `list[int] | None`, so "no solution" and "the empty solution" are different values. `None` means *no sublist sums to n*; `[]` means *the empty sublist works* (which happens when `n == 0`). That is why the code tests `isinstance(result, list)` instead of truthiness: `[]` is falsy but is a legitimate answer. This is the reason the slide lists "Use `isinstance(s, list)` to test whether `s` is a list" as a useful concept.

### 3. A linked list is a recursive data structure

> A linked list is either empty or a first value and the rest of the linked list.

Two representations are needed, one for each case of that sentence:

- **empty linked list**: the empty tuple `()`
- **non-empty linked list**: a `Link` instance, which has a `first` attribute and a `rest` attribute

`Link(3, Link(4, Link(5, ())))` is a chain of three `Link` objects. The slide notes that this is the same as `Link(3, Link(4, Link(5)))`, because `rest` defaults to `()`.

**Box-and-pointer reasoning in words:** picture three boxes side by side. Box 1 holds `first: 3` and its `rest` arrow points at Box 2. Box 2 holds `first: 4` and its `rest` arrow points at Box 3. Box 3 holds `first: 5` and its `rest` is the empty linked list (often drawn as a slash or as `()`). Only the *outermost* box needs a name: `s` is bound to Box 1, and everything else is reached by following `rest` arrows. So `s.rest.rest.first` is 5, and `s.rest.rest.rest == ()` is `True`.

Crucially, a `Link` object is **never empty**. If you have a `Link`, you are guaranteed `first` and `rest` exist. Emptiness lives entirely in the other case, `()`.

### 4. The `Link` data class and its types

```python
type LinkedList[T] = Link[T] | tuple[()]

@dataclass
class Link[T]:
    "A Link has a first value of type T and the rest of the linked list."
    first: T
    rest: LinkedList[T] = ()  # rest defaults to an empty linked list
```

- `type LinkedList[T] = Link[T] | tuple[()]` declares a **type alias**: a `LinkedList` of `T` is either a `Link` of `T` or an empty tuple. `tuple[()]` is the type of the empty tuple.
- `T` is a **type variable**: it stands for the type of the items. `Link(3, Link(4, Link(5)))` is a `Link[int]`, and also a `LinkedList[int]`.
- `@dataclass` generates `__init__` (so `Link(3, Link(4))` works), `__repr__` (so the REPL shows `Link(first=3, rest=Link(first=4, rest=()))`), and `__eq__` (so `==` compares structurally).
- `rest: LinkedList[T] = ()` gives `rest` a default, which is why `Link(5)` is a one-item linked list.
- The annotations are documentation for readers and type checkers; Python does not enforce them at runtime. `from __future__ import annotations` at the top of the file lets annotations mention names (like `Link` inside the class body) without evaluation-order problems.

### 5. Printing versus the repr

The data class gives a `repr` that shows the nesting, which is accurate but verbose. The lecture adds a `__str__` so `print` produces the compact Scheme-style form:

```python
def __str__(self):
    return format_link(self)

def format_link(s: Link):
    """Return a Link s formatted as items within parentheses."""
    string = '(' + str(s.first)
    remaining = s.rest
    while isinstance(remaining, Link):
        string += ' ' + str(remaining.first)
        remaining = remaining.rest
    assert remaining == (), f'{s!r} is not a LinkedList'
    return string + ')'
```

So: *"How to print it: `(3 4 5)`."* Note the two different displays of the same object:

```python
>>> s                                  # repr, from @dataclass
Link(first=3, rest=Link(first=4, rest=Link(first=5, rest=())))
>>> print(s)                           # str, from format_link
(3 4 5)
```

`format_link` also demonstrates the standard iterative traversal pattern: a `remaining` variable advanced by `remaining = remaining.rest`, with the loop condition `isinstance(remaining, Link)`. The final `assert remaining == ()` catches a malformed structure (something that is neither a `Link` nor `()` in a `rest` position).

Because `str` is called on each `first`, **nested** linked lists print as nested parentheses:

```python
>>> print(Link(s))
((3 4 5))
>>> print(Link(3, Link(Link(4, Link(5)), Link(6))))
(3 (4 5) 6)
```

`Link(s)` is a one-item linked list whose single item is itself a linked list, hence the double parentheses.

### 6. The `isinstance(s, Link)` discipline

Since a `LinkedList` is a union of two types, you must find out which one you have before touching `.first` or `.rest`:

```python
if isinstance(s, Link):
    ...            # here you may access s.first and s.rest
```

or the inverted form:

```python
if not isinstance(s, Link):
    ...            # the empty case
else:
    ...            # here you may access s.first and s.rest
```

The second form is what almost every recursive function in the lecture code uses, because the empty case is the base case.

Symmetrically, when you *construct*, the second argument to `Link` must itself be a linked list. The slide spells out the legal shapes:

- `Link(4, ())` or `Link(4, Link(...))`
- `Link(4, s)` as long as `s` names a linked list
- `Link(4, f(...))` as long as `f` returns a linked list

### 7. Build the rest first, then add to the front

A `Link` is created with its `rest` already in hand, so linked lists are naturally constructed **back to front**:

```python
>>> s = ()
>>> s = Link(5, s)
>>> s = Link(4, s)
>>> s = Link(3, s)
>>> print(s)
(3 4 5)
```

Each line creates a *new* `Link` whose `rest` is the previous value of `s`, then rebinds the name `s`. Nothing is mutated. In box-and-pointer terms, each step adds one box to the left of the existing chain and moves the name `s` onto the new box. The old chain is still intact and is now shared as the tail of the new one.

### 8. Iteration and recursion are two spellings of the same traversal

Every linked list function in the lecture appears in both forms. The pattern is mechanical:

| | iterative | recursive |
|---|---|---|
| advance | `s = s.rest` in a loop | pass `s.rest` to the recursive call |
| stop | `while isinstance(s, Link)` | `if not isinstance(s, Link): return <base>` |
| accumulate | mutate a local (`total += s.first`) | combine in the return (`s.first + sum_link_recursive(s.rest)`) |

The iterative versions reassign the *parameter* `s` as they walk, which is safe: that rebinds a local name and does not change the caller's linked list.

### 9. Recursion that works backward using an accumulator

The slides ask: `range_link` iteratively starts at `k = end - 1` and counts down, prepending as it goes. Can recursion do the same? Yes, by carrying the partially built list as an extra argument:

```python
def range_link_tail(start: int, end: int) -> LinkedList[int]:
    def f(k, s):
        if k < start:
            return s
        else:
            return f(k-1, Link(k, s))
    return f(end-1, ())
```

Here `s` plays exactly the role of the loop variable `s` in the `while` version, and `k` plays the role of the loop counter. The base case returns the accumulator rather than `()`. The slide writes the helper with the arguments in the other order, `f(s, k)` with the call `f((), end-1)`; the lecture code uses `f(k, s)` with `f(end-1, ())`. Either is fine, they just have to be consistent.

(extra context) This shape is called *tail recursion*: the recursive call is the entire return expression, so nothing is left to do after it returns. CPython does not optimize it away, so it still builds a new frame per step, but the pattern is the direct translation of a `while` loop and will matter a lot once we get to Scheme.

### 10. Linked lists can be elements of ordinary lists

`partitions` returns `list[LinkedList[int]]`: a Python list whose items are linked lists. This is the natural fit because the number of partitions is data we want to collect and concatenate (`with_m + without_m`), while each individual partition is built by prepending one part at a time (`Link(m, p)`).

A list comprehension turns a list of partitions into a list of longer partitions:

```python
with_m = [Link(m, p) for p in partitions(n-m, m)]
```

Note the annotation on the slide: each `p` in `partitions(n-m, m)` **sums to `n-m`, not `n`**; prepending `m` is what makes the whole thing sum to `n`.

### 11. Two base cases that look similar but mean opposite things

```python
if n == 0:
    return [()]   # a list containing just the empty partition: one way to make 0
elif n < 0 or m == 0:
    return []     # an empty list: no partitions at all
```

`[()]` has length 1 (success: there is exactly one way to sum to 0, using nothing). `[]` has length 0 (failure). Returning the wrong one silently makes every count zero or every count wrong. This mirrors the `None` versus `[]` distinction in `longest`.

---

## Definitions

- **Linked list**: a sequence that is either empty or consists of a first value together with the rest of the linked list. In this lecture, the empty linked list is the empty tuple `()` and a non-empty linked list is a `Link` instance.
- **`Link`**: a data class with attributes `first` (the first value, of type `T`) and `rest` (a `LinkedList[T]`, defaulting to `()`). A `Link` is never empty.
- **`first`**: the attribute holding the first item of a non-empty linked list.
- **`rest`**: the attribute holding the remainder of the linked list, which is itself a linked list (either another `Link` or `()`).
- **`LinkedList[T]`**: the type alias `Link[T] | tuple[()]`, that is, the union of the non-empty and empty cases.
- **`tuple[()]`**: the type whose only value is the empty tuple.
- **Type variable (`T`)**: a placeholder in a generic type that stands for the type of the contained items, so `LinkedList[int]` is a linked list of integers.
- **Data class (`@dataclass`)**: a class decorator that generates `__init__`, `__repr__`, and `__eq__` from the annotated class attributes.
- **`__str__` versus `__repr__`**: `__str__` is used by `print` (here producing `(3 4 5)`); `__repr__` is used when the REPL displays a value (here produced by `@dataclass`, showing `Link(first=3, ...)`).
- **Sublist (as used in `longest`)**: a subsequence, that is, items of the original list in their original order but not necessarily adjacent.
- **Tree recursion**: a recursive function that makes more than one recursive call per invocation, producing a branching call structure; used for searching and counting variants.
- **Partition of `n` using parts up to `m`**: a multiset of positive integers, each at most `m`, summing to `n`. The "increasing order" constraint in the recursive formulation (never using a part larger than the current `m` in subproblems) is what makes each partition be generated exactly once rather than once per ordering.
- **Accumulator**: an extra parameter that carries the partially built result through a recursive process, playing the role of a loop variable.

---

## Worked Examples

### Example 1: `longest`, tree recursion over a Python list

```python
def longest(s: list[int], n: int) -> list[int] | None:
    """Return the longest sublist of s that sums to n, or None if none exists.

    >>> longest([4, 1, 3, -1, 2, -1], 5)
    [4, 1, -1, 2, -1]
    >>> longest([3, 1, 4], 6)
    None
    """
    if len(s) == 0:
        if n == 0:
            return []
        else:
            return None
    minus_first = longest(s[1:], n - s[0])
    without_first = longest(s[1:], n)
    if isinstance(minus_first, list):
        with_first = [s[0]] + minus_first
        if isinstance(without_first, list):
            return max(with_first, without_first, key=len)
        else:
            return with_first
    else:
        return without_first
```

Step by step, for `longest([4, 1, 3, -1, 2, -1], 5)`:

1. **Base case.** `s` is empty only when we have consumed every item. At that point the empty sublist is the only candidate, so it works exactly when the remaining target `n` is 0: return `[]`. Otherwise there is no way to finish, so return `None`.
2. **Two subproblems.** `s[0]` is 4.
   - `minus_first = longest(s[1:], 5 - 4)` asks: what is the longest sublist of `[1, 3, -1, 2, -1]` summing to 1? Answer: `[1, -1, 2, -1]` (sums to 1, length 4).
   - `without_first = longest(s[1:], 5)` asks: what is the longest sublist of `[1, 3, -1, 2, -1]` summing to 5? Answer: `[1, 3, -1, 2]` (sums to 5, length 4).
3. **Rebuild.** Since `minus_first` is a list, including `s[0]` is viable: `with_first = [4] + [1, -1, 2, -1] = [4, 1, -1, 2, -1]`, which sums to 5 and has length 5.
4. **Combine.** `without_first` is also a list, so return `max(with_first, without_first, key=len)`, which is the length-5 `[4, 1, -1, 2, -1]`. That matches the docstring.
5. If `minus_first` is `None`, including `s[0]` is impossible, so just return `without_first` (which may itself be `None`, correctly propagating failure).

Why `isinstance` and not `if minus_first:`? Because `[]` is a perfectly good answer (it is the answer whenever the remaining target is 0) but it is falsy. `isinstance(x, list)` distinguishes "an answer, possibly empty" from "no answer".

Why `key=len`? `max` on two lists would otherwise compare them lexicographically by element. `key=len` makes it compare lengths, which is what "longest" means. Note the tie-breaking rule: when lengths are equal, `max` returns the *first* argument it saw, so `with_first` wins ties.

For `longest([3, 1, 4], 6)`: every subset of `{3, 1, 4}` sums to 0, 1, 3, 4, 5, 7, 8, or 4+3+1=8; none is 6, so every branch bottoms out in `None` and `None` propagates all the way up.

Two notes on the material: the slide's third annotation box reads "`[4, 1, 3, -1, 2]` is a way to make 5", which does not sum to 5; the PDF extraction appears to have garbled which blank that annotation belongs to, and the correct `with_first` for this call is `[4, 1, -1, 2, -1]`. Also, the docstring in `14.py` mistakenly uses the name `count_sums` in its doctest lines, so those doctests would fail as written even though the function body is correct; the slide version of the docstring uses `longest`.

### Example 2: Constructing and inspecting a linked list

```python
>>> s = Link(3, Link(4, Link(5)))
>>> s.first
3
>>> s.rest.first
4
>>> s.rest.rest.first
5
>>> s.rest.rest.rest == ()
True
>>> s
Link(first=3, rest=Link(first=4, rest=Link(first=5, rest=())))
>>> s.rest.rest
Link(first=5, rest=Link(first=5, rest=()))  # (no: see below)
```

The actual lecture output for the last one is:

```python
>>> s.rest.rest
Link(first=5, rest=())
```

Reading this as boxes: three `Link` boxes chained left to right. `s` names the leftmost. `s.rest` is the middle box, `s.rest.rest` is the rightmost box, and `s.rest.rest.rest` is `()`, the end of the chain. Notice that `s.rest.rest` is not a copy of anything, it *is* the third box, so any name bound to it refers to the very same object that is the tail of `s`.

`Link(5)` works because `rest` defaults to `()`. The `==` test against `()` succeeds because the third link's `rest` really is the empty tuple.

### Example 3: `len_link` and `sum_link`, iteratively

```python
def len_link(s: LinkedList):
    """Return the length of a linked list.

    >>> len_link(Link(3, Link(4, Link(5))))
    3
    """
    length = 0
    while isinstance(s, Link):
        length += 1
        s = s.rest
    return length
```

The loop walks a pointer along the chain. On entry `s` is box 1 (`isinstance` is true), so `length` becomes 1 and `s` becomes box 2; then 2 and box 3; then 3 and `()`. At that point `isinstance((), Link)` is false and the loop ends, returning 3. Reassigning the parameter `s` only rebinds the local name in this frame; the caller's linked list is untouched.

```python
def sum_link(s: LinkedList[float]) -> float:
    total = 0
    while isinstance(s, Link):
        total += s.first
        s = s.rest
    return total
```

Same walk, accumulating `s.first` instead of counting. `sum_link(Link(3, Link(4, Link(5))))` is `0 + 3 + 4 + 5 = 12`. The empty linked list correctly sums to 0 because the loop body never runs.

### Example 4: `getitem_link`, with bounds checking

```python
def getitem_link(s: LinkedList, i: int):
    """Return the item at index i of a non-empty linked list.

    >>> getitem_link(Link(3, Link(4, Link(5))), 1)
    4
    """
    while i > 0:
        assert isinstance(s, Link), 'Index out of range'
        s = s.rest
        i -= 1
    assert isinstance(s, Link), 'Index out of range'
    return s.first
```

Linked lists have no random access: to get index `i` you must take `i` steps, so this is linear time, not constant time like Python list indexing. The two asserts are both needed: the one inside the loop catches running off the end while advancing, and the one after the loop catches `i` pointing exactly one past the end (for example index 3 in a three-item list, where the loop finishes with `s == ()`).

Tracing `getitem_link(Link(3, Link(4, Link(5))), 1)`: `i` is 1, so one iteration runs, `s` becomes box 2 and `i` becomes 0. The loop exits, `s` is a `Link`, and `s.first` is 4.

### Example 5: The same three functions, recursively

```python
four = Link(1, Link(2, Link(3, Link(4))))

def len_link_recursive(s: LinkedList) -> int:
    """>>> len_link_recursive(four)
    4
    """
    if not isinstance(s, Link):
        return 0
    return 1 + len_link_recursive(s.rest)

def getitem_link_recursive(s: LinkedList, i: int):
    """>>> getitem_link_recursive(four, 2)
    3
    """
    assert isinstance(s, Link), 'Index out of range'
    if i == 0:
        return s.first
    return getitem_link_recursive(s.rest, i - 1)

def sum_link_recursive(s: LinkedList[float]) -> float:
    """>>> sum_link_recursive(four)
    10
    """
    if not isinstance(s, Link):
        return 0
    return s.first + sum_link_recursive(s.rest)
```

`len_link_recursive(four)` expands to `1 + (1 + (1 + (1 + len_link_recursive(()))))`, and the innermost call hits the base case and returns 0, giving 4. The recursive structure exactly mirrors the data structure: one frame per `Link`, plus one for `()`.

`getitem_link_recursive(four, 2)`: the index counts down as we walk in. Frame 1 has `i = 2` on box 1, frame 2 has `i = 1` on box 2, frame 3 has `i = 0` on box 3, which returns `s.first`, that is 3. The single assert at the top covers both out-of-range situations because every call re-checks before touching `.first` or `.rest`.

`sum_link_recursive(four)` is `1 + 2 + 3 + 4 + 0 = 10`.

### Example 6: `range_link`, three ways

```python
def range_link(start: int, end: int) -> LinkedList[int]:
    """>>> print(range_link(3, 7))
    (3 4 5 6)
    """
    s = ()
    k = end - 1
    while start <= k:
        s = Link(k, s)
        k = k - 1
    return s
```

Iterative, building from the back. For `range_link(3, 7)`: `s = ()`, `k = 6`. Then `s = Link(6, ())` and `k = 5`; `s = Link(5, Link(6))` and `k = 4`; `s = Link(4, Link(5, Link(6)))` and `k = 3`; `s = Link(3, Link(4, Link(5, Link(6))))` and `k = 2`. Now `3 <= 2` is false, so return. Prints `(3 4 5 6)`. Each iteration allocates exactly one new box and reuses the whole previously built chain as its `rest`.

```python
def range_link_recursive(start: int, end: int) -> LinkedList[int]:
    """>>> print(range_link_recursive(3, 7))
    (3 4 5 6)
    """
    if start >= end:
        return ()
    else:
        return Link(start, range_link_recursive(start + 1, end))
```

Recursive, building from the front in the *source text* but still allocating from the back at *run time*: the recursive call must return before `Link(start, ...)` can be evaluated, so the innermost box (holding 6) is created first. The base case `start >= end` returns `()`, which satisfies the rule that the second argument to `Link` must be a linked list.

```python
def range_link_tail(start: int, end: int) -> LinkedList[int]:
    def f(k, s):
        if k < start:
            return s
        else:
            return f(k-1, Link(k, s))
    return f(end-1, ())
```

This is the answer to the slide's fill-in-the-blanks question ("Can recursion start from `k = end - 1` and work backward using `k = k - 1`?"). Compare directly with the `while` version: `k < start` is the negation of the loop condition `start <= k`; `Link(k, s)` is the loop body's assignment; `k-1` is the decrement; `f(end-1, ())` is the initialization. The call sequence for `range_link_tail(3, 7)` is `f(6, ())`, `f(5, (6))`, `f(4, (5 6))`, `f(3, (4 5 6))`, `f(2, (3 4 5 6))`, and that last call returns the accumulator unchanged. Note that the inner `f` closes over `start`, so `start` does not need to be a parameter.

### Example 7: `extend_link`, concatenation

```python
def extend_link(s: LinkedList, t: LinkedList) -> LinkedList:
    """Return a linked list with the items of s followed by those of t.

    >>> print(extend_link(four, Link(5, Link(6))))
    (1 2 3 4 5 6)
    """
    if not isinstance(s, Link):
        return t
    else:
        return Link(s.first, extend_link(s.rest, t))
```

The base case is the key insight: when `s` runs out, the answer is just `t`. So the result is a *fresh copy of every box of `s`*, with the last copy's `rest` pointing at the original `t`. In box terms: four new boxes holding 1, 2, 3, 4, then an arrow into the existing `(5 6)` chain. `s` itself is not modified and `t` is not copied. This "copy the first argument, share the second" behavior is characteristic of linked list concatenation.

### Example 8: `map_link` and `filter_link`

```python
def map_link(f, s: LinkedList) -> LinkedList:
    """>>> print(map_link(lambda x: x * x, four))
    (1 4 9 16)
    """
    if not isinstance(s, Link):
        return s
    else:
        return Link(f(s.first), map_link(f, s.rest))
```

One new box per input box, holding `f` of the original value. The base case returns `s` rather than `()`; these are the same thing here, since reaching the base case means `s` already *is* the empty linked list. Returning `()` explicitly would be equally correct.

```python
def filter_link(f, s: LinkedList) -> LinkedList:
    """>>> print(filter_link(lambda x: x % 2 == 0, range_link(1, 10)))
    (2 4 6 8)
    """
    if not isinstance(s, Link):
        return s
    else:
        kept = filter_link(f, s.rest)
        if f(s.first):
            return Link(s.first, kept)
        else:
            return kept
```

Here the recursive result is computed first and named `kept`, then used in both branches. If `f(s.first)` is true we prepend a new box; if not, we return `kept` directly, which simply omits this item from the result. Tracing `filter_link(even, (1 2 3 4 5 6 7 8 9))`: the odd items each return their `kept` unchanged, and the even items each add one box, producing `(2 4 6 8)`.

### Example 9: `join_link`, building a string

```python
def join_link(s: LinkedList, separator: str) -> str:
    """>>> join_link(four, " + ")
    '1 + 2 + 3 + 4'
    """
    if not isinstance(s, Link):
        return ""
    elif not isinstance(s.rest, Link):
        return str(s.first)
    else:
        return str(s.first) + separator + join_link(s.rest, separator)
```

This needs **two** base cases because the separator goes *between* items, not after each one. The second case (`s.rest` is empty, so `s` is the last link) returns the item with no trailing separator. Note that the second case may only test `s.rest` because the first case already established that `s` is a `Link`. For `four` and `" + "`: `"1" + " + " + ("2" + " + " + ("3" + " + " + "4"))`, giving `'1 + 2 + 3 + 4'`.

### Example 10: `partitions`, a Python list of linked lists

```python
def partitions(n: int, m: int) -> list[LinkedList[int]]:
    """Return a list of partitions of n using parts of up to m.
    Each partition is represented as a linked list.
    """
    if n == 0:
        return [()]  # A list containing the empty partition
    elif n < 0 or m == 0:
        return []
    else:
        with_m = [Link(m, s) for s in partitions(n-m, m)]
        without_m = partitions(n, m-1)
        return with_m + without_m
```

The decomposition, straight from the slide: summing to `n` with pieces up to `m` means either using at least one `m` (then sum to `n-m`, still allowed to use `m` again) or not using `m` at all (then sum to `n` with pieces up to `m-1`).

- `with_m`: each `s` returned by `partitions(n-m, m)` sums to `n-m`; prepending `m` with `Link(m, s)` yields a linked list summing to `n`. The list comprehension does this for every partition of `n-m`.
- `without_m`: already a list of partitions of `n`, no modification needed.
- `with_m + without_m` is **Python list concatenation**, producing one list containing all partitions from both branches.

Base cases:
- `n == 0`: exactly one partition, the empty one, so return `[()]`, a list of length 1 whose single element is the empty linked list.
- `n < 0 or m == 0`: no partitions, so return `[]`, a list of length 0.

A full trace of `partitions(6, 4)`, using `(a b c)` for linked lists:

- `partitions(2, 4)` reduces through `m = 4, 3` (both overshoot) to `partitions(2, 2) = [(2), (1 1)]`, so `with_m` for `(6, 4)` is `[(4 2), (4 1 1)]`.
- `partitions(6, 3)` gives `[(3 3), (3 2 1), (3 1 1 1)]` from its `with_m`, plus `partitions(6, 2) = [(2 2 2), (2 2 1 1), (2 1 1 1 1), (1 1 1 1 1 1)]`.

Concatenating in order:

```python
def print_partitions(n: int, m: int) -> None:
    """>>> print_partitions(6, 4)
    4 + 2
    4 + 1 + 1
    3 + 3
    3 + 2 + 1
    3 + 1 + 1 + 1
    2 + 2 + 2
    2 + 2 + 1 + 1
    2 + 1 + 1 + 1 + 1
    1 + 1 + 1 + 1 + 1 + 1
    """
    for p in partitions(n, m):
        print(join_link(p, " + "))
```

Two orderings are visible here and they are worth separating. *Within* each partition the parts come out non-increasing (`4 + 1 + 1`), because `m` is prepended to partitions that only use parts of size at most `m`. *Across* partitions, the largest-first branch (`with_m`) is concatenated before the `without_m` branch, so the output is ordered by decreasing largest part. (The slide's phrase "in increasing order" refers to the standard constraint that keeps each partition from being generated in multiple orders, not to the direction in which the parts are printed.)

Note also that `print_partitions` is purely a driver: it has no recursion of its own, it just iterates over the Python list and formats each linked list with `join_link`.

---

## Common Pitfalls

1. **Accessing `.first` or `.rest` without checking.** `()` is a tuple and has no such attributes, so `s.rest.first` on a one-item linked list raises `AttributeError: 'tuple' object has no attribute 'first'`. Always guard with `isinstance(s, Link)`.
2. **Passing a non-linked-list as `rest`.** `Link(4, 5)` constructs happily (annotations are not enforced) but is not a valid `LinkedList`. The error surfaces later, for instance as the `assert remaining == ()` failure inside `format_link` when you try to print it. Remember: the second argument must be `()`, a `Link`, a name bound to a linked list, or a call returning a linked list.
3. **Writing `Link(3, 4, 5)` or expecting `Link([3, 4, 5])`.** `Link` takes at most two arguments. A three-item linked list is `Link(3, Link(4, Link(5)))`. And `Link([3, 4, 5])` is a one-item linked list whose single item happens to be a Python list.
4. **Confusing `Link(s)` with `s`.** `Link(s)` nests: it prints `((3 4 5))`, not `(3 4 5)`. A one-item linked list containing a linked list is not the same as that linked list.
5. **Using `len`, `in`, slicing, or `s[i]` on a linked list.** None of these work. Use `len_link`, a traversal, and `getitem_link`. Indexing is linear time, not constant.
6. **Forgetting that indexing walks the chain.** Writing a loop that calls `getitem_link(s, i)` for each `i` turns a linear traversal into a quadratic one.
7. **Mixing up the two "empty" results.** In `longest`, `None` (no solution) versus `[]` (the empty solution). In `partitions`, `[]` (no partitions) versus `[()]` (one partition, the empty one). Swapping these is a classic off-by-a-whole-answer bug.
8. **Using truthiness instead of `isinstance`.** `[]`, `()`, and `0` are all falsy, so `if minus_first:` would wrongly reject the valid empty answer. The slides call out `isinstance` for exactly this reason.
9. **Thinking `s = Link(4, s)` mutates `s`.** It creates a new `Link` and rebinds the name. The old chain is unchanged and becomes the `rest` of the new box. Likewise, reassigning the parameter `s` inside a loop does not affect the caller.
10. **Dropping the tail in `filter_link`.** If you return `()` when `f(s.first)` is false, you throw away every later item. The rejected item must be skipped, not the rest of the list.
11. **A single base case in `join_link`.** With only the `not isinstance(s, Link)` case you get a trailing separator (`'1 + 2 + 3 + 4 + '`). The "last link" case is required.
12. **In `partitions`, recursing with the wrong `m`.** `with_m` must use `partitions(n-m, m)`, keeping `m` available for reuse, because a partition may contain `m` more than once (for example `2 + 2 + 2`). Using `m-1` there would only find partitions with distinct parts.
13. **Forgetting that the items of `with_m` sum to `n-m`, not `n`.** The slide annotates this explicitly; it is why `Link(m, p)` is needed rather than just returning the inner result.
14. **Assuming `print(s)` and `s` show the same thing.** In the REPL, `s` uses the data class `repr` (`Link(first=3, ...)`) while `print(s)` uses `__str__` (`(3 4 5)`). Exam questions distinguish these.
15. **Tie-breaking in `max(..., key=len)`.** On equal lengths, the first argument wins, so `longest` prefers the sublist that includes `s[0]`. Do not assume a different tie-break.
16. **Treating "sublist" as "contiguous".** `longest` searches subsequences; `[4, 1, -1, 2, -1]` skips the 3 in the middle.
17. **Mutation through shared structure.** (extra context) Because `@dataclass` makes `Link` mutable, two linked lists that share a tail see each other's changes: after `a = Link(1, s)` and `b = Link(2, s)`, assigning `s.first = 99` changes what both `a` and `b` print. All the lecture's functions build fresh boxes instead of mutating, which is why sharing is safe there.

---

## Likely Exam Points

### 1. Evaluating attribute chains and recognizing the structure

**Practice.** Given `s = Link(1, Link(Link(2, Link(3)), Link(4)))`, what are `s.rest.first.rest.first` and `print(s)`?

**Answer.** `s.rest` is `Link(Link(2, Link(3)), Link(4))`. Its `first` is the nested linked list `Link(2, Link(3))`. That object's `rest` is `Link(3)`, whose `first` is `3`. So the expression is `3`. `print(s)` outputs `(1 (2 3) 4)`.

### 2. `str` versus `repr`

**Practice.** For `t = Link(1, Link(2))`, give the output of `t` in the REPL and of `print(t)`.

**Answer.** `t` displays `Link(first=1, rest=Link(first=2, rest=()))` (the data class `repr`). `print(t)` displays `(1 2)` (via `__str__`, which calls `format_link`).

### 3. Filling in a recursive linked list function

**Practice.** Complete a function returning a linked list containing only the items of `s` that are greater than `x`:

```python
def greater(s: LinkedList[int], x: int) -> LinkedList[int]:
    if not isinstance(s, Link):
        return ______
    elif s.first > x:
        return ______
    else:
        return ______
```

**Answer.** `()` (or `s`); `Link(s.first, greater(s.rest, x))`; `greater(s.rest, x)`. This is `filter_link` specialized, and the third branch is the critical one: skip this item but keep processing the rest.

### 4. Converting between iteration and recursion

**Practice.** Rewrite `sum_link_recursive` as an iterative function without using recursion, and say what each piece corresponds to.

**Answer.**

```python
def sum_link(s):
    total = 0
    while isinstance(s, Link):
        total += s.first
        s = s.rest
    return total
```

The base case `return 0` becomes the initial value `total = 0`; the combination `s.first + ...` becomes `total += s.first`; the recursive argument `s.rest` becomes the reassignment `s = s.rest`; the guard `not isinstance(s, Link)` becomes the negated loop condition.

### 5. The accumulator / tail-recursive pattern

**Practice.** Fill in the blanks so `f` builds the list by counting down (this is the slide's question):

```python
def range_link(start: int, end: int) -> LinkedList[int]:
    def f(s, k):
        if __________:
            return s
        else:
            return ___________________
    return _____________
```

**Answer.** `k < start`; `f(Link(k, s), k-1)`; `f((), end-1)`. The accumulator `s` starts empty and grows at the front as `k` decreases, so the final list is in increasing order. (The lecture code writes the arguments in the order `f(k, s)` with the call `f(end-1, ())`, which is equivalent.)

### 6. Building a linked list by repeated prepending

**Practice.** What does the following print, and how many `Link` objects are created?

```python
s = ()
for k in [1, 2, 3]:
    s = Link(k, s)
print(s)
```

**Answer.** `(3 2 1)`. Three `Link` objects are created. Prepending reverses the order in which items are added, which is why `range_link` counts *down* from `end - 1` in order to produce an increasing list.

### 7. `partitions` base cases and the list comprehension

**Practice.** Why does `partitions` return `[()]` when `n == 0` rather than `[]`? And what would go wrong if `with_m` were written `[Link(m, s) for s in partitions(n-m, m-1)]`?

**Answer.** `[()]` is a list containing one partition (the empty one), recording that there is exactly one way to sum to 0. Returning `[]` would say there is no way to finish, and since every successful recursive path ends at `n == 0`, `partitions` would return `[]` for every input. Using `m-1` in `with_m` would forbid reusing `m`, so partitions with repeated largest parts (for example `(2 2 2)` for `n = 6`) would be missing; the function would only generate partitions into distinct parts.

### 8. Tracing `partitions` output

**Practice.** List the values of `partitions(4, 2)` in order, written in printed form.

**Answer.** `(2 2)`, `(2 1 1)`, `(1 1 1 1)`. The `with_m` branch (`m = 2`) prepends 2 to each of `partitions(2, 2) = [(2), (1 1)]`, giving `(2 2)` and `(2 1 1)`; then `without_m = partitions(4, 1) = [(1 1 1 1)]` is appended.

### 9. Include/exclude tree recursion with a sentinel return value

**Practice.** In `longest`, why is the test `isinstance(minus_first, list)` rather than `minus_first is not None`, and what does the function return for `longest([], 0)` versus `longest([], 3)`?

**Answer.** Both tests work here since the only two possible result types are `list` and `None`; `isinstance(..., list)` is preferred because it says positively "this is an answer" and is immune to the trap that `[]` is falsy. `longest([], 0)` returns `[]` (the empty sublist sums to 0). `longest([], 3)` returns `None` (no sublist of the empty list sums to 3).

### 10. Why `extend_link` behaves asymmetrically

**Practice.** After `a = Link(1, Link(2))`, `b = Link(3)`, and `c = extend_link(a, b)`, is `c.rest.rest` the same object as `b`? How many new `Link` objects were created?

**Answer.** Yes, `c.rest.rest is b` is `True`: the base case returns `t` unchanged, so the result's tail is literally `b`. Two new `Link` objects were created, one copy for each link of `a`. Neither `a` nor `b` was modified.

### 11. Distinguishing linked lists from Python lists in one problem

**Practice.** What is the type of `partitions(5, 3)`, and how do you get the number of parts in its first partition?

**Answer.** It is a Python `list` whose elements are `LinkedList[int]` values, so `len(partitions(5, 3))` is valid but `len(partitions(5, 3)[0])` is not. The number of parts in the first partition is `len_link(partitions(5, 3)[0])`.

---

## Summary

- A list can always be viewed as a first item `s[0]` and the rest `s[1:]`; searching for a sublist means trying both including and excluding `s[0]`, which is tree recursion.
- `longest` returns `list[int] | None`, so `[]` (the empty answer) and `None` (no answer) must be distinguished with `isinstance(x, list)`, and `max(a, b, key=len)` picks the longer result (ties go to the first argument).
- **A linked list is either empty or a first value and the rest of the linked list.** Empty is `()`; non-empty is a `Link` with `first` and `rest`, where `rest` defaults to `()`.
- `type LinkedList[T] = Link[T] | tuple[()]` names the union; `@dataclass class Link[T]` supplies `__init__`, `__repr__`, and `__eq__`; the lecture adds `__str__` via `format_link` so `print` gives `(3 4 5)` while the REPL repr gives `Link(first=3, rest=...)`.
- Always test `isinstance(s, Link)` before reading `.first` or `.rest`, and always pass a genuine linked list as the second argument to `Link`.
- Linked lists are built back to front: `s = Link(5, ())`, then `s = Link(4, s)`, then `s = Link(3, s)` gives `(3 4 5)`. Each step allocates one box and shares the existing chain as its tail; nothing is mutated.
- Every traversal has an iterative form (`while isinstance(s, Link): ... s = s.rest`) and a recursive form (base case on the empty list, recurse on `s.rest`): `len_link`, `getitem_link`, `sum_link`, `range_link` all appear both ways.
- A `while` loop that counts backward translates into recursion with an **accumulator** parameter: `range_link_tail` uses `f(k-1, Link(k, s))` with base case `k < start` returning `s`.
- `extend_link` copies the first argument and shares the second; `map_link` makes one new box per item; `filter_link` computes the recursive result first and either prepends or returns it unchanged; `join_link` needs a second base case for the last link so the separator only appears between items.
- Indexing, length, and concatenation on linked lists are all linear time; there is no slicing, no `len`, and no `s[i]`.
- `partitions(n, m)` returns a Python list of linked lists: `[Link(m, p) for p in partitions(n-m, m)] + partitions(n, m-1)`, with base cases `[()]` for `n == 0` (one way) and `[]` for `n < 0 or m == 0` (no ways). Each partition prints with its parts in non-increasing order via `join_link`.
