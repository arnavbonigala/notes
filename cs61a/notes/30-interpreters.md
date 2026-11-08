<!-- Wed, Nov 04, 2026 | sources: slides (no transcript available) -->
# Lecture 30: Interpreters

This lecture turns the course's attention from writing programs to writing the thing that runs programs. It opens with a practical detour: because Gleam compiles to JavaScript, you can declare `@external` functions that are implemented in JavaScript, and since JavaScript allows mutation, those external functions let you smuggle mutable state (a cache, a counter) into an otherwise immutable Gleam program. From there the lecture introduces the two interpreters you will build: a small `Calculator` language (Homework 8) that handles integer arithmetic with variables plus anonymous function definition and application, and the full `Evaluator` for the Animator project, which adds environments as sequences of frames, recursive user-defined functions, many more built-ins, and more expression and value types. The core machinery is three pieces: a parser that turns program text into an abstract syntax tree represented by `Expr` values, an `eval` function that takes an `Expr` and an `Env` and produces a `Value` (or an `EvalError`), and an `apply` function that takes a function value plus argument values and produces a result. The lecture closes by naming exactly which exam questions to expect (how is expression X represented, how is value Y represented, what gets evaluated when evaluating Z) and demos tracing evaluation by printing every expression evaluated, every application, and every return.

## Key Concepts

### External functions, and how immutable Gleam gets mutable state

Gleam has no mutation. No assignment to an existing binding, no mutable fields, no mutable dictionaries. That is a language-level guarantee, and it is why everything you have written in Gleam so far has had to thread state through arguments and return values.

But Gleam compiles to JavaScript, and JavaScript has unrestricted mutation. The `@external` attribute declares that a Gleam function's implementation lives in a JavaScript module rather than in Gleam:

```
@external(javascript, "./fib_ffi.mjs", "new_cache")
pub fn new_cache() -> Cache
```

Three pieces: the target (`javascript`), the module path (`"./fib_ffi.mjs"`), and the name of the exported JavaScript function (`"new_cache"`). The Gleam side gives a type signature and no body. The JavaScript side gives a body and no types.

The consequence is the interesting part. The JavaScript function can return a JavaScript object, and Gleam can hold onto that object through an opaque type:

```
pub type Cache
```

That declaration has no constructors. Gleam code cannot build a `Cache`, cannot pattern match on one, cannot inspect it. All it can do is pass it to the external functions that know what it really is. So `Cache` is a Gleam type name for a JavaScript object, and because that object is a JavaScript `Map`, it is mutable. You now have a mutable object in Gleam.

This is a deliberate escape hatch, not a loophole. Gleam marks it clearly at the boundary: every `@external` line is a place where the immutability guarantee stops holding.

### Types crossing the boundary

The JavaScript side has to produce values the Gleam side can consume. In `fib_ffi.mjs`:

```javascript
import { Ok, Error } from "./gleam.mjs";

export function lookup(cache, n) {
  return cache.has(n) ? new Ok(cache.get(n)) : new Error(undefined);
}
```

The Gleam compiler emits a `gleam.mjs` runtime module containing JavaScript classes for Gleam's built-in types. Importing `Ok` and `Error` from it lets JavaScript construct genuine Gleam `Result` values. The Gleam declaration

```
fn lookup(cache: Cache, n: Int) -> Result(Int, Nil)
```

says this returns a `Result(Int, Nil)`, and the JavaScript honors that by returning `new Ok(...)` or `new Error(undefined)` (JavaScript's `undefined` standing in for Gleam's `Nil`). The ternary `? :` is JavaScript's if/else expression, playing the role Gleam's `case` plays.

Note that the Gleam compiler cannot check this. If the JavaScript returned a bare number instead of an `Ok`, Gleam would believe the type signature and the program would break at runtime. The `@external` boundary is where you are trusting yourself.

### Consuming an external `Result` on the Gleam side

Back in Gleam, the `Result` gets unwrapped with the same `case` you have always used:

```
pub fn main() {
  let cache = new_cache()
  store(cache, 5, 25)
  let result = case lookup(cache, 5) {
    Ok(v) -> v
    Error(Nil) -> 0
  }
}
```

Trace this. `new_cache()` calls into JavaScript and gets back an empty `Map`, which Gleam holds as a `Cache`. `store(cache, 5, 25)` calls into JavaScript, mutates that same `Map` (the one `cache` points at, not a copy), and returns `25`. Then `lookup(cache, 5)` finds the key and returns `Ok(25)`, so the `case` matches the first arm and `result` is assigned `25`.

The mutation is the whole point. In pure Gleam, `store` would have to return a new cache and you would have to use that returned value. Here `cache` is never rebound and yet its contents changed. Thinking in box-and-pointer terms: `cache` is one name bound to one arrow pointing at one JavaScript `Map` object; `store` reached through that arrow and changed what the object contains, so a later `lookup` through the same arrow sees the change.

### Counting function calls: mutable state as instrumentation

The exercise applies this to a measurement problem. Here is naive `fib`:

```
/// The nth Fibonacci number.
pub fn fib(n: Int) -> Int {
  case n {
    0 | 1 -> n
    _ -> fib(n - 2) + fib(n - 1)
  }
}
```

The question is how many function calls `fib(42)` makes. You cannot answer that with a pure return value without changing what `fib` returns. With a mutable counter you can:

```javascript
export function new_counter() {
  return { calls: 0 };
}
export function increment(counter) {
  counter.calls += 1;
}
export function count(counter) {
  return counter.calls;
}
```

`new_counter` creates a JavaScript object with an attribute named `calls` whose value is `0`. `increment` mutates that attribute in place and returns nothing (Gleam types it as `Nil`). `count` reads it.

The completed Gleam:

```
pub fn fib(n: Int, counter: Counter) -> Int {
  increment(counter)
  case n {
    0 | 1 -> n
    _ -> fib(n - 2, counter) + fib(n - 1, counter)
  }
}

pub fn main() {
  let counter = new_counter()
  fib(42, counter)
  echo count(counter)
}
```

Four things to notice. The return type is still `Int`: counting does not change what `fib` computes. `increment(counter)` is the first statement in the body, so every single entry into `fib` is counted, base cases included. The `counter` must be threaded explicitly through both recursive calls, since Gleam has no globals. And one `Counter` object is shared by all of the exponentially many recursive calls, which is exactly why the total accumulates instead of each call having its own count.

The same technique is what makes memoization possible: the cache example above is the other half of this. A pure-Gleam `fib` cannot memoize, because each call would get its own copy of the cache. (extra context) The call count for naive `fib(42)` is large enough that the demo is dramatic: the number of calls grows like the Fibonacci numbers themselves.

### The two interpreters you will write

The lecture names two distinct targets, and keeping them apart matters for planning your work.

**Calculator** (Homework 8, assigned Friday) represents integer arithmetic with variables, so `(x + 3) * 5` is a legal program, and it supports defining and applying anonymous functions, so `fn(x, y) { 2 * y - x }` is too. That is the whole language: integers, variables, the four arithmetic operators, lambdas, and calls.

**Evaluator** (the Animator project) is the full version, and the lecture lists exactly how it is bigger:

- An environment is a **sequence of frames**, not a single mapping. This is what makes lexical scoping and nested function definitions work.
- User-defined functions **can be recursive**, which requires that a function's body be able to look up the function's own name.
- **More built-in functions and operators**, beyond `+`, `-`, `*`, `/`.
- **More types of expressions**: `case`, `let`, and dot expressions (field access).
- **More types of values**: bools, lists, functions, and records.

### Abstract syntax trees: programs as data

An interpreter does not work on text. The first stage, parsing, converts program text into a tree structure, and everything after that works on the tree. The lecture's framing: "The abstract syntax tree for some expression in a program is represented by an `Expr` value."

This is the central idea of the whole unit. A program is just a data structure. Once `1 + 1` is an `Expr` value rather than a three-character string, evaluating it is ordinary recursive processing of a tree, the same skill you have been practicing all semester on lists and trees.

Here is the `Expr` type for the useful subset of Gleam the project handles, with the example expression each variant represents:

```
pub type Expr {
  Var(String)                                              // x
  LiteralBool(Bool)                                        // True
  LiteralInt(Int)                                          // 61
  LiteralList(elems: List(Expr), rest: Option(Expr))       // [1, 2]  or  [1, 2, ..s]
  Call(fn_expr: Expr, operands: List(Expr))                // f(5)
  Lambda(params: List(String), body: Expr)                 // fn (x) {x > 5}
  Case(subject: Expr, arms: List(Arm))                     // case x {0 -> False  _ -> True}
  Let(name: String, value: Expr, body: Expr)               // let x = 5
  FieldAccess(record: Expr, field: String)                 // point.x
}
```

Several details are worth dwelling on.

`LiteralList` has two parts: the explicitly listed elements and an optional `rest`. `[1, 2]` has `elems` of length two and no `rest`; `[1, 2, ..s]` has the same `elems` plus a `rest` of `Some(Var("s"))`. The `rest` is itself an `Expr`, which must be evaluated to get a list to append onto.

`Call` stores `fn_expr` as an `Expr`, not as a name. That is what makes `f(5)` and `(fn(x) {x}) (5)` and `get_func()(5)` all representable with one variant: whatever expression produces the function gets evaluated like any other.

`Case` has `arms: List(Arm))`, where an "arm" is one `pattern -> result` clause of a case expression.

`Let` is the subtlest. It has three fields: `name`, `value`, and `body`. But Gleam source has no visible body for a `let`; you just write `let x = 5` and then more lines. The slide annotation resolves this: the `body` is "the expression on the line after the let line." So a sequence of statements gets parsed into nested `Let` expressions, with each `let`'s body being everything that follows it. This is why a `let` binding is visible to subsequent lines but not to earlier ones: scope falls out of the tree shape.

`FieldAccess` holds the `record` as an `Expr` (so you can write `f(x).y`) and the `field` as a plain `String` (field names are not computed).

### Representing values

Expressions and values are different types. Evaluating an expression in an environment produces a value, or an evaluation error:

```
pub type Value {
  VBool(Bool)                                  // True
  VInt(Int)                                    // 61
  VList(List(Value))                           // [1, 2]
  VData(tag: String, fields: List(Value))      // Position(5, 10)
  VFunc(Callable)                              // fn (x) {x > 5}
}
```

Compare this to `Expr` carefully, because conflating the two is the most common source of confusion.

`VList(List(Value))` holds a list of **values**, with no `rest` field. By the time you have a value, the `..rest` has been evaluated and spliced in; there is nothing left to defer. Contrast `LiteralList(elems: List(Expr), rest: Option(Expr))`, which holds unevaluated expressions.

`VData(tag, fields)` represents a record such as `Position(5, 10)`: a tag naming the constructor plus a list of field values. This is how custom types are represented at runtime, and it is what `FieldAccess` reaches into.

There is no `VVar`, because a variable is not a value. There is no `Var` equivalent on the value side at all: evaluating `Var("x")` looks `x` up and yields whatever value is bound there.

### Two kinds of callable

`VFunc` wraps a `Callable`, which has two forms:

```
pub type Callable {
  Builtin(name: String, num_args: Int, run: fn(List(Value)) -> Result(Value, EvalError))
  UserFn(name: String, params: List(String), body: Expr, env: Env)
}
```

A `Builtin` "calls some existing Gleam function." Its `run` field is an actual Gleam function of type `fn(List(Value)) -> Result(Value, EvalError)`: it takes the already-evaluated arguments and produces a result directly. This is how `+` and `*` and the other primitives get implemented: you do not interpret them, you delegate to real Gleam code. `num_args` lets the interpreter check arity before calling.

A `UserFn` is a function written in the interpreted language. It has a `body: Expr` that gets evaluated when the function is called, `params` naming the formal parameters, and crucially `env: Env`, "the environment in which it was defined."

That `env` field is the entire mechanism of lexical scoping. A `UserFn` carries its defining environment with it, so when it is later called from somewhere else entirely, the body is evaluated in a new frame whose parent is the defining environment, not the calling environment. This is the interpreter-level statement of everything you learned about frames and parent pointers in the Python environment-diagram unit.

### Eval and apply: the mutual recursion at the heart of it

Two functions, each calling the other, do all the work.

```
fn eval(expr: Expr, env: Env) -> Result(Value, EvalError) {
  case expr {
    Var(name) -> lookup(env, name)
    LiteralBool(b) -> Ok(VBool(b))
    LiteralInt(n) -> Ok(VInt(n))
    LiteralList(elems, rest) ->   // Evaluate & combine the elems and rest
    Call(fn_expr, operands) ->    // Evaluate the fn_expr & operands, then call apply
    Lambda(params, body) ->       // Construct a UserFn("lambda", ...)
    Case(s, arms) ->              // Evaluate s, then match its value to one of the arms
    Let(x, v, body) ->            // Evaluate v, bind its value to x, and then evaluate body
    FieldAccess(record, field) -> // Evaluate record, make sure it's a VData, and get a field
  }
}

pub fn apply(f: Value, args: List(Value)) -> Result(Value, EvalError) {
  case f {
    VFunc(Builtin(name, num_args, run)) ->       // Call run (a Gleam function) on args
    VFunc(UserFn(name, params, body, parent)) -> // Make a Local frame e, then eval(body, e)
  }
}
```

The signatures tell you most of the story. `eval` needs an environment because expressions contain variables whose meaning depends on where you are. `apply` does **not** take an environment, because its arguments are already values and the function value already carries its own environment in its `env` field. That asymmetry is not an accident; it is why lexical scoping works.

The three fully written cases show the base of the recursion. `LiteralInt(n) -> Ok(VInt(n))` is the simplest possible evaluation: an `Expr` that names an integer becomes a `Value` that is that integer. `Var(name) -> lookup(env, name)` is the only case that consults the environment directly, and note that `lookup` returns a `Result`, so an unbound name becomes an `EvalError` rather than a crash.

The mutual recursion runs through `Call` and `apply`. Evaluating `Call(fn_expr, operands)` means: evaluate `fn_expr` (recursive `eval`), evaluate every operand (recursive `eval` on each), then hand the resulting function value and argument values to `apply`. And `apply` on a `UserFn` means: make a new local frame binding `params` to `args` with `parent` as its parent, then `eval(body, e)`, which is back in `eval`. Every nested function call in the interpreted program corresponds to one lap around this loop.

`Lambda` is the case where no work happens yet. Evaluating `fn (x) {x > 5}` does not evaluate `x > 5`; it constructs a `UserFn("lambda", ["x"], <the body Expr>, env)`. The body is stored unevaluated and the current `env` is captured. Nothing in the body runs until `apply`.

`Let(x, v, body)` is three steps in order: evaluate `v`, bind the resulting value to `x`, evaluate `body` in the extended environment. The value expression is evaluated in the environment **before** the binding exists, which is why a non-recursive `let` cannot refer to itself.

Every branch returns a `Result(Value, EvalError)`, so errors (unbound variable, wrong arity, field access on a non-record, no matching case arm) propagate as data rather than as exceptions.

### Tracing evaluation

The final demo makes the recursion visible. Given:

```
fn f(s) {
  case s {
    [] -> [3]
    [first, ..rest] -> [first, ..f(rest)]
  }
}
```

the question is what happens when you evaluate `f([1, 2])`. To see it, instrument the interpreter to print:

- each expression being evaluated,
- each function being applied and its arguments,
- again each time a function returns a value.

This is a powerful idea: because the interpreter is a program you wrote, you can add logging to it and get a complete trace of the interpreted program for free. (extra context) `f([1, 2])` evaluates to `[1, 2, 3]`: the list pattern peels off `1`, then `2`, then hits the empty list and returns `[3]`, and the `[first, ..f(rest)]` arms rebuild the list on the way back out.

The important thing the trace reveals is how many `eval` calls one small source expression generates. Evaluating `f([1, 2])` means evaluating the `Call`, which means evaluating `Var("f")` and `LiteralList([LiteralInt(1), LiteralInt(2)], None)`, which means evaluating two `LiteralInt`s, and then applying, which means evaluating the `Case`, which means evaluating its subject, matching an arm, and evaluating that arm's result expression, which contains another `Call`.

## Definitions

**Interpreter**: a program that takes a representation of another program as input and carries out its computation, producing values.

**`@external` attribute**: a Gleam annotation declaring that a function's implementation lives in another language's module rather than in Gleam. Takes the target platform, the module path, and the exported name in that module: `@external(javascript, "./fib_ffi.mjs", "new_cache")`.

**Opaque type with no constructors**: a Gleam type declared as just `pub type Cache` with no variants. Gleam code can hold and pass values of this type but cannot construct, inspect, or pattern match them. Used to name a foreign (JavaScript) object from Gleam.

**`gleam.mjs`**: the Gleam runtime module emitted by the compiler, exporting JavaScript classes for Gleam's built-in types. JavaScript external code imports from it (`import { Ok, Error } from "./gleam.mjs"`) to construct real Gleam values.

**Mutation**: changing the contents of an existing object in place, so that every reference to that object sees the change. Not available in Gleam; available in JavaScript, and reachable from Gleam only through `@external` functions.

**Calculator**: the interpreter written for Homework 8. Its language is integer arithmetic with variables, such as `(x + 3) * 5`, plus definition and application of anonymous functions, such as `fn(x, y) { 2 * y - x }`.

**Evaluator**: the interpreter written for the Animator project. Adds environments as sequences of frames, recursive user-defined functions, more built-ins and operators, more expression types (`case`, `let`, dot expressions), and more value types (bools, lists, functions, records).

**Parsing**: the stage that converts program text into an abstract syntax tree.

**Abstract syntax tree (AST)**: the tree-structured representation of a program's grammatical structure. In this project, the AST for an expression is represented by an `Expr` value.

**`Expr`**: the Gleam type whose variants represent each kind of expression in the interpreted language: `Var`, `LiteralBool`, `LiteralInt`, `LiteralList`, `Call`, `Lambda`, `Case`, `Let`, `FieldAccess`.

**Arm**: one `pattern -> result` clause of a `case` expression. `Case` holds a `List(Arm)`.

**`Let` body**: the third field of a `Let` expression, which is the expression on the line after the `let` line. Sequential statements parse into nested `Let`s.

**`Value`**: the Gleam type whose variants represent each kind of runtime value in the interpreted language: `VBool`, `VInt`, `VList`, `VData`, `VFunc`.

**`VData(tag, fields)`**: the value representation of a record, holding the constructor's tag as a string and the field values as a list. `Position(5, 10)` becomes `VData("Position", [VInt(5), VInt(10)])`.

**`Callable`**: the type wrapped by `VFunc`, with two variants, `Builtin` and `UserFn`.

**`Builtin(name, num_args, run)`**: a function value implemented by an existing Gleam function. Its `run` field has type `fn(List(Value)) -> Result(Value, EvalError)` and is called directly on the evaluated arguments.

**`UserFn(name, params, body, env)`**: a function value defined in the interpreted language. `body` is an unevaluated `Expr`; `env` is the environment in which the function was defined.

**Environment (`Env`)**: a sequence of frames mapping names to values. Variable lookup searches it.

**`eval(expr, env)`**: the function that evaluates an `Expr` in an `Env`, returning `Result(Value, EvalError)`.

**`apply(f, args)`**: the function that calls a function value `f` on already-evaluated argument values `args`, returning `Result(Value, EvalError)`. Takes no environment.

**`EvalError`**: the error type returned when evaluation cannot produce a value (for example, an unbound variable).

## Worked Examples

The lecture's code is in Gleam and JavaScript. Per the assignment's request for Python, the examples below reimplement the lecture's own examples in Python, keeping the structure of the Gleam code visible so the mapping is exact.

### Example 1: The mutable counter, and why Gleam needs an external function for it

The lecture's exercise threads a mutable counter through `fib`. In Python, mutable objects are the default, so this is the version that shows what Gleam has to go outside the language to get:

```python
def new_counter():
    return {"calls": 0}

def increment(counter):
    counter["calls"] += 1

def count(counter):
    return counter["calls"]

def fib(n, counter):
    increment(counter)
    if n == 0 or n == 1:
        return n
    return fib(n - 2, counter) + fib(n - 1, counter)

counter = new_counter()
fib(10, counter)
print(count(counter))
```

Step by step. `new_counter()` builds a dict and returns it; `counter` in the global frame is bound to an arrow pointing at that dict. `fib(10, counter)` is called, and the name `counter` inside `fib`'s frame is bound to an arrow pointing at the **same** dict, not a copy. `increment(counter)` reaches through that arrow and changes `calls` from `0` to `1`. Then `fib` recurses twice, each recursive call passing the same arrow along, each one incrementing the same dict. By the time the outermost call returns, every call in the tree has bumped the one shared dict.

In box-and-pointer terms: one box for the dict, many arrows pointing at it from many frames, and `increment` modifying the contents of the box rather than creating a new box.

Why Gleam cannot do this natively: `counter["calls"] += 1` is mutation, and Gleam has no such operation. A pure Gleam version would need `increment` to return a new counter, and `fib` would have to return both the Fibonacci number and the updated counter, threading it through the additions. The `@external` route is what lets the Gleam signature stay `pub fn fib(n: Int, counter: Counter) -> Int`.

Why `increment` is the first statement: it runs on every entry to `fib`, including entries that immediately return in the base case. If you instead incremented only in the recursive branch, you would be counting internal nodes of the call tree and not leaves.

### Example 2: The cache, and the `Result` boundary

```python
def new_cache():
    return {}

def lookup(cache, n):
    if n in cache:
        return ("Ok", cache[n])
    return ("Error", None)

def store(cache, n, value):
    cache[n] = value
    return value

cache = new_cache()
store(cache, 5, 25)
match lookup(cache, 5):
    case ("Ok", v):
        result = v
    case ("Error", _):
        result = 0
print(result)
```

The `("Ok", v)` / `("Error", None)` tuples stand in for Gleam's `Result`. In the real code, the JavaScript returns `new Ok(cache.get(n))` or `new Error(undefined)`, constructed from the imported `gleam.mjs` classes, and Gleam pattern matches with `case lookup(cache, 5) { Ok(v) -> v  Error(Nil) -> 0 }`.

Trace: `cache` points at an empty dict. `store(cache, 5, 25)` mutates that dict to `{5: 25}` and returns `25`, which the Gleam version discards (it is called as a statement). `lookup(cache, 5)` sees `5 in cache`, so it returns `("Ok", 25)`, the first match arm fires, and `result` is `25`.

The important structural point: `store` returns `value` rather than the cache. The caller does not need the cache back, because the cache it already holds has changed. That is only true because of mutation; it is the signature you cannot write in pure Gleam.

### Example 3: Building the AST by hand

Before writing `eval`, get comfortable with the fact that an expression is a data structure. Here are the lecture's `Expr` variants as Python classes:

```python
from dataclasses import dataclass, field

@dataclass
class Var:
    name: str

@dataclass
class LiteralBool:
    value: bool

@dataclass
class LiteralInt:
    value: int

@dataclass
class LiteralList:
    elems: list
    rest: object = None

@dataclass
class Call:
    fn_expr: object
    operands: list

@dataclass
class Lambda:
    params: list
    body: object

@dataclass
class Let:
    name: str
    value: object
    body: object

@dataclass
class FieldAccess:
    record: object
    field: str
```

Now the lecture's own examples, written as trees:

```python
x_expr        = Var("x")
true_expr     = LiteralBool(True)
sixty_one     = LiteralInt(61)
list_expr     = LiteralList([LiteralInt(1), LiteralInt(2)])
list_rest     = LiteralList([LiteralInt(1), LiteralInt(2)], Var("s"))
call_expr     = Call(Var("f"), [LiteralInt(5)])
lambda_expr   = Lambda(["x"], Call(Var(">"), [Var("x"), LiteralInt(5)]))
field_expr    = FieldAccess(Var("point"), "x")
```

Read `call_expr` carefully. `f(5)` becomes `Call(Var("f"), [LiteralInt(5)])`. The function position holds `Var("f")`, an expression that must itself be evaluated, not the string `"f"`. That is why `Call` has field `fn_expr: Expr`.

Read `lambda_expr` carefully too. The body of `fn (x) {x > 5}` is itself a `Call`: the operator `>` applied to `x` and `5`. Operators are just function calls in the AST, which is why `Builtin` exists.

Now the first exam question from the slides, answered: **how is `1 + 1` represented?** It is a call to the `+` function with two integer literals as operands:

```python
one_plus_one = Call(Var("+"), [LiteralInt(1), LiteralInt(1)])
```

Not `LiteralInt(2)`. The AST records what was written, not what it computes.

And: **how is the value `[1, 2]` represented?** As a `VList` of two `VInt`s:

```python
value_of_list = ("VList", [("VInt", 1), ("VInt", 2)])
```

Not `LiteralList([LiteralInt(1), LiteralInt(2)])`, which is the **expression** `[1, 2]`, and not the Python list `[1, 2]`. Expression types and value types are different types with different variants.

### Example 4: `eval` and `apply`, written out

This is the lecture's skeleton filled in enough to run, in Python. It is the same structure as the Gleam: a dispatch on expression type, with `Call` handing off to `apply` and `apply` handing back to `eval`.

```python
class EvalError(Exception):
    pass

@dataclass
class UserFn:
    name: str
    params: list
    body: object
    env: object

@dataclass
class Builtin:
    name: str
    num_args: int
    run: object

class Env:
    def __init__(self, bindings, parent=None):
        self.bindings = bindings
        self.parent = parent

    def lookup(self, name):
        if name in self.bindings:
            return self.bindings[name]
        if self.parent is None:
            raise EvalError("unbound name: " + name)
        return self.parent.lookup(name)


def eval_expr(expr, env):
    if isinstance(expr, LiteralInt):
        return ("VInt", expr.value)
    if isinstance(expr, LiteralBool):
        return ("VBool", expr.value)
    if isinstance(expr, Var):
        return env.lookup(expr.name)
    if isinstance(expr, LiteralList):
        elems = [eval_expr(e, env) for e in expr.elems]
        if expr.rest is not None:
            rest_value = eval_expr(expr.rest, env)
            if rest_value[0] != "VList":
                raise EvalError("rest of a list must be a list")
            elems = elems + rest_value[1]
        return ("VList", elems)
    if isinstance(expr, Lambda):
        return ("VFunc", UserFn("lambda", expr.params, expr.body, env))
    if isinstance(expr, Let):
        value = eval_expr(expr.value, env)
        return eval_expr(expr.body, Env({expr.name: value}, env))
    if isinstance(expr, FieldAccess):
        record = eval_expr(expr.record, env)
        if record[0] != "VData":
            raise EvalError("field access on a non-record")
        tag, fields, field_names = record[1], record[2], record[3]
        return fields[field_names.index(expr.field)]
    if isinstance(expr, Call):
        fn_value = eval_expr(expr.fn_expr, env)
        args = [eval_expr(o, env) for o in expr.operands]
        return apply(fn_value, args)
    raise EvalError("unknown expression type")


def apply(f, args):
    if f[0] != "VFunc":
        raise EvalError("cannot call a non-function")
    callable_ = f[1]
    if isinstance(callable_, Builtin):
        if len(args) != callable_.num_args:
            raise EvalError("wrong number of arguments")
        return callable_.run(args)
    if isinstance(callable_, UserFn):
        if len(args) != len(callable_.params):
            raise EvalError("wrong number of arguments")
        local = Env(dict(zip(callable_.params, args)), callable_.env)
        return eval_expr(callable_.body, local)
    raise EvalError("not callable")
```

Line by line on the cases that matter.

`LiteralInt` and `LiteralBool` are the base cases: no recursion, no environment consulted, just wrap the literal in the corresponding value variant.

`Var` is the only case that touches the environment directly. `env.lookup` walks the parent chain, frame by frame, and raises `EvalError` if it reaches the end. This frame-chain walk is the "sequence of frames" the lecture describes.

`LiteralList` evaluates each element expression to a value, then if there is a `rest`, evaluates it and appends its elements. The result is a `VList` of values with no `rest` field, because the splicing has already happened. This is exactly why `Value` has `VList(List(Value))` while `Expr` has `LiteralList(elems, rest)`.

`Lambda` is the case with no recursive `eval` call. It captures `env` into the `UserFn` and stores `expr.body` unevaluated. Nothing in the body is looked at. If the body contained `Var("undefined_name")`, evaluating the lambda would succeed; only applying it would error.

`Let` evaluates `expr.value` in the **current** `env`, then builds a new `Env` with one binding whose parent is the current one, then evaluates `expr.body` there. The ordering is the whole of `let` semantics: the value cannot see the new binding, the body can.

`Call` is the pivot. Evaluate the function expression, evaluate every operand, then `apply`. Note that all of the operands are evaluated before `apply` is called: arguments are values by the time the function sees them.

`apply` on a `Builtin` just calls the stored Gleam (here, Python) function on the argument values. No `Expr` is involved, no environment is created. This is the escape from interpretation into the host language.

`apply` on a `UserFn` creates a new frame, `local`, binding `params` to `args`, whose parent is `callable_.env`, the function's **defining** environment. Then it evaluates the body there. The parent being the defining environment and not the caller's environment is lexical scoping; if you used the caller's environment instead, you would have dynamic scoping and the project's tests would fail.

### Example 5: Running the interpreter, with a trace

```python
def add(args):
    return ("VInt", args[0][1] + args[1][1])

GLOBAL = Env({
    "+": ("VFunc", Builtin("+", 2, add)),
})

print(eval_expr(Call(Var("+"), [LiteralInt(1), LiteralInt(1)]), GLOBAL))
```

What happens, in order:

1. `eval_expr` sees a `Call`. It evaluates `expr.fn_expr`, which is `Var("+")`.
2. `Var("+")` looks `+` up in `GLOBAL` and finds `("VFunc", Builtin("+", 2, add))`.
3. Back in the `Call` case, each operand is evaluated. `LiteralInt(1)` becomes `("VInt", 1)`. Twice.
4. `apply(("VFunc", Builtin(...)), [("VInt", 1), ("VInt", 1)])` is called.
5. `apply` sees a `Builtin`, checks that two arguments were supplied for `num_args == 2`, and calls `add` on them.
6. `add` unwraps both `VInt`s, adds the Python integers, and wraps the result: `("VInt", 2)`.

Four `eval_expr` calls and one `apply` for a two-character source expression. That multiplication is what the lecture's trace demo makes visible.

Now the third exam question from the slides: **what expressions are evaluated while evaluating `[y + 2]`?** Reason from the `eval` cases. `[y + 2]` is `LiteralList([Call(Var("+"), [Var("y"), LiteralInt(2)])], None)`. The `LiteralList` case evaluates each element, so:

1. `LiteralList([...], None)`, the whole expression.
2. `Call(Var("+"), [Var("y"), LiteralInt(2)])`, its one element.
3. `Var("+")`, the function expression of that call.
4. `Var("y")`, the first operand.
5. `LiteralInt(2)`, the second operand.

Five expressions, in that order, with no `rest` to evaluate. Then one `apply` of the `+` builtin.

### Example 6: The lecture's final demo, `f([1, 2])`

The Gleam function:

```
fn f(s) {
  case s {
    [] -> [3]
    [first, ..rest] -> [first, ..f(rest)]
  }
}
```

In Python, the equivalent logic on the interpreter's value representation:

```python
def f(s):
    if s == []:
        return [3]
    first, rest = s[0], s[1:]
    return [first] + f(rest)

print(f([1, 2]))
```

The recursion: `f([1, 2])` matches the second arm, binding `first` to `1` and `rest` to `[2]`, and must build `[1, ..f([2])]`. So `f([2])` is called, binding `first` to `2` and `rest` to `[]`, and must build `[2, ..f([])]`. So `f([])` is called, matches the first arm, and returns `[3]`. Unwinding: `f([2])` returns `[2, 3]`, then `f([1, 2])` returns `[1, 2, 3]`.

The point of the lecture's demo is not the answer but the volume of interpreter work behind it. Each `f(rest)` is a `Call` expression, so each recursive step involves evaluating `Var("f")`, evaluating `Var("rest")`, calling `apply`, creating a new frame whose parent is `f`'s defining environment, evaluating the `Case`, evaluating its subject `Var("s")`, matching that value against the arms, and evaluating the chosen arm's result expression, which is itself a `LiteralList` with a `rest` that is another `Call`. Printing every expression evaluated, every application with its arguments, and every return makes that visible, and reading such a trace is the skill the exam questions are testing.

Also note what the `[first, ..rest]` pattern does in the `Case` arm: pattern matching both tests the shape of the value and binds names. Matching `VList([VInt(1), VInt(2)])` against `[first, ..rest]` succeeds and adds `first: VInt(1)` and `rest: VList([VInt(2)])` to the new frame.

## Common Pitfalls

**Confusing `Expr` with `Value`.** These are separate types. `LiteralInt(5)` is an expression; `VInt(5)` is a value. `LiteralList([LiteralInt(1), LiteralInt(2)], None)` is an expression; `VList([VInt(1), VInt(2)])` is a value. The exam asks both "how is expression X represented" and "how is value Y represented," and answering one with the other's type is the most common way to lose the point.

**Representing `1 + 1` as `LiteralInt(2)`.** The AST records syntax, not the result. `1 + 1` is `Call(Var("+"), [LiteralInt(1), LiteralInt(1)])`. Evaluation is what produces `VInt(2)`.

**Forgetting that `VList` has no `rest`.** `LiteralList` has `elems` and an optional `rest` because `[1, 2, ..s]` is legal syntax. `VList` has just a list of values, because the `rest` is resolved during evaluation.

**Putting a string in `Call`'s function position.** `Call` holds `fn_expr: Expr`, so `f(5)` has `Var("f")` there, not `"f"`. The function expression is evaluated like any other.

**Giving `apply` an environment.** `apply(f, args)` takes a function value and argument values and nothing else. The environment it needs is already inside the `UserFn`.

**Using the caller's environment as the new frame's parent.** The parent must be `UserFn`'s `env`, the defining environment. Using the caller's environment gives dynamic scoping and breaks lexical scope.

**Evaluating a lambda's body when the lambda is evaluated.** Evaluating `Lambda(params, body)` only constructs a `UserFn`; the `body` is stored as an unevaluated `Expr`. An error inside the body surfaces at application time, not at definition time.

**Evaluating a `let`'s value in the extended environment.** `Let(x, v, body)` evaluates `v` first, in the environment that does not yet bind `x`, then binds, then evaluates `body`.

**Forgetting that the `let` body is the following line.** A sequence of statements becomes nested `Let` expressions. The `Let` variant has three fields even though Gleam source shows only two parts.

**Thinking Gleam can mutate without `@external`.** Mutation is only reachable through external JavaScript functions. Every mutable-state trick in this lecture passes through an `@external` declaration.

**Threading the counter or cache incorrectly.** In the `fib` exercise, `counter` must be passed to **both** recursive calls, and `increment(counter)` must be the first statement so that base cases are counted. Forgetting one recursive call undercounts an entire subtree.

**Expecting the Gleam compiler to check external functions.** `@external` declarations are trusted. If the JavaScript returns the wrong shape, nothing catches it at compile time.

**Mixing up Calculator and Evaluator scope.** Calculator (Homework 8) is integer arithmetic with variables plus anonymous functions. It does not have `case`, `let`, field access, lists, bools, records, or recursion. Those belong to the Animator Evaluator.

**Dropping `Result` handling in `eval`.** Every `eval` and `apply` branch returns `Result(Value, EvalError)`. Errors are values that must be propagated, not exceptions that unwind on their own.

## Likely Exam Points

The lecture explicitly names three question forms to expect. Those come first.

**1. How is a given Gleam expression represented in the evaluator?**

*Question:* How is `1 + 1` represented?

*Answer:* `Call(Var("+"), [LiteralInt(1), LiteralInt(1)])`. The operator is a variable reference in the function position, and the two operands are integer literal expressions. Not `LiteralInt(2)`: the AST records the written syntax.

**2. How is a given Gleam value represented in the evaluator?**

*Question:* How is the value `[1, 2]` represented?

*Answer:* `VList([VInt(1), VInt(2)])`. A `VList` holding two `VInt` values. No `rest` field, because `VList` does not have one, and not `LiteralList(...)`, which is the expression type.

**3. What expressions are evaluated while evaluating a given expression?**

*Question:* What expressions are evaluated while evaluating `[y + 2]`?

*Answer:* `LiteralList([Call(Var("+"), [Var("y"), LiteralInt(2)])], None)` (the whole list expression), then its single element `Call(Var("+"), [Var("y"), LiteralInt(2)])`, then that call's function expression `Var("+")`, then its operands `Var("y")` and `LiteralInt(2)`. Five expressions, followed by one `apply` of the `+` builtin.

**4. Distinguishing `Builtin` from `UserFn`.**

*Question:* What is the difference between the two `Callable` variants, and what does each store?

*Answer:* `Builtin(name, num_args, run)` stores a real Gleam function in `run`, of type `fn(List(Value)) -> Result(Value, EvalError)`; applying it calls that Gleam function directly on the argument values, with no `Expr` and no new frame. `UserFn(name, params, body, env)` stores an unevaluated `body: Expr` and the `env` in which the function was defined; applying it makes a new local frame binding `params` to `args` with `env` as parent, then evaluates `body` in that frame.

**5. Why `UserFn` stores an environment.**

*Question:* Why does `UserFn` have an `env` field, and whose environment is it?

*Answer:* It is the environment in which the function was defined. Storing it lets `apply` make the new call frame's parent be the defining environment rather than the calling environment, which is what gives the language lexical scoping. Without it, `apply` would have no way to resolve names that are free in the body.

**6. Completing an `eval` case.**

*Question:* Fill in the `Let(x, v, body)` case of `eval`.

*Answer:* Evaluate `v` in the current `env`; if that succeeds with value `val`, build a new frame binding `x` to `val` with `env` as parent; evaluate `body` in that new environment and return its result. Both steps return `Result`, so the error from either propagates.

**7. Order of operations in `Call`.**

*Question:* When evaluating `Call(fn_expr, operands)`, what is evaluated and in what order, and what is passed to `apply`?

*Answer:* `fn_expr` is evaluated first to get a function value, then each expression in `operands` is evaluated to get argument values. Then `apply` is called with the function value and the list of argument values. All arguments are fully evaluated before `apply` runs.

**8. Mutable state through external functions.**

*Question:* Gleam has no mutation. Explain how the counter in the `fib` exercise changes value, and why `fib`'s return type is still `Int`.

*Answer:* `new_counter`, `increment`, and `count` are declared `@external(javascript, ...)`, so their bodies are JavaScript, which permits mutation. `new_counter` returns a JavaScript object `{ calls: 0 }`, held in Gleam as the constructorless type `Counter`. `increment` mutates `counter.calls` in place, so every call that receives the same object sees the updated count. `fib`'s return type stays `Int` because the counting happens entirely through the side effect on the shared object, not through the return value.

**9. Filling in the `fib`-with-counter exercise.**

*Question:* Complete `pub fn fib(n: Int, counter: Counter) -> ___` and the `main` that echoes the number of calls in `fib(42)`.

*Answer:*
```
pub fn fib(n: Int, counter: Counter) -> Int {
  increment(counter)
  case n {
    0 | 1 -> n
    _ -> fib(n - 2, counter) + fib(n - 1, counter)
  }
}

pub fn main() {
  let counter = new_counter()
  fib(42, counter)
  echo count(counter)
}
```
The return type is `Int`; `increment(counter)` comes first so every entry is counted; both recursive calls pass `counter`; the base arm returns `n`.

**10. Calculator versus Evaluator scope.**

*Question:* Name two capabilities the Animator project's Evaluator has that the Calculator does not.

*Answer:* Any two of: environments as sequences of frames; recursive user-defined functions; built-ins beyond `+`, `-`, `*`, `/`; `case`, `let`, and field-access expressions; and value types beyond integers (bools, lists, functions, records).

**11. Reading a `LiteralList` with a `rest`.**

*Question:* How is `[1, 2, ..s]` represented, and what happens when it is evaluated?

*Answer:* `LiteralList(elems: [LiteralInt(1), LiteralInt(2)], rest: Some(Var("s")))`. Evaluating it evaluates `LiteralInt(1)` and `LiteralInt(2)` to `VInt(1)` and `VInt(2)`, evaluates `Var("s")` to a value (which must be a `VList`), and combines them into a single `VList` whose first two elements are those `VInt`s followed by the elements of `s`.

**12. When a lambda's body is evaluated.**

*Question:* Does evaluating `Lambda(["x"], body)` evaluate `body`? What if `body` references an unbound name?

*Answer:* No. The `Lambda` case of `eval` constructs `UserFn("lambda", ["x"], body, env)`, storing `body` unevaluated and capturing the current `env`. An unbound name in `body` causes no error at this point; the `EvalError` only arises when the function is applied and `eval(body, local)` runs.

**13. Field access.**

*Question:* What does `eval` do with `FieldAccess(record, field)`, and when does it error?

*Answer:* It evaluates the `record` expression, checks that the resulting value is a `VData`, and retrieves the named field from it. It errors if the record expression fails to evaluate, if the resulting value is not a `VData`, or if that `VData` has no such field.

## Summary

- Gleam has no mutation, but it compiles to JavaScript, and `@external(javascript, "module", "name")` declares a Gleam function whose body is JavaScript. JavaScript can mutate, so external functions give Gleam access to mutable objects.
- A type declared with no constructors (`pub type Cache`) is a Gleam name for a foreign object: it can be held and passed but not built or inspected from Gleam.
- JavaScript external code imports `Ok` and `Error` from the compiler-emitted `gleam.mjs` to return genuine Gleam `Result` values. The compiler trusts the declared signature and cannot verify it.
- The cache example (`new_cache`, `lookup`, `store`) and the counter example (`new_counter`, `increment`, `count`) are the same pattern: one shared mutable object that many calls reach through.
- In the `fib` exercise, `fib(n: Int, counter: Counter) -> Int` calls `increment(counter)` as its first statement and threads `counter` through both recursive calls; `main` echoes `count(counter)` after `fib(42, counter)`.
- Two interpreters this unit: **Calculator** (Homework 8), with integer arithmetic over variables and anonymous functions, and **Evaluator** (Animator project), adding frame-sequence environments, recursion, more built-ins, `case`/`let`/field-access expressions, and bool, list, function, and record values.
- Parsing converts program text into an abstract syntax tree; an expression's AST is represented by an `Expr` value.
- `Expr` variants: `Var`, `LiteralBool`, `LiteralInt`, `LiteralList(elems, rest)`, `Call(fn_expr, operands)`, `Lambda(params, body)`, `Case(subject, arms)`, `Let(name, value, body)`, `FieldAccess(record, field)`. A `Let`'s body is the expression on the following line, so statement sequences nest.
- `Value` variants: `VBool`, `VInt`, `VList(List(Value))`, `VData(tag, fields)`, `VFunc(Callable)`. `VList` has no `rest`; the splicing happened during evaluation.
- `Callable` has two forms: `Builtin(name, num_args, run)`, which delegates to a real Gleam function, and `UserFn(name, params, body, env)`, which stores an unevaluated body plus its defining environment.
- `eval(expr: Expr, env: Env) -> Result(Value, EvalError)` dispatches on expression type. `apply(f: Value, args: List(Value)) -> Result(Value, EvalError)` takes no environment, because the function value carries its own.
- `Call` evaluates its function expression and all operands, then calls `apply`; `apply` on a `UserFn` makes a local frame and calls `eval` on the body. That mutual recursion is the interpreter.
- The new frame's parent is the function's defining environment, not the caller's: that is lexical scoping at the implementation level.
- Expect exam questions of exactly three shapes: how an expression is represented, how a value is represented, and which expressions get evaluated during an evaluation.
- Instrumenting the interpreter to print each expression evaluated, each application with arguments, and each return makes the full recursion visible, as in the `f([1, 2])` demo.
