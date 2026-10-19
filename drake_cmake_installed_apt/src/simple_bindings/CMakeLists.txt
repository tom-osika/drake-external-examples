# SPDX-License-Identifier: MIT-0

# nanobind resolves Python through the unversioned `Python::Module` target, and
# needs it in this scope -- the same one where nanobind_add_module builds below.
# (The top-level project found the versioned `Python3`, which nanobind does not
# reuse.) Finding it here, before nanobind is added, is what gets the Python
# headers onto nanobind's own build.
find_package(Python ${drake_PYTHON_VERSION} EXACT REQUIRED
  COMPONENTS Interpreter Development.Module)

# Drake uses upstream nanobind and does not ship it, so downstream projects must
# provide their own copy, pinned to the SAME version Drake builds against (see
# the nanobind `bazel_dep` in Drake's MODULE.bazel) so the bindings are
# ABI-compatible with pydrake. Here we fetch and build it as part of the CMake
# build. A project with its own dependency infrastructure (pip, conan, vcpkg, a
# system package, ...) can instead delete this block and use
# `find_package(nanobind CONFIG REQUIRED)`. nanobind bundles its tsl-robin-map
# dependency as a git submodule (ext/robin_map), which FetchContent initializes
# automatically.
# nanobind disables STABLE_ABI unless CMake's Development.SABIModule component is
# found, which needs the libpython3.so that Debian/Ubuntu don't ship. On
# Linux/macOS an extension module links no libpython (symbols resolve at import),
# so that gate is spurious; we patch it to fire on Windows only. Drop the patch
# once it lands upstream.
find_package(Git REQUIRED)
include(FetchContent)
FetchContent_Declare(
  nanobind
  GIT_REPOSITORY https://github.com/wjakob/nanobind.git
  GIT_TAG v3.1.0
  GIT_SHALLOW ON
  # Reset-then-apply so re-configures don't fail on an already-patched tree.
  PATCH_COMMAND "${GIT_EXECUTABLE}" checkout -- cmake/nanobind-config.cmake
    COMMAND "${GIT_EXECUTABLE}" apply
      "${CMAKE_CURRENT_SOURCE_DIR}/patches/nanobind-stable-abi-linux.patch"
)
FetchContent_MakeAvailable(nanobind)

drake_example_nanobind_add_module(simple_bindings simple_bindings.cc)

drake_example_add_py_test(NAME simple_bindings_test COMMAND
    "${Python3_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/simple_bindings_test.py")

set_property(TEST simple_bindings_test APPEND PROPERTY ENVIRONMENT
    "PYTHONPATH=$<TARGET_FILE_DIR:simple_bindings>:${drake_PYTHON_DIR}")
