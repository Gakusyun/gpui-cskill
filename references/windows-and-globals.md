# Windows & globals

## Opening windows

`WindowOptions` is a **plain struct with public fields**. There is no `WindowOptions::new()`
and no fluent setters, so always build it as a struct literal and fill the rest with
`..Default::default()`:

```rust
use gpui::{
    Bounds, px, size, TitlebarOptions, WindowBounds, WindowKind, WindowOptions,
};

let bounds = Bounds::centered(None, size(px(900.), px(600.)), cx); // None = primary display

let handle = cx.open_window(
    WindowOptions {
        window_bounds: Some(WindowBounds::Windowed(bounds)),   // Maximized / Fullscreen too
        window_min_size: Some(size(px(720.), px(460.))),
        titlebar: Some(TitlebarOptions {
            title: Some("My App".into()),
            appears_transparent: true,        // macOS + Windows: hide the native bar
            traffic_light_position: None,     // macOS
        }),
        kind: WindowKind::Normal,             // see table below
        ..Default::default()
    },
    |window, cx| cx.new(|cx| MyView::new(window, cx)),
)?;   // anyhow::Result<WindowHandle<V>>
```

> `WindowOptions::new()` / `.window_bounds(..)` / `.titlebar(..)` do **not** exist in this
> fork — attempting them gives `no associated function ... named new found for struct
> WindowOptions`.

`WindowKind`: `Normal`, `PopUp` (always-on-top, sparingly), `AnchoredPopup(PopupOptions)`
(native parent-anchored popup for menus/comboboxes/tooltips), `Floating`, `Dialog`, and
`LayerShell(..)` (Wayland only, behind the `wayland` feature).

Other `WindowOptions` fields (set them in the struct literal):

| Field | Notes |
| --- | --- |
| `window_bounds: Option<WindowBounds>` | `Windowed` / `Maximized` / `Fullscreen`; `None` inherits |
| `window_min_size: Option<Size<Pixels>>` | minimum size |
| `focus: bool`, `show: bool` | focus/show on creation |
| `is_resizable`, `is_minimizable`, `is_movable` | window capabilities |
| `app_owns_titlebar_drag: bool` | macOS: when you draw + drag your own titlebar |
| `titlebar: Option<TitlebarOptions>` | title, transparent bar, traffic-light position |
| `display_id: Option<DisplayId>` | which monitor |
| `app_id: Option<String>` | Linux desktop grouping |
| `window_decorations: Option<WindowDecorations>` | X11/Wayland client vs server decorations |
| `icon: Option<Arc<RgbaImage>>` | X11/Wayland window icon |
| `inactive_frame_interval: Option<Duration>` | throttling while unfocused |
| `tabbing_identifier: Option<String>` | macOS native tabs |
| platform backgrounds | `macos_window_background`, `windows_window_background`, `linux_window_background` |

## Window API you will actually use

```rust
window.appearance();                       // Light/Dark/Vibrant*
window.viewport_size();                    // Size<Pixels>
window.bounds();                           // Bounds<Pixels>
window.is_window_active();
window.focus(&handle, cx);
window.focus_next(cx); / window.focus_prev(cx);
window.rem_size(); / window.set_rem_size(px(16.));
window.request_animation_frame();
window.spawn(cx, async move |cx| { /* ... */ });   // foreground task bound to this window
window.use_state(cx, |window, cx| ...);
window.use_keyed_transition(key, cx, motion, init);
window.start_window_move();                // custom titlebar dragging
window.on_next_frame(|window, cx| { /* ... */ });
window.defer(cx, |window, cx| { /* end of this update cycle, after all entity leases return */ });
window.close(); / window.remove_window();
```

Global element state lives on the window: `use_state`, `use_keyed_state`,
`use_transition`, `use_keyed_transition`, `with_element_state`.

## System appearance (light / dark)

`window.appearance()` returns a `WindowAppearance`, which has **four** variants — `Light`,
`VibrantLight`, `Dark`, `VibrantDark` — and **no `is_dark()` helper**. Match by hand:

```rust
use gpui::WindowAppearance;

fn is_dark(appearance: WindowAppearance) -> bool {
    matches!(appearance, WindowAppearance::Dark | WindowAppearance::VibrantDark)
}

let dark = is_dark(window.appearance());
```

To follow the system at runtime, subscribe with `Context::observe_window_appearance`:

```rust
self._appearance = Some(cx.observe_window_appearance(window, |this, window, cx| {
    // `this` is &mut Self; re-theme here, then cx.notify() if you don't mutate `this`
    this.set_theme(is_dark(window.appearance()), cx);
}));
```

Two things about the subscription:

- It returns a `Subscription`. **Store it** (e.g. in a field on the view) or call `.detach()`;
  dropping it unsubscribes.
- It fires only when the platform reports a **change**. Do not treat it as an initializer — read
  `window.appearance()` (or set the theme once) at startup. If you need to re-theme a value shared
  by many views, store the resolved theme in a `Global` (see below) and have views subscribe with
  `cx.observe_global::<Theme>(..)`.

## App lifecycle

```rust
cx.activate(true);                         // bring to front on launch
cx.quit();
cx.windows();                              // Vec<AnyWindowHandle>
cx.active_window();
cx.on_window_closed(|cx, window_id| { /* ... */ });   // returns Subscription
cx.on_action(|action: &MyAction, cx| { /* ... */ });
cx.set_menus(vec![ /* Menu */ ]);
cx.bind_keys([ /* KeyBinding */ ]);
cx.open_window(..);
cx.observe_new::<MyEntity>(|entity, window, cx| { /* ... */ });
cx.on_app_quit(|cx| async { /* cleanup */ });
```

Quit behaviour is controlled with `Application::with_quit_mode(QuitMode::..)`. The usual pattern is
to quit when the last window closes:

```rust
cx.on_window_closed(|cx, _| {
    if cx.windows().is_empty() {
        cx.quit();
    }
})
.detach();
```

## Globals

App-wide state that outlives windows:

```rust
use gpui::{App, Global, Rgba, rgb};

struct Theme { accent: Rgba }

impl Global for Theme {}

// install once, e.g. in run(|cx| ...)
cx.set_global(Theme { accent: rgb(0x2a63d9) });

// read anywhere with an &App / &Context
let theme = cx.global::<Theme>();
let accent = theme.accent;

// react to changes
cx.observe_global::<Theme>(|cx| { /* ... */ });
```

`cx.try_global::<T>()` returns `Option<&T>`; `cx.update_global::<T, _>(...)` mutates and notifies
observers. Removing with `cx.remove_global::<T>()`.

## Theming

`Colors::for_appearance(window)` / `Colors::dark()` / `Colors::light()` provide a ready palette
(see `layout-and-styling.md`). `GlobalColors` / `Colors::get_global(cx)` is the built-in global
slot if you want one palette shared across windows.

For window chrome, remember that `appears_transparent: true` + `window_control_area(...)` +
`.start_window_move()` is how GPUI-CE apps build custom titlebars.

## Custom title bars & window controls

Set `titlebar.appears_transparent = true` to hide the native bar, then draw your own. There are
two ways to make a region draggable:

```rust
use gpui::{WindowControlArea, div, px};

// Declarative: the platform hit-tests this region and treats it as the caption.
div().h(px(46.)).window_control_area(WindowControlArea::Drag)

// Imperative equivalent, e.g. from an on_click handler:
window.start_window_move();
```

Minimise / maximise / close buttons use the same mechanism. The platform performs the action, so
there is **no `on_click` handler** — just tag each button with its area:

```rust
fn window_button(area: WindowControlArea) -> impl IntoElement {
    div()
        .id("win-close")
        .size(px(30.))
        .window_control_area(area)   // Min / Max / Close
}
```

### The one rule: a `Drag` area must never be an ancestor of the buttons

This is the titlebar equivalent of a silent failure — the buttons render normally and simply do
nothing when pressed (or drag the window instead).

When the platform asks "what is under the cursor?" it walks the registered window-control hitboxes
in **registration order and returns the first match**. Paint registers an element before its
children, and the hit test includes ancestor hitboxes, so a `Drag` area on a wrapper wins over a
`Min`/`Max`/`Close` area on a descendant. The platform then sees `HTCAPTION` for the whole strip,
and the button hit codes are never reached.

```rust
// WRONG — pressing any button drags the window; the buttons are unreachable.
div()
    .window_control_area(WindowControlArea::Drag)
    .child(window_button(WindowControlArea::Min))
    .child(window_button(WindowControlArea::Max))
    .child(window_button(WindowControlArea::Close))

// RIGHT — the draggable strip is a *sibling* of the buttons, so exactly one control
// area contains the cursor when it is over a button.
div()
    .flex().flex_row().items_center()
    .child(
        div()
            .flex_1().h_full()
            .window_control_area(WindowControlArea::Drag),
    )
    .child(window_button(WindowControlArea::Min))
    .child(window_button(WindowControlArea::Max))
    .child(window_button(WindowControlArea::Close))
```

Related details:

- The button element needs a stable `.id(...)` for hover/group styling; the control area itself
  forces a hitbox to be registered, so it works even without one.
- The `Min`/`Max` mapping depends on `WindowOptions::is_minimizable` / `is_resizable` /
  `is_movable`: if minimise is disabled but the window is movable, `Min` degrades to `HTCAPTION`;
  if neither, it becomes `HTNOWHERE`. `Close` is always `HTCLOSE`.
- The three control areas are `Drag`, `Min`, `Max`, `Close`. `Max` toggles between maximised and
  restored by itself (the OS handles `HTMAXBUTTON`), so keep the area as `Max` in both states and
  only swap the button's **glyph** based on `window.is_maximized()` — do not add a click handler
  or a second area for restore.
