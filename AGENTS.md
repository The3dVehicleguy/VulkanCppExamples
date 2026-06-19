# AGENTS.md

## Cursor Cloud specific instructions

This is a collection of ~130 standalone Vulkan C++20 example executables (one CMake
target per example). Build tooling is CMake + Ninja; third-party deps (`glfw`, `glm`,
`tinygltf`) are git submodules. See `README.md` for the example catalog and `CMakeLists.txt`
for the build wiring.

### Environment already provisioned (via the startup update script + VM snapshot)
- System packages are preinstalled in the snapshot: Vulkan loader/headers + validation
  layers, `glslc` (GLSL→SPIR-V), `mesa-vulkan-drivers` (lavapipe software renderer),
  GLFW build deps (X11/Wayland), OpenGL headers, `g++-13`, `xvfb`, `ffmpeg`, `imagemagick`.
- CMake 3.31.6 is installed at `/opt/cmake-3.31.6-linux-x86_64` and symlinked into
  `/usr/local/bin` (the repo requires CMake >= 3.31; Ubuntu's apt cmake is 3.28 and too old).
- The startup update script runs `git submodule update --init --recursive`.

### No GPU — render with the lavapipe software Vulkan driver
This VM has no `/dev/dri` GPU. Vulkan works through Mesa's `lavapipe` (CPU) ICD, which is
auto-selected; `vulkaninfo` reports device `llvmpipe ... PHYSICAL_DEVICE_TYPE_CPU`.
Examples run correctly but slowly (software rasterization). Validation layers print a
benign `VUID-VkDeviceCreateInfo-pNext-pNext` warning about Vulkan 1.1 feature structs;
this does not stop the examples from rendering.

### Non-obvious build gotcha: glm include path
The pinned `glm` submodule keeps its headers at `ThirdParty/glm/glm/*.hpp`, but
`CMakeLists.txt` adds `ThirdParty/glm/include` to the include path (does not exist for
this commit). Configure with the glm repo root added to the compiler include path so
`#include <glm/glm.hpp>` resolves. Do NOT edit tracked files; pass it as a flag:

```bash
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_C_COMPILER=gcc-13 -DCMAKE_CXX_COMPILER=g++-13 \
  -DCMAKE_CXX_FLAGS="-I$PWD/ThirdParty/glm"
ninja -C build                 # builds all ~130 examples + compiles shaders
ninja -C build GetDeviceInfo   # or build a single target
```

Executables land in `bin/Debug/`. Shaders are compiled to SPIR-V into
`Shaders/<...>/glsl/spirv/` at build time by `glslc`.

### Running windowed examples (need a virtual X display)
Most examples open a GLFW window and present a swapchain, so they need an X display and
`XDG_RUNTIME_DIR`. `GetDeviceInfo` is the only one that runs without a window.

```bash
export XDG_RUNTIME_DIR=/tmp/xdg-runtime && mkdir -p "$XDG_RUNTIME_DIR" && chmod 700 "$XDG_RUNTIME_DIR"
Xvfb :99 -screen 0 1280x720x24 -ac +extension GLX +render -noreset &
sleep 1
export DISPLAY=:99
./bin/Debug/ClearScreenWithColor        # windowed; loops until window close / ESC
./bin/Debug/GetDeviceInfo               # console only, exits on its own
```

Capture rendered output from the virtual display with ImageMagick (`import -window <id>`)
or record video with `ffmpeg -f x11grab -video_size 1280x720 -i :99.0+0+0 ...`. The Cursor
`computerUse`/`RecordScreen` tools cannot see the Xvfb `:99` display.

### Tests
`ENABLE_EXAMPLE_TESTS` is ON and registers a CTest entry per example, but each test simply
launches the example. Windowed examples loop until the window is closed, so a full
`ctest` run will hang. Run individual non-interactive targets (e.g. `GetDeviceInfo`)
instead of the whole suite.
