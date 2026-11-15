<!-- Fri, Nov 13, 2026 | sources: slides (no transcript available) -->
# Lecture 33: Applications

This lecture steps outside the usual CS 61A sequence of language features and interpreters to ask a different question: how do you actually build software, and how does that change when a coding agent is doing much of the typing? There are two connected threads. The first is AI-assisted development: what a coding agent is (a large language model with access to tools like a terminal, a browser, and the ability to run programs), how to drive one well, and where human judgment still carries the weight. The second is software design: decomposition and encapsulation as the core compositional ideas, data modeling with a single source of truth, and types that make invalid states hard to represent. These threads meet in an extended live demo of building a web version of the Cucumber card game (from the Objects homework) with Cursor, decomposed into Cards, Games, Game Setup, and Game Table. The lecture's underlying argument is that an agent will fill in every detail you leave unspecified, so the parts of the job that stay human are deciding what to build, how the data is modeled, how the interface should feel, and whether the result is actually any good.

---

## Key Concepts

### What a coding agent is

Start from the model itself. A large language model is, in the lecture's framing, just a function `f(context)` that outputs text (and sometimes images or audio). Two properties matter:

1. **The output is randomized.** But randomized does not mean unpredictable in any useful sense. The lecture makes the point directly: randomized functions can be highly predictable. You already know this from sampling, flipping a fair coin a thousand times gives you a randomized result that is nonetheless tightly concentrated around 500 heads. An LLM that is right 95% of the time on a class of task is a randomized function you can plan around.
2. **The context and output often contain program text.** Source code is just text, so a function from text to text can read and write programs.

An **agent** is that function plus *tools*: a terminal to run commands and manipulate files, a browser and web search, and the ability to write, compile, and run programs. The tools are what change the character of the thing. The crucial consequence the lecture highlights: current LLMs are far more reliable and effective at software development when they can **check their work**, typically by running the programs they generate and observing or testing the output.

This is worth dwelling on, because it is the same principle you have been applying all semester in CS 61A. When you write a recursive function and then run it on a base case and a small input, you are not being pedantic, you are closing a feedback loop. An agent without tools is a student who writes code on paper and hands it in unexecuted. An agent with a terminal is a student who runs the doctests. The second one is dramatically better, and for the same reason.

### Expertise compounds, it does not become obsolete

The lecture cites a figure that is easy to skim past and shouldn't be: success rates of Claude Code sessions varied from **39% for novice users to 65% for expert users** (Oct 2025 to Apr 2026, from "Agentic coding and persistent returns to expertise," Anthropic, 2026).

Read what that says. The tool is the same in both cases. The gap is entirely in who is holding it. This is the empirical backbone of the whole lecture: human domain expertise appears to be very helpful at every step of AI-assisted development, and especially at the first one.

### Two different goals, two different workflows

The lecture is unusually explicit that *what you should do depends on what you want*, and it lays out two distinct processes.

**Goal A: minimize human effort to create useful functionality.**

- Step 1: Imagine what software you want.
- Step 2: Describe it and wait (go do something else).
- Step 3: Use it, or better yet, watch intended users use it, and describe changes.

The lecture's assessment of this loop is measured, not dismissive: this process works, has some issues (software can be complex, etc.), but will get better.

**Goal B: maximize your impact and capabilities by gaining domain expertise.**

This replaces and extends the middle of the loop:

- Step 2′: Describe it in full, but request each part **sequentially**.
- Step 2.5: Examine each part **critically**, interrogating the LLM to understand the program.
- Step 3.5: Make some changes **yourself** to see if you actually do understand the program.

Step 3.5 is the honest test, and it is the one students should notice. Reading generated code and nodding is not understanding. Successfully modifying it is evidence. The lecture also notes this path "often leads to improvement," meaning the sequential, interrogated process tends to produce a *better program*, not merely a better-educated programmer. Those two payoffs point the same direction.

If you are taking CS 61A, you are explicitly in the Goal B population. The point of the course is capability, and Step 3.5 is the version of that which survives contact with a coding agent.

### Decomposition and encapsulation

The design half of the lecture rests on one sentence: **software is compositional**. Two ideas implement that.

**Decomposition**: splitting a program into parts that can be **tested separately**. Note the criterion. It is not "parts that look tidy" or "parts that fit on one screen," it is parts you can test in isolation. Each part needs *types* to organize its data and *functions* to implement its behavior. The goal is that future changes to a program won't affect all the parts.

**Encapsulation**: each part hides its implementation behind a set of functions. Parts provide *abstractions*, and each part interacts with others only through those abstractions. The payoff the lecture names is specific and worth memorizing: encapsulation **limits the assumptions that each part must make about the rest of the program**.

That phrasing connects directly to abstraction barriers from earlier in CS 61A. When you built rational numbers with `numer`, `denom`, and `rational`, the reason not to reach in and index the pair directly was not aesthetics, it was that every direct index is an assumption about the representation, and every assumption is a thing that breaks when the representation changes.

The lecture gives three operational instructions:

- Think about the parts and what they assume about each other.
- Build and understand **one part at a time**.
- **Reuse** existing software components when possible.

The middle one rhymes with Step 2′ from the AI workflow. Sequential, part-by-part construction is good practice both because it is good design and because it is how you keep understanding what an agent is producing. The two halves of this lecture are the same advice arriving from two directions.

### Data modeling and the single source of truth

**Data modeling** is the process of deciding how your program will represent information. Three principles.

**Single Source of Truth**: store each aspect of the state of your program in **one place**. The lecture's example is sharp: if a list and the length of a list are both stored, which one is right? The moment the same fact lives in two places, they can disagree, and then your program has a state that means nothing.

The deeper form of the principle: **store what happened rather than derived consequences**, which makes programs easier to test. In Cucumber, store the sequence of cards played. Do not separately store "who is winning," because that is a *consequence* of the plays. You compute it. If you store it, you now have two things to keep in sync, and a bug is a disagreement rather than a wrong answer.

And the practical release valve: once the source of truth is established, it's fine to add **caching** to speed up the program. This matters, because the principle otherwise sounds like it forbids memoization, and it does not. The distinction is in the ordering. A cache is a derived value you have deliberately decided to store *after* establishing which value is authoritative, with a rule for invalidating it. Two independent stores of the same fact with no authority between them is the thing to avoid. (Compare the memoization you have already seen in CS 61A: the dictionary of computed Fibonacci values is a cache, and the recursive definition remains the source of truth.)

**Avoid invalid data**: use a type system to describe what states are allowed. Ideally, your types **can only represent valid states**. Use a small set of functions to create or update data, functions which ensure validity. This is the constructor/selector discipline again. If every card in the program comes from one function that builds cards, that function is a single choke point where "is this a legal card?" gets answered once.

**Structure data into types that functions will consume or return.** Each function should take **only the input it needs**, not the whole state. A function that takes the whole game state can read and depend on anything, so it assumes everything, so it breaks on any change. A function that takes a hand and a card assumes a hand and a card.

### AI-assisted user interfaces, where agents are weak

The lecture's most actionable warning. The coding agent will fill in **all the details you don't specify** about a program's design. That is not optional behavior you can switch off, it is what the thing does. Everything you leave vague gets decided by something other than you.

John's reported experience, from the slides:

- AI uses common approaches (which can be good), but AI-generated apps **look and feel similar**.
- Coding agents are **much weaker at assessing usability than at creating functionality**.

Notice the asymmetry: *creating* versus *assessing*. An agent can build a card-flip animation. Judging whether the animation makes the game clearer or more confusing is a different skill, and the one it is worse at. So the recommendations follow directly: spend time thinking about the UI and describe it in detail, including diagrams. Also, **use the software**. If you are not among the target audience, find someone who is and have them use it for you. (And take courses or read books and tutorials on human-computer interaction.)

### How to prompt an agent: ask, do not dictate

Three tips, each built on the same insight.

**Tip 1: Telling the agent how to do something can work against you and degrade your program.**

- Worse: "speed this up using memoization"
- Better: "what are ways to speed this up? Memoization? What are the consequences?"

When you specify the mechanism, you have pre-committed to a solution, and the agent will dutifully implement it even when it is wrong for your case. Memoization on a function with mutable arguments or an unbounded key space is a memory leak wearing a performance costume. The better prompt keeps the decision with you and gets the agent to supply the information you need to make it.

**Tip 2: The agent can build many alternatives quickly.** Use that.

- Worse: "animate the cards flipping over"
- Better: "show me some options for animating card flips"

The cost of generating three variants is near the cost of one. Asking for one and accepting it discards the main advantage you have. This is also the counter to "AI-generated apps all look the same": generating options and *choosing* is where your taste re-enters the loop.

**Tip 3: If you don't know why it did something, ask.** An unexplained line of code in your program is a line you cannot maintain. The agent is sitting right there and will explain it.

And the summary line: if you want to learn (and build useful software), build components and scrutinize them. Same instruction as Step 2′ and Step 2.5, same instruction as "build and understand one part at a time."

### Software and society

Framing, and the course clearly intends it to be taken seriously.

- Useful software improves human life, directly or indirectly.
- Humans (currently) own, control, and decide everything.
- AI is (so far) being used to **augment or automate human work, not make strategic decisions**.

Three implications for people building software for others, each stated as a cause and an effect:

- People are **accountable** for the software they produce, so spend effort on **reliability**.
- Software is adopted when people **decide** to use it, so expect to **communicate a lot**.
- **User experience and real-world usefulness** matter, so **involve people in the design**.

The accountability point is the one that pushes back hardest on careless agent use. "The model wrote it" is not a defense available to you. You shipped it.

### Announcements context: why write a Gleam interpreter in Gleam?

The announcements included the Animator Project and a note on the Gleam interpreter, which is a nice echo of the interpreter work from earlier lectures. Reasons given for building an interpreter for Gleam *in* Gleam:

- The interpreter compiles to **JavaScript** and can therefore run in the browser (the Gleam compiler itself is written in Rust, so it cannot).
- You get to **control how Gleam code is run**: limiting list sizes, the number of evaluation steps, recursion depth.

Both reasons should feel familiar. This is exactly the argument for writing your own interpreter at all: when you control the evaluator, you control the semantics, and you can impose limits the host language will not give you. Writing it in a language that targets the browser is a deployment decision, writing your own evaluator is a power decision.

---

## Definitions

**Large language model (LLM)**: a function `f(context)` that outputs text (and images or audio). Its output is randomized, though randomized functions can be highly predictable. Both its context and its output often include the text of programming languages.

**Agent**: an LLM that has access to computer-based tools, such as a terminal to run commands (e.g., manipulate files), a browser and web search, and the ability to write, compile, and run programs.

**Decomposition**: splitting a program into parts that can be tested separately. Each part needs types to organize its data and functions to implement its behavior. Ideally, future changes to a program won't affect all the parts.

**Encapsulation**: designing each part to hide its implementation behind a set of functions, so that parts provide abstractions and interact with one another only through those abstractions, which limits the assumptions each part must make about the rest of the program.

**Data modeling**: the process of deciding how your program will represent information.

**Single Source of Truth**: the principle of storing each aspect of the state of your program in exactly one place, preferring to store what happened rather than derived consequences.

**Caching**: storing a derived value for speed, acceptable once the source of truth has been established.

**Round (also: trick)**, in Cucumber: one cycle in which each player plays exactly one card.

**Taking control**, in Cucumber: playing a card at least as high as the highest played so far in the current round (Aces high, suits don't matter).

**Cucumber (the game)**: each player is dealt 5 cards to play 5 rounds. In a round, each player plays one card, starting with the winner of the previous round. A card at least as high as the highest so far takes control. The last player to take control wins the round and starts the next round. The goal is **not** to win the last round, you get 1 point (bad) each time you win the last round.

---

## Worked Examples

### Example 1: Cucumber's rules as code, and why "at least as high" matters

The lecture's rules, stated precisely: a card **at least as high** as the highest so far takes control, Aces high, suits don't matter, and the **last** player to take control wins the round.

The "at least as high" is doing real work, and it is the kind of detail an agent will guess at if you do not specify it. Here is the rule as a function:

```python
def takes_control(card_rank, highest_so_far):
    return highest_so_far is None or card_rank >= highest_so_far
```

Step by step on a four-player round where the played ranks are `[9, 11, 11, 4]`:

1. `highest_so_far` is `None`, player 0 plays 9. `None` means nothing has been played, so player 0 takes control. `highest_so_far` becomes 9, controller is player 0.
2. Player 1 plays 11. `11 >= 9` is true, so player 1 takes control. `highest_so_far` becomes 11, controller is player 1.
3. Player 2 plays 11. `11 >= 11` is true. This is the case the `>=` decides: with a strict `>` player 2 would not take control, and with `>=` they do. Per the lecture's rule, player 2 takes control. Controller is player 2.
4. Player 3 plays 4. `4 >= 11` is false. No change. Controller stays player 2.

Player 2 wins the round, because they were the **last** to take control. If this were the fifth round, player 2 gets 1 point, which is bad.

Why this is a good illustration of the lecture's point about specification: the difference between `>` and `>=` here is one character and it silently changes who wins. An agent told only "make a card game where high cards win" will pick one. You have to say which.

### Example 2: Determining the round winner from the source of truth

Now apply Single Source of Truth. The lecture says the Games part "tracks who played which card, and in what order," and separately "determines which plays are allowed and who won." Those are deliberately separate: one is *stored*, one is *computed*.

```python
def round_winner(plays):
    controller = None
    highest = None
    for player, rank in plays:
        if highest is None or rank >= highest:
            controller = player
            highest = rank
    return controller
```

`plays` is the source of truth: a list of `(player, rank)` pairs in the order played. That is "what happened." The winner is a derived consequence, so it is a function, not a stored field.

Tracing `round_winner([(0, 9), (1, 11), (2, 11), (3, 4)])`:

| Iteration | `player`, `rank` | Test | `controller` after | `highest` after |
|---|---|---|---|---|
| start | - | - | `None` | `None` |
| 1 | 0, 9 | `highest is None` → True | 0 | 9 |
| 2 | 1, 11 | `11 >= 9` → True | 1 | 11 |
| 3 | 2, 11 | `11 >= 11` → True | 2 | 11 |
| 4 | 3, 4 | `4 >= 11` → False | 2 | 11 |

Returns `2`.

**Environment reasoning in words**: calling `round_winner` opens a new frame whose parent is the global frame. In that frame, `plays` is bound to the *same list object* the caller holds, no copy is made, so the frame's `plays` and the caller's list are two names pointing at one list in the heap. The `for` loop rebinds `player` and `rank` in this frame on each pass. Critically, nothing in this function mutates `plays`, so the caller's record of what happened is untouched. That is exactly the property Single Source of Truth is protecting: the function *reads* the truth and *returns* a consequence. When the frame is discarded on return, `controller` and `highest` vanish, and only the returned `2` survives. If instead we had stored the winner into the game state, there would now be two places recording who won, the list of plays and the stored field, and any bug in the update logic would make them disagree.

### Example 3: Contrasting a bad data model with a good one

The lecture's own warning case is a list stored alongside its length. Here is that failure in a form closer to Cucumber.

**Bad (two sources of truth):**

```python
class BadRound:
    def __init__(self):
        self.plays = []
        self.highest = None
        self.controller = None

    def play(self, player, rank):
        self.plays.append((player, rank))
        if self.highest is None or rank > self.highest:
            self.controller = player
            self.highest = rank
```

The bug is live: that is `>` instead of `>=`, so on `[(0, 11), (1, 11)]` this object stores `controller = 0`, while `round_winner(self.plays)` says `1`. The object now contains two mutually contradictory answers about who won, and nothing in the program marks which is authoritative. Worse, to test this you need to construct an object, call `play` in the right order, and inspect internal attributes. The bug is reachable only through a sequence of mutations.

**Good (single source of truth, derived values computed):**

```python
class Round:
    def __init__(self):
        self.plays = []

    def play(self, player, rank):
        assert self.allows(rank), 'illegal play'
        self.plays.append((player, rank))

    def highest(self):
        return max((rank for _, rank in self.plays), default=None)

    def controller(self):
        return round_winner(self.plays)

    def allows(self, rank):
        return True
```

Now `plays` is the only stored state. `highest` and `controller` are consequences, computed on demand. The correctness of `controller` lives entirely in `round_winner`, a plain function over a plain list, which is why the lecture says storing what happened "makes programs easier to test": you test `round_winner([(0, 11), (1, 11)]) == 1` with no object construction and no mutation sequence at all.

This also satisfies the lecture's "small set of functions to create or update data, which ensure validity": `play` is the single update path, and it is where the `assert` lives. There is no way to append to `plays` through the intended interface without passing the check. (Python does not enforce this the way a stricter type system would, which is the gap the lecture gestures at with "ideally, your types can only represent valid states.")

And note `allows(self, rank)`: it takes a rank, not the whole game. That is "each function should take only the input it needs, not the whole state."

### Example 4: Decomposing Cucumber as the lecture does

The slides lay out four parts with their responsibilities and the relationships between them. Reconstructing the dependency structure (the slide's diagram is missing, but the arrows are labeled "Deal," "Create," and "Display," and the role of each is clear from the text):

**Cards**
- Cards can be shuffled.
- Hands can be dealt.

**Games**
- Tracks who played which card, and in what order.
- Determines which plays are allowed and who won.

**Game Setup**
- Choose number of players and which bots.
- Setup appears only when a game is done.

**Game Table**
- Each card has a position and flip.
- Clicking a card in your hand plays it.

The labeled relationships: Cards **deals** into Games, Game Setup **creates** a Game, and Games **displays** through the Game Table.

Why this decomposition is good by the lecture's own criteria:

- **Each part is separately testable.** You can test shuffling and dealing (Cards) with no notion of rounds. You can test "who won" (Games) by handing it a list of plays, with no UI and no browser, as in Example 2. You can test the Game Table's layout by giving it a position and a flip state for each card, with no rules engine.
- **Assumptions are minimal and one-directional.** Games assumes Cards can produce hands. Games does not know that cards have screen positions or that clicking plays them, that is the Game Table's business. The Game Table assumes it is handed cards with a position and a flip state, and does not know the control rules. The rules could change from "at least as high" to "strictly higher" without touching a line of rendering code.
- **The source of truth is in Games**, specifically "who played which card, and in what order." The Game Table's positions and flips are *presentation*, derived from that. Which card is face-up is a consequence of game state, not an independent fact to be tracked separately.

Here is the Cards part in isolation, small enough to see the whole abstraction:

```python
import random

RANKS = list(range(2, 15))
SUITS = ['S', 'H', 'D', 'C']

def deck():
    return [(rank, suit) for rank in RANKS for suit in SUITS]

def shuffled(cards):
    result = list(cards)
    random.shuffle(result)
    return result

def deal(cards, num_players, hand_size=5):
    assert num_players * hand_size <= len(cards), 'not enough cards'
    return [cards[i * hand_size:(i + 1) * hand_size] for i in range(num_players)]
```

Reading this step by step:

1. `RANKS` is `range(2, 15)` as a list, so 2 through 14. The 14 is Aces high: there is no separate Ace case anywhere in the program, because the representation already encodes the rule. An Ace *is* 14. This is the lecture's "ideally, your types can only represent valid states" applied in miniature, there is no way to construct a card whose Ace compares incorrectly, because comparison is just integer comparison.
2. `deck()` is a list comprehension over two loops, producing all 52 rank-suit pairs. Suits are carried because real cards have them, but no rules function will ever read the suit, since suits don't matter in Cucumber.
3. `shuffled` deliberately copies with `list(cards)` before shuffling. `random.shuffle` mutates in place, so without the copy, calling `shuffled(d)` would silently reorder the caller's `d`. **Box-and-pointer reasoning**: `cards` and the caller's list point at one object, `list(cards)` allocates a *new* list object with the same element pointers, and `random.shuffle` then permutes only the new list's slots. The caller's list is untouched, and the card tuples themselves are shared by both lists, which is harmless because tuples are immutable.
4. `deal` slices the shuffled deck into `num_players` consecutive chunks of `hand_size`. For 3 players at 5 cards it returns indices 0-4, 5-9, 10-14. Slicing creates new lists, so the hands are independent lists, and the per-hand mutation that happens when a player plays a card cannot corrupt another hand.
5. The `assert` is the validity check at the single creation point, matching the lecture's "use a small set of functions to create or update data, which ensure validity."

Testing this part needs nothing else in the program:

```python
def test_cards():
    d = deck()
    assert len(d) == 52
    assert len(set(d)) == 52
    original = deck()
    s = shuffled(original)
    assert original == deck(), 'shuffled must not mutate its argument'
    assert sorted(s) == sorted(original)
    hands = deal(s, 3)
    assert [len(h) for h in hands] == [5, 5, 5]
    flat = [c for h in hands for c in h]
    assert len(set(flat)) == 15, 'no card dealt twice'
```

That is the concrete meaning of "parts that can be tested separately." No Games, no Game Table, no browser.

### Example 5: The prompting tips applied to this code

The lecture's Tip 1 contrast, made concrete against the code above. Suppose dealing is slow because `deck()` is rebuilt constantly.

- **Worse prompt**: "speed this up using memoization." The agent adds `@lru_cache` to `deck()`. Now every caller of `deck()` receives *the same list object*, and the first caller that mutates it (say, by shuffling in place) corrupts the deck for everyone. You asked for a mechanism, got the mechanism, and inherited a bug that the mechanism caused.
- **Better prompt**: "what are ways to speed this up? Memoization? What are the consequences?" The answer includes the aliasing hazard above, and you learn that caching a mutable return value is only safe if callers never mutate it, which is exactly why `shuffled` copies. Now *you* decide: cache a frozen tuple, or leave it alone because 52 tuples is not a real cost.

That is the whole tip in one example. The mechanism was not wrong in the abstract. Committing to it before understanding the consequences was.

---

## Common Pitfalls

**Thinking "randomized output" means "unreliable."** The lecture explicitly heads this off: randomized functions can be highly predictable. The practical mistake in both directions is treating the randomness as either disqualifying or irrelevant, instead of as a property with a known distribution you plan around.

**Giving an agent tools but no way to check its work.** The lecture is clear that current LLMs are much more reliable when they can run programs and observe the output. A workflow where the agent writes code that is never executed throws away the main thing that makes agents effective.

**Prescribing the mechanism instead of asking about options.** "Speed this up using memoization" versus "what are ways to speed this up, and what are the consequences?" Once you name the technique, you have made the decision, and the agent will carry it out whether or not it fits.

**Accepting the first version when alternatives are nearly free.** The agent can build many alternatives quickly. "Animate the cards flipping over" gets one animation. "Show me some options for animating card flips" gets a choice. Taking the first version is both a worse result and the direct cause of AI-generated apps looking and feeling the same.

**Leaving UI details unspecified.** The coding agent will fill in all the details you don't specify. Vagueness is not neutrality, it is delegation. And since coding agents are much weaker at assessing usability than at creating functionality, this is precisely the area where delegating by default goes worst.

**Confusing "it works" with "it is usable."** Related but distinct. Functionality can be complete while the interface is bad, and the agent is not well positioned to tell you so. The remedy in the slides is to use the software, and if you are not in the target audience, find someone who is.

**Reading generated code and calling that understanding.** The lecture's Step 3.5 exists because of this: make some changes yourself to see if you actually do understand the program. Recognition is not comprehension.

**Letting unexplained code stay unexplained.** If you don't know why it did something, ask. An unexamined line is a line you cannot debug later.

**Storing a value and its derived consequences as peers.** The list-and-its-length case. Once both exist with no designated authority, they can disagree, and the program has states that mean nothing. Store what happened, compute the rest.

**Treating "no caching" as the rule.** It is not. The rule is *establish the source of truth first*, then caching is fine. Students sometimes over-apply the principle and avoid memoization they have already been taught to use.

**Passing the whole program state into every function.** Each function should take only the input it needs. A function handed everything may depend on anything, which maximizes exactly the assumptions encapsulation is meant to minimize.

**Decomposing by file size rather than by testability.** The lecture's definition of decomposition is parts that can be tested separately. Splitting a long module into three shorter ones that all reach into each other's internals is not decomposition.

**Assuming AI makes design and judgment obsolete.** The 39% versus 65% novice-expert gap says the opposite. The tool rewards expertise rather than replacing it.

---

## Likely Exam Points

CS 61A exams do test conceptual lecture material, and this lecture is unusually definition-dense and principle-dense, which is the kind of content that appears as short-answer or multiple-choice.

**1. The definition of an agent versus an LLM**

*Q: According to lecture, what distinguishes an "agent" from a large language model, and why does that distinction improve reliability on software tasks?*

*A:* An LLM is a function `f(context)` that outputs text. An agent is an LLM that additionally has access to computer-based tools: a terminal to run commands, a browser and web search, and the ability to write, compile, and run programs. Reliability improves because current LLMs are much more effective at software development when they can **check their work**, typically by running the programs they generate and observing or testing the output.

**2. Single Source of Truth**

*Q: A game program stores both `self.plays` (the list of cards played in order) and `self.winner` (the player who currently has control). Identify the design problem and state the fix in terms of a lecture principle.*

*A:* This violates Single Source of Truth: the same fact (who has control) is recorded in two places, so they can disagree, and nothing designates which is right. The principle is to store each aspect of state in one place, storing what happened rather than derived consequences. Keep `plays` as the source of truth and compute the winner from it with a function. Caching the winner is acceptable only after the source of truth is established, with a rule for keeping the cache valid.

**3. Decomposition versus encapsulation**

*Q: Define decomposition and encapsulation, and give the specific benefit lecture attributes to encapsulation.*

*A:* Decomposition is splitting a program into parts that can be tested separately, where each part needs types to organize its data and functions to implement its behavior. Encapsulation is designing each part to hide its implementation behind a set of functions, so parts provide abstractions and interact only through them. The benefit lecture names: encapsulation **limits the assumptions that each part must make about the rest of the program**.

**4. The Cucumber control rule**

*Q: Four players play ranks 7, 10, 10, 2 in that order (players 0, 1, 2, 3) in the final round. Who wins the round and what is the score consequence? Aces high, suits don't matter.*

*A:* Player 0 takes control at 7 (first play). Player 1 takes control at 10, since 10 is at least as high as 7. Player 2 **also** takes control at 10, since the rule is "at least as high," not "strictly higher." Player 3's 2 does not take control. The **last** player to take control wins, so player 2 wins the round. Because this is the last round, player 2 receives 1 point, which is bad, since the goal of Cucumber is not to win the last round.

**5. Prompting: specify the goal, not the mechanism**

*Q: Rewrite the prompt "speed this up using memoization" per the lecture's guidance, and explain why the rewrite is better.*

*A:* "What are ways to speed this up? Memoization? What are the consequences?" Telling the agent *how* to do something can work against you and degrade the program, because you have committed to a mechanism before knowing whether it fits. The better prompt keeps the decision with you and surfaces the tradeoffs needed to make it.

**6. The two workflows, and which steps are added for learning**

*Q: What three steps does lecture add or substitute when your goal is to gain domain expertise rather than minimize effort?*

*A:* Step 2′: describe the software in full but request each part **sequentially**. Step 2.5: examine each part **critically**, interrogating the LLM to understand the program. Step 3.5: make some changes **yourself** to see if you actually do understand the program. Lecture also notes this sequential path often leads to improvement in the program itself, not only in understanding.

**7. Where coding agents are weakest**

*Q: Lecture claims coding agents are much weaker at one aspect of UI work than another. Which, and what two remedies does it recommend?*

*A:* They are much weaker at **assessing usability** than at **creating functionality**. Remedies: spend time thinking about the UI and describe it in detail (including diagrams), since the agent fills in every detail you don't specify, and actually **use the software**, finding someone in the target audience to use it if you are not in that audience yourself.

**8. Avoiding invalid data**

*Q: What two techniques does lecture give for avoiding invalid data, and how does representing Aces as the integer 14 illustrate one of them?*

*A:* Use a type system to describe allowed states, ideally so that types can only represent valid states, and use a small set of functions to create or update data, functions which ensure validity. Representing an Ace as 14 means "Aces high" is built into the representation, so no comparison anywhere in the program can get Ace ordering wrong, and there is no invalid Ace state to guard against.

**9. Function inputs**

*Q: Why does lecture advise that each function take only the input it needs rather than the whole program state?*

*A:* A function given the whole state may read and depend on any part of it, so it makes assumptions about the entire program. That is precisely what encapsulation is meant to limit, and it means future changes anywhere in the state can break the function. Taking only what it needs confines its assumptions to that input.

**10. Expertise and AI**

*Q: Lecture cites Claude Code session success rates of 39% and 65%. What do those numbers compare, and what conclusion does lecture draw?*

*A:* They compare **novice users (39%)** with **expert users (65%)** over Oct 2025 to Apr 2026. The conclusion: human domain expertise appears to be very helpful at all steps of AI-assisted development, especially Step 1 (imagining what software you want). The tool rewards expertise rather than eliminating the need for it.

**11. Software and society implications**

*Q: State the three implications lecture gives for people building software for others, each with its stated cause.*

*A:* People are accountable for the software they produce, so spend effort on **reliability**. Software is adopted when people decide to use it, so expect to **communicate a lot**. User experience and real-world usefulness matter, so **involve people in the design**.

**12. Why write a Gleam interpreter in Gleam**

*Q: Lecture gives two reasons to build a Gleam interpreter in Gleam. What are they?*

*A:* First, the interpreter is JavaScript and so can run in the browser, whereas the Gleam compiler is written in Rust and cannot. Second, it lets you control how Gleam code is run, for example by limiting list sizes, the number of evaluation steps, or recursion depth.

---

## Summary

- An **LLM** is a function `f(context)` producing text; the output is randomized, but randomized functions can be highly predictable, and the context and output often contain program text.
- An **agent** is an LLM with tools: a terminal, a browser and web search, and the ability to write, compile, and run programs. Agents are much more reliable at software development when they can **check their work** by running programs and observing output.
- **Minimum-effort workflow**: imagine the software, describe it and wait, then use it (or watch intended users use it) and describe changes. It works, has issues, and will get better.
- **Expertise-building workflow** substitutes: describe it in full but request each part **sequentially** (2′), examine each part critically and interrogate the LLM (2.5), and **change it yourself** to verify you understand it (3.5). This often improves the program too.
- Human **domain expertise** helps at every step, especially Step 1. Claude Code session success rates ran **39% for novices versus 65% for experts** (Oct 2025 to Apr 2026).
- **Decomposition**: split a program into parts that can be **tested separately**, each with types for its data and functions for its behavior, so future changes don't affect all parts.
- **Encapsulation**: hide each part's implementation behind functions, so parts interact only through abstractions, which **limits the assumptions each part makes about the rest of the program**.
- Practical design habits: think about the parts and their mutual assumptions, build and understand **one part at a time**, and **reuse** existing components.
- **Data modeling** is deciding how the program represents information.
- **Single Source of Truth**: store each aspect of state in one place. Store **what happened**, not derived consequences, which makes programs easier to test. Caching for speed is fine **after** the source of truth is established.
- **Avoid invalid data**: use types so that ideally only valid states are representable, and funnel creation and updates through a small set of validity-ensuring functions.
- Give each function **only the input it needs**, not the whole state.
- **Cucumber**: 5 cards, 5 rounds; each round every player plays one card starting with the previous round's winner; a card **at least as high** as the highest so far takes control (Aces high, suits irrelevant); the **last** to take control wins the round; **winning the last round costs you 1 point**, which is bad.
- Cucumber decomposes into **Cards** (shuffle, deal), **Games** (who played what in what order, legal plays, who won), **Game Setup** (number of players, which bots, shown only when a game is done), and **Game Table** (card position and flip, clicking a card in your hand plays it), with Cards dealing into Games, Setup creating a Game, and Games displaying through the Table.
- Agents **fill in every detail you don't specify**. They are much **weaker at assessing usability** than at creating functionality, so describe the UI in detail (with diagrams) and actually use the software, or find a target user who will.
- Prompting: **ask about options, don't dictate mechanisms** ("what are ways to speed this up?" over "use memoization"); request **alternatives** ("show me options for animating card flips"); and if you don't know why it did something, **ask**.
- Society: useful software improves human life; humans currently own, control, and decide everything; AI so far augments or automates work rather than making strategic decisions. So: be accountable and invest in **reliability**, expect to **communicate** a lot to drive adoption, and **involve people** in design.
- To learn while building useful software: **build components and scrutinize them**.
