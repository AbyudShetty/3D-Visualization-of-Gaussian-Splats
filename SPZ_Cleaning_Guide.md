# **NIANTIC SPZ — Full Installation + Build + Cleanup Pipeline (Windows 10/11)**
This guide will help you install and build the Niantic SPZ library with Python bindings on Windows, using the zlib-ng compatibility version. It also includes a final cleanup script to process SPZ files.

---

# 1. Clone SPZ

```bash
git clone https://github.com/nianticlabs/spz.git
cd spz
```

---

# 2. Download ZLIB-NG (MUST BE THE *compat* VERSION)

From releases:

```
https://github.com/zlib-ng/zlib-ng/releases/
```

Download:

```
zlib-ng-win-x86-64-compat.zip
```

Extract to:

```
C:\Program Files\zlib
```

Folder structure must be:

```
C:\Program Files\zlib\
    include\
    lib\
    bin\
```

---

# 3. Set Environment Variables

Search **“Edit the system environment variables”** → Open

### Add two variables under **System variables**:

#### **Variable 1**

```
Name:  ZLIB_LIBRARY
Value: C:\Program Files\zlib\lib\zlibstatic.lib
```

#### **Variable 2**

```
Name:  ZLIB_INCLUDE_DIR
Value: C:\Program Files\zlib\include
```

Click **OK → OK → OK**

---

# 4. Open new CMD and verify

```cmd
echo %ZLIB_LIBRARY%
echo %ZLIB_INCLUDE_DIR%
```

Expected:

```
C:\Program Files\zlib\lib\zlibstatic.lib
C:\Program Files\zlib\include
```

### Just in case, add in User variables too.

---

# 🔧 5. Required: CMake & Ninja

If you don’t already have them:

```bash
pip install cmake ninja
```

---

# 6. Replace SPZ CMakeLists.txt With This Stable Version

This fixes:

* Windows name conflicts
* Python extension naming
* zlib detection
* nanobind ABI issues
* archive vs shared output clash

👉 **Place this file inside `spz/` replacing the existing CMakeLists.txt**

---

### **CMakeLists.txt**

```cmake
cmake_minimum_required(VERSION 3.10)

project(spz
  DESCRIPTION "A 3D Gaussians format"
  LANGUAGES C CXX
  VERSION 1.1.0)

include(GNUInstallDirs)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# -----------------------------
# ZLIB (must be zlib-ng compat)
# -----------------------------
find_package(ZLIB REQUIRED)

# -----------------------------
# Core SPZ sources
# -----------------------------
set(spz_sources
  "${CMAKE_CURRENT_SOURCE_DIR}/src/cc/load-spz.cc"
  "${CMAKE_CURRENT_SOURCE_DIR}/src/cc/splat-c-types.cc"
  "${CMAKE_CURRENT_SOURCE_DIR}/src/cc/splat-types.cc"
)

set(spz_headers
  "${CMAKE_CURRENT_SOURCE_DIR}/src/cc/load-spz.h"
  "${CMAKE_CURRENT_SOURCE_DIR}/src/cc/splat-c-types.h"
  "${CMAKE_CURRENT_SOURCE_DIR}/src/cc/splat-types.h"
)

# -----------------------------
# C++ library target
# -----------------------------
add_library(spz ${spz_sources})
add_library(spz::spz ALIAS spz)

target_link_libraries(spz PRIVATE ZLIB::ZLIB)

if (ANDROID)
  target_link_libraries(spz PRIVATE log)
endif()

target_include_directories(spz
  PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/cc>
  INTERFACE $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

set_target_properties(spz PROPERTIES
  PUBLIC_HEADER "${spz_headers}"
  CXX_STANDARD 17
  CXX_STANDARD_REQUIRED ON
)

# -----------------------------
# Installation
# -----------------------------
include(CMakePackageConfigHelpers)

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/spzConfig.cmake.in"
  "${CMAKE_BINARY_DIR}/cmake/spzConfig.cmake"
  INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/spz"
)

write_basic_package_version_file(
  "${CMAKE_BINARY_DIR}/cmake/spzConfigVersion.cmake"
  VERSION "${spz_VERSION}"
  COMPATIBILITY SameMajorVersion
)

install(FILES
    "${CMAKE_BINARY_DIR}/cmake/spzConfig.cmake"
    "${CMAKE_BINARY_DIR}/cmake/spzConfigVersion.cmake"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/spz"
)

install(TARGETS spz
  EXPORT spzTargets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(EXPORT spzTargets
  NAMESPACE spz::
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/spz"
)

# -----------------------------
# Command line tools
# -----------------------------
option(BUILD_TOOLS "Build command line tools" ON)

if (BUILD_TOOLS)
  add_executable(ply_to_spz cli_tools/src/ply_to_spz.cpp)
  target_link_libraries(ply_to_spz PRIVATE spz)

  add_executable(spz_to_ply cli_tools/src/spz_to_ply.cpp)
  target_link_libraries(spz_to_ply PRIVATE spz)

  add_executable(spz_info cli_tools/src/spz_info.cpp)
  target_link_libraries(spz_info PRIVATE spz)

  install(TARGETS ply_to_spz spz_to_ply spz_info
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  )
endif()

# -----------------------------
# Python bindings
# -----------------------------
option(BUILD_PYTHON_BINDINGS "Build Python bindings using nanobind" ON)

if (BUILD_PYTHON_BINDINGS)
  find_package(Python 3.8 REQUIRED COMPONENTS Interpreter Development.Module OPTIONAL_COMPONENTS Development.SABIModule)

  execute_process(
    COMMAND "${Python_EXECUTABLE}" -m nanobind --cmake_dir
    OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE nanobind_DIR
  )

  find_package(nanobind CONFIG REQUIRED)

  nanobind_add_module(
    spz_python
    STABLE_ABI
    NB_STATIC
    src/python/spz/spz.cc
  )

  set_target_properties(spz_python PROPERTIES
    OUTPUT_NAME spz
    ARCHIVE_OUTPUT_NAME spz_python
  )

  target_include_directories(spz_python PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
  target_link_libraries(spz_python PRIVATE spz::spz)

  install(TARGETS spz_python LIBRARY DESTINATION spz)
endif()
```

---

# 7. Build Python Package

Inside the `spz/` folder:

```bash
pip install .
```

If you get build errors:

```bash
pip install cmake ninja
pip install .
```

---

# 8. Fix Runtime (Windows DLL)

Find:

```
C:\Program Files\zlib\bin\zlib1.dll
```

### Or maybe zlib.dll

Copy → paste into:

```
C:\Users\<YourUsername>\AppData\Local\Programs\Python\Python310\Lib\site-packages\spz\
```

---

# 9. Verify install

Inside Python:

```python
import spz
print(spz)
```

If no error → SUCCESS.

---

# 10. Your Final Cleanup Script in Python