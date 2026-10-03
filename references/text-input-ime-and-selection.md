# Text input: IME & selection

The advanced half of a hand-rolled field. Read `text-input.md` first for the focus/key/caret/paste
recipe; this file covers the platform input handler (IME + plain text) and text selection.

## The platform input handler is the text path — not `key_char`

On Windows **every** typed character goes through the platform input handler:

- `WM_CHAR` → `replace_text_in_range(None, text)` — this is plain ASCII typing, not just IME.
- `WM_IME_COMPOSITION` → `replace_text_in_range` / `replace_and_mark_text_in_range`.

`with_input_handler(..)` returns `None` when no handler is installed, and the character is dropped
with **no log line**. Meanwhile `on_key_down` still receives `key_char: Some(..)`, which is why a
field that inserts `key_char` *appears* to work: ASCII arrives through the key handler, while CJK
produces nothing (or stray latin letters from the composition's keystrokes).

Two consequences to state outright:

- **Register an input handler for the focused field, unconditionally.** A field that never
  registers one types nothing at all, silently.
- **Do not do both.** Once the handler is installed it is the *only* text path; if `on_key_down`
  also inserts `key_char`, every character arrives twice. The key handler should own only the
  non-text keys (backspace, arrows, Enter, clipboard shortcuts).

### Where to register it, and where `bounds` comes from

`Window::handle_input` is `debug_assert_paint`ed, so it must run during **paint**, not `render`,
and it needs the element's `bounds`. A `canvas` overlay laid out during paint is the minimal shape
that works:

```rust
use gpui::{ElementInputHandler, canvas, prelude::*};

let focus = self.focus.clone();
let entity = cx.entity();
canvas(
    move |_bounds, _window, _cx| {},
    move |bounds, (), window, cx: &mut App| {
        // no-op unless this focus handle is the focused one in this frame
        window.handle_input(&focus, ElementInputHandler::new(bounds, entity.clone()), cx);
    },
)
.absolute()
.inset_0()            // cover the glyph area exactly, over a `.relative()` parent
```

`bounds_for_range` uses these bounds to place the IME candidate popup, so the canvas must cover
exactly the area the glyphs are drawn in — otherwise the popup lands beside the text.

### `EntityInputHandler`

Implement it for the view; the ranges it passes are **UTF-16 code-unit offsets** (Windows/IME
convention), not the byte offsets the rest of the field uses — convert at the boundary.

| Method | Purpose |
| --- | --- |
| `text_for_range(range, adjusted, ..)` | text in a UTF-16 range (OS may adjust the range) |
| `selected_text_range(..) -> UTF16Selection` | caret / selection, for `bounds_for_range` |
| `marked_text_range(..)` | the current composition range, if any |
| `unmark_text(..)` | composition committed/cancelled |
| `replace_text_in_range(range, text, ..)` | insert/replace; `None` range = replace selection |
| `replace_and_mark_text_in_range(range, new_text, selected, ..)` | composition update |
| `bounds_for_range(range, element_bounds, ..)` | screen rect of a range (IME candidate window) |
| `character_index_for_point(point, ..)` | hit-test a point to a UTF-16 offset |
| `set_selected_text_range`, `text_length_utf16`, `accepts_text_input` | defaults are usually fine |

## Selection

Selection is not built in — you track it. The parts that cost time:

- Keep the anchor as `Option<usize>` and treat **anchor == cursor as no selection**, or a
  zero-width highlight is painted after every caret move.
- A plain (non-extending) arrow move **collapses** the selection instead of stepping past it.
  Stepping past overshoots by one character — the classic off-by-one in a hand-rolled field.
- The pixel→offset mapping is `ShapedLine::closest_index_for_x(x)`. It needs the **same font and
  the same leftward scroll** the line was drawn with, so pass the shaped line you already have.
- Register `MouseDownEvent` / `MouseMoveEvent` from the same paint-phase `canvas` that registers
  the input handler — that gives you the bounds for free. `MouseMoveEvent::pressed_button` (an
  `Option<MouseButton>`) is what makes drag-select work.
- When painting the caret as a sibling span, emit one extra segment after the loop, or a caret at
  the very end of the line is never drawn (its edge collapses into the line's final boundary).

Typing and `Delete`/`Backspace` must **replace the selected range** first, then move the caret to
the end of the inserted text. All the string math stays byte-indexed; only the input-handler
boundary converts to/from UTF-16.
