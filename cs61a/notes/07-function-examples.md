<!-- Fri, Sep 11, 2026 | sources: slides + YouTube auto-transcript -->
# Lecture 7: Function Examples

## Overview

This lecture is a review session built entirely around worked examples from past Midterm 1 exams, plus one new topic (decorators). There are no new evaluation rules introduced for most of it: instead, the lecture drills the rules you already have (the environment model, how names are looked up, how call expressions are evaluated operator-first, how `def` and `lambda` create functions with parent frames) by running them on deliberately confusing code. The four recurring themes are: (1) "What would Python display?" questions, where you must separate the *value* of an expression from the *interactive output* it produces along the way; (2) environment diagram questions where the same name (`y`, `max`, `horse`, `mask`, `arggg`) is bound to different things in different frames; (3) a strategy for implementing a function under exam conditions (read, verify examples, pick one simple example, read the template, annotate names with values, write code, check you return the right thing, re-check on other examples); and (4) decorators, which are just syntactic sugar for applying a higher-order function to a function right after defining it. The closing message is motivational: "a little bit of slope makes up for a lot of y-intercept," i.e. your rate of improvement matters far more than where you started relative to classmates who have programmed for years.

---

## Key Concepts

### 1. Value vs. interactive output

Every expression you type into the interactive interpreter does two potentially separate things:

- It **evaluates to a value**.
- Along the way, it may **display** things as a side effect (because `print` was called).

The interpreter then displays the value of the whole expression, **unless that value is `None`**, in which case nothing extra is displayed.

So for `print(5)`:
- The value is `None`.
- The displayed output is `5` (from the call to `print` itself).
- Because the value is `None`, the interpreter adds nothing more.

For `print(print(5))`:
- Call expressions evaluate operator then operands. The operand `print(5)` is evaluated first: it displays `5` and evaluates to `None`.
- Then the outer `print` is applied to `None`, which displays `None`.
- The whole expression evaluates to `None`, so no extra line appears.
- Total interactive output: two lines, `5` then `None`.

The rule to memorize: **`print` always returns `None`, and it displays its arguments separated by spaces.** If you want to see a `None` on screen, something must explicitly print it.

### 2. `and` / `or` return operand values, not booleans

From the Fall 2022 Midterm 1 Q1(a) example on the slides: `(3 and 4) - 5`.

`and` and `or` **always give you the value of either the expression on the left or the expression on the right**. They never manufacture a `True` or `False` of their own. `3 and 4`: `3` is a true value, so `and` must keep going and the result is the right operand, `4`. So `(3 and 4) - 5` is `4 - 5`, which is `-1`.

This is why you can do arithmetic on the result of `and`/`or`, and also why you can get confusing type errors from it.

### 3. True and false values

`bool(x)` returns `True` for true values and `False` for false values. The lecture's table:

```
>>> bool(0)              False
>>> bool(-1)             True
>>> bool(0.0)            False
>>> bool(' ')            True     # a space is a non-empty string
>>> bool('')             False
>>> bool(False)          False
>>> bool(print('fool'))
fool
False                              # print displays 'fool', returns None, bool(None) is False
```

Note `bool(-1)` is `True`: falsiness is about being zero/empty, not about being negative. And `' '` (a single space) is a non-empty string, hence true; `''` is empty, hence false.

### 4. The three ways a name gets bound

The slides state this crisply. There are **three ways of assigning a name to a value**:

1. **Assignment statements** (`y = x`) assign names **in the current frame**.
2. **`def` statements** assign names **in the current frame**.
3. **Call expressions** assign arguments to new names **in a local frame** (the formal parameters of the function being called).

Everything in an environment diagram comes from one of these three. There is no fourth mechanism. In particular, calling a function never modifies the caller's bindings; it makes a new frame.

### 5. `def` is two steps

A `def` statement does exactly two things, in order:

1. **Create a function value** whose parent is the **current frame** (the first frame of the current environment).
2. **Bind the name** to that function value **in the current frame**.

Step 1 is where the parent comes from. Step 2 can *rebind* a name that already meant something else in that frame, in which case the old binding is simply gone, because a single frame cannot bind one name to two values. This is exactly what happens in the `horse_mask` example below.

### 6. Name lookup and shadowing

A name evaluates to the value bound to it in the **earliest frame of the current environment** in which the name is found. Lookup starts in the current frame, then the parent frame, then the parent's parent, and so on up to global.

The consequence the lecture stresses: **if a name is bound in the local frame, you can never reach an outer binding of the same name.** In the `pirate` example, the inner function `plunder` has a parameter also named `arggg`, so the outer `arggg` is permanently unreachable from inside `plunder`. That single fact collapses `pirate` down to "prints `matey`, returns the identity function."

### 7. Parent frames come from where a function was *defined*, not where it is called

When a `def` (or `lambda`) inside a function body creates a function, that function's parent is the frame of the call in which it was created. Calling that function later makes a new frame whose parent is that stored frame, so the body can see the enclosing call's names. In the `delay` example, the returned `g` remembers `arg` from the `delay` frame where it was born.

### 8. Complex (compound) operators

`delay(delay)()(6)()` is **one big call expression**. To evaluate it you peel from the outside in to find what must be evaluated first, then build back up:

- The outermost call has operator `delay(delay)()(6)` and **no** operands.
- That operator is itself a call with operator `delay(delay)()` and operand `6`.
- That operator is itself a call with operator `delay(delay)` and no operands.
- `delay(delay)` is the innermost call, which is evaluated first.

An error like `'int' object is not callable` almost always means an operator expression evaluated to a non-function. You can call functions; you cannot call integers.

### 9. Lambda and `def` are interchangeable in expressive power

**Any program containing lambda expressions can be rewritten using `def` statements.** The slides' example:

```python
>>> (lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)
81
```

is exactly the same computation as:

```python
>>> def twice(f):
...     def g(x):
...         return f(f(x))
...     return g
...
>>> def square(y):
...     return y * y
...
>>> twice(square)(3)
81
```

The only differences are that `def` binds a name and gives the function an intrinsic name for diagrams, whereas a lambda is anonymous (shown in diagrams as `func λ <line N>(params)`).

### 10. Function implementation strategy (the exam method)

The lecture gives an explicit checklist, cross-referenced with the "from discussion" version:

**From discussion:**
- Describe a process (in English) that computes the output from the input using simple steps.
- Figure out what additional names you will need to carry out this process.
- Implement the process in code using those additional names.

**The full checklist:**
1. Read the description.
2. Verify the examples and pick a simple one (verifying the doctests checks *your understanding*, not the code).
3. Read the template. It is either helpful (use it) or confusing (ignore it, solve the problem your way, then bend your solution to fit the template later, which is almost always possible).
4. Annotate names with values from your chosen example.
5. Write code to compute the result.
6. Ask: did you really return the right thing?
7. Check your solution against the other examples.

The lecture is honest that there is always "a moment of creation" or "element of invention" in these problems. The point of the structure is to make sure your thinking time is spent on the *actual hard part*, with a small enough example to hold in your head.

### 11. Decorators

A decorator is the `@name` annotation written above a `def`. Its meaning is purely mechanical:

```python
@trace1
def square(x):
    return x * x
```

is **identical** to:

```python
def square(x):
    return x * x
square = trace1(square)
```

That is: define the function, then apply the decorator (a higher-order function) to it, then rebind the original name to the result. `trace1` is the decorator; what you get back is the **decorated function**.

Why does the syntax exist if it is just sugar? The lecture gives three reasons: it is less to type, it is nice to know up front what decorations apply to a function (a more natural place to put them), and, most of all, not all Python programmers understand higher-order functions, but everybody can understand "the magic of a decorator."

---

## Definitions

**print's contract**: `print` displays its arguments separated by spaces and **always returns `None`**.

**Interactive output**: everything displayed on screen when an expression is entered at the `>>>` prompt: the side-effect output from any `print` calls, plus the repr of the expression's value *if that value is not `None`*.

**True value / false value**: a value `x` is true if `bool(x)` is `True`, false if `bool(x)` is `False`. False values include `0`, `0.0`, `''`, `False`, and `None`.

**`and` / `or`**: operators that always evaluate to the value of either the left operand or the right operand (never to a freshly created boolean).

**Assignment statement**: binds a name to a value **in the current frame**.

**`def` statement**: (1) creates a function whose **parent is the current frame**, and (2) binds the function's name to it **in the current frame**.

**Call expression**: evaluates the operator and operands, then applies the resulting function, creating a **local frame** in which the formal parameters are bound to the argument values. The local frame's parent is the **parent of the function being called**.

**Current environment**: the sequence of frames starting with the current frame and following parent links up to the global frame.

**Name lookup rule**: a name evaluates to the value bound to it in the **earliest frame of the current environment** in which that name is found.

**Shadowing**: when a name bound in an inner frame makes an identically named binding in an outer frame unreachable from within that inner frame.

**Compound (complex) operator**: an operator that is itself a call expression or other non-atomic expression, e.g. the `delay(delay)()` in `delay(delay)()(6)`.

**`'int' object is not callable`**: the error produced when the operator of a call expression evaluates to an integer rather than a function.

**Lambda expression**: an expression that evaluates to a function with no intrinsic name; its parent is the frame in which the lambda expression is evaluated.

**Decorator**: a higher-order function applied to a function via `@name` syntax above a `def`; equivalent to defining the function and then rebinding its name to the result of calling the decorator on it.

**Decorated function**: the function value that the decorator returns, bound to the original function's name.

**`nearest_prime(n)`** (problem spec): takes an integer `n` above 5, returns the nearest prime to `n`; on a tie, returns the larger one.

**`remove(n, digit)`** (problem spec): takes a non-negative `n` and a non-negative `digit` below 10, returns an integer containing all digits of `n` that are not `digit`, in their original order.

---

## Worked Examples

### Example A: What would Python display? (`print` basics)

Setup used throughout these problems:

```python
from operator import add, mul
def square(x):
    return x * x
```

| Expression | Value | Interactive output |
|---|---|---|
| `5` | `5` | `5` |
| `print(5)` | `None` | `5` |
| `print(print(5))` | `None` | `5` then `None` |

Reasoning for the last one, step by step:
1. `print(print(5))` is a call expression. Evaluate operator `print` (a function), then evaluate the operand `print(5)`.
2. Evaluating the operand calls `print` on `5`: **displays `5`**, returns `None`.
3. Now apply the outer `print` to `None`: **displays `None`**, returns `None`.
4. The whole expression's value is `None`, so the interpreter displays nothing further.

### Example B: `delay`

```python
def delay(arg):
    print('delayed')
    def g():
        return arg
    return g
```

What `delay` is: a function that takes any argument, prints `delayed`, and returns a function of no arguments that returns that argument.

The environment reasoning in words: when `delay(x)` is called, a `delay` frame is created binding `arg` to `x`. The `def g` statement inside creates a function whose **parent is that `delay` frame**, and binds `g` in that frame. When `g` is later called, its frame's parent is that `delay` frame, so looking up `arg` in `g`'s body walks up one link and finds whatever `arg` was bound to at definition time.

**B1: `delay(delay)()(6)()`**

Peel the call expression from the outside in:

- Outermost: `[ delay(delay)()(6) ]()` -- apply to no arguments, last.
- Next: `[ delay(delay)() ](6)`.
- Next: `[ delay(delay) ]()`.
- Innermost: `delay(delay)`.

Now build back up:

1. `delay(delay)`: calls `delay` with `arg` bound to the `delay` function itself. **Displays `delayed`.** Returns a `g` whose parent frame has `arg = delay`.
2. `delay(delay)()`: calls that `g` with no arguments, which returns `arg`, which is **the `delay` function**.
3. `delay(delay)()(6)`: so this is `delay(6)`. **Displays `delayed`** again. Returns a new `g` whose parent frame has `arg = 6`.
4. `delay(delay)()(6)()`: calls that `g`, which returns `6`.

Interactive output:
```
delayed
delayed
6
```
Value of the whole expression: `6`. (It is displayed because it is not `None`.)

**B2: `print(delay(print)()(4))`**

1. Evaluate the argument `delay(print)()(4)`.
2. `delay(print)`: **displays `delayed`**, returns a `g` with `arg = print`.
3. `delay(print)()`: returns the `print` function itself.
4. `delay(print)()(4)`: this is `print(4)`. **Displays `4`**, evaluates to `None`.
5. Now the outer call: `print(None)`. **Displays `None`**, evaluates to `None`.

Interactive output:
```
delayed
4
None
```
Value: `None`.

### Example C: `pirate` (shadowing kills the outer name)

```python
def pirate(arggg):
    print('matey')
    def plunder(arggg):
        return arggg
    return plunder
```

The critical observation: `plunder`'s **parameter** is also named `arggg`. When `plunder` is called, its local frame binds `arggg` to the argument passed in, and name lookup starts in the first frame of the current environment, so it finds `arggg` immediately and **never looks in the parent frame**. The outer `arggg` is unreachable.

Therefore `pirate` reduces to: **print `matey`, return the identity function.** The outer argument to `pirate` is irrelevant to the result. (Compare `delay`, which *does* use its enclosing `arg`; the structure is identical but the inner body differs, so the behavior differs.)

**C1: `add(pirate(3)(square)(4), 1)`**

1. Evaluate the operands of `add`. First: `pirate(3)(square)(4)`.
2. `pirate(3)`: **displays `matey`**, returns `plunder` (the identity function).
3. `pirate(3)(square)`: identity applied to `square`, so this is **the `square` function**.
4. `pirate(3)(square)(4)`: `square(4)` = `16`.
5. `add(16, 1)` = `17`.

Interactive output:
```
matey
17
```
Value: `17`.

**C2: `pirate(pirate(pirate))(5)(7)`**

1. Innermost: `pirate(pirate)`. **Displays `matey`**, returns the identity function.
2. `pirate( <identity> )`. **Displays `matey`**, returns another identity function.
3. `pirate(pirate(pirate))(5)`: identity applied to `5`, which is `5`.
4. `... (7)`: now we try to call `5` on `7`. **`5` is not a function.** You can only call functions; you cannot call integers.

Interactive output:
```
matey
matey
Error
```
The whole expression **has no value**, because the operator evaluated to `5`.

### Example D: `horse_mask` (the lecturer's favorite)

```python
def horse_mask(horse, mask):
    horse = mask
    def mask(horse):
        return horse
    return horse(mask)

mask = lambda horse: horse(2)
```

Setup in the global frame (reconstructed from the transcript's description):
- `horse_mask` is a function taking `horse` and `mask`, parent global.
- `mask` in global is a **lambda** `lambda horse: horse(2)`, parent global.
- We call `horse_mask(mask)` in the sense described: the `horse` parameter receives a function and `mask` receives the global lambda. The transcript describes calling with the global `mask` lambda passed in as the argument.

The whole point, in the lecturer's words: **"we're gonna have the words `horse` and `mask` mean different things in different environments."** You must track, at every step, which frame you are in and therefore which `horse` and which `mask` you mean.

Step-by-step through the environment:

1. **Call `horse_mask`**: create frame `f1`, parent global (because `horse_mask`'s parent is global). Bind the formal parameters: `mask` in `f1` is bound to the global lambda `lambda horse: horse(2)`.
2. **`horse = mask`**: an assignment statement, which **always changes the current frame**. Look up `mask` (found in `f1`: the lambda), bind `horse` in `f1` to that lambda.
3. **`def mask(horse): return horse`**: two steps. (a) Create a new function `mask(horse)` whose **parent is `f1`**, because `f1` is the current frame. (b) Bind the name `mask` **in `f1`** to that function. Since a frame cannot bind one name to two values, the old binding of `mask` in `f1` (the lambda) is **erased**. Note that `horse` in `f1` still points to the lambda, so the lambda is not lost.
4. **`return horse(mask)`**: look up `horse` in `f1` (the **lambda**) and `mask` in `f1` (the newly defined **`mask` function**). So we are calling the lambda with the `mask` function as its argument.
5. **Call the lambda**: create frame `f2`, labeled with the lambda, **parent global** (because the lambda was created in the global frame). Bind its parameter `horse` to the argument, which is the inner `mask` function.
6. **Lambda body: `return horse(2)`**: look up `horse` in `f2`: it is the inner `mask` function. Call it on `2`.
7. **Call `mask`**: create frame `f3`, **parent `f1`** (because that inner `mask` function's parent is `f1`). Bind its parameter `horse` to `2`.
8. **`mask` body: `return horse`**: look up `horse` in `f3`: it is `2`. **Return `2`.**
9. That `2` is the value of `horse(2)` in the lambda frame, so the **lambda returns `2`**.
10. That `2` is the value of `horse(mask)` in `f1`, so **`horse_mask` returns `2`**.

Final answer: no error, and the expression **evaluates to `2`**.

The trap: three different frames each bind the name `horse`, to three different things (a lambda, the inner `mask` function, and the integer `2`), and two frames bind `mask` differently. Getting this right is purely a matter of asking "which frame am I in right now?" at every single lookup.

### Example E: Fall 2022 Midterm 1, Question 2 (environment diagram)

```python
 1: def f(x):
 2:     """f(x)(t) returns max(x*x, 3*x)
 3:     if t(x) > 0, and 0 otherwise.
 4:     """
 5:     y = max(x * x, 3 * x)
 6:     def zero(t):
 7:         if t(x) > 0:
 8:             return y
 9:         return 0
10:     return zero
11:
12: # Find the largest positive y below 10
13: # for which f(y)(lambda z: z - y + 10)
14: # is not 0.
15: y = 1
16: while y < 10:
17:     if f(y)(lambda z: z - y + 10):
18:         max = y
19:     y = y + 1
```

Global frame bindings that build up: `f` -> `func f(x) [p=G]`, `y` (ends at `12`, since the loop increments past 10... it exits when `y` reaches `10`, and the diagram on the slide shows `y` reaching a final value), `max` -> `func max(...) [p=G]` initially, then rebound to `1`.

**First iteration (`y = 1`):**

- `f(y)` = `f(1)`: creates frame `f1`, parent `G`, with `x = 1`.
  - `y = max(1*1, 3*1)` = `max(1, 3)` = `3`. This `y` is bound in `f1`, **not** global. It shadows the global `y` inside `f1`'s scope.
  - `def zero(t)` creates `func zero(t) [p=f1]` and binds `zero` in `f1`.
  - Returns `zero`. **This is the value of `f(y)`.**
- The lambda `lambda z: z - y + 10` is created in the **global** frame (it appears at line 17 in global code), so it is `func λ <ln 17>(z) [p=G]`. Its `y` will be looked up in **global** at call time.
- Calling the returned `zero` on the lambda creates frame `f2`, label `zero`, **parent `f1`** (because `zero`'s parent is `f1`), with `t` bound to the lambda.
  - Body: `if t(x) > 0`. Look up `t` in `f2` (the lambda) and `x` in `f2`... not found, so up to `f1`: `x = 1`.
  - Calling the lambda creates frame `f3`, label `λ <ln 17>`, **parent `G`**, with `z = 1`.
    - Body: `z - y + 10`. `z` is `1` (in `f3`), `y` is looked up in `f3` then `G`: global `y` is `1`. So `1 - 1 + 10` = `10`. **Return `10`.**
  - Back in `f2`: `t(x)` is `10`, and `10 > 0`, so `return y`. Look up `y` in `f2`, not found, up to `f1`: `y = 3`. **Return `3`.**
- Back in global: `if 3:` is true, so line 18 executes: **`max = y`** binds the global name `max` to `1`.

**This is the disaster.** Line 18 rebinds the global name `max`, which was the built-in `max` function, to the integer `1`.

**Second iteration (`y = 2`):**

- `f(y)` = `f(2)`: creates frame `f4`, parent `G`, with `x = 2`.
  - Line 5: `y = max(x * x, 3 * x)`. Look up `max`: found in the global frame bound to **`1`, an integer**.
  - Attempting to call `1` on two arguments raises **`'int' object is not callable`**.

**"Why?"** (the question the slide poses): because line 18's assignment `max = y` is an assignment statement executing in the global frame, so it rebound the global name `max` away from the built-in function to the integer `1`. `f`'s body looks up `max` in its own frame, fails, then goes to its parent, global, and finds the integer. The moral: **assigning to a name that shadows a built-in in the global frame breaks every later use of that built-in.**

Also note the two distinct `y`s: the `y` inside `f1` (equal to `max(x*x, 3*x)`) and the global `y` (the loop counter). The lambda's `y` resolves to the **global** one because the lambda's parent is `G`; `zero`'s `y` resolves to the **`f1`** one because `zero`'s parent is `f1`. Same name, two different answers, decided entirely by the parent chain.

### Example F: `nearest_prime` (a slight variant of Fall 2022 Midterm 1 3(b))

> Implement `nearest_prime`, which takes an integer `n` above 5. It returns the nearest prime number to `n`. If two prime numbers are equally close to `n`, return the larger one. Assume `is_prime(n)` is implemented already.

```python
def nearest_prime(n):
    """Return the nearest prime number to n.
    In a tie, return the larger one.

    >>> nearest_prime(8)
    7
    >>> nearest_prime(11)
    11
    >>> nearest_prime(21)
    23
    """
    k = 0
    while True:
        if _______________:
            return _______
        if k > 0:
            k = -k
        else:
            k = ________
```

**Applying the strategy:**

*Step 1, read the description.* Nearest prime; larger on a tie.

*Step 2, verify examples and pick one.* `nearest_prime(8)`: primes near 8 are 7 (distance 1) and 11 (distance 3), so 7. Correct. `nearest_prime(11)`: 11 is itself prime, distance 0, so 11. Correct. `nearest_prime(21)`: 19 is distance 2, 23 is distance 2, a tie, so return the **larger**, 23. Correct, and this is the interesting example, so **use `n = 21`**.

*Step 3, read the template.* There is a `k` initialized to 0, an infinite loop, a test that returns, and then a two-branch update of `k`. So `k` is an offset and we are scanning outward.

*Step 4, describe the process in English.* Check whether a number is prime **in this order**:

```
n, n+1, n-1, n+2, n-2, n+3, n-3, n+4, ...
```

This order is exactly what the tie rule demands: you always try the larger candidate before the smaller one at the same distance.

*Step 5, notice the pattern.* All of these look like `n + k` for various `k`:

```
k:  0, +1, -1, +2, -2, +3, -3, +4, ...
```

So the update rule is: if `k` is positive, flip its sign (`k = -k`, going from `+j` to `-j`, same distance, smaller side). Otherwise (`k` is zero or negative), flip and step out by one (`k = -k + 1`, going from `-j` to `+(j+1)`, or from `0` to `+1`).

*Step 6, fill in.* With `n = 21`, the first `k` that works is `k = 2`, giving `n + k = 23`.

```python
def nearest_prime(n):
    k = 0
    while True:
        if is_prime(n + k):
            return n + k
        if k > 0:
            k = -k        # keep looking on the smaller side, same distance
        else:
            k = -k + 1    # step out to the next larger distance
```

*Step 7, check.* For `n = 21`: `k = 0` -> 21 not prime; `k = 1` -> 22 not prime; `k = -1` -> 20 not prime; `k = 2` -> 23 **is prime**, return 23. Correct. For `n = 8`: `k = 0` -> 8 no; `k = 1` -> 9 no; `k = -1` -> 7 **yes**, return 7. Correct. For `n = 11`: `k = 0` -> 11 **yes**, return 11. Correct.

### Example G: `remove(n, digit)` (the full "invent a strategy" walkthrough)

> Return all digits of non-negative `n` that are not `digit`, for some non-negative `digit` less than 10.

Examples: `remove(231, 3)` is `21`; `remove(243132, 2)` is `4313`.

The template:

```python
def remove(n, digit):
    kept, digits = 0, 0
    while ________:
        n, last = n // 10, n % 10
        if ________:
            kept = ________
            digits = ________
    return ________
```

**Chosen example: `n = 231`, `digit = 3`, expected result `21`.**

Reading the template: we initialize `kept` and `digits` to 0. `n, last = n // 10, n % 10` is a familiar pattern: peel off the last digit into `last`, and shrink `n`. For 231 that gives `last` = 1, then 3, then 2, processing digits **right to left**.

**Attempt 1: `kept = kept + last`.**
- `last = 1`, not 3, so `kept = 0 + 1 = 1`.
- `last = 3`, equals `digit`, skip.
- `last = 2`, not 3, so `kept = 1 + 2 = 3`.
- Got `3`, wanted `21`. **Wrong.** We need 2 and 1 to become 21, not 3.

**Attempt 2: `kept = kept * 10 + last`.**
- `last = 1`: `kept = 0 * 10 + 1 = 1`.
- `last = 3`: skip.
- `last = 2`: `kept = 1 * 10 + 2 = 12`.
- Got `12`, wanted `21`. **Backwards**, because we are consuming digits right to left but this formula builds left to right.

**Attempt 3: `kept = kept + last * 10`.**
- `last = 1`: `kept = 0 + 10 = 10`.
- `last = 3`: skip.
- `last = 2`: `kept = 10 + 20 = 30`.
- Got `30`. **Wrong.** The multiplier must grow, not be a constant 10.

**Insight (the moment of invention).** We want `1 * 1 + 2 * 10 = 21`. The multiplier for each kept digit should be **10 raised to the number of digits already in `kept`**. That is why the template gives us a second accumulator, `digits`: it counts how many digits we have kept so far.

**Attempt 4 (correct):**

```python
def remove(n, digit):
    """Return all digits of non-negative N that are not DIGIT,
    for some non-negative DIGIT less than 10.

    >>> remove(231, 3)
    21
    >>> remove(243132, 2)
    4313
    """
    kept, digits = 0, 0
    while n > 0:
        n, last = n // 10, n % 10
        if last != digit:
            kept = kept + last * 10 ** digits
            digits = digits + 1
    return kept
```

Trace on `n = 231`, `digit = 3`:
- `last = 1`, `digits = 0`: `kept = 0 + 1 * 10**0 = 1`; `digits` becomes 1.
- `last = 3`: equals `digit`, skipped, and crucially `digits` is **not** incremented.
- `last = 2`, `digits = 1`: `kept = 1 + 2 * 10**1 = 21`; `digits` becomes 2.
- Return `21`. Correct.

Check on a harder case, `n = 231`, `digit = 4` (nothing gets removed, expect `231`):
- `1 * 1 = 1`; `3 * 10 = 30`, running total 31; `2 * 100 = 200`, running total 231. Correct.

**An alternative solution** the lecture mentions: instead of powers of ten going up, divide by 10 each time to build up `2.1`, then at the end multiply by `10 ** (digits - 1)` to shift the decimal point back and `round` the float. This also solves the problem. **There is not always only one solution.** (The lecture notes the float version needs rounding, which is a hint that the integer version is cleaner.)

### Example H: `trace1`, and decorators

Start with ordinary functions:

```python
def square(x):
    return x * x

def sum_squares_up_to(n):
    k = 1
    total = 0
    while k <= n:
        total = total + square(k)
        k = k + 1
    return total
```

`square(12)` is `144`; `sum_squares_up_to(5)` is `55`.

Now implement a tracing higher-order function. It is called `trace1` because it only handles functions **of one argument**:

```python
def trace1(fn):
    """Return a version of fn that first prints before it is called.

    fn -- a function of 1 argument
    """
    def traced(x):
        print('Calling', fn, 'on argument', x)
        return fn(x)
    return traced
```

Reading it: `trace1` takes a function and returns `traced`, which is "just like `fn`, except it also prints." Inside `traced`, `fn` is found by looking up to the parent frame (the `trace1` frame), where it was bound as a parameter. `traced` returns `fn(x)`, so the decorated function computes exactly the same value as the original.

Apply it as a decorator:

```python
@trace1
def square(x):
    return x * x
```

Now `square(12)` first displays something like `Calling <function square> on argument 12`, then evaluates to `144`.

Decorate both functions and call `sum_squares_up_to(5)`: you see the call to `sum_squares_up_to` on `5`, then the calls to `square` on `1`, `2`, `3`, `4`, `5`, and then `55` is returned. You get a picture of all the work that happened.

**The equivalence, which is the exam-relevant fact:** the decorator form above is identical to

```python
def square(x):
    return x * x
square = trace1(square)
```

The name `square` is now bound to the **decorated** function (the `traced` function), not the original. The original `square` body still exists, but it is only reachable as the `fn` bound inside the `trace1` frame that `traced`'s parent points to.

---

## Common Pitfalls

1. **Writing `None` as interactive output when it is only the value.** If an expression evaluates to `None`, the interpreter displays *nothing* for it. `None` appears on screen only when something explicitly prints it.

2. **Forgetting that `print` returns `None`.** The most common downstream error: assuming `print(5)` "is" `5`, and then doing arithmetic or further calls on it.

3. **Thinking `and`/`or` produce booleans.** `3 and 4` is `4`, not `True`. `(3 and 4) - 5` is `-1`.

4. **Thinking `bool(-1)` is `False`.** Falsiness means zero or empty, not negative. Also `bool(' ')` is `True`: a space is a character.

5. **Confusing "where a function was defined" with "where it is called."** A function's parent frame is fixed at creation. In Midterm Q2, the lambda's parent is global (so its `y` is the loop counter), while `zero`'s parent is `f1` (so its `y` is `max(x*x, 3*x)`). Same name, two answers.

6. **Missing that an inner parameter shadows an outer one.** `pirate`'s inner `plunder(arggg)` makes the outer `arggg` permanently unreachable. That single detail is the entire difference between `pirate` and `delay`.

7. **Assigning to a built-in's name in the global frame.** `max = y` in Q2 destroys `max` for all subsequent calls, producing `'int' object is not callable` when `f` next runs line 5. Watch for `max`, `min`, `sum`, `abs`, `print`, and `list` being used as variable names in exam code.

8. **Trying to call a non-function.** `pirate(pirate(pirate))(5)(7)` errors because the operator evaluated to `5`. Any `'int' object is not callable` means "trace back and find which operator expression produced an int."

9. **Evaluating a nested call expression left to right instead of inside out.** You must peel `f(a)(b)(c)` into its operator/operand structure and evaluate the innermost operator first.

10. **Forgetting that a `def` inside a frame *rebinds* and erases the previous binding of that name in the same frame.** In `horse_mask`, `def mask(horse)` wipes out `f1`'s earlier binding of `mask` to the lambda. The lambda survives only because `horse` was already pointing at it.

11. **In `remove`, incrementing `digits` even when a digit is skipped.** The counter must count *kept* digits only, or the powers of ten will have gaps and the answer will be wrong.

12. **Building a number in the wrong direction.** `kept * 10 + last` versus `kept + last * 10 ** digits` give reversed results when you peel digits right to left. Trace an example; do not trust the formula by eye.

13. **Not checking what you actually return.** The checklist explicitly asks: "Did you really return the right thing?" It is easy to build the correct value in a local variable and then return the wrong one, or return a float where an int was requested.

14. **Assuming there is a unique correct solution.** Both the powers-of-ten version and the divide-then-round version solve `remove`. Similarly, if a template confuses you, solve it your own way first and adapt afterward.

---

## Likely Exam Points

### 1. "What would Python display?" with nested `print`

**Q.** What is the interactive output, and what does the expression evaluate to?
```python
print(print(1), print(2))
```

**A.** Operands are evaluated left to right before the outer call: `print(1)` displays `1` and returns `None`; `print(2)` displays `2` and returns `None`. Then the outer `print(None, None)` displays `None None` (separated by a space). Output:
```
1
2
None None
```
The whole expression evaluates to `None`, so nothing further is displayed.

### 2. `and` / `or` returning operand values

**Q.** What does `(3 and 4) - 5` evaluate to? What about `(0 or '') or 7`?

**A.** `3` is true, so `3 and 4` gives the right operand `4`; `4 - 5` is `-1`. For the second: `0` is false so `0 or ''` gives `''`; `''` is false so `'' or 7` gives `7`. Answers: `-1` and `7`. **(extra context: the second sub-question is my own construction in the same style as the lecture's example.)**

### 3. Truthiness table

**Q.** What is the interactive output of `bool(print(' '))`?

**A.** `print(' ')` displays a single space (a blank-looking line) and returns `None`. `bool(None)` is `False`. So the output is a blank line, then `False`. The trap: `bool(' ')` alone would be `True`, but here `print` intervenes and the argument to `bool` is `None`.

### 4. Higher-order function behavior: does the inner function use the outer name?

**Q.** Given

```python
def delay(arg):
    print('delayed')
    def g():
        return arg
    return g

def pirate(arggg):
    print('matey')
    def plunder(arggg):
        return arggg
    return plunder
```

what does `delay(7)()` evaluate to, and what does `pirate(7)(9)` evaluate to?

**A.** `delay(7)()` is `7`: `g` has no parameters, so `arg` is looked up in the parent `delay` frame where it is `7`. `pirate(7)(9)` is `9`: `plunder`'s own parameter `arggg` shadows the outer one, so `pirate` returns the identity function and the `7` is irrelevant. Both display one line first (`delayed`, `matey` respectively).

### 5. Compound operators and `'int' object is not callable`

**Q.** Using `pirate` above, what happens when you evaluate `pirate(pirate(pirate))(5)(7)`?

**A.** `pirate(pirate)` prints `matey` and returns identity. `pirate(<identity>)` prints `matey` and returns identity. Applying identity to `5` gives `5`. Then `5(7)` raises an error: you can only call functions, not integers. Output is `matey`, `matey`, then an error; the expression has no value.

### 6. Environment diagram: two bindings of the same name in different frames

**Q.** In Fall 2022 Midterm 1 Q2, when `f(1)(lambda z: z - y + 10)` is evaluated with the global `y` equal to `1`, which `y` does the lambda's body use, and which `y` does `zero`'s `return y` use?

**A.** The lambda was created in the global frame, so its parent is `G` and its `y` is the **global** `y` (the loop counter, `1`). The `zero` function was created inside the `f` frame `f1`, so its parent is `f1` and its `y` is the **`f1`** `y`, which is `max(x*x, 3*x) = max(1, 3) = 3`. The `zero` call returns `3`.

### 7. Shadowing a built-in in the global frame

**Q.** In Q2, why does the second loop iteration raise `'int' object is not callable`?

**A.** In the first iteration the condition on line 17 was true, so line 18 executed `max = y`, an assignment statement in the **global** frame that rebound the name `max` from the built-in function to the integer `1`. On the next call to `f`, line 5 evaluates `max(x * x, 3 * x)`; `max` is not in the `f` frame, so lookup goes to the global frame and finds `1`. Calling an integer is an error.

### 8. `def` as two steps, and rebinding within one frame

**Q.** In `horse_mask`, after `horse = mask` and then `def mask(horse): return horse` both execute in `f1`, what is `f1`'s `mask` bound to, and what is `f1`'s `horse` bound to?

**A.** `f1`'s `mask` is bound to the newly created `mask(horse)` function with parent `f1`; the earlier binding of `mask` to the global lambda in `f1` is erased, because one frame cannot bind a name to two values. `f1`'s `horse` is still bound to the lambda, since the assignment `horse = mask` happened *before* the `def` rebound `mask`.

### 9. Full `horse_mask` trace

**Q.** What does the `horse_mask` example evaluate to, and how many frames are created?

**A.** It evaluates to `2`, with no error. Three frames beyond global: `f1` (the `horse_mask` call), the lambda frame (parent **global**, `horse` bound to the inner `mask` function), and the `mask` frame (parent **`f1`**, `horse` bound to `2`). The innermost `return horse` gives `2`, which propagates back through the lambda and out of `horse_mask`.

### 10. Lambda-to-def translation

**Q.** Rewrite `(lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)` using only `def` statements, and give its value.

**A.**
```python
def twice(f):
    def g(x):
        return f(f(x))
    return g

def square(y):
    return y * y

twice(square)(3)
```
Value: `square(square(3))` = `square(9)` = `81`.

### 11. Function implementation from a template

**Q.** Fill in `nearest_prime`'s blanks and justify the order of candidates.

**A.**
```python
if is_prime(n + k):
    return n + k
if k > 0:
    k = -k
else:
    k = -k + 1
```
The `k` sequence is `0, 1, -1, 2, -2, 3, -3, 4, ...`, so candidates are checked as `n, n+1, n-1, n+2, n-2, ...`. Checking `n + j` before `n - j` is exactly what makes ties resolve to the **larger** prime.

### 12. Digit-accumulation loop

**Q.** Fill in `remove`'s blanks and explain why `digits` is incremented inside the `if` rather than outside it.

**A.**
```python
while n > 0:
    n, last = n // 10, n % 10
    if last != digit:
        kept = kept + last * 10 ** digits
        digits = digits + 1
return kept
```
`digits` counts how many digits are already in `kept`, so it supplies the correct power of 10 for the next kept digit. If it were incremented for skipped digits too, `remove(231, 3)` would compute `1 * 10**0 + 2 * 10**2 = 201` instead of `21`.

### 13. Decorator equivalence

**Q.** Rewrite

```python
@trace1
def square(x):
    return x * x
```

without using `@`, and say what name `square` is bound to afterward.

**A.**
```python
def square(x):
    return x * x
square = trace1(square)
```
Afterward, the global name `square` is bound to the `traced` function returned by `trace1`, **not** the original `square`. The original is still reachable only as the `fn` bound in the `trace1` frame that `traced`'s parent points to.

### 14. Implementing `trace1`

**Q.** Write `trace1(fn)` so that the returned function prints a message and then behaves identically to `fn` (for one-argument `fn`).

**A.**
```python
def trace1(fn):
    def traced(x):
        print('Calling', fn, 'on argument', x)
        return fn(x)
    return traced
```
The key is `return fn(x)`, not just `fn(x)`: the decorated function must return the same value as the original, or `sum_squares_up_to` would break.

---

## Summary

- **`print` always returns `None`** and displays its arguments separated by spaces. The interpreter displays an expression's value only if it is not `None`; `None` appears on screen only when explicitly printed.
- **`and` / `or` return the value of the left or right operand**, never a fresh boolean. `(3 and 4) - 5` is `-1`.
- **False values**: `0`, `0.0`, `''`, `False`, `None`. `-1` and `' '` are **true**.
- **Three ways names get bound**: assignment statements (current frame), `def` statements (current frame), call expressions (new local frame for the parameters). Nothing else.
- **`def` is two steps**: create a function whose parent is the current frame, then bind the name in the current frame, erasing any previous binding of that name there.
- **Name lookup** finds the earliest frame in the current environment that has the name. A local binding makes an outer binding of the same name unreachable (`pirate`'s `arggg`).
- **Parent frames come from definition site, not call site.** In Midterm Q2, `zero`'s `y` is the `f1` one; the lambda's `y` is the global one.
- **Assigning to a built-in's name in the global frame** (`max = y`) breaks all later uses: `'int' object is not callable`.
- **Nested call expressions** with compound operators must be peeled outside in and evaluated inside out. Only functions are callable.
- **`horse_mask` evaluates to `2`**: the entire difficulty is tracking which frame you are in when you look up `horse` and `mask`.
- **Any lambda program can be rewritten with `def`**; `(lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)` is `twice(square)(3)` is `81`.
- **Implementation strategy**: read the description, verify the examples, pick a simple one, read the template (use it or ignore it and adapt later), annotate names with values, write the code, confirm you return the right thing, re-check against other examples.
- **Expect a moment of invention** in every implementation problem. The structure exists so your thinking time is spent on the actual hard part; multiple correct solutions usually exist.
- **`nearest_prime`**: scan `k = 0, 1, -1, 2, -2, ...` via `k = -k` when positive and `k = -k + 1` otherwise; testing `n + j` before `n - j` implements the "larger on a tie" rule.
- **`remove`**: peel digits right to left, accumulate `kept = kept + last * 10 ** digits`, and increment `digits` only for digits actually kept.
- **A decorator is sugar**: `@trace1` above `def square` means exactly `square = trace1(square)` after the `def`. It exists for brevity, for visibility at the top of the definition, and because decorators are easier to understand than higher-order functions for most Python programmers.
- **Closing thought**: "a little bit of slope makes up for a lot of y-intercept." Your rate of learning matters more than your starting point; do more practice problems and make new friends in exam prep sections.
