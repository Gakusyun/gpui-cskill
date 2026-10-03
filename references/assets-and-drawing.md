# Assets, images, SVG & custom drawing

## Images and SVG

```rust
use gpui::{img, svg, ObjectFit, SharedString, rgb};

img("icons/logo.png").size_8();
img(SharedString::from("photos/cat.jpg")).object_fit(ObjectFit::Cover);

svg().path("icons/check.svg").size_4().text_color(rgb(0x22c55e));
```

- `img(source)` accepts a path string or `ImageSource`; supported formats include png/jpg/gif/webp/
  bmp/ico/tiff/avif/exr/svg (see `Img::extensions()`).
- Image styling (`StyledImage`): `.object_fit(...)`, `.image_cache(&entity)`.
- `svg()` has exactly **two** setters: `.path(..)` (resolved through the app's `AssetSource`) and
  `.external_path(..)` (read from disk). **There is no `.data(&bytes)`.** To embed icon bytes,
  include them in an `AssetSource` implementation and reference them by `path` anyway.

### SVG icons are alpha masks

`Window::paint_svg` renders the SVG to an **alpha mask** and tints it with the element's
`text_color`. Consequences:

- Icons are **monochrome by construction**. Multi-colour artwork silently becomes a silhouette
  and the SVG's own `fill` / `stroke` colours are ignored. Author icons as single-colour shapes.
- Colour **cascades** from the parent, so hover states on icons are free:

  ```rust
  div()
      .id("btn")
      .hover(|s| s.text_color(rgb(0x93c5fd)))
      .child(svg().path("icons/x.svg").size_4())
  ```

## Registering an AssetSource

`AssetSource` methods return `anyhow::Result<..>`, and `anyhow` is **not** re-exported as a
crate (only `gpui::Result`, which is `anyhow::Result`). Either `cargo add anyhow` or write
`gpui::Result<..>` in the impl signatures below.

```rust
use gpui::{AssetSource, SharedString};
use std::borrow::Cow;

struct Assets;

impl AssetSource for Assets {
    fn load(&self, path: &str) -> anyhow::Result<Option<Cow<'static, [u8]>>> {
        std::fs::read(path).map(Into::into).map_err(Into::into).map(Some)
    }

    fn list(&self, path: &str) -> anyhow::Result<Vec<SharedString>> {
        Ok(std::fs::read_dir(path)?
            .filter_map(|entry| {
                Some(SharedString::from(
                    entry.ok()?.path().to_string_lossy().into_owned(),
                ))
            })
            .collect())
    }
}

gpui_platform::application()
    .with_assets(Assets)
    .run(|cx| { /* ... */ });
```

The `embedded-assets` feature bundles assets with `rust-embed` instead of reading from disk.

## Custom drawing with `canvas`

`canvas` gives you a paint callback without defining a whole element:

```rust
use gpui::{canvas, fill, point, px, size, Bounds, PathBuilder};

canvas(
    move |_bounds, _window, _cx| { /* prepaint: compute state, return it */ },
    move |bounds, _state, window, _cx| {
        // paint: draw the state
        window.paint_quad(fill(bounds, rgb(0x5078f0)));

        window.paint_quad(fill(
            Bounds {
                origin: bounds.origin + point(px(10.), px(10.)),
                size: size(px(40.), px(40.)),
            },
            rgb(0xff0000),
        ));
    },
)
.size(px(80.))
```

The first callback runs during prepaint and its return value is passed to the second. Both are
`FnOnce` and must be `'static`.

Painting API on `Window`:

| API | Purpose |
| --- | --- |
| `paint_quad(PaintQuad)` | fill/stroke a quad; build with `fill(bounds, color)`, `outline(...)` |
| `paint_quad_with_corner_smoothing(quad, f32)` | quads with squircle corners |
| `paint_path(Path<Pixels>, color)` | fill/stroke an arbitrary path |
| `paint_underline(...)` | text underline / decoration |
| `paint_svg(...)` / `paint_image(...)` | draw a loaded asset at a given bounds |
| `paint_layer(bounds, f)` | clip subsequent drawing to `bounds` |

Build geometry with `PathBuilder`:

```rust
let mut builder = PathBuilder::fill(); // or PathBuilder::stroke(px(2.))
builder.move_to(point(px(10.), px(10.)));
builder.line_to(point(px(80.), px(10.)));
builder.curve_to(point(px(90.), px(20.)), point(px(80.), px(60.)), point(px(50.), px(60.)));
builder.close();
let path = builder.build()?;             // Path<Pixels>
```

## Custom elements

Prefer composition. A `#[derive(IntoElement)]` + `RenderOnce` component covers most needs:

```rust
#[derive(IntoElement)]
struct Badge { label: SharedString, tone: Rgba }

impl Badge {
    fn new(label: impl Into<SharedString>, tone: Rgba) -> Self {
        Self { label: label.into(), tone }
    }
}

impl RenderOnce for Badge {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div().px_2().py_0p5().rounded_full().bg(self.tone).child(self.label)
    }
}
```

To accept children, implement `ParentElement` (store `SmallVec<[AnyElement; N]>`):

```rust
#[derive(IntoElement)]
struct Panel { children: SmallVec<[AnyElement; 2]> }

impl Panel { fn new() -> Self { Self { children: SmallVec::new() } } }

impl ParentElement for Panel {
    fn extend(&mut self, elements: impl IntoIterator<Item = AnyElement>) {
        self.children.extend(elements);
    }
}

impl RenderOnce for Panel {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div().p_4().rounded_lg().children(self.children)
    }
}

// Panel::new().child("a").child("b")
```

Implement `Element` directly only for genuinely custom widgets (text editors, charts, virtualized
things). Required methods:

```rust
impl Element for MyWidget {
    type RequestLayoutState = ...;   // produced by request_layout
    type PrepaintState = ...;        // produced by prepaint, passed to paint

    fn id(&self) -> Option<ElementId>;
    fn source_location(&self) -> Option<&'static core::panic::Location<'static>>;
    fn request_layout(&mut self, id: Option<&GlobalElementId>, inspector_id: Option<&InspectorElementId>,
                      window: &mut Window, cx: &mut App) -> (LayoutId, Self::RequestLayoutState);
    fn prepaint(&mut self, id: Option<&GlobalElementId>, inspector_id: Option<&InspectorElementId>,
                bounds: Bounds<Pixels>, request_layout: &mut Self::RequestLayoutState,
                window: &mut Window, cx: &mut App) -> Self::PrepaintState;
    fn paint(&mut self, id: Option<&GlobalElementId>, inspector_id: Option<&InspectorElementId>,
             bounds: Bounds<Pixels>, request_layout: &mut Self::RequestLayoutState,
             prepaint: &mut Self::PrepaintState, window: &mut Window, cx: &mut App);
}
```

Accessibility hooks are optional: `a11y_role()`, `is_a11y_hidden()`, `write_a11y_info()`,
`a11y_synthetic_children()`.

## Other layout/overlay elements

| Constructor | What it does |
| --- | --- |
| `deferred(child)` | renders its child after the main layout/paint pass, so it can overlay the tree |
| `anchored()` | positions relative to a parent/point: `.anchor(...)`, `.position(...)`, `.offset(...)`, `.snap_to_window()` |
| `surface(source)` | embeds a native/foreign surface, e.g. a video or WGPU texture |
| `container_query(...)` | styles based on the container's measured size |
| `img(..)`, `svg(..)` | see above |
| `list(..)`, `uniform_list(..)` | see `lists-and-scrolling.md` |
