<!-- Wed, Sep 16, 2026 | sources: slides + code (no transcript available) -->
# Lecture 9: Recursion

## Overview

This lecture bridges two topics. It opens with a review of **code and environments**, making the point that you can determine which frame any name will be found in without drawing a full environment diagram: an expression outside any `def` or `lambda` is evaluated in the global frame, and an expression inside nested functions is evaluated in an environment with one frame per enclosing function, ordered inner to outer. This idea is exercised on the Fall 2026 Midterm 1 "New Print" problem, a higher-order function puzzle in which the built-in `print` is shadowed by a parameter and then rebound in the global frame twice. The lecture then introduces **recursive functions**: functions that call themselves on a smaller version of the problem. Using `fact` (factorial) as the running example, it contrasts the iterative `while` loop version with the recursive version, shows how the recursive computation "descends" to a base case and then multiplies back up as the calls return, and then shows how to convert iteration into recursion mechanically by **passing the state of the loop as arguments** (`fact_k`, `fact_tail`). The lecture closes with a discussion question applying that same conversion to the Twenty-One game: rewrite `play`'s `while` loop as a recursive inner function `f(n, who)` whose parameters are exactly the loop's changing state.

---

## Key Concepts

### 1. You can find a name's frame without the whole diagram

The environment in which an expression is evaluated is determined by *where the expression is written in the source code*, plus which function calls it sits inside at runtime. The lecture states two rules:

- **For any expression that appears outside a `def` statement or `lambda` expression, the environment is just the global frame.** Top-level code always runs in global.
- **For other expressions, the environment has a frame for every enclosing `def` statement or `lambda` expression, ordered from inner to outer**, ending in the global frame.

The slide's example:

```python
def exp(x):
    def base(b):
        return pow(b, x)
    return base
```

- `x` will be found in an `exp` frame (it is the argument to an `exp` call).
- `b` will be found in a `base` frame.
- The expression `pow(b, x)` will *always* be evaluated in an environment with **3 frames**: a `base` frame, then an `exp` frame, then global. That is because `pow(b, x)` is written inside `base`, which is written inside `exp`, which is written at top level.

The practical payoff: on an exam, when asked "where does this name come from?", count the enclosing `def`s/`lambda`s to know the shape of the environment, then walk outward until you find a frame that binds the name. You do not have to reconstruct every frame of an execution.

Note the subtlety the slide emphasizes: the frame chain for a nested function is determined by the **parent of the function value** (where it was defined), not by who called it. `base`'s parent is the `exp` frame that was active when the `def base` statement executed.

### 2. The New Print problem (Fall 2026 Midterm 1)

Only 9% of students got this right, which is why the lecture revisits it.

```python
def new_print(print):
    def f(x):
        value = print(x)
        if value != None:
            return print('What?')
        return x
    return f

og = print                      # original print, saved in global
print = new_print(print)        # global print is now f, whose print is the original
print = new_print(print)        # global print is now f, whose print is (f whose print is original)
og('print returned', print(2))
```

The whole trick is the environment reasoning from concept 1:

- Inside `f`, the name `print` is **not** the global `print`. `print` is the parameter of `new_print`, so `print` inside `f` is found in the enclosing `new_print` frame. Each `f` value "remembers" whichever print it was built with, via its parent frame.
- The call `new_print(print)` on line `print = new_print(print)` evaluates the *argument* expression `print` in the **global** frame, so it picks up whatever global `print` currently is at that moment. The first time that is the built-in; the second time it is the `f` from the first call.
- So after two rebindings, the global `print` is `f2`'s inner `f`: an `f` whose `print` is another `f`, whose `print` is the original built-in.

Tracing `print(2)` where `print` is the outer `f` (the one from the second `new_print` call, whose parent frame `f2` binds `print` to the first `f`):

1. **f4** (per the slide numbering): `x = 2`. Evaluate `value = print(x)`, where `print` here is the inner `f` from the first `new_print` call.
2. **f3**: that call has `x = 2`; it calls the *original* print, so `2` is displayed. The original `print` returns `None`, so `value = None` in this frame... wait, the slide shows f3 with `value 2`. Reading the slide's frame values: f3 has `x 2`, `value 2`, return value `'What?'`; f4 has `x 2`, `value None`, return value `2`.

   Reconciling with the code and the printed output (`2`, then `What?`, then `print returned What?`): the displayed output is `2`, then `What?`, then `print returned What?`. The consistent reading is that the innermost call actually invokes the original `print`, which displays `2` and returns `None`, so that frame returns `x`, i.e. `2`; the enclosing call then sees `value` = `2` (not `None`), so it evaluates `return print('What?')`, which recursively goes through the chain, displays `What?`, and returns `'What?'`. Finally `og('print returned', print(2))` displays `print returned What?`.
   *(extra context: the slide's frame labels are partially garbled by PDF extraction, so treat the printed output, `2` / `What?` / `print returned What?`, as the ground truth and the exact frame-by-frame value assignment as reconstructed.)*

The exam-worthy lessons, which the lecture states directly:

- `print` inside `f` "will be found in a `new_print` frame".
- The argument expression `print` in `print = new_print(print)` is "evaluated in global".
- `f` is "eval'd in some `f`, `new_print`, `Global`" environment (3 frames).
- Shadowing a built-in with a parameter is legal, and the parameter wins inside that function's body and inside any function defined within it.

### 3. What a recursive function is

A **recursive function** is one whose body calls the function itself. It works by reducing a problem to a smaller instance of the same problem, and handling the smallest instance(s) directly.

Every recursive function needs:

- **Base case(s):** input(s) small enough to answer without recursion. For `fact`, `n == 0 or n == 1` returns `1`.
- **Recursive case:** a call on a *smaller* input, whose result is combined into the answer. For `fact`, `return fact(n-1) * n`.

The essential move in writing and reading recursion is the **recursive leap of faith**: *(extra context: that name is from the CS 61A textbook, not stated on these slides.)* when you write `fact(n-1)`, assume it correctly returns `(n-1)!`, and just ask whether multiplying by `n` gives the right answer for `n`. You do not mentally unroll the whole chain. The lecture's diagrams reinforce this: `5! = 5 * 4 * 3 * 2 * 1`, where `4 * 3 * 2 * 1` is exactly `4!`, which is exactly what `fact(4)` promises to deliver.

### 4. How the recursive computation actually unfolds

The factorial slides show two phases, which is the key mental model.

**Descent (calls going down):** `fact(5)` needs `fact(4)`, which needs `fact(3)`, which needs `fact(2)`, which needs `fact(1)`. Each of these opens a new frame; none has computed anything yet. The slide's stack of `4!`, `3!`, `2!`, `1!` is this descent.

**Return (values coming back up):** `fact(1)` returns `1`; `fact(2)` returns `2 * 1 = 2`; `fact(3)` returns `3 * 2 = 6`; `fact(4)` returns `4 * 6 = 24`; `fact(5)` returns `5 * 24 = 120`. The slide's annotations `1`, `2`, `6`, `24` next to the subproblems are these returned values.

Crucially, the multiplication by `n` happens **after** the recursive call returns. So there is pending work sitting in every frame on the way down, and `n` must still be available when the call comes back. This is where box-and-pointer / environment reasoning matters (see Worked Examples).

### 5. Recursion versus iteration, and converting one to the other

The lecture puts the iterative and recursive factorials side by side:

```python
def fact(n):            # iterative
    result = 1
    while n > 0:
        result = result * n
        n -= 1
    return result
```

```python
def fact(n):            # recursive
    if n == 0 or n == 1:
        return 1
    else:
        return fact(n-1) * n
```

The comparison slide labels the loop version's `result` as the **running total**: `5`, then `20`, then `60`, then `120`. The loop accumulates *as it goes down*; the plain recursion accumulates *as it comes back up*. Same answer, opposite direction of accumulation.

**The conversion idea, stated on its own slide: "The state of the while loop is passed as arguments."** Whatever variables change across iterations of the loop become parameters of the recursive function. In the factorial case the loop's state is `(n, result)`, so the recursive function takes `(n, k)`:

```python
def fact_k(n, k):
    """Compute n factorial times k."""
    if n == 0 or n == 1:
        return k
    else:
        return fact_k(n-1, k * n)
```

Note how the docstring changes: `fact_k(n, k)` does not compute `n!`, it computes `n! * k`. The accumulator forces a **generalized** specification. That generalization is what makes the recursion work: `fact_k(5, 1) == 120`, `fact_k(5, 10) == 1200`, `fact_k(0, 10) == 10`.

Since callers only want `n!`, wrap it so the accumulator's initial value is hidden:

```python
def fact_tail(n):
    """Compute n factorial."""
    def f(n, k):
        if n == 0 or n == 1:
            return k
        else:
            return f(n-1, k * n)
    return f(n, 1)
```

Now `fact_tail` has the clean one-argument signature, and the inner `f` carries the loop state. This is the pattern: **inner helper function whose parameters are the loop variables, outer function supplies the initial values.**

The name `fact_tail` refers to *tail recursion*: the recursive call is the entire return expression, with no pending work after it returns. *(extra context: Python does not optimize tail calls, so `fact_tail` still builds a call stack of depth n, unlike a real loop. The slides do not discuss tail call optimization.)*

### 6. Twenty-One and the same conversion

**Rules as given:** two players alternate turns; on each turn a player adds 1, 2, or 3 to the current total; the total starts at 0; the game ends whenever the total is 21 or more; the last player to add to the total **loses**.

The iterative version:

```python
def play(strategy0, strategy1, goal=21):
    """Play twenty-one and return the winner."""
    n = 0
    who = 0   # Player 0 goes first
    while n < goal:
        if who == 0:
            n = n + strategy0(n)
            who = 1
        elif who == 1:
            n = n + strategy1(n)
            who = 0
    return who
```

The discussion question asks: rewrite `play` recursively, without a `while` statement, and answers three sub-questions.

- **Do you need a new inner function? Why?** Yes. `play`'s own parameters are `strategy0`, `strategy1`, `goal`, which do not change; the loop's changing state is `n` and `who`, which are not parameters of `play`. So you need a function that takes `n` and `who` as arguments. (You could instead add default-valued parameters to `play` itself, but that exposes them to callers; the inner helper keeps `play`'s interface clean, exactly as `fact_tail` does.)
- **What are its arguments?** `n` and `who`: precisely the variables reassigned inside the loop.
- **What is the base case and what does it return?** The base case is the loop's exit condition, negated: `n >= goal`. It returns `who`, matching the `return who` after the loop.

The recursive version:

```python
def play(strategy0, strategy1, goal=21):
    """Play twenty-one and return the winner."""
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

Notice the mechanical correspondence:

| `while` loop version | recursive version |
| --- | --- |
| `n = 0`, `who = 0` before the loop | `return f(0, 0)` (initial arguments) |
| `while n < goal:` | `if n >= goal: return who` (base case is the negated condition) |
| loop body | body of `f` (unchanged) |
| going back to the top of the loop | `return f(n, who)` |
| `return who` after the loop | the base case's `return who` |

The inner `f` can see `strategy0`, `strategy1`, and `goal` because those names are found in the enclosing `play` frame, which is exactly the environment rule from the first half of the lecture: an expression inside `f` is evaluated in an environment of `f` frame, then `play` frame, then global.

Also note: because `who` is flipped *after* the current player adds, when the total reaches or exceeds `goal`, `who` holds the player who did **not** just move, and the last player to add loses. So returning `who` correctly returns the winner.

---

## Definitions

- **Environment:** a sequence of frames, searched in order (inner to outer) when looking up a name.
- **Frame:** a binding of names to values created by a function call (or the global frame, which is created at startup).
- **Parent frame (of a function):** the frame in which the function's `def` statement or `lambda` expression was evaluated. It determines where names not bound locally are looked up, not the caller's frame.
- **Global frame:** the outermost frame; the environment for any expression written outside any `def` or `lambda`.
- **Shadowing:** binding a name in an inner frame that is also bound in an outer frame (e.g. the parameter `print` in `new_print`), so that the inner binding is found first inside that function.
- **Recursive function:** a function whose body contains a call to the function itself.
- **Base case:** a case handled directly without a recursive call, terminating the recursion (for `fact`: `n == 0 or n == 1`).
- **Recursive case:** the case that calls the function on a smaller/simpler input and combines the result.
- **Accumulator (running total):** an extra parameter carrying the partial result computed so far, e.g. `k` in `fact_k(n, k)`; it is how a `while` loop's running total becomes a recursive argument.
- **Tail recursion:** a recursive call that is the entire value returned, with no pending computation after it (as in `fact_k` / `fact_tail`, in contrast to `fact(n-1) * n` where a multiplication is still pending).
- **Twenty-One:** the game where two players alternately add 1, 2, or 3 to a total starting at 0; play ends when the total reaches 21 or more; the last player to add **loses**.
- **Strategy function:** a function of the current total `n` that returns how much (1, 2, or 3) to add; `strategy0` and `strategy1` are passed to `play` as higher-order arguments.

---

## Worked Examples

### Example 1: Environment shape without drawing the diagram

```python
def exp(x):
    def base(b):
        return pow(b, x)
    return base
```

Question: what environment does `pow(b, x)` get evaluated in?

Step by step:

1. `pow(b, x)` is written inside `def base`, which is written inside `def exp`, which is written at top level.
2. By the rule, the environment has a frame for every enclosing `def`, inner to outer: a `base` frame, then an `exp` frame, then the global frame. **Three frames, always.**
3. `b` is `base`'s parameter, so it is found in the first frame.
4. `x` is not bound in `base`, so lookup continues to `base`'s parent, an `exp` frame, where `x` is `exp`'s parameter. Found.
5. `pow` is bound in neither, so lookup continues to global (and then the builtins).

So `square = exp(2); square(3)` evaluates `pow(3, 2)` = `9`, with `b = 3` in the `base` frame and `x = 2` in the `exp` frame created by the earlier `exp(2)` call. Notice the `exp` frame outlives the `exp` call: the function value `base` holds a pointer to it, which is exactly what a box-and-pointer picture shows (the function object `func base(b) [parent=f1]` with an arrow to the `f1: exp` frame).

### Example 2: `fact`, iterative then recursive

```python
def fact(n):
    """Compute n factorial.

    >>> fact(5)
    120
    >>> fact(0)
    1
    """
    result = 1
    while n > 0:
        result = result * n
        n -= 1
    return result
```

Trace of `fact(5)`, one frame only, with two names being reassigned:

| after iteration | `result` | `n` |
| --- | --- | --- |
| start | 1 | 5 |
| 1 | 5 | 4 |
| 2 | 20 | 3 |
| 3 | 60 | 2 |
| 4 | 120 | 1 |
| 5 | 120 | 0 |

Loop exits (`n > 0` is false), returns `120`. These are exactly the "running total" values `5, 20, 60, 120` on the comparison slide.

Now the recursive version:

```python
def fact(n):
    """Compute n factorial."""
    if n == 0 or n == 1:
        return 1
    else:
        return fact(n-1) * n
```

Trace of `fact(5)`, in terms of frames:

1. `f1: fact` with `n = 5`. `5 != 0` and `5 != 1`, so evaluate `fact(4) * 5`. Python evaluates `fact(4)` first, so frame `f1` is suspended with a pending multiplication by `5`.
2. `f2: fact` with `n = 4`. Pending: multiply by `4`. Calls `fact(3)`.
3. `f3: fact` with `n = 3`. Pending: multiply by `3`. Calls `fact(2)`.
4. `f4: fact` with `n = 2`. Pending: multiply by `2`. Calls `fact(1)`.
5. `f5: fact` with `n = 1`. **Base case.** Returns `1`.

Now the returns unwind:

6. `f4` computes `1 * 2 = 2`, returns `2`.
7. `f3` computes `2 * 3 = 6`, returns `6`.
8. `f2` computes `6 * 4 = 24`, returns `24`.
9. `f1` computes `24 * 5 = 120`, returns `120`.

Environment reasoning worth internalizing: **there are five simultaneously live `fact` frames**, each with its own `n`. All five have parent = global (because `def fact` is at top level, so every `fact` function value has global as its parent). They are *not* nested in each other environment-wise; they just happen to be stacked in time. Each frame's `n` is private to that frame, which is precisely why `24 * 5` uses `f1`'s `n` and not `f5`'s.

Also note `fact(0)`: the base case catches `n == 0` and returns `1`, so `fact(0)` is `1` as the docstring requires. Without the `n == 0` test, `fact(0)` would call `fact(-1)`, `fact(-2)`, and recurse forever.

The lecture code imports `from ucb import trace`, which lets you decorate `fact` with `@trace` to see exactly this call-and-return pattern printed out.

### Example 3: `fact_k`, the accumulator version

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

Trace of `fact_k(5, 1)`:

| frame | `n` | `k` | action |
| --- | --- | --- | --- |
| f1 | 5 | 1 | returns `fact_k(4, 5)` |
| f2 | 4 | 5 | returns `fact_k(3, 20)` |
| f3 | 3 | 20 | returns `fact_k(2, 60)` |
| f4 | 2 | 60 | returns `fact_k(1, 120)` |
| f5 | 1 | 120 | base case, returns `120` |

Then `120` is returned unchanged all the way back out: `f4` returns `120`, `f3` returns `120`, and so on. Compare the `k` column, `1, 5, 20, 60, 120`, with the `while` loop's `result` column, `1, 5, 20, 60, 120`. **They are identical.** That is the whole point of the conversion slide: the loop's running total became an argument.

Contrast with the plain `fact`: there, nothing useful is computed on the way down and the real work happens on the way back up. Here, the answer is fully computed by the time the base case is reached, and the return trip is just passing `120` along unchanged. That is what "tail recursion" means.

Check the other doctests:
- `fact_k(5, 10)`: same descent but `k` starts at `10`, giving `10, 50, 200, 600, 1200`. Returns `1200 = 5! * 10`. ✓
- `fact_k(0, 10)`: base case immediately, returns `k = 10 = 0! * 10`. ✓

### Example 4: `fact_tail`, hiding the accumulator

```python
def fact_tail(n):
    """Compute n factorial."""
    def f(n, k):
        if n == 0 or n == 1:
            return k
        else:
            return f(n-1, k * n)
    return f(n, 1)
```

What happens on `fact_tail(5)`:

1. A `fact_tail` frame is created with `n = 5`, parent global.
2. `def f` executes, creating a function value `func f(n, k) [parent = the fact_tail frame]` and binding it to `f` in the `fact_tail` frame.
3. `return f(n, 1)` looks up `f` (found locally) and `n` (found locally, `5`), and calls `f(5, 1)`.
4. Each `f` frame has parent = the `fact_tail` frame (that is where `def f` ran). Inside `f`, the name `f` is not a local parameter, so lookup goes to the parent `fact_tail` frame, where `f` is bound. **This is how a nested function can call itself.**
5. The `n` inside `f` shadows `fact_tail`'s `n`: `f`'s frames each bind their own `n` as a parameter, so the outer `n = 5` is never consulted inside `f`.
6. The recursion proceeds exactly as in Example 3, returning `120`.

Box-and-pointer style summary in words: one `fact_tail` frame holds a pointer to the function object `f`; each `f` frame holds an arrow back to that single `fact_tail` frame as its parent; the chain of `f` frames is a stack in time, not a chain of parents.

### Example 5: Twenty-One, iteration to recursion

Given the iterative `play` above, apply the conversion recipe:

1. **Identify the loop's state.** Inside the `while`, only `n` and `who` are reassigned. `strategy0`, `strategy1`, `goal` are fixed.
2. **Make those the parameters of a helper.** `def f(n, who):`.
3. **Base case = negation of the loop condition, returning what the loop returns after exiting.** Loop condition is `n < goal`, so base case is `if n >= goal: return who`.
4. **Copy the loop body verbatim.**
5. **Replace "go back to the top of the loop" with a recursive call carrying the updated state:** `return f(n, who)`.
6. **Replace the initialization with the initial call:** `return f(0, 0)`.

Result:

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

A concrete trace (extra context: with a simple strategy that always adds 3 for both players, so `strategy0(n) = strategy1(n) = 3`):

| call | `n` on entry | `who` on entry | `n >= 21`? | new `n` | new `who` |
| --- | --- | --- | --- | --- | --- |
| `f(0, 0)` | 0 | 0 | no | 3 | 1 |
| `f(3, 1)` | 3 | 1 | no | 6 | 0 |
| `f(6, 0)` | 6 | 0 | no | 9 | 1 |
| `f(9, 1)` | 9 | 1 | no | 12 | 0 |
| `f(12, 0)` | 12 | 0 | no | 15 | 1 |
| `f(15, 1)` | 15 | 1 | no | 18 | 0 |
| `f(18, 0)` | 18 | 0 | no | 21 | 1 |
| `f(21, 1)` | 21 | 1 | **yes** | - | - |

Returns `1`. Player 0 made the move that pushed the total to 21, so player 0 was the last to add and therefore loses; player 1 wins. The returned `who` is `1`. ✓ Consistent with the rules.

Note that this is also tail recursive: `return f(n, who)` is the whole return expression, and the base case's `return who` value travels back out unchanged.

Regarding the sub-question "do you need a new inner function?": yes, because the state that changes across iterations (`n`, `who`) is not among `play`'s parameters, and `play` must still be callable as `play(strategy0, strategy1)`. The inner function also gets `strategy0`, `strategy1`, and `goal` for free through its parent frame, so they do not need to be passed along.

---

## Common Pitfalls

1. **Forgetting a base case, or writing one that is never reached.** `fact` with only `if n == 1` would recurse forever on `fact(0)`. The lecture's base case is `n == 0 or n == 1` for exactly this reason. In Python this produces a `RecursionError` rather than an infinite hang. *(extra context: the error name is not stated on the slides.)*

2. **Recursing on something that is not smaller.** `return fact(n) * n` never terminates. The argument must move toward the base case on every call.

3. **Assuming a name inside a nested function refers to the global binding.** In `new_print`, `print` inside `f` is the *parameter*, found in the `new_print` frame. This is the single biggest reason only 9% got that problem right.

4. **Confusing the caller's frame with the parent frame.** The environment for a call is (new frame) → (function's parent frame) → ... → global. A function defined at top level always has global as its parent, no matter how deeply nested the call stack is. So the five `fact` frames in Example 2 do not have each other as parents.

5. **Evaluating the argument expression in the wrong frame.** In `print = new_print(print)`, the argument `print` is evaluated in global *before* the assignment happens, so it is the *old* global `print`, which is why running that line twice builds a chain rather than looping.

6. **Thinking `fact_k(n, k)` computes `n!`.** It computes `n! * k`. If you write the docstring as `n!` you will pick the wrong base case return value (`1` instead of `k`) and break the accumulation.

7. **Forgetting to return the recursive call.** Writing `f(n, who)` instead of `return f(n, who)` makes the function return `None`. The recursive call's value must be propagated.

8. **Forgetting to start the accumulator correctly.** `f(n, 1)`, not `f(n, 0)`: for multiplication the identity is `1`. And in `play`, `f(0, 0)` reproduces `n = 0; who = 0`.

9. **Getting the base case condition backwards when converting a loop.** The loop runs `while n < goal`; the recursion *stops* at `n >= goal`. Negate the condition.

10. **Expecting Python to optimize tail calls.** `fact_tail(10000)` will still blow the recursion limit; the function name describes the shape of the recursion, not a Python guarantee. *(extra context.)*

11. **Ordering the multiplication in a way that changes nothing but confuses tracing.** The slides write both `n * fact(n-1)` and `fact(n-1) * n`. Both are correct for factorial, but the order matters for *when* the recursive call is made relative to other evaluation (Python evaluates operands left to right), and it matters a lot for non-commutative operations.

---

## Likely Exam Points

### 1. Determining which frame a name is found in

**Practice:** Given

```python
def outer(a):
    def middle(b):
        def inner(c):
            return a + b + c
        return inner
    return middle
```

In what environment is `a + b + c` evaluated, and in which frame is each name found?

**Answer:** In an environment of four frames, inner to outer: an `inner` frame, a `middle` frame, an `outer` frame, then global. `c` is found in the `inner` frame, `b` in the `middle` frame, `a` in the `outer` frame. This follows the lecture's rule: one frame per enclosing `def`, ordered inner to outer, then global.

### 2. Shadowing a built-in / higher-order rebinding (the New Print pattern)

**Practice:**

```python
def wrap(abs):
    def g(x):
        return abs(x) + 1
    return g

orig = abs
abs = wrap(abs)
abs = wrap(abs)
print(abs(-3))
```

What is printed, and where is the `abs` inside `g` found?

**Answer:** `abs` inside `g` is found in the enclosing `wrap` frame (it is `wrap`'s parameter), never in global. The first `wrap(abs)` captures the built-in, giving a `g` that computes `|x| + 1`. The second `wrap(abs)` evaluates its argument in global, picking up that first `g`, so the new `g` computes `g_first(x) + 1 = |x| + 1 + 1`. `abs(-3)` is `3 + 1 + 1 = 5`. Prints `5`.

### 3. Writing a recursive function with correct base case

**Practice:** Write a recursive `summation(n)` that returns `1 + 2 + ... + n` for `n >= 1`, with no loops.

**Answer:**

```python
def summation(n):
    if n == 1:
        return 1
    else:
        return n + summation(n - 1)
```

Base case `n == 1` returns `1`; recursive case trusts `summation(n-1)` to give the sum up to `n-1` and adds `n`. `summation(4)` = `4 + (3 + (2 + 1))` = `10`.

### 4. Tracing a recursion and counting frames

**Practice:** When `fact(4)` (the recursive version) reaches its base case, how many `fact` frames exist, and what is the parent of each?

**Answer:** Four frames, with `n` equal to `4, 3, 2, 1`. Every one has the **global frame** as its parent, because `def fact` is a top-level statement, so the `fact` function value's parent is global. The frames are stacked in the call stack, but they are not each other's parents.

### 5. Converting a `while` loop to recursion (the lecture's core skill)

**Practice:** Convert to recursion without a `while` statement:

```python
def count_down(n):
    total = 0
    while n > 0:
        total = total + n % 10
        n = n // 10
    return total
```

**Answer:** The loop state is `n` and `total`, so make them the helper's parameters; the base case is the negated loop condition `n <= 0` returning `total`; the initial call supplies `n` and `0`.

```python
def count_down(n):
    def f(n, total):
        if n <= 0:
            return total
        return f(n // 10, total + n % 10)
    return f(n, 0)
```

### 6. The `fact_k` generalization and its base case

**Practice:** Why does `fact_k`'s base case return `k` rather than `1`? What would break if it returned `1`?

**Answer:** Because `fact_k(n, k)` is specified to compute `n! * k`, not `n!`. When `n` reaches the base case, `k` already holds the entire accumulated product, so returning `k` returns the answer. Returning `1` would discard all accumulated work: `fact_k(5, 1)` would return `1` instead of `120`, and `fact_k(0, 10)` would return `1` instead of `10`.

### 7. Twenty-One recursive `play`

**Practice:** In the recursive `play`, why can the inner `f` use `goal` and `strategy0` without them being parameters, and why must the base case return `who` rather than `1 - who`?

**Answer:** `f`'s parent frame is the `play` frame, where `goal`, `strategy0`, and `strategy1` are bound as `play`'s parameters, so name lookup from inside `f` finds them one frame out. The base case returns `who` because `who` is flipped immediately after a player moves: when the total reaches `goal`, `who` names the player who did *not* make the last move, and since the last player to add loses, that player is the winner. This matches the iterative version's `return who` after the loop.

### 8. Identifying tail recursion

**Practice:** Which of `fact(n-1) * n` and `f(n-1, k * n)` leaves pending work in the calling frame, and why does that matter?

**Answer:** `fact(n-1) * n` leaves a pending multiplication: after the recursive call returns, the frame must still multiply by its own `n`, so every frame must be kept alive holding its `n`. `f(n-1, k * n)` leaves nothing pending: its value is returned unchanged, which is what makes the return trip a straight pass-through of `120`. (Python still keeps all the frames, but conceptually this version mirrors a loop exactly.)

---

## Summary

- **Environment shortcut:** an expression outside any `def`/`lambda` is evaluated in the global frame; otherwise the environment has one frame per enclosing `def`/`lambda`, inner to outer, ending in global. You can determine the frame a name will be found in without drawing the whole diagram.
- A function's **parent frame** is where its `def` ran, not where it was called. That is why `pow(b, x)` in `exp`/`base` is always evaluated in exactly three frames.
- **New Print (Fall 2026 Midterm 1, 9% correct):** `print` inside `f` is found in the enclosing `new_print` frame (the parameter, shadowing the built-in), while the argument in `print = new_print(print)` is evaluated in global. Doing this twice chains two wrappers around the original `print`. Output: `2`, `What?`, `print returned What?`.
- A **recursive function** calls itself on a smaller input. It needs a **base case** (answered directly) and a **recursive case** (smaller call, result combined).
- `fact`: base case `n == 0 or n == 1` returns `1`; recursive case `return fact(n-1) * n`. The calls descend to `1`, then the products `1, 2, 6, 24, 120` are built back up as the frames return.
- Each recursive call gets its **own frame with its own `n`**; frames stack in time but all share the same parent (global, for a top-level `fact`).
- **Iteration → recursion: pass the state of the `while` loop as arguments.** Loop variables become parameters; the base case is the negated loop condition; the post-loop return becomes the base case's return; the initialization becomes the initial call.
- `fact_k(n, k)` computes `n! * k`; the accumulator `k` reproduces the loop's running total (`1, 5, 20, 60, 120`), and the base case must return `k`.
- `fact_tail(n)` hides the accumulator in an inner `f(n, k)` and calls `f(n, 1)`, keeping the clean one-argument interface. Inner `f` finds its own name in the enclosing frame, which is how it recurses.
- **Twenty-One:** alternate adding 1, 2, or 3 to a total starting at 0; ends at 21 or more; last to add loses. The recursive `play` needs an inner `f(n, who)` (those are the loop's changing state), base case `if n >= goal: return who`, and initial call `f(0, 0)`. `strategy0`, `strategy1`, `goal` come from the enclosing `play` frame.
- **Tail recursion** (`fact_k`, `fact_tail`, recursive `play`) computes the answer on the way down with nothing pending on the way back up, mirroring a loop; plain `fact` computes on the way back up.
