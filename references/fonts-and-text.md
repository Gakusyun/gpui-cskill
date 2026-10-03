# Fonts & text

Text styling is fluent (`Styled`); font *selection* and fallback have sharp edges worth knowing.

## Family, weight, size

```rust
div()
    .font_family("MiSans")          // family for this element and its subtree
    .font_weight(FontWeight::MEDIUM)
    .text_size(px(13.5))            // .text_xs/sm/base/lg/xl/2xl/3xl too
    .line_height(px(20.))           // or .line_height(rems(1.4))
```

`.text_size(..)` cascades to descendant text (a `Div` pushes its text style onto a window-level
stack during layout — the one style that *does* inherit; see `layout-and-styling.md`).

## `Font` and `FontFallbacks`

There is **no fluent `font_fallbacks` setter**. `FontFallbacks` only moves through `.font(Font {
.. })` — which also overwrites family, weight, style and features, so seed it from the current
style — or by assigning the public refinement field:

```rust
use gpui::{FontFallbacks, FontWeight};

// Via Font: seed from the current style so weight/style/features survive.
let mut font = window.text_style().font();
font.family = "MiSans".into();
font.weight = FontWeight::MEDIUM;
font.fallbacks = Some(FontFallbacks::from_fonts(vec!["Simsun".into()]));
div().font(font);

// Or set the family fluently and the fallbacks directly on the refinement.
let mut el = div().font_family("MiSans");
el.text_style().font_fallbacks = Some(FontFallbacks::from_fonts(vec!["Simsun".into()]));
```

**`Font::fallbacks` is not a CSS font stack.** It is a *per-glyph* fallback for characters the
primary family lacks. Family **selection** uses `font.family` alone (`TextSystem::resolve_font`);
if that fails, the text system's own fallback stack wins and your list is never consulted — a
missing family silently renders in the system UI font. A "first installed family wins" stack has
to be resolved yourself:

```rust
use gpui::{App, FontFallbacks, SharedString};

/// Turn a CSS-style "first installed wins" list into GPUI's family + per-glyph fallbacks.
fn resolve_family(cx: &App, wanted: &[&str]) -> (SharedString, Option<FontFallbacks>) {
    let installed = cx.text_system().all_font_names();   // canonical spellings
    let mut present: Vec<String> = wanted
        .iter()
        .filter(|name| installed.iter().any(|f| f.eq_ignore_ascii_case(name)))
        .map(|name| (*name).to_string())
        .collect();
    let family = present.first().cloned().unwrap_or_else(|| ".SystemUIFont".into());
    // The winner becomes the family; the rest are per-glyph fallbacks for glyphs it lacks.
    let fallbacks = (!present.is_empty()).then(|| FontFallbacks::from_fonts(present.split_off(1)));
    (family.into(), fallbacks)
}
```

One more sharp edge: a line's fallback list is applied **from the first run only**. The Windows
backend builds its `IDWriteTextFormat1` (and calls `SetFontFallback`) from `font_runs[0]`; later
runs set their own family/size/style but never their fallbacks. So when one element mixes fonts,
only the first run's list is honoured.

## Measuring text (caret / truncation / scroll)

`TextStyle` is a plain struct with public fields; `TextStyle::font()` returns a `Font`. To get a
line's advance width before layout, shape it through the window's text system:

```rust
use gpui::{Font, FontWeight, TextRun, px};

// `window.text_style()` here is the *window default*, not the enclosing Div's style:
// ancestors push their text style during layout, which runs after `render`. Seed from it,
// then override family/weight to match what you are about to paint.
let font = Font {
    family: resolved_family.clone(),
    weight: FontWeight::MEDIUM,
    ..window.text_style().font()
};
let run = TextRun { len: text.len(), font, ..Default::default() };
let line = window.text_system().shape_line(text.into(), px(size), &[run], None);
let caret_x = line.split_at(cursor).0.width();   // Pixels; read with .as_f32(), not .0
```

The font **size** is the `px(..)` argument to `shape_line`, not a field of `Font`. A fuller
caret-scrolling recipe is in `text-input.md`.
