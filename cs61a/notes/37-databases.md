<!-- Mon, Nov 23, 2026 | sources: slides + code (no transcript available) -->
# Lecture 37: Databases

This lecture completes CS 61A's SQL unit by moving from reading data to creating and changing it, then connects SQL back to Python. You learn the three statements that define a table's existence and contents (`CREATE TABLE`, `DROP TABLE`, and the trio `INSERT`/`UPDATE`/`DELETE`), then see how the `sqlite3` module lets a Python program hold a live database connection, run SQL strings, and read results back as Python tuples. That combination raises a security problem: if you build SQL by concatenating user input into a command string, the user can write SQL of their own, which is the SQL injection attack (the xkcd "Little Bobby Tables" strip). The fix is parameterized queries with `?` placeholders, which SQLite substitutes after it has already planned the query. The lecture closes with Blackjack, a program where the SQL table is not a report but the game's actual state: every card dealt is a row, scoring is a `SELECT` with a `WHERE` filter, and clearing the table between hands is an `UPDATE`.

## Key Concepts

### Aggregation review

The lecture opens with a review of aggregation from the previous lecture. Aggregation is the part of SQL that collapses many rows into one value: `MAX`, `MIN`, `SUM`, `COUNT`, `AVG`, combined with `GROUP BY` to produce one result row per group and `HAVING` to filter those groups. The key distinction worth keeping straight is that `WHERE` filters individual rows before grouping, while `HAVING` filters groups after aggregation. Nothing new was introduced here; it is the hinge between the two halves of the SQL unit. (extra context: that `WHERE` versus `HAVING` split is the standard SQL rule, stated here because the slide text is only a section header.)

### Tables have to be created before they can be queried

Every SQL statement you have written so far assumed the table already existed. `CREATE TABLE` is how a table comes into being. In SQLite's dialect, the minimal form is just a name plus a list of column names:

```sql
CREATE TABLE users (id, first, last, email, role);
```

Notice what is missing compared to other databases you may have seen: no types. SQLite is unusual in being dynamically typed per value, so a column declaration can be a bare name. This fits CS 61A's purpose, since the course cares about relational structure rather than storage engineering. You may optionally attach a column constraint:

```sql
CREATE TABLE users (id UNIQUE, first, last, email, role);
```

`UNIQUE` on `id` tells the database to reject any insert whose `id` duplicates an existing row's `id`. This is the first time in the course that the database itself, rather than your program, enforces a rule about the data. That is the point of a constraint: the invariant lives next to the data, so no code path can violate it, not even a buggy one.

The slides show the syntax as railroad diagrams for `CREATE TABLE expression`, `column-def`, and `column-constraint`. The diagrams themselves were not extractable from the PDF, but the structure they encode is visible in the examples: a table definition is a name followed by a parenthesized, comma-separated list of column definitions, and each column definition is a column name optionally followed by constraints.

### Dropping tables, and the `IF EXISTS` / `IF NOT EXISTS` guards

`DROP TABLE` removes a table and all its rows. Two variants matter in practice, and both appear in the lecture's own code:

- `DROP TABLE IF EXISTS cards` succeeds whether or not `cards` exists. Without `IF EXISTS`, dropping a table that is not there is an error.
- `CREATE TABLE IF NOT EXISTS users (...)` succeeds whether or not `users` exists. Without `IF NOT EXISTS`, creating a table that already exists is an error.

These guards exist because a database file persists between program runs. A script that runs twice would crash on the second run without them. The Blackjack code uses this pattern deliberately: it drops `cards` and recreates it at startup so that each run of the program starts from an empty table rather than inheriting cards from the last session.

### The three statements that modify rows

Once a table exists, three statements change what is in it:

**`INSERT`** adds a row. The positional form supplies one value per column, in declaration order:

```sql
INSERT INTO users VALUES
  (0, "Kay", "Ousterhout", "kayo@berkeley.edu", "Instructor");
```

The slide's framing is explicit: "To insert a row into a users table with 5 columns." The count has to match, and the order has to match the order in `CREATE TABLE`. There is no name-based safety net in this form, so if you reorder your columns later, every positional `INSERT` silently puts data in the wrong place.

**`UPDATE`** changes existing rows. The slide's description is the one to memorize: "Update sets all entries in certain columns to new values, just for some subset of rows." Three pieces: which table, which columns get which new values (`SET`), and which rows are affected (`WHERE`). The last piece is optional, and that is exactly where it is dangerous. `UPDATE` with no `WHERE` clause rewrites the named columns in every row of the table. The lecture's Blackjack code does this on purpose:

```sql
UPDATE cards SET place="Discard";
```

That is "move every card in the table to the discard pile," and because there is no `WHERE`, it hits all of them, which is the intent when a hand ends.

**`DELETE`** removes rows: "Delete removes some or all rows from a table." The same asymmetry applies. `DELETE FROM t WHERE ...` removes matching rows; `DELETE FROM t` removes all of them. The difference between `DELETE FROM t` and `DROP TABLE t` is that `DELETE` leaves the table in place with zero rows, while `DROP` removes the table definition itself, so subsequent queries against it error out.

### Python can hold a database connection

The second half of the lecture connects SQL to Python. The pattern is four steps:

```python
import sqlite3

db = sqlite3.Connection('users.db')

db.execute('CREATE TABLE IF NOT EXISTS users (id UNIQUE, first, last, email, role);')

cmd = f"INSERT INTO users VALUES (0, 'Kay', 'Ousterhout', 'kayo@berkeley.edu', 'Instructor');"
db.executescript(cmd);

users = db.execute("SELECT * from users").fetchall()
for user in users:
    print(user)
```

Reading this in terms of the environment model: `db` is bound in the global frame to a `Connection` object. That object is not a value you compute with; it is a handle on an external resource, a file on disk named `users.db`. Every `db.execute(...)` call sends a string to that external engine and the engine does the work. In box-and-pointer terms, `db` is a single arrow pointing at one object, and all of the database's actual contents live outside your Python heap entirely, in the file. If your program exits and restarts, `db` is gone but `users.db` is not, which is the whole reason `IF NOT EXISTS` is needed on line 5.

Two methods appear, and the lecture uses them differently:

- **`db.execute(sql)`** runs one statement and returns a cursor object. A cursor is not the result; it is something you pull results out of. `.fetchall()` on a cursor gives you a list, and each element is a Python tuple holding one row, with one element per selected column. That is why `for user in users: print(user)` prints tuples rather than, say, dictionaries. In the 37.py code, the cursor is also iterated directly in a comprehension, which works because cursors are iterable.
- **`db.executescript(sql)`** runs a string that may contain multiple semicolon-separated statements. That extra power is precisely what makes it the vulnerable call in the injection demo.

### SQL injection

The attack slide is the conceptual center of the lecture. Here is the vulnerable pattern:

```python
name = "Robert'); DROP TABLE Students; --"
cmd = "INSERT INTO Students VALUES ('" + name + "');"
db.executescript(cmd)
```

Trace the string concatenation by hand. The template is `INSERT INTO Students VALUES ('` + name + `');`. Splice in that value of `name` and SQLite receives:

```sql
INSERT INTO Students VALUES ('Robert'); DROP TABLE Students; --');
```

Read what the engine sees. The `');` in the attacker's string closes the intended `VALUES (` and terminates the `INSERT` statement. What follows the semicolon is a second, complete statement: `DROP TABLE Students;`. Then `--` begins a SQL line comment, which neutralizes the leftover `');` that the original template contributed. The result is a syntactically valid two-statement script, so the insert succeeds and the table is destroyed. The programmer thought they were passing data; the attacker passed code. (This is the xkcd 327 "Little Bobby Tables" comic the slide links to.)

The fix is a parameterized query:

```python
db.execute("INSERT INTO Students VALUES (?)", [name])
```

The slide gives the mechanism in two annotations. First, "SQLite makes a query plan before substitution happens": the engine parses and plans `INSERT INTO Students VALUES (?)` as a complete statement in its own right, and that statement structure is fixed before any of your data is looked at. Second, "SQLite gets parameters separately": the value of `name` arrives through a separate channel, as an element of the list, not as characters in the SQL text. There is no step at which the attacker's semicolon could be read as a statement separator, because by the time the value is consumed, the engine has already decided it is reading one value for one placeholder. The injection does not fail because the characters were escaped or filtered; it fails because the data never passed through the parser at all.

Note that the fix also switches from `executescript` to `execute`. `execute` runs a single statement, so even a parser-level attempt at a second statement has nowhere to go.

### The Blackjack demo: a database as program state

The last demo is the payoff, and it is worth understanding why the lecture chose Blackjack specifically. The table is:

```sql
CREATE TABLE cards(card, place, hand);
```

Three columns: which card, where it currently is (`"Player"`, `"Dealer"`, or `"Discard"`), and which hand number it belongs to. That schema turns the whole game into table manipulation. Dealing a card is an `INSERT`. Computing a score is a `SELECT` with a `WHERE place = ?` filter. Ending a hand is a single `UPDATE`. The game logic in Python never maintains a list of the player's cards, because the table is the list.

This matters because of the dealer's hidden card, which is the one genuinely clever part of the design. In real Blackjack the dealer's second card is face down. The program models "face down" by simply not inserting it:

```python
hidden = deck.pop()
```

The card is popped off the deck (so it cannot be dealt to anyone else) and held in a plain Python local variable. It is not in the table, so `score(dealer)` cannot see it and neither can the player, who in the demo is looking at the table's contents. When the player stands, the program plays it: `play(hidden, dealer, hand)` inserts it, and from that moment it counts toward the dealer's score. Being in the table is exactly what "visible and in play" means in this program. The docstring on `play` says so: "Play a card so that the player can see it."

### Ace handling

One piece of non-SQL logic deserves attention because it is a neat use of a conditional:

```python
points = {'A': 1, 'J': 10, 'Q': 10, 'K': 10}
points.update({n: n for n in range(2, 11)})

def hand_score(hand):
    """Total score for a hand."""
    total = sum([points[card] for card in hand])
    if total <= 11 and 'A' in hand:
        return total + 10
    return total
```

An ace is worth 1 or 11. Rather than branching over every possible ace assignment, the code scores every ace as 1, then adds 10 back exactly once if doing so cannot bust. The condition `total <= 11` guarantees the upgraded total is at most 21, and `'A' in hand` guarantees there is an ace to upgrade. Only one ace is ever upgraded, which is correct: two aces counted as 11 each would be 22, a bust, so at most one ace can take the high value in any legal hand.

The dictionary construction is also a small point worth noting. `points` starts with the four letter cards, then `.update` merges in a dict comprehension mapping each number 2 through 10 to itself. `range(2, 11)` stops at 10, so the numeric cards are 2 through 10 inclusive. Those 13 keys are exactly a suit, which is why `list(points.keys()) * 4` builds a 52-card deck.

### `db.commit()`

The `play` function calls `db.commit()` after each insert. A commit makes pending changes permanent in the database file. Without it, changes live in an uncommitted transaction and would be lost if the program exited. In this program the commit matters because the demo's whole point is that someone else (the player, looking at the database) can see the cards, and uncommitted writes are not durable. (extra context: the broader transaction model, rollback, and isolation are beyond what this lecture covers; `commit` here is just "write it down for real.")

## Definitions

**Aggregation**: SQL features that compute a single value from many rows, such as `MAX`, `MIN`, `SUM`, `COUNT`, and `AVG`, typically used with `GROUP BY` and `HAVING`.

**`CREATE TABLE`**: a SQL statement that defines a new table by naming it and listing its column definitions in parentheses.

**Column definition (`column-def`)**: a column name, optionally followed by one or more column constraints. In SQLite, a type is optional, so a bare name is a complete column definition.

**Column constraint (`column-constraint`)**: a restriction attached to a column in `CREATE TABLE`, enforced by the database itself. `UNIQUE` is the example given: no two rows may share a value in that column.

**`IF NOT EXISTS`**: a clause on `CREATE TABLE` that makes the statement succeed silently when the table already exists, rather than raising an error.

**`IF EXISTS`**: a clause on `DROP TABLE` that makes the statement succeed silently when the table does not exist.

**`DROP TABLE`**: a SQL statement that deletes a table along with all its rows and its definition. Later references to that table are errors.

**`INSERT INTO ... VALUES (...)`**: a SQL statement that adds a row, supplying one value per column in the order the columns were declared.

**`UPDATE`**: a SQL statement that sets specified columns to new values in a subset of rows chosen by its `WHERE` clause. With no `WHERE` clause, it affects every row.

**`DELETE FROM`**: a SQL statement that removes some rows (with `WHERE`) or all rows (without `WHERE`) from a table, leaving the table itself in existence.

**`sqlite3`**: the Python standard library module providing access to SQLite databases.

**`sqlite3.Connection(path)`**: a constructor returning a connection object that represents an open SQLite database stored at the given file path.

**`db.execute(sql)`** / **`db.execute(sql, params)`**: a method that runs one SQL statement and returns a cursor. The optional second argument supplies values for `?` placeholders as a sequence.

**`db.executescript(sql)`**: a method that runs a string containing one or more semicolon-separated SQL statements.

**`.fetchall()`**: a cursor method returning a list of all result rows, each row represented as a Python tuple of column values.

**`db.commit()`**: a method that makes pending changes permanent in the database file.

**Parameterized query**: a SQL statement containing `?` placeholders, with the actual values passed separately so that the engine plans the statement before substituting data. This is the defense against SQL injection.

**SQL injection attack**: an attack in which a program builds a SQL command by concatenating untrusted input into a command string, allowing the attacker to supply characters that the engine parses as additional SQL rather than as data.

## Worked Examples

### Example 1: creating, filling, and reading a table from Python

This is the lecture's own `users.db` example. Taken line by line:

```python
import sqlite3

db = sqlite3.Connection('users.db')
```

`sqlite3.Connection('users.db')` opens the file `users.db` in the current directory, creating it if it does not exist, and returns a connection object. In the environment model, the global frame now binds `db` to that object. The object is a handle: all the real data lives in the file, not in Python's memory.

```python
db.execute('CREATE TABLE IF NOT EXISTS users (id UNIQUE, first, last, email, role);')
```

This defines a five-column table. `id UNIQUE` is a column definition with one constraint; the other four are bare names. `IF NOT EXISTS` is doing real work here: because `users.db` persists on disk, the second time you run this script the table is already there, and a plain `CREATE TABLE` would raise an error. With the guard, the statement is a no-op on reruns.

```python
cmd = f"INSERT INTO users VALUES (0, 'Kay', 'Ousterhout', 'kayo@berkeley.edu', 'Instructor');"
db.executescript(cmd);
```

Five values for five columns, in declaration order, so `0` lands in `id` and `'Instructor'` lands in `role`. The string uses single quotes for the SQL string literals so they do not collide with the Python double quotes. Note that this is written as an f-string even though there are no `{}` substitutions in it, which hints at the direction the demo was headed: the natural next step is to interpolate variables, and that is exactly the pattern the next slide shows to be dangerous.

Also note: if you run this script twice with the same `id` of `0`, the `UNIQUE` constraint on `id` rejects the second insert. The constraint is doing the job the lecture introduced it for.

```python
users = db.execute("SELECT * from users").fetchall()
for user in users:
    print(user)
```

`db.execute` returns a cursor. `.fetchall()` drains it into a Python list. Each element of that list is a tuple with five elements, matching the five selected columns, so the loop prints something shaped like `(0, 'Kay', 'Ousterhout', 'kayo@berkeley.edu', 'Instructor')`. This is the bridge worth internalizing: SQL rows arrive in Python as tuples, which means you can unpack them, and the Blackjack code does exactly that.

### Example 2: tracing the SQL injection string

```python
name = "Robert'); DROP TABLE Students; --"
cmd = "INSERT INTO Students VALUES ('" + name + "');"
db.executescript(cmd)
```

Build `cmd` by hand, piece by piece:

| Piece | Characters |
|---|---|
| literal prefix | `INSERT INTO Students VALUES ('` |
| `name` | `Robert'); DROP TABLE Students; --` |
| literal suffix | `');` |

Concatenated:

```sql
INSERT INTO Students VALUES ('Robert'); DROP TABLE Students; --');
```

Now read it as SQLite's parser does:

1. `INSERT INTO Students VALUES ('Robert')` is a complete statement. The closing `'` and `)` came from the attacker's input, not from your template.
2. `;` ends that statement.
3. `DROP TABLE Students` is a second complete statement, and the trailing `;` ends it.
4. `--` starts a comment that runs to end of line, swallowing the `');` your template appended. Without the `--`, that leftover text would be a syntax error and SQLite might reject the whole script, so the comment is what makes the attack clean.

`executescript` is willing to run multiple statements, so both run. The table is gone.

The parameterized version:

```python
db.execute("INSERT INTO Students VALUES (?)", [name])
```

Here the SQL string is a fixed constant. SQLite parses and plans `INSERT INTO Students VALUES (?)`, determining that it is one insert statement taking one value, and that plan is complete before it reads your list. Then it binds the single element of `[name]` into the placeholder as a value. The string `"Robert'); DROP TABLE Students; --"` is stored verbatim as the row's data, semicolons and all. Nothing is parsed, so nothing is executed. The slide's two annotations describe exactly these two facts: the plan comes first, and the parameters arrive separately.

### Example 3: `score` and `bust`, reading rows back as tuples

```python
def score(who):
    """Compute the hand score for the player or dealer."""
    cards = sql('SELECT * from cards where place = ?;', [who])
    return hand_score([card for card, place, hand in cards.fetchall()])

def bust(who):
    """Check if the player or dealer went bust."""
    return score(who) > 21
```

First, `sql` is just an alias: `sql = db.execute` binds the name `sql` to the bound method object, so `sql(...)` and `db.execute(...)` are the same call. This is a higher-order-functions idea reused for convenience: a method is a value, and binding it to a shorter name costs nothing.

Now trace `score("Player")`:

1. `sql('SELECT * from cards where place = ?;', ["Player"])` runs a parameterized query. The `?` is filled with `"Player"`, safely. This matters even here, where `who` comes from a module constant rather than user input: the lecture uses the safe form by default.
2. `.fetchall()` returns a list of tuples, one tuple per matching row, each of the form `(card, place, hand)` because `SELECT *` selects all three columns.
3. The comprehension `[card for card, place, hand in cards.fetchall()]` unpacks each tuple into three names and keeps only the first. The result is a plain list of card values, like `['K', 7, 'A']`. This is tuple unpacking inside a comprehension's `for` clause, exactly the pattern from earlier in the course.
4. That list is passed to `hand_score`, which sums `points[card]` for each card and applies the ace upgrade.

So the chain is: SQL filters, Python unpacks, Python scores. The important design point is that `score` holds no state. It asks the database every time it is called. That is why dealing a card changes the score with no bookkeeping anywhere: the next `score` call simply sees one more row.

`bust` then reduces to a one-line comparison, and because it calls `score`, it is also reading live from the table.

### Example 4: `play_hand`, step by step

```python
def play_hand(deck, hand):
    """Play a hand of Blackjack."""
    play(deck.pop(), player, hand)
    play(deck.pop(), dealer, hand)
    play(deck.pop(), player, hand)
    hidden = deck.pop()

    while 'y' in input("Hit? ").lower():
        play(deck.pop(), player, hand)
        if bust(player):
            print(player, "went bust!")
            return

    play(hidden, dealer, hand)

    while score(dealer) < 17:
        play(deck.pop(), dealer, hand)
        if bust(dealer):
            print(dealer, "went bust!")
            return

    print(player, score(player), "and", dealer, score(dealer))
```

**The deal.** Four `deck.pop()` calls, in the alternating order of a real deal: player, dealer, player, dealer. `deck` is a Python list and `.pop()` with no argument removes and returns the last element, so the deck shrinks by four. The first three cards go through `play`, which inserts them into the table. The fourth does not: it is bound to the local name `hidden`, held outside the table. At this moment the table has three rows for this hand and the dealer's visible total reflects only one card.

**The player's loop.** `input("Hit? ")` prompts, `.lower()` normalizes the response, and `'y' in ...` tests for a `y` anywhere in it, so `"y"`, `"yes"`, and `"Yeah"` all mean hit. Each iteration pops a card, inserts it for the player, and calls `bust(player)`, which re-queries the table. If the player exceeds 21, the function prints and returns immediately, so the dealer never plays and the hidden card is never revealed. Note the consequence: on a player bust, `hidden` is simply discarded along with the local frame. The card was popped off the deck but never entered the table.

**Revealing.** If the loop exits normally (the player stood), `play(hidden, dealer, hand)` inserts the held card. The dealer's score jumps to include it. In environment-model terms, `hidden`'s value was living in `play_hand`'s local frame the whole time; this line is what moves it into the external database.

**The dealer's loop.** The rule is mechanical: hit while the total is under 17. `score(dealer) < 17` re-queries on every check, so the loop condition reflects the table's current contents. If the dealer busts, print and return. Otherwise the loop ends with the dealer on 17 or more.

**The showdown.** The final `print` reports both scores. Observe what the function does not do: it never decides a winner. The lecture's program stops at reporting the two totals.

### Example 5: the game loop and `UPDATE` as a reset

```python
deck = list(points.keys()) * 4
random.shuffle(deck)
hand = 0
while len(deck) > 10:
    print('\nDealing...')
    play_hand(deck, hand)
    sql('UPDATE cards SET place="Discard";')
    hand += 1;
```

`points.keys()` has 13 keys (`'A'`, `'J'`, `'Q'`, `'K'`, and 2 through 10), so `list(...) * 4` builds a 52-element list. Note this is a deck of card *ranks*, not distinct suited cards: there are four copies of each rank, which is all the game needs since suits never affect scoring. `random.shuffle(deck)` permutes it in place, which is why the function has no return value to assign.

The loop plays hands until fewer than 11 cards remain, a cutoff that guarantees enough cards for another hand without running the deck dry mid-deal.

The `UPDATE` after each hand is the lecture's deliberate demonstration of an unrestricted update. `UPDATE cards SET place="Discard";` has no `WHERE` clause, so it rewrites `place` in *every* row of the table. The effect is that all previously dealt cards become discards, which means the next hand's `score("Player")` query, filtering on `place = "Player"`, matches zero old rows. The table keeps its full history (nothing is deleted, and the `hand` column records which hand each card belonged to), but the `place` column is what the queries filter on, so discarded cards are invisible to scoring.

This is a nice illustration of a design choice: rather than `DELETE FROM cards`, which would throw away the history, the program marks rows as discarded. The data stays queryable, which is exactly what the `hand` column is for. (extra context: with the `hand` column retained, you could write an aggregation query over past hands, which ties back to the aggregation review at the start of the lecture. The lecture code does not do this.)

## Common Pitfalls

**Confusing `DELETE` with `DROP TABLE`.** `DELETE FROM t` empties the table but leaves it defined, so queries still work and return nothing. `DROP TABLE t` removes the definition, so any later query on `t` is an error. Only one of these can be followed by an `INSERT`.

**Forgetting `WHERE` on `UPDATE` or `DELETE`.** The clause is optional, and omitting it silently applies the change to every row. The Blackjack code omits it on purpose; in most other situations the omission is a bug. When you read an `UPDATE` on an exam, look for the `WHERE` first.

**Assuming `CREATE TABLE` requires types.** It does not in SQLite. A bare column name is a complete column definition. Writing types is harmless but is not what the lecture's examples do.

**Forgetting `IF NOT EXISTS` / `IF EXISTS` in a script that runs more than once.** A database file outlives the process. The second run of a script that plainly creates a table will fail.

**Mismatching value count or order in a positional `INSERT`.** `INSERT INTO users VALUES (...)` must supply exactly one value per column, in declaration order. There is no name checking, so a wrong order inserts wrong-but-valid data rather than raising an error.

**Building SQL with `+` or f-strings from any value you did not write yourself.** This is the injection vulnerability. The habit to build is: if a value is a variable, it goes in the parameter list with a `?`, not in the string.

**Thinking escaping or sanitizing is equivalent to parameterizing.** Hand-filtering quotes and semicolons is a blacklist, and blacklists leak. Parameterization is structurally different: the value never reaches the parser. The slide's phrasing is the one to remember, "SQLite makes a query plan before substitution happens."

**Using `executescript` where `execute` suffices.** `executescript` accepts multiple statements, which is exactly the capability an injected `;` exploits. Use `execute` with parameters for anything involving a variable value.

**Treating a cursor as if it were the rows.** `db.execute("SELECT ...")` returns a cursor, not a list. You need `.fetchall()` (or to iterate it) to get rows. Also, a cursor is consumed as you read it, so calling `.fetchall()` a second time on the same cursor yields an empty list.

**Forgetting that query results are tuples.** A `SELECT *` on a three-column table yields three-element tuples. You unpack them (`for card, place, hand in ...`) or index them; you do not access columns by name.

**Thinking the hidden card is in the table.** It is a Python local variable until `play(hidden, dealer, hand)` runs. Before that line, no query can see it, which is the entire mechanism for a face-down card. Asking "what does `score(dealer)` return right after the deal" has a different answer before and after the reveal.

**Mis-reading the ace rule.** The code scores every ace as 1, then adds 10 once if `total <= 11 and 'A' in hand`. It does not add 10 per ace, and it does not add 10 when the total is 12 or more. A hand of `['A', 'A']` sums to 2, qualifies for one upgrade, and scores 12, not 22.

**Forgetting `db.commit()`.** Changes made but not committed are not durable. The lecture's `play` commits after every insert.

## Likely Exam Points

### 1. Writing a `CREATE TABLE` statement

Expect to produce a table definition from a description, possibly with a `UNIQUE` constraint.

**Q.** Write a SQL statement creating a table `courses` with columns `code`, `title`, and `units`, where no two rows may share a `code`, and the statement must not error if `courses` already exists.

**A.**
```sql
CREATE TABLE IF NOT EXISTS courses (code UNIQUE, title, units);
```

### 2. Predicting the effect of an `UPDATE` without a `WHERE`

This is the single most likely trap, because the lecture's own code uses the dangerous form deliberately.

**Q.** A table `cards` holds the rows `('K', 'Player', 0)`, `(7, 'Dealer', 0)`, and `('A', 'Player', 0)`. What does `UPDATE cards SET place="Discard";` leave in the table, and how many rows does `SELECT * FROM cards WHERE place = 'Player';` then return?

**A.** All three rows, with `place` changed to `'Discard'`: `('K', 'Discard', 0)`, `(7, 'Discard', 0)`, `('A', 'Discard', 0)`. The `card` and `hand` columns are untouched; only `place` is set. The subsequent `SELECT` returns zero rows. No rows were deleted, so `SELECT * FROM cards;` still returns three.

### 3. Tracing a SQL injection string

Expect to be given a concatenation and a malicious value and asked what SQLite receives.

**Q.** Given `template = "DELETE FROM Logs WHERE user = '" + name + "';"` and `name = "x'; DROP TABLE Logs; --"`, write the exact string passed to `executescript`, and explain in one sentence why `DROP TABLE Logs` executes.

**A.** The string is:
```sql
DELETE FROM Logs WHERE user = 'x'; DROP TABLE Logs; --';
```
The attacker's `'` closes the string literal and the `;` terminates the `DELETE`, so `DROP TABLE Logs` is parsed as a second complete statement, which `executescript` is willing to run; the trailing `--` comments out the template's leftover `';` so there is no syntax error.

### 4. Fixing a vulnerable call

**Q.** Rewrite `db.executescript("INSERT INTO Logs VALUES ('" + msg + "');")` safely, and state why the rewrite works.

**A.**
```python
db.execute("INSERT INTO Logs VALUES (?)", [msg])
```
SQLite parses and plans the statement, which contains no data at all, before it looks at the parameter list, and the value of `msg` is delivered separately rather than as SQL text. There is no point at which a semicolon in `msg` could be read as a statement separator. Switching from `executescript` to `execute` additionally restricts the call to one statement.

### 5. Computing a hand score, including an ace

**Q.** What does `hand_score(['A', 5, 'K'])` return? What about `hand_score(['A', 5])` and `hand_score(['A', 'A', 9])`?

**A.**
- `hand_score(['A', 5, 'K'])`: raw sum is `1 + 5 + 10 = 16`. Since `16 > 11`, no upgrade. Returns `16`.
- `hand_score(['A', 5])`: raw sum is `1 + 5 = 6`. Since `6 <= 11` and there is an ace, add 10. Returns `16`.
- `hand_score(['A', 'A', 9])`: raw sum is `1 + 1 + 9 = 11`. Since `11 <= 11` and there is an ace, add 10 once. Returns `21`.

### 6. Reasoning about the hidden card

**Q.** Immediately after the four deal lines in `play_hand` run (before the player's `while` loop), how many rows does the `cards` table contain for this hand, and whose cards are they?

**A.** Three rows: two for `"Player"` and one for `"Dealer"`. The fourth popped card is bound to the local variable `hidden` and is not inserted, so it is absent from the table and invisible to `score("Dealer")` until `play(hidden, dealer, hand)` runs after the player stands.

### 7. Unpacking query results in Python

**Q.** `score` contains `[card for card, place, hand in cards.fetchall()]`. Why does the `for` clause bind three names when the comprehension only produces one value per row?

**A.** Because `SELECT *` on a three-column table returns rows as three-element tuples, so each element of `fetchall()`'s list must be unpacked into three names. Only `card` is used in the result expression; `place` and `hand` are bound and discarded. Writing `[card for card in cards.fetchall()]` would instead build a list of whole tuples, and `points[card]` in `hand_score` would then fail.

### 8. Distinguishing `execute` from `executescript`

**Q.** State one capability `executescript` has that `execute` does not, and explain why that capability is a security concern.

**A.** `executescript` runs a string containing multiple semicolon-separated statements, while `execute` runs one. That is what makes string-concatenated SQL exploitable: an injected `;` creates a second statement that `executescript` will happily run.

### 9. `DELETE` versus `DROP TABLE`

**Q.** After `DELETE FROM cards;`, does `SELECT * FROM cards;` succeed? After `DROP TABLE cards;`?

**A.** After `DELETE`, yes: the table still exists and the query returns zero rows. After `DROP TABLE`, no: the table no longer exists and the query is an error (unless the table is recreated).

## Summary

- Aggregation (`MAX`, `MIN`, `SUM`, `COUNT`, `AVG`, with `GROUP BY` and `HAVING`) was reviewed as the bridge from the previous lecture.
- `CREATE TABLE name (col, col, ...)` defines a table. In SQLite, types are optional; a bare column name is a valid column definition. Constraints like `UNIQUE` are enforced by the database, not your code.
- `IF NOT EXISTS` on `CREATE TABLE` and `IF EXISTS` on `DROP TABLE` make scripts rerunnable against a persistent database file.
- `DROP TABLE` removes the table and its definition. `DELETE FROM t` removes rows but keeps the table.
- `INSERT INTO t VALUES (...)` adds a row, one value per column in declaration order.
- `UPDATE t SET col = val WHERE ...` rewrites columns in matching rows. With no `WHERE`, it rewrites every row, which is what Blackjack's `UPDATE cards SET place="Discard";` relies on.
- Python reaches SQLite via `sqlite3.Connection(path)`. `db.execute(sql)` runs one statement and returns a cursor; `.fetchall()` yields a list of tuples, one per row. `db.executescript(sql)` runs multiple statements. `db.commit()` makes changes permanent.
- Concatenating untrusted input into a SQL string is a SQL injection vulnerability: `"Robert'); DROP TABLE Students; --"` closes the intended statement, adds a `DROP TABLE`, and comments out the remainder.
- The fix is parameterization: `db.execute("INSERT INTO Students VALUES (?)", [name])`. SQLite plans the query before substitution and receives parameters through a separate channel, so data never reaches the parser.
- The Blackjack demo stores game state in the table itself: `cards(card, place, hand)`. Dealing is `INSERT`, scoring is `SELECT ... WHERE place = ?`, and clearing a hand is an unrestricted `UPDATE`.
- The dealer's face-down card is modeled by holding it in a Python local (`hidden`) and inserting it only when revealed. Not being in the table is what makes it unseeable.
- `hand_score` scores aces as 1, then adds 10 once if `total <= 11 and 'A' in hand`, which correctly handles at most one high ace per hand.
