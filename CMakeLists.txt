# Copyright (c) 2016 The Chromium Embedded Framework Authors. All rights
# reserved. Use of this source code is governed by a BSD-style license that
# can be found in the LICENSE file.

# See the included README.md file for usage instructions.

# For VS2022 and Xcode 12+ support.
cmake_minimum_required(VERSION 3.21)

# Only generate Debug and Release configuration types.
set(CMAKE_CONFIGURATION_TYPES Debug Release)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)


# Project name.
project(cef)

# Use folders in the resulting project files.
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
#
# CEF configuration.
#

# Specify the CEF distribution version.
set(CEF_VERSION "135.0.17+gcbc1c5b+chromium-135.0.7049.52")

# Determine the platform.
if("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  if("${PROJECT_ARCH}" STREQUAL "arm64")
    set(CEF_PLATFORM "macosarm64")
  elseif("${PROJECT_ARCH}" STREQUAL "x86_64")
    set(CEF_PLATFORM "macosx64")
  elseif("${CMAKE_HOST_SYSTEM_PROCESSOR}" STREQUAL "arm64")
    set(PROJECT_ARCH "arm64")
    set(CEF_PLATFORM "macosarm64")
  else()
    set(PROJECT_ARCH "x86_64")
    set(CEF_PLATFORM "macosx64")
  endif()
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  if("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "arm")
    set(CEF_PLATFORM "linuxarm")
  elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "arm64")
    set(CEF_PLATFORM "linuxarm64")
  elseif(CMAKE_SIZEOF_VOID_P MATCHES 8)
    set(CEF_PLATFORM "linux64")
  else()
    message(FATAL_ERROR "Linux x86 32-bit builds are discontinued.")
  endif()
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Windows")
  if("${CMAKE_CXX_COMPILER_ARCHITECTURE_ID}" STREQUAL "ARM64")
    set(CEF_PLATFORM "windowsarm64")
  elseif(CMAKE_SIZEOF_VOID_P MATCHES 8)
    set(CEF_PLATFORM "windows64")
  else()
    set(CEF_PLATFORM "windows32")
  endif()
endif()

# Add this project's cmake/ directory to the module path.
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Download and extract the CEF binary distribution (executes DownloadCEF.cmake).
include(DownloadCEF)
DownloadCEF("${CEF_PLATFORM}" "${CEF_VERSION}" "${CMAKE_SOURCE_DIR}/third_party/cef")

# Add the CEF binary distribution's cmake/ directory to the module path.
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CEF_ROOT}/cmake")

# Load the CEF configuration (executes FindCEF.cmake).
find_package(CEF REQUIRED)


#
# Python configuration.
#

# Support specification of the Python executable path via the command-line.
if(DEFINED ENV{PYTHON_EXECUTABLE})
  file(TO_CMAKE_PATH "$ENV{PYTHON_EXECUTABLE}" PYTHON_EXECUTABLE)
endif()

if(NOT PYTHON_EXECUTABLE)
  unset(PYTHON_EXECUTABLE)

  # Find the python interpreter.
  find_package(PythonInterp)

  if(NOT ${PYTHONINTERP_FOUND})
    message(FATAL_ERROR "A Python installation is required. Set the "
      "PYTHON_EXECUTABLE environment variable to explicitly "
      "specify the Python executable path.")
  endif()
endif()

message(STATUS "Using Python: ${PYTHON_EXECUTABLE}")


#
# Clang-format configuration.
#

if(OS_WINDOWS)
  set(GS_PLATFORM "win32")
  set(GS_HASHPATH "win/clang-format.exe.sha1")
  set(GS_OUTPATH "win/clang-format.exe")
elseif(OS_MACOSX)
  set(GS_PLATFORM "darwin")
  if("${CMAKE_HOST_SYSTEM_PROCESSOR}" STREQUAL "arm64")
    set(GS_HASHPATH "mac/clang-format.arm64.sha1")
  else()
    set(GS_HASHPATH "mac/clang-format.x64.sha1")
  endif()
  set(GS_OUTPATH "mac/clang-format")
elseif(OS_LINUX)
  set(GS_PLATFORM "linux*")
  set(GS_HASHPATH "linux64/clang-format.sha1")
  set(GS_OUTPATH "linux64/clang-format")
endif()

message(STATUS "Downloading clang-format from Google Storage...")
# execute_process(
#   COMMAND "${PYTHON_EXECUTABLE}"
#           "tools/buildtools/download_from_google_storage.py"
#           "--no_resume"
#           "--platform=${GS_PLATFORM}"
#           "--no_auth"
#           "--bucket" "chromium-clang-format"
#           "-s" "tools/buildtools/${GS_HASHPATH}"
#           "-o" "tools/buildtools/${GS_OUTPATH}"
#   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
#   RESULT_VARIABLE EXECUTE_RV
#   )
# if(NOT EXECUTE_RV STREQUAL "0")
#   message(STATUS "Download failed with variables:")
#   message(STATUS "  PYTHON_EXECUTABLE=${PYTHON_EXECUTABLE}")
#   message(STATUS "  GS_PLATFORM=${GS_PLATFORM}")
#   message(STATUS "  GS_HASHPATH=${GS_HASHPATH}")
#   message(STATUS "  GS_OUTPATH=${GS_OUTPATH}")
#   message(STATUS "  CMAKE_CURRENT_SOURCE_DIR=${CMAKE_CURRENT_SOURCE_DIR}")

#   message(STATUS "Failed command: ${PYTHON_EXECUTABLE} tools/buildtools/download_from_google_storage.py --no_resume --platform=${GS_PLATFORM} --no_auth --bucket chromium-clang-format -s tools/buildtools/${GS_HASHPATH} -o tools/buildtools/${GS_OUTPATH}")

#   message(FATAL_ERROR "Execution failed with unexpected result: ${EXECUTE_RV}")
# endif()


#
# Target configuration.
#

# Include the libcef_dll_wrapper target (executes libcef_dll/CMakeLists.txt).
add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll_wrapper)

# Include CEF's test application targets (executes <target>/CMakeLists.txt).
add_subdirectory(${CEF_ROOT}/tests/cefclient)
add_subdirectory(${CEF_ROOT}/tests/cefsimple)
add_subdirectory(${CEF_ROOT}/tests/gtest)
add_subdirectory(${CEF_ROOT}/tests/ceftests)


# Allow includes relative to the current source directory.
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

# Configure building of the examples.
option(WITH_EXAMPLES "Enable or disable building of examples." ON)
if(WITH_EXAMPLES)
  # Include example application targets.
  add_subdirectory(examples)
endif()

# TODO: Include other application targets here.

# Display configuration settings.
PRINT_CEF_CONFIG()


if(NOT WITH_EXAMPLES)
  message(STATUS "")
  message(STATUS "NOTE: Targets from the examples folder are not included.")
  message(STATUS "Add -DWITH_EXAMPLES=On to the CMake command-line to include them.")
  message(STATUS "")
endif()
