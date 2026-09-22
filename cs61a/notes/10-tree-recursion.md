<!-- Fri, Sep 18, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 10: Tree Recursion

This lecture completes the recursion arc of CS 61A by moving from single recursive calls to functions that call themselves more than once. It opens with a review of how to *reason* about recursion (tracing versus the "recursive leap of faith" / induction) and how to *get started* writing a recursive function (write the recursive case first, then discover the base cases). It then works through several single-recursion examples (`streak`, `hailstone`, `sevens`, `cascade`), introduces **mutual recursion**, where two functions call each other (`hailstone`/`even`/`odd`, and `unique_prime_factors`/`no_k`), and finally arrives at **tree recursion**: whenever the body of a function makes more than one call to itself, the computational process branches into a tree. The two headline tree-recursive examples are the naive Fibonacci function (which is correct but does an enormous amount of repeated work) and `count_partitions`, a problem that is genuinely hard to write without recursion and that illustrates the central tree-recursion idea: *explore two possibilities, solve each as a simpler instance of the same problem, and combine the results.*

---

## Key Concepts

### 1. Verifying a recursive function: tracing versus induction

There are two ways to convince yourself a recursive function is right:

- **Tracing**: diagram the entire computational process, frame by frame. This is only feasible for very small inputs. It is what Python Tutor does for you, and it is what exam environment-diagram questions ask for.
- **Induction (the recursive leap of faith)**: check that `f(n)` produces the right answer *assuming* `f(n-1)`, `f(n-2)`, ..., `f(0)` already work. You do not re-trace the recursive call; you trust its contract (its docstring).

The leap of faith is just **functional abstraction** applied to a function you happen to be in the middle of writing. When you read `streak(n // 10)`, you should read it as "whether the number with the last digit removed is a streak," not as "a bunch of frames I now have to simulate."

### 2. How to get started writing a recursive function

The lecture's recommended order is the opposite of how the finished code reads:

1. **Start with the recursive case.** Pick a concrete input, write down the recursive calls you *might* make, ask what they would return, and figure out how to combine those return values into the answer. For `streak(22222)`, ask: what would `streak(2222)` give me, and what else do I need?
2. **Then find the base case(s).** Once the recursive case exists, trace it on small inputs and see where it bottoms out. The base cases are whatever inputs the recursion eventually reaches and cannot decompose further.
3. **Describe the recursion in English first.** For `streak`: "In a streak, every digit except the last is a streak, and the last digit matches the second-to-last." That single sentence is the whole implementation.
4. **Convert from iteration if that is easier for you.** If you can write it with `while`, write that first, then mechanically turn the loop variables into parameters of a helper function. This is exactly what the lecture does with `sevens_iter` → `sevens`.

### 3. Order of operations: statements before and after the recursive call

A function call must **return before anything after it can happen**. This is the single most important fact for reading recursive output. In `cascade`, the `print(n)` before the recursive call runs on the way *down*, and the `print(n)` after it runs on the way *back up*, which is what produces the nested, symmetric shape. A frame stays open (has no return value yet) for the entire duration of the recursive call it made.

Also from the `cascade` trace: a function body that reaches its end without executing a `return` statement returns `None`. `cascade` returns `None` from every frame; all of its visible effect comes from `print`.

### 4. Mutual recursion

Two functions `f` and `g` are **mutually recursive** if `f` calls `g` and `g` calls `f`. This is still recursion: the same leap of faith applies, but now the "smaller problem" may be handed to a different function. The shape is useful when a problem naturally alternates between two states (even/odd in `hailstone`) or two phases (strip out one prime factor / move on to the next in `unique_prime_factors`).

### 5. Tree recursion

> **Tree recursion happens when the body of a function makes more than one call to itself.**

The resulting *process* is tree-shaped: the top-level call branches into two (or more) subcalls, each of which branches again, until base cases form the leaves. Computing `fib(5)` requires computing `fib(3)` and `fib(4)`; `fib(3)` requires `fib(1)` and `fib(2)`; and so on.

Two big ideas come with it:

- **Recursive decomposition**: express the problem as a combination of *simpler instances of the same problem*. For partitions, the two instances are "use at least one part of size `m`" and "use no part of size `m`."
- **Tree recursion as exploring choices**: each branch represents a decision. Because the branches partition the possibilities into **disjoint** sets that together cover everything, you can just **add** the counts. Missing cases or overlapping cases are the classic bug.

### 6. Tree recursion and repeated work

Naive `fib` is slow: `fib(35)` takes noticeably long, because the same subproblems are recomputed many times (`fib(3)` appears twice inside `fib(5)`, and the duplication explodes as `n` grows). The lecture is explicit about two caveats:

- There are faster ways to compute Fibonacci numbers.
- **Tree-recursive processes are not inherently slow.** This particular one is, because of redundancy. The fix (remembering computed values, i.e. memoization) comes later in the course.

### 7. The `trace` decorator

`from ucb import trace`, then put `@trace` on the line just before a `def`. It changes the function's behavior so that every call and every return value is printed, indented by depth. Applying it to `fib` makes the tree-structured process visible: `fib(5)` shows `fib(3)` and `fib(4)` subtrees, and `fib(15)` produces lines that run off the edge of the screen. It is provided with the projects and is a legitimate debugging tool for future work.

---

## Definitions

- **Recursive function**: a function whose body contains a call to itself (directly, or indirectly through other functions).
- **Base case**: a case handled without any recursive call; it terminates the recursion.
- **Recursive case**: a case that reduces the problem to one or more simpler instances of the same problem and combines their results.
- **Recursive leap of faith**: the inductive reasoning step of assuming the recursive calls on smaller inputs are already correct, and verifying only that the current call combines them correctly.
- **Tracing**: diagramming the full computational process (frames, calls, returns) to verify behavior; practical only for small inputs.
- **Mutually recursive functions**: functions `f` and `g` such that `f` calls `g` and `g` calls `f`.
- **Tree recursion**: recursion in which executing the body of a function makes **more than one** call to that same function, producing a tree-shaped computational process.
- **Recursive decomposition**: finding simpler instances of the same problem that together let you solve the original.
- **Partition of a positive integer `n` using parts up to size `m`**: a way of writing `n` as a sum of positive integers, each at most `m`, written in non-decreasing order. Order does not distinguish partitions: `2 + 4` counts, `4 + 2` is the same partition written in decreasing order and is not counted separately.
- **Dice integer**: a positive integer all of whose digits are from 1 to 6.
- **Hailstone sequence**: from `n`, repeatedly halve if even and apply `3n + 1` if odd, until reaching 1.
- **Decorator (`@trace`)**: syntax placing a function just above a `def` that transforms the function being defined; `trace` wraps it so calls and returns are printed.

---

## Worked Examples

### Example 1: `cascade` and the order of recursive calls

```python
def cascade(n):
    if n < 10:
        print(n)
    else:
        print(n)
        cascade(n // 10)
        print(n)
```

```
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

**What happens, in environment-diagram terms.** Call `cascade(123)` from the global frame. Frame f1 binds `n` to 123; its parent is the global frame (because `cascade` was defined there). `123 < 10` is false, so we print `123`: program output so far is just `123`. Next we evaluate `cascade(12)`, creating frame f2. **Frame f1 is now suspended and has no return value**; its final `print(n)` has not run. f2 prints `12` and calls `cascade(1)`, creating f3. `1 < 10`, so f3 prints `1` and reaches the end of the body with no `return` statement, so it returns `None`.

Now we unwind. Control returns into the middle of f2, at the line `cascade(n // 10)`, whose value turned out to be `None` (and is discarded). f2 then executes its trailing `print(n)`, printing `12` again, and returns `None`. Control returns into f1, which executes its trailing `print(n)`, printing `123`. Total output: `123, 12, 1, 12, 123`.

**Key readings of the diagram:**
- Each frame corresponds to one call; a frame's work after the recursive call is strictly "later" than the entire subcomputation.
- The third call printed the `1`; the second call printed *both* `12`s, one before and one after; the first call printed both `123`s.

**A shorter version** (the lecture compares the two):

```python
def cascade(n):
    print(n)
    if n >= 10:
        cascade(n // 10)
        print(n)
```

Same output, fewer lines, because the common `print(n)` is factored out of both cases. The lecture's verdict: if two implementations are equally clear, prefer the shorter one, since it takes a reader less time. But here the longer one is arguably clearer because it explicitly separates base case from recursive case in that order, which is the conventional shape for recursive functions and is the style recommended while you are learning. The broader principle: **programs are written for people to read, and only incidentally for machines to execute.**

### Example 2: `inverse_cascade` (in-lecture exercise)

Goal: `inverse_cascade(1234)` prints `1, 12, 123, 1234, 123, 12, 1`. The structure given in lecture:

```python
def inverse_cascade(n):
    grow(n)
    print(n)
    shrink(n)

def f_then_g(f, g, n):
    if n:
        f(n)
        g(n)
```

Note `if n:` uses `0` as the only false number, so the recursion stops when the number is reduced to `0`. The answer, using higher-order functions:

```python
grow = lambda n: f_then_g(grow, print, n // 10)
shrink = lambda n: f_then_g(print, shrink, n // 10)
```

(The exact lambda formulation above is the standard textbook phrasing of the answer the lecture describes verbally.)

**Why these work.** `grow` **grows first, then prints**: it does the small stuff before the big stuff, so the numbers come out increasing (`1`, then `12`, then `123`). `shrink` **prints first, then shrinks**: the large number comes out before the smaller ones, giving decreasing order. Each passes `n // 10` so the next level handles one fewer digit. This is the same before/after-the-recursive-call idea as `cascade`, but parameterized by which function goes first.

### Example 3: `streak` (Spring 2024 Midterm 1, Q4e)

```python
def streak(n):
    """Return whether positive n is a dice integer in which all the digits are the same.

    >>> streak(22222)
    True
    >>> streak(4)
    True
    >>> streak(22322)  # 2 and 3 are different digits.
    False
    >>> streak(99999)  # 9 is not allowed in a dice integer.
    False
    >>> streak(505)
    False
    >>> streak(707)
    False
    >>> streak(7070)
    False
    >>> streak(33333333333333)
    True
    """
    return (n >= 1 and n <= 6) or (n >= 10 and n % 10 == n // 10 % 10 and streak(n // 10))
```

(The slide writes the second guard as `n > 9`, which is the same condition as `n >= 10`.)

**The process, in English**: *in a streak, every digit except the last is a streak, and the last digit matches the one before it.*

**Reading it piece by piece:**
- `(n >= 1 and n <= 6)`: the base case, a single digit from 1 to 6. This is the *only* place the "digits must be 1 through 6" rule is enforced.
- `n >= 10`: we only recurse on multi-digit numbers; a single digit outside 1-6 falls through to `False`.
- `n % 10 == n // 10 % 10`: the last digit equals the second-to-last digit.
- `streak(n // 10)`: the leap of faith, everything except the last digit is itself a streak.

**Why one base-case digit check suffices**: since each step verifies that adjacent digits are equal, by the time we reach a single digit all the digits were equal to it. Checking that final digit is in 1-6 therefore checks them all. That is why `streak(99999)` is `False`: the recursion happily peels 9s off (each matches its neighbor) until it hits `streak(9)`, where `9 >= 1 and 9 <= 6` is false and `9 >= 10` is false, giving `False`, which propagates back up through the `and`s.

**Short-circuiting matters**: `or` returns as soon as the first operand is truthy, and `and` stops at the first falsy operand, so `streak(505)` never recurses at all (`5 != 0` short-circuits the `and` before the recursive call).

### Example 4: `hailstone` via mutual recursion

```python
def hailstone(n):
    """Print out the hailstone sequence starting at n, and return the
    number of elements in the sequence."""
    print(n)
    if n % 2 == 0:
        return even(n)
    else:
        return odd(n)

def even(n):
    return hailstone(n // 2) + 1

def odd(n):
    if n == 1:
        return 1
    return hailstone(n * 3 + 1) + 1
```

```
>>> a = hailstone(10)
10
5
16
8
4
2
1
>>> a
7
```

**How to read it.** `hailstone` does not call itself directly. It calls `even` or `odd`, and each of those calls `hailstone` back: that is mutual recursion (a cycle of length 2). The `print(n)` happens on the way down, so the sequence prints in order. The `+ 1` happens on the way *back up*: each level adds itself to the count returned by the rest of the sequence.

**The base case lives in `odd`**, not in `hailstone`: `n == 1` is odd, and it returns `1` (the sequence `1` has one element) without recursing. Counting for `n = 10`: the seven printed values are 10, 5, 16, 8, 4, 2, 1, and the six `+ 1`s applied to the innermost `return 1` give 7.

### Example 5: `sevens`, converting iteration to recursion

The game: players sit in a circle counting up from 1 clockwise. If a number is divisible by 7 **or** contains the digit 7 (or both), reverse direction. `sevens(n, k)` returns the position of the player who says `n` among `k` players.

The lecture's recipe for designing this:
1. Pick a concrete input/output: `sevens(18, 5)` should be `2`.
2. Describe the process in English with simple steps.
3. Figure out what **extra names** you need: the problem gives you `n` (the final number) and `k` (the number of players), but to carry out the process you also need `i` (the current number), `who` (the current player), and `direction` (which way the turn moves, `+1` or `-1`).
4. Implement using those names.

Step 3 is the crucial insight: the given parameters are not enough, so you introduce a helper whose parameters are exactly the state of the process.

```python
def has_seven(n):
    if n == 0:
        return False
    elif n % 10 == 7:
        return True
    else:
        return has_seven(n // 10)
```

**Iterative version:**

```python
def sevens_iter(n, k):
    i, who, direction = 1, 1, 1
    while i < n:
        if i % 7 == 0 or has_seven(i):
            direction = -direction
        who = who + direction
        if who > k:
            who = 1
        if who < 1:
            who = k
        i = i + 1
    return who
```

**Recursive version (the same process, loop variables turned into parameters):**

```python
def sevens(n, k):
    def f(i, who, direction):
        if i == n:
            return who
        if i % 7 == 0 or has_seven(i):
            direction = -direction
        who = who + direction
        if who > k:
            who = 1
        if who < 1:
            who = k
        return f(i + 1, who, direction)
    return f(1, 1, 1)
```

**Environment reasoning.** `f` is defined inside `sevens`, so every frame for `f` has the `sevens` frame as its parent. That is how `f` sees `n` and `k` without taking them as parameters: the name lookup fails in the local frame and succeeds in the parent. Each recursive call makes a fresh `f` frame with its own `i`, `who`, `direction`; the assignments inside a frame (`direction = -direction`, `who = who + direction`) only rebind that frame's local names, then the updated values are *passed forward* as arguments. Nothing happens after the recursive call, so the innermost frame's return value passes straight back out through every frame unchanged. This is the general "iteration becomes recursion" pattern: **`while` condition becomes the base-case test (negated), loop body becomes the work before the call, and the loop update becomes the arguments to the recursive call.**

**Trace of `sevens(18, 5)`**, showing the direction flips:

| `i` | flip? | `who` after |
|---|---|---|
| 1-6 | no | 2, 3, 4, 5, 1, 2 |
| 7 | yes (divisible by 7) | 1 |
| 8-13 | no | 5, 4, 3, 2, 1, 5 |
| 14 | yes (divisible by 7) | 1 |
| 15, 16 | no | 2, 3 |
| 17 | yes (contains 7) | 2 |

At `i == 18` the base case fires and returns `who == 2`, matching the picture on the slide.

### Example 6: `unique_prime_factors` (mutual recursion)

```python
def smallest_factor(n):
    """Return the smallest divisor of n above 1."""
    def smallest_divisor(k):
        "Return the smallest divisor of n above or equal to k."
        if n % k == 0:
            return k
        else:
            return smallest_divisor(k + 1)
    return smallest_divisor(2)

def unique_prime_factors(n):
    """Return the number of unique prime factors of n.

    >>> unique_prime_factors(51)  # 3 * 17
    2
    >>> unique_prime_factors(9)   # 3 * 3
    1
    >>> unique_prime_factors(576) # 2 * 2 * 2 * 2 * 2 * 2 * 3 * 3
    2
    """
    k = smallest_factor(n)
    def no_k(n):
        "Return the number of unique prime factors of n other than k."
        if n == 1:
            return 0
        elif n % k != 0:
            return unique_prime_factors(n)
        else:
            return no_k(n // k)
    return 1 + no_k(n)
```

**The strategy in three moves:**
1. Find the smallest factor `k` above 1 (it is necessarily prime: any smaller factor of `k` would be a smaller factor of `n`).
2. Keep dividing `k` out until it no longer divides (`no_k` calling itself).
3. Count the prime factors of whatever remains (`no_k` calling `unique_prime_factors`), then add 1 for `k` itself.

`no_k` and `unique_prime_factors` are mutually recursive: `unique_prime_factors` calls `no_k`, and `no_k` calls `unique_prime_factors`. The `1 +` in `return 1 + no_k(n)` counts `k`, which is exactly the factor `no_k` is defined to ignore. Note that `no_k` captures `k` from its enclosing `unique_prime_factors` frame, and each new call to `unique_prime_factors` creates a *fresh* `no_k` with a *different* captured `k`.

The slide traces `72000` (`= 2**6 * 3**2 * 5**3`):

| count so far | what remains |
|---|---|
| 0 | 72000 |
| 1 (`k = 2`) | 1125 |
| 2 (`k = 3`) | 125 |
| 3 (`k = 5`) | 1 |

`smallest_factor` itself uses a nested helper `smallest_divisor` that recurses on `k + 1` while reading `n` from the parent frame, the same closure trick as `sevens`.

### Example 7: `fib`, the canonical tree recursion

```python
def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n - 2) + fib(n - 1)
```

Two base cases (`fib(0) = 0`, `fib(1) = 1`) and one recursive case containing **two** calls to `fib`, which is precisely what makes it tree recursive.

**Order of evaluation for `fib(5)`.** Python evaluates `fib(n - 2)` completely before starting `fib(n - 1)`. So `fib(5)` first descends into `fib(3)`, which descends into `fib(1)` (the very first return value produced anywhere in the computation, returning 1), then `fib(2)`, which computes `fib(0)` and `fib(1)` and returns 1; `fib(3)` returns `1 + 1 = 2`. Only then does `fib(5)` begin `fib(4)`, which unfolds its own subtree (`fib(2)` and `fib(3)`) and returns 3. Finally `fib(5)` returns `2 + 3 = 5`.

**The cost.** `fib(20)` is 6765, `fib(30)` is 832040, and `fib(35)` is 9227465 but takes a long time to compute this way. Notice in the tree that `fib(3)` is computed twice inside `fib(5)`, and the duplication compounds at every level: the same arguments are recomputed over and over. With `@trace` you can watch this directly: `fib(15)` produces lines too long for the screen. The fix (remember values you have already computed) comes later in the course; tree recursion itself is not the problem.

### Example 8: `count_partitions` (the main event)

**Definition.** The number of partitions of a positive integer `n` using parts up to size `m` is the number of ways `n` can be written as a sum of positive integer parts, each at most `m`, in non-decreasing order.

The complete list for `count_partitions(6, 4)`:

```
2 + 4 = 6
1 + 1 + 4 = 6
3 + 3 = 6
1 + 2 + 3 = 6
1 + 1 + 1 + 3 = 6
2 + 2 + 2 = 6
1 + 1 + 2 + 2 = 6
1 + 1 + 1 + 1 + 2 = 6
1 + 1 + 1 + 1 + 1 + 1 = 6
```

So `count_partitions(6, 4)` returns **9**. Two things are excluded on purpose: `1 + 5` (the part 5 exceeds `m = 4`) and `4 + 2` (that is `2 + 4` written in decreasing order, not a separate partition).

**The recursive decomposition.** Split every partition of 6 with parts up to 4 into two disjoint groups:
- Those that **use at least one 4**. Strip off one 4; what remains is a partition of `6 - 4 = 2` using parts up to 4. That is `count_partitions(2, 4)`, which is 2 (`2` and `1 + 1`).
- Those that **use no 4 at all**. Those are exactly the partitions of 6 using parts up to 3: `count_partitions(6, 3)`, which is 7.

Every partition falls into exactly one group and none is counted twice, so the answer is `2 + 7 = 9`. The same split applies recursively: `count_partitions(6, 3)` splits into those using a 3 (`count_partitions(3, 3)`) and those not (`count_partitions(6, 2)`), and so on. **Tree recursion is a technique for exploring different choices**, and here we sum the branch results because we want all the alternatives.

```python
def count_partitions(n, m):
    if n == 0:
        return 1   # We found a way
    elif n < 0:
        return 0   # We subtracted too much
    elif m == 0:
        return 0   # We ran out of numbers
    else:
        with_m = count_partitions(n - m, m)
        without_m = count_partitions(n, m - 1)
        return with_m + without_m
```

**Justifying each base case:**
- `n == 0` returns **1**: there is exactly one way to partition 0, namely by adding nothing. This is the "success" leaf, and it is what all the counting ultimately sums up.
- `n < 0` returns **0**: we subtracted a part larger than what was left, so this choice was impossible. Negative parts are not allowed, so this branch contributes nothing.
- `m == 0` returns **0**: with a maximum part size of 0 you cannot build a positive `n`. This is the "ran out of options" leaf.

**Ordering of the base cases matters.** `n == 0` is checked first, so `count_partitions(0, 0)` returns 1 rather than 0, which is what we want: reaching `n == 0` is a success no matter what `m` is.

**Traced example: `count_partitions(5, 3) == 5`.** The five partitions are `1+1+1+1+1`, `1+1+1+2`, `1+2+2`, `1+1+3`, `2+3`.

- `count_partitions(5, 3)` = `count_partitions(2, 3)` + `count_partitions(5, 2)`.
- `count_partitions(2, 3)` (partitions of 5 that use a 3, i.e. `1+1+3` and `2+3` after removing the 3) = `count_partitions(-1, 3)` + `count_partitions(2, 2)` = `0 + 2` = **2**. Trying to use a part of size 3 to partition 2 is impossible, hence the 0.
- `count_partitions(5, 2)` (partitions of 5 with parts of size 2 or less) = `count_partitions(3, 2)` + `count_partitions(5, 1)` = `2 + 1` = **3**. The 2 covers `1+2+2` and `1+1+1+2`; the 1 covers the all-ones partition, found by `count_partitions(5, 1)` peeling off 1s until it hits `n == 0`.
- Total: `2 + 3 = 5`. ✓

**Environment reasoning.** Each call creates its own frame with its own `n` and `m`, all parented to the global frame. The frame for `count_partitions(5, 3)` stays open, holding `with_m = 2`, while the entire subtree of `count_partitions(5, 2)` runs. The names `with_m` and `without_m` are local to each frame and do not interfere across frames, which is why recursion on the same function with different arguments is safe.

**Why this example matters**: it is a tree-recursive process that is quite hard to write without recursion. Solving problems like this is one of the reasons the course teaches recursion at all.

### Example 9: `count_park` (from the lecture code file)

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
        return 2 * count_park(n - 1) + count_park(n - 2)
```

**The decomposition.** Look at the leftmost spot. Either it holds something one spot wide (an empty space `.` or a motorcycle `%`), which is **2** choices leaving `n - 1` spots, or a car `<>` starts there and occupies **two** spots, leaving `n - 2`. Hence `2 * count_park(n - 1) + count_park(n - 2)`. The branches are disjoint (each arrangement has exactly one thing occupying the leftmost spot) and exhaustive, so we add.

Base cases mirror `count_partitions`: `n == 0` returns 1 (one way to fill zero spots, the empty arrangement), and `n < 0` returns 0 (a car overhung the end, an invalid path). Check: `count_park(2) = 2*2 + 1 = 5`, `count_park(3) = 2*5 + 2 = 12`, `count_park(4) = 2*12 + 5 = 29`. ✓ This is a Fibonacci-shaped tree recursion with a coefficient, and it shows the pattern generalizes beyond partitions.

---

## Common Pitfalls

1. **Re-tracing instead of trusting.** Trying to simulate the whole tree in your head while writing the recursive case. Use the leap of faith: assume the call on the smaller input is correct.
2. **Writing the base case first.** You usually cannot tell what the base cases should be until you know how the recursive case shrinks the problem. Write the recursive case, then trace down to find where it bottoms out.
3. **Forgetting that work after a recursive call happens later.** In `cascade`, moving `print(n)` from after the recursive call to before it destroys the output shape. Statements can go before *or* after a recursive call, and the difference is everything.
4. **Assuming a function returns something.** `cascade` returns `None` from every frame because there is no `return` statement. `print` is not `return`.
5. **Base cases in the wrong order.** In `count_partitions`, checking `m == 0` before `n == 0` would make `count_partitions(0, 0)` return 0 and undercount. Order your conditions so the "success" case wins.
6. **Branches that overlap or leave gaps.** Tree recursion only lets you add the results if the possibilities are disjoint and exhaustive. Partitions split into "uses at least one `m`" and "uses no `m`", which is a clean split. "Uses an `m`" and "uses parts up to `m`" would double count.
7. **Getting the "uses at least one m" recursion wrong.** It is `count_partitions(n - m, m)`, not `count_partitions(n - m, m - 1)`. Keeping `m` allows using several parts of size `m`.
8. **Not introducing the helper parameters you need.** `sevens(n, k)` cannot recurse on itself usefully; the process needs `i`, `who`, and `direction`. Ask "what state does the process carry?" and make those the helper's parameters.
9. **Assuming tree recursion is inherently slow.** It is not. Naive `fib` is slow because of *repeated subproblems*, not because of branching. `count_partitions` and `count_park` are tree recursive too.
10. **Missing short-circuit behavior in boolean-returning recursions.** In `streak`, the recursive call sits behind `and`, so it never runs when the digit check already failed. Relying on this (or forgetting it) changes which calls happen.
11. **Digit-extraction slips.** `n % 10` is the last digit; `n // 10` removes it; `n // 10 % 10` is the second-to-last digit. Mixing `%` and `//` up is a frequent exam error.
12. **Mutual recursion with no reachable base case.** In `hailstone`, the only base case lives in `odd` (`n == 1`). If you put a base case only where the recursion never lands, it loops forever.

---

## Likely Exam Points

### 1. Fill in the blank in a recursive function

Blanks in a recursive body (like the actual Spring 2024 Midterm 1 Q4e `streak` problem) are the bread and butter of 61A exams. Strategy: state the English process, then match it to the skeleton.

**Practice.** Fill in the blanks so `all_odd(n)` returns whether every digit of positive `n` is odd.

```python
def all_odd(n):
    """
    >>> all_odd(1357)
    True
    >>> all_odd(1354)
    False
    >>> all_odd(9)
    True
    """
    if n < 10:
        return _______
    return _______ and all_odd(_______)
```

**Answer.** `n % 2 == 1`; `n % 10 % 2 == 1`; `n // 10`. The process: the last digit is odd and everything except the last digit has all-odd digits. (Equivalently, one could write `return n % 2 == 1 and (n < 10 or all_odd(n // 10))`.)

### 2. Predicting the output and order of prints

**Practice.** What does this print?

```python
def mystery(n):
    if n > 0:
        print(n)
        mystery(n - 1)
        print(-n)
mystery(3)
```

**Answer.**
```
3
2
1
-1
-2
-3
```
The `print(n)` before the call runs on the way down (3, 2, 1); the `print(-n)` after the call runs on the way back up, innermost first (-1, -2, -3). At the deepest level, `mystery(0)` does nothing and returns `None`.

### 3. Identifying tree recursion and counting calls

**Practice.** Is the following tree recursive, and how many total calls to `fib` (including the original) are made by `fib(4)` using the lecture's definition?

**Answer.** Yes: the body `fib(n - 2) + fib(n - 1)` contains two calls to `fib`, which is the definition of tree recursion. Let `C(n)` be the number of calls. `C(0) = C(1) = 1`, and `C(n) = 1 + C(n-2) + C(n-1)`. So `C(2) = 3`, `C(3) = 1 + 1 + 3 = 5`, `C(4) = 1 + 3 + 5 = 9`. Nine calls. (Useful identity for the general case: `C(n) = 2 * fib(n + 1) - 1`.) (extra context: the closed-form identity was not stated in lecture.)

### 4. `count_partitions` variants

Exams love perturbations of `count_partitions`: parts must be distinct, parts must be even, at most `k` parts, etc.

**Practice (a).** What does `count_partitions(4, 2)` return, and what are the partitions?

**Answer.** 3: `1+1+1+1`, `1+1+2`, `2+2`. Via the recursion: `count_partitions(4,2) = count_partitions(2,2) + count_partitions(4,1)`; `count_partitions(2,2) = count_partitions(0,2) + count_partitions(2,1) = 1 + 1 = 2`; `count_partitions(4,1) = count_partitions(3,1) + count_partitions(4,0) = 1 + 0 = 1`. Total 3.

**Practice (b).** Change `count_partitions` so that no part may be used more than once (all parts distinct). Which single character changes?

**Answer.** Change `with_m = count_partitions(n - m, m)` to `count_partitions(n - m, m - 1)`. Having used one `m`, you may not use `m` again, so the largest allowed part drops. (Check: distinct partitions of 6 with parts up to 4 are `2+4`, `1+2+3`, and nothing else with max part 4 or less... plus none more, giving 2 ways; the modified function returns that.)

### 5. Why naive `fib` is slow, and what the fix is

**Practice.** Explain in two sentences why `fib(35)` takes a long time with the lecture's implementation, and name the remedy.

**Answer.** The tree-recursive process recomputes the same subproblems many times: `fib(3)` is evaluated twice inside `fib(5)`, and the duplication multiplies at every level, so the number of calls grows exponentially in `n`. The remedy is to remember the value computed for each argument and reuse it instead of recomputing (memoization, covered later in the course); tree recursion in general is not slow, only this redundant instance is.

### 6. Converting iteration to recursion (and back)

**Practice.** Convert this to a recursive function without using `while`:

```python
def summation(n):
    total, k = 0, 1
    while k <= n:
        total, k = total + k, k + 1
    return total
```

**Answer.** Turn the loop variables into helper parameters, the loop condition into the base-case test, and the update into the recursive call's arguments:

```python
def summation(n):
    def f(total, k):
        if k > n:
            return total
        return f(total + k, k + 1)
    return f(0, 1)
```

`f` reads `n` from its parent frame (the `summation` frame) rather than taking it as a parameter. (A non-accumulator version, `return n + summation(n - 1)` with base case `0`, is also valid and does its work *after* the recursive call.)

### 7. Mutual recursion

**Practice.** For `unique_prime_factors`, what does `no_k(n)` promise, and why does the outer function return `1 + no_k(n)` rather than `no_k(n)`?

**Answer.** `no_k(n)` returns the number of unique prime factors of `n` **other than `k`**, where `k` is the smallest factor of the original `n` above 1. Since `k` itself is one prime factor that `no_k` deliberately excludes, the outer function adds 1 for it. The mutual recursion appears in `no_k`'s middle branch: when `k` no longer divides `n`, the remaining number is handed back to `unique_prime_factors`, which finds *its* smallest factor and repeats.

### 8. Environment-diagram questions

**Practice.** When `sevens(18, 5)` is evaluated, how many frames exist at the deepest point, and what is the parent frame of each `f` frame?

**Answer.** One frame for `sevens` plus 18 frames for `f` (`i` running from 1 to 18 inclusive), so 19 frames beyond global at the deepest point. Every `f` frame's parent is the single `sevens` frame, because `f` was *defined* there; the parent is determined by where a function is defined, not by who called it. That is how each `f` frame looks up `n` and `k`.

---

## Summary

- **Verify recursion two ways**: trace small examples fully, or use induction (the recursive leap of faith), assuming smaller calls are correct and checking only the combination step.
- **Write the recursive case first**, describe it in English, then trace to discover the base cases. Converting a `while` loop is a legitimate route.
- **Code before a recursive call runs on the way down; code after it runs on the way back up.** A frame stays open, with no return value, for the whole duration of the call it made. A body with no `return` returns `None`.
- **Prefer readable code**: shorter is better only when equally clear; explicitly separating base case from recursive case is the recommended style while learning.
- **Mutual recursion**: `f` calls `g` and `g` calls `f`. Examples: `hailstone`/`even`/`odd` (base case lives in `odd`), and `unique_prime_factors`/`no_k` (strip out one prime, then count the rest).
- **Helper functions with extra parameters** carry the state of a process (`sevens`: `i`, `who`, `direction`) and read the outer parameters (`n`, `k`) from the parent frame.
- **Tree recursion** = more than one recursive call in the body, producing a tree-shaped process. It is a way of **exploring different choices**; sum the branches when the choices are disjoint and exhaustive.
- **`fib(n) = fib(n-2) + fib(n-1)`** with base cases 0 and 1 is the canonical example. It is slow only because it recomputes the same subproblems; memoization fixes that later in the course. Tree recursion is not inherently slow.
- **`count_partitions(n, m)`** splits into "use at least one `m`" (`count_partitions(n - m, m)`) and "use no `m`" (`count_partitions(n, m - 1)`), with base cases `n == 0` → 1, `n < 0` → 0, `m == 0` → 0, checked in that order. `count_partitions(6, 4) == 9`, `count_partitions(5, 3) == 5`.
- **`count_park(n) = 2 * count_park(n-1) + count_park(n-2)`** applies the same pattern: branch on what occupies the first spot, with `n == 0` → 1 and `n < 0` → 0.
- **`@trace`** (from the `ucb` module) prints every call and return, making tree-structured processes visible; useful for debugging projects.
