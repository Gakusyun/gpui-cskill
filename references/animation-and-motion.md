# Animation & motion

Three related tools:

1. **`with_animation`** — declarative, per-element, driven by the framework.
2. **Transitions** — animate a value you own (`Transition<T>`), or animate style changes
   (`.transitions(...)`, see `layout-and-styling.md`).
3. **`Animated<T>`** — implicit "animate toward the latest target" values.

## 1. `with_animation`

```rust
use std::time::Duration;
use gpui::{
    Animation, AnimationExt as _, Transformation, bounce, ease_in_out, percentage,
};

svg()
    .size_16()
    .path("icons/arrow.svg")
    .with_animation(
        "spin",                                              // id: stable per element/key
        Animation::new(Duration::from_secs(2))               // Duration -> Motion
            .repeat()
            .with_easing(ease_in_out),
        |svg, delta| svg.with_transformation(Transformation::rotate(percentage(delta))),
    )
```

- The closure receives the element and `delta: f32` in `0..=1`, and returns the element.
- `Animation::new(impl Into<Motion>)`: a `Duration`, a `SpringConfig` (from `spring(..)`), or a
  `Motion` all work.
- Builder: `.repeat()`, `.repeat_synced()`, `.with_easing(f)`, `.with_max_fps(f32)`.
- Chain multiple animations by calling `.with_animation(id, ..)` repeatedly; they run in sequence.
- `Transformation::rotate/scale/translate`; `percentage(delta)` maps `0..1` to `0%..100%`.
- Easing helpers: `linear`, `ease_in_out`, `ease_out_quint()`, `bounce(easing)`.

## 2. Transitions

A `Transition<T>` animates a value you own and keeps its state across frames:

```rust
use gpui::{bounce, ease_in_out, spring, millis};
use std::time::Duration;

// Keyed: state persists while the key is stable (recommended).
let t = window
    .use_keyed_transition(("btn", "bounce"), cx, Duration::from_millis(1200), |_window, _cx| 0.0)
    .with_easing(bounce(ease_in_out));

let progress: f32 = *t.evaluate(window, cx);

// later, e.g. on click
t.reset(cx);
t.update(cx, |progress, cx| {
    *progress = 1.0;
    cx.notify();
});
```

- `window.use_transition(cx, motion, init)` recreates state every render;
  `window.use_keyed_transition(key, cx, motion, init)` persists it (use this).
- `motion` is anything `Into<Motion>`: `Duration`, `millis(200).with_easing(f)`,
  `spring(600., 22., 1.)`, or `millis(200).with_spring(spring(..))`.
- `Transition` API: `.with_easing(f)`, `.continuous(bool)`, `.evaluate(window, cx) -> Ref<T>`,
  `.evaluate_delta(cx)`, `.read_goal(cx)`, `.read_cache()`, `.update(cx, ..)`, `.jump_to(v, cx)`,
  `.scale_by(ratio, cx)`, `.reset(cx)`, `.entity_id()`.

Duration/motion helpers (`MotionDurationExt`):

```rust
use gpui::{millis, spring, ease_in_out};

millis(200);                               // Duration
millis(200).with_easing(ease_in_out);      // Motion
millis(200).with_spring(spring(600., 22., 1.));
spring(600., 22., 1.);                     // SpringConfig, also Into<Motion>
```

There is **no `seconds(..)` helper** in this fork — use `Duration::from_secs(n)`.

## 3. Implicit animated values

`Animated<T>` retargets smoothly whenever you `set` a new value:

```rust
use std::time::Instant;
use gpui::{Animated, ease_in_out, millis};

let mut anim = Animated::new(0.0_f32, millis(200).with_easing(ease_in_out));

// on a state change:
anim.set(1.0, millis(200).with_easing(ease_in_out), Instant::now());

// while rendering / ticking:
let sample = anim.sample(Instant::now());
// sample.value: T, sample.is_active: bool, sample.progress: Progress
if sample.is_active {
    window.request_animation_frame();
}
```

Also available: `anim.value()`, `anim.jump_to(v)`, `anim.reset()`. `T` must implement `Lerp`
(built in for `f32`, colors via `ColorExt`, points/sizes/bounds). When a new value arrives while
animating, presentation resumes from the current anchor instead of snapping.

## Style-change transitions

The most common animation for UI polish is animating style deltas:

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

Give repeated elements a stable `.id(...)` so each gets independent motion state.

## Performance notes

- Active animations schedule frames automatically; stop them when settled.
- Prefer `.transitions(...)` / `with_animation` over hand-written `cx.spawn` loops.
- `.with_max_fps(n)` throttles expensive animations.
- Do heavy per-frame work in `prepaint`, not `render`.
