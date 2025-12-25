# imgui_example_sdl3_wgpu

Small example project demonstrating Dear ImGui with SDL3 and WebGPU (WGPU-native or Emscripten).

This repository contains a minimal ImGui + SDL3 application that can be built for desktop using WGPU-native, or for the web using Emscripten.

## Repository layout

- `CMakeLists.txt` - main build configuration for desktop and Emscripten targets.
- `main.cpp` - example application entry.
- `shell_minimal.html` - minimal shell used for Emscripten builds.
- `build/` - desktop build artifacts (Visual Studio/msvc builds on Windows).
- `build_web/` - Emscripten/Ninja web build output.

## Prerequisites

- CMake >= 3.22
- A C++17-capable compiler (MSVC, clang, gcc)
- Git (for FetchContent)

For desktop (WGPU-native):
- Download a prebuilt WGPU-native SDK binary for your platform from: https://github.com/gfx-rs/wgpu-native/releases
- Unzip the downloaded SDK somewhere accessible. The top-level folder should contain `include/` and `lib/` (or equivalent) subfolders.

For Emscripten (Web):
- Install Emscripten SDK (emsdk) and activate a recent version (>= 4.0.15 recommended).
- Install Ninja build tool.

## Desktop build (WGPU-native)

1. Set `IMGUI_WGPU_DIR` to point to the WGPU-native SDK folder (the folder that contains `lib/` and `include/`). Example (from project root):

```powershell
cmake -B build -DIMGUI_WGPU_DIR="..\webgpu-native\v27.0.2.0\debug"
cmake --build build --config Debug
```

2. Executable will be placed in one of the following locations depending on generator/configuration:

- `build/Debug/example_sdl3_wgpu.exe`
- `build/example_sdl3_wgpu.exe`

Notes and tips:
- The CMake file forces SDL3 to build/static link on desktop builds (`BUILD_SHARED_LIBS OFF`).
- On Windows the native build links against a few OS libraries (e.g. `d3dcompiler`, `ws2_32`, `opengl32`, etc.).

## Web build (Emscripten)

1. Configure with `emcmake` and Ninja (from project root):

```bash
emcmake cmake -G Ninja -B build_web
cmake --build build_web --config Debug
emrun build_web/index.html
```

2. The Emscripten build produces an `index.html`/`index.js`/`index.wasm` set in `build_web/`.

Notes:
- The CMakeLists will detect Emscripten and apply the appropriate WebGPU flag. For modern Emscripten versions the default is `--use-port=emdawnwebgpu` (Dawn backend). Older Emscripten versions use `-sUSE_WEBGPU=1`.
- If you need to override the WebGPU port, pass `-DIMGUI_EMSCRIPTEN_WEBGPU_FLAG="--use-port=path/to/emdawnwebgpu.port.py"` to the CMake configure step.

Additional Emscripten notes (Windows):

- PowerShell may block running Emscripten's `.ps1` wrapper scripts by default due to execution policy. If you see an error like "emcmake.ps1 cannot be loaded because running scripts is disabled", either:
	- Run `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process` in the PowerShell session (temporary), or
	- Run the `emcmake.bat` (or use Git Bash / cmd.exe) to avoid the PowerShell wrapper being invoked.

- Newer Emscripten versions no longer accept the legacy `-sUSE_WEBGPU=1` flag; the supported approach is to use `--use-port=emdawnwebgpu` which is the default in this project. If you need the old behavior, install an older Emscripten release via `emsdk`.

- During development I added a local compatibility header at `emscripten/html5_webgpu.h` to bridge some header differences while switching modes. This file is a small, local shim — you can remove it once your SDK and port setup are stable.

## Development notes

## Using VS Code tasks

- The workspace includes several VS Code tasks under `.vscode/tasks.json` to streamline building and running both desktop and web targets.

- Common tasks:
	- **CMake: Configure (Desktop)** — configures the native desktop build (`cmake -S . -B build ...`).
	- **CMake: Build (Desktop)** — builds the native desktop binary.
	- **CMake: Configure (Emscripten)** — runs the Emscripten configure step using the `emcmake` wrapper. On Windows this task is set to call the bundled `emcmake.bat` via the `EMSDK` environment variable.
	- **CMake: Build (Emscripten)** — builds the web target (`build_web`). The configure task will pass `--use-port=emdawnwebgpu` by default.
	- **Run: Emscripten (emrun)** — launches `emrun` to serve `build_web/index.html` locally.

- On Windows ensure `EMSDK` environment variable points to your emsdk installation root (for example `C:\Users\you\emsdk`). The tasks reference `${env:EMSDK}` to call the `emcmake.bat`/`emrun.bat` wrappers. If you prefer, open a Git Bash or cmd.exe and run the `emcmake`/`emrun` commands manually.


- Dear ImGui sources are included via FetchContent from a fork (`https://github.com/BrutPitt/imgui.git`), and backend files for SDL3 + WGPU are built into the example.
- The project defines `IMGUI_EXAMPLE_SDL3_WGPU` for conditional compilation.

## Troubleshooting

- If the configure step cannot find the WGPU-native library, ensure `IMGUI_WGPU_DIR` points at the SDK root and contains `lib/` and `include/`.
- If building with Emscripten fails due to WebGPU flags, check your Emscripten SDK version; set `IMGUI_EMSCRIPTEN_WEBGPU_FLAG` explicitly if needed.

## License

This repository includes third-party code (Dear ImGui and SDL) under their respective licenses. See the `build/_deps/imgui-src/` folder for license details.
