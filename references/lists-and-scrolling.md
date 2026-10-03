# Lists & scrolling

## Uniform-height virtualized list

For large collections of equal-height rows, `uniform_list` only renders the visible range:

```rust
use gpui::{uniform_list, px};

let items = self.items.clone(); // the closure is 'static

uniform_list("items", items.len(), move |range, _window, _cx| {
    range
        .map(|ix| {
            div()
                .id(ix)
                .h(px(24.))
                .px_2()
                .child(items[ix].clone())
        })
        .collect()
})
.h(px(300.))
```

- First argument is the list id; second is the item count.
- The callback receives the visible `Range<usize>` and must return exactly that many elements.
- Give each row a stable `.id(...)` if it holds state.
- **The row callback gets `&mut App`, not `&mut Context<T>`** (`Fn(Range<usize>, &mut Window,
  &mut App) -> Vec<R>`). `cx.listener(...)` therefore **cannot** be used to build rows — it needs
  a `Context<T>`. The closure must be `'static`, so clone the data you need in and, for row
  callbacks that mutate state, capture `WeakEntity`/`Entity` clones:

  ```rust
  // Build this in Render::render, where you still have `&mut Context<Self>`.
  let this_entity = cx.entity(); // `listener_for` wants a strong Entity
  let items = self.items.clone();

  uniform_list("items", items.len(), move |range, window, _cx| {
      range
          .map(|ix| {
              div()
                  .id(ix)
                  // window.listener_for(&entity, ..) adapts a Context<T> handler into the
                  // `Fn(&E, &mut Window, &mut App)` shape on_click wants.
                  .on_click(window.listener_for(&this_entity, move |this, _event: &ClickEvent, _window, cx| {
                      this.select(ix);
                      cx.notify();
                  }))
                  .child(items[ix].clone())
          })
          .collect()
  })
  ```

  (`Window::handler_for(&entity, ..)` is the sibling for callbacks that take no event argument.)

  If that is too awkward, skip virtualization and use a plain `overflow_y_scroll` container
  built with `.children(...)` / `.child(...)`, where `cx.listener` works normally.

  Note: `List` (the variable-height list below) has the same `&mut App` callback signature, so
  the same constraint applies there.
- The list sets `overflow-y: scroll` itself; give it a bounded height (`.h(...)`, `.flex_1()`, …).

## Variable-height list

`ListState` lives on your view (it is `Clone`; the element keeps the shared state).

```rust
use gpui::{list, ListAlignment, ListState, px};

struct Rows {
    state: ListState,
    items: Vec<String>,
}

impl Rows {
    fn new(items: Vec<String>) -> Self {
        Self {
            // item_count, alignment, overdraw (extra px measured around the viewport)
            state: ListState::new(items.len(), ListAlignment::Top, px(100.)),
            items,
        }
    }

    fn push(&mut self, item: String, cx: &mut Context<Self>) {
        let ix = self.items.len();
        self.items.push(item);
        // Tell the list that `ix..ix` (an empty range) became 1 item.
        self.state.splice(ix..ix, 1);
        cx.notify();
    }

    fn clear(&mut self, cx: &mut Context<Self>) {
        self.items.clear();
        self.state.reset(0); // reset(new_item_count)
        cx.notify();
    }
}

// in render():
list(self.state.clone(), |ix, _window, _cx| {
    div().id(ix).px_2().py_1().child(self.items[ix].clone())
})
.size_full()
```

`ListAlignment::Top` scrolls like a normal list; `ListAlignment::Bottom` is for chat-log style
lists that grow from the bottom.

Useful `ListState` methods: `measure_all()` (measure the whole list up front, so scroll
extents/offsets are exact on the first frame),
`with_uniform_item_height(px(..))` (cheap height hint), `remeasure()`, `remeasure_items(range)`,
`reset(count)`, `reset_with_uniform_height(count, px(..))`, `splice(range, count)`,
`splice_focusable(range, focus_handles)`, `item_count()`, `is_scrolled_to_end()`,
`scroll_to(offset)`, `scroll_to_end()`, `scroll_to_reveal_item(ix)`, `logical_scroll_top()`,
`set_scroll_handler(...)`, `set_follow_mode(...)`, `pause_following_tail()`, `bounds_for_item(ix)`.

`list(...)` also supports `.with_sizing_behavior(ListSizingBehavior::…)`.

## Plain scrolling

```rust
div()
    .id("scroller")
    .size_full()
    .overflow_scroll()        // or .overflow_y_scroll() / .overflow_x_scroll()
    .children(rows)
```

Programmatic scrolling with `ScrollHandle` (attach with `.track_scroll(&handle)`):

```rust
use gpui::ScrollHandle;

struct View { scroll: ScrollHandle }

div()
    .id("scroller")
    .overflow_scroll()
    .track_scroll(&self.scroll)
    .children(rows)

// later
self.scroll.scroll_to_bottom();
self.scroll.scroll_to_item(ix);
self.scroll.scroll_to_top_of_item(ix);
let offset = self.scroll.offset();
let max = self.scroll.max_offset();
```

`uniform_list(..)` has its own `UniformListScrollHandle` via `.track_scroll(&handle)`.

## Scrollbars

gpui-ce 0.2.2 has **no scrollbar renderer**: `overflow_scroll()` / `overflow_y_scroll()` scroll
with the wheel (or a `ScrollHandle` / `ListState`), but never draw a track, thumb, hover reveal, or
drag affordance. There is also **no `.overflow_fade(...)`** — it is not a method, function, or type.

`.scrollbar_width(...)` is **layout-only**: it reserves space next to an `Overflow::Scroll` node,
and already defaults to `0`. With `0`, `Scroll` behaves like `Hidden` (the crate's own wording) —
content simply clips at the edge:

```rust
div().id("scroller").overflow_y_scroll().scrollbar_width(px(12.))
```

A visible bar or edge fade has to be painted by hand. `ScrollHandle` gives you the numbers to
drive it — it is `Clone`, `offset()` is the current position, and `max_offset()` is the scrollable
range `(content - viewport)`; both return `Point<Pixels>`:

```rust
div()
    .relative()
    .child(
        div()
            .id("scroller")
            .size_full()
            .overflow_y_scroll()
            .track_scroll(&self.scroll)
            .children(rows),
    )
    // Sibling overlay, not a child of the scroller, or it scrolls with the content.
    .child(div().absolute().top_0().right_0().w(px(8.)).h(px(40.)).bg(rgb(0x888888)))
```

## Decorative overlays don't block the wheel

A `Div` inserts a hitbox (`Div::should_insert_hitbox`) only when it has a listener, a
`mouse_cursor`, a `group` / `group_hover`, a `tracked_focus_handle`, a `scroll_offset`, or a
non-`Normal` `hitbox_behavior`. A plain absolutely-positioned overlay has none of these, so it has
**no hitbox**: wheel and click fall straight through to the scroller beneath it. There is no
`pointer-events: none` equivalent to remember:

```rust
// Painted over the list edge; the wheel still scrolls the list below.
div().absolute().bottom_0().left_0().w_full().h(px(48.)).bg(rgba(0x00000080))
```

Adding `.hover(..)`, `.on_click(..)`, `.group(..)`, `.occlude()`, or `.block_mouse_except_scroll()`
sets one of those conditions, so the overlay starts capturing those events.

## Details

- Scroll containers must be size-bounded, or they grow instead of scrolling.
- Custom wheel handling: `.on_scroll_wheel(...)` on the element, or `window.on_scroll_wheel`
  inside a custom `Element`.
