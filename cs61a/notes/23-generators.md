<!-- Mon, Oct 19, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 23: Generators

## Overview

This lecture builds directly on the iterator interface (`iter`, `next`, `StopIteration`) and introduces **generator functions**: functions that contain a `yield` statement instead of (or in addition to) `return`. Calling a generator function does not run its body; it immediately returns a **generator**, which is an iterator that runs the function's body lazily, pausing at each `yield` and resuming from exactly that point on the next call to `next`. Because the paused function keeps its entire environment (local variables, position in loops, position in the call stack), a generator function can express infinite sequences (`cycle`, `yield_ints`), interleaved sequences, and recursive enumerations (`park`, `partitions`) far more cleanly than code that builds and concatenates lists. The lecture also introduces `yield from`, which is shorthand for a `for` loop that yields every element of another iterable, and demonstrates the practical payoff of laziness: the generator version of `partitions(60, 50)` can hand you the first 10 of roughly 966,000 partitions instantly, while the list-building version must construct all of them first. The lecture closes with an unrelated list mutation / box-and-pointer warm-up (`chain`, `silver`, `gold`, `platinum`) from a past midterm.

---

## Key Concepts

### 1. Recap: the iterator interface

Two built-in functions define the whole interface:

- `iter(iterable)` returns a **new iterator** over the elements of an iterable value.
- `next(iterator)` returns the **next** element of an iterator, advancing it. When there are no more elements, it raises `StopIteration`.

An iterator has *state*: it remembers how far it has gotten. Calling `next` twice on the same iterator gives two different values; the iterator is consumed as you go and cannot be rewound.

```python
>>> t = iter([3, 4, 5, 6])
>>> next(t)
3
>>> next(t)
4
```

The warm-up question asks you to pull a fixed number of items off an iterator, which matters when the iterator is huge (or infinite) and you only want a prefix:

```python
def get_n(t, n):
    """Return a list of the next n items from iterator t."""
    return [next(t) for _ in range(n)]
```

The loop variable is `_` because it is never used: we are looping only to count to `n`. Note that `get_n` works on `iter(range(1000000000))` without building a billion-element list, because `range` objects and their iterators are lazy.

### 2. A generator function is a function with `yield` in it

```python
def plus_minus(x):
    yield x
    yield -x
```

What makes this a generator function is purely syntactic: the body contains a `yield` statement. The consequences are large:

- A normal function **returns once**. A generator function can **yield many times**.
- Calling a generator function does **not** execute its body at all. It immediately returns a **generator object**.
- That generator object is an iterator: you can call `next` on it, loop over it with `for`, pass it to `list`, etc.

```python
>>> t = plus_minus(3)
>>> next(t)
3
>>> next(t)
-3
>>> t
<generator object plus_minus ...>
```

So: *a generator is an iterator created automatically by calling a generator function, and it iterates over the values the function yields.*

### 3. The execution model: pause and resume

This is the conceptual heart of the lecture. Think of the generator as owning a *suspended frame* of the generator function call.

1. `t = f(args)` creates the generator and a frame for `f` with the arguments bound, but **executes no statements**.
2. The first `next(t)` starts executing the body from the top.
3. Execution continues until a `yield <expr>` is reached. The value of `<expr>` becomes the return value of `next(t)`, and the function **pauses right there**, keeping all local variables and the current position in every loop and `if` intact.
4. The next `next(t)` resumes execution on the statement *after* that `yield`.
5. If the body runs off the end (or hits a bare `return`), the generator raises `StopIteration`.

The lecture's `evens` example makes the pausing visible:

```python
def evens(start, end):
    even = start + start % 2   # round start up to the next even number
    while even < end:
        yield even
        even += 2
```

```python
>>> t = evens(2, 10)
>>> next(t)
2
>>> next(t)
4
>>> list(evens(1, 10))
[2, 4, 6, 8]
```

In environment terms: the frame for `evens` holds `start`, `end`, and `even`. After the first `next`, that frame still exists with `even = 2` and a saved "program counter" sitting at the `yield`. The second `next` resumes at `even += 2`, loops back to the `while` header, re-checks `even < end`, and yields again. Nothing re-runs from the top, which is why the loop makes progress instead of restarting.

The payoff is **laziness**: you can describe an arbitrarily long (even infinite) computation, and only the part that someone actually asks for gets executed.

### 4. Infinite generators are normal and useful

Because computation only happens on demand, `while True:` inside a generator function is a perfectly good design, not a bug:

```python
def cycle(s):
    """Iterate over the elements of s repeatedly."""
    while True:
        for x in s:
            yield x
```

```python
>>> t = cycle([3, 4, 5])
>>> [next(t) for _ in range(10)]
[3, 4, 5, 3, 4, 5, 3, 4, 5, 3]
```

Two nested control structures are both suspended across yields here: the `while` and the `for`. When `next` resumes the generator, it picks up in the middle of the `for` loop with the correct `x` position in `s`. The only rule is that the *consumer* must stop asking (do not call `list(cycle(s))`).

Similarly, `yield_ints` interleaves two infinite sequences by alternating yields:

```python
def yield_ints(start):
    """Yield all integers, expanding from start."""
    yield start
    offset = 1
    while True:
        yield start + offset
        yield start - offset
        offset += 1
```

```python
>>> t = yield_ints(100)
>>> [next(t), next(t), next(t), next(t), next(t)]
[100, 101, 99, 102, 98]
```

The single `yield start` before the loop handles the center value exactly once, and each pass of the loop emits the pair `(start + offset, start - offset)` before bumping `offset`. Writing this as a function that returns a list is impossible: the list is infinite.

### 5. `yield from`

Added in Python 3.3 (2012). `yield from <iterable>` yields **every** element of that iterable, one at a time, as elements of the current generator. It is exactly equivalent to:

```python
for x in <iterable>:
    yield x
```

but avoids naming the intermediate element.

```python
def a_then_b(a, b):
    yield from a
    yield from b
```

The crucial distinction is `yield gen` versus `yield from gen`:

- `yield gen` contributes **one** element, and that element *is the generator object itself*.
- `yield from gen` contributes **all of the elements** of `gen`.

The lecture's `countdown` shows the bug and the fix:

```python
def countdown(k):
    if k > 0:
        yield k
        yield from countdown(k - 1)   # WRONG if written: yield countdown(k - 1)
    else:
        yield 'Blast off'
```

```python
>>> for k in countdown(3):
...     print(k)
3
2
1
Blast off
```

If you write `yield countdown(k - 1)`, then `list(countdown(3))` gives `[3, <generator object ...>]`: the second item is a generator, not the number 2.

Two more recursive examples from the demo:

```python
def prefixes(s):
    if s:
        yield from prefixes(s[:-1])
        yield s

def substrings(s):
    if s:
        yield from prefixes(s)
        yield from substrings(s[1:])
```

```python
>>> list(prefixes('both'))
['b', 'bo', 'bot', 'both']
>>> list(substrings('tops'))
['t', 'to', 'top', 'tops', 'o', 'op', 'ops', 'p', 'ps', 's']
```

The output order of `prefixes` is shortest-first precisely because the recursive `yield from` comes *before* `yield s`. Swapping those two lines reverses the order. `substrings` reads as a clean statement of the idea: the substrings of `s` are the prefixes of `s`, followed by the substrings of everything after the first character.

### 6. Generator functions versus list-building functions

This is the lecture's main "why bother" argument, developed through partitions.

A **partition** of positive integer `n` using parts up to size `m` is a way of writing `n` as a sum of positive integers, each at most `m`, in increasing order.

The familiar counting version:

```python
def count_partitions(n, m):
    if n < 0 or m == 0:
        return 0
    else:
        exact_match = 0
        if n == m:
            exact_match = 1
        return exact_match + count_partitions(n - m, m) + count_partitions(n, m - 1)
```

The three cases are: use `m` exactly once to finish off `n` (only when `n == m`), use at least one `m` and keep going, or use no `m` at all.

The list version (reconstructed from the demo) must wrap everything in lists and glue them together:

```python
def list_partitions(n, m):
    if n < 0 or m == 0:
        return []
    else:
        exact_match = []
        if n == m:
            exact_match = [[m]]
        with_m = [p + [m] for p in list_partitions(n - m, m)]
        without_m = list_partitions(n, m - 1)
        return exact_match + with_m + without_m
```

The generator version replaces every "build a list of" with "yield", and every "concatenate lists of results" with `yield from`:

```python
def partitions(n, m):
    """Yield the partitions of n using parts up to size m, as strings."""
    if n > 0 and m > 0:
        if n == m:
            yield str(m)
        for p in partitions(n - m, m):
            yield p + ' + ' + str(m)
        yield from partitions(n, m - 1)
```

```python
>>> for p in partitions(6, 4):
...     print(p)
2 + 4
1 + 1 + 4
3 + 3
1 + 2 + 3
1 + 1 + 1 + 3
2 + 2 + 2
1 + 1 + 2 + 2
1 + 1 + 1 + 1 + 2
1 + 1 + 1 + 1 + 1 + 1
```

Three things the lecture emphasized about this transformation:

1. **No wrapper lists.** Instead of `[[m]]` you write `yield str(m)`: just think about one element at a time.
2. **No list comprehension for the recursive case.** You loop over the sub-results and yield a modified version of each.
3. **The base cases shrink.** Because a generator that yields nothing is the natural "no results" answer, the `return []` base case can be folded into the guard `if n > 0 and m > 0`, and there is no `return` value at all.

And the efficiency argument: `len(list(partitions(60, 50)))` is almost a million, and building that list takes a while. But asking the generator for one partition, or the first ten, is essentially instant, because the recursion only unwinds far enough to produce what was requested. The list version has no choice but to build all 966,000-odd results before you can look at the first one.

### 7. Where the filtering lives: generator versus consumer

Two of the lecture's exam-style problems contrast *who* does the filtering.

In `print_cycle`, the generator (`cycle`) produces everything and the **consumer** filters and prints:

```python
def print_cycle(n, s):
    t = cycle(s)
    for i in range(n):
        x = next(t)
        if i % 2 == 1:
            print(x)
```

Here `n` counts how many items are *pulled from the cycle*, not how many are printed.

In `check_sequence`, the **generator** filters, so the consumer is trivial:

```python
def check_sequence(check):
    """
    >>> t = check_sequence(lambda x, y: y > x)
    >>> [next(t) for i in range(4)]
    [1, 2, 4, 5]
    """
    previous = 1
    while True:
        if previous % 3 != 0:
            yield previous
        i = 1
        while not check(previous, i):
            i += 1
        previous = i

def print_check_sequence(check, n):
    t = check_sequence(check)
    for _ in range(n):
        print(next(t))
```

The sequence is: start at 1; the next term is the smallest positive `i` with `check(previous, i)` true. Terms divisible by 3 are computed (they are needed to find the following term) but skipped silently by the `if`, so `next(t)` always hands back a non-multiple of 3. Here `n` counts *printed* terms. This is the same structural idea as `print_cycle` with the filter moved across the boundary, and exams love to swap which side it is on.

---

## Definitions

- **Iterable**: a value that `iter` accepts, producing an iterator over its elements (lists, strings, tuples, dicts, ranges, and iterators themselves).
- **Iterator**: an object with mutable state representing a position in a sequence of values; `next(iterator)` returns the next value and advances, raising `StopIteration` when exhausted.
- **`iter(iterable)`**: returns an iterator over the elements of an iterable value.
- **`next(iterator)`**: returns the next element in an iterator, advancing its position.
- **`StopIteration`**: the exception raised by `next` when an iterator has no more elements; `for` loops and `list` catch it to terminate normally.
- **Generator function**: a function whose body contains at least one `yield` statement. Calling it executes none of the body and returns a generator.
- **Generator**: an iterator created automatically by calling a generator function; it iterates over the values yielded by the body of that function.
- **`yield <expr>`**: produces one element of the generator's sequence and suspends execution of the generator function's frame at that point, preserving all local state until the next `next` call resumes it.
- **`yield from <iterable>`**: produces, in order, every element of `<iterable>` as elements of the current generator; equivalent to `for x in <iterable>: yield x`.
- **Lazy evaluation**: computing values only when they are requested, rather than all at once in advance. Generators are the lecture's main example.
- **Partition of n using parts up to size m**: a way of expressing `n` as a sum of positive integer parts, each at most `m`, written in increasing order.
- **Parking string**: a string of length `n` using `%` for a motorcycle (1 spot), `<>` for a car (2 adjacent spots), and `.` for an empty spot, representing `n` adjacent parking spots.

---

## Worked Examples

### Example 1: `get_n`, pulling a prefix off an iterator

```python
def get_n(t, n):
    """ Return a list of the next n items from iterator t.

    >>> t = iter(range(1000000000))
    >>> get_n(t, 10)
    [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
    """
    return [next(t) for _ in range(n)]
```

Step by step: `range(n)` drives the comprehension `n` times. Each iteration evaluates `next(t)`, which mutates `t` (advancing it) and produces one value. The comprehension collects those `n` values into a new list. Because `t` is shared state, a second call `get_n(t, 10)` would return `[10, 11, ..., 19]`, not `[0, ..., 9]`. The billion-element range is never materialized; only ten `next` calls happen.

### Example 2: `plus_minus` and the pause/resume trace

```python
def plus_minus(x):
    yield x
    yield -x
```

- `t = plus_minus(3)`: a generator object is created. A frame for `plus_minus` exists with `x` bound to 3, but execution has not started. Nothing is printed, no error occurs even if the body is nonsense below the first yield.
- `next(t)`: body starts, reaches `yield x`, evaluates `x` to 3, returns 3, pauses at that line.
- `next(t)`: resumes at `yield -x`, returns -3, pauses.
- `next(t)`: resumes, falls off the end of the body, raises `StopIteration`.

### Example 3: `cycle` plus `print_cycle` (Fall 2026 Practice Midterm 2, Q3 modified)

```python
def cycle(s):
    """Iterate over the elements of s repeatedly.

    >>> t = cycle([3, 4, 5])
    >>> get_n(t, 10)
    [3, 4, 5, 3, 4, 5, 3, 4, 5, 3]
    """
    while True:
        for x in s:
            yield x
```

Why this works: the `for x in s` loop creates a fresh iterator over `s` each time the `while` loop comes around, so the elements repeat forever. Between `next` calls, the generator's frame remembers both that it is inside the `while` and where it is inside the `for`.

The slide version of the consumer filters by **value** (print only the odd numbers):

```python
def print_cycle(n, s: list):
    """Prints the first n items of the cycle sequence, but only the odd items.

    >>> pi = [3, 1, -4, -1, 5]
    >>> print_cycle(7, pi)   # [3] [1] -4 [-1] [5] [3] [1]
    3
    1
    -1
    5
    3
    1
    """
    t = cycle(s)
    for _ in range(n):
        x = next(t)
        if x % 2 == 1:
            print(x)
```

Trace for `print_cycle(7, [3, 1, -4, -1, 5])`: the cycle produces `3, 1, -4, -1, 5, 3, 1` over seven `next` calls. Each is tested for oddness; `-4` fails and is skipped. Six lines print. Note `-1 % 2` is `1` in Python, so negative odd numbers pass the test (this is exactly the kind of detail the question is probing).

The `23.py` code version filters by **index** instead, printing items at odd positions:

```python
def print_cycle(n, s):
    """
    >>> pi = [3, 1, -4, -1, 5]
    >>> print_cycle(7, pi)  # 3 [1] -4 [-1] 5 [3] 1
    1
    -1
    3
    >>> print_cycle(1, pi)  # 3
    >>> print_cycle(5, [5, 7])  # 5 [7] 5 [7] 5
    7
    7
    """
    t = cycle(s)
    for i in range(n):
        x = next(t)
        if i % 2 == 1:
            print(x)
```

Both variants appeared in the lecture materials (slides versus demo file); the only difference is `x % 2 == 1` versus `i % 2 == 1`. Read the docstring and doctests carefully to tell which one a problem wants. In the index version, `print_cycle(1, pi)` prints nothing because index 0 is even, and `print_cycle(5, [5, 7])` prints `7` twice because only indices 1 and 3 qualify.

### Example 4: `yield_ints` (Discussion Question)

```python
def yield_ints(start):
    """ Yield all integers, expanding from start.

    >>> t = yield_ints(100)
    >>> [next(t), next(t), next(t), next(t), next(t)]
    [100, 101, 99, 102, 98]
    """
    yield start
    offset = 1
    while True:
        yield start + offset
        yield start - offset
        offset += 1
```

The blanks, in order, were `yield start`, `True`, `yield start + offset`, `yield start - offset`.

Trace: `next` #1 runs `yield start` and gives 100. `next` #2 resumes at `offset = 1`, enters the loop, yields `100 + 1 = 101`. `next` #3 resumes immediately after that yield and gives `100 - 1 = 99`. `next` #4 resumes at `offset += 1` (now 2), loops back, and yields 102. `next` #5 gives 98. The generator never terminates, which is fine: the caller only asked for five values.

Key observation: `offset` lives in the suspended frame. There is no way to write this with a plain function that returns, and no way to write it with a list.

### Example 5: `park` (Spring 2023 Midterm 2, Q5b)

A motorcycle `%` takes one spot, a car `<>` takes two adjacent spots, `.` is an empty spot. Yield every way to fill `n` adjacent spots.

```python
def park(n):
    """Yield the ways to park cars and motorcycles in n adjacent spots.

    >>> sorted(park(1))
    ['%', '.']
    >>> sorted(park(2))
    ['%%', '%.', '.%', '..', '<>']
    >>> len(list(park(4)))   # some examples: '<><>', '.%%.', '%<>%', '%.<>'
    29
    """
    if n == 0:
        yield ''
    elif n > 0:
        for s in park(n - 1):
            yield '%' + s
            yield '.' + s
        for s in park(n - 2):
            yield '<>' + s
```

The recursive idea is the tree on the slides: look at the **first** spot and ask what goes there. There are three choices, and each one reduces the problem:

- Put a motorcycle there: one spot consumed, so prepend `'%'` to every way of parking the remaining `n - 1` spots.
- Leave it empty: same thing with `'.'`, also over `park(n - 1)`.
- Put a car there: two spots consumed, so prepend `'<>'` to every way of parking `park(n - 2)`.

Base cases:

- `n == 0`: there is exactly **one** way to park nothing, namely the empty string, so `yield ''`. Yielding `''` (rather than yielding nothing) is what makes the recursive prepending produce complete strings.
- `n < 0`: a car did not fit. The `elif n > 0` guard means the body does nothing, so the generator yields **zero** results, which correctly contributes nothing to the total. This is why `park(1)` works: its `for s in park(-1)` loop body never executes.

Counting check: `p(0) = 1`, `p(1) = 2`, and `p(n) = 2 * p(n-1) + p(n-2)`, giving `p(2) = 5`, `p(3) = 12`, `p(4) = 29`, matching the docstring.

The commented-out alternative in the demo file shows the non-generator version:

```python
return (['%'  + s for s in park(n-1)] +
        ['.'  + s for s in park(n-1)] +
        ['<>' + s for s in park(n-2)])
```

It is the same algorithm, but it calls `park(n-1)` twice, builds three lists, and concatenates them.

*(extra context)* Watch the **ordering**: the generator version yields `'%' + s` and `'.' + s` interleaved inside one loop, so `list(park(2))` is `['%%', '.%', '%.', '..', '<>']`, whereas the list-comprehension version groups all `%` results first and gives `['%%', '%.', '.%', '..', '<>']`. The slides use `sorted(park(n))` in the doctests precisely to make order irrelevant; the order-sensitive `list(park(2))` and `list(park(3))` doctests in `23.py` correspond to the commented-out list version. If an exam demands a specific order, use two separate loops over `park(n - 1)`.

### Example 6: `check_sequence` traced

```python
def check_sequence(check):
    previous = 1
    while True:
        if previous % 3 != 0:
            yield previous
        i = 1
        while not check(previous, i):
            i += 1
        previous = i
```

With `check = lambda x, y: y > x`: terms are 1, 2, 3, 4, 5, ... The 3 is computed and used to find 4, but never yielded, so `[next(t) for i in range(4)]` is `[1, 2, 4, 5]`.

With `check = lambda x, y: y > 2 * x`: from 1, the smallest `i` with `i > 2` is 3 (skipped, multiple of 3); from 3, smallest `i > 6` is 7 (yielded); from 7, smallest `i > 14` is 15 (skipped); from 15, smallest `i > 30` is 31. So `print_check_sequence(check, 3)` prints `1, 7, 31`.

With `check = lambda x, y: (y == x - 2 or y > x + 10)`: from 1, no positive `i` equals `-1`, so the smallest qualifying `i` is 12 (skipped); from 12, `i == 10` qualifies and is found first by the upward search (yielded); then 8; then 6 (skipped); then 4. Output: `1, 10, 8, 4`.

The inner `while not check(previous, i): i += 1` is a linear search for the smallest qualifying `i`, and it runs *between* yields inside the suspended frame.

### Example 7: List practice with box-and-pointer reasoning (Spring 2023 Midterm 2, Q1)

```python
def chain(s):
    return [s[0], s[1:]]

silver = [2, chain([3, 4, 5])]
gold = [silver[0], silver[1].pop()]
silver[0] = 1
platinum = chain(chain([6, 7, 8]))
```

Step by step, in environment terms:

1. `chain([3, 4, 5])` opens a frame with `s` bound to the list `[3, 4, 5]`. It returns a **new** list whose first element is `3` and whose second element is `s[1:]`, a **new** list `[4, 5]` (slicing copies). So `silver` is a new list `[2, [3, [4, 5]]]`, where the inner `[3, [4, 5]]` is the object returned by `chain`.
2. `gold = [silver[0], silver[1].pop()]`. Operands are evaluated left to right. `silver[0]` evaluates to `2` first. Then `silver[1]` is the list `[3, [4, 5]]`, and `.pop()` **mutates** it, removing and returning its last element, the list `[4, 5]`. So `gold` is `[2, [4, 5]]` and, as a side effect, `silver` is now `[2, [3]]`. `gold`'s second element is the *same object* that used to be inside `silver[1]`, not a copy.
3. `silver[0] = 1` rebinds index 0 of the `silver` list to `1`. `gold[0]` still holds the integer `2`: integers are immutable values that were copied into `gold` by reference to the value, so changing `silver`'s slot does not touch `gold`'s slot. Now `silver` is `[1, [3]]`.
4. `platinum = chain(chain([6, 7, 8]))`. Inner call: `chain([6, 7, 8])` returns `[6, [7, 8]]`. Outer call: `s` is `[6, [7, 8]]`, so `s[0]` is `6` and `s[1:]` is `[[7, 8]]` (a one-element list containing the list `[7, 8]`). Result: `[6, [[7, 8]]]`.

```python
>>> silver
[1, [3]]
>>> gold
[2, [4, 5]]
>>> platinum
[6, [[7, 8]]]
```

The two lessons: slicing creates a new list (a shallow copy, so the inner elements are shared), and `pop` mutates the list it is called on while the surrounding expression continues evaluating left to right.

---

## Common Pitfalls

1. **Expecting the body to run when you call the function.** `t = plus_minus(3)` executes nothing. A `print` at the top of a generator function will not fire until the first `next`. Likewise, an error on the first line of the body surfaces at the first `next`, not at the call.
2. **`yield gen` instead of `yield from gen`.** This yields the generator object as a single element. The symptom is seeing `<generator object ...>` in your output where you expected a number or string.
3. **Thinking a function can be "part generator".** A single `yield` anywhere in the body makes *every* call return a generator, even if that `yield` is in a branch that never runs.
4. **Using `return <value>` to produce output in a generator function.** In a generator, a bare `return` just stops iteration (triggers `StopIteration`); it does not add a value to the sequence. Use `yield`.
5. **Forgetting that generators are single-use.** Once exhausted, a generator stays exhausted. You cannot restart it, and `list(t)` after you have already consumed `t` gives `[]`. To iterate twice, call the generator function twice.
6. **Calling `list()` on an infinite generator.** `list(cycle([3, 4, 5]))` or `list(yield_ints(100))` hangs forever. Pull a bounded number of items instead.
7. **Mixing up "items consumed" with "items printed".** In the `print_cycle` family, `range(n)` controls how many values are pulled from the generator; the `if` controls how many are printed. These are different numbers.
8. **Missing the "do nothing" base case in recursive generators.** In `park`, the `elif n > 0` is load-bearing: without a guard, `park(-1)` would recurse forever or error. A generator that yields nothing is the natural representation of "no ways to do this".
9. **Yielding `''` versus yielding nothing at the base case.** `park(0)` must `yield ''` (one way to park zero spots). Yielding nothing there would make the whole generator produce zero results.
10. **Ordering surprises in recursive generators.** The order of your `yield` and `yield from` statements fully determines output order (see `prefixes`, and the `park` ordering note above). If a doctest is order-sensitive, trace the first few outputs by hand.
11. **Assuming the list version and the generator version have the same performance.** They do the same total work only if you consume everything. If you need a few results out of many, the list version pays for all of them up front.
12. **`x % 2 == 1` with negative numbers.** In Python, `-1 % 2` is `1`, so negative odd integers do satisfy `x % 2 == 1`. (In some other languages this would be `-1` and the test would fail.)

---

## Likely Exam Points

### 1. Trace a generator by hand

**Q.** What does the following print?

```python
def g():
    print('A')
    yield 1
    print('B')
    yield 2
    print('C')

t = g()
print('start')
print(next(t))
print(next(t))
```

**A.**
```
start
A
1
B
2
```
`t = g()` prints nothing. The first `next` runs until the first `yield`, printing `A` then returning 1. The second resumes after that yield, printing `B` then returning 2. `C` would only print on a third `next`, which would also raise `StopIteration`.

### 2. `yield` versus `yield from`

**Q.** Give `list(f(2))` for each:

```python
def a(k):
    if k > 0:
        yield k
        yield a(k - 1)

def b(k):
    if k > 0:
        yield k
        yield from b(k - 1)
```

**A.** `list(a(2))` is `[2, <generator object a ...>]`: only two elements, the second being a generator. `list(b(2))` is `[2, 1]`. `yield from` splices in all elements of the sub-generator; `yield` contributes exactly one element.

### 3. Fill in the blanks for an interleaving infinite generator

**Q.** Complete `yield_ints(start)` so that `[next(t) for _ in range(5)]` on `yield_ints(100)` gives `[100, 101, 99, 102, 98]`.

**A.**
```python
def yield_ints(start):
    yield start
    offset = 1
    while True:
        yield start + offset
        yield start - offset
        offset += 1
```
`yield start` before the loop emits the center exactly once; each loop pass emits the `+offset` then `-offset` pair and increments.

### 4. Write `cycle` and use it

**Q.** Implement `cycle(s)` so that `get_n(cycle([3, 4, 5]), 7)` is `[3, 4, 5, 3, 4, 5, 3]`, then implement `print_cycle(n, s)` that pulls the first `n` cycle items and prints only the odd ones.

**A.**
```python
def cycle(s):
    while True:
        for x in s:
            yield x

def print_cycle(n, s):
    t = cycle(s)
    for _ in range(n):
        x = next(t)
        if x % 2 == 1:
            print(x)
```
The `while True` wrapping a `for` is the whole trick; the generator's suspended frame remembers its position inside both loops.

### 5. Recursive enumeration generator (the `park` pattern)

**Q.** Implement `park(n)` yielding all strings of length `n` built from `%` (1 spot), `.` (1 spot), and `<>` (2 spots). What are the base cases and why?

**A.**
```python
def park(n):
    if n == 0:
        yield ''
    elif n > 0:
        for s in park(n - 1):
            yield '%' + s
            yield '.' + s
        for s in park(n - 2):
            yield '<>' + s
```
`n == 0` yields the empty string because there is exactly one way to fill zero spots, and that empty string is what the recursive prefixes get attached to. `n < 0` falls through both branches and yields nothing, representing "a car did not fit here", which is why `park(1)` correctly produces only `'%'` and `'.'`.

### 6. Convert a list-returning recursive function into a generator function

**Q.** Rewrite this as a generator function.

```python
def subsets(s):
    if not s:
        return [[]]
    rest = subsets(s[1:])
    return [[s[0]] + t for t in rest] + rest
```

**A.**
```python
def subsets(s):
    if not s:
        yield []
    else:
        for t in subsets(s[1:]):
            yield [s[0]] + t
        yield from subsets(s[1:])
```
The mechanical recipe from lecture: `return [x]` becomes `yield x`; a list comprehension over a recursive call becomes a `for` loop with a `yield` inside; concatenating a recursive result becomes `yield from`. *(extra context: this subsets example is not from the lecture; it is the same transformation applied to a standard problem. Note that calling `subsets(s[1:])` twice here costs extra work, just as in the list version.)*

### 7. Laziness and efficiency

**Q.** `partitions(60, 50)` has about 966,000 results. Explain why `next(partitions(60, 50))` is fast but `list_partitions(60, 50)[0]` is slow.

**A.** The generator runs only until it reaches its first `yield`, so only the small fraction of the recursion needed to produce one partition executes; the rest of the frame stays suspended and is never resumed. The list version must complete the entire recursion, allocating and concatenating all intermediate lists, before it can return anything, so indexing `[0]` cannot help.

### 8. Generators are exhaustible, iterators are shared state

**Q.** What is printed?

```python
t = (x for x in [1, 2, 3])   # a generator expression
print(list(t))
print(list(t))
```

**A.**
```
[1, 2, 3]
[]
```
The first `list(t)` consumes the generator to exhaustion. The second finds it already exhausted and builds an empty list. *(extra context: generator expressions were not the focus of this lecture, but the exhaustion point applies identically to generators from generator functions.)*

### 9. List mutation / box-and-pointer

**Q.** After the four statements of the `chain` example, what are `silver`, `gold`, and `platinum`?

**A.** `silver` is `[1, [3]]`, `gold` is `[2, [4, 5]]`, `platinum` is `[6, [[7, 8]]]`. The three traps are: `.pop()` mutates `silver[1]` as a side effect of building `gold`; `silver[0]` is evaluated (to 2) before the `pop` and before the later `silver[0] = 1`, and integers are immutable so `gold[0]` stays 2; and `s[1:]` on `[6, [7, 8]]` is the one-element list `[[7, 8]]`.

---

## Summary

- `iter(iterable)` makes an iterator; `next(iterator)` advances it and returns the next value; exhaustion raises `StopIteration`. `get_n(t, n)` is `[next(t) for _ in range(n)]`.
- A **generator function** is any function whose body contains `yield`. Calling it runs **no** code and returns a **generator**, which is an iterator.
- Each `next` runs the body until the next `yield`, returns that value, and **pauses**, preserving all local variables and the exact position inside every loop. The following `next` resumes right after that `yield`.
- A normal function returns once; a generator function can yield many times. Falling off the end of the body ends the iteration.
- Infinite generators (`while True`) are legitimate because of laziness: `cycle(s)` repeats a sequence forever, `yield_ints(start)` interleaves outward from a center.
- `yield from <iterable>` is shorthand for `for x in <iterable>: yield x`. Use it for recursive generators (`countdown`, `prefixes`, `substrings`, `partitions`). Writing plain `yield gen` instead yields the generator object as a single element, which is almost always a bug.
- Recursive generators replace list-building: `return [x]` becomes `yield x`, list comprehensions become `for` loops with `yield` inside, and list concatenation becomes `yield from`. Base cases that returned `[]` often collapse into a guard that simply yields nothing.
- `park(n)`: `yield ''` at `n == 0`, and for `n > 0` prepend `'%'` and `'.'` to `park(n-1)` and `'<>'` to `park(n-2)`; negative `n` yields nothing. Counts: 1, 2, 5, 12, 29.
- Laziness is a real performance property: the generator version of `partitions(60, 50)` hands you the first 10 of roughly a million results immediately, while the list version must build them all first.
- Decide deliberately whether filtering happens inside the generator (`check_sequence` skips multiples of 3, so `n` counts printed terms) or in the consumer (`print_cycle` filters after `next`, so `n` counts consumed terms).
- Side material: slicing copies a list (shallowly), `.pop()` mutates in place, operands evaluate left to right, and rebinding a slot holding an immutable value does not affect other lists that hold the same value.
