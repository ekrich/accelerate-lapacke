#[[
MIT License

CMake build script for the Accelerate LAPACKE project
Copyright (c) 2025 Tim Kaune

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
]]

include("../cmake/ParseTBDMacros.cmake")

# Paths to text-based .dylib stubs
set(BLAS_TBD_PATH "${LAPACK_Accelerate_LIBRARY}/Versions/A/Frameworks/vecLib.framework/Versions/A/libBLAS.tbd")
set(LAPACK_TBD_PATH "${LAPACK_Accelerate_LIBRARY}/Versions/A/Frameworks/vecLib.framework/Versions/A/libLAPACK.tbd")

parse_tbd_symbols(BLAS "${BLAS_TBD_PATH}")
filter_new_lapack_symbols(BLAS)
filter_new_lapack_ilp64_symbols(BLAS)

parse_tbd_symbols(LAPACK "${LAPACK_TBD_PATH}")
filter_new_lapack_symbols(LAPACK)
filter_new_lapack_ilp64_symbols(LAPACK)

build_aliases(BLAS_NEWLAPACK_SYMBOLS)
build_aliases(BLAS_NEWLAPACK_ILP64_SYMBOLS)
build_aliases(LAPACK_NEWLAPACK_SYMBOLS)
build_aliases(LAPACK_NEWLAPACK_ILP64_SYMBOLS)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/new-blas.alias.in"
    "${CMAKE_CURRENT_BINARY_DIR}/new-blas.alias"
    @ONLY
)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/new-lapack.alias.in"
    "${CMAKE_CURRENT_BINARY_DIR}/new-lapack.alias"
    @ONLY
)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/new-blas-ilp64.alias.in"
    "${CMAKE_CURRENT_BINARY_DIR}/new-blas-ilp64.alias"
    @ONLY
)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/new-lapack-ilp64.alias.in"
    "${CMAKE_CURRENT_BINARY_DIR}/new-lapack-ilp64.alias"
    @ONLY
)

add_library(NEW_BLAS INTERFACE IMPORTED GLOBAL)
target_link_libraries(NEW_BLAS INTERFACE ${BLAS_LIBRARIES})

add_library(NEW_LAPACK INTERFACE IMPORTED GLOBAL)
target_link_libraries(NEW_LAPACK INTERFACE ${LAPACK_LIBRARIES} NEW_BLAS)

# Add the $NEWLAPACK symbols to the linker flags of the NEW_BLAS and NEW_LAPACK
# targets
target_link_options(
    NEW_BLAS
    INTERFACE
    ${BLAS_LINKER_FLAGS}
    "LINKER:-alias_list,${CMAKE_CURRENT_BINARY_DIR}/new-blas.alias"
)
target_link_options(
    NEW_LAPACK
    INTERFACE
    ${LAPACK_LINKER_FLAGS}
    "LINKER:-alias_list,${CMAKE_CURRENT_BINARY_DIR}/new-lapack.alias"
)

add_library(NEW_BLAS64 INTERFACE IMPORTED GLOBAL)
target_link_libraries(NEW_BLAS64 INTERFACE ${BLAS_LIBRARIES})

add_library(NEW_LAPACK64 INTERFACE IMPORTED GLOBAL)
target_link_libraries(NEW_LAPACK64 INTERFACE ${LAPACK_LIBRARIES} NEW_BLAS64)

# Add the $NEWLAPACK$ILP64 symbols to the linker flags of the NEW_BLAS64 and
# NEW_LAPACK64 targets
target_link_options(
    NEW_BLAS64
    INTERFACE
    ${BLAS_LINKER_FLAGS}
    "LINKER:-alias_list,${CMAKE_CURRENT_BINARY_DIR}/new-blas-ilp64.alias"
)
target_link_options(
    NEW_LAPACK64
    INTERFACE
    ${LAPACK_LINKER_FLAGS}
    "LINKER:-alias_list,${CMAKE_CURRENT_BINARY_DIR}/new-lapack-ilp64.alias"
)

if (BUILD_INDEX64)
    # Add the $NEWLAPACK$ILP64 symbols to the BLAS and LAPACK libraries.
    set(BLAS_LIBRARIES "NEW_BLAS64")
    set(LAPACK_LIBRARIES "NEW_LAPACK64")
    set(CHECK_LAPACK_VERSION_SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/lapack64_version.c")
else ()
    # Add the $NEWLAPACK symbols to the linker flags of the lapacke target
    set(BLAS_LIBRARIES "NEW_BLAS")
    set(LAPACK_LIBRARIES "NEW_LAPACK")
    set(CHECK_LAPACK_VERSION_SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/lapack_version.c")
endif ()

set_target_properties(BLAS::BLAS PROPERTIES INTERFACE_LINK_LIBRARIES "${BLAS_LIBRARIES}")
set_target_properties(LAPACK::LAPACK PROPERTIES INTERFACE_LINK_LIBRARIES "${LAPACK_LIBRARIES}")

try_run(
    HAS_RUN_ILAVER HAS_ILAVER
    SOURCES ${CHECK_LAPACK_VERSION_SOURCE}
    LINK_LIBRARIES ${LAPACK_LIBRARIES}
    RUN_OUTPUT_VARIABLE ACCELERATE_LAPACK_ILAVER_VERSION
)

message(STATUS "The BLAS/LAPACK version provided by the Accelerate framework is confirmed to be: ${ACCELERATE_LAPACK_ILAVER_VERSION}")

FetchContent_Declare(
    reference-lapack
    GIT_REPOSITORY "https://github.com/Reference-LAPACK/lapack.git"
    GIT_TAG "v${ACCELERATE_NEW_LAPACK_VERSION}"
)

# Set configuration options for Reference LAPACK
set(BUILD_SHARED_LIBS ON CACHE BOOL "")
set(CBLAS OFF CACHE BOOL "")
set(LAPACKE ON CACHE BOOL "")

if (CMAKE_VERSION VERSION_GREATER_EQUAL 4)
    # CMake deprecated all CMake versions prior to v3.5. The relevant Reference
    # LAPACK versions still use v3.2, which leads to an error, unless we
    # override this:
    set(ENV{CMAKE_POLICY_VERSION_MINIMUM} 3.5)
endif ()

set(BLAS_LIBRARIES_BACKUP "${BLAS_LIBRARIES}")
set(LAPACK_LIBRARIES_BACKUP "${LAPACK_LIBRARIES}")

# Set the BLAS and LAPACK libraries to the new 32bit targets. This helps the
# Reference LAPACK CMake script to pass the CheckFortranFunctionExists tests for
# `dgemm` and `dgeqrt`.
set(BLAS_LIBRARIES "NEW_BLAS")
set(LAPACK_LIBRARIES "NEW_LAPACK")

FetchContent_MakeAvailable(reference-lapack)

# Restore BLAS and LAPACK libraries to what they were before (32bit or 64bit)
set(BLAS_LIBRARIES "${BLAS_LIBRARIES_BACKUP}")
set(LAPACK_LIBRARIES "${LAPACK_LIBRARIES_BACKUP}")

if (CMAKE_VERSION VERSION_GREATER_EQUAL 4)
    unset(ENV{CMAKE_POLICY_VERSION_MINIMUM})
endif ()

if (BUILD_INDEX64)
    # Replace the LINK_LIBRARIES of the lapacke64 target with the ILP64 ones.
    # The lapacke64 target was created linking to the NEW_LAPACK target, which
    # we had to add to satisfy the Check... tests, that the Reference LAPACK
    # build script does.
    set_target_properties(lapacke64 PROPERTIES LINK_LIBRARIES "${LAPACK_LIBRARIES}")
endif ()
