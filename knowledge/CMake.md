---
id: 019e8d91-765c-7454-8231-179ac543e299
name: CMake
aliases:
  - CMake
  - CMakeLists
  - cmake
  - cmakelists.txt
updated_at: '2026-06-07T08:10:00.258Z'
summary: >-
  A cross-platform build system generator that produces native build files
  (Makefiles, Ninja, Visual Studio projects) from declarative CMakeLists.txt
  configuration files.
sources:
  - 019e8d91-220d-763d-9a79-d85d7053d3bc
  - 019e8da5-ab9c-713f-b09d-4c32f39c0ef5
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# CMake

## Overview

CMake is a cross-platform, open-source **build system generator** widely used in C and C++ projects. Rather than building code directly, it reads declarative configuration files (`CMakeLists.txt`) and generates native build files appropriate to the host platform — Unix Makefiles, Ninja build files, Visual Studio solutions, Xcode projects, etc.

> [!note] Why "generator"?
> CMake doesn't compile anything itself. The same `CMakeLists.txt` describes the project once, and the generated build files take care of platform-specific details — that's what makes it cross-platform.

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

> [!tip] Out-of-source builds
> Keeping generated artifacts in a separate `build/` directory (rather than alongside sources) is the convention. Makes cleanup trivial — just delete the directory.

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

> [!warning] Avoid legacy directory-scoped commands
> Prefer **target-based** commands (`target_link_libraries`, `target_include_directories`) over directory-scoped ones like `include_directories` or global `CMAKE_CXX_FLAGS`. Set properties per-target instead.

- Use `PUBLIC`/`PRIVATE`/`INTERFACE` visibility to model dependency propagation
- Imported targets (e.g. `Foo::Foo` from `find_package(Foo)`) carry their own include paths and link flags — just `target_link_libraries` and you're done

---

## 한국어

### 개요

CMake는 C 및 C++ 프로젝트에서 널리 사용되는 크로스 플랫폼 오픈소스 **빌드 시스템 생성기(build system generator)** 다. 코드를 직접 빌드하는 것이 아니라, 선언적 설정 파일인 `CMakeLists.txt`를 읽어 호스트 플랫폼에 맞는 네이티브 빌드 파일(Unix Makefile, Ninja, Visual Studio 솔루션, Xcode 프로젝트 등)을 생성한다.

> [!note] 왜 "생성기"인가?
> CMake는 직접 컴파일을 하지 않는다. 동일한 `CMakeLists.txt`로 프로젝트를 한 번 기술하면, 생성된 빌드 파일이 플랫폼별 세부 사항을 처리한다 — 이것이 크로스 플랫폼인 이유다.

### 노트

#### 동작 방식 (두 단계)

1. **Configure / Generate 단계** — 소스 디렉토리를 가리켜 `cmake`를 실행한다. CMake는 `CMakeLists.txt`를 파싱하고, 변수와 조건문을 평가하며, 툴체인(컴파일러, 링커, 사용 가능한 라이브러리)을 탐색한 뒤, 빌드 디렉토리에 네이티브 빌드 파일을 작성한다.
2. **Build 단계** — 네이티브 빌드 도구(`make`, `ninja`, `msbuild` 등)를 직접 호출하거나, 이식성 있는 래퍼인 `cmake --build <dir>`를 사용해 실제로 컴파일과 링크를 수행한다.

#### 일반적인 워크플로우

```bash
mkdir build && cd build
cmake ..              # configure: CMakeLists.txt를 읽어 빌드 파일 생성
cmake --build .       # build: 네이티브 빌드 도구 실행
cmake --install .     # optional: 산출물 설치
```

> [!tip] Out-of-source 빌드
> 소스와 분리된 별도의 `build/` 디렉토리에 생성 산출물을 두는 것이 관례다. 정리가 간단하다 — 디렉토리만 지우면 끝.

#### 핵심 CMakeLists.txt 개념

- `cmake_minimum_required(VERSION x.y)` — 최소 CMake 버전 선언
- `project(name LANGUAGES CXX)` — 프로젝트 정의
- `add_executable(target src1.cpp src2.cpp)` — 실행 파일 빌드
- `add_library(target STATIC|SHARED src.cpp)` — 라이브러리 빌드
- `target_link_libraries(target PRIVATE other)` — 의존성 링크
- `target_include_directories(target PUBLIC include/)` — 헤더 경로 설정
- `find_package(Foo REQUIRED)` — 외부 의존성 탐색

#### 왜 쓰는가

- 크로스 플랫폼: 하나의 설정 파일이 Linux, macOS, Windows에서 모두 동작
- 툴체인 독립적: GCC, Clang, MSVC를 빌드 로직 수정 없이 전환 가능
- `find_package`와 모던 "imported targets"로 의존성 탐색
- C++ 오픈소스 프로젝트의 사실상 표준 (LLVM, Qt, OpenCV 등)
- IDE 통합: 대부분의 C++ IDE가 CMake 프로젝트를 기본 지원

#### 모던 CMake (3.x+) 관용구

> [!warning] 디렉토리 범위 명령은 피할 것
> `include_directories`나 전역 `CMAKE_CXX_FLAGS` 같은 디렉토리 범위 명령보다는 `target_link_libraries`, `target_include_directories` 같은 **타겟 기반** 명령을 선호한다. 속성은 타겟 단위로 설정한다.

- `PUBLIC`/`PRIVATE`/`INTERFACE` 가시성으로 의존성 전파를 모델링
- Imported target(예: `find_package(Foo)`에서 나오는 `Foo::Foo`)은 자체 include 경로와 링크 플래그를 갖는다 — `target_link_libraries`만 호출하면 끝

## Sources

- [[raw/conversations/019e8d91-220d-763d-9a79-d85d7053d3bc|019e8d91-220d-763d-9a79-d85d7053d3bc]]
- [[raw/conversations/019e8da5-ab9c-713f-b09d-4c32f39c0ef5|019e8da5-ab9c-713f-b09d-4c32f39c0ef5]]
