---
id: 019e8da6-8116-731e-959a-6ca84df7c0df
name: C++
aliases:
  - C++
  - c-plus-plus
  - cpp
  - cxx
updated_at: '2026-06-07T08:10:33.821Z'
summary: >-
  A statically typed, compiled, general-purpose programming language with
  support for procedural, object-oriented, and generic programming.
sources:
  - 019e8d91-220d-763d-9a79-d85d7053d3bc
  - 019e8da5-ab9c-713f-b09d-4c32f39c0ef5
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# C++

## Overview
C++ is a general-purpose programming language created by Bjarne Stroustrup as an extension of the C language. It supports multiple paradigms — procedural, object-oriented, generic, and functional — and is widely used in systems programming, game engines, high-performance applications, and embedded software.

> [!note] Compiled, multi-paradigm, close to the hardware — chosen when control over memory and performance matters more than developer ergonomics.

## Notes
- Compiled to native code via toolchains like GCC, Clang, and MSVC.
- Manual memory management with RAII (Resource Acquisition Is Initialization) and smart pointers (`std::unique_ptr`, `std::shared_ptr`).
- Standard Library (STL) provides containers (`vector`, `map`), algorithms, and iterators.
- Modern C++ (C++11/14/17/20/23) introduced lambdas, move semantics, concepts, ranges, coroutines, and modules.
- Typically built with [[CMake]], Make, Bazel, or Meson.
- Header files (`.h`/`.hpp`) declare interfaces; source files (`.cpp`) contain implementations.

## Build with CMake
[[CMake]] is the de facto build-system generator for C++ projects. Rather than compiling code itself, it reads a `CMakeLists.txt` describing targets and dependencies, then generates native build files (Makefiles, Ninja, Visual Studio, Xcode) for the host toolchain.

> [!example] Minimal `CMakeLists.txt`
> ```cmake
> cmake_minimum_required(VERSION 3.20)
> project(hello CXX)
> add_executable(hello main.cpp)
> target_compile_features(hello PRIVATE cxx_std_20)
> ```

### How it works
1. **Configure** — `cmake -S . -B build` parses `CMakeLists.txt`, probes the compiler/OS, and writes a build tree under `build/`.
2. **Generate** — emits native build files for the chosen generator (`-G Ninja`, `-G "Unix Makefiles"`, …).
3. **Build** — `cmake --build build` invokes the generator's tool to compile sources and link targets.
4. **Install/Test** — `cmake --install build` and `ctest` handle packaging and tests.

Targets (`add_executable`, `add_library`) and properties (`target_include_directories`, `target_link_libraries`, `target_compile_features`) compose graph-style; consumers inherit `PUBLIC`/`INTERFACE` settings transitively.

---

## 한국어

### 개요
C++는 Bjarne Stroustrup이 C 언어를 확장하여 만든 범용 프로그래밍 언어다. 절차적, 객체지향, 제네릭, 함수형 등 다중 패러다임을 지원하며 시스템 프로그래밍, 게임 엔진, 고성능 애플리케이션, 임베디드 소프트웨어에서 널리 쓰인다.

> [!note] 컴파일 기반, 다중 패러다임, 하드웨어에 가까운 언어 — 개발 편의성보다 메모리와 성능 제어가 중요한 곳에서 선택된다.

### 노트
- GCC, Clang, MSVC 같은 툴체인으로 네이티브 코드로 컴파일된다.
- RAII(Resource Acquisition Is Initialization)와 스마트 포인터(`std::unique_ptr`, `std::shared_ptr`)로 수동 메모리 관리를 한다.
- 표준 라이브러리(STL)는 컨테이너(`vector`, `map`), 알고리즘, 이터레이터를 제공한다.
- 모던 C++ (C++11/14/17/20/23)는 람다, 이동 시맨틱, concepts, ranges, 코루틴, 모듈을 도입했다.
- 보통 [[CMake]], Make, Bazel, Meson으로 빌드한다.
- 헤더 파일(`.h`/`.hpp`)은 인터페이스를 선언하고, 소스 파일(`.cpp`)은 구현을 담는다.

### CMake로 빌드하기
[[CMake]]는 C++ 프로젝트의 사실상 표준 빌드 시스템 생성기다. 직접 컴파일하는 게 아니라 `CMakeLists.txt`에서 타겟과 의존성을 읽어 호스트 툴체인용 네이티브 빌드 파일(Makefile, Ninja, Visual Studio, Xcode)을 생성한다.

> [!example] 최소 `CMakeLists.txt`
> ```cmake
> cmake_minimum_required(VERSION 3.20)
> project(hello CXX)
> add_executable(hello main.cpp)
> target_compile_features(hello PRIVATE cxx_std_20)
> ```

#### 동작 방식
1. **Configure** — `cmake -S . -B build`가 `CMakeLists.txt`를 파싱하고 컴파일러/OS를 탐지한 뒤 `build/` 아래에 빌드 트리를 만든다.
2. **Generate** — 선택한 generator(`-G Ninja`, `-G "Unix Makefiles"` 등)에 맞는 네이티브 빌드 파일을 만들어낸다.
3. **Build** — `cmake --build build`가 generator 도구를 호출해 소스를 컴파일하고 타겟을 링크한다.
4. **Install/Test** — `cmake --install build`와 `ctest`가 패키징과 테스트를 담당한다.

타겟(`add_executable`, `add_library`)과 속성(`target_include_directories`, `target_link_libraries`, `target_compile_features`)이 그래프처럼 조합되며, 소비자는 `PUBLIC`/`INTERFACE` 설정을 전이적으로 상속한다.

## Sources

- [[raw/conversations/019e8d91-220d-763d-9a79-d85d7053d3bc|019e8d91-220d-763d-9a79-d85d7053d3bc]]
- [[raw/conversations/019e8da5-ab9c-713f-b09d-4c32f39c0ef5|019e8da5-ab9c-713f-b09d-4c32f39c0ef5]]
