

# Building for desktop (WGPU-Native) with WGPU-Native:
#  1. cmake -B build
#  2. cmake --build build
# The resulting binary will be found at one of the following locations:
#   * build/Debug/sdl3_wgpu[.exe]
#   * build/sdl3_wgpu[.exe]

# Building for Emscripten:
#  1. Install Emscripten SDK following the instructions: https://emscripten.org/docs/getting_started/downloads.html
#  2. Install Ninja build system
#  3. emcmake cmake -G Ninja -B build_web
#  4. cmake --build build_web
#  5. emrun build_web/index.html

cmake_minimum_required(VERSION 3.22) # Requires CMake >= 3.22
project(SDL3_WGPU_EXE C CXX)

set(SDL3_WGPU_EXE sdl3_wgpu)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug CACHE STRING "" FORCE)
endif()

set(CMAKE_CXX_STANDARD 17) # Requires C++17

include(FetchContent)

# SDL3 - force static linking for desktop client
set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build SDL3 as static library" FORCE)
FetchContent_Declare(
SDL3
GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
GIT_TAG main
GIT_SHALLOW TRUE
)
FetchContent_MakeAvailable(SDL3)

# ImGui
FetchContent_Declare(
imgui
GIT_REPOSITORY https://github.com/BrutPitt/imgui.git
GIT_TAG master
GIT_SHALLOW TRUE
)
FetchContent_MakeAvailable(imgui)

# Create ImGui library target
set(IMGUI_DIR ${imgui_SOURCE_DIR})
add_library(imgui STATIC
${IMGUI_DIR}/imgui.cpp
${IMGUI_DIR}/imgui_demo.cpp
${IMGUI_DIR}/imgui_draw.cpp
${IMGUI_DIR}/imgui_tables.cpp
${IMGUI_DIR}/imgui_widgets.cpp
${IMGUI_DIR}/backends/imgui_impl_sdl3.cpp
${IMGUI_DIR}/backends/imgui_impl_wgpu.cpp
)
target_include_directories(imgui PUBLIC 
${IMGUI_DIR}
${IMGUI_DIR}/backends
)

# Ensure ImGui target sees SDL3 usage requirements (include dirs, etc.)
target_link_libraries(imgui PUBLIC SDL3-static)

if(EMSCRIPTEN)
    if(EMSCRIPTEN_VERSION VERSION_LESS "4.0.15")
        message(FATAL_ERROR "Using Emscripten with SDL3 needs Emscripten version >= 4.0.15")
    endif()
    # Default to the modern emdawnwebgpu port. You can override this by passing
    # `-DIMGUI_EMSCRIPTEN_WEBGPU_FLAG="-sUSE_WEBGPU=1"` to cmake if needed.
    if(NOT IMGUI_EMSCRIPTEN_WEBGPU_FLAG)
        set(IMGUI_EMSCRIPTEN_WEBGPU_FLAG "--use-port=emdawnwebgpu" CACHE STRING "Choose WebGPU implementation for Emscripten" FORCE)
    endif()

    add_compile_options(-sDISABLE_EXCEPTION_CATCHING=1 -DIMGUI_DISABLE_FILE_FUNCTIONS=1)
else() # Native/Desktop build
    
	set(WEBGPU_LINK_TYPE STATIC CACHE STRING "Link type for WebGPU" FORCE)
    FetchContent_Declare(
      webgpu
      GIT_REPOSITORY https://github.com/eliemichel/WebGPU-distribution.git
      GIT_TAG main
      GIT_SHALLOW TRUE
    )
    FetchContent_MakeAvailable(webgpu)
    target_link_libraries(imgui PUBLIC webgpu)
    target_compile_definitions(imgui PUBLIC IMGUI_IMPL_WEBGPU_BACKEND_WGPU)
    
endif()

add_executable(${SDL3_WGPU_EXE} main.cpp)

target_compile_definitions(${SDL3_WGPU_EXE} PUBLIC "IMGUI_EXAMPLE_SDL3_WGPU")

# Link the executable with imgui to provide headers and implementation
target_link_libraries(${SDL3_WGPU_EXE} PRIVATE imgui)

# Enable warning level compiler option only for IMGUI_EXAMPLE_SOURCE_FILES
if (MSVC)
    target_compile_options(${SDL3_WGPU_EXE} PUBLIC /W4) # warning level 4
else()
    target_compile_options(${SDL3_WGPU_EXE} PUBLIC -Wall) # -Wextra -Wpedantic
endif()

# IMGUI_IMPL_WEBGPU_BACKEND_DAWN/WGPU internal define is set according to:
#   EMSCRIPTEN: by used FLAG
#     --use-port=emdawnwebgpu --> IMGUI_IMPL_WEBGPU_BACKEND_DAWN enabled (+EMSCRIPTEN)
#     -sUSE_WEBGPU=1          --> IMGUI_IMPL_WEBGPU_BACKEND_WGPU enabled (+EMSCRIPTEN)
#   NATIVE: by used SDK installation directory
#     if IMGUI_DAWN_DIR is valid --> IMGUI_IMPL_WEBGPU_BACKEND_DAWN enabled
#     if IMGUI_WGPU_DIR is valid --> IMGUI_IMPL_WEBGPU_BACKEND_WGPU enabled

if(NOT EMSCRIPTEN) # WegGPU-Native settings
	target_compile_definitions(${SDL3_WGPU_EXE} PUBLIC "IMGUI_IMPL_WEBGPU_BACKEND_WGPU")
	target_include_directories(${SDL3_WGPU_EXE} PUBLIC ${IMGUI_WGPU_DIR}/include)
    target_link_libraries(${SDL3_WGPU_EXE} PRIVATE SDL3-static)
else() # Emscripten settings
    set(CMAKE_EXECUTABLE_SUFFIX ".html")

    # Try to locate WebGPU headers provided by emdawnwebgpu port or legacy emscripten headers
    # Prefer `webgpu/webgpu.h` (emdawnwebgpu port), fall back to `emscripten/html5_webgpu.h`.
    find_path(EMD_WEBGPU_H
        NAMES webgpu/webgpu.h
        HINTS ENV EMSDK
        PATHS "${EMSDK}/upstream/emscripten/system/include" "${EMSDK}/upstream/emscripten/system/include/webgpu"
        NO_DEFAULT_PATH
    )
    if(EMD_WEBGPU_H)
        get_filename_component(EMD_WEBGPU_DIR ${EMD_WEBGPU_H} DIRECTORY)
        target_include_directories(imgui PUBLIC ${EMD_WEBGPU_DIR})
        target_include_directories(${SDL3_WGPU_EXE} PUBLIC ${EMD_WEBGPU_DIR})
    endif()
    # If the port is available in the emscripten cache ports directory, add that include explicitly.
    if(NOT EMD_WEBGPU_H)
        # Try to derive the emscripten root from the CMake toolchain file location
        if(DEFINED CMAKE_TOOLCHAIN_FILE)
            get_filename_component(__ems_dir ${CMAKE_TOOLCHAIN_FILE} DIRECTORY)
            get_filename_component(__ems_dir ${__ems_dir} DIRECTORY)
            get_filename_component(__ems_dir ${__ems_dir} DIRECTORY)
            # __ems_dir should now point to .../upstream/emscripten
            set(EMD_PORT_INCLUDE "${__ems_dir}/cache/ports/emdawnwebgpu/emdawnwebgpu_pkg/webgpu/include")
            if(EXISTS "${EMD_PORT_INCLUDE}")
                target_include_directories(imgui PUBLIC ${EMD_PORT_INCLUDE})
                target_include_directories(${SDL3_WGPU_EXE} PUBLIC ${EMD_PORT_INCLUDE})
            endif()
        endif()
        # Also try common EMSDK location if the above did not find it
        if(NOT TARGET_INCLUDE_DIR AND EXISTS "C:/Users/niran/OneDrive/Desktop/Work/emsdk/upstream/emscripten/cache/ports/emdawnwebgpu/emdawnwebgpu_pkg/webgpu/include")
            target_include_directories(imgui PUBLIC "C:/Users/niran/OneDrive/Desktop/Work/emsdk/upstream/emscripten/cache/ports/emdawnwebgpu/emdawnwebgpu_pkg/webgpu/include")
            target_include_directories(${SDL3_WGPU_EXE} PUBLIC "C:/Users/niran/OneDrive/Desktop/Work/emsdk/upstream/emscripten/cache/ports/emdawnwebgpu/emdawnwebgpu_pkg/webgpu/include")
        endif()
    endif()

    find_path(EM_HTML5_WEBGPU_H
        NAMES emscripten/html5_webgpu.h
        HINTS ENV EMSDK
        PATHS "${EMSDK}/upstream/emscripten/system/include" "${EMSDK}/upstream/emscripten/system/include/emscripten"
        NO_DEFAULT_PATH
    )
    if(EM_HTML5_WEBGPU_H)
        get_filename_component(EM_HTML5_WEBGPU_DIR ${EM_HTML5_WEBGPU_H} DIRECTORY)
        target_include_directories(${SDL3_WGPU_EXE} PUBLIC ${EM_HTML5_WEBGPU_DIR})
    endif()

    if("${IMGUI_EMSCRIPTEN_WEBGPU_FLAG}" MATCHES "emdawnwebgpu")
        target_compile_options(${SDL3_WGPU_EXE} PRIVATE -std=gnu++17 "SHELL:--use-port=emdawnwebgpu")
        target_compile_definitions(${SDL3_WGPU_EXE} PRIVATE __EMSCRIPTEN__ IMGUI_IMPL_WEBGPU_BACKEND_DAWN)
    else()
        target_compile_definitions(${SDL3_WGPU_EXE} PUBLIC "IMGUI_IMPL_WEBGPU_BACKEND_WGPU")
    endif()
    message(STATUS "Using ${IMGUI_EMSCRIPTEN_WEBGPU_FLAG} WebGPU implementation")

    # Ensure our local emscripten compatibility headers (and project root) are available
    target_include_directories(${SDL3_WGPU_EXE} PUBLIC ${CMAKE_CURRENT_LIST_DIR} ${CMAKE_CURRENT_LIST_DIR}/emscripten)

    target_compile_options(${SDL3_WGPU_EXE} PUBLIC "-sUSE_SDL=3")
    target_link_options(${SDL3_WGPU_EXE} PRIVATE
            "${IMGUI_EMSCRIPTEN_WEBGPU_FLAG}"
            "-sUSE_SDL=3"
            "-sWASM=1"
            "-sASYNCIFY=1"
            "-sALLOW_MEMORY_GROWTH=1"
            "-sNO_EXIT_RUNTIME=0"
            "-sASSERTIONS=1"
            "-sDISABLE_EXCEPTION_CATCHING=1"
#            "-sNO_FILESYSTEM=1"
            "--shell-file=${CMAKE_CURRENT_LIST_DIR}/shell_minimal.html"
    )
    set_target_properties(${SDL3_WGPU_EXE} PROPERTIES OUTPUT_NAME "index")
endif()
