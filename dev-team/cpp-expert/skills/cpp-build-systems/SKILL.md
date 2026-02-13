---
name: C++ Build Systems
description: This skill should be used when the user asks about "CMake", "CMakeLists", "Conan", "vcpkg", "C++ package manager", "cross-compilation", "sanitizers in CMake", "clang-tidy", "CMake presets", "FetchContent", "C++ CI/CD", "CMake targets", "C++ static analysis", or "C++ project setup". It covers modern CMake best practices, package management with Conan and vcpkg, sanitizer and static analysis integration, cross-compilation, and CI/CD configuration for C++ projects.
version: 0.1.0
---

# C++ Build Systems

Modern CMake is the dominant build system for C++. This skill covers target-based CMake, package management, sanitizers, static analysis, and project structure.

## Modern CMake Principles

Use target-based commands exclusively. Never use global commands like `include_directories()`, `link_directories()`, or `link_libraries()`.

```cmake
cmake_minimum_required(VERSION 3.25)
project(myapp VERSION 1.0.0 LANGUAGES CXX)

# Library target
add_library(mylib
    src/parser.cpp
    src/renderer.cpp
)
target_include_directories(mylib
    PUBLIC  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:include>
    PRIVATE src
)
target_compile_features(mylib PUBLIC cxx_std_20)
target_compile_options(mylib PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)

# Executable target
add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE mylib)
```

Key principles:
- **PUBLIC** — propagates to dependents (public headers, required compile features)
- **PRIVATE** — used only by the target itself (implementation files, internal warnings)
- **INTERFACE** — propagates to dependents but not used by the target (header-only libraries)

## CMake Presets

Use `CMakePresets.json` to define standardized build configurations. Store personal overrides in `CMakeUserPresets.json` (gitignored).

```json
{
    "version": 6,
    "configurePresets": [
        {
            "name": "dev",
            "binaryDir": "build/dev",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
                "BUILD_TESTING": "ON"
            }
        },
        {
            "name": "release",
            "binaryDir": "build/release",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_INTERPROCEDURAL_OPTIMIZATION": "ON"
            }
        },
        {
            "name": "sanitize",
            "inherits": "dev",
            "binaryDir": "build/sanitize",
            "cacheVariables": {
                "ENABLE_SANITIZERS": "ON"
            }
        }
    ],
    "buildPresets": [
        { "name": "dev", "configurePreset": "dev" },
        { "name": "release", "configurePreset": "release" }
    ],
    "testPresets": [
        { "name": "dev", "configurePreset": "dev", "output": { "outputOnFailure": true } }
    ]
}
```

Usage: `cmake --preset dev && cmake --build --preset dev && ctest --preset dev`

## Dependency Management

### FetchContent — Simple Dependencies

Use for small dependencies or when a package manager is not available:

```cmake
include(FetchContent)

FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1
)
FetchContent_MakeAvailable(fmt)

target_link_libraries(myapp PRIVATE fmt::fmt)
```

### Conan 2.x — Full Package Manager

Create a `conanfile.py` or `conanfile.txt`:

```python
# conanfile.py
from conan import ConanFile
from conan.tools.cmake import cmake_layout

class MyAppConan(ConanFile):
    settings = "os", "compiler", "build_type", "arch"
    generators = "CMakeDeps", "CMakeToolchain"

    def requirements(self):
        self.requires("fmt/10.2.1")
        self.requires("spdlog/1.13.0")
        self.requires("boost/1.84.0")

    def layout(self):
        cmake_layout(self)
```

Install and build: `conan install . --build=missing && cmake --preset conan-release && cmake --build --preset conan-release`

### vcpkg — Manifest Mode

Create `vcpkg.json` in the project root:

```json
{
    "name": "myapp",
    "version": "1.0.0",
    "dependencies": [
        "fmt",
        "spdlog",
        { "name": "boost-asio", "version>=": "1.84.0" }
    ]
}
```

Configure with toolchain: `cmake -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake`

**When to use each**: FetchContent for 1-2 simple dependencies; Conan for complex projects with many dependencies and cross-compilation; vcpkg for projects already in the Microsoft ecosystem or wanting simpler setup.

## Sanitizer Integration

Enable sanitizers via a CMake option for development and CI builds:

```cmake
option(ENABLE_SANITIZERS "Enable ASan + UBSan" OFF)

if(ENABLE_SANITIZERS)
    set(SANITIZER_FLAGS -fsanitize=address,undefined -fno-omit-frame-pointer -fno-sanitize-recover=all)
    add_compile_options(${SANITIZER_FLAGS})
    add_link_options(${SANITIZER_FLAGS})
endif()
```

| Combination | Purpose |
|------------|---------|
| ASan + UBSan | Default for development — catches most memory and UB issues |
| TSan | Dedicated data race detection (incompatible with ASan) |
| MSan | Uninitialized memory detection (incompatible with ASan, requires instrumented libc++) |

Run sanitizer builds in CI as a separate job. Use suppressions files for known third-party issues: `ASAN_OPTIONS=suppressions=asan_suppressions.txt`

## Static Analysis

### clang-tidy

Integrate clang-tidy directly into the build:

```cmake
# Enable clang-tidy for all targets
set(CMAKE_CXX_CLANG_TIDY clang-tidy --config-file=${CMAKE_SOURCE_DIR}/.clang-tidy)
```

Create `.clang-tidy` in the project root:

```yaml
Checks: >
  -*,
  bugprone-*,
  clang-analyzer-*,
  cppcoreguidelines-*,
  modernize-*,
  performance-*,
  readability-*,
  -modernize-use-trailing-return-type,
  -readability-magic-numbers
WarningsAsErrors: 'bugprone-*,clang-analyzer-*'
HeaderFilterRegex: 'include/.*'
```

### Compiler Warnings

Enable comprehensive warnings as the first line of defense:

```cmake
target_compile_options(mylib PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:
        -Wall -Wextra -Wpedantic -Wshadow -Wconversion
        -Wsign-conversion -Wnon-virtual-dtor -Wold-style-cast
        -Woverloaded-virtual -Wnull-dereference>
    $<$<CXX_COMPILER_ID:MSVC>:
        /W4 /w14640 /w14242 /w14254 /w14263 /w14265 /w14287>
)
```

## Testing Integration

Use CTest with Google Test via FetchContent:

```cmake
enable_testing()

FetchContent_Declare(googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0
)
FetchContent_MakeAvailable(googletest)

add_executable(mylib_tests tests/test_parser.cpp tests/test_renderer.cpp)
target_link_libraries(mylib_tests PRIVATE mylib GTest::gtest_main)

include(GoogleTest)
gtest_discover_tests(mylib_tests)
```

For code coverage:

```cmake
if(ENABLE_COVERAGE)
    target_compile_options(mylib PRIVATE --coverage)
    target_link_options(mylib PRIVATE --coverage)
endif()
# Generate report: lcov --capture --directory . --output-file coverage.info
```

## Cross-Compilation

Create a toolchain file for the target platform:

```cmake
# aarch64-linux-toolchain.cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER   aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

Configure: `cmake -B build-arm --toolchain aarch64-linux-toolchain.cmake`

For Conan cross-compilation, create host and build profiles: `conan install . --profile:host=arm64 --profile:build=default`

## Project Structure

Recommended directory layout for a library + application project:

```
myproject/
├── CMakeLists.txt              # Top-level: project(), add_subdirectory()
├── CMakePresets.json            # Build presets
├── .clang-tidy                  # Static analysis config
├── .clang-format                # Code formatting config
├── conanfile.py                 # Dependencies (or vcpkg.json)
├── include/
│   └── mylib/                   # Public headers
│       ├── parser.h
│       └── renderer.h
├── src/
│   ├── CMakeLists.txt           # add_library(mylib ...)
│   ├── parser.cpp
│   ├── renderer.cpp
│   └── internal/                # Private headers
│       └── helpers.h
├── app/
│   ├── CMakeLists.txt           # add_executable(myapp ...)
│   └── main.cpp
├── tests/
│   ├── CMakeLists.txt           # Test targets
│   ├── test_parser.cpp
│   └── test_renderer.cpp
└── cmake/
    └── toolchains/              # Cross-compilation toolchain files
```

Place public headers under `include/mylib/` to enforce proper include paths (`#include <mylib/parser.h>`).

## Additional Resources

### Reference Files

For in-depth CMake and package management guidance, consult:

- **`references/cmake-best-practices.md`** — Target properties, generator expressions, installing/exporting, Config.cmake generation, custom commands, header-only libraries, compiler feature detection, multi-config generators
- **`references/package-management.md`** — Conan 2.x deep dive (recipes, generators, profiles), vcpkg deep dive (manifest, triplets, registries), FetchContent patterns, CPM.cmake, comparison matrix, CI/CD workflows for C++
