<!-- Mon, Nov 16, 2026 | sources: slides + YouTube auto-transcript -->
# Lecture 34: SQL

This lecture introduces SQL (the Structured Query Language), the language used to talk to database management systems, and frames it as a fundamentally different kind of programming language from Python, Scheme, or Gleam. Where those languages are **imperative** (a program describes a computational process, and the interpreter carries out the evaluation rules), SQL is **declarative** (a program describes the desired result, and the system figures out how to produce it efficiently). The lecture opens by connecting back to algebraic data types: a SQL table is a *relation*, holding all the records of one variant, so each row corresponds to one record. It then covers the structure of a database (tables made of rows and columns, where a column has a name and a type, and a row has one value per column), the four guarantees databases give applications (atomicity, consistency, isolation, durability), the anatomy of a `SELECT` statement (`SELECT` columns, `FROM` a table, `WHERE` a condition, `ORDER BY` an ordering, `LIMIT` a row count), the use of `CREATE TABLE ... AS` to give a query result a permanent name, arithmetic inside column descriptions with `AS` to name the output column, and finally why declarativeness matters: the same query can be executed by scanning every row or by using an index, and the database picks, which is what makes query planners and execution plans possible.

## Key Concepts

### Database management systems hold the data

A DBMS is the software responsible for recording and retrieving data. It is the primary data storage for most software applications. The lecture is emphatic that these are crucial, heavily used, and interesting pieces of software: this is not a niche topic bolted onto the end of the course.

A DBMS organizes information into **tables**. A table is a collection of **records**, which are rows. Three facts fit together and are worth holding in your head as a unit:

- A table has columns and rows.
- A row has a value for each column.
- A column has a name and a type.

The lecture's two running mental models:

A users table:

| ID | first_name | last_name | email | role |
|----|-----------|-----------|-------|------|
| 1 | Kay | Ousterhout | kayo@berkeley.edu | Instructor |
| 2 | John | DeNero | denero@berkeley.edu | Instructor |
| 3 | Esha | Telang | esha_telang@berkeley.edu | TA |

And a cities table used in the transcript: three records (Berkeley, Cambridge, Minneapolis) with three columns (latitude, longitude, name). Here `name` holds strings while `latitude` and `longitude` hold numbers. The payoff of that example: **if you look across a row, you see values of different types**, because each column carries its own type. A row is simply a value for each column in the table.

### SQL is a language you almost certainly will use

The slides describe SQL as "perhaps the most widely used programming language," and the transcript repeats this: perhaps the most widely used programming language in the world. It is the language you use to interact with a database, and it is a common language shared across many different database systems: SQLite, MySQL, PostgreSQL, and a wide variety of other commercial and open-source systems.

The course uses SQLite, specifically because it can run on a single machine rather than requiring a separate database server. This is also why SQLite is widely used commercially and in individual projects. Mac and Linux machines typically ship with it: type `sqlite3` and you are ready to go. On Windows you would need to install it, but SQLite also comes bundled with Python, so the course provides a Python program that runs SQLite through the Python installation. The course also provides a web-based interpreter, which has the advantage of coming pre-loaded with tables from past labs and homeworks (run `sqlite3` bare on your own machine and you have no tables to play with).

### Databases in applications: ACID

Databases are used to provide a **single source of truth** and to **ensure data validity**. The slides spell out four properties, which together are conventionally called ACID (the slides name each property but do not use the acronym itself; the acronym is *(extra context)*):

- **Atomicity**: validity might require that multiple updates all happen together, or not at all. You should never see half of a transfer.
- **Consistency**: validity might require that a set of constraints are always met.
- **Isolation**: an application could run concurrently on many machines against a shared database, and each should behave as though it had the database to itself.
- **Durability**: programs and machines could crash, but the database ensures updates are persistent.

The reason these matter conceptually: everything you have written this semester assumed one process with one memory. A database is shared mutable state across machines and across crashes, and these four properties are the contract that makes sharing it safe.

### The bridge from algebraic data types: a table is a relation

The lecture's review segment asks about the Animator interpreter's value type in Gleam:

```
pub type Value {
    VBool(Bool)
    VInt(Int)
    VList(List(Value))
    VData(tag: String, fields: List(Value))
    VFunc(Callable)
}
```

In Python this would be written as a type union: `type Value = bool | int | list[Value] | tuple[str, list[Value]] | ...`. In Gleam each variant has a name used in two ways: to **construct** records (`VList([VInt(3), VInt(5)])`) and to **pattern match**, decomposing them in `case` expressions (`VList([a, ..])`). The same syntax composes and decomposes data.

The review question is: why isn't `VList(List(Value))` just `VList(Value)` or `List(Value)`? The answer is structural: `VList` is a *variant tag* that wraps a list of values so the list itself can be a `Value`, while `List(Value)` is the raw list type. The wrapper is what lets a list live inside the same type as a bool or an int.

The connection to SQL, stated explicitly on the slides:

- Each row of a SQL table corresponds to a **record** in Gleam.
- A record is described by **one variant** of a type.
- So a SQL table holds **all the records for one variant**.
- A SQL table is also called a **relation**, as in *relational database*.

So the `users` table above is exactly the set of all records of a hypothetical `User(id, first_name, last_name, email, role)` variant. The column names are the field names; the column types are the field types; each row is one constructed record.

### Declarative versus imperative programming

This is the conceptual heart of the lecture.

**Imperative** languages (Python, Scheme): a program is a description of a computational process. The interpreter carries out the execution or evaluation rules for the process you described. You wrote the *how*.

**Declarative** languages (SQL): a program is a description of the desired result. The interpreter figures out how to generate the result. You write the *what*, and it is up to the computer to figure out how to compute or build it for you.

The transcript's illustration is a query over the cities table that produces rows with the literal string "West Coast" in a `Region` column and the city name in a `Name` column, restricted to cities with large longitude, ordered by latitude from least to greatest:

| Region | Name |
|--------|------|
| West Coast | Los Angeles |
| West Coast | Berkeley |

Notice what the output table demonstrates:
- The `Region` column's values come from the **query itself**, not the input table.
- The `Name` column's values come from the **original cities table**.
- The rows are a **subset** of the input rows: only the ones with large longitude survived.

And notice what you did *not* have to say. You did not say whether to sort by longitude and pick the first few, or to iterate over rows appending high-longitude rows to a list. You did not say whether to filter before ordering by latitude or after, or whether to project out the name before filtering or after. As the lecture puts it: all those details really aren't your problem. Your problem is just to describe the table you want to build, and SQL will find a way to build it for you.

### Why declarativeness pays off: indexes and query plans

The slides make the payoff concrete with this query against the IMDb `names` table:

```sql
SELECT name, 2026 - birth FROM names WHERE death = '\N' and birth != '\N';
```

**Possible process 1**: scan all rows, checking `death = '\N'` and `birth != '\N'` on each. Linear in the size of the table.

**Possible process 2**: use an **index**. An index is a separate data structure that keeps track of where certain values are in the table. A `death` index conceptually looks like a mapping from each distinct death value to the set of row locations holding it: `{2014: ..., '\N': ...}`. With that index, you look up only the entries where `death = '\N'` instead of touching every row.

Both processes answer the same query. You wrote the query once. The database chose. That choice is the whole point of declarative programming: **the database is responsible for determining how to run this quickly.** The lecture closes by showing a SQL Server query plan diagram (the actual execution plan, from Microsoft's docs), which is the system reporting back the process it chose for your description of a result.

### Anatomy of a SELECT statement

The general form shown throughout the lecture:

```sql
SELECT [columns] FROM [table] WHERE [condition] ORDER BY [order];
```

Plus `LIMIT [n]` for a maximum row count, and `CREATE TABLE [name] AS` wrapped around the front to persist the result.

Reading clause by clause:

- `SELECT [columns]`: **which columns**. A comma-separated list of column *descriptions*. Each description is an **expression** (usually just a column name, but it can be a number or arithmetic), optionally followed by `AS [name]` to name that column in the result.
- `FROM [table]`: the input table from which you select rows and columns.
- `WHERE [condition]`: **which rows**. A condition about the values in some columns.
- `ORDER BY [order]`: the ordering over the remaining rows. Ascending (least to greatest) by default; add `DESC` for descending.
- `LIMIT [n]`: the maximum number of rows to select. Leave it out and you get all rows.

The order of the **columns** in the output is determined by nothing but the order in which you write the expressions in the column description list.

A select statement transforms one table into another. The output table may have different rows and columns than the input. This operation is sometimes called **projecting** the table: a select statement takes an existing table and picks out some of the columns and some of the rows, and it can reorder the columns and reorder the rows.

The special column description `*` refers to all columns in the table.

### SQLite shell commands

These are not SQL, they are commands for the SQLite shell program (note the leading dot):

- `.tables`: list all tables in the current database.
- `.mode box`: make the display pretty. The transcript also mentions `.mode column` as a nicer-formatted alternative. Either way it is the same data, just differently displayed: the raw default output does not show you what the columns are called.

## Definitions

- **Database management system (DBMS)**: software responsible for recording and retrieving data, serving as the primary data storage for most software applications.
- **Table**: a collection of records, which are rows that have a value for each column. Also called a *relation*.
- **Row**: a single record in a table; a value for each column in the table.
- **Column**: a named, typed field of a table. A column has a name and a type.
- **Record**: one row of a SQL table; corresponds to a record described by one variant of an algebraic data type.
- **Relation**: another name for a SQL table, as in *relational database*; a table holds all the records for one variant.
- **SQL (Structured Query Language)**: the declarative language used to interact with a database; perhaps the most widely used programming language. Shared across SQLite, MySQL, PostgreSQL, and many other systems.
- **SQLite**: a SQL database system that can run on a single machine rather than requiring a separate database server. Used in this course; typically pre-installed on Mac and Linux, and bundled with Python.
- **Declarative programming**: a paradigm in which a program is a description of the desired result, and the interpreter figures out how to generate that result.
- **Imperative programming**: a paradigm in which a program is a description of a computational process, and the interpreter carries out the execution or evaluation rules for the process described. Python and Scheme are imperative.
- **Query**: a statement whose role is to take an existing table and build another table.
- **SELECT statement**: by far the most common statement in SQL; specifies output columns via column descriptions, an input table via `FROM`, selected rows via `WHERE`, row order via `ORDER BY`, and a maximum row count via `LIMIT`.
- **Column description**: an expression in a `SELECT` list, optionally named with `AS [name]`, that determines how each input row is projected into a result row.
- **Projection**: going from one table to a different version of that same table, picking out some columns and some rows, possibly reordering either.
- **`CREATE TABLE [name] AS [select statement]`**: gives the result of a `SELECT` statement a permanent name that can be referred to later. Without it, the result is displayed but not stored anywhere.
- **Index**: a separate data structure that keeps track of where certain values are in a table, allowing the database to look up matching rows directly instead of scanning all of them.
- **Query plan / execution plan**: the process the database chooses in order to produce the result your query described.
- **Atomicity**: the guarantee that multiple updates all happen together or not at all.
- **Consistency**: the guarantee that a set of constraints is always met.
- **Isolation**: the guarantee that an application running concurrently on many machines against a shared database behaves correctly.
- **Durability**: the guarantee that updates are persistent even though programs or machines could crash.
- **`\N`**: in the IMDb dataset, the string used to mark a missing value (for example, a person with no death year recorded).

## Worked Examples

Note on language: this lecture's "real code" is SQL, so the examples below are SQL, with Python used where the lecture itself reaches for Python (the type union for `Value`) and for *(extra context)* on how SQL runs from Python. SQL has no environment diagrams; the analogous reasoning is **row-by-row evaluation of column descriptions**, which I spell out in words below.

### Example 1: Selecting columns and filtering rows (the users table)

```sql
SELECT email, first_name FROM users WHERE role = "Instructor";
```

Step by step, over the `users` table:

1. `FROM users` names the input table: three rows, five columns (`ID`, `first_name`, `last_name`, `email`, `role`).
2. `WHERE role = "Instructor"` keeps only the rows whose `role` value is `"Instructor"`. Row 1 (Kay) and row 2 (John) survive; row 3 (Esha, a TA) is dropped.
3. `SELECT email, first_name` projects each surviving row down to two columns, **in that order**: email first, then first name. The original table listed `first_name` before `email`, but the output order follows the `SELECT` list, not the input table.
4. No `ORDER BY`, so no ordering is declared.

Result:

| email | first_name |
|-------|-----------|
| kayo@berkeley.edu | Kay |
| denero@berkeley.edu | John |

The row-by-row reasoning worth internalizing: think of the `WHERE` condition and each `SELECT` expression as being **evaluated once per row**, with column names evaluating to that row's values. For row 1, `role` evaluates to `"Instructor"`, so the condition is true; then `email` evaluates to `"kayo@berkeley.edu"` and `first_name` evaluates to `"Kay"`, producing one output row. Repeat for each row.

### Example 2: Persisting a result with CREATE TABLE

```sql
CREATE TABLE ta_emails AS
  SELECT email FROM users WHERE role = "TA";
```

1. The inner `SELECT` runs exactly as in Example 1, but with condition `role = "TA"`, so only row 3 survives, projected to the single `email` column.
2. `CREATE TABLE ta_emails AS` takes that one-column, one-row result and gives it a permanent name.

Result, stored as `ta_emails`:

| email |
|-------|
| esha_telang@berkeley.edu |

Why this matters: the lecture is explicit that the result of a bare `SELECT` statement is **displayed, not stored anywhere**. If you want to refer to it later, you must `CREATE TABLE`. The transcript demonstrates this with a `first_three` table: after `CREATE TABLE first_three AS SELECT year AS release_year, title FROM titles LIMIT 3;`, a later query `SELECT release_year FROM first_three;` returns three rows **even with no `LIMIT`**, because the `first_three` table itself only has three rows. The limit was baked in at creation time.

### Example 3: `SELECT *`, `LIMIT`, and `AS` on the titles table

The `titles` table from the IMDb subset:

| tconst | title | year | runtime | genres |
|--------|-------|------|---------|--------|
| tt0012349 | The Kid | 1921 | 68 | Comedy,Drama,Family |
| ... | | | | |

```sql
SELECT * FROM titles LIMIT 5;
```

`*` means all columns; `LIMIT 5` means at most five rows. This shows the first 5 rows of the table. The transcript's warning: `SELECT * FROM titles;` with no limit dumps a thousand rows to your terminal, so be careful.

Now restrict and rename:

```sql
SELECT year AS release_year, title FROM titles LIMIT 3;
```

1. `FROM titles` is the input.
2. The column description list has two entries. The first is the expression `year`, named `release_year` by `AS`. The second is `title`, unnamed, so it keeps the name `title`.
3. `LIMIT 3` caps the output at three rows.

The output table has **two** columns instead of five, in the order `release_year, title` (year *before* title, the reverse of the input table's order), and the first column is called `release_year`, not `year`. Three things changed at once: which columns, their order, and their names. The lecture stresses that the way you reorder columns is simply the order you write them in the `SELECT` list.

### Example 4: ORDER BY with LIMIT (the "oldest movies" query)

Create a new table with the 5 oldest movie titles and years in the dataset:

```sql
CREATE TABLE oldest AS
  SELECT title, year FROM titles ORDER BY year LIMIT 5;
```

1. `FROM titles` is the input: 1000 rows.
2. No `WHERE`, so all rows are kept.
3. `ORDER BY year` sorts the rows by year, **least to greatest** (ascending is the default). Oldest first.
4. `LIMIT 5` then takes the first five of that ordering, which are the five oldest.
5. `SELECT title, year` projects each to two columns.
6. `CREATE TABLE oldest AS` stores the result.

The key insight: `ORDER BY` plus `LIMIT` is how you express "the top/bottom k." You never write a sort, never write a loop, never track a running minimum. You declare the ordering and the count.

### Example 5: Arithmetic in a column description plus DESC (the "longest movies" query)

Select the title and runtime in hours of the 20 longest movies:

```sql
SELECT title, runtime / 60.0 FROM titles ORDER BY runtime DESC LIMIT 20;
```

1. `runtime / 60.0` is an **expression**, not a bare column name. SQL can perform arithmetic in the column description. For each row, `runtime` evaluates to that row's runtime in minutes, and the expression produces runtime in hours.
2. The `60.0` is deliberately a float. Dividing by `60.0` rather than `60` is what makes this a real-valued hour count *(extra context: integer division semantics vary by SQL dialect, so the float literal is the safe habit)*.
3. `ORDER BY runtime DESC` sorts greatest to least. Without `DESC` you would get the twenty *shortest* movies.
4. `LIMIT 20` takes the top twenty of that descending order.

Note that `ORDER BY runtime` works even though `runtime` is not in the output column list: you ordered by a column of the *input* table, and the output shows `runtime / 60.0`. The ordering clause and the column descriptions are independent concerns.

### Example 6: ORDER BY keeps rows intact (the 2025 movies query)

```sql
SELECT title, runtime, genres FROM titles WHERE year = 2025 ORDER BY runtime;
```

1. `WHERE year = 2025` keeps only rows whose year is 2025. In the thousand-movie dataset these are all highly rated movies, so this is not all movies released in 2025, just the ones in the subset.
2. `ORDER BY runtime` sorts the surviving rows by runtime, least to greatest.
3. No `LIMIT`, so every row that survived the `WHERE` is shown.

The point the lecture hammers here: **`ORDER BY` does not just order that one column, it orders all of the rows.** After sorting, "How to Train Your Dragon" is still 125 minutes, exactly as it was before the sort. The different parts of a row stay together. A row is an indivisible unit; sorting rearranges rows, it does not shuffle one column independently of the others.

### Example 7: Computing ages on the names table, and handling missing data

The `names` table:

| nconst | name | birth | death | profession | knownForTitles |
|--------|------|-------|-------|------------|----------------|
| nm0000002 | Lauren Bacall | 1924 | 2014 | actress,miscellaneous,soundtrack | tt0037382,tt0075213,... |
| nm0000250 | Renée Zellweger | 1969 | \N | actress,producer,writer | tt0299658,... |
| nm0000255 | Ben Affleck | 1972 | \N | producer,actor,writer | tt1024648,... |

Select the names and ages of all people:

```sql
SELECT name, death - birth FROM names;
```

For each row, `death - birth` is evaluated with that row's values. Lauren Bacall gives `2014 - 1924`, so `90`. This works for people who have died. It does **not** work for people who are alive, because their `death` is the string `'\N'`, not a number.

Select the names and ages of all alive people, where birth is not `'\N'`:

```sql
SELECT name, 2026 - birth FROM names WHERE death = '\N' and birth != '\N';
```

1. `WHERE death = '\N'` keeps only the rows whose death value is the missing-data marker, which is exactly the people who are (recorded as) alive.
2. `and birth != '\N'` additionally drops rows with no recorded birth year, since `2026 - '\N'` is meaningless. Two conditions combined with `and`; `!=` is inequality.
3. `2026 - birth` computes the age for each surviving row. For Renée Zellweger, `2026 - 1969 = 57`. For Ben Affleck, `2026 - 1972 = 54`.
4. The year `2026` is a hard-coded constant in the query, a value that comes from the query itself rather than from the table (compare the "West Coast" literal in the cities example).

Why both conditions are needed: this is the practical shape of missing-data handling in real datasets. The sentinel value `'\N'` is a *string* sitting in a column you want to do *arithmetic* on, so every arithmetic query over that column needs a guard in the `WHERE` clause.

### Example 8: Arithmetic over multiple columns (the ski lift)

This example builds a table from scratch rather than loading a database. Conceptually: a ski lift has chairs; each chair carries some number of single riders and some number of couples. The number on the little plaque on the chair is the `chair` value.

```sql
CREATE TABLE lift(chair, single, couple);
INSERT INTO lift VALUES (101, 2, 2);
INSERT INTO lift VALUES (102, 0, 3);
INSERT INTO lift VALUES (103, 4, 1);
```

(The lecture does not dwell on the create/insert syntax, and the exact literal values are reconstructed from the described scenario: chair 101 has two singles and two couples, chair 102 has three couples, chair 103 has four singles and one couple.)

Now: how would we check that no chair has more than six people on it? We need arithmetic across two columns:

```sql
SELECT chair, single + 2 * couple AS total FROM lift;
```

Step by step, and this is the crucial mechanic:

1. The table has columns `chair`, `single`, and `couple`, and you refer to them by name just as in every earlier query.
2. **The expression `single + 2 * couple` is evaluated once for every row.** For a given row, `single` evaluates to that row's single-rider count and `couple` to that row's couple count. A couple is two people, hence the `2 *`.
3. `AS total` names the resulting column `total`. The `AS` refers to the value of *this expression*, which becomes one of the two columns in the resulting table.
4. The result has three rows, just as `lift` had three rows, and two columns (`chair` and `total`).

| chair | total |
|-------|-------|
| 101 | 6 |
| 102 | 6 |
| 103 | 6 |

Six people on each chair, so nothing is overloaded. Verify by hand: 101 is `2 + 2*2 = 6`; 102 is `0 + 2*3 = 6`; 103 is `4 + 2*1 = 6`.

The reasoning-in-words analogue of an environment diagram: imagine a frame created per row, in which each column name is bound to that row's value for that column. The column description is evaluated in that frame. The frame is discarded and the next row gets a fresh one. Rows never see each other's values. This is why a `SELECT` with no aggregation produces exactly as many output rows as input rows that survived the `WHERE`.

### Example 9 *(extra context)*: Running SQL from Python

The lecture mentions that SQLite ships with Python and that the course provides a Python wrapper, but does not show the API. For completeness:

```python
import sqlite3

db = sqlite3.connect("imdb.db")
for row in db.execute("SELECT title, year FROM titles ORDER BY year LIMIT 5;"):
    print(row)
```

Each row comes back as a Python tuple of column values, which lines up neatly with the lecture's "a row is a value for each column" and with the Gleam record analogy.

## Common Pitfalls

- **Forgetting that a bare `SELECT` does not save anything.** The result is displayed. It is not stored. If you need to refer to it in a later query, you must wrap it in `CREATE TABLE [name] AS`.
- **Thinking `ORDER BY` sorts only the named column.** It reorders entire rows. The values in a row stay together: a movie's runtime stays attached to its title.
- **Assuming the output column order matches the input table's column order.** It matches the order you wrote the expressions in the `SELECT` list, and nothing else.
- **Running `SELECT * FROM titles;` with no `LIMIT`.** A thousand rows scroll past. Add `LIMIT`.
- **Doing arithmetic on a column containing the missing-data sentinel `'\N'`.** `2026 - '\N'` is not a number. You need `WHERE birth != '\N'` (and, depending on the question, `WHERE death = '\N'` or `death != '\N'`).
- **Computing age as `death - birth` for living people.** That only works for people with a recorded death year. For living people you subtract the birth year from the current year.
- **Forgetting `DESC` when you want the largest values.** `ORDER BY runtime LIMIT 20` gives the twenty *shortest* movies. The twenty longest need `ORDER BY runtime DESC LIMIT 20`.
- **Writing `60` instead of `60.0` when converting minutes to hours.** The lecture's query uses `60.0` deliberately.
- **Confusing SQLite shell commands with SQL.** `.tables` and `.mode box` start with a dot and are instructions to the shell program, not queries. They take no semicolon.
- **Forgetting the semicolon at the end of a SQL statement.** Every SQL statement in the lecture ends in `;`.
- **Trying to control *how* the query runs.** The whole point of declarative programming is that you do not. Whether the system scans all rows or uses an index, and whether it filters before or after ordering, is the database's decision.
- **Mistaking `AS` in a column description for `AS` in `CREATE TABLE`.** In `SELECT year AS release_year`, `AS` names an output *column*. In `CREATE TABLE oldest AS SELECT ...`, `AS` introduces the query that defines a *table*. Same keyword, two jobs.
- **Confusing a variant wrapper with a raw type.** In the review material, `VList(List(Value))` wraps a list so it can *be* a `Value`; `List(Value)` is just the list type and is not itself a `Value`.

## Likely Exam Points

### 1. Declarative versus imperative

Exams love to test whether you can state the distinction crisply and say which side a given language falls on.

**Practice**: In one sentence each, describe what a program *is* in a declarative language versus an imperative language, and give one example language of each from this course.

**Answer**: In a declarative language, a program is a description of the desired result, and the interpreter figures out how to generate it (SQL). In an imperative language, a program is a description of a computational process, and the interpreter carries out the execution or evaluation rules for that process (Python, or Scheme).

### 2. Filling in a `SELECT` statement to match a described result

The slides are literally fill-in-the-blank exercises, which is a strong signal about the exam format.

**Practice**: The `titles` table has columns `tconst`, `title`, `year`, `runtime`, `genres`. Write a statement that creates a table `recent_long` containing the title and year of the 10 longest movies released in or after 2020, longest first.

**Answer**:
```sql
CREATE TABLE recent_long AS
  SELECT title, year FROM titles WHERE year >= 2020 ORDER BY runtime DESC LIMIT 10;
```

### 3. Predicting the output table of a given query

**Practice**: Given the `users` table from the lecture, what table does `SELECT last_name, role FROM users WHERE role != "TA" ORDER BY last_name;` produce? Give the exact columns, their order, and the rows in order.

**Answer**: Two columns, `last_name` then `role`. The `WHERE` drops Esha Telang. Ordering ascending by last name puts DeNero before Ousterhout:

| last_name | role |
|-----------|------|
| DeNero | Instructor |
| Ousterhout | Instructor |

### 4. Arithmetic in a column description, evaluated per row

**Practice**: The `lift` table has columns `chair`, `single`, `couple` with rows `(101, 2, 2)`, `(102, 0, 3)`, `(103, 4, 1)`. What does `SELECT chair, single + 2 * couple AS total FROM lift;` return, and how many rows does it have?

**Answer**: Three rows, one per input row, with columns `chair` and `total`: `(101, 6)`, `(102, 6)`, `(103, 6)`. The expression `single + 2 * couple` is evaluated once per row using that row's column values; a couple counts as two people. `AS total` names the computed column.

### 5. Missing-data handling with `'\N'`

**Practice**: The `names` table has `name`, `birth`, `death`, where `'\N'` marks a missing value. Write a query giving the name and age at death of everyone who has died and whose birth year is recorded.

**Answer**:
```sql
SELECT name, death - birth FROM names WHERE death != '\N' and birth != '\N';
```
Both guards are needed: without `death != '\N'` you would subtract from a non-number, and without `birth != '\N'` you would subtract a non-number.

### 6. Table as relation: the connection to algebraic data types

**Practice**: Explain, in terms of records and variants, what a single SQL table holds, and give the term for a table that reflects this.

**Answer**: A SQL table holds all of the records for one variant of a type. Each row is one record, each column corresponds to one field of that variant (with a name and a type). Such a table is called a *relation*, which is why these are called relational databases.

### 7. ACID properties

**Practice**: Name the property that guarantees each of the following, as described in lecture: (a) a set of updates either all happen or none do; (b) updates survive a machine crash; (c) many machines can share one database concurrently; (d) a set of constraints is always met.

**Answer**: (a) Atomicity, (b) Durability, (c) Isolation, (d) Consistency.

### 8. Indexes and why declarativeness enables optimization

**Practice**: For the query `SELECT name, 2026 - birth FROM names WHERE death = '\N' and birth != '\N';`, describe two different processes a database could use to produce the result, and say who chooses between them.

**Answer**: (1) Scan all rows, testing `death = '\N'` and `birth != '\N'` on each. (2) Use an index on `death` (a separate data structure recording where each death value occurs in the table) to look up only the rows with `death = '\N'`, then test the birth condition on those. The database chooses, not the programmer, because the query describes the desired result rather than the process.

### 9. `ORDER BY` semantics

**Practice**: True or false: `ORDER BY runtime` sorts the `runtime` column while leaving the other columns in place. Justify.

**Answer**: False. `ORDER BY` reorders whole rows, not a single column. The lecture's check is that after `ORDER BY runtime`, "How to Train Your Dragon" still has runtime 125: the parts of each row stay together.

### 10. `CREATE TABLE` persistence and baked-in limits

**Practice**: After running `CREATE TABLE first_three AS SELECT year AS release_year, title FROM titles LIMIT 3;`, how many rows does `SELECT release_year FROM first_three;` return, and why, given that it has no `LIMIT`?

**Answer**: Three. The `LIMIT 3` applied when `first_three` was created, so the stored table itself contains only three rows. A query with no `LIMIT` returns all rows of its input table, and that input table has three.

## Summary

- A **DBMS** records and retrieves data and is the primary data store for most applications; it organizes data into **tables**.
- A table has columns and rows; a **row** has one value for each column; a **column** has a name and a type. Across a row you see values of different types.
- **SQL** is the language for interacting with a database, shared across SQLite, MySQL, PostgreSQL, and others, and is perhaps the most widely used programming language. The course uses **SQLite** because it runs on a single machine.
- Databases provide a single source of truth and ensure validity via **atomicity** (all-or-nothing updates), **consistency** (constraints always met), **isolation** (safe concurrent access from many machines), and **durability** (updates survive crashes).
- A SQL table is a **relation**: it holds all the records for one variant of an algebraic data type. Each row is a record; each column is a named, typed field.
- SQL is **declarative**: a program describes the desired result and the interpreter figures out how to generate it. Python and Scheme are **imperative**: a program describes a process and the interpreter carries it out.
- The role of a **query** is to take an existing table and build another table. The output may have different rows and columns than the input. This is called **projecting**.
- `SELECT [columns] FROM [table] WHERE [condition] ORDER BY [order] LIMIT [n];`: columns picks *which columns* (and their order and names), `WHERE` picks *which rows*, `ORDER BY` picks the *row order* (ascending by default, `DESC` for descending), `LIMIT` caps the row count.
- A **column description** is an expression, optionally named with `AS [name]`. `*` means all columns. Expressions are evaluated once per row, with column names evaluating to that row's values, so SQL can do arithmetic across columns and with constants.
- `ORDER BY` reorders whole rows, keeping each row's values together.
- A bare `SELECT` result is displayed but not stored. `CREATE TABLE [name] AS [select]` gives it a permanent name you can query later; any `LIMIT` is baked in at creation.
- SQLite shell commands (not SQL): `.tables` lists tables, `.mode box` / `.mode column` improve the display.
- In the IMDb dataset, `'\N'` marks missing values, so arithmetic on `birth` or `death` requires `WHERE` guards against `'\N'`.
- An **index** is a separate data structure tracking where values live in a table. Because queries describe results rather than processes, the database can choose between a full scan and an index lookup, and can build a **query plan**. That freedom is the payoff of declarative programming.
