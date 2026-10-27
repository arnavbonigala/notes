<!-- Mon, Oct 26, 2026 | sources: slides + code (no transcript available) -->
# Lecture 26: Midterm 3 Review

This lecture is a review session for Midterm 3, built entirely around worked problems rather than new material. It walks through six problem areas: (1) list mutation and aliasing, using a buggy `prefixes` function whose fix is a single `list(...)` copy; (2) a general "recursion recipe" for attacking recursive problems; (3) tree recursion, via the Spring 2023 Midterm 2 `exclude` problem, which only 20% of students solved; (4) string manipulation with slicing and f-strings (`display`); (5) a combined dictionary/tree-recursion enumeration problem (`buy`) that uses a nested helper with accumulator parameters; (6) object-oriented programming, covering `Mic`/`Speaker` (one object mutating another object's attribute), attribute lookup with inheritance (`A`/`B`), and class attributes shared across instances (`Transaction`); (7) generators, with `yield from` on a tree and the laziness of a partially consumed generator; and (8) building a list recursively by mutation (`fibs_rec`). The unifying themes are mutation versus copying, where state lives (frames, instances, classes, generator suspensions), and a disciplined process for writing recursive code.

## Key Concepts

### 1. Aliasing: one list object, many names

A list value in Python is an object on the heap. Assignment, `append`, and passing to a function all copy *references*, never the list itself. So if you append the same list object to a result list three times, the result holds three pointers to one object, and every mutation of that object is visible through all three.

This is exactly the bug in `prefixes`. In box-and-pointer terms: `so_far` points to a single list box. `result` is a second list box whose three slots all point at that same `so_far` box. When the loop finishes, that one box contains `[1, 2, 3]`, so printing `result` shows `[[1, 2, 3], [1, 2, 3], [1, 2, 3]]`.

The fix is to take a **snapshot** at each step: `list(so_far)` constructs a *new* list with the same elements, so each slot of `result` points to a distinct box frozen at that moment. (Equivalently, `so_far[:]` or `so_far + []` would also copy. (extra context))

The same idea reappears in `Transaction.__init__` with `self.prior = list(self.log)`: because the shared `log` keeps growing, you must copy it to record its state at construction time.

### 2. The recursion recipe

The lecture gives an explicit procedure for writing a recursive function:

- Write down an example input.
- Ask: what **small initial choice** can I make? (Which one decision reduces the problem?)
- Ask: what **recursive call** do I make for each option?
  - Write down the recursive call(s) for your example input.
  - Write down **what each of those calls returns** for your example input.

The crucial and most-skipped step is the last one. If you can state in concrete terms what `exclude(branch, x)` returns for a specific branch of a specific tree, the remaining code almost writes itself: you only need to decide what to *do* with those return values. The `exclude` slides are annotated with exactly these two questions ("What will the recursive call on each branch return?" and "What should we do with those return values?").

### 3. Tree recursion and the Tree ADT

The lecture's `Tree` is a dataclass with a `label` and a list of `branches` (defaulting to empty via `field(default_factory=list)`), a `__str__` that delegates to `format_tree` (each branch indented two spaces below its label), and a helper `is_leaf(t)` returning `not t.branches`.

The standard tree-recursion shape is: recurse on every branch, then combine the results with the current label. In `exclude`, the combination step is unusual and is what makes the problem hard: a returned branch might itself be excluded (its root label equals `x`), in which case its *children* get promoted to take its place. That is why `bs.extend(b.branches)` is used instead of `bs.append(...)`.

Note the ordering: the recursive call happens **first**, so by the time you inspect `b`, all of `b`'s descendants have already been cleaned. The root of the returned subtree is the only node that might still be labeled `x`, since `exclude` never removes the root of the tree it is given. That invariant ("non-root nodes have labels not equal to `x`") is what makes `extend` safe.

### 4. Enumerating possibilities with a nested recursive helper

`buy` must print every combination of counts that sums exactly to `total_amount`, with at least one of each fruit. The design pattern is a nested helper function with **accumulator parameters**:

- `fruits`: which fruits remain to be decided (shrinks by one per level)
- `amount`: how much money is left (shrinks by `price * k`)
- `cart`: the string built so far (grows by `display(fruit, k)`)

The "small initial choice" is: how many of `fruits[0]` do I buy? Each value of `k` is one branch of the recursion, hence the `for` loop around the recursive call. The success base case is "nothing left to buy and no money left" (`fruits == [] and amount == 0`), so failures simply fall off the end and print nothing. This is a clean example of printing at the base case rather than returning and combining values; the helper returns `None`.

The dictionary `prices` is used purely as a lookup table (`prices[fruit]`), and fruits in `prices` that are not in `fruits_to_buy` are never bought.

### 5. Strings: slicing and f-strings

`display` combines three small skills: `fruit[-1]` to inspect the last character, `fruit[:-1]` to drop the last character (making a plural singular), and an f-string `f'<{count} {fruit}>'` to interpolate values into a template. The `assert count >= 1 and fruit[-1] == 's'` documents the function's preconditions. Note that strings are immutable, so `fruit = fruit[:-1]` rebinds the local name to a brand-new string and never affects the caller.

### 6. Objects: where attributes live and who mutates them

Three distinct OOP ideas appear:

**Attribute lookup order.** `self.x` looks in the instance's own attribute dictionary first; if not found, it searches the class, then its base classes in order. This single rule explains the `A`/`B` example completely.

**One object mutating another object's attributes.** `Speaker.connect(self, m, location)` executes `m.speakers[location] = self`. The speaker reaches into the mic's `speakers` dictionary and inserts itself. Nothing is stored on the speaker about the mic. This is a common exam pattern: the method's own `self` ends up stored inside a *different* object.

**Class attributes are shared.** `Transaction.log = []` is one list, created once when the class statement executes, shared by every instance. `self.log.append(self)` *mutates* that shared list (good). `self.log = ...` would instead create a new *instance* attribute that shadows the class attribute for that one object (usually a bug). Access is equivalent through `self.log`, `type(self).log`, or `Transaction.log`.

Also note that `Speaker(str.lower)` passes a function (`str.lower`, an unbound string method usable as a one-argument function) as a value, and `Speaker(lambda x: x)` passes the identity transform, so the Front speaker repeats the signal unchanged.

### 7. Generators and laziness

A function containing `yield` is a **generator function**: calling it runs no body code and immediately returns a generator object (hence the `<generator object all_elements at 0x...>` repr). Each `next()` resumes the suspended frame until the next `yield`, then suspends again. `yield from <iterable>` delegates: it yields every element of that iterable before continuing, which is what makes recursive tree traversal so compact.

Two consequences dominate exam questions:

- **Order matters.** `all_elements` does `yield from` on all branches *before* `yield t.label`, producing a post-order (bottom-up, left-to-right) traversal.
- **Generators are consumed.** Once you call `next(a)`, that element is gone forever. A later `list(a)` yields only the *remaining* elements. The Waldo puzzle depends entirely on this.

### 8. Building a list recursively by mutation

`fibs_rec` shows that recursion and mutation compose: the inner `f` appends one element per call and recurses, with the list `s` living in the enclosing `fibs_rec` frame. No `nonlocal` is needed because `s` is only ever *mutated*, never rebound. The "base case" is implicit: when `i >= n`, the `if` body is skipped and the call returns `None`, ending the chain. Each call can read the two most recently appended values as `s[-2]` and `s[-1]`.

## Definitions

- **Aliasing**: the situation in which two or more names (or container slots) refer to the same mutable object, so a mutation through one is visible through all.
- **Mutation**: a change to the contents of an existing object (for example `lst.append(x)`, `d[k] = v`, `obj.attr = v`) as opposed to creating a new object.
- **Copy (shallow)**: a new container with the same element references, produced by `list(s)` or `s[:]`; the elements themselves are not copied.
- **Prefix of a list**: a list consisting of the first `k` elements of the original, for some `k`. `prefixes([1, 2, 3])` returns `[[1], [1, 2], [1, 2, 3]]` (the lecture's version omits the empty prefix and includes the whole list).
- **Accumulator parameter**: a parameter that carries partial results (such as `cart` or `amount` in `buy`) down the recursion so the base case can report a complete answer.
- **Tree (the lecture's ADT)**: an object with a `label` and a list of `branches`, each branch itself being a `Tree`. Branches default to the empty list.
- **Node / label / root / branch / leaf**: a node is one `Tree` object; its `label` is the value it holds; the `root` is the top node of a tree; a `branch` is a subtree directly below a node; a `leaf` is a node with no branches (`is_leaf(t)` returns `not t.branches`).
- **`field(default_factory=list)`**: dataclass syntax specifying that each new instance gets a fresh empty list as its default, rather than sharing one mutable default.
- **`list.append(x)` vs `list.extend(xs)`**: `append` adds `x` as a single new element; `extend` adds each element of the iterable `xs` individually.
- **`map(f, iterable)`**: a lazy iterator applying `f` to each element; it is not a list and can be iterated only once.
- **f-string**: a string literal prefixed with `f` in which `{expression}` is replaced by the value of the expression, for example `f'<{count} {fruit}>'`.
- **Instance attribute**: an attribute stored on one object, found first during attribute lookup, created by assignment such as `self.z = 'A'`.
- **Class attribute**: an attribute stored on the class itself, shared by all instances, with exactly one value regardless of how many instances exist.
- **Inheritance (`class B(A)`)**: `B` is a subclass of `A`; attributes not found on a `B` instance or on `B` are looked up on `A`.
- **Bound method**: an attribute lookup of a function on an instance, which produces a callable with `self` already supplied (`self.speakers[k].repeat` is bound to that speaker).
- **Generator function**: a function whose body contains `yield`; calling it returns a generator object without executing the body.
- **Generator object**: an iterator holding a suspended frame; `next()` resumes it until the next `yield`, and `StopIteration` is raised when the function returns.
- **`yield from <iterable>`**: yields every element of the iterable in order, as if by a loop of `yield` statements; used for recursive delegation.
- **Lazy evaluation**: computing values only when demanded, as generators, `map`, and `range` do.

## Worked Examples

### Example 1: `prefixes` and the aliasing bug

```python
def prefixes(s):
    """Return a list of all of the list prefixes of s, including s.

    >>> prefixes([1, 2, 3])
    [[1], [1, 2], [1, 2, 3]]
    """
    so_far = []
    result = []
    for x in s:
        so_far.append(x)
        result.append(so_far)
        # Fixed version: result.append(list(so_far))
    return result
```

Step by step on `prefixes([1, 2, 3])`, reasoning about boxes and pointers:

1. `so_far` is bound to a new empty list box, call it **L1**. `result` is bound to a different new empty list box, **L2**.
2. `x = 1`: `so_far.append(1)` mutates **L1** to `[1]`. `result.append(so_far)` stores a *pointer to L1* in **L2**. Now `L2 = [->L1]`.
3. `x = 2`: `so_far.append(2)` mutates **L1** to `[1, 2]`. `result.append(so_far)` stores a second pointer to **L1**. Now `L2 = [->L1, ->L1]`.
4. `x = 3`: **L1** becomes `[1, 2, 3]`, and `L2 = [->L1, ->L1, ->L1]`.
5. Printing `result` follows all three pointers to the same box, so the output is `[[1, 2, 3], [1, 2, 3], [1, 2, 3]]`, not the documented answer.

The fix shown on the slide inserts `list(so_far)`: `result.append(list(so_far))`. Now step 2 stores a pointer to a *fresh* box containing `[1]`, step 3 stores a pointer to a fresh box containing `[1, 2]`, and step 4 one containing `[1, 2, 3]`. Three distinct boxes, none of which is `so_far` itself, so later mutations of `so_far` cannot change them, and the result is `[[1], [1, 2], [1, 2, 3]]`.

### Example 2: `exclude` (Spring 2023 Midterm 2, Question 4a)

Problem statement: implement `exclude`, which takes a `Tree t` and a value `x`. It returns a `Tree` containing the root node of `t` as well as each non-root node of `t` with a label not equal to `x`. The parent of a node in the result is its nearest ancestor node that is not excluded.

```python
def exclude(t, x):
    """Return a tree with the non-root nodes of tree t labeled anything but x.

    >>> t = Tree(1, [Tree(2, [Tree(2), Tree(3), Tree(4)]), Tree(5, [Tree(1)])])
    >>> print(exclude(t, 2))
    1
      3
      4
      5
        1
    """
    filtered_branches = map(lambda y: exclude(y, x), t.branches)
    bs = []
    for b in filtered_branches:
        if b.label == x:
            bs.extend(b.branches)
        else:
            bs.append(b)
    return Tree(t.label, bs)
```

Applying the recursion recipe to the doctest input (`x = 2`):

- **Example input**: the tree with root `1`, branches `Tree(2, [Tree(2), Tree(3), Tree(4)])` and `Tree(5, [Tree(1)])`.
- **Recursive calls**: `exclude(b, 2)` for each branch `b`, built lazily by `map(lambda y: exclude(y, x), t.branches)`.
- **What each call returns**: a tree with the *same root label as its input* (roots are never excluded) and all non-root `2`s removed.

Trace, innermost first:

1. `exclude(Tree(2), 2)` (a leaf): `t.branches` is empty, so the loop body never runs and `bs` stays `[]`. Returns `Tree(2, [])`. Note the label `2` survives because it is the root of *this* call.
2. `exclude(Tree(3), 2)` returns `Tree(3, [])`; `exclude(Tree(4), 2)` returns `Tree(4, [])`.
3. `exclude(Tree(2, [Tree(2), Tree(3), Tree(4)]), 2)`: the three recursive results are `Tree(2)`, `Tree(3)`, `Tree(4)`. The loop sees `b.label == 2` for the first, so it extends `bs` with `b.branches`, which is `[]`, contributing nothing. The other two are appended. Returns `Tree(2, [Tree(3), Tree(4)])`.
4. `exclude(Tree(5, [Tree(1)]), 2)`: the inner call returns `Tree(1)`, whose label is not `2`, so it is appended. Returns `Tree(5, [Tree(1)])`.
5. Top level: `bs` starts empty. First branch result is `Tree(2, [Tree(3), Tree(4)])`, whose label *is* `2`, so `bs.extend([Tree(3), Tree(4)])` makes `bs = [Tree(3), Tree(4)]`. The nodes `3` and `4` have now been re-parented to their nearest non-excluded ancestor, the root `1`, which is exactly what the spec demands. Second branch result `Tree(5, [Tree(1)])` is appended. Returns `Tree(1, [Tree(3), Tree(4), Tree(5, [Tree(1)])])`.
6. `print` via `format_tree` indents each branch two spaces under its label, producing the expected output.

Why `extend` and not `append`: appending would keep the excluded node as a placeholder in the tree. Extending splices its (already-cleaned) children directly into the parent's branch list, which implements "the parent of a node in the result is its nearest ancestor that is not excluded."

The slide reports that in Spring 2023 only **20% of students got this question right**, with the individual blanks at roughly **37%**, **30%** (and that blank was one of only four plausible options), and **24%**. The lesson the lecture draws: the hard part is not the syntax, it is answering "what does the recursive call return?" before writing the combination step.

### Example 3: `display` (string warm-up)

```python
def display(fruit: str, count: int) -> str:
    """Display a count of a fruit in angle brackets.

    >>> display('apples', 3)
    '<3 apples>'
    >>> display('apples', 1)
    '<1 apple>'
    >>> display('kiwis', 12)
    '<12 kiwis>'
    >>> print(display('apples', 3) + display('kiwis', 3))
    <3 apples><3 kiwis>
    """
    assert count >= 1 and fruit[-1] == 's'
    if count == 1:
        fruit = fruit[:-1]
    return f'<{count} {fruit}>'
```

Step by step on `display('apples', 1)`:

1. The assertion checks `count >= 1` (true) and that the name is plural, `fruit[-1] == 's'` (true for `'apples'`).
2. `count == 1`, so `fruit = fruit[:-1]` rebinds the local `fruit` to the new string `'apple'`. The caller's string is unaffected because strings are immutable and only the local name was rebound.
3. The f-string substitutes `count` (1) and `fruit` (`'apple'`), returning `'<1 apple>'`.

On `display('kiwis', 12)` the `if` is skipped and the result is `'<12 kiwis>'`. The lecture code file writes the same logic with two returns (`return f'<{count} {fruit[:-1]}>'` inside the `if`); either shape is fine, and the slide's single-return version is the one that fits the two blanks on the exam.

Note the last doctest: `display` *returns* a string rather than printing, which is what lets `buy` concatenate the pieces into a cart.

### Example 4: `buy` (dictionary plus tree recursion)

```python
def buy(fruits_to_buy: list[str], prices: dict[str, int], total_amount: int) -> None:
    """Print ways to buy some of each fruit so that the sum of prices is amount.

    >>> prices = {'oranges': 4, 'apples': 3, 'bananas': 2, 'kiwis': 9}
    >>> buy(['apples', 'oranges', 'bananas'], prices, 12)
    <2 apples><1 orange><1 banana>
    >>> buy(['apples', 'oranges', 'bananas'], prices, 16)
    <2 apples><1 orange><3 bananas>
    <2 apples><2 oranges><1 banana>
    >>> buy(['apples', 'kiwis'], prices, 36)
    <3 apples><3 kiwis>
    <6 apples><2 kiwis>
    <9 apples><1 kiwi>
    """
    def add(fruits: list[str], amount: int, cart: str) -> None:
        if fruits == [] and amount == 0:
            print(cart)
        elif fruits and amount > 0:
            fruit = fruits[0]
            price = prices[fruit]
            for k in range(1, amount // price + 1):
                add(fruits[1:], amount - price * k, cart + display(fruit, k))
    add(fruits_to_buy, total_amount, '')
```

How each blank is justified:

- `price = prices[fruit]`: a dictionary lookup of the cost of one unit of the current fruit.
- `range(1, amount // price + 1)`: start at `1` because **at least one** of each fruit must be bought; stop at `amount // price` inclusive because buying more than that would overshoot the remaining money. `+ 1` is needed since `range` excludes its endpoint.
- Recursive call `add(fruits[1:], amount - price * k, cart + display(fruit, k))`: drop the fruit we just decided on, subtract what we spent, and extend the cart string with the formatted entry.

Trace of `buy(['apples', 'oranges', 'bananas'], prices, 16)`:

1. `add(['apples', 'oranges', 'bananas'], 16, '')`. `fruit = 'apples'`, `price = 3`, so `k` ranges over `1..5` (since `16 // 3 == 5`).
2. `k = 1`: recurse with `(['oranges', 'bananas'], 13, '<1 apple>')`. Oranges cost 4, so `k` ranges `1..3`, leaving 9, 5, or 1 for bananas. Bananas cost 2, and no subcase reaches exactly `amount == 0` with `fruits == []` (9, 5, and 1 are all odd), so nothing prints.
3. `k = 2`: recurse with `(['oranges', 'bananas'], 10, '<2 apples>')`. Oranges `k` ranges `1..2`:
   - 1 orange leaves 6 for bananas; `k` ranges `1..3`, and `k = 3` leaves `6 - 6 = 0` with `fruits == []`, so the base case prints `<2 apples><1 orange><3 bananas>`.
   - 2 oranges leave 2 for bananas; `k = 1` leaves 0, printing `<2 apples><2 oranges><1 banana>`.
4. `k = 3`: 7 left; 1 orange leaves 3, 1 banana leaves 1, never 0. Nothing prints.
5. `k = 4`: 4 left; 1 orange leaves exactly 0, but `fruits` is still `['bananas']`, so the first condition fails (not `fruits == []`) and the second fails too (`amount > 0` is false). Nothing prints, correctly rejecting a cart with zero bananas.
6. `k = 5`: 1 left; `1 // 4 == 0`, so `range(1, 1)` is empty and the orange loop body never runs.

The printed order matches the doctest because the loops run in increasing `k` with apples decided first. Also note `kiwis` is in `prices` but not in `fruits_to_buy`, so it is never considered: the list, not the dictionary, controls what gets bought.

### Example 5: `Mic` and `Speaker` (Spring 2024 Midterm)

```python
class Mic:
    """A microphone connected to speakers.

    >>> m = Mic()  # Front is connected automatically
    >>> m.sing('Is this thing on?')
    Front - Is this thing on?
    >>> Speaker(str.lower).connect(m, 'Left Side')
    >>> m.sing("You BELONG with me.")
    Front - You BELONG with me.
    Left Side - you belong with me.
    """
    def __init__(self):
        self.speakers = {'Front': Speaker(lambda x: x)}

    def sing(self, lyrics: str):
        for k in self.speakers.keys():
            print(k, '-', self.speakers[k].repeat(lyrics))

class Speaker:
    def __init__(self, transform):
        self.transform = transform

    def connect(self, m: Mic, location: str):
        m.speakers[location] = self

    def repeat(self, s: str) -> str:
        return self.transform(s)
```

Step by step:

1. `m = Mic()` runs `__init__`, creating a dictionary with one entry: key `'Front'`, value a new `Speaker` whose `transform` is `lambda x: x`. That identity lambda is what makes the Front speaker "repeat the exact same signal it receives."
2. `m.sing('Is this thing on?')` iterates the keys of `m.speakers` (just `'Front'`), looks the speaker up with `self.speakers[k]`, and calls its `repeat` method. `repeat` returns `self.transform(s)`, which is the unchanged string. `print('Front', '-', 'Is this thing on?')` inserts spaces between arguments, giving `Front - Is this thing on?`.
3. `Speaker(str.lower)` builds a speaker whose transform is the built-in string method used as a plain one-argument function: `str.lower('ABC')` returns `'abc'`. The expression `Speaker(str.lower).connect(m, 'Left Side')` constructs and connects in one line; the only lasting reference to that speaker is the one stored inside `m.speakers`.
4. Inside `connect`, `self` is the new speaker and `m` is the mic. `m.speakers[location] = self` mutates the *mic's* dictionary, adding key `'Left Side'`. `connect` returns `None`, which is why the doctest line shows no output.
5. `m.sing("You BELONG with me.")` now iterates two keys. Dictionaries preserve insertion order, so `'Front'` prints first with the text unchanged, then `'Left Side'` prints with `str.lower` applied: `you belong with me.`

The exam-relevant moves here: store objects as dictionary values keyed by a string, store a *function* as an instance attribute and call it later, and have a method write into another object's attribute. (extra context: `Mic.__init__` mentions `Speaker` before the `Speaker` class statement appears in the file, which is fine because the name is only looked up when `__init__` actually runs.)

### Example 6: Three attributes (attribute lookup with inheritance)

```python
class A:
    x, y, z = 0, 1, 2

    def f(self):
        return [self.x, self.y, self.z]

class B(A):
    """What would Python Do?

    >>> A().f()
    [0, 1, 2]
    >>> B().f()
    [6, 1, 'A']
    """
    x = 6
    def __init__(self):
        self.z = 'A'
```

The lecture's diagram shows: the `A` class holds `x: 0`, `y: 1`, `z: 2`; the `B` class holds `x: 6`; an `A` instance holds nothing of its own; a `B` instance holds `z: 'A'`.

- `A().f()`: the `A` instance has no instance attributes, so each of `self.x`, `self.y`, `self.z` is found on class `A`, giving `[0, 1, 2]`.
- `B().f()`: constructing a `B` runs `B.__init__`, which sets the instance attribute `z = 'A'`. `f` is not defined on `B`, so it is inherited from `A`, and `self` is still the `B` instance.
  - `self.x`: not an instance attribute; found on `B` as `6`. `B`'s value shadows `A`'s `0` because `B` is searched before `A`.
  - `self.y`: not on the instance, not on `B`; found on `A` as `1`.
  - `self.z`: found on the *instance* as `'A'`, which shadows both `A.z` and any class value.
  - Result: `[6, 1, 'A']`.

The general rule to memorize: **instance, then class, then base classes**, and the first match wins.

### Example 7: `Transaction` (class attributes)

A class attribute can be accessed from either an instance or its class. There is only one value for a class attribute, regardless of how many instances exist.

```python
class Transaction:
    """A logged transaction.

    >>> s = [20, -3, -4]
    >>> ts = [Transaction(x) for x in s]
    >>> ts[1].balance()
    17
    >>> ts[2].balance()
    13
    """
    log = []  # always bound to some Transaction instance

    def __init__(self, amount):
        self.amount = amount
        self.prior = list(self.log)  # a list of Transactions
        self.log.append(self)

    def balance(self):
        """The sum of amounts for this transaction and all prior transactions"""
        return self.amount + sum([t.amount for t in self.prior])
```

Step by step on the doctest:

1. `log = []` executes once, when the class statement runs, creating a single list owned by the class. The lecture's diagram shows the `Transaction` class pointing at one list object, with every instance's `prior` pointing at its own separate list.
2. `Transaction(20)`: `self.amount = 20`; `self.prior = list(self.log)` copies the currently empty log, so `prior` is a fresh empty list (the slide annotates this as "empty list"); then `self.log.append(self)` mutates the shared class list to hold this first transaction.
3. `Transaction(-3)`: `prior` is a copy of `[t0]`; then the log grows to `[t0, t1]`.
4. `Transaction(-4)`: `prior` is a copy of `[t0, t1]`; the log grows to `[t0, t1, t2]`.
5. `ts[1].balance()` is `-3 + sum([20])`, which is `17`.
6. `ts[2].balance()` is `-4 + sum([20, -3])`, which is `-4 + 17`, that is `13`.

Two critical details:

- The copy in `list(self.log)` is required. Without it, every `prior` would alias the one growing log, and each transaction's balance would include transactions created *after* it (and `t0.prior` would even contain `t0` itself). The slide notes the equivalent forms `list(type(self).log)` and `list(Transaction.log)`.
- `self.log.append(self)` mutates the shared list and therefore updates the class attribute for everyone. Writing `self.log = self.log + [self]` instead would create a per-instance attribute and the shared log would never grow.

### Example 8: Generator "Where's Waldo"

```python
def all_elements(t: Tree):
    for b in t.branches:
        yield from all_elements(b)
    yield t.label
```

```
>>> t = Tree('o', [Tree('a'), Tree('W', [Tree('o'), Tree('a', [Tree('d'), Tree('l')])])])
>>> a = all_elements(t)
>>> a
<generator object all_elements at 0x104e4ddd0>
>>> next(a)
'a'
>>> list(reversed(list(a)[:-1]))
['W', 'a', 'l', 'd', 'o']
```

First, the tree shape (from the slide diagram): root `'o'` has branches `'a'` (a leaf) and `'W'`; `'W'` has branches `'o'` (a leaf) and `'a'`; that inner `'a'` has leaves `'d'` and `'l'`.

Because each label is yielded *after* all of its branches, the traversal is bottom-up and left-to-right. The full sequence of yields is:

1. `'a'` (the first leaf)
2. `'o'` (leaf under `'W'`)
3. `'d'`, then `'l'` (leaves under the inner `'a'`)
4. `'a'` (the inner node, after its children)
5. `'W'` (after both of its branches)
6. `'o'` (the root, last of all)

So the generator produces `'a', 'o', 'd', 'l', 'a', 'W', 'o'`.

Now the puzzle, step by step:

1. `a = all_elements(t)` runs **no** body code; it just creates the generator object (hence the repr line).
2. `next(a)` resumes it until the first `yield`, returning `'a'`. That element is now permanently consumed.
3. `list(a)` drains the remaining six elements: `['o', 'd', 'l', 'a', 'W', 'o']`.
4. `[:-1]` drops the final element (the root `'o'`), giving `['o', 'd', 'l', 'a', 'W']`.
5. `reversed(...)` on that list produces `'W', 'a', 'l', 'd', 'o'`, and `list(...)` materializes it: `['W', 'a', 'l', 'd', 'o']`, spelling "Waldo."

The slicing and `reversed` are applied to a *list*, not to `a` directly, because generators support neither slicing nor `reversed`.

### Example 9: `fibs_rec` (recursive list building by mutation)

List mutations can be used to build up a list incrementally, even using recursion.

```python
def fibs_rec(n):
    """Return a list of the first n fibonacci numbers.

    >>> fibs_rec(10)
    [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
    """
    s = []
    def f(i):
        if i < n:
            if i < 2:
                s.append(i)
            else:
                s.append(s[-2] + s[-1])
            f(i + 1)
    f(0)
    return s
```

The three blanks on the slide are `if i < n:`, the recursive call `f(i + 1)`, and the initial call `f(0)`.

Step by step for `n = 10`:

1. `s = []` in the `fibs_rec` frame. `f` is defined with `fibs_rec`'s frame as its parent, so `f`'s body can see both `s` and `n`.
2. `f(0)`: `0 < 10`, and `0 < 2`, so `s.append(0)`. Then `f(1)`.
3. `f(1)`: `1 < 2`, so `s.append(1)`. Now `s = [0, 1]`. Then `f(2)`.
4. `f(2)`: `2 >= 2`, so `s.append(s[-2] + s[-1])`, which is `0 + 1 == 1`. Now `s = [0, 1, 1]`. Then `f(3)`.
5. The pattern continues: each call appends the sum of the last two elements already in `s` and advances `i`. Successive appends are `2, 3, 5, 8, 13, 21, 34`.
6. `f(10)`: `10 < 10` is false, the body is skipped, and the call returns `None`. That implicit do-nothing case terminates the recursion.
7. All ten calls return in turn, and `fibs_rec` returns `s`, which is `[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]`.

Two points worth internalizing: `f` does not need `nonlocal s` because it only mutates the list that `s` already points to (it never rebinds the name `s`), and `f` returns nothing at all, since the entire result is communicated through the shared list. Also note `fibs_rec(0)` returns `[]`, and `fibs_rec(1)` returns `[0]`, which the guard `i < n` handles automatically.

## Common Pitfalls

- **Appending the same list repeatedly.** The `prefixes` bug. Any time you accumulate "snapshots" of a list you are mutating, copy with `list(...)` or a slice. The symptom is a result full of identical sublists.
- **Confusing copying with mutating.** `list(s)` returns a new list and does not change `s`; `s.append(x)` changes `s` and returns `None`. Writing `s = s.append(x)` sets `s` to `None`.
- **`append` where `extend` is needed (or vice versa).** In `exclude`, `bs.append(b.branches)` would nest a list inside the branches list, breaking the Tree ADT. `bs.extend(b)` would be wrong too, since a `Tree` is not iterable here.
- **Checking the label before recursing in `exclude`.** If you test `b.label == x` on the *original* branch rather than on the result of `exclude(b, x)`, nested excluded nodes deeper in the tree survive. Recurse first, then inspect.
- **Forgetting that the root is never excluded.** `exclude` keeps `t.label` unconditionally; a top-level call whose root label equals `x` still returns a tree with that root. Removal happens only in the parent's loop.
- **`map` returns a one-shot iterator, not a list.** `filtered_branches` can be looped over exactly once, and `len(filtered_branches)` or indexing it is an error.
- **Off-by-one in `range`.** `range(1, amount // price + 1)` needs both the `1` (at least one of each fruit) and the `+ 1` (inclusive upper bound). Starting at `0` would allow buying zero of a required fruit and would also violate `display`'s `count >= 1` assertion.
- **Getting the base case wrong in an enumeration.** `fruits == [] and amount == 0` must require *both*. Checking only `amount == 0` would print carts missing the trailing fruits; checking only `fruits == []` would print carts that do not spend the full amount.
- **Rebinding a class attribute by accident.** `self.log = ...` creates an instance attribute that shadows the shared class attribute. To update shared state, mutate it: `self.log.append(...)`.
- **Assuming a subclass value hides an instance value.** It is the other way round: the instance's own attributes are checked first, which is why `self.z` is `'A'` and not `2` in the `B` example.
- **Confusing `self` with another object in a method.** In `Speaker.connect`, the assignment target is `m.speakers[location]`, not `self.speakers[location]`; the speaker registers itself with the mic.
- **Passing `str.lower()` instead of `str.lower`.** You store the function object; calling it happens later inside `repeat`.
- **Expecting a generator function to run when called.** Nothing executes until the first `next()`. Likewise, printing inside a generator function will not print until the generator is advanced.
- **Re-using an exhausted or partially consumed generator.** After `next(a)`, `list(a)` is missing the first element. A generator cannot be rewound or iterated twice.
- **Applying list operations to generators.** `a[:-1]` and `reversed(a)` do not work on a generator; convert with `list(a)` first.
- **Putting the label yield in the wrong place.** Moving `yield t.label` above the loop in `all_elements` changes the traversal from bottom-up to top-down and breaks the Waldo answer.
- **Forgetting the implicit base case in `fibs_rec`.** Without the `if i < n` guard, `f` recurses forever (until Python's recursion limit).
- **Mutating a string.** `fruit[:-1]` creates a new string; strings have no `append` and cannot be changed in place.
- **Shadowing a built-in.** The code file's doctest uses `all = all_elements(t)`, which hides the built-in `all` in that scope. Harmless in a doctest, but a habit worth avoiding. (extra context)

## Likely Exam Points

**1. Aliasing in a mutation-plus-accumulation loop**

*Question.* What does the following print, and why?

```python
def f(s):
    out, acc = [], []
    for x in s:
        acc.append(x * 2)
        out.append(acc)
    return out
print(f([1, 2]))
```

*Answer.* `[[2, 4], [2, 4]]`. Both slots of `out` point to the same list object `acc`, which ends up as `[2, 4]`. Replacing `out.append(acc)` with `out.append(list(acc))` would print `[[2], [2, 4]]`.

**2. Writing the recursive-call blank in a tree problem**

*Question.* In `exclude`, what exactly does `exclude(b, x)` return for a branch `b`, and why does that make `bs.extend(b.branches)` correct?

*Answer.* It returns a tree with the same root label as `b` (roots are never excluded) in which every non-root node has a label different from `x`. So if the returned root's label equals `x`, its branches are already fully cleaned and can be spliced directly into the parent's branch list, re-parenting those nodes to their nearest non-excluded ancestor.

**3. Hand-tracing a tree-recursive transformation**

*Question.* For `t = Tree(1, [Tree(2, [Tree(2), Tree(3)]), Tree(2)])`, what does `print(exclude(t, 2))` output?

*Answer.*
```
1
  3
```
The inner `Tree(2, [Tree(2), Tree(3)])` becomes `Tree(2, [Tree(3)])` (its own `2` child contributes no branches), and at the top level its label is `2`, so only `Tree(3)` is promoted. The second branch `Tree(2)` has label `2` and no branches, so it disappears entirely.

**4. String slicing plus f-strings**

*Question.* Fill in the body so that `tag('cats', 1)` returns `'[1 cat]'` and `tag('cats', 4)` returns `'[4 cats]'`, assuming the name always ends in `'s'`.

*Answer.*
```python
def tag(word, count):
    if count == 1:
        word = word[:-1]
    return f'[{count} {word}]'
```
`word[:-1]` drops the final letter, and the f-string interpolates both values.

**5. Enumerating combinations with a helper and accumulators**

*Question.* In `buy`, why is the loop `for k in range(1, amount // price + 1)` rather than `range(0, amount // price + 1)`, and what would go wrong with the latter?

*Answer.* The spec requires at least one of each fruit in `fruits_to_buy`. Starting at `0` would print carts that omit a required fruit, and it would also trip `display`'s `assert count >= 1`.

**6. Base-case reasoning in an exact-sum search**

*Question.* With `prices = {'apples': 3, 'bananas': 2}`, what does `buy(['apples', 'bananas'], prices, 5)` print?

*Answer.* `<1 apple><1 banana>`. Apples: `k` ranges `1..1` (since `5 // 3 == 1`), leaving 2; bananas: `k` ranges `1..1`, leaving 0 with `fruits == []`, so the base case prints the cart. No other combination reaches exactly 5 with at least one of each.

**7. Attribute lookup with inheritance (WWPD)**

*Question.* Given the `A`/`B` classes above, what are `B().f()`, `B.x`, `A.x`, and `B().y`?

*Answer.* `[6, 1, 'A']`, `6`, `0`, and `1`. `B.x` shadows `A.x` for `B` and its instances but does not change `A.x`. `y` is found on `A` by inheritance, and `z` is found as an instance attribute set in `B.__init__`.

**8. Class attributes shared across instances**

*Question.* Why is `self.prior = list(self.log)` written with `list(...)`? What would `ts[1].balance()` be without it?

*Answer.* `log` is a single shared class attribute that keeps growing, so `prior` must be a snapshot copy. Without the copy, every `prior` would alias the final log `[t0, t1, t2]`, and `ts[1].balance()` would be `-3 + (20 + -3 + -4) == -3 + 13 == 10` instead of `17` (and would wrongly include the transaction itself and later ones).

**9. One object mutating another's attributes**

*Question.* Complete `Speaker.connect` so that `Speaker(str.upper).connect(m, 'Back')` makes `m.sing('hi')` also print `Back - HI`.

*Answer.* `m.speakers[location] = self`. The method stores the speaker (its own `self`) into the mic's `speakers` dictionary under the given location key; `sing` then finds it with `self.speakers[k]` and calls `repeat`, which returns `self.transform(s)`.

**10. Generator output order with `yield from`**

*Question.* For `t = Tree(1, [Tree(2), Tree(3, [Tree(4)])])`, what is `list(all_elements(t))`? What if `yield t.label` were moved to the top of the function?

*Answer.* As written: `[2, 4, 3, 1]` (every label comes after all of its descendants). Moving `yield t.label` above the loop gives a top-down traversal: `[1, 2, 3, 4]`.

**11. Partially consumed generators**

*Question.* Using the Waldo tree, what does `list(a)` evaluate to after a single `next(a)` call, and why isn't the first element `'a'` included?

*Answer.* `['o', 'd', 'l', 'a', 'W', 'o']`. The generator resumed past the first `yield` during `next(a)`, so `'a'` was already consumed and cannot be produced again; generators are single-pass iterators with no rewind.

**12. Recursion that mutates an enclosing list**

*Question.* In `fibs_rec`, why is no `nonlocal s` declaration needed, and what is the terminating condition?

*Answer.* `f` only calls `s.append(...)`, which mutates the existing list object; it never assigns to the name `s`, so no rebinding (and therefore no `nonlocal`) is required. Recursion stops when `i < n` is false: the body is skipped, nothing is appended, and no further recursive call is made.

**13. Writing an accumulation recursion from scratch**

*Question.* Fill in the blanks so `squares(4)` returns `[0, 1, 4, 9]`.

```python
def squares(n):
    s = []
    def f(i):
        if ______:
            s.append(i * i)
            ______
    ______
    return s
```

*Answer.* `if i < n:`, then `f(i + 1)`, then `f(0)`. This mirrors `fibs_rec` exactly: an implicit base case, an append, a tail recursive advance, and a kickoff call.

## Summary

- **Aliasing**: appending the same mutable list repeatedly stores repeated references, not snapshots. `prefixes` is fixed by `result.append(list(so_far))`. The same copy discipline appears in `Transaction`'s `self.prior = list(self.log)`.
- **Recursion recipe**: write an example input, identify the smallest initial choice, write the recursive call(s) for that input, and write down *what each call returns*. That last step is what makes the combination step obvious.
- **`exclude`**: `map(lambda y: exclude(y, x), t.branches)` recurses first; then for each returned branch, `bs.extend(b.branches)` if `b.label == x` (promoting grandchildren to their nearest non-excluded ancestor) else `bs.append(b)`; finally `return Tree(t.label, bs)`. Roots are never excluded. Only 20% of students solved this on the Spring 2023 exam.
- **Strings**: `fruit[-1]` inspects, `fruit[:-1]` drops the last character, and `f'<{count} {fruit}>'` formats. Strings are immutable, so slicing always makes a new string.
- **`buy`**: a nested helper with accumulators (`fruits`, `amount`, `cart`); base case `fruits == [] and amount == 0` prints; the loop `for k in range(1, amount // price + 1)` enumerates how many of `fruits[0]` to buy (at least one, at most what the money allows); the recursive call is `add(fruits[1:], amount - price * k, cart + display(fruit, k))`.
- **Attribute lookup**: instance attributes, then the class, then base classes; the first match wins. `B().f()` is `[6, 1, 'A']`.
- **Class attributes**: one shared value for all instances. Mutate shared state (`self.log.append(self)`); assigning `self.log = ...` silently creates an instance attribute instead.
- **Cross-object mutation**: `Speaker.connect` does `m.speakers[location] = self`, registering itself inside another object. `Mic` starts with `{'Front': Speaker(lambda x: x)}`, and `repeat` returns `self.transform(s)`.
- **Generators**: calling a generator function runs no code; `yield from` delegates to a sub-iterator; `all_elements` yields branches before the label, giving a bottom-up traversal; a consumed element is gone forever, which is why `list(reversed(list(a)[:-1]))` spells `['W', 'a', 'l', 'd', 'o']` after one `next(a)`.
- **Recursive list building**: `fibs_rec` uses an inner `f(i)` guarded by `if i < n`, appending `i` for `i < 2` and `s[-2] + s[-1]` otherwise, then calling `f(i + 1)`, with `f(0)` to start. No `nonlocal` is needed because the list is mutated, not rebound.
