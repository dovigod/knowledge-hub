---
id: 019e8d91-7130-7206-8544-52196c88c1bf
title: C++ CMake 빌드 시스템
topics:
  - c++
  - cmake
  - 빌드 시스템
sources:
  - 019e8d91-220d-763d-9a79-d85d7053d3bc
  - 019e8da5-ab9c-713f-b09d-4c32f39c0ef5
created_at: '2026-06-03T12:59:39.695Z'
updated_at: '2026-06-03T13:22:32.414Z'
---
## Overview

CMake is a **build system generator** for [[C++]] projects. It doesn't compile directly — instead, it generates platform-appropriate build files (Makefile, Ninja, Visual Studio solutions, etc.).

## Why It's Needed

[[C++]] has no build/package system in the language standard (no equivalent to JS's npm or Rust's cargo). Using the compiler directly:

```bash
g++ -std=c++20 -I./include -L./lib main.cpp utils.cpp -lfmt -o app
```

When files number in the hundreds, compilers differ by platform (Linux/macOS/Windows), and external library paths are inconsistent, management becomes impossible. [[CMake]] abstracts this.

## How It Works

```
CMakeLists.txt  →  [configure]  →  Makefile/Ninja  →  [build]  →  executable
   (you write)      cmake generates             make/ninja compiles
```

```bash
cmake -B build          # 1. configure: generate build files in build/
cmake --build build     # 2. build: actual compilation
```

## Basic CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(myapp CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Executable
add_executable(myapp src/main.cpp src/utils.cpp)

# Header path
target_include_directories(myapp PRIVATE include)

# Find and link external library
find_package(fmt REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

## Core Concepts

**1. Target-centric design** — The heart of modern [[CMake]] (3.x). Executables and libraries are "targets", and all settings attach to targets:

```cmake
add_library(mylib STATIC src/lib.cpp)        # Static library target
add_executable(app src/main.cpp)             # Executable target
target_link_libraries(app PRIVATE mylib)     # Dependency linking
```

**2. PRIVATE / PUBLIC / INTERFACE** — Dependency propagation scope:

| Keyword | Used by self | Propagated to linkers of self |
|---|---|---|
| `PRIVATE` | ✅ | ❌ |
| `PUBLIC` | ✅ | ✅ |
| `INTERFACE` | ❌ | ✅ (used for header-only libraries) |

Example: if `mylib`'s public headers include `fmt`, use `PUBLIC`; if only the implementation (.cpp) uses it, use `PRIVATE`.

**3. Fetching external libraries**

```cmake
# Find packages installed on the system
find_package(Boost REQUIRED COMPONENTS filesystem)

# Download source directly and build (dependency vendoring)
include(FetchContent)
FetchContent_Declare(json
  GIT_REPOSITORY https://github.com/nlohmann/json
  GIT_TAG v3.11.3)
FetchContent_MakeAvailable(json)
target_link_libraries(app PRIVATE nlohmann_json::nlohmann_json)
```

When used with package managers ([[vcpkg]], [[Conan]]), `find_package` locates libraries installed by them.

**4. Build types**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug      # -g, no optimization
cmake -B build -DCMAKE_BUILD_TYPE=Release    # -O3, no debug info
```

**5. Subdirectory structure** — Large projects separate by module:

```
project/
├── CMakeLists.txt          # add_subdirectory(src), etc.
├── src/CMakeLists.txt
├── libs/core/CMakeLists.txt
└── tests/CMakeLists.txt
```

## Frequently Used Commands

```bash
cmake -B build -G Ninja                  # Use Ninja generator (faster than make)
cmake --build build -j8                  # Parallel build
cmake --build build --target myapp       # Specific target only
ctest --test-dir build                   # Run tests (CTest integration)
cmake --install build --prefix /usr/local # Install
```

## Good to Know

- **Use modern [[CMake]] style**: Use `target_*` family commands. Global variables (`include_directories()`, directly manipulating `CMAKE_CXX_FLAGS`) are outdated and break dependency tracking.
- **Out-of-source builds**: Always put build artifacts in a separate directory like `build/`. Keep the source tree clean.
- **`compile_commands.json`**: Generated with `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`, this lets LSPs like [[clangd]] understand the project.
- [[CMake]]'s own language honestly has poor syntax (string-based, many subtle pitfalls), but it's effectively the C++ ecosystem standard with no real alternative.

---

## 한국어

### 개요

[[CMake]]는 [[C++]] 프로젝트의 **빌드 시스템 생성기(build system generator)**입니다. 직접 컴파일하는 게 아니라, 플랫폼에 맞는 빌드 파일(Makefile, Ninja, Visual Studio 솔루션 등)을 생성해주는 도구입니다.

### 왜 필요한가

[[C++]]은 언어 표준에 빌드/패키지 시스템이 없습니다 (JS의 npm, Rust의 cargo 같은 게 없음). 컴파일러를 직접 쓰면:

```bash
g++ -std=c++20 -I./include -L./lib main.cpp utils.cpp -lfmt -o app
```

파일이 수백 개가 되고, 플랫폼별(Linux/macOS/Windows) 컴파일러가 다르고, 외부 라이브러리 경로가 제각각이면 관리가 불가능해집니다. [[CMake]]가 이걸 추상화합니다.

### 동작 흐름

```
CMakeLists.txt  →  [configure]  →  Makefile/Ninja  →  [build]  →  실행파일
   (당신이 작성)      cmake가 생성              make/ninja가 컴파일
```

```bash
cmake -B build          # 1. configure: build/ 에 빌드 파일 생성
cmake --build build     # 2. build: 실제 컴파일
```

### 기본 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(myapp CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 실행 파일
add_executable(myapp src/main.cpp src/utils.cpp)

# 헤더 경로
target_include_directories(myapp PRIVATE include)

# 외부 라이브러리 찾아서 링크
find_package(fmt REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

### 핵심 개념

**1. 타겟(Target) 중심 설계** — 모던 [[CMake]](3.x)의 핵심. 실행 파일이나 라이브러리가 "타겟"이고, 모든 설정을 타겟에 붙입니다:

```cmake
add_library(mylib STATIC src/lib.cpp)        # 정적 라이브러리 타겟
add_executable(app src/main.cpp)             # 실행 파일 타겟
target_link_libraries(app PRIVATE mylib)     # 의존성 연결
```

**2. PRIVATE / PUBLIC / INTERFACE** — 의존성 전파 범위:

| 키워드 | 자신이 사용 | 나를 링크한 쪽에 전파 |
|---|---|---|
| `PRIVATE` | ✅ | ❌ |
| `PUBLIC` | ✅ | ✅ |
| `INTERFACE` | ❌ | ✅ (헤더온리 라이브러리에 사용) |

예: `mylib`의 공개 헤더가 `fmt`를 include하면 `PUBLIC`, 구현(.cpp)에서만 쓰면 `PRIVATE`.

**3. 외부 라이브러리 가져오기**

```cmake
# 시스템에 설치된 패키지 찾기
find_package(Boost REQUIRED COMPONENTS filesystem)

# 소스를 직접 다운로드해서 빌드 (의존성 벤더링)
include(FetchContent)
FetchContent_Declare(json
  GIT_REPOSITORY https://github.com/nlohmann/json
  GIT_TAG v3.11.3)
FetchContent_MakeAvailable(json)
target_link_libraries(app PRIVATE nlohmann_json::nlohmann_json)
```

패키지 매니저([[vcpkg]], [[Conan]])와 함께 쓰면 `find_package`가 그쪽에서 설치한 라이브러리를 찾아줍니다.

**4. 빌드 타입**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug      # -g, 최적화 없음
cmake -B build -DCMAKE_BUILD_TYPE=Release    # -O3, 디버그 정보 없음
```

**5. 서브디렉토리 구조** — 큰 프로젝트는 모듈별로 분리:

```
project/
├── CMakeLists.txt          # add_subdirectory(src) 등
├── src/CMakeLists.txt
├── libs/core/CMakeLists.txt
└── tests/CMakeLists.txt
```

### 자주 쓰는 명령어 정리

```bash
cmake -B build -G Ninja                  # Ninja 제너레이터 사용 (make보다 빠름)
cmake --build build -j8                  # 병렬 빌드
cmake --build build --target myapp       # 특정 타겟만
ctest --test-dir build                   # 테스트 실행 (CTest 통합)
cmake --install build --prefix /usr/local # 설치
```

### 알아두면 좋은 것

- **모던 [[CMake]] 스타일을 쓰세요**: `target_*` 계열 명령 사용. 전역 변수(`include_directories()`, `CMAKE_CXX_FLAGS` 직접 조작)는 구식이고 의존성 추적이 깨집니다.
- **out-of-source 빌드**: 빌드 산출물은 항상 `build/` 같은 별도 디렉토리에. 소스 트리를 더럽히지 않습니다.
- **`compile_commands.json`**: `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`으로 생성하면 [[clangd]] 같은 LSP가 프로젝트를 이해할 수 있게 됩니다.
- [[CMake]] 자체 언어는 솔직히 문법이 좋지 않지만(문자열 기반, 미묘한 함정 많음), 사실상 [[C++]] 생태계 표준이라 대안이 없는 수준입니다.
