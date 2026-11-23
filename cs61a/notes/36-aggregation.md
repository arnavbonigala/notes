<!-- Fri, Nov 20, 2026 | sources: slides (no transcript available) -->
# Lecture 36: Aggregation

This lecture covers SQL aggregation: the ability to collapse many rows of a table into a single summary row, either across the whole table or separately for each group of rows that share a value. The core mechanism is the `GROUP BY` clause, which partitions the input rows into groups (one output row per unique value of the grouping expression), combined with aggregate functions (`COUNT`, `MAX`, `MIN`, `SUM`, `AVG`) in the `SELECT` clause that compute a single value from each group's rows. `HAVING` then filters which groups survive into the output, which is distinct from `WHERE`, which filters which input rows enter the grouping in the first place. The lecture also presents a disciplined three-step method for writing any select statement (pick input rows, form and filter groups, format output), and works through several examples on an IMDb-style schema (`principals`, `titles`, `names`, `ratings`), including finding remakes by title and computing a vote-weighted average rating per actor across joined tables.

## Key Concepts

### The full shape of a select statement

The lecture's central template is:

```sql
SELECT [columns] FROM [table] GROUP BY [expression] HAVING [expression];
```

Each clause has one job, and the slides spell them out explicitly:

- **SELECT**: the values each output row contains (and the column labels)
- **FROM**: the source of input rows
- **WHERE**: which input rows
- **GROUP BY**: how to form output rows
- **HAVING**: which output rows

This mapping is the whole lecture in miniature. Notice the asymmetry: `WHERE` talks about *input* rows, `HAVING` talks about *output* rows. `FROM` and `WHERE` work in the world of the original table; `GROUP BY` and `HAVING` work in the world of groups; `SELECT` decides what each surviving group looks like in the result.

### Grouping: one output row per unique value

Without `GROUP BY`, a select statement that uses an aggregate function treats the entire table as one single group, producing exactly one output row. With `GROUP BY [expression]`, the rows are partitioned: every row for which the expression has the same value lands in the same group, and each group becomes exactly one output row.

The lecture's running illustration uses the `principals` table:

| tconst | ordering | nconst | category | character |
|---|---|---|---|---|
| tt0012349 | 2 | nm0701012 | actor | The Woman |
| tt0012349 | 3 | nm0001067 | actor | The Child |
| tt0017136 | 1 | nm0375609 | actor | Maria |
| tt0012349 | 13 | nm0000122 | director | \N |
| tt0013442 | 11 | nm0003638 | director | \N |

With `GROUP BY category`, the three `actor` rows collapse into one output row, and the two `director` rows collapse into another output row. Two distinct values of `category`, therefore two output rows. The slides annotate exactly this: "Output row for actor" covering the three actor rows, "Output row for director" covering the two director rows.

The important intuition: grouping does not throw information away immediately. It bundles rows together and then hands each bundle to the aggregate functions in `SELECT`, which decide how to combine them. The question the `SELECT` clause answers for a grouped query is literally the slide's phrasing: "How do we want to combine the rows?"

### Aggregate functions

An aggregate function in the `[columns]` clause computes a value from a group of rows (or from all rows, if there are no groups). The lecture lists:

- `COUNT(*)`: the number of rows in a group
- `MAX([expression])`: the largest value of `[expression]` for any row in the group
- `MIN`, `SUM`, and `AVG`: analogous (smallest value, total, and mean)

Each of these takes many rows and returns one value. That is the defining property: a non-aggregate column expression produces one value *per row*, an aggregate produces one value *per group*.

### Column labels with AS

The slides introduce the labeling syntax for the `SELECT` clause:

```
[expression] AS [name], [expression] AS [name], ...
```

This matters more for aggregates than for plain columns, because an expression like `COUNT(*)` or `MIN(year)` has no natural column name. Writing `COUNT(*) AS total` gives the output column a readable header, which is also what lets later clauses or external code refer to it by name.

### WHERE versus HAVING

This is the distinction most likely to be tested, and the slides draw it precisely:

- `WHERE [expression]` keeps only input rows for which the expression is true. It runs *before* grouping, so it reduces the set of rows that are available to be grouped.
- `HAVING [expression]` keeps only groups for which the expression is true. It runs *after* grouping, so it can refer to aggregate functions over the group.

So `WHERE category="actor"` filters away director rows before any grouping happens. `HAVING COUNT(*) > 15` cannot be expressed in `WHERE` at all, because at `WHERE` time there is no group yet and therefore no count.

### The three-step method for writing select statements

The lecture gives an explicit procedure. Describe the output table, then:

1. Determine which existing rows are needed to express the result (`FROM` and `WHERE`)
2. Form groups and determine which groups should appear as output rows (`GROUP BY` and `HAVING`)
3. Format the output rows (`SELECT`)

This is deliberately *not* the order the clauses are written in. You write `SELECT` first but you should think about it last, because you cannot decide how to format a row until you know what a row is. This ordering also happens to match the logical evaluation order of the clauses, which is why it works so reliably.

### Why the data is split across tables (extra context on a discussion prompt)

The lecture poses the question "Why store the data like this?" about the four-table IMDb schema (`titles`, `principals`, `names`, `ratings`) as a poll, and the slides do not record an answer. The standard reason is normalization: each fact is stored once in the table where it belongs, so an actor's name lives only in `names` and a film's rating lives only in `ratings`, and `principals` holds just the many-to-many relationship between films and people. This avoids duplicating a name across every film the person appears in, and avoids inconsistency when a value is updated. Joins then reassemble the pieces on demand.

## Definitions

- **Aggregate function**: a function in the `[columns]` clause that computes a single value from a group of rows, or from all rows if there are no groups. The lecture's set: `COUNT`, `MAX`, `MIN`, `SUM`, `AVG`.
- **`COUNT(*)`**: an aggregate function returning the number of rows in a group.
- **`MAX([expression])`**: an aggregate function returning the largest value of `[expression]` among the rows of a group. `MIN` returns the smallest, `SUM` the total, `AVG` the mean.
- **`GROUP BY [expression]`**: a clause that produces one output row for each unique value of `[expression]` among the input rows.
- **`HAVING [expression]`**: a clause that keeps only those groups for which `[expression]` is true.
- **`WHERE [expression]`**: a clause that keeps only those input rows for which `[expression]` is true (applied before grouping).
- **`AS [name]`**: syntax in the `SELECT` clause that assigns a column label to an expression.
- **Group**: the set of input rows sharing a single value of the `GROUP BY` expression; it becomes exactly one output row.
- **`principals`**: lecture table with columns `tconst` (title id), `ordering`, `nconst` (person id), `category` (role such as actor or director), and `character`.
- **`titles`**: lecture table with columns `tconst`, `title`, `year`, `runtime`, `genres`.
- **`names`**: lecture table with columns `nconst`, `name`, `birth`, `death`, `profession`, `knownforTitles`.
- **`ratings`**: lecture table with columns `tconst`, `averageRating`, `numVotes`.
- **`\N`**: the placeholder appearing in the IMDb data for a missing value, visible in the `character` column of director rows.

## Worked Examples

### Example 1: Count rows per category

**Goal (from the slides):** For each category, select the category and the total number of rows for that category.

```sql
SELECT category, COUNT(*) AS total
  FROM principals
  GROUP BY category;
```

**Step by step.** `FROM principals` supplies every row of the table as input; there is no `WHERE`, so nothing is filtered out. `GROUP BY category` partitions those rows by the value in the `category` column. On the slide's sample data, three rows have `category` equal to `actor` and two have `category` equal to `director`, so there are two groups and therefore two output rows. `SELECT category, COUNT(*) AS total` then formats each group: `category` is the shared value that defines the group (safe to select, because it is identical for every row in the group), and `COUNT(*)` is evaluated over the group's rows.

**Result on the sample data:**

```
actor      3
director   2
```

The `AS total` labels the second column `total` rather than leaving it with the raw expression text as a header.

**A useful mental picture.** Think of the table as a pile of index cards. `GROUP BY category` sorts the pile into stacks, one stack per distinct category. `COUNT(*)` reads each stack and writes down how tall it is. The output table has one line per stack.

### Example 2: Actors with more than 15 characters played

**Goal (from the slides):** Select the `nconst` and the total number of characters for each actor who had more than 15 characters played.

```sql
SELECT nconst, COUNT(*)
  FROM principals
  WHERE category="actor"
  GROUP BY nconst
  HAVING COUNT(*) > 15;
```

**Step by step, using the three-step method.**

*Step 1, which input rows?* The result concerns actors only, so the source is `principals` and the filter is `WHERE category="actor"`. Director rows, writer rows, and so on are discarded before anything else happens. This is a per-row condition, so it belongs in `WHERE`.

*Step 2, what are the groups and which ones survive?* The output has one row per actor, and an actor is identified by `nconst`, so `GROUP BY nconst`. Each group is now "all the actor-role rows belonging to one person," and since each such row records one character played, the size of the group is the number of characters that person played. The requirement "more than 15 characters played" is a property of the group, not of any individual row, so it belongs in `HAVING COUNT(*) > 15`.

*Step 3, how is each output row formatted?* `SELECT nconst, COUNT(*)` gives the person's id and their character count.

**Why the two filters cannot be swapped.** `WHERE COUNT(*) > 15` is meaningless: at `WHERE` time, rows are being examined one at a time and no group exists, so there is nothing to count. Conversely, putting `category="actor"` in `HAVING` would be wrong in general, because by then rows have already been grouped and the category of an individual row is no longer a per-group property in the way the question needs.

**A subtle point about the count.** The query counts *rows*, and each matching row in `principals` is one credited character for one person in one film. So `COUNT(*)` here is the number of character credits, which is what "characters played" means in this schema. The lecture marks this query with "(Demo)", indicating it was run live against the real database.

### Example 3: Remakes that share a title

**Goal (from the slides):** Create a table of remakes that have the same title, with columns `title`, `original`, and `remake`.

The desired output looks like:

```
title                             original        remake
-------------------------------   --------        ------
How to Train Your Dragon          2010            2025
The Girl with the Dragon Tattoo   2009            2011
```

The answer developed on the slide:

```sql
SELECT title, MIN(year) AS first, MAX(year) AS second
  FROM titles
  GROUP BY title
  HAVING COUNT(*) > 1;
```

**Step by step.** `FROM titles` supplies all films; no `WHERE` filter is needed because every title is a candidate. `GROUP BY title` puts all films sharing a title string into one group. A title that was only ever made once forms a group of size one; a title made twice forms a group of size two. `HAVING COUNT(*) > 1` keeps only the groups with at least two films, which is exactly the definition of "there is a remake." Then `SELECT title, MIN(year) AS first, MAX(year) AS second` formats each surviving group: the title shared by the group, the earliest year in the group (the original), and the latest year in the group (the remake).

**Why `MIN` and `MAX` are the right tools.** The output needs two different years drawn from the same group. You cannot write `SELECT title, year` here, because a group with two films has two different values of `year` and a single output row cannot hold both unaggregated. `MIN(year)` and `MAX(year)` each collapse the group to one value, and because "original" means earliest and "remake" means latest, they land on exactly the right rows.

**A limitation worth noticing.** If a title was made three times, this query reports only the first and the last year, silently hiding the middle one. The slide's column labels `first` and `second` reflect the two-film case. (extra context) Handling three or more versions would require a different approach, such as a self-join of `titles` to itself on matching title with a strict inequality on year.

### Example 4: Vote-weighted average rating per actor

**Goal (from the slides):** Select each actor's name and their average rating.

```sql
SELECT names.name,
       SUM(ratings.averageRating * ratings.numVotes) / SUM(ratings.numVotes)
  FROM ratings JOIN names JOIN principals
  ON names.nconst=principals.nconst AND ratings.tconst=principals.tconst
  GROUP BY names.nconst
  ORDER BY averageRating DESC
  LIMIT 10;
```

**Step by step.**

*The join.* Three tables are joined. `principals` is the bridge: it has a `nconst` linking to `names` and a `tconst` linking to `ratings`. The `ON` clause states both links: `names.nconst=principals.nconst` connects a credit to the person who earned it, and `ratings.tconst=principals.tconst` connects a credit to the rating of the film it appeared in. Each row of the joined table is therefore one credit, carrying the person's name alongside that film's `averageRating` and `numVotes`.

*The grouping.* `GROUP BY names.nconst` puts all of one person's credits into one group. Grouping on `nconst` rather than on `name` is deliberate: `nconst` is the unique identifier, so two different people who happen to share a name are kept separate. (extra context) Grouping by `name` would merge them incorrectly.

*The aggregate.* This is the most interesting expression in the lecture, because it is not simply `AVG(averageRating)`. The query computes

```
SUM(averageRating * numVotes) / SUM(numVotes)
```

which is a **weighted** mean: each film's rating counts in proportion to how many people voted on it. Note that both `SUM`s aggregate over the same group, and the division happens between the two aggregate results, not row by row.

Why weight by votes? A film with 8 votes averaging 9.5 and a film with 500,000 votes averaging 7.0 should not count equally toward an actor's overall standing. `AVG(averageRating)` would treat them as equal and let an obscure, barely-rated film swing the result. The weighted version says: pool all the individual votes across all of the actor's films and take the mean over that pooled set of votes. `averageRating * numVotes` reconstructs the total of all the vote scores for one film, summing that over the group gives the total of all vote scores across the actor's filmography, and dividing by the total number of votes gives the overall mean.

*Ordering and limiting.* `ORDER BY averageRating DESC LIMIT 10` sorts the output rows from highest to lowest and keeps the top ten.

**Reading this in environment-diagram-style terms.** (extra context) Although SQL has no environments, it is useful to reason about scope in an analogous way. Inside a grouped query there are two levels of "visibility." At the row level, the raw column values (`ratings.averageRating`, `ratings.numVotes`, `names.name`) are individually visible, and expressions like `averageRating * numVotes` are computed once per row. At the group level, only the grouping key and aggregate results are available: `SUM(...)` reaches down across all rows in its group and returns one number. An unaggregated column in `SELECT` that is not the grouping key is ambiguous at the group level, in the same way a name is unbound if it was never defined in an accessible frame.

### Example 5: Spring 2023 Final, Question 7

**Setup.** The `finals` table has columns `hall` (strings) and `course` (strings), with one row for each lecture hall in which a course is holding its final exam. The `sizes` table has columns `room` (strings) and `seats` (numbers), with one row per unique room on campus giving the number of seats in that room. All lecture halls are rooms.

**Goal.** Create a table with two columns, `course` (string) and `seats` (number), with one row containing the name of the course and the total number of seats in final rooms for that course. Only include a row for each course that uses at least two rooms for its final.

**Answer from the slides:**

```sql
SELECT course, SUM(seats) AS seats
  FROM finals, sizes
  WHERE hall=room
  GROUP BY course
  HAVING COUNT(*) > 1;
```

**Step by step, with the three-step method.**

*Step 1, which input rows?* The question mentions both the room assignments (`finals`) and the seat counts (`sizes`), so both tables are needed. `FROM finals, sizes` forms the Cartesian product of the two: every `finals` row paired with every `sizes` row. That product is mostly nonsense, so `WHERE hall=room` keeps only the pairings where the lecture hall matches the room whose seat count is being read. After this step, each surviving row is "one course, one hall it uses, and that hall's seat count."

*Step 2, groups and group filter.* The output has one row per course, so `GROUP BY course`. Each group is all the hall-and-seats rows for one course. The restriction "at least two rooms" is a statement about the size of the group, so `HAVING COUNT(*) > 1`. Since each row in a group is one distinct hall for that course, counting rows counts halls.

*Step 3, formatting.* `SELECT course, SUM(seats) AS seats` gives the course name and the total seating across all of its final rooms. The `AS seats` is what makes the output column match the required name.

**Note on the join style.** This query uses the comma form (`FROM finals, sizes` with the matching condition in `WHERE`) rather than the explicit `JOIN ... ON` form used in Example 4. Both express the same idea: form combinations of rows from two tables, then keep only the combinations that line up correctly. Choosing one or the other is a matter of style in SQLite.

**Note on "at least two."** "At least two" means two or more, so the condition is `COUNT(*) > 1`, equivalently `COUNT(*) >= 2`. Writing `COUNT(*) > 2` would be an off-by-one error that drops every course using exactly two rooms.

## Common Pitfalls

- **Putting an aggregate condition in `WHERE`.** `WHERE COUNT(*) > 15` is wrong. `WHERE` filters individual input rows before any group exists, so there is nothing to count. Aggregate conditions go in `HAVING`.

- **Putting a per-row condition in `HAVING` when `WHERE` is intended.** `HAVING category="actor"` is not the same as `WHERE category="actor"`. The `WHERE` version removes non-actor rows before grouping, so they never affect any count; the `HAVING` version operates after grouping and does not do what the question asks.

- **Selecting an unaggregated column that is not the grouping key.** If you `GROUP BY title` and then `SELECT title, year`, a group containing two films has two different years and only one output row to put them in. The lecture's remakes example sidesteps this correctly with `MIN(year)` and `MAX(year)`. (extra context) SQLite will not error here; it silently picks a value from an arbitrary row in the group, which makes this a quiet correctness bug rather than a visible one.

- **Forgetting that `GROUP BY` produces one row per unique value.** The number of output rows is the number of distinct values of the grouping expression (after `HAVING` filtering), not the number of input rows.

- **Confusing `AVG(x)` with a weighted average.** The actor-ratings example deliberately uses `SUM(rating * votes) / SUM(votes)` and not `AVG(rating)`. These give different answers whenever the films have different vote counts. Using the plain average lets a film with a handful of votes count as much as a blockbuster.

- **Grouping by a display name instead of an identifier.** `GROUP BY names.nconst` separates two distinct people who share a name; `GROUP BY names.name` would merge them.

- **Off-by-one in `HAVING`.** "More than 15" is `> 15`. "At least two" is `> 1`, not `> 2`. Read the English carefully.

- **Omitting the join condition.** `FROM finals, sizes` with no `WHERE hall=room` produces every pairing of every course-hall with every room on campus, inflating every count and sum. In the three-table actor query, both halves of the `ON` condition are required; dropping either one produces a similarly inflated product.

- **Writing clauses in the order you think about them.** The written order is fixed: `SELECT`, `FROM`, `WHERE`, `GROUP BY`, `HAVING`. The thinking order is `FROM`/`WHERE`, then `GROUP BY`/`HAVING`, then `SELECT`.

- **Skipping `AS` when the question specifies column names.** The final exam question asks for a column literally named `seats`, which requires `SUM(seats) AS seats`. Without `AS`, the column header is the raw expression.

## Likely Exam Points

### 1. Choosing between `WHERE` and `HAVING`

This is the single most tested idea from this lecture. Any condition that can be checked by looking at one row alone goes in `WHERE`; any condition involving an aggregate over a group goes in `HAVING`.

**Practice:** The `principals` table has columns `tconst`, `ordering`, `nconst`, `category`, and `character`. Write a query selecting each `tconst` and the number of actors credited in it, for films with more than 3 credited actors.

**Answer:**
```sql
SELECT tconst, COUNT(*)
  FROM principals
  WHERE category="actor"
  GROUP BY tconst
  HAVING COUNT(*) > 3;
```
`category="actor"` is per-row, so `WHERE`. "More than 3 actors" counts rows in a group, so `HAVING`.

### 2. Determining the number of output rows

**Practice:** Given the five sample `principals` rows from the lecture (three with `category` equal to `actor`, two with `category` equal to `director`), how many rows does `SELECT category, COUNT(*) FROM principals GROUP BY category;` produce, and what are they?

**Answer:** Two rows: `actor 3` and `director 2`. One output row per unique value of `category`.

### 3. Picking the right aggregate function for a stated goal

**Practice:** Using the `titles` table (`tconst`, `title`, `year`, `runtime`, `genres`), write a query that, for each year, gives the year and the longest runtime of any film released that year.

**Answer:**
```sql
SELECT year, MAX(runtime) AS longest
  FROM titles
  GROUP BY year;
```
"Longest runtime of any row in the group" is exactly `MAX`.

### 4. Using two aggregates on the same group

The remakes query is a clean template: group by the shared key, use `HAVING COUNT(*) > 1` to require a collision, and use `MIN` and `MAX` to pull out the two extremes.

**Practice:** Write a query producing one row per genre string in `titles`, with the genre, the earliest year, and the latest year, only for genres with at least 5 films.

**Answer:**
```sql
SELECT genres, MIN(year) AS first, MAX(year) AS last
  FROM titles
  GROUP BY genres
  HAVING COUNT(*) > 4;
```
"At least 5" is `COUNT(*) > 4`, equivalently `>= 5`.

### 5. Joining then grouping

Exam questions frequently require combining two tables before aggregating. The pattern is: `FROM a, b WHERE a.key=b.key GROUP BY ...` (or the `JOIN ... ON` equivalent).

**Practice:** Using `finals` (`hall`, `course`) and `sizes` (`room`, `seats`), write a query giving each course and the number of seats in its *largest* final room, for courses using at least two rooms.

**Answer:**
```sql
SELECT course, MAX(seats) AS biggest
  FROM finals, sizes
  WHERE hall=room
  GROUP BY course
  HAVING COUNT(*) > 1;
```
Same skeleton as the lecture's version of this question, with `SUM` swapped for `MAX`.

### 6. Weighted averages

The actor-rating query is the lecture's one non-obvious aggregate expression and a natural exam target, since it tests whether you understand that `AVG` is not always the right notion of "average."

**Practice:** Explain in one or two sentences why the lecture computes `SUM(averageRating * numVotes) / SUM(numVotes)` rather than `AVG(averageRating)`, and give a two-film case where the answers differ.

**Answer:** `AVG(averageRating)` treats every film as equally informative regardless of how many people rated it, so a film with very few votes can swing an actor's score as much as a heavily-voted one. The weighted form pools the underlying votes. Example: a film rated 9.0 with 10 votes and a film rated 7.0 with 1000 votes give `AVG` a value of 8.0, while the weighted form gives `(9.0*10 + 7.0*1000) / 1010 = 7090/1010`, about 7.02, which reflects the overwhelming weight of the second film.

### 7. Applying the three-step method to a word problem

**Practice:** State the three steps for writing a select statement, in the order you should think about them, and which clauses each step produces.

**Answer:**
1. Determine which existing rows are needed: `FROM` and `WHERE`.
2. Form groups and determine which groups appear as output rows: `GROUP BY` and `HAVING`.
3. Format the output rows: `SELECT`.

### 8. Column labeling

**Practice:** The question asks for an output table with columns named `course` and `seats`. What goes wrong with `SELECT course, SUM(seats) FROM finals, sizes WHERE hall=room GROUP BY course HAVING COUNT(*) > 1;`?

**Answer:** The second column is labeled with the raw expression `SUM(seats)` rather than `seats`. Add `AS seats` to satisfy the required column name.

## Summary

- `SELECT [columns] FROM [table] GROUP BY [expression] HAVING [expression];` is the full template for aggregation.
- Clause roles: `SELECT` is the values each output row contains and their labels, `FROM` is the source of input rows, `WHERE` is which input rows, `GROUP BY` forms output rows, `HAVING` is which output rows.
- `GROUP BY [expression]` produces exactly one output row for each unique value of `[expression]`.
- An aggregate function in the `[columns]` clause computes one value from a group of rows, or from all rows if there is no grouping: `COUNT(*)`, `MAX`, `MIN`, `SUM`, `AVG`.
- `WHERE` filters rows *before* grouping; `HAVING` filters groups *after* grouping and is the only place aggregate conditions can appear.
- Label aggregate columns with `[expression] AS [name]`, especially when the question specifies output column names.
- Write select statements in three steps: pick input rows (`FROM`, `WHERE`), form and filter groups (`GROUP BY`, `HAVING`), then format output (`SELECT`).
- `SELECT category, COUNT(*) FROM principals GROUP BY category;` gives the number of rows per category, the canonical minimal example.
- Remakes pattern: `GROUP BY title HAVING COUNT(*) > 1`, with `MIN(year)` as the original and `MAX(year)` as the remake.
- Joining before grouping is standard: `FROM a, b WHERE a.key=b.key GROUP BY ...` or `FROM a JOIN b ON ...`, used for both the three-table actor-rating query and the finals-and-sizes exam question.
- A weighted average is `SUM(value * weight) / SUM(weight)`, which is what the actor-rating query uses so that heavily-voted films count more than lightly-voted ones.
- Group by a unique identifier (`nconst`) rather than a display value (`name`) when distinct entities might share the display value.
