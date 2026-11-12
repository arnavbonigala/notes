<!-- Fri, Nov 06, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 31: Software Engineering II: Managing Complexity

> Note on materials: the textbook chapter bundled with this lecture is Chapter 31 (Quicksort), which does not correspond to what this lecture actually covered. These notes follow the slides and the lecture recording. Quicksort is not discussed here.

Guest lecture by Aniruth Narayanan (CS 61B TA, Berkeley EECS and Business class of 2024, now a PM at Snowflake working on context for data agents). The lecture is explicitly non-examinable course content ("none of what we're going to talk about for the next hour will be covered in any course content"), and it is a sampling of how 61B ideas show up in real engineering. The through-line: in 61A you optimized for correctness, in 61B you optimize for engineering judgment, picking among several correct options by reasoning about tradeoffs. Modern software is almost never limited by hardware (your phone outclasses all of the Apollo program's computing; a 90 MHz Pentium ran RollerCoaster Tycoon, a 6 GHz i9 is ~67x faster per core before counting cores), so the real binding constraint is **complexity**: "anything related to the structure of a software system that makes it hard to understand and modify it" (John Ousterhout, *A Philosophy of Software Design*). Complexity grows roughly exponentially with functionality because every new feature must interact with every existing feature in every combination, so the discipline is to separate unavoidable (essential) complexity from avoidable complexity and then attack the avoidable kind with simpler code, modules, abstraction, and interfaces. The lecture frames this as a choice between **tactical programming** (fast, workaround-driven, great for prototypes and great for LLMs) and **strategic programming** (planned, maintainable, simple, where humans now spend most of their time steering LLMs). It closes with an extended case study: designing an Apache Iceberg-style distributed table format on cloud object storage, where each design decision is an application of trees, statistics, iterators, immutability, interfaces/polymorphism, and maps.

---

## Key Concepts

### 1. From correctness (61A) to engineering (61B)

In 61A the question was "is this program correct?" In 61B there are usually many correct programs and you pick among them under constraints: runtime, memory, API restrictions, specifications. The lecture's framing is that this skill does not transfer automatically from small assignments. On a small project you hold the whole thing in your head. On a large project the task itself is partly defined by you, the pieces are designed separately, and the failures come from how pieces interact. Experience is the only real teacher here; the lecture is a sampling.

### 2. Material constraints are gone; design is the constraint

Other engineering disciplines are bounded by imperfect materials: chemical engineers by temperature, materials scientists by brittleness, civil engineers by the strength of concrete. Computer science solved most of its underlying material constraints decades ago (the current GPU supply crunch aside). Evidence from the lecture:

- ENIAC, the first general-purpose programmable computer, was ~1000x faster than anything before it. Before that, "computer" meant a human doing calculations by hand, in rooms full of people checking each other's work.
- RollerCoaster Tycoon (1999) was 99% assembly, 1% C, written that way to extract performance from a 90 MHz Pentium. At that time C counted as a *high-level* language. Assembly means reasoning directly about the ~32 registers inside the processor and shuffling values between them millions of times per second, which is brutally hard for humans to reason about. (Register-level detail is CS 61C material.)
- Minecraft's recommended specs are ~3.5 GHz and 8 GB RAM. In 1996 the average machine had 8 to 16 MB of RAM.

The honest reading of this is not that software needs the power: the average product does not. Developers have gotten freer to spend hardware, which is why a few years after buying a machine nothing feels faster. The limiting factor moved to *the creative ways we plan and design what we build*.

Two consequences stated on the slides:

1. An individual programmer cannot effectively manage a large software system.
2. Any one programmer should only need to understand a fraction of the codebase.

The lecture extends (1) to LLMs: a model's context window is finite too, so you cannot feed in the whole codebase either. The constraint is architectural, not biological. Point (2) is a design *requirement*: if reading the whole codebase is infeasible, then needing to read the whole codebase to make a change is a bug in the design. A comment saying "to understand this function, go read that other file for the rationale" is a violation.

### 3. Complexity: definition, cost, and scaling

Definition (Ousterhout): anything about the structure of a software system that makes it hard to understand and modify. As a program gets more feature-rich, complexity increases. Why we care: complex systems require enormous effort for small improvements.

- It takes longer to understand how the code works.
- It is harder to fix bugs, because it is harder to be confident a change is correct and localized.
- It is harder to modify functionality, and in particular you hit **unknown unknowns**: you do not even know what you would need to know in order to safely make the change. The lecture calls these one of the most common sources of bugs when deploying at scale, and says they are extremely common in large codebases.

The lecture's intuition pump: think of complexity as inertia. The more features you have, the harder it is to change direction or keep accelerating. Microsoft shipping a percentage improvement to File Explorer or search takes many months or years, not because nobody is working hard, but because the codebase is millions of lines and you cannot change things without confidence about what else you touched.

Empirical data point: Spotify's engineering blog reported over 1 billion lines of code, ~60 million used in production, thousands of components, with component count scaling far faster than engineer count.

Why exponential: each new piece of functionality has to interact with all existing functionality, for all possible combinations. Build intuition by counting: feature *k* can interact with each of the *k-1* existing features, and interactions can compose, so the space of behaviors you must reason about blows up far faster than the line count. (extra context: pairwise interactions alone are already quadratic, n(n-1)/2, and once interactions compose you get combinatorial growth; the slide's word is "exponentially," and the practical point is only that it grows much faster than the feature count.)

### 4. Two kinds of complexity, and the levers for the avoidable kind

1. **Unavoidable (essential) complexity**: inherent, inescapable, caused by the functionality itself. You cannot build an `ArrayDeque` without an array underneath and without methods to manage indices. There is a floor.
2. **Avoidable complexity**: the result of your choices. This is the part you fight.

Levers against avoidable complexity:

- **Make code simpler and more obvious.** The lecture's 61B example is sentinel nodes in Project 1. A dummy node that is always present collapses the edge cases (empty list, adding the first item, removing the last item) so you handle one uniform case instead of many special ones. The point is structural, not cosmetic: you reduce the number of cases that exist, rather than writing more code to handle them.
- **Modules**, which give you:
  - **Abstraction**: the ability to use a piece without understanding how it works, based on a specification. The lecture's broadest example is the GUI: people who have never opened a terminal can still send email and write documents, because the graphical interface is a powerful abstraction over the machine. Java versus C versus Python are themselves different abstractions over what hardware can do.
  - **Interfaces**: separating *how something behaves* from *how it is implemented*. `HashMap` and `BSTMap` are both `Map`s. You pick between them on runtime grounds, or on data grounds (if your keys all collide under a bad hash function, the hash table's advantage evaporates and a different implementation wins). The caller does not change.
- **Deep modules** (named on the Iceberg catalog slide): a module whose interface is small relative to the functionality it hides. "Each request is to update the table given some version. Our storage system abstracts the file level changes." A deep module is the good case; a module with a huge interface hiding very little is the bad case.

### 5. The LLM dimension

The lecture takes AI as a present-tense fact, not a forecast, citing spring-2026-era reports that many companies no longer write code by hand (Airbnb at ~60%, plus Google and Anthropic datapoints). The speaker's own team at Snowflake writes essentially none by hand; the model does the Git operations, the code, the tests, the syntax fixes, dependency and virtualenv management.

Testing has changed shape with it: deterministic unit tests (JUnit-style) plus **simulated** tests, where agents drive the product, take screenshots, reason about what they see, feed inputs, and simulate scenarios. A recurring product question is literally "can an agent try out my product?" The lecture describes a **nightly QA loop**: merge the day's changes into a dev build, have agents run a list of common scenarios (mixed reads, writes, updates, in random patterns), and wake up to a report of what works and what does not. This augments existing testing, it does not replace it.

**Software factories**: engineers spend their time designing the system that writes the code, specifically the guardrails, boundaries, documentation, test signal, and customer feedback the agents iterate against.

The two postures on complexity in an LLM world:

| | Approach 1: LLMs handle everything | Approach 2: we steer LLMs |
|---|---|---|
| Who plans | The model; engineers hand over planning and tasks | Engineers design what the model should accomplish and guide it to an optimal, simple solution |
| Result | The model is free to build anything | Simpler codebases, which are easier for the model to parse and reason about: cheaper (more token efficient) and faster |

The speaker is openly biased toward Approach 2, and the argument is not sentimental: **simplicity that helps humans also helps models**, because a simpler codebase costs fewer tokens and less reasoning to work in. The "meat proxy" joke makes the counterpoint: if your entire job is pasting someone's question into a model and pasting the answer back, you added no value and were not required. The value-add is judgment about modules, abstractions, and simplicity.

What engineers are no longer opinionated about: the exact language. Ten years ago JavaScript versus TypeScript versus Java was a hot debate. Now the agent can often figure that out, with occasional exceptions where a lower-level language like Rust buys real performance.

Open question the lecture poses and deliberately does not answer: have LLMs changed our consideration of complexity and tradeoffs? If a model writes the code, is reasoning about asymptotics the best use of your time? The slides offer opinions and tradeoffs, not an answer. (And explicitly: this is not permission to violate course policy on AI use.)

### 6. Tactical versus strategic programming

**Tactical programming**: focus on getting something working quickly, leveraging workarounds. The canonical shape is deeply nested `if` statements, five or six levels, one branch added per failing test, until nobody can explain the code. The lecture concedes this is probably what you do today and what the speaker did in class.

- Good for: prototypes, proofs of concept, showing something *could* work, getting fast feedback. Excellent for LLMs: describe the idea at a high level, ignore maintainability, see what happens.
- Costs: minimal time on overall design; workarounds that can conflict with each other, so it is hard to verify a workaround actually works; refactoring is expensive and can mean restarting; and the big one, the prototype often ships to production anyway because there was no time, which pushes the restart-and-refactor cost later and makes it worse.

**Strategic programming**: emphasize long-term strategy over quick fixes, paying planning time to get code that works elegantly. Code should be:

- **Maintainable**: you can keep updating it.
- **Simple**: the lecture's working test is how intuitive it is the first time you look at it.
- **Future-proof**: and note the contrast the slide draws, 61B projects have deadlines and then you throw the code away. Microsoft cannot throw away Windows.

The key discipline: **if the strategy is insufficient, go back to the drawing board before continuing work.** You discard the plan rather than forcing it to work. In the LLM workflow this is where the bulk of human time goes: iterating on the approach, asking the model what it has tried, proposing designs from your own knowledge, then using that plan to incrementally rewrite the tactical output.

### 7. Software rewrites, and why their economics changed

Companies usually *intend* strategic programming and find tactical fixes creeping in. The lecture's causes:

- **Tacit knowledge** living in employees' heads, who may leave (and then you discover you needed that person to explain a subsystem).
- **Deadline pressure** to ship.
- **Business priority and offering changes.**
- **Partial technology stack migrations**, for example on-premises to cloud, which leave you straddling two worlds.

Before LLMs, rewrites were long and careful. The lecture's case study: Discord's 2020 blog post on moving from Go to Rust. The short version is that at scale, when many users log on at once, they hit memory pressure, and they were effectively forking and rewriting pieces of the Go libraries they depended on, so they rewrote in Rust. The post is long and details how much time and consideration it took. The contrast: a few days before this lecture, Discord's CTO posted casually on Twitter that they are rebuilding the whole thing in Rust, backend plus all the front-end applications. Five or ten years ago you would not announce that casually, because it meant a year or more with no new user-facing features. The cost of a rewrite is no longer prohibitive, which shifts where the tactical/strategic line sits.

### 8. Case study: designing an Iceberg-style table format

Caveats from the lecture: this example is deliberately more complex than the ones this lecture used in prior semesters, it is a distributed systems problem, jargon has been stripped, and you should not worry if it feels complicated. The payoff is that 61B primitives solve it.

**Background, part 1: data is enormous.** OpenAI has 600+ PB; Netflix has 1+ EB. Your laptop is maybe 1 TB, and that includes everything, not just data. So: no single device can hold it, and owning the drives is economically infeasible. (The 1990s alternative was every company running its own server racks with its own cooling, electrical, and mechanical engineering. The lecture's aside: Jane Street hung racks from the ceiling anchored to the building's concrete foundations because the racks were too heavy for the floor.)

**Background, part 2: big data access is analytical.** Queries aggregate (`SUM`, `COUNT`, `AVG`, `MAX`, `MIN`) over a column and group rows by some dimension (`GROUP BY`, `HAVING`). "How much inventory do I have left per state?" becomes `GROUP BY` the state column, `COUNT` inventory items by code in the item column, `SUM` value using the price column. The takeaway is not the SQL: it is that queries touch *a few specific columns*, not all the data in every row.

**Background, part 3: cloud object storage.** AWS S3, Azure Blob/Data Lake Storage, Google Cloud Storage. Data centers providing the illusion of highly reliable storage, accessed over the internet. Remarkably cheap, because the cooling/electrical/mechanical costs are amortized across all customers. It separates storage from compute so each scales independently (previously you scaled them together, the way your laptop bundles processor and memory; now you store cheaply and pay for compute only when you ask questions). It is very slow by storage standards, on the order of seconds versus nanoseconds-to-microseconds locally, but has high throughput. Structurally it is a `Map<String, bytes>` giving you a *file* interface. We want a *table* interface: rows, columns, something spreadsheet-like you can manipulate.

**Background, part 4: distributed execution.** A **driver** receives the question, plans how to execute it, splits it into tasks, and assigns them to **workers**, each reading one or more data shards. Sometimes one worker spans multiple shards; sometimes multiple workers hit the same shard. The analogy: instead of you alone counting how many times "Berkeley" appears across a stack of books, you hand one book to each of five people; you are the driver, they report back.

**Background, part 5: failure is a when, not an if.** Worker crashes lose compute, which is recoverable (hand the book to someone else). Crashes *during a write* are the hard case: a writer died mid-way, and you do not know how much it wrote or whether the data is corrupt. The lecture's image is five people each writing a page of an essay and the middle one walking out silently.

**The problem statement:** design a distributed large-scale data storage system that handles failures, as simply as possible, using 61B principles.

The design then proceeds as a sequence of questions. For each, the student answer from the recording is given, since the exam-relevant content is the *reasoning*, not the Iceberg trivia.

**Approach 1: which files are relevant to a query?** Student answer: metadata. Correct. Add metadata files (data about your data) describing what each data file contains, organized as a **tree** with leaves pointing to data files: `table1` at the root, `meta1`/`meta2`/`meta3` beneath, each pointing to `data1`/`data2`/`data3`. This lets you skip irrelevant files, called **pruning**.

**File-level optimizations.** Minimize, in priority order: (1) the number of files read, (2) the amount of content within each file. The priority exists because loading a file from S3 goes over the internet and takes on the order of seconds largely independent of file size, while jumping around inside a file already in memory is fast. The three techniques:

1. **Add statistics to the tree** so you know which paths are worth exploring. (A student suggested "sort it like a binary search tree," which is the right instinct; the lecture notes the real-world structure is a B-tree, covered in CS 186, and that a B-tree is closely related to the LLRB you already know.)
2. **Scan files lazily using iterators**, so a file is opened only when required, as late as possible.
3. **Store data in each file in a columnar format**, since queries care about columns, so co-locating a column's values beats co-locating a row's. (File loading mechanics go deeper in CS 61C.)

**Tree-level statistics in action.** Metadata nodes carry key ranges: `meta1` covers "A" to "C", `meta2` covers "D" to "G", `meta3` covers "M" to "Q". Query: where is Monterey? "M" falls in `meta3`'s range, so you read `data3` and never open `data1` or `data2`. This is exactly BST-style search-space elimination, applied to files instead of nodes.

**Approach 2: surviving write failures and partial writes.** A student proposed backups and restoring the last good state, which is close. The lecture's answer is sharper: **we do not support partial writes at all.** Make files **immutable**. Every update writes new data files and new metadata files; nothing is ever edited in place. A partially-written set of files is simply a set of files nobody points at.

Then the lecture asks: is immutability complexity or a feature? A student answered "both," and elaborated that it is more complicated because editing a file in place would be simpler than creating new files plus new metadata, and useful because you now inherently have backups and updated metadata. The lecture confirms both, with one correction (you do *not* have to copy all the data, see immutable writes below):

- **Feature: time travel.** You can read any historical version. Companies routinely go back years; a trading firm can backtest a strategy against years of stored prices.
- **Complexity: cost and cleanup.** Holding many files is more expensive and demands good cleanup. You can come back and **combine** files. Streaming workloads in particular create many tiny files that are fast to write and slow to read, so you periodically **compact** them.

**Immutable writes reuse old files.** Version v1 has `meta1` ("A"-"C"), `meta2` ("D"-"G"), `meta3` ("M"-"Q"). Adding "Hayward" produces version v2 that writes one new data file `data4` and one new metadata file `meta4` ("H"), and *points at the existing* `meta1`, `meta2`, `meta3`. Only the changed path is rewritten. This is structural sharing, and it is why immutability does not mean copying everything. (extra context: this is the same persistent-data-structure trick as a path-copying immutable tree, where an insert rebuilds only the nodes along one root-to-leaf path and shares the rest.)

**Approach 3: concurrent writers.** Two problems: two writers writing at the same time with divergent edits, and a past writer that created files then died, leaving files whose relevance and deletability are unknown. Student suggestions during the lecture, and their dispositions:

- Branches, like version control: branches *are* an Iceberg feature built on the metadata tree, but they are not the mechanism needed here.
- One writer per bucket / locking: this is **pessimistic lock control**, how traditional transactional databases work, one active writer holding a lock. Rejected here because we want multiple concurrent writers.
- Forward-only progression: works on the read path, not for concurrent writes.

The lecture's answer: **add a separate, much smaller service, the catalog.** Its one job is to hold a pointer to the latest metadata file. The data is at petabyte/exabyte scale; the catalog pointer is tiny. The protocol is **compare-and-swap**:

- The catalog accepts a write only if it agrees with the most recent table version.
- If the writer has the wrong version (another writer committed first), the catalog rejects it and tells the writer to start again.
- The rejected writer's files have no valid pointer to them, so they are garbage and get removed in cleanup.

The slide's framing is the important part: **this approach defines concurrency bugs out of existence.** There is no torn state to reason about, because unreferenced files are not state. The speaker's honest aside: this is "kind of cheating" and amounts to "use a database," which is not a fair move in 61B, but it is genuinely how the real system works.

**Approach 4: N catalogs x M systems.** There are many catalog implementations. Naively, N catalogs and M compute/storage systems means **N x M** integrations. The fix, named explicitly: **polymorphism.** Define an interface (a set of APIs) that every catalog implements. Now you have **N + M** integrations, one per item to the central interface. Each catalog request is just "update the table given some version," and the storage system abstracts away the file-level changes, so the modules are deep.

**Approach 5: efficient deletes.** If every write creates new files, deleting one row out of a million would mean rewriting 999,999 rows. A student proposed time-based expiry, which the lecture says is exactly how time-travel file expiration works when the version set grows too large, but it is wrong for deleting a specific row (you cannot delete data just because nobody read it recently). The answer: **delete files**. Keep the original data file intact (`0: "alice"`, `1: "bob"`, `2: "charlie"`, ...) and write a tiny separate file recording `deleted: 1`. At read time, merge the data file with the delete file, suppressing index 1. Logically the row is gone; physically nothing large was rewritten. (This is the same tombstone idea as deletion marking, applied at file granularity.)

**Approach 6: efficient column renames.** Column names live inside data files, and you do not want to rewrite data files for a rename. Answer: **assign columns to IDs** and keep a map from IDs to field names. Data files store `id1`, `id2`, `id3`, `id4`; the map says `id1 -> Timberwolves`, `id2 -> Sixers`, `id3 -> Grizzlies`, `id4 -> Knicks`. A rename changes the map, not the data. One map update replaces a full rewrite. (The lecture's table holds basketball players under each ID: `id1` has Ant, LaMelo, Gobert; `id2` has LeBron, JB, Maxey; and so on. Ty Jerome appears under both names in the example because nobody would recognize either half alone.)

**The assembled solution.**

1. A catalog service stores a pointer to the last correct table state.
2. A new write arrives and the catalog verifies it: the last correct table state is updated, and any writes in progress must restart.
3. Reads continue uninterrupted, either on the latest version or on a previous version (time travel).

Further optimizations the lecture names without developing: **hashing file names** to spread data files across object storage and optimize retrieval/network latency; **catalog caching** to speed up commonly accessed tables, keeping a local copy instead of re-reading metadata each time; and **advanced filters and compressed data structures** for stats, indexes, and delete files, specifically **Roaring bitmaps** and **Bloom filters**. The real Iceberg spec looks very similar to what the lecture derived, with one extra layer added because of the sheer number of files they must spread out.

### 9. Final takeaways

- Build simple things. Even in an LLM world, steer systems to be minimally complex and easy to understand. Note how every design decision in the case study could have gone the other way: you could have insisted an LLM "support partial writes," and made the system permanently worse.
- The real world gives you ambiguous problems with loosely defined constraints. Your job is to make sense of them, then pick the best solution given constraints while weighing tradeoffs.
- 61B concepts are genuinely applicable. It feels like you are learning random things that nobody will ever ask about at an internship, but trees, iterators, interfaces, immutability, and maps are what the case study was built out of.
- Good code is more than working code.
- Code complexity scales with functionality and poses real maintenance challenges.
- If this is interesting, read engineering blogs.

---

## Definitions

- **Complexity**: anything related to the structure of a software system that makes it hard to understand and modify it (John Ousterhout, *A Philosophy of Software Design*).
- **Unavoidable (essential) complexity**: inherent, inescapable complexity caused by the underlying functionality itself; a floor you cannot design below.
- **Avoidable complexity**: complexity that results from your design choices and can therefore be removed by better choices.
- **Unknown unknowns**: things you do not know that you need to know in order to safely modify code; a leading source of bugs in large codebases.
- **Abstraction**: the ability to use a piece of a system without understanding how it works, relying only on a specification.
- **Module**: a unit of a system with a defined boundary, used so that any one programmer (or model) only needs to understand a fraction of the codebase.
- **Deep module**: a module whose interface is small relative to the functionality it hides, so callers issue simple requests and the module absorbs the complexity.
- **Interface**: a specification of behavior separated from implementation, letting you swap implementations (`Map` implemented by `HashMap` or `BSTMap`) without changing callers.
- **Polymorphism**: the mechanism by which many implementations satisfy one interface; used in the case study to reduce N x M catalog integrations to N + M.
- **Tactical programming**: programming focused on getting something working quickly via workarounds, with minimal time spent on overall design.
- **Strategic programming**: programming that emphasizes long-term strategy over quick fixes, paying planning time for maintainable, simple, future-proof code, and discarding an insufficient strategy rather than forcing it.
- **Software factory**: a setup where engineers design the systems that write code, focusing on guardrails, boundaries, test signal, and general direction rather than writing the code themselves.
- **Simulated test**: a non-deterministic test where an agent exercises the product like a user (inputs, screenshots, reasoning about results), complementing deterministic unit tests.
- **Nightly QA loop**: merging changes into a dev build each night and having agents run a list of common scenarios, producing a morning report of what works.
- **Meat proxy**: a person who adds no value by merely relaying questions to a model and relaying answers back; the lecture's cautionary image for Approach 1.
- **Analytical access pattern**: a query that aggregates (`SUM`, `COUNT`, `AVG`, `MAX`, `MIN`) over one or a few columns and groups rows by dimensions (`GROUP BY`, `HAVING`), as opposed to touching whole rows.
- **Cloud object storage**: data centers (S3, Azure Blob/Data Lake Storage, Google Cloud Storage) providing the illusion of highly reliable storage over the internet; cheap, separates storage from compute, slow per access but high throughput, and structurally a `Map<String, bytes>` with a file interface.
- **Driver**: in a distributed query, the coordinator that receives the question, plans execution, splits it into tasks, and assigns them to workers.
- **Worker**: a distributed compute unit executing a task against one or more data shards and reporting results to the driver.
- **Metadata file ("stats")**: a file describing what the data files contain (for example a key range like "A" to "C"), used to prune which data files a query must read.
- **Pruning**: using metadata/statistics to skip files or subtrees that cannot contain relevant data.
- **Columnar format**: storing values of the same column together within a file, rather than storing each row's fields together, to match analytical access patterns.
- **Immutable files**: files that are never edited in place; every update writes new data and metadata files, so partial writes are impossible and old versions remain readable.
- **Time travel**: the ability to read a previous version of a table, a direct feature consequence of immutability.
- **Compaction**: periodically combining many small files (typical of streaming writes, fast to write and slow to read) into fewer larger files.
- **Catalog**: a small separate service holding a pointer to the latest valid metadata file, which accepts a write only if it agrees with the most recent table version and otherwise rejects it and tells the writer to restart.
- **Compare-and-swap (as used here)**: the catalog's commit protocol, comparing the writer's expected version to the current one and swapping the pointer only if they match.
- **Pessimistic lock control**: the traditional database approach of allowing one active writer at a time via locks; considered and rejected here because concurrent writers are desired.
- **Delete file**: a small file recording which row indexes are deleted, merged with the data file at read time instead of rewriting the data file.
- **Column ID mapping**: storing columns under stable IDs plus a map from IDs to field names, so a rename changes the map rather than the data.
- **Roaring bitmap / Bloom filter**: compressed data structures mentioned as advanced ways to store stats, indexes, and delete information. (extra context: a Bloom filter is a space-efficient probabilistic set with false positives but no false negatives, exactly the right tradeoff for "might this file contain key k?" pruning.)

---

## Worked Examples

The lecture is conceptual, so its own "examples" are the sentinel node, the `Map` interface, and the six Iceberg design moves. The Java below makes each of those concrete. These are faithful reconstructions of the lecture's ideas in code, not code shown on the slides.

### Example 1: Sentinel nodes, the lecture's "make code simpler and more obvious"

Without a sentinel, `addFirst` and `removeLast` must each ask "is the list empty?" and "is this the only node?" and patch `front`/`back` accordingly.

```java
public class NoSentinelDLList {
    private static class Node {
        int item;
        Node prev, next;
        Node(int i) { item = i; }
    }

    private Node front;
    private Node back;
    private int size;

    public void addFirst(int x) {
        Node n = new Node(x);
        if (front == null) {
            front = n;
            back = n;
        } else {
            n.next = front;
            front.prev = n;
            front = n;
        }
        size += 1;
    }

    public int removeLast() {
        if (size == 0) {
            throw new IllegalStateException("empty");
        }
        int result = back.item;
        if (size == 1) {
            front = null;
            back = null;
        } else {
            back = back.prev;
            back.next = null;
        }
        size -= 1;
        return result;
    }
}
```

Step by step on `addFirst`: allocate the node, then branch. In the empty case both `front` and `back` must point at the new node. In the nonempty case you must wire `n.next`, `front.prev`, and then move `front`. Two distinct shapes of pointer surgery live in one method, and `removeLast` has the mirror-image problem (size 1 nulls both ends, size > 1 walks `back` backwards). Every future method repeats the same two-case reasoning.

With a sentinel, the empty case stops existing:

```java
public class SentinelDLList {
    private static class Node {
        int item;
        Node prev, next;
        Node(int i) { item = i; }
    }

    private final Node sentinel;
    private int size;

    public SentinelDLList() {
        sentinel = new Node(0);
        sentinel.next = sentinel;
        sentinel.prev = sentinel;
    }

    public void addFirst(int x) {
        Node n = new Node(x);
        n.prev = sentinel;
        n.next = sentinel.next;
        sentinel.next.prev = n;
        sentinel.next = n;
        size += 1;
    }

    public int removeLast() {
        if (size == 0) {
            throw new IllegalStateException("empty");
        }
        Node last = sentinel.prev;
        last.prev.next = sentinel;
        sentinel.prev = last.prev;
        size -= 1;
        return last.item;
    }
}
```

Box-and-pointer reasoning in words: after the constructor, the heap holds one `Node` box whose `next` and `prev` arrows both loop back to itself, and `sentinel` points at it. There is no `null` anywhere in the link structure. So in `addFirst`, the expression `sentinel.next` is always a real node (possibly the sentinel itself), and `sentinel.next.prev = n` never dereferences `null`. When the list is empty, `sentinel.next` *is* `sentinel`, so that line sets `sentinel.prev = n`, which is exactly the "also update the back pointer" special case, achieved with no branch. In `removeLast`, `sentinel.prev` is the last real node when size > 0, and `last.prev` is either an earlier node or the sentinel; either way the two reassignments splice it out correctly.

Why this is the lecture's point: the sentinel does not *handle* edge cases, it **removes them from existence**, which is the structural move. The only remaining check is the genuinely-essential one (removing from an empty list is a caller error, which the lecture's "never simplify away input validation at trust boundaries" instinct says to keep). Compare this to the Iceberg catalog, which likewise "defines concurrency bugs out of existence" rather than detecting and repairing them.

### Example 2: Interfaces and abstraction, the lecture's `Map` example

```java
public interface SimpleMap<K, V> {
    void put(K key, V value);
    V get(K key);
    boolean containsKey(K key);
    int size();
}
```

Two implementations, chosen on different grounds:

```java
public class BSTSimpleMap<K extends Comparable<K>, V> implements SimpleMap<K, V> {
    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> left, right;
        Node(K k, V v) { key = k; value = v; }
    }

    private Node<K, V> root;
    private int size;

    public void put(K key, V value) {
        root = put(root, key, value);
    }

    private Node<K, V> put(Node<K, V> n, K key, V value) {
        if (n == null) {
            size += 1;
            return new Node<>(key, value);
        }
        int cmp = key.compareTo(n.key);
        if (cmp < 0) {
            n.left = put(n.left, key, value);
        } else if (cmp > 0) {
            n.right = put(n.right, key, value);
        } else {
            n.value = value;
        }
        return n;
    }

    public V get(K key) {
        Node<K, V> n = root;
        while (n != null) {
            int cmp = key.compareTo(n.key);
            if (cmp < 0) {
                n = n.left;
            } else if (cmp > 0) {
                n = n.right;
            } else {
                return n.value;
            }
        }
        return null;
    }

    public boolean containsKey(K key) { return get(key) != null; }
    public int size() { return size; }
}
```

```java
public class ChainingSimpleMap<K, V> implements SimpleMap<K, V> {
    private static class Entry<K, V> {
        K key;
        V value;
        Entry<K, V> next;
        Entry(K k, V v, Entry<K, V> nx) { key = k; value = v; next = nx; }
    }

    private Entry<K, V>[] buckets;
    private int size;

    @SuppressWarnings("unchecked")
    public ChainingSimpleMap() {
        buckets = (Entry<K, V>[]) new Entry[16];
    }

    private int bucketOf(K key) {
        return Math.floorMod(key.hashCode(), buckets.length);
    }

    public void put(K key, V value) {
        int i = bucketOf(key);
        for (Entry<K, V> e = buckets[i]; e != null; e = e.next) {
            if (e.key.equals(key)) {
                e.value = value;
                return;
            }
        }
        buckets[i] = new Entry<>(key, value, buckets[i]);
        size += 1;
    }

    public V get(K key) {
        for (Entry<K, V> e = buckets[bucketOf(key)]; e != null; e = e.next) {
            if (e.key.equals(key)) {
                return e.value;
            }
        }
        return null;
    }

    public boolean containsKey(K key) { return get(key) != null; }
    public int size() { return size; }
}
```

Now the client:

```java
public class Client {
    static int countOccurrences(SimpleMap<String, Integer> counts, String[] words) {
        for (String w : words) {
            Integer prior = counts.get(w);
            counts.put(w, prior == null ? 1 : prior + 1);
        }
        return counts.size();
    }
}
```

What this does and why it matters: `countOccurrences` compiles against `SimpleMap` only. Swapping `new BSTSimpleMap<>()` for `new ChainingSimpleMap<>()` at the call site changes zero lines inside the method. That substitutability is the entire content of "abstraction: the ability to use a piece without understanding how it works based on some specification."

Environment reasoning in words: at the call site, the local variable `counts` holds a reference to an object on the heap. The *static* type of that variable is `SimpleMap`, which is what the compiler checks `counts.get(...)` against. The *dynamic* type of the heap object is `BSTSimpleMap` or `ChainingSimpleMap`, and that is what determines which `get` body actually runs at each call. Java resolves the method by the dynamic type, which is why one compiled method body works for both implementations.

And the lecture's reason for *choosing* between them, which is the 61B engineering judgment: pick `ChainingSimpleMap` for expected constant-time operations, but note the lecture's caveat, if your keys happen to hash to the same value under a bad hash function, every key lands in one bucket, every `get` degrades to a linked-list scan, and the hash map's advantage disappears. The BST version's performance depends on key order and balance instead. Same interface, different failure modes, and *that* is the tradeoff you are being trained to see.

### Example 3: The metadata tree with statistics, and lazy iterator scanning

This encodes Approaches 1 and the file-level optimizations: prune by stats first, open files as late as possible.

```java
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.NoSuchElementException;

public class MetadataNode {
    private final String minKey;
    private final String maxKey;
    private final List<MetadataNode> children = new ArrayList<>();
    private final String dataFilePath;

    public MetadataNode(String minKey, String maxKey, String dataFilePath) {
        this.minKey = minKey;
        this.maxKey = maxKey;
        this.dataFilePath = dataFilePath;
    }

    public void addChild(MetadataNode child) {
        children.add(child);
    }

    private boolean mightContain(String key) {
        return minKey.compareTo(key) <= 0 && key.compareTo(maxKey) <= 0;
    }

    public List<String> filesToScan(String key) {
        List<String> result = new ArrayList<>();
        collect(key, result);
        return result;
    }

    private void collect(String key, List<String> out) {
        if (!mightContain(key)) {
            return;
        }
        if (dataFilePath != null) {
            out.add(dataFilePath);
            return;
        }
        for (MetadataNode child : children) {
            child.collect(key, out);
        }
    }
}
```

Step by step for the lecture's own query, "Where is Monterey?":

1. Build the tree: root `table1` spanning "A" to "Q", with children `meta1` ("A" to "C", pointing at `data1`), `meta2` ("D" to "G", pointing at `data2`), `meta3` ("M" to "Q", pointing at `data3`).
2. `filesToScan("Monterey")` calls `collect` on the root. `mightContain` is true ("A" <= "Monterey" <= "Q"), and `dataFilePath` is `null` at the root, so recurse into children.
3. `meta1`: `"C".compareTo("Monterey") <= 0` is false, so `mightContain` returns false and we return immediately. `data1` is never opened.
4. `meta2`: same, "G" < "Monterey", pruned. `data2` is never opened.
5. `meta3`: "M" <= "Monterey" <= "Q" holds, and `dataFilePath` is non-null, so `data3` is added and we stop descending.
6. Result: `["data3"]`. One file instead of three.

This is literally BST search logic. The only difference is that eliminating a subtree saves you a multi-second network fetch rather than a pointer dereference, which is why the lecture ranks "minimize number of files" above "minimize content per file."

Now the lazy part, so that even the one surviving file is opened only if the caller actually consumes it:

```java
public class LazyFileScanner implements Iterable<String> {
    private final List<String> paths;

    public LazyFileScanner(List<String> paths) {
        this.paths = paths;
    }

    public Iterator<String> iterator() {
        return new Iterator<String>() {
            private int fileIndex = 0;
            private Iterator<String> currentRows = null;

            private boolean advance() {
                while (currentRows == null || !currentRows.hasNext()) {
                    if (fileIndex >= paths.size()) {
                        return false;
                    }
                    currentRows = openFile(paths.get(fileIndex)).iterator();
                    fileIndex += 1;
                }
                return true;
            }

            public boolean hasNext() {
                return advance();
            }

            public String next() {
                if (!advance()) {
                    throw new NoSuchElementException();
                }
                return currentRows.next();
            }
        };
    }

    private static List<String> openFile(String path) {
        List<String> rows = new ArrayList<>();
        rows.add(path + ":row0");
        rows.add(path + ":row1");
        return rows;
    }
}
```

Why this is lazy and why that matters: `new LazyFileScanner(paths)` performs no I/O at all. `openFile` runs for the first time only inside the first `hasNext()`/`next()` call, and file *k+1* is opened only after file *k* is exhausted. So a caller doing `for (String row : scanner) { if (matches(row)) return row; }` that finds its answer in the first file never pays the multi-second fetch for the rest. That is the slide's "scan files lazily using iterators, so files are opened when required." Note also that `hasNext` is implemented via the same `advance()` helper as `next`, which keeps the two from disagreeing, the same "remove the edge case" instinct as the sentinel.

### Example 4: Polymorphic catalog interface with compare-and-swap (Approaches 3 and 4)

This is the N x M to N + M move plus the concurrency protocol, in one piece.

```java
public interface Catalog {
    String currentMetadataPointer(String tableName);

    boolean compareAndSwap(String tableName, String expectedPointer, String newPointer);
}
```

```java
import java.util.HashMap;
import java.util.Map;

public class InMemoryCatalog implements Catalog {
    private final Map<String, String> pointers = new HashMap<>();

    public synchronized String currentMetadataPointer(String tableName) {
        return pointers.get(tableName);
    }

    public synchronized boolean compareAndSwap(String tableName,
                                              String expectedPointer,
                                              String newPointer) {
        String actual = pointers.get(tableName);
        boolean matches = (actual == null && expectedPointer == null)
                || (actual != null && actual.equals(expectedPointer));
        if (!matches) {
            return false;
        }
        pointers.put(tableName, newPointer);
        return true;
    }
}
```

```java
public class TableWriter {
    private final Catalog catalog;
    private final String tableName;

    public TableWriter(Catalog catalog, String tableName) {
        this.catalog = catalog;
        this.tableName = tableName;
    }

    public boolean attemptCommit(String newRowKey, String newRowValue) {
        String basePointer = catalog.currentMetadataPointer(tableName);

        String newDataFile = writeImmutableDataFile(newRowKey, newRowValue);
        String newMetadataFile = writeImmutableMetadataFile(basePointer, newDataFile);

        return catalog.compareAndSwap(tableName, basePointer, newMetadataFile);
    }

    private String writeImmutableDataFile(String key, String value) {
        return "data-" + key + "-" + System.nanoTime();
    }

    private String writeImmutableMetadataFile(String basePointer, String newDataFile) {
        return "meta[base=" + basePointer + ",new=" + newDataFile + "]";
    }
}
```

Trace two concurrent writers, which is the whole point:

1. Writer A reads `currentMetadataPointer("inventory")` and gets `"meta-v1"`. Writer B reads it at the same moment and also gets `"meta-v1"`.
2. Both write their own brand-new, immutable data and metadata files. These writes do not conflict, because nothing is edited in place: A's files and B's files are entirely separate objects in storage.
3. A calls `compareAndSwap("inventory", "meta-v1", "meta-A")`. The stored pointer is still `"meta-v1"`, so it matches, the pointer becomes `"meta-A"`, and A returns `true`.
4. B calls `compareAndSwap("inventory", "meta-v1", "meta-B")`. The stored pointer is now `"meta-A"`, which does not equal `"meta-v1"`, so the swap is refused and B returns `false`. B must restart from the new current pointer.
5. B's files are still sitting in object storage, but **no valid pointer reaches them**, so they are unreachable garbage and get removed by cleanup. This is exactly the lecture's answer to "a past writer creates files but fails midway, how do we know to delete them?": reachability from the catalog pointer is the sole definition of what counts as part of the table.

The design payoff, in the slide's words: this **defines concurrency bugs out of existence.** There is no window in which a reader can observe a half-applied write, because the only way a write becomes visible is a single atomic pointer swap, and the files it points to were complete before the swap was attempted. Contrast with the rejected pessimistic-locking design, which serializes writers to get the same safety.

The `Catalog` *interface* is the Approach 4 move. Every storage/compute system programs against `Catalog`; every concrete catalog implements it. `TableWriter` does not know or care whether it is talking to `InMemoryCatalog` or anything else, which is the same static-type/dynamic-type dispatch as Example 2, applied at a system boundary. N + M integrations instead of N x M. And the interface is deep: two methods hide all the file-level machinery.

### Example 5: Delete files and column ID mapping (Approaches 5 and 6)

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class TableView {
    private final List<String> dataRows;
    private final Set<Integer> deletedIndexes;
    private final Map<String, String> columnIdToName;

    public TableView(List<String> dataRows,
                     Set<Integer> deletedIndexes,
                     Map<String, String> columnIdToName) {
        this.dataRows = dataRows;
        this.deletedIndexes = deletedIndexes;
        this.columnIdToName = columnIdToName;
    }

    public List<String> visibleRows() {
        List<String> out = new ArrayList<>();
        for (int i = 0; i < dataRows.size(); i += 1) {
            if (!deletedIndexes.contains(i)) {
                out.add(dataRows.get(i));
            }
        }
        return out;
    }

    public String displayNameOf(String columnId) {
        return columnIdToName.get(columnId);
    }

    public static void main(String[] args) {
        List<String> rows = new ArrayList<>();
        rows.add("alice");
        rows.add("bob");
        rows.add("charlie");

        Set<Integer> deleted = new HashSet<>();
        deleted.add(1);

        Map<String, String> names = new HashMap<>();
        names.put("id1", "Timberwolves");
        names.put("id2", "Sixers");

        TableView v = new TableView(rows, deleted, names);
        System.out.println(v.visibleRows());
        System.out.println(v.displayNameOf("id1"));

        names.put("id1", "Wolves");
        System.out.println(v.displayNameOf("id1"));
    }
}
```

Walking through it:

1. `dataRows` is the untouched immutable data file: indexes 0, 1, 2 holding "alice", "bob", "charlie". Nothing in this example ever rewrites it.
2. `deletedIndexes` is the delete file, containing just `1`. It is tiny regardless of how large the data file is, which is exactly the point: deleting one row out of a million writes a handful of bytes instead of 999,999 rows.
3. `visibleRows()` is the read-time merge the lecture described. It walks the data file and suppresses any index present in the delete set, printing `[alice, charlie]`. Logically "bob" is gone; physically "bob" is still on disk and is still visible to a time-travel read of an older version that does not include this delete file.
4. `displayNameOf("id1")` prints `Timberwolves`. Then `names.put("id1", "Wolves")` performs a column rename, and the same call now prints `Wolves`. The data file was never consulted, let alone rewritten, because data files store IDs and the name lives only in the map.

The common structure across both halves: **indirection converts an O(data size) rewrite into an O(1) metadata update.** That is the same reason the metadata tree avoids opening files and the same reason immutable writes reuse unchanged subtrees. One idea, applied three times.

---

## Common Pitfalls

- **Treating all complexity as avoidable.** There is an essential floor. You cannot build an `ArrayDeque` without an array and index management. Trying to abstract away essential complexity usually relocates it somewhere worse.
- **Treating all complexity as unavoidable.** The opposite error, and the one that produces the five-deep nested `if`. "It has to be this way" is usually "I did not go back to the drawing board."
- **Confusing "simple" with "short."** The sentinel version of `addFirst` is not fewer lines than the non-sentinel version; it is fewer *cases*. Count distinct situations you must reason about, not characters.
- **Thinking tactical programming is always wrong.** The lecture explicitly endorses it for prototypes, proofs of concept, and fast feedback, and says it is great for LLMs. The pitfall is not using it, it is *shipping* it, which happens constantly for lack of time.
- **Believing a prototype will be rewritten before deployment.** The slide says the opposite is common: "often, the prototype ends up deployed in the real world due to a lack of time."
- **Forcing an insufficient strategy to work.** The strategic-programming discipline is to discard the plan and replan, not to keep patching a plan you already know is wrong.
- **Assuming "the model writes it, so design does not matter."** The lecture's argument runs the other way: a complex codebase is harder for models too (finite context, more tokens, more reasoning, slower, more expensive). Simplicity is now a cost optimization as well as a comprehension one.
- **Becoming a meat proxy.** If your contribution is relaying prompts and answers, you have added nothing. The value-add is judgment about modules, abstraction, and simplicity.
- **Forgetting that "any one programmer should only need to understand a fraction of the codebase" is a requirement, not an observation.** Code that forces you to read three other files to understand one function violates it.
- **Designing for failure detection instead of failure impossibility.** The strongest moves in the case study did not detect and repair bad states, they made bad states unrepresentable: immutable files mean no partial writes exist; catalog compare-and-swap means no torn commit is visible; unreferenced files are definitionally not part of the table.
- **Optimizing bytes-per-file before files-per-query.** The lecture's priority order is explicit and counterintuitive: minimize the *number* of files first, because an S3 fetch costs seconds largely independent of size, while in-memory traversal is fast.
- **Eagerly opening files.** Without lazy iterators you pay network latency for files the query may never need.
- **Rewriting data for a metadata change.** Deleting a row by rewriting the file, or renaming a column by rewriting every data file, are the mistakes that delete files and column ID maps exist to prevent.
- **Assuming immutability is free.** It buys time travel and eliminates partial writes, but costs storage and demands active cleanup, file combining, and compaction (especially under streaming, which produces many tiny files that are fast to write and slow to read).
- **Reaching for locks reflexively on concurrency.** Pessimistic locking is a real answer (and is how traditional transactional databases work) but it serializes writers. The compare-and-swap design keeps writers concurrent and resolves conflicts by rejection plus retry.
- **Writing one integration per pair of systems.** N catalogs times M systems is N x M. One interface in the middle makes it N + M. This is the practical cash value of polymorphism.
- **Over-reading this lecture for exam purposes.** The lecturer says plainly that none of this is covered in course content, and encouraged students not to take notes. Study it for understanding and for the 61B connections it draws.

---

## Likely Exam Points

A caution up front: the lecturer stated that none of this lecture's material is course content, so none of it is likely to be tested directly. These are the conceptual checkpoints worth being able to answer anyway, since each one is a 61B idea wearing different clothes.

**1. Essential versus avoidable complexity**

*Q:* You are implementing a `BSTMap` and find yourself writing a private helper that handles the "node is null" case inside `put`. Is that complexity essential or avoidable? What about a `put` method with eight nested `if` statements distinguishing left/right, found/not-found, and root/non-root?

*A:* The null-node case is essential: a BST genuinely has empty subtrees and insertion genuinely must create nodes, so some representation of "nothing here yet" is inherent to the functionality. The eight nested `if`s are avoidable: recursive `put(Node n, K key, V value)` returning the (possibly new) subtree root collapses root/non-root and found/not-found into a single uniform shape, as in Example 2. Test for distinguishing them: ask whether the case exists because of *what the data structure is* or because of *how you chose to write it*.

**2. Why complexity scales faster than functionality**

*Q:* A system has 10 features. You add an 11th. Roughly why is the added reasoning burden worse than one-eleventh of the existing burden?

*A:* Because the new feature must interact correctly with all 10 existing features, and those interactions can compose, so you must reason about combinations rather than items. Pairwise alone that is 10 new interactions on top of the existing 45; with composition it is worse. The slides state complexity scales exponentially with functionality for exactly this reason. The practical consequence is the Windows/Spotify observation: small improvements take months in large codebases, not because of effort but because of the verification surface.

**3. Unknown unknowns**

*Q:* Define unknown unknowns and explain why they are worse than ordinary bugs.

*A:* Unknown unknowns are the things you do not know you need to know in order to safely modify the code. Ordinary bugs are failures of a known check, so you can test for them. Unknown unknowns cannot be tested for because you do not know to write the test, which is why the lecture calls them one of the most common sources of bugs when deploying at scale, and especially common in large codebases.

**4. Tactical versus strategic programming**

*Q:* Give one situation where tactical programming is the correct choice, one where it is the wrong choice, and name the single discipline that most distinguishes strategic programming.

*A:* Correct: a prototype or proof of concept where the goal is to demonstrate that something could theoretically work and get fast feedback (and the lecture notes this is where LLMs shine). Wrong: a system that will be maintained and extended for years, where workarounds conflict, refactoring costs restarts, and the prototype gets deployed anyway for lack of time. The distinguishing discipline: if the strategy is insufficient, go back to the drawing board before continuing work, rather than forcing the existing plan to work.

**5. Interfaces and the N x M problem**

*Q:* You have 4 catalog implementations and 6 compute/storage systems that each need to talk to each catalog. How many integrations with and without a shared interface? Which OOP concept is doing the work?

*A:* Without: 4 x 6 = 24. With a shared API that every catalog implements: 4 + 6 = 10, one per item to the central interface. The concept is polymorphism, and the practical benefit is that each system programs against the interface's static type while the actual implementation is resolved at runtime by dynamic type.

**6. Why immutability eliminates partial writes**

*Q:* A writer crashes halfway through updating a table. Explain why an immutable-file design means there is no corrupted state to repair, and explain what defines whether its leftover files are part of the table.

*A:* Because files are never edited in place, the crashed writer only ever created *new* files; no existing file was left half-modified. A write becomes part of the table solely when the catalog's pointer is swapped to point at its new metadata file. The crashed writer never got to swap, so nothing reaches its files: reachability from the catalog pointer is the sole definition of table membership, and unreachable files are garbage removed in cleanup. The feature side-effect is time travel, since every old version's files still exist; the cost is storage plus cleanup, file combining, and compaction.

**7. Compare-and-swap versus locking**

*Q:* Two writers read version `v1` concurrently and each build a new metadata file. Describe what happens under the catalog's compare-and-swap protocol, and state one reason you would prefer it to pessimistic locking.

*A:* Both writers write their own separate immutable files without conflict. The first to call compare-and-swap finds the stored pointer still equal to `v1`, so the swap succeeds. The second finds the pointer now equal to the first writer's metadata file, so its expected version does not match, the catalog rejects it, and it must restart from the new current version; its orphaned files are cleaned up. Preferred over pessimistic locking because locking admits only one active writer at a time, whereas compare-and-swap lets writers proceed concurrently and only penalizes actual conflicts. The slide's framing: it defines concurrency bugs out of existence.

**8. Pruning with tree statistics**

*Q:* A metadata tree has children covering key ranges "A"-"C", "D"-"G", and "M"-"Q", each pointing at one data file. For the query "where is Monterey?", how many data files are read, and what 61B structure is this?

*A:* One, the file under "M"-"Q". "Monterey" falls outside the first two ranges, so those subtrees are pruned and their data files are never opened. This is BST-style search-space elimination applied to files, and in production the structure is a B-tree (CS 186), closely related to the LLRB from 61B. Why it matters more here than in a BST: each pruned subtree saves a multi-second network fetch, not a pointer dereference.

**9. The file-count priority**

*Q:* You can either halve the number of files a query reads or halve the bytes within each file. Which do you do first, and why?

*A:* Halve the number of files. Loading a file from cloud object storage happens over the internet and costs on the order of seconds largely independent of the file's size, whereas once a file is in memory, jumping around inside it is fast. Hence the slides' ordering: primarily the number of files, secondarily the content within them. Both are then helped by columnar storage (analytical queries touch few columns) and lazy iterators (open a file only when actually required).

**10. Delete files and column ID maps**

*Q:* In an immutable-file system, how do you delete one row out of a million, and how do you rename a column, without rewriting the data file? What is the common principle?

*A:* For the delete, write a small separate delete file recording the deleted index, and merge it with the data file at read time to suppress that row. For the rename, store columns under stable IDs in the data files and keep a map from ID to field name, so a rename updates the map only. The common principle is indirection: both convert an O(data size) rewrite into an O(1) metadata update, which is the same move as the metadata tree avoiding file opens and immutable writes reusing unchanged subtrees.

**11. Does AI change the complexity calculus?**

*Q:* Argue for the claim that design simplicity matters *more*, not less, when a model is writing the code.

*A:* Three reasons from the lecture. First, the constraint that motivates modularity applies to models too: a context window is finite, so no agent can hold the whole codebase either, which means "any one programmer should only need to understand a fraction of the codebase" generalizes to agents. Second, simplicity is now directly economic: a simpler codebase takes fewer tokens and less reasoning to work in, so it is cheaper and faster. Third, a model asked to satisfy a bad requirement will faithfully produce a worse system; in the case study, demanding "support partial writes" would have produced permanent complexity where immutability eliminated the problem outright. The human's contribution is choosing the requirement, which is the difference between steering and being a meat proxy.

---

## Summary

- 61A optimized for correctness; 61B optimizes for engineering judgment, choosing among correct options by weighing tradeoffs, on larger projects where you partly define the task.
- Hardware stopped being the binding constraint decades ago (Apollo versus your phone; RollerCoaster Tycoon's 99% assembly on a 90 MHz Pentium versus a 6 GHz i9; 8 MB of RAM in 1996 versus Minecraft's 8 GB). The constraint is how we plan and design.
- Complexity (Ousterhout): anything about a system's structure that makes it hard to understand and modify. Its cost is that small improvements take enormous effort: slower comprehension, harder bug fixes, harder modification, and **unknown unknowns**.
- Complexity scales much faster than functionality, because each new feature must interact with every existing feature in every combination. Spotify: 1B+ lines, 60M in production, thousands of components, components growing far faster than engineers.
- Two kinds: **essential** (inherent to the functionality, a floor) and **avoidable** (yours to remove).
- Levers on avoidable complexity: simpler and more obvious code (sentinel nodes remove edge cases rather than handling them), and modules, which give abstraction (use without understanding, per a specification) and interfaces (`Map` via `HashMap` or `BSTMap`), ideally **deep** modules with small interfaces hiding a lot.
- Two design requirements worth stating as rules: no individual can manage a large system, and no one should need to understand more than a fraction of the codebase.
- **Tactical programming**: fast, workaround-driven, nested `if`s; right for prototypes and proofs of concept and great for LLMs; costs are minimal design, conflicting workarounds, expensive refactors, and the prototype shipping anyway.
- **Strategic programming**: plan first for maintainable, simple, future-proof code; discard an insufficient strategy instead of forcing it. This is where human time goes in an LLM workflow, outlining solutions and rewriting tactical output.
- Rewrites used to be prohibitive (Discord's careful 2020 Go-to-Rust post) and now are casual (Discord's full Rust rebuild announced in a tweet), because LLMs changed the cost. Tactical creep comes from tacit knowledge walking out the door, deadlines, shifting business priorities, and partial stack migrations.
- AI posture: Approach 1 hands everything to the model; Approach 2 steers it. Approach 2 wins because simplicity that helps humans also makes codebases cheaper, more token-efficient, and faster for models. Don't be a meat proxy. Testing is now deterministic unit tests plus simulated agent tests plus nightly QA loops; engineers increasingly build the **software factory** rather than the software.
- Iceberg case study, each step a 61B primitive: a **metadata tree with key-range stats** to prune files (BST search, B-tree in practice); **lazy iterators** to open files only when required; **columnar storage** to match analytical access; **immutable files** so partial writes cannot exist, which buys time travel and costs storage plus cleanup and compaction, while reusing unchanged subtrees so no bulk copying is needed; a small **catalog** doing **compare-and-swap** on the latest metadata pointer, which defines concurrency bugs out of existence and makes unreferenced files definitionally garbage; a **catalog interface** turning N x M integrations into N + M via polymorphism; **delete files** merged at read time; and **column ID maps** so renames touch the map, not the data. Further optimizations: file-name hashing for retrieval, catalog caching, Bloom filters and Roaring bitmaps.
- Priority order for I/O: number of files first (seconds per network fetch, near-independent of size), content per file second.
- The recurring design move across every good decision here: make bad states **impossible** rather than detectable, and use indirection to turn O(data) rewrites into O(1) metadata updates.
- Closing: build simple things; the real world hands you ambiguous, loosely constrained problems and it is your job to make sense of them and choose well; 61B concepts genuinely transfer; good code is more than working code.
- Housekeeping from lecture: attendance poll at hugcode.com/mango, and the Q&A at the end was unrecorded.
