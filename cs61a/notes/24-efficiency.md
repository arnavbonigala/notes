<!-- Wed, Oct 21, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 24: Efficiency

## Overview

This lecture shifts attention from *what* a program computes to *how much work* it takes to compute it. The running example is the tree-recursive `fib`, which recomputes the same subproblems over and over: computing `fib(30)` requires about 2.69 million calls to `fib` even though the answer is only 832,040. We measure this cost first by counting function calls (using a counting decorator), then we fix it with **memoization**, a decorator that caches previously computed return values in a dictionary so each distinct argument is computed exactly once. The second half of the lecture compares two implementations of exponentiation to show that the interesting question is not "how many microseconds did this take?" but "how does the cost *grow* as the input grows": the naive `exp` grows proportionally to `n`, while the repeated-squaring version grows far more slowly. We then look at concrete constant-factor-and-worse differences among Python list operations (append, assignment, and comprehensions are fast; insert-at-front, slicing, and `+` are slow), and close with generator practice: `powers`, `prefix`, `map_pairs`, and generators that walk linked lists and trees with `yield from`.

---

## Key Concepts

### 1. Counting calls as a proxy for time

Running time depends on your machine, the weather, and what else your laptop is doing. A cleaner first measurement is **how many times a function gets called**, because each call does a roughly fixed amount of work (open a frame, evaluate a few expressions, return). If you can describe how the number of calls grows with the input, you have described how the time grows.

To count calls without editing `fib`, the lecture uses a wrapper (a decorator-style higher-order function) that returns a new function with the same behavior plus a side effect of incrementing a counter.

```python
class CallCounter:
    def __init__(self):
        self.n = 0

    def count(self, f):
        def counted(n):
            self.n += 1
            return f(n)
        return counted
```

`count` takes a one-argument function `f` and returns `counted`. Calling `counted(5)` bumps `self.n` and then returns `f(5)`, so from the caller's point of view nothing changed except that a tally is being kept. In the live demo the instructor wrote the equivalent thing as a plain decorator `count` that stored a `call_count` attribute on the returned function; the class version in `24.py` just keeps the tally on an object instead of on the function.

### 2. Tree recursion repeats work, explosively

```python
def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n-2) + fib(n-1)
```

The call tree for `fib(5)` branches twice at every non-base node. The slide diagram shows `fib(5)` calling `fib(3)` and `fib(4)`; `fib(3)` calls `fib(1)` and `fib(2)`; `fib(4)` calls `fib(2)` and `fib(3)`, and that second `fib(3)` redoes the *entire* subtree that was already computed on the left. The repetition compounds at every level.

Measured counts from the demo:

| call | result | calls to `fib` |
|---|---|---|
| `fib(5)` | 5 | 15 |
| `fib(30)` | 832,040 | ~2,690,000 |

Two observations worth internalizing:

- The number of calls is roughly *proportional to the answer itself*, and Fibonacci numbers grow exponentially. So the cost grows exponentially in `n`.
- Most of that work is garbage: the vast majority of those 2.69 million calls return 0 or 1.

*(extra context)* Exactly, the number of calls is `2 * fib(n+1) - 1`. Check: `fib(6) = 8`, and `2*8 - 1 = 15`, matching the measured 15 calls for `fib(5)`. For `n = 30`: `2 * 1,346,269 - 1 = 2,692,537`.

Also note the counter is **cumulative**: calling `fib(5)` twice in a session leaves the count at 30, not 15.

### 3. Memoization: remember what you already computed

> Idea: Remember the results that have been computed before.

```python
def memo(f):
    cache = {}
    def memoized(n):
        if n not in cache:
            cache[n] = f(n)
        return cache[n]
    return memoized
```

Environment reasoning in words: `memo(f)` creates a frame in which `cache` is bound to a fresh empty dictionary and `f` is bound to the original function. The function `memoized` is defined in that frame, so its **parent frame is that call to `memo`**. When `memoized` later looks up `cache`, the name is not local, so Python follows the parent pointer and finds the *same* dictionary every time. That dictionary therefore survives between calls to `memoized` and accumulates results. Each `memo(f)` call makes its own private cache; two separately memoized functions do not share one.

Keys are arguments, values are return values.

**Why rebinding the global name matters.** The crucial move in the demo is:

```python
fib = memo(fib)
```

The body of the original `fib` contains the names `fib(n-2)` and `fib(n-1)`, which are looked up in the **global frame at call time**. After the assignment, the global name `fib` refers to `memoized`, so the recursive calls go through the cache. If you instead wrote `fast_fib = memo(fib)`, only the outermost call would be cached and the entire exponential tree underneath would still run.

**Memoized call counts for `fib(30)`** (from the demo, which stacked both wrappers):

```python
counter = CallCounter()
fib = counter.count(fib)              # counts calls that reach the original fib body
memo_counter = CallCounter()
fib = memo_counter.count(memo(fib))   # counts calls to the memoized layer
fib(30)
memo_counter.n   # 59
counter.n        # 31
```

- **31** calls reach the original `fib` body, one for each distinct argument `0, 1, 2, ..., 30`. This is the point: each value is computed once.
- **59** calls reach the memoized wrapper. Every one of the 29 computed values with `n >= 2` makes two recursive calls (`2 * 29 = 58`), plus the one call from the user, giving 59. The extra 28 calls beyond 31 are **cache hits**.

The slide's annotated tree uses three labels for nodes: **call to fib** (a real computation), **found in cache** (the wrapper returned immediately), and **skipped** (the node never happened at all, because its parent was a cache hit). Tracing `fib(5)`: `fib(5) -> fib(3) -> fib(1)` is a real call returning 1; `fib(0)` is a real call; the second `fib(1)` is a cache hit; `fib(2)` and `fib(3)` return. Then `fib(4)` calls `fib(2)`, which is already cached, so the `fib(0)` and `fib(1)` beneath it are **skipped entirely**. Likewise `fib(4)`'s call to `fib(3)` is a cache hit and its whole subtree disappears. Note that `fib(4)` and `fib(5)` are each called only once, since nothing asks for them a second time.

With memoization, `fib(300)` is nearly instant.

### 4. Memoization only preserves behavior for pure functions

A **pure function**'s return value depends only on its arguments and it has no side effects. `memoized` skips calling `f` on a cache hit, so:

- If `f` printed something, the print happens only on the first call per argument.
- If `f` depended on mutable state, time, or randomness, the cached answer can be stale or wrong.

The slide says it directly: "Same behavior as f, if f is a pure function." Also implicit: arguments must be **hashable** to serve as dictionary keys, and this particular `memo` only works for single-argument functions.

### 5. Measuring growth, not single timings

The exponentiation demo compared two implementations of `b**n`, reconstructed from the lecture demo (the code was shown live, not in the slides or `24.py`):

```python
def exp(b, n):
    """Return b ** n, the slow way."""
    if n == 0:
        return 1
    return b * exp(b, n-1)

def exp_fast(b, n):
    """Return b ** n, using repeated squaring."""
    if n == 0:
        return 1
    elif n % 2 == 0:
        return square(exp_fast(b, n // 2))
    else:
        return b * exp_fast(b, n-1)

def square(x):
    return x * x
```

Both rest on a recursive definition of exponentiation. `exp` uses `b**n = b * b**(n-1)`. `exp_fast` adds the observation that for even `n`, `b**n = (b**(n//2))**2`, which **halves the exponent in one step**.

The measurement methodology from the demo:

- `timeit` from the standard library times calling a zero-argument function a given number of times. Timing one call is noise; timing 100 calls and dividing by 100 gives a usable average (about 100 microseconds per call for the slow version at the sizes shown).
- Repeat the 100-call batch 15 times and **take the minimum**, not the mean. The reasoning: other processes on the computer can only make a measurement slower, so the fastest observed time is the best estimate of the true cost.
- Convert to microseconds and plot time against input size `n` for `n = 100, 200, ..., 800` or similar.

Results:

- `exp`: the time plot is roughly a straight line (about 20 microseconds at `n = 400`, about 50 at `n = 800`). Counting recursive calls instead of timing gives a *perfectly* straight line: `n = 400` takes exactly 400 calls. Cost is **proportional to `n`**.
- `exp_fast`: dramatically smaller numbers (about 2.5 microseconds where the slow version took 50) and the curve **bends downward**, growing much more slowly than `n`. At `n = 800` it makes only about 15 recursive calls.
- The `exp_fast` plot is **jagged**, and the jaggedness is real rather than timing noise, because counting calls shows the same pattern. The reason: sometimes a step halves `n` (cheap progress) and sometimes `n` is odd so the step only subtracts 1 (then halves). Exponents that halve cleanly all the way down are the deep dips in the graph.
- The jaggedness does not matter. **The shape of the curve is what matters.**

The lecture's framing of the conclusion: computer scientists measure efficiency "not by calculating exactly how long it takes or how many recursive calls are made, but by evaluating how these metrics change as the input data grows." Does it **scale well** (work barely grows as input grows) or **scale poorly** (work grows a lot)? The practical recipe: pick a range of input sizes realistic for your program, measure time or call counts across that range, and compare implementations by growth.

*(extra context)* The formal vocabulary for this, big-Theta / big-O notation, is the subject of the following lectures. In that language `exp` is linear, Θ(n), and `exp_fast` is logarithmic, Θ(log n). This lecture deliberately stays at the level of plots and call counts.

### 6. List efficiency

Different ways of building the same list differ enormously, and here the difference comes from how Python represents lists in memory (a contiguous block of element references).

**Fast: appending, assigning to an existing index, and list comprehensions.** Appending writes into space at the end of the list; assigning `s[k] = ...` overwrites one slot; a comprehension builds the result once.

**Slow: inserting at the beginning or middle, slicing, and adding lists.**

- `s.insert(0, x)` has to **shift every existing element** one position over to make room at the front. Doing that `n` times means work proportional to `n` on each of `n` iterations.
- `s = s + [x]` **builds a brand new list** containing copies of all `n` references plus the new one, then rebinds `s`. The old list is thrown away. Again, quadratic total work, and more allocation than `insert`.
- Slicing likewise copies the slice into a new list.

The slide's benchmark, building a list of the first `n` perfect squares:

```python
def using_list_comprehension(n):
    return [k * k for k in range(n)]

def using_append(n):
    s = []
    for k in range(n):
        s.append(k * k)
    return s

def using_assign(n):
    s = [0 for k in range(n)]
    for k in range(n):
        s[k] = k * k
    return s

def using_insert(n):
    s = []
    for k in range(n):
        s.insert(0, k * k)
    return s

def using_add(n):
    s = []
    for k in range(n):
        s = s + [k * k]
    return s
```

With `n = 100,000`:

| implementation | time |
|---|---|
| `using_list_comprehension` | 1.58 ms |
| `using_append` | 1.76 ms |
| `using_assign` | 2.58 ms |
| `using_insert` | 1,470 ms |
| `using_add` | 9,210 ms |

The first three are within a factor of 2 of each other. `using_insert` is roughly 1,000 times slower and `using_add` nearly 6,000 times slower. Note also that `using_insert` and `using_add` produce lists in opposite orders from each other (`insert(0, ...)` reverses), so they are not even interchangeable.

The takeaway is not to memorize numbers but to recognize the pattern: **an operation whose cost grows with the current list length, performed inside a loop over the list, gives you quadratic behavior.**

### 7. Generators, revisited with practice

A **generator function** is any function whose body contains `yield`. Calling it does not run the body; it returns a **generator**, which is an iterator. The body runs only as `next` is called on that generator, pausing at each `yield` and resuming from exactly where it left off, with its local environment intact.

Consequences used in this lecture:

- A generator can be **infinite**, because nothing is computed until requested. `powers(2)` yields powers of 2 forever, and `while True:` is perfectly safe in a generator.
- You consume a generator by calling `next(t)` explicitly, by iterating with `for x in t`, or by passing it to a function like `list(...)` or `sum(...)`.
- Iterating partway through a generator and then continuing to iterate **resumes** where you stopped; the consumed items are gone. This is what makes `map_pairs` work.
- `yield from <iterable>` yields every element of that iterable in order. It is the natural tool for recursive generators over trees.

---

## Definitions

- **Efficiency**: how much of a resource (time, calls, memory) a computation consumes, and more usefully, how that consumption grows as the size of the input grows.
- **Scales well / scales poorly**: an implementation scales well if the work required grows slowly as the input grows, and poorly if the work grows steeply.
- **Call count**: the number of times a function is invoked during a computation, used as a machine-independent proxy for running time.
- **Tree recursion**: a recursive pattern in which a function makes more than one recursive call per invocation, producing a tree-shaped computation. `fib` is the canonical example.
- **Memoization**: a technique for speeding up a computation by storing the results of previous calls in a cache and returning the stored result instead of recomputing it.
- **Cache**: in `memo`, a dictionary whose keys are the arguments previously passed and whose values are the corresponding return values.
- **Cache hit**: a call whose argument is already a key in the cache, so the underlying function is not called.
- **Pure function**: a function whose return value depends only on its arguments and which has no side effects. Memoizing a pure function preserves its behavior; memoizing an impure function may change it.
- **Decorator / higher-order wrapper**: a function that takes a function and returns a new function with added behavior (counting, caching) but the same calling interface. `memo` and `CallCounter.count` are both of this form.
- **`timeit`**: standard library module whose `timeit` function reports the time taken to call a zero-argument function a specified number of times.
- **Repeated squaring**: computing `b**n` by using `b**n == (b**(n//2))**2` when `n` is even, which halves the exponent in a single step.
- **Generator function**: a function containing `yield` whose invocation returns a generator rather than running its body.
- **Generator**: an iterator produced by calling a generator function; running its body lazily, one `yield` at a time.
- **Iterator**: an object that produces a sequence of values one at a time in response to `next`, and raises `StopIteration` when exhausted.
- **`yield from`**: a statement that yields every element of an iterable (often a recursive generator call) from within a generator function.

---

## Worked Examples

### Example 1: counting calls to `fib`

```python
def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n-2) + fib(n-1)

counter = CallCounter()
fib = counter.count(fib)
```

Step by step:

1. `counter.count(fib)` is called. Inside that call, `self` is bound to the `CallCounter` instance and `f` is bound to the original `fib` function. A function `counted` is created whose parent frame is this `count` call, so `counted` can see both `self` and `f` forever after.
2. `fib = ...` rebinds the **global** name `fib` to `counted`. The original function object is now reachable only through `f` in the closure.
3. Now call `fib(5)`. This enters `counted` with `n = 5`, increments `self.n` to 1, and calls `f(5)`, which is the original body.
4. The original body evaluates `fib(3) + fib(4)`. The name `fib` is looked up at call time and found in the global frame, where it is now `counted`. So **every recursive call is also counted**. This is why wrapping works without touching the recursive calls.
5. `counter.n` ends at 15.
6. `fib(30)` leaves `counter.n` at about 2.69 million while returning 832,040.

### Example 2: memoizing `fib`

```python
def memo(f):
    cache = {}
    def memoized(n):
        if n not in cache:
            cache[n] = f(n)
        return cache[n]
    return memoized

fib = memo(fib)
fib(300)   # effectively instant
```

Trace of the memoized version on a small input, `fib(5)`:

1. `fib(5)`: 5 is not in the cache, so call the original body, which needs `fib(3)` and `fib(4)`.
2. `fib(3)`: not cached, call the body, which needs `fib(1)` and `fib(2)`.
3. `fib(1)`: not cached, the original body hits the base case and returns 1. `cache[1] = 1`.
4. `fib(2)`: not cached, needs `fib(0)` and `fib(1)`. `fib(0)` is a real call returning 0, so `cache[0] = 0`. `fib(1)` is now a **cache hit**: the wrapper returns 1 without entering the original body. `cache[2] = 1`.
5. `fib(3)` returns `1 + 1 = 2`, and `cache[3] = 2`.
6. Back in `fib(5)`, now `fib(4)`: not cached, needs `fib(2)` and `fib(3)`. **Both are cache hits.** The entire subtree underneath them, including two more calls to `fib(0)` and `fib(1)`, is **skipped and never executed**. `cache[4] = 3`.
7. `fib(5)` returns `2 + 3 = 5`.

Important subtlety: the cache lives in the frame created by the single call `memo(fib)`. Every recursive call shares it, which is exactly what makes the method work. If `memo` created the dictionary *inside* `memoized`, it would be a fresh empty dictionary on every call and would never help.

Why `n not in cache` rather than `cache.get(n)` with a truthiness test: `fib(0)` returns `0`, which is falsy. Testing membership is correct; testing truth would recompute `fib(0)` every time.

### Example 3: `exp` versus `exp_fast`

Reconstructed from the lecture demo (see Key Concepts 5 for the code).

Tracing `exp(2, 10)`: `2 * exp(2, 9)`, then `2 * exp(2, 8)`, down to `exp(2, 0) = 1`, then multiplying back up to 1024. That is 11 calls, and in general `n + 1` calls. Measuring confirms a straight line: input 400 gives 400 recursive calls.

Tracing `exp_fast(2, 10)`:

1. `n = 10` is even, so `square(exp_fast(2, 5))`.
2. `n = 5` is odd, so `2 * exp_fast(2, 4)`.
3. `n = 4` is even, so `square(exp_fast(2, 2))`.
4. `n = 2` is even, so `square(exp_fast(2, 1))`.
5. `n = 1` is odd, so `2 * exp_fast(2, 0)`.
6. `n = 0`, return 1.
7. Returning back up: 2, then 4, then 16, then 32, then 1024.

Six calls instead of eleven, and the gap widens dramatically: at `n = 800`, `exp` makes 800 calls while `exp_fast` makes about 15. Notice in the trace that steps 2 and 5 were "wasted" odd steps that only subtracted 1. That is the source of the jagged graph: exponents like 1024 that halve all the way down are the fastest, while exponents that keep landing on odd numbers cost roughly twice as many steps. The jaggedness is a constant factor and does not change the shape of the growth.

### Example 4: list building (why `using_add` is catastrophic)

```python
def using_add(n):
    s = []
    for k in range(n):
        s = s + [k * k]
    return s
```

Pointer reasoning in words: on iteration `k`, `s` names a list of length `k`. The expression `s + [k*k]` allocates a **new** list of length `k + 1` and copies all `k` element references into it, then copies the new element. The name `s` is rebound to this new list, and the old one becomes garbage. So iteration `k` costs about `k` units of copying, and the total over all iterations is about `0 + 1 + 2 + ... + (n-1)`, which is roughly `n**2 / 2`. For `n = 100,000` that is about 5 billion copies, hence 9.21 seconds.

```python
def using_insert(n):
    s = []
    for k in range(n):
        s.insert(0, k * k)
    return s
```

Here no new list object is created, but every existing element must shift one slot to the right to open position 0. Same quadratic total work, with a smaller constant (no allocation), which matches 1,470 ms versus 9,210 ms.

```python
def using_append(n):
    s = []
    for k in range(n):
        s.append(k * k)
    return s
```

`append` writes to the end, where Python keeps slack capacity, so each iteration costs a constant amount on average and the total is proportional to `n`. 1.76 ms.

### Example 5: `powers` and `prefix`

```python
def powers(b):
    """Yield the powers of b."""
    n = 0
    while True:
        yield pow(b, n)
        n += 1
```

The blanks on the slide were `while True:` and `yield pow(b, n)`. The `while True` loop never terminates, which is fine: `powers(2)` returns a generator and runs nothing. Each `next` resumes at the `yield`, executes `n += 1`, loops, and yields `pow(b, n)` for the new `n`. The generator's local `n` persists between resumptions.

```python
def prefix(t, n):
    """Yield the first n items of an infinite iterator t.

    >>> list(prefix(powers(2), 10))
    [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]
    """
    for i in range(n):
        yield next(t)
```

The slide blank was `yield next(t)`. `prefix` is itself a generator function, so it is lazy too: `list(...)` is what actually drives it. Each iteration pulls one item out of `t` with `next` and immediately hands it to `prefix`'s own consumer. The `24.py` version uses a `while n > 0: yield next(t); n -= 1` loop instead of `for i in range(n)`, which behaves identically.

Why this is the right tool: you cannot write `list(powers(2))`, since that would loop forever. `prefix` lets you take a finite window out of an infinite stream.

### Example 6: `map_pairs`, a higher-order generator function

```python
from operator import add

def map_pairs(f, t):
    """Yield the result of calling f on each pair of consecutive items in t.
    (You can assume t has at least two items)

    >>> list(map_pairs(add, iter([3, 4, 5, 6, 7])))
    [7, 9, 11, 13]
    >>> list(prefix(map_pairs(lambda x, y: (x, y), powers(2)), 5))
    [(1, 2), (2, 4), (4, 8), (8, 16), (16, 32)]
    """
    previous = next(t)
    for x in t:
        yield f(previous, x)
        previous = x
```

Step by step on `map_pairs(add, iter([3, 4, 5, 6, 7]))`:

1. `previous = next(t)` consumes the first item, 3. This is the key idea: you grab one item up front so that the loop always has a left-hand partner available.
2. The `for x in t` loop **continues from where `next` left off**, so `x` takes the values 4, 5, 6, 7. It does not restart at 3. This only works because `t` is an iterator, not a list.
3. First iteration: `x = 4`, yield `add(3, 4) = 7`, then set `previous = 4`.
4. Then yield `add(4, 5) = 9`, `add(5, 6) = 11`, `add(6, 7) = 13`. Four results from five inputs, as expected for consecutive pairs.

The second doctest shows why laziness matters: `powers(2)` is infinite, `map_pairs` over it is infinite, and `prefix(..., 5)` takes just five pairs. Nothing beyond `32` is ever computed. Note also that `f` can return anything, here a tuple built by a lambda, so `map_pairs` is a genuinely higher-order generator function.

Also note the second doctest passes `powers(2)` directly: since generators are already iterators, no `iter` call is needed, whereas the list in the first doctest must be wrapped with `iter` so that `next` works on it.

### Example 7: a generator over a linked list

```python
type LinkedList[T] = Link[T] | tuple[()]

@dataclass
class Link[T]:
    first: T
    rest: LinkedList[T] = ()   # rest defaults to empty
```

```python
def items(s: LinkedList):      # named link_gen in 24.py
    """Yield the items in a linked list.

    >>> nums = Link(3, Link(5, Link(7)))
    >>> sum(items(nums))
    15
    """
    while isinstance(s, Link):
        yield s.first
        s = s.rest
```

The slide blanks were `while isinstance(s, Link)` and `s = s.rest`.

Reasoning in words: `nums` points to a `Link` whose `first` is 3 and whose `rest` points to a second `Link` (5), whose `rest` points to a third (7), whose `rest` is the empty tuple `()`. The loop test `isinstance(s, Link)` is the standard way to ask "have I run off the end?", since the end is represented by `()`, not by a `Link`. Each iteration yields the current `first` and then rebinds the generator's local `s` to walk one link forward. Rebinding `s` **does not mutate the list**; it just moves the local pointer. Because `s` is a local of the generator frame, its value is preserved while the generator is suspended at the `yield`.

`sum(items(nums))` works because `sum` accepts any iterable, and `items(nums)` is a generator. No intermediate list is built.

### Example 8: a recursive generator over a tree

```python
@dataclass
class Tree[T]:
    label: T
    branches: list[Tree[T]] = ...   # branches defaults to []
```

```python
def labels(t: Tree):
    """Yield the labels of a tree.

    >>> list(labels(Tree(1, [Tree(2, [Tree(3), Tree(4)]), Tree(5)])))
    [1, 2, 3, 4, 5]
    """
    yield t.label
    for b in t.branches:
        yield from labels(b)
```

The slide blanks were `yield t.label` and `yield from labels(b)`.

Step by step on the doctest tree (root 1, with branches [2 [3, 4], 5]):

1. `labels(root)` yields 1 first, so this is a **preorder** traversal: label before branches.
2. The loop reaches branch `Tree(2, [Tree(3), Tree(4)])`. `yield from labels(b)` creates a sub-generator and re-yields everything it produces: 2, then 3, then 4. Each of those comes from its own nested generator frame, suspended and resumed in turn.
3. The loop reaches `Tree(5)`, a leaf. `labels(Tree(5))` yields 5, then its `for` loop over the empty `branches` list does nothing, so the sub-generator is exhausted.
4. The outer loop finishes, the outer generator is exhausted, `list` stops, result `[1, 2, 3, 4, 5]`.

Why `yield from` and not `yield`: `yield labels(b)` would yield **generator objects**, giving something like `[1, <generator>, <generator>]`. `yield from` flattens. The equivalent longhand is `for x in labels(b): yield x`.

Note that leaves need no special case: a tree with no branches simply skips the loop, so the base case is handled implicitly.

### Example 9: `partition` returning linked lists

This appears in `24.py` without being discussed in the recorded portion, but it ties the two halves of the lecture together.

```python
def partition(n, m) -> list[LinkedList[int]]:
    """Partitions of n using parts up to size m."""
    if n == 0:
        return [()]  # partition 0 with an empty linked list
    elif n < 0 or m == 0:
        return []    # no way to partition these
    with_m = [Link(m, s) for s in partition(n-m, m)]
    without = partition(n, m-1)
    return with_m + without
```

How it works: the classic two-case recursion on partitions. Either use at least one part of size `m`, which leaves `n - m` to partition with parts still up to `m`, or use no part of size `m`, leaving `n` to partition with parts up to `m - 1`. Rather than counting partitions, this version **returns the partitions themselves**, each as a linked list of part sizes.

- Base case `n == 0`: there is exactly one partition of 0, the empty one, so return a list containing one empty linked list, `[()]`. Returning `[]` here would be wrong: that means "no partitions exist", not "one empty partition exists".
- Base case `n < 0 or m == 0`: no partitions, return the empty list `[]`.
- `with_m` uses a list comprehension to prepend `m` onto each partition of `n - m`, building `Link(m, s)` so that the parts come out in nonincreasing order.
- The results are combined with `with_m + without`.

Efficiency connection: `with_m + without` creates a new list, which is one of the slow operations from the list efficiency slide. Here it is acceptable because it happens once per recursive call rather than once per element inside a loop. Also notice that `partition` is tree-recursive and recomputes subproblems, so it is a natural candidate for memoization, except that this `memo` takes only one argument and the return values are mutable lists.

---

## Common Pitfalls

1. **Naming the memoized function something new.** `fast_fib = memo(fib)` memoizes only the outer call. The recursive calls inside `fib`'s body look up the global `fib`, which is still the slow original. You must rebind the name that the recursive calls use, or use the `@memo` decorator form.

2. **Putting the cache inside the inner function.** If `cache = {}` appears inside `memoized`, a fresh empty dictionary is created per call and nothing is ever remembered. The cache must live in the enclosing `memo` frame.

3. **Assuming memoization is always safe.** It changes behavior for impure functions: side effects happen only on the first call per argument, and results can be stale. Arguments must also be hashable.

4. **Thinking memoization helps every slow function.** It helps when the same subproblems recur. It does nothing for `exp`, where each recursive call has a different argument and nothing repeats. The fix there was a better algorithm (repeated squaring), not caching.

5. **Forgetting that call counters are cumulative.** Calling `fib(5)` twice gives a count of 30. Reset or use a fresh counter between measurements.

6. **Reading a single timing as meaningful.** Timing one call is dominated by noise. Time many calls, repeat, and take the **minimum** of the repeats (competing processes can only inflate a measurement).

7. **Confusing "faster at this input" with "scales better."** The lecture's point is that the curve's shape matters. The jagged dips in the `exp_fast` plot are real but irrelevant; the downward bend is what counts.

8. **Using `s = s + [x]` or `s.insert(0, x)` inside a loop.** This turns a linear job into a quadratic one. Use `append` and reverse at the end if order matters, or use a list comprehension.

9. **Writing `yield labels(b)` instead of `yield from labels(b)`.** The former yields generator objects.

10. **Trying to iterate an already-partly-consumed iterator from the start.** In `map_pairs`, after `previous = next(t)` the `for` loop resumes at the *second* item. This is intentional. Conversely, if you passed a *list* instead of an iterator, `next(t)` would raise a `TypeError`, which is why the doctest wraps the list in `iter`.

11. **Expecting a generator function to run when called.** `powers(2)` returns a generator immediately and executes none of the body, which is why an infinite `while True` is harmless. Calling `list` on an infinite generator, however, hangs.

12. **Using truthiness instead of membership for a cache check.** `fib(0)` returns `0`; `if not cache.get(n)` would recompute it forever.

---

## Likely Exam Points

**1. Count the calls in a tree-recursive function.**

*Q:* For the lecture's `fib`, how many total calls to `fib` are made by evaluating `fib(4)`? Show the reasoning.

*A:* Let `C(n)` be the number of calls. `C(0) = C(1) = 1` (base cases, one call each). Otherwise `C(n) = 1 + C(n-2) + C(n-1)`. So `C(2) = 1 + 1 + 1 = 3`, `C(3) = 1 + 1 + 3 = 5`, `C(4) = 1 + 3 + 5 = 9`. Nine calls. *(extra context: this equals `2 * fib(5) - 1 = 2 * 5 - 1 = 9`.)*

**2. Count calls under memoization.**

*Q:* Starting from an empty cache, `fib = memo(fib)` and then `fib(6)` is evaluated. How many times does the body of the original `fib` run, and how many times is the memoized wrapper called?

*A:* The original body runs once per distinct argument, `0` through `6`, so **7 times**. For the wrapper: each computed value with `n >= 2` makes 2 recursive calls through the wrapper, and there are 5 such values (2, 3, 4, 5, 6), giving 10, plus the 1 initial call from the user, so **11 calls** to the wrapper. (Of those 11, 11 - 7 = 4 are cache hits.)

**3. Fill in `memo` or explain why the closure works.**

*Q:* Why must `cache = {}` be written in `memo`'s body rather than inside `memoized`? Answer using frames.

*A:* Each call to `memo` creates one frame containing one dictionary, and `memoized` is defined in that frame, so `memoized`'s parent frame is that `memo` call. Every invocation of `memoized` therefore resolves the non-local name `cache` to that single dictionary, which persists across calls and accumulates results. If `cache = {}` were inside `memoized`, it would be a local rebound to a fresh empty dictionary on every call, so no result would ever be reused.

**4. Identify when memoization preserves behavior.**

*Q:* Give a function for which `memo` produces different behavior, and say why.

*A:* Any impure function, for example:
```python
def f(n):
    print('computing', n)
    return n * n
```
`memo(f)` prints only on the first call with a given `n`, since later calls return from the cache without entering `f`. Other examples: a function reading a global that changes, or one returning a random value. `memo` preserves behavior only for pure functions.

**5. Compare growth of two implementations.**

*Q:* `exp` makes how many recursive calls for `exp(2, 64)`? Roughly how many does `exp_fast(2, 64)` make, and why is the difference qualitative rather than just a constant factor?

*A:* `exp(2, 64)` makes 64 recursive calls (one per decrement of `n`), plus the initial call. `exp_fast(2, 64)` halves the exponent at every step, since 64, 32, 16, 8, 4, 2, 1 are all even until the last step, so about 7 calls. The difference is qualitative because `exp`'s count grows **proportionally to `n`** while `exp_fast`'s grows **much more slowly than `n`**: doubling `n` adds 64 more calls in the first case but only about one more in the second.

**6. Explain the jaggedness of the `exp_fast` plot.**

*Q:* Why is the call-count graph for `exp_fast` uneven rather than smooth?

*A:* When `n` is even the function halves the exponent, making fast progress; when `n` is odd it only subtracts 1 before it can halve again. Exponents whose halving chain stays even the whole way (powers of 2) require the fewest calls and show up as dips; exponents that repeatedly land on odd values need roughly twice as many steps. It is a real effect in the number of calls, not timing noise, and it does not change the overall shape of the growth.

**7. Rank or explain list operation costs.**

*Q:* Order these from fastest to slowest for large `n`, and explain the two slow ones: `[k*k for k in range(n)]`, repeated `s.append(...)`, repeated `s.insert(0, ...)`, repeated `s = s + [...]`.

*A:* Comprehension (about 1.58 ms at `n = 100,000`), then `append` (1.76 ms), then `insert(0, ...)` (1,470 ms), then `s = s + [...]` (9,210 ms). `insert(0, ...)` must shift all existing elements right to open position 0, so each of `n` iterations costs work proportional to the current length, giving quadratic total work. `s = s + [x]` is worse because it additionally **allocates a whole new list** and copies every element into it on each iteration, then discards the old one. Assignment into a pre-sized list (`s[k] = ...`) is in the fast group (2.58 ms).

**8. Complete a generator function.**

*Q:* Fill in the blanks so `powers` yields `1, b, b**2, ...` forever.
```python
def powers(b):
    n = 0
    _________:
        _________
        n += 1
```
*A:* `while True:` and `yield pow(b, n)`. This is safe despite being an infinite loop because the body executes only as `next` is called on the resulting generator; nothing runs when `powers(b)` is called.

**9. Write a generator that consumes an iterator statefully.**

*Q:* Complete `map_pairs(f, t)` so that `list(map_pairs(add, iter([3, 4, 5, 6, 7])))` is `[7, 9, 11, 13]`.

*A:*
```python
previous = next(t)
for x in t:
    yield f(previous, x)
    previous = x
```
`next(t)` pulls off the first item so the loop always has a left-hand partner, and because `t` is an iterator the `for` loop resumes at the second item rather than restarting. Yielding `f(previous, x)` before updating `previous` is essential.

**10. Write a recursive tree generator.**

*Q:* Complete `labels(t)` to yield every label of a `Tree` in preorder.

*A:*
```python
yield t.label
for b in t.branches:
    yield from labels(b)
```
`yield from` re-yields each element of the recursive sub-generator, flattening the results. `yield labels(b)` would instead yield generator objects. No leaf base case is needed: a tree with no branches just skips the loop.

**11. Write a generator that walks a linked list.**

*Q:* Complete `items(s)` so `sum(items(Link(3, Link(5, Link(7)))))` is 15.

*A:*
```python
while isinstance(s, Link):
    yield s.first
    s = s.rest
```
The `isinstance` test is the end-of-list check because the empty list is `()`, not a `Link`. Rebinding the local `s` advances the walk without mutating the list, and the binding survives while the generator is suspended at the `yield`.

**12. Base case of `partition`.**

*Q:* In `partition`, why does `n == 0` return `[()]` rather than `[]`?

*A:* The return value is a list of partitions. `[()]` is a list containing one partition, the empty linked list, which correctly says "there is exactly one way to partition 0: use no parts." `[]` would mean "there are zero ways to partition 0", which would make every `with_m` comprehension produce nothing and the whole function return no partitions. `[]` is the right answer only for the genuinely impossible cases, `n < 0` or `m == 0` with `n > 0`.

---

## Summary

- **Efficiency means growth.** Measure how time or work changes as input size grows, not the exact milliseconds of one run.
- **Counting calls** is a machine-independent proxy for running time; a counting wrapper adds the tally without changing a function's interface or its recursive calls (because the recursive lookups find the rebound global name).
- **Tree-recursive `fib` repeats enormous amounts of work.** `fib(5)` takes 15 calls; `fib(30)` takes about 2.69 million calls to return 832,040. Call count grows about as fast as the answer, which is exponential in `n`.
- **Memoization** caches results in a dictionary keyed by arguments, so each distinct argument is computed once. Memoized `fib(30)`: 31 calls to the original body (one per argument `0..30`) and 59 calls to the wrapper (the extra 28 are cache hits). Whole subtrees are **skipped**, never executed.
- The cache must live in the **enclosing** `memo` frame so it is shared across all calls, and you must **rebind the name the recursive calls use** for the speedup to reach inside the recursion.
- Memoization preserves behavior **only for pure functions**, and arguments must be hashable.
- **A better algorithm can beat caching.** `exp` is proportional to `n` (input 400 gives exactly 400 calls), while `exp_fast` halves the exponent whenever it is even and grows far more slowly (about 15 calls at `n = 800`). Memoization would not help `exp`, since no subproblem repeats.
- **Measuring method:** `timeit` over many calls, repeat the batch and take the **minimum**, convert to microseconds, plot against a realistic range of input sizes. The curve's shape matters; local jaggedness (from odd versus even exponents) does not.
- **List operations differ hugely.** Comprehensions, `append`, and index assignment are fast (1.58 to 2.58 ms at `n = 100,000`). `insert(0, ...)` shifts all elements (1,470 ms) and `s = s + [x]` allocates and copies a whole new list each time (9,210 ms). Any per-element operation whose cost grows with the current length makes a loop quadratic.
- **Generators are lazy iterators.** A generator function's body runs only under `next`, which makes infinite streams like `powers` practical; `prefix` takes a finite window from an infinite one.
- Generator functions can be **higher-order** (`map_pairs` takes `f`), can consume an iterator **statefully** (`next(t)` first, then `for x in t` resumes), can **iterate** over linked lists with an `isinstance(s, Link)` loop, and can **recurse** over trees with `yield from`.
