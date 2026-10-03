# Setup

## Installing dependencies

```sh
cargo new my-app && cd my-app

# Core framework: package `gpui-ce`, lib name `gpui` -> `use gpui::...`
cargo add gpui-ce

# Platform entry point: --rename so code can call `gpui_platform::application()`
cargo add gpui_ce_platform --rename gpui_platform
```

| Command | Crate name in Rust code | What it provides |
| --- | --- | --- |
| `cargo add gpui-ce` | `gpui` | `div()`, `Entity<T>`, `Render`, all core APIs |
| `cargo add gpui_ce_platform --rename gpui_platform` | `gpui_platform` | `application()`, `headless()`, `current_platform()` |
| `cargo add gpui_ce_platform` | `gpui_ce_platform` | default name without `--rename`; examples would not compile |

> Why `--rename`: the name used in Rust code is the dependency's **lib target name**
> (`gpui_ce_platform`) unless the dependency is explicitly renamed, in which case the rename
> alias (`gpui_platform`) wins. `gpui-ce`'s lib name is already `gpui`, so it needs no rename.

Dev/test dependencies:

```sh
cargo add gpui-ce --dev --features test-support
cargo add gpui_ce_platform --dev --features test-support --rename gpui_platform
```

Optional features:

```sh
cargo add gpui-ce --features inspector       # in-app element inspector
cargo add gpui-ce --features hot-patching    # Dioxus/Subsecond hot patching (debug)
```

If the registry has no usable release, switch to a git dependency — still via cargo, no manifest
editing by hand:

```sh
cargo add gpui-ce --git https://github.com/gpui-ce/gpui-ce
cargo add gpui_ce_platform --git https://github.com/gpui-ce/gpui-ce --rename gpui_platform
```

## Version / edition requirements

- **Rust ≥ 1.95** (the crates use `Vec::push_mut`). Run `rustup update stable` first on older toolchains.
- **Edition 2024**. `cargo new` uses the current toolchain's default edition; after `rustup update`
  it will be 2024.
- If you must, `cargo add gpui-ce` prints the resolved version; do not pin a lower one.

## Toolchain / system deps

- Install Git and Rust via [rustup](https://rustup.rs).
- `rustup component add rustfmt clippy`
- **Windows**: MSVC toolchain + Visual Studio C++ build tools + Windows SDK.
- **macOS**: Xcode or the Command Line Tools.
- **Linux (Debian/Ubuntu)**:

```sh
sudo apt-get install -y \
  build-essential pkg-config \
  libxkbcommon-dev libxkbcommon-x11-dev libwayland-dev libx11-dev \
  libxcb-shape0-dev libxcb-xfixes0-dev libxcb-randr0-dev libxcb-xinput-dev \
  libegl1-mesa-dev libgles2-mesa-dev libglib2.0-dev libfontconfig-dev
```

Desktop examples need a graphical session. WASM builds target `wasm32-unknown-unknown` and use a
separate backend (see `gpui_platform::application_with_web_backend` / `web_init`).

## Features on `gpui-ce`

Default features: `wayland`, `x11`, `windows-manifest`.

| feature | purpose |
| --- | --- |
| `test-support` | `#[gpui::test]`, `TestAppContext`, deterministic executors, proptest |
| `bench-support` | `gpui::bench` + Criterion |
| `inspector` | in-app element inspector |
| `hot-patching` | Dioxus/Subsecond live code patching (debug builds only) |
| `screen-capture` | platform screen capture APIs |
| `leak-detection` | backtrace-based entity-leak detection |
| `stacker` | stack-growth handling for deep layout trees |
| `embedded-assets` | bundle assets with `rust-embed` |
| `custom-gpu` / `wgpu-surfaces` | opt-in WGPU surface/device sharing |
| `profiler` | frame/task profiling |

Typical dev-only setup:

```sh
cargo add gpui-ce --dev --features test-support
cargo add gpui_ce_platform --dev --features test-support --rename gpui_platform
```

## Hot patching (optional)

Enable `hot-patching` on `gpui-ce`, install Dioxus CLI 0.7.10, then `dx serve --hot-patch`.
Description: a `Render::render` edit can redraw the running window without a restart.
Caveats: experimental, debug builds only, not on WASM, and it does not work for examples launched
directly from the GPUI-CE workspace (use a separate app crate that depends on it).

## Running

```sh
cargo run                # desktop app
cargo check              # fast feedback
cargo fmt && cargo clippy
```
