<!-- Fri, Nov 20, 2026 | sources: slides + YouTube auto-transcript -->
# Lecture 36: Aggregation

## Overview

This lecture covers SQL **aggregation**: computing a single value from a group of rows instead of processing one row at a time. Up to now, every SQL expression we wrote referred to values within a single row, and each input row produced one output row. Aggregate functions (`COUNT`, `MAX`, `MIN`, `SUM`, `AVG`) break that one-to-one correspondence: they collapse many rows into one. The `GROUP BY` clause partitions the input rows into groups based on the unique values of an expression, so aggregation happens once per group rather than once over the whole table, yielding one output row per group. The `HAVING` clause then filters which *groups* survive into the output, in contrast to `WHERE`, which filters which *input rows* enter the computation at all. The lecture closes with a methodology for writing select statements (decide input rows, then form and filter groups, then format output) and applies it to multi-table queries on the IMDb dataset that combine `JOIN`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, and `LIMIT` at once.

## Key Concepts

### Aggregate functions collapse rows

Every SQL expression we have written so far refers to values in a single row. A statement processes each row using the expressions in the `SELECT` clause and generates one output row from each input row. An **aggregate function** placed in the columns region of a `SELECT` statement computes a value from a *group* of rows. By default, if there is no `GROUP BY`, the group is **all rows** in the input.

The lecture uses a small `animals` table (kinds of animals, number of legs, typical weight). With rows for ferret/dog/cat (4 legs) and parrot/penguin/t-rex (2 legs):

```sql
SELECT MAX(legs) FROM animals;
```

This takes the max over all `legs` values in the table and returns `4`. The crucial observation: **there were six rows in the input but only one row in the output**. That is the defining property of aggregate functions, they change the output to have only one row (per group).

### The aggregate functions covered

- `COUNT(*)`: the number of rows in a group.
- `MAX([expression])`: the largest value of `[expression]` for any row in the group.
- `MIN([expression])`: the smallest value.
- `SUM([expression])`: the total.
- `AVG([expression])`: the average.

### Aggregation applies to an expression, not just a column

`MAX` does not pull information across rows inside its argument. In the lecture:

```sql
SELECT MAX(legs - weight) FROM animals;
```

For **each row** we compute `legs - weight`, and then we find the largest result among those per-row values. The answer came from the parrot (2 legs, weight 6). It did **not** subtract 6 from 4 (max legs minus min weight). The instructor's debugging tip: if you want to see the steps, first run `SELECT legs - weight FROM animals;` to see exactly what values the `MAX` is being taken over.

### Multiple aggregate functions in one expression are independent aggregations

If you *do* want to compare the maximum legs against the minimum weight, use two aggregate functions:

```sql
SELECT MAX(legs) - MIN(weight) FROM animals;
```

Here the maximum legs came from the ferret/dog/cat rows while the minimum weight came from the parrot row. **These do not have to come from the same row.** They are two separate aggregations over the same group, and the subtraction happens on the two resulting scalars. You may also list them as two separate columns (`SELECT MAX(legs), MIN(weight) FROM animals;`); both forms are fine.

### Do not mix aggregated and non-aggregated columns

```sql
SELECT MAX(legs), weight FROM animals;   -- don't do this
```

SQLite will give you *a* result, but other SQL database systems will raise an error, and the instructor's point is that it does not make sense semantically. The query asks for one row. `MAX(legs)` is well-defined. But "the weight" is the weight of *what*? There were multiple weight values in the group, and we never said how to combine them. SQLite returns one, but that does not tell you which one it was or why. The rule: **either perform aggregation for all the columns, or don't aggregate at all.** Not aggregating is fine, you just end up with six rows in the output. But if you aggregate, aggregate everything.

The one legitimate exception appears with `GROUP BY` (see below).

### Why you cannot average averages: the weighted average

The IMDb `ratings` table has 1,000 rows, one per movie, each with an `averageRating` (the average of that movie's votes) and `numVotes` (how many votes that movie received). To get the average rating across **all the votes**, you cannot just average the `averageRating` column, because different movies received different numbers of votes.

The lecture's two-movie illustration: movie 1 has 80 votes averaging 7; movie 2 has 20 votes averaging 9. The overall average is **not** the average of 7 and 9 (which would be 8). It is closer to 7, because there were far more votes of 7:

```
(80 × 7 + 20 × 9) / (80 + 20) = (560 + 180) / 100 = 7.4
```

Generalized in SQL:

```sql
SELECT SUM(averageRating * numVotes) / SUM(numVotes) FROM ratings;
```

For each row, multiply the average rating by the number of votes to recover the total rating points contributed by that movie. Summing those gives the total rating points across the dataset; dividing by the total number of votes gives the true average. In this dataset the result was 8, not because of rating inflation but because the dataset was deliberately built from 1,000 highly-rated movies.

### GROUP BY: aggregation per group

```sql
SELECT [columns] FROM [table] GROUP BY [expression] HAVING [expression];
```

- `GROUP BY [expression]`: produces **one output row for each unique value of the expression**.
- `HAVING [expression]`: keeps only the groups for which the expression is true.

With the `animals` table:

```sql
SELECT legs, MAX(weight) FROM animals GROUP BY legs;
```

There are two unique values of `legs` (4 and 2), so there are two groups and two output rows: the four-legged beasts and the two-legged beasts. The result shows the heaviest four-legged animal at 20 and the heaviest two-legged animal at 12,000 (the t-rex).

### Why a bare `legs` is legal here

Writing `legs` without aggregating it is fine in this query precisely because we grouped by `legs`. Within each output group, there is only one `legs` value shared by every row in that group, so naming it unambiguously identifies the group. This is the exception to the "aggregate everything or nothing" rule: an expression you grouped by is safe to select directly.

### GROUP BY on an arbitrary expression

You can group by any expression, not just a column:

```sql
SELECT weight / legs, COUNT(*) FROM animals GROUP BY weight / legs HAVING COUNT(*) > 1;
```

Integer division (floor division) produces values like 5, 2, 3, 5, 6000 for the various animals. Some of those repeat: two 2s and two 5s. The `HAVING COUNT(*) > 1` clause performs an aggregation for each group to count its rows and keeps the group only if it has more than one row. So although there were four distinct values of the grouping expression, only two groups survive to the output; the two singleton groups are discarded.

### WHERE vs HAVING

This distinction is the conceptual heart of the lecture:

- **`WHERE`** decides whether to keep an **individual input row**. Because it looks at one row at a time, you **cannot** use an aggregate function in a `WHERE` expression.
- **`HAVING`** decides whether to keep a **whole group**. Because it asks about a property of the entire group, you **can** use an aggregate function in a `HAVING` expression.

### Methodology for writing select statements

The lecture gives an explicit recipe. Describe the output table, then:

1. Determine which existing rows are needed to express the result (`FROM` and `WHERE`).
2. Form groups and determine which groups should appear as output rows (`GROUP BY` and `HAVING`).
3. Format the output rows (`SELECT`).

Clause-by-clause responsibilities:

| Clause | Role |
| --- | --- |
| `SELECT` | Values each output row contains (and column labels) |
| `FROM` | Source of input rows |
| `WHERE` | Which input rows |
| `GROUP BY` | Form output rows |
| `HAVING` | Which output rows |

### Column labels with AS

Output columns can be named:

```
[expression] AS [name], [expression] AS [name], ...
```

So `COUNT(*) AS total` labels the aggregate column `total`. An important practical benefit shown in the decade example: once you name a computed value, you can refer to that **name** in `ORDER BY` rather than retyping the whole expression.

### Why the IMDb data is split across tables

The lecture poses this as a poll question when showing `titles`, `ratings`, `principals`, and `names` as four separate tables. The query for actor ratings needs to join `names` (actor names), `principals` (which actor appeared in which title, and in what role), and `ratings` (the rating of each title). The implied answer is that splitting avoids repeating the same information in many rows: an actor's name is stored once in `names` rather than duplicated for every movie they appeared in, and a movie's rating is stored once in `ratings` rather than duplicated for every cast member. (extra context: this is the normalization principle in database design.)

## Definitions

- **Aggregation**: putting together information from multiple rows to compute a single value.
- **Aggregate function**: a function used in the columns clause of a `SELECT` statement that computes a value from a group of rows (or from all rows, if there are no groups). Its presence reduces the output to one row per group.
- **`COUNT(*)`**: aggregate function returning the number of rows in a group.
- **`MAX([expression])`**: aggregate function returning the largest value of `[expression]` for any row in a group. `MIN`, `SUM`, and `AVG` are analogous.
- **`GROUP BY [expression]`**: clause that partitions input rows into groups; the output contains one row for each unique value of `[expression]`.
- **`HAVING [expression]`**: clause that filters groups, keeping only those for which `[expression]` is true. Aggregate functions are permitted here.
- **`WHERE [expression]`**: clause that filters individual input rows. Aggregate functions are not permitted here.
- **`[expression] AS [name]`**: syntax for labeling an output column, allowing the name to be reused elsewhere in the statement (for example, in `ORDER BY`).
- **Floor division in SQL**: dividing one integer by another performs integer (floor) division. `43 / 10` evaluates to `4`.

## Worked Examples

> Note on language: this lecture is entirely SQL, so the examples below are SQL statements rather than Python. The notes preserve the lecture's own queries and reason through them row by row (the SQL analogue of box-and-pointer tracing: track which input rows land in which group, and what each aggregate sees).

### Example 1: Basic aggregation over all rows (`animals`)

```sql
SELECT MAX(legs) FROM animals;
```

Step by step:
1. `FROM animals` supplies all six input rows (ferret, dog, cat, parrot, penguin, t-rex).
2. There is no `GROUP BY`, so the entire set of six rows forms a single implicit group.
3. `MAX(legs)` scans the `legs` value of every row in that group and keeps the largest: `4`.
4. Output: one row, one column, value `4`.

The "pointer" intuition: instead of six separate output rows each pointing at one input row, there is a single output row that summarizes all six.

```sql
SELECT SUM(weight) FROM animals;
```

Same structure: one implicit group, one output row, holding the sum of all six weight values.

### Example 2: Aggregating an expression vs aggregating separately

```sql
SELECT MAX(legs - weight) FROM animals;
```

1. For **each** row independently, compute `legs - weight`. The parrot (2 legs, weight 6) gives `-4`; the t-rex gives a very large negative number; and so on.
2. `MAX` is applied to those six per-row results.
3. The winner is the parrot's row.

Contrast with:

```sql
SELECT MAX(legs) - MIN(weight) FROM animals;
```

1. `MAX(legs)` aggregates the whole group to `4` (from ferret, dog, or cat).
2. `MIN(weight)` aggregates the whole group independently to the parrot's weight of `6`.
3. Only then does subtraction happen, on two scalars: `4 - 6`.

The two queries give different answers because the first never crosses row boundaries inside the subtraction, while the second deliberately combines results from two different rows. This is the single most useful distinction in the lecture for debugging aggregate expressions.

### Example 3: The illegal mixed query

```sql
SELECT MAX(legs), weight FROM animals;
```

Walk through why it is incoherent: the output must be one row. `MAX(legs)` is `4`. But four different animals had four different weights and no aggregation was specified for `weight`, so there is no principled answer. SQLite will print some weight value, but you cannot say which row it came from or why. Other database systems reject the query outright. Fix it by aggregating:

```sql
SELECT MAX(legs), AVG(weight) FROM animals;
```

The lecture notes the average weight comes out to roughly 2,000, dragged up entirely by the t-rex. (This is a good reminder that `AVG` is sensitive to outliers.)

### Example 4: Weighted average across the whole ratings table

```sql
SELECT SUM(averageRating * numVotes) / SUM(numVotes) FROM ratings;
```

1. `FROM ratings` supplies 1,000 rows, one per movie.
2. No `GROUP BY`, so all 1,000 rows form one group.
3. Inside the first `SUM`, each row contributes `averageRating * numVotes`, the total rating points that movie's voters awarded. Summing gives total rating points across the dataset.
4. The second `SUM` independently totals `numVotes`, the number of votes cast overall.
5. Dividing yields the vote-weighted average: `8` for this dataset.

Why the naive version is wrong: `SELECT AVG(averageRating) FROM ratings;` weights a movie with 100 votes exactly as heavily as a movie with 100,000 votes.

### Example 5: GROUP BY with a bare grouped column

```sql
SELECT legs, MAX(weight) FROM animals GROUP BY legs;
```

1. `FROM animals` supplies six rows.
2. `GROUP BY legs` scans the `legs` values and finds two unique ones: `4` and `2`. Two groups form. Group "4" holds ferret, dog, cat. Group "2" holds parrot, penguin, t-rex.
3. One output row is produced per group.
4. For group "4": `legs` is unambiguously `4` (every row in the group agrees), and `MAX(weight)` over ferret/dog/cat is `20`.
5. For group "2": `legs` is `2`, and `MAX(weight)` over parrot/penguin/t-rex is `12000`.

Output:

```
legs  MAX(weight)
----  -----------
2     12000
4     20
```

### Example 6: GROUP BY an expression, filtered by HAVING

```sql
SELECT weight / legs, COUNT(*) FROM animals GROUP BY weight / legs HAVING COUNT(*) > 1;
```

1. For each row compute `weight / legs` using integer division. The lecture's values: `5`, `2`, `3`, `5`, `6000` (and another `2`).
2. `GROUP BY` forms a group per unique value: a group for `5` (two rows), a group for `2` (two rows), a group for `3` (one row), a group for `6000` (one row). Four groups.
3. `HAVING COUNT(*) > 1` evaluates `COUNT(*)` **within each group**. The `5` and `2` groups have 2 rows and pass. The `3` and `6000` groups have 1 row and are discarded.
4. Output: two rows, even though the grouping expression had four distinct values.

Note that this filtering is impossible in `WHERE`: when `WHERE` runs, no groups exist yet and `COUNT(*)` has nothing to count over.

### Example 7: Count rows per category (`principals`)

> From the slides. For each category, select the category and the total number of rows for that category.

```sql
SELECT category, COUNT(*) AS total FROM principals GROUP BY category;
```

With the slide's sample rows (three `actor` rows and two `director` rows):

1. `FROM principals` supplies all rows.
2. `GROUP BY category` creates one group per unique category: `actor`, `director`, and so on.
3. The `actor` group gathers `tt0012349/nm0701012`, `tt0012349/nm0001067`, `tt0017136/nm0375609`. The `director` group gathers `tt0012349/nm0000122` and `tt0013442/nm0003638`.
4. `category` is safe to select bare because we grouped by it. `COUNT(*) AS total` counts rows per group, labeled `total`.

Output (for the slide excerpt):

```
category  total
--------  -----
actor     3
director  2
```

### Example 8: Actors with more than 15 characters played

> From the slides. Select the `nconst` and the total number of characters for each actor who had more than 15 characters played.

```sql
SELECT nconst, COUNT(*)
  FROM principals
  WHERE category = "actor"
  GROUP BY nconst
  HAVING COUNT(*) > 15;
```

Reading it through the methodology:

1. **Which input rows?** `FROM principals`, restricted by `WHERE category = "actor"` so that directors never enter the computation. This is per-row filtering, so it belongs in `WHERE`.
2. **Form groups.** `GROUP BY nconst` creates one group per distinct person, gathering all of that actor's credit rows.
3. **Which groups?** `HAVING COUNT(*) > 15` keeps only actors with more than 15 rows in their group. This is a property of the whole group, so it belongs in `HAVING`, not `WHERE`.
4. **Format output.** `SELECT nconst, COUNT(*)`: `nconst` is bare but legal (we grouped by it), and `COUNT(*)` reports the group size.

The important structural lesson is that both filters coexist and do different jobs: `WHERE` prunes rows before grouping, `HAVING` prunes groups after.

### Example 9: Remakes with the same title

> From the slides. Create a table of remakes that have the same title, with columns `title`, `original`, `remake`.

```sql
SELECT title, MIN(year) AS first, MAX(year) AS second
  FROM titles
  GROUP BY title
  HAVING COUNT(*) > 1;
```

Step by step:
1. `FROM titles` supplies every title row.
2. `GROUP BY title` gathers all rows sharing the same title string. A unique title lands in a group of size 1; a remade title lands in a group of size 2 or more.
3. `HAVING COUNT(*) > 1` discards every title that appears only once, leaving exactly the titles that were made more than once.
4. `MIN(year)` finds the earliest release in the group (the original) and `MAX(year)` finds the latest (the remake). Note again that these two aggregates read different rows of the same group, which is precisely what we want.

Target output shape:

```
title                             original  remake
-------------------------------   --------  ------
How to Train Your Dragon          2010      2025
The Girl with the Dragon Tattoo   2009      2011
```

(Note that the slide's `AS` labels are `first` and `second` while the described output header says `original` and `remake`; the `AS` names are what the query actually produces.)

### Example 10: Movies per decade (floor division + grouping)

> From the transcript. Create a table with columns `decade` and `movies` containing one row per decade in which a movie was released, and the count of movies from the 1,000-row dataset released in that decade.

The hint is integer division: `43 / 10` is `4` in SQL.

```sql
SELECT year / 10 * 10 AS decade, COUNT(*) AS movies
  FROM titles
  GROUP BY decade;
```

Step by step:
1. `year / 10` drops the last digit: `1922 / 10` is `192`.
2. `* 10` puts a zero back: `192 * 10` is `1920`. So 1922 maps to the decade 1920.
3. `AS decade` names the computed column, which lets `GROUP BY decade` reuse the name instead of retyping `year / 10 * 10`. The instructor explicitly recommends this over retyping the expression.
4. `GROUP BY decade` forms one group per decade value, and `COUNT(*) AS movies` counts rows within each group.

### Example 11: Average rating for each actor (join + group)

> From the slides.

```sql
SELECT names.name,
       SUM(ratings.averageRating * ratings.numVotes) / SUM(ratings.numVotes)
  FROM ratings JOIN names JOIN principals
  ON names.nconst = principals.nconst AND ratings.tconst = principals.tconst
  GROUP BY names.nconst
  ORDER BY averageRating DESC LIMIT 10;
```

What each piece does:
1. The three-way join stitches together a person (`names`), their credit on a title (`principals`), and that title's rating (`ratings`). The `ON` conditions enforce internal coherence of each joined row: the `nconst` must match between `names` and `principals`, and the `tconst` must match between `ratings` and `principals`. Without both conditions you get meaningless cross-product rows pairing an actor with an unrelated movie's rating.
2. `GROUP BY names.nconst` gathers all of one person's joined rows. Grouping by the ID rather than the display name is the safer choice since two people can share a name.
3. The weighted-average expression from Example 4 is reused, now computed per actor instead of over the whole table.
4. `ORDER BY ... DESC LIMIT 10` returns the top ten.

### Example 12: Top 20 actors by total votes (full walkthrough from the demo)

> From the transcript. Create a table with columns `name` and `votes` containing one row for each of the 20 actors with the most votes, in descending order. An actor's votes is the total `numVotes` of all movies in which they were first in the credits.

The instructor narrates the reasoning for needing a join: each output row (an actor's name plus their total votes) depends on the `names` table for the name, on many rows of `principals` to discover which movies that actor led, and on many rows of `ratings` to look up the votes for each of those movies. The `titles` table is **not** needed, because we never display a movie title or year, and `principals` already connects an actor to the movie they were in.

The query, built up clause by clause:

```sql
SELECT names.name, SUM(ratings.numVotes) AS votes
  FROM names JOIN principals JOIN ratings
  ON names.nconst = principals.nconst AND principals.tconst = ratings.tconst
  WHERE ordering = 1 AND category = "actor"
  GROUP BY names.name
  ORDER BY votes DESC LIMIT 20;
```

Step by step:
1. **`SELECT`**: the actor's name and the summed votes, labeled `votes` so `ORDER BY` can refer to it.
2. **`FROM ... ON ...`**: join all three tables and constrain the join so each joined row is internally consistent: one row telling us the actor's name, the role they played (with their credit ordering), and the number of votes that movie received.
3. **`WHERE ordering = 1 AND category = "actor"`**: keep only rows where the person was first in the credits (the lead) and is an actor rather than a director. Both are per-row conditions, so `WHERE` is correct.
4. **`GROUP BY names.name`**: one group per actor, containing all of their lead-role movies.
5. **`SUM(ratings.numVotes) AS votes`**: now that each group holds all of one actor's lead movies, we can total the votes across them.
6. **`ORDER BY votes DESC LIMIT 20`**: the top 20, descending.

The instructor ran this with SQLite initialized against the IMDb-1000 database, hit an error from a typo (a missing letter `N`), fixed it, switched to column mode for readable output, and got: Leonardo DiCaprio first (his lead roles summed to the most votes), then Tom Hanks, Christian Bale, and other famous names. The instructor noted that all of the results are men, which reflects the gender imbalance in Hollywood lead roles, and that querying for actors who were *second* in the credits surfaces a different list that includes women.

### Example 13: Exam question (Spring 2023 CS 61A Final, Q7)

> From the slides. The `finals` table has columns `hall` (strings) and `course` (strings), with one row per lecture hall in which a course holds its final. The `sizes` table has columns `room` (strings) and `seats` (numbers), with one row per unique room on campus. All lecture halls are rooms. Create a table with columns `course` (string) and `seats` (number), one row per course, giving the total seats across that course's final rooms. Only include courses that use at least two rooms.

```sql
SELECT course, SUM(seats) AS seats
  FROM finals, sizes
  WHERE hall = room
  GROUP BY course
  HAVING COUNT(*) > 1;
```

Step by step:
1. **`FROM finals, sizes`** forms the cross product of the two tables.
2. **`WHERE hall = room`** keeps only the coherent pairings, attaching each final-exam hall to its own seat count. Without this, every hall would be paired with every room's seat count.
3. **`GROUP BY course`** gathers all of a course's hall rows into one group.
4. **`HAVING COUNT(*) > 1`** enforces "at least two rooms" by counting the rows in the group. Note that "at least two" becomes `> 1`, not `>= 1` and not `> 2`.
5. **`SUM(seats) AS seats`** totals the seats across the group's rooms, with the required column label.

This question is a compact rehearsal of the whole lecture: join via `WHERE`, group, filter groups with an aggregate, and aggregate in `SELECT`.

## Common Pitfalls

- **Mixing aggregated and non-aggregated columns.** `SELECT MAX(legs), weight FROM animals;` runs in SQLite but is meaningless and errors in other SQL systems. Aggregate all the columns or none of them. The only safe bare column is one you grouped by.
- **Confusing `MAX(a - b)` with `MAX(a) - MIN(b)`.** The first computes `a - b` per row and then takes the max; the second performs two independent aggregations over the group and then subtracts. They generally give different answers. When unsure, run the inner expression alone first to see what the aggregate is scanning.
- **Averaging averages.** `AVG(averageRating)` is not the average rating across votes. Use `SUM(averageRating * numVotes) / SUM(numVotes)` when each row's average already summarizes a different-sized population.
- **Putting an aggregate in `WHERE`.** `WHERE` tests one row at a time, so `WHERE COUNT(*) > 1` is invalid. Group conditions go in `HAVING`.
- **Putting a per-row condition in `HAVING`.** Conversely, filtering out directors with `HAVING category = "actor"` is the wrong tool; that is a per-row test and belongs in `WHERE` before the groups form.
- **Forgetting the join condition.** `FROM finals, sizes` without `WHERE hall = room`, or a multi-way `JOIN` missing one of its `ON` conjuncts, produces rows that are not internally coherent (an actor paired with an unrelated movie's votes). Aggregating those rows yields confidently wrong numbers rather than an error.
- **Off-by-one in `HAVING` thresholds.** "More than 15 characters" is `COUNT(*) > 15`. "At least two rooms" is `COUNT(*) > 1`. "Appears more than once" is `COUNT(*) > 1`. Translate the English carefully.
- **Expecting floating-point division from integers.** `year / 10` floors. That is exactly what makes the decade trick (`year / 10 * 10`) work, but it will silently truncate when you wanted a fraction.
- **Retyping a long grouping expression.** Label it with `AS` and reuse the name in `GROUP BY` or `ORDER BY` rather than duplicating the expression.
- **Grouping by a display name when an ID exists.** `GROUP BY names.name` works in the demo but `GROUP BY names.nconst` is the more robust choice, since distinct people can share a name. (extra context: the slides' actor-ratings query groups by `nconst` for exactly this reason.)
- **Joining in a table you do not need.** The demo explicitly skips `titles` because no title or year is displayed and `principals` already links actor to movie. Extra joins add cost and risk of duplicate rows.

## Likely Exam Points

### 1. `WHERE` vs `HAVING`

**Practice:** Given a table `trips(driver, miles)`, write a query returning each driver who took more than 3 trips of over 100 miles each, along with the number of such trips.

**Answer:**
```sql
SELECT driver, COUNT(*) FROM trips WHERE miles > 100 GROUP BY driver HAVING COUNT(*) > 3;
```
The per-trip condition (`miles > 100`) goes in `WHERE`; the per-driver condition (`COUNT(*) > 3`) goes in `HAVING`.

### 2. Reading off the number of output rows from `GROUP BY`

**Practice:** The `animals` table has six rows: ferret (4 legs), dog (4), cat (4), parrot (2), penguin (2), t-rex (2). How many rows does `SELECT legs, COUNT(*) FROM animals GROUP BY legs;` produce? How many does `SELECT COUNT(*) FROM animals;` produce?

**Answer:** Two rows for the first (one per unique `legs` value: 2 and 4). One row for the second (no `GROUP BY`, so all six rows form a single implicit group).

### 3. `MAX(a - b)` vs `MAX(a) - MIN(b)`

**Practice:** For `animals` with parrot (2 legs, weight 6) and ferret (4 legs, weight 20), explain in one sentence why `MAX(legs - weight)` and `MAX(legs) - MIN(weight)` can differ.

**Answer:** `MAX(legs - weight)` evaluates `legs - weight` within each row and then takes the largest of those per-row values (the parrot's row wins), whereas `MAX(legs) - MIN(weight)` performs two independent aggregations over the whole group and subtracts the results, so the two operands may come from different rows.

### 4. The weighted average pattern

**Practice:** The `ratings` table has `averageRating` and `numVotes` per movie. Write a query for the overall average rating across all votes, and say why `AVG(averageRating)` is wrong.

**Answer:**
```sql
SELECT SUM(averageRating * numVotes) / SUM(numVotes) FROM ratings;
```
`AVG(averageRating)` treats every movie equally regardless of how many votes it received, so a movie with 20 votes would count as much as one with 80.

### 5. Self-grouping to find duplicates

**Practice:** Using `titles(tconst, title, year, runtime, genres)`, produce one row per title that was made more than once, showing the title, the earliest year, and the latest year.

**Answer:**
```sql
SELECT title, MIN(year) AS first, MAX(year) AS second
  FROM titles GROUP BY title HAVING COUNT(*) > 1;
```
Grouping by `title` and keeping groups with more than one row isolates exactly the repeated titles; `MIN` and `MAX` then read the earliest and latest rows in each group.

### 6. Join, filter, group, order, limit in one statement

**Practice:** Using `names(nconst, name, ...)`, `principals(tconst, ordering, nconst, category, character)`, and `ratings(tconst, averageRating, numVotes)`, write a query for the 20 actors with the most total votes across the movies where they were first in the credits, in descending order.

**Answer:**
```sql
SELECT names.name, SUM(ratings.numVotes) AS votes
  FROM names JOIN principals JOIN ratings
  ON names.nconst = principals.nconst AND principals.tconst = ratings.tconst
  WHERE ordering = 1 AND category = "actor"
  GROUP BY names.name
  ORDER BY votes DESC LIMIT 20;
```
Both `ON` conjuncts are required so each joined row is internally coherent; `titles` is unnecessary because no title or year appears in the output.

### 7. Integer division for bucketing

**Practice:** Write a query over `titles` producing one row per decade with the count of movies released in that decade.

**Answer:**
```sql
SELECT year / 10 * 10 AS decade, COUNT(*) AS movies FROM titles GROUP BY decade;
```
`year / 10` floors away the last digit and `* 10` restores it as a zero, mapping 1922 to 1920. The `AS decade` label lets `GROUP BY` reuse the expression by name.

### 8. The illegal mixed select

**Practice:** Is `SELECT MIN(weight), name FROM animals;` a well-formed query? Explain.

**Answer:** No, not meaningfully. It requests one output row, but while `MIN(weight)` is well-defined, `name` was never aggregated and the group holds many names, so there is no principled value. SQLite returns something, other SQL systems error. Either aggregate `name` too, or drop the aggregate and return all rows.

### 9. The exam-style two-table aggregation (Spring 2023 Final Q7)

**Practice:** `finals(hall, course)` and `sizes(room, seats)`. Produce `course` and total `seats` for each course using at least two rooms for its final.

**Answer:**
```sql
SELECT course, SUM(seats) AS seats
  FROM finals, sizes WHERE hall = room
  GROUP BY course HAVING COUNT(*) > 1;
```
`WHERE hall = room` is the join condition, `GROUP BY course` forms one group per course, `HAVING COUNT(*) > 1` encodes "at least two rooms", and `SUM(seats) AS seats` totals and labels.

### 10. Why the data is split into multiple tables

**Practice:** Why does the IMDb dataset store titles, ratings, principals, and names in four tables rather than one wide table?

**Answer:** Each fact is stored once rather than repeated. An actor's name lives once in `names` instead of being duplicated for every credit, and a movie's rating lives once in `ratings` instead of being duplicated for every cast member. Queries then recombine the facts with joins. (extra context: this avoids update anomalies and is the standard normalization argument.)

## Summary

- An **aggregate function** in the `SELECT` columns computes one value from a group of rows; with no `GROUP BY`, that group is all rows, so six input rows become one output row.
- The functions covered: `COUNT(*)`, `MAX`, `MIN`, `SUM`, `AVG`.
- `MAX([expr])` evaluates `[expr]` once per row and then takes the largest result; it never combines values across rows inside the expression. `MAX(a) - MIN(b)` is two separate aggregations whose operands may come from different rows.
- Do not mix aggregated and non-aggregated columns. SQLite tolerates it, other systems reject it, and the result is meaningless. The exception is a column you grouped by, which has one value per group.
- You cannot average averages when the rows summarize different-sized populations. Use `SUM(value * weight) / SUM(weight)`.
- `GROUP BY [expression]` yields **one output row per unique value** of the expression, and the expression can be any computed value (for example `weight / legs`, or `year / 10 * 10` for decades using SQL's integer floor division).
- `WHERE` filters individual input rows and cannot contain aggregates. `HAVING` filters whole groups and can contain aggregates.
- Methodology: decide which input rows you need (`FROM`, `WHERE`), then form and filter groups (`GROUP BY`, `HAVING`), then format the output (`SELECT`).
- Clause roles: `SELECT` = values and labels in each output row; `FROM` = source of input rows; `WHERE` = which input rows; `GROUP BY` = form output rows; `HAVING` = which output rows.
- Label output columns with `[expression] AS [name]`, then reuse the name in `GROUP BY` or `ORDER BY` instead of retyping the expression.
- Multi-table aggregation requires a complete join condition so that each joined row is internally coherent; an incomplete `ON` or a missing `WHERE hall = room` produces wrong numbers silently rather than an error.
- Only join the tables you actually need: the top-20-actors demo skips `titles` entirely because `principals` already links actors to movies.
