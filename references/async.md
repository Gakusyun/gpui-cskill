# Async

GPUI has two executors: a **foreground** executor (main thread, can touch UI) and a **background**
executor (worker threads, must not touch UI).

## Foreground tasks on a view

`Context<T>::spawn` gives you a `WeakEntity<T>` and an `AsyncApp`:

```rust
cx.spawn(async move |this, cx| {
    cx.background_executor().timer(Duration::from_millis(500)).await;

    this.update(cx, |this, cx| {
        this.done = true;
        cx.notify();
    })
    .ok();          // Result: Err if the view was dropped
})
.detach();
```

On `App` the closure takes only the `AsyncApp`:

```rust
cx.spawn(async move |cx| {
    let value = cx.background_spawn(async { compute() }).await;
    // ...
})
.detach();
```

## Background work

```rust
let task: Task<u64> = cx.background_spawn(async move { heavy_computation() });
let value = task.await;          // await from a foreground task
```

`cx.background_executor()` gives a `BackgroundExecutor` with `timer(Duration) -> Task<()>` and
`spawn(future)`. The closure must be `Send + 'static`.

> `background_spawn` is a method of the `AppContext` trait. In a module that imports names
explicitly instead of `use gpui::prelude::*`, omitting it gives the opaque error
`no method named background_spawn found for mutable reference &mut AsyncApp`. Add `AppContext`
(the prelude covers it).

## Task lifetime

- `Task<T>` is `#[must_use]`; **dropping it cancels** the future.
- `.detach()` runs it to completion without a return value.
- `.detach_and_log_err(cx)` (from `TaskExt`) logs errors on `Task<Result<_, E>>`.
- Store a task in a field to keep it alive / replace it to cancel: `self.task = Some(...)`, then
  `self.task = None;`
- `.is_ready()` reports completion.

```rust
struct View { task: Option<Task<()>> }

impl View {
    fn start(&mut self, cx: &mut Context<Self>) {
        self.task = Some(cx.spawn(async move |this, cx| { /* ... */ }));
    }
    fn stop(&mut self) { self.task = None; }   // cancels
}
```

## Rules and pitfalls

1. Closures must be `'static`: capture `Entity`/`WeakEntity` clones, never `&self`.
2. `Entity::update` / `WeakEntity::update` return `Result` — `.ok()` is the usual pattern.
3. Only mutate UI from the foreground. Do CPU-heavy or blocking work with
   `cx.background_spawn`, then hop back via `this.update(...)`.
4. `std::thread::sleep` inside a *foreground* task blocks the whole UI. Use
   `background_executor().timer(..).await` or offload to a background task.
5. Tasks spawned from a view's `Context` are automatically bound to the foreground executor;
   `window.spawn(cx, future)` does the same from a `Window`.

## Timers and scheduling helpers

```rust
cx.background_executor().timer(Duration::from_millis(16)).await;
window.request_animation_frame();                        // schedule a repaint
cx.on_next_frame(window, |this, window, cx| { /* runs on the next frame */ });
```
