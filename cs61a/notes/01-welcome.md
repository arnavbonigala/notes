<!-- Wed, Aug 26, 2026 | sources: slides + your recording -->
# Lecture 1: Welcome

## Overview

The first lecture of CS 61A is roughly two-thirds course logistics and one-third actual computer science. On the logistics side: CS 61A is a course about **managing complexity** through abstraction, problem solving, and techniques for organizing complex programs, taught in three languages (Python, Gleam, and SQL); the course runs on a "videos first, then lecture" model where the posted videos and the textbook (composingprograms.com) carry the complete content and lecture covers the highlights, examples, and perspective; learning happens through practice in required labs and discussions, weekly homework, four projects, and quizzes that are just homework problems redone; and AI use is tightly restricted for the first 11 weeks (only the course's own Preceptor tool, which points at problems rather than giving answers) before the final project introduces AI-assisted programming deliberately. On the technical side, the lecture introduced the vocabulary and mechanics that the rest of the semester builds on: an **expression** describes a computation and **evaluates to a value**; every expression in Python can be written as a **call expression**; and a call expression is evaluated by a three-step procedure (evaluate the operator, evaluate each operand, apply the resulting function to the resulting arguments) that is *recursive*, meaning the same procedure applies to subexpressions, which is exactly what lets you evaluate arbitrarily nested expressions like `mul(add(4, mul(4, 6)), add(3, 5))` by building an **expression tree**. The lecture closed with a demo of the end-of-semester project: a program that draws generative art, extended by DeNero into a spirograph that can trace arbitrary curves (including the Cal logo) out of spinning circles.

---

## Key Concepts

### 1. What CS 61A is actually about

The headline claim from the slides is that CS 61A is **a course about managing complexity**, broken into three pieces:

- **Mastering abstraction.** Building pieces you can use without re-deriving how they work inside.
- **Solving problems.** A skill, developed by doing, not by watching.
- **Techniques for organizing complex programs.** How to structure code so that a big program stays comprehensible.

It is *also* an introduction to programming: programming language fundamentals in Python, large projects that exist specifically to demonstrate complexity management, and how computers interpret programming languages.

### 2. Why three languages

The course uses **Python**, **Gleam**, and **SQL**. The motivating idea from lecture: in many domains you can get where you're going in almost any language, but in computer science a lot of problems have a *right* way to be solved, and different languages are built for different kinds of problems. Recognizing which language fits which problem is presented as fundamental, not trivia.

Answers given to student questions in lecture:
- **SQL** is a way to interact with **databases**. Databases store the information about the world: the example given was credit card history. You may never have seen someone write SQL because it usually sits behind other software, not in front of users.
- **Gleam** is a **functional programming language**, and it is how the course covers functional programming.
- Gleam appears late: the instructor's answer to "how much do we use it" was essentially that you learn it shortly before you use it ("just a week before we learn it").

### 3. The AI stance, and the reasoning behind it

This is worth understanding as an argument, not just as a rule, because the rule follows from the argument.

The premise the instructor grants up front: **AI has genuinely changed how software is built.** Many working software builders are no longer typing out every line; they work with AI assistants. He includes himself in that group.

The conclusion he draws is not "so don't bother learning to program." It's the opposite. From talking to those practitioners (and from being one): **understanding programs and being able to program yourself gives you a large advantage when using AI**, compared with not understanding what's going on. So the best way to learn to use AI well is to learn to program.

The mechanism by which overuse hurts you, stated explicitly:
- If you ask AI every time you get stuck, **you never learn how to get yourself unstuck.**
- **You never learn why the wrong answers don't work**, which is a large part of understanding.
- You don't develop the skill of **understanding what is happening inside a program.**

And a framing he was direct about: the prohibition isn't because he cares about the programs you turn in (though he does), it's because he cares about **what you're learning**. The only reliable path anyone has found to the expertise that makes AI use effective is solving the problems yourself.

He also offered a rough split for the skill of using AI: roughly **90% is understanding what you're trying to build, how you want it to be, and describing it**, and only about **10% is learning the tools**. That 90% is exactly what the course teaches. This connects to a separate point made about discussion sections: in the age of AI, *being able to describe things* is more valuable than ever, because you describe what you're doing both to people and to language models.

**The two sanctioned uses of AI in the course:**

| Phase | Policy |
|---|---|
| First 11 weeks | You write programs on your own. One course AI tool (**Preceptor**) is available if you get stuck. It tells you *where* your problem is; it does not hand you the answer. |
| End of semester | Explicit unit on AI-assisted programming, applied to the last project. |

Do **not** use ChatGPT, Gemini, Copilot, or similar. Practical reason given beyond the policy: they know all the problems in this course, because past students posted them on the web, so they will simply tell you the answer.

### 4. Videos, textbook, lecture: the concert analogy

The structural claim: **videos posted to cs61a.org are essential viewing *before* lecture, and all course content is covered in the videos.** The textbook, composingprograms.com, is written to be concise and useful, and its content is very similar to the videos. Both are made by DeNero, so pick either, or do both.

**Lecture is not comprehensive.** It covers the most important content from the videos (but not all of it), works through examples, discusses problem-solving strategies, and gives perspective.

The analogy from lecture: **the videos are the album, lecture is the concert (at the Greek Theatre).** You listen to the album first so you know all the songs; then you arrive at the concert ready to actually enjoy it. Show up to lecture having not watched, and you're at a concert for a band you've never heard.

Logistics mentioned: about **25 minutes of video before each lecture**, split into three or four videos in a playlist. Recommended viewing technique, from both slides and lecture: **type the examples out yourself as you watch.** Past-student advice quoted on the slides warns specifically against 2x speed ("if you don't take time processing information, it will leave your brain by next morning").

### 5. Why practice is required, not optional

The lecture's framing: you cannot learn computer science by coming to lecture and listening, any more than **you learn to play the piano by listening to someone else play it**. It's enjoyable, but it isn't learning. Problem solving becomes easier only with practice, and finding it hard at first is normal.

That principle is what the course structure implements:

- **Lab, Mon/Tue**: attendance required (unless you are in mega lab).
- **Discussion, Wed/Thu/Fri**: attendance required (unless you are in mega discussion). Starts this week.
- These prepare you for **weekly homework** and **four larger programming projects**.
- **Quizzes during lab**: you redo a homework problem with slight variations.
- **Drop-in one-on-one help** (called "office hours" at Cal): starts next week, about **eight hours a week in the new Gateway building**, with tutors and TAs, in addition to the instructors' own office hours.

The quiz rationale, stated bluntly: some people have AI do their homework and learn nothing. If you do that, you fail the quiz. If you do the homework yourself, you pass, because the quiz is just a slight variant of a problem you already solved. **Quizzes are low-stress conditional on actually doing your homework.**

### 6. What discussion section is (and isn't)

The slides contrast "Expectation" with "Reality" via two images (missing from the PDF extraction, but the point survives in the transcript): a discussion is *not* a bunch of people watching a presentation. That would be a lecture. In 61A, discussion means getting into groups and discussing.

Mechanics:
- You have a **group number**, with about **5 students** in your group, plus a room and a time. **Multiple groups share a room**, so you have to find yours.
- You get an **online worksheet** of problems, already posted on the course site, and work them together.
- **Discussion problems aren't graded** and you don't have to solve them all. The goal is to get better at solving similar problems over the 80 minutes.
- **"If you solve them all but don't talk to anyone, you've missed out."** Talking is the point.
- Bring a laptop or tablet; a phone or paper works in a pinch.
- A lecture-style walkthrough option exists if the group format truly doesn't work for you, but try the discussion format first.
- **Mega discussion** means you have opted out and just read the document on your own. You can switch into a real discussion via the "add or change discussions" link in the announcements at the top of cs61a.org. If there's no space, post on Ed and more space will be made.
- Missing discussion week 1 is not a crisis, but go, because that's how you meet your group.

### 7. Lab 0, the toolchain, and Provenance

**Lab 0** is a take-home assignment (released the night of this lecture) whose entire job is to set up your computer. **Everyone uses the same setup**, deliberately, so the whole class can talk to each other about the same configuration and stay a community. Drop-in help is available if you get stuck.

What gets installed:
- **Python**
- **Visual Studio Code** (a Microsoft program; the window title bar says "code")
- Two course-specific VS Code extensions:
  - **Preceptor**: the AI helper. It gives suggestions when your code is wrong and checks your understanding. It does not give answers. You can turn it off if you prefer.
  - **Provenance**: tracks *how* you completed the assignment.

The 10-second VS Code orientation given in lecture: there are **files** (one of which you fill out), and there is a **terminal**. A terminal is where you write commands to tell your computer to do things: it controls your computer the way a mouse does, but with text. It's where you run your programs.

**Provenance, explained carefully**, because the instructor anticipated the privacy objection ("this might seem like a terrible privacy violation, maybe it is"):
- It logs how you produced your file: **what you typed vs. what you pasted**, and whether you switched out of the app and back.
- **Nothing is sent from your computer to the internet.** The log is stored locally and *you* decide whether to submit it.
- It only tracks changes to the files in that assignment's directory. Activity elsewhere on your computer, or in other directories, is not tracked.
- If you don't want to submit a particular log, you can re-download the assignment and redo it, and submit that version instead. You have control.
- The rationale: **it no longer makes sense to grade answers alone**, because an answer could have come from anywhere. Checking the process is how the staff can tell you can actually do the homework.

### 8. Collaboration policy

Working together is **highly encouraged**. The line is drawn at three places:

1. **Don't look at someone else's code.** Exceptions: lab, your project partner, or after you have already solved the problem yourself.
2. **Don't tell other people the answers.** You *can* point out what's wrong and describe how to fix it, or show them a related example.
3. **Don't use AI (ChatGPT, Copilot, etc.) to write answers for you.** Exception: the AI-assisted part of the last project.

On the web generally: **you may read the web and learn from it.** You may not look up answers to the course's questions.

With a project partner specifically, you can work together directly and look at each other's screens. With everyone else in the course, you can still discuss ("this problem was like that one from lecture, here's how I got going").

The underlying value stated in lecture: **the point of the course is to learn what you don't already know, not to demonstrate what you already know.** You get no credit for the latter. So if you're working with someone who hasn't learned everything yet, help them and don't give them a hard time.

### 9. Community, harassment, and reporting

Disparaging remarks are unacceptable, especially remarks targeting a gender, race, or ethnicity. The example called out in lecture was mocking someone for not figuring something out ("how come that person couldn't figure it out"), which "ruins everybody's day."

Quoted commitments:
- Berkeley Principles of Community: "We affirm the dignity of all individuals and strive to uphold a just community in which discrimination and hate are not tolerated."
- EECS mission: "Diversity, equity, and inclusion are core values... Our excellence can only be fully realized by faculty, students, and staff who share our commitment to these values."

Reporting channels:
- Tell the instructor directly.
- **denero.org/feedback.html** for anonymous feedback.
- **EECS Student Climate & Incident Reporting Form**, or contact **Susanne Kauer (skauer@berkeley.edu)** directly.

### 10. Expressions and values

Now the computer science. The core definition:

> **An expression describes a computation and evaluates to a value.**

The slides give four small examples spanning different *kinds* of expression:

| Expression | Kind |
|---|---|
| `2` | Number or numeral |
| `25 + 3` | (a compound arithmetic expression) |
| `pi` | Name |
| `'are you human'` | String |

The slide then shows a collection of mathematical notations: $2^{100}$, $\sqrt[3]{2}$, $7 \bmod 2$, $|-1869|$, $f(x)$, $\sum_{i=1}^{100} i$, $\sin \pi$, $\sqrt{3493161}$, $\binom{69}{18}$, $\log_2 1024$, $\lim_{x \to \infty} \frac{1}{x}$. The point of this collection is that mathematics uses a sprawl of *different notations* for different operations: superscripts, radicals, bars, big sigmas, subscripts, infix words like "mod." Each one is its own special syntax you have to learn separately.

The punchline on the next slide is the contrast: **in Python, all expressions can be written as call expressions.** One uniform notation, `operator(operand, operand, ...)`, replaces that whole zoo. This is the first real instance of the course's abstraction theme: a single regular form you can learn once and apply everywhere, which is also what makes a uniform evaluation procedure possible.

### 11. Anatomy of a call expression

```
add ( 1 + 1 , 3 )
 |      |     |
 |      |     └── operand
 |      └──────── operand
 └─────────────── operator
```

Critically: **the operator and the operands are themselves expressions.** `add` is a name expression. `1 + 1` is an expression. `3` is an expression. This is exactly what makes the structure recursive.

Note the vocabulary distinction the lecture was careful about, and which exams test:

- **operator / operand**: what the parts are *before* evaluation (syntactic roles).
- **function / argument**: what they become *after* evaluation (the values).

The operator evaluates to a **function**. Each operand evaluates to an **argument**.

### 12. The evaluation procedure for call expressions

```
(1) Evaluate the operator          → a function
(2) Evaluate each operand          → arguments
(3) Apply the function to the arguments
```

Walking the lecture's own narration of `add(1 + 1, 3)`:

1. **Evaluate the operator.** Python finds the thing before the parentheses, `add`, and figures out what it is. Informally: *"what are we doing? Okay, we're adding."* The result is a function.
2. **Evaluate each operand.** `1 + 1` evaluates to `2`. `3` evaluates to `3`. Once evaluated, these are called **arguments**.
3. **Apply the function to the arguments.** Apply `add` to `2` and `3`, giving `5`.

### 13. Why this procedure handles nesting: it refers to itself

This was flagged in lecture as the "nifty" part and it is the conceptual heart of the day.

The three-step procedure **never says what to do about a big expression**. It doesn't say what to do when one function call takes another function call as an argument, which takes yet another. There is no special case for depth, no "and if the operand is itself a call, then...".

It doesn't need one, because **the definition refers to itself**. Step 2 says "evaluate each operand." An operand might itself be a call expression, so evaluating it means running this same three-step procedure again, one level down. The procedure bottoms out at things like numerals and names, which evaluate directly.

This is your first encounter with a **recursive definition**, and the course will lean on this shape repeatedly.

### 14. Expression trees

Because evaluation recurses into subexpressions, a nested expression naturally draws as a **tree**:

- **Leaves** are the primitive expressions: numerals like `4`, `6`, `3`, `5`, and names like `mul`, `add`.
- **Internal nodes** are call expressions.
- **Each node's box holds the value that subexpression evaluates to.**
- Values flow **upward**: children are evaluated, then combined by applying the function at the node, and the result becomes an argument to the node above.

Terminology attached to the diagram in the slides (and named explicitly in lecture, "so you can tell all your friends you learned some great new words"):

- The whole diagram: the **expression tree**.
- `add(4, mul(4, 6))`: an **operand subexpression**. It's an operand, but it's also an expression, and it's part of a bigger call expression, hence "subexpression."
- The box next to it holding `28`: the **value of the subexpression**, which is also the **first argument to `mul`**.
- The box at the top holding `224`: the **value of the whole expression**.

A poll was run at `pollev.com/cs61a` asking **which part of the expression gets evaluated first**, with students asked to discuss with a neighbor first.

---

## Definitions

**Expression**: something that describes a computation and evaluates to a value.

**Value**: the result of evaluating an expression.

**Primitive expression**: an expression that is a single evaluation step, requiring no subexpressions. The lecture's categories were numbers/numerals (`2`), names (`pi`), and strings (`'are you human'`). *(extra context: "primitive expression" is the textbook's umbrella term for these; the slide listed the categories without that label.)*

**Numeral / Number**: a literal expression that denotes a number, such as `2` or `25`.

**Name**: an expression consisting of an identifier, such as `pi`, `add`, or `mul`, which evaluates to whatever value is currently bound to that name.

**String**: an expression written in quotes, such as `'are you human'`, that evaluates to a text value.

**Call expression**: an expression of the form `operator(operand, operand, ...)`, which applies a function to arguments. All expressions in Python can be written as call expressions.

**Operator**: in a call expression, the expression that appears *before* the parentheses. It evaluates to a function.

**Operand**: in a call expression, an expression that appears *inside* the parentheses, separated from others by commas. Each operand evaluates to an argument.

**Function**: the value that the operator evaluates to; the thing that gets applied.

**Argument**: the value that an operand evaluates to; the thing a function gets applied *to*.

**Apply**: step (3) of evaluation, where the function is carried out on the arguments to produce a value.

**Operand subexpression**: an operand that is itself a compound expression (typically a call expression) nested inside a larger call expression.

**Expression tree**: the tree-shaped diagram of a nested expression, whose leaves are primitive expressions, whose internal nodes are call expressions, and whose node boxes record the value each subexpression evaluates to.

**Evaluation procedure for call expressions**: (1) evaluate the operator, (2) evaluate each operand, (3) apply the resulting function to the resulting arguments. Recursive, because step (2) may require running the whole procedure again.

**Terminal**: a place where you write text commands to tell your computer what to do; it controls the computer the way a mouse does, but with text. Where you run your programs.

**Preceptor**: the course's VS Code AI extension. Tells you where your problems are and checks understanding, rather than giving answers.

**Provenance**: the course's VS Code extension that logs how you produced your assignment file (typed vs. pasted, app switching), stored locally, submitted at your discretion.

**Mega discussion / mega lab**: the opt-out versions of discussion and lab. You read the document on your own and attendance is not required.

---

## Worked Examples

### Example 1: `add(1 + 1, 3)` (the anatomy slide)

```python
>>> from operator import add, mul
>>> add(1 + 1, 3)
5
```

*(extra context: the `from operator import add, mul` line is what makes the names `add` and `mul` available; the lecture demo had already imported them.)*

Step by step, following the procedure exactly:

1. **Evaluate the operator.** The operator is the expression `add`, appearing before the parentheses. It is a name. Evaluating a name means looking up what it's bound to, which here is the built-in addition function. *"What are we doing? Okay, we're adding."*

2. **Evaluate each operand.**
   - First operand is `1 + 1`. This is itself an expression, not a primitive one, so it must be evaluated: it yields `2`. Once evaluated, `2` is an **argument**, not an operand.
   - Second operand is `3`. A numeral, a primitive expression, evaluates immediately to `3`. That's also now an argument.

3. **Apply the function to the arguments.** Apply the addition function to `2` and `3`. Result: `5`.

The value of the whole call expression is `5`.

The thing to internalize: `1 + 1` did not get handed to `add` as the text "1 + 1". It was **fully evaluated first**. Python always reduces operands to values before applying anything.

### Example 2: `mul(add(4, mul(4, 6)), add(3, 5))` (the nested-expression slide)

```python
>>> from operator import add, mul
>>> mul(add(4, mul(4, 6)), add(3, 5))
224
```

**The expression tree**, described in words:

- The **root** node is the outermost call, `mul(...)`. Its box will hold `224`.
- The root has three children, reflecting the three parts of a call expression:
  - the operator `mul` (a leaf, a name),
  - the first operand `add(4, mul(4, 6))` (an internal node, itself a call expression),
  - the second operand `add(3, 5)` (an internal node, itself a call expression).
- The node `add(4, mul(4, 6))` has three children: the operator `add` (leaf), the operand `4` (leaf), and the operand `mul(4, 6)` (internal node). Its box will hold `28`.
- The node `mul(4, 6)` has three children: `mul` (leaf), `4` (leaf), `6` (leaf). Its box will hold `24`.
- The node `add(3, 5)` has three children: `add` (leaf), `3` (leaf), `5` (leaf). Its box will hold `8`.

**Evaluation, in order:**

1. Start at the whole expression. Step (1): evaluate the operator `mul`. It's a name; it evaluates to the multiplication function.

2. Step (2): evaluate each operand. Start with the first, `add(4, mul(4, 6))`. This is a call expression, so we **re-enter the same procedure** one level down.

   2a. Evaluate its operator, `add` → the addition function.

   2b. Evaluate its operands.
   - First operand: `4`, a numeral → `4`.
   - Second operand: `mul(4, 6)`, another call expression → **re-enter the procedure again**.
     - Evaluate the operator `mul` → the multiplication function.
     - Evaluate the operands `4` → `4` and `6` → `6`.
     - Apply multiplication to `4` and `6` → **`24`**. Put `24` in that node's box.

   2c. Apply addition to `4` and `24` → **`28`**. Put `28` in that node's box. This `28` is the **first argument to `mul`**.

3. Still in step (2) of the outer expression: evaluate the second operand, `add(3, 5)`. Re-enter the procedure: operator `add` → addition function; operands `3` → `3` and `5` → `5`; apply → **`8`**. That's the second argument to `mul`.

4. Step (3) of the outer expression: apply multiplication to `28` and `8` → **`224`**. That is the value of the whole expression.

**Which part gets evaluated first?** Following step (1) strictly, the very first thing evaluated is the **outermost operator, `mul`**. Among the *arithmetic* that actually computes a number, the first to complete is the innermost call, **`mul(4, 6)` → `24`**, because it is the deepest node on the leftmost branch and nothing above it can finish until it does. This is precisely the poll question, and the distinction between "first thing looked at" and "first arithmetic result produced" is the subtlety being probed.

**The shape of the traversal**, stated generally: you descend from the root down into subexpressions, and values propagate back upward. Nothing at a node can be applied until all of that node's children have values.

### Example 3: mathematical notation rewritten as call expressions

*(extra context: the slides showed the mathematical notations and separately asserted that all expressions can be written as call expressions, but did not print these specific Python translations. They are included here because the connection is the slide's evident point.)*

```python
>>> from operator import add, sub, mul, mod, abs
>>> pow(2, 100)
1267650600228229401496703205376
>>> mod(7, 2)
1
>>> abs(-1869)
1869
>>> from math import sqrt, log, sin, pi
>>> sqrt(3493161)
1869.0
>>> log(1024, 2)
10.0
```

Each line replaces a distinct piece of mathematical notation (a superscript, the word "mod", vertical bars, a radical sign, a subscripted log) with the *same* uniform syntax: a name, parentheses, comma-separated operands. That uniformity is exactly why one three-step evaluation procedure suffices for the entire language.

### Example 4: reading an expression tree backwards

*(extra context: a constructed exercise in the lecture's style, not an example given in class.)*

Given:

```python
>>> add(mul(2, 3), mul(add(1, 1), 4))
```

Work it out by the procedure:

- Operator `add` → addition function.
- First operand `mul(2, 3)`: operator `mul` → multiplication; operands `2`, `3`; apply → `6`.
- Second operand `mul(add(1, 1), 4)`:
  - operator `mul` → multiplication;
  - first operand `add(1, 1)`: operator `add`, operands `1` and `1`, apply → `2`;
  - second operand `4` → `4`;
  - apply multiplication to `2` and `4` → `8`.
- Apply addition to `6` and `8` → **`14`**.

The innermost completed call is `add(1, 1)` or `mul(2, 3)` depending on left-to-right order of operand evaluation: Python evaluates operands left to right, so `mul(2, 3)` finishes first.

---

## Common Pitfalls

**Confusing operator/operand with function/argument.** These name the *same parts* at *different stages*. `add` is an operator; the addition function is what it evaluates to. `1 + 1` is an operand; `2` is the argument. Saying "the argument is `1 + 1`" is wrong on an exam.

**Thinking operands are passed unevaluated.** Operands are fully reduced to values *before* the function is applied. `add(1 + 1, 3)` never hands `add` the text `1 + 1`.

**Forgetting step (1) exists.** The operator is an expression too, and it gets evaluated first. Students often jump straight to the operands. On the "what is evaluated first" question, the answer for `mul(add(4, mul(4, 6)), add(3, 5))` is the operator `mul`, not `mul(4, 6)`, unless the question specifically asks what arithmetic completes first.

**Expecting the procedure to have a special case for nesting.** It doesn't, and that's the point. There is exactly one procedure, applied repeatedly to itself. Looking for an extra rule to handle deep expressions means you've missed the recursive structure.

**Reading the expression tree top-down as an evaluation order.** You *descend* top-down, but values are *produced* bottom-up. Nothing at a node is applied until all its children have values.

**Skipping the videos and treating lecture as complete.** Lecture explicitly covers the most important content but not all of it. The videos and the textbook are where the complete content lives. Coming to lecture cold means lecture becomes your first exposure rather than your second, which is the failure mode the quoted student advice describes ("the reason why I was so lost during the second half of the semester").

**Watching videos at 2x.** Called out directly in the quoted student advice as something a past student regretted: information that isn't processed leaves your brain by the next morning.

**Watching videos passively.** Both the slides and lecture say to **type the examples out yourself**. Watching someone program is the piano-listening failure mode.

**Using ChatGPT/Gemini/Copilot for homework.** Against policy, and self-defeating in two concrete ways: those models already know this course's problems (past students posted them), so they'll just hand you answers; and the quizzes are homework problems redone, so you'll fail them.

**Attending discussion silently.** Solving every problem on the worksheet while talking to nobody is explicitly described as missing the point. The worksheet isn't graded; the discussing is the deliverable.

**Showing up to discussion without finding your group.** Multiple groups share one room. You need your group number.

**Assuming Provenance is optional surveillance you can't control.** The log stays on your machine, covers only the assignment directory, and you choose whether to submit it. If you dislike a log, re-download and redo the assignment.

**Confusing "you may read the web" with "you may look up answers."** Learning from the internet is fine. Looking up solutions to course problems is not.

**Taking 61A as an absolute beginner without thinking about it.** About three quarters of students have programmed before and taken a course. Students without prior experience tend to work many more hours and still score lower on exams. It's possible, but the advice given was that most such students have a better time taking something like **CS 10 (The Beauty and Joy of Computing)** first, and that **there is no rush**: finishing a semester earlier confers no advantage. If you feel lost after three weeks, switching to a course with a more moderate pace is a good idea, and "there's nothing wrong with being a beginner."

---

## Likely Exam Points

### 1. Order of evaluation in a nested call expression

This is the single most-tested idea from this lecture, and it recurs all semester in environment-diagram problems.

**Q.** In `mul(add(4, mul(4, 6)), add(3, 5))`, which part is evaluated first, and which call produces a value first?

**A.** By step (1) of the procedure, the first thing evaluated is the **operator of the outermost call, `mul`**. The first *call* to produce a value is the innermost one, **`mul(4, 6)` → `24`**, since it sits at the deepest point of the leftmost operand and every enclosing call must wait on it.

### 2. Naming the parts

**Q.** In `add(1 + 1, 3)`, identify the operator, the operands, the function, and the arguments.

**A.** Operator: `add`. Operands: `1 + 1` and `3`. Function: the value `add` evaluates to (the addition function). Arguments: `2` and `3`, the values the operands evaluate to.

### 3. Stating the evaluation procedure

**Q.** State the evaluation procedure for call expressions, and explain how it handles arbitrarily deep nesting without any extra rule.

**A.** (1) Evaluate the operator to get a function. (2) Evaluate each operand to get arguments. (3) Apply the function to the arguments. It handles nesting because the definition refers to itself: an operand may itself be a call expression, so step (2) recursively invokes the whole procedure. Recursion bottoms out at primitive expressions (numerals, names, strings), which evaluate in one step.

### 4. Counting evaluation results / tracing values

**Q.** Give the value of every node in the expression tree for `mul(add(4, mul(4, 6)), add(3, 5))`.

**A.** `mul(4, 6)` → `24`; `add(4, 24)` → `28`; `add(3, 5)` → `8`; the whole expression `mul(28, 8)` → `224`. Leaf names `mul` and `add` evaluate to the multiplication and addition functions; leaf numerals evaluate to themselves.

### 5. Kinds of expressions

**Q.** Classify each: `2`, `pi`, `'are you human'`, `25 + 3`, `add(2, 3)`.

**A.** `2` is a numeral (a primitive expression). `pi` is a name. `'are you human'` is a string. `25 + 3` and `add(2, 3)` are compound expressions; the latter is a call expression, and *(extra context)* the former is equivalent to the call expression `add(25, 3)`.

### 6. The uniform-notation claim

**Q.** Why does the lecture contrast mathematical notation ($2^{100}$, $\sqrt{x}$, $|x|$, $\sum$, $\log_2$) with Python call expressions?

**A.** Mathematics uses many different special-purpose notations, each of which must be learned separately. In Python, **all expressions can be written as call expressions**, a single uniform `operator(operand, ...)` form. That uniformity is what makes one general evaluation procedure sufficient for the whole language, and it's an early example of the course's theme of abstraction.

### 7. Course-policy questions (plausible on a syllabus quiz)

**Q.** May you look at a classmate's code? May you use ChatGPT on homework?

**A.** You may not look at someone else's code, except in lab, with your project partner, or after you have already solved the problem yourself. You may not use ChatGPT/Copilot to write answers; the only sanctioned AI is the course's **Preceptor** tool during the first 11 weeks, plus the AI-assisted portion of the final project. You may help someone by pointing out what is wrong, describing how to fix it, or showing a related example, but not by telling them the answer. Reading the web to learn is fine; looking up answers to course problems is not.

**Q.** What are quizzes, and how do you prepare?

**A.** Quizzes happen during lab and ask you to redo a homework problem with slight variations. Preparation is simply doing the homework yourself.

---

## Summary

**Course**

- CS 61A is about **managing complexity**: mastering abstraction, solving problems, organizing complex programs. It is also an intro to programming, language fundamentals, big projects, and how computers interpret languages.
- Three languages: **Python**, **Gleam** (functional), **SQL** (databases). Different languages suit different kinds of problems.
- Instructors: **Kay Ousterhout** (kayo@berkeley.edu, OH Wed 1-2pm Wheeler 210) and **John DeNero** (denero@berkeley.edu, OH Fri 1-2pm Wheeler 210). Extra instructor OH next week. Staff list at cs61a.org/fa26/staff/.
- Contact: **Ed** (private posts reach staff, public posts reach students), the instructors' emails, or **cs61a@berkeley.edu**. All linked from the course website's Contact section.

**How to succeed**

- **Videos before lecture.** The videos cover all content (about 25 min per lecture, 3-4 videos). Lecture covers only the highlights, examples, strategies, and perspective. Videos are the album; lecture is the concert.
- **composingprograms.com** is the textbook: concise, content very similar to the videos. Either works; both is better.
- **Type out the examples yourself.** Don't watch at 2x.
- You don't learn piano by listening. Practice is mandatory: **lab (Mon/Tue, attendance required)**, **discussion (Wed/Thu/Fri, attendance required, starts this week)**, weekly homework, **four projects**, **quizzes in lab** (homework problems redone), and **drop-in help starting next week** (~8 hrs/week, Gateway building).
- Discussion: ~5-person group, group number, multiple groups per room, ungraded online worksheet, **talking is the point**. Bring a laptop or tablet. Mega discussion = opt out; you can switch in via the announcements at the top of cs61a.org.

**AI**

- AI has genuinely changed how software is built, which is *why* you must learn to program: understanding programs is a huge advantage when using AI. Roughly **90% of using AI well is understanding and describing what you're building**; 10% is the tools.
- Overuse blocks learning: you never learn to get unstuck, never learn why wrong answers are wrong, never learn to see inside a program.
- **Weeks 1-11: write programs yourself**, with **Preceptor** (points at problems, checks understanding, doesn't give answers) as the only AI. **End of semester: explicit AI-assisted programming** on the last project.
- Don't use ChatGPT/Gemini/Copilot: against policy, and they already know this course's problems.

**Setup and policies**

- **Lab 0** (released tonight, take-home) installs **Python**, **VS Code**, and the **Preceptor** and **Provenance** extensions. Everyone uses the same setup on purpose.
- **Provenance** logs typed vs. pasted and app switching, **locally only**, scoped to the assignment directory, and **you choose whether to submit**. Grading answers alone no longer makes sense, so the process is checked.
- Collaboration encouraged; **don't look at others' code** (except lab, project partner, or post-solution), **don't give answers**, **don't use AI to write answers**. Learn from the web, don't look up answers.
- **No harassment or discrimination.** Report to the instructor, anonymously via denero.org/feedback.html, via the EECS Student Climate & Incident Reporting Form, or to Susanne Kauer (skauer@berkeley.edu).
- Syllabus at cs61a.org/fa26/syllabus/; read it before next week.
- ~3/4 of students have prior programming experience. Beginners can pass but work much harder; **CS 10** is a reasonable first course, and there is no rush.

**Computer science content**

- An **expression** describes a computation and **evaluates to a value**. Kinds shown: numerals (`2`), names (`pi`), strings (`'are you human'`), compound expressions (`25 + 3`).
- Math uses many notations ($2^{100}$, $\sqrt{\,}$, $|\,|$, $\sum$, $\log_2$, $\lim$); **Python writes them all as call expressions**, one uniform form.
- A **call expression** is `operator(operand, operand, ...)`. **Operator and operands are themselves expressions.**
- **Evaluation procedure:** (1) evaluate the operator → function; (2) evaluate each operand → arguments; (3) apply the function to the arguments.
- Operator/operand are the roles *before* evaluation; function/argument are what they become *after*.
- The procedure is **recursive**: it has no special case for nesting, because step (2) can invoke the whole procedure again. This is what lets it evaluate arbitrarily deep expressions.
- A nested expression draws as an **expression tree**: leaves are primitive expressions, internal nodes are calls, each box holds a subexpression's value, values flow upward.
- Worked case: `mul(add(4, mul(4, 6)), add(3, 5))` → `mul(4, 6)` = `24`, `add(4, 24)` = `28`, `add(3, 5)` = `8`, whole expression = **`224`**.
- Vocabulary from the tree: **operand subexpression**, **value of the subexpression**, **first argument to `mul`**, **value of the whole expression**, **expression tree**.

**Project preview**

- The final project is a program that **draws generative art**, and it's flexible: changing a few lines changed four sequential drawings into four simultaneous ones, producing a different picture.
- DeNero's own extension was a **spirograph**: circles inside circles with a pen, drawing curves made of circles, extensible by adding more circles in code, and ultimately able to trace an arbitrary target curve (he demoed the Cal logo) by finding circles whose spinning draws it.
- You pick your own artistic extension. It's a **web app**, a big project, with some AI assistance permitted at that point.
