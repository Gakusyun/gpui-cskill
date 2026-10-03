# Core concepts

## The object graph

```
Application ── owns ──> Window ── renders ──> element tree (Div, Text, Img, ...)
     │                                   │
     └── entity map: Entity<T> <─────────┘   state + listeners
```

| Type | Role |
| --- | --- |
| `App` | Global context: windows, globals, keymap, executors, HTTP client, asset registry. |
| `Window` | Per-window state: layout, focus, input, painting, element state, `spawn`. |
| `Context<'a, T>` | `App` + a handle to entity `T`. Derefs to `App`. Most view methods take this. |
| `AsyncApp` | `App` handle usable across `.await`. |
| `Entity<T>` | Clonable reference-counted handle to shared state. Views are entities. |
| `WeakEntity<T>` | Non-owning handle; required inside `'static` closures and spawned tasks. |
| `Subscription` | Handle to an observation; `.detach()` to keep it alive, drop to cancel. |
| `Task<R>` | Spawned future; `.detach()` to run to completion, dropping it cancels. |
| `FocusHandle` / `ScrollHandle` | Focus and scroll state, cheaply clonable. |

## Views: `Render` vs `RenderOnce`

```rust
// A view: an Entity<T> where T: Render. Persistent state, events, async.
struct Counter { count: i32 }

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().child(format!("{}", self.count))
    }
}

// A component: consumed when rendered. Stateless props in, callbacks out.
#[derive(IntoElement)]
struct Badge { label: SharedString, tone: Rgba }

impl RenderOnce for Badge {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div().px_2().rounded_full().bg(self.tone).child(self.label)
    }
}
```

Notes:

- `Render::render` takes `&mut self`; `RenderOnce::render` takes `self` (by value).
- `#[derive(IntoElement)]` on a type requires `RenderOnce` and makes the type usable as an element.
- To accept children, implement `ParentElement` (store `SmallVec<[AnyElement; N]>` and implement
  `extend`) and expose a `.child(...)` builder.
- Implement the low-level `Element` trait directly only for genuinely custom widgets; see
  `assets-and-drawing.md`.

## Creating and holding entities

```rust
let counter: Entity<Counter> = cx.new(|cx| Counter { count: 0 });

let n = counter.read(cx).count;
counter.update(cx, |this, cx| {
    this.count += 1;
    cx.notify();               // request a re-render
});

let weak: WeakEntity<Counter> = counter.downgrade();
weak.update(cx, |this, cx| { /* ... */ }).ok();   // Result: Err if the entity is gone
```

In a window/entity context you can also use `cx.entity()` (strong) and `cx.weak_entity()`.

## Three ways to hold component state

1. **`window.use_state`** — hook-like, scoped to the element's lifetime:

   ```rust
   let state: Entity<MyState> = window.use_state(cx, |_window, _cx| MyState::default());
   ```

   Keyed by the call site; inside loops use `use_keyed_state(key, cx, init)` with a stable key.

2. **`RenderOnce` component** — no internal state. Props flow down, events flow up via callbacks.
   Best default for presentational components.

3. **`Render` view backed by `Entity<T>`** — persistent state, `observe`/`subscribe`, async tasks,
   and a stable identity you can pass around.

## Re-rendering

- `cx.notify()` inside `Context<T>` marks that view dirty.
- `cx.notify(entity_id)` on `App` does it without an entity context.
- `cx.observe(&other, ...)` reacts to another entity's `notify()`.
- Element-scoped caching: `div().id("x")` plus `.cached(...)` on the view context can skip
  re-rendering unchanged subtrees — get this right before optimizing.

## The built-in HTTP client is GET-only

The `HttpClient` trait that `App` owns has a single method:

```rust
fn get(&self, url: &str, follow_redirects: bool)
    -> BoxFuture<'static, anyhow::Result<HttpResponse>>;
```

There is **no POST, PUT, or DELETE, and no way to set request headers**. That rules out JSON-RPC
control of a bundled sidecar (aria2, yt-dlp, ffmpeg progress sockets, …). For those, spawn a
small blocking client such as `ureq` from `cx.background_spawn` instead.

## Where things come from

- Traits you almost always need are in `gpui::prelude::*`.
- Palette is `gpui::colors::Colors` (not in the prelude).
- Geometry helpers: `px`, `relative`, `rems`, `size`, `point`, `bounds`, `percentage`.

## See also

- `state-events-and-input.md` for listeners, focus, actions, and observe/subscribe.
- `async.md` for `cx.spawn` and task lifetimes.
- `layout-and-styling.md` for the element/style API.
