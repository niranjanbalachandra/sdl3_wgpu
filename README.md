# sdl3_wgpu

Small example project demonstrating Dear ImGui with SDL3 and WebGPU (WGPU-native or Emscripten).

This repository contains a minimal ImGui + SDL3 application that can be built for desktop using WGPU-native, or for the web using Emscripten.

## Prerequisites

- `cmake` >= 3.22
- A C++17 capable compiler (MSVC, clang, gcc)
- `git` (FetchContent uses Git)

Desktop (native WGPU):
- Download a prebuilt WGPU-native SDK for your platform (github.com/gfx-rs/wgpu-native/releases).
- The SDK folder should contain `include/` and `lib/` (or platform-equivalent layout).

Web (Emscripten):
- Install Emscripten SDK (`emsdk`) and activate a recent toolchain.
- Install `ninja` for fast builds.

## Desktop build (example)

From the project root, configure and build (example Windows PowerShell):

```powershell
cmake -B build -DIMGUI_WGPU_DIR="C:\path\to\webgpu-native-sdk"
cmake --build build --config Debug
```

Built executable locations (depending on generator/config):

- `build/Debug/example_sdl3_wgpu.exe`
- `build/example_sdl3_wgpu.exe`

Notes:

- The CMake configuration builds SDL3 statically for desktop builds (`BUILD_SHARED_LIBS=OFF`).
- On Windows the native build may link OS libraries like `d3dcompiler`, `ws2_32`, and `opengl32` as needed.

## Web build (Emscripten example)

Configure with `emcmake` and `Ninja`, then build and run locally with `emrun`:

```bash
emcmake cmake -G Ninja -B build_web -S .
cmake --build build_web --config Debug
```

Build artifacts: `build_web/index.html`, `build_web/index.js`, `build_web/index.wasm`.

## Run (Desktop)

After a successful build, run the example binary from the `build` folder, for example:

```powershell
.\build\Debug\example_sdl3_wgpu.exe
```

## Run (Emscripten)

After a successful build, run the example binary from the `build` folder, for example:

```powershell
emrun build_web/index.html
```

## VS Code tasks

- **CMake: Configure (Desktop)**: configure desktop build (uses `-B build`).
- **CMake: Build (Desktop)**: build desktop binary.
- **CMake: Configure (Emscripten)**: runs `emcmake`/`emcmake.bat` (Windows) and configures `build_web`.
- **CMake: Build (Emscripten)**: builds the Emscripten target.
- **Run: Emscripten (emrun)**: serves `build_web/index.html` using `emrun`.


## Troubleshooting

## License & Third-party

This repository includes Dear ImGui and SDL sources under their respective licenses. See `build/_deps/imgui-src/` for license texts.

## Contact

If you need help building or running the example, open an issue or contact the repository maintainer.
