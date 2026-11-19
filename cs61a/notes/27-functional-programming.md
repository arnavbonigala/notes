<!-- Wed, Oct 28, 2026 | sources: slides + YouTube auto-transcript -->
# Lecture 27: Functional Programming

This lecture introduces a third perspective on what a program is, alongside the imperative view (programs are instructions for updating memory) and the object-oriented view (programs define objects that behave like the entities they represent). In the functional view, a program is a set of data definitions and pure functions that generate data: a pure function takes input, computes a result, and does nothing else. To explore this perspective, the lecture introduces Gleam, a small functional language where function bodies are expressions, functions are first-class values, anonymous functions are convenient and used liberally, all values are immutable, and there are no `for` or `while` loops. Along the way we cover Gleam's expression forms (literals, calls, `let`, `echo`, `case`), guards with `if` inside `case`, higher-order functions and function types, `let` as scope-introducing expression, blocks, iteration via recursion, evaluation by substitution, tail calls and tail recursion (plus how Gleam's JavaScript compiler turns tail calls into `while` loops), and generic functions with lowercase type variables.

## Key Concepts

### Three perspectives on programming

**Imperative:** programs are instructions for updating memory. This is closest to how hardware actually works: the processor loads values from memory, computes, and writes back to memory. Any time you build a list and then mutate its contents to get the answer you want, you are programming imperatively.

**Object-oriented:** programs define objects that behave like the entities they represent. The computation happens by objects calling each other's methods, each managing its own local state. This doesn't change *what* can be computed (you could write the same program as one long list of memory updates), but it is a powerful way to manage complexity in large programs. Note that the fundamental mechanism is still memory update, just distributed across many small stores (instance attributes) bundled with behavior.

**Functional:** programs are data definitions and pure functions that generate data. No memory is updated at all. A pure function does nothing except the computation needed to produce a result from its inputs.

These views are not mutually exclusive. Python supports all three, and real programmers mix them, picking whichever fits the problem. The reason to lean functional: programs defined only in terms of pure functions are easier to reason about, and separate parts of such a program can run at the same time without one part mutating data another part is reading.

### Why immutability is central to "pure"

Immutability isn't an arbitrary restriction bolted onto functional languages. If a function mutates a value, it changes how the rest of the program behaves in a way that is not captured by its return value. That is exactly what disqualifies it from being pure. So "all values are immutable" and "functions are pure" are two sides of the same idea. The absence of `for`/`while` follows too: a loop is fundamentally a list of instructions to repeat, with a counter variable being updated, which is imperative by nature.

### What makes Gleam a functional language

- Program behavior is just functions, and function bodies are expressions. No classes, no inheritance, no methods inside classes.
- Functions are first-class values: they can be passed as arguments and returned from other functions.
- Anonymous functions are convenient to create and used liberally (these are Gleam's analogue of Python's `lambda`, and like lambdas they get a name by being bound or passed as an argument).
- All values are immutable, and there are no `for`/`while` loops that update variables.

### Why Gleam specifically

- It's small and simple: essentially only functions, names, and one form of conditional (`case`). The whole language fits in a few lectures.
- It's a nice language for learning algebraic data types and pattern matching, ideas that are spreading quickly through industry languages.
- It's good preparation for Rust, Swift, and TypeScript, which use these same ideas but are much larger languages, so learning all their details would take far longer.

### Modules and `main`

Gleam programs are organized into modules (files), each holding a set of functions. One module can import functions from another, so a program can span multiple files and still be nothing but functions. For now we write one module. There must be a `main` function accessible to users of the module, which means marked `pub` (public, i.e. callable from other modules). Functions not marked `pub` are private to the module, which is appropriate for helper functions that are just part of an implementation.

`echo` displays any value, similar to Python's `print`.

### Everything is an expression

An expression is evaluated in an environment (which gives names meaning) to produce a value. Gleam function bodies are all expressions. The forms:

- **Literal:** `2`, `3.3`, `True`, `"Hello"` (integers, floats, booleans, strings)
- **Call:** `f(2, 3)`
- **Let:** `let x = 4`
- **Echo:** `echo x`
- **Case:** the conditional form

There is no `return` in Gleam. The value of a call expression is the value of the function's body expression.

### `case` expressions

`case` is how you choose between expressions, but it works by pattern matching rather than by a single boolean test. Each line is a pattern, an arrow, and a result expression. The value of the whole `case` is the value of the result expression belonging to the first pattern that matches.

```gleam
case x > 3 {
  True -> "big"
  False -> "not big"
}
```

Here you match on the boolean value of `x > 3`, listing what happens for `True` and for `False`. Alternatively you match the value directly:

```gleam
case x {
  3 -> "three"
  4 -> "four"
  _ -> "not three or four"
}
```

`_` is the wildcard pattern matching anything else. Patterns are tried top to bottom, so the wildcard goes last.

**Exhaustiveness is checked at compile time.** If you write patterns for `1`, `2`, `3` on an `Int` and leave out everything else, the program will not even start: Gleam reports that the `case` is not exhaustive and even tells you which pattern you need to add. This is a static check, not a runtime error.

### Types and type inference

Gleam has static types, and names have types associated with them. If a function is declared to take an `Int`, passing anything else is a type mismatch error reported before the program runs. Python, by contrast, would happily accept `"hello"` and only complain when you tried to multiply it by itself.

Gleam's type inference is strong, so you can omit annotations and it will figure types out from how values are used. But you should annotate function parameters and return types anyway: it's good documentation and keeps you from confusing yourself. The lecture emphasizes this repeatedly, including for generic functions.

String concatenation uses `<>`, not `+`. You cannot concatenate a number with a string; convert first with `int.to_string` (which requires `import gleam/int`).

### Guards: `if` inside `case`

Gleam does have `if`, but never standalone. It introduces a **guard** on a pattern inside a `case`: the pattern matches only if the guard expression is true.

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

`_ if n < 0` is a wildcard pattern restricted by a guard, so it matches any `n` that is negative. The `|` in `2 | 3 | 4` lets several patterns share one result expression.

A consequence worth noticing: because guards do the testing, the value you `case` on need not be referenced at all. Both of these work identically:

```gleam
pub fn sign(n: Int) -> Int {       pub fn sign(n: Int) -> Int {
  case n {                           case Nil {
    _ if n > 0 -> 1                    _ if n > 0 -> 1
    _ if n < 0 -> -1                   _ if n < 0 -> -1
    _ -> 0                             _ -> 0
  }                                  }
}                                  }
```

The right-hand version never looks at `n` in a pattern, only in the guards, so it can match on `Nil` instead. `Nil` is "a value that is returned by functions that have nothing else to return, as all functions must return something."

### Higher-order functions and function types

Anonymous functions are created with `fn`: `fn(x, y) { x + y }`. Function *types* are also written with `fn`: `fn(Int, Int) -> Int`. The similarity is deliberate, and the two are used together to build higher-order functions.

```gleam
fn make_adder(n: Int) -> fn(Int) -> Int {
  fn(k) { n + k }
}
```

`fn(Int) -> Int` is the type of a one-argument function on integers, so this says `make_adder` returns such a function. `make_adder(3)(4)` evaluates to 7: the first call supplies `n = 3` and returns the inner function, the second call supplies `k = 4`.

```gleam
fn combine(x: Int, y: Int, z: Int, f: fn(Int, Int) -> Int) -> Int {
  f(f(x, y), z)
}
```

Here `fn(Int, Int) -> Int` is a *parameter* type: `combine` takes a two-argument integer function. `combine(2, 3, 4, fn (x, y) { x * y })` computes `(2 * 3) * 4 = 24`.

### `let` is an expression that introduces a scope

`let` assigns the value of an expression to a local name. But if function bodies are just expressions, what is `let` doing? The right way to think about it: a `let` is a multi-line expression with two parts. The top line evaluates an expression and binds its value to a name. The rest is a body whose value is the value of the whole `let`.

Each `let` introduces a new scope, equivalent to a new local frame. This is exactly equivalent to introducing a new function and immediately calling it with the value you wanted to bind (see Worked Examples).

**This is not Python assignment.** Python assignment is imperative: it changes the value stored at a name. Gleam's `let` only affects expressions that come *after* it; it cannot reach back and change what an earlier expression means.

```python
>>> x = 3                  pub fn main() {
>>> f = lambda: x            let x = 3
>>> x = 4                    let f = fn () {x}
>>> f()                      let x = 4
4                            echo f()
                           }
```

Python prints `4`. Gleam prints `3`. In Python, `f`'s enclosing frame has one binding for `x` and reassignment changed it, so the later lookup finds `4`. In Gleam, the second `let x = 4` introduces a *new* scope with a new `x`; the function `f` closed over the earlier scope where `x` is `3`, and nothing can change that. Gleam also warns that the new `x` is never used, because the `x` that `f` returns is the earlier one.

### Blocks

A block is code inside curly braces. It can contain multiple lines (multiple expressions) and evaluates to the value of its last expression. A `let` inside a block does not affect expressions outside the block.

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

The block's value (`a * x2`) is what gets bound to `ax2`. `x2` lives only inside the block and does not escape.

Blocks also serve the role parentheses play in arithmetic, controlling order of evaluation:

```gleam
let celsius = { fahrenheit - 32 } * 5 / 9
echo celsius
```

A very common use of blocks is to write multiple lines of work as the result expression of a `case` branch.

### Iteration requires recursion

Gleam has no `for` or `while`, so you iterate by recursion. The structural correspondence to a `while` loop is tight:

```python
def sum_first(n):              fn sum_first(n) {
    k = 1                        sum_from(n, 1, 0)
    total = 0                  }
    while k <= n:
        k, total = k + 1, total + k
    return total               fn sum_from(n, k, total) {
                                 case k <= n {
sum_first(5)  # 15                 True -> sum_from(n, k + 1, total + k)
                                   False -> total
                                 }
                               }
```

Mapping the pieces:
- The initial values of the loop variables (`k = 1`, `total = 0`) become the initial arguments to the recursive helper.
- The `while` condition becomes the expression the `case` matches on.
- The `True` branch corresponds to the simultaneous assignment `k, total = k + 1, total + k`: the new values are passed as arguments to the recursive call.
- The `False` branch corresponds to `return total`.

### Evaluation by substitution

With no loops and no mutation, call expressions can be evaluated by **substitution**: to apply a function to arguments, evaluate the body of the function with each formal parameter replaced by the corresponding argument.

This works precisely *because* there is no reassignment and no mutation. Environments and frames still describe what is happening, but substitution is often easier to carry out by hand.

### Tail calls and tail recursion

Some recursive processes grow as they run; others stay a fixed size. Substitution makes the difference visible: the non-tail-recursive `sum_first` produces longer and longer expressions (`5 + 4 + 3 + ...`), while the tail-recursive `sum_from` produces a sequence of expressions all the same size.

A language that executes an iterative process as efficiently as a `while` statement, even when the process is described by a recursive procedure, is called **tail-recursive**. This works in Gleam (and in tail-recursive languages generally) when the recursive call appears in a **tail context**:

- The body of a function (after any `let`)
- A result of a `case` that is itself in a tail context

A recursive call that is part of a larger expression is **not** in a tail context. The intuition the lecture gives: in a tail context, once you have computed the thing, you are done. In `_ -> n + sum_first(n-1)`, after `sum_first(n-1)` returns you still have to add `n`, so you are not done, and that call is not a tail call.

When a recursive call is in a tail context, calling the function is fast and uses space just like a `while` loop.

### How this actually works: compilation

Gleam is **compiled** (checked and translated) into either JavaScript (for the browser) or Erlang (used for distributed messaging systems like WhatsApp and Discord). This course compiles to JavaScript.

JavaScript is not tail-recursive, so the Gleam compiler **generates `while` loops** for recursive tail calls:

```gleam
fn sum_from(n, k, total) {          function sum_from(loop$n, loop$k, loop$total) {
  case k > n {                        while (true) {
    True -> total                       let n = loop$n;
    False -> sum_from(n, k + 1,         let k = loop$k;
                      total + k)        let total = loop$total;
  }                                     let $ = k > n;
}                                       if ($) {
                                          return total;
                                        } else {
                                          loop$n = n;
                                          loop$k = k + 1;
                                          loop$total = total + k;
                                        }
                                      }
                                    }
```

You do not need to know JavaScript's details. The point is to recognize the `while` loop: the tail-recursive Gleam program really runs as a loop, which is very fast. The playground lets you look at the generated JavaScript and check whether your function produced a loop. A non-tail-recursive function compiles to JavaScript that still looks recursive (with `if`/`return` instead of `case`), with no loop.

When compiling to Erlang this transformation is unnecessary, because Erlang is itself tail-recursive and handles tail calls as efficiently as a loop.

### Generic functions

A **lowercase type name** is a placeholder for the actual type used to call a function. It behaves like a variable that can take on any type, and using the same lowercase name in several places requires those positions to all be the same type.

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

`twice(add_one, 10)` gives `12`: `t` is `Int` here, so `f` must be `fn(Int) -> Int` and `x` must be an `Int`. `twice(exclaim, "Hello")` gives `"Hello!!"`: `t` is `String`. The signature does not say `Int` or `String`, it says "whatever type, as long as it's the same type in all three places."

Writing `fn twice(f, x) { f(f(x)) }` with no annotations also works, since inference figures the details out, but including the types is better documentation and is how it's done in practice.

## Definitions

**Imperative programming:** the perspective that programs are instructions for updating memory.

**Object-oriented programming:** the perspective that programs define objects that behave like the entities they represent, communicating by method calls and each managing its own local state.

**Functional programming:** the perspective that programs are data definitions and pure functions that generate data.

**Pure function:** a function that does nothing except perform the computation needed to produce a result from its inputs. It performs no mutation and has no effect on the rest of the program beyond its return value.

**Immutable value:** a value whose contents can never be changed after creation. All Gleam values are immutable.

**First-class value:** a value that can be passed as an argument to a function and returned from a function. Gleam functions are first-class values.

**Anonymous function:** a function created without being given a name, written in Gleam with `fn`, e.g. `fn(x, y) { x + y }`. Gleam's analogue of a Python `lambda`.

**Module:** a Gleam file containing a set of functions. Modules can import functions from other modules.

**`pub`:** a marker meaning public, i.e. the function can be called from functions in other modules. `main` must be `pub`.

**`echo`:** a Gleam expression that displays any value, similar to Python's `print`.

**Expression:** something evaluated in an environment (which gives names meaning) to produce a value. Gleam expression forms: literal, call, `let`, `echo`, and `case`.

**`case` expression:** Gleam's only conditional form. It matches a value against a sequence of patterns and evaluates to the value of the result expression of the first matching pattern.

**Pattern:** the part of a `case` line before the arrow, compared against the value in the `case` header. `_` is the wildcard pattern matching anything. `|` separates multiple patterns sharing one result expression.

**Exhaustive `case`:** a `case` whose patterns cover all possible values of the matched expression. Gleam refuses to run a program containing a non-exhaustive `case`.

**Guard:** an `if` condition attached to a pattern inside a `case`, restricting the pattern to match only when the condition is true. Gleam's `if` only appears in this position.

**`Nil`:** "a value that is returned by functions that have nothing else to return, as all functions must return something."

**Function type:** a type written with `fn`, e.g. `fn(Int, Int) -> Int`, the type of a two-argument function on integers returning an integer.

**Higher-order function:** a function that takes a function as an argument and/or returns a function.

**`let`:** an expression that binds the value of an expression to a local name and introduces a new scope (equivalent to a new local frame) for the expressions that follow it.

**Block:** code in curly braces that may contain multiple expressions and evaluates to the value of its last expression. Names bound by `let` inside a block do not escape it.

**Evaluation by substitution:** evaluating a call by evaluating the body of the function with each formal parameter replaced by the corresponding argument. Valid in Gleam because there are no loops and no mutation.

**Tail context:** a position where, once the value is computed, the enclosing computation is done. Specifically: the body of a function (after any `let`), and the result of a `case` that is itself in a tail context.

**Tail call:** a call appearing in a tail context.

**Tail-recursive language:** a language that executes an iterative process as efficiently as a `while` statement, even when that process is described by a recursive procedure.

**Compiled:** checked and translated into another language. Gleam compiles to JavaScript or Erlang.

**Generic function:** a function written to accept multiple types of values, using a lowercase type name as a placeholder for the actual type used at the call site.

## Worked Examples

### 1. `sum_squares`: no `return`, body is an expression

```python
def sum_squares(x, y):          fn sum_squares(x, y) {
    return x * x + y * y          x * x + y * y
                                }
```

In the Python version, `return` is a statement that hands a value back. In Gleam there is no `return`: the value of a call to `sum_squares` *is* the value of its body expression. So `sum_squares(3, 4)` evaluates `3 * 3 + 4 * 4` to `25`, and that is the call's value.

With annotations, written as it was in the demo:

```gleam
pub fn sum_squares(x: Int, y: Int) -> Int {
  square(x) + square(y)
}

fn square(z: Int) -> Int {
  z * z
}
```

Two things differ sharply from Python. First, `sum_squares("hello", 4)` is a type mismatch, reported before the program runs; Python would accept it and only fail later when it tried `"hello" * "hello"`. Second, order does not matter: `square` can be defined before or after `sum_squares` and Gleam will find it. Marking `square` `pub` makes it callable from other modules; leaving it unmarked keeps it as an implementation detail of this module.

### 2. `fib`: multiple returns become one `case`

```python
def fib(n):                          fn fib(n) {
    if n == 0:                         case n {
        return 0                         0 -> 0
    elif n == 1:                         1 -> 1
        return 1                         _ -> fib(n - 2) + fib(n - 1)
    else:                              }
        return fib(n - 2) + fib(n - 1) }
```

The Python version has three `return` statements and the one that executes determines the result. The Gleam version has one expression, and *which branch of the `case` matches* determines the result. Reading the Gleam version: what we return depends on `n`. If `n` is `0`, the value is `0`. If `1`, the value is `1`. Otherwise (`_`) the value is the sum of the two preceding Fibonacci numbers. Each `case` line is a pattern compared against the header value, then an arrow, then the expression to evaluate; exactly one of those expressions is evaluated to produce the value of the whole `case`, which is the value of the function body, which is the value of the call.

### 3. `describe` with concatenation, built up as in the demo

Starting point, matching on the number:

```gleam
import gleam/int

pub fn main() {
  echo describe(12)
  echo describe(1 + 1)
}

fn describe(n: Int) -> String {
  int.to_string(n) <> " is " <> case n {
    1 -> "one"
    2 -> "two"
    _ -> "not one and not two"
  }
}
```

Why it's written this way: the first draft repeated the whole message in all three branches ("n is one", "n is two", "n is neither one nor two"). Noticing the repetition, the shared prefix moves outside the `case`, which is possible because the `case` is an expression with a value. `int.to_string(n)` is needed because Gleam will not concatenate a number with a string, and `import gleam/int` is needed to get `int.to_string` (without it, Gleam reports module not found and suggests the import).

Running it: `describe(12)` gives `"12 is not one and not two"`; `describe(1 + 1)` gives `"2 is two"`.

Now remove the `_` branch and replace it with a `3` pattern:

```gleam
case n {
  1 -> "one"
  2 -> "two"
  3 -> "three"
}
```

The program will not even attempt to run. Gleam reports that the `case` is not exhaustive: there is no pattern covering all the other `Int` values, and it tells you which pattern to add. Adding `_` back fixes it.

### 4. `let` desugared into anonymous function calls

```gleam
fn sum_squares(x, y) {              fn sum_squares(x, y) {
    let x2 = x * x                    fn (x2) {
    let y2 = y * y     equivalent       fn (y2) {
    x2 + y2            to                 x2 + y2
}                                        }(y * y)
                                      }(x * x)
                                    }
```

Read the right-hand version from the inside out. The innermost expression `x2 + y2` is the body of a function of `y2`, which is immediately called on `y * y`. The value of that call is the body of a function of `x2`, which is immediately called on `x * x`. So the whole thing evaluates `x * x`, binds it to `x2`, evaluates `y * y`, binds it to `y2`, and evaluates `x2 + y2`, which is the value of the inner call, which is the value of the outer call, which is the value of `sum_squares`.

In environment terms (described in words, since the diagrams are missing): each immediate call creates a new frame whose single binding is the name being bound by that `let`, with the enclosing frame as its parent. This is exactly what "each `let` introduces a new scope, equivalent to a new local frame" means. Nothing is ever rebound in an existing frame.

### 5. `let` vs Python assignment: why the answers differ

```python
>>> x = 3                      pub fn main() {
>>> f = lambda: x                let x = 3
>>> x = 4                        let f = fn () {x}
>>> f()                          let x = 4
4                                echo f()        // 3
                               }
```

Python: `x = 3` creates a binding for `x` in the global frame. `f` is a lambda whose parent is that frame. `x = 4` *rebinds* the same name in the same frame. When `f()` runs, its body looks up `x` in its parent, finds the current value `4`.

Gleam: `let x = 3` creates a scope where `x` is `3`. `let f = fn () {x}` creates a function whose enclosing scope is that one, so `f`'s `x` is `3` permanently. `let x = 4` creates a *new* scope with a *new* `x`, affecting only expressions after it; it cannot alter the scope `f` already captured. `echo f()` prints `3`. Gleam also warns that this `x` is never used, which is a clue: the `x` that matters to `f` is a different binding.

The general rule: `let` can change what a name means for expressions that come *after* it, never for what has already happened.

### 6. `quadratic`: block scope

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

Step by step: the block runs `let x2 = x * x`, then evaluates `a * x2`. The block's value is the value of its last expression, `a * x2`, and that value is bound to `ax2`. Then `ax2 + b * x + c` is evaluated, which is the function body's value. `ax2` is available on that last line because it was bound outside the block. `x2` is not: it was bound by a `let` inside the block, and that does not escape.

### 7. Blocks for arithmetic order and for multi-line `case` results

```gleam
let celsius = { fahrenheit - 32 } * 5 / 9
echo celsius
```

Instead of parentheses, a block groups the subtraction so it happens before the multiplication. The block evaluates to `fahrenheit - 32`, and that number is then multiplied by 5 and divided by 9.

Blocks in `case` results (factorial, from the demo):

```gleam
fn fact(n: Int) -> Int {
  case n {
    0 -> 1
    _ -> {
      let rest = fact(n - 1)
      n * rest
    }
  }
}
```

The block is the result expression for the `_` pattern, letting that branch do several steps. Its value (`n * rest`) becomes the value of the `case`, hence of the function. Note this version is *not* tail recursive: after `fact(n - 1)` returns there is still a multiplication to do.

### 8. `sum_first` and `sum_from`: iteration by recursion

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

`sum_first(5)` should be `1 + 2 + 3 + 4 + 5 = 15`. `sum_first` just kicks off the helper with the loop's initial values: `k = 1`, `total = 0`. `sum_from` checks whether to keep going (`k <= n`). If so, it recurses with the updated values, which plays the role of `k, total = k + 1, total + k`. If not, it produces `total`, which plays the role of `return total`.

### 9. Evaluation by substitution, non-tail version

```gleam
fn sum_first(n) {
  case n {
    0 -> 0
    _ -> n + sum_first(n-1)
  }
}
```

To find out what `sum_first(5)` does, substitute `5` for `n` everywhere in the body:

```
sum_first(5)
5 + sum_first(4)
5 + 4 + sum_first(3)
5 + 4 + 3 + sum_first(2)
5 + 4 + 3 + 2 + sum_first(1)
5 + 4 + 3 + 2 + 1 + sum_first(0)
5 + 4 + 3 + 2 + 1 + 0
```

At each step: `5` does not match `0`, so it matches `_`, giving `5 + sum_first(5-1)` which is `5 + sum_first(4)`. Substitute `4`, and so on. When the argument reaches `0`, the `0` pattern matches and contributes `0`. Notice the expression gets **longer at every step**. That growth is the space cost of this process.

### 10. Evaluation by substitution, tail version

```gleam
fn sum_from(n, k, total) {
  case k > n {
    True -> total
    False -> sum_from(n, k + 1, total + k)
  }
}
```

```
sum_first(5)
sum_from(5, 1, 0)
sum_from(5, 2, 1)
sum_from(5, 3, 3)
sum_from(5, 4, 6)
sum_from(5, 5, 10)
sum_from(5, 6, 15)
15
```

Tracing it: `sum_first(5)` calls `sum_from(5, 1, 0)`. Is `1 > 5`? No, so take the `False` branch: `sum_from(5, 1 + 1, 0 + 1)` = `sum_from(5, 2, 1)`. Is `2 > 5`? No: `sum_from(5, 3, 3)`. Then `sum_from(5, 4, 6)`, `sum_from(5, 5, 10)`, `sum_from(5, 6, 15)`. Now `6 > 5` is `True`, so the value is `total`, which is `15`.

The crucial contrast: instead of getting longer and longer, each line is **the same size**. That is what it means for the process to have fixed size, and why this runs like a `while` loop.

### 11. Identifying tail contexts

```gleam
fn sum_from(n, k, total) {
  case k > n {
    True -> total
    False -> sum_from(n, k + 1, total + k)   // tail call
  }
}
```

The whole body is a tail context. The body is a `case`, so each of that `case`'s result expressions is in a tail context. The recursive call *is* one of those result expressions, so it is a tail call. Fast, constant space.

```gleam
fn sum_first(n) {
  case n {
    0 -> 0
    _ -> n + sum_first(n-1)   // NOT a tail call
  }
}
```

The body is a tail context, and the `case` results are tail contexts, but `sum_first(n-1)` is not the whole result expression: it's an operand inside `n + ...`. After the call returns, work remains. Not a tail call, grows.

### 12. Tail-recursive factorial, and checking the compiled output

```gleam
fn fact(n: Int) -> Int {
  fact_tail(n, 1)
}

fn fact_tail(n: Int, product: Int) -> Int {
  case n {
    0 -> product
    _ -> fact_tail(n - 1, n * product)
  }
}
```

`fact_tail` carries the next number to multiply (`n`) and the product accumulated so far (`product`). At `n == 0` there's nothing left to do, so return `product`. Otherwise recurse with `n - 1` and `n * product`.

Two things to check. Does it work? Yes, `fact(10)` gives the same answer as the earlier non-tail version. Does it run as a loop? Look at the generated JavaScript in the playground: `fact_tail` now compiles to a `while` loop with no recursive call at all, and `fact` is non-recursive (it just calls `fact_tail`). The earlier blocky version produced no loop. Because the recursive call sits in a tail context, Gleam knows it can run this as a loop.

### 13. Tail-recursive `fib`

```python
def fib(n):                      fn fib(n) {
    prev, curr = 1, 0              fib_iter(1, 0, n)
    while n > 0:                 }
        prev, curr, n = curr, prev + curr, n - 1
    return curr                  fn fib_iter(prev, curr, steps) {
                                   case steps {
                                     0 -> curr
                                     _ -> fib_iter(curr, prev + curr, steps - 1)
                                   }
                                 }
```

`fib_iter` tracks the previous value, the current value, and how many steps remain. The initial state `prev = 1`, `curr = 0` matches the Python loop's initialization. At `0` steps we're done and the answer is `curr`. Otherwise recurse with the previous becoming the current, the current becoming the sum of the last two, and one fewer step.

This is tail recursive: the recursive call is the entire result expression of a `case` that is the entire function body. It runs as fast as the Python loop, and typically faster, because JavaScript is heavily optimized.

Compare with the naive version, which is not tail recursive (the recursive calls are operands of `+`) and also does exponentially redundant work:

```gleam
fn fib(n) {
  case n {
    0 | 1 -> n
    _ -> fib(n-2) + fib(n-1)
  }
}
```

Note the `0 | 1 -> n` trick: both base cases return `n` itself, so they share one result expression.

### 14. Generic `twice`

```gleam
pub fn main() {
  let add_one = fn(x) { x + 1 }
  let exclaim = fn(x) { x <> "!" }

  echo twice(add_one, 10)       // 12
  echo twice(exclaim, "Hello")  // "Hello!!"
}

fn twice(f: fn(t) -> t, x: t) -> t {
  f(f(x))
}
```

Step through `twice(add_one, 10)`: `t` is instantiated as `Int`, so `f` is `add_one` and `x` is `10`. The body `f(f(x))` computes `add_one(10)` = `11`, then `add_one(11)` = `12`.

Step through `twice(exclaim, "Hello")`: `t` is `String`. `exclaim("Hello")` gives `"Hello!"`, then `exclaim("Hello!")` gives `"Hello!!"`.

The signature is what makes both legal: using `t` three times says all three positions share a type, but does not say which. The requirement that `f` return the same type it accepts is exactly what makes `f(f(x))` type-check.

### 15. Exercise: `k_times`

> Write the generic function `fn k_times(f: fn(t) -> t, x: t, k: Int) -> t`, which calls `f` repeatedly `k` times on `x`. For example, `k_times(add_one, 10, 3)` evaluates `add_one(add_one(add_one(10)))`.

Solution (worked out as an application of the lecture's tools, since the slides left it as an exercise):

```gleam
fn k_times(f: fn(t) -> t, x: t, k: Int) -> t {
  case k {
    0 -> x
    _ -> k_times(f, f(x), k - 1)
  }
}
```

Reasoning: `k` is the countdown. With zero applications left, the answer is `x` unchanged. Otherwise apply `f` once and recurse with one fewer application remaining. This is tail recursive, since the recursive call is the whole result expression of the `case` that is the whole body. Tracing `k_times(add_one, 10, 3)`: `k_times(add_one, 11, 2)`, `k_times(add_one, 12, 1)`, `k_times(add_one, 13, 0)`, which returns `13`, the same as `add_one(add_one(add_one(10)))`.

An alternative that is *not* tail recursive and builds the calls outward instead:

```gleam
fn k_times(f: fn(t) -> t, x: t, k: Int) -> t {
  case k {
    0 -> x
    _ -> f(k_times(f, x, k - 1))
  }
}
```

Here `k_times(f, x, k - 1)` is an operand of `f(...)`, so work remains after it returns. Both are correct; the first is the one that compiles to a loop.

## Common Pitfalls

**Writing `return`.** There is no `return` in Gleam. The value of a call is the value of the body expression. Writing multiple exits means writing one `case` with multiple result expressions.

**Writing a standalone `if`.** `if` in Gleam exists only as a guard on a pattern inside a `case`. You cannot write `if cond { ... } else { ... }` as a top-level expression.

**Non-exhaustive `case`.** Forgetting `_` (or otherwise failing to cover every possible value) means the program will not run at all. This is not a runtime surprise on some unlucky input: Gleam refuses to compile and tells you which pattern to add.

**Putting `_` before other patterns.** Patterns are tried top to bottom, so a wildcard placed early shadows everything after it. Put `_` last (guarded wildcards like `_ if n < 0` are an exception, since the guard restricts what they match).

**Using `+` to concatenate strings.** Gleam uses `<>`. Also, you cannot concatenate a number with a string: convert with `int.to_string` and remember `import gleam/int`.

**Expecting Python's type flexibility.** A parameter declared `Int` accepts only `Int`. Mismatches are caught before the program runs, unlike Python which fails only when the bad operation actually executes.

**Expecting `let` to behave like Python assignment.** `let x = 4` after `let x = 3` does not update anything; it introduces a new scope with a new `x` for subsequent expressions. A function that already captured the earlier `x` keeps seeing the old value. If Gleam warns "this `x` is never used," you may be assuming mutation.

**Expecting a `let` inside a block to escape it.** In `quadratic`, `x2` is bound inside the block and is not available after it. Only the block's value (bound to `ax2`) escapes.

**Thinking a recursive call "near the end" is a tail call.** `n + sum_first(n-1)` looks final but isn't: the call is an operand of `+`, so work remains afterward. The test is whether, once that call produces a value, the enclosing computation is finished.

**Assuming recursion means slow.** Tail-recursive Gleam is as fast and as space-efficient as a `while` loop, because the JavaScript backend generates an actual `while` loop (and Erlang handles tail calls natively).

**Assuming all recursion gets optimized.** Only calls in tail contexts do. Non-tail recursion compiles to recursive JavaScript with no loop, and grows.

**Confusing "fixed-size process" with "fewer steps."** Both `sum_first` and `sum_from` take about `n` steps. The difference is the *size of the expression* at each step: growing versus constant.

**Infinite loops in the playground.** Creating one will crash the playground, so make sure the recursion has a reachable base case before running.

## Likely Exam Points

### 1. Translate a Python function to Gleam

**Q.** Translate this to Gleam, with type annotations:
```python
def grade(score):
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    else:
        return "F"
```

**A.** Multiple returns become one `case`, and the comparisons must be guards since they are not literal patterns:
```gleam
pub fn grade(score: Int) -> String {
  case score {
    _ if score >= 90 -> "A"
    _ if score >= 80 -> "B"
    _ -> "F"
  }
}
```
No `return`; the `case`'s value is the function's value. The order matters: a score of 95 would also satisfy `score >= 80`, but the first matching pattern wins.

### 2. Identify tail calls

**Q.** For each marked recursive call, is it in a tail context?
```gleam
fn a(n) { case n { 0 -> 0
                   _ -> a(n - 1) } }           // (i)

fn b(n) { case n { 0 -> 1
                   _ -> n * b(n - 1) } }       // (ii)

fn c(n, acc) { let d = n - 1
               case n { 0 -> acc
                        _ -> c(d, acc + n) } } // (iii)

fn e(n) { case n { 0 -> 0
                   _ -> case n > 10 { True -> e(n - 10)
                                      False -> e(n - 1) } } }  // (iv)
```

**A.**
(i) Yes. It's the whole result expression of a `case` that is the whole body.
(ii) No. It's an operand of `*`; a multiplication remains after it returns.
(iii) Yes. The `case` is the body after a `let`, which counts as a tail context, and the call is a whole result expression.
(iv) Yes, both. The inner `case` is a result expression of the outer `case`, which is in a tail context, so the inner `case`'s results are tail contexts too ("a result of a `case` in a tail context").

### 3. Convert a non-tail-recursive function to tail-recursive

**Q.** Rewrite this tail-recursively:
```gleam
fn count_down_sum(n) {
  case n {
    0 -> 0
    _ -> n + count_down_sum(n - 1)
  }
}
```

**A.** Add an accumulator parameter so the pending addition happens *before* the recursive call rather than after:
```gleam
fn count_down_sum(n) {
  cds_iter(n, 0)
}

fn cds_iter(n, total) {
  case n {
    0 -> total
    _ -> cds_iter(n - 1, total + n)
  }
}
```
Now nothing remains to be done after the recursive call returns, so it is a tail call.

### 4. `let` scope vs Python assignment

**Q.** What does each print?
```python
x = 10                         pub fn main() {
g = lambda: x * 2                let x = 10
x = 20                           let g = fn () { x * 2 }
print(g())                       let x = 20
                                 echo g()
                               }
```

**A.** Python prints `40`; Gleam prints `20`. Python's `x = 20` rebinds the name in the frame `g` looks up, so `g` sees `20` and doubles it. Gleam's second `let x = 20` creates a new scope with a new `x`; `g` captured the scope where `x` is `10`, so `g()` is `20`. Gleam also warns that the second `x` is unused.

### 5. Block scope

**Q.** Does this compile? If not, why?
```gleam
pub fn f(x: Int) -> Int {
  let y = {
    let z = x + 1
    z * 2
  }
  y + z
}
```

**A.** No. `z` is bound by a `let` inside the block, so it does not exist outside it: `y + z` refers to an unknown name. The block's only contribution to the outside is its value (`z * 2`), which is bound to `y`. Fix by using `y` alone, or by binding `z` outside the block.

### 6. Evaluation by substitution

**Q.** Use substitution to show the sequence of expressions for `sum_from(3, 1, 0)` with:
```gleam
fn sum_from(n, k, total) {
  case k > n { True -> total
               False -> sum_from(n, k + 1, total + k) }
}
```
Does the expression grow?

**A.**
```
sum_from(3, 1, 0)
sum_from(3, 2, 1)
sum_from(3, 3, 3)
sum_from(3, 4, 6)
6
```
`1 > 3` false, so recurse with `k + 1 = 2`, `total + k = 1`. Then `2 > 3` false: `(3, 3, 3)`. Then `3 > 3` false: `(3, 4, 6)`. Then `4 > 3` true, so the value is `total = 6`. The expression does **not** grow: each line is the same size, which is why this runs like a `while` loop.

### 7. Higher-order functions and function types

**Q.** What is the type of `make_adder` below, and what does `make_adder(5)(7)` evaluate to?
```gleam
fn make_adder(n: Int) -> fn(Int) -> Int {
  fn(k) { n + k }
}
```

**A.** `make_adder` has type `fn(Int) -> fn(Int) -> Int`: it takes an `Int` and returns a one-argument integer function. `make_adder(5)` returns the anonymous function `fn(k) { 5 + k }`; calling it on `7` gives `12`.

### 8. Write a higher-order function signature

**Q.** Write the type annotation for a function `apply_both` that takes two one-argument integer functions and an integer, and returns an integer.

**A.** `fn apply_both(f: fn(Int) -> Int, g: fn(Int) -> Int, x: Int) -> Int`. Each function parameter's type is written with `fn(...) -> ...`, same keyword used to create anonymous functions, but in type position.

### 9. Generic functions

**Q.** Why does `twice(f: fn(t) -> t, x: t) -> t` require `f`'s argument and return types to match, and what would break if `f` had type `fn(t) -> u`?

**A.** The same lowercase name `t` appears in all three positions, which requires them to all be the same type (whichever type the call site supplies). The body is `f(f(x))`: the result of the inner `f(x)` is fed back into `f`, so `f`'s return type must be something `f` can accept. With `fn(t) -> u`, the inner call produces a `u` but `f` requires a `t`, so `f(f(x))` would not type-check unless `u` and `t` happened to be the same.

### 10. Why immutability matters for purity

**Q.** In one or two sentences, explain why a language that wants pure functions makes its values immutable.

**A.** A pure function does nothing but compute a result from its inputs. Mutating a value changes how the rest of the program behaves in a way not captured by the return value, so a mutating function is not pure. Making all values immutable removes that possibility.

### 11. Guards and `Nil`

**Q.** Explain why these two definitions of `sign` are equivalent:
```gleam
case n { _ if n > 0 -> 1        case Nil { _ if n > 0 -> 1
          _ if n < 0 -> -1                  _ if n < 0 -> -1
          _ -> 0 }                          _ -> 0 }
```

**A.** Every pattern is a wildcard, so the matched value is never actually inspected; all the decision-making is done by the guards, which reference `n` directly from the surrounding scope. Since the matched value is irrelevant, it can be `Nil` instead of `n` with no change in behavior.

### 12. Compilation to JavaScript

**Q.** Gleam compiles to JavaScript, which is not tail-recursive. How does tail-recursive Gleam code still run in constant space?

**A.** The Gleam compiler generates a `while` loop for recursive tail calls: parameters become `loop$`-prefixed variables that are reassigned at the bottom of the loop instead of a recursive call being made. So the compiled program is literally a loop. (Compiling to Erlang needs no such transformation, since Erlang is itself tail-recursive.)

## Summary

- Three perspectives: imperative (instructions updating memory), object-oriented (objects behaving like the entities they represent), functional (data definitions and pure functions generating data). Not mutually exclusive; Python supports all three.
- A pure function does nothing except compute a result from its inputs. Immutability is required for purity: mutation changes program behavior outside the return value.
- Benefits of leaning functional: easier to understand, and separate parts can run concurrently without one mutating what another reads.
- Gleam: function bodies are expressions, functions are first-class, anonymous functions used liberally, all values immutable, no `for`/`while`.
- Why Gleam: small and simple (functions, names, `case`), good vehicle for algebraic data types and pattern matching, good preparation for Rust/Swift/TypeScript.
- Programs are modules (files). There must be a `pub fn main()`. `pub` means callable from other modules. `echo` is like Python's `print`.
- Expression forms: literals (`2`, `3.3`, `True`, `"Hello"`), calls, `let`, `echo`, `case`.
- No `return`: the value of a call is the value of the body expression. Multiple Python `return`s become one `case` with multiple result expressions.
- `case` matches a value against patterns top to bottom; `_` is the wildcard, `|` shares a result expression among patterns. Non-exhaustive `case` is a compile-time error that names the missing pattern.
- Gleam is statically typed with strong inference; annotate function signatures anyway as documentation. `<>` concatenates strings; use `int.to_string` (with `import gleam/int`) to mix in numbers.
- `if` exists only as a guard on a pattern inside `case`. A guarded wildcard like `_ if n < 0` is how comparisons get used as conditions. `Nil` is the value returned by functions with nothing else to return.
- `fn` creates anonymous functions (`fn(x, y) { x + y }`) and writes function types (`fn(Int, Int) -> Int`), which together build higher-order functions like `make_adder` and `combine`.
- `let` binds a name and introduces a new scope (a new local frame), equivalent to creating and immediately calling an anonymous function. It affects only expressions after it, so unlike Python assignment it cannot change what an already-created function sees.
- Blocks `{ ... }` hold multiple expressions, evaluate to their last one, keep their `let` bindings private, and replace parentheses for arithmetic grouping.
- No loops means recursion for iteration: loop variables become helper parameters, the loop condition becomes the `case` header, the loop body becomes the recursive call's arguments, and `return` becomes the terminating branch.
- Evaluation by substitution (replace each formal parameter with its argument in the body) is valid because there is no mutation or reassignment.
- Tail context: the body of a function (after any `let`), and a result of a `case` in a tail context. A call inside a larger expression is not in a tail context. Intuition: in a tail context, once the value is computed you are done.
- Tail-recursive calls run as fast and in as little space as a `while` loop; non-tail recursion produces a growing process.
- Gleam compiles to JavaScript (browser) or Erlang (distributed messaging). The JavaScript backend emits actual `while` loops for tail calls; the playground lets you check whether your function produced one.
- Tail-recursive patterns seen: `sum_from` with a running `total`, `fact_tail` with a running `product`, `fib_iter` with `prev`, `curr`, `steps`.
- Lowercase type names are generic placeholders. Repeating the same name (`fn twice(f: fn(t) -> t, x: t) -> t`) requires those positions to share one type, whichever the call site supplies.
