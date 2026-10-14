<!-- Mon, Oct 12, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 20: Attributes

This lecture is about how Python actually resolves a dot expression, and what that implies for object-oriented programming. Every interaction with an object goes through `<expression> . <name>`, and the rule for evaluating it is a two-stage lookup: first check the instance's own attributes, then fall back to the class's attributes, and if the value found in the class is a function, wrap it into a *bound method* with `self` already filled in. From this single rule, a lot follows: why `b.deposit` can be stored in a variable and still deposit into `b`'s account, why a class attribute like `interest = 0.02` is shared by every instance without being stored in any of them, why assigning `jim_account.interest = 0.08` creates a *new* instance attribute that shadows the class attribute instead of modifying it, and how composed expressions like `self.clients[email.recipient].inbox.append(email)` can be read by tracking the type of each sub-expression. The lecture also walks through two puzzle classes (`Town` and `Beach`) to exercise attribute reasoning, and builds a small email server/client system as a composition example.

---

## Key Concepts

### Dot expressions are expressions

A dot expression has the form:

```
<expression> . <name>
```

The part to the left of the dot can be *any* valid Python expression: a name, a call, an indexing operation, anything. The part to the right of the dot is **just a name**, not a complex expression. You cannot write `obj.(some_expression)` or `obj.x + y` and expect the `+ y` to be part of the lookup.

A dot expression evaluates to **the value of the attribute looked up by `<name>` in the object that is the value of `<expression>`**.

### Method calls are call expressions with a dot expression as the operator

Consider:

```python
tom_account.deposit(10)
```

The whole thing is an ordinary call expression. Its operator sub-expression happens to be the dot expression `tom_account.deposit`, and its single operand is `10`. Evaluation order is the usual one for call expressions: evaluate the operator first (which performs the attribute lookup and yields a bound method that already knows it is depositing into Tom's account), then evaluate the operands, then apply.

There is no special "method call" rule in Python. There is only "evaluate a call expression" plus "evaluate a dot expression."

### The attribute lookup procedure (memorize this)

To evaluate `<expression> . <name>`:

1. Evaluate `<expression>` to the left of the dot. This yields **the object of the dot expression**.
2. Match `<name>` against the **instance attributes** of that object. If an attribute with that name exists, its value is returned.
3. If not, `<name>` is looked up in the **class** of that object, which yields a class attribute value.
4. That value is returned **unless it is a function**, in which case a **bound method** is returned instead.

The lecture explicitly compares this to name lookup in environments: just as looking up a name searches multiple frames (local, then enclosing, then global), looking up an attribute searches multiple places (instance, then class). It is a procedure you just have to know.

### Bound methods

A **bound method** couples together a function and the object on which that method will be invoked. When the lookup in step 3 finds a function (because methods are class attributes that happen to be functions), step 4 pre-fills the `self` parameter with the object from step 1.

This means the same underlying `deposit` function can be reached two ways:

- `Account.deposit` is looked up on the *class*, so the object of the dot expression is a class, not an instance. You get a plain **function**. You must pass both arguments: `Account.deposit(tom_account, 10)`.
- `tom_account.deposit` is looked up on an *instance*. You get a **bound method** with `self` already filled in. You pass only the remaining arguments: `tom_account.deposit(10)`.

Crucially, a bound method is a value like any other. You can name it, put it in a list, pass it around, and it keeps its binding to the instance:

```python
f = a.deposit   # f is a bound method that deposits into a, forever
f(5)            # deposits 5 into a, even though 'a' appears nowhere in this line
```

### Class attributes vs instance attributes

When a `class` statement is executed, a new class is created and bound to a name in the first frame of the current environment. Then the assignment statements and `def` statements inside the class body create **class attributes**: attributes of the class itself, not of any instance. (Normally `=` and `def` create bindings in frames; inside a class body they create class attributes instead.)

The lecture notes you can write essentially *any* code inside a class statement. The slides' illustrative (deliberately useless) example was along the lines of a `nose = 'big and red'` assignment plus a `def dance(...)` definition, just to show that both forms produce class attributes: `nose` holds a string, `dance` holds a function you could call.

The useful reason to create a class attribute: it is **shared across all instances**, because it lives in one place (the class) rather than being duplicated in each instance.

```python
class Account:
    interest = 0.02            # class attribute

    def __init__(self, account_holder):
        self.balance = 0       # instance attribute
        self.holder = account_holder
```

```python
>>> tom_account = Account('Tom')
>>> jim_account = Account('Jim')
>>> tom_account.interest
0.02
>>> jim_account.interest
0.02
```

The lecture stresses: `interest` is **not** part of the instance. It is part of the class. It only *looks* like an attribute of `tom_account` because the lookup procedure falls through to the class. And class attributes are **not locked in at instance creation time**: if the class attribute changes later, every instance that doesn't have its own instance attribute of that name will see the new value.

### Attribute assignment statements

An assignment statement whose left-hand side is a dot expression is an **attribute assignment statement**. It affects the attributes of *the object of that dot expression*:

- If that object is an **instance**, the assignment sets an **instance attribute** (adding it if it did not exist).
- If that object is a **class**, the assignment sets a **class attribute**.

The asymmetry to internalize: `tom_account.interest` on the *right* of `=` performs the full two-stage lookup and can find the class attribute. But `tom_account.interest` on the *left* of `=` does **no lookup at all**: it just writes an attribute directly onto `tom_account`. It never reaches into the class.

### Shadowing

Once an instance has its own attribute with a given name, step 2 of the lookup succeeds and step 3 never runs. The instance attribute **shadows** the class attribute for that instance only. Subsequent class attribute assignments do not erase the specialization. The lecture's framing: a bank might give one customer a special interest rate, and changing the bank-wide rate shouldn't wipe out that special case.

### `getattr` and `hasattr`

In addition to dot expressions, there are built-in functions:

- `getattr(tom_account, 'balance')` does exactly the same thing as `tom_account.balance`, using the same lookup procedure.
- `hasattr(tom_account, 'balance')` tells you whether an attribute of that name exists, **either** on the instance **or** on its class.

The advantage over a dot expression is that the attribute name is now a *string*, so it can be computed at runtime.

### Attribute composition

Attribute values are themselves objects, which have their own attributes and methods. Any type of object can be the attribute value of any other type. So expressions can chain arbitrarily deep, and the way to read or write a complicated one is to **think about the type of each sub-expression**:

- `self` always has the type of the class being defined.
- `self.x.y`: `self.x` is an object (an attribute of `self`) that has an attribute `y`.
- `self.f(x).y`: `self.f(x)` is the return value of method `f`, an object that has an attribute `y`.
- `self.f(x).g(y)`: the return value of `self`'s method `f` is an object that has a method `g`.

---

## Definitions

**Attribute**: a name-value pair associated with an object. All objects have attributes; they can be looked up by name (via a dot expression or `getattr`).

**Dot expression**: an expression of the form `<expression> . <name>`, which evaluates to the value of the attribute named `<name>` in the object that is the value of `<expression>`.

**Object of a dot expression**: the value of the expression to the left of the dot; the object in which the attribute lookup is performed.

**Class**: a type; a category of objects. Classes are themselves objects, so they have attributes too.

**Instance**: an object constructed from a class.

**Instance attribute**: an attribute of an instance of a class. In `Account`, `balance` and `holder` are instance attributes because they are set on the new instance passed through `__init__` as `self`.

**Class attribute**: an attribute of the class itself, created by an assignment or `def` statement in the suite of a `class` statement (or by attribute assignment to the class). It is shared by all instances and can still be accessed from an instance via the lookup procedure.

**Method**: a class attribute that is also a function.

**Bound method**: an object that couples a function (which was a class attribute) with an instance of the class, with the function's first parameter `self` already bound to that instance. Dot expressions evaluate to bound methods for class attributes that are functions.

**Attribute assignment statement**: an assignment statement with a dot expression on its left-hand side. It sets an attribute on the object of that dot expression (an instance attribute if that object is an instance, a class attribute if it is a class).

**Shadowing (extra context: the term itself, the behavior is from lecture)**: when an instance attribute with the same name as a class attribute exists, lookups on that instance find the instance attribute and never reach the class attribute.

---

## Worked Examples

### 1. The `Account` class

```python
class Account:
    """An account has a balance and a holder.
    All accounts share a common interest rate.

    >>> a = Account('John')
    >>> a.holder
    'John'
    >>> a.deposit(100)
    100
    >>> a.withdraw(90)
    10
    >>> a.withdraw(90)
    'Insufficient funds'
    >>> a.balance
    10
    >>> a.interest
    0.02
    >>> Account.interest
    0.02
    """
    interest = 0.02

    def __init__(self, account_holder):
        self.holder = account_holder
        self.balance = 0

    def deposit(self, amount):
        """Add amount to balance."""
        self.balance = self.balance + amount
        return self.balance

    def withdraw(self, amount):
        """Subtract amount from balance if funds are available."""
        if amount > self.balance:
            return 'Insufficient funds'
        self.balance = self.balance - amount
        return self.balance
```

Tracing `a.withdraw(90)` the second time:

1. Evaluate the operator `a.withdraw`. Object of the dot expression is the instance `a`. Look for an instance attribute named `withdraw`: `a` has only `holder` and `balance`, so no.
2. Fall through to the class `Account`, which has `withdraw`. The value is a function, so a **bound method** is returned with `self` bound to `a`.
3. Evaluate the operand `90`.
4. Apply the bound method: a new frame is created for `withdraw` with `self` bound to `a` and `amount` bound to `90`.
5. `amount > self.balance` is `90 > 10`, true, so `'Insufficient funds'` is returned and `self.balance` is never modified. This is why `a.balance` is still `10` afterward.

Note the three attribute lookups going on in `withdraw`'s body alone: `self.balance` appears as a lookup (step 2 of the procedure finds the instance attribute directly) and, in `deposit`, also on the left of an `=`, where it is an attribute *assignment* onto the instance.

Also note: `a.interest` gives `0.02` via class lookup, and `Account.interest` gives `0.02` directly. These are the same value found by different paths.

### 2. Bound methods as first-class values

```python
>>> b = Account('Ada')
>>> f = b.deposit
>>> f(5)
5
>>> f(25)
30
>>> b.balance
30
```

Reasoning in environment terms: evaluating `b.deposit` performs the lookup, finds the function `deposit` in the class, and builds a bound method pairing that function with the object `b`. The name `f` in the global frame is bound to that bound method value. Calling `f(5)` creates a frame for `deposit` where `self` points at the same `Account` instance that `b` points at (one object, two paths to it), so the mutation `self.balance = self.balance + amount` mutates the object `b` names. Hence `b.balance` changes even though `b` never appears in the call.

Now with a second account:

```python
>>> a = Account('Alan')
>>> [a.deposit(n) for n in range(10)]
[0, 1, 3, 6, 10, 15, 21, 28, 36, 45]
>>> f = a.deposit
>>> [f(n) for n in range(5)]
[45, 46, 48, 51, 55]
>>> a.balance
55
```

The first list comprehension deposits `0, 1, 2, ..., 9` into `a` in order; each call returns the new balance, giving running sums (the triangular numbers). Then `f` is rebound to a bound method for `a`'s `deposit`. Depositing `0, 1, 2, 3, 4` starting from `45` gives `45, 46, 48, 51, 55`. In lecture, the important observation was that the *other* account's balance is completely unaffected: `f` is bound to one specific instance, and reassigning `f` does not create a new function, just a new pairing.

### 3. `max` with a key function (demo aside)

```python
>>> d = {1: 10, 2: 5, 3: 15, 4: 8, 5: 4}
>>> max(d.keys(), key=d.get)
3
```

`d.get` is a bound method of the dictionary `d`. Passing it as `key` means `max` compares keys of `d` by the value they map to. The largest value is `15` at key `3`, so the result is `3`. This is a nice illustration that bound methods are just values you can hand to higher-order functions. (`d.keys()` here is itself a method call on `d`.)

### 4. Discussion question: Where's Waldo? (`Town`)

> Write an expression with no quotes or `+` that evaluates to `'Waldo'`.

```python
class Town:
    """Waldo in town.

    >>> Town(1, 7).street[2]
    'Waldo'
    """
    def __init__(self, w, aldo):
        if aldo == 7:
            self.street = {self.f(w): 'Waldo'}

    def f(self, x):
        return x + 1
```

Step by step:

- `Town(1, 7)` constructs an instance and calls `__init__` with `w = 1`, `aldo = 7`.
- `aldo == 7` is true, so the body runs. Note that if you passed anything other than `7` as the second argument, `self.street` would never be created, and the attribute lookup later would fail.
- `self.f(w)` is a dot expression in the operator position: look up `f` on the instance (not found), fall through to the class `Town`, find a function, get a bound method with `self` bound to the new instance. Call it with `1`, which returns `1 + 1 = 2`. The `+` here is inside the class, so it does not violate the "no `+`" constraint on *your expression*.
- So `self.street` is assigned the dictionary `{2: 'Waldo'}`, an instance attribute.
- `Town(1, 7).street` looks up `street` on the freshly constructed instance (found as an instance attribute) and yields `{2: 'Waldo'}`.
- `[2]` indexes that dictionary with key `2`, yielding `'Waldo'`.

This is an example of the `<expression>` to the left of the dot being something complex: a whole call expression that constructs an object.

### 5. Discussion question: Where's Waldo? (`Beach`)

```python
class Beach:
    """Waldo at the beach.

    >>> Beach().walk(0).wave(0)
    'Waldo'
    """
    def __init__(self):
        sand = ['Wal', 'do']
        self.dig = sand.pop

    def walk(self, x):
        self.wave = lambda y: self.dig(x) + self.dig(y)
        return self
```

Reminder from the slides: `s.pop(k)` removes and returns the item at index `k`.

Step by step:

- `Beach()` constructs an instance. In `__init__`, `sand` is a **local** name in the `__init__` frame bound to the list `['Wal', 'do']`. It is *not* an attribute, so you can never reach the list by name from outside.
- `self.dig = sand.pop` stores a **bound method** as an instance attribute. `sand.pop` is bound to that particular list object. Even after `__init__` returns and its frame's local name `sand` goes away, the list is still reachable: the bound method `self.dig` holds a reference to it. This is the key insight of the example.
- `.walk(0)` looks up `walk` (not on the instance, found in the class, becomes a bound method) and calls it with `x = 0`.
- Inside `walk`, `self.wave` is assigned a lambda. The lambda's parent environment is the `walk` frame, so it closes over `self` and `x`. It is stored as an instance attribute. Note the lambda is not called yet, so nothing is popped yet.
- `walk` returns `self`, the same `Beach` instance. This is why chaining works: `Beach().walk(0)` evaluates to the `Beach` object, which has a `wave` attribute.
- `.wave(0)` looks up `wave` on the instance (found, it is an instance attribute holding a plain lambda, **not** a class attribute, so **no** `self` is auto-filled) and calls it with `y = 0`.
- The lambda body evaluates `self.dig(x) + self.dig(y)` left to right: `self.dig(0)` pops index `0` from the list, returning `'Wal'` and leaving the list as `['do']`. Then `self.dig(0)` pops index `0` again, returning `'do'` and leaving the list empty.
- `'Wal' + 'do'` is `'Waldo'`.

Two subtleties worth sitting with: evaluation order of the operands of `+` matters (it determines that you get `'Waldo'`, not `'doWal'`), and `self.dig` is an *instance* attribute holding a bound method of a list, so calling it does not pass the `Beach` instance.

### 6. Composition example: message delivery

The design, as described on the slides: a `Client` can compose an `Email` to send it to the `Server`, which appends it to the `inbox` of the `Client` named as the recipient. Each `Server` has a `clients` dictionary from the name of each `Client` to that `Client`, and each `Client` adds itself to `clients`.

```python
from dataclasses import dataclass

@dataclass
class Email:
    msg: str
    sender: str
    recipient: str

class Server:
    """The server's clients can send messages to each other.

    >>> s = Server()
    >>> a = Client(s, 'Alice')
    >>> b = Client(s, 'Bob')
    >>> a.compose('Hello!', 'Bob')
    >>> b.inbox[0]
    Email(msg='Hello!', sender='Alice', recipient='Bob')
    """
    def __init__(self):
        self.clients: dict[str, Client] = {}

    def send(self, email: Email) -> None:
        "Append the email to the inbox of the recipeint's client."
        self.clients[email.recipient].inbox.append(email)

class Client:
    """A client adds itself to the server when constructed."""
    def __init__(self, server: Server, name: str):
        self.server = server
        self.name = name
        self.inbox: list[Email] = []
        server.clients[name] = self

    def compose(self, message: str, recipient: str) -> None:
        "Send an email with the given message to the recipient."
        self.server.send(Email(message, self.name, recipient))
```

(The slide version names the `Email` field `recipient_name` and so writes `email.recipient_name`; the lecture code file uses `recipient`. The structure is identical.)

**Reading `send`'s body by type**, which is exactly the exercise the slides annotate sub-expression by sub-expression:

| Sub-expression | Type |
| --- | --- |
| `self` | `Server` |
| `self.clients` | `dict` (from `str` to `Client`) |
| `email.recipient` | `str` |
| `self.clients[email.recipient]` | `Client` |
| `self.clients[email.recipient].inbox` | `list[Email]` |
| `...inbox.append(email)` | mutates the list, returns `None` |

This is the payoff of the composition slide: you build up a long expression by asking, at each step, "what type do I have now, and what attribute of that type gets me closer?"

**The blank in `Client.__init__`** is `server.clients[name] = self`. Note it is *not* `self.server.clients[...]`, though that would also work at that point in the body; the slide uses the parameter `server` directly. The key move is that the instance registers *itself* (`self`) in the server's dictionary under its own name. This creates a cycle of references: the client holds the server (`self.server`), and the server holds the client (`self.clients['Alice']`). That is perfectly fine; objects can refer to each other.

**The blank in `compose`** is `Email(message, self.name, recipient)`. The sender is `self.name` because the client composing the message is the sender. The client does not deliver the email itself; it hands it to the server, which does the lookup. `compose` returns `None`, which is why the doctest line `>>> a.compose('Hello!', 'Bob')` shows no output.

Tracing the doctest:

1. `s = Server()` creates a server whose `clients` is `{}`.
2. `a = Client(s, 'Alice')` sets `a.server = s`, `a.name = 'Alice'`, `a.inbox = []`, and mutates `s.clients` to `{'Alice': a}`.
3. `b = Client(s, 'Bob')` does the same, leaving `s.clients` as `{'Alice': a, 'Bob': b}`. Both clients point at the *same* `Server` object.
4. `a.compose('Hello!', 'Bob')` builds `Email('Hello!', 'Alice', 'Bob')` and calls `s.send` on it (via `self.server.send`, a dot expression whose object is `s`).
5. `send` looks up `s.clients['Bob']`, which is the object `b`, takes its `inbox` list, and appends the email, mutating the list that `b.inbox` names.
6. `b.inbox[0]` is the email. Because `Email` is a dataclass, its automatic `__repr__` prints `Email(msg='Hello!', sender='Alice', recipient='Bob')`.

### 7. Attribute assignment: the extended `Account` trace

This is the slide-17 sequence. Picture the `Account` class holding class attributes `interest`, `withdraw`, `deposit`, `__init__`, and each instance holding only its own `balance` and `holder`.

```python
>>> jim_account = Account('Jim')
>>> tom_account = Account('Tom')
>>> tom_account.interest
0.02
>>> jim_account.interest
0.02
```

Neither instance has an `interest` attribute. Both lookups fall through to the class and find `0.02`.

```python
>>> Account.interest = 0.04
>>> tom_account.interest
0.04
>>> jim_account.interest
0.04
```

The object of the dot expression on the left of `=` is the **class**, so this is a class attribute assignment: the class's `interest` is now `0.04`. Both instances still have no `interest` of their own, so both reflect the change. As the lecture put it: instances do **not** have their class attributes locked in at creation time, even though these instances were created when the rate was `0.02`.

```python
>>> jim_account.interest = 0.08
>>> jim_account.interest
0.08
>>> tom_account.interest
0.04
```

Here the object of the dot expression on the left is an **instance**, so this is an instance attribute assignment. `jim_account` had no `interest` attribute before, so one is **added**. Now `interest` exists in two places: on `jim_account` (`0.08`) and on the class (`0.04`). Looking up `jim_account.interest` succeeds at step 2 and returns `0.08`. Looking up `tom_account.interest` fails at step 2, falls to step 3, and returns `0.04`. Jim has been *specialized*.

```python
>>> Account.interest = 0.05
>>> tom_account.interest
0.05
>>> jim_account.interest
0.08
```

Another class attribute assignment. Tom, who relies on the class, sees `0.05`. Jim, who has his own instance attribute shadowing the class, still sees `0.08`. Assigning to the class does not erase the special cases.

---

## Common Pitfalls

**Thinking assignment to `instance.attr` modifies the class attribute.** It never does. The left side of an attribute assignment performs no lookup; it always writes onto the object named to the left of the dot. `jim_account.interest = 0.08` adds an instance attribute and leaves `Account.interest` untouched.

**Forgetting that a class attribute lookup through an instance is "live."** `tom_account.interest` re-performs the lookup every time it is evaluated. If the class attribute changes, the instance's view changes.

**Confusing `Account.deposit` with `tom_account.deposit`.** The first is a plain function needing two arguments (`Account.deposit(tom_account, 10)`); the second is a bound method needing one (`tom_account.deposit(10)`). Calling `Account.deposit(10)` raises a `TypeError` about missing arguments, and calling `tom_account.deposit(tom_account, 10)` passes too many.

**Assuming bound-method behavior for *instance* attributes that hold functions.** `self` is only auto-filled when the function was found as a **class** attribute (step 3 into step 4). In `Beach`, `self.dig` and `self.wave` are instance attributes holding callables, so calling `self.wave(0)` passes `0` as the lambda's `y`, with no `Beach` instance inserted. This is a very common source of confusion.

**Losing track of mutation through a bound method.** `f = a.deposit; f(5)` changes `a`, even though `a` is not written on the second line. Likewise `self.dig = sand.pop` keeps the local list `sand` alive after `__init__` returns, and each call to `self.dig` mutates that list, so repeated calls give different results.

**Treating the right-hand side of a dot as an expression.** It is just a name. `obj.x` with `x = 'balance'` does *not* look up `balance`; it looks up the attribute literally named `x`. Use `getattr(obj, x)` when the name is computed.

**Forgetting that the left side of a dot can be arbitrary.** `Town(1, 7).street[2]` is fine. Exam questions exploit this.

**Assuming methods return something useful.** `compose` and `send` return `None`, and `list.append` returns `None`. Chaining off a `None` return value fails. Conversely `walk` returns `self` precisely to enable chaining.

**Assuming attributes exist unconditionally.** In `Town.__init__`, `self.street` is only created if `aldo == 7`. Conditional attribute creation means `Town(1, 8).street` raises an `AttributeError`.

**Thinking names assigned in a class body become local variables or globals.** Inside a `class` suite, assignment and `def` create class attributes, not frame bindings.

---

## Likely Exam Points

### 1. Trace a class-vs-instance attribute assignment sequence

Nearly every 61A exam has one of these. Track two tables: the class's attributes and each instance's attributes. For each lookup, ask: does the instance have it? If not, go to the class.

**Practice.** Given `class Account` with `interest = 0.02` as above:

```python
a = Account('A')
b = Account('B')
a.interest = 0.03
Account.interest = 0.07
b.interest = Account.interest + 0.01
Account.interest = 0.09
```

What are `a.interest`, `b.interest`, and `Account.interest`?

**Answer.** `a.interest` is `0.03` (instance attribute set on line 3, shadows all later class changes). `b.interest` is `0.08`: line 5's right side evaluates `Account.interest` (which is `0.07` at that moment) plus `0.01`, and the left side creates an instance attribute on `b`; line 6 cannot affect it. `Account.interest` is `0.09`.

### 2. Function vs bound method

**Practice.** Using the lecture's `Account`, which of these successfully deposits `10` into `tom_account`, and what happens with the others?

```python
(a) tom_account.deposit(10)
(b) Account.deposit(tom_account, 10)
(c) Account.deposit(10)
(d) tom_account.deposit(tom_account, 10)
```

**Answer.** (a) works: `tom_account.deposit` is a bound method, `self` already filled in. (b) works: looking up `deposit` on the class gives a plain function, so you supply `self` yourself. (c) is a `TypeError`: the function is missing its `amount` argument (`10` binds to `self`, then `self.balance` would also fail). (d) is a `TypeError`: the bound method already has `self`, so it receives three arguments for two parameters.

### 3. Writing an expression that evaluates to a target value ("Where's Waldo")

These test whether you can chain dot expressions, calls, and indexing, and whether you notice what each method returns.

**Practice.** With the lecture's `Beach` class, what does `Beach().walk(1).wave(0)` evaluate to, and what does `b = Beach(); b.walk(0); b.wave(0); b.wave(0)` do on the last line?

**Answer.** `Beach().walk(1).wave(0)`: the lambda evaluates `self.dig(1)` first, popping index `1` from `['Wal', 'do']`, which returns `'do'` and leaves `['Wal']`; then `self.dig(0)` pops index `0`, returning `'Wal'`. The result is `'dowal'`... more precisely `'do' + 'Wal'` is `'doWal'`. For the second part: `b.walk(0)` sets `b.wave`; the first `b.wave(0)` returns `'Waldo'` and empties the list; the second `b.wave(0)` raises an `IndexError` (pop from empty list), because the list is shared state that was consumed.

### 4. Fill in a blank in a composed-object program

**Practice.** In `Server.send`, suppose you wrote `self.clients[email.sender].inbox.append(email)` instead. What would the doctest produce?

**Answer.** The email would be appended to `s.clients['Alice'].inbox`, that is, to the sender's own inbox, so `a.inbox[0]` would be the email and `b.inbox` would stay empty. `b.inbox[0]` would raise an `IndexError`. The lookup key must be the recipient's name because `clients` maps names to `Client` objects and the spec says to deliver to the recipient.

### 5. Name the type of each sub-expression

**Practice.** In `Client.compose`, give the type of `self`, `self.server`, `self.name`, `Email(message, self.name, recipient)`, and `self.server.send(...)`.

**Answer.** `self` is a `Client`; `self.server` is a `Server`; `self.name` is a `str`; `Email(message, self.name, recipient)` is an `Email`; `self.server.send(...)` evaluates to `None` (its annotation is `-> None`).

### 6. `getattr` / `hasattr`

**Practice.** For `tom_account = Account('Tom')` with no instance `interest` attribute, what are `getattr(tom_account, 'interest')` and `hasattr(tom_account, 'interest')`? What about `hasattr(tom_account, 'rate')`?

**Answer.** `getattr(tom_account, 'interest')` is `0.02`, found via the class, exactly as a dot expression would. `hasattr(tom_account, 'interest')` is `True`, because `hasattr` reports an attribute found on either the instance or its class. `hasattr(tom_account, 'rate')` is `False`.

### 7. Aliasing a bound method

**Practice.** What does this print?

```python
a = Account('A')
b = Account('B')
f = a.deposit
g = b.deposit
f(10)
g(100)
f(1)
print(a.balance, b.balance)
```

**Answer.** `11 100`. `f` is permanently bound to `a` and `g` to `b`; deposits of `10` and `1` go to `a`, and `100` goes to `b`.

---

## Summary

- A dot expression is `<expression> . <name>`: the left part can be any expression, the right part is just a name.
- A method call is an ordinary call expression whose operator is a dot expression; the dot expression is evaluated first.
- Attribute lookup: (1) evaluate the left side to get the object, (2) check the object's instance attributes, (3) otherwise check its class, (4) if the class attribute found is a function, return a bound method with `self` pre-filled.
- A **bound method** pairs a function with an instance. `instance.method` is a bound method; `Class.method` is a plain function requiring `self` explicitly.
- Bound methods are first-class values: `f = a.deposit` keeps depositing into `a` wherever `f` goes.
- Executing a `class` statement creates a class and binds assignments and `def`s in its suite as **class attributes**, not frame bindings.
- Class attributes are shared by all instances because they live in the class, not in each instance; they are looked up live, so later changes are visible to instances.
- An assignment with a dot expression on the left is an **attribute assignment**: instance object on the left gives an instance attribute, class on the left gives a class attribute. No lookup happens on the left side.
- An instance attribute **shadows** a same-named class attribute for that instance only; later class attribute assignments do not remove the specialization.
- `getattr(obj, 'name')` is equivalent to `obj.name`; `hasattr(obj, 'name')` checks instance and class.
- Attribute values are objects with their own attributes, so expressions compose: read `self.x.y` and `self.f(x).g(y)` by tracking the type of each sub-expression.
- Lecture examples to review: `Account` (class attribute `interest`, bound-method aliasing), `Town` (`Town(1, 7).street[2]`), `Beach` (`Beach().walk(0).wave(0)`, bound method of a list stored as an instance attribute, `return self` for chaining), and `Server`/`Client`/`Email` (`self.clients[email.recipient].inbox.append(email)`).
