<!-- Fri, Sep 25, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 13: Objects

## Overview

This lecture bridges two topics: it wraps up the data-abstraction and recursion threads from earlier lectures, then opens the door to object-oriented programming. It begins with dictionaries as a concrete example of an object (a value that bundles data with behavior, such as `d.get(4)`), introduces **dictionary comprehensions** as a compact way to build dictionaries, and then shows how the `@dataclass` decorator lets you define your own types with named attributes and a custom `__str__`. In between, the lecture revisits tree recursion through `count_partitions` (`cp`), showing three distinct lessons: adding base cases can skip work, varying the recursive calls can avoid base cases, and (most importantly) **to enforce a new constraint, add an argument that tracks whether the constraint has been violated**. The lecture closes with a midterm-style tree recursion problem, `count_park`, which counts the ways to fill `n` parking spots with motorcycles (1 spot), cars (2 spots), and empty spaces. The unifying theme: values in Python are objects that behave like the things they represent, and their behavior is reached through dot expressions.

---

## Key Concepts

### 1. Dictionaries as objects

A dictionary stores a correspondence between keys and values: `d = {3: 9, 4: 16}`. That correspondence is the **data** the dict represents. But a dict also *does* things: `d.get(4)` evaluates to `16`. That is **behavior**. The lecture's point is that data and behavior are bundled together, and this bundle is what we call an object.

Key facts demonstrated in `dict_demos()`:

| Expression | Result / behavior |
|---|---|
| `numerals['X']` | `10` (lookup by key) |
| `numerals['X-ray']` | **Error**: key not present |
| `numerals[10]` | **Error**: keys are looked up, not values |
| `len(numerals)` | `3` (number of key-value pairs) |
| `list(numerals)` | `['I', 'V', 'X']` (iterating a dict gives **keys**) |
| `numerals.values()` | a view of the values; `list(...)` gives `[1, 5, 10]` |
| `sum(numerals.values())` | `16` |
| `dict([[3, 9], [4, 16]])` | `{3: 9, 4: 16}` (build from a sequence of pairs) |
| `numerals.get('X', 0)` | `10` (key present, so default ignored) |
| `numerals.get('X-ray', 0)` | `0` (key absent, return the default) |
| `numerals.get('X-ray')` | `None` (no default supplied) |
| `{1: 2, 1: 3}` | `{1: 3}` (duplicate keys: later wins, no error) |
| `{[1]: 2}` | **Error**: a list cannot be a key (keys must be immutable) |
| `{1: [2]}` | `{1: [2]}` (fine: **values** may be lists) |

The asymmetry in the last two lines is worth internalizing: restrictions apply to keys, not values.

### 2. Dictionary comprehensions

Same shape as a list comprehension, but with a colon-separated key and value expression, and curly braces:

```
{<key exp>: <value exp> for <name> in <iter exp> if <filter exp>}
```

The short form drops the filter:

```
{<key exp>: <value exp> for <name> in <iter exp>}
```

Evaluation order intuition: for each element of the iterable, bind `<name>`, check `<filter exp>` (if present), and if it passes, evaluate `<key exp>` and `<value exp>` to form one entry. Because it's a comprehension, `<name>` lives in its own frame, not in the enclosing one.

The slide showed `{'Dem': 0}` as a minimal dict literal example.

### 3. Objects, classes, attributes, methods

From the transcript's `datetime` demo:

- A **class** is a type of object. `date` is a class.
- You create an **instance** by calling the class: `date(2015, 2, 20)`.
- An object "behaves like the value it represents": subtracting two dates gives a sensible answer describing a span of days, not nonsense.
- **Attributes** are the main mechanism for behavior, accessed with a **dot expression**: `today.year`, `today.month`.
- A dot expression has the form `<expression> . <name>`. The left side is any expression that evaluates to an object (often just a name); the right side is the attribute name to look up.
- An attribute bound to a function is a **method**. `today.strftime('%A, %B %d')` returns `'Friday, February 20'`. Notice that nobody told the date it was a Friday, or how to spell "February": generating those is simply part of what it means to be a date.
- Classes are **first-class values** in Python: they can be passed as arguments, bound to names, returned, etc.
- **Every value in Python is an object**, including numbers and strings. Each has data plus behavior determined by its type.

### 4. Functions vs. objects (design intuition)

A well-designed *function* does one thing: arguments in, return value out. An *object* bundles many related behaviors. A date is used in many different ways, so it has many methods, each of which is like a function. Object-oriented programming uses this as an organizing metaphor: a large program is not one big thing but a collection of objects sending messages to each other. The dot-expression syntax exists to make that metaphor readable in code.

### 5. Data classes: creating your own types

A `class` statement creates a new type. The `@dataclass` decorator (from the `dataclasses` module) simplifies the class statement: in a data class, **each name with a type hint becomes an attribute of instances**.

```python
from dataclasses import dataclass

@dataclass
class Line:
    slope: float
    intercept: float

    def __str__(self):
        return format_line(self)
```

What you get for free from `@dataclass`:

- A constructor: `Line(3, 4)` binds `slope` to `3` and `intercept` to `4`, in the order the attributes were declared.
- A useful `repr`: typing `c` at the REPL shows `Line(slope=3, intercept=4)`.
- `type(c) == Line` is `True`, and `isinstance(c, Line)` is `True`.

What you write yourself:

- `__str__` determines what happens when you call `print` or `str` on a `Line`. Here `print(c)` produces `y = 3x + 4`.

Note the style used in the lecture: `parallel` and `format_line` are plain *functions* that take `Line` arguments and use type hints in their signatures (`def parallel(c: Line, d: Line) -> bool:`), not methods inside the class. Type hints here are documentation for readers; Python does not enforce them.

The two equivalent formattings shown:

```python
return 'y = ' + str(c.slope) + 'x + ' + str(c.intercept)
# OR
return f'y = {c.slope}x + {c.intercept}'
```

### 6. Recursion so far: two patterns

The lecture organized everything you've seen into two families.

**Pattern A: deal with one item, recurse for the rest.**
- `double_eights(s)`: check if the first two elements are both 8s; recurse on everything but the first element.
- `streak(n)`: check whether the last digit matches the previous; recurse on everything but the last digit.
- `reverse(s)`: get the first element into place; recurse on the rest.

**Pattern B: tree recursion, make a SMALL choice, and for each option, recurse.**
- `count_partitions(n, m)`: use a piece of size `m` and recurse for the rest, **and** don't use any pieces of size `m` and recurse for the rest. The sum of those two counts is the answer.

The words "small choice" matter: a choice small enough that the remaining subproblem is the same *kind* of problem with smaller inputs.

### 7. `cp(n, m)` and its variants

```python
def cp(n, m):
    if n == 0:
        return 1
    elif n < 0 or m == 0:
        return 0
    return cp(n-m, m) + cp(n, m-1)
```

`cp(n, m)` counts the ways to make `n` by summing positive pieces up to `m` in increasing order. Base cases:
- `n == 0`: exactly one way to make 0 (use nothing). Return `1`.
- `n < 0`: we overshot, that path is invalid. Return `0`.
- `m == 0`: no pieces available and `n > 0`. Return `0`.

Recursive case: `cp(n-m, m)` counts partitions that use **at least one** piece of size `m` (having used one, we may use more), and `cp(n, m-1)` counts partitions that use **no** `m` at all.

Three variations, each teaching a separate lesson:

**(a) Adding base cases can skip work.** Adding `elif m == 1: return 1` short-circuits a whole chain of calls, since there is exactly one way to make any positive `n` out of 1s.

**(b) Avoiding base cases by varying the recursive calls.** If `m > n`, using a piece of size `m` is guaranteed to overshoot, so skip that branch entirely:

```python
def cp_fast(n, m):
    if n == 0:
        return 1
    elif m == 0:
        return 0
    if m > n:
        return cp_fast(n, m-1)
    return cp_fast(n-m, m) + cp_fast(n, m-1)
```

Because we never call with a negative first argument, the `n < 0` base case is no longer needed at all.

**(c) To add a constraint, add arguments that track whether the constraint was violated.** This is the headline lesson of the recursion half of the lecture. You *cannot* write a base case for "we used too many pieces" in the original `cp`, because nothing in `(n, m)` records how many pieces have been used. So thread a counter through.

### 8. Tracking state through extra arguments

**Variant 1: at most 3 pieces total.** `2 + 2 + 2` is fine; `1 + 1 + 2 + 2` is not.

```python
def cp_3total(n, m):
    return cp_3total_helper(n, m, 3)

def cp_3total_helper(n, m, k):
    """Count partitions of n using up to k pieces of size m or less."""
    if n == 0:
        return 1
    elif n < 0 or m == 0:
        return 0
    elif k == 0:
        return 0
    return cp_3total_helper(n-m, m, k-1) + cp_3total_helper(n, m-1, k)
```

`k` is the remaining budget of pieces. Using a piece of size `m` spends one (`k-1`); declining to use `m` at all spends none (`k` unchanged). Running out of budget with `n > 0` left is a failure, so return `0`.

**Variant 2 (discussion question): at most 3 of each piece.** `1 + 1 + 2 + 2` is fine; `1 + 1 + 1 + 1 + 2` is not.

Now one counter is not enough: we need a per-size budget. The solution uses two arguments, `k` (how many of the *current* size `m` we may still use) and `t` (the per-size limit to reset to when we move to a smaller size).

```python
def cp_3_of_each_piece(n, m):
    return cp_3_of_each_piece_helper(n, m, 3, 3)

def cp_3_of_each_piece_helper(n, m, k, t):
    """Count partitions of n using pieces of size m or less.
    At most k of m and at most t of each of the rest."""
    if n == 0:
        return 1
    elif n < 0 or m == 0:
        return 0
    if k == 0:
        return cp_3_of_each_piece_helper(n, m-1, t, t)
    return cp_3_of_each_piece_helper(n-m, m, k-1, t) + \
           cp_3_of_each_piece_helper(n, m-1, t, t)
```

The slide's blanks were filled as `no_m = cpk(n, m-1, t, t)`: when we drop down to size `m-1`, the budget for that new size resets to the full per-size limit `t`. The `if k == 0: return no_m` line handles exhausting the budget for size `m`: we can no longer use `m`, but we can still use smaller pieces, so we do **not** return 0 here. That's the key difference from the "at most 3 total" version.

Sanity values from the docstrings: `cp(6, 4) == 9`, `cp_3total(6, 4) == 5`, `cp_3_of_each_piece(6, 4) == 7`; and `cp(4, 2) == 3`, `cp_3total(4, 2) == 2`, `cp_3_of_each_piece(4, 2) == 2`.

### 9. Tree recursion by choosing the first item: `count_park`

(Spring 2023 Midterm 2, Q5(a), modified.) A motorcycle `%` takes 1 spot, a car `<>` takes 2 adjacent spots, `.` is an empty spot. Count arrangements in `n` adjacent spots. Example string: `'.%%.<><>'`.

The framing question on the slide: *"We haven't parked anything yet. What's a first choice we can make?"* The small choice is **how to fill the first space**, and there are exactly three options:

1. Put a motorcycle there: 1 spot consumed, recurse on `n-1`.
2. Leave it empty: 1 spot consumed, recurse on `n-1`.
3. Put a car there: 2 spots consumed, recurse on `n-2`.

```python
def count_park(n):
    """Count the ways to park cars and motorcycles in n adjacent spots.
    >>> count_park(1) # '.' or '%'
    2
    >>> count_park(2) # '..', '.%', '%.', '%%', or '<>'
    5
    >>> count_park(4) # some examples: '<><>', '.%%.', '%<>%', '%.<>'
    29
    """
    if n < 0:
        return 0
    elif n == 0:
        return 1
    else:
        return count_park(n-1) + count_park(n-1) + count_park(n-2)
```

Reasoning about the base cases, as the slide suggests: *which recursive calls lead to these cases, and what should their values be?*
- `n == 0`: we filled every spot exactly. That is one complete, valid arrangement. Return `1`.
- `n < 0`: this happens only when we tried to park a car in the last remaining single spot (`n == 1` then `n-2 == -1`). That arrangement doesn't fit, so it isn't a valid way. Return `0`.

The two identical-looking `count_park(n-1)` terms are *not* redundant: one counts arrangements beginning with `%`, the other arrangements beginning with `.`. They happen to have the same count because both choices consume one spot. Writing `2 * count_park(n-1) + count_park(n-2)` would be equivalent.

Check: `count_park(3) == 13`, and the slide enumerated exactly those 13 strings: `%%%`, `%%.`, `%.%`, `%..`, `%<>`, `.%%`, `.%.`, `..%`, `...`, `.<>`, `<>%`, `<>.`, and (from `.` then `.%`/etc., the omitted-for-brevity branch) `<><>` appears at `n = 4`. The slide's tree also traced leaves like `%%<>` and `%.<>` down to the `-1 spaces left` row, which is precisely where the `n < 0` base case returns `0`.

---

## Definitions

- **Object**: a value that consists of data and behavior bundled together to create an abstraction, and that behaves like the thing it represents. Objects can represent things, properties of things, interactions, or processes.
- **Class**: a type of object. Classes are first-class values in Python (they can be passed to functions, bound to names, etc.).
- **Instance**: a particular object of some class, created by calling the class, e.g. `date(2015, 2, 20)` or `Line(3, 4)`.
- **Attribute**: a name-value pair associated with an object, reached via a dot expression.
- **Dot expression**: an expression of the form `<expression> . <name>`, where the left operand evaluates to an object and `<name>` is the attribute to look up.
- **Method**: an attribute bound to a function, i.e. a function invoked on an object using a dot expression, such as `d.get(4)` or `today.strftime('%A')`.
- **Object-oriented programming**: an approach to organizing large programs around the metaphor that a program is a collection of individual objects communicating by sending messages to each other.
- **Data class**: a class defined with the `@dataclass` decorator, in which each name with a type hint becomes an attribute of instances; the decorator supplies a constructor and a readable `repr` automatically.
- **`__str__`**: a method that determines what happens when `print` or `str` is called on an instance.
- **Type hint**: an annotation like `slope: float` or `def parallel(c: Line, d: Line) -> bool`. In a data class the attribute annotations declare the attributes; in function signatures they are documentation (extra context: Python does not check them at runtime).
- **Dictionary comprehension**: an expression `{<key exp>: <value exp> for <name> in <iter exp> if <filter exp>}` that builds a dict by evaluating the key and value expressions once per element that passes the filter.
- **`cp(n, m)` / count partitions**: the number of ways to make `n` by summing positive pieces of size up to `m`, where pieces appear in increasing order (so order of pieces doesn't create new arrangements).
- **Tree recursion**: a recursive pattern in which a function makes a small choice and recurses once for each option, typically producing more than one recursive call per frame.

---

## Worked Examples

### Example 1: `multiples` with a dictionary comprehension

```python
def multiples(s, factors):
    """Create a dictionary where each factor is a key and each value
    is the elements of s that are multiples of the key.

    >>> multiples([3, 4, 5, 6, 7, 8], [2, 3])
    {2: [4, 6, 8], 3: [3, 6]}
    >>> multiples([1, 2, 3, 4, 5], [2, 5, 8])
    {2: [2, 4], 5: [5], 8: []}
    """
    return {d: [x for x in s if x % d == 0] for d in factors}
```

Reading the blanks from the outside in:
1. The keys come from `factors`, so the outer comprehension is `for d in factors`. This is forced by the spec: "each element of `factors` is a key."
2. The value for key `d` must be a list, so the value expression is a **list comprehension nested inside a dict comprehension**.
3. That inner list iterates over `s` (`for x in s`) and filters to multiples of `d` (`if x % d == 0`).

Step through `multiples([3, 4, 5, 6, 7, 8], [2, 3])`:
- `d = 2`: scan `s`. `3 % 2 == 1` (drop), `4 % 2 == 0` (keep), `5` drop, `6` keep, `7` drop, `8` keep. Entry `2: [4, 6, 8]`.
- `d = 3`: `3` keep, `4` drop, `5` drop, `6` keep, `7` drop, `8` drop. Entry `3: [3, 6]`.
- Result `{2: [4, 6, 8], 3: [3, 6]}`.

Step through `multiples([1, 2, 3, 4, 5], [2, 5, 8])`:
- `d = 2`: `[2, 4]`.
- `d = 5`: `[5]`.
- `d = 8`: nothing in `s` is a multiple of 8, so the list comprehension yields `[]`. The key still appears, mapped to the empty list. This is why the comprehension form is right: every factor gets a key unconditionally, and only the *elements* are filtered.

Environment reasoning (in words): calling `multiples` opens a frame with `s` and `factors` bound to the argument lists (the frame's `s` points to the *same* list object the caller passed, no copy). Each comprehension evaluates in its own frame, so `d` and `x` never appear in the `multiples` frame; the inner list comprehension's frame can still see `d` and `s` because of lexical scoping. Each iteration of the inner comprehension builds a **fresh** list object; the returned dict holds pointers to two distinct list objects, and neither one is `s`.

### Example 2: tracing `cp(4, 2)` by hand

```python
def cp(n, m):
    if n == 0:
        return 1
    elif n < 0 or m == 0:
        return 0
    return cp(n-m, m) + cp(n, m-1)
```

`cp(4, 2)` should be `3`, corresponding to `2+2`, `1+1+2`, and `1+1+1+1`.

The full tree from the slide:

```
cp(4,2) = 3
├── cp(2,2) = 2                 [used one 2; 2 left to make]
│   ├── cp(0,2) = 1             [2+2]
│   └── cp(2,1) = 1
│       ├── cp(1,1) = 1
│       │   ├── cp(0,1) = 1     [1+1+2]
│       │   └── cp(1,0) = 0
│       └── cp(2,0) = 0
└── cp(4,1) = 1                 [no 2s at all]
    ├── cp(3,1) = 1
    │   ├── cp(2,1) = 1
    │   │   ├── cp(1,1) = 1
    │   │   │   ├── cp(0,1) = 1 [1+1+1+1]
    │   │   │   └── cp(1,0) = 0
    │   │   └── cp(2,0) = 0
    │   └── cp(3,0) = 0
    └── cp(4,0) = 0
```

Reading the answer off the tree: every leaf returning `1` is one valid partition, and the returned value at any node is the number of `1`-leaves beneath it. The narration from the slide:
- "choose at least one `m = 2`": `cp(4-2, 2) = cp(2, 2) = 2` ways, which are `2+2` and `1+1+2`.
- "choose no more `m = 2`": `cp(4, 1) = 1` way, which is `1+1+1+1`.
- Total `3`.

Notice `cp(2, 1)` is computed twice (once in each main branch). Tree recursion routinely recomputes subproblems; that is the source of its cost.

### Example 3: why `cp_fast` needs no `n < 0` case

```python
def cp_fast(n, m):
    if n == 0:
        return 1
    elif m == 0:
        return 0
    if m > n:
        return cp_fast(n, m-1)
    return cp_fast(n-m, m) + cp_fast(n, m-1)
```

Trace `cp_fast(4, 2)`: `m = 2` is not `> 4`, so both branches run. `cp_fast(2, 2)`: `2 > 2` is false, so both branches run, giving `cp_fast(0, 2) = 1` plus `cp_fast(2, 1) = 1`. On the other side `cp_fast(4, 1) = ... = 1`. Total `3`, same as before.

Now consider the dangerous call that made `n < 0` necessary in the original: `cp(1, 2)` would compute `cp(-1, 2) + cp(1, 1)`. In `cp_fast`, `m = 2 > n = 1`, so we return `cp_fast(1, 1)` directly and the negative call is never made. Once no call can produce a negative `n`, the `n < 0` guard is dead code and can be deleted. This is the "avoid base cases by varying recursive calls" lesson.

### Example 4: `cp_3total(4, 2)` vs. `cp(4, 2)`

`cp(4, 2) == 3` (`2+2`, `1+1+2`, `1+1+1+1`), but `cp_3total(4, 2) == 2`, because `1+1+1+1` uses four pieces. Let's see the budget do its work:

`cp_3total_helper(4, 2, 3)`
- Not a base case. Returns `helper(2, 2, 2) + helper(4, 1, 3)`.
- `helper(2, 2, 2)` (used one 2, two pieces left): `helper(0, 2, 1) + helper(2, 1, 2)`.
  - `helper(0, 2, 1)` hits `n == 0`, returns `1`. This is `2+2`.
  - `helper(2, 1, 2)`: `helper(1, 1, 1) + helper(2, 0, 2)`. The second is `0` (`m == 0`). The first: `helper(0, 1, 0) + helper(1, 0, 1)`. `helper(0, 1, 0)` checks `n == 0` **first**, so it returns `1`, giving `1+1+2`. `helper(1, 0, 1)` returns `0`.
  - So `helper(2, 2, 2) = 1 + 1 = 2`.
- `helper(4, 1, 3)`: trying to make 4 out of 1s with only 3 pieces. Each step spends one from `k`: `helper(3, 1, 2)`, `helper(2, 1, 1)`, `helper(1, 1, 0)`. At `helper(1, 1, 0)`, `n == 1` is not `0`, `n` is not negative, `m` is not `0`, but `k == 0`, so return `0`. The `m-1` siblings all hit `m == 0` and return `0`. So `helper(4, 1, 3) = 0`.
- Total: `2 + 0 = 2`. Correct.

Note the base-case ordering: `n == 0` is checked *before* `k == 0`, so exhausting the budget exactly as you finish still counts as a success.

### Example 5: `cp_3_of_each_piece(4, 2)`

Expected `2`. Partitions of 4 with pieces up to 2, using each size at most 3 times: `2+2` (two 2s, fine), `1+1+2` (two 1s and one 2, fine), `1+1+1+1` (four 1s, **violates** the limit on size 1). So the answer is `2`.

`helper(4, 2, 3, 3)`:
- `k = 3 != 0`, so return `helper(2, 2, 2, 3) + helper(4, 1, 3, 3)`.
- `helper(2, 2, 2, 3)` = `helper(0, 2, 1, 3)` + `helper(2, 1, 3, 3)`.
  - `helper(0, 2, 1, 3) = 1` → `2+2`.
  - `helper(2, 1, 3, 3)` = `helper(1, 1, 2, 3)` + `helper(2, 0, 3, 3)`. The latter is `0`. The former = `helper(0, 1, 1, 3)` + `helper(1, 0, 3, 3)` = `1 + 0 = 1` → `1+1+2`.
  - Subtotal `2`.
- `helper(4, 1, 3, 3)`: making 4 out of 1s, at most 3 of them. `helper(3, 1, 2, 3)` → `helper(2, 1, 1, 3)` → `helper(1, 1, 0, 3)`. At that last call `k == 0`, so instead of returning `0` we return `helper(1, 0, 3, 3)`, which hits `m == 0` and returns `0`. So this whole branch is `0`.
- Total `0 + 2 = 2`. Correct.

The crucial line is `if k == 0: return cpk(n, m-1, t, t)`: exhausting size `m` doesn't kill the path, it just forces us down to smaller pieces with a fresh budget `t`.

### Example 6: `count_park(2)` fully traced

```python
count_park(2)
= count_park(1) + count_park(1) + count_park(0)
```
- First `count_park(1)` (first spot is `%`): `= count_park(0) + count_park(0) + count_park(-1) = 1 + 1 + 0 = 2`. These are `%%` and `%.`; the `-1` branch is the illegal `%<>` which needs 3 spots.
- Second `count_park(1)` (first spot is `.`): also `2`, giving `..` and `.%`.
- `count_park(0)` (first two spots are a car `<>`): `1`, giving `<>`.
- Total `2 + 2 + 1 = 5`, matching the docstring's `'..'`, `'.%'`, `'%.'`, `'%%'`, `'<>'`.

### Example 7: the `Line` data class

```python
from dataclasses import dataclass

@dataclass
class Line:
    slope: float
    intercept: float

    def __str__(self):
        return format_line(self)

def parallel(c: Line, d: Line) -> bool:
    return c.slope == d.slope

def format_line(c: Line) -> str:
    return 'y = ' + str(c.slope) + 'x + ' + str(c.intercept)
```

Session, with explanation:

```python
>>> c = Line(3, 4)          # calling the class creates an instance
>>> c.slope                 # dot expression looks up an attribute
3
>>> c.intercept
4
>>> c                       # repr supplied by @dataclass
Line(slope=3, intercept=4)
>>> type(c) == Line         # the class statement created a new type
True
>>> isinstance(c, Line)
True
>>> print(c)                # print calls __str__, which calls format_line
y = 3x + 4
```

Object-level reasoning (in words): `Line(3, 4)` creates a new object in memory with two attributes; the name `c` in the global frame points to that object. `c.slope` does not look in a frame, it looks *inside the object*. When `print(c)` runs, Python finds `Line`'s `__str__`, calls it with `self` bound to the same object `c` points to, and that call's body calls the global function `format_line` passing `self` along. So `print(c)` involves three connected things: an instance, a method on its class, and a plain function.

And `parallel`:

```python
>>> parallel(Line(3, 4), Line(3, 10))
True
>>> parallel(Line(3, 4), Line(5, 4))
False
```

Two lines are parallel exactly when their slopes match, regardless of intercept.

---

## Common Pitfalls

1. **Indexing a dict by value.** `numerals[10]` is an error even though `10` is a value in the dict. Square brackets on a dict look up **keys**, not positions and not values.
2. **Assuming a missing key raises.** `d['missing']` raises, but `d.get('missing')` quietly returns `None`, and `d.get('missing', 0)` returns `0`. Know which one you want.
3. **Using a list as a key.** `{[1]: 2}` is an error; `{1: [2]}` is fine. Keys must be immutable; values may be anything.
4. **Expecting duplicate keys to error.** `{1: 2, 1: 3}` evaluates to `{1: 3}`. The later entry silently wins.
5. **Forgetting that iterating a dict gives keys.** `list(numerals)` is `['I', 'V', 'X']`, not the values and not the pairs. Use `.values()` for values.
6. **Filtering the wrong thing in `multiples`.** Putting the condition on the outer loop (over `factors`) would drop keys whose lists are empty; the spec requires `8: []` to appear. The filter belongs on the inner comprehension over `s`.
7. **Swapping the two recursive calls in `cp`.** `cp(n-m, m)` keeps `m` available (so a size can be reused); `cp(n, m-1)` reduces the largest allowed piece without consuming any of `n`. Writing `cp(n-m, m-1)` would forbid repeats and give the wrong answer.
8. **Base-case order in `cp`.** `n == 0` must be checked before `m == 0`, otherwise `cp(0, 0)` returns `0` instead of `1`. Likewise in `cp_3total_helper`, `n == 0` must precede `k == 0`.
9. **Trying to enforce a count-limit without an extra argument.** The slides are explicit: "there is no way to add a base case for using too many pieces, because we don't know how many pieces we've used." If a constraint concerns history, the history must be an argument.
10. **Returning `0` when a per-piece budget runs out.** In `cp_3_of_each_piece_helper`, `k == 0` means "no more of size `m`", not "dead end". You must still recurse on `m-1` with the budget reset to `t`. Returning `0` there would be the "at most `k` total" behavior instead.
11. **Forgetting to reset the budget when the piece size shrinks.** Passing `k` instead of `t` in the `m-1` calls would carry a partially-spent budget to a different size.
12. **Collapsing `count_park`'s two `n-1` calls into one.** They represent two different choices (`%` and `.`). If you write a single `count_park(n-1)`, you undercount. `2 * count_park(n-1) + count_park(n-2)` is the correct collapse.
13. **Omitting the `n < 0` base case in `count_park`.** Without it, trying to park a car in one remaining spot recurses forever downward (or at least produces wrong results). It must return `0`, not `1`: an overhanging car is not a valid arrangement.
14. **Confusing `n == 0` returning `1` with "nothing happened".** Reaching exactly zero remaining spots (or zero remaining sum) means one complete valid arrangement was built, so it contributes `1` to the count.
15. **Calling a method without parentheses.** `d.get` is the method object itself; `d.get(4)` invokes it. Similarly `today.year` is a value-attribute (no call), while `today.strftime('%A')` is a method call.
16. **Thinking type hints enforce types.** `slope: float` declares an attribute and documents intent; `Line(3, 4)` with ints works fine.

---

## Likely Exam Points

### 1. Writing a dictionary comprehension (often with a nested list comprehension)

**Practice.** Implement `by_length`, which takes a list of strings `words` and a list of positive integers `lengths`, and returns a dict mapping each integer in `lengths` to the list of words in `words` with that length.

```python
>>> by_length(['a', 'bb', 'cc', 'ddd'], [1, 2, 4])
{1: ['a'], 2: ['bb', 'cc'], 4: []}
```

**Answer.**
```python
def by_length(words, lengths):
    return {n: [w for w in words if len(w) == n] for n in lengths}
```
The outer comprehension iterates over `lengths` (the keys), and the inner one filters `words`. Every `n` gets a key even when the list is empty, as with `4: []`.

### 2. Dictionary behavior trivia

**Practice.** Give the value of each expression, or write `Error`, given `d = {'a': 1, 'b': 2}`.
(i) `len(d)` (ii) `list(d)` (iii) `sum(d.values())` (iv) `d.get('c')` (v) `d.get('c', -1)` (vi) `d[1]` (vii) `{'a': 1, 'a': 5}` (viii) `dict([['x', 7]])`

**Answer.** (i) `2` (ii) `['a', 'b']` (iii) `3` (iv) `None` (v) `-1` (vi) `Error` (looking up key `1`, which isn't present) (vii) `{'a': 5}` (viii) `{'x': 7}`.

### 3. Filling blanks in a `cp` variant with a tracking argument

**Practice.** Fill in the blanks so that `cp_even(n, m)` counts the ways to make `n` from positive pieces up to `m` in increasing order, using **only an even number of copies of each piece size** (0, 2, 4, ... copies of each size). For example, `cp_even(4, 2) == 2` (`2+2` and `1+1+1+1`).

```python
def cp_even(n, m):
    return helper(n, m, 0)

def helper(n, m, used):
    """used = how many copies of size m we have already placed."""
    if n == 0 and used % 2 == 0:
        return 1
    elif n < 0 or m == 0:
        return 0
    use_m = helper(________, ________, ________)
    skip_m = helper(________, ________, ________) if used % 2 == 0 else 0
    return use_m + skip_m
```

**Answer.**
```python
    use_m = helper(n-m, m, used+1)
    skip_m = helper(n, m-1, 0) if used % 2 == 0 else 0
```
`use_m` places one more copy of `m` and increments the counter. `skip_m` moves on to size `m-1`, resetting the counter to `0` for the new size, but is only legal if the count for `m` is currently even. Note `n == 0` alone is not enough for success: we must also have finished size `m` with an even count. This is exactly the lecture's lesson (add an argument that tracks whether the constraint was violated) plus the budget-reset idea from `cp_3_of_each_piece_helper`.

### 4. Distinguishing "limit per piece" from "limit in total"

**Practice.** For `n = 4`, `m = 2`: list the partitions counted by (a) `cp`, (b) `cp_3total`, (c) `cp_3_of_each_piece`, and explain why the counts differ.

**Answer.**
(a) `cp(4, 2) = 3`: `2+2`, `1+1+2`, `1+1+1+1`.
(b) `cp_3total(4, 2) = 2`: drops `1+1+1+1` because it uses 4 pieces total.
(c) `cp_3_of_each_piece(4, 2) = 2`: also drops `1+1+1+1`, but for a different reason (it uses four copies of the *same* size). Here the two answers coincide; they differ in general, e.g. `1+1+2+2` is allowed by (c) but not (b), which is exactly the example given on the slide.

### 5. Tree recursion from a "first choice" prompt (the `count_park` family)

**Practice.** Suppose trucks are added, taking 3 adjacent spots and written `<=>`. Modify `count_park` accordingly, and give the new value for `n = 3`.

**Answer.**
```python
def count_park_truck(n):
    if n < 0:
        return 0
    elif n == 0:
        return 1
    return (count_park_truck(n-1) + count_park_truck(n-1)
            + count_park_truck(n-2) + count_park_truck(n-3))
```
The small choice is still "what fills the first space", now with four options: `%` (1 spot), `.` (1 spot), `<>` (2 spots), `<=>` (3 spots). For `n = 3`: `f(0)=1`, `f(1)=2f(0)+f(-1)+f(-2)=2`, `f(2)=2f(1)+f(0)+f(-1)=5`, `f(3)=2f(2)+f(1)+f(0)=10+2+1=13`... plus the truck term is `f(0)=1` and is already included, so recompute carefully: `f(3) = f(2)+f(2)+f(1)+f(0) = 5+5+2+1 = 13`. So `13`, which is the original `count_park(3)` count of 13 plus the one all-truck arrangement `<=>`... note that the original `count_park(3)` is also 13, and the truck version is `13 + 1 = 14`. Recompute cleanly: with trucks, `f(1)=2`, `f(2)=2·2+1+0=5`, `f(3)=2·5+2+1=13`; without trucks, `g(1)=2`, `g(2)=2·2+1=5`, `g(3)=2·5+2=12`. So the answer is `13` with trucks versus `12` without, and the extra arrangement is `<=>`.

(Correction to the enumeration on the slide as I listed it earlier: `count_park(3) == 12`, and the slide's 13-line list included `<><>`, which is a length-4 string; the 12 length-3 arrangements are `%%%`, `%%.`, `%.%`, `%..`, `%<>`, `.%%`, `.%.`, `..%`, `...`, `.<>`, `<>%`, `<>.`.)

### 6. Data class attributes and `__str__`

**Practice.** Given the `Line` data class from lecture, what does each of the following display?
(i) `Line(2, 0).slope` (ii) `Line(2, 0)` at the REPL (iii) `print(Line(2, 0))` (iv) `isinstance(Line(2, 0), Line)` (v) `parallel(Line(2, 0), Line(2, 9))`

**Answer.** (i) `2` (ii) `Line(slope=2, intercept=0)` (iii) `y = 2x + 0` (iv) `True` (v) `True`.

### 7. Writing your own data class

**Practice.** Define a data class `Point` with attributes `x: float` and `y: float`, a `__str__` that produces `(3, 4)` for `Point(3, 4)`, and a function `same_x(p: Point, q: Point) -> bool` returning whether two points share an x-coordinate.

**Answer.**
```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

    def __str__(self):
        return f'({self.x}, {self.y})'

def same_x(p: Point, q: Point) -> bool:
    return p.x == q.x
```
The two type-hinted names become the attributes, in declaration order, so `Point(3, 4)` binds `x` to `3`. `__str__` takes `self` and is reached through `print`/`str`.

### 8. Vocabulary: attribute vs. method vs. dot expression

**Practice.** In `today.strftime('%A')` and `today.year`, identify the dot expressions, say which one accesses a method, and explain why `today.year` needs no parentheses.

**Answer.** Both `today.strftime` and `today.year` are dot expressions (an expression on the left, a dot, an attribute name on the right). `today.strftime` is an attribute bound to a function, hence a method, and `today.strftime('%A')` calls it. `today.year` is an attribute bound to a value (an integer), so there is nothing to call; adding parentheses would attempt to call an integer and error.

### 9. Recursion pattern identification

**Practice.** Classify each as "deal with one item, recurse for the rest" or "make a small choice, recurse for each option": `reverse(s)`, `count_partitions(n, m)`, `double_eights(s)`, `count_park(n)`, `streak(n)`.

**Answer.** `reverse`, `double_eights`, and `streak` are the first kind (one recursive call, one item handled per frame). `count_partitions` and `count_park` are the second kind, tree recursion, with multiple recursive calls per frame, one per option of a small choice.

---

## Summary

- A **dict** is an object: the key-value correspondence is its data, and methods like `.get` are its behavior. Iterating a dict gives keys; `.values()` gives values; `.get(k, default)` avoids a missing-key error; keys must be immutable but values need not be; duplicate keys silently keep the last.
- **Dictionary comprehension**: `{<key exp>: <value exp> for <name> in <iter exp> if <filter exp>}`. `multiples` is `{d: [x for x in s if x % d == 0] for d in factors}`, a list comprehension nested inside a dict comprehension, with the filter on the *inner* loop so every factor still gets a key.
- **Objects** bundle data and behavior and behave like what they represent. A **class** is a type of object; an **instance** is created by calling the class; **attributes** are reached by **dot expressions**; attributes bound to functions are **methods**. Classes are first-class values. Every Python value is an object.
- **Functions** should do one thing; **objects** bundle many related behaviors. Object-oriented programming organizes a large program as objects exchanging messages.
- `@dataclass` turns type-hinted names into instance attributes and supplies a constructor and a readable `repr`. You define `__str__` to control `print`/`str`. Operations may be written as plain functions with `Line` type hints (`parallel`, `format_line`).
- Recursion so far comes in two shapes: handle one item and recurse on the rest (`double_eights`, `streak`, `reverse`), or make a small choice and recurse for each option (**tree recursion**: `count_partitions`, `count_park`).
- `cp(n, m) = cp(n-m, m) + cp(n, m-1)`, with `n == 0 → 1` and (`n < 0` or `m == 0`) `→ 0`. The first call uses at least one `m`; the second uses none.
- Three `cp` variations: adding base cases (e.g. `m == 1 → 1`) skips work; varying recursive calls (`if m > n: return cp(n, m-1)`) removes the need for the `n < 0` base case; and **to add a constraint, add arguments that track whether the constraint was violated**.
- "At most 3 pieces total" threads one counter `k`, decremented when a piece is used, with `k == 0 → 0`. "At most 3 of each piece" threads `k` (remaining for current size) and `t` (the per-size limit), and when `k == 0` it recurses on `m-1` with the budget **reset** to `t` rather than returning `0`.
- `count_park(n)`: the small choice is how to fill the first space, giving `count_park(n-1)` for `%`, `count_park(n-1)` for `.`, and `count_park(n-2)` for `<>`; base cases `n < 0 → 0` (a car that doesn't fit) and `n == 0 → 1` (one complete arrangement). Values: `count_park(1) = 2`, `count_park(2) = 5`, `count_park(4) = 29`.
