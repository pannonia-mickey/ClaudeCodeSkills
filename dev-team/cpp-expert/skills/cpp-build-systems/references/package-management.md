# Package Management — Deep Dive Reference

## Conan 2.x

### conanfile.py Recipe

```python
from conan import ConanFile
from conan.tools.cmake import CMake, cmake_layout, CMakeDeps, CMakeToolchain

class MyAppConan(ConanFile):
    name = "myapp"
    version = "1.0.0"
    settings = "os", "compiler", "build_type", "arch"
    generators = "CMakeDeps", "CMakeToolchain"

    def requirements(self):
        self.requires("fmt/10.2.1")
        self.requires("spdlog/1.13.0")
        self.requires("boost/1.84.0")
        self.requires("gtest/1.14.0", visible=False)  # Test dependency

    def build_requirements(self):
        self.tool_requires("cmake/3.28.1")

    def layout(self):
        cmake_layout(self)

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()

    def package(self):
        cmake = CMake(self)
        cmake.install()
```

### Generators

| Generator | Purpose |
|-----------|---------|
| `CMakeDeps` | Generates `Find<pkg>.cmake` files for each dependency |
| `CMakeToolchain` | Generates `conan_toolchain.cmake` with compiler/arch settings |
| `CMakePresets` | Generates CMakePresets.json entries for Conan builds |

### Profiles

Create profiles for different build configurations:

```ini
# ~/.conan2/profiles/default
[settings]
os=Linux
compiler=gcc
compiler.version=13
compiler.libcxx=libstdc++11
build_type=Release
arch=x86_64

[conf]
tools.cmake.cmaketoolchain:generator=Ninja
```

```ini
# ~/.conan2/profiles/sanitize
include(default)

[settings]
build_type=Debug

[conf]
tools.build:cflags=["-fsanitize=address,undefined", "-fno-omit-frame-pointer"]
tools.build:cxxflags=["-fsanitize=address,undefined", "-fno-omit-frame-pointer"]
tools.build:sharedlinkflags=["-fsanitize=address,undefined"]
tools.build:exelinkflags=["-fsanitize=address,undefined"]
```

### Common Conan Commands

```bash
# Install dependencies
conan install . --build=missing

# Install with specific profile
conan install . --profile=sanitize --build=missing

# Cross-compilation with host and build profiles
conan install . --profile:host=arm64 --profile:build=default --build=missing

# Build and test
conan build .

# Create a package
conan create . --build=missing

# Search for packages
conan search "boost/*" -r conancenter

# Lock dependencies
conan lock create .
```

### Options

```python
class MyLibConan(ConanFile):
    options = {
        "shared": [True, False],
        "with_ssl": [True, False],
    }
    default_options = {
        "shared": False,
        "with_ssl": True,
    }

    def requirements(self):
        if self.options.with_ssl:
            self.requires("openssl/3.2.1")
```

```bash
# Override options
conan install . -o mylib/*:shared=True -o mylib/*:with_ssl=False
```

## vcpkg

### Manifest Mode (vcpkg.json)

```json
{
    "name": "myapp",
    "version": "1.0.0",
    "dependencies": [
        "fmt",
        "spdlog",
        {
            "name": "boost-asio",
            "version>=": "1.84.0"
        },
        {
            "name": "openssl",
            "platform": "linux"
        }
    ],
    "overrides": [
        {
            "name": "fmt",
            "version": "10.2.1"
        }
    ],
    "builtin-baseline": "a1a1cbc975abf909a6c8985a6a2b8a0c2dff5fed"
}
```

### Toolchain Integration

```cmake
# Configure with vcpkg toolchain
cmake -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake

# Or in CMakePresets.json
{
    "configurePresets": [{
        "name": "vcpkg",
        "cacheVariables": {
            "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
        }
    }]
}
```

### Custom Triplets

```cmake
# custom-triplets/x64-linux-sanitize.cmake
set(VCPKG_TARGET_ARCHITECTURE x64)
set(VCPKG_CRT_LINKAGE dynamic)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Linux)
set(VCPKG_CXX_FLAGS "-fsanitize=address,undefined")
set(VCPKG_C_FLAGS "-fsanitize=address,undefined")
set(VCPKG_LINKER_FLAGS "-fsanitize=address,undefined")
```

```bash
vcpkg install --triplet x64-linux-sanitize --overlay-triplets=custom-triplets
```

### Binary Caching

```bash
# Enable binary caching (default: ~/.cache/vcpkg/archives)
export VCPKG_BINARY_SOURCES="clear;default,readwrite"

# Use NuGet-based binary cache
export VCPKG_BINARY_SOURCES="clear;nuget,https://my-feed.example.com/nuget/v3/index.json,readwrite"

# Use GitHub Actions cache
export VCPKG_BINARY_SOURCES="clear;x-gha,readwrite"
```

### Registries

```json
{
    "registries": [
        {
            "kind": "git",
            "repository": "https://github.com/myorg/vcpkg-registry.git",
            "baseline": "abc123",
            "packages": ["mylib", "myutils"]
        }
    ],
    "default-registry": {
        "kind": "builtin",
        "baseline": "a1a1cbc975abf909a6c8985a6a2b8a0c2dff5fed"
    }
}
```

## FetchContent Patterns

### Basic Usage

```cmake
include(FetchContent)

FetchContent_Declare(json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG        v3.11.3
)
FetchContent_MakeAvailable(json)

target_link_libraries(myapp PRIVATE nlohmann_json::nlohmann_json)
```

### Pinning Versions

```cmake
# Pin to specific commit for reproducibility
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        e69e5f977d458f2650bb346dadf2ad30c5320281  # v10.2.1
)

# Or use a release archive (faster download, no git history)
FetchContent_Declare(fmt
    URL https://github.com/fmtlib/fmt/releases/download/10.2.1/fmt-10.2.1.zip
    URL_HASH SHA256=1250e4cc58bf06ee631567523f48848dc4596133e163f02615c97f78bab6c811
)
```

### OVERRIDE_FIND_PACKAGE

Make FetchContent work transparently with find_package:

```cmake
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        v10.2.1
    OVERRIDE_FIND_PACKAGE  # find_package(fmt) uses FetchContent instead
)

# Now this works whether fmt is installed system-wide or fetched
find_package(fmt REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

## CPM.cmake

Simplified FetchContent wrapper:

```cmake
# Download CPM.cmake
include(cmake/CPM.cmake)

# Simple dependency addition
CPMAddPackage("gh:fmtlib/fmt#10.2.1")
CPMAddPackage("gh:nlohmann/json@3.11.3")
CPMAddPackage("gh:gabime/spdlog@1.13.0")

# With options
CPMAddPackage(
    NAME boost_asio
    GITHUB_REPOSITORY boostorg/asio
    GIT_TAG boost-1.84.0
    OPTIONS "BOOST_ASIO_NO_DEPRECATED ON"
)
```

## Comparison Matrix

| Feature | Conan 2.x | vcpkg | FetchContent | CPM.cmake |
|---------|----------|-------|-------------|-----------|
| Package count | 1,500+ (ConanCenter) | 2,000+ | Any CMake project | Any CMake project |
| Binary caching | Built-in | Built-in | No | Optional |
| Cross-compilation | Excellent (profiles) | Good (triplets) | Manual | Manual |
| Version management | Full (lockfiles) | Baselines | Manual (git tags) | Manual |
| Build system | Any (CMake, Meson, etc.) | CMake only | CMake only | CMake only |
| Build from source | Configurable | Default | Always | Always |
| CI integration | Strong | Strong | Built-in | Built-in |
| Learning curve | Medium | Low | Low | Low |
| Reproducibility | Excellent (lockfiles) | Good (baselines) | Good (hashes) | Good |

### When to Use Each

- **Conan**: Large projects, many dependencies, cross-compilation, need binary caching, non-CMake builds
- **vcpkg**: Microsoft ecosystem, Visual Studio integration, simpler setup
- **FetchContent**: 1-3 simple CMake dependencies, no package manager overhead
- **CPM.cmake**: Like FetchContent but with caching and simpler syntax

## CI/CD Workflows for C++

### GitHub Actions

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        build_type: [Debug, Release]
        compiler:
          - { cc: gcc-13, cxx: g++-13 }
          - { cc: clang-17, cxx: clang++-17 }
        exclude:
          - os: windows-latest
            compiler: { cc: gcc-13, cxx: g++-13 }
          - os: macos-latest
            compiler: { cc: gcc-13, cxx: g++-13 }

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Install Conan
        run: pip install conan

      - name: Cache Conan packages
        uses: actions/cache@v4
        with:
          path: ~/.conan2
          key: conan-${{ matrix.os }}-${{ matrix.compiler.cxx }}-${{ hashFiles('conanfile.py') }}

      - name: Install dependencies
        run: conan install . --build=missing -s build_type=${{ matrix.build_type }}
        env:
          CC: ${{ matrix.compiler.cc }}
          CXX: ${{ matrix.compiler.cxx }}

      - name: Configure
        run: cmake --preset conan-${{ matrix.build_type == 'Debug' && 'debug' || 'release' }}

      - name: Build
        run: cmake --build --preset conan-${{ matrix.build_type == 'Debug' && 'debug' || 'release' }}

      - name: Test
        run: ctest --preset conan-${{ matrix.build_type == 'Debug' && 'debug' || 'release' }} --output-on-failure

  sanitizers:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure with sanitizers
        run: |
          cmake -B build \
            -DCMAKE_BUILD_TYPE=Debug \
            -DENABLE_SANITIZERS=ON \
            -DCMAKE_CXX_COMPILER=clang++-17

      - name: Build
        run: cmake --build build

      - name: Test with sanitizers
        run: ctest --test-dir build --output-on-failure
        env:
          ASAN_OPTIONS: detect_leaks=1:halt_on_error=1

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure with compile_commands.json
        run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

      - name: Run clang-tidy
        run: |
          find src include -name '*.cpp' -o -name '*.hpp' | \
            xargs clang-tidy-17 -p build --warnings-as-errors='*'
```

### Caching Dependencies

```yaml
# Conan cache
- uses: actions/cache@v4
  with:
    path: ~/.conan2
    key: conan-${{ runner.os }}-${{ hashFiles('conanfile.py') }}

# vcpkg cache
- uses: actions/cache@v4
  with:
    path: |
      ${{ env.VCPKG_ROOT }}/installed
      ${{ env.VCPKG_ROOT }}/packages
    key: vcpkg-${{ runner.os }}-${{ hashFiles('vcpkg.json') }}

# CMake build cache (ccache)
- uses: hendrikmuhs/ccache-action@v1
  with:
    key: ${{ runner.os }}-${{ matrix.compiler.cxx }}
```
