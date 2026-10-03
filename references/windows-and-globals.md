# Windows & globals

## Opening windows

```rust
use gpui::{
    Bounds, px, size, TitlebarOptions, WindowBounds, WindowKind, WindowOptions,
};

let bounds = Bounds::centered(None, size(px(900.), px(600.)), cx); // None = primary display

let handle = cx.open_window(
    WindowOptions::new()
        .window_bounds(Some(WindowBounds::Windowed(bounds)))   // Maximized / Fullscreen too
        .titlebar(Some(TitlebarOptions {
            title: Some("My App".into()),
            appears_transparent: true,        // macOS + Windows: hide the native bar
            traffic_light_position: None,     // macOS
        }))
        .kind(WindowKind::Normal),            // see table below
    |window, cx| cx.new(|cx| MyView::new(window, cx)),
)?;   // anyhow::Result<WindowHandle<V>>
```

`WindowKind`: `Normal`, `PopUp` (always-on-top, sparingly), `AnchoredPopup(PopupOptions)`
(native parent-anchored popup for menus/comboboxes/tooltips), `Floating`, `Dialog`, and
`LayerShell(..)` (Wayland only, behind the `wayland` feature).

Other `WindowOptions` fields (all have fluent setters):

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
window.defer(cx, |window, cx| { /* after this update cycle */ });
window.close(); / window.remove_window();
```

Global element state lives on the window: `use_state`, `use_keyed_state`,
`use_transition`, `use_keyed_transition`, `with_element_state`.

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
