# CMake Best Practices — Deep Dive Reference

## Target Properties

### Key Properties

| Property | Scope | Purpose |
|----------|-------|---------|
| `COMPILE_FEATURES` | PUBLIC/PRIVATE/INTERFACE | Required C++ standard features |
| `COMPILE_OPTIONS` | PUBLIC/PRIVATE/INTERFACE | Compiler flags |
| `COMPILE_DEFINITIONS` | PUBLIC/PRIVATE/INTERFACE | Preprocessor defines |
| `INCLUDE_DIRECTORIES` | PUBLIC/PRIVATE/INTERFACE | Header search paths |
| `LINK_LIBRARIES` | PUBLIC/PRIVATE/INTERFACE | Dependencies |
| `SOURCES` | PRIVATE | Source files |

### PUBLIC vs PRIVATE vs INTERFACE

```cmake
# PUBLIC: used by this target AND propagated to dependents
target_include_directories(mylib PUBLIC include/)
# Consumers of mylib automatically get include/ in their search path

# PRIVATE: used only by this target
target_include_directories(mylib PRIVATE src/)
# Internal headers not exposed to consumers

# INTERFACE: NOT used by this target, only propagated to dependents
# Used for header-only libraries
add_library(header_only INTERFACE)
target_include_directories(header_only INTERFACE include/)
target_compile_features(header_only INTERFACE cxx_std_20)
```

### Transitive Dependencies

```cmake
# If mylib links fmt as PUBLIC, consumers of mylib also link fmt
target_link_libraries(mylib PUBLIC fmt::fmt)

# If mylib links spdlog as PRIVATE, consumers don't see spdlog
target_link_libraries(mylib PRIVATE spdlog::spdlog)
```

## Generator Expressions

Generator expressions are evaluated at generate time (not configure time), enabling per-config and per-platform logic.

### Build vs Install Interface

```cmake
target_include_directories(mylib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)
# During build: uses source tree headers
# After install: uses installed headers
```

### Per-Configuration

```cmake
target_compile_definitions(mylib
    PRIVATE
        $<$<CONFIG:Debug>:DEBUG_MODE=1>
        $<$<CONFIG:Release>:NDEBUG>
)

target_compile_options(mylib
    PRIVATE
        $<$<CONFIG:Debug>:-O0 -g>
        $<$<CONFIG:Release>:-O3>
)
```

### Per-Compiler

```cmake
target_compile_options(mylib PRIVATE
    $<$<CXX_COMPILER_ID:GNU>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:Clang>:-Wall -Wextra -Wpedantic -Wno-c++98-compat>
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive->
)
```

### Per-Platform

```cmake
target_compile_definitions(mylib PRIVATE
    $<$<PLATFORM_ID:Windows>:WIN32_LEAN_AND_MEAN NOMINMAX>
    $<$<PLATFORM_ID:Linux>:_GNU_SOURCE>
)

target_link_libraries(mylib PRIVATE
    $<$<PLATFORM_ID:Linux>:pthread>
)
```

### Boolean Logic

```cmake
# AND, OR, NOT
$<AND:$<CXX_COMPILER_ID:GNU>,$<VERSION_GREATER:$<CXX_COMPILER_VERSION>,10>>
$<OR:$<CXX_COMPILER_ID:GNU>,$<CXX_COMPILER_ID:Clang>>
$<NOT:$<CONFIG:Debug>>

# Conditional
$<IF:$<CONFIG:Debug>,debug_value,release_value>
```

## Installing and Exporting

### Install Targets

```cmake
include(GNUInstallDirs)

install(TARGETS mylib
    EXPORT mylib-targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# Install headers
install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
```

### Export Targets

```cmake
# Generate and install the export file
install(EXPORT mylib-targets
    FILE mylibTargets.cmake
    NAMESPACE mylib::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/mylib
)
```

### Config.cmake Generation

```cmake
include(CMakePackageConfigHelpers)

# Generate version file
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/mylibConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

# Generate config file from template
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/mylibConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/mylibConfig.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/mylib
)

# Install config files
install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/mylibConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/mylibConfigVersion.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/mylib
)
```

Config template (`cmake/mylibConfig.cmake.in`):

```cmake
@PACKAGE_INIT@

include("${CMAKE_CURRENT_LIST_DIR}/mylibTargets.cmake")

check_required_components(mylib)
```

### Consumer Usage

```cmake
find_package(mylib REQUIRED)
target_link_libraries(app PRIVATE mylib::mylib)
```

## Custom Commands and Targets

### Code Generation

```cmake
# Generate a header from a template
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/version.h
    COMMAND ${CMAKE_COMMAND} -E echo
        "#define VERSION \"${PROJECT_VERSION}\"" > version.h
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Generating version.h"
)

# Depend on the generated file
target_sources(mylib PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/version.h)
target_include_directories(mylib PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
```

### Custom Build Steps

```cmake
# Run protobuf compiler
add_custom_command(
    OUTPUT ${PROTO_SRCS} ${PROTO_HDRS}
    COMMAND protoc --cpp_out=${CMAKE_CURRENT_BINARY_DIR} ${PROTO_FILES}
    DEPENDS ${PROTO_FILES}
    COMMENT "Running protoc"
)

# Custom target that always runs
add_custom_target(run_linter
    COMMAND clang-tidy ${ALL_SOURCES}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    COMMENT "Running clang-tidy"
)
```

## Header-Only Libraries

```cmake
add_library(mylib INTERFACE)

target_include_directories(mylib
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

target_compile_features(mylib INTERFACE cxx_std_20)

# Optional: add sources for IDE integration
target_sources(mylib INTERFACE
    FILE_SET HEADERS
    BASE_DIRS include
    FILES include/mylib/header.hpp
)
```

## Compiler Feature Detection

### Require C++ Standard

```cmake
# Require C++20 as a minimum
target_compile_features(mylib PUBLIC cxx_std_20)

# Don't set CMAKE_CXX_STANDARD globally — use target_compile_features instead
# This allows different targets to use different standards
```

### Feature Detection

```cmake
# Check for specific compiler features
include(CheckCXXSourceCompiles)

check_cxx_source_compiles("
    #include <concepts>
    template <std::integral T>
    T add(T a, T b) { return a + b; }
    int main() { return add(1, 2); }
" HAS_CONCEPTS)

if(HAS_CONCEPTS)
    target_compile_definitions(mylib PRIVATE HAS_CONCEPTS=1)
endif()
```

### Feature Test Macros

```cmake
# C++20 feature test macros in code
target_compile_definitions(mylib PRIVATE
    $<$<COMPILE_FEATURES:cxx_std_20>:HAS_CXX20=1>
)
```

```cpp
// In code
#if __has_include(<format>)
    #include <format>
    #define HAS_STD_FORMAT 1
#endif

#ifdef __cpp_concepts
    // Use concepts
#else
    // Fallback to SFINAE
#endif
```

## Multi-Config Generators

Visual Studio and Ninja Multi-Config generate build files for all configurations simultaneously:

```cmake
# Don't use CMAKE_BUILD_TYPE with multi-config generators
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# Per-config output directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR}/Debug)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR}/Release)

# Per-config compile definitions using generator expressions
target_compile_definitions(mylib PRIVATE
    $<$<CONFIG:Debug>:MY_DEBUG=1>
    $<$<CONFIG:Release>:MY_RELEASE=1>
)
```

Build specific configurations:

```bash
# Ninja Multi-Config
cmake -B build -G "Ninja Multi-Config"
cmake --build build --config Release
cmake --build build --config Debug

# Visual Studio
cmake -B build -G "Visual Studio 17 2022"
cmake --build build --config Release
```

## CMake Anti-Patterns

### Avoid These

```cmake
# BAD: global include directories
include_directories(include/)
# GOOD: target-specific
target_include_directories(mylib PUBLIC include/)

# BAD: global link directories
link_directories(/usr/local/lib)
# GOOD: use find_package or target_link_libraries with full paths

# BAD: global definitions
add_definitions(-DFOO=1)
# GOOD: target-specific
target_compile_definitions(mylib PRIVATE FOO=1)

# BAD: setting CMAKE_CXX_STANDARD globally
set(CMAKE_CXX_STANDARD 20)
# GOOD: per-target compile features
target_compile_features(mylib PUBLIC cxx_std_20)

# BAD: file(GLOB ...) for source files
file(GLOB SOURCES "src/*.cpp")  # Not re-run when files added/removed
# GOOD: explicit source list
set(SOURCES src/main.cpp src/parser.cpp src/renderer.cpp)

# BAD: CMAKE_CXX_FLAGS for all targets
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
# GOOD: per-target options
target_compile_options(mylib PRIVATE -Wall)
```
