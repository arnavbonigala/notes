<!-- Mon, Nov 09, 2026 | sources: slides (no transcript available) -->
# Lecture 32: Browsers

This lecture steps outside Python to look at how web applications are built, and it does so for a reason: the Hog project in CS 61A ends with a graphical interface, and that interface runs in a browser. The core idea is that a web browser is an interpreter, just like the Python interpreter you have been studying all semester, except it interprets three languages at once: HTML (structure), CSS (appearance), and JavaScript (behavior). Parsing HTML produces the Document Object Model (DOM), a tree of elements, and JavaScript can read and mutate that tree to change what the user sees. Because users act unpredictably, web code is event driven: instead of running top to bottom and exiting, the program registers event handlers and waits for the browser to call them. The lecture then introduces the Model-View-Update (MVU) architecture, a functional discipline for managing the state of an event-driven app, traces its history from Elm (2014) through Redux (2015) to Lustre (2023), and closes with the Hog GUI, where a Gleam client in the browser talks to a Python server that simulates the game.

## Key Concepts

### A browser is an interpreter (for three languages)

You already know the shape of this idea. A Python interpreter reads source text, parses it into a structure it can evaluate, and then evaluates that structure. A browser does the same thing, but it has three languages to juggle and they each have a distinct job:

- **HTML** describes the page as a *tree of elements*. An element like `<div>` is a box, `<button>` is a clickable control. Nesting in the source text becomes parent/child structure in the tree. This should feel familiar: it is the same shape as the `Tree` abstraction you have been using, where a node has a label (the tag and its attributes) and a sequence of branches (its children).
- **CSS** describes how each element is *displayed*: size, color, font, position. CSS does not change the tree, it changes how the tree is painted.
- **JavaScript** describes the *behavior* of the page. JavaScript is the one that can run arbitrary computation, read the tree, and change it.

The lecture also notes that **Gleam compiles to JavaScript**. That is why Gleam, a statically typed functional language, can be used to write browser code at all: the browser only runs JavaScript, so Gleam is translated into JavaScript first. Gleam is the language the Hog GUI client is written in.

### The DOM is a mutable tree

Parsing HTML creates the **Document Object Model**, usually just called the DOM. The DOM is the live, in-memory tree of elements that the browser is currently displaying. Two things make it important:

1. **It is the single source of truth for what is on screen.** The browser renders the DOM, not the original HTML text. Once the page has loaded, editing the HTML source would change nothing; you have to change the DOM.
2. **It is mutable.** JavaScript can change the DOM's structure (add a child, remove an element) or its attributes (change a color, disable a button), and the page updates to match.

In 61A terms, this is a mutable tree with aliasing. The browser holds a reference to the root, your handler holds a reference to some node inside it, and mutating through either reference is visible through the other. All the reasoning you built up about mutable lists and nonlocal state applies: there is one object, many names for it, and a change through any name is a change for all of them.

### Event-driven programming

A Python script you write for a 61A lab runs top to bottom and finishes. A web application does not work that way. Web applications are **event driven**: code is executed in response to events, and the lecture gives three kinds:

- The user clicks a button.
- Some scheduled time is reached.
- New information is retrieved by the app.

Notice that you, the programmer, do not control *when* any of these happen. So you cannot write a program that says "first get the name, then get the score, then show the result." Instead you write functions, called **event handlers**, and associate each one with an element in the DOM. The handler is a function you hand to the browser, and the browser calls it later, when the matching event occurs. The handler can then change the DOM, which changes the page.

This is a **higher-order function** pattern, and it is one you have seen. You pass a function as a value to something that will call it on your behalf: exactly like passing a `key` function to `sorted`, or passing a predicate to `filter`. The twist is the timing. With `sorted`, the function you passed gets called before `sorted` returns. With an event handler, it gets called possibly minutes later, possibly never, possibly a hundred times. Control is inverted: the browser calls your code, your code does not call the browser.

The lecture's handler example is this JavaScript:

```javascript
document.addEventListener('click', (e) => {
  e.target.remove();
});
```

Read it as: register a handler on the whole document for the `'click'` event. The `(e) => { ... }` is a lambda (JavaScript's arrow function syntax) taking one parameter `e`, the event object. `e.target` is the DOM element that was actually clicked, and `.remove()` mutates the tree by deleting that element. So the whole thing is: clicking anything on this page deletes the thing you clicked. It is a one-line demonstration that handlers receive information about the event and can mutate the DOM.

### Browsers provide a component inventory, sandboxing, and accessibility

The lecture makes the point that you do not build a web app from nothing. The browser hands you a large inventory of standard application components:

- **Input:** buttons, text entry boxes, checkboxes, menus, sliders, and more.
- **Output:** text rendering, layout controls, image and video display.
- **Communication:** links between pages, and network communication with a server.
- **Device access:** local storage, camera and microphone, location.

Two properties of this platform are worth internalizing because they are part of why the web is the delivery mechanism for something like a Hog GUI:

- **Pages are sandboxed.** A page cannot read your files or use your camera without permission. Compare this to running a downloaded Python script, which has whatever access your user account has. The browser is an interpreter that deliberately refuses to expose most of the machine to the code it is interpreting. That is what makes it safe to open an arbitrary link.
- **Interface components are accessible.** The standard components support screen readers and different input methods. Using a real `<button>` gets you keyboard focus, keyboard activation, and a screen reader announcement for free; hand-rolling a clickable `<div>` gets you none of that.

### Model, View, Update

Event handlers that directly mutate the DOM work fine for one line of JavaScript. They scale badly. Once a dozen handlers are each poking at different parts of the tree, the state of the application lives scattered across the DOM, and no single place in your code tells you what the app currently *is*. The MVU architecture exists to fix this. It has three pieces:

- **The model** is a data structure describing the *entire logical state* of the application. Not part of it, all of it.
- **The view** is a *function of the model* that returns the tree of elements to display. Same model in, same tree out. The view does not mutate anything and does not hold any state of its own.
- **The update** function takes the current model and a message, and returns an *updated model*. It does not edit the page.

The cycle the lecture describes is: to change something, you update the model, then the framework calls `view`, then the framework edits the DOM efficiently. That last clause matters. You write a function that produces a whole description of the page, as if from scratch, and the framework takes responsibility for diffing it against the current DOM and making only the changes that are needed. You get the simplicity of "recompute everything" with the performance of targeted mutation.

Messages are how user actions get back into the system. **The view annotates each element with the message that will be sent when a user interacts with it.** The lecture's example: when a user clicks this save button, send the `UserClickedSave` message. So the view is not just producing boxes and text, it is wiring each interactive element to a message value.

Finally, **the update function can also return follow-on effects from a message.** For a `UserClickedSave` message, the follow-on effect would be a network request. And **effects can send additional messages, which are then passed to update.** This closes the loop: the server's response arrives as another message, `update` handles it, the model changes, the view runs again.

The reason this is a big deal conceptually: `view` and `update` are pure functions in the sense 61A cares about. `view` is a function of the model with no side effects. `update` is a function of the model and a message that returns a new model. All the unpredictable, stateful, side-effecting parts (when clicks happen, when the network replies, how the DOM gets edited) are pushed into the framework. Your code is functions on data structures, which is exactly the kind of code you can reason about and test.

### Why "functional" web programming has a history

The lecture gives a short timeline:

- **2014: The Elm Architecture (TEA).** Introduced the model-view-update pattern. Purely functional. The stated motivation is striking: *no runtime exceptions.* If the language's type system can guarantee that `view` and `update` handle every case of your model and message types, the app cannot crash at runtime with the web equivalent of a `TypeError`.
- **2015: Redux.** Introduced at Facebook to help with scaling. Adapted the Elm framework to work within the JavaScript and React ecosystem. This is the pattern meeting the industry: same idea, but living inside the dominant JavaScript tooling.
- **2023: Lustre.** A modernized implementation of The Elm Architecture, in Gleam.

The thing to take from the timeline is that MVU is not a quirk of one framework. It was invented to solve a real problem (apps whose state was scattered and whose failures were unpredictable), and it was adopted and reimplemented repeatedly because it solved it.

### The Lustre framework

Lustre is the framework the Hog GUI client uses. The lecture's characterization of it is mostly about how *little* of it is web-specific:

- **The model doesn't use any web-specific features; it's just a regular Gleam type.** Your application state is an ordinary data structure.
- **A message doesn't use any web-specific features; it's just a regular Gleam type.** Messages are ordinary values, typically a tagged union of the things that can happen.
- **The view function uses Lustre-specific Gleam functions** that create: elements and text nodes, attributes that style elements, and attributes that describe the events that trigger messages sent to `update`. So the view is where the web shows up, and even there it shows up as ordinary function calls that build a tree.
- **The update function is almost just regular Gleam**, but also includes Lustre-specific effects.

The documentation link given is `https://lustre.hexdocs.pm/`.

### The Hog GUI: splitting work across client and server

The Hog GUI uses **Python for the game simulation and Gleam for the web interface**. The division of labor the lecture lays out:

| Client (Gleam, in the browser) | Server (Python) |
| --- | --- |
| 1. Configure a game | 1. Randomly roll dice |
| 2. Let human players take turns | 2. Simulate a game |
| 3. Show outcomes | 3. Computer strategies |

The client is responsible for everything the user touches: setting up the game, collecting the human player's decisions, displaying results. The server owns the game logic you wrote in the Hog project: dice rolls, `play`, and the strategy functions.

This matters for understanding why the architecture looks the way it does. When a human player takes a turn, the client cannot compute the outcome, the server has to. So the click produces a message, `update` handles that message by returning a follow-on effect (a network request to the Python server), and later the server's response arrives as another message, which `update` uses to produce a new model, which `view` renders. Every one of the MVU pieces is doing exactly the job it was described as doing.

## Definitions

- **Web browser:** An interpreter for three languages: HTML, CSS, and JavaScript, which combine to create a page.
- **HTML:** A language that describes a page as a tree of elements, such as `<div>` (a box) and `<button>`.
- **CSS:** A language that describes how each element is displayed: its size, color, font, and position.
- **JavaScript:** A language that describes the behavior of the page. JavaScript accesses the DOM and can change its structure or attributes, changing the page.
- **Document Object Model (DOM):** The tree of elements created by parsing HTML. It is what the browser renders and what JavaScript reads and mutates.
- **Event-driven programming:** A style in which code is executed based on user actions and other events, rather than running straight through from start to finish.
- **Event:** Something that happens which code can respond to: a user clicks a button, a scheduled time is reached, new information is retrieved by the app.
- **Event handler:** A JavaScript function associated with an element in the DOM. When the event occurs, the browser calls the handler, which can change the DOM.
- **Sandboxing:** The property that pages can't read your files or use your camera without permission.
- **Accessibility (of interface components):** The property that standard browser interface components support screen readers and different input methods.
- **Model:** A data structure describing the entire logical state of the application.
- **View:** A function of the model that returns the tree of elements to display.
- **Update:** A function that takes the current model and a message, and returns an updated model. It can also return follow-on effects.
- **Message:** A value representing something that happened. The view annotates each element with the message that will be sent when a user interacts with it (e.g., `UserClickedSave`).
- **Effect (follow-on effect):** Something an update returns in addition to the new model, such as a network request. Effects can send additional messages, which are then passed to update.
- **The Elm Architecture (TEA):** The 2014, purely functional architecture that introduced the model-view-update pattern, motivated by eliminating runtime exceptions.
- **Redux:** A 2015 framework introduced at Facebook to help with scaling, which adapted the Elm framework to work within the JavaScript and React ecosystem.
- **Lustre:** A 2023 modernized implementation of The Elm Architecture, in Gleam.
- **Gleam:** The language used for the Hog GUI client. Gleam compiles to JavaScript, which is why it can run in a browser.

## Worked Examples

The lecture's own code is one line of JavaScript, so the examples below work through that line directly, then translate the lecture's architectural ideas into Python so you can reason about them with the environment and object models you already know. The Python translations are labeled as **(extra context)**, since the lecture presented these ideas in JavaScript and Gleam, not Python.

### Example 1: The lecture's event handler, line by line

```javascript
document.addEventListener('click', (e) => {
  e.target.remove();
});
```

Step by step:

1. `document` is the object representing the whole page, the root of the DOM tree.
2. `addEventListener` is a method on it. It takes two arguments: the name of an event (`'click'`) and a function to call when that event happens.
3. `(e) => { e.target.remove(); }` is JavaScript's lambda syntax. In Python it would be written `lambda e: e.target.remove()`. It takes one parameter, `e`, which is the event object the browser will pass in.
4. Calling `addEventListener` **registers** the function. Nothing visible happens. The function is not called here. The browser stores a reference to it.
5. Later, when the user clicks, the browser constructs an event object and calls the stored function with it.
6. Inside the call, `e.target` is the specific DOM element the user clicked. `.remove()` mutates the DOM by removing that element from its parent's children.
7. The browser re-renders, and the clicked thing is gone.

The critical observation, and the one most likely to be tested: **registering a handler and running a handler are two different moments.** The frame for the lambda's body is not created at line 1; it is created each time a click happens. If the user clicks four times, four frames are created, one after another, each with its own `e`.

### Example 2 (extra context): Why handlers need the inverted-control shape, in Python

Suppose you wanted to write a dice game as a straight-line Python program:

```python
def play_turn_blocking():
    num_rolls = int(input('How many dice? '))
    score = roll_dice(num_rolls)
    print('You scored', score)
```

This works in a terminal because `input` **blocks**: the program stops and waits. A browser cannot do this. If JavaScript blocked waiting for a click, the whole page would freeze: no scrolling, no other buttons, no rendering. So the browser offers no blocking `input`. Instead you register a handler:

```python
def on_roll_clicked(event):
    num_rolls = read_number_from_text_box()
    score = roll_dice(num_rolls)
    display(score)

register('click', roll_button, on_roll_clicked)
```

Trace the environment for the registration call. `register` is called with three arguments; one of them is the *function value* `on_roll_clicked`, not a call to it. `register` stores that value somewhere and returns. The global frame now has a name `on_roll_clicked` bound to a function, and the browser's internal table also holds that same function object. Two names, one function.

Nothing in `on_roll_clicked`'s body has executed. No local frame for it exists. It executes only when the browser calls it, and then a frame is created with `event` bound to whatever the browser passed, and the parent of that frame is the global frame (because the function was defined there), **not** the frame of whatever browser code happened to call it. This is the lexical-scoping rule you already know, and it is exactly why handlers can reliably see the names that were in scope where they were defined.

### Example 3 (extra context): Model, View, Update written out in Python

MVU translates cleanly into Python because its three pieces are just functions on data. Here is a minimal counter.

```python
def init():
    return {'count': 0}

def update(model, message):
    if message == 'Increment':
        return {'count': model['count'] + 1}
    elif message == 'Decrement':
        return {'count': model['count'] - 1}
    elif message == 'Reset':
        return {'count': 0}
    return model

def view(model):
    return ['div',
            ['button', {'on_click': 'Decrement'}, '-'],
            ['span', str(model['count'])],
            ['button', {'on_click': 'Increment'}, '+']]
```

What each piece is doing, and why:

- `init` produces the starting model. The model here is one dictionary with one key, because the entire logical state of a counter is one number. If this app also had a name field and a "has the user saved yet" flag, those would go in the same model, not in separate globals and not in the DOM.
- `update` takes a model and a message and **returns a new model**. Look at what it does not do: it never touches the page, never calls `view`, and never mutates its `model` argument. `{'count': model['count'] + 1}` builds a fresh dictionary. In box-and-pointer terms, the old model object is left untouched and a new object is created; the framework will rebind its reference to the new one. This is why you can take any `(model, message)` pair and test `update` in isolation, and why the framework can keep old models around for features like undo.
- `view` takes the model and returns a *description* of a tree. The nested lists are the tree: a `div` whose branches are a button, a span, and a button. Notice `{'on_click': 'Decrement'}`: this is the lecture's point that *the view annotates each element with the message that will be sent when a user interacts.* The button does not contain code to decrement. It contains the message `'Decrement'`, and `update` is the only place that knows what that message means.

Now trace the full cycle after the user clicks `+` when the count is 0:

1. The framework looks up the clicked element's annotation and finds the message `'Increment'`.
2. It calls `update(current_model, 'Increment')`. A frame is created with `model` bound to `{'count': 0}` and `message` bound to `'Increment'`. The first branch matches, and a new dictionary `{'count': 1}` is constructed and returned.
3. The framework replaces the current model with that new dictionary.
4. The framework calls `view({'count': 1})`, which returns a new tree where the span's text is `'1'` instead of `'0'`.
5. The framework compares that tree against the DOM, sees that only the span's text differs, and edits only that text node. The two buttons are left alone, which is the "edits the DOM efficiently" step.

Step 5 is the part you do not write. It is the entire value proposition of the framework: you describe the page as a pure function of state, and the framework handles the mutation.

### Example 4 (extra context): An update that returns a follow-on effect

The lecture says `update` can also return follow-on effects, and that effects can send additional messages which are passed back to `update`. Here is that shape in Python, written as a Hog turn that must ask the server to roll.

```python
def update(model, message):
    if message[0] == 'UserClickedRoll':
        num_rolls = message[1]
        new_model = {**model, 'waiting': True}
        effect = ('http_post', '/roll', {'n': num_rolls}, 'ServerRolled')
        return new_model, effect
    elif message[0] == 'ServerRolled':
        points = message[1]
        new_model = {**model,
                     'waiting': False,
                     'score': model['score'] + points}
        return new_model, None
    return model, None
```

Walk the sequence of events for one human turn in the Hog GUI:

1. The user clicks Roll. The view had annotated that button with `('UserClickedRoll', 3)`.
2. The framework calls `update`. The first branch runs. It returns two things: a new model with `waiting` set to `True`, and an **effect**, a description of a network request to the Python server.
3. The framework re-renders. Because the model says `waiting` is `True`, `view` can return a tree with the button disabled and a spinner showing. The page stays responsive the whole time.
4. The framework performs the effect: it sends the request to the Python server. The Python side rolls the dice and simulates, because that is the server's job in the Hog GUI split.
5. The response arrives, maybe 50 milliseconds later. The framework packages it as the message `('ServerRolled', 8)`, the message name the effect specified.
6. The framework calls `update` again with that message. The second branch runs, clears `waiting`, adds the points, and returns a new model with no effect.
7. `view` runs once more and the new score appears.

The thing to notice: `update` ran twice, and both times it was an ordinary function from `(model, message)` to a new model. It never waited for the network. It never knew how the request was actually sent. All the asynchrony lives in the framework, between steps 4 and 5. This is why the effect is *returned as data* rather than performed directly: a function that returns a description of a request is still a pure function and is still testable, while a function that performs one is neither.

## Common Pitfalls

- **Thinking the HTML source is what you change.** Editing the original HTML text after load changes nothing on screen. The browser renders the DOM. JavaScript changes the DOM.
- **Confusing registering a handler with calling it.** `addEventListener('click', f)` does not call `f`. It hands the browser a reference to `f` for later. Passing `f()` instead of `f` calls the function immediately and registers whatever it returned, which is a classic bug and a natural exam target given 61A's emphasis on functions as values.
- **Expecting blocking input.** There is no browser equivalent of sitting at an `input()` call waiting for the user. If your mental model is "then ask the user," you will not be able to write an event-driven app. The correct model is "register a handler and return."
- **Putting application state in the DOM instead of in the model.** Reading the current score out of a text node, or tracking whose turn it is by which button is highlighted, means the logical state is scattered through the page. The MVU rule is that the model describes the *entire* logical state, and the view is derived from it, never the other way around.
- **Mutating the model inside `update` instead of returning a new one.** `update` is defined as a function that takes the current model and a message and *returns an updated model*. Mutating in place breaks the property that makes the architecture work.
- **Calling `view` yourself, or editing the DOM from `update`.** The cycle is: update the model, then the *framework* calls `view`, then the *framework* edits the DOM. Reaching into the DOM from `update` sidesteps the whole architecture.
- **Putting behavior in the view.** The view attaches a *message* to an element, not a block of logic. If your view is deciding what should happen on a click, that decision belongs in `update`.
- **Thinking a whole-page `view` must be slow.** It would be, if the browser re-rendered everything. It is not, because the framework diffs the returned tree against the DOM and edits only what changed.
- **Mixing up which side of the Hog GUI does what.** Dice rolling, game simulation, and computer strategies are on the Python server. Configuring the game, taking human turns, and showing outcomes are on the Gleam client.
- **Hand-rolling interface components.** A clickable `<div>` is not a `<button>`. The lecture's point about accessibility is that the standard components already support screen readers and different input methods; custom substitutes lose that.
- **Treating MVU as a Lustre invention.** The pattern comes from Elm in 2014. Redux adapted it for JavaScript and React in 2015. Lustre is a 2023 implementation of the same architecture in Gleam.

## Likely Exam Points

**1. The three languages and their roles.**

*Practice:* A page displays a red button that, when clicked, adds a new row to a list. Name which of HTML, CSS, and JavaScript is responsible for each of: the button existing in the page's structure, the button being red, and the row being added on click.

*Answer:* HTML describes the structure, so the button's existence as an element in the tree is HTML. CSS describes how elements are displayed, so the red color is CSS. JavaScript describes the behavior of the page and can change the DOM's structure, so adding the row on click is JavaScript.

**2. What the DOM is and how it relates to HTML.**

*Practice:* In one sentence, what is the relationship between HTML and the DOM?

*Answer:* Parsing HTML creates the DOM, which is the tree of elements the browser renders; JavaScript then accesses that tree and can change its structure or attributes, changing the page.

**3. Reading the lecture's event handler.**

*Practice:* What does the following do, and at what moment is the arrow function called?

```javascript
document.addEventListener('click', (e) => {
  e.target.remove();
});
```

*Answer:* It registers a click handler on the document, so that clicking any element removes that element from the DOM (`e.target` is the clicked element, `.remove()` deletes it). The arrow function is not called when `addEventListener` runs; it is called by the browser each time a click event occurs, once per click.

**4. Handlers as function values (a higher-order function question in web clothing).**

*Practice:* What is wrong with `button.addEventListener('click', handle_click())`?

*Answer:* The parentheses call `handle_click` immediately and register its return value, rather than registering the function itself. The handler must be passed as a function value, `handle_click`, so the browser can call it later when the event occurs.

**5. Why web applications are event driven.**

*Practice:* Give two kinds of events besides a button click that can cause code to run in a web application.

*Answer:* A scheduled time being reached, and new information being retrieved by the app.

**6. The three parts of MVU and their types.**

*Practice:* State what each of model, view, and update is, including what view and update take and return.

*Answer:* The model is a data structure describing the entire logical state of the application. The view is a function of the model that returns the tree of elements to display. The update function takes the current model and a message and returns an updated model, and it can also return follow-on effects.

**7. The order of operations when state changes.**

*Practice:* A user clicks a button. Put these in order: the framework edits the DOM, the framework calls `view`, the model is updated.

*Answer:* The model is updated, then the framework calls `view`, then the framework edits the DOM efficiently.

**8. What the view attaches to an element.**

*Practice:* When a user clicks a save button, what did the view put on that element so the app knows what to do?

*Answer:* The view annotated the element with the message to send when the user interacts with it, for example `UserClickedSave`. That message is then passed to the update function, which decides what happens.

**9. Effects and the loop they close.**

*Practice:* A `UserClickedSave` message arrives. What can `update` return besides the new model, and what happens next?

*Answer:* It can return a follow-on effect, here a network request. Effects can send additional messages, which are then passed to `update`, so the server's response comes back in as another message that produces another updated model.

**10. The history and its motivation.**

*Practice:* Which framework introduced model-view-update, in what year, and what was the motivation?

*Answer:* The Elm Architecture, in 2014. It was purely functional, and the motivation was no runtime exceptions. Redux (2015, at Facebook) adapted it to the JavaScript and React ecosystem to help with scaling, and Lustre (2023) is a modernized implementation of the same architecture in Gleam.

**11. What is and is not web-specific in Lustre.**

*Practice:* Which of the model, the message, the view function, and the update function use Lustre-specific features?

*Answer:* The model and the message are just regular Gleam types with no web-specific features. The view function uses Lustre-specific Gleam functions that create elements, text nodes, styling attributes, and attributes describing which events trigger which messages. The update function is almost just regular Gleam, but also includes Lustre-specific effects.

**12. The Hog GUI client/server split.**

*Practice:* In the Hog GUI, which side handles dice rolling and computer strategies, and which side handles letting human players take turns?

*Answer:* The Python server randomly rolls dice, simulates a game, and runs computer strategies. The Gleam client configures a game, lets human players take turns, and shows outcomes.

**13. Sandboxing.**

*Practice:* Why is it reasonably safe to open an unfamiliar web page, but risky to run an unfamiliar downloaded program?

*Answer:* Pages are sandboxed: they can't read your files or use your camera without permission. A downloaded program has no such restriction.

**14. Why Gleam can run in a browser at all.**

*Practice:* A browser interprets HTML, CSS, and JavaScript. How does Gleam code end up running in one?

*Answer:* Gleam compiles to JavaScript.

## Summary

- A web browser is an **interpreter for three languages**: HTML (a page as a tree of elements such as `<div>` and `<button>`), CSS (how each element is displayed: size, color, font, position), and JavaScript (the behavior of the page). Gleam compiles to JavaScript.
- Parsing HTML creates the **DOM**, the tree of elements. CSS styles elements. JavaScript accesses the DOM and can change its structure or attributes, changing the page.
- Web applications are **event driven**: code runs in response to events such as a user clicking a button, a scheduled time being reached, or new information being retrieved.
- **Event handlers** are JavaScript functions associated with DOM elements. When an event occurs, the browser calls the handler, which can change the DOM. `document.addEventListener('click', (e) => { e.target.remove(); })` removes whatever element the user clicks.
- Registering a handler is not calling it: handlers are function values the browser calls later, which inverts control compared to a straight-line script.
- Browsers supply a large inventory of standard components for **input** (buttons, text boxes, checkboxes, menus, sliders), **output** (text, layout, image and video), **communication** (links, network), and **device use** (storage, camera and microphone, location).
- Pages are **sandboxed** (no file or camera access without permission), and standard interface components are **accessible** (screen readers, different input methods).
- **MVU:** the **model** is a data structure describing the entire logical state; the **view** is a function of the model returning the tree of elements to display; the **update** function takes the current model and a message and returns an updated model.
- To change the page you update the model, then the framework calls `view`, then the framework edits the DOM efficiently.
- The **view annotates each element with the message** to send on interaction (for example, `UserClickedSave`). `update` can also return **follow-on effects** such as a network request, and effects can send additional messages back into `update`.
- History: **Elm (2014)** introduced MVU, purely functional, motivated by no runtime exceptions. **Redux (2015, Facebook)** adapted it for the JavaScript and React ecosystem to help with scaling. **Lustre (2023)** is a modernized implementation of the Elm Architecture in Gleam (`https://lustre.hexdocs.pm/`).
- In **Lustre**, the model and messages are plain Gleam types; only the view (elements, text nodes, style and event attributes) and the update function's effects are Lustre-specific.
- **Hog GUI:** Gleam client configures the game, lets human players take turns, and shows outcomes; Python server rolls dice, simulates games, and runs computer strategies.
