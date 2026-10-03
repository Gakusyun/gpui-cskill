# Pitfalls & API index

## Compile-error cheat-sheet

| Symptom | Cause / fix |
| --- | --- |
| `no method named on_click/on_hover/on_drag found for Div` | Interactive methods need `.id(...)` (`StatefulInteractiveElement`). |
| `Application::new` not found | There is none. Use `gpui_platform::application()` or `Application::with_platform(..)`. |
| `cannot find crate gpui_platform` | Install with `cargo add gpui_ce_platform --rename gpui_platform`. |
| `the parameter type may not live long enough` / `'static` | Clone `Entity`s, capture `WeakEntity`, use `cx.listener(..)` instead of holding `&self`. |
| `no method named notify` on `&mut App` / `&mut Window` | `notify` lives on `Context<T>`. Use `cx.notify(entity_id)` on `App`. |
| `expected Entity<T>, found T` | Read with `entity.read(cx)`, mutate with `entity.update(cx, ..)`. |
| Can't call `self.method()` inside a closure | `cx.listener(Self::method)`, or clone a `WeakEntity` and call `.update`. |
| `impl IntoElement` return mismatch | `Render::render` must return `impl IntoElement`; return `div()`, not a `Div` value typed wrongly. |
| `.child(option)` does not compile | `Option<T>` is not `IntoElement`; use `.when_some(option, \|el, v\| el.child(v))`. |
| View never updates | You forgot `cx.notify()`; observers only fire on `notify`. |
| `ElementId` type error | `.id()` takes `impl Into<ElementId>`: `&str`, `usize`, or tuples like `("row", ix)`. |
| Loop rows share state / wrong row updates | Add a stable per-row `.id(...)`, or `window.use_keyed_state(key, ..)`. |
| `ListState::default()` not found | `ListState::new(item_count, ListAlignment::Top, px(overdraw))`. |
| Windows build fails | Install MSVC C++ build tools + Windows SDK; keep the default `windows-manifest` feature. |
| `rust-version` / edition errors | Requires edition 2024 and Rust ≥ 1.95. |
| Foreground task freezes the UI | Move blocking work to `cx.background_spawn` or await `background_executor().timer(..)`. |
| Detached work silently stops | Dropping a `Task` cancels it; call `.detach()` or store it in a field. |

## Differences from upstream GPUI / Zed

- Entry point is `gpui_platform::application()`, not `Application::new()`.
- No `h_flex()` / `v_flex()` helpers — use `div().flex().flex_row()/.flex_col()`.
- Added: `.transitions(..)` style transitions, `Motion`/springs, `.blur`, `.backdrop_blur`,
  `.rounded_smoothing`, `container_query`, `surface`.
- Platform backends are split into `gpui_ce_platform` + per-OS crates; upstream assumes they are wired.
- Crate/package names differ: package `gpui-ce` (lib `gpui`), package `gpui_ce_platform`.

## API quick index

**Contexts & handles**
`App`, `Window`, `Context<T>`, `AsyncApp`, `Entity<T>`, `WeakEntity<T>`, `Subscription`,
`Task<R>`, `FocusHandle`, `ScrollHandle`, `ListState`, `TestAppContext`, `VisualTestContext`.

**Traits (all in `gpui::prelude::*`)**
`Render`, `RenderOnce`, `IntoElement`, `Element`, `ParentElement`, `InteractiveElement`,
`StatefulInteractiveElement`, `Styled`, `StyledImage`, `VisualContext`, `AppContext`,
`FluentBuilder`, `Refineable`, `TaskExt`.

**Element constructors**
`div()`, `img()`, `svg()`, `canvas()`, `list()`, `uniform_list()`, `deferred()`, `anchored()`,
`surface()`, `container_query()`.

**Styling**
`.flex*`, `.grid_*`, `.items_*`, `.justify_*`, `.gap_*`, `.p*/px*/py*/m*`, `.w/.h/.size*`,
`.bg`, `.text_*`, `.font_weight`, `.rounded*`, `.border*`, `.shadow*`, `.overflow_*`,
`.absolute/.relative`, `.cursor_*`, `.when/.when_some/.when_else/.map`,
`.hover/.active/.focus/.focus_visible`, `.group/.group_hover`, `.transitions`,
`.blur/.backdrop_blur`, `.rounded_smoothing`.

**State & events**
`cx.new`, `Entity::read/update/downgrade`, `cx.listener`, `cx.observe`, `cx.subscribe`, `cx.emit`,
`EventEmitter`, `cx.observe_global`, `cx.observe_release`, `Window::use_state`,
`Window::use_keyed_state`, `Window::use_keyed_transition`.

**Input & actions**
`on_click`, `on_hover`, `on_mouse_down/up/move`, `on_key_down/up`, `on_scroll_wheel`, `on_pinch`,
`on_drag`, `on_drop`, `on_focus`, `on_blur`, `on_action`, `capture_action`, `actions!`,
`#[derive(Action)]`, `KeyBinding`, `Menu`, `MenuItem`, `cx.bind_keys`, `cx.set_menus`.

**Async**
`cx.spawn`, `cx.background_spawn`, `background_executor().timer(..)`, `Task::detach`,
`Task::is_ready`, `TaskExt::detach_and_log_err`, `window.spawn`.

**Painting**
`window.paint_quad`, `paint_path`, `paint_svg`, `paint_image`, `paint_layer`, `paint_underline`,
`fill()`, `outline()`, `PathBuilder`, `Path`, `Bounds`, `point()`, `size()`.

**Animation**
`Animation`, `AnimationExt`, `Transformation`, `Motion`, `MotionDurationExt`, `millis`, `spring`,
`ease_in_out`, `linear`, `bounce`, `Animated`, `Transition`, `use_transition`,
`use_keyed_transition`, `.with_animation`, `.transitions`.

**Windows & app**
`WindowOptions`, `WindowBounds`, `TitlebarOptions`, `WindowKind`, `WindowAppearance`, `QuitMode`,
`cx.open_window`, `cx.activate`, `cx.quit`, `cx.on_window_closed`, `cx.set_menus`,
`cx.windows`, `cx.active_window`, `Global`, `cx.set_global`, `cx.global`, `cx.observe_global`.

## When a signature is unclear

Prefer the patterns in this skill (they match the current fork). For exact signatures, consult the
public docs: <https://gpui-ce.github.io/> and the repository at
github.com/gpui-ce/gpui-ce.
