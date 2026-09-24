<!-- Wed, Sep 23, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 12: Containers

This lecture is a consolidation lecture: it takes lists (introduced earlier) and pushes them in three directions at once. First, it nails down exactly what the square-bracket notation `[]` means in its several distinct roles (list literal, list comprehension, element selection, slicing) and what happens at the boundaries (indexing off the end is an `IndexError`, slicing off the end is fine). Second, it introduces **box-and-pointer notation** so that lists can appear in environment diagrams, which matters because lists satisfy the **closure property** (a list can contain lists), so container values form hierarchical structures that a single box cannot describe. Third, it shows the two main ways to *process* containers: writing your own iteration or recursion (using slices such as `s[1:]` as "the rest of the list"), and calling built-in **aggregation** functions (`sum`, `max`, `min`, `all`, `any`) that collapse an iterable into one value, often with a `key` function. The lecture then treats **strings** as a sequence type that is deliberately a little different from lists (element selection yields a one-character string, and `in` tests for substrings), and finishes with tree recursion that *builds* strings rather than just counting: the parking-lot problem (`count_park` and `park`), where the base case `park(0) == ['']` is the crux. The recorded video segments bundled with this lecture also cover **dictionaries** and **dictionary comprehensions**, which are included below and flagged as such.

---

## Key Concepts

### 1. The many meanings of `[]`

Square brackets do four different jobs, and the lecture's "List Review" slide deliberately puts them side by side so you learn to tell them apart by *context*:

| Use | Example | What it produces |
|---|---|---|
| List literal | `[1, 8, 0, 1]` | a brand-new list you wrote out element by element |
| List comprehension | `[d * 100 for d in digits if d < 5]` | a brand-new list, described by telling Python *how* to make each element |
| Element selection | `digits[1]` | the one element at that index (not a list) |
| Slicing | `digits[1:]` | a brand-new list containing some of the elements |

Plus `+` on two lists makes a new list with all elements of the first followed by all elements of the second:

```python
>>> digits = [1, 8, 0, 1]
>>> same_digits = [digits[0]] + digits[1:]
>>> same_digits
[1, 8, 0, 1]
```

Note that `same_digits` is *equal* to `digits` but is a **different list object**: `[digits[0]]` is a new one-element list and `digits[1:]` is a new three-element list, and `+` builds a fourth new list from them.

### 2. Indexing is strict; slicing is forgiving

This asymmetry is a favorite exam target:

```python
>>> digits[1]
8
>>> digits[100]
IndexError: list index out of range
>>> digits[:1000]      # ok to go off the end
[1, 8, 0, 1]
>>> digits[1000:]      # ok to start past the end
[]
```

A slice clamps its bounds to the list; an index does not. This is why `s[1:]` on a one-element list quietly gives `[]` (which makes recursion on lists work smoothly) while `s[1]` on a one-element list crashes.

Slice bounds follow the same rule as `range`: **include the lower bound, exclude the upper bound**. As the video shows, `odds[1:3]` on `[3, 5, 7, 9, 11]` gives `[5, 7]`, and slicing is just a compact spelling of `[odds[i] for i in range(1, 3)]`.

An omitted start means "from the beginning", an omitted end means "to the end", and omitting both (`s[:]`) gives you all the elements (in a new list).

### 3. Expressions compose, so `[]` can stack

```python
>>> [d * 100 for d in digits if d < 5][1]
0
```
The comprehension evaluates to `[100, 0, 100]`, and then `[1]` selects from that anonymous list. There is no special rule here: it is just an element selection applied to whatever the preceding expression evaluated to.

### 4. Slicing always creates new values

From the video: if you start with `digits` and take three different slices, the resulting environment diagram contains **four separate lists**, and `digits` itself is unchanged. Slicing never modifies the original and never shares boxes with it. This is why recursive list functions that pass `s[1:]` around are safe but also why they do real work (each call builds a new list).

### 5. The closure property and hierarchical data

> A method for combining data values satisfies the **closure property** if the result of combination can itself be combined using the same method.

Lists satisfy it: if you can put items in a list, you can put a list in a list. Closure is what lets you build **hierarchical structures**, structures made of parts that are themselves made of parts. This is the reason box-and-pointer notation is needed at all: once values can nest arbitrarily, you need a notation that tracks what is inside what.

### 6. Box-and-pointer notation

The rule is exactly two sentences:

- A list is drawn as a **row of index-labeled adjacent boxes, one box per element** (indexes written small in the corner, starting at 0).
- **Each box either contains a primitive value or points to a compound value.**

So for `pair = [1, 2]`, the name `pair` in the global frame points to a two-box row containing `1` and `2`.

For the lecture's bigger example:

```python
nested_list = [[1, 2], [],
               [[3, False, None],
                [4, lambda: 5]]]
```

- `nested_list` is a **three**-element list, so three adjacent boxes (indexes 0, 1, 2).
- Every element here happens to be a list, so all three boxes hold **arrows** rather than values.
- Index 0 points to a two-box list holding `1` and `2`.
- Index 1 points to the **empty list** (drawn as an empty box or a slash, with no elements).
- Index 2 points to a two-element list, whose element 0 points to a three-box list holding the primitives `3`, `False`, and `None` written directly in their boxes, and whose element 1 points to a two-box list holding the primitive `4` and an **arrow to a function value** (`lambda: 5`, drawn the way lambdas are always drawn: `func λ() [parent=Global]`).

Anything with multiple parts (a list, a function) gets an arrow; anything primitive (`int`, `bool`, `None`, and in practice strings too) is written inside the box.

*(The transcript says "the element at index three" for the last element; that is a caption slip, since a three-element list has indexes 0, 1, 2.)*

### 7. Passing a list to a function does not copy it

This is the point of the discussion question. When you call `f(t[1])`, the parameter `s` is bound to **the very same list object** that `t`'s box 1 points to. Two arrows, one list. Nothing is duplicated. Copies only appear when you slice, concatenate, or build a new list literal/comprehension.

### 8. Two idioms for iterating over a sequence

`double_eights` is presented twice on purpose:

- **Positions (indices)**: loop `for i in range(len(s) - 1)` and compare `s[i]` with `s[i+1]`. The `- 1` is there because you look one step ahead.
- **Slices (recursion)**: check the front (`s[:2] == [8, 8]`), then recurse on `s[1:]` (the rest of the list), with `len(s) < 2` as the failure base case.

The slice version is the "list recursion" pattern: `s[0]` is the first element, `s[1:]` is the rest, and the base case fires when the list is too short.

### 9. Aggregation: built-ins that collapse an iterable

> Several built-in functions take iterable arguments and aggregate them into a value.

- `sum(iterable[, start]) -> value`: return the sum of an iterable (not of strings) plus `start`, which defaults to `0`. If the iterable is empty, return `start`.
- `max(iterable[, key=func]) -> value` **or** `max(a, b, c, ...[, key=func]) -> value`: with a single iterable, return its largest item; with two or more arguments, return the largest argument.
- `all(iterable) -> bool`: return `True` if `bool(x)` is `True` for every `x` in the iterable. **If the iterable is empty, return `True`.**
- `min` and `any` are the complements of `max` and `all`.

Three things worth internalizing:

1. **The square brackets in the docs mean "optional argument"**, not "pass a list here". You do not type them.
2. **`start` is not just for skipping ahead.** It supplies the identity value *of the right type*. `sum([[2, 3], [4]])` fails because it tries `0 + [2, 3]`; `sum([[2, 3], [4]], [])` succeeds and gives `[2, 3, 4]`. Same reasoning for why `sum` refuses strings.
3. **`key=func` maps before comparing.** `max` applies `func` to each candidate, compares the *return values*, but returns the original element. `max(range(10), key=lambda x: 7 - (x - 4) * (x - 2))` returns `3`, because that parabola peaks at `x = 3` (value `8`), and `3` is what comes back, not `8`.

### 10. Truthiness, revisited

`all` and `any` are defined in terms of `bool(x)`, so the Boolean-context rules from the conditionals lecture come back: false values include `0`, `False`, `''`, `[]`, and `None`; everything else (`5`, `-1`, `'hello'`) is a true value. Hence `all(range(5))` is `False` (because `0` is in there) even though `all([x < 5 for x in range(5)])` is `True`.

### 11. Strings are a sequence abstraction, with quirks

Strings represent textual data, and the lecture stresses that they are an **abstraction**: you use them without caring how characters are encoded. They can represent numbers (`'1.25'`, `'1.25e-6'`), natural language, and even **programs** (a Python source file is just a string; `eval`/`exec` on a string can define functions).

Mechanics covered:

- Three ways to write a string: single quotes, double quotes, and triple quotes. Single and double quoted strings are equivalent; double quotes let you include an apostrophe (`"Where's Waldo?"`) without ending the string.
- A **triple-quoted string can span multiple lines** (which is why docstrings use them). When such a value is displayed, the newlines show up as `\n`.
- A backslash **escapes** the following character, so `\n` is *one* character in the sequence, a line feed.
- `len('Berkeley')` is `8`. Each element is a **character**.
- **Element selection returns a string, not a character type.** `'Berkeley'[3]` is `'k'`, a one-character string. Lists do not behave this way: selecting from a list of numbers gives a number, not a one-element list.
- **`in` / `not in` test for substrings, not just single elements.** `'here' in "Where's Waldo?"` is `True`. For lists, `in` only checks one element at a time: `234 in [1, 2, 3, 4, 5]` is `False`, and `[2, 3] in [1, 2, 3, 4]` is `False`. The rationale: with text you usually care about whole words, not individual letters.

### 12. Tree recursion that builds strings

The parking problem is the payoff. Two versions of the same recursive structure:

- `count_park(n)` **counts** arrangements.
- `park(n)` **returns the arrangements themselves**, as a list of strings.

The recursive decomposition is "what goes in the leftmost spot?": a motorcycle `'%'` (1 spot), an empty spot `'.'` (1 spot), or a car `'<>'` (2 spots). Two of the three branches consume one spot and one consumes two, which is why `count_park` adds `count_park(n-1)` **twice** plus `count_park(n-2)` once.

Going from counting to listing is a mechanical transformation:

| counting | listing |
|---|---|
| `0` (no ways) | `[]` (no arrangements) |
| `1` (one way: park nothing) | `['']` (the one arrangement is the empty string) |
| `a + b` | `list_a + list_b` |
| `count_park(n-1)` | `['%' + s for s in park(n-1)]` (prefix every sub-arrangement) |

The base case `park(0) == ['']` is the whole trick. `['']` is a list with one element (the empty string), so `['%' + s for s in park(0)]` gives `['%']`. If you wrongly wrote `return []`, every comprehension would iterate over nothing and `park` would always return `[]`.

### 13. Dictionaries *(covered in the lecture video segments; the extracted slide deck for this lecture ends at `park`)*

A dictionary holds **key-value pairs**, written with curly braces and colons:

```python
>>> numerals = {'I': 1, 'V': 5, 'X': 10}
```

- **Lookup uses the same square-bracket syntax as lists, but you put a key in, not an index.** `numerals['V']` is `5`; `numerals[0]` is a `KeyError` (there is no key `0`); `numerals['X-ray']` is a `KeyError`.
- Lookup goes **one way only**: you cannot ask for the key that maps to `5`.
- Iterating a dictionary (or calling `list` on it) gives you its **keys**. `numerals.values()` gives a `dict_values` object, a sequence that is *not* a list (you can `sum` it or `for`-loop over it; call `list(...)` if you truly need a list). This parallels how `range` is a sequence but not a list.
- `len(d)` is the **number of keys**, regardless of how big the values are.
- Two restrictions on keys:
  1. **A key cannot be a list or a dictionary** (`TypeError: unhashable type: 'list'`). This comes from Python's implementation. ("Unhashable" is left for CS 61B.) Values, by contrast, can be anything, including lists and dictionaries.
  2. **Keys cannot repeat.** `{1: 'first', 1: 'second'}` keeps key `1` once. There is at most one value per key; that restriction is part of the dictionary *abstraction*. If you need several values for one key, make the value a list.

### 14. Dictionary comprehensions

```python
{<key exp>: <value exp> for <name> in <iter exp> if <filter exp>}
```

The filter is optional. Evaluation mirrors the list comprehension rule: add a new frame whose parent is the current frame, create an empty result dictionary, then for each element of the iterable value, bind `<name>` to it in the new frame, and if the filter is a true value, add an entry pairing the key expression's value to the value expression's value.

```python
>>> {x * x: x for x in [1, 2, 3, 4, 5] if x > 2}
{9: 3, 16: 4, 25: 5}
```
`1` and `2` are filtered out; `3`, `4`, `5` contribute `9: 3`, `16: 4`, `25: 5`.

---

## Definitions

- **Container / sequence**: a value that holds other values and supports length and element selection. Lists, ranges, strings, and (as sequences of keys) dictionaries are the examples in this lecture.
- **Closure property**: a method of combining data values has it if the result of a combination can itself be combined by the same method. Lists have it, so lists can contain lists.
- **Hierarchical structure**: a structure made up of parts that are themselves made up of parts, enabled by closure.
- **Box-and-pointer notation**: the environment-diagram convention for drawing lists as a row of index-labeled adjacent boxes, one per element, where each box either contains a primitive value or points (by arrow) to a compound value.
- **Primitive value**: a value written directly inside its box in a diagram (numbers, booleans, `None`).
- **Compound value**: a value with multiple parts (a list, a function), drawn separately and referred to by an arrow.
- **Element selection / indexing**: `s[i]`, which evaluates to the single element at index `i`, and raises `IndexError` if `i` is out of range.
- **Slicing operator**: `s[start:end]`, square brackets containing a colon, which evaluates to a **new** sequence containing the elements from index `start` up to but not including `end`. Omitting `start` means the beginning, omitting `end` means the end, and out-of-range bounds are clamped rather than an error.
- **List comprehension**: `[<map exp> for <name> in <iter exp> if <filter exp>]`, an expression that evaluates to a new list by describing how each element is created.
- **Dictionary comprehension**: `{<key exp>: <value exp> for <name> in <iter exp> if <filter exp>}`, an expression that evaluates to a new dictionary.
- **Aggregation**: collapsing an iterable into a single value, e.g. with `sum`, `max`, `min`, `all`, `any`.
- **Iterable**: a value that can be iterated over, such as a list, range, string, `dict_values`, or dictionary.
- **`key` function**: an optional function argument to `max`/`min` that is applied to each candidate; comparison uses the returned values, but the **original element** is returned.
- **`start` (of `sum`)**: the value the summation begins from, defaulting to `0`; it is what an empty iterable returns, and it is how you sum non-numbers such as lists.
- **`bool(x)`**: the built-in that reports whether `x` is a true or false value in a Boolean context. `all`/`any` are defined in terms of it.
- **Character**: a single element of a string. In Python, selecting one yields a string of length 1.
- **Escape sequence**: a backslash plus a following character that together denote a single character in the string, e.g. `\n` (line feed).
- **Key** (dictionary): the value used to look something up; cannot be a list or dictionary, and cannot be duplicated within a dictionary.
- **Value** (dictionary): the thing associated with a key; may be arbitrarily complicated, including a list or another dictionary.

---

## Worked Examples

### Example 1: `reverse` (the four poll options)

The reference solution:

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

Why it works: reverse the rest of the list, then stick the old first element on the **end**. On `[4, 6, 2]`: `reverse([6, 2]) + [4]` → `(reverse([2]) + [6]) + [4]` → `((reverse([]) + [2]) + [6]) + [4]` → `[2] + [6]` → `[2, 6]` → `[2, 6, 4]`.

Now the four candidates:

**(A) `reverse(s[1:] + [s[0]])`**

```python
def reverseA(s):
    if not s:
        return []
    reverseA(s[1:] + [s[0]])      # note: no return, even
```
The argument `s[1:] + [s[0]]` is a **rotation**, not a shrink. On `[4, 6, 2]`: `s[1:]` is `[6, 2]`, `[s[0]]` is `[4]`, so the recursive call gets `[6, 2, 4]`, still length 3. The list never becomes empty, so the base case never fires and this **runs forever** (`RecursionError` in practice). Two separate bugs: no progress toward the base case, and (in the code version) no `return`, so it would evaluate to `None` even if it terminated.

**(B) `[s[-1]] + reverse(s[:-1])`** ✅ correct

```python
def reverseB(s):
    if not s:
        return []
    return [s[-1]] + reverseB(s[:-1])
```
This is the mirror image of the reference answer: put the **last** element first, then reverse everything except the last. On `[4, 6, 2]`: `[2] + reverseB([4, 6])` → `[2] + ([6] + reverseB([4]))` → `[2] + [6, 4]` → `[2, 6, 4]`. It shrinks every call (`s[:-1]` is one shorter), so it terminates.

**(C) `[s[-1 * i] for i in range(len(s))]`** ❌ wrong, returns `[4, 2, 6]`

```python
def reverseC(s):
    if not s:
        return []
    return [s[-1 * i] for i in range(len(s))]
```
`range(len(s))` gives `i = 0, 1, 2`, so the indexes used are `s[-1*0] == s[0]`, `s[-1*1] == s[-1]`, `s[-1*2] == s[-2]`. That is `4`, `2`, `6`, so the result is `[4, 2, 6]`. The bug is that `-1 * 0` is `0`, which is the **first** element, not the last: negative indexing starts at `-1`, and there is no `-0`. It is not recursive at all (the recursive spirit is absent), and it terminates but computes the wrong thing.
*(extra context)* The fix is to shift by one: `[s[-1 - i] for i in range(len(s))]`.

**(D) `[s[-x + 1] for x in range(reverse(s))]`** ❌ runs forever

```python
def reverseD(s):
    if not s:
        return []
    return [s[-x + 1] for x in range(reverseD(s))]
```
To evaluate `range(reverseD(s))`, Python must first evaluate the operand `reverseD(s)`, with the **same** `s`. No progress, so infinite recursion, and the `range` is never even reached. *(extra context)* Even if it were, `range` of a list would be a `TypeError`.

*(extra context)* Outside of an exercise like this, `s[::-1]` reverses a sequence in one step, but extended slices with a step were not part of this lecture.

### Example 2: Discussion question (box-and-pointer)

```python
def f(s):
    x = s[0]
    return [x]

t = [3, [2+2, 5]]
u = [f(t[1]), t]
print(u)
```

Step by step:

1. `def f(s)` binds `f` in the Global frame to a function value.
2. `t = [3, [2+2, 5]]`: the inner list literal is evaluated first, producing a two-box list holding `4` and `5` (the `2+2` is evaluated at construction time, so `4` is stored, not an expression). The outer list is a two-box row: box 0 contains the primitive `3`, box 1 holds an **arrow** to the `[4, 5]` list. `t` points to the outer list.
3. `u = [f(t[1]), t]`: evaluate the two operand expressions of the list literal left to right.
   - `t[1]` evaluates to the inner list `[4, 5]`, **the same object**, not a copy. Calling `f` on it opens a new frame `f1` whose `s` points to that very same `[4, 5]` list (so the diagram has two arrows into it: one from `t`'s box 1, one from `s`). In that frame, `x = s[0]` binds `x` to the primitive `4`. Then `return [x]` builds a brand-new one-box list containing `4`.
   - `t` evaluates to the outer list (again, the same object, no copy).
4. So `u` points to a new two-box list: box 0 points to the new `[4]` list, box 1 points to the same outer list that `t` points to.

**What gets printed:**

```
[[4], [3, [4, 5]]]
```

The moral: `print` shows structure but hides sharing. The printed output gives no hint that `u[1]` **is** `t` rather than a copy of it; only the diagram shows that.

### Example 3: `double_eights`, two ways

**Positions (indices):**

```python
def double_eights(s):
    """Return whether two consecutive items of list s are 8.

    >>> double_eights([1, 2, 8, 8])
    True
    >>> double_eights([8, 8, 0])
    True
    >>> double_eights([5, 3, 8, 8, 3, 5])
    True
    >>> double_eights([2, 8, 4, 6, 8, 2])
    False
    """
    for i in range(len(s) - 1):
        if s[i] == 8 and s[i + 1] == 8:
            return True
    return False
```

Why `len(s) - 1`: the body looks at `s[i + 1]`, so the largest safe `i` is `len(s) - 2`, and `range(len(s) - 1)` stops exactly there. On `[5, 3, 8, 8, 3, 5]` the loop checks index pairs (0,1), (1,2), (2,3) and returns `True` at `i = 2`. On `[2, 8, 4, 6, 8, 2]` there are two 8s but never adjacent, so the loop finishes and `return False` runs. Note the `return False` is **after** the loop, not inside it: returning `False` on the first non-match would abandon the search too early.

**Slices (recursion):**

```python
def double_eights(s):
    if s[:2] == [8, 8]:
        return True
    elif len(s) < 2:
        return False
    else:
        return double_eights(s[1:])
```

Reasoning: either the doubled 8s are at the very front, or they are somewhere in `s[1:]`. `s[:2]` is safe on short lists because slicing clamps (on `[8]` it gives `[8]`, which is not `[8, 8]`). Trace on `[5, 3, 8, 8, 3, 5]`: front is `[5, 3]`, recurse on `[3, 8, 8, 3, 5]`; front `[3, 8]`, recurse on `[8, 8, 3, 5]`; front is `[8, 8]`, return `True`, and that `True` is returned back up the chain unchanged. On `[2, 8, 4, 6, 8, 2]` it keeps shrinking until `len(s) < 2` and returns `False`.

### Example 4: `summation` with an aggregation

```python
def cube(k):
    return pow(k, 3)

def summation(n, term):
    """Sum the first n terms of a sequence.

    >>> summation(5, cube)
    225
    """
    total, k = 0, 1
    while k <= n:
        total, k = total + term(k), k + 1
    return total
```
`summation(5, cube)` computes `1 + 8 + 27 + 64 + 125 = 225`.

The one-line rewrite:

```python
def summation2(n, term):
    return sum([term(x) for x in range(1, n + 1)])
```

Two details to get exactly right:
- `range(1, n + 1)` produces `1, 2, 3, 4, 5` for `n = 5`. Starting at `1` matches `k = 1`, and the `+ 1` is needed because `range` excludes its upper bound.
- The comprehension applies `term` to each value, producing the list of terms; `sum` then aggregates. No `start` argument is needed because the terms are numbers and the default `0` is the right identity.

### Example 5: Spring 2023 Midterm 2 question (prefix sums)

The slide is an image, so only the two filled blanks survived in the text: `sum(s[:k+1])` and `range(len(s))`. Together they form a comprehension over positions whose element is the sum of the list up to and including that position:

```python
# (extra context: reconstruction of the problem's shape from the two answers shown)
[sum(s[:k + 1]) for k in range(len(s))]
```

Reading it: for each index `k` of `s`, `s[:k+1]` is the prefix ending at `k` (the `+1` is because slicing excludes the upper bound), and `sum` collapses it. On `[1, 2, 3, 4]` this gives `[1, 3, 6, 10]`, the running totals. The transferable lesson is the pairing of `range(len(s))` (to get every valid index) with `s[:k+1]` (to get everything up to and including that index).

### Example 6: Aggregation demos from `12.py`

```python
min(range(10), key=lambda i: abs(50 ** 0.5 - i))
```
`50 ** 0.5` is about `7.071`. The key gives the distance from each candidate `i` to `7.071`: `i = 7` scores about `0.071`, `i = 8` about `0.93`, everything else worse. So this evaluates to **`7`**: the element of `range(10)` closest to the square root of 50. Note `min` returns the element (`7`), not the key's value (`0.071`).

```python
sum([2, 3, 4], sum([20, 30, 40]))
```
Inner `sum([20, 30, 40])` is `90`, used as the `start`, so this is `90 + 2 + 3 + 4` = **`99`**.

```python
any([x > 10 for x in range(10)])
```
The comprehension is ten `False` values (the largest `x` is `9`), so this is **`False`**. `any` returns `True` if at least one element is a true value; the complement, `all`, needs every element true.

More from the video:

```python
>>> sum([2, 3, 4])
9
>>> sum([[2, 3], [4]], [])      # start supplies a list to begin from
[2, 3, 4]
>>> max(range(5))
4
>>> max(range(10), key=lambda x: 7 - (x - 4) * (x - 2))
3
>>> all([x < 5 for x in range(5)])
True
>>> all(range(5))               # 0 is a false value
False
```

### Example 7: Strings demo

```python
def str_example():
    """
    >>> s = '3 * '
    >>> print(s + s + s + 10)
    Traceback (most recent call last):
        ...
    TypeError: can only concatenate str (not "int") to str
    >>> print(s + s + s + str(10))
    3 * 3 * 3 * 10
    >>> eval(s + s + s + str(10))
    270
    """
```

Line by line:
- `s + s + s` is `'3 * 3 * 3 * '`. String `+` is concatenation, and it requires both operands to be strings, hence the `TypeError` when you append the integer `10`.
- `str(10)` converts the number to the string `'10'`, so concatenation succeeds and `print` displays the text `3 * 3 * 3 * 10`.
- That text happens to be a valid Python expression, and `eval` evaluates a string as an expression: `3 * 3 * 3 * 10` is `270`. This is the "strings can represent programs" point made concrete. The video's other version of this idea: the string `'lambda f: lambda x: lambda y: f(x, y)'` is inert text until you execute it, at which point you have a working `curry`.

Related string facts demonstrated:

```python
>>> len('Berkeley')
8
>>> 'Berkeley'[3]
'k'                      # a string of length 1, not a "char" type
>>> 'here' in "Where's Waldo?"
True                     # substring search
>>> 234 in [1, 2, 3, 4, 5]
False                    # lists check one element at a time
```

### Example 8: `count_park` (tree recursion, counting)

```python
def count_park(n):
    """Count the ways to park cars and motorcycles in n adjacent spots.

    >>> count_park(1)  # '.' or '%'
    2
    >>> count_park(2)  # '..', '.%', '%.', '%%', or '<>'
    5
    >>> count_park(4)  # some examples: '<><>', '.%%.', '%<>%', '%.<>'
    29
    """
    if n < 0:
        return 0
    elif n == 0:
        return 1
    else:
        return count_park(n - 2) + count_park(n - 1) + count_park(n - 1)
```

The setup: a motorcycle `%` takes 1 spot, a car `<>` takes 2 adjacent spots, `.` is an empty spot, and a string of length `n` represents `n` adjacent spots (for example `'.%%.<><>'`).

The recursion asks "what occupies the leftmost spot?" There are exactly three answers:
- a car, using 2 spots, leaving `n - 2` → `count_park(n - 2)`
- a motorcycle, using 1 spot, leaving `n - 1` → `count_park(n - 1)`
- nothing (an empty spot), using 1 spot, leaving `n - 1` → `count_park(n - 1)`

The last two look identical as numbers but are genuinely different choices, hence the term appears **twice**. Because each branch commits the leftmost spot, the three branches never produce the same arrangement, so adding them does not double count.

The base cases:
- `n == 0` returns `1`: there is exactly **one** way to fill zero spots, namely do nothing. Returning `0` here would make the whole function return `0`.
- `n < 0` returns `0`: this arises only from the car branch when `n == 1`, where a car does not fit. Returning `0` means "this choice was impossible", so it contributes nothing to the sum. This is cleaner than special-casing `n == 1`.

Verifying against the doctests:
- `count_park(1) = count_park(-1) + count_park(0) + count_park(0) = 0 + 1 + 1 = 2` ✓
- `count_park(2) = count_park(0) + count_park(1) + count_park(1) = 1 + 2 + 2 = 5` ✓
- `count_park(3) = count_park(1) + count_park(2) + count_park(2) = 2 + 5 + 5 = 12` (matches the 12 strings the slide lists for `park(3)`)
- `count_park(4) = count_park(2) + count_park(3) + count_park(3) = 5 + 12 + 12 = 29` ✓

### Example 9: `park` (tree recursion, building strings)

```python
def park(n):
    """Return the ways to park cars and motorcycles in n adjacent spots.

    >>> park(1)
    ['%', '.']
    >>> park(2)
    ['%%', '%.', '.%', '..', '<>']
    >>> len(park(4))  # some examples: '<><>', '.%%.', '%<>%', '%.<>'
    29
    """
    if n < 0:
        return []
    elif n == 0:
        return ['']
    else:
        return ([('%' + s) for s in park(n - 1)]
                + [('.' + s) for s in park(n - 1)]
                + [('<>' + s) for s in park(n - 2)])
```

Same three branches, but now each branch **prefixes** its own symbol onto every arrangement of the remaining spots, and `+` concatenates the three lists of results.

Base cases, and why they are what they are:
- `park(0)` returns `['']`, a list of **one** string, the empty string. That says "there is one way to park in zero spots, and it is written as the empty string." It is the list-shaped version of `count_park`'s `return 1`. If you wrote `return []` (a common error: reaching for "empty" without thinking about what is being counted), then `['%' + s for s in park(0)]` would iterate zero times and every call would produce `[]`.
- `park(-1)` returns `[]`, a list of **zero** arrangements, the list-shaped version of `return 0`. The car branch on `n == 1` contributes nothing.

Trace:
- `park(0)` = `['']`
- `park(1)` = `['%' + '']` + `['.' + '']` + `[]` = `['%', '.']` ✓
- `park(2)` = `['%%', '%.']` + `['.%', '..']` + `['<>' + '']` = `['%%', '%.', '.%', '..', '<>']` ✓ (the `'<>'` comes from `park(0)`, and note how the order of the three concatenated lists determines the order of the output, matching the doctest exactly)
- `park(3)` produces the slide's 12 strings in this order: `%%%, %%., %.%, %.., %<>, .%%, .%., ..%, ..., .<>, <>%, <>.` Reading that list confirms the structure: the first five all begin with `%` (they are `'%' + s` for the five elements of `park(2)`), the next five begin with `.`, and the last two begin with `<>` (they are `'<>' + s` for the two elements of `park(1)`).
- `len(park(4))` is `29`, agreeing with `count_park(4)`, which is the whole point: the two functions share one recursive structure.

### Example 10: `index` (dictionary comprehension) *(from the video segments)*

```python
def index(keys, values, match):
    """Return a dictionary from keys k to a list of values v for which
    match(k, v) is a true value.

    >>> index([7, 9, 11], range(30, 50), lambda k, v: v % k == 0)
    {7: [35, 42, 49], 9: [36, 45], 11: [33, 44]}
    """
    return {k: [v for v in values if match(k, v)] for k in keys}
```

How to read it:
- "A dictionary from keys to lists" means each entry pairs one of the given keys with a list value.
- The keys are handed to us, so the key expression is just `k`, and the iteration expression is `keys`.
- The value expression is itself a **list comprehension**: all `v` in `values` for which `match(k, v)` is true. This works because the inner comprehension is nested inside the outer one, so its body can refer to both `v` (its own loop name) and `k` (the outer loop name) through the frame chain.
- With `match = lambda k, v: v % k == 0`, the list for key `k` is the multiples of `k` in the range: multiples of 7 in `range(30, 50)` are `35, 42, 49`; of 9, `36, 45`; of 11, `33, 44`.
- `values` here is a `range`, which can be iterated once per key without being exhausted, which is what makes reusing it in the inner comprehension safe.

---

## Common Pitfalls

1. **Confusing "index off the end" with "slice off the end."** `s[len(s)]` is an `IndexError`; `s[len(s):]` is `[]`. Recursive list code leans on the second behavior, so do not add defensive length checks that are already handled by clamping (but *do* keep the checks that stop the recursion).
2. **Forgetting slicing makes a copy.** `s[1:]` does not shrink `s`; it builds a new list. So `s[1:]` inside a recursive call cannot affect the caller's list, and `[digits[0]] + digits[1:]` is a different object from `digits` even though it prints identically.
3. **Assuming a printed list reveals its structure.** `print(u)` for the discussion question shows `[[4], [3, [4, 5]]]` whether or not `u[1]` is the same object as `t`. Sharing is invisible in output and visible only in the diagram.
4. **Drawing a copy when a list is passed to a function.** `f(t[1])` binds `s` to the existing list. Draw a second arrow, not a second row of boxes.
5. **Miscounting boxes.** A list of `k` elements gets exactly `k` boxes, indexes `0` through `k-1`. The empty list gets zero boxes (not one empty box holding nothing meaningful).
6. **Writing a recursive call that does not shrink the problem.** `reverse(s[1:] + [s[0]])` (rotation) and `range(reverse(s))` (same argument) both loop forever. Always ask: is the argument strictly smaller?
7. **Off-by-one with negative indexes.** `-1` is the last element and there is no `-0`, which is precisely why `[s[-1 * i] for i in range(len(s))]` starts from the front and produces `[4, 2, 6]`.
8. **Omitting `return` in front of a recursive call.** `reverseA` computes the answer and throws it away, returning `None`.
9. **Putting `return False` inside a search loop.** In `double_eights`, `return False` must come after the loop; inside, it would quit on the first non-match.
10. **Typing the square brackets from the docs.** `sum(iterable[, start])` means `start` is optional. You write `sum(lst)` or `sum(lst, 0)`, not `sum(lst[, 0])`.
11. **Calling `sum` on a list of lists without a `start`.** `sum([[2, 3], [4]])` fails because `0 + [2, 3]` is a `TypeError`. Pass `[]`. And `sum` refuses strings entirely.
12. **Thinking `key` changes the return value.** `max(iterable, key=f)` returns the winning **element**, not `f` of it.
13. **Forgetting the empty-iterable conventions.** `all([])` is `True`, `any([])` is `False`, and `sum([])` is `start` (default `0`).
14. **Forgetting `0` and `''` are false values.** `all(range(5))` is `False` because of the `0`, not because of any comparison.
15. **Expecting `in` to behave the same on strings and lists.** `'234' in '12345'` is `True`, but `234 in [1, 2, 3, 4, 5]` and `[2, 3] in [1, 2, 3, 4]` are both `False`.
16. **Expecting a character type.** `'Berkeley'[3]` is the string `'k'`, so `len('Berkeley'[3])` is `1`, and you can index it again (`'Berkeley'[3][0]` is `'k'`).
17. **Using `[]` instead of `['']` as the base case of a list-building recursion.** The single most likely way to get `park` wrong. "One way to do nothing" is `['']`, not `[]`.
18. **Collapsing the two `count_park(n - 1)` terms into `2 * count_park(n - 1)` and then forgetting which is which in `park`.** In `park`, the two branches are genuinely different (`'%' + s` versus `'.' + s`) and cannot be merged.
19. **Indexing a dictionary by position.** `numerals[0]` is a `KeyError` unless `0` is literally a key. Dictionaries have no order-based indexing in the `d[i]` syntax.
20. **Trying to look up a key by its value, or using a list/dict as a key.** Neither works; the latter gives `TypeError: unhashable type: 'list'`. Lists as *values* are fine.
21. **Treating `d.values()` as a list.** It is a `dict_values` sequence; `sum` and `for` work, but wrap it in `list(...)` if you need list operations.

---

## Likely Exam Points

### 1. Evaluating index/slice/comprehension expressions

**Q.** Given `digits = [1, 8, 0, 1]`, what does each of these evaluate to (or what error)?
(a) `digits[1:]` (b) `digits[1000:]` (c) `digits[100]` (d) `[d * 100 for d in digits if d < 5][1]`

**A.** (a) `[8, 0, 1]`. (b) `[]` (slice bounds are clamped). (c) `IndexError: list index out of range`. (d) `0`: the comprehension keeps `1, 0, 1` and scales them to `[100, 0, 100]`, and index `1` of that is `0`.

### 2. Box-and-pointer diagrams with sharing

**Q.** Draw the diagram and give the printed output:
```python
def g(s):
    return [s[1], s]

a = [1, [2, 3]]
b = g(a)
print(b)
```

**A.** `a` points to a two-box list: box 0 contains `1`, box 1 points to a two-box list `[2, 3]`. Calling `g` opens a frame with `s` bound to the **same** outer list as `a`. `[s[1], s]` builds a new two-box list whose box 0 points to the existing `[2, 3]` list and whose box 1 points to the existing outer list. So there are only **two** lists' worth of new structure: one new row, and three arrows into pre-existing lists. Printed: `[[2, 3], [1, [2, 3]]]`. The two occurrences of `[2, 3]` in the output are the *same* list object shown twice.

### 3. "Which of these is correct, and what do the wrong ones do?"

**Q.** For `def reverse(s)` with base case `if not s: return []`, classify each body: (A) `return reverse(s[1:] + [s[0]])`, (B) `return [s[-1]] + reverse(s[:-1])`, (C) `return [s[-1 * i] for i in range(len(s))]`.

**A.** (A) infinite recursion: the argument is a rotation of `s`, always the same length, so the base case is never reached. (B) correct. (C) terminates but is wrong: the indexes used are `s[0]`, `s[-1]`, `s[-2]`, so on `[4, 6, 2]` it returns `[4, 2, 6]`; the bug is that `-1 * 0` is `0`, the first element.

### 4. Rewriting a loop as an aggregation (and vice versa)

**Q.** Fill in the blank so the two agree: `def summation2(n, term): return ____`.

**A.** `sum([term(x) for x in range(1, n + 1)])`. `range(1, n + 1)` yields `1` through `n` inclusive, the comprehension applies `term`, and `sum` aggregates.

### 5. `sum` / `max` / `min` / `all` / `any` semantics, including `key` and `start`

**Q.** Evaluate: (a) `sum([2, 3, 4], sum([20, 30, 40]))` (b) `min(range(10), key=lambda i: abs(50 ** 0.5 - i))` (c) `any([x > 10 for x in range(10)])` (d) `all([])` (e) `sum([[1], [2]], [])`

**A.** (a) `99` (`90` as the start, plus `2 + 3 + 4`). (b) `7` (closest integer in `range(10)` to about `7.071`; the element is returned, not the key value). (c) `False`. (d) `True` (an empty iterable satisfies `all` vacuously). (e) `[1, 2]`.

### 6. String behavior versus list behavior

**Q.** True or False, with reasons: (a) `len('Berkeley'[3]) == 1` (b) `'234' in '12345'` (c) `[2, 3] in [1, 2, 3, 4]` (d) `sum(['1', '2'])` works.

**A.** (a) True: selecting from a string gives a one-character string. (b) True: `in` on strings tests for a substring. (c) False: `in` on a list compares against individual elements, and no element equals the list `[2, 3]`. (d) False: `sum` explicitly does not work on strings.

### 7. Tree recursion base cases, counting and listing

**Q.** Suppose a student writes `park` with `elif n == 0: return []` and everything else correct. What does `park(3)` return, and why?

**A.** `[]`. Each comprehension iterates over the result of a recursive call, and with `park(0) == []` there is nothing to prefix, so `park(1)` is `[] + [] + []`, and the emptiness propagates all the way up. The correct base case is `['']`: exactly one arrangement of zero spots, written as the empty string.

**Q.** Why does `count_park` include `count_park(n - 1)` twice rather than once?

**A.** Two distinct choices each consume one spot: placing a motorcycle, and leaving the spot empty. They yield the same count but different arrangements (`'%' + s` versus `'.' + s` in `park`), so both must be counted.

**Q.** Write `count_park(n)` in terms of `park(n)`.

**A.** `len(park(n))`. (This is the check `len(park(4)) == 29`.)

### 8. Dictionary rules *(from the video segments)*

**Q.** Given `d = {1: [1, 2], 3: 'hello'}`, evaluate `len(d)`, `d[3]`, `d[0]`, and `{[1]: 'x'}`.

**A.** `len(d)` is `2` (the number of keys, regardless of value sizes). `d[3]` is `'hello'`. `d[0]` is a `KeyError` (square brackets take keys, not positions). `{[1]: 'x'}` raises `TypeError: unhashable type: 'list'`, since a key cannot be a list (values can be).

### 9. Writing a dictionary comprehension

**Q.** Evaluate `{x * x: x for x in [1, 2, 3, 4, 5] if x > 2}`.

**A.** `{9: 3, 16: 4, 25: 5}`. `1` and `2` fail the filter; each surviving `x` contributes the pair `x*x: x`.

---

## Summary

- `[]` means four different things depending on context: list literal, list comprehension, element selection, and slicing. Learn to read which one you are looking at.
- **Indexing out of range is an `IndexError`; slicing out of range is clamped.** Slice bounds include the start and exclude the end, like `range`; an omitted bound means "the beginning" or "the end".
- **Slicing (and `+`, and comprehensions, and list literals) always creates new lists.** The original is untouched.
- Lists satisfy the **closure property**, so they nest and form hierarchical structures, which is why diagrams need box-and-pointer notation.
- **Box-and-pointer**: a list is a row of index-labeled adjacent boxes, one per element; each box holds a primitive value or an arrow to a compound value (list or function).
- **Passing a list to a function shares it; it does not copy it.** Printed output hides sharing, diagrams reveal it.
- Recursion on lists uses `s[0]` for the first element and `s[1:]` (or `s[-1]` and `s[:-1]`) for the rest. A recursive call must strictly shrink its argument, or it runs forever.
- `reverse(s)` = `reverse(s[1:]) + [s[0]]` = `[s[-1]] + reverse(s[:-1])`. Beware `-1 * 0 == 0`.
- `double_eights` has an index version (`for i in range(len(s) - 1)`, compare `s[i]` and `s[i+1]`, `return False` after the loop) and a slice version (`s[:2] == [8, 8]`, base case `len(s) < 2`, recurse on `s[1:]`).
- **Aggregation built-ins** collapse an iterable: `sum(iterable[, start])`, `max`/`min(iterable[, key=func])` or `max(a, b, ...)`, `all`/`any(iterable)`. Square brackets in the docs mean "optional".
- `start` supplies the identity value of the right **type** (use `[]` to sum lists); `key` maps before comparing but the **original element** is returned; `all([])` is `True` and `any([])` is `False`; truthiness is `bool(x)`, so `0`, `''`, `[]`, `None` are false.
- **Strings** are an abstraction over textual data that can represent numbers, language, and even programs (`eval` on a string evaluates it as an expression).
- Strings use single, double, or triple quotes (triple can span lines); `\n` is one escaped character (line feed); element selection yields a **one-character string**; `in` tests for **substrings**, unlike lists which test single elements.
- **Tree recursion on the parking problem**: three choices for the leftmost spot (`%` costs 1, `.` costs 1, `<>` costs 2), so `count_park(n) = count_park(n-2) + count_park(n-1) + count_park(n-1)`, with `count_park(0) == 1` and `count_park(n < 0) == 0`.
- Turning counting into listing: `0` becomes `[]`, `1` becomes `['']`, `+` becomes list concatenation, and each recursive term gets prefixed via a comprehension. `park(0) == ['']` is the essential base case, and `len(park(n)) == count_park(n)`.
- *(From the lecture video segments)* **Dictionaries** are collections of key-value pairs written `{k: v, ...}`; lookup uses `d[key]` (a missing key is a `KeyError`); iterating a dict gives keys and `d.values()` gives a `dict_values` sequence; `len(d)` counts keys; keys cannot be lists or dictionaries and cannot repeat, while values can be anything.
- *(From the lecture video segments)* **Dictionary comprehension**: `{<key exp>: <value exp> for <name> in <iter exp> if <filter exp>}`, and its value expression can itself be a list comprehension that refers to the outer name, as in `index`: `{k: [v for v in values if match(k, v)] for k in keys}`.
