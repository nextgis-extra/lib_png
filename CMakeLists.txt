# CMakeLists.txt

# Copyright (C) 2007,2009-2017 Glenn Randers-Pehrson
# Written by Christian Ehrlicher, 2007
# Revised by Roger Lowman, 2009-2010
# Revised by Clifford Yapp, 2011-2012
# Revised by Roger Leigh, 2016
# Revised by Andreas Franek, 2016
#
# This code is released under the libpng license.
# For conditions of distribution and use, see the disclaimer
# and license in png.h
#
# Modified for borsch v2
# Copyright (C) 2016-2018, NextGIS <info@nextgis.com>
# Auhor: Dmitry Baryshnikov <dmitry.baryshnikov@nextgis.com>

cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS true)

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

project(libpng)
enable_testing()

set(PACKAGE_NAME png)
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
  set(PNG_FRAMEWORK ON CACHE INTERNAL "OSX library type" FORCE)
  set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
  set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
  set(INSTALL_INC_DIR ${INSTALL_LIB_DIR}/${PACKAGE_NAME}.framework/Headers CACHE INTERNAL "Installation directory for headers" FORCE)
  set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PACKAGE_NAME}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
  set(SKIP_INSTALL_HEADERS ON)
  set(SKIP_INSTALL_EXECUTABLES ON)
  set(SKIP_INSTALL_FILES ON)
  set(SKIP_INSTALL_EXPORT ON)
  set(CMAKE_MACOSX_RPATH ON)
else()
  include(GNUInstallDirs)

  set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
  set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
  set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE INTERNAL "Installation directory for headers" FORCE)
  set(INSTALL_MAN_DIR ${CMAKE_INSTALL_MANDIR} CACHE INTERNAL "Installation directory for man files" FORCE)
  set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PACKAGE_NAME}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
endif()

if(CMAKE_GENERATOR_TOOLSET MATCHES "v([0-9]+)_xp")
    add_definitions(-D_WIN32_WINNT=0x0501)
endif()

include(util)

check_version(PNGLIB_MAJOR PNGLIB_MINOR PNGLIB_RELEASE)

set(PNGLIB_NAME libpng${PNGLIB_MAJOR}${PNGLIB_MINOR})
set(PNGLIB_VERSION ${PNGLIB_MAJOR}.${PNGLIB_MINOR}.${PNGLIB_RELEASE})
set(VERSION ${PNGLIB_VERSION})

report_version(${PROJECT_NAME} ${VERSION})

if(OSX_FRAMEWORK)
    set(FRAMEWORK_VERSION "${PNGLIB_MAJOR}")
endif()

configure_file(${CMAKE_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake IMMEDIATE @ONLY)


# needed packages
include(FindAnyProject)
find_anyproject(ZLIB REQUIRED)

if(NOT WIN32)
  find_library(M_LIBRARY
    NAMES m
    PATHS /usr/lib /usr/local/lib
  )
  set(TARGET_LINK_LIB ${TARGET_LINK_LIB} m)
  if(NOT M_LIBRARY)
    message(STATUS "math lib 'libm' not found; floating point support disabled")
  endif()
else()
  # not needed on windows
  set(M_LIBRARY "")
endif()

# COMMAND LINE OPTIONS
if(NOT OSX_FRAMEWORK)
    if(BUILD_SHARED_LIBS)
        set(PNG_SHARED ON)
    else()
        set(PNG_STATIC ON)
    endif()
endif()

option(PNG_TESTS  "Build libpng tests" ON)

# Many more configuration options could be added here
option(PNG_DEBUG     "Build with debug output" OFF)
option(PNGARG        "Disable ANSI-C prototypes" OFF)

option(PNG_HARDWARE_OPTIMIZATIONS "Enable Hardware Optimizations" ON)

if(CMAKE_CROSSCOMPILING AND ( NEON OR ENABLE_NEON OR CMAKE_SYSTEM_PROCESSOR STREQUAL aarch64 ))
    set(PNG_ARM_NEON "on")
endif()

if(PNG_HARDWARE_OPTIMIZATIONS)
# set definitions and sources for arm
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm" OR
  CMAKE_SYSTEM_PROCESSOR MATCHES "^aarch64")
  set(PNG_ARM_NEON_POSSIBLE_VALUES check on off)
  set(PNG_ARM_NEON "check" CACHE STRING "Enable ARM NEON optimizations:
     check: (default) use internal checking code;
     off: disable the optimizations;
     on: turn on unconditionally.")
  set_property(CACHE PNG_ARM_NEON PROPERTY STRINGS
     ${PNG_ARM_NEON_POSSIBLE_VALUES})
  list(FIND PNG_ARM_NEON_POSSIBLE_VALUES ${PNG_ARM_NEON} index)
  if(index EQUAL -1)
    message(FATAL_ERROR
      " PNG_ARM_NEON must be one of [${PNG_ARM_NEON_POSSIBLE_VALUES}]")
  elseif(NOT ${PNG_ARM_NEON} STREQUAL "no")
    set(libpng_arm_sources
      arm/arm_init.c
      arm/filter_neon.S
      arm/filter_neon_intrinsics.c
      arm/palette_neon_intrinsics.c)

    if(${PNG_ARM_NEON} STREQUAL "on")
      add_definitions(-DPNG_ARM_NEON_OPT=2)
    elseif(${PNG_ARM_NEON} STREQUAL "check")
      add_definitions(-DPNG_ARM_NEON_CHECK_SUPPORTED)
    endif()
  else()
    add_definitions(-DPNG_ARM_NEON_OPT=0)
  endif()
endif()

# set definitions and sources for powerpc
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^powerpc*" OR
	CMAKE_SYSTEM_PROCESSOR MATCHES "^ppc64*" )
  set(PNG_POWERPC_VSX_POSSIBLE_VALUES on off)
  set(PNG_POWERPC_VSX "on" CACHE STRING "Enable POWERPC VSX optimizations:
     off: disable the optimizations.")
  set_property(CACHE PNG_POWERPC_VSX PROPERTY STRINGS
     ${PNG_POWERPC_VSX_POSSIBLE_VALUES})
  list(FIND PNG_POWERPC_VSX_POSSIBLE_VALUES ${PNG_POWERPC_VSX} index)
  if(index EQUAL -1)
    message(FATAL_ERROR
      " PNG_POWERPC_VSX must be one of [${PNG_POWERPC_VSX_POSSIBLE_VALUES}]")
  elseif(NOT ${PNG_POWERPC_VSX} STREQUAL "no")
    set(libpng_powerpc_sources
      powerpc/powerpc_init.c
      powerpc/filter_vsx_intrinsics.c)
    if(${PNG_POWERPC_VSX} STREQUAL "on")
      add_definitions(-DPNG_POWERPC_VSX_OPT=2)
    endif()
  else()
    add_definitions(-DPNG_POWERPC_VSX_OPT=0)
  endif()
endif()

# set definitions and sources for intel
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^i?86" OR
	CMAKE_SYSTEM_PROCESSOR MATCHES "^x86_64*" )
  set(PNG_INTEL_SSE_POSSIBLE_VALUES on off)
  set(PNG_INTEL_SSE "on" CACHE STRING "Enable INTEL_SSE optimizations:
     off: disable the optimizations")
  set_property(CACHE PNG_INTEL_SSE PROPERTY STRINGS
     ${PNG_INTEL_SSE_POSSIBLE_VALUES})
  list(FIND PNG_INTEL_SSE_POSSIBLE_VALUES ${PNG_INTEL_SSE} index)
  if(index EQUAL -1)
    message(FATAL_ERROR
      " PNG_INTEL_SSE must be one of [${PNG_INTEL_SSE_POSSIBLE_VALUES}]")
  elseif(NOT ${PNG_INTEL_SSE} STREQUAL "no")
    set(libpng_intel_sources
      intel/intel_init.c
      intel/filter_sse2_intrinsics.c)
    if(${PNG_INTEL_SSE} STREQUAL "on")
      add_definitions(-DPNG_INTEL_SSE_OPT=1)
    endif()
  else()
    add_definitions(-DPNG_INTEL_SSE_OPT=0)
  endif()
endif()

# set definitions and sources for MIPS
if(CMAKE_SYSTEM_PROCESSOR MATCHES "mipsel*" OR
	CMAKE_SYSTEM_PROCESSOR MATCHES "mips64el*" )
  set(PNG_MIPS_MSA_POSSIBLE_VALUES on off)
  set(PNG_MIPS_MSA "on" CACHE STRING "Enable MIPS_MSA optimizations:
     off: disable the optimizations")
  set_property(CACHE PNG_MIPS_MSA PROPERTY STRINGS
     ${PNG_MIPS_MSA_POSSIBLE_VALUES})
  list(FIND PNG_MIPS_MSA_POSSIBLE_VALUES ${PNG_MIPS_MSA} index)
  if(index EQUAL -1)
    message(FATAL_ERROR
      " PNG_MIPS_MSA must be one of [${PNG_MIPS_MSA_POSSIBLE_VALUES}]")
  elseif(NOT ${PNG_MIPS_MSA} STREQUAL "no")
    set(libpng_mips_sources
      mips/mips_init.c
      mips/filter_msa_intrinsics.c)
    if(${PNG_MIPS_MSA} STREQUAL "on")
      add_definitions(-DPNG_MIPS_MSA_OPT=2)
    endif()
  else()
    add_definitions(-DPNG_MIPS_MSA_OPT=0)
  endif()
endif()
endif(PNG_HARDWARE_OPTIMIZATIONS)

# SET LIBNAME
set(PNG_LIB_NAME png${PNGLIB_MAJOR}${PNGLIB_MINOR})

# to distinguish between debug and release lib
set(CMAKE_DEBUG_POSTFIX "d")

include(CheckCSourceCompiles)
option(ld-version-script "Enable linker version script" ON)
if(ld-version-script AND NOT APPLE)
  # Check if LD supports linker scripts.
  file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/conftest.map" "VERS_1 {
        global: sym;
        local: *;
};

VERS_2 {
        global: sym2;
                main;
} VERS_1;
")
  set(CMAKE_REQUIRED_FLAGS_SAVE ${CMAKE_REQUIRED_FLAGS})
  set(CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS} "-Wl,--version-script='${CMAKE_CURRENT_BINARY_DIR}/conftest.map'")
  check_c_source_compiles("void sym(void) {}
void sym2(void) {}
int main(void) {return 0;}
" HAVE_LD_VERSION_SCRIPT)
  if(NOT HAVE_LD_VERSION_SCRIPT)
    set(CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS_SAVE} "-Wl,-M -Wl,${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
  check_c_source_compiles("void sym(void) {}
void sym2(void) {}
int main(void) {return 0;}
" HAVE_SOLARIS_LD_VERSION_SCRIPT)
  endif()
  set(CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS_SAVE})
  file(REMOVE "${CMAKE_CURRENT_BINARY_DIR}/conftest.map")
endif()

# Find symbol prefix.  Likely obsolete and unnecessary with recent
# toolchains (it's not done in many other projects).
function(symbol_prefix)
  set(SYMBOL_PREFIX)

  execute_process(COMMAND "${CMAKE_C_COMPILER}" "-E" "-"
                  INPUT_FILE /dev/null
                  OUTPUT_VARIABLE OUT
                  RESULT_VARIABLE STATUS)

  if(CPP_FAIL)
    message(WARNING "Failed to run the C preprocessor")
  endif()

  string(REPLACE "\n" ";" OUT "${OUT}")
  foreach(line ${OUT})
    string(REGEX MATCH "^PREFIX=" found_match "${line}")
    if(found_match)
      STRING(REGEX REPLACE "^PREFIX=(.*\)" "\\1" prefix "${line}")
      string(REGEX MATCH "__USER_LABEL_PREFIX__" found_match "${prefix}")
      if(found_match)
        STRING(REGEX REPLACE "(.*)__USER_LABEL_PREFIX__(.*)" "\\1\\2" prefix "${prefix}")
      endif()
      set(SYMBOL_PREFIX "${prefix}")
    endif()
  endforeach()

    message(STATUS "Symbol prefix: ${SYMBOL_PREFIX}")
    set(SYMBOL_PREFIX "${SYMBOL_PREFIX}" PARENT_SCOPE)
endfunction()

if(UNIX)
  symbol_prefix()
endif()

# find_program(AWK NAMES gawk awk)

include_directories(${CMAKE_CURRENT_BINARY_DIR})

if(NOT AWK OR ANDROID)
  # No awk available to generate sources; use pre-built pnglibconf.h
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/pnglibconf.h.prebuilt
                 ${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h)
  add_custom_target(genfiles) # Dummy
else()
  include(CMakeParseArguments)
  # Generate .chk from .out with awk
  # generate_chk(INPUT inputfile OUTPUT outputfile [DEPENDS dep1 [dep2...]])
  function(generate_chk)
    set(options)
    set(oneValueArgs INPUT OUTPUT)
    set(multiValueArgs DEPENDS)
    cmake_parse_arguments(_GC "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})
    if (NOT _GC_INPUT)
      message(FATAL_ERROR "Invalid arguments.  generate_out requires input.")
    endif()
    if (NOT _GC_OUTPUT)
      message(FATAL_ERROR "Invalid arguments.  generate_out requires output.")
    endif()

    add_custom_command(OUTPUT "${_GC_OUTPUT}"
                       COMMAND "${CMAKE_COMMAND}"
                               "-DINPUT=${_GC_INPUT}"
                               "-DOUTPUT=${_GC_OUTPUT}"
                               -P "${CMAKE_CURRENT_BINARY_DIR}/scripts/genchk.cmake"
                       DEPENDS "${_GC_INPUT}" ${_GC_DEPENDS}
                       WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
  endfunction()

  # Generate .out from .c with awk
  # generate_out(INPUT inputfile OUTPUT outputfile [DEPENDS dep1 [dep2...]])
  function(generate_out)
    set(options)
    set(oneValueArgs INPUT OUTPUT)
    set(multiValueArgs DEPENDS)
    cmake_parse_arguments(_GO "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})
    if (NOT _GO_INPUT)
      message(FATAL_ERROR "Invalid arguments.  generate_out requires input.")
    endif()
    if (NOT _GO_OUTPUT)
      message(FATAL_ERROR "Invalid arguments.  generate_out requires output.")
    endif()

    add_custom_command(OUTPUT "${_GO_OUTPUT}"
                       COMMAND "${CMAKE_COMMAND}"
                               "-DINPUT=${_GO_INPUT}"
                               "-DOUTPUT=${_GO_OUTPUT}"
                               -P "${CMAKE_CURRENT_BINARY_DIR}/scripts/genout.cmake"
                       DEPENDS "${_GO_INPUT}" ${_GO_DEPENDS}
                       WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
  endfunction()

  # Generate specific source file with awk
  # generate_source(OUTPUT outputfile [DEPENDS dep1 [dep2...]])
  function(generate_source)
    set(options)
    set(oneValueArgs OUTPUT)
    set(multiValueArgs DEPENDS)
    cmake_parse_arguments(_GSO "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})
    if (NOT _GSO_OUTPUT)
      message(FATAL_ERROR "Invalid arguments.  generate_source requires output.")
    endif()

    add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${_GSO_OUTPUT}"
                       COMMAND "${CMAKE_COMMAND}"
                               "-DOUTPUT=${_GSO_OUTPUT}"
                               -P "${CMAKE_CURRENT_BINARY_DIR}/scripts/gensrc.cmake"
                       DEPENDS ${_GSO_DEPENDS}
                       WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
  endfunction()

  # Copy file
  function(generate_copy source destination)
    add_custom_command(OUTPUT "${destination}"
                       COMMAND "${CMAKE_COMMAND}" -E remove "${destination}"
                       COMMAND "${CMAKE_COMMAND}" -E copy "${source}"
                                                          "${destination}"
                       DEPENDS "${source}")
  endfunction()

  # Generate scripts/pnglibconf.h
  generate_source(OUTPUT "scripts/pnglibconf.c"
                  DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/scripts/pnglibconf.dfa"
                          "${CMAKE_CURRENT_SOURCE_DIR}/scripts/options.awk"
                          "${CMAKE_CURRENT_SOURCE_DIR}/pngconf.h")

  # Generate pnglibconf.c
  generate_source(OUTPUT "pnglibconf.c"
                  DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/scripts/pnglibconf.dfa"
                          "${CMAKE_CURRENT_SOURCE_DIR}/scripts/options.awk"
                          "${CMAKE_CURRENT_SOURCE_DIR}/pngconf.h")

  if(PNG_PREFIX)
    set(PNGLIBCONF_H_EXTRA_DEPENDS
        "${CMAKE_CURRENT_BINARY_DIR}/scripts/prefix.out"
        "${CMAKE_CURRENT_SOURCE_DIR}/scripts/macro.lst")
    set(PNGPREFIX_H_EXTRA_DEPENDS
        "${CMAKE_CURRENT_BINARY_DIR}/scripts/intprefix.out")
  endif()

  generate_out(INPUT "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.c"
               OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.out")

  # Generate pnglibconf.h
  generate_source(OUTPUT "pnglibconf.h"
                  DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.out"
                          ${PNGLIBCONF_H_EXTRA_DEPENDS})

  generate_out(INPUT "${CMAKE_CURRENT_SOURCE_DIR}/scripts/intprefix.c"
               OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/scripts/intprefix.out"
               DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h")

  generate_out(INPUT "${CMAKE_CURRENT_SOURCE_DIR}/scripts/prefix.c"
               OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/scripts/prefix.out"
               DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/png.h"
                       "${CMAKE_CURRENT_SOURCE_DIR}/pngconf.h"
                       "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.out")

  # Generate pngprefix.h
  generate_source(OUTPUT "pngprefix.h"
                  DEPENDS ${PNGPREFIX_H_EXTRA_DEPENDS})

  generate_out(INPUT "${CMAKE_CURRENT_SOURCE_DIR}/scripts/sym.c"
               OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/scripts/sym.out"
               DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h")

  generate_out(INPUT "${CMAKE_CURRENT_SOURCE_DIR}/scripts/symbols.c"
               OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/scripts/symbols.out"
               DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/png.h"
                       "${CMAKE_CURRENT_SOURCE_DIR}/pngconf.h"
                       "${CMAKE_CURRENT_SOURCE_DIR}/scripts/pnglibconf.h.prebuilt")

  generate_out(INPUT "${CMAKE_CURRENT_SOURCE_DIR}/scripts/vers.c"
               OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/scripts/vers.out"
               DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/png.h"
                       "${CMAKE_CURRENT_SOURCE_DIR}/pngconf.h"
                       "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h")

  generate_chk(INPUT "${CMAKE_CURRENT_BINARY_DIR}/scripts/symbols.out"
               OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/scripts/symbols.chk"
               DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/scripts/checksym.awk"
                       "${CMAKE_CURRENT_SOURCE_DIR}/scripts/symbols.def")

  add_custom_target(symbol-check DEPENDS
                    "${CMAKE_CURRENT_BINARY_DIR}/scripts/symbols.chk")

  generate_copy("${CMAKE_CURRENT_BINARY_DIR}/scripts/sym.out"
                "${CMAKE_CURRENT_BINARY_DIR}/libpng.sym")
  generate_copy("${CMAKE_CURRENT_BINARY_DIR}/scripts/vers.out"
                "${CMAKE_CURRENT_BINARY_DIR}/libpng.vers")

  add_custom_target(genvers DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/libpng.vers")
  add_custom_target(gensym DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/libpng.sym")

  add_custom_target("genprebuilt"
                    COMMAND "${CMAKE_COMMAND}"
                            "-DOUTPUT=scripts/pnglibconf.h.prebuilt"
                            -P "${CMAKE_CURRENT_BINARY_DIR}/scripts/gensrc.cmake"
                    WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")

  # A single target handles generation of all generated files.  If
  # they are dependend upon separately by multiple targets, this
  # confuses parallel make (it would require a separate top-level
  # target for each file to track the dependencies properly).
  add_custom_target(genfiles DEPENDS
    "${CMAKE_CURRENT_BINARY_DIR}/libpng.sym"
    "${CMAKE_CURRENT_BINARY_DIR}/libpng.vers"
    "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.c"
    "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h"
    "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.out"
    "${CMAKE_CURRENT_BINARY_DIR}/pngprefix.h"
    "${CMAKE_CURRENT_BINARY_DIR}/scripts/intprefix.out"
    "${CMAKE_CURRENT_BINARY_DIR}/scripts/pnglibconf.c"
    "${CMAKE_CURRENT_BINARY_DIR}/scripts/prefix.out"
    "${CMAKE_CURRENT_BINARY_DIR}/scripts/sym.out"
    "${CMAKE_CURRENT_BINARY_DIR}/scripts/symbols.chk"
    "${CMAKE_CURRENT_BINARY_DIR}/scripts/symbols.out"
    "${CMAKE_CURRENT_BINARY_DIR}/scripts/vers.out")
endif(NOT AWK OR ANDROID)

# OUR SOURCES
set(libpng_public_hdrs
  png.h
  pngconf.h
  "${CMAKE_CURRENT_BINARY_DIR}/pnglibconf.h"
)
set(libpng_private_hdrs
  pngpriv.h
  pngdebug.h
  pnginfo.h
  pngstruct.h
)
if(AWK AND NOT ANDROID)
  list(APPEND libpng_private_hdrs "${CMAKE_CURRENT_BINARY_DIR}/pngprefix.h")
endif()
set(libpng_sources
  ${libpng_public_hdrs}
  ${libpng_private_hdrs}
  png.c
  pngerror.c
  pngget.c
  pngmem.c
  pngpread.c
  pngread.c
  pngrio.c
  pngrtran.c
  pngrutil.c
  pngset.c
  pngtrans.c
  pngwio.c
  pngwrite.c
  pngwtran.c
  pngwutil.c
  ${libpng_arm_sources}
  ${libpng_intel_sources}
  ${libpng_mips_sources}
  ${libpng_powerpc_sources}
)
set(pngtest_sources
  pngtest.c
)
set(pngvalid_sources
  contrib/libtests/pngvalid.c
)
set(pngstest_sources
  contrib/libtests/pngstest.c
)
set(pngunknown_sources
  contrib/libtests/pngunknown.c
)
set(pngimage_sources
  contrib/libtests/pngimage.c
)
set(pngfix_sources
  contrib/tools/pngfix.c
)
set(png_fix_itxt_sources
  contrib/tools/png-fix-itxt.c
)

if(MSVC)
  add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
endif(MSVC)

if(PNG_DEBUG)
  add_definitions(-DPNG_DEBUG)
endif()

# NOW BUILD OUR TARGET
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

unset(PNG_LIB_TARGETS)

if(PNG_SHARED)
  add_library(png SHARED ${libpng_sources})
  set(PNG_LIB_TARGETS png)
  set_target_properties(png PROPERTIES OUTPUT_NAME ${PNG_LIB_NAME})
  add_dependencies(png genfiles)
  if(MSVC)
    # msvc does not append 'lib' - do it here to have consistent name
    set_target_properties(png PROPERTIES PREFIX "lib")
    set_target_properties(png PROPERTIES IMPORT_PREFIX "lib")
  endif()
  target_link_extlibraries(png)

  if(UNIX AND AWK)
    if(HAVE_LD_VERSION_SCRIPT)
      set_target_properties(png PROPERTIES LINK_FLAGS
        "-Wl,--version-script='${CMAKE_CURRENT_BINARY_DIR}/libpng.vers'")
    elseif(HAVE_SOLARIS_LD_VERSION_SCRIPT)
      set_target_properties(png PROPERTIES LINK_FLAGS
        "-Wl,-M -Wl,'${CMAKE_CURRENT_BINARY_DIR}/libpng.vers'")
    endif()
  endif()
endif()

if(PNG_STATIC)
    # set(PACKAGE_NAME png_static)
  # does not work without changing name
  set(PNG_LIB_NAME_STATIC png_static)
  add_library(png_static STATIC ${libpng_sources})
  add_dependencies(png_static genfiles)
  # MSVC doesn't use a different file extension for shared vs. static
  # libs.  We are able to change OUTPUT_NAME to remove the _static
  # for all other platforms.
  if(NOT MSVC)
    set_target_properties(png_static PROPERTIES
      OUTPUT_NAME "${PNG_LIB_NAME}"
      CLEAN_DIRECT_OUTPUT 1)
  else()
    set_target_properties(png_static PROPERTIES
      OUTPUT_NAME "${PNG_LIB_NAME}_static"
      CLEAN_DIRECT_OUTPUT 1)
  endif()
  list(APPEND PNG_LIB_TARGETS png_static)
  if(MSVC)
    # msvc does not append 'lib' - do it here to have consistent name
    set_target_properties(png_static PROPERTIES PREFIX "lib")
  endif()
  target_link_extlibraries(png_static)
endif()

if(PNG_FRAMEWORK)
  set(PNG_LIB_NAME_FRAMEWORK png)
  add_library(${PNG_LIB_NAME_FRAMEWORK} SHARED ${libpng_sources})
  add_dependencies(${PNG_LIB_NAME_FRAMEWORK} genfiles)
  list(APPEND PNG_LIB_TARGETS ${PNG_LIB_NAME_FRAMEWORK})
  set_target_properties(${PNG_LIB_NAME_FRAMEWORK} PROPERTIES
    FRAMEWORK TRUE
    FRAMEWORK_VERSION "${FRAMEWORK_VERSION}"
    MACOSX_FRAMEWORK_SHORT_VERSION_STRING "${PNGLIB_MAJOR}.${PNGLIB_MINOR}"
    MACOSX_FRAMEWORK_BUNDLE_VERSION ${PNGLIB_VERSION}
    MACOSX_FRAMEWORK_IDENTIFIER org.libpng.libpng
    XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
    PUBLIC_HEADER "${libpng_public_hdrs}")
  target_link_extlibraries(${PNG_LIB_NAME_FRAMEWORK})
endif()

if(NOT PNG_LIB_TARGETS)
  message(SEND_ERROR
    "No library variant selected to build. "
    "Please enable at least one of the following options: "
    " PNG_STATIC, PNG_SHARED, PNG_FRAMEWORK")
endif()

if(PNG_SHARED AND WIN32)
  set_target_properties(png PROPERTIES DEFINE_SYMBOL PNG_BUILD_DLL)
endif()

function(png_add_test)
  set(options)
  set(oneValueArgs NAME COMMAND)
  set(multiValueArgs OPTIONS FILES)
  cmake_parse_arguments(_PAT "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

  if (NOT _PAT_NAME)
    message(FATAL_ERROR "Invalid arguments.  png_add_test requires name.")
  endif()
  if (NOT _PAT_COMMAND)
    message(FATAL_ERROR "Invalid arguments.  png_add_test requires command.")
  endif()

  set(TEST_OPTIONS "${_PAT_OPTIONS}")
  set(TEST_FILES "${_PAT_FILES}")

  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/scripts/test.cmake.in"
                 "${CMAKE_CURRENT_BINARY_DIR}/tests/${_PAT_NAME}.cmake" @ONLY)
  if(CMAKE_MAJOR_VERSION GREATER 2) # have generator expressions
    add_test(NAME "${_PAT_NAME}"
             COMMAND "${CMAKE_COMMAND}"
             "-DLIBPNG=$<TARGET_FILE:png>"
             "-DTEST_COMMAND=$<TARGET_FILE:${_PAT_COMMAND}>"
             -P "${CMAKE_CURRENT_BINARY_DIR}/tests/${_PAT_NAME}.cmake")
  else() # old 2.x add_test; limited and won't work well on Windows
    # Note LIBPNG is a dummy value as there are no generator expressions
    add_test("${_PAT_NAME}" "${CMAKE_COMMAND}"
             "-DLIBPNG=${CMAKE_CURRENT_BINARY_DIR}/libpng.so"
             "-DTEST_COMMAND=./${_PAT_COMMAND}"
             -P "${CMAKE_CURRENT_BINARY_DIR}/tests/${_PAT_NAME}.cmake")
  endif()
endfunction()

if(PNG_TESTS AND NOT CMAKE_CROSSCOMPILING)
  if(WIN32)
      add_custom_command(TARGET ${PNG_LIB_TARGETS}
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -DDST_PATH="$<TARGET_FILE_DIR:${PNG_LIB_TARGETS}>" -DSRC_PATH="${PROJECT_BINARY_DIR}" -P ${PROJECT_SOURCE_DIR}/cmake/test_dependency.cmake
        COMMENT "Copy DLLs"
    )
  endif()
  # Find test PNG files by globbing, but sort lists to ensure
  # consistency between different filesystems.
  file(GLOB PNGSUITE_PNGS "${CMAKE_CURRENT_SOURCE_DIR}/contrib/pngsuite/*.png")
  list(SORT PNGSUITE_PNGS)
  file(GLOB TEST_PNGS "${CMAKE_CURRENT_SOURCE_DIR}/contrib/testpngs/*.png")
  list(SORT TEST_PNGS)

  set(PNGTEST_PNG "${CMAKE_CURRENT_SOURCE_DIR}/pngtest.png")

  add_executable(pngtest ${pngtest_sources})
  target_link_libraries(pngtest PRIVATE png)

  png_add_test(NAME pngtest COMMAND pngtest FILES "${PNGTEST_PNG}")

  add_executable(pngvalid ${pngvalid_sources})
  target_link_libraries(pngvalid PRIVATE png ${ZLIB_LIBRARIES})

  png_add_test(NAME pngvalid-gamma-16-to-8
               COMMAND pngvalid OPTIONS --gamma-16-to-8)
  png_add_test(NAME pngvalid-gamma-alpha-mode
               COMMAND pngvalid OPTIONS --gamma-alpha-mode)
  png_add_test(NAME pngvalid-gamma-background
               COMMAND pngvalid OPTIONS --gamma-background)
  png_add_test(NAME pngvalid-gamma-expand16-alpha-mode
               COMMAND pngvalid OPTIONS --gamma-alpha-mode --expand16)
  png_add_test(NAME pngvalid-gamma-expand16-background
               COMMAND pngvalid OPTIONS --gamma-background --expand16)
  png_add_test(NAME pngvalid-gamma-expand16-transform
               COMMAND pngvalid OPTIONS --gamma-transform --expand16)
  png_add_test(NAME pngvalid-gamma-sbit
               COMMAND pngvalid OPTIONS --gamma-sbit)
  png_add_test(NAME pngvalid-gamma-threshold
               COMMAND pngvalid OPTIONS --gamma-threshold)
  png_add_test(NAME pngvalid-gamma-transform
               COMMAND pngvalid OPTIONS --gamma-transform)
  png_add_test(NAME pngvalid-progressive-interlace-standard
               COMMAND pngvalid OPTIONS --standard --progressive-read --interlace)
  png_add_test(NAME pngvalid-progressive-size
               COMMAND pngvalid OPTIONS --size --progressive-read)
  png_add_test(NAME pngvalid-progressive-standard
               COMMAND pngvalid OPTIONS --standard --progressive-read)
  png_add_test(NAME pngvalid-standard
               COMMAND pngvalid OPTIONS --standard)
  png_add_test(NAME pngvalid-transform
               COMMAND pngvalid OPTIONS --transform)

  add_executable(pngstest ${pngstest_sources})
  target_link_libraries(pngstest PRIVATE png)

  foreach(gamma_type 1.8 linear none sRGB)
    foreach(alpha_type none alpha)
      set(PNGSTEST_FILES)
      foreach(test_png ${TEST_PNGS})
        string(REGEX MATCH ".*-linear[-.].*" TEST_PNG_LINEAR "${test_png}")
        string(REGEX MATCH ".*-sRGB[-.].*" TEST_PNG_SRGB "${test_png}")
        string(REGEX MATCH ".*-1.8[-.].*" TEST_PNG_G18 "${test_png}")
        string(REGEX MATCH ".*-alpha-.*" TEST_PNG_ALPHA "${test_png}")

        set(TEST_PNG_VALID TRUE)

        if(TEST_PNG_ALPHA)
          if (NOT "${alpha_type}" STREQUAL "alpha")
            set(TEST_PNG_VALID FALSE)
          endif()
        else()
          if ("${alpha_type}" STREQUAL "alpha")
            set(TEST_PNG_VALID FALSE)
          endif()
        endif()

        if(TEST_PNG_LINEAR)
          if(NOT "${gamma_type}" STREQUAL "linear")
            set(TEST_PNG_VALID FALSE)
          endif()
        elseif(TEST_PNG_SRGB)
          if(NOT "${gamma_type}" STREQUAL "sRGB")
            set(TEST_PNG_VALID FALSE)
          endif()
        elseif(TEST_PNG_G18)
          if(NOT "${gamma_type}" STREQUAL "1.8")
            set(TEST_PNG_VALID FALSE)
          endif()
        else()
          if(NOT "${gamma_type}" STREQUAL "none")
            set(TEST_PNG_VALID FALSE)
          endif()
        endif()

        if(TEST_PNG_VALID)
          list(APPEND PNGSTEST_FILES "${test_png}")
        endif()
      endforeach()
      # Should already be sorted, but sort anyway to be certain.
      list(SORT PNGSTEST_FILES)
      png_add_test(NAME pngstest-${gamma_type}-${alpha_type}
                   COMMAND pngstest
                   OPTIONS --tmpfile "${gamma_type}-${alpha_type}-" --log
                   FILES ${PNGSTEST_FILES})
    endforeach()
  endforeach()

  add_executable(pngunknown ${pngunknown_sources})
  target_link_libraries(pngunknown PRIVATE png)

  png_add_test(NAME pngunknown-discard COMMAND pngunknown OPTIONS --strict default=discard FILES "${PNGTEST_PNG}")
  png_add_test(NAME pngunknown-IDAT COMMAND pngunknown OPTIONS --strict default=discard IDAT=save FILES "${PNGTEST_PNG}")
  png_add_test(NAME pngunknown-if-safe COMMAND pngunknown OPTIONS --strict default=if-safe FILES "${PNGTEST_PNG}")
  png_add_test(NAME pngunknown-sAPI COMMAND pngunknown OPTIONS --strict bKGD=save cHRM=save gAMA=save all=discard iCCP=save sBIT=save sRGB=save FILES "${PNGTEST_PNG}")
  png_add_test(NAME pngunknown-save COMMAND pngunknown OPTIONS --strict default=save FILES "${PNGTEST_PNG}")
  png_add_test(NAME pngunknown-sTER COMMAND pngunknown OPTIONS --strict sTER=if-safe FILES "${PNGTEST_PNG}")
  png_add_test(NAME pngunknown-vpAg COMMAND pngunknown OPTIONS --strict vpAg=if-safe FILES "${PNGTEST_PNG}")

  add_executable(pngimage ${pngimage_sources})
  target_link_libraries(pngimage png)

  png_add_test(NAME pngimage-quick COMMAND pngimage OPTIONS --list-combos --log FILES ${PNGSUITE_PNGS})
  png_add_test(NAME pngimage-full COMMAND pngimage OPTIONS --exhaustive --list-combos --log FILES ${PNGSUITE_PNGS})
endif()

if(PNG_SHARED OR PNG_FRAMEWORK)
  add_executable(pngfix ${pngfix_sources})
  target_link_libraries(pngfix PRIVATE png ${ZLIB_LIBRARIES})
  set(PNG_BIN_TARGETS pngfix)

  add_executable(png-fix-itxt ${png_fix_itxt_sources})
  target_link_extlibraries(png-fix-itxt)
  list(APPEND PNG_BIN_TARGETS png-fix-itxt)
endif()

# Set a variable with CMake code which:
# Creates a symlink from src to dest (if possible) or alternatively
# copies if different.
include(CMakeParseArguments)

function(CREATE_SYMLINK DEST_FILE)

  cmake_parse_arguments(S "" "FILE;TARGET" "" ${ARGN})

  if(NOT S_TARGET AND NOT S_FILE)
    message(FATAL_ERROR "Specify either a TARGET or a FILE for CREATE_SYMLINK to link to.")
  endif(NOT S_TARGET AND NOT S_FILE)

  if(S_TARGET AND S_FILE)
    message(FATAL_ERROR "CREATE_SYMLINK called with both source file ${S_FILE} and build target ${S_TARGET} arguments - can only handle 1 type per call.")
  endif(S_TARGET AND S_FILE)

  if(S_FILE)
    # If we don't need to symlink something that's coming from a build target,
    # we can go ahead and symlink/copy at configure time.

    if(CMAKE_HOST_WIN32 AND NOT CYGWIN AND NOT MSYS)
      execute_process(
	COMMAND "${CMAKE_COMMAND}" -E copy_if_different ${S_FILE} ${DEST_FILE}
	WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
	)
    else(CMAKE_HOST_WIN32 AND NOT CYGWIN AND NOT MSYS)
      execute_process(
	COMMAND ${CMAKE_COMMAND} -E create_symlink ${S_FILE} ${DEST_FILE}
	WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
	)
    endif(CMAKE_HOST_WIN32 AND NOT CYGWIN AND NOT MSYS)
  endif(S_FILE)

  if(S_TARGET)
    # We need to use generator expressions, which can be a bit tricky, so for
    # simplicity make the symlink a POST_BUILD step and use the TARGET
    # signature of add_custom_command.

    if(CMAKE_HOST_WIN32 AND NOT CYGWIN AND NOT MSYS)
      add_custom_command(TARGET ${S_TARGET} POST_BUILD
	COMMAND "${CMAKE_COMMAND}" -E copy_if_different $<TARGET_LINKER_FILE_NAME:${S_TARGET}> $<TARGET_LINKER_FILE_DIR:${S_TARGET}>/${DEST_FILE}
	)
    else(CMAKE_HOST_WIN32 AND NOT CYGWIN AND NOT MSYS)
      add_custom_command(TARGET ${S_TARGET} POST_BUILD
	COMMAND "${CMAKE_COMMAND}" -E create_symlink $<TARGET_LINKER_FILE_NAME:${S_TARGET}> $<TARGET_LINKER_FILE_DIR:${S_TARGET}>/${DEST_FILE}
	)
    endif(CMAKE_HOST_WIN32 AND NOT CYGWIN AND NOT MSYS)

  endif(S_TARGET)

endfunction()

# Create source generation scripts.
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/genchk.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/genchk.cmake @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/genout.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/genout.cmake @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scripts/gensrc.cmake.in
               ${CMAKE_CURRENT_BINARY_DIR}/scripts/gensrc.cmake @ONLY)


# libpng is a library so default to 'lib'
if(NOT DEFINED CMAKE_INSTALL_LIBDIR)
  set(CMAKE_INSTALL_LIBDIR lib)
endif(NOT DEFINED CMAKE_INSTALL_LIBDIR)

# CREATE PKGCONFIG FILES
# we use the same files like ./configure, so we have to set its vars
# Only do this on Windows for Cygwin - the files don't make much sense outside
# a UNIX look alike
# if(NOT WIN32 OR CYGWIN OR MINGW)
#   set(prefix      ${CMAKE_INSTALL_PREFIX})
#   set(exec_prefix ${CMAKE_INSTALL_PREFIX})
#   set(libdir      ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR})
#   set(includedir  ${CMAKE_INSTALL_PREFIX}/include)
#   set(LIBS        "-lz -lm")
#   configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libpng.pc.in
#     ${CMAKE_CURRENT_BINARY_DIR}/${PNGLIB_NAME}.pc @ONLY)
#   CREATE_SYMLINK(libpng.pc FILE ${PNGLIB_NAME}.pc)
#
#   configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libpng-config.in
#     ${CMAKE_CURRENT_BINARY_DIR}/${PNGLIB_NAME}-config @ONLY)
#   CREATE_SYMLINK(libpng-config FILE ${PNGLIB_NAME}-config)
# endif(NOT WIN32 OR CYGWIN OR MINGW)

# SET UP LINKS
if(PNG_SHARED OR PNG_FRAMEWORK)
  set_target_properties(png PROPERTIES
#   VERSION 16.${PNGLIB_RELEASE}.1.6.34
    VERSION 16.${PNGLIB_RELEASE}.0
    SOVERSION 16
    CLEAN_DIRECT_OUTPUT 1)
endif()

# INSTALL
if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
  install(TARGETS ${PNG_LIB_TARGETS}
      EXPORT ${PACKAGE_UPPER_NAME}Targets
      RUNTIME DESTINATION ${INSTALL_BIN_DIR}
      LIBRARY DESTINATION ${INSTALL_LIB_DIR}
      ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
      INCLUDES DESTINATION ${INSTALL_INC_DIR}
      FRAMEWORK DESTINATION ${INSTALL_LIB_DIR} )

  if(PNG_SHARED)
    # Create a symlink for libpng.dll.a => libpng16.dll.a on Cygwin
    if(CYGWIN OR MINGW)
      CREATE_SYMLINK(libpng${CMAKE_IMPORT_LIBRARY_SUFFIX} TARGET png)
      install(FILES $<TARGET_LINKER_FILE_DIR:png>/libpng${CMAKE_IMPORT_LIBRARY_SUFFIX} DESTINATION ${INSTALL_LIB_DIR})
    endif(CYGWIN OR MINGW)

    if(NOT WIN32)
      CREATE_SYMLINK(libpng${CMAKE_SHARED_LIBRARY_SUFFIX} TARGET png)
      install(FILES $<TARGET_LINKER_FILE_DIR:png>/libpng${CMAKE_SHARED_LIBRARY_SUFFIX} DESTINATION ${INSTALL_LIB_DIR})
    endif(NOT WIN32)
  endif(PNG_SHARED)

  if(PNG_STATIC)
    if(NOT WIN32 OR CYGWIN OR MINGW)
      CREATE_SYMLINK( libpng${CMAKE_STATIC_LIBRARY_SUFFIX} TARGET png_static)
      install(FILES $<TARGET_LINKER_FILE_DIR:png_static>/libpng${CMAKE_STATIC_LIBRARY_SUFFIX} DESTINATION ${INSTALL_LIB_DIR})
    endif(NOT WIN32 OR CYGWIN OR MINGW)
 endif()
endif()

if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL )
  install(FILES ${libpng_public_hdrs}   DESTINATION ${INSTALL_INC_DIR})
  install(FILES ${libpng_public_hdrs}   DESTINATION ${INSTALL_INC_DIR}/${PNGLIB_NAME})
endif()
# if(NOT SKIP_INSTALL_EXECUTABLES AND NOT SKIP_INSTALL_ALL )
#   if(NOT WIN32 OR CYGWIN OR MINGW)
#     install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/libpng-config DESTINATION ${INSTALL_BIN_DIR})
#     install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${PNGLIB_NAME}-config
#             DESTINATION ${INSTALL_BIN_DIR})
#   endif(NOT WIN32 OR CYGWIN OR MINGW)
# endif()

if(NOT SKIP_INSTALL_PROGRAMS AND NOT SKIP_INSTALL_ALL )
  install(TARGETS ${PNG_BIN_TARGETS}
      RUNTIME DESTINATION ${INSTALL_BIN_DIR})
endif()

if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL )
  # Install man pages
  # if(NOT PNG_MAN_DIR)
  #   set(PNG_MAN_DIR "share/man")
  # endif()
  install(FILES libpng.3 libpngpf.3      DESTINATION ${INSTALL_MAN_DIR}/man3)
  install(FILES png.5                    DESTINATION ${INSTALL_MAN_DIR}/man5)
  # Install pkg-config files
  # if(NOT CMAKE_HOST_WIN32 OR CYGWIN OR MINGW)
  #   install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libpng.pc
  #           DESTINATION ${INSTALL_LIB_DIR}/pkgconfig)
  #   install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/libpng-config
  #           DESTINATION ${INSTALL_BIN_DIR})
  #   install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PNGLIB_NAME}.pc
  #           DESTINATION ${INSTALL_LIB_DIR}/pkgconfig)
  #   install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${PNGLIB_NAME}-config
  #           DESTINATION ${INSTALL_BIN_DIR})
  # endif(NOT CMAKE_HOST_WIN32 OR CYGWIN OR MINGW)
endif()

# On versions of CMake that support it, create an export file CMake
# users can include() to import our targets
# if(PNG_EXPORT_RULE AND NOT SKIP_INSTALL_EXPORT AND NOT SKIP_INSTALL_ALL )
#   install(EXPORT libpng DESTINATION lib/libpng FILE lib${PNG_LIB_NAME}.cmake)
# endif()

# what's with libpng-manual.txt and all the extra files?

# UNINSTALL
# do we need this?
add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

# DIST
# do we need this?

# to create msvc import lib for mingw compiled shared lib
# pexports libpng.dll > libpng.def
# lib /def:libpng.def /machine:x86

# Export package ===============================================================

# Add path to includes to build-tree export
target_include_directories(${PNG_LIB_TARGETS} INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
)

# Add all targets to the build-tree export set
export(TARGETS ${PNG_LIB_TARGETS}
    FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

if(REGISTER_PACKAGE)
    # Export the package for use from the build-tree
    # (this registers the build-tree with a global CMake-registry)
    export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the <Package>Config.cmake file
configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    # Install the <Package>Config.cmake
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
