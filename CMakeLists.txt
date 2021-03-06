# ==================================================================================================
# CMake
# ==================================================================================================
cmake_minimum_required(VERSION 3.1)

# ==================================================================================================
# Project declaration
# ==================================================================================================
project(TNT)

# ==================================================================================================
# Options
# ==================================================================================================

option(ENABLE_JAVA "Compile Java projects, requires a JDK and the JAVA_HOME env var" ON)

option(USE_EXTERNAL_GLES3 "Experimental: Compile Filament against OpenGL ES 3" OFF)

# ==================================================================================================
# OS specific
# ==================================================================================================
if (UNIX AND NOT APPLE AND NOT ANDROID AND NOT WEBGL)
    set(LINUX TRUE)
endif()

if (WIN32)
    # Link statically against c/c++ lib to avoid missing redistriburable such as
    # "VCRUNTIME140.dll not found. Try reinstalling the app.", but give users
    # a choice to opt for the shared runtime if they want.
    option(USE_STATIC_CRT "Link against the static runtime libraries." ON)

    # On Windows we need to instruct cmake to generate the .def in order to get the .lib required
    # when linking against dlls. CL.EXE will not generate .lib without .def file (or without pragma
    # __declspec(dllexport) in front of each functions).
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

    if (${USE_STATIC_CRT})
        set(CRT_FLAGS_RELEASE "/MT")
        set(CRT_FLAGS_DEBUG "/MTd")
    else()
        set(CRT_FLAGS_RELEASE "/MD")
        set(CRT_FLAGS_DEBUG "/MDd")
    endif()

    # TODO: Figure out why pdb generation messes with incremental compilaton.
    # IN RELEASE_WITH_DEBUG_INFO, generate debug info in .obj, no in pdb.
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} ${CRT_FLAGS_RELEASE} /Z7")
    set(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} ${CRT_FLAGS_RELEASE} /Z7")

    # In RELEASE, also generate PDBs.
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${CRT_FLAGS_RELEASE} /Zi")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${CRT_FLAGS_RELEASE} /Zi")

    # In DEBUG, avoid generating a PDB file which seems to mess with incremental compilation.
    # Instead generate debug info directly inside obj files.
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${CRT_FLAGS_DEBUG} /Z7")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${CRT_FLAGS_DEBUG} /Z7")
endif()

# ==================================================================================================
#  Paths
# ==================================================================================================
# Where our external libs are
set(EXTERNAL ${CMAKE_CURRENT_SOURCE_DIR}/third_party)

# Where our libraries are
set(LIBRARIES ${CMAKE_CURRENT_SOURCE_DIR}/libs)

# Where our filament code is
set(FILAMENT ${CMAKE_CURRENT_SOURCE_DIR})

# Where our tools are
set(TOOLS ${CMAKE_CURRENT_SOURCE_DIR}/tools)

# ==================================================================================================
# Compiler check
# ==================================================================================================
set(MIN_CLANG_VERSION "5.0")

if (CMAKE_C_COMPILER_ID MATCHES "Clang")
    if (CMAKE_C_COMPILER_VERSION VERSION_LESS MIN_CLANG_VERSION)
        message(FATAL_ERROR "Detected C compiler Clang ${CMAKE_C_COMPILER_VERSION} < ${MIN_CLANG_VERSION}")
    endif()
else()
    message(FATAL_ERROR "Detected C compiler ${CMAKE_C_COMPILER_ID} is unsupported")
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS MIN_CLANG_VERSION)
        message(FATAL_ERROR "Detected CXX compiler Clang ${CMAKE_CXX_COMPILER_VERSION} < ${MIN_CLANG_VERSION}")
    endif()
else()
    message(FATAL_ERROR "Detected CXX compiler ${CMAKE_CXX_COMPILER_ID} is unsupported")
endif()

# Detect use of the clang-cl.exe frontend, which does not support all of clangs normal options
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" AND "${CMAKE_CXX_SIMULATE_ID}" STREQUAL "MSVC")
    set(CLANG_CL true)
endif()

# ==================================================================================================
# General compiler flags
# ==================================================================================================
set(CXX_STANDARD "-std=c++14")
if (WIN32)
    set(CXX_STANDARD "/std:c++14")
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CXX_STANDARD} -fstrict-aliasing -Wno-unknown-pragmas -Wno-unused-function")

if (USE_EXTERNAL_GLES3)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DUSE_EXTERNAL_GLES3")
endif()

if (WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_USE_MATH_DEFINES=1")
endif()

if (LINUX)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
    if (FILAMENT_REQUIRES_CXXABI)
        # Required in CI environment with custom libc++ and libc++abi
        link_libraries("-lc++abi")
    endif()
    # To distribute our binaries, we must remove the dependency on libc++ and libgcc.
    if (CMAKE_BUILD_TYPE STREQUAL "Release")
        link_libraries("-static-libgcc -static-libstdc++")
    endif()
else()
    if (FILAMENT_REQUIRES_CXXABI)
        message("The option FILAMENT_REQUIRES_CXXABI is unsupported on this platform")
    endif()
endif()

if (CYGWIN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions -fno-rtti")
endif()

if (CLANG_CL)
    # Since the "secure" replacements that MSVC suggests are not portable, disable
    # the deprecation warnings. Also disable warnings about use of POSIX functions (i.e. "unlink").
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_DEPRECATE")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_DEPRECATE")
endif()

# Add colors to ninja builds
if (UNIX AND CMAKE_GENERATOR STREQUAL "Ninja")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fcolor-diagnostics")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcolor-diagnostics")
endif()

# ==================================================================================================
# Release compiler flags
# ==================================================================================================
if (NOT CLANG_CL)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fomit-frame-pointer -ffunction-sections -fdata-sections")
endif()

# On Android RELEASE builds, we disable exceptions and RTTI to save some space (about 75 KiB
# saved by -fno-exception and 10 KiB saved by -fno-rtti).
if (ANDROID OR WEBGL)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fno-exceptions -fno-unwind-tables -fno-asynchronous-unwind-tables -fno-rtti")
endif()

# ==================================================================================================
# Debug compiler flags
# ==================================================================================================
# ASAN is deactivated for now because:
#  -fsanitize=undefined causes extremely long link times
#  -fsanitize=address causes a crash with assimp, which we can't explain for now
#set(EXTRA_SANITIZE_OPTIONS "-fsanitize=undefined -fsanitize=address")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fstack-protector")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${EXTRA_SANITIZE_OPTIONS}")

# ==================================================================================================
# Linker flags
# ==================================================================================================
# Strip unused sections
set(GC_SECTIONS "-Wl,--gc-sections")
set(B_SYMBOLIC_FUNCTIONS "-Wl,-Bsymbolic-functions")

if (APPLE)
    set(GC_SECTIONS "-Wl,-dead_strip")
    set(B_SYMBOLIC_FUNCTIONS "")

    # tell ranlib to ignore empty compilation units
    set(CMAKE_C_ARCHIVE_FINISH   "<CMAKE_RANLIB> -no_warning_for_no_symbols <TARGET>")
    set(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols <TARGET>")
    # prevents ar from invoking ranlib, let CMake do it
    set(CMAKE_C_ARCHIVE_CREATE   "<CMAKE_AR> qc -S <TARGET> <LINK_FLAGS> <OBJECTS>")
    set(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> qc -S <TARGET> <LINK_FLAGS> <OBJECTS>")
elseif (CLANG_CL)
    set(GC_SECTIONS "")
    set(B_SYMBOLIC_FUNCTIONS "")
endif()

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${GC_SECTIONS}")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${GC_SECTIONS} ${B_SYMBOLIC_FUNCTIONS}")

if (WEBGL)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -s USE_WEBGL2=1")
endif()

# ==================================================================================================
# Project flags
# ==================================================================================================
# Debug modes only
if (CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    set(TNT_DEV true)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DTNT_DEV")
endif()

# By default, build with Vulkan support on desktop platforms, although clients must request to use
# it at run time. On Android, the build does not include Vulkan support unless CMake is invoked
# with -DFILAMENT_SUPPORTS_VULKAN=ON.
# Vulkan is not supported on Windows.
if (ANDROID OR WIN32 OR WEBGL)
    option(FILAMENT_SUPPORTS_VULKAN "Include the Vulkan backend" OFF)
else()
    option(FILAMENT_SUPPORTS_VULKAN "Include the Vulkan backend" ON)
endif()
if (FILAMENT_SUPPORTS_VULKAN)
    add_definitions(-DFILAMENT_DRIVER_SUPPORTS_VULKAN)
endif()

# ==================================================================================================
# Distribution
# ==================================================================================================
# choose where to put the objects in the dist folder
if (NOT DIST_ARCH)
    set(DIST_ARCH x86_64)
endif()
if (NOT DIST_DIR)
    set(DIST_DIR "${DIST_ARCH}")
endif()

# ==================================================================================================
# Functions
# ==================================================================================================
function(list_licenses OUTPUT MODULES)
    file(WRITE ${OUTPUT} "R\"FILAMENT__(\n")
    set(_MODULES ${MODULES} ${ARGN})
    foreach(module ${_MODULES})
        set(license_path "../../third_party/${module}/LICENSE")
        get_filename_component(fullname "${license_path}" ABSOLUTE)
        file(APPEND ${OUTPUT} "License and copyrights for ${module}:\n\n")
        file(READ ${license_path} license)
        file(APPEND ${OUTPUT} ${license})
        file(APPEND ${OUTPUT} "\n\n")
    endforeach()
    file(APPEND ${OUTPUT} ")FILAMENT__\"\n")
endfunction(list_licenses)

# ==================================================================================================
# Configuration for CMAKE_CROSSCOMPILING.
# ==================================================================================================
if (WEBGL)
    set(IMPORT_EXECUTABLES ${FILAMENT}/${IMPORT_EXECUTABLES_DIR}/ImportExecutables-Release.cmake)
else()
    set(IMPORT_EXECUTABLES ${FILAMENT}/${IMPORT_EXECUTABLES_DIR}/ImportExecutables-${CMAKE_BUILD_TYPE}.cmake)
endif()

# ==================================================================================================
# Try to find Vulkan if the SDK is installed, otherwise fall back to the bundled version.
# This needs to stay in our top-level CMakeLists because it sets up variables that are used by the
# "bluevk" and "samples" targets.
# ==================================================================================================

if (FILAMENT_SUPPORTS_VULKAN AND APPLE)
    find_library(Vulkan_LIBRARY NAMES vulkan HINTS "$ENV{VULKAN_SDK}/lib" "$ENV{VULKAN_SDK}/macOS/lib")
    if (Vulkan_LIBRARY)
        set(Vulkan_FOUND ON)
        message(STATUS "Found Vulkan library in SDK: ${Vulkan_LIBRARY}.")
        add_definitions(-DFILAMENT_VKLIBRARY_PATH=\"${Vulkan_LIBRARY}\")
    endif()
endif()

# ==================================================================================================
# Common Functions
# ==================================================================================================

# Sets the following variables: RESGEN_HEADER, RESGEN_SOURCE, RESGEN_FLAGS, RESGEN_SOURCE_FLAGS,
# and RESGEN_OUTPUTS
function(get_resgen_vars ARCHIVE_DIR ARCHIVE_NAME)
    set(OUTPUTS
        ${ARCHIVE_DIR}/${ARCHIVE_NAME}.bin
        ${ARCHIVE_DIR}/${ARCHIVE_NAME}.S
        ${ARCHIVE_DIR}/${ARCHIVE_NAME}.apple.S
        ${ARCHIVE_DIR}/${ARCHIVE_NAME}.h
    )
    if (NOT WIN32)
        set(ASM_ARCH_FLAG "-arch ${DIST_ARCH}")
    endif()
    if (APPLE)
        set(ASM_SUFFIX ".apple")
    endif()
    if (WEBGL)
        set(RESGEN_HEADER "${ARCHIVE_DIR}/${ARCHIVE_NAME}.h" PARENT_SCOPE)
        set(RESGEN_OUTPUTS "${OUTPUTS};${ARCHIVE_DIR}/${ARCHIVE_NAME}.c" PARENT_SCOPE)
        set(RESGEN_FLAGS -cx ${ARCHIVE_DIR} -p ${ARCHIVE_NAME} PARENT_SCOPE)
        set(RESGEN_SOURCE "${ARCHIVE_DIR}/${ARCHIVE_NAME}.c" PARENT_SCOPE)
    else()
        set(RESGEN_HEADER "${ARCHIVE_DIR}/${ARCHIVE_NAME}.h" PARENT_SCOPE)
        set(RESGEN_OUTPUTS "${OUTPUTS}" PARENT_SCOPE)
        set(RESGEN_FLAGS -x ${ARCHIVE_DIR} -p ${ARCHIVE_NAME} PARENT_SCOPE)
        set(RESGEN_SOURCE "${ARCHIVE_DIR}/${ARCHIVE_NAME}${ASM_SUFFIX}.S" PARENT_SCOPE)
        set(RESGEN_SOURCE_FLAGS "-I${ARCHIVE_DIR} ${ASM_ARCH_FLAG}" PARENT_SCOPE)
    endif()
endfunction()

# ==================================================================================================
# Sub-projects
# ==================================================================================================
# spirv-tools must come before filamat, as filamat relies on the presence of the
# spirv-tools_SOURCE_DIR variable.
add_subdirectory(${EXTERNAL}/spirv-tools)
add_subdirectory(${EXTERNAL}/glslang/tnt)
add_subdirectory(${EXTERNAL}/spirv-cross/tnt)

# Common to all platforms
add_subdirectory(${EXTERNAL}/libgtest/tnt)
add_subdirectory(${LIBRARIES}/filabridge)
add_subdirectory(${LIBRARIES}/filaflat)
add_subdirectory(${LIBRARIES}/filamat)
add_subdirectory(${LIBRARIES}/filameshio)
add_subdirectory(${LIBRARIES}/image)
add_subdirectory(${LIBRARIES}/math)
add_subdirectory(${LIBRARIES}/utils)
add_subdirectory(${FILAMENT}/filament)
add_subdirectory(${FILAMENT}/shaders)
add_subdirectory(${EXTERNAL}/robin-map/tnt)
add_subdirectory(${EXTERNAL}/smol-v/tnt)
add_subdirectory(${EXTERNAL}/benchmark/tnt)
add_subdirectory(${EXTERNAL}/meshoptimizer)

if (FILAMENT_SUPPORTS_VULKAN)
    add_subdirectory(${LIBRARIES}/bluevk)
    add_subdirectory(${EXTERNAL}/vkmemalloc/tnt)
endif()

if (APPLE)
    add_subdirectory(${EXTERNAL}/moltenvk/tnt)
endif()

set (FILAMENT_SAMPLES_BINARY_DIR ${PROJECT_BINARY_DIR}/samples)

if (WEBGL)
    add_subdirectory(web/filament-js)
    add_subdirectory(web/samples)
    add_subdirectory(web/docs)

    add_subdirectory(${EXTERNAL}/imgui/tnt)
    add_subdirectory(${EXTERNAL}/stb/tnt)
endif()

if (NOT ANDROID AND NOT WEBGL AND NOT IOS)
    add_subdirectory(${FILAMENT}/samples)

    add_subdirectory(${LIBRARIES}/bluegl)
    add_subdirectory(${LIBRARIES}/filagui)
    add_subdirectory(${LIBRARIES}/imageio)

    add_subdirectory(${FILAMENT}/java)
    add_subdirectory(${EXTERNAL}/astcenc/tnt)
    add_subdirectory(${EXTERNAL}/etc2comp)
    add_subdirectory(${EXTERNAL}/getopt)
    add_subdirectory(${EXTERNAL}/imgui/tnt)
    add_subdirectory(${EXTERNAL}/libassimp/tnt)
    add_subdirectory(${EXTERNAL}/libpng/tnt)
    add_subdirectory(${EXTERNAL}/libsdl2/tnt)
    add_subdirectory(${EXTERNAL}/libz/tnt)
    add_subdirectory(${EXTERNAL}/skylight/tnt)
    add_subdirectory(${EXTERNAL}/stb/tnt)
    add_subdirectory(${EXTERNAL}/tinyexr/tnt)

    add_subdirectory(${TOOLS}/cmgen)
    add_subdirectory(${TOOLS}/filamesh)
    add_subdirectory(${TOOLS}/matc)
    add_subdirectory(${TOOLS}/matinfo)
    add_subdirectory(${TOOLS}/mipgen)
    add_subdirectory(${TOOLS}/normal-blending)
    add_subdirectory(${TOOLS}/resgen)
    add_subdirectory(${TOOLS}/roughness-prefilter)
    add_subdirectory(${TOOLS}/skygen)
    add_subdirectory(${TOOLS}/specular-color)
endif()

# Generate exported executables for cross-compiled builds (Android, WebGL, and iOS)
if (NOT CMAKE_CROSSCOMPILING)
    export(TARGETS matc cmgen filamesh mipgen resgen FILE ${IMPORT_EXECUTABLES})
endif()
