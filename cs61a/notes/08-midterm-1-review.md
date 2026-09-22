<!-- Mon, Sep 14, 2026 | sources: slides + code (no transcript available) -->
# Lecture 8: Midterm 1 Review

This lecture is Part 2 of the Midterm 1 review, and it is entirely worked problems rather than new material. It drills three skills: (1) building functions out of other functions using higher-order tools like `curry` and `reverse`, with no `lambda` and no `**` allowed, (2) writing `while` loops over the digits of an integer using the `n % 10` / `n // 10` idioms, including the "restart the scan" pattern, and (3) reading and writing `lambda` expressions and understanding how a nested `def` reads names from its enclosing frame. The problems come from Spring 2025 Midterm 1 (Questions 3a, 3b, 3c, and 4), and the lecture also presents an explicit problem-solving checklist from discussion: read the description, verify the doctests and pick a simple one, read the template, annotate the names with concrete values from that example, describe the process in English in simple steps, write the code, then ask "did I really return the right thing?" and re-check against the other examples. Almost every hint on the slides is a question about *meaning*: what does this helper do, what does this parameter represent, can you stop early.

## Key Concepts

### Functions are values, so you can build new functions instead of new code

`square` and `cube` are normally defined with `def` and a `return` statement. The lecture asks you to define them as *expressions* instead:

```python
square = curry(reverse(pow))(2)
cube   = curry(reverse(pow))(3)
```

Nothing here is a new algorithm. `pow` already computes powers. All that is missing is the right *shape*: `pow` takes the base first and the exponent second, and it takes both at once. Two small higher-order functions fix the shape.

### `reverse`: changing the argument order

```python
def reverse(f):
    return lambda x, y: f(y, x)
```

`reverse` takes a two-argument function and returns a new two-argument function that calls the original with the arguments swapped. The slide's identity is `reverse(pow)(2, 3) == pow(3, 2)`. So `reverse(pow)` means "raise the second argument to the first argument," i.e. `reverse(pow)(exponent, base)`.

The returned `lambda` is a closure: its parent frame is the frame for the call to `reverse`, which is where the name `f` lives. That is why the lambda can still reach `f` long after `reverse` has returned.

### `curry`: taking arguments one at a time

```python
def curry(f):
    def g(x):
        def h(y):
            return f(x, y)
        return h
    return g
```

Curry converts a two-argument function into a function you call twice, once with the first argument and once with the second: `curry(pow)(5)(2) == pow(5, 2)`. The value of `curry(pow)(5)` is a *one-argument function* that has already committed to a first argument of 5. That is exactly what we want: a one-argument function.

Putting the two together: `reverse(pow)` takes the exponent first, so currying it and supplying 2 gives a one-argument function that takes a base and squares it.

### Digit iteration: `% 10` and `// 10`

Every iteration problem in this lecture is over the decimal digits of a positive integer.

- `n % 10` is the last (rightmost) digit of `n`.
- `n // 10` is `n` with the last digit removed.
- `while n:` loops until `n` becomes 0, because 0 is the only falsy integer. It processes exactly the digits of `n`, right to left.

### Early exit from a universal claim

`all_digits(n, cond)` asks whether *every* digit satisfies `cond`. A single counterexample settles it, so you `return False` the moment you find a digit where `cond` is false, and `return True` only after the loop finishes without finding one. This is the standard "return False inside, return True after" shape for "for all" questions. The slide makes this a hint: "Can you ever stop before going through all of the digits?"

### Prefixes of an integer

A **prefix** of a positive integer `n` is `n // pow(10, p)` for some non-negative integer `p`. The prefixes of 3456 are 3456, 345, 34, 3, and 0. Repeatedly applying `// 10` walks through the prefixes from longest to shortest, which is exactly the order you want when you are looking for the *largest* prefix with some property.

### Using a helper by choosing the right `check`

`prefix_digits` is implemented by passing a function into a given helper:

```python
def process(n, check):
    while n:
        if check(n):
            return n
        n = n // 10
    return 0
```

`process` walks the prefixes of `n` from longest to shortest and returns the first one that passes `check`, or 0 if none does. The crucial reading skill: the parameter `k` in `lambda k: ...` is bound to **a whole prefix**, not a digit, because that is what `process` passes to `check`. So the test to apply is `all_digits(k, cond)`.

### The "shorten and restart" loop

Question 3(c) asks for the same behavior with no helper and no recursion, using this template:

```python
k = 0
while n >= pow(10, k):
    if cond(n // pow(10, k) % 10):
        k += 1
    else:
        n = n // 10
        k = 0
return n
```

Here `k` is a digit position counted from the right: `k = 0` is the last digit, `k = 1` is the second to last, and so on. `n // pow(10, k) % 10` extracts the digit at position `k`. The loop condition `n >= pow(10, k)` means "there is still a digit at position `k`," so the loop ends when `k` has walked off the left end of `n`, which means every digit passed. If any digit fails, you drop the last digit of `n` and reset `k = 0` to start scanning the shorter number from the right again.

### Nested `def` reading enclosing names

In Question 4, the inner function `f` takes no parameters at all. It reads `n` and `k` from the enclosing `hailstone` frame each time it is called, so it always sees their *current* values. It only reads them, never rebinds them, so no `nonlocal` is needed.

### `lambda` and `def` are interchangeable in power

The closing slide's point: any program containing `lambda` expressions can be rewritten with `def` statements.

```python
>>> (lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)
81
```

is the same computation as

```python
>>> def twice(f):
...     def g(x):
...         return f(f(x))
...     return g
>>> def square(y):
...     return y * y
>>> twice(square)(3)
81
```

The only differences are that a `lambda` is anonymous and its body must be a single expression.

## Definitions

- **Higher-order function**: a function that takes a function as an argument, returns a function, or both.
- **`reverse(f)`**: returns a two-argument function that calls `f` with its arguments swapped, so `reverse(f)(x, y)` is `f(y, x)`.
- **`curry(f)`**: converts a two-argument function into a function of one argument that returns a function of one argument, so `curry(f)(x)(y)` is `f(x, y)`.
- **Lambda expression**: an expression that evaluates to a function with no intrinsic name, written `lambda <params>: <single expression>`; the body is evaluated only when the function is called.
- **Closure / parent frame**: a function value records the frame in which it was defined; when called, its new frame's parent is that frame, so names not found locally are looked up there.
- **Prefix of a positive integer `n`**: the value `n // pow(10, p)` for some non-negative integer `p`. The prefixes of 3456 are 3456, 345, 34, 3, and 0.
- **`all_digits(n, cond)`**: returns `True` if `cond(d)` is true for every digit `d` of the positive integer `n`, and `False` otherwise.
- **`prefix_digits(n, cond)`**: returns the largest prefix of positive `n` all of whose digits satisfy `cond` (0 if there is no nonzero such prefix).
- **Hailstone step**: from `n`, the next value is `3 * n + 1` if `n` is odd and `n // 2` if `n` is even. (extra context: this is the Collatz sequence; it is conjectured, not proven, that it always reaches 1.)

## Worked Examples

### 1. `square` and `cube` from `pow`, in one line each

```python
def reverse(f):
    return lambda x, y: f(y, x)

def curry(f):
    def g(x):
        def h(y):
            return f(x, y)
        return h
    return g

square = curry(reverse(pow))(2)
cube   = curry(reverse(pow))(3)
```

Step by step, evaluating `square(3)`:

1. `reverse(pow)` opens a frame for `reverse` with `f` bound to `pow`, and returns `lambda x, y: f(y, x)`. That lambda's parent is the `reverse` frame, so it remembers `f = pow`. Call it `R`. Note `R(a, b)` is `pow(b, a)`.
2. `curry(R)` opens a frame for `curry` with `f` bound to `R`, and returns `g`, whose parent is that `curry` frame.
3. `curry(R)(2)` opens a frame for `g` with `x = 2`, and returns `h`, whose parent is that `g` frame. So `h` remembers `x = 2` and (through the chain) `f = R`.
4. `square` is now bound to `h`. Calling `square(3)` opens a frame for `h` with `y = 3` and evaluates `f(x, y)`. `f` is found in the `curry` frame as `R`, `x` is found in the `g` frame as 2, so this is `R(2, 3)`.
5. `R(2, 3)` is `pow(3, 2)`, which is 9.

In box-and-pointer / environment terms: three frames stay alive after the assignment (the `reverse` frame holding `f = pow`, the `curry` frame holding `f = R`, and the `g` frame holding `x = 2`), chained as parents of the function value stored in `square`.

Why not `curry(pow)(2)`? That would give `lambda y: pow(2, y)`, i.e. 2 to the *y*, so `square(3)` would be 8. The `reverse` is what puts the exponent first.

### 2. Spring 2025 Q3(a): `all_digits`

```python
def all_digits(n, cond):
    """Return whether cond returns true for every digit of positive n.

    >>> odd = lambda d: d % 2 == 1
    >>> all_digits(123, odd)
    False
    >>> all_digits(357, odd)
    True
    """
    while n:
        if not cond(n % 10):
            return False
        n = n // 10
    return True
```

The lecture's annotation walks `all_digits(123, odd)`:

| `n` at loop top | `n % 10` | `cond(n % 10)` | action |
| --- | --- | --- | --- |
| 123 | 3 | `odd(3)` is `True` | `n = 12` |
| 12 | 2 | `odd(2)` is `False` | `return False` |

For `all_digits(357, odd)`: digits 7, 5, 3 all pass, `n` becomes 0, the `while` condition is falsy, and the function reaches `return True`.

Why `not cond(...)` and `return False` rather than accumulating? Because a single failing digit is a complete answer, and stopping early is both correct and cheaper.

### 3. Spring 2025 Q3(b): `prefix_digits` using `process`

```python
def process(n, check):
    """A function to help implement prefix_digits."""
    while n:
        if check(n):
            return n
        n = n // 10
    return 0

def prefix_digits(n, cond):
    """Return the largest prefix of positive n for which cond returns true for every digit.

    >>> odd = lambda d: d % 2 == 1
    >>> prefix_digits(94720, odd)
    9
    >>> prefix_digits(919321, odd)
    9193
    >>> prefix_digits(2025, odd)
    0
    >>> prefix_digits(20252025, lambda d: d < 4)
    202
    >>> prefix_digits(20252025, lambda d: True)
    20252025
    """
    return process(n, lambda k: all_digits(k, cond))
```

Answering the slide's three questions:

- *What does `process` do?* It tries `n`, then `n // 10`, then `n // 100`, and so on, returning the first one for which `check` is true, or 0 if none is.
- *How does that help?* Those values are exactly the prefixes of `n`, tried longest first, so the first success is the largest qualifying prefix.
- *What does `k` represent?* A whole prefix (an integer), not a digit. So the check we want is "do all digits of this prefix satisfy `cond`," which is `all_digits(k, cond)`.

Trace of `prefix_digits(94720, odd)`:

| prefix `n` | `all_digits(n, odd)` | why |
| --- | --- | --- |
| 94720 | `False` | 0 is even |
| 9472 | `False` | 2 is even |
| 947 | `False` | 4 is even |
| 94 | `False` | 4 is even |
| 9 | `True` | returns 9 |

Trace of `prefix_digits(2025, odd)`: 2025, 202, 20, 2 all fail, `n` becomes 0, loop exits, `return 0`. Matches the doctest.

And `prefix_digits(20252025, lambda d: True)` succeeds immediately on the first prefix and returns 20252025.

### 4. Spring 2025 Q3(c): `prefix_digits` with a single loop

```python
def prefix_digits(n, cond):
    k = 0
    while n >= pow(10, k):
        if cond(n // pow(10, k) % 10):
            k += 1
        else:
            n = n // 10
            k = 0
    return n
```

The process in English, as stated on the slide: use `k` to call `cond` on each digit of `n`; if `cond` ever returns false, shorten `n` and start over.

The lecture's trace of `prefix_digits(94720, odd)`, with `n` taking the values 94720, 9472, 947, 94, 9 and `k` cycling 0, 1, 0, 1:

| `n` | `k` | digit `n // pow(10, k) % 10` | `cond` result | action |
| --- | --- | --- | --- | --- |
| 94720 | 0 | 0 | `odd(0)` is `False` | `n = 9472`, `k = 0` |
| 9472 | 0 | 2 | `odd(2)` is `False` | `n = 947`, `k = 0` |
| 947 | 0 | 7 | `odd(7)` is `True` | `k = 1` |
| 947 | 1 | 4 | `odd(4)` is `False` | `n = 94`, `k = 0` |
| 94 | 0 | 4 | `odd(4)` is `False` | `n = 9`, `k = 0` |
| 9 | 0 | 9 | `odd(9)` is `True` | `k = 1` |
| 9 | 1 | (loop condition `9 >= 10` is false) | | exit, `return 9` |

Two details worth internalizing:

- `n // pow(10, k) % 10` parses as `(n // pow(10, k)) % 10` because `//` and `%` have equal precedence and associate left to right. Shift right by `k` digits, then take the last digit.
- `while n >= pow(10, k)` is the "does position `k` still exist in `n`" test. When `k` equals the number of digits of `n`, `pow(10, k)` exceeds `n` and the loop stops, meaning all digits passed. It also terminates correctly when `n` reaches 0, since `0 >= 1` is false.

### 5. Spring 2025 Q4: `hailstone` with a zero-argument inner function

```python
def hailstone(n):
    """Print numbered updates in the hailstone sequence.

    >>> hailstone(10)
    1 10 -> 5
    2 5 -> 16
    3 16 -> 8
    4 8 -> 4
    5 4 -> 2
    6 2 -> 1
    """
    def f():
        if n % 2 == 1:
            m = 3 * n + 1
        else:
            m = n // 2
        print(k, n, '->', m)
        return m
    k = 1
    while n > 1:
        k, n = k + 1, f()
```

Answering the slide's questions: `f` computes the next hailstone value `m` from the current `n`, prints the numbered update line, and returns `m`. `n` is the current value, `m` is the next value, and `k` is the line number (the step counter starting at 1).

Environment reasoning: `f` is defined in `hailstone`'s frame, so `f`'s parent is that frame. `f` has no parameters, so `n`, `k`, and the built-in `print` are all found by following the parent chain. Because lookup happens at call time, each call to `f` sees the values of `n` and `k` as they currently are. `m` is local to `f`.

The assignment `k, n = k + 1, f()` is critical. Python evaluates the entire right-hand side first (left to right: `k + 1`, then the call `f()`), and only then rebinds `k` and `n`. So `f` runs with the *old* `k` and the *old* `n`, which is exactly what the printed line needs.

Trace of `hailstone(10)`:

| before the step | `k` | `n` | inside `f` | printed | after |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 | 10 | even, `m = 5` | `1 10 -> 5` | `k = 2`, `n = 5` |
| 2 | 2 | 5 | odd, `m = 16` | `2 5 -> 16` | `k = 3`, `n = 16` |
| 3 | 3 | 16 | even, `m = 8` | `3 16 -> 8` | `k = 4`, `n = 8` |
| 4 | 4 | 8 | even, `m = 4` | `4 8 -> 4` | `k = 5`, `n = 4` |
| 5 | 5 | 4 | even, `m = 2` | `5 4 -> 2` | `k = 6`, `n = 2` |
| 6 | 6 | 2 | even, `m = 1` | `6 2 -> 1` | `k = 7`, `n = 1` |

Then `n > 1` is false and `hailstone` returns `None` implicitly, having printed six lines.

### 6. Lambda rewritten as `def`

```python
>>> (lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)
81
```

Reading it left to right: `(lambda f: lambda x: f(f(x)))` is `twice`, and `(lambda y: y * y)` is `square`. Calling `twice(square)` returns a one-argument function that applies `square` twice. Applying it to 3 gives `square(square(3))` which is `square(9)` which is 81. The equivalent `def` version on the slide produces the same 81.

## Common Pitfalls

- **Writing `curry(pow)(2)` for `square`.** That is `lambda y: pow(2, y)`, which computes 2 to the *y*. You need `curry(reverse(pow))(2)` so that the fixed argument lands in the exponent slot.
- **Applying `reverse` and `curry` in the wrong order.** `reverse(curry(pow))` is a type error waiting to happen: `curry(pow)` takes one argument, but `reverse` returns a function expecting two.
- **Forgetting the constraint.** The problem forbids `lambda` and `**` in the answer, so `square = lambda x: x ** 2` and `square = lambda x: pow(x, 2)` both fail the problem's rules even though they work.
- **In `all_digits`, returning `True` inside the loop.** Returning `True` on the first passing digit answers the wrong question (it becomes "does *some* digit satisfy `cond`"). `True` belongs after the loop.
- **In `all_digits`, forgetting `n = n // 10`.** The loop never terminates.
- **Confusing digits and prefixes in Q3(b).** `lambda k: cond(k)` is wrong because `process` passes whole prefixes to `check`, not single digits. The correct body is `all_digits(k, cond)`.
- **In Q3(c), forgetting `k = 0` after shortening `n`.** If `k` keeps its old value after `n = n // 10`, you skip digits and get wrong answers.
- **In Q3(c), extracting the digit with `n % pow(10, k)` instead of `n // pow(10, k) % 10`.** The first gives the last `k` digits as a number, not the single digit at position `k`.
- **In Q4, putting `return m` before the `print`.** Nothing after a `return` in the same block executes, so no output is produced.
- **In Q4, giving `f` parameters.** The template writes `f()` with no arguments and the call site is `f()`, so `f` must read `n` and `k` from the enclosing frame.
- **Assuming `f` can update `n` in `hailstone`.** Assigning to `n` inside `f` would create a local `n` (and would not change the loop variable). The loop's own `k, n = k + 1, f()` does the updating.
- **Misreading simultaneous assignment.** `k, n = k + 1, f()` does *not* update `k` before calling `f`. Both right-hand expressions are evaluated first.
- **Confusing `print` and `return`.** `hailstone` prints and returns `None`; `all_digits`, `prefix_digits`, `square`, and `cube` return values and print nothing.

## Likely Exam Points

**1. Compose given higher-order functions to hit a target signature.**

*Practice:* Given `reverse` and `curry` as defined in lecture, define `halve` in one line (no `lambda`, no `/`) so that `halve(10)` is 5.0, using the built-in `truediv`-style two-argument function `div = lambda a, b: a / b` (assume `div` is given).
*Answer:* `halve = curry(reverse(div))(2)`. Then `halve(10)` is `reverse(div)(2, 10)` which is `div(10, 2)` which is 5.0.

**2. Evaluate a chained call on curried functions.**

*Practice:* What does `curry(reverse(pow))(3)(2)` evaluate to?
*Answer:* 8. It is `reverse(pow)(3, 2)`, which is `pow(2, 3)`.

**3. Fill in a digit loop with early exit.**

*Practice:* Complete `any_digit(n, cond)`, which returns `True` if `cond(d)` is true for at least one digit of positive `n`.
*Answer:*
```python
def any_digit(n, cond):
    while n:
        if cond(n % 10):
            return True
        n = n // 10
    return False
```
This is `all_digits` with the two boolean roles swapped and the `not` removed.

**4. Choose the right lambda to pass to a given helper.**

*Practice:* Using the lecture's `process`, complete `smallest_nonzero_prefix(n)` returning the smallest nonzero prefix of `n` that is greater than 50, or 0 if there is none. Explain what `k` is bound to.
*Answer:* You cannot use `process` for "smallest" directly, since `process` returns the *first* (largest) passing prefix. But `process(n, lambda k: k > 50)` returns the largest prefix over 50, and `k` is bound to a whole prefix each time. Recognizing that a helper's search order fixes which extreme you get is the point being tested.

**5. Trace a loop that mutates a counter and the data together.**

*Practice:* What is `prefix_digits(919321, odd)` under the Q3(c) implementation, and what is the value of `k` when the loop exits?
*Answer:* 9193. The digit 1 (last), then 2 fails, so `n` shortens to 91932, then 91932's last digit 2 fails, `n` becomes 9193; all of 3, 9, 1, 9 are odd so `k` climbs 0, 1, 2, 3, 4, and the loop exits when `9193 >= pow(10, 4)` is false, i.e. with `k = 4`.

**6. Nested `def` with no parameters reading enclosing names.**

*Practice:* In `hailstone`, which frame does the name `m` live in, and which frame does `k` live in?
*Answer:* `m` is a local name in each frame for a call to `f`. `k` lives in the `hailstone` frame; `f` finds it by following its parent pointer.

**7. Predict printed output vs returned value.**

*Practice:* In an interactive session, what appears after `>>> hailstone(4)`?
*Answer:*
```
1 4 -> 2
2 2 -> 1
```
and nothing else, because `hailstone` returns `None` and the REPL does not display `None`.

**8. Convert between `lambda` and `def`.**

*Practice:* Rewrite `compose = lambda f: lambda g: lambda x: f(g(x))` using only `def` statements.
*Answer:*
```python
def compose(f):
    def on_g(g):
        def on_x(x):
            return f(g(x))
        return on_x
    return on_g
```

**9. Evaluate an immediately applied lambda chain.**

*Practice:* What does `(lambda f: lambda x: f(f(x)))(lambda y: y + 3)(1)` evaluate to?
*Answer:* 7. The inner function adds 3, applied twice to 1.

**10. (extra context) Environment diagram drawing.** Midterm 1 in CS 61A typically includes an environment diagram question. For `square = curry(reverse(pow))(2)`, be ready to draw: a frame for `reverse` (with `f` bound to `pow` and a lambda whose parent is that frame), a frame for `curry` (with `f` bound to that lambda, and `g` defined there), a frame for `g` (with `x = 2`, and `h` defined there), and the global binding `square` pointing at `h`. Confirm the exact scope of your exam with the course staff, since this lecture only reviews the higher-order function, iteration, and lambda portions.

## Summary

- `reverse(f)` swaps a two-argument function's arguments: `reverse(pow)(2, 3) == pow(3, 2)`.
- `curry(f)` splits a two-argument function into two one-argument calls: `curry(pow)(5)(2) == pow(5, 2)`.
- The lecture's answers: `square = curry(reverse(pow))(2)` and `cube = curry(reverse(pow))(3)`.
- Digit iteration idioms: `n % 10` is the last digit, `n // 10` drops it, `while n:` visits every digit right to left.
- "For all" questions get an early `return False` inside the loop and `return True` after it, as in `all_digits`.
- A prefix of `n` is `n // pow(10, p)`; repeated `// 10` enumerates prefixes longest first, so the first success is the *largest* qualifying prefix.
- `prefix_digits(n, cond)` with the helper is `process(n, lambda k: all_digits(k, cond))`; the parameter `k` is a whole prefix, not a digit.
- The loop-only version uses `k` as a digit position from the right, tests `n // pow(10, k) % 10`, increments `k` on success, and on failure does `n = n // 10` followed by `k = 0` to restart the scan.
- In `hailstone`, the parameterless `f` reads `n` and `k` from the enclosing frame, computes `m`, prints `k, n, '->', m`, and returns `m`; `k, n = k + 1, f()` evaluates the right side (and so calls `f`) before rebinding anything.
- Any `lambda` program can be rewritten with `def`; `(lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)` is `twice(square)(3)` is 81.
- The method that generalizes: read the description, verify the doctests and pick a simple one, read the template, annotate names with concrete values, describe the process in plain English, write the code, check that you return the right thing, then re-test on the other examples.
