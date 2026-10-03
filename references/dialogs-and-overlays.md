# Native dialogs & modal overlays

Three different things people mean by "a dialog", in increasing order of customization.

## Native file / folder / save dialogs

GPUI-CE ships real platform pickers — on Windows they are the modern `IFileOpenDialog`, not a
hand-rolled tree view. **Do not add a file-dialog crate** (`rfd`, `tinyfiledialogs`, …); the
framework already has this. They are methods on `App` (so `cx.prompt_for_paths(..)` works from a
`Context<T>` too), run the dialog on the platform's foreground executor, and report cancellation
as `Ok(None)`.

```rust
use gpui::{PathPromptOptions, prelude::*};

let receiver = cx.prompt_for_paths(PathPromptOptions {
    files: true,          // allow selecting files
    directories: false,   // allow selecting folders (set this for a folder picker)
    multiple: true,       // multi-select
    prompt: Some("Import".into()),   // the *OK button label*, not the dialog title
});

cx.spawn(async move |this, cx| {
    match receiver.await {
        Ok(Ok(Some(paths))) => { /* paths: Vec<PathBuf> */ }
        Ok(Ok(None))        => { /* cancelled */ }
        Ok(Err(err))        => { /* Linux only, when the picker can't open */ }
        Err(_)              => { /* sender dropped */ }
    }
})
.detach();
```

The returned `oneshot::Receiver` is a `Future`, so `await` it directly (no `.recv()`).

The save-file counterpart sets the initial directory and suggested name:

```rust
let receiver = cx.prompt_for_new_path(
    std::path::Path::new("C:\\Users\\me\\Downloads"),
    Some("movie.mkv"),                 // suggested file name
);                                     // -> Result<Option<PathBuf>>
```

Also on `App`, and often more useful than a dialog: `cx.open_url("https://…")` opens the default
browser, and `cx.reveal_path(path)` reveals a path in Explorer/Finder.

## Built-in message prompt

`Window::prompt(..)` is a ready-made modal for "are you sure" confirmations. It returns the
**index of the clicked button** through a oneshot channel; it is fixed-layout (no custom content).

```rust
use gpui::{PromptLevel, prelude::*};

.on_click(cx.listener(|this, _, window, cx| {
    let receiver = window.prompt(
        PromptLevel::Warning,                 // Info / Warning / Critical
        "Clear finished downloads?",
        Some("This only removes them from the list."),
        &["Cancel", "Clear"],                 // labels; see the PromptButton note below
        cx,
    );
    cx.spawn(async move |this, cx| {
        if let Ok(index) = receiver.await {
            // index into the `answers` slice above
            if index == 1 { this.clear_finished(cx); }
        }
    })
    .detach();
}))
```

A `&str` answer converts with `PromptButton::from`: `"ok"` and `"cancel"` (case-insensitive) become
the semantic `Ok`/`Cancel` buttons; anything else is an `Other`. Use
`PromptButton::ok(..)` / `.cancel(..)` / `.new(..)` when you want labelled semantic buttons. The
prompt is single-use: `Window::prompt` panics on re-entrancy, so don't call it again before the
previous one is resolved.

## Custom modal overlay

For anything with custom content, build it yourself. The framework's own prompt is exactly this
shape; the three parts that matter are the scrim, `occlude`, and `stop_propagation`.

```rust
use gpui::{div, prelude::*, rgba};

// Rendered last, above the rest of the tree, so it overlays everything.
div()
    .id("modal-scrim")
    .absolute()
    .inset_0()                 // top/right/bottom/left = 0 (also: .top_0().left_0().size_full())
    .flex()
    .items_center()
    .justify_center()
    .bg(rgba(0x00000080))
    .occlude()                 // stop the mouse reaching elements *behind* the scrim
    .on_click(cx.listener(|this, _, _, cx| this.dismiss(cx)))
    .child(
        div()
            .id("modal-card")
            // Without this, clicking a control in the card also bubbles to the scrim
            // and dismisses the modal.
            .on_click(|_, _, cx| cx.stop_propagation())
            .child(/* card content, including its own buttons */),
    )
```

- `.occlude()` sets the hitbox to `BlockMouse`; use `.block_mouse_except_scroll()` if the
  background should still scroll.
- `.on_click` needs an `.id(..)`, same as any interactive element.
- `App::stop_propagation()` works from any event callback and stops the event bubbling further up.
- Render the overlay as the last child of a `relative()` container (or as a `deferred(..)` / the
  root's last child) so it sits on top. `deferred(..)` is handy when the overlay is logically a
  sibling but must paint after the main content.
