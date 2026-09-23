<!-- Wed, Sep 16, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 9: Recursion

## Overview

This lecture makes the jump from functions that loop to functions that call themselves. It opens with a review of how to reason about environments quickly (you can determine which frames an expression is evaluated in, and therefore where each name is found, just by looking at the nesting of `def` statements and `lambda` expressions, without drawing a full diagram), applied to the Midterm 1 "New Print" problem that only 9% of students got right. It then introduces recursive functions: functions whose body calls the function itself, either directly (`fact` calling `fact`) or indirectly (mutual recursion, where `luhn_sum` and `luhn_sum_double` call each other). The core ideas are the anatomy of a recursive function (base case tested first, recursive case that calls the function on a *simpler* problem), tracing recursion in an environment diagram (many frames for the same function, each with its own binding for `n`, all with the same parent frame), verifying correctness via the *recursive leap of faith* (verify the base case, then assume the recursive call is correct and check that it combines correctly), and the relationship between iteration and recursion (iteration is a special case of recursion; converting a `while` loop to recursion is mechanical because the loop's state simply becomes the arguments of a recursive call, while going the other direction requires more thought). The lecture closes with two applications of that conversion rule: `fact_tail` (factorial with an accumulator) and the Twenty-One game (`play` rewritten with no `while` statement).

---

## Key Concepts

### 1. Reading environments off the code (no diagram needed)

The lecture starts by generalizing what you already know about environment diagrams into a shortcut you can apply by inspection:

- **For any expression that appears outside a `def` statement or `lambda` expression**, the environment is just the global frame.
- **For any other expression**, the environment has one frame for every enclosing `def` statement or `lambda` expression, ordered from inner to outer, ending in the global frame.

Example from the slides:

```python
def exp(x):
    def base(b):
        return pow(b, x)
    return base
```

The expression `pow(b, x)` sits inside `base`, which sits inside `exp`, which sits in the global frame. So that expression is **always** evaluated in an environment with exactly three frames: a `base` frame, then an `exp` frame, then global. Therefore:

- `b` will always be found in a `base` frame (it is `base`'s parameter).
- `x` will always be found in an `exp` frame (it is the argument to some `exp` call).
- `pow` will be found in the global frame (or builtins).

The payoff: **for any name, you know which frame it will be found in without drawing the whole environment diagram.** You still might not know the *value* (a different call to `exp` binds a different `x`), but you know *where to look*.

Why this works: the parent of a frame is determined by *where the function was defined*, not where it was called. So the chain of frames you walk when looking up a name mirrors the chain of `def` nesting in the source code. (extra context: this is called lexical/static scoping.)

### 2. What a recursive function is

> A recursive function is a function whose body calls itself, either directly or indirectly.

Recursion is not exclusive to computer science: the lecture mentions the Sierpinski Triangle, which is defined as three smaller Sierpinski Triangles, each of which is itself three Sierpinski Triangles, as an example from art/math/nature.

The key insight is that executing the body of a recursive function may require applying that same function again, to a **simpler** version of the problem.

### 3. Anatomy of a recursive function

Every recursive function in this lecture has the same three-part shape:

1. **Header** (`def name(params):`): looks like any other function definition, no special syntax.
2. **Base case(s)**: a conditional statement at the top checking for very simple versions of the problem. Base cases are **evaluated without recursive calls**; they can usually be computed directly (just `return n`, just `return 1`, just `return k`).
3. **Recursive case(s)**: evaluated **with** recursive calls, where each recursive call is made on a *simpler* problem that is closer to the base case.

In `sum_digits`, the base case is "n has only one digit" and the recursive case shrinks `n` by one digit each time. In `fact`, the base case is `n == 0 or n == 1` and the recursive case decreases `n` by 1 each time.

### 4. Recursion in environment diagrams

When you trace `fact(3)`, several things happen that are worth naming explicitly:

- **The same function is called multiple times.** The first call comes from global; subsequent calls come from inside `fact`'s own body.
- **Different frames keep track of the different arguments in each call.** This is exactly why the computer does not get confused calling one function inside itself: each call gets a fresh frame with its own binding.
- **What `n` evaluates to depends on the current environment.** Every expression containing `n` can evaluate to a different value depending on which frame is first in the current environment.
- **Each call solves a simpler problem than the last.** There is less work to do for `n = 0` than `n = 1`, less for `n = 1` than `n = 2`, and so on, which guarantees you eventually hit the base case.
- **All the `fact` frames have the same parent: global.** `fact` was *defined* in the global frame, so every frame created by calling it has global as its parent, regardless of which frame the call was made from.
- **Returning "unwinds" the stack.** When the innermost call returns, control goes back to exactly the expression that made that call, in the frame that was active at the time, and the pending multiplication finally happens.

### 5. The recursive leap of faith

How do you convince yourself a recursive function is correct without mentally simulating every call? The lecture's strategy, using `fact`:

1. **Verify the base case.** If `n` is 0 (or 1), `fact` returns 1, which is correct by definition.
2. **Treat `fact` as a functional abstraction.** For the recursive call `fact(n-1)`, do *not* think about how it is implemented. Think only about what it is *supposed* to do: return `(n-1)!`.
3. **Assume `fact(n-1)` is correct, then verify the whole thing.** If `fact(n-1)` really returns `(n-1)!`, then `n * fact(n-1)` really is `n!`. Done.

Concretely: is `fact(4)` equal to `4 * fact(3)`? Yes: `24 == 4 * 6`.

Generally: **assume the function is correctly defined for the simpler case used in the recursive call, then verify that under that assumption it is correct for the problem you were given.** This is why it feels like a leap of faith: you trust the function you are still in the middle of writing.

### 6. Iteration versus recursion

**Iteration is a special case of recursion.** Anything a `while` loop does, a recursive function can do.

Compare:

```python
def fact_iter(n):           # iterative
    total, k = 1, 1
    while k <= n:
        total, k = total * k, k + 1
    return total

def fact(n):                # recursive
    if n == 0:
        return 1
    else:
        return n * fact(n - 1)
```

These correspond to two different but equally correct mathematical definitions:

- Iterative: `n! = 1 * 2 * 3 * ... * n` (multiply in `k` for `k` from 1 to `n`).
- Recursive: `n! = 1` if `n = 0`, otherwise `n! = n * (n-1)!`.

The recursive version is shorter, quicker to explain, and (the lecture argues) often easier to follow. Look at how many *names* each version needs: the iterative version juggles `n`, `total`, `k`, and `fact_iter`; the recursive version needs only `n` and `fact`. In the recursive version, the frames of the environment diagram keep track of where you are in the computation, so you do not have to track it by hand with extra variables.

### 7. Converting iteration to recursion (the mechanical direction)

This direction is **straightforward**, precisely because iteration is a special case of recursion. The recipe:

> **Idea: the state of the `while` loop is passed as arguments.**

Step by step:

1. Identify the **state maintained across iterations** of the `while` suite (every name that gets reassigned inside the loop, plus the loop variable).
2. Make each piece of state a **parameter** of the recursive function.
3. The `while` condition becomes the **negation** of the base-case test: `while n > 0` becomes `if n == 0: return ...`.
4. The loop body becomes the recursive case, and **updates via assignment become arguments to the recursive call.**

Example:

```python
def sum_digits_iter(n):
    digit_sum = 0
    while n > 0:
        n, last = split(n)
        digit_sum = digit_sum + last
    return digit_sum

def sum_digits_rec(n, digit_sum):
    if n == 0:
        return digit_sum
    else:
        n, last = split(n)
        return sum_digits_rec(n, digit_sum + last)
```

The state `n` and `digit_sum` became the two parameters. The reassignment `digit_sum = digit_sum + last` became the argument `digit_sum + last`.

### 8. Converting recursion to iteration (the harder direction)

This can be tricky. The approach: figure out what state needs to be maintained across each pass through the `while` statement. Clues come from what gets **passed into** each recursive call and what gets **returned** from it. For `sum_digits`, what is passed in is "what is left to sum" (`n`), and what is returned is a partial sum. Those become the two loop variables `n` and `digit_sum`.

### 9. Accumulators and the running total (`fact_k` / `fact_tail`)

The slides contrast two shapes of `fact`:

- **Recursive version 1**: the multiplication by `n` happens *after* the recursive call returns. The pending multiplications pile up: `5 * (4 * (3 * (2 * 1)))`. The work happens "on the way back up."
- **`fact_tail` / `while` loop style**: a **running total** is carried along. By the time you reach the base case, the answer `120` is already computed, and it is simply passed back up unchanged. The work happens "on the way down."

`fact_k(n, k)` computes "n factorial times k", which generalizes the problem so that the running total `k` fits as a parameter. `fact_tail(n)` hides that extra parameter behind an inner helper so the public interface is still just `fact_tail(n)`.

(extra context: a call like `return f(n-1, k*n)`, where the recursive call is the entire return expression with nothing left to do afterward, is called a **tail call**. Python does *not* optimize tail calls, so this still uses one frame per call. The transformation is presented here for clarity and for its relationship to loops, not for efficiency.)

### 10. Mutual recursion

**Mutual recursion occurs when two different functions call each other.** This is the "indirectly calls itself" case from the definition. `luhn_sum` calls `luhn_sum_double`, and `luhn_sum_double` calls `luhn_sum`. Base cases can appear in both functions or in only one of them; in the Luhn example they appear in both.

### 11. Why digit sums matter (motivation)

- A number is divisible by 9 if and only if its digit sum is divisible by 9.
- **Checksum digits**: credit card numbers are long and humans mistype them. The 16th digit is not part of your account number; it is computed from the other digits. If the check digit does not match the computation on the rest, the number was typed in wrong.
- The **Luhn algorithm** is the real checksum used for credit cards. A valid number always has a Luhn sum that is a multiple of 10. If any single digit is wrong, the Luhn sum will not be a multiple of 10, and almost all transpositions (swapping two adjacent digits) are also detected.

---

## Definitions

- **Recursive function**: a function whose body calls itself, either directly (the function names itself) or indirectly (through another function that calls back).
- **Base case**: a case of a recursive function that is evaluated **without** any recursive calls. It handles the simplest version(s) of the problem and can be computed directly.
- **Recursive case**: a case that is evaluated **with** one or more recursive calls, each applied to a simpler problem than the original (closer to a base case).
- **Recursive leap of faith**: the verification strategy of (1) checking the base case, (2) treating the recursive call as a functional abstraction that is assumed to be correct for the simpler input, and (3) checking that the result is correctly built from that assumed-correct value.
- **Functional abstraction**: thinking about *what* a function is supposed to do (its behavior/contract), not *how* it is implemented. Required to take the recursive leap of faith.
- **Mutual recursion**: the situation in which two (or more) different functions call each other, forming a recursive cycle.
- **Iteration**: repeated execution via a `while` (or `for`) statement, in which state is updated by assignment. Iteration is a special case of recursion.
- **State (of a loop)**: the collection of names whose values are maintained and updated across each pass through the loop body. When converting to recursion, these become the parameters of the recursive function.
- **Environment**: a sequence of frames, ordered from innermost to global, in which an expression is evaluated. For any expression, the environment has a frame for every enclosing `def` or `lambda`, inner to outer, ending in global.
- **Parent frame**: the frame determined by where a function was **defined** (not where it was called). Every call to `fact` (defined globally) creates a frame whose parent is the global frame.
- **Digit sum**: the sum of the decimal digits of a non-negative integer, for example `sum_digits(2013) == 2 + 0 + 1 + 3 == 6`.
- **Checksum digit**: a digit computed from all the other digits of a number, appended so that typos can be detected. The last digit of a credit card number is a checksum digit.
- **Luhn sum**: starting from the rightmost digit (the check digit) and moving left, double the value of every second digit; if the product of that doubling is greater than 9, sum the digits of the product; then take the sum of all the resulting digits. A valid credit card number has a Luhn sum that is a multiple of 10.
- **Running total / accumulator**: an extra parameter (like `k` in `fact_k`) that carries the partial result computed so far down through the recursive calls, so that the base case can return the final answer directly.

---

## Worked Examples

### Example 1: `split` and `sum_digits`

The goal is to sum digits **without** a `while` statement, which forces recursion.

First, the building block:

```python
def split(n):
    """Split positive n into all but its last digit and its last digit."""
    return n // 10, n % 10
```

`split(2013)` returns the two-element result `(201, 3)`: everything but the last digit, and the last digit. Integer division by 10 chops off the last digit; the remainder mod 10 *is* the last digit.

Now the recursion:

```python
def sum_digits(n):
    """Sum the digits of positive integer n."""
    if n < 10:
        return n
    else:
        all_but_last, last = split(n)
        return sum_digits(all_but_last) + last
```

**Why this is correct**, in the lecture's own words: if I can sum the digits of `201`, and I add `3` to that, I get the sum for `2013`.

- **Base case** (`n < 10`): `n` is a single digit, so its digit sum is trivially itself. The sum of the digits of 7 is just 7.
- **Recursive case**: we do **not** call `sum_digits(n)` again (that would be infinite). We call it on `all_but_last`, which has *fewer digits*, so it is strictly simpler and moves toward the base case.

**Trace of `sum_digits(2013)`:**

| Call | `n` | `all_but_last`, `last` | Returns |
|---|---|---|---|
| `sum_digits(2013)` | 2013 | 201, 3 | `sum_digits(201) + 3` |
| `sum_digits(201)` | 201 | 20, 1 | `sum_digits(20) + 1` |
| `sum_digits(20)` | 20 | 2, 0 | `sum_digits(2) + 0` |
| `sum_digits(2)` | 2 | (base case) | `2` |

Unwinding: `2` then `2 + 0 = 2` then `2 + 1 = 3` then `3 + 3 = 6`. Result: **6**.

### Example 2: `fact` and its environment diagram

The recursive factorial, in the two forms shown:

```python
# transcript / Python Tutor version
def fact(n):
    if n == 0:
        return 1
    else:
        return n * fact(n - 1)

# slides / 09.py version (equivalent, with an extra base case)
def fact(n):
    """Compute n factorial.

    >>> fact(5)
    120
    >>> fact(0)
    1
    """
    if n == 0 or n == 1:
        return 1
    else:
        return fact(n - 1) * n
```

**Environment reasoning for `fact(3)`**, described in words:

1. The **global frame** binds `fact` to a function object `func fact(n) [parent=Global]`.
2. Calling `fact(3)` creates **f1: fact**, parent Global, with `n` bound to `3`. (Parent is Global because `fact` was *defined* in the global frame.)
3. In f1, `3 == 0` is false, so we evaluate `n * fact(n-1)`. To do that we must first compute `fact(2)`. **f1's return value is still blank** at this moment.
4. Calling `fact(2)` creates **f2: fact**, parent Global, `n` bound to `2`. Note that the parent is Global, *not* f1: the call came from inside `fact`, but the parent depends on the definition site.
5. `2 == 0` is false, so we need `fact(1)`, creating **f3: fact**, parent Global, `n = 1`.
6. `1 == 0` is false, so we need `fact(0)`, creating **f4: fact**, parent Global, `n = 0`.
7. In f4, `0 == 0` is true. We return `1`. **f4's return value is 1.**
8. Control returns to the exact expression that made the call: the `fact(n-1)` inside f3's return statement. In f3, `n` is `1`, so we compute `1 * 1 = 1`. **f3's return value is 1.**
9. Control returns to f2's return statement. In f2, `n` is `2`, so `2 * 1 = 2`. **f2's return value is 2.**
10. Control returns to f1's return statement. In f1, `n` is `3`, so `3 * 2 = 6`. **f1's return value is 6.**
11. `fact(3)` evaluates to **6**, which is correct: `3 * 2 * 1 = 6`.

Things to notice in that diagram, all of which the lecture calls out explicitly:

- There are four different frames all for the *same* function, and four different bindings for the *same* name `n`. Which one `n` refers to depends entirely on which frame is first in the current environment.
- At the moment f4 returns, f1, f2, and f3 are all still open, each stuck mid-way through evaluating a return expression, with blank return values.
- The problem gets strictly simpler at each level, guaranteeing termination.

### Example 3: `fact_k` and `fact_tail` (running total)

```python
def fact_k(n, k):
    """Compute n factorial times k.

    >>> fact_k(5, 1)
    120
    >>> fact_k(5, 10)
    1200
    >>> fact_k(0, 10)
    10
    """
    if n == 0 or n == 1:
        return k
    else:
        return fact_k(n-1, k * n)
```

Read the docstring carefully: `fact_k(n, k)` does **not** compute `n!`, it computes `n! * k`. That generalization is what makes the accumulator work, and it is what you verify with the recursive leap of faith:

- Base case: if `n` is 0 or 1, `n!` is 1, so `n! * k == k`. Correct.
- Recursive case: assume `fact_k(n-1, k*n)` correctly returns `(n-1)! * (k*n)`. Rearranged, that is `n * (n-1)! * k == n! * k`, exactly what `fact_k(n, k)` is supposed to return. Correct.

`fact_tail` hides the accumulator behind a helper so the public signature stays clean:

```python
def fact_tail(n):
    """Compute n factorial.

    >>> fact_tail(5)
    120
    >>> fact_tail(0)
    1
    """
    def f(n, k):
        if n == 0 or n == 1:
            return k
        else:
            return f(n-1, k * n)
    return f(n, 1)
```

**Trace of `fact_tail(5)`:** `f(5, 1)` then `f(4, 5)` then `f(3, 20)` then `f(2, 60)` then `f(1, 120)` which hits the base case and returns `120`. Then `120` is passed back up through every frame unchanged. Compare to version 1, where `5 * (4 * (3 * (2 * 1)))` is assembled on the way back up. Same answer, opposite direction of work.

**Environment note:** `f` is defined inside `fact_tail`, so every frame created by calling `f` has the `fact_tail` frame (call it f1) as its parent, not the global frame and not the previous `f` frame. Applying the shortcut rule from the start of lecture: any expression inside `f`'s body is evaluated in an environment with three frames (an `f` frame, the `fact_tail` frame, then global).

Also note the iterative `fact` at the top of `09.py`, which is the loop this recursion mirrors:

```python
def fact(n):
    result = 1
    while n > 0:
        result = result * n
        n -= 1
    return result
```

The loop state is `result` and `n`; in `fact_tail` those became the parameters `k` and `n`.

### Example 4: Mutual recursion, the Luhn algorithm

Rule, from Wikipedia as quoted in lecture: from the rightmost digit (the check digit), moving left, double the value of every second digit; if the product of this doubling is greater than 9, sum the digits of the product; then take the sum of all the digits.

Worked by hand on `138743`, right to left:

| Digit (right to left) | 3 | 4 | 7 | 8 | 3 | 1 |
|---|---|---|---|---|---|---|
| Doubled? | no | yes | no | yes | no | yes |
| After doubling | 3 | 8 | 7 | 16 | 3 | 2 |
| After digit-summing products > 9 | 3 | 8 | 7 | 7 | 3 | 2 |

Total: `3 + 8 + 7 + 7 + 3 + 2 = 30`, which is a multiple of 10, so this passes the check.

The code splits the work between two functions that call each other:

```python
def luhn_sum(n):
    """Return the Luhn sum of n, where n's last digit is NOT doubled."""
    if n < 10:
        return n
    else:
        all_but_last, last = split(n)
        return luhn_sum_double(all_but_last) + last

def luhn_sum_double(n):
    """Return the Luhn sum of n, where n's last digit IS doubled."""
    all_but_last, last = split(n)
    luhn_digit = sum_digits(2 * last)
    if n < 10:
        return luhn_digit
    else:
        return luhn_sum(all_but_last) + luhn_digit
```

**Why two functions?** Because the rule alternates. Whether a digit gets doubled depends on its position parity. Rather than track parity with an extra parameter, we encode it in *which function we are in*: `luhn_sum` means "this number's last digit is not doubled", and `luhn_sum_double` means "this number's last digit is doubled". Each one recurses into the *other*, which flips the parity automatically. That is mutual recursion.

Note `luhn_sum_double` computes `luhn_digit = sum_digits(2 * last)` before checking its base case, which handles the "if the product is greater than 9, sum its digits" clause: `sum_digits(16)` is `7`, and `sum_digits(8)` is just `8`, so one call covers both situations.

**Trace of `luhn_sum(32)`:**
1. `luhn_sum(32)`: `32 >= 10`, split into `(3, 2)`, return `luhn_sum_double(3) + 2`.
2. `luhn_sum_double(3)`: split `3` into `(0, 3)`, `luhn_digit = sum_digits(2 * 3) = sum_digits(6) = 6`. Since `3 < 10`, return `6`.
3. Back in step 1: `6 + 2 = 8`.

Result **8**, matching the lecture: "the 3 gets doubled to 6 plus 2 is 8." And `luhn_sum(2)` is just `2`.

### Example 5: `sum_digits` iterative and back to recursive

```python
def sum_digits_iter(n):
    digit_sum = 0
    while n > 0:
        n, last = split(n)
        digit_sum = digit_sum + last
    return digit_sum
```

To get here from the recursive version, ask: what is passed in to each recursive call, and what comes back? Passed in: what is left to sum (`n`). Returned: a partial sum. Those two become the loop's state, `n` and `digit_sum`.

Going the other way is mechanical. The state `n` and `digit_sum` become parameters; `while n > 0` becomes the opposite test `if n == 0`; assignments become arguments:

```python
def sum_digits_rec(n, digit_sum):
    if n == 0:
        return digit_sum
    else:
        n, last = split(n)
        return sum_digits_rec(n, digit_sum + last)
```

> **Updates via assignment become arguments to a recursive call.** This can be done quite generally for every iterative implementation.

### Example 6: Twenty-One (discussion question)

**Rules:** two players alternate turns; on each turn a player adds 1, 2, or 3 to the current total. The total starts at 0. The game ends whenever the total is 21 or more. **The last player to add to the total loses.**

Iterative version:

```python
def play(strategy0, strategy1, goal=21):
    """Play twenty-one and return the winner.

    >>> play(some_strat, some_other_strat)
    1
    """
    n = 0
    who = 0  # Player 0 goes first
    while n < goal:
        if who == 0:
            n = n + strategy0(n)
            who = 1
        elif who == 1:
            n = n + strategy1(n)
            who = 0
    return who
```

Recursive version, no `while` statement:

```python
def play(strategy0, strategy1, goal=21):
    """Play twenty-one and return the winner.

    >>> play(some_strat, some_other_strat)
    1
    """
    def f(n, who):
        if n >= goal:
            return who
        if who == 0:
            n = n + strategy0(n)
            who = 1
        elif who == 1:
            n = n + strategy1(n)
            who = 0
        return f(n, who)
    return f(0, 0)
```

The three discussion questions answered:

- **Do you need a new inner function? Why?** Yes. The loop maintains two pieces of state, `n` and `who`, and those must become parameters. But `play`'s signature is fixed at `(strategy0, strategy1, goal=21)`, so we cannot add parameters to `play` itself. An inner function `f` gets the new parameters while still having access to `strategy0`, `strategy1`, and `goal` through its parent frame (the `play` frame).
- **What are its arguments?** `n` (the current total) and `who` (whose turn it is), exactly the state maintained across passes of the `while` loop. It is started with `f(0, 0)`, matching the loop's initialization `n = 0; who = 0`.
- **What is the base case, and what is returned?** The base case is `n >= goal`, the negation of the loop condition `n < goal`. It returns `who`, exactly what the loop returned after finishing. This is correct because `who` has already been flipped to the player who did *not* just add to the total, and the last player to add **loses**, so `who` is the winner.

Notice the body of `f` after the base case is a verbatim copy of the `while` suite, and the assignment updates are simply handed to the recursive call `f(n, who)`.

### Example 7: The "New Print" midterm problem (announcement review)

```python
def new_print(print):
    def f(x):
        value = print(x)
        if value != None:
            return print('What?')
        return x
    return f

og = print                    # og is the original print
print = new_print(print)      # f whose print is the original print
print = new_print(print)      # f whose print is (f whose print is the original print)
og('print returned', print(2))
```

**Applying the environment shortcut first.** Inside `f`'s body, `print` is not a local name of `f`, so we walk outward: the next enclosing `def` is `new_print`, and `print` *is* `new_print`'s parameter. So **`print` inside `f` is always found in a `new_print` frame**, never the global frame. That is the crux of the problem: rebinding the global `print` afterwards does not affect what `print` means inside an already-created `f`.

**The frames, in order:**

- **f1: `new_print`**, parent Global, `print` bound to the original built-in print. Returns a function `func f(x) [parent=f1]`, "the `f` whose `print` is the original".
- **f2: `new_print`**, parent Global, `print` bound to that first `f`. Returns `func f(x) [parent=f2]`, "the `f` whose `print` is the `f` whose `print` is the original". After this line, the global `print` names this second `f`.
- Evaluating `print(2)` in the last line calls the second `f`, creating **f3: `f`**, parent **f2**, `x = 2`.
- In f3, `print` is found in f2, which is the *first* `f`. Calling it creates **f4: `f`**, parent **f1**, `x = 2`.
- In f4, `print` is found in f1, which is the *original* print. So `print(2)` **prints `2`** and returns `None`. So in f4, `value` is `None`, the `if` is skipped, and f4 returns `x`, which is `2`.
- Back in f3, `value` is `2`, which is not `None`, so we evaluate `return print('What?')`. That calls the first `f` again, creating **f5: `f`**, parent **f1**, `x = 'What?'`.
- In f5, the original print **prints `What?`** and returns `None`, so `value` is `None`, the `if` is skipped, and f5 returns `'What?'`.
- So f3 returns `'What?'`.
- Finally, `og('print returned', 'What?')` uses the saved original print and **prints `print returned What?`**.

**Printed output, in order:**

```
2
What?
print returned What?
```

Only 9% of students got this right on Midterm 1. As the slide put it: "Success is not final, failure is not fatal: it is the courage to continue that counts."

---

## Common Pitfalls

1. **Making the recursive call on the same problem.** Writing `return sum_digits(n) + last` instead of `sum_digits(all_but_last) + last` produces infinite recursion. **Every recursive call must be on a strictly simpler problem.**

2. **Forgetting or mis-testing the base case.** With no base case, recursion never stops. With the wrong base case (for example, testing `n == 0` for `sum_digits` when the argument shrinks past single digits in a way that never hits exactly 0), you get wrong answers or infinite recursion.

3. **Confusing "where a function was called" with "where it was defined" when assigning parent frames.** All four `fact` frames in the `fact(3)` trace have parent **Global**, even though three of them were called from inside `fact`. The parent depends on the definition site. Conversely, every frame for `f` inside `fact_tail` has the `fact_tail` frame as parent, not global and not a previous `f` frame.

4. **Trying to resolve a name without checking the enclosing `def`s.** In the New Print problem, `print` inside `f` is found in a `new_print` frame, not the global frame. Rebinding the global `print` after `f` was created changes nothing about that `f`.

5. **Assuming a frame's return value is filled in as soon as the call is made.** While recursive calls are outstanding, the outer frames sit with blank return values, paused in the middle of evaluating an expression. They only fill in as the recursion unwinds.

6. **Refusing to take the leap of faith and trying to trace every level.** For deep recursion this is impossible to do reliably. Verify the base case, assume the recursive call is correct, verify the combination step.

7. **Misreading a generalized helper's contract.** `fact_k(n, k)` returns `n! * k`, not `n!`. If you "verify" it against the wrong specification, correct code will look broken.

8. **In mutual recursion, forgetting that base cases may be needed in both functions.** In the Luhn example, both `luhn_sum` and `luhn_sum_double` have base cases. Base cases can appear in both functions or in only one, and you need to reason about which.

9. **When converting a loop to recursion, dropping a piece of state.** If a name is reassigned in the loop body, it must appear as a parameter. Twenty-One needs both `n` and `who`; dropping `who` makes the problem unsolvable.

10. **Forgetting to negate the loop condition for the base case.** `while n < goal` becomes `if n >= goal`, not `if n < goal`.

11. **Trying to add parameters to a function whose signature is fixed.** When a problem says "rewrite `play` as a recursive function", you cannot change `play`'s parameters, so you need an inner helper.

---

## Likely Exam Points

### 1. Fill in the blanks of a recursive function

The classic format: a function is given with its base case test, base case return value, or recursive call arguments blanked out.

**Practice:** Fill in the blanks so that `count_digits(n)` returns the number of digits in positive integer `n`.

```python
def count_digits(n):
    if ____________:
        return ____________
    else:
        return ____________
```

**Answer:**
```python
def count_digits(n):
    if n < 10:
        return 1
    else:
        return count_digits(n // 10) + 1
```
The base case is a single-digit number (one digit). The recursive case strips the last digit with `n // 10` and counts one more. Verify by the leap of faith: assume `count_digits(n // 10)` correctly counts the digits of all but the last, then adding 1 accounts for the last digit.

### 2. Environment diagram with recursion: how many frames, and what are their parents?

**Practice:** For the code below, how many frames are created in total (excluding the global frame), and what is the parent of each?

```python
def fact(n):
    if n == 0 or n == 1:
        return 1
    else:
        return fact(n-1) * n

fact(4)
```

**Answer:** Four frames: `fact` with `n=4`, `n=3`, `n=2`, `n=1`. (The call stops at `n == 1` because of the `or n == 1` base case; there is no `n=0` frame.) **Every one of them has the global frame as its parent**, because `fact` is defined in the global frame. Parent frames come from the definition site, not the call site.

### 3. Order of printed output / return values in a deeply nested trace

**Practice:** What does this print?

```python
def f(n):
    if n == 0:
        return 0
    print(n)
    result = f(n - 1)
    print(n * 10)
    return result
f(3)
```

**Answer:**
```
3
2
1
10
20
30
```
The first three prints happen on the way *down* (before each recursive call), the last three on the way back *up* (after each recursive call returns), so the second group is in reverse order.

### 4. Convert a `while` loop to recursion

**Practice:** Rewrite the following without a `while` statement, keeping the signature `total(n)` unchanged.

```python
def total(n):
    s = 0
    while n > 0:
        s = s + n
        n = n - 1
    return s
```

**Answer:** The state is `n` and `s`, so we need a helper with both as parameters (the signature of `total` cannot change). The `while n > 0` condition becomes the base case `if n == 0`, and the assignments become the arguments of the recursive call.

```python
def total(n):
    def helper(n, s):
        if n == 0:
            return s
        return helper(n - 1, s + n)
    return helper(n, 0)
```
(A direct version `return n + total(n-1)` with base case `if n == 0: return 0` is also correct, but the mechanical conversion is the one the lecture teaches.)

### 5. Identify and justify base case / recursive case

**Practice:** In `sum_digits`, why is the recursive call made on `n // 10` rather than on `n`, and why does the recursion terminate?

**Answer:** `n // 10` is a strictly simpler problem because it has one fewer digit than `n`. Calling `sum_digits(n)` would repeat the identical problem forever. Termination is guaranteed because each call reduces the digit count by one, so after finitely many calls the argument is less than 10 and the base case `if n < 10: return n` fires without a recursive call.

### 6. Verify correctness (the recursive leap of faith)

**Practice:** State the three steps you would use to argue that `fact_k(n, k)` correctly returns `n! * k`.

**Answer:**
1. **Base case:** if `n` is 0 or 1, then `n!` is 1, so the correct answer is `1 * k == k`, which is exactly what is returned.
2. **Functional abstraction:** treat the call `fact_k(n-1, k*n)` as a black box and assume it returns `(n-1)! * (k*n)`, which is its documented behavior.
3. **Verify the combination:** `(n-1)! * (k * n) == n * (n-1)! * k == n! * k`, which is what `fact_k(n, k)` is supposed to return. Since the return statement passes that value straight back, the function is correct.

### 7. Mutual recursion

**Practice:** What is mutual recursion, and why does the Luhn algorithm use it instead of a single function?

**Answer:** Mutual recursion occurs when two different functions call each other, so each calls itself *indirectly*. The Luhn rule alternates between doubling and not doubling successive digits. Using two functions, `luhn_sum` (last digit not doubled) and `luhn_sum_double` (last digit doubled), lets the alternation be encoded in *which function you are in*: each one recurses into the other, flipping the parity on every step, with no extra parameter needed to track position. Base cases appear in both functions here, though in general they may appear in only one.

### 8. Trace `luhn_sum` on a small number

**Practice:** What is `luhn_sum(138)`?

**Answer:** 
- `luhn_sum(138)`: split into `(13, 8)`, return `luhn_sum_double(13) + 8`.
- `luhn_sum_double(13)`: split into `(1, 3)`, `luhn_digit = sum_digits(2 * 3) = sum_digits(6) = 6`. Since `13 >= 10`, return `luhn_sum(1) + 6`.
- `luhn_sum(1)`: `1 < 10`, return `1`.
- So `luhn_sum_double(13) = 1 + 6 = 7`, and `luhn_sum(138) = 7 + 8 = 15`.

Sanity check by hand, right to left: `8` not doubled is 8; `3` doubled is 6; `1` not doubled is 1. Total `8 + 6 + 1 = 15`. Not a multiple of 10, so `138` would fail the Luhn check.

### 9. Name lookup without drawing a diagram

**Practice:** In the code below, in which frame will each of `b`, `x`, and `pow` be found when `pow(b, x)` is evaluated?

```python
def exp(x):
    def base(b):
        return pow(b, x)
    return base
```

**Answer:** `pow(b, x)` is inside `base`, inside `exp`, inside global, so it is always evaluated in an environment of exactly three frames: a `base` frame, an `exp` frame, then global. `b` is found in the `base` frame (it is `base`'s parameter), `x` is found in the `exp` frame (it is the argument to the `exp` call that created `base`), and `pow` is found in the global frame (as a built-in). You do not need to draw anything to determine this.

### 10. Higher-order functions plus shadowing (the New Print style problem)

**Practice:** In the New Print code, after both `print = new_print(print)` lines have run, what does `print` inside the body of the *first* `f` (the one returned by the first `new_print` call) refer to?

**Answer:** The **original built-in print**. Inside `f`, the name `print` is found by looking in `f`'s frame (not there), then its parent, the `new_print` frame f1, where `print` is bound to the original print that was passed in as an argument. Rebinding the **global** `print` twice afterwards has no effect on this lookup, because `f`'s environment never includes the global frame until after the `new_print` frame has already supplied a binding.

---

## Summary

- A **recursive function** calls itself, either **directly** (`fact` calls `fact`) or **indirectly** (**mutual recursion**: `luhn_sum` and `luhn_sum_double` call each other).
- **Anatomy**: an ordinary `def` header, then a conditional testing for **base case(s)** (evaluated without recursive calls, computed directly), then **recursive case(s)** (evaluated with recursive calls on **simpler** problems that approach the base case).
- Recursion is normal function application: each call gets its **own frame with its own bindings**, which is why nested calls to the same function do not interfere. A name like `n` evaluates differently depending on which frame is first in the current environment.
- **A frame's parent comes from where the function was defined, not where it was called.** All `fact` frames have Global as parent; all frames for an inner helper `f` have the enclosing function's frame as parent.
- While recursive calls are outstanding, the outer frames are paused mid-expression with **blank return values**; they fill in as the recursion unwinds, innermost first.
- **The recursive leap of faith** verifies correctness in three steps: (1) check the base case, (2) treat the recursive call as a **functional abstraction** assumed correct for the simpler input, (3) verify that the result is correctly built from that value. Assume correct for `n-1`, show correct for `n`.
- **Iteration is a special case of recursion.** The recursive `fact` needs only two names (`n`, `fact`); the iterative one needs four (`n`, `total`, `k`, `fact_iter`), because frames replace the manual bookkeeping.
- **Converting iteration to recursion is mechanical**: the **state maintained across the loop** becomes the **parameters**, the `while` condition becomes the **negated** base case test, and **updates via assignment become arguments to a recursive call**. This works quite generally.
- **Converting recursion to iteration is harder**: identify what needs to be maintained across passes, using what is passed into and returned from each recursive call as clues.
- An **accumulator / running total** parameter (`fact_k`, `fact_tail`) does the work on the way *down* so the base case returns the final answer, versus the classic form which assembles the answer on the way *back up*. A helper inside the function keeps the public signature unchanged.
- **Twenty-One**: needs an inner `f(n, who)` because `play`'s signature is fixed, base case `n >= goal` returning `who` (the player who did not just add, since the last player to add loses).
- **Environment shortcut**: an expression outside any `def`/`lambda` is evaluated in the global frame; otherwise its environment has one frame per enclosing `def`/`lambda`, inner to outer, then global. This tells you exactly which frame every name will be found in without drawing the diagram, which is the key to problems like New Print.
- **Digit sums and Luhn**: `split(n)` returns `(n // 10, n % 10)`; `sum_digits` recurses on all but the last digit and adds the last; the Luhn algorithm doubles every second digit from the right (digit-summing products over 9), and a valid credit card number has a Luhn sum that is a multiple of 10.
