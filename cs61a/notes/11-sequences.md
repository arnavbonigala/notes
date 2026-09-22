<!-- Mon, Sep 21, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 11: Sequences

## Overview

This lecture introduces Python's **sequence** abstraction and the tools for working with it. We start with **lists**: how to write list literals, select elements by index (including negative indices and nested indexing), test membership with `in`, and combine lists with `+` and `*`. We then replace the awkward `while`-loop-with-an-index pattern with the **`for` statement**, which binds a name to each element of a sequence in turn (and can unpack fixed-length sequences into multiple names). Next comes the **`range`** type, a sequence of consecutive integers that is not a list, with its half-open `[start, end)` convention that makes length and element selection trivial. Then **list comprehensions**, a compact expression form that maps and filters a sequence into a new list. Finally, we connect lists back to recursion: the slice `s[1:]` gives "the rest of the list," so any list can be viewed as a first element `s[0]` plus a shorter list, which is exactly the structure recursion needs. The lecture closes with several recursive list examples: `reverse`, `sum_list`, `sums` (all partitions of `n`), and `large` (a tree-recursive sublist problem).

---

## Key Concepts

### Lists are values that contain other values

A list is created with a **list literal**: square brackets around comma-separated expressions. Each element expression is evaluated, and the resulting values are collected into a single list value.

```python
>>> digits = [1, 8, 0, 8]
```

Crucially, the elements are *expressions*, not just literals. `[2 * 3, 5 - 4]` and `[6, 1]` produce equal lists. And the elements can be *any* values, including other lists, which is how we get nested structure.

In box-and-pointer terms (described in words, since the slide diagrams are missing): think of `digits` as a name in the current frame pointing to a box with four slots, numbered 0 through 3, each slot holding a value. For a nested list like `x = [3, 1, [4, 1, [5, 2], 6, 5], 3, 5]`, slot 2 of the outer box does not hold numbers; it holds a *pointer to another box*, and slot 2 of *that* box points to yet another box `[5, 2]`.

### Indexing is offset from the beginning

`digits[2]` is an **element selection expression**. The index is best understood not as "the third element" but as **the offset from the beginning**: offset 0 is the beginning itself, offset 1 is one past it, and so on. This immediately explains why the last element of a length-4 list is at index 3.

Two important points from the lecture:

1. **Both parts are arbitrary expressions.** The syntax is `<expression>[<expression>]`. The left expression is evaluated first and must produce a sequence; the right expression is evaluated second and must produce an integer index. So all of these are legal:

```python
>>> digits[get_index()]        # index comes from a function call
8
>>> get_list(3)[get_index()]   # the sequence comes from a function call
4
>>> odds[odds[3] - odds[2]]    # index computed from the list itself
```

2. **Negative indices count from the end.** For `digits = [8, 0, 8, 1]`, the valid indices are `0, 1, 2, 3` and equivalently `-4, -3, -2, -1`. So `digits[-1]` is the last element, `1`.

```
digits = [ 8,  0,  8,  1]
index     0   1   2   3
negative -4  -3  -2  -1
```

### Nested indexing chains left to right

For `x = [3, 1, [4, 1, [5, 2], 6, 5], 3, 5]`, to reach the `2`:

- `x[2]` evaluates to `[4, 1, [5, 2], 6, 5]`
- `x[2][2]` evaluates to `[5, 2]`
- `x[2][2][1]` evaluates to `2`

Each `[...]` applies to the value produced by everything to its left. Read it as: select element 2 of `x`, then element 2 of *that*, then element 1 of *that*.

### `in` tests for elements, not subsequences

The `in` operator evaluates both operands and asks whether the left value is **equal to some element** of the right container. It does not search inside nested structure and it does not look for subsequences.

```python
>>> digits = [1, 8, 0, 8]
>>> 1 in digits
True
>>> 2 in digits
False
>>> [1, 8] in digits            # NOT a subsequence search
False
>>> other_digits = [1, 2, [1, 8]]
>>> [1, 8] in other_digits      # here [1, 8] IS an element
True
```

Also, equality means value equality of the right type: `'1' in digits` is `False` for `digits = [1, 8, 0, 8]`, because the string `'1'` is not the integer `1`. There is a companion operator `not in`, where `5 not in digits` is equivalent to `not (5 in digits)`.

### Combining lists: `+` and `*`

`+` concatenates two lists into a new, longer list. `*` with an integer repeats the elements.

```python
>>> digits + other_digits
[1, 8, 0, 8, 1, 2, [1, 8]]
>>> [2, 7] + digits * 2
[2, 7, 1, 8, 0, 8, 1, 8, 0, 8]
```

Note `digits * 2` replicates the *elements* of `digits`; it does not multiply anything numerically. The `operator` module has function equivalents: `add`, `mul`, and `getitem` (the function form of `s[i]`).

### `len` and `type`

```python
>>> len(digits)
4
>>> type(digits)
<class 'list'>
```

`len` is a built-in function giving the number of elements. The index of the last element is always `len(s) - 1`.

### The `for` statement

Iterating over a sequence with `while` requires managing an index by hand. Here is the lecture's `count` function written both ways:

```python
def count(s, value):
    """Count the number of times value occurs in sequence s.

    >>> count([1, 2, 1, 3, 1], 1)
    3
    """
    total, index = 0, 0
    while index < len(s):
        element = s[index]
        if element == value:
            total += 1
        index += 1
    return total
```

The index bookkeeping is pure overhead: the only interesting line is the `if`. The `for` statement removes all of it:

```python
def count(s, value):
    """Count the number of times value occurs in sequence s.

    >>> count([1, 2, 1, 3, 1], 1)
    3
    """
    total = 0
    for element in s:
        if element == value:
            total += 1
    return total
```

Note also the augmented assignment `total += 1`, shorthand for `total = total + 1`.

**Execution procedure for `for <name> in <expression>: <suite>`:**

1. Evaluate the header `<expression>`, which must yield an **iterable** value (for now, think "a sequence"; the precise definition of iterable comes several weeks later).
2. For each element in that sequence, in order:
   - Bind `<name>` to that element **in the current frame**.
   - Execute the `<suite>`.

The critical environment fact: **a `for` statement creates no new frame.** The loop name is bound in the first (current) frame of the environment, rebound on each pass, and it persists after the loop finishes with the value of the last element.

### Sequence unpacking in a `for` header

If you are iterating over a sequence of fixed-length sequences (a list of pairs, say), you can name the components directly:

```python
pairs = [[1, 2], [2, 2], [3, 2], [4, 4]]

same_count = 0
for x, y in pairs:
    if x == y:
        same_count += 1
# same_count is 2
```

Here `x` and `y` are bound to the elements *within* each pair, rather than `x` being bound to the pair itself. This is the same mechanism as multiple assignment (`x, y = pair`), just written in the loop header. It only works when every element of the outer sequence has the same fixed length.

### Ranges

A **range** is another sequence type: it is a sequence but it is *not* a list. A range represents a sequence of consecutive integers.\*

Picture the infinite integer number line. `range(-2, 2)` marks off a finite chunk of it. The starting value is **included** and the ending value is **excluded**. The lecture emphasizes drawing the two markers as pointing *just before* the indicated numbers, so `-2` is in and `2` is out.

This half-open convention buys two clean formulas:

- **Length:** `ending value - starting value`. For `range(-2, 2)`, that is `2 - (-2) = 4`.
- **Element selection:** `starting value + index`. Element 0 is `-2`; element 3 is `-2 + 3 = 1`.

Because it has a length and supports element selection, a range is a sequence.

```python
>>> range(3)
range(0, 3)               # a range displays as itself, not as a list
>>> list(range(-2, 2))    # the list constructor
[-2, -1, 0, 1]
>>> list(range(4))        # one argument: start defaults to 0
[0, 1, 2, 3]
>>> list(range(5, 8))
[5, 6, 7]
```

`list` is the **list constructor**: a built-in function that, called on any sequence, returns a list of that sequence's elements. A range is not a list, but `list(some_range)` gives you one.

\*Ranges can actually represent more general integer sequences (with a step), but this lecture focuses on consecutive integers.

**Two uses of ranges:**

```python
def sum_below(n):
    """Sum the non-negative integers below n.

    >>> sum_below(5)
    10
    """
    total = 0
    for i in range(n):
        total += i
    return total
```

```python
def cheer():
    for _ in range(3):
        print('Go Bears!')
```

In the second case we do not care about the element values at all; we just want to repeat something three times. The convention is to name the loop variable `_` (a single underscore) to signal to other programmers that the name is deliberately unused. Any name would work; `_` communicates intent.

You can also use a range to iterate over indices when you need them:

```python
>>> digits = [1, 0, 1, 8]
>>> for i in range(len(digits)):
...     print(digits[i])
```

### List comprehensions

A list comprehension is a **form of combination**: an expression that builds a new list from an existing sequence.

```
[<map exp> for <name> in <iter exp> if <filter exp>]

short version:  [<map exp> for <name> in <iter exp>]
```

`<name>` is a name you choose. Evaluation: evaluate `<iter exp>` to get a sequence; for each element, bind `<name>` to it, evaluate `<filter exp>` (if present) and skip the element if it is false, then evaluate `<map exp>` and append its value to the result list.

```python
>>> odds = [1, 3, 5, 7, 9]
>>> [x + 1 for x in odds]
[2, 4, 6, 8, 10]
>>> [x for x in odds if 25 % x == 0]
[1, 5]
>>> [x + 1 for x in odds if 25 % x == 0]
[2, 6]
```

From the lecture code:

```python
>>> digits = [1, 0, 1, 8]
>>> [100 * d for d in digits]
[100, 0, 100, 800]
>>> [100 * d for d in digits if d < 5]
[100, 0, 100]
```

And from the slides, the `letters` example: `[letters[i] for i in [3, 4, 6, 8]]` evaluates to `['d', 'e', 'm', 'o']`, which shows that the map expression can be an element selection into some *other* sequence.

A comprehension inside a function:

```python
def divisors(n):
    """Return the divisors of n that are less than n.

    >>> divisors(1)
    [1]
    >>> divisors(4)
    [1, 2]
    >>> divisors(12)
    [1, 2, 3, 4, 6]
    """
    return [1] + [x for x in range(2, n) if n % x == 0]
```

The `[1]` is handled separately because 1 divides everything (and it makes `divisors(1)` come out as `[1]`).

### Lists, slices, and recursion

The bridge from lists to recursion is the **slice**:

> For any list `s`, the expression `s[1:]` is called a slice from index 1 to the end (or 1 onward).

Three facts to memorize:

- The value of `s[1:]` is a list whose **length is one less** than the length of `s`.
- It contains **all the elements of `s` except `s[0]`**.
- **Slicing `s` does not affect `s`.** It builds a new list; the original is untouched.

```python
>>> s = [2, 3, 6, 4]
>>> s[1:]
[3, 6, 4]
>>> s
[2, 3, 6, 4]
```

This gives the central recursive view: **in a list `s`, the first element is `s[0]` and the rest of the elements are `s[1:]`.** The rest is itself a list, so it can be processed the same way, but it is *shorter*, which is what guarantees progress toward a base case. The base case is almost always the empty list, tested with `len(s) == 0`, `s == []`, or most idiomatically `not s`.

(Full details of slicing syntax, including two-index and stepped slices, come in a later lecture.)

---

## Definitions

- **Sequence:** a value that has a length and supports element selection by index. Lists and ranges are both sequences; strings are too (extra context, mentioned only implicitly here).
- **List:** a built-in Python data type representing an ordered collection of values, written as a list literal `[e0, e1, ...]`.
- **List literal:** an expression consisting of comma-separated element expressions inside square brackets; each element expression is evaluated and the values are collected into a new list.
- **Element selection expression:** `<expression>[<expression>]`; the left operand is evaluated to a sequence, the right to an index, and the result is the element of that sequence at that index.
- **Index:** the offset of an element from the beginning of a sequence. The first element has index 0; the last has index `len(s) - 1`.
- **Negative index:** an index counted from the end; `s[-1]` is the last element, `s[-len(s)]` is the first.
- **`in` operator:** evaluates to `True` if its left value is equal to some element of the container on the right. Tests individual elements only, not subsequences and not nested contents. `not in` is its negation.
- **`len`:** built-in function returning the number of elements in a sequence.
- **`for` statement:** a compound statement of the form `for <name> in <expression>: <suite>` that evaluates `<expression>` to an iterable, then for each element binds `<name>` to it in the current frame and executes `<suite>`. It introduces no new frame.
- **Iterable value:** a value that a `for` statement can iterate over. Precise definition deferred; for now, read it as "a sequence."
- **Sequence unpacking:** binding multiple names to the components of a fixed-length sequence at once, as in `for x, y in pairs:` or `x, y = pair`.
- **Range:** a sequence type representing consecutive integers. `range(start, end)` includes `start` and excludes `end`; its length is `end - start` and its element at index `i` is `start + i`. `range(end)` uses an implicit start of 0.
- **List constructor:** the built-in function `list`, which when called on a sequence returns a list of that sequence's elements.
- **List comprehension:** an expression `[<map exp> for <name> in <iter exp> if <filter exp>]` that builds a new list by evaluating `<map exp>` once for each element of `<iter exp>` (bound to `<name>`) that satisfies `<filter exp>`.
- **Slice (from 1 onward):** the expression `s[1:]`, whose value is a new list containing all elements of `s` except `s[0]`. The original `s` is unchanged.
- **Sublist of `s`:** a list consisting of some of the elements of `s`, in order. It could contain none of them or all of them.
- **Augmented assignment:** `total += 1`, shorthand for `total = total + 1`.

---

## Worked Examples

### 1. Nested indexing

```python
x = [3, 1, [4, 1, [5, 2], 6, 5], 3, 5]
```

**Goal:** write an expression that evaluates to `2`.

Step by step:

- The outer list `x` has 5 elements at indices 0-4: `3`, `1`, `[4, 1, [5, 2], 6, 5]`, `3`, `5`.
- The `2` is buried inside index 2, so start with `x[2]` → `[4, 1, [5, 2], 6, 5]`.
- That list has 5 elements at indices 0-4: `4`, `1`, `[5, 2]`, `6`, `5`. The `2` is inside index 2 again, so `x[2][2]` → `[5, 2]`.
- That list has `5` at index 0 and `2` at index 1, so `x[2][2][1]` → `2`.

**Answer:** `x[2][2][1]`

In box-and-pointer terms: `x` points to a five-slot box; slot 2 points to a second five-slot box; slot 2 of *that* points to a two-slot box; slot 1 of that box holds `2`. Each bracket follows one pointer.

### 2. Element selection with arbitrary expressions

```python
>>> digits = [1, 8, 0, 8]
>>> def get_index():
...     return 1
>>> digits[get_index()]
8
```

Evaluation order: evaluate `digits` (a list), then evaluate `get_index()` (a call that opens a frame and returns `1`), then select element 1 of the list, which is `8`.

```python
>>> def get_list(n):
...     return [n, n + 1, n + 2]
>>> get_list(3)[get_index()]
4
```

Here the *sequence* is itself produced by a call. `get_list(3)` returns a brand new list `[3, 4, 5]`; `get_index()` returns `1`; element 1 of `[3, 4, 5]` is `4`.

### 3. `evens` with a list comprehension

```python
def evens(n: int) -> list[int]:
    """Return a list of the first n even numbers.

    >>> evens(0)
    []
    >>> evens(3)
    [0, 2, 4]
    """
    return [2 * x for x in range(n)]
```

**Why it works (following the slide's "write down an example" advice):** take `n = 3`.

- `list(range(3))` is `[0, 1, 2]`: three values, exactly the count we need.
- We need `[0, 2, 4]`. Comparing position by position: `0 → 0`, `1 → 2`, `2 → 4`. Each output is twice its input.
- So the map expression is `2 * x`, giving `[2 * 0, 2 * 1, 2 * 2] = [0, 2, 4]`.

For `n = 0`, `range(0)` is empty, so the comprehension produces `[]`, matching the doctest.

The general technique on display: pick a concrete input, write down what `range(n)` gives you, write down the answer you want, and find the expression that maps one to the other.

### 4. `promoted`: stable partition with two comprehensions

```python
def promoted(s, f):
    """Return a list with the same elements as s, but with all
    elements e for which f(e) is a true value placed first.

    >>> promoted(range(10), odd) # odds in front
    [1, 3, 5, 7, 9, 0, 2, 4, 6, 8]
    """
    return [e for e in s if f(e)] + [e for e in s if not f(e)]
```

**Reasoning:** start with `0, 1, 2, ..., 9` and the desired result `[1, 3, 5, 7, 9, 0, 2, 4, 6, 8]`. Notice the result is two blocks: the elements satisfying `f`, in their original order, followed by the elements failing `f`, in their original order.

- `[e for e in s if f(e)]` walks `s` once keeping only the true ones: `[1, 3, 5, 7, 9]`.
- `[e for e in s if not f(e)]` walks `s` again keeping only the false ones: `[0, 2, 4, 6, 8]`.
- `+` concatenates them.

Two properties fall out for free. **Order is preserved within each group** because a comprehension processes `s` in order. And **`s` can be any sequence**, not just a list: the doctest passes a `range`, and the `+` still works because both operands of `+` are lists produced by the comprehensions.

### 5. `reverse` by recursion

```python
def reverse(s):
    """Return s in reverse order.

    >>> reverse([4, 6, 2])
    [2, 6, 4]
    """
    if not s:
        return []
    return reverse(s[1:]) + [s[0]]
```

**Recursive idea:** the reverse of a list is the reverse of *the rest of the list*, followed by the first element.

**Trace of `reverse([4, 6, 2])`:**

| Call | `s[0]` | `s[1:]` | Recursive result | Returns |
|---|---|---|---|---|
| `reverse([4, 6, 2])` | `4` | `[6, 2]` | `[2, 6]` | `[2, 6] + [4]` = `[2, 6, 4]` |
| `reverse([6, 2])` | `6` | `[2]` | `[2]` | `[2] + [6]` = `[2, 6]` |
| `reverse([2])` | `2` | `[]` | `[]` | `[] + [2]` = `[2]` |
| `reverse([])` | - | - | - | `[]` (base case) |

Note `[s[0]]`, not `s[0]`: `+` on lists needs a *list* on both sides, so the single element must be wrapped in brackets. Also note that each call gets a strictly shorter list (because `s[1:]` has length one less), which guarantees we reach the base case.

Environment-wise: each recursive call opens its own frame with its own binding of `s`. The frame for `reverse([4, 6, 2])` stays open, waiting, while the frame for `reverse([6, 2])` runs; only when the inner call returns can the outer one finish computing `[2, 6] + [4]`.

### 6. `sum_list` by recursion

```python
def sum_list(s):
    """Return the sum of the numbers in list s.

    >>> sum_list([2, 4, 1, 3])
    10
    """
    if len(s) == 0:
        return 0
    return s[0] + sum_list(s[1:])
```

**Recursive idea:** the sum of a list is the first element plus the sum of the rest.

For `[2, 4, 1, 3]`: `s[0]` is `2` (a number), `s[1:]` is `[4, 1, 3]` (a list), and `sum_list([4, 1, 3])` is `8` (a number). So we add `2 + 8 = 10`. The type discipline matters: we are adding a number to a number, not a number to a list.

Base case: the sum of the elements of the empty list is `0`.

(This duplicates Python's built-in `sum`; it is written here purely as practice.)

### 7. `sums`: all lists of positive integers adding to `n`

```python
def sums(n: int) -> list[list[int]]:
    """Return a list of all of the possible lists of
    positive integers whose elements add up to n.

    >>> sums(3)
    [[1, 1, 1], [1, 2], [2, 1], [3]]
    """
    result = []
    for first in range(1, n):
        result = result + [[first] + rest for rest in sums(n - first)]
    return result + [[n]]
```

**How to reason to this answer (following the slide):** take `n = 3`.

- The loop runs `first` over `range(1, 3)`, that is `1` and `2`. These are the possible first elements *other than* `n` itself.
- Consider `first = 1`. Every answer starting with `1` must be `1` followed by some list of positive integers summing to `3 - 1 = 2`. Those are exactly `sums(2)`, which is `[[1, 1], [2]]`.
- So we need `[[1, 1, 1], [1, 2]]`, which is `[1] + [1, 1]` and `[1] + [2]`. The expression that produces this is `[[first] + rest for rest in sums(2)]`. Note `[first]` must be a one-element list so that `+` concatenates lists.
- Consider `first = 2`. `sums(3 - 2)` is `sums(1)` = `[[1]]`, so we get `[[2, 1]]`.
- After the loop, `result` is `[[1, 1, 1], [1, 2], [2, 1]]`.
- The final case is the list `[n]` itself, the single-element partition, handled by `return result + [[n]]`. This is also what terminates the recursion: `sums(1)` has `range(1, 1)` empty, so it returns `[[1]]` immediately.

**Why `range(1, n)` and not `range(1, n + 1)`:** the case `first == n` is the `[[n]]` added at the end. Including it in the loop would recurse on `sums(0)`, which is not handled.

**`sums(3)` step by step:**

```
sums(3):
  result = []
  first = 1: sums(2) = [[1, 1], [2]]
             [[1] + rest for rest in [[1, 1], [2]]] = [[1, 1, 1], [1, 2]]
             result = [[1, 1, 1], [1, 2]]
  first = 2: sums(1) = [[1]]
             [[2] + rest for rest in [[1]]] = [[2, 1]]
             result = [[1, 1, 1], [1, 2], [2, 1]]
  return result + [[3]] = [[1, 1, 1], [1, 2], [2, 1], [3]]
```

### 8. `large`: tree recursion over sublists

A **sublist** of `s` is a list containing some of the elements of `s` (possibly none, possibly all). `large(s, n)` returns the sublist of `s` with the largest sum that is less than or equal to `n`, where `s` is a list of positive numbers and `n` is non-negative.

```python
def large(s, n):
    """Return the sublist of positive numbers s with the largest
    sum that is less than or equal to n.

    >>> large([4, 2, 5, 6, 7], 3)
    [2]
    >>> large([4, 2, 5, 6, 7], 8)
    [2, 6]
    >>> large([4, 2, 5, 6, 7], 19)
    [4, 2, 6, 7]
    >>> large([4, 2, 5, 6, 7], 20)
    [2, 5, 6, 7]
    """
    if s == []:
        return []
    first, rest = s[0], s[1:]
    if first > n:
        return large(rest, n)
    else:
        with_first = [first] + large(rest, n - first)
        without_first = large(rest, n)
        if sum_list(with_first) > sum_list(without_first):
            return with_first
        else:
            return without_first
```

**The recursive idea:** the first element is either in the answer or it is not. Once that is decided, you still have to decide which sublist of *the rest* to keep, and that is the recursive call.

Walking the cases:

- **Base case:** the only sublist of `[]` is `[]`, and since `n` is non-negative, `[]` (sum 0) is a valid answer.
- **`first > n`:** the first element alone already exceeds the budget, so it cannot be in any valid sublist. Skip it: `large(rest, n)`. Notice `first` appears nowhere in the returned expression, so it is genuinely excluded. In `large([4, 2, 5, 6, 7], 3)`, `4 > 3`, so we skip `4` and solve `large([2, 5, 6, 7], 3)`, which gives `[2]`.
- **Including `first`:** `[first] + large(rest, n - first)`. The budget shrinks by `first` because the remaining elements may only sum to `n - first`; then `first` is prepended to the result.
- **Excluding `first`:** `large(rest, n)`. The full budget `n` remains, since nothing has been spent.
- **Choosing:** try both and keep whichever has the larger sum. This comparison is genuinely necessary. In `large([4, 2, 5, 6, 7], 8)`: using `4` leaves budget `4` for `[2, 5, 6, 7]`, and the best you can do is `[2]`, for a total of `6`. Not using `4` lets `[2, 5, 6, 7]` use the full budget `8`, giving `[2, 6]`, sum `8`. Since `8 > 6`, we return `[2, 6]`. Greedily grabbing the first element would have given the wrong answer.

This is **tree recursion**: each call (in the non-skipping case) makes two recursive calls, and the recursion branches.

---

## Common Pitfalls

1. **Off-by-one on the last index.** The last element of `s` is `s[len(s) - 1]`, not `s[len(s)]`. Thinking of the index as an *offset from the beginning* rather than an ordinal position prevents this.

2. **Assuming `in` searches deeply or finds subsequences.** `[1, 8] in [1, 8, 0, 8]` is `False`. `in` compares against whole elements only. It is `True` only when the list literally contains `[1, 8]` as one of its elements, as in `[1, 2, [1, 8]]`.

3. **Confusing values of different types.** `'1' in [1, 8, 0, 8]` is `False`; the string is not the integer.

4. **Forgetting that `range` excludes its ending value.** `range(4)` is `0, 1, 2, 3`. `range(1, n)` stops at `n - 1`. When you need `n` itself, write `range(1, n + 1)` (or handle it separately, as `sums` does).

5. **Printing a range and expecting a list.** `range(3)` displays as `range(0, 3)`, not `[0, 1, 2]`. Wrap it in `list(...)` to see the elements. A range *is* a sequence, but it is not a list.

6. **Adding a bare element to a list.** `reverse(s[1:]) + s[0]` is an error; `+` needs lists on both sides. Write `+ [s[0]]`. Same for `[first] + rest` in `sums`, where `[first]` must be bracketed.

7. **Expecting `for` to create a new frame.** It does not. The loop name is bound in the current frame and survives after the loop ends, holding the last element's value.

8. **Trying to unpack when the elements are not fixed-length sequences.** `for x, y in pairs:` only works when every element of `pairs` has exactly two components.

9. **Thinking slicing mutates.** `s[1:]` creates a new list and leaves `s` completely alone. Evaluating `s[1:]` on its own line accomplishes nothing.

10. **Mixing up the filter and map positions in a comprehension.** The order is `[<map exp> for <name> in <iter exp> if <filter exp>]`: the thing you compute comes *first*, the condition comes *last*.

11. **Missing recursive progress.** A recursive list function must recurse on `s[1:]` (strictly shorter), not on `s`. Similarly, `sums(n - first)` requires `first >= 1` for the argument to decrease.

12. **In `large`, forgetting to reduce the budget.** When you include `first`, the recursive call must use `n - first`, not `n`. When you exclude it, the budget stays `n`.

---

## Likely Exam Points

### 1. Evaluating nested index expressions

**Practice:** Given `x = [3, 1, [4, 1, [5, 2], 6, 5], 3, 5]`, what do `x[2][3]`, `x[-1]`, and `x[2][-3][0]` evaluate to?

<details>
<summary>Answer</summary>

- `x[2]` is `[4, 1, [5, 2], 6, 5]`, so `x[2][3]` is `6`.
- `x[-1]` is the last element of `x`, which is `5`.
- `x[2][-3]` is the third-from-last element of `[4, 1, [5, 2], 6, 5]`, which is `[5, 2]`; then `[0]` gives `5`.
</details>

### 2. `in` semantics

**Practice:** What does each of the following print?

```python
s = [1, [2, 3], 4]
print([2, 3] in s)
print(2 in s)
print(4 not in s)
```

<details>
<summary>Answer</summary>

`True`, `False`, `False`. `[2, 3]` is literally an element of `s`. The value `2` is not an element of `s` (it is nested inside one, and `in` does not look inside). `4` is an element, so `not in` is `False`.
</details>

### 3. Range arithmetic

**Practice:** What is `len(range(-3, 5))`? What is `list(range(-3, 5))[2]`? What does `list(range(3, 3))` evaluate to?

<details>
<summary>Answer</summary>

Length is `ending - starting` = `5 - (-3)` = `8`. Element at index 2 is `starting + index` = `-3 + 2` = `-1`. `range(3, 3)` has length `0`, so `list(range(3, 3))` is `[]`.
</details>

### 4. Writing a list comprehension to a spec

**Practice:** Fill in the blank so the doctest passes.

```python
def squares_of_odds(n):
    """Return the squares of the odd numbers below n.

    >>> squares_of_odds(8)
    [1, 9, 25, 49]
    """
    return ________________________
```

<details>
<summary>Answer</summary>

`[x * x for x in range(n) if x % 2 == 1]`

Check with `n = 8`: `range(8)` is `0..7`; the filter keeps `1, 3, 5, 7`; the map squares them into `[1, 9, 25, 49]`.
</details>

### 5. `promoted`-style two-comprehension partitioning

**Practice:** Implement `demoted(s, f)`, which returns a list with the same elements as `s` but with all elements `e` for which `f(e)` is a true value placed *last*, preserving relative order within each group.

<details>
<summary>Answer</summary>

```python
def demoted(s, f):
    return [e for e in s if not f(e)] + [e for e in s if f(e)]
```

Just swap the order of the two comprehensions from `promoted`.
</details>

### 6. Filling in a recursive list function

**Practice:** Fill in the blank.

```python
def double_all(s):
    """Return a list with each element of s appearing twice in a row.

    >>> double_all([1, 2, 3])
    [1, 1, 2, 2, 3, 3]
    """
    if not s:
        return []
    return _________________________
```

<details>
<summary>Answer</summary>

`[s[0], s[0]] + double_all(s[1:])`

The first element contributes two copies; the rest of the list is handled by the recursive call on `s[1:]`, which is strictly shorter and reaches `[]`.
</details>

### 7. Tracing a recursive list function

**Practice:** Trace `reverse([1, 2, 3])`, listing each call and what it returns.

<details>
<summary>Answer</summary>

| Call | Returns |
|---|---|
| `reverse([])` | `[]` |
| `reverse([3])` | `[] + [3]` = `[3]` |
| `reverse([2, 3])` | `[3] + [2]` = `[3, 2]` |
| `reverse([1, 2, 3])` | `[3, 2] + [1]` = `[3, 2, 1]` |

The calls go down first (each shortening `s`), then the additions happen on the way back up.
</details>

### 8. `for`-statement environment behavior

**Practice:** What does this print, and why?

```python
for x in [10, 20, 30]:
    pass
print(x)
```

<details>
<summary>Answer</summary>

`30`. The `for` statement creates no new frame; `x` is bound in the current frame on each pass and retains the last element's value after the loop.
</details>

### 9. Sequence unpacking

**Practice:** What is `total` after this runs?

```python
total = 0
for a, b in [[1, 2], [3, 4], [5, 6]]:
    total += b - a
```

<details>
<summary>Answer</summary>

`3`. Each pair contributes `1` (`2-1`, `4-3`, `6-5`), and there are three pairs.
</details>

### 10. Understanding `sums`

**Practice:** What is `sums(2)`? What is `sums(4)`?

<details>
<summary>Answer</summary>

`sums(2)`: the loop runs `first` over `range(1, 2)` = `[1]`. For `first = 1`, `sums(1)` = `[[1]]`, giving `[[1, 1]]`. Then `+ [[2]]` yields `[[1, 1], [2]]`.

`sums(4)`: `first` runs over `1, 2, 3`.
- `first = 1`: `sums(3)` = `[[1,1,1],[1,2],[2,1],[3]]` → `[[1,1,1,1],[1,1,2],[1,2,1],[1,3]]`
- `first = 2`: `sums(2)` = `[[1,1],[2]]` → `[[2,1,1],[2,2]]`
- `first = 3`: `sums(1)` = `[[1]]` → `[[3,1]]`
- Plus `[[4]]`.

Result: `[[1,1,1,1],[1,1,2],[1,2,1],[1,3],[2,1,1],[2,2],[3,1],[4]]`
</details>

### 11. The budget update in `large`

**Practice:** In `large`, why is the recursive call in the "include `first`" branch written `large(rest, n - first)` rather than `large(rest, n)`?

<details>
<summary>Answer</summary>

Because `first` has already been spent. The whole returned sublist must sum to at most `n`, and it includes `first`, so the part drawn from `rest` may sum to at most `n - first`. Using `n` would let the total exceed the limit.
</details>

---

## Summary

- A **list** is created with a list literal; its elements are values of any type, including other lists.
- `s[i]` is an **element selection expression**; both `s` and `i` can be arbitrary expressions. The index is an **offset from the beginning**, starting at 0; negative indices count from the end (`s[-1]` is the last element).
- Nested indexing chains left to right: `x[2][2][1]` follows three levels of structure.
- `in` tests whether a value equals **some element** of a container; it does not find subsequences or search nested structure. `not in` negates it.
- `+` concatenates lists, `*` repeats them, `len` gives the element count.
- The **`for` statement** binds a name to each element of an iterable in turn, **in the current frame, with no new frame created**. It replaces the manual index-and-`while` pattern.
- **Sequence unpacking** in a `for` header (`for x, y in pairs:`) names the components of each fixed-length element.
- A **range** is a sequence (not a list) of consecutive integers: `range(start, end)` includes `start`, excludes `end`, has length `end - start`, and has element `start + i` at index `i`. `range(end)` starts at 0. Use `list(...)` to see the elements. Use `_` as the loop name when the values are irrelevant.
- **List comprehensions**: `[<map exp> for <name> in <iter exp> if <filter exp>]` builds a new list by filtering then mapping, preserving order.
- Two comprehensions plus `+` cleanly solve stable-partition problems, as in `promoted`.
- The slice **`s[1:]`** is the rest of the list: one shorter, missing `s[0]`, and it does **not** modify `s`. A list is "a first element `s[0]` and the rest `s[1:]`," which is the standard decomposition for recursion.
- Recursive list patterns: base case on the empty list; combine `s[0]` with a recursive call on `s[1:]` (`sum_list`, `reverse`); build up lists of lists with a comprehension over a recursive result (`sums`); branch on include/exclude for tree recursion over sublists (`large`).
- When `+`-ing a single element onto a list, wrap it in brackets: `[s[0]]`, `[first] + rest`.
- Problem-solving habit stressed throughout: **write down a concrete example**, note what the recursive call or `range` gives you, note the answer you want, and find the expression connecting them.
