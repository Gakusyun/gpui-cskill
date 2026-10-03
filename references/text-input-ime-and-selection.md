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
- **Do not insert `key_char` as well, unless you also stop propagation.** The backend only calls
  `TranslateMessage` (which produces `WM_CHAR`) when the `KeyDownEvent` was *not* consumed, i.e.
  when `cx.propagate_event` is still true after dispatch. So inserting `key_char` in `on_key_down`
  without `cx.stop_propagation()` lets the following `WM_CHAR` insert the same character again.
  Either let the input handler own text and reserve `on_key_down` for non-text keys (backspace,
  arrows, Enter, clipboard shortcuts), or insert `key_char` and call `cx.stop_propagation()`.

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

Registration is also gated on focus: `handle_input` only pushes the handler when
`focus_handle.is_focused(window)` is true. And key dispatch requires the element to have called
`.track_focus(..)` — a field that *looks* focused (caret and ring drawn, `is_focused() == true`)
but never called `track_focus` receives no keys at all. See the checklist in `text-input.md`.

### `EntityInputHandler`

Implement it for the view; the ranges it passes are **UTF-16 code-unit offsets** (Windows/IME
convention), not the byte offsets the rest of the field uses — convert at the boundary.

| Method | Purpose |
| --- | --- |
| `text_for_range(range, adjusted, ..)` | text in a UTF-16 range (OS may adjust the range) |
| `selected_text_range(..) -> UTF16Selection` | caret / selection, for `bounds_for_range` |
| `marked_text_range(..)` | the current composition range, if any |
| `unmark_text(..)` | composition committed/cancelled |
| `replace_text_in_range(range, text, ..)` | insert/replace; `None` = replace the **marked/composing** range, else the selection |
| `replace_and_mark_text_in_range(range, new_text, selected, ..)` | composition update |
| `bounds_for_range(range, element_bounds, ..)` | screen rect of a range (IME candidate window) |
| `character_index_for_point(point, ..)` | hit-test a point to a UTF-16 offset |
| `set_selected_text_range`, `text_length_utf16`, `accepts_text_input` | defaults are usually fine |

### `None` range means "replace the composing text" — not "insert at the caret"

This mirrors `NSTextInputClient.insertText(_:replacementRange:)`: a `None` replacement range means
"replace whatever the input method currently owns". If you treat `None` as "selection or caret
insert", IME composition leaves the phonetic text behind — type `z`, pick 中, and you get `z中`.

Resolve a `None` range in this order:

1. `marked_text_range()` — the active composition range, if any;
2. the current selection;
3. a plain insert at the caret.

The Windows backend can deliver both `GCS_RESULTSTR` and `GCS_COMPSTR` in a single
`WM_IME_COMPOSITION` frame. Handle `GCS_RESULTSTR` first (it commits the composition via
`replace_text_in_range`), then `GCS_COMPSTR` (`replace_and_mark_text_in_range`); each callback
should also update the field's `marked_text_range` state so the next `None` resolves correctly.

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
- `MouseUpEvent` is window-level and carries no bounds, so clear the `dragging` flag there rather
  than on the field's own handlers.
- `bounds_for_range` must return **window coordinates**; that is another reason `handle_input`
  has to run during paint, where the element's `bounds` are known (a `prepaint` callback has none).
- When painting the caret as a sibling span, emit one extra segment after the loop, or a caret at
  the very end of the line is never drawn (its edge collapses into the line's final boundary).

Typing and `Delete`/`Backspace` must **replace the selected range** first, then move the caret to
the end of the inserted text. All the string math stays byte-indexed; only the input-handler
boundary converts to/from UTF-16.
