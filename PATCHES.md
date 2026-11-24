# GLFW fork for Minecraft on Wayland

This branch is upstream GLFW **3.5.1** with a small set of patches that make
Minecraft (LWJGL 3) work well on Wayland, and that remove the fractional-scale
blur KWin produces.

## Patches

### 1. `Wayland: add IME/text-input support`

Imported from the [clear-code/glfw](https://github.com/clear-code/glfw)
`im-support` branch (tip `9af719e6`). It adds a cross-platform text input (IME)
API on top of GLFW: preedit/commit callbacks, IME status, preedit cursor
rectangles, and backends for Wayland (`text-input-unstable-v1` and `-v3`), X11
(XIM), Win32 (IMM) and macOS (`NSTextInputClient`).

Upstream GLFW 3.5.1 and current master do **not** have IME support, so this is
still needed. The import is a single squashed commit, so the original history
stays in clear-code/glfw.

### 2. `Wayland: don't raise errors for unsupported window management`

Wayland compositors own window icons and placement, so `glfwSetWindowIcon`,
`glfwGetWindowPos` and `glfwSetWindowPos` cannot be implemented. Upstream raises
`GLFW_FEATURE_UNAVAILABLE`, and LWJGL turns GLFW errors into exceptions;
Minecraft calls `glfwSetWindowIcon` during startup and then aborts.

This patch turns the three functions into documented no-ops and makes
`glfwGetWindowPos` report `(0, 0)` instead of leaving the output untouched.

### 3. `Wayland: implement glfwSetCursorPos with wp_pointer_warp_v1`

Binds the private `wp_pointer_warp_v1` global when the compositor provides it
(KWin, Mutter, ...) and implements `glfwSetCursorPos` using `warp_pointer`.
Without the protocol, or before the pointer entered the surface (no valid enter
serial), it still reports `GLFW_FEATURE_UNAVAILABLE`.

### 4. `Wayland: support KDE xx_fractional_scale_v2`

At a fractional scale the compositor has to round the logical surface size to
whole logical pixels. On a 2560x1600 output at 150% a fullscreen window is
1707x1067 logical, which is 2560.5x1600.5 device pixels. The compositor then
resamples the frame (soft image) and direct scanout uses a scaled hardware
plane (visible moire on 1px content).

`xx_fractional_scale_v2` (a KWin private protocol, see
`deps/wayland/xx-fractional-scale-v2.xml`) lets the client declare the
coordinate space its surface acts in. GLFW declares it to be pixels, so
2560 / 1.5 = 1706.666... logical and the device mapping stays exact.

GLFW's public API keeps using screen coordinates, so the conversion happens at
the protocol boundary: configure sizes and pointer positions are divided by the
scale, viewport destinations, size limits and cursor warps are multiplied by it.

Requirements and limitations:

- KWin must be started with `KWIN_WAYLAND_SUPPORT_XX_FRACTIONAL_SCALE_V2=1`
  (Plasma 6.8). Check with `wayland-info | grep xx_fractional_scale`.
- Only the Wayland backend, only when the window uses server-side decorations
  (the client-side fallback decoration coordinates are not converted).
- Enabled automatically whenever the compositor advertises the protocol.

### 5. `build: add development CMake presets`

Optional convenience presets (Wayland-only release build, mingw-w64 cross
build). Drop this commit if you don't want it in a published branch.

## Building

```sh
cmake -S . -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_SHARED_LIBS=ON \
    -DGLFW_BUILD_WAYLAND=ON \
    -DGLFW_BUILD_X11=OFF
cmake --build build
```

This produces `build/src/libglfw.so.3.5`.

## Using it with Minecraft

Point LWJGL at the patched library:

```
-Dorg.lwjgl.glfw.libname=/path/to/libglfw.so
```