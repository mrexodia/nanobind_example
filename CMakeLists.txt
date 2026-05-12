cmake_minimum_required(VERSION 3.15...3.26)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
  message(FATAL_ERROR "In-tree builds are not supported. Run CMake from a separate directory: cmake -B build")
endif()

# Keep compile_commands.json available for clangd and other C++ tooling.
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

project(nanobind_example LANGUAGES CXX)

# Try to import all Python components potentially needed by nanobind
find_package(Python 3.12
  REQUIRED COMPONENTS Interpreter Development.Module
  OPTIONAL_COMPONENTS Development.SABIModule
)

if (NOT SKBUILD)
  message(WARNING "\
  This CMake file is meant to be executed using 'scikit-build'. Running
  it directly will almost certainly not produce the desired result. If
  you are a user trying to install this package, please use the command
  below, which will install all necessary build dependencies, compile
  the package in an isolated environment, and then install it.
  =====================================================================
   $ pip install .
  =====================================================================
  If you are a software developer, and this is your own package, then
  it is usually much more efficient to install the build dependencies
  in your environment once and use the following command that avoids
  a costly creation of a new virtual environment at every compilation:
  =====================================================================
   $ uv sync
  =====================================================================
  When using uv run, the package is already rebuilt automatically if
  it detects sources listed in cache-keys are updated.")

  # NOTE: We allow fetching nanobind directly here for development this allows
  # `cmake -B build -G Ninja -DCMAKE_EXPORT_COMPILE_COMMANDS=ON` to work as
  # expected and get proper language server support in editors like VSCode.

  # Extract the minimum nanobind version from pyproject.toml
  file(READ "${CMAKE_CURRENT_SOURCE_DIR}/pyproject.toml" PYPROJECT_TOML_CONTENT)
  if(PYPROJECT_TOML_CONTENT MATCHES "\"nanobind *[>=]= *([^\"]+)\"")
    set(NANOBIND_TAG "v${CMAKE_MATCH_1}")
  else()
    message(FATAL_ERROR "Could not find nanobind version in pyproject.toml")
  endif()
  
  # Fix warnings about DOWNLOAD_EXTRACT_TIMESTAMP
  if(POLICY CMP0135)
    cmake_policy(SET CMP0135 NEW)
  endif()
  include(FetchContent)
  message(STATUS "Fetching nanobind (${NANOBIND_TAG})...")
  FetchContent_Declare(nanobind
    GIT_REPOSITORY
      "https://github.com/wjakob/nanobind"
    GIT_TAG
      ${NANOBIND_TAG}
  )
  FetchContent_MakeAvailable(nanobind)
else()
  # Use the nanobind provided by scikit-build-core
  find_package(nanobind REQUIRED)
endif()

set(PACKAGE_NAME nanobind_example)
set(EXTENSION_SOURCES
  src/nanobind_example.cpp
)

# We are now ready to compile the actual extension module
nanobind_add_module(
  # Name of the extension
  ${PACKAGE_NAME}

  # Target the stable ABI for Python 3.12+, which reduces
  # the number of binary wheels that must be built. This
  # does nothing on older Python versions
  STABLE_ABI

  # Build libnanobind statically and merge it into the
  # extension (which itself remains a shared library)
  #
  # If your project builds multiple extensions, you can
  # replace this flag by NB_SHARED to conserve space by
  # reusing a shared libnanobind across libraries
  NB_STATIC

  # Source code goes here
  ${EXTENSION_SOURCES}
)

target_compile_features(${PACKAGE_NAME} PRIVATE cxx_std_17)

# Generate type stubs for the extension module
nanobind_add_stub(
  ${PACKAGE_NAME}_stub
  MODULE ${PACKAGE_NAME}
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}.pyi
  PYTHON_PATH $<TARGET_FILE_DIR:${PACKAGE_NAME}>
  # Have stubgen create py.typed as a tracked output instead of writing it by hand.
  MARKER_FILE py.typed
  DEPENDS ${PACKAGE_NAME}
  # Keep names ending in "_" (common for Python keyword workarounds) in stubs.
  INCLUDE_PRIVATE
)

# Install the extension module as a package (nanobind_example/__init__.abi3.so)
# Renaming at install time keeps the build artifacts named normally for stub generation
get_target_property(_EXT_SUFFIX ${PACKAGE_NAME} SUFFIX)
install(FILES $<TARGET_FILE:${PACKAGE_NAME}>
  DESTINATION ${PACKAGE_NAME}
  RENAME __init__${_EXT_SUFFIX}
)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/py.typed" DESTINATION ${PACKAGE_NAME})
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PACKAGE_NAME}.pyi
  DESTINATION ${PACKAGE_NAME}
  RENAME __init__.pyi
)
