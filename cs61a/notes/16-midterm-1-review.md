<!-- Fri, Oct 02, 2026 | sources: slides (no transcript available) -->
# Lecture 16: Midterm 2 Review

This lecture is a pure practice session: no new material, just four problems drilled in the style of Midterm 2. It walks through (1) a linked list traversal written two ways, once printing with a `None`-returning helper and once building a brand new linked list, (2) a list problem solved first with an indexed `for` loop and then collapsed into a one-liner using `min` with a `key` function, and (3) the Spring 2023 Midterm 2 parking problem, first as a counting problem (`count_park`) and then as an enumeration problem (`park`) that returns a list of strings. Along the way it steps back twice: once to catalog the recursion strategies seen so far in the course ("deal with one item, recurse on the rest" versus "make a small choice, recurse"), and once to review how `+` and list comprehensions behave on lists and strings, which is exactly the machinery the final `park` solution needs. The unifying theme is that almost every one of these problems is solved by asking "what is one choice or one element I can handle right now, and what does the recursive call give me for the rest?"

---

## Key Concepts

### 1. Linked lists are recursive data, so write recursive functions over them

A `Link` object has two attributes: `first` (a label) and `rest` (either another `Link` or the empty list). Because `rest` is itself a linked list, any function over a linked list naturally has the shape:

```python
if not isinstance(t, Link):   # base case: t is the empty list
    ...
else:
    ... t.first ...           # handle the one element in front of us
    ... process(t.rest) ...   # recurse on the rest
```

The lecture's test for "am I at the end?" is `isinstance(t, Link)`. This matters because the empty linked list in 61A is `()`, which is *not* a `Link`, so you cannot ask for `t.first` on it. Checking `isinstance` before touching `.first` or `.rest` is the linked list equivalent of checking `if s == []` on a Python list.

*(extra context)* The `Link` class the lecture assumes looks like this:

```python
class Link:
    empty = ()
    def __init__(self, first, rest=empty):
        assert rest is Link.empty or isinstance(rest, Link)
        self.first = first
        self.rest = rest
```

The slide's displayed output `Link(first=5, rest=())` tells you this version's `__repr__` prints keyword-style, and that `Link(5)` means "a one element list" because `rest` defaults to `()`.

### 2. Why `above_first` needs an inner helper function

The problem is: print every label larger than the *first* label of the whole list. The recursion walks down the list, so the parameter changes on every call, but the thing we compare against must *not* change. The solution is to define a helper inside:

```python
def above_first(s):
    def process(t):
        ...
    process(s)
```

`s` is bound once in `above_first`'s frame and never rebound. `process` is defined inside `above_first`, so (by lexical scoping) `process`'s parent frame is `above_first`'s frame. Every recursive call to `process` creates a new frame with its own `t`, but all of those frames share the same parent, so `s.first` always resolves to the original first label. In environment diagram terms: you get a chain of `process` frames, each with a different `t`, each with a parent arrow pointing back at the single `above_first` frame that holds `s`. This "outer function holds the fixed value, inner function holds the moving value" pattern is one of the most heavily tested ideas on Midterm 2.

### 3. Printing versus building: two different return contracts

The lecture deliberately writes the same traversal twice.

- `above_first` **prints**. Its helper's job is a side effect, so the helper returns `None` and the recursive call `process(t.rest)` is a *statement* on its own line. The order of the `print` and the recursive call determines the output order.
- `above_first_new` **returns a new linked list**. Now every branch must `return` something of the right type: `()` for the empty case, and `Link(t.first, process(t.rest))` when we keep an element. The crucial move is trusting that `process(t.rest)` already hands back a correctly filtered linked list, so all we do is glue one new `Link` onto the front of it.

Notice the filtering shape: there are two recursive branches, `return Link(t.first, process(t.rest))` when we keep and `return process(t.rest)` when we skip. "Skip" means returning the recursive result unchanged.

In box-and-pointer terms, `above_first_new` **does not mutate or share** the original spine: it allocates a fresh `Link` for every kept element. The original list is untouched, and the new list's `first` attributes point at the same label values but live in new boxes.

### 4. Iterating over indices when you need both position and value

`closest_to_index` needs, for each element, both its index `i` and its value `s[i]`, because the quantity being minimized is `abs(i - s[i])`. That forces `for i in range(len(s))` rather than `for x in s`. The loop keeps a running "best so far" as an *index* (`smallest`) rather than as a value, because the answer requires `s[smallest]` and the comparison requires the index too.

### 5. The same problem as a one-liner with `min` and `key`

```python
return s[min(range(len(s)), key=lambda i: abs(i - s[i]))]
```

`min(iterable, key=f)` returns the element of `iterable` for which `f(element)` is smallest, but it returns the *element*, not `f`'s value. Here the iterable is `range(len(s))`, so the elements are indices, so `min` returns the best index, and we index back into `s` to get the value. This is a compact illustration of a higher-order function taking a `lambda`, and of choosing what to iterate over so that the answer falls out.

### 6. Two distinct recursion strategies (the "Recursion so far" slide)

The lecture organizes everything seen so far into two families:

**Family A: deal with one item or digit, recurse for the rest.** One recursive call per level. The recursion is essentially a loop.
- `double_eights(s)`: check whether the first two elements are both 8, else recurse on everything but the first element.
- `streak(n)`: check whether the last digit is a valid die digit and matches the previous one, then recurse on everything but the last digit.
- `reverse(s)`: get the first element into place, recurse on the rest.

**Family B: tree recursion, make a SMALL choice and recurse.** Several recursive calls per level, one per available choice, and you add (or concatenate) the results.
- `count_partitions(n, m)`: either use a piece of size `m` and recurse, or use no pieces of size `m` and recurse.
- `park(n)`: use a motorcycle and recurse, use nothing and recurse, use a car and recurse.

The word "SMALL" is the operative hint: do not try to decide the whole arrangement, decide only the very next spot.

### 7. The parking problem: one choice for the first spot

Given `n` adjacent spots, the first spot can be filled in exactly three ways:
- a motorcycle `%`, consuming 1 spot, leaving `n - 1`
- nothing `.`, consuming 1 spot, leaving `n - 1`
- a car `<>`, consuming 2 spots, leaving `n - 2`

These three cases are mutually exclusive and cover everything, so the counts add:

```python
count_park(n-1) + count_park(n-1) + count_park(n-2)
```

The lecture's slide labels the three terms explicitly: "Number of ways to fill 3 spots if you start with a motorcycle" + "... if you start with an empty space" + "... if you start with a car". The two `n-1` terms are *not* a typo: a motorcycle and an empty spot both consume one spot but are different arrangements.

### 8. Base cases found by asking "which recursive calls land here?"

The slide's method for base cases is to look at the bottom of the recursion tree and ask what each leaf must return.

- `n == 0`: zero spots left, and there is exactly **one** way to fill zero spots (do nothing). Return `1`. Returning `0` here would make the whole function return `0`.
- `n < 0`: this is reached when you place a car with only 1 spot remaining, which is illegal. That partial arrangement must contribute nothing, so return `0`.

Check: `count_park(1) = 1 + 1 + 0 = 2`, `count_park(2) = 2 + 2 + 1 = 5`, `count_park(3) = 5 + 5 + 2 = 12` (the slide lists all 12 strings), `count_park(4) = 12 + 12 + 5 = 29`. All match the doctests. *(extra context)* These are the Pell numbers; the `n < 0` case could alternatively be avoided by guarding the car term with `if n >= 2`.

### 9. From counting to enumerating: `1` becomes `['']` and `0` becomes `[]`

`park(n)` returns the actual list of strings instead of a count. The structure is identical; only the "units" change:

| counting version | enumerating version | why |
| --- | --- | --- |
| `return 0` | `return []` | no arrangements at all |
| `return 1` | `return ['']` | exactly one arrangement, the empty one |
| `a + b + c` (numbers) | `a + b + c` (list concatenation) | combine the alternatives |
| (implicit) | `['%' + s for s in park(n-1)]` | prepend the choice to every sub-arrangement |

The "Quick Review: Adding Lists & Strings" slide exists precisely to make the two base cases obvious:

```python
>>> s = []
>>> ['cal' + x for x in s]
[]
>>> s = ['']
>>> ['cal' + x for x in s]
['cal']
```

A comprehension over `[]` produces `[]`, so the illegal `n < 0` branch correctly annihilates everything prepended to it. A comprehension over `['']` produces exactly one string, the prefix alone, so `['']` is the right "one way to do nothing". That poll question (`pollev.com/cs61a`) has answers: `s = ['']` gives `['cal']`, and `s = []` gives `[]`.

The same slide also drills the distinction between string `+` and list `+`:

```python
>>> x, y = 'cal', 'bears'
>>> x + y          # string concatenation
'calbears'
>>> [x] + [y]      # list concatenation, two elements
['cal', 'bears']
>>> ['go ' + z for z in [x, y]]
['go cal', 'go bears']
```

---

## Definitions

- **Linked list**: either the empty list (`()` in this course's convention) or a `Link` object with a `first` attribute (the label) and a `rest` attribute that is itself a linked list.
- **`Link`**: the class representing one non-empty linked list node; `Link(3, Link(0))` is a two element list, and `Link(5)` is a one element list because `rest` defaults to empty.
- **`LinkedList`** (the type annotation used on the slides): the type "`Link` or the empty list", i.e. what a linked list parameter or return value may be.
- **`isinstance(t, Link)`**: the test for "`t` is a non-empty linked list"; its negation is the base case of a linked list recursion.
- **Helper function with a closure**: an inner function defined inside an outer function so that it can read the outer function's parameters (here, `s`) at every level of recursion without passing them along.
- **Side effect versus return value**: `above_first` produces output via `print` and returns `None`; `above_first_new` produces a value via `return` and prints nothing.
- **`min(iterable, key=f)`**: returns the element `e` of `iterable` minimizing `f(e)`; it returns `e` itself, not `f(e)`.
- **`lambda i: abs(i - s[i])`**: an anonymous one-expression function; here it maps an index to that index's distance from its own value, and it closes over `s`.
- **Tree recursion**: a recursive pattern with more than one recursive call per invocation, typically one call per available choice, whose results are combined (summed for counting, concatenated for enumeration).
- **Small choice**: in tree recursion, committing only to the very next step (the next parking spot, the next partition piece) and delegating everything after it to the recursive calls.
- **Motorcycle / car / empty spot (parking problem)**: `'%'` occupies 1 spot, `'<>'` occupies 2 adjacent spots, `'.'` is 1 empty spot. A length-`n` string such as `'.%%.<><>'` represents `n` adjacent spots.
- **`count_park(n)`**: the number of ways to park in `n` adjacent spots, for positive integer `n`; some or all spots may be empty.
- **`park(n)`**: the list of all those ways, each represented as a string of length `n`.
- **Base case value `['']`**: the single-element list containing the empty string, representing "exactly one way, which contributes no characters"; the enumeration analogue of the count `1`.

---

## Worked Examples

### Example 1: `above_first` (print all labels larger than the first)

The skeleton on the slide, with four blanks, was filled in as:

```python
def above_first(s: LinkedList):
    """Print all of the labels of s that are larger than first.

    >>> above_first(Link(3, Link(0, Link(8, Link(5)))))
    8
    5
    """
    def process(t: LinkedList):
        if isinstance(t, Link):
            if t.first > s.first:
                print(t.first)
            process(t.rest)
    process(s)
```

**Step by step on `Link(3, Link(0, Link(8, Link(5))))`:**

1. `above_first` is called; its frame binds `s` to the four element list. `def process` creates a function whose parent is this frame. Then `process(s)` is called.
2. `process(t = 3 -> 0 -> 8 -> 5)`. `isinstance(t, Link)` is true. `t.first` is `3`; `s.first` is `3`; `3 > 3` is false, so nothing prints. Then `process(t.rest)`.
3. `process(t = 0 -> 8 -> 5)`. `0 > 3` is false. Recurse.
4. `process(t = 8 -> 5)`. `8 > 3` is true, so print `8`. Recurse.
5. `process(t = 5)`. `5 > 3` is true, so print `5`. Then `process(t.rest)` where `t.rest` is `()`.
6. `process(t = ())`. `isinstance((), Link)` is false, the whole `if` body is skipped, and the function falls off the end returning `None`. The stack unwinds and `above_first` returns `None`.

Output is `8` then `5`, matching the doctest.

**Why this shape:** the `print` happens *before* the recursive call, so labels come out in list order. If `process(t.rest)` came before the `if`, the output would be `5` then `8`. Also note the first element is never printed, since no label is strictly greater than itself; the problem says "larger than first", not "larger than or equal to".

**Scoping detail worth internalizing:** there are five `process` frames alive at the deepest point, each with its own `t`, and all five have the *same* parent frame (the one `above_first` frame). That is why `s.first` is a stable `3` in all of them. If you had instead written `process(s, t)` and passed `s` along, you would have to be careful never to replace `s` with `s.rest` by accident; the closure makes that mistake impossible.

### Example 2: `above_first_new` (return a new filtered linked list)

```python
def above_first_new(s: LinkedList) -> LinkedList:
    """Return a new list with only the items in s larger than first.

    >>> above_first_new(Link(3, Link(5, Link(1))))
    Link(first=5, rest=())
    >>> above_first_new(Link(3, Link(0, Link(8, Link(5)))))
    Link(first=8, rest=Link(first=5, rest=()))
    """
    def process(t: LinkedList) -> LinkedList:
        if not isinstance(t, Link):
            return ()
        if t.first > s.first:
            return Link(t.first, process(t.rest))
        return process(t.rest)
    return process(s)
```

**Step by step on `Link(3, Link(0, Link(8, Link(5))))`:**

1. `process(3 -> 0 -> 8 -> 5)`: `t` is a `Link`; `3 > 3` is false; so return whatever `process(0 -> 8 -> 5)` returns. This frame contributes nothing of its own.
2. `process(0 -> 8 -> 5)`: `0 > 3` is false; return `process(8 -> 5)`.
3. `process(8 -> 5)`: `8 > 3` is true; return `Link(8, process(5))`. Python must evaluate `process(5)` first, so this frame waits.
4. `process(5)`: `5 > 3` is true; return `Link(5, process(()))`, so it waits on `process(())`.
5. `process(())`: not a `Link`, return `()`.
6. Unwinding: step 5 gives `()`, so step 4 builds `Link(5, ())`. Step 3 builds `Link(8, Link(5, ()))`. Steps 2 and 1 pass that value straight up unchanged. `above_first_new` returns it.

Printed, that value is `Link(first=8, rest=Link(first=5, rest=()))`, matching the doctest.

**Box-and-pointer reasoning in words:** before the call there is one chain of four `Link` boxes holding `3, 0, 8, 5`. After the call there is a *second*, separate chain of two brand new boxes holding `8` and `5`, with the second box's `rest` pointing at `()`. No box in the original chain was modified, and no box is shared between the two chains, because every kept element is wrapped in a freshly constructed `Link`. Constructing `Link(t.first, ...)` copies the label, not the node.

**Why the three-branch shape:** this is the standard filter-on-a-linked-list template. Base case returns the empty list. Keep case conses the current label onto the recursive result. Skip case returns the recursive result untouched. Compare with `above_first`, where the same three situations were handled by "do nothing", "print then recurse", and "recurse", because there was no value to pass upward.

### Example 3: `closest_to_index` with an indexed loop

The skeleton (with `biggest = 0` on the unfilled slide) was completed as:

```python
def closest_to_index(s):
    """Return the item in non-empty list s whose value has the
    smallest absolute difference from its index.

    >>> closest_to_index([4, 2, 10])
    2
    >>> closest_to_index([-1, 100])
    -1
    """
    smallest = 0
    for i in range(len(s)):
        if abs(i - s[i]) < s[smallest]:
            smallest = i
    return s[smallest]
```

**Step by step on `[4, 2, 10]`:**

1. `smallest = 0`, meaning "index 0 is my best guess so far".
2. `i = 0`: `abs(0 - 4)` is `4`; compare to `s[0]` which is `4`; `4 < 4` is false, no update.
3. `i = 1`: `abs(1 - 2)` is `1`; compare to `s[0]` which is `4`; `1 < 4` is true, so `smallest = 1`.
4. `i = 2`: `abs(2 - 10)` is `8`; compare to `s[1]` which is `2`; `8 < 2` is false, no update.
5. Return `s[1]`, which is `2`. Correct: the distances are `4, 1, 8` and the smallest is at index 1.

**Step by step on `[-1, 100]`:** distances are `abs(0 - (-1)) = 1` and `abs(1 - 100) = 99`, so the answer should be index 0, value `-1`. The loop: `i = 0` gives `1 < s[0] = -1`, false; `i = 1` gives `99 < s[0] = -1`, false; return `s[0] = -1`. Correct.

**A correctness caveat on the slide's filled-in comparison (extra context):** the right-hand side `s[smallest]` is the best element's *value*, not the best element's *distance*. To compare like with like it should be `abs(smallest - s[smallest])`:

```python
    for i in range(len(s)):
        if abs(i - s[i]) < abs(smallest - s[smallest]):
            smallest = i
```

The slide's version happens to pass both doctests, but it fails whenever the current best has a negative value. For instance `closest_to_index([-5, -1])` should return `-1` (distances `5` and `2`), while the slide's version compares `2 < -5`, finds it false, and returns `-5`. Learn the pattern with the `abs(...)` on both sides; that is what the `min`-with-`key` version below is actually doing.

### Example 4: `closest_to_index` as a one-liner

```python
def closest_to_index(s):
    return s[min(range(len(s)), key=lambda i: abs(i - s[i]))]
```

**How to read it from the inside out, on `[4, 2, 10]`:**

1. `range(len(s))` is the indices `0, 1, 2`.
2. `min` applies the key to each index: `lambda 0` gives `abs(0 - 4) = 4`, `lambda 1` gives `abs(1 - 2) = 1`, `lambda 2` gives `abs(2 - 10) = 8`.
3. The smallest key value is `1`, produced by index `1`, so `min` returns `1` (the *index*, not the key value `1`; they coincide here by accident).
4. `s[1]` is `2`.

The `lambda` closes over `s` from the enclosing `closest_to_index` frame, which is why it can write `s[i]`. Ties go to the earliest index, since `min` keeps the first minimum it sees, which matches the loop version's strict `<`.

### Example 5: `count_park` (tree recursion, counting)

```python
def count_park(n):
    """Count the ways to park cars and motorcycles in n adjacent spots.
    >>> count_park(1) # '.' or '%'
    2
    >>> count_park(2) # '..', '.%', '%.', '%%', or '<>'
    5
    >>> count_park(4) # some examples: '<><>', '.%%.', '%<>%', '%.<>'
    29
    """
    if n < 0:
        return 0
    elif n == 0:
        return 1
    else:
        return count_park(n-1) + count_park(n-1) + count_park(n-2)
```

**How the recursion tree is built (following the slide's diagram):** start at the root with "3 spaces left, haven't parked anything". The choice is how to fill the *first* space. Branch left with `%` (now 2 spaces left), branch middle with `.` (now 2 spaces left), branch right with `<>` (now 1 space left). Each child repeats the same three-way choice. A path from root to a node with 0 spaces left spells out one complete valid arrangement, read off the edge labels in order. The slide annotates one such path as "motorcycle, motorcycle, motorcycle", which spells `'%%%'`.

**The `-1 spaces left` nodes:** these come from placing `<>` when only one space remained, e.g. the paths `%%<>`, `%.<>`, and `<><>` in the diagram when `n` started at 3. Wait: `<><>` uses four spots, so from `n = 3` placing a car leaves 1, then placing another car leaves `-1`. Those nodes represent impossible arrangements, so `count_park` must return `0` there to delete them from the total.

**Full evaluation of `count_park(3)`:**

- `count_park(0) = 1`, `count_park(-1) = 0`.
- `count_park(1) = count_park(0) + count_park(0) + count_park(-1) = 1 + 1 + 0 = 2`. The two arrangements: `'%'` and `'.'`.
- `count_park(2) = count_park(1) + count_park(1) + count_park(0) = 2 + 2 + 1 = 5`. The five: `'%%', '%.', '.%', '..', '<>'`.
- `count_park(3) = count_park(2) + count_park(2) + count_park(1) = 5 + 5 + 2 = 12`. The slide enumerates exactly these twelve: `%%%, %%., %.%, %.., %<>, .%%, .%., ..%, ..., .<>, <>%, <>.`
- `count_park(4) = 12 + 12 + 5 = 29`, matching the doctest.

Notice how the twelve strings for `n = 3` split: five begin with `%`, five begin with `.`, two begin with `<>`. That is the recurrence made visible.

### Example 6: `park` (tree recursion, enumerating strings)

```python
def park(n):
    """Return the ways to park cars and motorcycles in n adjacent spots.
    >>> park(1)
    ['%', '.']
    >>> park(2)
    ['%%', '%.', '.%', '..', '<>']
    >>> len(park(4)) # some examples: '<><>', '.%%.', '%<>%', '%.<>'
    29
    """
    if n < 0:
        return []
    elif n == 0:
        return ['']
    else:
        return (['%' + s for s in park(n-1)]
                + ['.' + s for s in park(n-1)]
                + ['<>' + s for s in park(n-2)])
```

**Step by step on `park(2)`:**

1. `park(2)` needs `park(1)` twice and `park(0)` once.
2. `park(1)` needs `park(0)` twice and `park(-1)` once.
   - `park(0)` returns `['']`.
   - `['%' + s for s in ['']]` is `['%']`.
   - `['.' + s for s in ['']]` is `['.']`.
   - `park(-1)` returns `[]`, so `['<>' + s for s in []]` is `[]`: no car fits in one spot, and the empty comprehension expresses that automatically.
   - Result: `['%'] + ['.'] + []`, which is `['%', '.']`. Matches the doctest.
3. Back in `park(2)`:
   - `['%' + s for s in ['%', '.']]` is `['%%', '%.']`.
   - `['.' + s for s in ['%', '.']]` is `['.%', '..']`.
   - `['<>' + s for s in ['']]` is `['<>']`.
   - Concatenated: `['%%', '%.', '.%', '..', '<>']`. Matches the doctest, including the order.

**Three things to notice:**

- The *order* of the output is determined by the order of the three concatenated pieces (motorcycle first, then nothing, then car) and by the order within each recursive result. The doctests pin this order down, so on an exam write the pieces in the order the doctest implies.
- Every returned string has length exactly `n`, because the base case contributes `''` (length 0) and each level prepends exactly the number of characters it consumed: 1 for `%`, 1 for `.`, 2 for `<>`.
- The comprehension variable is also named `s` on the slide, which is fine: a comprehension's variable is local to the comprehension and does not leak out or clash with anything.

**Relationship to `count_park`:** `len(park(n)) == count_park(n)` for every `n`, and the proof is structural, since `len([])` is `0` (matching the `0` base case), `len([''])` is `1` (matching the `1` base case), prepending a prefix preserves length, and `len(a + b + c) == len(a) + len(b) + len(c)` (matching the sum). The doctest `len(park(4))` being `29` is exactly this check.

---

## Common Pitfalls

1. **Touching `.first` or `.rest` on the empty list.** `()` has no attributes. Always guard with `isinstance(t, Link)` before accessing them, and put the guard first.
2. **Writing `isinstance(t, Link)` without the colon**, as literally appears mid-fill on slide 5. The finished line is `if isinstance(t, Link):`.
3. **Confusing `Link.empty` with `None`.** This course's empty linked list is `()`, so `if t is None` and `if not t` are not the tests to use; `isinstance` is.
4. **Letting the fixed value drift.** In `above_first`, comparing `t.first > t.first` (always false) or recursing in a way that rebinds `s` destroys the problem. The whole point of the inner helper is that `s` cannot change.
5. **Mixing up printing and returning.** In `above_first` the recursive call must be a bare statement; adding `return process(t.rest)` would stop the traversal from printing later elements only if placed before the `print`, but more importantly `above_first`'s doctest shows printed output with no `Link(...)` echo, so it must return `None`. Conversely, in `above_first_new`, forgetting `return` in front of `process(t.rest)` in the skip branch makes the function return `None`.
6. **Forgetting the skip branch's shape.** In a filter, "skip this element" is `return process(t.rest)`, not `return Link(process(t.rest))` and not `return ()`.
7. **Iterating over values when you need indices.** `for x in s` cannot compute `abs(i - s[i])`. Use `for i in range(len(s))`.
8. **Storing the best value instead of the best index**, then being unable to return `s[smallest]` or to recompute the comparison. Track the index.
9. **Comparing a distance against a value**, which is the bug in the slide's filled comparison `abs(i - s[i]) < s[smallest]`. Both sides of the comparison must be distances: `abs(i - s[i]) < abs(smallest - s[smallest])`. *(extra context: flagged because it passes the given doctests but is wrong for inputs like `[-5, -1]`.)*
10. **Thinking `min(..., key=f)` returns `f`'s value.** It returns the element. If you want the smallest key value, that is `min(f(x) for x in ...)` instead.
11. **Collapsing the two `n-1` branches by mistake.** `count_park(n-1) + count_park(n-2)` is wrong; motorcycle-first and empty-first are two different families of arrangements. (`2 * count_park(n-1) + count_park(n-2)` is a legitimate rewrite for the counting version, but for `park` the two branches prepend different characters and must stay separate.)
12. **Using `0` instead of `1` for the zero-spots base case**, which makes every count `0`. "There is one way to do nothing" is the rule.
13. **Using `1` or `[0]`-style values instead of `0` / `[]` for the negative case**, which counts illegal arrangements where a car hangs off the end.
14. **Using `[]` instead of `['']` as the enumeration base case**, which makes `park(n)` return `[]` for every `n`, since every comprehension over `[]` is empty. This is exactly the point of the `'cal'` poll.
15. **Prepending with `+` on the wrong type.** `'%' + s` concatenates strings inside the comprehension; `['%'] + park(n-1)` would instead produce a list whose first element is the bare string `'%'`, which is not what the doctest shows.
16. **Off-by-one on car length.** The car is the two-character string `'<>'` occupying 2 spots, so it pairs `'<>' + s` with `park(n-2)`. Pairing `'<>'` with `park(n-1)` produces strings of the wrong length.
17. **Assuming spots must be filled.** The problem explicitly allows "some or all spots" to be empty, which is why `.` is one of the three choices rather than an afterthought.

---

## Likely Exam Points

### A. Linked list traversal with a fixed reference value (closure over an outer parameter)

**Practice:** Implement `below_last(s)` that prints every label of a non-empty linked list `s` that is smaller than the *last* label, in list order.

**Answer:**
```python
def below_last(s):
    def last(t):
        if not isinstance(t.rest, Link):
            return t.first
        return last(t.rest)
    end = last(s)
    def process(t):
        if isinstance(t, Link):
            if t.first < end:
                print(t.first)
            process(t.rest)
    process(s)
```
The structure mirrors `above_first`: compute the fixed value once in the outer frame, then let the inner recursive helper read it from its parent frame at every level.

### B. Converting a printing traversal into a list-building one

**Practice:** Rewrite `above_first_new` so it returns a *Python list* of the qualifying labels instead of a linked list.

**Answer:**
```python
def above_first_list(s):
    def process(t):
        if not isinstance(t, Link):
            return []
        if t.first > s.first:
            return [t.first] + process(t.rest)
        return process(t.rest)
    return process(s)
```
The base case becomes `[]` instead of `()`, and `Link(t.first, rest)` becomes `[t.first] + rest`. Everything else is unchanged, which is the point: the recursive *shape* is the reusable part.

### C. Tracing a linked list recursion / what gets printed in what order

**Practice:** What does `above_first(Link(5, Link(7, Link(5, Link(9)))))` print?

**Answer:** `7` then `9`. The first element `5` is skipped (`5 > 5` is false), the third element `5` is also skipped for the same reason, and because `print` precedes the recursive call the output follows list order.

### D. `for i in range(len(s))` accumulator problems that track an index

**Practice:** Fill in a function `farthest_from_index(s)` returning the item of non-empty `s` with the *largest* absolute difference from its index.

**Answer:**
```python
def farthest_from_index(s):
    biggest = 0
    for i in range(len(s)):
        if abs(i - s[i]) > abs(biggest - s[biggest]):
            biggest = i
    return s[biggest]
```
Or as a one-liner, `return s[max(range(len(s)), key=lambda i: abs(i - s[i]))]`. Note the only changes from the lecture version: `<` becomes `>` and `min` becomes `max`.

### E. `min`/`max` with a `key` lambda, and knowing what they return

**Practice:** What does `min(range(4), key=lambda i: abs(i - 3))` evaluate to, and what does `min([abs(i - 3) for i in range(4)])` evaluate to?

**Answer:** `3` and `0`. The first returns the *element* (the index `3`, whose key value is `0`); the second returns the *key value* `0`. Confusing these is a classic multiple-choice trap.

### F. Choosing the right recursion strategy

**Practice:** For each, say whether it is "handle one item, recurse on the rest" or "make a small choice, recurse" (tree recursion): (i) reverse a list, (ii) count the ways to make change for `n` cents using coins of up to `m`, (iii) check whether a list contains two consecutive 8s.

**Answer:** (i) handle one item (get the first element into place, recurse on the rest); (ii) tree recursion (use a coin of size `m` and recurse, or use no coins of size `m` and recurse, which is `count_partitions`); (iii) handle one item (check the first two, recurse on everything but the first). The lecture's "Recursion so far" slide groups exactly these.

### G. Writing the recurrence for a counting problem with variable-size pieces

**Practice:** Suppose a bus takes 3 adjacent spots (`'[-]'`) in addition to motorcycles, cars, and empty spots. Write `count_park_bus(n)`.

**Answer:**
```python
def count_park_bus(n):
    if n < 0:
        return 0
    elif n == 0:
        return 1
    else:
        return (count_park_bus(n-1) + count_park_bus(n-1)
                + count_park_bus(n-2) + count_park_bus(n-3))
```
One term per choice, each subtracting that choice's footprint. The `n < 0` case now also absorbs a bus placed with 1 or 2 spots left.

### H. Base cases for tree recursion: `0` versus `1`, `[]` versus `['']`

**Practice:** A student writes `park` with `return [''] ` for `n < 0` and `return ['']` for `n == 0`. What goes wrong, concretely, for `park(1)`?

**Answer:** The car branch becomes `['<>' + s for s in park(-1)]`, which is `['<>']`, so `park(1)` returns `['%', '.', '<>']`: a two-character arrangement in a one-spot lot. The `n < 0` base case must be `[]` so that the comprehension over it is empty and the illegal branch contributes nothing. Symmetrically, using `[]` for `n == 0` makes every result empty.

### I. Converting a count into an enumeration (the `count_park` to `park` transformation)

**Practice:** Given `count_partitions(n, m)` with base cases `n == 0 -> 1` and `n < 0 or m == 0 -> 0` and body `count_partitions(n-m, m) + count_partitions(n, m-1)`, write `list_partitions(n, m)` returning a list of strings like `'3 + 1'`.

**Answer:** The mechanical version of the transformation is: `1` becomes `['']` (or a one-element list representing "the empty partition"), `0` becomes `[]`, `+` on numbers becomes `+` on lists, and each recursive term gets wrapped in a comprehension that prepends the choice made:
```python
def list_partitions(n, m):
    if n == 0:
        return ['']
    elif n < 0 or m == 0:
        return []
    with_m = [str(m) + (' + ' + p if p else '') for p in list_partitions(n-m, m)]
    without_m = list_partitions(n, m-1)
    return with_m + without_m
```
The exam-relevant point is the correspondence of base cases and the comprehension-prepends-the-choice pattern, not the string formatting details. *(extra context: `list_partitions` itself was not on the slides; it is the natural analogue of the lecture's `count_park` to `park` move applied to the other tree recursion the slides name.)*

### J. List and string `+` and empty-comprehension behavior (the poll)

**Practice:** For which `s` does `['cal' + x for x in s]` evaluate to `['cal']`, and for which does it evaluate to `[]`?

**Answer:** `s = ['']` gives `['cal']` (one iteration, prefix plus empty string). `s = []` gives `[]` (zero iterations). More generally, a comprehension over an empty sequence is always `[]` regardless of the expression, which is what makes `[]` the correct "impossible" base case.

### K. Verifying your own recursion against given doctests

**Practice:** Without running code, confirm `count_park(4) == 29` from the recurrence.

**Answer:** `count_park(0) = 1`, `count_park(-1) = 0`; `count_park(1) = 1+1+0 = 2`; `count_park(2) = 2+2+1 = 5`; `count_park(3) = 5+5+2 = 12`; `count_park(4) = 12+12+5 = 29`. Building this little table upward from the base cases is the fastest exam check that your base cases and recurrence agree with the doctests.

---

## Summary

- **Linked list recursion template:** guard with `isinstance(t, Link)` first, handle `t.first`, recurse on `t.rest`. The empty linked list is `()` and has no attributes.
- **Use an inner helper with a closure** when the recursion needs a value that must stay fixed (`s.first` in `above_first`) while another value moves (`t`). All helper frames share the single outer frame as parent, so the fixed value never drifts.
- **Printing versus returning are different contracts:** `above_first` prints and returns `None`, with the recursive call as a bare statement; `above_first_new` returns a new linked list, with `()` in the base case, `Link(t.first, process(t.rest))` to keep, and `process(t.rest)` to skip.
- **`above_first_new` builds fresh `Link` boxes** and shares no structure with the input, which stays unmodified.
- **When a problem couples position and value**, loop over `for i in range(len(s))` and track the best *index*, returning `s[best]` at the end. The lecture's filled comparison should read `abs(i - s[i]) < abs(smallest - s[smallest])` to compare distances to distances *(extra context: the slide's `< s[smallest]` passes the doctests but fails on negative values)*.
- **`min(iterable, key=f)` returns the element, not the key value**, which makes `s[min(range(len(s)), key=lambda i: abs(i - s[i]))]` a correct one-liner; the `lambda` closes over `s`.
- **Two recursion families:** "deal with one item or digit, recurse for the rest" (`double_eights`, `streak`, `reverse`) versus tree recursion, "make a SMALL choice, recurse" (`count_partitions`, `count_park`, `park`).
- **Parking recurrence:** fill only the first spot, three ways: `%` (1 spot), `.` (1 spot), `<>` (2 spots), giving `count_park(n-1) + count_park(n-1) + count_park(n-2)`. Keep the two `n-1` terms separate.
- **Base cases by asking which calls reach them:** `n == 0` returns `1` (one way to do nothing); `n < 0` returns `0` (a car hanging off the end is illegal). Check: 2, 5, 12, 29 for `n = 1, 2, 3, 4`.
- **Count to enumeration is a mechanical translation:** `1` becomes `['']`, `0` becomes `[]`, numeric `+` becomes list `+`, and each term becomes a comprehension prepending that choice: `['%' + s for s in park(n-1)] + ['.' + s for s in park(n-1)] + ['<>' + s for s in park(n-2)]`.
- **Why `['']` and `[]` are the right units:** a comprehension over `['']` yields just the prefix, and a comprehension over `[]` yields `[]`, so the illegal branch cancels itself out. Hence `len(park(n)) == count_park(n)`.
- **Keep string `+` and list `+` straight:** `'cal' + 'bears'` is `'calbears'`, `['cal'] + ['bears']` is `['cal', 'bears']`, and `['go ' + z for z in [x, y]]` is `['go cal', 'go bears']`.
