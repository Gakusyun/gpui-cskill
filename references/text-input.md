# Building a text input

**GPUI-CE ships no text input element.** There is no `TextInput`, `TextArea`, or `TextEdit` — the
element set is `div`, `text`, `img`, `svg`, `list`, `uniform_list`, `canvas`, `deferred`,
`anchored`, `surface`, `container_query`. Any URL bar, search box, or rename dialog must be
assembled from a focus-tracked `div` plus key handling.

This file is the minimal, working recipe. It covers a single-line field with a caret and
clipboard paste. Selection and IME composition — and the platform input handler that is the real
text path on Windows — are in `text-input-ime-and-selection.md`; multi-line layout is noted at the
end.

## 1. State and focus

```rust
use gpui::{
    ClipboardItem, Context, FocusHandle, KeyDownEvent, MouseButton, Render, Window,
    div, prelude::*, px,
};

struct TextField {
    focus: FocusHandle,
    value: String,
    cursor: usize, // byte index into `value`
}

impl TextField {
    fn new(cx: &mut Context<Self>) -> Self {
        Self { focus: cx.focus_handle(), value: String::new(), cursor: 0 }
    }
}
```

`cx.focus_handle()` creates the handle owned by the view. `cursor` is a **byte** index so it can
be handed straight to `String::insert_str` / `split_at`.

## 2. Focus the field on click

`track_focus` makes the subtree focusable, but **does not focus it on click** — you must call
`window.focus` yourself:

```rust
div()
    .id("field")
    .track_focus(&self.focus)
    .cursor_text()                       // I-beam cursor
    .key_context("TextField")            // optional: enables action bindings
    .on_mouse_down(MouseButton::Left, cx.listener(|this, _event, window, cx| {
        window.focus(&this.focus, cx);
    }))
    .on_key_down(cx.listener(Self::on_key))
```

`is_focused` takes a **`&Window`**, so focus checks work in `render` but not in spawned tasks:

```rust
let focused = self.focus.is_focused(window); // in Render::render
```

> **Checklist — every item is load-bearing, and a missing one fails silently:**
>
> 1. `.id(..)` + **`.track_focus(&self.focus)`** on the element that handles keys — `track_focus`
>    is what registers the handle in the dispatch tree. Omit it and `is_focused()` still returns
>    `true` (caret and focus ring draw), yet key dispatch falls back to the framework's root node
>    (`DispatchNodeId(0)`, *not* your outermost `div`) and every `on_key_down` — on the field and on
>    its ancestors — is discarded with no warning.
> 2. `window.focus(&self.focus, cx)` on click (above); `track_focus` alone does not focus on
>    click.
> 3. Register the platform input handler during paint
>    (`text-input-ime-and-selection.md`) — otherwise typed text (including ASCII) is dropped on
>    Windows.
> 4. `Context::on_blur` for commit-on-blur, if needed (see below).

## 3. Key handling

Typed characters arrive in `keystroke.key_char` (`Option<String>`) — **not** in `keystroke.key`,
which is the physical key (`"a"`, `"backspace"`, …). Shift/caps/AltGr already fold into
`key_char`; `key` stays ASCII so shortcuts keep working.

> **This is the ASCII-only shortcut.** On Windows typed characters come through the field's input
> handler (`WM_CHAR` → `EntityInputHandler`); `key_char` works for Latin text but silently drops
> CJK/IME composition. For Chinese/Japanese, register a handler
> (`text-input-ime-and-selection.md`), and do **not** also insert `key_char`, or characters arrive twice.

```rust
impl TextField {
    fn on_key(&mut self, event: &KeyDownEvent, _window: &mut Window, cx: &mut Context<Self>) {
        let ks = &event.keystroke;

        match ks.key.as_str() {
            "backspace" => {
                if let Some(ix) = self.value[..self.cursor].char_indices().next_back().map(|(i, _)| i) {
                    self.value.remove(ix);
                    self.cursor = ix;
                }
            }
            "delete" => {
                if self.cursor < self.value.len() {
                    self.value.remove(self.cursor);
                }
            }
            "left" => {
                if let Some((ix, _)) = self.value[..self.cursor].char_indices().next_back() {
                    self.cursor = ix;
                }
            }
            "right" => {
                if let Some(ch) = self.value[self.cursor..].chars().next() {
                    self.cursor += ch.len_utf8();
                }
            }
            "home" => self.cursor = 0,
            "end" => self.cursor = self.value.len(),
            // Clipboard shortcuts.
            "v" if ks.modifiers.secondary() => {
                if let Some(text) = cx.read_from_clipboard().and_then(|item| item.text()) {
                    self.value.insert_str(self.cursor, &text);
                    self.cursor += text.len();
                }
            }
            "c" if ks.modifiers.secondary() => {
                cx.write_to_clipboard(ClipboardItem::new_string(self.value.clone()));
            }
            _ => {
                // Insert typed text. `key_char` is None for cmd/ctrl combos.
                if let Some(text) = &ks.key_char {
                    if !ks.modifiers.control && !ks.modifiers.platform {
                        self.value.insert_str(self.cursor, text);
                        self.cursor += text.len();
                    }
                }
            }
        }

        cx.notify();
    }
}
```

Key names are the usual `"backspace"`, `"delete"`, `"left"`, `"right"`, `"home"`, `"end"`,
`"enter"`, `"escape"`, `"tab"`. Check modifiers with `ks.modifiers.{control,alt,shift,platform}`
or `ks.modifiers.secondary()` (Cmd on macOS, Ctrl elsewhere).

## 4. Caret without measuring text

Render the value as **two halves split at the cursor**, with the caret element between them.
Layout positions the caret, so no text measurement is needed for the common case:

```rust
impl Render for TextField {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let (before, after) = self.value.split_at(self.cursor);
        let focused = self.focus.is_focused(window);

        div()
            .id("field")
            .flex().items_center()
            .h(px(30.)).px_2().rounded_md()
            .bg(gpui::white()).text_color(gpui::black())
            .border_1()
            .border_color(if focused { gpui::rgb(0x2a63d9) } else { gpui::rgb(0xcccccc) })
            .track_focus(&self.focus)
            .cursor_text()
            .on_mouse_down(MouseButton::Left, cx.listener(|this, _event, window, cx| {
                window.focus(&this.focus, cx);
            }))
            .on_key_down(cx.listener(Self::on_key))
            .child(before.to_string())
            .child(div().w(px(1.)).h(px(16.)).bg(gpui::rgb(0x2a63d9)))
            .child(after.to_string())
    }
}
```

To make the caret blink, store a `bool` on the view and flip it from an existing frame/poll tick;
do not create a task that needs `is_focused`, because that requires a `&Window`.

## 5. Measuring the caret (only when you must scroll)

When the value is wider than the field you need the caret's pixel `x`. Shape the line through the
window's text system and use `split_at` / `width`:

```rust
use gpui::{Font, FontWeight, TextRun, px};

// Measure with what you are about to paint with — NOT bare `window.text_style()`, which is
// the *window default* during `render` (ancestors push their text style onto a window-level
// stack during layout, and layout runs after `render`). Seed from it so features/style
// survive, then override family and weight.
let font = Font {
    family: family_you_are_rendering_with.clone(),   // the *resolved* family from settings
    weight: FontWeight::MEDIUM,
    ..window.text_style().font()
};

let run = TextRun {
    len: self.value.len(),                 // byte length covered by the run
    font,                                  // measuring only needs the font; colour is irrelevant
    ..Default::default()
};

let line = window.text_system().shape_line(
    self.value.clone().into(),
    px(13.),                               // the font size actually used for shaping
    &[run],
    None,                                  // force_width
);
let (before, _after) = line.split_at(self.cursor);
let caret_x = before.width();              // Pixels; read with .as_f32(), not .0
```

Feed `caret_x` into your scroll offset / `ScrollHandle` so the caret stays visible.

If the family comes from user settings, measure with the **resolved** family — otherwise the
caret drifts as soon as the user picks a different font.

## Committing the field when it loses focus

`Context::on_blur(&handle, window, listener)` fires only on the transition *away* from that
handle. It needs a `&mut Window`, so register it where the focus handle is created, not inside
`render`:

```rust
// Register once, where the view already exists (e.g. a `wire(..)` called from `new`).
// `self._blur` stores the Subscription so it is not dropped (which would unsubscribe).
self._blur = Some(cx.on_blur(&self.focus, window, |this, _window, cx| {
    // commit the draft / persist the value here
    this.commit(cx);
}));
```

The returned `Subscription` must be stored on the view (or `.detach()`ed or it unsubscribes
immediately). Trade-off: committing only on blur means the value lands when the user clicks away,
so a live preview has to be pushed from `on_key_down` instead. A common middle ground is to keep a
**draft** string for the caret/rendering and derive the committed value from it on each input.

**GPUI does not blur on click-away.** Clicking elsewhere in the window leaves the field focused;
if you want click-away-to-commit, either call `window.blur()` yourself (e.g. from a root-level
`on_mouse_down`) or test the click position and blur when it lands outside the field's bounds.

## Limits and extensions

- **Selection and IME are their own topic** — see `text-input-ime-and-selection.md`.
  `key_char` is not the real text path on Windows; a field that must accept Chinese/Japanese needs
  an `EntityInputHandler` registered during paint, and selection (anchor, drag, `closest_index_for_x`)
  is tracked by hand. Do not skip that file if either applies.
- **Multi-line** needs wrapping and a line-indexed caret. Consider a custom `Element`, or split
  the value on `\n` and render one row per line with per-line caret placement.
- **Password fields** are the same widget; render `•` per character instead of the value.
