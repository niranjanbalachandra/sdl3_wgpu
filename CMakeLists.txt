

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

# EMSDK needed for web
set(EMSDK ../emsdk)

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
GIT_REPOSITORY https://github.com/ocornut/imgui.git
GIT_TAG docking
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

add_executable(${SDL3_WGPU_EXE} main.cpp)

# Link the executable with imgui to provide headers and implementation
target_link_libraries(${SDL3_WGPU_EXE} PRIVATE imgui)

# Enable warning level compiler option only for IMGUI_EXAMPLE_SOURCE_FILES
if (MSVC)
    target_compile_options(${SDL3_WGPU_EXE} PUBLIC /W4) # warning level 4
else()
    target_compile_options(${SDL3_WGPU_EXE} PUBLIC -Wall) # -Wextra -Wpedantic
endif()

if(NOT EMSCRIPTEN) # WegGPU-Native settings
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
	
	target_compile_definitions(${SDL3_WGPU_EXE} PUBLIC "IMGUI_IMPL_WEBGPU_BACKEND_WGPU")
	target_include_directories(${SDL3_WGPU_EXE} PUBLIC ${IMGUI_WGPU_DIR}/include)
    target_link_libraries(${SDL3_WGPU_EXE} PRIVATE SDL3-static)
else() # Emscripten settings
    
	if(EMSCRIPTEN_VERSION VERSION_LESS "4.0.15")
        message(FATAL_ERROR "Using Emscripten with SDL3 needs Emscripten version >= 4.0.15")
    endif()
	
	set(CMAKE_EXECUTABLE_SUFFIX ".html")

    target_include_directories(imgui PUBLIC "${EMSDK}/upstream/emscripten/cache/ports/emdawnwebgpu/emdawnwebgpu_pkg/webgpu/include")
    target_include_directories(${SDL3_WGPU_EXE} PUBLIC "${EMSDK}/upstream/emscripten/cache/ports/emdawnwebgpu/emdawnwebgpu_pkg/webgpu/include")

    target_compile_options(${SDL3_WGPU_EXE} PRIVATE -std=gnu++17 "SHELL:--use-port=emdawnwebgpu")
    target_compile_definitions(${SDL3_WGPU_EXE} PRIVATE __EMSCRIPTEN__ IMGUI_IMPL_WEBGPU_BACKEND_DAWN)    

    # Ensure our local emscripten compatibility headers (and project root) are available
    target_include_directories(${SDL3_WGPU_EXE} PUBLIC ${CMAKE_CURRENT_LIST_DIR} ${CMAKE_CURRENT_LIST_DIR}/emscripten)

    target_compile_options(${SDL3_WGPU_EXE} PUBLIC "-sUSE_SDL=3")
    target_link_options(${SDL3_WGPU_EXE} PRIVATE
            "--use-port=emdawnwebgpu"
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
