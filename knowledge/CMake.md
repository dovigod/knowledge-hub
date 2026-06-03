---
id: 019e8d91-765c-7454-8231-179ac543e299
name: CMake
aliases:
  - CMakeLists
  - cmake
updated_at: '2026-06-03T12:59:41.021Z'
summary: >-
  A cross-platform build system generator that produces native build files
  (Makefiles, Ninja, Visual Studio projects) from declarative CMakeLists.txt
  configuration files.
sources:
  - 019e8d91-220d-763d-9a79-d85d7053d3bc
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# CMake

## Overview

CMake is a cross-platform, open-source build system generator widely used in C and C++ projects. Rather than building code directly, it reads declarative configuration files (`CMakeLists.txt`) and generates native build files appropriate to the host platform — Unix Makefiles, Ninja build files, Visual Studio solutions, Xcode projects, etc.

This indirection is what makes CMake "cross-platform": the same `CMakeLists.txt` describes the project once, and the generated build files take care of platform-specific details.

## Notes

### How it works (two phases)

1. **Configure / Generate phase** — You run `cmake` pointing at a source directory. CMake parses `CMakeLists.txt`, evaluates variables and conditionals, probes the toolchain (compiler, linker, available libraries), and writes out native build files into a build directory.
2. **Build phase** — You invoke the native build tool (`make`, `ninja`, `msbuild`, etc.) — or `cmake --build <dir>` as a portable wrapper — which actually compiles and links the project.

### Typical workflow

```bash
mkdir build && cd build
cmake ..              # configure: read CMakeLists.txt, generate build files
cmake --build .       # build: invoke native build tool
cmake --install .     # optional: install artifacts
```

Out-of-source builds (a separate `build/` directory) are the convention — keeps generated artifacts isolated from sources.

### Core CMakeLists.txt concepts

- `cmake_minimum_required(VERSION x.y)` — declare minimum CMake version
- `project(name LANGUAGES CXX)` — define the project
- `add_executable(target src1.cpp src2.cpp)` — build an executable
- `add_library(target STATIC|SHARED src.cpp)` — build a library
- `target_link_libraries(target PRIVATE other)` — link dependencies
- `target_include_directories(target PUBLIC include/)` — header search paths
- `find_package(Foo REQUIRED)` — locate external dependencies

### Why it's used

- Cross-platform: one config file works on Linux, macOS, Windows
- Toolchain-agnostic: switch between GCC, Clang, MSVC without rewriting build logic
- Dependency discovery via `find_package` and modern "imported targets"
- Industry standard for C++ open-source projects (LLVM, Qt, OpenCV, etc.)
- IDE integration: most C++ IDEs natively understand CMake projects

### Modern CMake (3.x+) idioms

- Prefer **target-based** commands (`target_link_libraries`, `target_include_directories`) over directory-scoped ones
- Use `PUBLIC`/`PRIVATE`/`INTERFACE` visibility to model dependency propagation
- Avoid global variables like `CMAKE_CXX_FLAGS`; set properties per-target instead

## Sources

- [[raw/conversations/019e8d91-220d-763d-9a79-d85d7053d3bc|019e8d91-220d-763d-9a79-d85d7053d3bc]]
