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
| Actions | `on_action(cx.listener(Self::handler))`, `capture_action` |

> **`on_focus` / `on_blur` / `on_focus_in` / `on_focus_out` are NOT element handlers.**
> They exist only on `Context<T>` (they subscribe the view to its *own* focus events).
> `Div` / `StatefulInteractiveElement` have no such methods, so
> `div().on_focus(..)` / `div().on_blur(..)` will not compile. To ask "is my field focused?"
> inside `render`, call `self.focus.is_focused(window)` (see below).

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
// Own a focus handle on the view (e.g. cx.focus_handle() in the constructor)
let focus_handle = cx.focus_handle();
window.focus(&focus_handle, cx);

div()
    .id("editor")
    .track_focus(&self.focus_handle)   // makes the subtree focusable and routes keys here
    .key_context("Editor")
    .on_key_down(cx.listener(Self::on_key))
```

Two facts that are easy to get wrong:

- **`track_focus` alone does not focus on click.** A mouse press does not move focus into the
  element; you must do it explicitly:

  ```rust
  div()
      .id("field")
      .track_focus(&self.focus)
      .on_mouse_down(MouseButton::Left, cx.listener(|this, _, window, cx| {
          window.focus(&this.focus, cx);
      }))
  ```

- **`FocusHandle::is_focused` takes a `&Window`, not an `&App`.**

  ```rust
  // inside Render::render — the usual way to tell "should I draw a caret / focus ring?"
  let focused = self.focus.is_focused(window);
  ```

  Because it needs `&Window`, it is **unavailable inside `cx.spawn` / `background_spawn`
  callbacks** (those only hand you an `&mut App` / `AsyncApp`). A self-contained blinking-caret
  timer task therefore cannot check focus; drive the blink from a frame tick or from an
  existing poll loop instead, or cache the focused flag in the entity on focus events.

### Mouse and key events bubble

Mouse handlers **bubble**: a press is dispatched through the tree before the `on_click` that fires
on release, so a parent's `on_mouse_down` still runs when the user clicks a button inside it. The
classic breakage is a toolbar whose press handler clears a text field: clicking "Add" first wipes
the value, then the button submits an empty field and looks dead. Swallow the press on the child:

```rust
div()
    .id("submit")
    .on_mouse_down(MouseButton::Left, |_: &MouseDownEvent, _, cx| cx.stop_propagation())
    .on_click(cx.listener(|this, _, _, cx| this.submit(cx)))
```

`App::stop_propagation()` stops further bubbling; the child's own `on_click` still fires.

Tab order:

```rust
let h = cx.focus_handle().tab_index(1).tab_stop(true);
window.focus_next(cx);
window.focus_prev(cx);
```

Inside `Context<T>` (these *are* available here, just not as element methods):
`cx.on_focus(&handle, window, ...)`, `cx.on_focus_in(...)`, `cx.on_blur(...)`,
`cx.on_focus_out(...)`, and `cx.focus_self()`.

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
