---
id: 019e8d91-7130-7206-8544-52196c88c1bf
title: CMake 내부 동작 원리
topics:
  - c++
  - cmake
  - 빌드 시스템
sources:
  - 019e8d91-220d-763d-9a79-d85d7053d3bc
  - 019e8da5-ab9c-713f-b09d-4c32f39c0ef5
  - 019e8da6-4386-7787-acfe-527b94c3a704
created_at: '2026-06-03T12:59:39.695Z'
updated_at: '2026-06-03T13:24:24.839Z'
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

## How CMake Works Internally

Broadly divided into **Configure → Generate → Build**, three stages.

```
CMakeLists.txt
     │
     ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Configure   │ ──▶ │  Generate   │ ──▶ │   Build     │
│ Interpret    │     │ Emit build  │     │ make/ninja  │
│ script       │     │ files       │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
  CMakeCache.txt      build.ninja or       .o → link →
  Target graph        Makefile             executable
  construction
```

### Stage 1: Configure

When you run `cmake -B build`, [[CMake]] interprets `CMakeLists.txt` **top-to-bottom like a script**.

**(1) Compiler detection and verification**

First, it finds the compiler on the system, then actually compiles a small test program to confirm it works:

```
-- The CXX compiler identification is AppleClang 15.0.0
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/clang++ - works
```

The result (compiler path, supported flags, ABI info) is all stored in the cache.

**(2) Script execution → in-memory target graph construction**

Commands like `add_executable` and `target_link_libraries` do not immediately build anything; they **accumulate target objects and properties in memory**:

```cmake
add_library(mylib STATIC a.cpp)            # Create target "mylib"
target_compile_definitions(mylib PUBLIC X) # Add X to mylib's properties
add_executable(app main.cpp)               # Create target "app"
target_link_libraries(app PRIVATE mylib)   # Add edge to graph: app → mylib
```

What is created at this point is a **directed graph (DAG)**:

```
app ──link──▶ mylib
 │              │
 └─ main.cpp    └─ a.cpp, PUBLIC define X
```

`PUBLIC`/`PRIVATE`/`INTERFACE` operate here — after configure finishes, [[CMake]] walks the graph and **propagates usage requirements**. `mylib`'s `PUBLIC` define `X` flows along the edge into `app`'s compile flags.

**(3) How `find_package` works**

```cmake
find_package(fmt REQUIRED)
```

This is not magic but **file lookup**. [[CMake]] searches a defined path order (`CMAKE_PREFIX_PATH`, system paths, etc.) for a `fmtConfig.cmake` file and executes it. Inside that file an **IMPORTED target** called `fmt::fmt` is defined — a fake target carrying the location and header paths of an already-built library. [[vcpkg]]/[[Conan]] integration is ultimately about inserting their install folders into this search path.

**(4) CMakeCache.txt**

The configure result is stored as key-value in `build/CMakeCache.txt`:

```
CMAKE_CXX_COMPILER:FILEPATH=/usr/bin/clang++
CMAKE_BUILD_TYPE:STRING=Debug
fmt_DIR:PATH=/opt/homebrew/lib/cmake/fmt
```

This is to skip expensive work like compiler detection on the next configure. 90% of "I changed a setting but it didn't take effect" is because an old value remains in the cache, which is why the remedy is to delete `build/` and reconfigure.

### Stage 2: Generate

The completed target graph is converted into actual build files by a **generator**.

- `-G Ninja` → `build.ninja`
- `-G "Unix Makefiles"` → `Makefile`
- `-G "Visual Studio 17 2022"` → `.sln` / `.vcxproj`

At this point the graph is unfolded into concrete compile commands. For example, the `app → mylib` edge becomes the following in a ninja file:

```ninja
build CMakeFiles/mylib.dir/a.cpp.o: CXX_COMPILER a.cpp
build libmylib.a: CXX_STATIC_LIBRARY CMakeFiles/mylib.dir/a.cpp.o
build CMakeFiles/app.dir/main.cpp.o: CXX_COMPILER main.cpp
  DEFINES = -DX                          # ← Result of PUBLIC propagation embedded here
build app: CXX_EXECUTABLE main.cpp.o | libmylib.a
```

[[CMake]]'s role ends here. **[[CMake]] does not compile.**

### Stage 3: Build

`cmake --build build` is actually just a wrapper that invokes ninja or make. The real work is done by the build tool:

**(1) Dependency-based incremental build**

ninja/make compares file timestamps and **recompiles only what changed**. The key is header dependency: by passing the `-MD` flag to the compiler, it extracts a list (a `.d` file) of "which headers this .cpp included":

```
main.cpp.o: main.cpp utils.h config.h
```

So even if you only modify `utils.h`, every .cpp that included it is precisely recompiled.

**(2) Auto-trigger reconfigure**

The generated build files also contain a rule that says "if CMakeLists.txt changes, re-run cmake." So after modifying CMakeLists.txt, just running build will automatically re-run configure.

**(3) Compile → link**

Each `.cpp` is independently compiled into a `.o` (object file) (why parallelization is possible), and finally the linker bundles all `.o` files and libraries into the executable. It proceeds in topological order of the target graph — `mylib` first, `app` link last.

### In one sentence

> [[CMake]] interprets CMakeLists.txt, **builds a target dependency graph in memory**, propagates properties along graph edges, then **translates** the result into the native build tool's language (ninja/make) — you can think of it as a translating compiler. The actual build is done by ninja/make, the actual compilation by clang/gcc.

This is why distinguishing stages when debugging matters:
- configure error → CMakeLists.txt syntax, `find_package` failure, cache issue
- build error → compile error (code problem) or link error (missing library)

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

### CMake가 내부적으로 동작하는 방식

크게 **Configure → Generate → Build** 3단계로 나뉩니다.

```
CMakeLists.txt
     │
     ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Configure   │ ──▶ │  Generate   │ ──▶ │   Build     │
│ 스크립트 해석  │     │ 빌드파일 생성  │     │ make/ninja  │
└─────────────┘     └─────────────┘     └─────────────┘
  CMakeCache.txt      build.ninja 또는      .o → 링크 →
  타겟 그래프 구성       Makefile            실행파일
```

#### 1단계: Configure (구성)

`cmake -B build`를 실행하면 [[CMake]]는 `CMakeLists.txt`를 **위에서 아래로 스크립트처럼 해석**합니다.

**(1) 컴파일러 탐지와 검증**

가장 먼저 시스템에서 컴파일러를 찾고, 실제로 동작하는지 작은 테스트 프로그램을 컴파일해봅니다:

```
-- The CXX compiler identification is AppleClang 15.0.0
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/clang++ - works
```

이 결과(컴파일러 경로, 지원 플래그, ABI 정보)는 전부 캐시에 저장됩니다.

**(2) 스크립트 실행 → 인메모리 타겟 그래프 구성**

`add_executable`, `target_link_libraries` 같은 명령은 즉시 뭔가를 빌드하는 게 아니라, **메모리상에 타겟 객체와 속성을 쌓아가는 것**입니다:

```cmake
add_library(mylib STATIC a.cpp)            # 타겟 "mylib" 생성
target_compile_definitions(mylib PUBLIC X) # mylib의 속성에 X 추가
add_executable(app main.cpp)               # 타겟 "app" 생성
target_link_libraries(app PRIVATE mylib)   # 그래프에 간선 추가: app → mylib
```

이 시점에 만들어지는 건 **방향 그래프(DAG)**입니다:

```
app ──link──▶ mylib
 │              │
 └─ main.cpp    └─ a.cpp, PUBLIC define X
```

`PUBLIC`/`PRIVATE`/`INTERFACE`가 여기서 작동합니다 — configure가 끝난 뒤 [[CMake]]는 그래프를 따라가며 **사용 요구사항(usage requirements)을 전파**합니다. `mylib`의 `PUBLIC` define `X`는 간선을 타고 `app`의 컴파일 플래그로 흘러 들어갑니다.

**(3) `find_package`의 동작**

```cmake
find_package(fmt REQUIRED)
```

이건 마법이 아니라 **파일 탐색**입니다. [[CMake]]가 정해진 경로 순서(`CMAKE_PREFIX_PATH`, 시스템 경로 등)에서 `fmtConfig.cmake` 파일을 찾아 실행합니다. 그 파일 안에서 `fmt::fmt`라는 **IMPORTED 타겟**이 정의되는데, 이미 빌드된 라이브러리의 위치와 헤더 경로를 담은 가짜 타겟입니다. [[vcpkg]]/[[Conan]] 연동이란 결국 이 탐색 경로에 자기네 설치 폴더를 끼워넣는 것입니다.

**(4) CMakeCache.txt**

configure 결과는 `build/CMakeCache.txt`에 키-값으로 저장됩니다:

```
CMAKE_CXX_COMPILER:FILEPATH=/usr/bin/clang++
CMAKE_BUILD_TYPE:STRING=Debug
fmt_DIR:PATH=/opt/homebrew/lib/cmake/fmt
```

다음번 configure 때 컴파일러 탐지 같은 비싼 작업을 건너뛰기 위함입니다. "설정 바꿨는데 반영이 안 돼요"의 90%는 캐시에 옛 값이 남아있어서이고, 해법이 `build/` 삭제 후 재구성인 이유입니다.

#### 2단계: Generate (생성)

완성된 타겟 그래프를 **제너레이터**가 실제 빌드 파일로 변환합니다.

- `-G Ninja` → `build.ninja`
- `-G "Unix Makefiles"` → `Makefile`
- `-G "Visual Studio 17 2022"` → `.sln` / `.vcxproj`

이때 그래프가 구체적인 컴파일 명령으로 펼쳐집니다. 예를 들어 `app → mylib` 간선은 ninja 파일에서 이렇게 됩니다:

```ninja
build CMakeFiles/mylib.dir/a.cpp.o: CXX_COMPILER a.cpp
build libmylib.a: CXX_STATIC_LIBRARY CMakeFiles/mylib.dir/a.cpp.o
build CMakeFiles/app.dir/main.cpp.o: CXX_COMPILER main.cpp
  DEFINES = -DX                          # ← PUBLIC 전파 결과가 여기 박힘
build app: CXX_EXECUTABLE main.cpp.o | libmylib.a
```

[[CMake]]의 역할은 여기서 끝입니다. **[[CMake]]는 컴파일하지 않습니다.**

#### 3단계: Build (빌드)

`cmake --build build`는 사실 ninja나 make를 호출하는 래퍼일 뿐입니다. 실제 작업은 빌드 도구가 합니다:

**(1) 의존성 기반 증분 빌드**

ninja/make는 파일 타임스탬프를 비교해 **바뀐 것만 다시 컴파일**합니다. 핵심은 헤더 의존성인데, 컴파일러에 `-MD` 플래그를 줘서 "이 .cpp가 어떤 헤더들을 include했는지" 목록(`.d` 파일)을 뽑아냅니다:

```
main.cpp.o: main.cpp utils.h config.h
```

그래서 `utils.h`만 고쳐도 그걸 include한 모든 .cpp가 정확히 재컴파일됩니다.

**(2) 재구성 자동 트리거**

생성된 빌드 파일에는 "CMakeLists.txt가 바뀌면 cmake를 다시 실행하라"는 규칙도 들어 있습니다. 그래서 CMakeLists.txt 수정 후 그냥 빌드만 해도 configure가 자동으로 다시 돕니다.

**(3) 컴파일 → 링크**

각 `.cpp`는 독립적으로 `.o`(오브젝트 파일)로 컴파일되고(병렬화 가능한 이유), 마지막에 링커가 모든 `.o`와 라이브러리를 묶어 실행 파일을 만듭니다. 타겟 그래프의 위상 순서대로 진행됩니다 — `mylib`이 먼저, `app` 링크가 마지막.

#### 전체를 한 문장으로

> [[CMake]]는 CMakeLists.txt를 해석해 **타겟 의존성 그래프를 메모리에 만들고**, 속성을 그래프 간선을 따라 전파한 뒤, 그 결과를 네이티브 빌드 도구의 언어(ninja/make)로 **번역해주는 컴파일러**라고 볼 수 있습니다. 실제 빌드는 ninja/make가, 실제 컴파일은 clang/gcc가 합니다.

이래서 디버깅할 때 단계를 구분하는 게 중요합니다:
- configure 에러 → CMakeLists.txt 문법, find_package 실패, 캐시 문제
- build 에러 → 컴파일 에러(코드 문제) 또는 링크 에러(라이브러리 누락)
