cmake_minimum_required(VERSION 3.21)
project(quickjs-builder LANGUAGES C)

if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "No CMAKE_BUILD_TYPE specified, defaulting to Debug")
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "Build type" FORCE)
endif()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/lib")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/lib")

if(WIN32)
  # MoonBit native objects on Windows are built against the static CRT
  # (`LIBCMT`), so keep the QuickJS archive aligned with that runtime.
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded")
endif()

include(FetchContent)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

cmake_policy(SET CMP0169 OLD)

set(QUICKJS_GIT_TAG
    "d7ae12ae71dfd6ab2997527d295014a8996fa0f9"
    CACHE STRING "Pinned upstream QuickJS revision")

FetchContent_Declare(
    quickjs_upstream
    GIT_REPOSITORY https://github.com/bellard/quickjs.git
    GIT_TAG "${QUICKJS_GIT_TAG}"
    GIT_SHALLOW TRUE)

FetchContent_GetProperties(quickjs_upstream)
if(NOT quickjs_upstream_POPULATED)
  FetchContent_Populate(quickjs_upstream)
endif()

file(READ "${quickjs_upstream_SOURCE_DIR}/VERSION" QUICKJS_UPSTREAM_VERSION)
string(STRIP "${QUICKJS_UPSTREAM_VERSION}" QUICKJS_UPSTREAM_VERSION)
string(REPLACE "-" "." QUICKJS_CMAKE_VERSION "${QUICKJS_UPSTREAM_VERSION}")

set(GENERATED_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(GENERATED_INCLUDE_DIR "${GENERATED_DIR}/include")
set(PATCHED_SOURCE_DIR "${GENERATED_DIR}/quickjs")

file(MAKE_DIRECTORY "${GENERATED_INCLUDE_DIR}/sys")
file(MAKE_DIRECTORY "${PATCHED_SOURCE_DIR}")

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/compat/sys/time.h"
    "${GENERATED_INCLUDE_DIR}/sys/time.h"
    COPYONLY)

file(READ "${quickjs_upstream_SOURCE_DIR}/cutils.h" CUTILS_H_SOURCE)
file(READ "${quickjs_upstream_SOURCE_DIR}/quickjs.h" QUICKJS_H_SOURCE)
file(READ "${quickjs_upstream_SOURCE_DIR}/quickjs.c" QUICKJS_C_SOURCE)

string(REPLACE
    "return (JSValue)v;"
    "return v;"
    QUICKJS_H_SOURCE
    "${QUICKJS_H_SOURCE}")

if(WIN32)
  string(REPLACE
      "#if defined(EMSCRIPTEN)\n#define DIRECT_DISPATCH  0\n#else\n#define DIRECT_DISPATCH  1\n#endif"
      "#if defined(EMSCRIPTEN) || defined(_WIN32)\n#define DIRECT_DISPATCH  0\n#else\n#define DIRECT_DISPATCH  1\n#endif"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "-1.0 / 0.0"
      "-INFINITY"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "1.0 / 0.0"
      "INFINITY"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
endif()

if(WIN32)
  set(QUICKJS_ATOMICS_BLOCK [=[/* define to include Atomics.* operations which depend on the OS
   threads */
#if !defined(EMSCRIPTEN)
#define CONFIG_ATOMICS
#endif]=])

  set(QUICKJS_WINDOWS_ATOMICS_BLOCK [=[/* define to include Atomics.* operations which depend on the OS
   threads */
#if !defined(EMSCRIPTEN) && !defined(_WIN32)
#define CONFIG_ATOMICS
#endif]=])

  string(REPLACE
      "${QUICKJS_ATOMICS_BLOCK}"
      "${QUICKJS_WINDOWS_ATOMICS_BLOCK}"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")

  string(REPLACE
      "prop = js_symbol_to_atom(ctx, (JSValue)name);"
      "prop = js_symbol_to_atom(ctx, name);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "prs = find_own_property(&pr, p, js_symbol_to_atom(ctx, (JSValue)brand));"
      "prs = find_own_property(&pr, p, js_symbol_to_atom(ctx, brand));"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "val = (JSValueConst)JS_NewUint32(ctx, array_length);"
      "val = JS_NewUint32(ctx, array_length);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "sf->cur_func = (JSValue)func_obj;"
      "sf->cur_func = func_obj;"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "JS_FreeValue(ctx, (JSValue)args[0]);"
      "JS_FreeValue(ctx, args[0]);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "JS_FreeValue(ctx, (JSValue)args[1]);"
      "JS_FreeValue(ctx, args[1]);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "args[args_len++] = (JSValueConst)str;"
      "args[args_len++] = str;"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "return (JSValue)val;"
      "return val;"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "return (JSValueConst)map_normalize_key(ctx, (JSValue)key);"
      "return map_normalize_key(ctx, key);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "args[2] = (JSValue)this_val;"
      "args[2] = this_val;"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "resolve_element_data[1] = (JSValueConst)JS_NewInt32(ctx, index);"
      "resolve_element_data[1] = JS_NewInt32(ctx, index);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "func_data[0] = (JSValueConst)JS_NewBool(ctx, done);"
      "func_data[0] = JS_NewBool(ctx, done);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "static void __attribute((unused)) dump_token(JSParseState *s,"
      "static void dump_token(JSParseState *s,"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "JS_FreeValue(ctx, (JSValue)argv[0]);"
      "JS_FreeValue(ctx, argv[0]);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
  string(REPLACE
      "JS_FreeValue(ctx, (JSValue)argv[1]);"
      "JS_FreeValue(ctx, argv[1]);"
      QUICKJS_C_SOURCE
      "${QUICKJS_C_SOURCE}")
endif()

file(WRITE "${GENERATED_INCLUDE_DIR}/cutils.h" "${CUTILS_H_SOURCE}")
file(WRITE "${GENERATED_INCLUDE_DIR}/quickjs.h" "${QUICKJS_H_SOURCE}")
file(WRITE "${PATCHED_SOURCE_DIR}/quickjs.c" "${QUICKJS_C_SOURCE}")

add_library(
    quickjs STATIC
    "${PATCHED_SOURCE_DIR}/quickjs.c"
    "${quickjs_upstream_SOURCE_DIR}/cutils.c"
    "${quickjs_upstream_SOURCE_DIR}/dtoa.c"
    "${quickjs_upstream_SOURCE_DIR}/libregexp.c"
    "${quickjs_upstream_SOURCE_DIR}/libunicode.c")

set_target_properties(
    quickjs
    PROPERTIES
      OUTPUT_NAME quickjs
      C_STANDARD 11
      C_STANDARD_REQUIRED YES
      C_EXTENSIONS YES)

target_include_directories(
    quickjs
    PUBLIC
      "$<BUILD_INTERFACE:${GENERATED_INCLUDE_DIR}>"
      "$<BUILD_INTERFACE:${quickjs_upstream_SOURCE_DIR}>"
      "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/quickjs>")

target_compile_definitions(
    quickjs
    PRIVATE
      CONFIG_VERSION="${QUICKJS_UPSTREAM_VERSION}")

if(MSVC)
  file(TO_NATIVE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/compat/msvc_compat.h" MSVC_COMPAT_HEADER)
  target_compile_options(quickjs PRIVATE "/FI${MSVC_COMPAT_HEADER}")
  target_compile_definitions(
      quickjs
      PRIVATE
        _CRT_SECURE_NO_WARNINGS
        _CRT_NONSTDC_NO_WARNINGS
        NOMINMAX)
else()
  target_compile_options(quickjs PRIVATE -fwrapv)
endif()

target_link_libraries(quickjs INTERFACE "$<$<PLATFORM_ID:Linux>:m>")

install(
    TARGETS quickjs
    EXPORT quickjsTargets
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}")

install(
    FILES
      "${GENERATED_INCLUDE_DIR}/quickjs.h"
      "${quickjs_upstream_SOURCE_DIR}/LICENSE"
      "${quickjs_upstream_SOURCE_DIR}/VERSION"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/quickjs")

write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/quickjs-config-version.cmake"
    VERSION "${QUICKJS_CMAKE_VERSION}"
    COMPATIBILITY ExactVersion)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/quickjs-config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/quickjs-config.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/quickjs")

install(
    FILES
      "${CMAKE_CURRENT_BINARY_DIR}/quickjs-config.cmake"
      "${CMAKE_CURRENT_BINARY_DIR}/quickjs-config-version.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/quickjs")

install(
    EXPORT quickjsTargets
    FILE quickjs-targets.cmake
    NAMESPACE quickjs::
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/quickjs")
