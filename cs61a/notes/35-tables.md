<!-- Wed, Nov 18, 2026 | sources: slides + YouTube auto-transcript -->
# Lecture 35: Tables

SQL is a declarative language: a query describes the table you want, and the database engine figures out how to compute it efficiently (possibly by scanning every row, possibly by using an index). This lecture builds on that foundation to cover the single most important SQL skill: **joining tables**. When the result you want draws information from more than one table, or from more than one row of the same table, you join. A join glues a row from one table onto a row from another to form one long row, and then you select the columns you care about out of that long row. Without a matching condition (the `ON` clause) you get a cross join: every possible pairing, almost always nonsense. With the right conditions you get coherent rows. The lecture works through two-table joins (titles with ratings), three-table joins (titles with principals with names, to find who played James Bond), self-joins with aliases (remakes of the same film, actors playing two characters), and five-table joins (co-stars of 2024 films). It closes with numeric and string expressions in SQL: arithmetic, comparison, concatenation with `||`, and partial matching with `LIKE`.

## Key Concepts

### Declarative programming, revisited

In declarative programming a "program" is a *description of the desired result*, and the interpreter figures out *how* to generate it. Compare:

- **Imperative** (Python): you write the loop, the accumulator, the control flow. You specify *how*.
- **Declarative** (SQL): you write `SELECT name, 2026 - birth FROM names WHERE death = '\N' AND birth != '\N';`. You specify *what*.

The database is responsible for running this quickly. Two different execution strategies for the same query:

1. **Scan** all rows, testing `death = '\N' AND birth != '\N'` on each.
2. Use an **index** on `death`: a separate data structure that tracks where each value lives in the table, conceptually like a dictionary `{2014: <row locations>, '\N': <row locations>}`. Look up the `'\N'` bucket directly and skip everything else.

Your query does not change. The query optimizer chooses. This is the payoff of declarative programming: the engine can get smarter without you rewriting anything.

### What a join actually is

A join takes **a row from table A** and **a row from table B** and concatenates them into **one long row**. That is the whole mechanism. `SELECT *` on a join of a 3-column table and a 3-column table gives you a 6-column table.

The crucial mental model from the lecture: **when you build an output table, think about building one of its rows.** Ask: what pieces of information does a single output row need, and which table does each piece live in? That tells you exactly how many tables (and how many copies of each table) must appear in the `FROM` clause.

### Cross join: the default, and why it is almost never what you want

If you omit `ON` (or `WHERE`), SQL forms **all combinations** of a row from A and a row from B. With the lecture's tiny tables:

```
titles                                  ratings
tconst      title                year   tconst      avgRating  numVotes
tt0058331   Mary Poppins         1964   tt0058331   7.8        195397
tt14205554  KPop Demon Hunters   2025   tt14205554  7.6        95933
tt8404614   The Two Popes        2019
```

```sql
SELECT * FROM titles JOIN ratings;
```

3 rows × 2 rows = 6 output rows:

```
tconst       title                year   tconst       avgRating  numVotes
tt0058331    Mary Poppins         1964   tt0058331    7.8        195397   <- coherent
tt0058331    Mary Poppins         1964   tt14205554   7.6        95933    <- nonsense
tt14205554   KPop Demon Hunters   2025   tt0058331    7.8        195397   <- nonsense
tt14205554   KPop Demon Hunters   2025   tt14205554   7.6        95933    <- coherent
tt8404614    The Two Popes        2019   tt0058331    7.8        195397   <- nonsense
tt8404614    The Two Popes        2019   tt14205554   7.6        95933    <- nonsense
```

Only 2 of 6 rows mean anything. The rest pair a title from one movie with a rating from a totally different movie. Worse: `The Two Popes` appears with ratings even though `ratings` has no row for it at all, so the output *looks* like it has a rating for that film when it does not. This is the specific danger the instructor emphasizes: a missing `ON` clause does not produce an error, it produces confidently wrong data.

### The `ON` clause: your job, not SQL's

```sql
SELECT * FROM titles JOIN ratings ON titles.tconst = ratings.tconst;
```

Now only the 2 coherent rows survive. The instructor is blunt about why this matters: **"SQL doesn't work that out for you."** There is nothing magic about a column being named `tconst` in both tables. You must explicitly state that the `tconst` value in the `titles` row has to equal the `tconst` value in the `ratings` row. Shared column names are a convention that *you* exploit, not a relationship the database infers.

Note the role `tconst` plays: it is needed **during** the join to match rows, but it is usually **not wanted** in the output. After matching, you select just the useful columns:

```sql
SELECT title, avgRating FROM titles JOIN ratings ON titles.tconst = ratings.tconst;
```

```
title                 avgRating
-------------------   ---------
Mary Poppins          7.8
KPop Demon Hunters    7.6
```

### Two syntaxes: explicit and implicit

These two queries do the same thing:

```sql
-- Explicit (modern, preferred)
SELECT title, avgRating FROM titles JOIN ratings ON titles.tconst = ratings.tconst;

-- Implicit (older, still common in the wild)
SELECT title, avgRating FROM titles, ratings WHERE titles.tconst = ratings.tconst;
```

A **comma** in the `FROM` clause *is* a join. And `ON` is, from SQL's perspective, just a condition, so it can live in `WHERE` instead. The instructor's point: from the engine's view, `ON` and `WHERE` are interchangeable here, but there is a **style rule** worth following:

- Put conditions that **match rows across tables** in `ON`.
- Put conditions that **filter a single table** in `WHERE`.

This makes queries readable: `ON` answers "how do these tables connect?", `WHERE` answers "which rows do I want?".

One mechanical difference the lecture notes: you can have **only one** `WHERE` clause, but you can chain as many conditions as you like inside it with `AND`.

### Joining three (or more) tables

The IMDb schema in this lecture:

```
titles      (tconst, title, year, runtime, genres)
ratings     (tconst, avgRating, numVotes)
principals  (tconst, ordering, nconst, category, character)
names       (nconst, name, birth, death, profession, knownForTitles)
```

`titles` and `names` have **no column in common**. A movie row does not know about actors; a person row does not know about films (the `knownForTitles` string is a comma-separated blob, which the lecture later points out is a bad way to store structured data). So there is no way to directly join them.

`principals` is the **bridge table**: it exists to record which people worked on which titles. Each `principals` row has both a `tconst` and an `nconst`. To go from a title to an actor's name you must route through it:

```
titles.tconst = principals.tconst  AND  principals.nconst = names.nconst
```

The insight to internalize: **a table can appear in the `FROM` clause even if none of its columns appear in the output.** You join in `principals` not because you want its columns, but because without it the titles rows and names rows cannot be made coherent.

### Self-joins and aliases

Sometimes one output row depends on **two rows of the same input table**. Examples from the lecture:

- Two movies with the same title but different years (remakes).
- Two characters in the same film played by the same actor.
- Two co-stars in the same film.
- Two cities, to compute the distance between them.

In all of these, the table appears twice in the `FROM` clause, and **aliases become mandatory**, because `titles.title` is ambiguous when `titles` is in the query twice. An alias is introduced with `AS`:

```sql
FROM titles AS old JOIN titles AS new
```

Now `old` and `new` are two independent rows from the same table, and `old.title` / `new.title` are unambiguous.

The diagnostic question from the lecture: **"Does an output row depend on data from two input rows?"** If yes, self-join.

Aliases are also useful (not required) for brevity in ordinary multi-table joins: `principals AS p` lets you write `p.tconst` instead of `principals.tconst`. Aliases can be any length; one letter is just convenient.

### Numeric expressions

SQL expressions look a lot like Python expressions and can appear anywhere in a `SELECT` statement: in the column list, in `WHERE`, in `ORDER BY`.

- Arithmetic: `+ - * /`
- Built-in function calls: `abs`, `round`, and others
- Comparison: `< <= > >= =`
- Two forms of not-equal: `!=` and `<>`
- **Equality is a single `=`**, not `==`. There is no assignment in a query, so `=` is free for comparison.

```sql
SELECT name, death - birth FROM names;          -- lifespan of everyone
SELECT name, 2026 - birth FROM names
  WHERE death = '\N' AND birth != '\N';         -- current age of living people
```

Note `'\N'` is how this dataset represents "missing". It is a two-character string, not SQL `NULL`, so you compare it with `=` like any other string.

### String expressions

- **Concatenation** uses `||` (two vertical bars), not `+`:
  ```sql
  sqlite> SELECT "hello," || " world";
  hello, world
  ```
- **Partial match** uses `LIKE` with `%` as a wildcard, and it is case-insensitive:
  ```sql
  sqlite> SELECT title FROM titles WHERE title LIKE "%dog%";
  Dog Day Afternoon
  Reservoir Dogs
  Ghost Dog: The Way of the Samurai
  Dogville
  Slumdog Millionaire
  Hachi: A Dog's Tale
  Isle of Dogs
  ```
  `%dog%` means "contains dog anywhere". Case insensitivity is why `Dogville` and `Slumdog` both match.

The lecture is deliberately dismissive of SQL's deeper string manipulation (`substr`, finding positions with `instr`, and so on): the language's support is weak, and these functions are used more than they should be. Worse is using strings to fake structured values, for example storing a linked list as a `cdr` column holding `"2,3,4"` and then digging out the "cadr" by hunting for the first comma. It is possible. It is a bad idea. `knownForTitles` in the real IMDb data is exactly this anti-pattern.

## Definitions

- **Declarative programming**: A programming style in which a program describes the desired result rather than the procedure for computing it; the interpreter determines how to generate the result.
- **Index**: A separate data structure maintained by the database that records where particular values appear in a table, letting the engine find matching rows without scanning the whole table.
- **Join**: An operation that combines a row from one table with a row from another table into a single longer row containing all columns of both. Written with the keyword `JOIN` or with a comma in the `FROM` clause.
- **Cross join**: A join with no matching condition, producing every possible combination of a row from the first table and a row from the second. If A has *m* rows and B has *n* rows, the result has *m × n* rows.
- **`ON` clause**: The part of a join that states the condition two rows must satisfy to be combined into one output row. Mechanically equivalent to putting the condition in `WHERE`, but conventionally reserved for row-matching conditions.
- **Dot expression** (in SQL): `table.column`, used to disambiguate a column name that appears in more than one table of the query, for example `titles.tconst`.
- **Table alias**: An alternate, usually shorter name for a table within a single `SELECT` statement, introduced with `AS` (for example `titles AS t`). Required when the same table appears more than once in a join.
- **Self-join**: A join of a table with itself, used when one output row depends on two different input rows of the same table. Requires aliases.
- **Bridge (junction) table**: A table whose purpose is to relate two other tables, holding a key from each. `principals` bridges `titles` and `names`. *(extra context: "junction table" is standard database terminology, not a term used in this lecture.)*
- **Explicit join syntax**: `FROM a JOIN b ON condition`.
- **Implicit join syntax**: `FROM a, b WHERE condition`.
- **`||`**: The SQL string concatenation operator.
- **`LIKE`**: A string comparison operator for partial, case-insensitive matching, where `%` matches any sequence of characters.
- **`!=` / `<>`**: The two equivalent SQL spellings of not-equal.
- **Column alias**: A name given to an output column with `AS`, for example `old.year AS first`.

## Worked Examples

### 1. Two-table join: titles with their ratings

```sql
SELECT title, avgRating
  FROM titles JOIN ratings ON titles.tconst = ratings.tconst;
```

**Step by step:**

1. `FROM titles JOIN ratings` names the two tables being combined.
2. `ON titles.tconst = ratings.tconst` tells SQL to keep only those pairings where the movie identifier agrees. Conceptually the engine considers each (titles row, ratings row) pair and tests the condition; in practice it may use an index on `tconst` to jump straight to matches.
3. Each surviving pair becomes one 8-column row (5 from `titles` + 3 from `ratings`, in the full schema).
4. `SELECT title, avgRating` projects out just the two interesting columns. `tconst` did its job during matching and is discarded.

**Why `ON` is doing real work:** without it, `Mary Poppins` would appear paired with every rating in the table. The matching condition is the only reason row content lines up with row meaning.

**Extending with sorting:**

```sql
SELECT title, avgRating
  FROM titles JOIN ratings ON titles.tconst = ratings.tconst
  ORDER BY numVotes DESC LIMIT 5;
```

`ORDER BY numVotes` ascending gives the least-voted films; `DESC` reverses it, surfacing the most-voted (the lecture mentions *The Shawshank Redemption* dominating here). Note `numVotes` is used in `ORDER BY` even though it is not in the `SELECT` list: it is available because the join put it in the long row.

### 2. Two-table join through the bridge: actors and their characters

```sql
SELECT names.name, principals.character
  FROM principals JOIN names ON principals.nconst = names.nconst
  WHERE principals.category = "actor";
```

**Step by step:**

1. One output row needs a person's *name* (lives in `names`) and a *character* (lives in `principals`). Two tables, so one join.
2. `ON principals.nconst = names.nconst` guarantees the character and the name describe the same person.
3. `WHERE principals.category = "actor"` filters down to acting credits only. This is a single-table filter, so it belongs in `WHERE`, not `ON`.
4. The dot expressions `names.name` and `principals.character` are optional here, since `name` and `character` each appear in only one of the two tables. `nconst` appears in both, so *it* genuinely needs disambiguation.

### 3. Three-table join: who played James Bond?

Goal: for each James Bond film, show the title and the name of the actor who played Bond.

```sql
SELECT titles.title, names.name
  FROM titles JOIN principals ON titles.tconst = principals.tconst
              JOIN names      ON principals.nconst = names.nconst
  WHERE principals.character = "James Bond";
```

**Reasoning through it:**

An output row is `(title, name)`. The title comes from `titles`. The name comes from `names`. But those two tables share nothing, so a third table is needed to connect them: `principals`, which knows both which film (`tconst`) and which person (`nconst`) a credit refers to. Hence three tables in `FROM` even though only two contribute output columns.

Two matching conditions are needed, forming a chain:

- `titles.tconst = principals.tconst`: the credit is for this film.
- `principals.nconst = names.nconst`: the credit is for this person.

**There is deliberately no condition connecting `titles` directly to `names`.** The instructor calls this out explicitly: the connection is *transitive* through the `principals` row. Writing a fourth condition would be meaningless since those tables have no comparable columns.

With the slides' sample data, `character = "James Bond"` matches `tconst = tt0058150` (*Goldfinger*) and `nconst = nm0000125`, which `names` resolves to `Sean Connery`:

```
title        name
----------   ------------
Goldfinger   Sean Connery
```

Observe that `James W. Gettys`, `James Conway`, `James Lingk`, `James Cole`, and `J. Jonah Jameson` all contain "James" but none equal `"James Bond"`, so `=` excludes them correctly. (If you used `LIKE "%James%"` you would wrongly pull in all of them.)

### 4. What goes wrong when you omit a matching condition

The lecture demonstrates this live. The goal is title, actor name, and character for 2023 films, first actor only, but one condition is deliberately left out:

```sql
SELECT titles.title, names.name, principals.character
  FROM titles, names, principals
  WHERE titles.tconst = principals.tconst
    AND titles.year = 2023
    AND principals.ordering = 1;
```

**What happens:** The query takes a long time (it is building a huge intermediate table) and the output is nonsense. The film and the character match up correctly: *Mission: Impossible* pairs with `Ethan Hunt`, because `titles.tconst = principals.tconst` is present. But the name is wrong, because **nothing constrains `names`**. Every row of `names` gets paired with every (film, role) combination. If `names` has *k* rows, the result is *k* times too long, and `Ethan Hunt` appears next to every person in the database instead of `Tom Cruise`.

**The fix** is one more condition:

```sql
SELECT titles.title, names.name AS star
  FROM titles, names, principals
  WHERE titles.tconst = principals.tconst
    AND principals.nconst = names.nconst
    AND titles.year = 2023
    AND principals.ordering = 1;
```

**Rule of thumb:** with *n* tables in the `FROM` clause you generally need *n - 1* matching conditions to tie them into one coherent chain. Any table left unconstrained multiplies your output by that table's row count.

Here is the same query with aliases, which is what the slides show:

```sql
SELECT t.title, n.name AS star
  FROM titles AS t, principals AS p, names AS n
  WHERE t.tconst = p.tconst
    AND p.nconst = n.nconst
    AND t.year = 2023
    AND p.ordering = 1;
```

Identical meaning, less typing. The instructor also notes all of these conditions could equally sit in `ON` clauses; the best practice is `ON` for the matching conditions and `WHERE` for the query-specific filters (`year`, `ordering`).

### 5. Self-join: finding remakes

Goal: pairs of films with the same title made in different years.

```sql
SELECT old.title, old.year AS first, new.year AS second
  FROM titles AS old JOIN titles AS new
    ON old.title = new.title AND old.year < new.year;
```

**Step by step:**

1. **Start with `FROM`.** The instructor's advice: figure out the tables first, columns last. An output row needs two years for the same title, and years live in `titles`, so `titles` must appear twice.
2. `AS old` and `AS new` are **required**, not stylistic. Without them `title` would be hopelessly ambiguous.
3. `ON old.title = new.title`: same film name.
4. `AND old.year < new.year`: this does two jobs at once.
   - It excludes a row joining with *itself* (a row cannot have `year < year`).
   - It fixes the order, so each remake pair appears once as (earlier, later) rather than twice in both orders. Using `!=` instead of `<` would give you both `(2010, 2025)` and `(2025, 2010)`.
5. `SELECT old.title` picks the title (either copy works, they are equal by the `ON` condition), and the column aliases `first` / `second` make the output readable.

```
title                             first   second
-------------------------------   -----   ------
How to Train Your Dragon          2010    2025
The Girl with the Dragon Tattoo   2009    2011
```

**Thinking about rows:** the original `titles` table contains two separate rows named *How to Train Your Dragon*, with different `tconst` values, different runtimes, even different genres (one is animated). They are distinct rows with equal `title` values. The join pairs those two input rows to produce one output row. That is the signature of a self-join: **one output row built from two input rows of the same table.**

### 6. Self-join: actors playing two characters in one film

Goal: a table with columns `lead` and `other`, holding pairs of characters from the same film played by the same actor, where `lead` is the `ordering = 1` actor.

Only `principals` is needed. The film's name and the actor's name are irrelevant to the output; `principals` alone contains character names, `tconst`, `nconst`, and `ordering`.

```sql
SELECT a.character AS lead, b.character AS other
  FROM principals AS a JOIN principals AS b
    ON a.tconst = b.tconst AND a.nconst = b.nconst
  WHERE a.ordering = 1 AND b.ordering > 1
    AND a.category = "actor" AND b.category = "actor";
```

**Step by step:**

1. Two characters from the same film means two `principals` rows, so a self-join with aliases `a` and `b`.
2. `a.tconst = b.tconst`: both credits belong to the same film.
3. `a.nconst = b.nconst`: both credits belong to the same person. This is the condition that makes the query *about dual roles* rather than about any two cast members.
4. `a.ordering = 1`: `a` is the lead.
5. `b.ordering > 1`: `b` is a different, later credit. This also prevents `a` and `b` being the same row.
6. Both must be `category = "actor"`.

Results include `Superman` / `Clark Kent` and, from *Back to the Future*, Michael J. Fox as `Marty McFly` plus his son, his daughter, and his great-great-great-grandfather.

### 7. Five-table join: co-stars of 2024 films

Goal: one row per 2024 film with the title, the name of the first-billed actor, and the name of the second-billed actor, excluding cases where they are the same person.

**Count the rows an output row needs** (this is the whole technique):

| Output piece | Source |
| --- | --- |
| the film's title | a row of `titles` |
| proof that actor 1 was in this film, and who they are | a row of `principals` |
| proof that actor 2 was in this film, and who they are | another row of `principals` |
| actor 1's human-readable name | a row of `names` |
| actor 2's human-readable name | another row of `names` |

Five rows, so five table references, so four matching conditions.

```sql
SELECT title, first.name AS first, second.name AS second
  FROM titles
       JOIN principals AS pfirst  ON titles.tconst = pfirst.tconst
       JOIN principals AS psecond ON titles.tconst = psecond.tconst
       JOIN names      AS first   ON pfirst.nconst = first.nconst
       JOIN names      AS second  ON psecond.nconst = second.nconst
  WHERE titles.year = 2024
    AND pfirst.ordering = 1
    AND psecond.ordering = 2
    AND pfirst.nconst != psecond.nconst
    AND pfirst.category = "actor"
    AND psecond.category = "actor";
```

**Reading the conditions in two groups:**

*Coherence conditions* (make the five rows describe one situation):
- `titles.tconst = pfirst.tconst` and `titles.tconst = psecond.tconst`: both credits are for this film. All three film-related rows now agree.
- `pfirst.nconst = first.nconst` and `psecond.nconst = second.nconst`: each name goes with its own credit, not the other one's.

*Query conditions* (narrow to what was asked):
- `year = 2024`
- `ordering = 1` and `ordering = 2`: top billing and second billing
- `pfirst.nconst != psecond.nconst`: excludes one actor credited twice, the "Batman and Bruce Wayne" case
- both `category = "actor"`

The instructor's warning: **leaving out any one of these gives a much longer and much less correct result.** Note also that the five rows do *not* all have to be about the same person (two different people are the point), but they do all have to be about the same movie.

Running it from a file:

```
$ sqlite3 -init costars.sql imdb1000.db
```

Sample output: *Furiosa: A Mad Max Saga* with Anya Taylor-Joy and Chris Hemsworth; *Dune: Part Two* with Timothée Chalamet and Zendaya.

### 8. Strings: highest-rated comedies

```sql
SELECT title, avgRating
  FROM titles JOIN ratings ON titles.tconst = ratings.tconst
  WHERE genres LIKE "%Comedy%"
  ORDER BY avgRating DESC LIMIT 10;
```

**Step by step:**

1. Join to get ratings alongside titles.
2. `genres` is a comma-separated string like `"Comedy,Drama,Family"`, so exact equality would fail on almost everything. `LIKE "%Comedy%"` matches `Comedy` anywhere in the string, case-insensitively.
3. `ORDER BY avgRating DESC` puts the best first; `LIMIT 10` takes the top ten.

This `genres` column is the string-as-structured-value anti-pattern the lecture criticizes: a properly normalized schema would have a separate `genres(tconst, genre)` table, and then this would be a clean join instead of a substring hunt.

### 9. Self-join with arithmetic: north-south distance between cities

The warm-up demo, using a `cities(latitude, longitude, name)` table with Berkeley, Cambridge, Minneapolis, San Diego, Miami, and the North Pole.

Simple filter and a named result:

```sql
CREATE TABLE cold AS
  SELECT name FROM cities WHERE latitude >= 43;
```

```sql
sqlite> SELECT name FROM cold;
Minneapolis
North Pole
```

Adding a row that is not in `cities` with `UNION`:

```sql
SELECT name FROM cold UNION SELECT "Chicago";
```

Now the self-join with arithmetic. One degree of latitude is 60 nautical miles, so:

```sql
CREATE TABLE distances AS
  SELECT a.name AS first, b.name AS second,
         60 * (b.latitude - a.latitude) AS distance
    FROM cities AS a JOIN cities AS b;
```

**Step by step:**

1. One output row needs **two** cities' latitudes, so `cities` appears twice with aliases `a` and `b`.
2. `60 * (b.latitude - a.latitude)` is an ordinary arithmetic expression in the column list, aliased to `distance`. Negative means `b` is south of `a`.
3. This is intentionally a **cross join** (no `ON`): every ordered pair of cities is wanted here, which is one of the rare legitimate uses.

Querying it:

```sql
SELECT second FROM distances WHERE first = "Minneapolis" ORDER BY distance;
```

Ascending order puts the most negative distance first, so the farthest-south destinations lead: Miami, then San Diego, with the North Pole last.

### 10. Strings: generating sentences

```sql
CREATE TABLE nouns AS
  SELECT "dog" AS phrase UNION
  SELECT "cat"           UNION
  SELECT "bird";

SELECT subject.phrase || " chased " || object.phrase
  FROM nouns AS subject JOIN nouns AS object
  WHERE subject.phrase != object.phrase;
```

**Step by step:**

1. A sentence needs two nouns, so `nouns` self-joins with aliases `subject` and `object`.
2. `||` concatenates the subject, the literal `" chased "`, and the object.
3. `subject.phrase != object.phrase` prevents "the dog chased the dog".

Output: six sentences (3 × 3 = 9 pairs, minus 3 self-pairs).

Then compound noun phrases, built by querying the result of the same pattern:

```sql
CREATE TABLE ands AS
  SELECT first.phrase || " and " || second.phrase AS phrase
    FROM nouns AS first JOIN nouns AS second
    WHERE first.phrase != second.phrase;

SELECT subject.phrase || " chased " || object.phrase
  FROM ands AS subject JOIN ands AS object
  WHERE subject.phrase != object.phrase;
```

The second query is structurally identical to the first; only the source table changed. Output: "the dog and the bird chased the cat and the dog", and so on. This shows how a derived table composes: the result of one query is just another table you can join.

## Common Pitfalls

1. **Forgetting the `ON` clause.** This does not error. You silently get a cross join: *m × n* rows, most of them nonsense, and output that looks plausible. If a query is unexpectedly slow or returns far too many rows, count your matching conditions first.

2. **Under-constraining a multi-table join.** With *n* tables you generally need *n - 1* matching conditions. The lecture's live demo leaves out `principals.nconst = names.nconst` and gets every person paired with every role. Each unconstrained table multiplies the output.

3. **Expecting SQL to infer the relationship from column names.** Two tables both having `tconst` means nothing to the engine. You must write the condition.

4. **Omitting aliases in a self-join.** Referring to `titles` when `titles` appears twice is an error; the engine cannot tell which copy you mean. Aliases are required, not optional, for self-joins.

5. **Forgetting a tie-breaking condition in a self-join.** Without something like `old.year < new.year` or `b.ordering > 1`, every row joins with itself (producing bogus "remakes" where a film is its own remake) and every genuine pair appears twice in both orders. Using `!=` where `<` is correct gives duplicates.

6. **Using `+` for string concatenation.** SQL uses `||`. `+` on strings will not do what you expect.

7. **Using `==` for equality.** SQL equality is a single `=`. There is no assignment to conflict with it.

8. **Using `=` on a comma-separated column like `genres`.** `genres = "Comedy"` matches almost nothing because real values look like `"Comedy,Drama,Family"`. Use `LIKE "%Comedy%"`. Conversely, do not use `LIKE "%James%"` where you mean exactly `"James Bond"`: it will also match `James Conway` and `J. Jonah Jameson`.

9. **Forgetting `LIKE` is case-insensitive.** Convenient usually, surprising occasionally.

10. **Selecting `*` from a join and keeping redundant columns.** A join of titles and ratings gives two identical `tconst` columns. Project down to what you need.

11. **Putting matching conditions in `WHERE` and filters in `ON`.** Legal, but it makes queries hard for humans to read. Match in `ON`, filter in `WHERE`.

12. **Writing two `WHERE` clauses.** Only one is allowed; chain conditions with `AND`.

13. **Treating `'\N'` as SQL `NULL`.** In this dataset it is the literal two-character string `\N`, compared with `=` and `!=` like any string.

14. **Writing the `SELECT` columns first.** The lecture's repeated advice: start with `FROM` (which tables and how many copies), then the matching conditions, then filters, and only then the output columns. Working in that order prevents under-constrained joins.

## Likely Exam Points

### 1. Writing a two-table join

**Q:** Given `titles(tconst, title, year)` and `ratings(tconst, avgRating, numVotes)`, write a query selecting the titles of films from 2020 or later that have an `avgRating` above 8.0.

**A:**
```sql
SELECT title
  FROM titles JOIN ratings ON titles.tconst = ratings.tconst
  WHERE year >= 2020 AND avgRating > 8.0;
```
The join condition goes in `ON`; both filters are single-table conditions and go in `WHERE`, chained with one `AND`.

### 2. Predicting cross-join output size

**Q:** Table `A` has 4 rows and 2 columns; table `B` has 5 rows and 3 columns. How many rows and columns does `SELECT * FROM A, B;` produce?

**A:** 20 rows (4 × 5) and 5 columns (2 + 3). A comma with no `WHERE` is a cross join, giving every pairing, and `*` keeps all columns from both tables.

### 3. Counting required tables and conditions

**Q:** You want one row per film giving the title and the name of the director. `titles(tconst, title, ...)`, `principals(tconst, ordering, nconst, category, character)`, `names(nconst, name, ...)`. How many tables must appear in `FROM`, and how many matching conditions do you need?

**A:** Three tables (`titles`, `principals`, `names`) and two matching conditions (`titles.tconst = principals.tconst` and `principals.nconst = names.nconst`). `principals` contributes no output columns but is required to connect the other two, which share no column. Then `WHERE principals.category = "director"`.

### 4. Diagnosing a broken join

**Q:** This query returns a huge table in which every actor name appears with every film. What is wrong?
```sql
SELECT titles.title, names.name
  FROM titles, principals, names
  WHERE titles.tconst = principals.tconst AND principals.ordering = 1;
```

**A:** `names` is unconstrained: no condition links it to the other tables, so every `names` row is paired with every matching (titles, principals) pair, multiplying the output by the size of `names`. Add `AND principals.nconst = names.nconst`.

### 5. Recognizing when a self-join is needed

**Q:** Which of these require joining a table with itself? (a) Each film's title and rating. (b) Pairs of actors born in the same year. (c) Each film's title and runtime. (d) Pairs of films released in the same year with different titles.

**A:** (b) and (d). The test: does one output row depend on two different input rows of the same table? (b) needs two `names` rows; (d) needs two `titles` rows. (a) joins two *different* tables; (c) needs just one row of one table.

### 6. Writing a self-join with correct tie-breaking

**Q:** Using `names(nconst, name, birth, death)`, write a query producing pairs of distinct people born in the same year, with each pair appearing only once.

**A:**
```sql
SELECT a.name, b.name, a.birth
  FROM names AS a JOIN names AS b
    ON a.birth = b.birth AND a.nconst < b.nconst
  WHERE a.birth != '\N';
```
Aliases are mandatory. `a.nconst < b.nconst` both prevents a row joining with itself and ensures each pair appears once rather than in both orders. (Using `!=` there would give every pair twice.) The `birth != '\N'` guard avoids pairing up everyone with an unknown birth year.

### 7. Translating between join syntaxes

**Q:** Rewrite using explicit `JOIN` / `ON` syntax:
```sql
SELECT name, character FROM principals, names
  WHERE principals.nconst = names.nconst AND category = "actress";
```

**A:**
```sql
SELECT name, character
  FROM principals JOIN names ON principals.nconst = names.nconst
  WHERE category = "actress";
```
The comma becomes `JOIN`, the matching condition moves to `ON`, and the single-table filter stays in `WHERE`.

### 8. String operators

**Q:** Write a query that returns, for each living person in `names` whose name contains "ana", a single string of the form `"NAME (born YEAR)"`. Birth years are integers, and `'\N'` marks unknown.

**A:**
```sql
SELECT name || " (born " || birth || ")"
  FROM names
  WHERE name LIKE "%ana%" AND death = '\N' AND birth != '\N';
```
`||` for concatenation, not `+`. `LIKE` with `%` on both sides for "contains", and it is case-insensitive so `Ana` and `ANA` also match.

### 9. Arithmetic in expressions

**Q:** Using `names(name, birth, death)` where `'\N'` means unknown, select the names and ages at death of people who have died, oldest first.

**A:**
```sql
SELECT name, death - birth AS age
  FROM names
  WHERE death != '\N' AND birth != '\N'
  ORDER BY age DESC;
```
Arithmetic is allowed directly in the column list; `AS` names the computed column so `ORDER BY` can reference it.

### 10. Why declarative programming matters

**Q:** The query `SELECT name FROM names WHERE death = '\N';` can be executed by scanning every row or by consulting an index on `death`. Who decides, and what does this illustrate?

**A:** The database decides. This illustrates declarative programming: the query states *what* result is wanted, not *how* to compute it, so the engine is free to pick the fastest strategy. The same query text can get faster when an index is added, with no change to the program.

### 11. `ON` versus `WHERE`

**Q:** Is `FROM a JOIN b ON a.x = b.x WHERE a.y = 5` equivalent to `FROM a JOIN b ON a.x = b.x AND a.y = 5`? Should you write it either way?

**A:** For an inner join the two are equivalent to SQL: both are just conditions. The style convention is to put row-matching conditions (`a.x = b.x`) in `ON` and single-table filters (`a.y = 5`) in `WHERE`, because that makes the query's structure readable. *(extra context: for outer joins, which this lecture does not cover, `ON` and `WHERE` are genuinely not interchangeable.)*

## Summary

- SQL is **declarative**: the query describes the result, the database chooses the execution strategy (full scan, or an **index** lookup).
- A **join** concatenates a row from one table with a row from another into one long row; `SELECT *` on a join returns all columns of both.
- `JOIN` and a **comma** mean the same thing; `ON` and `WHERE` are mechanically interchangeable for inner joins. Explicit `JOIN ... ON` is the modern convention, but older code uses commas.
- With no matching condition you get a **cross join**: all *m × n* pairings. It does not error, it just produces mostly meaningless rows.
- **SQL will not infer** how to match rows. Shared column names mean nothing; you must write the condition.
- Rule of thumb: ***n* tables in `FROM` need *n - 1* matching conditions.** Any unconstrained table multiplies your output by its row count.
- A table can appear in `FROM` **without contributing output columns**. `principals` bridges `titles` and `names`, which share no column.
- Matching conditions **chain** (titles to principals to names). You do not write a direct condition between tables that have nothing comparable.
- **Design method:** think about building one output row. List the pieces it needs and where each lives. That gives you the `FROM` clause. Write `FROM` first, then matching conditions, then filters, then the columns.
- **Self-join** when one output row depends on **two rows of the same table** (remakes, dual roles, co-stars, city pairs). **Aliases with `AS` are required.**
- Self-joins need a tie-break (`old.year < new.year`, `b.ordering > 1`, `a.nconst < b.nconst`) to exclude self-pairs and avoid each pair appearing twice.
- Aliases also shorten dot expressions in ordinary joins (`principals AS p`, then `p.tconst`).
- Numeric expressions mirror Python: arithmetic, `abs`, `round`, comparisons. Equality is a single `=`; not-equal is `!=` or `<>`.
- Strings concatenate with `||`, not `+`. `LIKE` with `%` wildcards does case-insensitive partial matching.
- Storing structured data in strings (`genres`, `knownForTitles`, comma-separated lists) works but is a bad design; SQL's string manipulation is weak and overused.
