<!-- Mon, Sep 28, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 14: Linked Lists

This lecture finishes the container/type-hint material (tuples and type hints for containers, including recursively defined types), works one more tree-recursion-over-lists problem (`longest`), and then introduces the **linked list**: a recursively defined sequence built from `Link` objects, where each `Link` has a `first` value and a `rest` that is itself a linked list (with the empty tuple `()` standing in for the empty linked list). Because Python has no built-in linked list, we define our own `Link` data class, and then we re-implement all the familiar sequence operations (length, indexing, sum, range, concatenation, map, filter, join) both iteratively (with the `s = s.rest` pattern) and recursively (with the `Link(x, recursive_call(...))` pattern). The two big habits to internalize: **always check `isinstance(s, Link)` before touching `.first` or `.rest`**, and **whenever you call `Link(x, r)`, the second argument `r` must itself evaluate to a linked list**. The lecture closes with `partitions`, which returns a Python list of linked lists, combining a list comprehension with the linked-list constructor.

---

## Key Concepts

### 1. Tuples (lead-in material from the recording)

A tuple is another sequence type, similar to a list, but immutable.

- Written with parentheses: `(4, 5, 6, 7)`. The parentheses are technically optional (`4, 5, 6, 7` is also a tuple), but you should always write them for readability.
- Indexing and slicing work just like lists: `s[0]`, `s[1:]`.
- `list(s)` converts a tuple to a list with the same elements.
- `+` concatenates tuples; `*` repeats them, just like lists.
- The empty tuple is `()`. A one-element tuple **needs a trailing comma**: `(5,)`. Without the comma, `(5)` is just the integer `5`.
- **Tuples can be dictionary keys**; lists and dictionaries cannot. This matters when you want a composite key. (Caveat from lecture: if the tuple itself contains a list or dict, it can no longer be used as a key.)
- Multiple assignment is secretly tuples. Given `x = 3`, the statement `x, y = 5, x + 1` builds the tuple `(5, 4)` first (the right-hand side is fully evaluated *before* any name is rebound), then unpacks it, so `y` is `4`, not `6`.
- Returning "multiple values" from a function returns a tuple.

### 2. Type hints for containers

- `list[int]`: a list whose elements are all `int`. `list[list[int]]`: a list of lists of ints. **There is no way to express the length of a list in its type**, so a `list[int]` hint says nothing about whether the list is empty.
- `dict[int, str]`: integer keys, string values. `dict[int, list[int]]`: integer keys, values that are lists of ints.
- **Tuple hints are different**: they are positional and fixed-length. `tuple[int, int]` means exactly two elements, both ints. `tuple[str, float]` means a string then a float. For a tuple of arbitrary length and contents, just write `tuple`.
- `tuple[()]` is the type that matches only the empty tuple. This is exactly what the linked-list type alias uses.

### 3. Recursive type aliases

The lecture motivated recursive types with "nested list of integers": a list containing integers and nested lists of integers, to any depth. Several naive hints fail:

- `list[int | list]`: too permissive, the inner list could contain a string.
- `list[int | list[int]]`: too restrictive, rules out a list of lists of lists of ints.
- `list[int | list[int | list]]`: still too permissive at depth 3.

The fix is to name the type and use it inside its own definition:

```python
type NestedList = list[int | NestedList]   # (exact slide form not in the provided text; reconstructed from the recording)
```

This same trick gives us the linked-list type:

```python
type LinkedList[T] = Link[T] | tuple[()]
```

"A `LinkedList` of `T` is either a `Link` of `T` (non-empty) or the empty tuple."

### 4. Tree recursion over lists: first item and the rest

A key reframing used throughout this lecture: **a Python list `s` can be viewed as a first item `s[0]` plus the rest `s[1:]`**. Search problems over sublists then become tree recursion with two branches:

- include `s[0]` in the answer, and recursively solve the smaller problem on `s[1:]`
- exclude `s[0]`, and recursively solve on `s[1:]` with the same goal

This is exactly the shape of `longest` below, and the same include/exclude idea appears in `partitions` (use an `m` or do not).

### 5. Linked list structure

> A linked list is either empty or a first value and the rest of the linked list.

The sequence `3, 4, 5` is three `Link` instances chained together:

- A `Link` with `first: 3` whose `rest` is ...
- a `Link` with `first: 4` whose `rest` is ...
- a `Link` with `first: 5` whose `rest` is `()`.

Box-and-pointer in words: draw three two-compartment boxes side by side. The left compartment holds the value (3, then 4, then 5); the right compartment holds an arrow to the next box. The last box's right compartment gets a slash (the drawing convention for the empty linked list) rather than an arrow.

The crucial insight the lecture emphasized: **each individual `Link` in the chain represents an entire sequence**. The last `Link` *is* the linked list `(5)`. The second-to-last *is* the linked list `(4 5)`. The first *is* `(3 4 5)`. The `rest` attribute is not "a pointer to one more element", it is "a whole linked list containing all the remaining elements".

Constructed in one expression:

```python
Link(3, Link(4, Link(5, ())))
# or, using the default value for rest:
Link(3, Link(4, Link(5)))
```

**How to print it:** `(3 4 5)`. Parentheses distinguish it from a built-in list, and no commas distinguish it from a tuple. This display convention comes from the Lisp programming language.

### 6. The `Link` data class

```python
type LinkedList[T] = Link[T] | tuple[()]

@dataclass
class Link[T]:
    "A Link has a first value of type T and the rest of the linked list."
    first: T
    rest: LinkedList[T] = ()  # rest defaults to an empty linked list

    def __str__(self):
        return format_link(self)
```

Points worth noticing:

- A `Link` object is **never empty**: it always has a `first` and a `rest`. Emptiness is represented by `()`, which is a different type entirely. That is why `LinkedList` is a union of two types.
- `rest` has default value `()`, so `Link(5)` is a one-element linked list.
- `T` is a **type variable** describing the type of the items in the list. `Link(3, Link(4, Link(5)))` has type `Link[int]`, and also `LinkedList[int]`, and you may also just write `Link` or `LinkedList` without parameterizing.
- Because it is a dataclass, you get `__repr__` and `__eq__` for free: `repr(s)` is `Link(first=3, rest=Link(first=4, rest=Link(first=5, rest=())))`, and `s.rest.rest.rest == ()` is `True`. (extra context: the generated `__eq__` compares structurally, so two separately built `Link(3, Link(4))` objects are `==` to each other even though they are distinct objects.)
- `__str__` delegates to `format_link`, which is why `print(s)` gives the compact `(3 4 5)`.

### 7. Two rules for working with linked lists

**Rule 1: check before you access.** For a `LinkedList s`, confirm it is a `Link` before touching `.first` or `.rest`:

```python
if isinstance(s, Link):          if not isinstance(s, Link):
    ...  # s.first, s.rest OK        ...  # s is ()
                                 else:
                                     ...  # s.first, s.rest OK
```

Either arrangement works; the `not isinstance(...)` version is the common shape of a recursive base case, since repeated `s = s.rest` or recursion on `s.rest` eventually reaches `()`. A type checker in your editor understands this narrowing too, so `.first` and `.rest` stop being flagged inside the guarded block. If emptiness is an *error* rather than a base case, use `assert isinstance(s, Link), 'Index out of range'`.

**Rule 2: the second argument to `Link` must be a linked list.** Legal forms:

- `Link(4, ())` or the shorthand `Link(4)`
- `Link(4, Link(...))`
- `Link(4, s)` where `s` is a name bound to a linked list
- `Link(4, f(...))` where `f` returns a linked list, very commonly a recursive call

### 8. Iteration versus recursion, and the order things happen

Both work, but they build and process in **opposite orders**, which is the single most important structural fact in this lecture.

- **Iteration** naturally works *backward*: start with `s = ()` and repeatedly do `s = Link(k, s)`, adding each new item to the *front*. To get `(3 4 5)` this way you must add 5, then 4, then 3.
- **Recursion** naturally works *forward*: `Link(start, range_link_recursive(start + 1, end))`. Because operand expressions are evaluated before the call, the rest of the list `(4 5)` is fully constructed *before* the `Link` whose `first` is 3 is created.

So: "to make a linked list, you must already have made the rest of it." If you want to reason from the first element onward, recursion is natural. If you want to build from the back forward, the `s = Link(k, s)` loop is natural.

---

## Definitions

- **Tuple**: an immutable built-in sequence type, written with parentheses; supports indexing, slicing, `+`, `*`; usable as a dictionary key (when its contents are themselves hashable).
- **`tuple[()]`**: the type hint that matches only the empty tuple.
- **Type variable (`T`)**: a placeholder in a generic class or type alias standing for the type of the contained items, e.g. the `T` in `Link[T]` and `LinkedList[T]`.
- **Recursive type alias**: a `type` definition that mentions its own name, needed for types whose values nest to arbitrary depth (nested lists of ints, linked lists).
- **Linked list**: a sequence that is either empty or consists of a first value and a rest, where the rest is itself a linked list.
- **`Link`**: the data class describing a *non-empty* linked list, i.e. one link in the chain; it has attributes `first` (the item) and `rest` (the remainder of the linked list, defaulting to `()`).
- **`LinkedList[T]`**: the type `Link[T] | tuple[()]`, i.e. a possibly-empty linked list of items of type `T`.
- **`first`**: attribute of a `Link` holding the element at index 0 of the sequence it represents.
- **`rest`**: attribute of a `Link` holding the entire linked list of all remaining elements.
- **Empty linked list**: represented in this course by the empty tuple `()`; drawn as a slash rather than an arrow.
- **Sublist (as used in `longest`)**: a subsequence obtained by choosing, for each item in order, whether to include it; the relative order of chosen items is preserved.
- **Mapping a function over a sequence**: applying that function to each element, producing a new sequence of the results.
- **Filtering a sequence**: keeping exactly those elements for which a one-argument predicate returns a true value.
- **Partition of `n` using parts up to `m`**: a multiset of positive integers, each at most `m`, that sums to `n`; here represented as a linked list of its parts.

---

## Worked Examples

### Example 1: `longest` (tree recursion over a Python list)

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

Step by step, and why each piece is there:

1. **Base case.** If `s` is empty, the only sublist available is `[]`, which sums to 0. So if `n == 0` return `[]` (a successful empty answer), otherwise return `None` (no sublist exists). Note that `[]` and `None` are very different results here: `[]` means "found it, the answer has no items", `None` means "impossible".
2. **Two recursive calls (the tree recursion).** `minus_first = longest(s[1:], n - s[0])` asks: among sublists of the rest, what is the longest one summing to `n - s[0]`? If one exists, prepending `s[0]` to it gives a sublist of `s` summing to `n`. `without_first = longest(s[1:], n)` asks the same question without using `s[0]` at all.
3. **Why `isinstance(..., list)` and not a truthiness test.** Each recursive call returns either a list or `None`, so you must distinguish "returned `None`" from "returned a list". And `[]` is a *valid* answer that is falsy, so `if minus_first:` would be wrong. `isinstance(minus_first, list)` is the correct test.
4. **Combining.** If both branches succeeded, return the longer one: `max(with_first, without_first, key=len)`. The `key=len` makes `max` compare by length rather than by comparing the lists element-wise. If only one succeeded, return that one. If neither, `without_first` is `None` and returning it propagates the failure.

Tracing the docstring example `longest([4, 1, 3, -1, 2, -1], 5)`:

- `s[0]` is 4, so `minus_first = longest([1, 3, -1, 2, -1], 1)`, which finds `[1, -1, 2, -1]` (sums to 1, length 4, the longest such).
- `with_first = [4] + [1, -1, 2, -1] = [4, 1, -1, 2, -1]`, which sums to 5 and has length 5.
- `without_first = longest([1, 3, -1, 2, -1], 5)` finds `[1, 3, -1, 2]` (sums to 5, length 4).
- `max(..., key=len)` picks the length-5 answer: `[4, 1, -1, 2, -1]`.

Two notes on the provided material: the slide annotation "`[4, 1, 3, -1, 2]` is a way to make 5" does not sum to 5 (it sums to 9) and appears to be a slide typo for `[4, 1, -1, 2, -1]`; and the docstring in `14.py` mistakenly calls the function `count_sums` in its doctests, so those doctests would not run as written.

(extra context) `max` returns the **first** maximal element on a tie, so when `with_first` and `without_first` have equal length, `with_first` wins. Any longest sublist is an acceptable answer, so this is fine, but it does determine which one you get.

### Example 2: Building and traversing a linked list

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
Link(first=5, rest=Link... )   # actually: Link(first=5, rest=())
>>> print(s)
(3 4 5)
```

Reading this as box-and-pointer: `s` names the leftmost box. `s.rest` names the middle box, which by itself is the linked list `(4 5)`. `s.rest.rest` names the rightmost box, the linked list `(5)`. `s.rest.rest.rest` is `()`, the empty linked list, the slash at the end of the chain.

So the practical lesson: if you `print(s)` and see `(3 4 5)`, the way to reach the `4` is `s.rest.first`, not `s[1]`. **Linked lists do not support indexing, slicing, `len`, or iteration with `for`** unless you write those operations yourself.

Nesting works too, because `first` can be any value, including another linked list:

```python
>>> print(Link(s))
((3 4 5))
>>> print(Link(3, Link(Link(4, Link(5)), Link(6))))
(3 (4 5) 6)
```

`Link(s)` is a one-element linked list whose single element is the linked list `(3 4 5)`, so it prints as `((3 4 5))`. This works because `format_link` calls `str` on each `first`, and `str` of a `Link` is its parenthesized form.

### Example 3: `format_link` (how printing works)

```python
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

This is the canonical iterative pattern. Start with the open paren and the first item (safe, because `format_link` is only called on a `Link`). Then walk down the chain with `remaining = remaining.rest`, appending each item, stopping when `remaining` is no longer a `Link`. The `assert` after the loop catches the case where someone built `Link(4, 5)`: the loop would exit because `5` is not a `Link`, but `5 != ()`, so the assertion fires with a clear message.

### Example 4: `len_link` and `getitem_link`, iteratively

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

The loop condition *is* the emptiness check, so `s.rest` inside the body is always safe. Rebinding the parameter `s` is fine and idiomatic here: it only changes the local frame's `s`, not the caller's list, and the list itself is never modified.

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

Walking through `getitem_link(Link(3, Link(4, Link(5))), 1)` as the lecture did:

- In the new frame, `s` is the whole list `(3 4 5)` and `i` is 1.
- `i > 0`, so assert that `s` is a `Link` (it is), then `s = s.rest` rebinds `s` to the linked list `(4 5)`, and `i` becomes 0.
- The loop exits. The second `assert` checks that the *new* `s` is still non-empty: this is a separate check, because the first assertion only told us the *old* `s` was non-empty, and a one-element list has an empty `rest`.
- Return `s.first`, which is `4`. Note this is not the original first element, because `s` was rebound.

**Why there are two asserts:** every access to `.rest` needs a prior check, and every access to `.first` needs a prior check, and rebinding `s` invalidates the previous check. This is the general discipline for linked lists.

### Example 5: `sum_link`, iteratively and recursively, and the order of operations

```python
def sum_link(s: LinkedList[float]) -> float:
    total = 0
    while isinstance(s, Link):
        total += s.first
        s = s.rest
    return total

def sum_link_recursive(s: LinkedList[float]) -> float:
    if not isinstance(s, Link):
        return 0
    return s.first + sum_link_recursive(s.rest)
```

Both return 12 for `Link(3, Link(4, Link(5)))`, but they add in opposite orders:

- Iterative: `0 + 3 = 3`, then `3 + 4 = 7`, then `7 + 5 = 12`. Front to back.
- Recursive: the outer call knows it will add 3 to something but must evaluate the recursive call first; likewise for 4; the innermost call returns 0; then `5 + 0 = 5`, then `4 + 5 = 9`, then `3 + 9 = 12`. Back to front.

For addition the order does not matter, but for operations that are not associative or commutative, or for building a linked list (where you need the rest before you can make the `Link`), it matters a great deal.

Recursive versions of the other two:

```python
def len_link_recursive(s: LinkedList) -> int:
    if not isinstance(s, Link):
        return 0
    return 1 + len_link_recursive(s.rest)

def getitem_link_recursive(s: LinkedList, i: int):
    assert isinstance(s, Link), 'Index out of range'
    if i == 0:
        return s.first
    return getitem_link_recursive(s.rest, i - 1)
```

In `getitem_link_recursive`, the assert comes first and covers both the `s.first` on the next line and the `s.rest` in the recursive call. Each recursive call shrinks both the list and the index together, so index 0 of the shortened list is the right target.

### Example 6: `range_link` three ways

```python
def range_link(start: int, end: int) -> LinkedList[int]:
    """Return a linked list containing the items of range(start, end).

    >>> print(range_link(3, 7))
    (3 4 5 6)
    """
    s = ()
    k = end - 1
    while start <= k:
        s = Link(k, s)
        k = k - 1
    return s
```

Walking `range_link(3, 6)` (which should produce `(3 4 5)`):

- `s = ()`, `k = 5`.
- `3 <= 5`: `s = Link(5, ())`, so `s` is `(5)`. `k` becomes 4.
- `3 <= 4`: `s = Link(4, s)`, so `s` is `(4 5)`. `k` becomes 3.
- `3 <= 3`: `s = Link(3, s)`, so `s` is `(3 4 5)`. `k` becomes 2.
- `3 <= 2` is false; return `(3 4 5)`.

This must go from largest to smallest, because each new `Link` must be given an already-built rest.

```python
def range_link_recursive(start: int, end: int) -> LinkedList[int]:
    """Return a linked list containing the items of range(start, end).

    >>> print(range_link_recursive(3, 7))
    (3 4 5 6)
    """
    if start >= end:
        return ()
    else:
        return Link(start, range_link_recursive(start + 1, end))
```

Here the recursion counts *up*. `range_link_recursive(3, 6)` returns `Link(3, range_link_recursive(4, 6))`. The operand must be evaluated before `Link` is called, so `(4 5)` is built first, then the `Link` whose `first` is 3 is created around it. The code reads forward while the construction happens backward.

**The slide question:** can recursion instead start from `k = end - 1` and work backward with `k = k - 1`, mimicking the loop? Yes, with an accumulator (tail-recursive form). The slide's filled-in version:

```python
def range_link(start: int, end: int) -> LinkedList[int]:
    def f(s, k):
        if k < start:
            return s
        else:
            return f(Link(k, s), k-1)
    return f((), end-1)
```

The code file `14.py` has the same function with the parameters in the other order, which is equivalent:

```python
def range_link_tail(start: int, end: int) -> LinkedList[int]:
    def f(k, s):
        if k < start:
            return s
        else:
            return f(k-1, Link(k, s))
    return f(end-1, ())
```

Compare the three pieces to the loop: `k < start` is the negation of the loop condition `start <= k`; the accumulator `s` plays the role of the loop variable `s`, updated to `Link(k, s)`; the initial call `f((), end-1)` supplies the loop's initializers `s = ()` and `k = end - 1`. The base case returns the accumulated list rather than `()`, which is the hallmark of accumulator-style recursion.

### Example 7: `extend_link` (concatenation)

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

Reasoning with `s = (3 4)` and `t = (5 6)`:

- If `s` is empty, then "all of `s` followed by all of `t`" is just `t`, so return `t` directly. This is both the base case and a correct answer, not a placeholder.
- Otherwise the answer starts with `s.first`, and its rest is "all of `s.rest` followed by all of `t`", which is exactly a recursive call. `extend_link(s.rest, t)` returns a linked list, which is why it is legal as the second argument to `Link`.

(extra context) Only the links of `s` are rebuilt; the resulting list's tail *is* the original `t` object, shared rather than copied.

### Example 8: `map_link`, `filter_link`, and `join_link` (the comprehension pipeline)

For built-in lists you would write `[square(x) for x in range(1, 6) if odd(x)]`. For linked lists there is no comprehension syntax, so you write functions and compose them: `map_link(square, filter_link(odd, range_link(1, 6)))`.

```python
def map_link(f, s: LinkedList) -> LinkedList:
    """Return a linked list of f applied to each item of s.

    >>> print(map_link(lambda x: x * x, four))
    (1 4 9 16)
    """
    if not isinstance(s, Link):
        return s
    else:
        return Link(f(s.first), map_link(f, s.rest))
```

Note the base case `return s`. At that point `s` is `()`, so this is the same as `return ()`, but writing `return s` keeps the type checker satisfied and reads as "an empty input maps to an empty output". The recursive case applies `f` to `s.first` (safe, because we are in the guarded branch) and recursively maps the rest.

```python
def filter_link(f, s: LinkedList) -> LinkedList:
    """Return a linked list with the items of s for which f returns a true value.

    >>> print(filter_link(lambda x: x % 2 == 0, range_link(1, 10)))
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

The lecture developed this incrementally: filtering the rest is needed in *both* branches, so name it `kept` once. Then the only question is whether `s.first` survives. If `f(s.first)` is true, the answer is `Link(s.first, kept)`; note the rest is `kept`, **not** `s.rest`, because `s.rest` still contains the items that should be filtered out. If `f(s.first)` is false, the answer is just `kept`, which is shorter than `s` by one link.

```python
def join_link(s: LinkedList, separator: str) -> str:
    """Return a string of all items in s separated by separator.

    >>> join_link(four, " + ")
    '1 + 2 + 3 + 4'
    """
    if not isinstance(s, Link):
        return ""
    elif not isinstance(s.rest, Link):
        return str(s.first)
    else:
        return str(s.first) + separator + join_link(s.rest, separator)
```

Three cases, because a separator goes *between* items, not after the last one. Empty list gives `""`. A one-element list (checked with `not isinstance(s.rest, Link)`, which is safe only because the first branch already established that `s` is a `Link`) gives just the item with no separator. Otherwise, item, separator, then the joined rest.

Checking the pipeline in the interpreter:

```python
>>> print(range_link(1, 6))
(1 2 3 4 5)
>>> print(filter_link(odd, range_link(1, 6)))
(1 3 5)
>>> print(map_link(square, filter_link(odd, range_link(1, 6))))
(1 9 25)
```

which matches `[square(x) for x in range(1, 6) if odd(x)]`.

### Example 9: `partitions` returning linked lists

```python
def partitions(n: int, m: int) -> list[LinkedList[int]]:
    """Return a list of partitions of n using parts of up to m.
    Each partition is represented as a linked list.
    """
    if n == 0:
        return [()]   # A list containing just the empty partition
    elif n < 0 or m == 0:
        return []     # An empty list with no partitions at all
    else:
        with_m = [Link(m, s) for s in partitions(n-m, m)]
        without_m = partitions(n, m-1)
        return with_m + without_m
```

The recursive decomposition (same include/exclude shape as `longest`): summing to `n` with parts up to `m` means either using at least one `m` and then summing to `n - m` with parts up to `m`, or using no `m` at all and summing to `n` with parts up to `m - 1`.

Read the types very carefully, since **two different kinds of list** are in play:

- The return value is a **Python list** of partitions.
- Each **partition** is a `LinkedList[int]`.
- Base case `n == 0`: there is exactly one way to sum to 0, namely take nothing, so return `[()]`: a Python list of length 1 containing the empty linked list. Returning `()` or `[[]]` here would be wrong.
- Base case `n < 0 or m == 0`: no partitions at all, so return `[]`, a Python list of length 0.
- `with_m` uses a **list comprehension** over `partitions(n-m, m)`. Each `s` in that result is a partition summing to `n - m`; `Link(m, s)` puts the part `m` at the front, producing a partition summing to `n`. The slide's annotations make this contrast explicit: the overall value sums to `n`, while the thing iterated over sums to `n - m`.
- `with_m + without_m` uses `+` on Python lists to concatenate the two collections of partitions.

Because `Link(m, s)` places `m` at the front and every part of `s` is at most `m`, each partition is produced with its parts in non-increasing order, which is what prevents the same partition from being generated more than once in a different order. (The slide phrases this ordering constraint as "in increasing order"; the printed output lists the largest part first.)

```python
def print_partitions(n: int, m: int) -> None:
    """Print the partitions of n using parts up to size m.

    >>> print_partitions(6, 4)
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

Note how `print_partitions` iterates with a `for` loop over the Python list, but uses `join_link` to render each linked list. The output order follows directly from `with_m + without_m`: every partition containing a 4 comes first, then those with largest part 3, then 2, then 1.

Quick check of the `with_m` branch for `partitions(6, 4)`: `partitions(2, 4)` returns `[(2), (1 1)]`, so `with_m` is `[(4 2), (4 1 1)]`, giving the first two printed lines.

---

## Common Pitfalls

1. **Accessing `.first` or `.rest` on an empty linked list.** `().first` raises `AttributeError: 'tuple' object has no attribute 'first'`. Always guard with `isinstance(s, Link)` (or `assert`) first.
2. **Forgetting that rebinding invalidates your check.** After `s = s.rest`, you no longer know that `s` is a `Link`. `getitem_link` needs a second `assert` for exactly this reason.
3. **Checking `s.rest.first` without checking `s.rest`.** Two attribute accesses need two levels of guarantee. `join_link` checks `isinstance(s, Link)` before it dares to look at `s.rest`.
4. **Passing a non-linked-list as `rest`.** `Link(4, 5)` constructs an object without error (a dataclass does not enforce type hints at runtime), but it is not a valid linked list; `format_link`'s assertion is what finally catches it. The second argument must be `()`, a `Link`, a name bound to a linked list, or an expression that evaluates to one.
5. **Confusing `Link(s)` with `s`.** `Link(s)` is a one-element list whose element is the list `s`: it prints as `((3 4 5))`, not `(3 4 5)`.
6. **Confusing the printed form with a tuple or a list.** `(3 4 5)` has no commas and is not Python syntax you can type back in; `Link(first=3, rest=...)` is the actual `repr`.
7. **Treating a linked list like a built-in sequence.** `len(s)`, `s[1]`, `s[1:]`, and `for x in s` all fail. You must use `len_link`, `getitem_link`, `.rest`, and a `while isinstance(...)` loop or recursion.
8. **Trying to build a linked list front to back with a loop.** You cannot create `Link(3, ...)` until the rest exists, so an iterative build must start from the last element and work backward with `s = Link(k, s)`.
9. **In `filter_link`, using `s.rest` instead of the filtered rest.** `Link(s.first, s.rest)` would keep the unfiltered tail, silently reintroducing items that should have been dropped.
10. **In `partitions`, mixing up the two list layers.** `[()]` (one empty partition) versus `[]` (no partitions) is the classic trap; so is writing `partitions(n-m, m-1)` instead of `partitions(n-m, m)`, which would forbid reusing the same part size.
11. **Using truthiness instead of `isinstance` when `None` or `[]` is meaningful.** In `longest`, `if minus_first:` is wrong because `[]` is a legitimate falsy answer. (extra context: `if s:` happens to work for emptiness on linked lists, since `Link` instances are truthy and `()` is falsy, but `isinstance(s, Link)` is the pattern this course expects and the one that type checkers understand.)
12. **Forgetting the comma in a one-element tuple.** `(5)` is `5`; you need `(5,)`.
13. **Expecting `max(a, b)` to compare by length.** Without `key=len`, `max` compares lists element by element, which is not what `longest` wants.

---

## Likely Exam Points

### 1. Hand-evaluate attribute chains and `print` output

**Q.** Given `s = Link(1, Link(Link(2, Link(3)), Link(4)))`, what are `print(s)`, `s.rest.first.rest.first`, and `len_link(s)`?

**A.** `print(s)` displays `(1 (2 3) 4)`. `s.rest.first` is the linked list `(2 3)`, so `.rest.first` of that is `3`. `len_link(s)` is `3`, since the top-level list has three elements: `1`, the nested list `(2 3)`, and `4`.

### 2. Fill in the blanks in a recursive linked-list function

**Q.** Complete `count_link(s, x)`, returning how many items of `s` equal `x`.

```python
def count_link(s: LinkedList, x) -> int:
    if not isinstance(s, Link):
        return ____
    return ____ + count_link(s.rest, x)
```

**A.** First blank: `0`. Second blank: `(1 if s.first == x else 0)`. The base case is the empty list, which contains no matches; the recursive case adds 1 when the first item matches and recurses on the rest. (`s.first` is safe because the base case already returned for the empty case.)

### 3. Convert between iteration and recursion

**Q.** Rewrite `sum_link` recursively without an accumulator, then explain in what order the additions occur in each version.

**A.** `if not isinstance(s, Link): return 0` else `return s.first + sum_link_recursive(s.rest)`. The iterative version adds front to back (`0+3`, `+4`, `+5`), while the recursive version must finish the recursive call before adding, so additions happen back to front (`5+0`, `4+5`, `3+9`).

### 4. Accumulator / tail-recursive rewrite of a loop

**Q.** Fill in the blanks so that `f` mirrors the `while` loop version of `range_link`.

```python
def range_link(start, end):
    def f(s, k):
        if ________:
            return s
        else:
            return ________________
    return ____________
```

**A.** `k < start`; `f(Link(k, s), k-1)`; `f((), end-1)`. The condition is the negation of `start <= k`, `s` is the accumulator updated by prepending `k`, and the outer call supplies the loop's initial values.

### 5. Build a linked list, respecting "rest must be a linked list"

**Q.** Which of these are valid linked lists? `Link(1, 2)`, `Link(1, Link(2))`, `Link(1)`, `Link(1, [2, 3])`, `Link(1, range_link(2, 4))`.

**A.** Valid: `Link(1, Link(2))`, `Link(1)` (rest defaults to `()`), and `Link(1, range_link(2, 4))` (a function call returning a linked list). Invalid: `Link(1, 2)` and `Link(1, [2, 3])`, since the rest must be a `Link` or `()`. Both construct without an immediate error but break any function that walks the chain.

### 6. Where the emptiness checks must go

**Q.** Why does `getitem_link` need two `assert` statements rather than one?

**A.** The first assert protects the `s = s.rest` inside the loop. Once `s` is rebound, the earlier guarantee no longer applies: a non-empty `s` may have an empty `rest`. The second assert protects the final `s.first`. Every `.first`/`.rest` access needs a check that is still valid for the current binding of `s`.

### 7. Linked list versions of `map` and `filter`

**Q.** Write `map_link` and state what the base case returns and why.

**A.** As in Example 8. The base case returns `s`, which at that point is `()`: mapping over an empty list gives an empty list. The recursive case is `Link(f(s.first), map_link(f, s.rest))`, which is legal because the recursive call returns a linked list.

### 8. Tree recursion with include/exclude over a Python list

**Q.** In `longest`, why is the test `isinstance(minus_first, list)` rather than `minus_first is not None` or `if minus_first:`?

**A.** `isinstance(..., list)` and `is not None` are both correct here, since the only possible returns are a list or `None`. `if minus_first:` is wrong, because `[]` is a valid successful answer (the empty sublist sums to 0) and is falsy, so it would be misread as failure.

### 9. `partitions` and the two levels of list

**Q.** Why does `partitions` return `[()]` when `n == 0` instead of `()` or `[]`?

**A.** The return type is a Python list of partitions. When `n == 0` there is exactly one partition (the empty one), so the answer is a one-element Python list whose single element is the empty linked list `()`. Returning `[]` would claim there are no ways to sum to 0, which would make `with_m` empty and lose every partition. Returning `()` would be the wrong type entirely.

### 10. Trace `partitions` output order

**Q.** What are the first two lines of `print_partitions(6, 4)` and why do they come first?

**A.** `4 + 2` then `4 + 1 + 1`. `with_m` is computed and concatenated before `without_m`, so all partitions that include a 4 are listed before any that do not, and within those, `partitions(2, 4)` produces `(2)` before `(1 1)` for the same reason.

### 11. Type hints for containers and recursive types

**Q.** Why can't "a nested list of integers" be described by `list[int | list[int]]`, and what is the fix?

**A.** That hint allows a list of ints and a list of lists of ints, but not deeper nesting such as a list of lists of lists of ints. Adding another layer manually is always both too restrictive at some depth and too permissive at others. The fix is a recursive type alias that refers to its own name, the same technique used for `type LinkedList[T] = Link[T] | tuple[()]`.

---

## Summary

- A **tuple** is an immutable sequence; it supports indexing, slicing, `+`, `*`, can serve as a dict key, needs a trailing comma when it has one element, and underlies multiple assignment and multiple return values.
- Container **type hints** carry element types but not lengths; tuple hints are positional and fixed-length; `tuple[()]` matches only the empty tuple. Types that nest to arbitrary depth require **recursive type aliases**.
- Viewing a Python list as **`s[0]` plus `s[1:]`** turns sublist search problems into tree recursion with include/exclude branches; `longest` does this and uses `isinstance(..., list)` to distinguish "no answer" (`None`) from "the empty answer" (`[]`), plus `max(..., key=len)` to pick the longer result.
- A **linked list is either empty or a first value and the rest of the linked list**. `Link` is a dataclass representing a *non-empty* linked list, with `first` and `rest` (defaulting to `()`); `()` represents emptiness; `type LinkedList[T] = Link[T] | tuple[()]`.
- Every `Link` in the chain represents an entire sequence: `s.rest` is not one element, it is the whole remaining linked list.
- Printed form is Lisp-style `(3 4 5)`: parentheses, no commas; `repr` is `Link(first=3, rest=...)`; nested lists print as `(3 (4 5) 6)`.
- **Rule 1:** check `isinstance(s, Link)` (or `assert` it) before every `.first` or `.rest` access, and re-check after rebinding `s`.
- **Rule 2:** the second argument to `Link` must evaluate to a linked list: `()`, another `Link`, a name, or a call such as a recursive call.
- **Iteration** walks with `s = s.rest` and builds backward with `s = Link(k, s)`; **recursion** uses `not isinstance(s, Link)` as the base case and builds the rest before the front via `Link(x, recursive_call(...))`. Same results, opposite order of operations.
- Core operations re-implemented from scratch: `len_link`, `getitem_link`, `sum_link` (each iterative and recursive), `range_link` (loop, recursive, and accumulator/tail-recursive), `extend_link`, `map_link`, `filter_link`, `join_link`, and `format_link`.
- Linked lists have no comprehension syntax, so `map_link`/`filter_link`/`range_link` are composed as functions to replace `[square(x) for x in range(1, 6) if odd(x)]`.
- `partitions(n, m)` returns a **Python list of linked lists**: base cases `[()]` (one empty partition) and `[]` (none), recursive cases `[Link(m, p) for p in partitions(n-m, m)] + partitions(n, m-1)`, with each partition built largest part first so no partition is generated twice.
