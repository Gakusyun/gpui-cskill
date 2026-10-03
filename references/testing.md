# Testing

Enable the `test-support` feature (normally as a dev-dependency):

```sh
cargo add gpui-ce --dev --features test-support
cargo add gpui_ce_platform --dev --features test-support --rename gpui_platform
```

GPUI ships deterministic executors, so async tests do not flake.

## `#[gpui::test]`

```rust
use gpui::{Modifiers, TestAppContext, point, prelude::*, px};

#[gpui::test]
fn counter_increments(cx: &mut TestAppContext) {
    // add_window_view returns the root Entity<V> plus a &mut VisualTestContext,
    // which acts as Window + App and can simulate real input.
    let (view, cx) = cx.add_window_view(|window, cx| MyView::new(window, cx));

    cx.update(|window, cx| {
        let handle = view.read(cx).focus_handle.clone();
        window.focus(&handle, cx);
    });
    cx.simulate_input("hello");                 // VisualTestContext
    cx.simulate_click(point(px(10.), px(10.)), Modifiers::default());
    cx.run_until_parked();                      // drain timers + foreground/background tasks

    assert_eq!(cx.read_entity(&view, |v, _| v.count), 1);
}
```

- The macro understands multiple `TestAppContext` parameters for multi-context/collaboration tests:
  `#[gpui::test] fn t(cx_a: &mut TestAppContext, cx_b: &mut TestAppContext)`.
- `TestAppContext` is `App`-shaped: `cx.update(|cx| ...)`, `cx.read(|cx| ...)`,
  `cx.read_entity(&entity, |e, cx| ...)`, `cx.run_until_parked()`, `cx.advance_clock(dur)`.
- `VisualTestContext` (from `add_window_view` / `open_offscreen_window`) adds
  `simulate_input`, `simulate_keystrokes`, `simulate_click`, and `update(|window, cx| ...)`.

## Useful helpers

| Helper | Purpose |
| --- | --- |
| `cx.add_window_view(\|window, cx\| View::new(window, cx))` | open a real root view, get `(Entity<V>, &mut VisualTestContext)` |
| `cx.open_offscreen_window(\|window, cx\| ...)` | render without a visible window |
| `cx.run_until_parked()` | run all ready work, including timers, to completion |
| `cx.advance_clock(Duration)` | advance the deterministic clock (fires delayed tasks) |
| `cx.simulate_input("...")` | type text (goes through IME/input handlers) |
| `cx.simulate_keystrokes("cmd-s")` | dispatch keystrokes / actions |
| `cx.simulate_click(point(px(..), px(..)), Modifiers::default())` | click at a position |
| `cx.read_entity(&entity, \|e, cx\| ..)` | read entity state in a test |
| `cx.background_executor()` | deterministic background executor |

## Accessibility / visual assertions

- `window.debug_a11y_tree_json()` (built when accessibility is enabled via
  `Application::with_accessibility_forced`) lets you assert on roles/labels.
- Off-screen rendering tests can inspect painted frames; macOS visual tests **must run on the main
  thread** (`cargo test -- --test-threads=1`, and many are `#[ignore]` by default).

## Property tests

```rust
#[gpui::property_test]
fn layout_never_panics(x: i32, y: i32) {
    let _ = (x, y);
}
```

`SEED` controls both the deterministic scheduler seed and proptest's RNG, so failures reproduce.

## Tips

- Prefer asserting on entity state (`cx.read_entity`) over pixel output.
- Call `cx.run_until_parked()` after spawning tasks before asserting.
- Keep test views small; build them the same way `main` does.
