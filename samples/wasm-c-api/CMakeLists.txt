# Copyright (C) 2019 Intel Corporation. All rights reserved.
# SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception

cmake_minimum_required (VERSION 2.8)

if (NOT WAMR_BUILD_PLATFORM STREQUAL "windows")
  project(c-api)
else()
  project (c-api C ASM)
  enable_language (ASM_MASM)
endif()

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()
################  runtime settings  ################
string (TOLOWER ${CMAKE_HOST_SYSTEM_NAME} WAMR_BUILD_PLATFORM)
if (APPLE)
  add_definitions(-DBH_PLATFORM_DARWIN)
endif ()

# Resetdefault linker flags
set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS "")
set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS "")

# WAMR features switch
set(WAMR_BUILD_TARGET "X86_64")

if(NOT DEFINED WAMR_BUILD_INTERP)
  set(WAMR_BUILD_INTERP 1)
endif()

if(NOT DEFINED WAMR_BUILD_AOT)
  set(WAMR_BUILD_AOT 0)
endif()

if(NOT DEFINED WAMR_BUILD_JOT)
  set(WAMR_BUILD_JIT 0)
endif()

set(WAMR_BUILD_LIBC_BUILTIN 1)
set(WAMR_BUILD_LIBC_WASI 0)
set(WAMR_BUILD_MULTI_MODULE 1)

if(NOT DEFINED WAMR_BUILD_FAST_INTERP)
  set(WAMR_BUILD_FAST_INTERP 0)
endif()

if (NOT MSVC)
  # compiling and linking flags
  set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -pie -fPIE")
  if (NOT (CMAKE_C_COMPILER MATCHES ".*clang.*" OR CMAKE_C_COMPILER_ID MATCHES ".*Clang"))
    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--gc-sections")
  endif ()
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wformat -Wformat-security")
  if (WAMR_BUILD_TARGET MATCHES "X86_.*" OR WAMR_BUILD_TARGET STREQUAL "AMD_64")
    if (NOT (CMAKE_C_COMPILER MATCHES ".*clang.*" OR CMAKE_C_COMPILER_ID MATCHES ".*Clang"))
      set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mindirect-branch-register")
    endif ()
  endif ()
endif()
# build out vmlib
set(WAMR_ROOT_DIR ${CMAKE_CURRENT_LIST_DIR}/../..)
set(WAMRC ${WAMR_ROOT_DIR}/wamr-compiler/build/wamrc)
include (${WAMR_ROOT_DIR}/build-scripts/runtime_lib.cmake)

add_library(vmlib STATIC ${WAMR_RUNTIME_LIB_SOURCE})
if (MSVC)
  target_compile_definitions(vmlib PRIVATE WASM_API_EXTERN=)
endif()
################################################

################  application related  ################
## locate wat2wasm
find_program(WAT2WASM
  wat2wasm
  PATHS /opt/wabt/bin /opt/wabt-1.0.18/bin
  REQUIRED
)

if(NOT WAT2WASM)
  message(SEND_ERROR "can not find wat2wasm")
endif()

include (${SHARED_DIR}/utils/uncommon/shared_uncommon.cmake)

set(MM_UTIL src/utils/multi_module_utils.c)
# build executable for each .c
file(GLOB SOURCES src/*.c)
foreach(SRC ${SOURCES})
  get_filename_component(APPNAME ${SRC} NAME_WE)

  add_executable(${APPNAME} ${SRC} ${UNCOMMON_SHARED_SOURCE} ${MM_UTIL})
  target_include_directories(${APPNAME} PRIVATE ${UNCOMMON_SHARED_DIR})
  target_link_libraries(${APPNAME} vmlib -lpthread -lm)
  if (MSVC)
    target_compile_definitions(${APPNAME} PRIVATE WASM_API_EXTERN=)
  endif()
endforeach()

# wat to wasm
file(GLOB WAT_FILES src/*.wat)
foreach(WAT_FILE ${WAT_FILES})
  get_filename_component(WATNAME ${WAT_FILE} NAME_WE)

  add_custom_target(${WATNAME}_WASM ALL
    COMMAND ${WAT2WASM} ${WAT_FILE} -o ${PROJECT_BINARY_DIR}/${WATNAME}.wasm
    DEPENDS ${WAT_FILE}
    BYPRODUCTS ${PROJECT_BINARY_DIR}/${WATNAME}.wasm
    VERBATIM
    SOURCES ${WAT_FILE}
  )
endforeach()
################################################
