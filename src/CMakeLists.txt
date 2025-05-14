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

get_target_property(ACCELERATE_NEW_LAPACK_VERSION LAPACK::NEW_LAPACK VERSION)

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

set(BLAS_LIBRARIES_BACKUP "BLAS::BLAS")
set(LAPACK_LIBRARIES_BACKUP "LAPACK::LAPACK")

# Set the BLAS and LAPACK libraries to the new 32bit targets. This helps the
# Reference LAPACK CMake script to pass the CheckFortranFunctionExists tests for
# `dgemm` and `dgeqrt`.
set(BLAS_LIBRARIES "BLAS::NEW_BLAS_IMPORTED")
set(LAPACK_LIBRARIES "LAPACK::NEW_LAPACK_IMPORTED")

FetchContent_MakeAvailable(reference-lapack)

# Restore BLAS and LAPACK libraries to what they were before (32bit or 64bit)
set(BLAS_LIBRARIES "${BLAS_LIBRARIES_BACKUP}")
set(LAPACK_LIBRARIES "${LAPACK_LIBRARIES_BACKUP}")

if (CMAKE_VERSION VERSION_GREATER_EQUAL 4)
    unset(ENV{CMAKE_POLICY_VERSION_MINIMUM})
endif ()

if (BUILD_INDEX64)
    # Replace the LINK_LIBRARIES of the lapacke64 target with the ILP64 ones.
    # The lapacke64 target was created linking to the NEW_LAPACK_IMPORTED
    # target, which we had to add to satisfy the Check... tests, that the
    # Reference LAPACK build script does.
    set_target_properties(lapacke64 PROPERTIES LINK_LIBRARIES "${LAPACK_LIBRARIES}")
else ()
    # Replace the LINK_LIBRARIES of the lapacke target with the NEWLAPACK ones.
    # The lapacke target was created linking to the NEW_LAPACK_IMPORTED target,
    # which we had to add to satisfy the Check... tests, that the Reference
    # LAPACK build script does.
    set_target_properties(lapacke PROPERTIES LINK_LIBRARIES "${LAPACK_LIBRARIES}")
endif ()
