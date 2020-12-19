cmake_minimum_required(VERSION 2.8.12)

if(CMAKE_EXECUTABLE_SUFFIX)
    set(CMAKE_EXECUTABLE_SUFFIX_TMP ${CMAKE_EXECUTABLE_SUFFIX})
endif()

project(libjpeg-turbo C)
set(VERSION 2.0.90)
string(REPLACE "." ";" VERSION_TRIPLET ${VERSION})
list(GET VERSION_TRIPLET 0 VERSION_MAJOR)
list(GET VERSION_TRIPLET 1 VERSION_MINOR)
list(GET VERSION_TRIPLET 2 VERSION_REVISION)
function(pad_number NUMBER OUTPUT_LEN)
    string(LENGTH "${${NUMBER}}" INPUT_LEN)
    if(INPUT_LEN LESS OUTPUT_LEN)
        math(EXPR ZEROES "${OUTPUT_LEN} - ${INPUT_LEN} - 1")
        set(NUM ${${NUMBER}})
        foreach(C RANGE ${ZEROES})
            set(NUM "0${NUM}")
        endforeach()
        set(${NUMBER} ${NUM} PARENT_SCOPE)
    endif()
endfunction()
pad_number(VERSION_MINOR 3)
pad_number(VERSION_REVISION 3)
set(LIBJPEG_TURBO_VERSION_NUMBER ${VERSION_MAJOR}${VERSION_MINOR}${VERSION_REVISION})

string(TIMESTAMP DEFAULT_BUILD "%Y%m%d")
set(BUILD ${DEFAULT_BUILD} CACHE STRING "Build string (default: ${DEFAULT_BUILD})")

# NOTE: On Windows, this does nothing except when using MinGW or Cygwin.
# CMAKE_BUILD_TYPE has no meaning in Visual Studio, and it always defaults to
# Debug when using NMake.
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()
message(STATUS "CMAKE_BUILD_TYPE = ${CMAKE_BUILD_TYPE}")

message(STATUS "VERSION = ${VERSION}, BUILD = ${BUILD}")

# Detect CPU type and whether we're building 64-bit or 32-bit code
math(EXPR BITS "${CMAKE_SIZEOF_VOID_P} * 8")
string(TOLOWER ${CMAKE_SYSTEM_PROCESSOR} CMAKE_SYSTEM_PROCESSOR_LC)
if(CMAKE_SYSTEM_PROCESSOR_LC MATCHES "x86_64" OR
        CMAKE_SYSTEM_PROCESSOR_LC MATCHES "amd64" OR
        CMAKE_SYSTEM_PROCESSOR_LC MATCHES "i[0-9]86" OR
        CMAKE_SYSTEM_PROCESSOR_LC MATCHES "x86" OR
        CMAKE_SYSTEM_PROCESSOR_LC MATCHES "ia32")
    if(BITS EQUAL 64 OR CMAKE_C_COMPILER_ABI MATCHES "ELF X32")
        set(CPU_TYPE x86_64)
    else()
        set(CPU_TYPE i386)
    endif()
    if(NOT CMAKE_SYSTEM_PROCESSOR STREQUAL ${CPU_TYPE})
        set(CMAKE_SYSTEM_PROCESSOR ${CPU_TYPE})
    endif()
elseif(CMAKE_SYSTEM_PROCESSOR_LC STREQUAL "aarch64" OR
        CMAKE_SYSTEM_PROCESSOR_LC MATCHES "arm*")
    if(BITS EQUAL 64)
        set(CPU_TYPE arm64)
    else()
        set(CPU_TYPE arm)
    endif()
elseif(CMAKE_SYSTEM_PROCESSOR_LC MATCHES "ppc*" OR
        CMAKE_SYSTEM_PROCESSOR_LC MATCHES "powerpc*")
    set(CPU_TYPE powerpc)
else()
    set(CPU_TYPE ${CMAKE_SYSTEM_PROCESSOR_LC})
endif()
message(STATUS "${BITS}-bit build (${CPU_TYPE})")


###############################################################################
# INSTALL DIRECTORIES
###############################################################################

if(WIN32)
    set(CMAKE_INSTALL_DEFAULT_PREFIX "c:/${CMAKE_PROJECT_NAME}-gcc")
    if(BITS EQUAL 64)
        set(CMAKE_INSTALL_DEFAULT_PREFIX "${CMAKE_INSTALL_DEFAULT_PREFIX}64")
    endif()
else()
    if(NOT CMAKE_INSTALL_DEFAULT_PREFIX)
        set(CMAKE_INSTALL_DEFAULT_PREFIX /opt/${CMAKE_PROJECT_NAME})
    endif()
endif()
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "${CMAKE_INSTALL_DEFAULT_PREFIX}" CACHE PATH
            "Directory into which to install ${CMAKE_PROJECT_NAME} (default: ${CMAKE_INSTALL_DEFAULT_PREFIX})"
            FORCE)
endif()
message(STATUS "CMAKE_INSTALL_PREFIX = ${CMAKE_INSTALL_PREFIX}")

# When the prefix is /opt/${CMAKE_PROJECT_NAME}, we assume that an "official"
# build is being created, and thus we install things into specific locations.

if(CMAKE_INSTALL_PREFIX STREQUAL "${CMAKE_INSTALL_DEFAULT_PREFIX}")
    set(CMAKE_INSTALL_DEFAULT_DATAROOTDIR "")
    set(CMAKE_INSTALL_DEFAULT_DOCDIR "<CMAKE_INSTALL_DATAROOTDIR>/doc")
    set(CMAKE_INSTALL_DEFAULT_JAVADIR "<CMAKE_INSTALL_DATAROOTDIR>/classes")
    if(BITS EQUAL 64)
        set(CMAKE_INSTALL_DEFAULT_LIBDIR "lib64")
    elseif(CMAKE_C_COMPILER_ABI MATCHES "ELF X32")
        set(CMAKE_INSTALL_DEFAULT_LIBDIR "libx32")
    else()
        set(CMAKE_INSTALL_DEFAULT_LIBDIR "lib32")
    endif()
endif()

macro(boolean_number var)
    if(${var})
        set(${var} 1 ${ARGN})
    else()
        set(${var} 0 ${ARGN})
    endif()
endmacro()

# Explicitly setting CMAKE_POSITION_INDEPENDENT_CODE=FALSE disables PIC for all
# targets, which will cause the shared library builds to fail.  Thus, if shared
# libraries are enabled and CMAKE_POSITION_INDEPENDENT_CODE is explicitly set
# to FALSE, we need to unset it, thus restoring the default behavior
# (automatically using PIC for shared library targets.)
if(DEFINED CMAKE_POSITION_INDEPENDENT_CODE AND
        NOT CMAKE_POSITION_INDEPENDENT_CODE)
    unset(CMAKE_POSITION_INDEPENDENT_CODE CACHE)
endif()

set(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_FULL_LIBDIR})

set(BITS_IN_JSAMPLE 8)
set(D_ARITH_CODING_SUPPORTED 1)
set(C_ARITH_CODING_SUPPORTED 1)

set(MEM_SRCDST_SUPPORTED 1)
set(MEM_SRCDST_FUNCTIONS "global:  jpeg_mem_dest;  jpeg_mem_src;")

set(SO_AGE 2)
set(SO_AGE 3)

set(JPEG_LIB_VERSION 62)

math(EXPR JPEG_LIB_VERSION_DIV10 "${JPEG_LIB_VERSION} / 10")
math(EXPR JPEG_LIB_VERSION_MOD10 "${JPEG_LIB_VERSION} % 10")
if(JPEG_LIB_VERSION STREQUAL "62")
    set(DEFAULT_SO_MAJOR_VERSION ${JPEG_LIB_VERSION})
else()
    set(DEFAULT_SO_MAJOR_VERSION ${JPEG_LIB_VERSION_DIV10})
endif()
if(JPEG_LIB_VERSION STREQUAL "80")
    set(DEFAULT_SO_MINOR_VERSION 2)
else()
    set(DEFAULT_SO_MINOR_VERSION 0)
endif()

if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
    # Use the maximum optimization level for release builds
    foreach(var CMAKE_C_FLAGS_RELEASE CMAKE_C_FLAGS_RELWITHDEBINFO)
        if(${var} MATCHES "-O2")
            string(REGEX REPLACE "-O2" "-O3" ${var} "${${var}}")
        endif()
    endforeach()
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
    if(CMAKE_C_COMPILER_ID MATCHES "SunPro")
        # Use the maximum optimization level for release builds
        foreach(var CMAKE_C_FLAGS_RELEASE CMAKE_C_FLAGS_RELWITHDEBINFO)
            if(${var} MATCHES "-xO3")
                string(REGEX REPLACE "-xO3" "-xO5" ${var} "${${var}}")
            endif()
            if(${var} MATCHES "-xO2")
                string(REGEX REPLACE "-xO2" "-xO5" ${var} "${${var}}")
            endif()
        endforeach()
    endif()
endif()

string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UC)

set(EFFECTIVE_C_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
message(STATUS "Compiler flags = ${EFFECTIVE_C_FLAGS}")

set(EFFECTIVE_LD_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${CMAKE_EXE_LINKER_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
message(STATUS "Linker flags = ${EFFECTIVE_LD_FLAGS}")

include(CheckCSourceCompiles)
include(CheckIncludeFiles)
include(CheckTypeSize)

check_type_size("size_t" SIZE_T)
check_type_size("unsigned long" UNSIGNED_LONG)

if(SIZE_T EQUAL UNSIGNED_LONG)
    check_c_source_compiles("int main(int argc, char **argv) { unsigned long a = argc;  return __builtin_ctzl(a); }"
            HAVE_BUILTIN_CTZL)
endif()

# Check for headers
check_include_files(locale.h HAVE_LOCALE_H)
check_include_files(stddef.h HAVE_STDDEF_H)
check_include_files(stdlib.h HAVE_STDLIB_H)
check_include_files(sys/types.h NEED_SYS_TYPES_H)

# Check for functions
include(CheckSymbolExists)
check_symbol_exists(memset string.h HAVE_MEMSET)
check_symbol_exists(memcpy string.h HAVE_MEMCPY)
if(NOT HAVE_MEMSET AND NOT HAVE_MEMCPY)
    set(NEED_BSD_STRINGS 1)
endif()

# Check for types
check_type_size("unsigned char" UNSIGNED_CHAR)
check_type_size("unsigned short" UNSIGNED_SHORT)

# Check for compiler features
check_c_source_compiles("int main(void) { typedef struct undefined_structure *undef_struct_ptr;  undef_struct_ptr ptr = 0;  return ptr != 0; }"
        INCOMPLETE_TYPES)
if(INCOMPLETE_TYPES)
    message(STATUS "Compiler supports pointers to undefined structures.")
else()
    set(INCOMPLETE_TYPES_BROKEN 1)
    message(STATUS "Compiler does not support pointers to undefined structures.")
endif()

if(CMAKE_CROSSCOMPILING)
    set(RIGHT_SHIFT_IS_UNSIGNED 0)
else()
    include(CheckCSourceRuns)
    check_c_source_runs("
  #include <stdio.h>
  #include <stdlib.h>
  int is_shifting_signed (long arg) {
    long res = arg >> 4;
    if (res == -0x7F7E80CL)
      return 1; /* right shift is signed */
    /* see if unsigned-shift hack will fix it. */
    /* we can't just test exact value since it depends on width of long... */
    res |= (~0L) << (32-4);
    if (res == -0x7F7E80CL)
      return 0; /* right shift is unsigned */
    printf(\"Right shift isn't acting as I expect it to.\\\\n\");
    printf(\"I fear the JPEG software will not work at all.\\\\n\\\\n\");
    return 0; /* try it with unsigned anyway */
  }
  int main (void) {
    exit(is_shifting_signed(-0x7F7E80B1L));
  }" RIGHT_SHIFT_IS_UNSIGNED)
endif()

set(INLINE_OPTIONS "__inline__;inline")
list(INSERT INLINE_OPTIONS 0 "inline __attribute__((always_inline))")
list(INSERT INLINE_OPTIONS 0 "__inline__ __attribute__((always_inline))")
foreach(inline ${INLINE_OPTIONS})
    check_c_source_compiles("${inline} static int foo(void) { return 0; } int main(void) { return foo(); }"
            INLINE_WORKS)
    if(INLINE_WORKS)
        set(INLINE ${inline})
        break()
    endif()
endforeach()
if(NOT INLINE_WORKS)
    message(FATAL_ERROR "Could not determine how to inline functions.")
endif()
message(STATUS "INLINE = ${INLINE} (FORCE_INLINE = ${FORCE_INLINE})")

set(THREAD_LOCAL "__thread")

# Include directories and compiler definitions
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_SOURCE_DIR})


###############################################################################
# TARGETS
###############################################################################

if(CMAKE_EXECUTABLE_SUFFIX_TMP)
    set(CMAKE_EXECUTABLE_SUFFIX ${CMAKE_EXECUTABLE_SUFFIX_TMP})
endif()
message(STATUS "CMAKE_EXECUTABLE_SUFFIX = ${CMAKE_EXECUTABLE_SUFFIX}")

set(JPEG_SOURCES
        libjpeg-turbo/jcapimin.c
        libjpeg-turbo/jcapistd.c
        libjpeg-turbo/jccoefct.c
        libjpeg-turbo/jccolor.c
        libjpeg-turbo/jcdctmgr.c
        libjpeg-turbo/jchuff.c
        libjpeg-turbo/jcicc.c
        libjpeg-turbo/jcinit.c
        libjpeg-turbo/jcmainct.c
        libjpeg-turbo/jcmarker.c
        libjpeg-turbo/jcmaster.c
        libjpeg-turbo/jcomapi.c
        libjpeg-turbo/jcparam.c
        libjpeg-turbo/jcphuff.c
        libjpeg-turbo/jcprepct.c
        libjpeg-turbo/jcsample.c
        libjpeg-turbo/jctrans.c
        libjpeg-turbo/jdapimin.c
        libjpeg-turbo/jdapistd.c
        libjpeg-turbo/jdatadst.c
        libjpeg-turbo/jdatasrc.c
        libjpeg-turbo/jdcoefct.c
        libjpeg-turbo/jdcolor.c
        libjpeg-turbo/jddctmgr.c
        libjpeg-turbo/jdhuff.c
        libjpeg-turbo/jdicc.c
        libjpeg-turbo/jdinput.c
        libjpeg-turbo/jdmainct.c
        libjpeg-turbo/jdmarker.c
        libjpeg-turbo/jdmaster.c
        libjpeg-turbo/jdmerge.c
        libjpeg-turbo/jdphuff.c
        libjpeg-turbo/jdpostct.c
        libjpeg-turbo/jdsample.c
        libjpeg-turbo/jdtrans.c
        libjpeg-turbo/jerror.c
        libjpeg-turbo/jfdctflt.c
        libjpeg-turbo/jfdctfst.c
        libjpeg-turbo/jfdctint.c
        libjpeg-turbo/jidctflt.c
        libjpeg-turbo/jidctfst.c
        libjpeg-turbo/jidctint.c
        libjpeg-turbo/jidctred.c
        libjpeg-turbo/jquant1.c
        libjpeg-turbo/jquant2.c
        libjpeg-turbo/jutils.c
        libjpeg-turbo/jmemmgr.c
        libjpeg-turbo/jmemnobs.c
        libjpeg-turbo/jaricom.c
        libjpeg-turbo/jcarith.c
        libjpeg-turbo/jdarith.c)

add_subdirectory(libjpeg-turbo/simd)
if(NEON_INTRINSICS)
    add_definitions(-DNEON_INTRINSICS)
endif()

target_include_directories(simd PRIVATE ./include ./libjpeg-turbo)

add_library(jpeg SHARED ${JPEG_SOURCES} $<TARGET_OBJECTS:simd> ${SIMD_OBJS})
target_include_directories(jpeg PRIVATE ./include ./libjpeg-turbo)

set(TURBOJPEG_SOURCES
        ${JPEG_SOURCES}
        $<TARGET_OBJECTS:simd>
        ${SIMD_OBJS}
        libjpeg-turbo/turbojpeg.c
        libjpeg-turbo/transupp.c
        libjpeg-turbo/jdatadst-tj.c
        libjpeg-turbo/jdatasrc-tj.c
        libjpeg-turbo/rdbmp.c
        libjpeg-turbo/rdppm.c
        libjpeg-turbo/wrbmp.c
        libjpeg-turbo/wrppm.c)
add_library(turbojpeg SHARED ${TURBOJPEG_SOURCES})
target_include_directories(turbojpeg PRIVATE ./include ./libjpeg-turbo)
set_property(TARGET turbojpeg PROPERTY COMPILE_FLAGS "-DBMP_SUPPORTED -DPPM_SUPPORTED")
