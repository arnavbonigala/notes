<!-- Wed, Sep 09, 2026 | sources: slides + code (no transcript available) -->
# Lecture 6: Abstraction

This lecture is the payoff of the higher-order function material: it introduces **lambda expressions**, the syntax for writing a function as an *expression* instead of a statement, and then uses them to sharpen your understanding of how Python looks up names inside functions. The recurring example is `summation(n, term)`, a function that abstracts the pattern "add up the first `n` terms of a sequence" and takes the per-term computation as an argument. The lecture then works through three related ideas: **lambda environments** (a function's body looks up free variables in the environment where the function was *defined*, at the moment it is *called*, never in the caller's frame), **currying** (turning a two-argument function into a chain of one-argument functions), and **zero-argument functions** (a function of no arguments as a way to delay a computation, demoed with dice). Finally it revisits the Lab 02 `cake`/`pie`/`snake` problem and the identity that any program using lambdas can be rewritten with `def` statements (but not the reverse). The unifying theme behind the lecture title is abstraction: functions are values, so the varying part of a pattern can be *passed in* rather than hard-coded, and lambda gives you a lightweight way to write that varying part inline.

---

## Key Concepts

### 1. The pattern being abstracted: `summation`

From the previous lecture:

```python
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

The important structural facts, exactly as annotated on the slides:

- `term` is a **formal parameter that will be bound to a function**. It is not the name of any particular function; it is a slot.
- The function passed in must be a **function of a single argument**. Nothing requires that function to be *called* `term`; `cube`, `identity`, and `pi_term` all work.
- `summation(5, cube)` passes **the cube function itself as an argument value**. There are no parentheses after `cube`, so `cube` is not called at that point.
- The passed function gets called inside the loop, at `term(k)`.

`summation(5, cube)` computes `1 + 8 + 27 + 64 + 125 = 225`. Swap the `term` argument and you get a different series with zero changes to `summation`. That is the abstraction: one control structure, many series.

The slides also show `pi_term`:

```python
def pi_term(k):
    return 8 / (k * 4 - 3) / (k * 4 - 1)
```

which gives the series `8/(1*3) + 8/(5*7) + 8/(9*11) + ...`, converging to π. (The minus signs were lost in the PDF extraction; the reconstruction is unambiguous from the standard form of this series.)

### 2. Lambda expressions

A `def` statement is a **statement**: it binds a name to a new function as a side effect. A lambda is an **expression**: it *evaluates to* a function, and that value can be used anywhere a value can be used, including as an argument.

```python
summation(5, lambda x: pow(1/2, x))
```

Read that lambda out loud the way the slides label it: "a function with formal parameter `x` that returns the value of `pow(1/2, x)`."

Syntax rules to internalize:

- **No `return` keyword.** The body is an expression, and its value is automatically returned. Writing `lambda x: return x` is a `SyntaxError`.
- **The body must be a single expression.** No assignment statements, no `while`, no `if` statements (a conditional *expression* like `lambda x: 0 if x < 0 else x` is fine, since that is an expression).
- The parameter list goes between `lambda` and the colon, with no parentheses: `lambda x, y: x + y`.
- Evaluating a lambda expression creates a brand-new function object each time it is evaluated.

### 3. Lambda versus `def`: three equivalent spellings

The slides give three ways to express the same thing:

```python
# 1. Inline, anonymous
summation(5, lambda x: pow(1/2, x))

# 2. Lambda in an assignment statement
term = lambda x: pow(1/2, x)
summation(5, term)

# 3. def statement
def term(x):
    return pow(1/2, x)
summation(5, term)
```

And the directional claim: **all lambda expressions can be rewritten using a `def`, but not vice versa.** The reason is that `def` bodies can contain arbitrary statements (loops, multiple assignments, multiple returns, nested `def`s), while a lambda body is one expression. So `def` is strictly more expressive.

The slide "Lambda and Def" restates this at the top level: *any program containing lambda expressions can be rewritten using def statements*, with the example

```python
>>> (lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)
81
```

rewritten as

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

The slide annotates the first lambda as "twice" and the second as "square", which is the key to reading such expressions: give each anonymous function the name you would have given it with a `def`, then read the calls.

One difference worth knowing (this matters for what Python *displays*): a `def` gives the function an intrinsic name, so it prints as `<function cube at ...>`, while a lambda's intrinsic name is `<lambda>`, so even after `term = lambda x: ...`, `term` prints as `<function <lambda> at ...>`.

### 4. Lambda environments (the heart of the lecture)

This is the part students most often get wrong. Two rules:

1. **When a function is created** (by a `def` statement or by evaluating a lambda expression), the function records as its **parent** the frame in which it was created. A lambda defined at the top level has the global frame as its parent; a lambda defined inside a function body has that function's call frame as its parent.
2. **When a name in a function body is not a parameter and not assigned locally** (a *free variable*), it is looked up at **call time**, starting in the current call frame, then in the parent frame, then that frame's parent, and so on up to the global frame and then the builtins. It is **never** looked up in the frame of whatever code called the function.

The Python Tutor demo linked from the slides is designed precisely to test rule 2:

```python
common_ratio = 1/2
term = lambda x: pow(common_ratio, x)
common_ratio = 1/3

def summation(n, term):
    common_ratio = 1/4
    total, k = 0, 1
    while k <= n:
        total, k = total + term(k), k + 1
    return total

common_ratio = 1/5
print(summation(5, term))
```

There are four different values assigned to a name `common_ratio`, and exactly one of them matters. The lambda's parent is the **global** frame, so each time `term(k)` runs, `common_ratio` is looked up in the global frame *as it is at that moment*, which is `1/5`. The `common_ratio = 1/4` inside `summation` is a purely local binding in `summation`'s frame and is invisible to `term`, even though `term` is called from inside `summation`. The `1/2` and `1/3` were overwritten before the call. So the result is `0.2 + 0.04 + 0.008 + 0.0016 + 0.00032 = 0.24992`.

The lecture code file `06.py` shows the contrasting design, where the value *is* captured:

```python
common_ratio = 1/2

def get_term(common_ratio):
    def term(k):
        return pow(common_ratio, k)
    return term

common_ratio = 1/3
term = get_term(common_ratio)
```

Here `term`'s parent is the frame for the call to `get_term`, and in that frame `common_ratio` is a **parameter** bound to `1/3` (the value of the global `common_ratio` at the moment of the call). Later reassignments to the global `common_ratio` cannot reach it, because lookup from `term` finds `common_ratio` in `get_term`'s frame and stops there. This is the standard way to "freeze" a value into a returned function.

### 5. Currying

> Convert a function that takes multiple arguments into a chain of functions that each take a single argument.

```python
pow(1/2, 5)        # 0.03125
curry(pow)(1/2)(5) # 0.03125
```

From `06.py`:

```python
def curry(f):
    def g(x):
        def h(y):
            return f(x, y)
        return h
    return g
```

Read it inside out. `curry(f)` returns `g`, which remembers `f` (parent = `curry`'s frame). Calling `g(x)` returns `h`, which remembers both `x` (parent = `g`'s frame) and `f` (grandparent = `curry`'s frame). Calling `h(y)` finally does the work: `f(x, y)`.

Why this is an abstraction tool: `summation` demands a **one-argument** function, but `pow` takes two. Currying adapts `pow` to that interface without writing a new helper:

```python
print(summation(5, curry(pow)(3)))
```

`curry(pow)(3)` is a one-argument function `y -> pow(3, y)`, so this sums `3 + 9 + 27 + 81 + 243 = 363`.

### 6. Zero-argument functions

A lambda or `def` with no parameters is still a function; calling it re-runs its body. Two uses:

- **Delayed evaluation.** `lambda: expensive()` packages up a computation to be run later, possibly never, possibly many times.
- **Re-running something non-deterministic.** The dice demo is the motivating case: a "die" is naturally represented as a zero-argument function you call to get a roll, so each call can produce a different value. (The exact demo code is not in the provided material; in CS 61A this is conventionally something like `def make_dice(n): return lambda: randint(1, n)`. *(extra context)*)

Compare with a plain number: `four = 4` is a fixed value, while `four = lambda: 4` is a function you must call, and `four()` recomputes the body every time.

### 7. Function values, names, and `==`: the `cake`/`pie`/`snake` problem

Lab 02 Q2 is revisited to drive home that a function is an ordinary value bound to a name, and that rebinding the name does not change the function object. See the worked example below for the full trace.

---

## Definitions

- **Lambda expression**: an expression of the form `lambda <params>: <expression>` that evaluates to a new function with those formal parameters, whose body is the single expression, and whose return value is the value of that expression. It contains no `return` keyword and binds no name.
- **Anonymous function**: a function value that is not bound to any name by its creation. Lambdas create anonymous functions (they may later be bound by assignment).
- **Intrinsic name**: the name recorded inside a function object, used when Python displays it. A `def` statement supplies the defined name; a lambda expression's intrinsic name is `<lambda>`.
- **Higher-order function**: a function that takes a function as an argument, returns a function as its value, or both. `summation`, `curry`, `get_term`, and `cake` are all higher-order functions.
- **Formal parameter**: a name in a function's parameter list, bound to an argument value in the function's call frame when it is called. `term` in `summation` is a formal parameter bound to a function.
- **Frame**: a record of the name bindings created by one call to one function, plus a link to its parent frame.
- **Parent frame (of a function)**: the frame in which the function was created. Every function value carries this link; it determines name lookup for the function's body.
- **Environment**: a sequence of frames, starting with a local frame and following parent links up to the global frame. Name lookup proceeds along this sequence.
- **Free variable**: a name used in a function body that is neither a parameter of that function nor assigned within it. Free variables are resolved in the function's parent environment, at call time.
- **Lexical (static) scope**: the rule that a function's free variables are resolved using the environment where the function was *defined*, not where it was *called*.
- **Currying**: transforming a function of n arguments into a chain of n functions, each taking one argument, such that `f(a, b)` becomes `curry(f)(a)(b)`.
- **Zero-argument function**: a function with an empty parameter list, called as `f()`, typically used to delay or repeat a computation.
- **Call expression**: an expression of the form `<operator>(<operands>)`. It is evaluated by evaluating the operator subexpression, then each operand subexpression, then applying the resulting function to the resulting argument values. The operator may itself be a call, as in `curry(pow)(3)`.

---

## Worked Examples

### Example 1: `summation(5, lambda x: x)`

```python
sum = summation(5, lambda x: x)
print(sum)
```

Step by step:

1. Evaluate the operands of the call. `5` is `5`. The lambda expression is evaluated, creating a new function with parameter `x` and body `x`, parent frame = global.
2. Call `summation`: a new frame binds `n = 5` and `term` to that lambda function.
3. The loop runs with `k = 1..5`, each iteration computing `total + term(k)`, that is, `total + k`.
4. `1 + 2 + 3 + 4 + 5 = 15`, so `15` is printed.

This lambda is exactly the `identity` function from the slides, written inline instead of with `def`.

*(Pitfall in this snippet, extra context: naming the variable `sum` shadows the builtin `sum` in the global frame. Harmless here, but avoid it in your own code.)*

### Example 2: filling in the blank

The slide asks:

```python
sum = summation(5, ________)
print(sum)
```

for "this series", whose diagram did not survive PDF extraction. The two candidates from the surrounding slide content are the cubes and the π series, so both answers are worth being able to write:

```python
# cubes: 1 + 8 + 27 + 64 + 125 = 225
summation(5, lambda k: pow(k, 3))

# the pi series: 8/(1*3) + 8/(5*7) + ... ~= 3.0
summation(5, lambda k: 8 / (k * 4 - 3) / (k * 4 - 1))
```

Each is a direct lambda translation of `cube` and `pi_term` respectively. (Which one the slide intended is a reconstruction; the mechanics of writing either are what is being tested.)

### Example 3: `summation(5, lambda x: pow(1/2, x))`

The geometric series with ratio 1/2:

`0.5 + 0.25 + 0.125 + 0.0625 + 0.03125 = 0.96875`

Note that `1/2` is evaluated *inside* the lambda body, so it is recomputed on each call. That is fine here since it is a constant, but it is the setup for the environments demo, where the ratio is a *name* instead of a literal.

### Example 4: the lambda environments demo, traced in words

```python
common_ratio = 1/2
term = lambda x: pow(common_ratio, x)
common_ratio = 1/3

def summation(n, term):
    common_ratio = 1/4
    total, k = 0, 1
    while k <= n:
        total, k = total + term(k), k + 1
    return total

common_ratio = 1/5
print(summation(5, term))
```

Environment reasoning:

- Global frame after the first three lines: `common_ratio: 1/3`, `term: <function <lambda>>` whose **parent is the global frame**. Crucially, the lambda's body was *not* evaluated, so nothing about `1/2` was captured; only the parent link was recorded.
- `def summation` adds `summation` to the global frame (parent: global).
- `common_ratio = 1/5` rebinds the global name a final time. The lambda is unaffected as an object, but the value it will *find* has changed.
- `summation(5, term)` creates frame `f1` with parent global, binding `n: 5`, `term: <the lambda>`, then `common_ratio: 1/4` (a **local** binding in `f1`, not a mutation of the global one), then `total` and `k`.
- Each `term(k)` creates a frame `f2` with parent **global** (the lambda's parent), not `f1`. Looking up `common_ratio` in `f2` fails locally, so lookup goes to `f2`'s parent, the global frame, and finds `1/5`.
- Result: `0.2 + 0.04 + 0.008 + 0.0016 + 0.00032 = 0.24992`.

The moral, worth memorizing: **the calling frame is not part of the callee's environment.** A picture in words: frames `f1` and `f2` are siblings, both pointing up at global; there is no arrow from `f2` to `f1`.

### Example 5: `06.py` end to end

```python
def cube(k):
    return pow(k, 3)

common_ratio = 1/2

def get_term(common_ratio):
    def term(k):
        return pow(common_ratio, k)
    return term

common_ratio = 1/3
term = get_term(common_ratio)

def summation(n, term):
    """Sum the first n terms of a sequence.

    >>> summation(5, cube)
    225
    """
    common_ratio = 1/4
    total, k = 0, 1
    while k <= n:
        total, k = total + term(k), k + 1
    return total

common_ratio = 1/5
print(summation(5, term))
```

1. `common_ratio = 1/2` in global, then `get_term` defined, then `common_ratio = 1/3`.
2. `get_term(common_ratio)` first evaluates the operand `common_ratio` in the global frame, giving `1/3`. A frame `f1` for `get_term` binds its **parameter** `common_ratio: 1/3`. Note the parameter name deliberately shadows the global name.
3. Inside `f1`, the `def term` statement creates a function whose **parent is `f1`** and binds it to `term` in `f1`. `get_term` returns that function, which is bound to the global name `term`.
4. `common_ratio = 1/5` changes the global binding. `term` does not care: lookup from `term`'s frame goes to `f1`, finds `common_ratio: 1/3`, and stops before ever reaching global.
5. `summation(5, term)` sums `(1/3)^1 + ... + (1/3)^5 = 121/243`, printing `0.4979423868312757`.

Contrast with Example 4: the only structural change is that the ratio became a *parameter* of an enclosing function, and that single change is what makes the value stick.

Then:

```python
def curry(f):
    def g(x):
        def h(y):
            return f(x, y)
        return h
    return g

print(summation(5, curry(pow)(3)))
```

6. `curry(pow)`: frame for `curry` binds `f: pow`; returns `g` (parent = that frame).
7. `curry(pow)(3)`: the operator subexpression `curry(pow)` is evaluated first (yielding `g`), then `g(3)` runs in a new frame binding `x: 3`, returning `h` (parent = that frame). `h` is a one-argument function.
8. `summation(5, h)` calls `h(1) ... h(5)`. Inside `h`, `y` is local, `x` is found in `g`'s frame (`3`), and `f` is found one level further up in `curry`'s frame (`pow`). So `h(k) == pow(3, k)`.
9. `3 + 9 + 27 + 81 + 243 = 363` is printed.

### Example 6: Lab 02 Q2, the full transcript

```python
>>> def cake():
...    print('beets')
...    def pie():
...        print('sweets')
...        return 'cake'
...    return pie
...
>>> chocolate = cake()
beets
>>> chocolate
<function cake.<locals>.pie at ...>
>>> chocolate()
sweets
'cake'
```

- `cake()` executes the body: prints `beets`, defines `pie` locally, and **returns the function `pie` without calling it**. So `beets` appears once, and `chocolate` is the `pie` function object. Displaying it shows its repr, which includes `cake.<locals>.pie` because `pie` was defined inside `cake`.
- `chocolate()` calls `pie`: prints `sweets` (side effect, appears first) and returns the string `'cake'`, which the REPL displays with quotes. Distinguishing "printed" from "returned and displayed" is exactly what the exam tests.

```python
>>> more_chocolate, more_cake = chocolate(), cake
sweets
>>> more_chocolate
'cake'
```

- The right-hand side is a tuple of two expressions, evaluated left to right: `chocolate()` prints `sweets` and yields `'cake'`; `cake` is just a name lookup yielding the `cake` function (no call, no `beets`). So `more_chocolate` is the string `'cake'` and `more_cake` is the `cake` function.

```python
>>> def snake(x, y):
...    if cake == more_cake:
...        return chocolate
...    else:
...        return x + y
...
>>> snake(10, 20)
<function cake.<locals>.pie at ...>
>>> snake(10, 20)()
sweets
'cake'
>>> cake = 'cake'
>>> snake(10, 20)
30
```

- `cake` and `more_cake` are free variables in `snake`, looked up in the global frame **at call time**.
- First `snake(10, 20)`: both names are bound to the same function object, so `cake == more_cake` is `True` and `chocolate` (the `pie` function) is returned and displayed.
- `snake(10, 20)()` calls that returned function: prints `sweets`, returns `'cake'`.
- After `cake = 'cake'`, the global name `cake` refers to a string, which is not equal to the function in `more_cake`, so the comparison is `False` and `10 + 20 = 30` is returned. The function object itself never changed; only what the name `cake` points to did.

### Example 7: nested lambda application

```python
>>> (lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)
81
```

1. The whole thing is a call expression whose operator is itself a call expression. Evaluate the innermost call first: `(lambda f: lambda x: f(f(x)))(lambda y: y * y)`.
2. Operator: a function of `f` that returns a function. Operand: a new function `square` = `lambda y: y * y`, parent global.
3. Calling it creates a frame binding `f: square`. Its body is another lambda expression, which is evaluated *now*, creating a function `twice_of_f` = `lambda x: f(f(x))` whose **parent is that frame**. That is returned.
4. Apply `twice_of_f` to `3`: a new frame binds `x: 3`. Evaluating `f(f(x))` looks up `f` in the parent frame, finding `square`. Inner call: `square(3) = 9`. Outer call: `square(9) = 81`.
5. The value is `81`.

The `def`-based rewrite (`twice`/`square`/`g` on the slide) has exactly the same frame structure; only the intrinsic names differ.

---

## Common Pitfalls

1. **Writing `return` inside a lambda.** `lambda x: return x * x` is a `SyntaxError`. The value of the body expression *is* the return value.
2. **Trying to put statements in a lambda.** No `while`, no `for`, no assignment (`=`), no multiple lines. If you need those, use `def`. A conditional *expression* (`a if c else b`) is allowed because it is an expression.
3. **Assuming a lambda captures the current value of a free variable.** It captures a *frame*, not a value. In Example 4, the answer is `1/5`, not `1/2` or `1/3`. To freeze a value, make it a parameter of an enclosing function (Example 5).
4. **Thinking the caller's local names are visible to the callee.** `common_ratio = 1/4` inside `summation` is a red herring and affects nothing. Lookup follows the parent chain from the function's *definition* site.
5. **Confusing `f` with `f()`.** Passing `summation(5, cube())` would call `cube` with no arguments (a `TypeError`); `summation(5, cube)` passes the function. Symmetrically, `chocolate` displays a function while `chocolate()` runs it.
6. **Missing a level of parentheses with curried functions.** `curry(pow)(3)` is a function; `curry(pow)(3)(2)` is `9`; `curry(pow)(3, 2)` is a `TypeError` because `g` takes exactly one argument.
7. **Forgetting that a returned inner function is not called.** `cake()` prints `beets` once and hands back `pie`; `sweets` only appears when the returned function is called.
8. **Mixing up printed output and displayed values in a REPL trace.** `print` output has no quotes and appears in execution order; the displayed return value of the whole expression appears last and shows the repr (quotes on strings, `<function ...>` on functions, nothing at all for `None`).
9. **Expecting a lambda bound by assignment to print its variable name.** `term = lambda x: x` still displays as `<function <lambda> at ...>`.
10. **Believing lambdas are more powerful (or required) because they look advanced.** They are strictly less expressive than `def`. Use them for short, one-expression functions used once, typically as arguments.
11. **Re-evaluating a lambda expression and expecting the same object.** Each evaluation makes a new function, so two structurally identical lambdas are not `==`.

---

## Likely Exam Points

### 1. Translate between lambda and `def`

**Q.** Rewrite `g = lambda x, y: x * y + 1` as a `def` statement, and rewrite the following with a single lambda expression:
```python
def h(n):
    return pow(2, n)
```
**A.**
```python
def g(x, y):
    return x * y + 1

h = lambda n: pow(2, n)
```
Note the `return` appears only in the `def` version.

### 2. "What would Python display?" with functions and prints

**Q.** What does this display?
```python
>>> def f():
...     print('a')
...     def g():
...         print('b')
...         return 'c'
...     return g
...
>>> x = f()
>>> x
>>> x()
```
**A.**
```
a                                   # printed by the call f()
<function f.<locals>.g at ...>      # x is the function g, displayed
b                                   # printed by the call x()
'c'                                 # returned value, displayed with quotes
```

### 3. Free-variable lookup / lambda environments

**Q.** What does this print?
```python
n = 1
f = lambda x: x * n
def g(n):
    n = 100
    return f(n) + n
n = 10
print(g(2))
```
**A.** `110`. `f`'s parent is the global frame, so `n` inside `f` is the global `n`, which is `10` at call time. Inside `g`, the local `n` is `100`, so `f(100) = 100 * 10 = 1000`... careful: `f(n)` passes the local `n = 100` as `x`, giving `100 * 10 = 1000`, then `+ n` adds the local `100`, so the result is `1100`. The `n = 100` in `g` is local and invisible to `f`; only the global `n = 10` is used for `f`'s free variable. (Work these arithmetically and slowly; the trap is using `g`'s local `n` inside `f`.)

### 4. Currying

**Q.** Given the lecture's `curry`, what are the values of `curry(pow)(2)(10)` and `curry(lambda a, b: a - b)(1)(4)`? What is the type of `curry(pow)(2)`?
**A.** `1024` and `-3`. `curry(pow)(2)` is a function (of one argument `y`, returning `pow(2, y)`). Also be ready to *write* `curry`, and to write a curried version by hand: `lambda x: lambda y: f(x, y)`.

### 5. Making a two-argument function fit a one-argument interface

**Q.** Using `summation` (which requires a one-argument `term`), write an expression that computes `2**1 + 2**2 + 2**3 + 2**4 + 2**5` in two ways: once with a lambda, once with `curry` and `pow`.
**A.** `summation(5, lambda k: pow(2, k))` and `summation(5, curry(pow)(2))`. Both give `62`.

### 6. Capturing a value in a returned function

**Q.** Write `make_adder(n)` (from the slide's docstring: "Return a function that takes one argument `k` and returns `k + n`") as a one-line `return` of a lambda. Then explain why the returned function still works after the global name `n` changes.
**A.**
```python
def make_adder(n):
    return lambda k: k + n
```
The returned lambda's parent is the frame for the `make_adder` call, where `n` is a parameter bound to the argument value. Lookup of `n` stops there and never reaches the global frame.

### 7. Nested lambdas applied immediately

**Q.** Evaluate `(lambda x: lambda y: x(y(2)))(lambda a: a + 1)(lambda b: b * 10)`.
**A.** `21`. The first call binds `x` to the increment function and returns `lambda y: x(y(2))`. The second call binds `y` to the times-ten function. Then `y(2) = 20` and `x(20) = 21`.

### 8. Zero-argument functions and delayed evaluation

**Q.** What is the difference between `a = 6 * 7` and `a = lambda: 6 * 7`? How do you get `42` from each?
**A.** The first evaluates `6 * 7` immediately and binds `a` to `42`; write `a`. The second binds `a` to a function and does not multiply anything yet; write `a()`, which recomputes the body on each call. Zero-argument functions matter when the body has a side effect or is non-deterministic (the dice demo), or when you want to postpone work.

### 9. Function identity versus names

**Q.** After
```python
def f():
    return 1
g = f
f = 2
```
what is `g()`, and what is `f()`?
**A.** `g()` is `1`; `g` still refers to the original function object. `f()` raises a `TypeError` because `f` is now the integer `2`. This is the `cake`/`more_cake` point from Lab 02 Q2.

*(The slides also reference Fall 2022 Midterm 1 Question 4(a) as a poll; the problem text is not in the provided material, so practice the categories above instead.)*

---

## Summary

- `summation(n, term)` abstracts a pattern: the loop stays fixed, the per-term function is a parameter. `summation(5, cube)` is `225`; `summation(5, pi_term)` approximates π.
- A lambda expression `lambda <params>: <expr>` **evaluates to** a function. No `return` keyword, body is exactly one expression, no name bound.
- `def` is a statement with a side effect (binding a name) and can contain any statements. Every lambda can be rewritten as a `def`; not every `def` can be rewritten as a lambda.
- Three equivalent forms: inline lambda argument, lambda bound by assignment, `def`. A lambda's intrinsic name is `<lambda>`, so it prints differently.
- Every function records its **parent frame** = the frame where it was created. Free variables are resolved along that parent chain **at call time**, never in the caller's frame.
- Consequence 1 (slide demo): a top-level lambda reading a global `common_ratio` sees the value at call time (`1/5`), ignoring earlier values and a same-named local in `summation` (`1/4`). Result `0.24992`.
- Consequence 2 (`06.py`): making the value a **parameter** of an enclosing `get_term` freezes it (`1/3`), so later global reassignment is irrelevant. Result `121/243 = 0.4979...`.
- **Currying** converts `f(x, y)` into `curry(f)(x)(y)` via nested one-argument functions: `curry(pow)(1/2)(5) == pow(1/2, 5)`. It lets a two-argument function satisfy a one-argument interface: `summation(5, curry(pow)(3)) == 363`.
- **Zero-argument functions** delay or repeat computation; each call re-runs the body, which is why a die is naturally a function of no arguments.
- Lab 02 Q2 lessons: returning an inner function does not call it; the REPL shows printed output before the displayed return value; comparisons like `cake == more_cake` depend on what the names refer to at call time, and rebinding a name never changes the function object.
- Reading tools for exams: name the anonymous functions, evaluate operator before operands, and draw the parent links before chasing any variable.
