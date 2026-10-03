# Lists & scrolling

## Uniform-height virtualized list

For large collections of equal-height rows, `uniform_list` only renders the visible range:

```rust
use gpui::{uniform_list, px};

uniform_list("items", self.items.len(), |range, _window, _cx| {
    range
        .map(|ix| {
            div()
                .id(ix)
                .h(px(24.))
                .px_2()
                .child(self.items[ix].clone())
        })
        .collect()
})
.h(px(300.))
```

- First argument is the list id; second is the item count.
- The callback receives the visible `Range<usize>` and must return exactly that many elements.
- Give each row a stable `.id(...)` if it holds state.
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

Useful `ListState` methods: `measure_all()` (exact scrollbar on the first frame),
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

## Details

- Scroll containers must be size-bounded, or they grow instead of scrolling.
- Hiding/restyling scrollbars: `.scrollbar_width(px(0.))`, `.overflow_fade(...)`.
- Custom wheel handling: `.on_scroll_wheel(...)` on the element, or `window.on_scroll_wheel`
  inside a custom `Element`.
