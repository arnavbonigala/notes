<!-- Wed, Oct 28, 2026 | sources: slides (no transcript available) -->
# Lecture 27: Functional Programming

This lecture starts a new unit of the course by stepping outside Python and into **Gleam**, a small functional language. Where imperative programming treats a program as instructions for updating memory and object-oriented programming treats it as a collection of interacting objects, *functional* programming treats a program as **data definitions plus pure functions that generate data**. Gleam makes that perspective concrete: every function body is a single expression, there is no `return`, there are no `for`/`while` statements, all values are immutable, functions are first-class values, and the only conditional form is the `case` expression (optionally with `if` guards). Because nothing is mutated, a call can be understood purely by **substitution**: replace each formal parameter with its argument and evaluate the body. The lecture then uses that substitution model to distinguish recursive processes that *grow* (non-tail recursion, like `n + sum_first(n-1)`) from recursive processes of *fixed size* (tail recursion, like `sum_from(n, k+1, total+k)`), and shows that Gleam compiles tail calls into actual `while` loops in JavaScript. It closes with two Gleam features that generalize ideas you already know from Python: higher-order functions with explicit **function types** (`fn(Int, Int) -> Int`), and **generic functions** whose lowercase type names (`t`) stand for whatever type the caller uses.

---

## Key Concepts

### 1. Three perspectives on what a program *is*

- **Imperative:** programs are instructions for updating memory. (Python assignment statements, `while` loops, mutating a list.)
- **Object-oriented:** programs define objects that behave like the entities they represent. (Python classes.)
- **Functional:** programs are data definitions and *pure* functions that generate data.

The slides stress these are **not mutually exclusive**: ideas from each can be combined, and real languages borrow from all three. Python itself supports all three styles; Gleam is chosen here because it *only* supports the functional one, which makes the ideas unavoidable rather than optional.

The unit ahead (from the "What's Next?" slide): algebraic data types and pattern matching (as in Rust, Swift, TypeScript), working effectively with immutable data (Rust, Erlang, Scala), value-based error handling (Rust, Go, Elm), then interpreters (implementing and controlling the rules for how programs run), then applications (program design, web technologies, AI-assisted software development).

### 2. Why Gleam

Characteristics that make it functional:

- Program behavior is just functions, and **function bodies are expressions**.
- Functions are **first-class values**: they can be passed as arguments and returned.
- **Anonymous functions** are convenient to create and used liberally.
- **All values are immutable**, and there are no `for`/`while` loops that update variables.

Why this language in particular: it is small and simple (only functions, names, and one conditional form, `case`); it is a nice vehicle for learning algebraic data types and pattern matching; and it prepares you for larger languages like Rust, Swift, and TypeScript. Programs are organized into **modules** (files); for now everything lives in one module. Code is run in the playground at `https://playground.gleam.run/`.

### 3. Everything is an expression

> An expression is evaluated in an environment (that gives names meaning) to produce a value.

That sentence is the same definition of evaluation you have used since Lecture 1; what changes is that in Gleam *there is nothing else*. The forms of expression the lecture lists:

| Form | Example |
|---|---|
| Literal | `2`, `3.3`, `True`, `"Hello"` |
| Call | `f(2, 3)` |
| Let | `let x = 4` |
| Echo | `echo x` |
| Case | `case x > 3 { True -> "big" False -> "not big" }` |

`echo` displays any value, similar to Python's `print`. A module must have a `main` function that is accessible to users of the module, marked `pub`:

```gleam
pub fn main() {
  echo 61
}
```

Since `let` and `echo` are themselves expressions, there is no statement/expression distinction to keep track of, which is exactly why substitution works as an evaluation model later.

### 4. No `return`; the body's value is the call's value

> No `return` in Gleam; the value of a call expression is the value of its body expression.

```python
def sum_squares(x, y):
    return x * x + y * y
```

```gleam
fn sum_squares(x, y) {
  x * x + y * y
}
```

A Python function with *multiple* `return` statements becomes a Gleam function whose body is a single `case` expression, because a `case` expression yields exactly one value:

```python
def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n - 2) + fib(n - 1)
```

```gleam
fn fib(n) {
  case n {
    0 -> 0
    1 -> 1
    _ -> fib(n - 2) + fib(n - 1)
  }
}
```

Read `case n { ... }` as: match the value of `n` against each **pattern** in order, and the value of the whole expression is the value of the first matching branch's result. `_` is the wildcard pattern that matches anything, so it plays the role of `else`.

### 5. `case` patterns, alternation, and guards

```gleam
pub fn describe(n: Int) -> String {
  case n {
    _ if n < 0 -> "Negative"
    0 -> "None"
    1 -> "One"
    2 | 3 | 4 -> "Few"
    _ -> "Many"
  }
}
```

Three things to notice:

1. **There is `if` in Gleam**, but it is not a statement or a standalone expression: it introduces a **guard** on a pattern inside a `case`. A branch applies only if its pattern matches *and* its guard is true.
2. `2 | 3 | 4` is **alternation**: one branch for several patterns.
3. Branches are tried **in order**, so `_ if n < 0` being first is what makes negative numbers report `"Negative"` rather than falling through to `"Many"`.

Type annotations like `(n: Int) -> String` are optional in general but shown here.

Because a guard can test anything in scope, the value you `case` on does not have to be the value you test. The lecture makes this point with two equivalent `sign` functions:

```gleam
pub fn sign(n: Int) -> Int {       pub fn sign(n: Int) -> Int {
  case n {                           case Nil {
    _ if n > 0 -> 1                    _ if n > 0 -> 1
    _ if n < 0 -> -1                   _ if n < 0 -> -1
    _ -> 0                             _ -> 0
  }                                  }
}                                  }
```

In the left version there is **no reference to `n`** in any pattern (only in the guards), so the subject of the `case` is irrelevant and can be replaced by `Nil`. **`Nil`** is described in the slides as "a value that is returned by functions that have nothing else to return, as all functions must return something." It is Gleam's analogue of Python's `None`, and here it is used as a dummy subject so the `case` is really just a chain of guards.

### 6. Higher-order functions and function types

- An **anonymous function** is created with `fn`: `fn(x, y) { x + y }`.
- A **function type** is written with `fn` too: `fn(Int, Int) -> Int`.

The two are used together to build higher-order functions:

```gleam
fn make_adder(n: Int) -> fn(Int) -> Int {
  fn(k) { n + k }
}
```

The return type `fn(Int) -> Int` is "a one-argument function of integers." `make_adder(3)(4)` evaluates to `7`: calling `make_adder` returns a function, which is then called on `4`. This is exactly Python's

```python
def make_adder(n):
    return lambda k: n + k
```

and the returned `fn(k) { n + k }` is a **closure**: it carries the environment in which it was created, which is where `n` is found.

Taking a function as a parameter looks like this:

```gleam
fn combine(x: Int, y: Int, z: Int, f: fn(Int, Int) -> Int) -> Int {
  f(f(x, y), z)
}
```

`combine(2, 3, 4, fn (x, y) { x * y })` computes `f(f(2, 3), 4)` = `(2 * 3) * 4` = `24`. The annotation `f: fn(Int, Int) -> Int` is "a two-argument function of integers," and it is *checked at compile time*, which is new relative to Python: passing a one-argument function here is a compile error, not a runtime one.

### 7. `let`, scope, and why rebinding is not mutation

`let` assigns a value to a local name:

```gleam
fn sum_squares(x, y) {
  let x2 = x * x
  let y2 = y * y
  x2 + y2
}
```

The crucial claim on this slide is that **each `let` introduces a new scope, equivalent to a new local frame**, and that the `let` form is *equivalent to* an immediately-called anonymous function:

```gleam
fn sum_squares(x, y) {
  fn (x2) {
    fn (y2) {
      x2 + y2
    }(y * y)
  }(x * x)
}
```

Read from the inside out: the whole body is a call of `fn (x2) { ... }` on `x * x`, and inside it a call of `fn (y2) { x2 + y2 }` on `y * y`. Each call creates a frame binding one name, so `let x2 = ...` followed by more code is literally "make a frame where `x2` means this value, then evaluate the rest in it." This is the desugaring to remember: **`let` is a nested lambda**, so Gleam needs no notion of assignment-as-mutation at all.

The consequence is shown by the side-by-side closure example:

```python
>>> x = 3
>>> f = lambda: x
>>> x = 4
>>> f()
4
```

```gleam
pub fn main() {
  let x = 3
  let f = fn () {x}
  let x = 4
  echo f()
}
```

In Python, `x = 4` *rebinds the same global frame*, and `f` looks up `x` only when called, so it prints `4` (this output is on the slide). In Gleam, `let x = 4` does not change anything: it opens a *new* scope that happens to also have a name `x`. The function `f` was created in the earlier scope, and its parent environment still binds `x` to `3`, so `echo f()` displays `3`. *(Reconstruction: the slide shows only Python's `4`; the Gleam output follows from the stated rule that each `let` introduces a new frame. The point of putting the two programs side by side is that the answers differ.)* This is **shadowing**, not mutation: no existing binding is ever overwritten, which is precisely the property that makes substitution a valid evaluation model.

### 8. Blocks

```gleam
/// Compute a*x*x + b*x + c
pub fn quadratic(x: Int, a: Int, b: Int, c: Int) -> Int {
  let ax2 = {
    let x2 = x * x
    a * x2
  }

  // ax2 is in scope here, but x2 is not
  ax2 + b * x + c
}
```

Three rules from the slide:

1. Blocks can contain multiple lines, and they **evaluate to the value of their last expression**.
2. A `let` inside a block **does not affect expressions outside** the block (hence `x2` is not in scope after the block, while `ax2`, bound outside it, is).
3. Blocks are used to **change the order of evaluation in arithmetic**, in place of parentheses:

```gleam
let celsius = { fahrenheit - 32 } * 5 / 9
echo celsius
```

So `{ ... }` serves double duty: grouping a sequence of `let`s into one value, and grouping a subexpression the way parentheses do in Python. Note that `/` on `Int` is integer division. *(Extra context: `///` is a documentation comment and `//` an ordinary comment, which is why the `quadratic` docstring is written with `///`.)*

### 9. Iteration must be recursion

> Gleam has no `for`/`while` statements, so recursion is required to iterate.

```python
>>> def sum_first(n):
...     k = 1
...     total = 0
...     while k <= n:
...         k, total = k + 1, total + k
...     return total
...
>>> sum_first(5)
15
```

```gleam
fn sum_first(n) {
  sum_from(n, 1, 0)
}

fn sum_from(n, k, total) {
  case k <= n {
    True -> sum_from(n, k + 1, total + k)
    False -> total
  }
}
```

The translation recipe is worth internalizing: **the loop variables become parameters**, the loop condition becomes the `case` subject, the loop body's updates become the arguments of the recursive call, and the value returned after the loop becomes the base-case result. Here `k` and `total` are the two loop variables, so `sum_from` takes them as parameters alongside the fixed `n`.

### 10. Evaluation by substitution

> With no loops or mutation, call expressions can be evaluated by substitution: to apply a function to arguments, evaluate the body of the function with each formal parameter replaced by the corresponding argument.

This is a genuine simplification over environment diagrams, and it is *only* available because nothing is mutated: a parameter's value can never change after the call starts, so writing the value in place of the name loses no information. For the growing version of `sum_first`,

```gleam
fn sum_first(n) {
  case n {
    0 -> 0
    _ -> n + sum_first(n-1)
  }
}
```

the slide shows the body being re-written for `n` = 5, 4, 3, 2, 1, 0 in turn (`case 5 { 0 -> 0  _ -> 5 + sum_first(5-1) }`, then the same with 4, and so on), producing this sequence of expressions:

```
sum_first(5)
5 + sum_first(4)
5 + 4 + sum_first(3)
5 + 4 + 3 + sum_first(2)
5 + 4 + 3 + 2 + sum_first(1)
5 + 4 + 3 + 2 + 1 + sum_first(0)
5 + 4 + 3 + 2 + 1 + 0
```

### 11. Comparing two processes: growing vs fixed size

> Some recursive processes grow, while others have a fixed size.

Put the two substitution traces next to each other:

```
sum_first(5)                        sum_first(5)
5 + sum_first(4)                    sum_from(5, 1, 0)
5 + 4 + sum_first(3)                sum_from(5, 2, 1)
5 + 4 + 3 + sum_first(2)            sum_from(5, 3, 3)
5 + 4 + 3 + 2 + sum_first(1)        sum_from(5, 4, 6)
5 + 4 + 3 + 2 + 1 + sum_first(0)    sum_from(5, 5, 10)
5 + 4 + 3 + 2 + 1 + 0               sum_from(5, 6, 15)
15                                  15
```

The left column's expressions get **longer** at every step: there is pending work (`5 + 4 + 3 + ...`) that must be remembered until the base case is reached, which is exactly what stack frames are for. The right column's expressions are all the **same size**: each one is a complete description of the remaining computation, with the partial result carried along in `total`. That accumulator is what buys the fixed size.

### 12. Tail calls and tail contexts

> A language that executes an iterative process as efficiently as a `while` statement, even if the iterative process is described by a recursive procedure, is called **tail-recursive**.

This works in Gleam (and in tail-recursive languages generally) if the recursive call appears in a **tail context**:

- The body of a function (after any `let`)
- A result of a `case` in a tail context

In `sum_from`, the call `sum_from(n, k + 1, total + k)` is the entire result of the `False ->` branch, and that `case` is the whole function body, so it is a tail call. When the recursive call is in a tail context, **calling the function is fast and uses space just like a `while` loop**, and the slide says it "should use resources like"

```python
def sum_from(n, k, total):
    while k <= n:
        k, total = k + 1, total + k
    return total
```

A recursive call that is **part of a larger expression is not in a tail context**. In

```gleam
fn sum_first(n) {
  case n {
    0 -> 0
    _ -> n + sum_first(n-1)      // NOT a tail context
  }
}
```

the call `sum_first(n-1)` is an operand of `+`, so after it returns there is still an addition to perform; the frame cannot be reused. The definition composes recursively: "a result of a `case` in a tail context" means nested `case` expressions preserve tailness, but wrapping a call in any operator or in another call destroys it.

### 13. How this is implemented: compiling Gleam

> Gleam is compiled (checked and translated) into either JavaScript or Erlang.

JavaScript is **not** tail-recursive, so the Gleam compiler **generates `while` loops for recursive tail calls**:

```gleam
fn sum_from(n, k, total) {
  case k > n {
    True -> total
    False -> sum_from(n, k + 1, total + k)
  }
}
```

compiles to

```javascript
function sum_from(loop$n, loop$k, loop$total) {
  while (true) {
    let n = loop$n;
    let k = loop$k;
    let total = loop$total;
    let $ = k > n;
    if ($) {
      return total;
    } else {
      loop$n = n;
      loop$k = k + 1;
      loop$total = total + k;
    }
  }
}
```

This is the clearest possible evidence for the claim in the previous section. A tail call is "not really a call": it is a jump back to the top of the function with new parameter values, which the compiler implements literally by assigning to `loop$n`, `loop$k`, `loop$total` and continuing the loop. Notice the compiler's output is *imperative* code with mutation and a loop; immutability is a property of the source language, not a cost paid at runtime.

### 14. Generic functions

> A lowercase type name is a placeholder for the actual type used to call a function.

```gleam
pub fn main() {
  let add_one = fn(x) { x + 1 }
  let exclaim = fn(x) { x <> "!" }

  echo twice(add_one, 10)
  echo twice(exclaim, "Hello")
}

// The name t refers to the same type multiple times
fn twice(f: fn(t) -> t, x: t) -> t {
  f(f(x))
}
```

`twice` works on both an `Int` function and a `String` function because `t` is a **type variable**, not a concrete type. Because the *same* name `t` is used in all three positions, the annotation also expresses a constraint: whatever `x`'s type is, `f` must map that type to itself, and the result has that type too. `twice(add_one, 10)` is `12`; `twice(exclaim, "Hello")` is `"Hello!!"` (`<>` is string concatenation). The slide notes that the annotations can simply be omitted, since

```gleam
fn twice(f, x) {
  f(f(x))
}
```

also works: Gleam infers the general type. Annotations are documentation plus a deliberate restriction, not a requirement.

---

## Definitions

- **Imperative programming:** a perspective in which programs are instructions for updating memory.
- **Object-oriented programming:** a perspective in which programs define objects that behave like the entities they represent.
- **Functional programming:** a perspective in which programs are data definitions and pure functions that generate data.
- **Expression:** something that is evaluated in an environment (which gives names meaning) to produce a value. In Gleam, function bodies are expressions, and the forms include literal, call, `let`, `echo`, and `case` expressions.
- **Module:** a Gleam file. Names marked `pub` are accessible to users of the module; a program needs a `pub fn main()`.
- **`echo`:** an expression that displays any value, similar to Python's `print`.
- **`case` expression:** Gleam's only conditional form. It matches a subject value against patterns in order and evaluates to the result of the first matching branch.
- **Pattern:** the left side of a `case` branch. `_` is a wildcard matching anything; `2 | 3 | 4` is an alternation matching any of several patterns.
- **Guard:** a condition introduced by `if` on a pattern in a `case` expression (`_ if n < 0 -> ...`). The branch applies only if the pattern matches and the guard is true.
- **`Nil`:** "a value that is returned by functions that have nothing else to return, as all functions must return something."
- **Anonymous function:** a function value created with `fn`, as in `fn(x, y) { x + y }`, with no name attached.
- **Function type:** a type written with `fn`, as in `fn(Int, Int) -> Int`, describing a function's parameter types and return type.
- **Higher-order function:** a function that takes a function as an argument or returns a function as its value (for example `combine` and `make_adder`).
- **`let` expression:** binds a value to a local name. Each `let` introduces a new scope, equivalent to a new local frame, and is equivalent to immediately calling an anonymous function with that name as its parameter.
- **Block (`{ ... }`):** an expression that may contain multiple lines and evaluates to the value of its last expression. `let` bindings inside a block do not affect expressions outside it. Also used instead of parentheses to change the order of evaluation in arithmetic.
- **Immutable value:** a value that cannot be changed after creation. All Gleam values are immutable.
- **Evaluation by substitution:** to apply a function to arguments, evaluate the body of the function with each formal parameter replaced by the corresponding argument.
- **Tail context:** the body of a function (after any `let`), or a result of a `case` that is itself in a tail context.
- **Tail call:** a call appearing in a tail context. A call that is part of a larger expression is not a tail call.
- **Tail-recursive language:** a language that executes an iterative process as efficiently as a `while` statement, even when that process is described by a recursive procedure.
- **Generic function:** a function whose type annotations use lowercase type names (such as `t`) as placeholders for the actual types used at the call site; the same name used in several positions must refer to the same type.

---

## Worked Examples

The lecture's examples are in Gleam, so each one below is given in Gleam exactly as the slides have it, with the Python equivalent next to it where the slides provide one (and a faithful Python translation where they do not, marked as such). Environment/box-and-pointer reasoning is described in words since the slide diagrams did not survive extraction.

### Example 1: `sum_squares`, and `let` as nested lambdas

```gleam
fn sum_squares(x, y) {
  let x2 = x * x
  let y2 = y * y
  x2 + y2
}
```

Evaluate `sum_squares(3, 4)`:

1. A frame is created binding `x` to `3` and `y` to `4`, with the module as its parent.
2. `let x2 = x * x` is evaluated. By the lecture's rule, this opens a **new** frame whose parent is the frame from step 1, binding `x2` to `9`. Everything after this `let` is evaluated in that new frame.
3. `let y2 = y * y` opens another frame, child of step 2's, binding `y2` to `16`. Looking up `y` succeeds because the lookup walks up the chain to step 1's frame.
4. The final expression `x2 + y2` is evaluated in the innermost frame: `9 + 16` = `25`. There is no `return`, so the value of the body is the value of the call.

The slide's equivalent form makes this chain of frames syntactically visible:

```gleam
fn sum_squares(x, y) {
  fn (x2) {
    fn (y2) {
      x2 + y2
    }(y * y)
  }(x * x)
}
```

Here the frames in steps 2 and 3 are ordinary call frames of two anonymous functions. Reading it as a diagram: the outer `fn (x2) {...}` is a function value whose parent environment is the `sum_squares` frame; calling it on `9` makes a frame binding `x2`; inside, `fn (y2) { x2 + y2 }` is a function value whose parent is *that* frame; calling it on `16` makes a frame binding `y2`; the body then resolves `y2` locally and `x2` one frame up. In Python this is

```python
def sum_squares(x, y):          # Python translation, not on the slides
    return (lambda x2: (lambda y2: x2 + y2)(y * y))(x * x)
```

and the fact that this is awkward in Python but *is the definition* of `let` in Gleam is the whole point.

### Example 2: `fib` by `case`

```python
def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n - 2) + fib(n - 1)
```

```gleam
fn fib(n) {
  case n {
    0 -> 0
    1 -> 1
    _ -> fib(n - 2) + fib(n - 1)
  }
}
```

Evaluate `fib(3)` by substitution:

1. `case 3 { 0 -> 0  1 -> 1  _ -> fib(3-2) + fib(3-1) }`. The patterns `0` and `1` do not match `3`; `_` does, so the value is `fib(1) + fib(2)`.
2. `fib(1)` matches pattern `1`, giving `1`.
3. `fib(2)` matches `_`, giving `fib(0) + fib(1)` = `0 + 1` = `1`.
4. Total: `1 + 1` = `2`.

Note that in step 1 the recursive calls are operands of `+`, so **neither is a tail call**, and the substitution trace grows exactly like `sum_first` did. This version does the same exponential amount of work as the Python one.

### Example 3: `describe` and guard ordering

```gleam
pub fn describe(n: Int) -> String {
  case n {
    _ if n < 0 -> "Negative"
    0 -> "None"
    1 -> "One"
    2 | 3 | 4 -> "Few"
    _ -> "Many"
  }
}
```

- `describe(-5)`: branch 1's pattern `_` matches and its guard `-5 < 0` is true, so `"Negative"`.
- `describe(0)`: branch 1's pattern matches but the guard `0 < 0` is false, so the branch is skipped; branch 2's pattern `0` matches, giving `"None"`.
- `describe(3)`: branches 1 through 3 fail, branch 4's alternation `2 | 3 | 4` matches, giving `"Few"`.
- `describe(100)`: only the final `_` matches, giving `"Many"`.

Order matters: if `_ if n < 0` were moved to the end it would be unreachable, because the earlier `_ -> "Many"` would already match everything.

### Example 4: `sign`, and `case Nil`

```gleam
pub fn sign(n: Int) -> Int {
  case n {
    _ if n > 0 -> 1
    _ if n < 0 -> -1
    _ -> 0
  }
}
```

Every pattern here is `_`, so the `case` subject is never actually inspected: all the decision-making lives in the guards, which look `n` up in the environment. Since the subject is irrelevant, it can be replaced with any value at all, and the slide replaces it with `Nil`:

```gleam
pub fn sign(n: Int) -> Int {
  case Nil {
    _ if n > 0 -> 1
    _ if n < 0 -> -1
    _ -> 0
  }
}
```

`sign(7)` = `1`, `sign(-7)` = `-1`, `sign(0)` = `0` in both versions. The takeaway: `case <anything> { _ if c1 -> ...  _ if c2 -> ...  _ -> ... }` is how you write a plain `if`/`elif`/`else` chain in a language that only has pattern matching.

### Example 5: `make_adder` and `combine`

```gleam
fn make_adder(n: Int) -> fn(Int) -> Int {
  fn(k) { n + k }
}
```

Evaluate `make_adder(3)(4)`:

1. `make_adder(3)` creates a frame binding `n` to `3`.
2. The body is the expression `fn(k) { n + k }`, which evaluates to a **function value** whose parent environment is that frame. Picture an arrow from the function value back to the frame where `n` is `3`.
3. That function value is the result of `make_adder(3)`. The frame for `n` cannot be discarded, because the returned function still points at it.
4. Calling it on `4` creates a frame binding `k` to `4` whose parent is the frame from step 1. The body `n + k` finds `k` = `4` locally and `n` = `3` in the parent: `7`.

Note the return type `fn(Int) -> Int`, read as "returns a one-argument function of integers." That annotation is the Gleam way of saying in the type system what Python can only say in a docstring.

```gleam
fn combine(x: Int, y: Int, z: Int, f: fn(Int, Int) -> Int) -> Int {
  f(f(x, y), z)
}
```

Evaluate `combine(2, 3, 4, fn (x, y) { x * y })`:

1. A frame binds `x` = `2`, `y` = `3`, `z` = `4`, and `f` to the anonymous multiplying function.
2. `f(x, y)` calls it on `2` and `3`, giving `6`. (Inside the anonymous function, *its own* `x` and `y` shadow `combine`'s; they are different frames.)
3. `f(6, 4)` gives `24`, the value of the body and hence of the call.

The Python counterpart is `def combine(x, y, z, f): return f(f(x, y), z)` called as `combine(2, 3, 4, lambda x, y: x * y)`, the difference being that Gleam checks at compile time that the function you pass takes two `Int`s.

### Example 6: `sum_first` two ways, traced

Non-tail-recursive version (the growing process):

```gleam
fn sum_first(n) {
  case n {
    0 -> 0
    _ -> n + sum_first(n-1)
  }
}
```

Substituting repeatedly for `sum_first(5)`:

```
sum_first(5)
5 + sum_first(4)
5 + 4 + sum_first(3)
5 + 4 + 3 + sum_first(2)
5 + 4 + 3 + 2 + sum_first(1)
5 + 4 + 3 + 2 + 1 + sum_first(0)
5 + 4 + 3 + 2 + 1 + 0
15
```

Each line is longer than the last. The additions cannot be performed until the base case is reached, so the amount of information that must be held grows linearly with `n`: that is linear space, or in environment terms, five pending frames at the deepest point.

Tail-recursive version (the fixed-size process):

```gleam
fn sum_first(n) {
  sum_from(n, 1, 0)
}

fn sum_from(n, k, total) {
  case k > n {
    True -> total
    False -> sum_from(n, k + 1, total + k)
  }
}
```

Substituting for `sum_first(5)`:

```
sum_first(5)
sum_from(5, 1, 0)        case 1 > 5 -> False, so sum_from(5, 1+1, 0+1)
sum_from(5, 2, 1)        case 2 > 5 -> False, so sum_from(5, 2+1, 1+2)
sum_from(5, 3, 3)        case 3 > 5 -> False, so sum_from(5, 3+1, 3+3)
sum_from(5, 4, 6)        case 4 > 5 -> False, so sum_from(5, 4+1, 6+4)
sum_from(5, 5, 10)       case 5 > 5 -> False, so sum_from(5, 5+1, 10+5)
sum_from(5, 6, 15)       case 6 > 5 -> True, so total
15
```

Every line has the same size, and `total` accumulates `1, 3, 6, 10, 15`. The recursive call is the entire result of the `False` branch of a `case` that is the entire body, so it is in a tail context and Gleam reuses the frame. (The Iteration slide writes the test as `case k <= n { True -> recurse  False -> total }` and the Tail Calls slide as `case k > n { True -> total  False -> recurse }`; these are the same function with the branches swapped.) The Python equivalent on the slide is the obvious loop:

```python
def sum_from(n, k, total):
    while k <= n:
        k, total = k + 1, total + k
    return total
```

and the correspondence is exact: `k, total = k + 1, total + k` in the loop is `sum_from(n, k + 1, total + k)` in the tail call.

### Example 7: tail-recursive `fib`

The lecture poses this as an exercise: "Try to implement `fib` as a tail-recursive function," starting from the tree-recursive version and the Python loop.

```python
def fib(n):
    prev, curr = 1, 0
    while n > 0:
        prev, curr, n = curr, prev + curr, n - 1
    return curr
```

```gleam
fn fib(n) {
  fib_iter(1, 0, n)
}

fn fib_iter(prev, curr, steps) {
  case steps {
    0 -> curr
    _ -> fib_iter(curr, prev + curr, steps - 1)
  }
}
```

The three loop variables `prev`, `curr`, `n` become the three parameters `prev`, `curr`, `steps`; the initial values `1, 0` become the initial arguments; the loop condition `n > 0` becomes the pattern match on `steps` against `0`; the simultaneous update becomes the argument expressions of the tail call; and the `return curr` becomes the base-case result.

Trace `fib(5)`:

| call | `prev` | `curr` | `steps` |
|---|---|---|---|
| `fib_iter(1, 0, 5)` | 1 | 0 | 5 |
| `fib_iter(0, 1, 4)` | 0 | 1 | 4 |
| `fib_iter(1, 1, 3)` | 1 | 1 | 3 |
| `fib_iter(1, 2, 2)` | 1 | 2 | 2 |
| `fib_iter(2, 3, 1)` | 2 | 3 | 1 |
| `fib_iter(3, 5, 0)` | 3 | 5 | 0 |

`steps` is `0`, so the result is `curr` = `5`, which is `fib(5)` with `fib(0)` = `0`. The recursive call is the whole result of the `_` branch, so this runs in constant space and linear time, versus exponential time for the tree-recursive version. Also note the tree-recursive original used the alternation pattern `0 | 1 -> n` to handle both base cases in one branch.

The slide warns: **making an infinite loop will crash the playground**, which is a real hazard here, since forgetting `steps - 1` produces a tail call that loops forever without ever overflowing a stack.

### Example 8: generic `twice`, and the `k_times` exercise

```gleam
pub fn main() {
  let add_one = fn(x) { x + 1 }
  let exclaim = fn(x) { x <> "!" }

  echo twice(add_one, 10)
  echo twice(exclaim, "Hello")
}

fn twice(f: fn(t) -> t, x: t) -> t {
  f(f(x))
}
```

- `twice(add_one, 10)`: `t` is `Int` for this call. The body computes `add_one(add_one(10))` = `add_one(11)` = `12`.
- `twice(exclaim, "Hello")`: `t` is `String` for this call. The body computes `exclaim(exclaim("Hello"))` = `exclaim("Hello!")` = `"Hello!!"`.

The same source function serves both because `t` is a placeholder resolved per call site. The requirement is only that `f`'s argument and return types match each other and match `x`, which is what makes `f(f(x))` well typed.

**Exercise (from the slides):** write `fn k_times(f: fn(t) -> t, x: t, k: Int) -> t`, which calls `f` repeatedly `k` times on `x`, so that `k_times(add_one, 10, 3)` evaluates `add_one(add_one(add_one(10)))`.

```gleam
fn k_times(f: fn(t) -> t, x: t, k: Int) -> t {
  case k {
    0 -> x
    _ -> k_times(f, f(x), k - 1)
  }
}
```

Trace `k_times(add_one, 10, 3)`:

```
k_times(add_one, 10, 3)   k is not 0, so recurse with f(x) = 11 and k - 1 = 2
k_times(add_one, 11, 2)   recurse with 12 and 1
k_times(add_one, 12, 1)   recurse with 13 and 0
k_times(add_one, 13, 0)   k matches 0, so the result is x = 13
```

`13`, which is `add_one` applied three times to `10`. This version is **tail-recursive**: the recursive call is the entire result of the `_` branch, and `f` is applied *on the way down* so that `x` acts as an accumulator. Contrast the non-tail alternative:

```gleam
fn k_times(f: fn(t) -> t, x: t, k: Int) -> t {
  case k {
    0 -> x
    _ -> f(k_times(f, x, k - 1))     // correct, but NOT a tail call
  }
}
```

Here the recursive call is an operand of a call to `f`, so there is pending work at every level and the trace grows: `f(f(f(10)))` must be built up before anything is computed. Both give `13`; only the first runs in constant space. The signature also type-checks in both, and `k_times(exclaim, "Hi", 2)` would give `"Hi!!"`.

---

## Common Pitfalls

1. **Writing `return`.** There is no `return` in Gleam. The value of a call is the value of its body expression, and an attempt to `return` early is simply not expressible: restructure with a `case` instead.
2. **Writing `x = 4` instead of `let x = 4`.** Binding a name always requires `let`.
3. **Thinking a second `let` on the same name mutates it.** It does not. Each `let` opens a new scope that shadows the old binding; functions created earlier still see the earlier binding. This is why the Python closure example prints `4` while the Gleam one prints `3`.
4. **Reaching for a loop.** There are no `for`/`while` statements. Any repetition must be recursion, which usually means adding parameters for what would have been the loop variables, plus a helper function (`sum_from`, `fib_iter`) so the top-level function keeps its simple signature.
5. **Assuming "recursive" means "slow and stack-hungry."** In a tail-recursive language, a recursive call in a tail context costs no more than a loop iteration. Conversely, do not assume *all* recursion is cheap: `n + sum_first(n-1)` grows.
6. **Misidentifying tail contexts.** `n + f(n-1)`, `g(f(n-1))`, and `f(n-1) * 2` are all non-tail, because the recursive call is part of a larger expression. Being "the last thing written" is not the test; being the *whole* result of the body or of a `case` branch in tail position is.
7. **Expecting an accumulator for free.** Converting a growing process into a fixed-size one usually requires adding an accumulator parameter (`total`, or `prev`/`curr`) and moving work to happen *before* the recursive call rather than after it.
8. **Confusing `if` with a conditional expression.** `if` in Gleam exists only as a guard attached to a pattern inside a `case`. You cannot write `if cond { a } else { b }` as a standalone expression.
9. **Getting `case` branch order wrong.** The first matching branch wins, so a `_` branch placed early makes later branches unreachable, and a guard that should take priority (like `_ if n < 0`) must come before more general patterns.
10. **Forgetting that a block's value is its last expression.** A block ending in a `let` has no useful value, and names bound inside a block are not visible outside it (`x2` in `quadratic`).
11. **Using parentheses where blocks are needed for grouping.** The lecture's idiom is `{ fahrenheit - 32 } * 5 / 9`.
12. **Infinite loops in the playground.** A buggy tail-recursive function will spin forever rather than raising a recursion error, and the slides warn that this crashes the playground. Always check that the recursive call makes progress toward the base case.
13. **Pinning a generic type down unnecessarily.** Writing `fn twice(f: fn(Int) -> Int, x: Int) -> Int` makes `twice(exclaim, "Hello")` a type error. Use a lowercase type name (or omit annotations) when the function does not care about the type.

---

## Likely Exam Points

**1. Translate a Python function with multiple `return`s into Gleam.**

*Q:* Translate to Gleam, using one `case` expression:
```python
def parity(n):
    if n % 2 == 0:
        return "even"
    else:
        return "odd"
```
*A:*
```gleam
fn parity(n: Int) -> String {
  case n % 2 {
    0 -> "even"
    _ -> "odd"
  }
}
```
(Equivalently, with a guard: `case Nil { _ if n % 2 == 0 -> "even"  _ -> "odd" }`. No `return`, and the body is a single expression.)

**2. Identify whether a call is in a tail context.**

*Q:* In each of these, is the recursive call in a tail context?
```gleam
fn a(n) { case n { 0 -> 1  _ -> a(n-1) } }
fn b(n) { case n { 0 -> 1  _ -> 2 * b(n-1) } }
fn c(n, t) { let m = n - 1  case n { 0 -> t  _ -> c(m, t + n) } }
fn d(n) { case n { 0 -> 0  _ -> case n > 10 { True -> d(n-1)  False -> n } } }
```
*A:* `a`: yes, it is the result of a `case` that is the body. `b`: no, the call is an operand of `*`, so work remains after it returns. `c`: yes; the body after a `let` is still a tail context. `d`: yes; a `case` in a tail context makes its branch results tail contexts too, and this nested `case` is the result of a branch of the outer one.

**3. Give the substitution trace, and say whether the process grows.**

*Q:* Write the sequence of expressions produced by evaluating `sum_first(3)` for `fn sum_first(n) { case n { 0 -> 0  _ -> n + sum_first(n-1) } }`, and state the space used.
*A:*
```
sum_first(3)
3 + sum_first(2)
3 + 2 + sum_first(1)
3 + 2 + 1 + sum_first(0)
3 + 2 + 1 + 0
6
```
The expressions get longer at each step, so the process grows: linear space in `n`. The recursive call is an operand of `+`, hence not a tail call.

**4. Convert a growing recursion into a tail recursion.**

*Q:* Write a tail-recursive Gleam `factorial(n)`.
*A:*
```gleam
fn factorial(n: Int) -> Int {
  fact_iter(n, 1)
}

fn fact_iter(k: Int, total: Int) -> Int {
  case k {
    0 -> total
    _ -> fact_iter(k - 1, total * k)
  }
}
```
The trick is the accumulator `total`: the multiplication happens *before* the recursive call, so the call is the entire result of the branch and nothing is pending afterwards. `factorial(4)`: `fact_iter(4,1)`, `fact_iter(3,4)`, `fact_iter(2,12)`, `fact_iter(1,24)`, `fact_iter(0,24)`, `24`.

**5. Predict the output of the `let`-shadowing closure example.**

*Q:* What does each program display, and why?
```python              
x = 3                  
f = lambda: x          
x = 4                  
f()                    
```
```gleam
pub fn main() {
  let x = 3
  let f = fn () {x}
  let x = 4
  echo f()
}
```
*A:* Python displays `4`: `x = 4` rebinds the name in the same (global) frame, and the lambda looks up `x` only when called. Gleam displays `3`: each `let` introduces a new scope, so `let x = 4` creates a new binding that shadows the old one rather than changing it, and `f`'s parent environment still binds `x` to `3`. (The slide shows the `4` for Python and the general rule; the Gleam value follows from the rule.)

**6. Explain what `let` desugars to.**

*Q:* Rewrite `fn g(a) { let b = a + 1  let c = b * 2  b + c }` using only anonymous functions, no `let`.
*A:*
```gleam
fn g(a) {
  fn (b) {
    fn (c) {
      b + c
    }(b * 2)
  }(a + 1)
}
```
Each `let` becomes an immediately-called one-parameter anonymous function, which is why "each `let` introduces a new scope (equivalent to a new local frame)." `g(3)`: `b` = `4`, `c` = `8`, result `12`.

**7. Write or read a function type.**

*Q:* What is the type of `make_adder` below, and what does `make_adder(10)(5)` evaluate to?
```gleam
fn make_adder(n: Int) { fn(k) { n + k } }
```
*A:* `fn(Int) -> fn(Int) -> Int`: it takes an `Int` and returns a one-argument function of integers. `make_adder(10)` returns a closure that remembers `n` = `10`; calling it on `5` gives `15`.

**8. Use a guard, including the `case Nil` idiom.**

*Q:* Write `fn clamp(n: Int) -> Int` that returns `0` for `n` below `0`, `100` for `n` above `100`, and `n` otherwise, using only guards.
*A:*
```gleam
fn clamp(n: Int) -> Int {
  case Nil {
    _ if n < 0 -> 0
    _ if n > 100 -> 100
    _ -> n
  }
}
```
Because no pattern refers to the subject, the subject can be `Nil`; all the logic is in the guards, and branches are tried top to bottom.

**9. Explain how tail calls are implemented.**

*Q:* Gleam compiles to JavaScript, which is not tail-recursive. How can Gleam still promise that tail recursion uses space like a loop?
*A:* The compiler translates a function containing recursive tail calls into a `while (true)` loop. Parameters become `loop$`-prefixed variables; a tail call is compiled not as a call but as an assignment of the new argument values to those variables followed by another iteration, and a non-tail branch becomes a `return`. So the recursion disappears entirely at the target-language level.

**10. Write a generic higher-order function.**

*Q:* Give the type and definition of `k_times`, which applies `f` to `x` exactly `k` times, and say whether your version is tail-recursive.
*A:*
```gleam
fn k_times(f: fn(t) -> t, x: t, k: Int) -> t {
  case k {
    0 -> x
    _ -> k_times(f, f(x), k - 1)
  }
}
```
`t` is a type variable, so the same function works for `k_times(add_one, 10, 3)` = `13` and `k_times(exclaim, "Hi", 2)` = `"Hi!!"`. It is tail-recursive because the recursive call is the whole result of the `_` branch; writing `f(k_times(f, x, k - 1))` instead would be correct but non-tail.

**11. Reason about blocks and scope.**

*Q:* What does `quadratic(2, 3, 4, 5)` return for the slide's definition, and why is `x2` not usable on the last line?
*A:* `ax2` is the value of the block `{ let x2 = x * x  a * x2 }` = `3 * 4` = `12`; then `ax2 + b * x + c` = `12 + 8 + 5` = `25`. `x2` is bound by a `let` *inside* the block, and a `let` within a block does not affect expressions outside the block, so only the block's value (bound to `ax2`) escapes.

---

## Summary

- **Three perspectives:** imperative (instructions that update memory), object-oriented (objects representing entities), functional (data definitions and pure functions). They can be combined.
- **Gleam** is a small functional language: function bodies are expressions, functions are first-class, anonymous functions are everywhere, all values are immutable, and there are no `for`/`while` loops.
- Programs live in **modules**; a `pub fn main()` is the entry point; `echo` displays a value like Python's `print`.
- **Expression forms:** literals, calls, `let`, `echo`, and `case`.
- **No `return`:** a call's value is its body expression's value. Multiple Python `return`s become one `case` expression.
- **`case`** matches a subject against patterns in order, first match wins. `_` is a wildcard, `2 | 3 | 4` is alternation, and `if` introduces a **guard** on a pattern. `Nil` is the value returned when there is nothing else to return, and `case Nil { _ if ... }` turns a `case` into a pure guard chain.
- **Higher-order functions** use `fn` twice: `fn(x, y) { x + y }` makes a function value, `fn(Int, Int) -> Int` is its type. `make_adder` returns a closure; `combine` takes a function parameter.
- **`let` introduces a new scope** (a new frame) and is equivalent to immediately calling an anonymous function. Rebinding a name shadows rather than mutates, so closures keep seeing their original bindings.
- **Blocks** `{ ... }` hold several lines and evaluate to the last expression; their `let`s do not leak out; they also replace parentheses for grouping arithmetic.
- **Iteration is recursion:** loop variables become parameters, the loop test becomes the `case`, the updates become the recursive call's arguments.
- **Substitution** is a valid evaluation model because nothing mutates: replace each formal parameter with its argument and evaluate the body.
- **Growing vs fixed-size processes:** `n + sum_first(n-1)` produces ever-longer expressions (linear space); `sum_from(n, k+1, total+k)` produces same-size expressions thanks to an accumulator (constant space).
- **Tail contexts** are the body of a function (after any `let`) and the results of a `case` in a tail context. A call inside a larger expression is not a tail call. Calls in tail position cost what a `while` loop costs, which is what makes a language **tail-recursive**.
- Gleam **compiles** to JavaScript or Erlang, and generates `while (true)` loops with `loop$` variables for tail calls because JavaScript is not tail-recursive.
- **Tail-recursive `fib`** uses `fib_iter(prev, curr, steps)` with `fib(n) = fib_iter(1, 0, n)`, mirroring the Python `while` loop exactly and turning exponential tree recursion into linear iteration.
- **Generic functions** use lowercase type names as placeholders: `fn twice(f: fn(t) -> t, x: t) -> t` works for both `Int` and `String`; annotations may be omitted entirely.
