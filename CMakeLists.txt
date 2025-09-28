cmake_minimum_required(VERSION 3.16)
project(Dolphin LANGUAGES C CXX)

# --------------------------------------------------------------------
# Basic settings
# --------------------------------------------------------------------
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif()

# --------------------------------------------------------------------
# Options (many mirror options from large projects)
# --------------------------------------------------------------------
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(ENABLE_LIBRETRO "Build libretro core (dolphin_libretro)" ON)
option(ENABLE_GUI "Build GUI (qt/gtk frontends)" OFF)
option(ENABLE_TESTS "Build unit tests" OFF)
option(ENABLE_LTO "Enable LTO" OFF)
option(ENABLE_NEON "Enable NEON optimizations when available" ON)
option(ENABLE_GENERIC "Force generic interpreter build (disable JIT)" OFF)
option(ENABLE_WERROR "Treat warnings as errors" OFF)
option(ENABLE_STRIP "Strip binaries in release" ON)
option(ENABLE_EXTRAS "Build extra helper tools" OFF)

# --------------------------------------------------------------------
# Platform detection
# --------------------------------------------------------------------
if(NOT ANDROID)
  set(BUILD_PLATFORM "${CMAKE_SYSTEM_NAME}")
else()
  set(BUILD_PLATFORM "Android")
endif()

message(STATUS "Dolphin build:")
message(STATUS "  Platform: ${BUILD_PLATFORM}")
message(STATUS "  Build Type: ${CMAKE_BUILD_TYPE}")
message(STATUS "  Shared libs: ${BUILD_SHARED_LIBS}")
message(STATUS "  Libretro: ${ENABLE_LIBRETRO}")
message(STATUS "  Generic (disable JIT): ${ENABLE_GENERIC}")

# --------------------------------------------------------------------
# Android defaults (override via -DANDROID_PLATFORM or via NDK)
# --------------------------------------------------------------------
if(DEFINED ENV{ANDROID_NDK_HOME} AND NOT ANDROID)
    set(ANDROID_TOOLCHAIN_FILE $ENV{ANDROID_NDK_HOME}/build/cmake/android.toolchain.cmake)
endif()

if(ANDROID)
    if(NOT DEFINED ANDROID_PLATFORM)
        set(ANDROID_PLATFORM 21 CACHE STRING "Android API level")
    endif()
    if(NOT DEFINED ANDROID_ABI)
        set(ANDROID_ABI "armeabi-v7a" CACHE STRING "Android ABI")
    endif()
    message(STATUS "Android ABI = ${ANDROID_ABI}, PLATFORM = ${ANDROID_PLATFORM}")
endif()

# --------------------------------------------------------------------
# Force generic (no-JIT) on ARMv7 when requested or when building on Android 32-bit
# --------------------------------------------------------------------
if(ANDROID AND ANDROID_ABI MATCHES "armeabi-v7a")
    set(ENABLE_GENERIC ON CACHE BOOL "Force generic interpreter build for ARMv7")
endif()

if(ENABLE_GENERIC)
    add_definitions(-DDOLPHIN_GENERIC_BUILD -DDISABLE_JIT)
    message(STATUS "Generic interpreter build enabled (JIT disabled).")
endif()

# --------------------------------------------------------------------
# Compiler flags and warnings
# --------------------------------------------------------------------
if(MSVC)
    add_compile_options("/W3")
else()
    add_compile_options(-Wall -Wextra -Wno-unused-parameter)
    if(ENABLE_WERROR)
        add_compile_options(-Werror)
    endif()
endif()

# LTO
if(ENABLE_LTO AND CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    include(CheckIPOSupported)
    check_ipo_supported(RESULT LTO_SUPPORTED)
    if(LTO_SUPPORTED)
        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
        message(STATUS "LTO enabled")
    else()
        message(WARNING "LTO not supported by this compiler")
    endif()
endif()

# --------------------------------------------------------------------
# Project structure variables
# --------------------------------------------------------------------
set(SRC_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/Source)
set(EXTERNALS_ROOT ${CMAKE_CURRENT_SOURCE_DIR}/Externals)

# Convenience list of expected Source subfolders (will be tested)
set(EXPECTED_MODULE_DIRS
    "${SRC_ROOT}/Core"
    "${SRC_ROOT}/Common"
    "${SRC_ROOT}/VideoCommon"
    "${SRC_ROOT}/InputCommon"
    "${SRC_ROOT}/AudioCommon"
    "${SRC_ROOT}/DiscIO"
    "${SRC_ROOT}/Libretro"
)

# Print what exists
foreach(dir IN LISTS EXPECTED_MODULE_DIRS)
  if(EXISTS ${dir})
    message(STATUS "Found module dir: ${dir}")
  else()
    message(WARNING "Missing expected module dir: ${dir} (some functionality may be reduced)")
  endif()
endforeach()

# --------------------------------------------------------------------
# Externals: try to add known externals if present
# --------------------------------------------------------------------
if(EXISTS "${EXTERNALS_ROOT}")
    message(STATUS "Externals folder present")
    if(EXISTS "${EXTERNALS_ROOT}/zlib")
        add_subdirectory(${EXTERNALS_ROOT}/zlib)
        set(HAVE_ZLIB TRUE)
    endif()
    if(EXISTS "${EXTERNALS_ROOT}/minizip")
        add_subdirectory(${EXTERNALS_ROOT}/minizip)
        set(HAVE_MINIZIP TRUE)
    endif()
    if(EXISTS "${EXTERNALS_ROOT}/libpng")
        add_subdirectory(${EXTERNALS_ROOT}/libpng)
        set(HAVE_LIBPNG TRUE)
    endif()
    # add other externals detection as needed...
endif()

# --------------------------------------------------------------------
# Helper function: add module as static lib if sources exist
# --------------------------------------------------------------------
function(add_module name path)
    set(module_dir "${SRC_ROOT}/${path}")
    if(EXISTS "${module_dir}")
        file(GLOB_RECURSE module_sources CONFIGURE_DEPENDS
             "${module_dir}/*.cpp" "${module_dir}/*.c" "${module_dir}/*.cc" "${module_dir}/*.cxx")
        if(module_sources)
            add_library(${name} STATIC ${module_sources})
            target_include_directories(${name} PUBLIC ${module_dir} ${SRC_ROOT})
            message(STATUS "Added module ${name} from ${module_dir} (${module_sources})")
        else()
            message(WARNING "No sources found for module ${name} in ${module_dir}")
            # create a dummy to avoid missing target issues
            file(WRITE "${CMAKE_BINARY_DIR}/dummy_${name}.cpp" "int dummy_${name}(){return 0;}\n")
            add_library(${name} STATIC "${CMAKE_BINARY_DIR}/dummy_${name}.cpp")
            target_include_directories(${name} PUBLIC ${module_dir})
        endif()
    else()
        message(WARNING "Module directory ${module_dir} does not exist, creating dummy target ${name}")
        file(WRITE "${CMAKE_BINARY_DIR}/dummy_${name}.cpp" "int dummy_${name}(){return 0;}\n")
        add_library(${name} STATIC "${CMAKE_BINARY_DIR}/dummy_${name}.cpp")
    endif()
endfunction()

# --------------------------------------------------------------------
# Add main modules (these names mirror typical dolphin structure)
# --------------------------------------------------------------------
add_module(dolphin_common "Common")
add_module(dolphin_core "Core")
add_module(dolphin_videocommon "VideoCommon")
add_module(dolphin_inputcommon "InputCommon")
add_module(dolphin_audiocommon "AudioCommon")
add_module(dolphin_discio "DiscIO")
add_module(dolphin_libretro_frontend "Libretro")

# optional UI or tools modules if present
if(EXISTS "${SRC_ROOT}/UI")
    add_module(dolphin_ui "UI")
endif()
if(EXISTS "${SRC_ROOT}/Tools")
    add_module(dolphin_tools "Tools")
endif()

# --------------------------------------------------------------------
# Configure platform-specific options
# --------------------------------------------------------------------
if(ANDROID)
    find_program(ANDROID_NDK_PROG NAMES ndk-build)
    if(DEFINED ANDROID_TOOLCHAIN_FILE)
        set(CMAKE_TOOLCHAIN_FILE ${ANDROID_TOOLCHAIN_FILE} CACHE PATH "Android toolchain file")
    endif()
    add_definitions(-D__ANDROID__)
endif()

# Architecture-specific tuning
if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm" OR CMAKE_ANDROID_ARCH_ABI STREQUAL "armeabi-v7a")
    if(ENABLE_NEON)
        add_definitions(-DENABLE_NEON)
    endif()
endif()

# --------------------------------------------------------------------
# Collect libretro frontend sources explicitly, prefer Source/Libretro then fallback to other paths
# --------------------------------------------------------------------
set(LIBRETRO_SRC_DIR "${SRC_ROOT}/Libretro")
file(GLOB_RECURSE LIBRETRO_SOURCES CONFIGURE_DEPENDS
     "${LIBRETRO_SRC_DIR}/*.cpp" "${LIBRETRO_SRC_DIR}/*.c" "${LIBRETRO_SRC_DIR}/*.cc")

if(NOT LIBRETRO_SOURCES)
    # Try alternative locations
    file(GLOB_RECURSE LIBRETRO_SOURCES CONFIGURE_DEPENDS
         "${SRC_ROOT}/libretro/*.cpp" "${SRC_ROOT}/libretro/*.c")
endif()

# --------------------------------------------------------------------
# Build the libretro target (shared library)
# --------------------------------------------------------------------
if(ENABLE_LIBRETRO)
    add_library(dolphin_libretro SHARED ${LIBRETRO_SOURCES})

    # include top-level sources optionally inside libretro (if libretro front-end needs access to core)
    # Link static modules into the shared libretro so it becomes self-contained
    target_link_libraries(dolphin_libretro
        PRIVATE
            dolphin_core
            dolphin_common
            dolphin_videocommon
            dolphin_inputcommon
            dolphin_audiocommon
            dolphin_discio
    )

    # If externals produced libs, link them
    if(HAVE_ZLIB)
        target_link_libraries(dolphin_libretro PRIVATE zlibstatic)
    elseif(TARGET z)
        target_link_libraries(dolphin_libretro PRIVATE z)
    endif()

    # Platform-specific link items
    if(ANDROID)
        target_link_libraries(dolphin_libretro PRIVATE log android EGL GLESv2 m)
    else()
        find_package(OpenGL QUIET)
        if(OpenGL_FOUND)
            target_link_libraries(dolphin_libretro PRIVATE OpenGL::GL)
        endif()
    endif()

    # compile definitions for libretro core
    target_compile_definitions(dolphin_libretro PRIVATE -DLIBRETRO -DDOLPHIN_LIBRETRO -DDOLPHIN_GENERIC_BUILD)
    target_include_directories(dolphin_libretro PRIVATE ${SRC_ROOT})
endif()

# --------------------------------------------------------------------
# Tools / extras (optional)
# --------------------------------------------------------------------
if(ENABLE_EXTRAS)
    if(EXISTS "${SRC_ROOT}/Tools")
        file(GLOB_RECURSE TOOLS_SOURCES "${SRC_ROOT}/Tools/*.cpp")
        add_executable(dolphin_tool ${TOOLS_SOURCES})
        target_link_libraries(dolphin_tool PRIVATE dolphin_core dolphin_common)
    endif()
endif()

# --------------------------------------------------------------------
# Tests
# --------------------------------------------------------------------
if(ENABLE_TESTS)
    enable_testing()
    if(EXISTS "${SRC_ROOT}/Tests")
        file(GLOB_RECURSE TEST_SOURCES "${SRC_ROOT}/Tests/*.cpp")
        add_executable(dolphin_tests ${TEST_SOURCES})
        target_link_libraries(dolphin_tests PRIVATE dolphin_core dolphin_common)
        add_test(NAME dolphin_core_tests COMMAND dolphin_tests)
    endif()
endif()

# --------------------------------------------------------------------
# Install rules (so users can install built core)
# --------------------------------------------------------------------
if(ENABLE_LIBRETRO)
    install(TARGETS dolphin_libretro
            LIBRARY DESTINATION lib
            RUNTIME DESTINATION bin)
endif()

# --------------------------------------------------------------------
# Post-build: strip binary on release for smaller artifact (Android)
# --------------------------------------------------------------------
if(ENABLE_STRIP AND CMAKE_BUILD_TYPE STREQUAL "Release")
    if(ANDROID)
        add_custom_command(TARGET dolphin_libretro POST_BUILD
            COMMAND ${CMAKE_STRIP} $<TARGET_FILE:dolphin_libretro>
            COMMENT "Stripping dolphin_libretro for release")
    else()
        add_custom_command(TARGET dolphin_libretro POST_BUILD
            COMMAND ${CMAKE_STRIP} $<TARGET_FILE:dolphin_libretro>
            COMMENT "Stripping dolphin_libretro for release")
    endif()
endif()

# --------------------------------------------------------------------
# Summary output
# --------------------------------------------------------------------
message(STATUS "Configuration summary:")
message(STATUS "  ENABLE_LIBRETRO = ${ENABLE_LIBRETRO}")
message(STATUS "  ENABLE_GENERIC = ${ENABLE_GENERIC}")
message(STATUS "  ANDROID = ${ANDROID}")
message(STATUS "  ANDROID_ABI = ${ANDROID_ABI}")
message(STATUS "  ANDROID_PLATFORM = ${ANDROID_PLATFORM}")
message(STATUS "  Externals: zlib=${HAVE_ZLIB}, minizip=${HAVE_MINIZIP}, libpng=${HAVE_LIBPNG}")

# End of CMakeLists.txt
