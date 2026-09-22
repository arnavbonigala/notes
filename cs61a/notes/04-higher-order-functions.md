<!-- Wed, Sep 02, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 4: Higher-Order Functions

This lecture finishes the story of iteration (the `fib` example), then asks a foundational question: why does a programming language need control statements at all, if it has functions? The answer is that call expressions always evaluate every operand before applying the function, while `if` and `while` get to *skip* code, and `and`/`or` get to *short-circuit*. From there the lecture pivots to function design (domain, range, behavior; one job per function; Don't Repeat Yourself) and to the central idea of the course so far: functions are just values, so they can be passed as arguments and returned as results. Generalizing `sum_naturals` and `sum_cubes` into a single `summation(n, term)` shows functions-as-arguments; `make_adder(n)` shows functions-as-return-values; and the Twenty-One game with pluggable strategies (`make_heckling_strategy`) shows why this matters for real program design: modularity, abstraction, and separation of concerns.

---

## Key Concepts

### 1. Iteration requires deciding what state to track

When you write a `while` loop, the key design question is: **what information must I carry from one iteration to the next?** For Fibonacci, the next number is the sum of the previous two, so you must track both of them (`pred` and `curr`), plus where you are in the sequence (`k`). Every name in a loop is there because some future iteration needs it.

The loop invariant is what makes the code readable: throughout `fib`, "`curr` is the `k`th Fibonacci number." Once you state that, the update `pred, curr = curr, pred + curr` together with `k = k + 1` is obviously correct, and the exit condition tells you the answer is in `curr`.

### 2. Control statements are not functions

Consider trying to replace `if` with a function:

```python
def if_(c, t, f):
    if c:
        return t
    else:
        return f
```

This *looks* like it works, but it is fundamentally different, because of the evaluation rules:

- **Conditional statement:** clauses are considered in order; the header expression is evaluated, and only the suite of the first true header (or the `else`) is executed. The other suites are **never executed**.
- **Call expression:** evaluate the operator, then evaluate **all** operands left to right, then apply. Nothing is skipped.

So `if_(x >= 0, sqrt(x), 0)` evaluates `sqrt(x)` *no matter what x is*, and crashes for negative `x`. The whole reason control statements exist as a separate kind of thing in the language is that they control which code runs and how many times. Functions cannot do that with their arguments.

### 3. Short-circuiting: expressions that do skip evaluation

`and`, `or`, and `not` are the exception: they are expressions that may leave a subexpression unevaluated.

- `<left> and <right>`: evaluate `<left>`. If it is a **false value** `v`, the whole expression is `v`. Otherwise the whole expression is the value of `<right>`.
- `<left> or <right>`: evaluate `<left>`. If it is a **true value** `v`, the whole expression is `v`. Otherwise the whole expression is the value of `<right>`.
- `not <exp>`: evaluate `<exp>`; result is `True` if it was a false value, `False` otherwise. (`not` always returns a bool.)

Two consequences worth internalizing:

1. **`and`/`or` return the last subexpression evaluated, not necessarily a boolean.** `2 and 3` evaluates to `3`. `'' or 'Ousterhout'` evaluates to `'Ousterhout'`.
2. **Short-circuiting is a form of control**, and it is how you write guards: `x > 0 and sqrt(x) > 10` never calls `sqrt` on a negative number, and `n == 0 or 1/n != 0` never divides by zero.

False values in Python so far: `False`, `0`, `''`, `None` (more to come later in the course).

### 4. Describing a function: domain, range, behavior

For `def square(x): return x * x`:

- **Domain:** the set of all inputs it might take. Here, `x` is a number.
- **Range:** the set of values it might return. Here, a non-negative real number.
- **Behavior** (for a pure function): the relationship it creates between input and output. Here, it returns the square of `x`.

Docstrings should describe all three. This vocabulary is how you will be asked to specify functions on assignments and exams.

### 5. Design guidelines: one job, applied broadly, and DRY

- **Give each function exactly one job, but make it apply to many related situations.** `round` has one job (rounding), but takes an optional second argument so the one job covers many cases: `round(1.23)` is `1`, `round(1.23, 1)` is `1.2`, `round(1.23, 5)` is `1.23`.
- **Don't Repeat Yourself (DRY):** implement a process once, execute it many times. When you find yourself copy-pasting a line or a loop, that is a signal to factor it into a function or a parameter.

### 6. Generalization, first over values, then over processes

**Over values:** the areas of a square, circle, and regular hexagon with characteristic length `r` are `r*r`, `pi*r*r`, and `(3*sqrt(3)/2)*r*r`. They differ only in a **constant**. So parameterize by that constant and write the shared work (including the `assert`) once.

**Over processes:** the sums 1+2+...+n, 1³+2³+...+n³, and the pi series all differ only in *how each term is computed from k*. That difference is not a number, it is an **expression**, that is, a computation. To parameterize over a computation you must pass a **function**. This is exactly the step that requires higher-order functions.

### 7. Functions are first-class values

A value is anything an expression can evaluate to: `2`, `3.14`, `'are you a human'`, `False`, and `pow`. Functions are just another value. Being **first-class** means functions can be:

- bound to names,
- passed as arguments to other functions,
- returned as the result of other functions.

A **higher-order function** is one that does either of the last two: takes a function as an argument, or returns a function.

Why this matters (the lecture's own three reasons):
1. Express general methods of computation (how to sum, without committing to what is summed).
2. Remove repetition from programs.
3. Separate concerns: the general method is one function's job; the specific term/strategy is another function's job.

### 8. Locally defined functions and returning functions

A `def` statement can appear inside another function's body. When the enclosing function is called, the inner `def` executes and binds a new function to a name in that call's **local frame**. That inner function can refer to the formal parameters of the enclosing function, so the returned function "remembers" them. That is what makes `make_adder(3)` produce a function that adds specifically 3.

Also note: the operator of a call expression is **any expression that evaluates to a function**, so it can itself be a call expression. `make_adder(1)(2)` is legal: evaluate `make_adder(1)` to get the `adder` function, then call that on `2`.

### 9. Modularity, abstraction, separation of concerns

The lecture's mail analogy: to send a letter to Grandma in Columbia, South Carolina, you put it in a mailbox with an address.

- **Abstraction:** you only provide the address. You do not specify the route, the trucks, or the plane.
- **Modularity:** the system is broken into pieces (postal worker, distribution center, truck, pilot, local carrier) each doing one job.
- **Separation of concerns:** the postal worker does not know how to pilot an airplane, and does not need to.

Same for `curl https://cs61a.org`: you give an address, and layers below handle where the machine is, how to route there, how to push bits over a wire, and how to share that wire.

---

## Definitions

- **Iteration:** repeated execution of a block of code, controlled by a `while` statement, with state carried in names that are rebound each pass.
- **Control statement:** a statement such as `if` or `while` that determines *which* parts of a program are executed and *how many times*. Unlike a call expression, it may skip code entirely.
- **Short-circuiting:** the behavior of `and` and `or` in which Python stops evaluating as soon as it knows the answer and returns the value of the most recently evaluated subexpression.
- **False value:** a value that counts as false in a boolean context. So far: `False`, `0`, `''`, `None`.
- **Domain (of a function):** the set of all inputs it might possibly take as arguments.
- **Range (of a function):** the set of output values it might possibly return.
- **Pure function:** a function whose only effect is the relationship it creates between its inputs and its return value (no side effects such as printing).
- **Behavior (of a pure function):** the input/output relationship it establishes.
- **DRY (Don't Repeat Yourself):** implement a process just once, but execute it many times.
- **`assert <expression>, <message>`:** a statement that evaluates the expression in a boolean context; if the value is false, an `AssertionError` is raised showing the message; if true, nothing happens.
- **First-class value:** a value that can be bound to a name, passed as an argument, and returned from a function. In Python, functions are first-class.
- **Higher-order function:** a function that takes a function as an argument and/or returns a function as its result.
- **Formal parameter:** the name in a function's `def` header that is bound to an argument value in the local frame when the function is called. In `summation(n, term)`, `term` is a formal parameter that will be bound to a function.
- **Operator / operand:** in a call expression, the operator is the subexpression evaluated to get the function; the operands are the subexpressions evaluated to get the arguments. The operator may be any expression that evaluates to a function, including another call expression.
- **Local function definition:** a `def` statement inside another function's body; it creates a function and binds it to a name in the local frame of the enclosing call.
- **Modularity:** breaking a system into separate pieces, each with its own job.
- **Abstraction:** using a component through a simple interface without knowing how it is implemented.
- **Separation of concerns:** each component only needs to know about its own job, not the jobs of other components.

### Evaluation Procedure Cheat Sheet (from the slides)

**Call expressions** (parens, e.g. `f(x)`):
1. Evaluate the operator (function).
2. Evaluate the operands, from left to right (arguments).
3. Apply the function to the arguments.

**Calling user-defined functions:**
1. Add a local frame, forming a new environment.
2. Bind the function's formal parameters to its arguments in that frame.
3. Execute the body of the function in that new environment.

**Assignment** (`=` sign, e.g. `x = 1 + 2`): the expression on the right is evaluated, and its value is assigned to the name on the left.

**Boolean expressions** (`and`, `or`, `not`): see the short-circuiting rules above.

---

## Worked Examples

### Example 1: Iterative Fibonacci, and a better set of starting values

```python
def fib(n):
    """Compute the nth Fibonacci number, for n >= 1."""
    pred, curr = 0, 1   # 0th and 1st Fibonacci numbers
    k = 1               # curr is the kth Fibonacci number
    while k < n:
        pred, curr = curr, pred + curr
        k = k + 1
    return curr
```

**Step by step for `fib(5)`:**

A local frame for `fib` is created with `n` bound to `5`. Then `pred` is bound to `0`, `curr` to `1`, `k` to `1`. The invariant is "`curr` is the `k`th Fibonacci number," and it holds now: the 1st Fibonacci number is 1. `n` never changes.

Tracking the frame's bindings after each execution of `k = k + 1`:

| pass | pred | curr | k |
|---|---|---|---|
| start | 0 | 1 | 1 |
| 1 | 1 | 1 | 2 |
| 2 | 1 | 2 | 3 |
| 3 | 2 | 3 | 4 |
| 4 | 3 | 5 | 5 |

Now `k < n` is false (`5 < 5`), so the loop ends and `curr`, which is `5`, is returned. That is the 5th Fibonacci number.

Note the simultaneous assignment `pred, curr = curr, pred + curr`: the right-hand side is fully evaluated *first*, using the old values, then both names are rebound. Writing it as two separate statements would be wrong.

**Discussion question from lecture:** what if we started with `pred, curr = 1, 0` and `k = 0`?

This version is still correct for all `n >= 1`, and it is strictly **better**, because it also handles `n = 0`. With `n = 0`: `curr` is `0`, `k` is `0`, so `k < n` is immediately false, the body never runs, and `0` is returned, which is the 0th Fibonacci number. The original version would have returned `1`. For `n = 5` this version executes the loop body five times instead of four (`curr` goes 0, 1, 1, 2, 3, 5) and still returns 5.

### Example 2: Why `if_` as a function fails

```python
from math import sqrt

def real_sqrt(x):
    """Return the real part of the square root of x."""
    if x >= 0:
        return sqrt(x)
    else:
        return 0
```

`real_sqrt(16)` is `4.0`; `real_sqrt(-16)` is `0`, because for negative `x` the whole square root is imaginary (the square root of -4 is 2i, whose real part is 0). Crucially, `sqrt(x)` is *never evaluated* when `x` is negative: the `if` statement skips that suite.

Now the "convenient one-liner" version:

```python
def if_(c, t, f):
    if c:
        return t
    else:
        return f

def real_sqrt(x):
    return if_(x >= 0, sqrt(x), 0)
```

`real_sqrt(16)` still gives `4.0`. But `real_sqrt(-16)` raises a `ValueError` from `math.sqrt`.

**Why, in evaluation-rule terms:** to evaluate `if_(x >= 0, sqrt(x), 0)`, Python evaluates the operator `if_`, then evaluates the operands left to right: `x >= 0` gives `False`, then `sqrt(x)` is evaluated and **crashes**, before `if_` is ever applied. The body of `if_` is irrelevant; the error happens during operand evaluation. Call expressions cannot skip operands. Control statements can skip suites. That is the whole difference.

### Example 3: Short-circuiting as a guard

```python
def has_big_sqrt(x):
    """Return whether the real part of the square root of x is greater than 10."""
    return x > 0 and sqrt(x) > 10
```

- `has_big_sqrt(1)`: `1 > 0` is a true value, so evaluate `sqrt(1) > 10`, which is `False`. Result: `False`.
- `has_big_sqrt(1000)`: `True`, since `sqrt(1000)` is about 31.6.
- `has_big_sqrt(-1000)`: `-1000 > 0` is `False`, a false value, so the whole `and` expression is `False` **and `sqrt(-1000)` is never evaluated**. No crash.

```python
def reasonable(n):
    """Return whether 1/n is a non-zero number (or n is 0)."""
    return n == 0 or 1/n != 0
```

- `reasonable(0)`: `0 == 0` is a true value, so `1/0` is never evaluated. Result: `True`. No `ZeroDivisionError`.
- `reasonable(10)`: `True`.
- `reasonable(10**100)`: `True`; `1/10**100` is tiny but non-zero.
- `reasonable(10**10000)`: `1/n` underflows to `0.0`, so `1/n != 0` is `False`. Result: `False`.

(The lecture noted along the way that `10**1000` is representable as an int, but `1/(10**1000)` rounds to `0.0` because floats only go so small.)

### Example 4: Short-circuiting and false values in `get_name` (from 04.py)

```python
def get_name(first_name, last_name):
    """
    >>> first_name = 'Kay'
    >>> first_name and True
    True
    >>> get_name("", "Ousterhout")
    'Ousterhout'
    """
    if first_name != "":
        return first_name
    else:
        return last_name
```

`'Kay' and True` evaluates to `True`: the left operand `'Kay'` is a non-empty string, hence a true value, so the value of the whole expression is the value of the right operand. Note it did not return `'Kay'`.

Because `''` is a false value, this function's body is equivalent to `return first_name or last_name` (extra context: the lecture showed the `and`/`or` rules and this function separately; combining them is the natural next step and a classic 61A refactor). If `first_name` is `''`, `or` moves on and returns `last_name`; otherwise it returns `first_name`.

### Example 5: `same_length` and what DRY is complaining about (from 04.py)

```python
def same_length(a, b):
    """Return whether a and b have the same number of digits.

    >>> same_length(4, 8)
    True
    >>> same_length(100, 657)
    True
    >>> same_length(4, 100)
    False
    """
    a_digits = 0
    while a > 0:
        a_digits = a_digits + 1
        a = a // 10

    b_digits = 0
    while b > 0:
        b_digits = b_digits + 1
        b = b // 10

    return a_digits == b_digits
```

The digit-counting loop appears twice, character for character except for the names. That is a DRY violation. The fix is to give the repeated process its own function with exactly one job:

```python
def count_digits(n):
    """Return the number of digits in non-negative integer n."""
    digits = 0
    while n > 0:
        digits = digits + 1
        n = n // 10
    return digits

def same_length(a, b):
    return count_digits(a) == count_digits(b)
```

(extra context: the refactored version is the natural application of the lecture's DRY slide; the file itself only contains the repeated version.)

Also note that `a = a // 10` inside the function rebinds the *local* name `a`; the caller's value is unaffected, which is why it is safe to destroy the parameter while looping.

### Example 6: Generalizing over a constant (areas)

Starting point, with repetition:

```python
from math import pi, sqrt

def area_square(r):
    return r * r

def area_circle(r):
    return r * r * pi

def area_hexagon(r):
    return r * r * 3 * sqrt(3) / 2
```

`area_hexagon(10)` is about 259.8, which is right, but `area_hexagon(-10)` returns the same thing, which is not. Fix with an `assert`:

```python
def area_square(r):
    assert r > 0, 'A length must be positive'
    return r * r
```

Now that assertion needs to be copy-pasted into all three, which violates DRY. Factor out everything the three share (the assertion and `r * r`) and parameterize the one thing that differs (the constant):

```python
def area(r, shape_constant):
    assert r > 0, 'A length must be positive'
    return r * r * shape_constant

def area_square(r):
    return area(r, 1)

def area_circle(r):
    return area(r, pi)

def area_hexagon(r):
    return area(r, 3 * sqrt(3) / 2)
```

`area_hexagon(10)` gives the same answer as before; `area_hexagon(-10)` now raises `AssertionError: A length must be positive`. Note that `area` by itself is not very intuitive and needs documentation, which is typical of a generalized helper.

### Example 7: Generalizing over a process (`summation`)

The two functions we want to unify:

```python
def sum_naturals(n):
    """Sum the first n natural numbers.

    >>> sum_naturals(5)
    15
    """
    total, k = 0, 1
    while k <= n:
        total, k = total + k, k + 1
    return total

def sum_cubes(n):
    """Sum the first n cubes of natural numbers.

    >>> sum_cubes(5)
    225
    """
    total, k = 0, 1
    while k <= n:
        total, k = total + pow(k, 3), k + 1
    return total
```

These are identical except for `k` versus `pow(k, 3)`. The difference is an *expression in k*, so we capture it as a function of a single argument:

```python
def identity(k):
    return k

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

And the originals collapse:

```python
def sum_naturals(n):
    return summation(n, identity)

def sum_cubes(n):
    return summation(n, cube)
```

**Vocabulary check from the slides.** `cube` is a function of a single argument, and it is **not called `term`**: it is called `cube`, and it cubes numbers. `term` is a **formal parameter of `summation`** that will be bound to a function. At the call site `summation(5, cube)`, the `cube` function is passed as an argument *value*. Inside the body, `term(k)` is where the function bound to `term` actually gets called. Deciding *which* function to apply to `k` happens at the call expression for `summation`, not inside it.

**Environment reasoning for `summation(5, cube)`, in words:**

1. Evaluate the operator `summation` in the global frame: it looks up the name and finds a function value (`func summation(n, term)`).
2. Evaluate the operands left to right: `5` is a numeral evaluating to 5; `cube` is a **name**, looked up in the global frame, evaluating to the function value `func cube(k)`. It is *not* called here. No parentheses, no call.
3. Apply: add a local frame for `summation`. In it, `n` is bound to `5` and `term` is bound to the same function object that the global name `cube` is bound to. In a box-and-pointer picture, two arrows (`cube` in global, `term` in the local frame) point at one function object.
4. Execute the body in that new environment. `total` and `k` are bound to `0` and `1` in the same local frame.
5. Each time through the loop, `term(k)` is a call expression: the operator `term` is looked up in the local frame, yielding the cube function, and the operand `k` is looked up, yielding the current index. Applying it creates a **new local frame for `cube`** with `k` bound to that index, which returns `pow(k, 3)`, and then that frame's work is done.
6. The loop accumulates `0 + 1 + 8 + 27 + 64 + 125 = 225`, which is returned.

Note there are now two different names `k` alive in two different frames: `k` in `summation`'s frame (the loop index) and `k` in `cube`'s frame (the argument). They are independent; a frame's bindings are local to that frame.

**The pi series.** The same `summation` works on a term function nobody anticipated when it was written:

```python
from math import pi

def pi_term(k):
    return 8 / (k * 4 - 3) / (k * 4 - 1)
```

`summation(1000000, pi_term)` gives about `3.141592...`, converging to pi. That is the payoff of a general method of computation: write the loop once, reuse forever.

(Note: the minus signs in `pi_term` were lost when the slide PDF was converted to text. The transcript states the formula as 8 divided by (4k minus 3) and by (4k minus 1), which is what is written above.)

### Example 8: Returning a function, `make_adder`

```python
def make_adder(n):
    """Return a function that takes one argument k and returns k + n.

    >>> add_three = make_adder(3)
    >>> add_three(4)
    7
    """
    def adder(k):
        return k + n
    return adder
```

**Structure first.** There is a `def` statement inside a `def` statement. `return k + n` is part of the body of `adder`. `return adder` is part of the body of `make_adder`. So `make_adder` returns a **function**; `adder` returns a **number**. The interesting part: `adder` can use both its own formal parameter `k` and the formal parameter `n` of the surrounding function.

**Environment reasoning for `add_three = make_adder(3)` then `add_three(4)`:**

1. `make_adder(3)` creates a local frame for `make_adder` with `n` bound to `3`.
2. The `def adder(k)` statement executes *inside that frame*. It creates a new function value and binds the name `adder` in that local frame. Because the function was defined there, it retains access to that frame, so it can still find `n = 3` later.
3. `return adder` returns that function value. The name `add_three` in the global frame is now bound to it. The function "remembers" that 3 is the thing it adds, because it remembers the frame where `n` is 3.
4. `add_three(4)` creates a new local frame with `k` bound to `4`, and the body `return k + n` finds `k = 4` locally and `n = 3` in the remembered enclosing frame, returning `7`.

(extra context: the formal machinery for "remembers the frame where it was defined," namely parent frames and lexical scoping in environment diagrams, is developed in the next lecture. This lecture states the fact and shows it working.)

**Chained call expressions.** What is `make_adder(1)(2)`?

It is a call expression whose **operator is itself a call expression** and whose operand is the numeral `2`. An operator is any expression that evaluates to a function, which is exactly what `make_adder` gives back.

- Evaluate the operator `make_adder(1)`: evaluate *its* operator `make_adder` (a function), its operand `1`, apply, get back the `adder` function that adds 1.
- Evaluate the operand `2`.
- Apply the adder to `2`, giving `3`.

`make_adder(2000)(13)` gives `2013`. It can equivalently be split into two steps: `f = make_adder(2000)` then `f(13)`.

### Example 9: Twenty-One with pluggable strategies (from 04.py)

**Rules:** two players alternate turns; on each turn a player adds 1, 2, or 3 to the current total. The total starts at 0. The game ends whenever the total is 21 or more. The **last player to add to the total loses.**

A **strategy** is a function: it takes the current score and returns how much to add.

```python
def simple_strategy(score):
    if score < 15:
        return 3
    return 2

def interactive_strategy(score):
    print("Current score", score, "What do you want to play (1-3)?")
    next_play = int(input())
    return next_play
```

The game loop takes two strategies as arguments, so it is a higher-order function:

```python
def play(strategy0, strategy1):
    current_score = 0
    player0 = 0
    player1 = 1
    current_player = player0
    while current_score < 21:
        if current_player == player0:
            # Do player 0's turn
            current_score = current_score + strategy0(current_score)
            current_player = player1
        else:
            # Do player 1's turn
            current_score = current_score + strategy1(current_score)
            current_player = player0
    # Current score has reached 21
    print("Player", current_player, "wins the game")
```

**Why the final print is correct:** after a player moves, `current_player` is switched to the *other* player. So when the loop exits, `current_player` is the player who did **not** make the last move. Since the last player to add loses, the one who did not add last wins. The switch is what makes this one line correct.

And the higher-order function that makes new strategies out of old ones:

```python
def make_heckling_strategy(strategy, heckle):
    def heckling_strategy(s):
        print(heckle)
        return strategy(s)

    return heckling_strategy


play(
    make_heckling_strategy(simple_strategy, "you're going down!"),
    make_heckling_strategy(simple_strategy, "thanks for playing with me!"),
)
```

`make_heckling_strategy` both **takes a function** (`strategy`) and **returns a function** (`heckling_strategy`), so it is doubly higher-order. The returned function has the same interface as any other strategy (takes a score, returns 1, 2, or 3), so `play` neither knows nor cares that it was wrapped. That is separation of concerns in miniature: `play` knows the rules, the strategies know how to choose, and the heckler knows how to trash-talk.

**Trace of the game as written** (both players use the simple strategy): scores go 0, 3, 6, 9, 12, 15 (player 0 moved), then player 1 sees 15 and adds 2 to reach 17, player 0 adds 2 to reach 19, player 1 adds 2 to reach 21. The loop exits with `current_player` equal to `player0`, so it prints `Player 0 wins the game`. Player 1 made the final addition and therefore lost.

---

## Common Pitfalls

1. **Writing `term()` or `cube(k)` when you mean to pass the function.** `summation(5, cube)` passes the function. `summation(5, cube(5))` calls `cube` first and passes the number `125`, then crashes inside `summation` when it tries `term(k)` on an int (`TypeError: 'int' object is not callable`). Parentheses mean "call."

2. **Thinking the parameter name and the function name must match.** `summation`'s parameter is `term`; the function passed in is named `cube`. Inside `summation`, that function is reachable only as `term`. Outside, only as `cube`.

3. **Assuming `and`/`or` return booleans.** `2 and 3` is `3`, not `True`. `0 or ''` is `''`. They return the value of the last subexpression evaluated.

4. **Putting the guard on the wrong side of `and`.** `sqrt(x) > 10 and x > 0` crashes for negative `x`. The protecting condition must come first, because short-circuiting only helps in the left-to-right direction.

5. **Believing you can replace control statements with functions.** Any function you write to "be" an `if` will evaluate all of its arguments. This is the single most tested conceptual point from the first half of this lecture.

6. **Forgetting that a returned inner function captures the enclosing parameter.** `make_adder(3)` does not return `3`; it returns a function. Calling `make_adder(3)` and then ignoring the result does nothing useful. Conversely, `add_three` is not a number, so `add_three + 4` is an error while `add_three(4)` is 7.

7. **Confusing `make_adder(1)(2)` with `make_adder(1, 2)`.** The first is two calls of one argument each; the second is a single call of two arguments, and `make_adder` only takes one, so it raises a `TypeError`.

8. **Splitting a simultaneous assignment.** `pred, curr = curr, pred + curr` is not the same as `pred = curr` followed by `curr = pred + curr`. The right side is evaluated completely before any name is rebound.

9. **Missing that a local `def` does not run its body.** Executing `def adder(k): ...` only creates and names a function. The body runs only when `adder` is called.

10. **Using an `assert` where the condition itself can crash.** The assertion condition is an ordinary expression and is fully evaluated, so it needs the same short-circuit guards any other expression does.

11. **Shadowing confusion with reused names.** `k` in `summation` and `k` in `cube` are different bindings in different frames. Looking a name up always means looking in the current environment, not "the last place I saw that name."

---

## Likely Exam Points

### 1. What does an `and`/`or` expression evaluate to?

Exams love asking for the *value*, not just true/false, and whether an error occurs.

**Practice:** What does each display in the interpreter?
```python
>>> 0 or 'hello'
>>> 'hello' and 0
>>> None or False
>>> 1 and 0 or 2
>>> not ''
```

**Answer:** `'hello'` (0 is false, so return the right side). `0` (left is true, so return the right side, which is `0`). `False` (left is false so evaluate the right; the result is `False`). `2` (`1 and 0` evaluates to `0`, which is false, so `or` returns `2`). `True` (`''` is a false value, and `not` always returns a bool).

### 2. Does this crash? Short-circuit guards

**Practice:** Which of these raise an error when `n = 0`?
```python
def a(n): return 1/n != 0 or n == 0
def b(n): return n == 0 or 1/n != 0
def c(n): return n != 0 and 1/n != 0
def d(n): return 1/n != 0 and n != 0
```

**Answer:** `a` and `d` raise `ZeroDivisionError`, because `1/n` is the left operand and is always evaluated. `b` returns `True` (the `or` short-circuits on `n == 0`). `c` returns `False` (the `and` short-circuits on `n != 0` being `False`). The guard must be on the left.

### 3. `if` statement versus `if_` function

**Practice:** Given `def if_(c, t, f): return t if c else f` (equivalently the `if`/`else` version), what happens when you call `if_(True, 1, 1/0)`?

**Answer:** `ZeroDivisionError`. Evaluating a call expression evaluates *all* operands before applying the function, so `1/0` is evaluated even though `c` is `True` and the function would have returned `t`. An actual `if` statement would have skipped the `else` suite. This is exactly why control statements exist in the language rather than being ordinary functions.

### 4. Is this a higher-order function, and what is its domain/range?

**Practice:** For `make_heckling_strategy(strategy, heckle)`, state its domain, its range, and whether it is higher-order.

**Answer:** Domain: `strategy` is a function that takes one number (a score) and returns a number; `heckle` is a string. Range: a function that takes one number and returns a number (a strategy). It is higher-order twice over: it takes a function as an argument and returns a function.

### 5. What is printed / returned? Chained calls and `make_adder`

**Practice:**
```python
def make_adder(n):
    def adder(k):
        return k + n
    return adder

f = make_adder(10)
g = make_adder(f(5))
print(g(1))
print(make_adder(make_adder(2)(3))(4))
```

**Answer:** `f(5)` is `15`, so `g` adds 15, and `g(1)` prints `16`. For the second: `make_adder(2)` returns an adder-of-2; calling it on `3` gives `5`; `make_adder(5)` returns an adder-of-5; calling it on `4` gives `9`. So `9` is printed.

### 6. Fill in the blank so a higher-order call produces a given value

This is the classic "what expression goes in the blank" question.

**Practice:** Fill in the blanks so that `summation(4, ____)` returns `30`, using only `summation`, arithmetic, and a `def`.

**Answer:** 30 is 1 + 4 + 9 + 16, the sum of the first four squares, so the term function must square its argument:
```python
def square(k):
    return k * k

summation(4, square)   # 30
```
The blank must be an expression that **evaluates to a function of one argument**, so `square` (no parentheses) is correct and `square(k)` is not.

### 7. Trace `summation` with a given term function

**Practice:** What is `summation(3, identity)` where `identity(k)` returns `k`? What is `summation(0, cube)`?

**Answer:** `summation(3, identity)` is `1 + 2 + 3 = 6`. `summation(0, cube)` is `0`: `total` starts at `0`, `k` starts at `1`, and `1 <= 0` is false, so the loop body never runs and `0` is returned.

### 8. Environment/frames: how many frames are created?

**Practice:** When `summation(3, cube)` is evaluated starting from the global frame, how many local frames are created in total, and what is bound in each?

**Answer:** Four. One frame for `summation`, with `n` bound to `3` and `term` bound to the cube function (and later `total` and `k` rebound in that same frame as the loop runs). Then three frames for `cube`, one per loop iteration, each with `k` bound to `1`, `2`, and `3` respectively. Each `cube` frame disappears from active use once it returns its value. The `k` in `cube`'s frames is independent of the `k` in `summation`'s frame.

### 9. Spot the DRY violation and fix it

**Practice:** What is wrong with `same_length` as written, and how would you fix it while giving each function exactly one job?

**Answer:** The digit-counting `while` loop is written twice. Factor it into `count_digits(n)`, which has exactly one job, then write `same_length(a, b)` as `return count_digits(a) == count_digits(b)`. The result is shorter, easier to test, and `count_digits` is reusable in other problems.

### 10. Rewrite a family of functions to remove repetition using a parameter

**Practice:** You have `area_square`, `area_circle`, and `area_hexagon`, each asserting `r > 0`. What is the general method, and why can the shape difference be a plain number here but not in `summation`?

**Answer:** The general method is `area(r, shape_constant)`, which asserts `r > 0` and returns `r * r * shape_constant`; each specific function calls it with `1`, `pi`, or `3 * sqrt(3) / 2`. A number suffices because the three formulas differ only by a multiplicative constant. In the summation case the formulas differ by an entire *expression in k* (a computation, not a value), so the parameter must be a function.

### 11. Twenty-One game logic

**Practice:** In `play`, why does the loop print `current_player` as the winner rather than the other player?

**Answer:** Because `current_player` is switched immediately after each move. When the loop exits, `current_player` names the player whose turn it *would* be, which is the player who did not make the final addition. Since the last player to add loses, that player wins.

### 12. Terminology precision

**Practice:** In `summation(5, cube)`, is `cube` a formal parameter, an argument, or an operand? What about `term`?

**Answer:** `cube` is an **operand** of the call expression, and the function it evaluates to is the **argument** value. `term` is the **formal parameter** of `summation` that gets bound to that argument in `summation`'s local frame. The function's own name is `cube`, not `term`.

---

## Summary

- **Iteration design:** decide what state must persist across iterations, and state the loop invariant. The improved `fib` (`pred, curr = 1, 0`, `k = 0`) works for `n >= 0`, not just `n >= 1`.
- **Call expressions evaluate every operand**, left to right, before applying the function. **Control statements skip code.** This is why `if_(x >= 0, sqrt(x), 0)` crashes on negative `x` but the `if` statement version does not, and why control statements must be part of the language.
- **`and`, `or`, `not` short-circuit.** `and` returns the first false value or the right operand; `or` returns the first true value or the right operand; `not` always returns a bool. They return the value of the last subexpression evaluated, not necessarily `True`/`False`.
- **False values so far:** `False`, `0`, `''`, `None`.
- **Describe a function** by its domain (possible inputs), its range (possible outputs), and, if pure, its behavior (the input/output relationship).
- **Design principles:** each function gets exactly one job but should apply to many related situations (like `round`); don't repeat yourself.
- **`assert <exp>, <msg>`** raises `AssertionError` with the message when the expression is a false value, and does nothing otherwise.
- **Generalize over values** by adding a parameter (`area(r, shape_constant)`). **Generalize over computations** by adding a *function* parameter (`summation(n, term)`).
- **Functions are first-class values:** bindable, passable as arguments, returnable as results.
- **A higher-order function** takes a function as an argument and/or returns a function. `summation` takes one; `make_adder` returns one; `make_heckling_strategy` does both.
- **A local `def`** binds a function in the enclosing call's local frame, and that function can use the enclosing function's formal parameters, which is why `make_adder(3)` returns a function that remembers 3.
- **The operator of a call expression can be any expression that evaluates to a function**, including another call: `make_adder(1)(2)` is `3`.
- **Higher-order functions pay off** by expressing general methods of computation, removing repetition, and separating concerns, which is the programming version of modularity and abstraction in the postal system and in `curl https://cs61a.org`.
- **Twenty-One** demonstrates the payoff concretely: `play` implements the rules once and accepts any strategy function, and `make_heckling_strategy` builds new strategies from old ones without `play` knowing anything about it.
