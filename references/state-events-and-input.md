# State, events & input

## Entity operations

```rust
let counter: Entity<Counter> = cx.new(|cx| Counter::new(cx));

let n = counter.read(cx).count;
counter.update(cx, |this, cx| { this.count += 1; cx.notify(); });

let weak = counter.downgrade();
weak.update(cx, |this, cx| { /* ... */ }).ok();   // Result
```

`WeakEntity::update` / `Entity::update` return `Result`; `.ok()` is the common idiom inside
callbacks where the entity may already be gone.

## Event listeners inside a view

Use `cx.listener(...)` to get `&mut self` back inside an element callback:

```rust
div()
    .id("btn")
    .on_click(cx.listener(|this, _event: &ClickEvent, _window, cx| {
        this.count += 1;
        cx.notify();
    }))
```

> Interactive methods only exist on **stateful** elements. Add `.id(...)` — any
> `impl Into<ElementId>`: `&str`, `usize`, or a tuple like `("row", ix)`. Without it you get a
> "no method named `on_click`" compile error.

`cx.listener` needs `&Context<T>`. For closures that must not borrow the context, build the
handler manually via `window.listener_for(&entity, ...)` or `window.handler_for(...)`.

## Common event handlers

| Category | Methods |
| --- | --- |
| Click | `on_click`, `on_aux_click`; `ClickEvent::click_count()` distinguishes double/triple |
| Hover | `on_hover(|this, &bool, window, cx|)` (bool = entered/left) |
| Mouse | `on_mouse_down(MouseButton::Left, ..)`, `on_mouse_up`, `on_mouse_move`, `on_mouse_exit`, `on_mouse_pressure` |
| Keyboard | `on_key_down`, `on_key_up`, `on_modifiers_changed` |
| Scroll / pinch | `on_scroll_wheel`, `on_pinch` |
| Drag & drop | `on_drag(data, \|data, position, window, cx\| cx.new(...))`, `on_drop(\|this, data, window, cx\|)` |
| Focus | `on_focus`, `on_blur`, `on_focus_in`, `on_focus_out` |
| Actions | `on_action(cx.listener(Self::handler))`, `capture_action` |

Drag example (make a source and a target):

```rust
#[derive(Clone)]
struct DragData { index: usize }

div()
    .id(("item", i))
    .on_drag(DragData { index: i }, |data, position, _window, cx| {
        cx.new(|_| DragPreview { data: data.clone(), position })
    })

div()
    .id("drop-zone")
    .on_drop(cx.listener(|this, data: &DragData, _window, cx| {
        this.dropped = Some(data.index);
        cx.notify();
    }))
```

## Focus

```rust
// Own a focus handle on the view
let focus_handle = cx.focus_handle();
window.focus(&focus_handle, cx);

div()
    .id("editor")
    .track_focus(&self.focus_handle)   // makes the subtree focusable and routes keys here
    .key_context("Editor")
    .on_key_down(cx.listener(Self::on_key))
```

Tab order:

```rust
let h = cx.focus_handle().tab_index(1).tab_stop(true);
window.focus_next(cx);
window.focus_prev(cx);
```

Inside `Context<T>`: `cx.focus_self()`, `cx.on_focus(&handle, window, ...)`,
`cx.on_focus_out(...)`, `cx.on_blur(...)`.

## Actions & keybindings

Actions are the keyboard-driven command system. Declare unit actions with `actions!`:

```rust
use gpui::{actions, KeyBinding, Menu, MenuItem};

actions!(my_app, [Increment, Reset]);

cx.bind_keys([
    KeyBinding::new("space", Increment, Some("Counter")), // 3rd arg = key context predicate
    KeyBinding::new("backspace", Reset, Some("Counter")),
    KeyBinding::new("cmd-q", Quit, None),
]);

cx.set_menus(vec![Menu {
    name: "My App".into(),
    items: vec![MenuItem::action("Quit", Quit)],
    disabled: false,
}]);

cx.on_action(|_: &Quit, cx| cx.quit());
```

Element side — the `key_context` name must match the binding's context:

```rust
div()
    .id("counter")
    .key_context("Counter")
    .track_focus(&self.focus_handle)
    .on_action(cx.listener(Self::increment))   // fn(&mut Self, &Increment, &mut Window, &mut Context<Self>)
```

Actions with data:

```rust
#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema, gpui::Action)]
#[action(namespace = my_app)]
struct Paste { content: SharedString }

// or, for a manual impl: #[register_action] + impl gpui::Action
```

The derive needs `Clone + PartialEq`, plus `serde::Deserialize` and `schemars::JsonSchema` unless
you add `#[action(no_json)]`. `#[action(no_register)]` skips automatic registration.

Global app-level handlers: `cx.on_action(...)` (fires regardless of focus), and per-element
`capture_action` intercepts before children see it.

## Observing / subscribing / emitting

```rust
// Watch another entity: fires when it calls cx.notify().
let sub: Subscription = cx.observe(&other, |this, other, cx| { /* ... */ });
sub.detach();   // or store it on self so it lives as long as the view
// Prefer storing: `self._subs.push(sub);`

// Typed events.
struct Changed;

impl EventEmitter<Changed> for MyView {}

cx.subscribe(&other, |this, other, event: &Changed, cx| { /* ... */ });
cx.emit(Changed);

// Globals
cx.observe_global::<MyGlobal>(|cx| { /* ... */ });
```

Lifecycle/plumbing hooks: `cx.on_release(...)`, `cx.observe_release(...)`, `cx.defer_in(...)`,
`cx.on_next_frame(...)`, `cx.observe_self(...)`.

## See also

- `core-concepts.md` for `Entity`/`Render` fundamentals.
- `async.md` for `cx.spawn`.
- `windows-and-globals.md` for app lifecycle and `Global` state.
