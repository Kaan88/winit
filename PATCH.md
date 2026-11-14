# Kaan88/winit, branch `md-0.30.13`

winit 0.30.13 (tag `v0.30.13`, version number unchanged so `[patch.crates-io]` can replace the
crates.io release) plus one Wayland-only feature: **a content margin**. It lets an app say "my
window is the inner rectangle of a larger transparent surface", so it can draw a drop shadow (or
any invisible resize border) inside its own buffer on every Wayland compositor, which have no
shadow of their own for undecorated windows. Behaviour is unchanged unless the new call is used.

```toml
[patch.crates-io]
winit = { git = "https://github.com/Kaan88/winit", rev = "<sha>" }
```

## API (`winit::platform::wayland::WindowExtWayland`)

- `set_content_margin(&self, margin: u32)`: logical px; 0 turns it off. No-op on X11.
- `content_margin_now(&self) -> u32`: the margin in force: the one set, or 0 (see states).

## Behaviour

| Window state | margin in force | surface (`inner_size()`, `Resized`) | `xdg_surface.set_window_geometry` |
|---|---|---|---|
| normal, winit has no visible CSD frame | the margin | content + 2 * margin | `(m, m, content_w, content_h)` |
| maximized, fullscreen, tiled on ANY edge | 0 | = geometry (what the compositor sent) | `(0, 0, w, h)` as stock |
| winit's own frame (sctk-adwaita) visible | 0 | stock | stock |

- **Sizes.** The compositor's configure sizes are window-geometry sizes (content); the surface is
  those plus `2 * margin`. `inner_size()`, `Resized` and `request_inner_size` (and the initial
  size from `with_inner_size`) are SURFACE sizes, as the app allocates its buffer from them.
  `set_min_inner_size` / `set_max_inner_size` stay CONTENT sizes (they are applied to
  `xdg_toplevel` min/max, which are window-geometry sizes), so an app's existing minimum keeps its
  meaning. `set_content_margin` called after the first configure keeps the surface size and
  carves the margin out of it (a 672x472 window becomes 600x400 content with margin 36).
- **A change of the margin in force** (maximize, restore, tile, fullscreen) comes with a configure
  whose state differs, which already produced a `Resized`; so does `set_content_margin` through the
  redraw it requests. The app re-reads `inner_size()` / `content_margin_now()`.
- **`wp_viewport` destination** (fractional scale) is the surface size.
- **Opaque region**: none when the window is transparent (as stock); not transparent: the content
  rectangle only (the margin is not opaque).
- **Input region**: with a margin in force, the content rectangle grown by 6 logical px
  (`RESIZE_BAND`) on every side, so the shadow lets clicks through to the window behind while an
  app can still start `drag_resize_window` from just outside its content. Margin 0: the region is
  reset to the whole surface (only if one was set).
- Tiling: sctk's `WindowConfigure::is_tiled()` is true only when all four `TILED_*` bits are set
  (a `contains`), so a half-screen tile is not "tiled" there; the patch uses `intersects`.
  (`is_stateless`, and with it the stock stateless-size fallback, is left as it was.)

The patch touches `src/platform/wayland.rs` (public trait), `src/platform_impl/linux/wayland/
window/mod.rs` (two forwarding methods) and `window/state.rs` (field `content_margin`, helpers
`margin_now`/`margin_for`/`shrink`/`input_rect`, and `inner_size`, `configure`, `resize`,
`request_inner_size`, `reload_transparency_hint` changed to use them; one unit test).
`WindowState::size` now means the window-geometry (content) size, `inner_size()` the surface.

## Rebasing onto a newer winit

1. `git fetch <upstream> tag vX.Y.Z; git checkout -b md-X.Y.Z vX.Y.Z; git cherry-pick <patch sha>`
   (the patch is one commit besides this file).
2. Conflicts are limited to the three files above; in `state.rs` re-check that every use of
   `self.size` is the geometry size and every place that reports a size to the app or the
   viewport goes through `inner_size()`: `grep -n "inner_size()\|self.size\|set_destination"`.
3. `cargo check --features wayland`, `cargo test --lib --features wayland input_region`, then run
   `crates/mdshared-ui/examples/shadow.rs` of Kaan88/markdown-shared-rs (`--features client-margin`)
   on a Wayland compositor: normal, maximized, tiled, restored.
4. Upstream: the proper form is an `xdg_surface` geometry offset on `WindowAttributes`/`Window`
   (and an input-region setter); this branch is the smallest version that works.

Not run: Windows/macOS/X11 builds (the change is under the Wayland cfgs), winit's full test suite
(only `cargo check --features wayland` and the unit test above; `cargo clippy` reports 4 errors
that are in stock winit 0.30.13, none in these files).
