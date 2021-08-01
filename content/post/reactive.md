---
title: "Exploring reactive UI"
date: 2021-08-01T13:49:36+10:00
draft: false
---

## Past

Many are familiar with the practicality of the Qt API. It abides by what most already know of object-oriented programming and manually mutated states. By far the biggest advantage of such an API is its clarity; there is nothing to learn or understand as what you write is exactly what you mean.

```cpp
int state = 5;

// ...

QPushButton* add = new QPushButton(this);
add->setLayout(top);

display->setValue(state);

connect(add, &QPushButton::pressed, [&]() {
	state += this->spinBox->value();
	display->setValue(state);
});
```

Already we see a flaw however. We are duplicating procedures. `display->setValue` must be called twice in order to have the UI behave as we want.

This problem is really the consequence of two concerns in GUI APIs:

- Initialization/update phases
- Lack of a single source of truth

The former issue is less of a concern, however the latter is strikingly critical. There is some widget `display` that depends on our data and as a result we must propagate any mutations to that data. What we have done - and you may not have realized it - is write an **implied dependency** between `state` and some data inside `display`. In fact, we do this kind of thing all the time without realizing it.

Unfortunately, this implied dependency is brittle. It can easily be shattered by any third party that inconsistently mutates `state`/`display`, being completely unaware of the **unspoken link** (i.e., implied dependency) between `state` and `display` that we wrote here.

Whose fault would that be? I argue it's the API's fault.

## Present

Thankfully, we already figured this one out; let's keep the dependencies implicit but instead lock it down under a declarative interface.

A prime, and maybe familiar example, is React.

```js
function Stateful() {
	const [state, setState] = useState(5);
	const [spinBox, setSpinBox] = useState(0);

	return (
		// ...
		<button onClick={() => setState(state + spinBox)}>
		<Display value={state}>
		// ...
	);
}
```

The UI system still has no clue that `display` and `state` are linked, yet our code (and in particular our button callback) is more concise. No initialization phase needed!

So how is black magic like this possible? Simple; it's all phony. Of course there is no underlying update procedure being generated, how could there be one if React doesn't even know the state dependencies! No, instead React simply forgets about trying to keep track of all that and just recreates our UI tree every time something happens, then generates stable identities to remember the previous frame's state (which is all wrapped up and abstracted away into a single `useState`).

Fairly inefficient at runtime, yet UIs are far more expressive with React. As a side bonus, the UI tree is declarative and thus shifting around the UI structure dynamically is a breeze compared to something like Qt.

I also include Dear ImGui and its kin in this category. If anything, Dear ImGui is even more dynamic than React since it doesn't even bother to do any diffing.

## Explicit dependencies

Now we venture out into the unknown in an attempt to scope out possible solutions. We've seen two examples of the opposite extremas of GUI APIs:

- Qt with its highly imperative and object-oriented procedures, proving to be conservative and practical
- React with an expressive and declarative interface at the cost of performance

It's fair to say then that the ideal solution is a declarative interface, like React, that is performant, like Qt. The key to such a GUI API is something neither Qt, React, or any other GUI library like them have: **Explicit dependencies**.

More specifically, an explicit dependency *graph* - in other words, a directed acyclic graph between all state.

All the GUIs we write will always have a dependency graph, hiding in plain sight, but they are never written out. You never write down that two states depend on each other. You only describe the relationship through mutating procedures such as callbacks. Although to be fair, you probably don't want to write out those dependencies by hand either - and you don't have to.

The idea is simple: GUIs should be a metaprogram which encode dependencies and mutations, after which the UI system steps in, builds an explicit dependency graph, and writes the actual GUI program with all the dependencies resolved back down to an implied form.

In fact, Svelte already does exactly this using a precompiler.

Here's a rough sketch of exactly what I mean. Let's start with the "metaprogram" of the UI. This will just be our React code (yup, the UI metaprogram is identical to regular UI code).

```js
function Stateful() {
	const [state, setState] = useState(5);
	const [spinBox, setSpinBox] = useState(0);

	return (
		// ...
		<button onClick={() => setState(state + spinBox)}>
		<Display value={state}>
		// ...
	);
}
```

To really understand what I mean by "UI metaprogram", let's look at it from a different perspective. Instead of seeing the returned UI tree as something to be recreated every frame, you should view it as a static *template*, waiting to be filled with real data. Instead of seeing `useState` as a call to grab the cached state, see it as a node in the dependency graph. Finally, consider the properties (e.g., `value={state}`) to be a directed edge from `state` to `value`.

That's it. That's all the UI system would need to generate retained-mode UI code. Let me rewrite it in a more step-by-step sense:

```js
function Stateful(graph) {
	var w = graph.addWidget();

	graph.addNode(w, "state", 5);
	graph.addNode(w, "spinBox", 0);

	var btn = button(graph);
	var display = Display(graph);

	graph.addReaction(btn, "onClick", graph.mutate(w, "state", (state, graph) => state + graph.get(w, "spinBox")));
	graph.addDependency(display, "value", w, "state");

	return w;
}
```

For example, if this `graph` was an interface to a UI precompiler like Svelte, then it may output the following.

```js
var state = 5;   // given by graph.addNode
var spinBox = 0; // given by graph.addNode

var btn = new Button();
var display = new Display();

btn.onClick = () => { // given by graph.addReaction
	state = state + spinBox; // given by graph.mutate
	display.value = state;   // given by graph.addDependency
};
```

The real challenge comes from doing this *without* a precompiler.

### Black box

One way of finding mutating code paths from declarative code is by running "tracers" through the function. If it were a black box (which it effectively *is*), then it can be reverse engineered by tweaking inputs and observing outputs.

Say there is a mysterious function, `mystery`, which takes application state and produces a UI tree. To make this easier, assume we're given two additional helper functions; `state_muts` (which produces all the possible mutations of the state), and `find_mutation` (which diffs the UI tree to find and encode the exact UI tree mutations that occurred).

```rust
let mut state = State::default();
let mut ui = mystery(&state);
let mut lut = HashMap::<StateMutation, UiMutation>::new();
for state_mut in state_muts() {
	state_mut.patch(state);
	let new_ui = mystery(state);
	lut.insert(state_mut, find_mutation(&ui, &new_ui));
	ui = new_ui;
}
```

By doing this, we have built a look-up table, `lut`, which maps state mutations to UI mutations. Having this LUT means that whenever the state does change, we don't need to run the entire process again since we can simply consult this LUT to find how we should change the UI correspondingly. This is something of a differential of `mystery`; small changes to the input to observe the small change to the output.

Unfortunately this notion only works in the abstract sense. It would be quite elegant if this was indeed reduced to a calculus problem, but such UI mutations are veiled behind string labels and other widget properties. To make this work, it would require a property system - something to direly avoid (at least on a library-level).

### Maps

Many parts of a UI fall down to a simple mapping, `State -> UI`, `Number -> Formatted String`, `Formatted String -> Label`, etc. We can leverage this and formalize the notion that state is derived off of other state.

```rust
let count = State::new(0);
let formatted_string = count.map(|count| format!("Count: {}", count));
label.set_label(formatted_string);
```

This enables us to write the UI in whatever manner we desire (imperative or declarative), yet it will be able to update itself without any additional reactivity by us. This solution still poses some problems:

- State mappings must persistently hold a reference to the state they map.
- State mappings should be lazily evaluated (otherwise we're back to square one in terms of performance).

The first problem can be solved with `Rc<Cell<T>>`, and as much as I despise the use of interior mutability in favour of letting data flow in the correct direction, state mappings are a graph and graphs have no single linear order.

The second problem is solved by using a semaphore-like system where each state holds the number of mutations, and so does each mapping. Whenever those numbers fall out of sync then the mapping must be re-evaluated.

Take note of what we've done here. Dependencies are now crystal clear. By just looking at the code you can be certain that e.g., the label text is entirely dependent on `count`, because of the `count.map(...)`.

[Here's a proof-of-concept implementation in Rust](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=e885b458ab41ed09792bf766615c93f7).

## Getting wild with async

Stepping away from the matter of dependencies for a second, let’s explore a more novel approach to reactive UI as a whole.

This one is quite exotic and I haven't seen it done before. The core idea is to have each widget be an async task which polls its in-going message channel and can spawn sub-tasks to act as child widgets. This implies that every widget essentially has its own event loop, rather than a single application-wide event loop for all widgets.

A possible interface could like this.

```rust
async fn my_widget(rcv: Receiver, out: Rc<Sender>) {
	let mut count = 0;
	while let Some(event) = rcv.next().await {
		match event {
			Event::Update(my_props) => /* update props */,
			Event::Msg(my_msg) => match my_msg {
				Msg::Incr => { count += 1; out.render(); },
				Msg::Decr => { count -= 1; out.render(); },
			},
			Event::Render(to) => {
				to.send(|| {
					WidgetTask::new(column, (
						(label, "Increment"),
						(button, || out.send_msg(Msg::Incr)), // out sends to rcv
						(button, || out.send_msg(Msg::Decr)),
					))
				});
			}
		}
	}
}
```

Under the hood it would presumably use diffing, yet the diffing can be much more targeted as you can signal directly into a widget task to tell it to update itself. You can also set up a feedback loop to let yourself know of an external event. Moreover, using channels means you can set up any communication pipeline you want between any two widgets. I haven't explored the limits of this kind of messaging and eventing system yet.

On the other hand, the interface above is verbose and ugly so it's a much better idea to abstract the core message loop away and expose programmable parts of it with an interface.

```rust
enum Msg {
	Incr,
	Decr,
}

struct Counter {
	count: i32,
}

#[async_trait]
impl Widget for Counter {
	// Properties are the public interface of the widget (in place of setters).
	type Props = ();
	type Message = Msg;

	// Called on spawn
	async fn new(_: Self::Props) -> Self {
		Counter {
			count: 0,
		}
	}

	// Called on diff
	async fn reduce(&mut self, _: Self::Props) {}

	async fn update(&mut self, msg: &Msg) {
		match msg {
			Msg::Incr => self.count += 1,
			Msg::Decr => self.count -= 1,
		}
	}

	// This is only called after Self::update.
	// Since this only returns a tree of properties, the widget driver can either:
	// 1) Spawn the tasks if this is the first render.
	// 2) Diff and call `reduce` on the existing tree.
	async fn view(&self, out: MessageOut<Msg>) -> Column {
		// These functions return (Self::Props, Self::spawn)
		Column::with_children([
			Label::with_text(format!("Count: {}", self.count)),
			Row::with_children([
				Button::with_child(Label::with_text("Increment"))
					.on_click(|_| out.send(Msg::Incr)), // Eventually invokes Self::update
				Button::with_child(Label::with_text("Decrement"))
					.on_click(|_| out.send(Msg::Decr)),
			]),
		])
	}
}
```

It shows strong resemblance to the Elm architecture, however in this example `Msg` can be piped directly into the widget's message loop rather than cascading the messages down the view hierarchy. Therefore, the exact widget branch that was updated can be identified and diffed instead of blindly diffing the entire UI tree.

Some things to investigate:

- Event order
- Layout and rendering
- Performance

## Compromising

I think that reactive UI is quite elegant. It welcomes curious-minded exploration, functional purity, and theoretical analysis. However, I think that it appears so elegant that some are blinded by its applicability. Reactive declarative UI code looks beautiful when writing a counter, but what about a more "ugly" application? Something much more heavy-duty? Say, a video editor, or DAW, or drawing software. How will you map any "pure" data down?

The problem here is that the core of these kinds of applications is actually incredibly unrelated to the UI. A DAW's core is just a DSP. A video editor's core is mostly just a compositor. Blender's mesh editor core comes down to a 3D scene stored in a few vertex buffers. Obviously a DAW isn't just an interface to a DSP, it's a big shell to add VSTs, MIDI, drums, etc. A video editor isn't just an interface to a compositor, it's a way to blend VFX, 3D scenes, and A/V. Then most obviously, Blender mesh editor isn't just an interface to upload bytes to a vertex/index buffer.

All the value of these kinds of applications comes down to the UI and as such there's practically no way to bridge the gap between the core and the UI without introducing a thick intermediate layer that has state only for the sake of the UI. That is something that reactive UI does not appreciate.

Reactive UI works extremely well where there's a clear and direct connection between the UI and state. For example, a counter state maps directly to an incrementing button and a formatted label. However, a list of audio tracks does not quite map to a DAW.

There's a time and place for reactive and declarative UI, and there's a time and place for procedural and imperative UI. I advocate for a UI platform that allows you to mix the two seamlessly such that one can use imperative UI for the nitty-gritty widgets and declarative UI for the composition of widgets inside forms/panels. In fact, this idea is not so alien. Iced, for example, does exactly this, wherein the elementary building blocks of the declarative UI are irreducible widgets.

## Closing thoughts

There is lots of room for playing around when it comes to reactive UI. In particular, I'm a fan of [Raph Linus's blog](https://raphlinus.github.io/) where he has made several posts exploring this very topic. I hope to see more work and movement occur in this space as it may very well shape the future of GUI libraries.