---
name: gpui-cskill
description: Build desktop and web UIs with GPUI-CE, the community edition fork of Zed's GPU-accelerated Rust UI framework (crates `gpui-ce` / `gpui_ce_platform`). Use when writing, refactoring, debugging, or reviewing Rust code that shows a GUI, uses `div()`, `Entity<T>`, `Render`/`RenderOnce`, `Context`, actions/keybindings, `uniform_list`, `cx.spawn`, animations/transitions, `canvas`, or when the user mentions GPUI, gpui-ce, gpui_platform, or Zed-style UI.
---

# GPUI-CE

GPUI-CE is a community fork of Zed's **GPUI**: a retained-view, GPU-accelerated UI framework for
Rust with Tailwind-ish fluent styling and CSS-like flex/grid layout.

```rust
div().flex().gap_2().rounded_lg().bg(rgb(0x1f2937)).hover(|s| s.bg(rgb(0x374151)))
```

It is *mostly* API compatible with upstream GPUI/Zed, **but the entry point and some subsystems
differ**. When in doubt, trust this skill over remembered upstream snippets.

## Key facts

- Crates: `gpui-ce` (library name `gpui`) + `gpui_ce_platform` (usually aliased to `gpui_platform`).
- Edition 2024, **MSRV Rust 1.95+**.
- Styling is fluent and web-inspired; layout is flexbox by default, grid available.
- State lives in `Entity<T>`; a *view* is an `Entity<T>` where `T: Render`.
- Targets: macOS (Metal), Windows (Direct3D), Linux (X11/Wayland), WASM.

## Minimal app

```sh
cargo new my-app && cd my-app
cargo add gpui-ce
cargo add gpui_ce_platform --rename gpui_platform
```

```rust
use gpui::{
    App, Bounds, Context, Render, Window, WindowBounds, WindowOptions, div, prelude::*,
    px, size,
};

struct Hello;

impl Render for Hello {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div().size_full().flex().items_center().justify_center().child("Hello, GPUI-CE!")
    }
}

fn main() {
    gpui_platform::application().run(|cx: &mut App| {
        let bounds = Bounds::centered(None, size(px(640.), px(480.)), cx);
        cx.open_window(
            WindowOptions::new().window_bounds(Some(WindowBounds::Windowed(bounds))),
            |_window, cx| cx.new(|_cx| Hello),
        )
        .expect("failed to open window");

        cx.on_window_closed(|cx, _| {
            if cx.windows().is_empty() { cx.quit(); }
        })
        .detach();
    });
}
```

## Non-negotiable rules

1. Entry point is `gpui_platform::application()` — **there is no `gpui::Application::new()`**.
2. Always `use gpui::prelude::*;` (brings in `Render`, `IntoElement`, `InteractiveElement`,
   `ParentElement`, `Styled`, `VisualContext`, `AppContext`, `FluentBuilder`, `Refineable`, `TaskExt`).
3. Interactive elements need `.id(...)` — `on_click`/`on_hover`/`on_drag` only exist on
   **stateful** elements.
4. Re-render by calling `cx.notify()` from a `&mut Context<T>`; observers only fire on `notify`.
5. Closures must be `'static`: clone `Entity`s, capture `WeakEntity`s, use `cx.listener(...)`
   for `&mut self` access — never hold `&self` across a callback.
6. This fork has **no `h_flex()` / `v_flex()`**; use `div().flex().flex_row()/flex_col()`.

## Reference index — load only what the task needs

All paths are relative to this skill's directory. Read the smallest set that covers the task.

| File | Read it when you need to… |
| --- | --- |
| `references/setup.md` | set up a project/workspace, toolchain deps, or pick cargo features |
| `references/core-concepts.md` | understand App/Window/Context/Entity/Render and how state is held |
| `references/layout-and-styling.md` | lay out or style anything (flex/grid/spacing/colors/blur/transitions) |
| `references/state-events-and-input.md` | wire up events, focus, actions, keybindings, observe/subscribe/emit |
| `references/async.md` | spawn tasks, timers, background work, cancel/detach |
| `references/lists-and-scrolling.md` | virtualized/uniform lists, scrolling, `ListState`/`ScrollHandle` |
| `references/assets-and-drawing.md` | images/SVG/assets, `canvas`, custom elements/widgets |
| `references/animation-and-motion.md` | `with_animation`, transitions, springs, motion |
| `references/windows-and-globals.md` | window options/titlebars/lifecycle, globals, theming |
| `references/testing.md` | write `#[gpui::test]` tests, simulate input, run headless |
| `references/pitfalls-and-api-index.md` | hit a compile error, or need a fast symbol lookup |

Typical first pass: this file + `references/core-concepts.md` + the one or two references the
task touches. Do not preload every reference.
