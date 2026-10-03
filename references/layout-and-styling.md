# Layout & styling

`div()` is a flex container by default. Style helpers are fluent and return `Self`.

```rust
div()
    .flex()                 // .flex_col() / .flex_row() / .grid() / .block() / .hidden()
    .items_center()         // items_start/end/center/baseline/stretch
    .justify_between()      // justify_start/end/center/between/around/evenly
    .flex_1()               // flex_none() / flex_auto() / flex_grow(n) / flex_shrink(n)
    .flex_wrap()
    .gap_2()
    .w(px(240.)).h(px(120.))
    .max_w(px(600.)).min_h_full()
    .p_4().px_2().py_1().mt_2().mx_auto()
    .bg(rgb(0x1f2937))
    .text_color(rgb(0xffffff))
    .text_sm()              // text_xs/sm/base/lg/xl/2xl/3xl, .text_size(px(18.))
    .font_weight(FontWeight::SEMIBOLD)
    .rounded_lg()           // rounded_sm/md/lg/xl/full, .rounded(px(12.))
    .border_1().border_color(rgb(0x333333)).border_dashed()
    .shadow_md()            // shadow_sm/md/lg/xl
    .opacity(0.8)
    .overflow_hidden()      // overflow_scroll() / overflow_y_scroll()
    .absolute().top_2().left_2()   // also .relative() and .inset_0()
    .cursor_pointer()
    .child("text or any IntoElement")
    .children(iter.map(|x| div().child(x)))
```

## Spacing scale

Tailwind's rem scale (1rem = 16px by default, scaled by `window.rem_size()`):

| method | size |
| --- | --- |
| `p_0p5` | 2px |
| `p_1` | 4px |
| `p_2` | 8px |
| `p_4` | 16px |
| `p_8` | 32px |

Prefixes: `p/pt/pb/pl/pr/px/py`, `m/mt/mb/ml/mr/mx/my`, `gap/gap_x/gap_y`,
`w/h/size/min_w/min_h/min_size/max_w/max_h/max_size`, `inset/top/right/bottom/left`.

Suffixes include `_auto` (margins only), `_px`, `_full`, fractions (`_1_2`, `_2_3`, `_1_4`, …) and
large steps (`_16` = 64px, `_96` = 384px). Arbitrary values use the plain method: `.w(px(10.))`,
`.max_w(px(600.))`, `.p(px(6.))`, `.gap(px(12.))`.

## Grid

```rust
div().grid().grid_cols(3).grid_rows(2).gap_2()
    .child(div().col_span(2).row_span(1).child("wide"))
    .child(div().col_span_full().child("full-width"))
```

Also `grid_cols_min_content` / `max_content`, `col_start/end`, `row_start/end`, `row_span_full`.

## Positioning

- `.relative()` establishes a containing block; `.absolute()` children position against it.
- `.inset_0()` pins to all edges; `.top_2()/.left(px(8.))` etc. set individual edges.
- `.size_full()` / `.w_full()` / `.h_full()` fill the parent.

## Conditional styling

```rust
div()
    .id("card")
    .when(is_selected, |el| el.border_color(accent))
    .when_some(label, |el, label| el.child(label))
    .when_else(disabled, |el| el.opacity(0.5), |el| el.cursor_pointer())
    .map(|el| if compact { el.p_2() } else { el.p_4() })
```

`when`, `when_some`, `when_else`, and `map` come from `FluentBuilder` (in the prelude).

## Interaction pseudo-styles & groups

```rust
div()
    .id("btn")
    .hover(|style| style.bg(hover_bg))
    .active(|style| style.bg(active_bg))
    .focus(|style| style.border_color(accent))
    .focus_visible(|style| style.shadow_sm())
    .group("card")                              // tag this element as a group
    .child(
        div()
            .opacity(0.)
            .group_hover("card", |style| style.opacity(1.)),
    )
```

These are default methods on `InteractiveElement`, so they need an `.id(...)` when the element is
also stateful (click/hover handlers, transitions).

## Colors

```rust
use gpui::{rgb, rgba, hsla, white, black, transparent_black, red, green, blue, yellow};

rgb(0x2a63d9);                       // opaque
rgba(0xffffff30);                    // hex alpha
hsla(0.6, 0.8, 0.5, 1.0);
colors.disabled.with_alpha(0.5);     // ColorExt / WithAlpha
```

Built-in palette:

```rust
use gpui::colors::Colors;

let colors = Colors::for_appearance(window);   // light/dark from window appearance
// fields: text, selected_text, background, disabled, selected, border, separator, container

let colors = Colors::get_global(cx);           // if a GlobalColors has been set
```

## Effects (GPUI-CE extras)

```rust
// Frosted glass: blur whatever is painted behind this element.
div().bg(rgba(0xffffff30)).backdrop_blur(px(24.))

// Content blur: blur this element and its children as a group.
div().blur(px(4.))

// Squircle-ish corners (0.0 = normal rounded rect), or the iOS preset.
div().rounded_xl().rounded_smoothing(0.8)
div().rounded_xl().rounded_smoothing_ios()
```

## Style transitions

Animate style changes between frames (per element, keyed by `.id(...)`):

```rust
div()
    .id("btn")
    .bg(base)
    .hover(|s| s.bg(hover))
    .transitions(|t| {
        t.bg(millis(200).with_easing(ease_in_out))
            .p(spring(600., 22., 1.))
    })
```

`transitions` comes from `InteractiveElement`. For value/material animations (`with_animation`,
`use_keyed_transition`), see `animation-and-motion.md`.
