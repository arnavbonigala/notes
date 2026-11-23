<!-- Mon, Nov 23, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 37: The Social Implications of Computing

This lecture steps away from data structures entirely and asks what the technical skills you built this semester actually do to the world once you take them out of the classroom. Josh Hug and Manuel Sabin walk through three movements: first, tech careers, using engagement-driven products (Candy Crush, Snapchat, TikTok) versus mission-driven ones (Khan Academy) as the case study for how the same engineering competence can be aimed at very different ends; second, a brief history of software's reshaping of human society from the 1994 Today Show "what is the internet, anyway?" clip through the 2020 GPT-3 demos, 2023-2026 LLM progress, and the observation that AI has suddenly lifted the intellectual ceiling on what one programmer can build; and third, fulfillment, arguing via run-and-tumble bacteria and the hedonic treadmill that constraints and achievable struggle are what make work worth doing. The through-line is that your skills are the limiting reagent that many organizations want, revenue-per-employee at the big tech firms runs into the millions, so the scarce thing is not opportunity but the judgment about where to point yourself. There is no code and no testable algorithmic content in this lecture; the companion textbook chapter (37. Software Engineering IV) is a retrospective on the history of CS 61B itself.

## Key Concepts

### Engagement mechanics are engineered, not accidental

The lecture opens with three concrete product features and asks why each exists:

- **Candy Crush streak counter.** Every consecutive day of play earns a reward: a visible progress indicator, plus up to two hours of a special item that makes the game more fun that day. Miss a day and the counter resets to zero.
- **Snapchat snap streaks.** Every day you and a friend both communicate, the streak extends by one.
- **TikTok.** The instant you open the app, a video is already playing. Swiping to the next one takes milliseconds.

The point of grouping these is that none of them is a side effect of good engineering. Each is a deliberate design decision whose purpose is to increase engagement. The slides give the student-generated answers to "why does this feature exist?": for addiction, to keep you playing, to increase engagement, to keep people coming back, and also, honestly, because it is fun and rewarding for the player.

The streak mechanic in particular is worth understanding structurally. The reset-to-zero rule is what gives it its grip: it converts a reward into a thing you can *lose*. A counter that only went up would be a mild pleasure. A counter that resets creates a daily obligation. That asymmetry is the whole design.

The lecture also notes that the person who implements a streak counter is doing exactly the kind of work you have been doing all semester: keeping state, handling the time-based edge cases, persisting the count. The technical work is value-neutral. The decision to reset on a miss is not.

### The same feature has real positive and real negative impacts

The class brainstormed both columns, and the lecture takes the positive column seriously rather than treating it as a throat-clearing before the real criticism.

Positives named in the slides:
- Connection and community: streaks and notifications genuinely do keep you in touch with people.
- Habit-building in tools that deserve it. The slide quotes a student: "In tools like study apps, they can create a true positive tool for people (me) to build good habits (I have an app that gives me virtual tomatoes and it makes me happy)."
- Opportunities and learning.

Negatives named:
- Addiction and time consumption.
- Shallow interaction. The student phrasing: "my online friendships aren't as fruitful."

Other engagement-generating features the class surfaced from their own lives: likes on social media, video games with daily rewards, Duolingo streaks (the lecturer mentions a 700-day streak, having run out of Spanish and moved on to chess, "nature's language"), notifications (the lecturer keeps these off, allows SMS, sometimes Slack), autoplay on Netflix / Amazon Video / YouTube, and Oski Eats, which "drives engagement by existing" because calories and menus are genuinely useful.

That last one is the interesting case. Oski Eats is engaging because it is useful, not because it was tuned to be sticky. The lecture does not draw a bright line here, but the contrast is there: engagement that falls out of utility versus engagement that was manufactured.

### The net-impact question, and the honesty of the answer

The lecture then asks the harder question: is TikTok a net positive for the world? The recorded room answer is blunt and worth quoting as-is because it is the actual pedagogical point:

> "Overall more net negative, but I will work there for $150,000/year."

This is not presented as a gotcha. It is presented as the realistic shape of the decision most students will actually face. You can believe a product is net-negative and still take the job, and the lecture's response is not "don't," it is "notice that you are doing it, and notice why."

By contrast, the Khan Academy vote was "almost all net positive, a few net-neutral." But the Khan Academy discussion is not a softball either; the class generated substantive criticisms:

Positives: free education, a tool for human-to-human teaching (you can use it to teach someone else), reduces educational inequality, gamifies learning in a way that encourages people to keep going, and encourages people to make mistakes while giving guidance when they do.

Negatives: answers are very specific so it is hard to succeed and the education is narrow; some people get hooked and go overboard ("they don't eat, it's chaos"); it advertises itself as a singular form of learning and doesn't link out to the wider world; and it can give biased information, since if you want to learn history you are following *their* curriculum.

Note the symmetry: gamification appears in the positive column for Khan Academy and in the negative column for TikTok. The mechanic is the same. What differs is what it is pulling you toward.

### Why people choose the engagement job anyway

The class listed reasons someone might prefer ByteDance (TikTok's parent) over Khan Academy:

- Money. The slide's framing is pointed: it is "easier to monetize suffering (addicted customers) over success."
- Brand and prestige, more impressive to mention at a party.
- More power over others.
- Opens more career doors later.
- Better networking.
- More opportunity for growth.
- Impact (reach, in the raw sense of how many people touch the thing you built).

### The economics behind those reasons

The lecture backs this up with numbers rather than leaving it as vibes.

| | Revenues | Profits | # Employees | Revenue / Human Employee |
|---|---|---|---|---|
| ByteDance | $186 billion (2025) | $50 billion (2025) | 150,000+ (2025) | ~$1,240,000 |
| Khan Academy | $0.090b (donations), $0.022b (programs) [2026 figures] | N/A | 227 total: 198 humans, 23 dogs, 5 cats, 1 bearded dragon [April 2025] | ~$60,000 (revenue only), ~$590,000 (revenue + donations) |

The bearded dragon is a real line item in the slide. It is also doing rhetorical work: the headcount joke underlines how small a non-profit is next to a company whose revenue per employee is north of a million dollars a year.

### Revenue per employee as a measure of scarcity

The lecture generalizes this with a second table, the "limiting reagent" argument. Historically, STEM experts have been a rare commodity, and the revenue each one is attached to shows it:

| Company | Revenue per Employee | Profit per Employee |
|---|---|---|
| Meta | $2.55 million | $767,000 |
| Alphabet (Google) | $2.11 million | $693,000 |
| Amazon | $459,000 | $50,000 |
| Microsoft | $1.24 million | $447,000 |
| Apple | $2.51 million | $675,000 |

(The slides give two columns of figures; the first is labeled revenue per employee, the second is the corresponding profit-side figure. Amazon's low numbers reflect its enormous non-engineering workforce.)

The conclusion the lecture draws: the skills you are building will be in high demand from companies, non-profits, government agencies, educational institutions, and more. **The choice of how to spend your career is yours.** The scarce resource is you, which means the leverage over where your work points is also yours.

### Software as the purest creative medium, and the AI inflection

This section borrows a framing from John Ousterhout's lecture (Ousterhout is a Stanford professor and the author of *A Philosophy of Software Design*; the lecture does not elaborate on who he is, that is extra context):

> Programming: the purest creative medium in the history of the world?
> - Not limited by physical constraints.
> - If you can imagine it, you can build it.
> - Limitations are intellectual: can we understand what we have built?

That third bullet is the load-bearing one for the rest of the lecture. A bridge is constrained by steel. A program is constrained only by whether a human mind can hold it. Every abstraction you learned this semester, every ADT, every interface, exists to raise that ceiling, to let you hold more system in your head at once.

The lecture's claim is that AI has suddenly lifted that intellectual ceiling considerably, and Hug's best guess is that the impact on society will be immense, on par with the discovery of electricity.

### The timeline

The lecture walks a chronology of how fast this moved:

- **1994:** The Today Show clip, hosts asking on air what the internet even is. The reference point for how recently software was marginal to ordinary life.
- **2007:** (slide shown, presumably the iPhone announcement, reconstructed from context and not stated in the extracted text).
- **2020:** Sharif Shameem's viral demo (the tweet is linked in the slides) showing GPT-3 generating working UI code from a plain-English description.
- **2023:** (slide shown, no extracted text).
- **2024:** GPT o1 scored in the top 4% of the class.
- **2025:** ChatGPT o3 could do the entire CS 61B project without trouble.
- **2026:** Hug reports using LLMs for things that would have been impossible six months earlier. The concrete example: training an LLM to review student design docs for Project 4B, with a linked Google Doc of the setup. The class was polled on how they found the LLM-generated WordNet design doc feedback.
- **March 2026:** The lecture points at the Knight First Amendment Institute piece "AI as Normal Technology" (knightcolumbia.org) as having great thoughts on adoption speed.

The "AI as Normal Technology" citation is a deliberate counterweight. The rest of the timeline reads as acceleration; that link argues adoption is slower and more ordinary than the capability curve suggests. The lecture puts both on the table.

The forward-looking claim: these tools will continue to **amplify the power of folks who have the knowledge to use them.** Note the conditional. The argument is not that knowledge stops mattering, it is that the returns to knowledge go up, because the person who understands what the system is doing can direct a far larger amount of machine output than the person who doesn't.

This connects back to the fulfillment section's premise: software is largely unconstrained by the laws of physics, the limitation is what you can keep in your head, and higher-level abstractions yield bigger systems. Until recently, huge numbers of programmers were needed to translate ideas into code. Now the skill set required is growing more advanced.

### Fulfillment: Master of Magic and the case for constraints

Hug describes a game he played as a kid, Master of Magic. It was trivial to edit your saved game file so your starting town was already huge and prosperous. And then the game wasn't fun to play.

The conclusion drawn: **constraints and achievable, reasonable struggle are what make life worth living.** Note both qualifiers. Not struggle in general, and not arbitrary struggle. Achievable and reasonable. A challenge you cannot possibly meet is not fulfilling either; it is just grinding.

This is the lecture's answer to the AI section. If AI removes the struggle from building things, the Master of Magic story says that is not straightforwardly a win.

### Run-and-tumble and the hedonic treadmill

From Hug's microbiology background in grad school: flagella-bearing bacteria navigate using **run-and-tumble**. They swim in a straight line (run), then randomly reorient (tumble), then run again. The trick is that if conditions are improving, they run longer before tumbling. They have no map and no ability to perceive the gradient directly; they only sense whether things are getting better. That biased random walk reliably climbs a nutrient gradient.

The analogy: humans climb gradients too, and gain happiness in the process. Which leads to the **hedonic treadmill**: we adapt to improvements, so the happiness comes from the climbing more than from the arriving. The lecture distinguishes two kinds:

- **Transient:** pay increases, a new car. You adapt, the baseline resets, and you need the next increment.
- **Durable:** relationships, good habits, fulfilling work.

The practical implication, which the lecture leaves implicit: the TikTok-versus-Khan-Academy decision is often framed as a salary question, and salary is explicitly in the transient column. "Fulfilling work" is in the durable one.

There is also a quiet rhyme between run-and-tumble and the streak mechanics from the opening. A streak counter is an artificial gradient. It gives you the sensation of climbing without any underlying nutrient. That is precisely why it works and precisely why it is hollow.

### The textbook chapter: 37. Software Engineering IV

The companion chapter (by Mihir Mirchandani) is a different kind of reflection, a history of CS 61B itself rather than of computing. Briefly:

61B has been taught since roughly Spring 1994; before that it was CS 60C, going back to at least 1988. Four varieties have existed in modern times:

- **Hilfinger:** four extremely long real-world projects, somewhat based on data structures material.
- **Hug:** one (or two) long real-world projects; the remaining material ties tightly to lectures.
- **61BL:** lab-based, heavy data structures focus, with one large real-world project (Gitlet).
- **Shewchuk / Yelick (extinct):** focus on implementing data structures, no large real-world project.

**61B 1.0 (Spring 2015):** Hug's first solo offering, and where Gitlet was first offered. Projects had significant student authorship: Project 0 Bomb Checkers (Hug, implemented by Jimmy Lee), Project 1 ngordnet (Hug), Project 2 Gitlet (Joey Moghadam), Project 3 Fun with Tries (Hug, adapted from his old Princeton homeworks). Spring/Summer 2015 Gitlet was way too hard: no testing provided, no tips on persistence.

**61B 2.0 (Spring 2016, 2017):** The Fall 2014 / Spring 2015 observation was that students hated splitting time between core data structures content and a huge unrelated project. The fix: put the messy real-world project *before* the data structures material. 2016 was a text editor; 2017 was software for manipulating text databases (the chapter recommends trying it and then taking CS 186).

**Current version (3.6), Fall 2022 and Spring 2023:** Replaced Gitlet with Ngordnet, which is much more data-structures focused. 2A is building a TimeSeries and an NgramMap; 2B is building whatever is needed for additional functionality, including implementing a graph somehow. GSIs voted roughly 3 to 1 to keep Ngordnet over Gitlet.

The chapter's closing note: the course was meant to help with software engineering, and many students come out of it signing up for Linguistics 100 after learning about hyponyms in Ngordnet.

## Definitions

**Engagement (as a product metric):** the degree to which users return to and spend time in a product. The explicitly stated purpose of streaks, autoplay, notifications, and instant-play feeds.

**Streak mechanic:** a feature that counts consecutive days of a behavior, grants escalating rewards, and resets to zero on a miss. The reset is what converts the reward into a loss-avoidance obligation.

**Net positive / net negative:** the lecture's framing device for evaluating a product: summing its real benefits against its real harms and asking which side wins. Applied to TikTok (room: net negative) and Khan Academy (room: almost all net positive).

**Revenue per employee:** total revenue divided by headcount. Used in the lecture as a proxy for how much economic value an organization attaches to each worker, and therefore for how scarce and in-demand that worker's skills are.

**Limiting reagent:** borrowed from chemistry, the input that runs out first and therefore caps the whole reaction. The lecture's claim is that STEM experts have historically been the limiting reagent in building software, which is why their revenue per employee is so high and why the choice of where to spend a career carries leverage.

**Run-and-tumble:** the motility strategy of flagella-bearing bacteria: swim straight (run), randomly reorient (tumble), repeat, with runs lengthened when conditions are improving. Produces reliable gradient-climbing from purely local information.

**Hedonic treadmill:** the tendency to adapt to improvements in circumstance so that happiness returns toward a baseline, meaning satisfaction comes more from climbing a gradient than from occupying a position on it.

**Transient happiness:** happiness that fades through adaptation. Lecture examples: pay increases, a new car.

**Durable happiness:** happiness that persists. Lecture examples: relationships, good habits, fulfilling work.

**Intellectual ceiling (on software):** the limit on system size imposed by what a human can understand and hold in mind, as opposed to any physical limit. Raised historically by better abstractions, and, per the lecture, suddenly raised much further by AI.

## Worked Examples

This lecture contains no Java code and no algorithmic content. There is nothing to trace with box-and-pointer diagrams. What follows instead are worked applications of the lecture's analytical framework, which is the thing the lecture actually teaches.

### Worked Example 1: Applying the lecture's framework to a feature

Take the Snapchat snap streak and run the lecture's own three-step analysis.

**Step 1: What does the feature do, mechanically?** For each pair of users, maintain a counter. Increment it on each day both users send to each other. Reset it to zero on any day they don't. Display it prominently.

**Step 2: Why does it exist?** The lecture's answer: to keep people coming back. The reset rule means the cost of skipping a day is not "no reward today," it is "lose everything accumulated." For a 300-day streak, the perceived cost of one missed day is enormous and entirely manufactured.

**Step 3: Positive and negative impacts.**
- Positive, from the lecture's list: connection and community. The streak does cause two people to communicate daily who otherwise might drift.
- Negative, from the lecture's list: shallow interaction. The student phrasing was that online friendships "aren't as fruitful." The streak optimizes for *any* message, not a meaningful one, so it reliably produces a daily exchange of content-free snaps sent purely to preserve the number.

**The honest conclusion:** the mechanic produces contact but not necessarily connection. The metric it optimizes (daily message exchange) is a proxy for the thing we actually care about (friendship), and the feature causes the proxy and the goal to come apart.

(extra context) This proxy-divergence pattern is Goodhart's law: when a measure becomes a target, it stops being a good measure. The lecture does not name it, but it is the clean formulation of what the streak critique is describing.

### Worked Example 2: Reading the revenue-per-employee tables

Work through what the ByteDance / Khan Academy table actually tells you.

ByteDance: $186 billion revenue across 150,000+ employees gives roughly $1,240,000 per employee. Profit is $50 billion, so roughly 27% of revenue is profit, a margin that would be extraordinary for a manufacturer and is ordinary for software. That margin exists because software has near-zero marginal cost: serving the ten-millionth video costs essentially nothing more than serving the first.

Khan Academy: $0.090 billion in donations plus $0.022 billion in program revenue. Dividing by 198 human employees gives roughly $590,000 per person counting donations, and roughly $60,000 counting program revenue alone.

**What to take from the comparison.** ByteDance generates about twice as much per employee as Khan Academy on the donations-inclusive figure and about twenty times on the revenue-only figure. But the important reading is not "ByteDance employees are twenty times more productive." It is that the two organizations capture value in entirely different ways. ByteDance converts attention into advertising revenue, a mechanism that scales with engagement, which is exactly why the engagement features from the opening section exist. Khan Academy delivers roughly comparable per-person *value to users* and captures almost none of it, by design, because capturing it would mean charging for education.

This is the slide's point about it being "easier to monetize suffering (addicted customers) over success." The compensation gap between the two jobs is not a measure of how much good each one does. It is a measure of how much of the good each business model is able to convert into revenue.

### Worked Example 3: The Master of Magic argument, stated carefully

The structure of the argument:

1. Premise: in Master of Magic, editing the save file to start with a huge prosperous town was trivial.
2. Observation: having done so, the game stopped being fun.
3. Inference: the fun was not in *possessing* the prosperous town. It was in the process of getting there from a position where it was not guaranteed.
4. Generalization: constraints and achievable, reasonable struggle make life worth living.

**Where this bites in your actual life.** It is an argument against a certain reading of the AI section. If LLMs can do the entire 61B project without trouble (as the 2025 slide says o3 could), one reading is "great, the struggle is removed." The Master of Magic story says that is the save-file edit: you now possess the finished project and have skipped the thing that was actually worth having.

**The honest counter the lecture itself supplies.** The lecture does not say "therefore don't use the tools." It says these tools will amplify the power of folks who have the knowledge to use them. Both things are held at once: the struggle is where the knowledge comes from, and the knowledge is what makes the tools worth having. Editing your save file in 61B means arriving at the amplifier with nothing to amplify.

### Worked Example 4: Run-and-tumble as a search strategy

Trace what a bacterium actually does, since the mechanism is the point of the analogy.

The bacterium has no map of where food is and cannot perceive the gradient as a direction. It can only detect, over the last few seconds, whether nutrient concentration is rising or falling.

The algorithm, informally:
- Run: swim in a straight line.
- Sample: is concentration higher than it was a moment ago?
- If yes, keep running (delay the tumble).
- If no, tumble: reorient to a random new direction.
- Repeat.

Over many iterations this is a biased random walk that reliably climbs the gradient, despite the bacterium never knowing which way "up" is.

**The analogy the lecture draws.** Humans climb gradients the same way, and the happiness arrives during the run, not at the destination. Which is the setup for the hedonic treadmill: because you adapt to each new position, the satisfaction is in the climbing. Pay raises and new cars are tumbles that briefly register as improvement and then stop registering. Relationships, good habits, and fulfilling work are gradients you can keep running up.

(extra context) The computational connection, which the lecture gestures at but doesn't develop: this is hill climbing with random restarts, and it has the same failure mode, getting stuck on a local maximum. The career-advice reading is that a local maximum can feel like improvement right up until it stops, and the only escape is a tumble.

## Common Pitfalls

**Treating "net negative" as "don't work there, obviously."** The lecture explicitly records the honest answer, net negative but I will work there for $150,000 a year, without condemning it. The framework asks you to make the judgment consciously, not to arrive at a predetermined conclusion. Reading the lecture as a lecture against industry jobs misses its actual position.

**Treating gamification as inherently bad.** The same mechanic appears in Khan Academy's positive column ("encourages people to learn by gamifying education") and in TikTok's negative one. The slides also have a student praising a study app that gives them virtual tomatoes. The mechanic is not the variable; what it pulls you toward is.

**Treating the engagement-driven product as having no upside.** The positive column is real: connection and community, opportunities and learning. The lecture asks for both columns and then a net judgment, not for a one-sided prosecution.

**Reading the revenue-per-employee tables as productivity rankings.** They measure what each business model can capture, not how much good each worker does. Amazon's $459,000 per employee against Meta's $2.55 million mostly reflects Amazon's vast warehouse and logistics headcount, not that Amazon engineers do less.

**Reading the AI timeline as "learning the fundamentals no longer matters."** The lecture's explicit claim is the opposite: the tools amplify the power of folks who *have the knowledge to use* them. Amplification needs something to amplify. The Master of Magic story is the same argument from the other direction.

**Reading the AI timeline as uncritical hype.** The "AI as Normal Technology" link is in the slides precisely as a counterweight, flagged for its thoughts on adoption speed. The lecture puts the capability curve and the slower adoption curve side by side on purpose.

**Confusing transient and durable happiness when evaluating offers.** The lecture places pay increases explicitly in the transient column and fulfilling work in the durable one. The salary comparison that dominates the TikTok-versus-Khan-Academy decision is optimizing the column that fades.

**Confusing the lecture with the textbook chapter.** The slides are about the social implications of computing. The chapter (Software Engineering IV) is a history of CS 61B's own projects and course structure. They are both "Lecture 37" material and they are about entirely different things.

## Likely Exam Points

This lecture is not a source of algorithmic exam problems. There is no code, no runtime analysis, no data structure. In 61B, this material is typically either not examined at all or appears as a short reflection or discussion question. The items below are framed accordingly: they test whether you absorbed the framework, not whether you can trace a pointer.

**1. The purpose of a streak mechanic.**

*Q: Why does Candy Crush reset the consecutive-days counter to zero when you miss a day, rather than simply pausing it?*

*A:* Resetting converts the accumulated streak into something you can lose, which is a far stronger motivator than the prospect of a further small gain. A paused counter costs you one day's reward if you skip; a resetting counter costs you the entire accumulated streak. The feature exists to increase engagement and keep you playing, and the reset rule is the specific mechanism that does it.

**2. Both columns for a given feature.**

*Q: Give one genuine positive and one genuine negative impact of social-media streak features, as discussed in lecture.*

*A:* Positive: connection and community, they keep you in regular contact with people you would otherwise drift from. Negative: shallow interaction, since the feature rewards any daily message rather than a meaningful one, so online friendships become, in the student's phrasing, less fruitful.

**3. Why the engagement job pays more.**

*Q: ByteDance generates roughly $1,240,000 in revenue per employee; Khan Academy generates roughly $60,000 per employee in program revenue. Does this mean ByteDance employees do twenty times more good?*

*A:* No. The ratio measures how much value each business model *captures*, not how much it creates. ByteDance converts user attention into advertising revenue, which scales directly with engagement. Khan Academy deliberately gives its product away, so it captures almost none of the value it produces. The slides put it as: it is easier to monetize suffering (addicted customers) than success.

**4. The limiting reagent argument.**

*Q: What does the lecture mean by calling STEM experts a "limiting reagent," and what conclusion does it draw?*

*A:* A limiting reagent is the input that runs out first and caps the whole reaction. Historically, people who can build software have been that scarce input, which is why revenue per employee at Meta, Alphabet, Microsoft, and Apple runs between $1.24 million and $2.55 million. The conclusion: your skills are in demand from companies, non-profits, government agencies, and educational institutions alike, so the choice of how to spend your career is genuinely yours.

**5. Ousterhout's framing of programming.**

*Q: In what sense is programming "not limited by physical constraints," and what limits it instead?*

*A:* Unlike physical engineering, software is not bounded by materials, cost of matter, or the laws of physics: if you can imagine it, you can build it. The binding limit is intellectual, whether we can understand what we have built. This is why abstraction matters: higher-level abstractions let you hold more system in your head and therefore build bigger ones.

**6. The AI claim and its conditional.**

*Q: What does the lecture predict about AI's effect on programmers, and what condition does it attach?*

*A:* That AI has suddenly lifted the intellectual ceiling on software considerably, with societal impact Hug guesses is on par with the discovery of electricity. The condition: these tools will amplify the power of *folks who have the knowledge to use them*. The returns to understanding go up, they do not go to zero.

**7. The Master of Magic argument.**

*Q: What point is Hug making with the story about editing his Master of Magic save file?*

*A:* Editing the save to start with a huge prosperous town was easy, and it made the game not fun. The fun was in getting there, not in having it. The generalization: constraints and achievable, reasonable struggle are what make life worth living. Both qualifiers matter, struggle that is unachievable or unreasonable is not fulfilling either.

**8. Run-and-tumble and the hedonic treadmill.**

*Q: What is run-and-tumble, and why does the lecture bring it up in a discussion of fulfillment?*

*A:* Run-and-tumble is how flagella-bearing bacteria climb nutrient gradients: swim straight, randomly reorient, and extend the straight runs when conditions are improving. The bacterium never perceives the gradient directly, only whether things are getting better. Humans climb gradients similarly and gain happiness from the climbing. This sets up the hedonic treadmill: some gains are transient (pay increases, a new car) because we adapt to them, while others are durable (relationships, good habits, fulfilling work).

**9. Transient versus durable happiness.**

*Q: The lecture sorts sources of happiness into two categories. Name them and give the lecture's examples of each.*

*A:* Transient: pay increases, a new car. Durable: relationships, good habits, fulfilling work.

**10. Course history (textbook chapter).**

*Q: Why was Gitlet replaced with Ngordnet, and what was the earlier structural change in 61B 2.0?*

*A:* Ngordnet is much more data-structures focused (2A: build a TimeSeries and an NgramMap; 2B: build whatever is needed for additional functionality, including implementing a graph), and GSIs voted roughly 3 to 1 to keep it over Gitlet. The earlier 61B 2.0 change came from the Fall 2014 / Spring 2015 observation that students hated splitting time between core data structures content and a huge unrelated project; the fix was to place the messy real-world project *before* the data structures material (a text editor in 2016, text-database software in 2017).

## Summary

- Engagement features are deliberate engineering: Candy Crush streaks (reset on a miss), Snapchat snap streaks, TikTok's instant autoplay and millisecond swipes. All exist to increase engagement and keep users returning.
- Every such feature has a real positive column (connection and community, habit-building, opportunities and learning) and a real negative column (addiction and time consumption, shallow interaction). The lecture asks for both, then a net judgment.
- Room verdict: TikTok, net negative ("but I will work there for $150,000/year"). Khan Academy, almost all net positive, with real criticisms (narrow education, over-specific answers, can be addictive, presents itself as a singular form of learning, curriculum carries bias).
- The same mechanic (gamification) appears as a positive for Khan Academy and a negative for TikTok. What it pulls you toward is the variable, not the mechanic.
- ByteDance: $186b revenue, $50b profit, 150,000+ employees, ~$1,240,000 per employee. Khan Academy: $0.090b donations + $0.022b programs, 227 staff (198 humans, 23 dogs, 5 cats, 1 bearded dragon), ~$590,000 or ~$60,000 per person depending on what you count.
- Reasons people pick the engagement job: money (easier to monetize suffering than success), brand and prestige, power, career doors, networking, growth, reach.
- Limiting reagent: STEM experts are historically scarce, revenue per employee runs $459,000 (Amazon) to $2.55 million (Meta). Your skills are wanted by companies, non-profits, government, and education alike, so the choice of where to point them is yours.
- Ousterhout's framing: programming may be the purest creative medium, unbound by physics, if you can imagine it you can build it. The real limit is intellectual, whether you can understand what you built.
- AI has suddenly lifted that ceiling. Timeline: 1994 Today Show, 2020 GPT-3 UI demos, 2024 o1 in the class's top 4%, 2025 o3 completing the whole 61B project, 2026 LLMs reviewing student design docs. "AI as Normal Technology" (Knight Columbia, March 2026) is cited as a counterweight on adoption speed.
- The claim is amplification of people who have the knowledge to use the tools, not replacement of knowledge. Hug's guess on societal impact: on par with electricity.
- Master of Magic: editing the save file to start rich made the game not fun. Constraints and achievable, reasonable struggle are what make life worth living.
- Run-and-tumble: bacteria climb nutrient gradients by running straight, tumbling randomly, and extending runs when things improve, using only local information. Humans climb gradients too and get happiness from the climbing.
- Hedonic treadmill: transient happiness (pay increases, a new car) fades through adaptation; durable happiness (relationships, good habits, fulfilling work) does not.
- Textbook chapter (Software Engineering IV) is a separate retrospective on 61B's own history: taught since ~Spring 1994 (CS 60C before that, back to ~1988), four modern varieties (Hilfinger, Hug, 61BL, Shewchuk/Yelick), Gitlet introduced Spring 2015 and too hard at first (no testing, no persistence tips), 61B 2.0 moved the messy project before the data structures material, and version 3.6 swapped Gitlet for the more data-structures-focused Ngordnet by a ~3-to-1 GSI vote.
