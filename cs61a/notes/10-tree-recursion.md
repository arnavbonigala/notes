<!-- Fri, Sep 18, 2026 | sources: YouTube auto-transcript -->
# Lecture 10: Tree Recursion

## Overview

This lecture is about what happens *when* during a recursive computation, and what becomes possible once a single call can branch into more than one recursive call. It opens with `cascade`, a linear recursive function whose print statements sit both before and after the recursive call, which forces you to reason precisely about the fact that a function call must return before anything written after it can run. That same idea is then re-examined through an environment diagram, through a shorter rewrite of `cascade` (with a discussion of which version is better to read), and through an `inverse_cascade` exercise built out of higher-order functions (`grow`, `shrink`, and a helper `f_then_g`). The second half introduces **tree recursion**: any function whose body makes more than one call to itself, producing a tree-shaped computational process rather than a chain. The two examples are `fib` (simple, familiar, and extremely wasteful because it recomputes the same subproblems over and over, illustrated live with the `trace` decorator from the `ucb` module) and `count_partitions` (the payoff example: a problem that is genuinely hard to write *without* tree recursion, where the two recursive calls correspond to exploring two disjoint possibilities and the results are summed).

---

## Key Concepts

### 1. A call must return before the next statement runs

The single most important rule for reading recursive code: when a function calls another function, that call must complete and produce a value before execution moves on to whatever comes after it. Nothing surprising happens just because the called function happens to be the *same* function. So in a body like:

```python
print(n)
cascade(n // 10)
print(n)
```

the entire recursive cascade, all of its own nested output, happens *between* the two prints. This is exactly why `cascade` produces its nested, symmetric shape: the "on the way down" work all comes from statements before the recursive call, and the "on the way back up" work all comes from statements after it.

### 2. Statements can appear before or after a recursive call

A recursive function body is not "base case, then recurse, the end." Work can be scheduled in two different phases:

- **Before the recursive call**: runs in order from the largest problem down to the base case.
- **After the recursive call**: runs in order from the base case back up to the largest problem.

In `cascade(123)`, the first frame prints `123` (before), the second prints `12` (before), the third prints `1` (base case), then the second frame prints `12` again (after), then the first prints `123` again (after). The lecture emphasizes: it was the third call that printed `1`, the second call that printed *both* `12`s, and the first call that printed both `123`s.

### 3. Frames, return values, and "not yet finished"

In the environment diagram for `cascade(123)` there are three separate `cascade` frames, one per call, each with its own binding for `n` (123, 12, 1). Each frame's parent is the global frame (because `cascade` was defined in the global frame), not the calling frame. A frame is not finished until a return value appears next to it. The critical moment the lecture highlights: after the second call returns, the first call has already printed `123` and has produced all of `12 / 1 / 12`, but it has *not yet* printed its final `123`. That last print is still pending in a half-finished frame.

### 4. Falling off the end of a body returns `None`

`cascade` has no `return` statement anywhere. When Python reaches the end of a function body without executing a `return`, the call returns `None`. So each `cascade` frame gets return value `None`, and the expression statement `cascade(n // 10)` evaluates to `None`, which is simply discarded. The function is useful entirely for its side effect (printing).

### 5. Shorter is not automatically better

The lecture shows two correct versions of `cascade` and argues explicitly that programs are written for other people to read and only incidentally for a computer to execute. Prefer the shorter version when the two are *equally* clear; here the longer version is arguably clearer because it lays out base case first, recursive case second, which is the conventional shape for reading recursive functions. Recommendation for students learning recursion: write the explicit base-case / recursive-case form, but both are legitimately recursive functions.

### 6. Tree recursion

**Tree recursion happens when one function makes more than one recursive call in its body.** The resulting computational process is tree-shaped: the original call branches into several subcalls, each of which branches again, down to base cases at the leaves. Values flow back up from the leaves to the root. Contrast with the linear recursion of `cascade` or `factorial`, where the process is a single chain of frames.

### 7. Tree recursion can repeat work

`fib(5)` computes `fib(3)` twice (once directly as `fib(n-2)`, once inside `fib(4)`), and the repetition compounds: `fib(35)` takes so long that the lecture literally waits on it live. The problem is not that tree recursion is inherently slow; it is that *this particular* decomposition recomputes identical subproblems. The lecture promises the fix (memoization, remembering computed values) "in a few weeks" and notes there are faster ways to compute Fibonacci numbers anyway.

### 8. Tree recursion as exploring choices

The `count_partitions` framing is the conceptual heart of the lecture: **tree recursion is a technique for exploring different possibilities.** Split the set of things you want to count into disjoint subsets based on a choice (use at least one part of size `m`, or use no part of size `m` at all), solve each smaller subproblem recursively, and then combine (here, sum, because we want the total count across all alternatives). Because the two subsets are disjoint and together cover everything, summing gives the correct total with no double counting.

---

## Definitions

- **Recursive function**: a function whose body calls the function itself (directly or indirectly).
- **Base case**: a case handled without any recursive call, where the result is returned or produced directly.
- **Recursive case**: a case whose result is expressed in terms of one or more simpler instances of the same problem.
- **Linear recursion**: recursion in which each call makes at most one recursive call, producing a chain of frames.
- **Tree recursion**: recursion in which executing the body of the function makes **more than one** call to that same function, producing a tree-shaped computational process.
- **Recursive decomposition**: expressing a problem in terms of simpler instances of the same kind of problem.
- **Fibonacci sequence**: indexed from 0, with `fib(0) = 0`, `fib(1) = 1`, and `fib(n) = fib(n-2) + fib(n-1)` for larger `n`. Values: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, ... and `fib(35) = 9227465`.
- **Partition of a positive integer `n` using parts up to size `m`**: a way of expressing `n` as a sum of positive parts, each of size at most `m`, written in increasing order. Order does not create new partitions: `2 + 4` counts, `4 + 2` is the same partition and is not counted again.
- **`count_partitions(n, m)`**: the *number* of such partitions, not a list of them.
- **Decorator**: syntax written as `@name` on the line just before a `def`, which changes the behavior of the function being defined. In this lecture, `@trace` (imported from the course's `ucb` module) wraps a function so it prints each call and each return value, indented by depth.
- **Implicit `None` return**: the value returned when a function body finishes without executing a `return` statement.

---

## Worked Examples

### Example 1: `cascade`, the explicit version

```python
def cascade(n):
    """Print a cascade of prefixes of n."""
    if n < 10:
        print(n)
    else:
        print(n)
        cascade(n // 10)
        print(n)
```

```
>>> cascade(5)
5
>>> cascade(12345)
12345
1234
123
12
1
12
123
1234
12345
```

**Why it produces that shape.** Each call does three things in order: print `n`, run the entire smaller cascade to completion, print `n` again. Because the middle step must finish before the third step starts, all of the smaller output is sandwiched between the two copies of `n`. The first and last lines both come from the same call (the original one). The only line printed exactly once is the base-case line (`1`), which is the only output produced by the `if` branch.

**Note on the digits.** `n // 10` is floor division: it drops the last digit. `12345 // 10` is `1234`. Using `/` would produce a float (`1234.5`) and break everything.

### Example 2: Environment reasoning for `cascade(123)`

Walk through it in words, matching the lecture's Python Tutor demo:

1. `def cascade(n)` binds the name `cascade` in the global frame to a function value whose parent is the global frame.
2. `cascade(123)` opens **frame f1** with `n = 123`. Its parent is the global frame. `123 >= 10`, so we take the `else` branch. `print(123)` runs. Output so far: `123`.
3. `cascade(123 // 10)` is evaluated. The operand `123 // 10` evaluates to `12`, then **frame f2** opens with `n = 12`, parent global. `print(12)` runs. Output: `123`, `12`. Frame f1 is now paused mid-statement.
4. `cascade(12 // 10)` opens **frame f3** with `n = 1`. `1 < 10`, so `print(1)` runs and the body ends. f3's return value is `None`.
5. Control returns into f2, which discards the `None` and executes its final `print(n)` with `n = 12`. Output gains a second `12`. f2 ends, return value `None`.
6. Control returns into f1, which executes its final `print(n)` with `n = 123`. Output gains a second `123`. f1 ends, return value `None`.

Final output: `123, 12, 1, 12, 123`. Three frames, three distinct bindings of `n`, and each `n` is looked up in *its own* frame, which is why the "on the way up" prints show the right values. Nothing is shared or mutated; each frame remembers its own `n` across the recursive call.

### Example 3: `cascade`, the shorter version

```python
def cascade(n):
    print(n)
    if n >= 10:
        cascade(n // 10)
        print(n)
```

This factors out the `print(n)` that both branches shared and moves it to the top, which leaves the base case with nothing to do, so the `if` now tests only for the recursive case. It produces identical output. The lecture's verdict: both are recursive, both are fine, the longer one is clearer to many readers because it names the base case and recursive case separately in that order, and when two versions are equally clear you should prefer the shorter one because it costs the reader less time.

(Note: the transcript says "when n is greater than 10"; the correct condition is `n >= 10`, matching the original's `n < 10` base case.)

### Example 4: `inverse_cascade` with higher-order functions

Goal: `inverse_cascade(1234)` prints

```
1
12
123
1234
123
12
1
```

The structure given in lecture:

```python
def inverse_cascade(n):
    grow(n)
    print(n)
    shrink(n)

def f_then_g(f, g, n):
    if n:
        f(n)
        g(n)

grow = lambda n: f_then_g(grow, print, n // 10)
shrink = lambda n: f_then_g(print, shrink, n // 10)
```

**Reading it.**

- `f_then_g(f, g, n)` does nothing when `n` is `0`, because `0` is the only false number, so `if n:` is the base case test. Otherwise it calls `f(n)` and then `g(n)`, in that order.
- `grow(n)` is supposed to print everything *smaller* than `n`, increasing. It passes `n // 10` (strictly smaller) and asks for `grow` first, `print` second: grow all the way down to nothing, then print on the way back up, which yields increasing order. That is the "do the small stuff first, then the big stuff" argument from lecture.
- `shrink(n)` is supposed to print everything smaller than `n`, decreasing. It also passes `n // 10`, but asks for `print` first, `shrink` second: print immediately on the way down, so larger values come out before smaller ones.
- `inverse_cascade` itself prints the longest line in the middle, between the growing half and the shrinking half.

**Why `n // 10` is passed in both cases**: `grow(n)` and `shrink(n)` must *not* print `n` itself (that is `inverse_cascade`'s job), so they start from the prefix one digit shorter.

**Trace for `inverse_cascade(123)`**: `grow(123)` calls `f_then_g(grow, print, 12)`, which calls `grow(12)` first, which calls `f_then_g(grow, print, 1)`, which calls `grow(1)` first, which calls `f_then_g(grow, print, 0)` and does nothing. Unwinding: print `1`, then print `12`. Back in `inverse_cascade`, print `123`. Then `shrink(123)` calls `f_then_g(print, shrink, 12)`, printing `12` and then recursing to print `1`. Result: `1, 12, 123, 12, 1`.

### Example 5: `fib`, the tree recursive Fibonacci

```python
def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n - 2) + fib(n - 1)
```

Two base cases and one recursive case containing **two** calls to `fib`, which is exactly what makes it tree recursive.

**The shape of the process for `fib(5)`**: `fib(5)` needs `fib(3)` and `fib(4)`. `fib(3)` needs `fib(1)` and `fib(2)`; `fib(2)` needs `fib(0)` and `fib(1)`. `fib(4)` needs `fib(2)` and `fib(3)`, and that `fib(3)` expands all over again. Only calls at `fib(0)` and `fib(1)` return directly with no further recursion; those are the leaves.

**Order of return values.** Because the `+` operator's left operand is evaluated first, `fib(n-2)` is fully computed before `fib(n-1)` is even started. The first return value ever reached inside `fib(5)` is the `1` from `fib(1)` at the bottom left.

**Using the `trace` decorator** (from the course's `ucb` module, provided with the project):

```python
from ucb import trace

@trace
def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n - 2) + fib(n - 1)
```

`fib(5)` then prints an indented trace whose shape is exactly the tree:

```
fib(5):
    fib(3):
        fib(1):
        fib(1) -> 1
        fib(2):
            fib(0):
            fib(0) -> 0
            fib(1):
            fib(1) -> 1
        fib(2) -> 1
    fib(3) -> 2
    fib(4):
        fib(2):
            fib(0):
            fib(0) -> 0
            fib(1):
            fib(1) -> 1
        fib(2) -> 1
        fib(3):
            fib(1):
            fib(1) -> 1
            fib(2):
                fib(0):
                fib(0) -> 0
                fib(1):
                fib(1) -> 1
            fib(2) -> 1
        fib(3) -> 2
    fib(4) -> 3
fib(5) -> 5
```

(The exact indentation and formatting of `ucb.trace` output may differ slightly; the structure and ordering are what matter.)

**The inefficiency.** In the trace above, `fib(3)` appears twice and `fib(2)` three times, each time recomputed from scratch. Scaling up: `fib(10)` contains a full `fib(9)` and a full `fib(8)`; `fib(15)` produces lines running off the edge of the screen; `fib(35)` (which is 9227465) takes noticeably long. The lecture's takeaway is *not* "tree recursion is slow" but "this decomposition repeats itself, and remembering results would fix it."

### Example 6: `count_partitions`

**The problem.** `count_partitions(6, 4)` should return `9`. The nine partitions of 6 using parts of size at most 4, each written in increasing order:

```
1 + 1 + 1 + 1 + 1 + 1
1 + 1 + 1 + 1 + 2
1 + 1 + 2 + 2
2 + 2 + 2
1 + 1 + 1 + 3
1 + 2 + 3
3 + 3
1 + 1 + 4
2 + 4
```

`4 + 2` is not counted separately from `2 + 4` (increasing order only), and `1 + 5` is excluded entirely because 5 exceeds the maximum part size 4.

**The recursive decomposition.** Split every partition of 6 with parts up to 4 into two disjoint groups:

1. Those that **use at least one 4**. Remove one 4; what remains is a partition of `6 - 4 = 2` using parts up to size 4 (still 4, because we are allowed to use another 4 later). So this group has `count_partitions(2, 4)` members. Those are `2` and `1 + 1`, giving 2 partitions, which correspond to `2 + 4` and `1 + 1 + 4`.
2. Those that **use no 4 at all**, meaning every part is at most 3. That group has `count_partitions(6, 3)` members, which is the remaining 7.

`2 + 7 = 9`. The two groups are disjoint and exhaustive, so summing is correct.

The same split then applies recursively: `count_partitions(6, 3)` splits into those using at least one 3 (`count_partitions(3, 3)`) and those using none (`count_partitions(6, 2)`), and so on.

**The implementation.**

```python
def count_partitions(n, m):
    """Count the ways to partition n using parts up to size m."""
    if n == 0:
        return 1
    elif n < 0:
        return 0
    elif m == 0:
        return 0
    else:
        with_m = count_partitions(n - m, m)
        without_m = count_partitions(n, m - 1)
        return with_m + without_m
```

**Justifying every line.**

- `n == 0` returns `1`: there is exactly one way to partition 0, namely by adding nothing together. This is the base case that actually *counts* a successful partition, so it has to return 1, not 0.
- `n < 0` returns `0`: we overshot by subtracting a part larger than what remained. Negative parts are not allowed, so this path contributes nothing.
- `m == 0` returns `0`: you cannot build a positive number out of parts of size 0, so if we have run out of allowed part sizes with `n` still positive, there are no partitions down this path.
- Recursive case: `with_m` keeps `m` available (you may reuse a part of size `m`) but shrinks `n`. `without_m` keeps `n` the same but permanently forbids `m` and everything larger. Every recursive call strictly decreases either `n` or `m`, which is why the process terminates.
- For `(6, 4)`: `with_m` is `count_partitions(2, 4)` and `without_m` is `count_partitions(6, 3)`.

**Walking the smaller live example, `count_partitions(5, 3) = 5`.** The five partitions are `1+1+1+1+1`, `1+1+1+2`, `1+2+2`, `1+1+3`, `2+3`.

- `count_partitions(5, 3)` = `count_partitions(2, 3)` + `count_partitions(5, 2)`.
- `count_partitions(2, 3)` = `count_partitions(-1, 3)` + `count_partitions(2, 2)` = `0 + 2` = `2`. The 0 is because you cannot fit a part of size 3 into a remaining total of 2. The 2 counts `2` and `1 + 1`. Adding the 3 back gives `2 + 3` and `1 + 1 + 3`.
- `count_partitions(5, 2)` = `count_partitions(3, 2)` + `count_partitions(5, 1)` = `2 + 1` = `3`. The 2 covers the partitions of 5 that use at least one 2 (`1 + 2 + 2` and `1 + 1 + 1 + 2`), and the 1 is the all-ones partition `1 + 1 + 1 + 1 + 1`, which is the only partition of 5 with parts limited to size 1.
- Total: `2 + 3 = 5`.

**Why this example matters.** The lecture calls it "a tree recursive process that is actually quite hard to write without tree recursion." Unlike `fib`, there is no obvious loop-based formula; the branching structure of the recursion *is* the enumeration of choices.

---

## Common Pitfalls

1. **Assuming the recursive call happens "last."** Any statement can come before or after a recursive call, and the two schedules produce opposite orderings of output. If you move `print(n)` from after `cascade(n // 10)` to before it, you get a different sequence entirely.

2. **Forgetting that a call must fully return first.** The whole subtree of work generated by `cascade(n // 10)` or `fib(n - 2)` completes before the next statement or the right operand of `+` begins.

3. **Expecting a return value when there is no `return`.** `cascade` returns `None` from every call. Writing `print(cascade(123))` would print all the cascade output and then a final `None`.

4. **Using `/` instead of `//`.** `n / 10` produces a float, which breaks both the digit-dropping logic and the base-case comparisons.

5. **Confusing "tree recursion" with "recursion on a tree data structure."** Tree recursion means the *process* is tree-shaped because the body makes more than one recursive call. It has nothing to do with the tree data type.

6. **Concluding that tree recursion is inherently slow.** `fib` is slow because it recomputes identical subproblems, not because it branches. Many tree recursive processes (including `count_partitions`, which genuinely explores distinct possibilities) are not doing redundant work in the same way. The general fix for redundancy, memoization, comes later in the course.

7. **`count_partitions` base cases in the wrong order or with the wrong values.** Returning `0` for `n == 0` makes the whole function return 0. Checking `m == 0` before `n == 0` mishandles the `count_partitions(0, 0)` case (it should be 1, one way to make 0 with nothing). Checking `n < 0` after the recursive call, or not at all, causes runaway recursion.

8. **Passing the wrong `m` in the `with_m` branch.** It is `count_partitions(n - m, m)`, not `count_partitions(n - m, m - 1)`: after using one part of size `m`, you are still allowed to use another part of size `m`.

9. **Double counting by treating orderings as distinct.** The `m`-decreasing structure is exactly what enforces "increasing order" and prevents counting `2 + 4` and `4 + 2` as two partitions.

10. **In `f_then_g`, writing `if n > 0` versus `if n:`.** These behave the same for the non-negative inputs used here, but the lecture's point is that `0` is the only false number, so `if n:` is the base case test.

11. **Passing `n` rather than `n // 10` to `grow` or `shrink`.** That causes infinite recursion, since `n` would never shrink toward 0.

---

## Likely Exam Points

### 1. Predicting the exact output order of a recursion with work before and after the call

**Q.** What does this print?

```python
def f(n):
    if n > 0:
        print(n)
        f(n - 1)
        print(-n)

f(3)
```

**A.**
```
3
2
1
-1
-2
-3
```
The `print(n)` calls run on the way down in decreasing order; each `print(-n)` is deferred until the entire inner call returns, so they run on the way back up in increasing order of `n`, giving `-1, -2, -3`.

### 2. Counting frames / identifying which call produced which output

**Q.** When `cascade(12345)` is run, how many `cascade` frames are opened, how many lines are printed, and which call prints the line `1`?

**A.** 5 frames (`n` = 12345, 1234, 123, 12, 1). 9 lines printed (2 lines per non-base call times 4, plus 1 from the base case). The line `1` is printed by the fifth and final call, the only one that reaches the base case. (In general, a `k`-digit input opens `k` frames and prints `2k - 1` lines. Extra context: the counting formula is a natural extension, not stated verbatim in lecture.)

### 3. Identifying tree recursion

**Q.** Which of these are tree recursive? (a) `cascade`, (b) `fib`, (c) `count_partitions`, (d) `grow` as defined above.

**A.** (b) and (c). Tree recursion requires more than one call to the function from within its own body. `cascade` makes exactly one. `grow` makes one call to `grow` (via `f_then_g`), so it is linear recursion, not tree recursion, even though it is a higher-order construction.

### 4. Tracing `fib` and reasoning about repeated work

**Q.** In the full computation of `fib(5)` using the lecture's definition, how many times is `fib(2)` called, and what is the first return value produced?

**A.** `fib(2)` is called three times: once inside the first `fib(3)`, once directly inside `fib(4)`, and once inside the `fib(3)` nested in `fib(4)`. The first return value reached is `1`, from the `fib(1)` at the far bottom-left, because `fib(n-2)` is evaluated before `fib(n-1)` and the recursion descends leftward first.

**Q (follow-up).** Why does `fib(35)` take so long?

**A.** Because the tree recursive process recomputes the same subproblems exponentially many times; there is no memory of previously computed values. (Extra context: the number of calls is `2 * fib(n+1) - 1`, which grows exponentially in `n`. The lecture shows this empirically rather than deriving the count.)

### 5. Base cases of `count_partitions`

**Q.** Explain why `count_partitions(0, m)` returns 1 and why `count_partitions(n, 0)` returns 0 for positive `n`. What breaks if the first returns 0 instead?

**A.** Reaching `n == 0` means the parts chosen so far sum exactly to the original total, which is one complete, valid partition, so it contributes 1 to the count. Reaching `m == 0` with `n` still positive means no allowable part sizes remain and the remaining amount cannot be built, so it contributes 0. If `n == 0` returned 0, then every leaf of the tree would return 0 and the function would return 0 for all inputs.

### 6. Hand-evaluating `count_partitions`

**Q.** Compute `count_partitions(4, 3)` by hand using the recursion, and list the partitions to check.

**A.** `cp(4,3) = cp(1,3) + cp(4,2)`.
`cp(1,3) = cp(-2,3) + cp(1,2) = 0 + [cp(-1,2) + cp(1,1)] = 0 + [0 + (cp(0,1) + cp(1,0))] = 0 + [0 + (1 + 0)] = 1`.
`cp(4,2) = cp(2,2) + cp(4,1)`. `cp(2,2) = cp(0,2) + cp(2,1) = 1 + 1 = 2`. `cp(4,1) = 1`. So `cp(4,2) = 3`.
Total: `1 + 3 = 4`. Check: `1+1+1+1`, `1+1+2`, `2+2`, `1+3`. That is 4.

### 7. Filling in a `count_partitions` skeleton (very common exam format)

**Q.** Fill in the blanks:

```python
def count_partitions(n, m):
    if n == 0:
        return ____
    elif n < 0 or m == 0:
        return ____
    else:
        return count_partitions(____, ____) + count_partitions(____, ____)
```

**A.** `1`; `0`; then `n - m, m` and `n, m - 1`. The first recursive call uses at least one part of size `m` (so `m` stays available), the second forbids `m` entirely.

### 8. Modifying the partition recursion

**Q.** How would you change `count_partitions` so that each part size may be used **at most once** (distinct parts)?

**A.** Change the first recursive call to `count_partitions(n - m, m - 1)`, since after using the single allowed part of size `m` you must move on to strictly smaller parts. (Extra context: this variant was not shown in lecture, but it is the standard way exam questions probe whether you understand why `m` is kept in the original.)

### 9. Higher-order recursion (`inverse_cascade`)

**Q.** If you swapped the definitions to `grow = lambda n: f_then_g(print, grow, n // 10)`, what would `inverse_cascade(123)` print?

**A.** `12`, `1`, `123`, `12`, `1`. The "grow" half would now print on the way down instead of on the way up, producing decreasing order for both halves and destroying the intended symmetric shape.

### 10. Return value of a function with no `return`

**Q.** What is the value of the expression `cascade(7)` (as opposed to its output)?

**A.** `None`. It prints `7` as a side effect, but the body ends without a `return`, so the call returns `None`.

---

## Summary

- A function call must **return** before any statement after it can execute; this is the whole key to reading the order of recursive output.
- Statements placed **before** a recursive call run in top-down order (largest to base case); statements placed **after** run in bottom-up order (base case back to largest).
- `cascade(n)` prints `n`, recurses on `n // 10`, and prints `n` again, producing a nested symmetric cascade; only the base-case line is printed exactly once.
- Each recursive call gets its **own frame** with its own binding of `n`; a frame is unfinished until a return value appears beside it. All `cascade` frames have the global frame as parent.
- A body that ends without a `return` statement returns **`None`**.
- Two correct implementations can differ in readability: prefer the shorter one when they are equally clear, but the explicit base-case-then-recursive-case form is recommended while learning. Write programs for people to read.
- `inverse_cascade` builds the increasing-then-decreasing pattern from `grow` (recurse then print) and `shrink` (print then recurse), both expressed through the higher-order helper `f_then_g(f, g, n)`, which does nothing when `n` is `0`.
- **Tree recursion**: the body makes more than one call to the function itself, producing a tree-shaped process with base cases at the leaves.
- `fib(n) = fib(n-2) + fib(n-1)` with base cases `fib(0) = 0`, `fib(1) = 1` is tree recursive and extremely redundant: `fib(3)` is computed twice inside `fib(5)`, and `fib(35) = 9227465` takes a long time. The fix (remembering results) comes later; tree recursion is not slow by nature.
- The `@trace` decorator from the `ucb` module prints each call and its return value, making the tree structure and its ordering directly visible.
- `count_partitions(n, m)` counts ways to write `n` as an increasing sum of parts of size at most `m`. `count_partitions(6, 4)` is 9; `count_partitions(5, 3)` is 5.
- Its recursion splits all partitions into two disjoint groups and sums them: `count_partitions(n - m, m)` (uses at least one `m`) plus `count_partitions(n, m - 1)` (uses no `m`).
- Base cases: `n == 0` returns 1 (the empty sum is one valid partition), `n < 0` returns 0, `m == 0` returns 0.
- Big idea: tree recursion is a technique for **exploring alternatives**, and some problems (like counting partitions) are genuinely hard to express without it.
