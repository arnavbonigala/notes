<!-- Mon, Nov 02, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 29: Software Engineering I: BYOW

## Overview

This lecture has two halves and almost no new data structures. The first half is a retrospective on Project 4C (the WordNet/synset web server): Professor Hug walks through what he considers the "clean" design (three maps supporting three fundamental queries) versus a messier but working student design (the "supergraph"), and uses the measured rise in clean designs across semesters (22% to 80%) to open a frank discussion about LLM-assisted design, why the course still asks you to decompose and write code yourself, and why academic honesty matters even when the stakes feel artificial. The second half is a live-coded preview of Project 5, BYOW ("Build Your Own World"): a 2D tile-based interactive world with three deliverables (a seeded random world generator, an interactive avatar you can walk around, and your own "ambition" features). The live coding builds a hexagon world bottom-up from a `drawLine` primitive, hits and fixes a fencepost bug, hardcodes a size-3 hexagon and then discusses how to generalize it into two nested loops, then adds a `while (true)` game loop reading keystrokes with `StdDraw.hasNextKeyTyped()` / `nextKeyTyped()`, where the avatar leaves a trail of `@` clones and we see both a lazy fix (clear the whole world each frame) and a clean fix (remember what used to be in the tile you stepped onto). The explicit instruction was: **do not focus on the code, focus on the process** (decompose into small verifiable primitives, test them visually, accept temporary mess, then refactor).

---

## Key Concepts

### 1. Design projects versus implementation projects

Projects 4C/4D and 5 are the course's two **design** projects. In 4C/4D you are given a very specific task (answer a fixed API's questions about English words) but complete freedom "under the hood." In Project 5 even the task is loose: you must build a 2D interactive world, but what the world *is* is up to you. The transferable skill is reasoning top-down: figure out which operations your program fundamentally needs, then choose instance variables and data structures that make those operations cheap, then write code.

### 2. The important lesson of 4C: pick data structures from your queries, not from your mental picture of the data

The lecture's claim is that 4C reduces to exactly three fundamental lookups:

1. Given a **word**, which synset IDs contain it? (This is where graph search starts.)
2. Given a **synset ID**, what are its immediate hyponyms? (These are the graph edges.)
3. Given a **synset ID**, which words does it contain? (This converts your answer set back into words for the web interface.)

If you write down those three queries first, the instance variables fall out almost mechanically:

```java
Map<Integer, List<Integer>> hyponyms;   // query 2: an adjacency list for a DAG
Map<String, Set<Integer>>   wordToIds;  // query 1
Map<Integer, List<String>>  idToWords;  // query 3
```

Once you have these, "the rest of the project is pretty straightforward." Crucially, the spec never literally said "here are your three queries"; extracting them was the puzzle. Hug's point: if you ask an LLM to summarize the spec and propose instance variables, you skip exactly the part that was worth doing, and you are left "filling in code," which you can already do.

### 3. A messier design still works, and still teaches you something

One student's submission (affectionately called the "supergraph") used an object-graph design instead: a `Map<String, SuperNode>` from word to node, and each node storing a list of children, a list of parents, and the words belonging to it. This is a tree-shaped mental model for something that is actually a DAG, and it had **no** data structure for query 3, so that code scanned the entire graph looking for nodes containing the target ID. It is slower and harder to follow, but it works, and the student who designed and debugged it learned more about the consequences of design choices than someone who outsourced the design. The takeaway is not "messy is fine forever," it is "you learn the cost of a design by living with it."

### 4. The data on clean designs, and how to read it

Fraction of sampled submissions classified as a "clean design" (three-map style):

| Semester | Clean designs |
|---|---|
| First offering of the project | 22% |
| Spring 2023 | about the same |
| Fall 2023 | 30% |
| Spring 2026 | 76% |
| This section (Fall 2026) | 80% |

Hug was careful about confounders: the spec has evolved, a design document is now mandatory, course staff are better trained and steer students toward particular designs, and the class composition has shifted (fewer EECS/CS majors, more Data Science and Applied Math). He also notes the classification itself was done by sampling and having an LLM label the designs. So LLM influence is a *suspected* major factor, not a proven one.

### 5. Telltale LLM code, and why it matters

```java
words.sort((a, b) -> Double.compare(wordCounts.get(b), wordCounts.get(a)));
```

This is a one-liner that sorts in descending order by a map value using a comparator lambda. It is perfectly good Java, and it is also not what a 61B student who has been writing explicit `Comparator` classes typically produces. The concrete request for 4D: do not submit code you cannot explain. Hug describes interviewing a candidate who had a lambda in their own code and could not say what a lambda was.

### 6. Why write code yourself if an LLM can do it

The reasons raised in the room, kept faithfully:

- **Comprehension / supervision:** you need to be able to read what the model wrote and know roughly what is happening inside, and to steer it.
- **Decomposition is the real skill:** programming is a low-level instantiation of breaking a problem into parts. Code is "almost like a playground where you can practice decomposition."
- **LLMs are weaker at structure at scale:** in large industrial systems, high-level structuring still needs human guidance (with the caveat that some of that review can itself be delegated to another model).
- **It is intellectually enjoyable and genuinely useful:** see a problem, imagine a solution structure, build it, and have it be yours. The analogy given was hand-making socks from raw wool at a sheep-shearing festival: humans doing human things.
- **A cynical reason, acknowledged openly:** the whole educational apparatus (autograders, assignments) is already built around this model, and changing it is enormous work.

The counterweight Hug grants: students increasingly build software, proofs, and essays out of high-level off-the-shelf parts, the nature of work is genuinely changing, and he will keep adapting 61B. The instruction to you is "make sure you're learning something useful here."

### 7. Honesty

From his CS 195 (Social Implications of Computing) class: in a hypothetical where you had personally promised not to look at published exam answers before a makeup exam, and would never be caught, about 50% admitted they would peek. Hug's framing: coursework is low-stakes and cheating on homework harms no one directly, which is exactly why the arguments have to be about something other than harm. Arguments raised:

- It "just feels wrong" (virtue ethics / aesthetics / upbringing), which is a real reason even if it does not reduce further.
- Self-interest: doing the work prepares you for exams, where the incentives *are* aligned.
- It changes your self-image and how you see others (we project our own behavior onto other people).
- Coordination: absent a prisoner's dilemma, we would all prefer the honest equilibrium.
- **Ethics is a habit:** "as soon as you clean up a minor violation, the next one is already on its way" (described as looking through a priority queue of ever-larger violations). People who practice acting consciously ethically on small things tend to make better choices on large ones.

### 8. Observations across CS

A few years ago there were roughly **7 times** as many Ed posts per student, office hours were packed (students sitting on the floor of the 4th floor of Soda), and a large fraction of students had to fully immerse themselves in projects to finish. The anecdote about Rachel (Fall 2014) bursting into tears of relief after passing the autograder frames the open question: is the old intensity good or bad? Hug has reduced 61B's difficulty somewhat, and students have reduced it somewhat on their own.

### 9. The Project 5 tile engine: a 2D array *is* the world

The whole engine rests on one representation: `TETile[][] world`, indexed as `world[x][y]` where **x is the column and y is the row, with y = 0 at the bottom of the screen**. This y-up convention is the single most common source of confusion (Hug himself mixed up x and y several times live). Consequences:

- A "top left" corner has the *largest* y in the shape, so going down a shape means **decreasing** y.
- `TETile.toString(TETile[][] world)` must print row `height - 1` first, so the printed text matches what you see on screen.
- Every slot must be non-null before rendering.

The rendering model is: you mutate the array, then hand the whole array to the renderer, which draws each tile. There is no "draw a circle at pixel (300, 412)" in your game logic; all drawing is "set tiles, then render."

### 10. Separation of concerns in the engine: `TETile`, `Tileset`, `TERenderer`

- `TETile` is an **immutable** value object: a character, a text color, a background color, a description (for mouseover text), an optional 16x16 image filepath, and an integer `id`. Its `equals` compares only `id`, so two tiles are "the same kind of thing" when their ids match, even if `colorVariant` gave one a slightly different shade.
- `Tileset` is a bag of `public static final TETile` constants (`AVATAR` id 0, `WALL` 1, `FLOOR` 2, `NOTHING` 3, `GRASS` 4, `WATER` 5, `FLOWER` 6, `LOCKED_DOOR` 7, `UNLOCKED_DOOR` 8, `SAND` 9, `MOUNTAIN` 10, `TREE` 11, `CELL` 12). Because the tiles are immutable, one shared object per tile type can be referenced from thousands of array slots safely.
- `TERenderer` owns all the StdDraw configuration: `TILE_SIZE = 16`, canvas size `width * 16` by `height * 16`, `setXscale(0, width)` and `setYscale(0, height)` so that world coordinates *are* drawing coordinates, a font chosen per-platform so Unicode glyphs like `♠` render, and `enableDoubleBuffering()`.

### 11. Double buffering and the game loop contract

`TERenderer.initialize` turns on double buffering, which means nothing appears on screen until you call `StdDraw.show()`. There are then two ways to draw a frame:

```java
ter.renderFrame(world);   // = StdDraw.clear(black) + drawTiles(world) + StdDraw.show()
```
or the manual version used in the live code, which gives you control over the clear color and the frame delay:
```java
StdDraw.clear();          // white canvas
ter.drawTiles(world);     // draw, but do not show
StdDraw.show(30);         // flush the buffer, then pause 30 ms
```
The `show(30)` pause matters: without it the loop spins as fast as the CPU allows. In lecture, the first attempt flickered badly precisely because the show/clear discipline was wrong.

### 12. Keyboard input: poll, do not block

```java
if (StdDraw.hasNextKeyTyped()) {
    char c = StdDraw.nextKeyTyped();
    ...
}
```
`nextKeyTyped()` crashes if there is nothing queued, so **always guard it with `hasNextKeyTyped()`**. `GameLoopDemo` shows the more robust pattern: an inner `while (StdDraw.hasNextKeyTyped())` that drains every unprocessed key before re-rendering, so fast typing is not dropped.

### 13. The avatar trail bug: mutating a world that remembers

Writing `world[playerX][playerY] = Tileset.AVATAR` each frame, and only updating `playerX`, leaves an `@` behind at every tile you ever occupied, because the array is persistent state and nothing ever un-wrote the old slot. `StdDraw.clear()` does not help: it clears the *canvas*, not the *array*, and the array is re-drawn immediately afterward. Two fixes:

- **Lazy fix (used live):** at the top of each loop iteration, reset every tile to `Tileset.NOTHING` and re-generate the world from scratch. Correct for this toy, but it throws away any state the world accumulated and is O(width \* height) per frame on top of regenerating content.
- **Clean fix (in `HexWorldFinished`):** keep a variable `whatUsedToBeInMyNewSpot` holding the tile that the avatar is currently standing on, restore it into the vacated square when you move, then save the tile at the new square before overwriting it with the avatar.

### 14. Seeded randomness

`RandomWorldDemo` keeps `private static final long SEED` and `new Random(SEED)`, so the same seed always generates the same world. This is the core requirement of Project 5 part 1 (a world is reproducible from its seed) and is also what makes a randomly generated world testable. Note the comment in the code: in `RANDOM.nextInt(3)`, the 3 is a **bound**, not a seed, giving 0, 1, or 2.

### 15. The process lesson (the actual point of the lecture)

Watch the order of operations in the live coding:

1. Get a window with an empty world on screen. Verify.
2. Fill it with one tile type (sand). Verify.
3. Write the smallest primitive, `drawLine`. Verify visually with one call.
4. Build `drawHexagon` on top of `drawLine`, hardcoded for size 3. Verify.
5. Only then discuss generalizing to arbitrary size with two nested loops.
6. Only then think about tiling many hexagons, and choose the *ordering* that makes the arithmetic easiest.
7. Only then add interactivity.

Hug explicitly called out why step 3 mattered: he made a mistake in `drawLine` while talking, and found it quickly "because I limited the scope of my thoughts. If it were draw-a-hexagon with nested loops, I would have to think too much." This is the testing lecture's lesson applied to graphics: small units you can confirm by eye. He also sanctions temporary mess: "it is okay to use these hacks while working, but take a break from time to time, refactor, and think: was this the best way?" and "mess is good, I want you to face the mess."

---

## Definitions

- **BYOW (Build Your Own World):** Project 5, in which you build a 2D interactive tile world. Three parts: (1) a seeded random world generator, (2) interactivity (walk around with an avatar), (3) "ambition" features, i.e. something unique and interesting of your own design.
- **Design project:** a project where the task is given but the internal structure, data structures, and decomposition are entirely yours. 4C/4D and 5 are the course's two design projects.
- **Tile engine:** the provided rendering layer (`TETile`, `Tileset`, `TERenderer`) that turns a `TETile[][]` into a window of 16x16 character cells.
- **`TETile`:** an immutable object representing one tile, holding `character`, `textColor`, `backgroundColor`, `description`, optional image `filepath`, and an `int id`. Equality is defined by `id` alone.
- **`Tileset`:** a utility class of `public static final TETile` constants so the same tile objects are reused rather than re-allocated.
- **`TERenderer`:** utility class that configures StdDraw and draws a `TETile[][]`. `TILE_SIZE = 16` pixels per tile.
- **`initialize(w, h)` / `initialize(w, h, xOff, yOff)`:** creates a `w` by `h` tile window; the offset version shifts where `drawTiles` starts, leaving blank tiles on the left and bottom (useful for reserving space for a HUD/GUI).
- **`renderFrame(world)`:** `StdDraw.clear(black)`, then `drawTiles(world)`, then `StdDraw.show()`. One complete frame.
- **`drawTiles(world)`:** draws every tile without clearing and without showing. Throws `IllegalArgumentException` if any tile is null.
- **Double buffering:** drawing into an off-screen buffer and only copying it to the screen on `StdDraw.show()`, which eliminates flicker. Enabled inside `TERenderer.initialize`.
- **`StdDraw.show(ms)`:** flush the buffer to the screen, then sleep `ms` milliseconds (used as a crude frame limiter).
- **`StdDraw.hasNextKeyTyped()` / `nextKeyTyped()`:** poll whether an unprocessed typed key exists, and consume the next one. Calling `nextKeyTyped()` with an empty queue crashes.
- **Game loop:** a `while (true)` loop that each iteration (a) processes pending input, (b) updates world state, (c) renders a frame.
- **Biome:** in this lecture, simply the `TETile` used to fill a region (sand, grass, mountain, tree, flower, locked door), so that each hexagon looks like its own terrain type.
- **Hexagon of size `s`:** a tile shape whose top row is `s` wide, where each of the next `s - 1` rows grows by 2 and shifts left by 1, then mirrors vertically. Total height `2s`, maximum width `3s - 2`.
- **Adjacency list:** representation of a graph mapping each vertex to a collection of its out-neighbors; in 4C, `Map<Integer, List<Integer>>` from synset ID to immediate hyponym IDs.
- **DAG (directed acyclic graph):** the shape of the 4C hyponym relation, and the reason a "tree of parents and children" mental model is slightly wrong for it.
- **Clean design (as used in the 4C statistics):** the three-map design above, as opposed to designs that omit a needed index and compensate with a full scan.
- **Seed:** the `long` passed to `new Random(seed)`, fixing the entire pseudorandom sequence and therefore making a generated world reproducible.
- **Ambition features:** the open-ended third part of Project 5 where you add something distinctive (the lecture does not enumerate these).

---

## Worked Examples

All code below is from the lecture's own demo package.

> Note on syntax (extra context): the demos declare `void main()` (an instance main method with no `String[] args`) and use `IO.println`, both from very recent Java versions. On older Java you would write `public static void main(String[] args)` and `System.out.println`.

### Example 1: `BoringWorldDemo`, the minimal three-step pattern

```java
private static final int WIDTH = 60;
private static final int HEIGHT = 30;

void main() {
    TERenderer ter = new TERenderer();
    ter.initialize(WIDTH, HEIGHT);

    TETile[][] world = new TETile[WIDTH][HEIGHT];
    for (int x = 0; x < WIDTH; x++) {
        for (int y = 0; y < HEIGHT; y++) {
            world[x][y] = Tileset.NOTHING;
        }
    }

    for (int x = 20; x < 35; x++) {
        for (int y = 5; y < 10; y++) {
            world[x][y] = Tileset.WALL;
        }
    }

    ter.renderFrame(world);
}
```

Step by step:

1. `ter.initialize(60, 30)` sets the canvas to 960 by 480 pixels, sets the x and y scales to 0..60 and 0..30 so that tile coordinates are drawing coordinates, clears to black, and enables double buffering.
2. `new TETile[60][30]` allocates, in box-and-pointer terms, one array of 60 references, each pointing to an array of 30 references, and **every one of those 30 \* 60 references is `null`**. Rendering now would throw.
3. The first double loop points every slot at the single `Tileset.NOTHING` object. No tiles are copied: 1800 array slots now hold 1800 copies of the same address.
4. The second double loop repoints a 15-wide by 5-tall rectangle (x from 20 to 34, y from 5 to 9) at the single `Tileset.WALL` object. The old `NOTHING` references are simply overwritten.
5. `renderFrame` clears, calls `world[x][y].draw(x, y)` for every slot (each `draw` paints a filled square of the background color then the character in the text color, offset by 0.5 to center it in the cell), and shows. Because y = 5..9 is near the *bottom* of the screen, the wall block appears low and slightly left of center.

### Example 2: `RandomWorldDemo`, seeded generation

```java
private static final long SEED = 2873123;
private static final Random RANDOM = new Random(SEED);

public static void fillWithRandomTiles(TETile[][] tiles) {
    int height = tiles[0].length;
    int width = tiles.length;
    for (int x = 0; x < width; x++) {
        for (int y = 0; y < height; y++) {
            tiles[x][y] = randomTile();
        }
    }
}

private static TETile randomTile() {
    int tileNum = RANDOM.nextInt(3);
    return switch (tileNum) {
        case 0 -> Tileset.WALL;
        case 1 -> Tileset.FLOWER;
        default -> Tileset.NOTHING;
    };
}
```

What this does and why:

- `tiles.length` is the number of columns (the x dimension) and `tiles[0].length` is the number of rows (the y dimension). The method derives both from the array instead of reading the `WIDTH`/`HEIGHT` constants, so it works on any rectangular array you hand it. This is a small but real design win: the helper does not secretly depend on global state.
- `RANDOM` is created once with a fixed seed, so running the program twice draws the identical world. If you instead wrote `new Random()` inside `randomTile()`, you would get a different world every run and, worse, a fresh generator seeded from the clock on every single call, which can produce visible correlation artifacts.
- The `switch` expression with `->` and `default` returns a value directly. `default` covers `tileNum == 2`, which also satisfies the compiler's exhaustiveness requirement.
- Each of the three outcomes has probability 1/3 because `nextInt(3)` is uniform on {0, 1, 2}.

### Example 3: `drawLine`, including the bug Hug hit live

The horizontal-line primitive, written **first**, before any hexagon logic:

```java
public static void drawLine(TETile biome, int startX, int y, int size, TETile[][] world) {
    for (int i = 0; i < size; i += 1) {
        world[startX + i][y] = biome;
    }
}
```

The version first typed in lecture was essentially:

```java
for (int x = startX; x < size; x += 1) {   // BUG
    world[x][y] = biome;
}
```

Trace it with the live call `drawLine(Tileset.LOCKED_DOOR, 20, 15, 5, world)`:

- Buggy version: `x` starts at 20, the condition is `20 < 5`, which is false, so the loop body never runs. Nothing is drawn, and nothing is printed, so the program looks like it did nothing at all. A breakpoint on the loop body confirmed it never executed.
- The conceptual error: `startX` is a **position** and `size` is a **count**, and the loop mixed them. Comparing a coordinate against a length is a category error.
- Correct version: `i` runs 0, 1, 2, 3, 4 and writes `world[20][15]` through `world[24][15]`, exactly 5 tiles. To go from count to position you must add the offset to the start: the first slot is `startX`, the last is `startX + size - 1`.

Why this example is in the notes: this is the lecture's own argument for decomposition. The bug was found in seconds because the only thing on screen was supposed to be five characters in a row. The same bug buried inside a nested hexagon loop would have produced a confusing partial shape.

The refactored version in the `finished` package pulls this out into its own utility class, which is the shape you should aim for:

```java
public class DrawUtils {
    public static void drawHorizontalLine(int x, int y, int lineWidth, TETile biome, TETile[][] world) {
        for (int xOffset = 0; xOffset < lineWidth; xOffset += 1) {
            world[x + xOffset][y] = biome;
        }
    }
}
```

Note the parameter order changed to put `biome` late. Hug reordered arguments mid-lecture ("don't create problems for yourself") because inconsistent parameter order between `drawLine` and `drawHexagon` was itself a bug source.

### Example 4: `drawHexagon`, hardcoded for size 3

```java
public static void drawHexagon(TETile biome, int topLeftX, int topLeftY, int size, TETile[][] world) {
    drawLine(biome, topLeftX,     topLeftY,     size,     world);
    drawLine(biome, topLeftX - 1, topLeftY - 1, size + 2, world);
    drawLine(biome, topLeftX - 2, topLeftY - 2, size + 4, world);
    drawLine(biome, topLeftX - 2, topLeftY - 3, size + 4, world);
    drawLine(biome, topLeftX - 1, topLeftY - 4, size + 2, world);
    drawLine(biome, topLeftX,     topLeftY - 5, size,     world);
}
```

Called as `drawHexagon(Tileset.LOCKED_DOOR, 20, 15, 3, world)`, the six rows are, from top (y = 15) down to bottom (y = 10):

| Row (y) | Start x | Length | Covers x |
|---|---|---|---|
| 15 | 20 | 3 | 20..22 |
| 14 | 19 | 5 | 19..23 |
| 13 | 18 | 7 | 18..24 |
| 12 | 18 | 7 | 18..24 |
| 11 | 19 | 5 | 19..23 |
| 10 | 20 | 3 | 20..22 |

Reasoning in words: start at the top-left corner. Each row down, the shape begins one column further left and is two tiles longer, which is exactly what makes the diagonal edges. After `size` rows you are at the widest point, and then the pattern mirrors: shift one column right, shrink by two, until you return to the original width and x. Lengths 3, 5, 7, 7, 5, 3 sum to 30 tiles.

Two honest observations the lecture made about this code:

- It is **wrong for any size other than 3**: the number of lines is baked in as 6, so a size-4 hexagon would need 8 lines. The class constant `HEX_SIZE = 4` in `HexWorld` is never actually passed here; the literal `3` is.
- The orientation was discovered experimentally. The first attempt increased y and drew the hexagon upward from the "top left," which is backwards. In a y-up world, moving down a shape means subtracting from y. Doing a quick run to check the orientation is a legitimate move on a project like this.

### Example 5: generalizing to the `Hexagon` class (two nested loops)

The hint given in lecture, restated as code in the `finished` package. The key insight is that the awkward offset sequence (-1, -2, -2, -1, 0) is awkward only because you are treating the hexagon as one loop. Split it into the top half and the bottom half and each half has a clean monotone pattern.

```java
public class Hexagon {
    private int size;
    private TETile biome;
    private int topLeftX;
    private int topLeftY;

    public Hexagon(int size, TETile biome, int topLeftX, int topLeftY) {
        this.size = size;
        this.biome = biome;
        this.topLeftX = topLeftX;
        this.topLeftY = topLeftY;
    }

    /* I'm destructively putting myself in the world. */
    public void addHexagon(TETile[][] world) {
        // top half: size lines, each 2 longer and 1 further left than the last
        for (int lineNum = 0; lineNum < size; lineNum += 1) {
            drawHorizontalLine(topLeftX - lineNum, topLeftY - lineNum,
                               size + lineNum * 2, biome, world);
        }

        int startBottomHalfX = topLeftX - size + 1;
        int startBottomHalfY = topLeftY - size;
        int startBottomHalfSize = size + size * 2 - 2;   // = 3 * size - 2

        // bottom half: size lines, each 2 shorter and 1 further right
        for (int lineNum = 0; lineNum < size; lineNum += 1) {
            drawHorizontalLine(startBottomHalfX + lineNum, startBottomHalfY - lineNum,
                               startBottomHalfSize - lineNum * 2, this.biome, world);
        }
    }
}
```

Verification against Example 4 with `size = 3`, `topLeftX = 20`, `topLeftY = 15`:

- Top half: `lineNum = 0` gives (20, 15, 3); `lineNum = 1` gives (19, 14, 5); `lineNum = 2` gives (18, 13, 7). Matches rows 15, 14, 13.
- Bottom-half anchors: `startBottomHalfX = 20 - 3 + 1 = 18`, `startBottomHalfY = 15 - 3 = 12`, `startBottomHalfSize = 3 + 6 - 2 = 7`.
- Bottom half: `lineNum = 0` gives (18, 12, 7); `lineNum = 1` gives (19, 11, 5); `lineNum = 2` gives (20, 10, 3). Matches rows 12, 11, 10. The generalized loops reproduce the hardcoded hexagon exactly, and now also work for size 4, 5, and beyond.

Design notes worth absorbing:

- Hug originally chose a **static function** taking the world, explicitly flagging that as a decision ("I made the choice that this is a hexagon function ... and that it is not a separate class"). The finished version chose a **class** instead, with size, biome, and position as instance variables. Both work; the class version is nicer once hexagons acquire more behavior, because the parameter list stops growing.
- `addHexagon` is documented as destructive: it mutates the world you pass in rather than returning a new one. The comment also notes bounds checking was skipped, so an out-of-range hexagon will throw `ArrayIndexOutOfBoundsException`.

### Example 6: tiling hexagons by columns

The discussion asked: in what order do you place many hexagons? Options raised and critiqued:

- **Spiral** (like laying out Catan): "a real hassle," the index arithmetic is painful.
- **Row by row:** awkward, because the rows are not regular. The top row has one hexagon, the next has two that are not adjacent to each other, the next has three that are not adjacent.
- **Middle outward:** even more complicated.
- **Column by column:** preferred, "because the calculations are very simple." Within a column, successive hexagons are a fixed vertical distance apart (`2 * size`), and each next column is a fixed step right and up.

```java
private static void drawColumnOfHexagons(int x, int y, int numHexagons, TETile[][] world) {
    for (int i = 0; i < numHexagons; i += 1) {
        Hexagon h = new Hexagon(HEX_SIZE, pickRandomBiome(), x, y - HEX_SIZE * 2 * i);
        h.addHexagon(world);
    }
}

private static TETile pickRandomBiome() {
    int choice = StdRandom.uniformInt(5);
    switch (choice) {
        case 0: return Tileset.SAND;
        case 1: return Tileset.FLOWER;
        case 2: return Tileset.MOUNTAIN;
        case 3: return Tileset.TREE;
        case 4: return Tileset.GRASS;
        default: return Tileset.LOCKED_DOOR;
    }
}
```

and the five columns that make the Catan-style patch (3, 4, 5, 4, 3 hexagons):

```java
int firstX = 6;
int firstY = 36;
drawColumnOfHexagons(firstX,                  firstY,                 3, world);
drawColumnOfHexagons(firstX + 2*HEX_SIZE - 1, firstY + HEX_SIZE,      4, world);
drawColumnOfHexagons(firstX + 4*HEX_SIZE - 2, firstY + 2*HEX_SIZE,    5, world);
drawColumnOfHexagons(firstX + 6*HEX_SIZE - 3, firstY + HEX_SIZE,      4, world);
drawColumnOfHexagons(firstX + 8*HEX_SIZE - 4, firstY,                 3, world);
```

Why the magic numbers are not magic:

- **Vertical step inside a column is `2 * HEX_SIZE`** because a hexagon of size `s` is exactly `2s` rows tall, so stacking at that spacing leaves no gap and no overlap. With `HEX_SIZE = 4` that is 8, and `y - 8 * i` walks downward.
- **Horizontal step between columns is `2 * HEX_SIZE - 1`.** The x arguments are `firstX + k * (2*HEX_SIZE - 1)` for k = 0..4, just written out with the multiplication distributed. For size 4 that is 7 columns of shift. A hexagon's widest row spans `3s - 2 = 10` columns, and neighboring columns interlock so that the narrow top of one sits beside the wide middle of the next.
- **Vertical offset between columns is `+ HEX_SIZE`,** half a hexagon's height, which is what staggers the columns so the hexagons nest instead of lining up.
- `pickRandomBiome` gives each hexagon its own terrain. Note it uses `StdRandom.uniformInt(5)` (unseeded by default), so the `default` branch returning `LOCKED_DOOR` is unreachable here but keeps the compiler happy; a `switch` with `return` in each arm needs a guaranteed return.

### Example 7: the live game loop and the avatar trail bug

The loop as written in lecture:

```java
int playerX = 21;
int playerY = 15;

while (true) {
    StdDraw.clear();
    for (int x = 0; x < WIDTH; x += 1) {           // the lazy fix
        for (int y = 0; y < HEIGHT; y += 1) {
            world[x][y] = Tileset.NOTHING;
        }
    }

    drawHexagon(Tileset.LOCKED_DOOR, 20, 15, 3, world);
    world[playerX][playerY] = Tileset.AVATAR;
    ter.drawTiles(world);

    if (StdDraw.hasNextKeyTyped()) {
        char c = StdDraw.nextKeyTyped();
        if (c == 'a') {
            playerX -= 1;
            // we have a bug here
            //  in the next iteration,
            // world[playerX][playerY] will get an @
            // but the old one stays
        }
        if (c == 'd') {
            playerX += 1;
        }
    }

    StdDraw.show(30); // draws and waits 30 milliseconds
}
```

Trace of the bug (before the clearing loop was added). Suppose the avatar is at (21, 15).

1. Frame 1: `world[21][15] = AVATAR`, render. Screen shows one `@`.
2. You press `a`. `playerX` becomes 20. **Nothing resets `world[21][15]`,** so that slot still references `Tileset.AVATAR`.
3. Frame 2: `world[20][15] = AVATAR`, render. Screen shows `@` at 20 **and** at 21.
4. Hold `a` and you paint a horizontal streak of avatars across the world.

Why `StdDraw.clear()` did not fix it: `clear()` wipes the off-screen canvas, but the very next thing the loop does is redraw every tile from the array, and the array still says AVATAR at the old positions. The canvas is a view; the array is the truth.

The lazy fix shown live resets the entire array to `NOTHING` and regenerates the hexagon every frame, which makes the stale avatars disappear. Hug was explicit that this is "not necessarily a good idea" but fixes the problem for a toy: it is wasteful, and it would erase any world state (doors you unlocked, items you picked up) that is not regenerated from scratch each frame.

Also note the state of this code as left in lecture: only `a` and `d` are handled, there is no bounds checking (walking off the left edge of a 45-wide world throws `ArrayIndexOutOfBoundsException`), and there is no check preventing you from stepping onto `NOTHING` or through a wall. Those were handed to the student as exercises.

### Example 8: the clean fix in `HexWorldFinished`

```java
int avatarX = 5;
int avatarY = 9;

TETile whatUsedToBeInMyNewSpot = world[avatarX][avatarY];
world[avatarX][avatarY] = Tileset.AVATAR;

StdDraw.clear();
ter.drawTiles(world);
StdDraw.show();

while (true) {
    if (StdDraw.hasNextKeyTyped()) {
        char c = StdDraw.nextKeyTyped();

        if (c == 's') {
            avatarY -= 1;
            world[avatarX][avatarY + 1] = whatUsedToBeInMyNewSpot;
        } else if (c == 'a') {
            avatarX -= 1;
            world[avatarX + 1][avatarY] = whatUsedToBeInMyNewSpot;
        } else if (c == 'd') {
            avatarX += 1;
            world[avatarX - 1][avatarY] = whatUsedToBeInMyNewSpot;
        } else if (c == 'w') {
            avatarY += 1;
            world[avatarX][avatarY - 1] = whatUsedToBeInMyNewSpot;
        }

        whatUsedToBeInMyNewSpot = world[avatarX][avatarY];
        world[avatarX][avatarY] = Tileset.AVATAR;

        StdDraw.clear();
        ter.drawTiles(world);
        StdDraw.show();
    }
}
```

Step by step, with the single-variable invariant stated plainly: **`whatUsedToBeInMyNewSpot` always holds the tile that the avatar is currently covering.**

1. Before the loop, save `world[5][9]` (whatever the hexagon generator put there), then stamp the avatar on top. Invariant established.
2. Press `d`. `avatarX` increments to 6. Then `world[avatarX - 1][avatarY] = whatUsedToBeInMyNewSpot` writes the saved tile back into the square just vacated (x = 5). No trail.
3. After the branch, `whatUsedToBeInMyNewSpot = world[6][9]` saves whatever is at the destination, re-establishing the invariant, and then the avatar is stamped at (6, 9).
4. Clear, draw, show. Exactly one `@` on screen, and the terrain is undamaged.

In box-and-pointer terms this is clean because `TETile` is immutable and shared: `whatUsedToBeInMyNewSpot` is just a reference to one of the twelve `Tileset` constants, so "saving the tile" costs one reference copy and cannot be invalidated by anything else mutating that tile. The approach does not scale to multiple moving entities (each would need its own saved tile, and two avatars standing on each other's squares get confusing), which is why real implementations usually keep a separate terrain layer and entity layer. (extra context)

Note also that this version re-renders **only when a key is typed**, rather than on a fixed 30 ms clock. That is fine for a purely turn-based world and avoids burning CPU, but it means nothing can animate on its own.

### Example 9: `GameLoopDemo`, draining the input queue and switching on keys

```java
public static void toggle(TETile[][] world, int x, int y) {
    if (world[x][y].equals(Tileset.FLOOR)) {
        world[x][y] = Tileset.WALL;
    } else if (world[x][y].equals(Tileset.WALL)) {
        world[x][y] = Tileset.FLOOR;
    }
}
```

```java
char c;
while (true) {
    while (StdDraw.hasNextKeyTyped()) {
        c = StdDraw.nextKeyTyped();
        c = Character.toLowerCase(c);
        switch (c) {
            case '1': toggle(world, 0, 0); break;
            case '2': toggle(world, 1, 0); break;
            case '3': toggle(world, 2, 0); break;
            case '4': toggle(world, 3, 0); break;
            case '5': toggle(world, 4, 0); break;
            case 'q': System.exit(0); break;
            default: break;
        }
    }
    ter.renderFrame(world);
}
```

Points to notice:

- The **outer** `while (true)` keeps the game alive; the **inner** `while (hasNextKeyTyped())` processes every key that arrived since the last frame, so rapid keypresses are not lost. If there are no keys, the inner loop does not execute and we fall through to re-render.
- `renderFrame` is outside the inner loop, which is the right place: you want one render per frame, not one render per key.
- `toggle` uses `.equals`, not `==`. With the provided `Tileset` constants `==` would happen to work (they are the same objects), but `.equals` compares `id`, so it also works for `colorVariant` copies of a tile, which are different objects with the same id. Using `.equals` is the habit you want.
- `switch` with explicit `break` statements is used here as a readable replacement for a long if-else chain. Forgetting a `break` would fall through to the next case.
- `System.exit(0)` closes the StdDraw window and terminates the JVM. The `break` after it is unreachable but harmless.
- The comment in the source warns that if you do not fill the grid you get a crash on render. Precisely: the current `TERenderer.drawTiles` checks for null and throws `IllegalArgumentException("Tile at position x=..., y=... is null.")`. Either way, fill your grid.

### Example 10: `ExampleSaveLoad`, persisting state to a file

```java
void main() {
    String filename = "name.txt";
    File file = new File(filename);

    // Step 1: read the saved name if the file exists
    if (file.exists()) {
        In in = new In(file);
        if (in.hasNextLine()) {
            String savedName = in.readLine();
            IO.println("Hello, the last time you ran this program, your name was " + savedName);
        } else {
            IO.println("I don't know your name.");
        }
    } else {
        IO.println("I don't know your name.");
    }

    // Step 2: prompt for the new name
    IO.print("Please enter your name: ");
    In keyboard = new In();  // Reads from standard input
    String currentName = keyboard.readLine();

    // Step 3: save it
    Out out = new Out(filename);
    out.println(currentName);
    out.close();
}
```

What to take from it:

- `new File(filename)` does not touch the disk; it is just a path object, which is why you must call `file.exists()` before reading.
- Two guards, not one: the file might not exist, and it might exist but be empty, so `hasNextLine()` is checked separately.
- `new In(file)` reads from a file; `new In()` with no argument reads from standard input. Same class, two sources.
- `new Out(filename)` **overwrites** the file, and `out.close()` is what guarantees the bytes are actually flushed to disk. Forgetting `close()` is a classic way to end up with an empty save file.
- This is the seed of Project 5's save and load feature: on quit, write enough information (the seed plus the sequence of moves, for instance) that reloading reconstructs the identical world. (extra context: the specific save format is defined by the project spec, not by this lecture.)

---

## Common Pitfalls

1. **Forgetting to fill the `TETile[][]`.** A fresh `new TETile[W][H]` is all nulls. `drawTiles` throws `IllegalArgumentException` naming the offending (x, y) (the demo's comment says NullPointerException, which is what older versions of the engine did). Always do the initialize-to-`NOTHING` double loop.
2. **Mixing up x and y, or forgetting that y = 0 is the bottom.** `world[x][y]`, x is the column. Hug mixed these up repeatedly on purpose and by accident. If your shape appears upside down, you are probably adding to y where you should subtract.
3. **Confusing a position with a count.** The `drawLine` bug: `for (int x = startX; x < size; ...)` compares a coordinate against a length. The correct loop runs an offset `i` from 0 to `size - 1` and writes `world[startX + i][y]`.
4. **Expecting `StdDraw.clear()` to fix the avatar trail.** `clear()` erases the canvas, not the array. Stale state lives in the `TETile[][]`.
5. **Calling `nextKeyTyped()` without `hasNextKeyTyped()`.** This crashes when the queue is empty.
6. **Nothing shows up on screen at all.** Double buffering is on, so you must call `StdDraw.show()` (or use `renderFrame`, which calls it for you). A missing `show()` produces a blank or stale window.
7. **A busy loop with no delay.** `while (true)` with no `show(ms)` or pause will peg a core and may flicker. `StdDraw.show(30)` both flushes and sleeps.
8. **Processing only one key per frame.** A single `if (hasNextKeyTyped())` drops keystrokes under fast typing; `GameLoopDemo`'s inner `while` drains the queue.
9. **Hardcoding a shape and calling it generalized.** `drawHexagon` has six `drawLine` calls, so it silently produces a wrong shape for any `size != 3`, and the `HEX_SIZE = 4` constant in the same file is a trap: it is never passed to `drawHexagon`.
10. **No bounds checking.** `addHexagon` and `drawLine` will happily index out of the array ("could do bounds checking, but I'm craaazzayuy"), and the live movement code lets the avatar walk off the edge of the world.
11. **Creating a `Random` inside the generation helper, or using an unseeded `Random`/`StdRandom`.** You lose reproducibility, which breaks the "same seed, same world" requirement and makes the generator untestable.
12. **Treating `nextInt(3)` as a seed.** It is an exclusive upper bound. The seed goes in the `Random` constructor.
13. **Comparing tiles with `==` where `equals` is meant.** `TETile.equals` compares `id`, so it survives `colorVariant` recoloring; `==` does not.
14. **Mutating a world you did not intend to own.** `addHexagon` is destructive by design, and `TETile.copyOf` exists precisely for when you want a snapshot. (That copy duplicates the column arrays but shares the tile objects, which is safe only because `TETile` is immutable.)
15. **Inconsistent parameter order across helpers.** `drawLine(biome, x, y, size, world)` versus `drawHorizontalLine(x, y, lineWidth, biome, world)`: pick one convention, or you will pass arguments in the wrong slots. Hug reordered his live to avoid exactly this.
16. **Submitting code you cannot explain.** The explicit 4D request. The lambda one-liner is the canonical example.
17. **Refactoring never, or refactoring constantly.** The lecture's balance: hacks are acceptable while you make progress, but pause periodically, refactor, and ask whether this was the best way, so the program does not become incomprehensible.

---

## Likely Exam Points

(extra context: this lecture is primarily project preview and discussion, so BYOW-specific API trivia is unlikely to be heavily tested. What *is* fair game is 2D array index reasoning, loop and fencepost reasoning, reference semantics with shared immutable objects, `equals` versus `==`, and seeded randomness. The questions below target those.)

**1. Fencepost reasoning in a line-drawing helper.**

*Q:* Given `drawLine(TETile biome, int startX, int y, int size, TETile[][] world)` implemented as `for (int i = 0; i < size; i += 1) { world[startX + i][y] = biome; }`, which tiles does `drawLine(Tileset.WALL, 7, 3, 4, world)` set, and what is the largest x index it touches?

*A:* It sets `world[7][3]`, `world[8][3]`, `world[9][3]`, `world[10][3]`: four tiles. The largest index touched is `startX + size - 1 = 10`. A `world` array must therefore have length at least 11 in the x dimension to avoid `ArrayIndexOutOfBoundsException`.

**2. The buggy loop.**

*Q:* Suppose `drawLine` instead contained `for (int x = startX; x < size; x += 1) { world[x][y] = biome; }`. For which calls does it draw nothing, and for which does it draw something (possibly wrong)?

*A:* It draws nothing whenever `startX >= size`, which is the common case for coordinates in a 45-wide world (for example `startX = 20, size = 5`). It draws `size - startX` tiles when `startX < size`, starting at `startX`, which is a wrong but non-empty line. The bug is comparing a position against a count.

**3. Hexagon geometry.**

*Q:* For a hexagon of size `s` drawn by `Hexagon.addHexagon` with top-left corner `(topLeftX, topLeftY)`, give (a) the total height in rows, (b) the width of the widest row, (c) the y coordinate of the bottom row, and (d) the total number of tiles for `s = 4`.

*A:* (a) `2s` rows. (b) `3s - 2`, which is `size + size*2 - 2` in the code. (c) The top half occupies y from `topLeftY` down to `topLeftY - s + 1`; the bottom half starts at `topLeftY - s` and runs down to `topLeftY - 2s + 1`, so the bottom row is `topLeftY - 2s + 1`. (d) For `s = 4` the row lengths are 4, 6, 8, 10, 10, 8, 6, 4, totalling 56 tiles.

**4. Why two loops instead of one.**

*Q:* Rewriting the hardcoded six-line hexagon as loops, why is one loop awkward and two loops clean?

*A:* As a single pass the x offsets go -1, -2, -2, -1, 0 and the lengths go up then down, so neither is monotone in the loop variable and you need case analysis at the midpoint. Split at the midpoint: the top half is `size` lines where line `k` starts at `topLeftX - k`, sits at `topLeftY - k`, and has length `size + 2k`; the bottom half is `size` lines where line `k` starts at `startBottomHalfX + k`, sits at `startBottomHalfY - k`, and has length `startBottomHalfSize - 2k`. Each half is a simple arithmetic progression.

**5. Tessellation arithmetic.**

*Q:* In `HexWorldFinished`, why is the vertical step within a column `2 * HEX_SIZE` and the horizontal step between columns `2 * HEX_SIZE - 1`?

*A:* A hexagon of size `s` is exactly `2s` rows tall, so spacing stacked hexagons by `2s` makes them touch without overlapping. Horizontally, the interlocking of the diagonal edges means the next column's narrow top sits flush against this column's wide middle at an offset of `2s - 1` columns, combined with a vertical shift of `+s` (half a hexagon) to stagger them. The five calls use x offsets `k * (2*HEX_SIZE - 1)` for k = 0..4, written out with the multiplication distributed.

**6. The avatar trail bug.**

*Q:* A student writes a game loop that reads a key, updates `avatarX`, calls `StdDraw.clear()`, sets `world[avatarX][avatarY] = Tileset.AVATAR`, calls `ter.drawTiles(world)` and `StdDraw.show()`. Pressing `d` three times from (5, 9) leaves four `@` symbols on screen. Explain why `clear()` did not help, and give a fix that preserves the terrain.

*A:* `clear()` erases the drawing canvas, but the stale avatars are stored in the `world` array itself, and `drawTiles` immediately repaints them from the array. Fix: maintain a variable holding the tile that the avatar currently covers. On each move, write that saved tile back into the vacated square, then save the tile at the new square, then write `AVATAR` there. (A cruder fix is to reset the whole array to `NOTHING` and regenerate the world every frame, but that destroys any accumulated world state.)

**7. Input polling.**

*Q:* What is wrong with `char c = StdDraw.nextKeyTyped();` at the top of a game loop, and why does `GameLoopDemo` nest a `while` inside a `while`?

*A:* `nextKeyTyped()` crashes if no key is queued, so it must be guarded by `hasNextKeyTyped()`. The inner `while (hasNextKeyTyped())` drains all keys that arrived since the last frame so that fast typing is not dropped; the outer `while (true)` keeps the game running and renders one frame per iteration.

**8. `equals` versus `==` on tiles.**

*Q:* `TETile t = TETile.colorVariant(Tileset.WALL, 10, 10, 10, new Random(1)); world[0][0] = t;` What do `world[0][0] == Tileset.WALL` and `world[0][0].equals(Tileset.WALL)` evaluate to, and why?

*A:* `==` is `false`, because `colorVariant` constructed a brand-new `TETile` object at a different address. `.equals` is `true`, because `TETile.equals` compares only the `id` field, and the copy constructor carries the original's id (1 for `WALL`) through. This is why `GameLoopDemo.toggle` uses `.equals`.

**9. Seeded randomness.**

*Q:* In `RandomWorldDemo`, what would change if `randomTile()` were rewritten as `int tileNum = new Random().nextInt(3);`? What does the 3 mean?

*A:* The world would differ on every run, breaking reproducibility from a seed, and a fresh generator would be constructed per tile (seeded from the system clock, so nearby calls can be correlated and the output can show artifacts). The 3 is an exclusive upper bound on `nextInt`, yielding 0, 1, or 2 uniformly, not a seed; the seed is the argument to the `Random` constructor.

**10. Design-from-queries (the 4C lesson).**

*Q:* You must support three operations on a WordNet-style DAG: word to containing synset IDs, synset ID to immediate hyponym IDs, and synset ID to its words. Give instance variables that make all three constant-ish time, and say what goes wrong if you omit the third.

*A:* `Map<String, Set<Integer>> wordToIds`, `Map<Integer, List<Integer>> hyponyms` (an adjacency list), and `Map<Integer, List<String>> idToWords`. Omitting the third forces you to answer "which words are in this synset?" by scanning every node in the graph, which is what the "supergraph" design did: correct, but linear in the size of the whole graph per query instead of proportional to the answer.

**11. Why the course asks you to design and code it yourself.**

*Q:* Give three distinct reasons the lecture offered for writing your own code and design rather than delegating to an LLM.

*A:* Any three of: you must be able to read, verify, and steer generated code; decomposition is the real transferable skill and programming is the practice ground for it; large systems still need human-imposed structure; doing it yourself is intellectually satisfying and builds the habit of turning a problem into a working artifact. The lecture also acknowledged a cynical reason (the course infrastructure is already built this way) and the counterargument (we are legitimately moving to higher levels of abstraction).

---

## Summary

- **Part 1, reflection on 4C.** Derive your data structures from the queries you must answer. The clean design is three maps: word to synset IDs, synset ID to hyponym IDs (an adjacency list for a DAG), and synset ID to words. The messier "supergraph" design omitted the third and compensated with a full graph scan: it worked, and its author learned the cost of that choice firsthand.
- Clean designs rose from 22% to 80% across semesters. Confounders exist (evolving spec, mandatory design doc, better-trained staff, different class composition), but LLM-assisted design is suspected to be a large part of it. Extracting the design from the spec was the part worth doing.
- **For 4D:** do not submit code you cannot explain. A comparator lambda one-liner is the canonical tell.
- Reasons to still write your own code: comprehension and supervision, decomposition as the real skill, structure at scale, and the plain satisfaction of building. Reasons for honesty: it feels wrong for real reasons, it prepares you for exams, it shapes your self-image and your view of others, and ethics is mostly habit, since small violations lead to bigger ones.
- **Part 2, Project 5 (BYOW)** has three deliverables: a seeded random world generator, interactivity (walk around), and ambition features. Work starts in lab next week.
- The world is a `TETile[][]` indexed `world[x][y]` with **y = 0 at the bottom**. You mutate the array, then render it. Every slot must be non-null or rendering throws.
- `TETile` is immutable with equality by `id`; `Tileset` holds shared tile constants; `TERenderer` owns StdDraw setup, with `TILE_SIZE = 16`, double buffering, and `renderFrame = clear + drawTiles + show`.
- Game loop: `while (true)` with `hasNextKeyTyped()` guarding `nextKeyTyped()`, state update, render, and `StdDraw.show(30)` as a frame limiter. Drain the key queue with an inner `while` so keystrokes are not lost.
- **The avatar trail bug:** `StdDraw.clear()` clears the canvas, not the array. Lazy fix: reset the whole world each frame. Clean fix: keep `whatUsedToBeInMyNewSpot`, restore it into the vacated square, and save the destination tile before stamping the avatar.
- **Decomposition in practice:** `drawLine` first (verified by eye, which is how the fencepost bug `x < size` was caught in seconds), then a hardcoded size-3 hexagon, then generalize into a top-half loop plus a bottom-half loop, then tile the plane column by column (vertical step `2 * size`, horizontal step `2 * size - 1`, vertical stagger `+ size`) because that ordering has the simplest arithmetic.
- Seed your `Random` once and keep it, so the same seed always builds the same world, and remember that `nextInt(3)` is a bound, not a seed.
- **The meta-lesson:** build small primitives you can verify immediately, accept temporary mess to keep moving, and then periodically stop and refactor before the program becomes incomprehensible.
