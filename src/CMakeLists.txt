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

FetchContent_Declare(
    reference-lapack
    GIT_REPOSITORY "https://github.com/Reference-LAPACK/lapack.git"
    GIT_TAG "v${ACCELERATE_LAPACK_ILAVER_VERSION}"
)

# Set configuration options for Reference LAPACK
set(BUILD_SHARED_LIBS ON CACHE BOOL "")
set(CBLAS OFF CACHE BOOL "")
set(LAPACKE ON CACHE BOOL "")

if (BLA_SIZEOF_INTEGER EQUAL 8)
    set(BUILD_INDEX64 ON CACHE BOOL "")
else ()
    set(BUILD_INDEX64 OFF CACHE BOOL "")
endif ()

# Inject the 32 bit BLAS and LAPACK link flags into the
# CheckFortranFunctionExists macro to help the Reference LAPACK CMake script to
# pass the tests for `dgemm` and `dgeqrt`.
set(CMAKE_REQUIRED_LINK_OPTIONS ${LAPACK32_LINKER_FLAGS})

# CMake deprecated all CMake versions prior to v3.5. The relevant Reference
# LAPACK versions still use v3.2, which leads to an error, unless we
# override this.
if (CMAKE_VERSION VERSION_GREATER_EQUAL 4 AND ACCELERATE_LAPACK_ILAVER_VERSION VERSION_LESS 3.12.0)
    # This will create a CMAKE_POLICY_VERSION_MINIMUM cache variable when
    # including ReferenceLAPACK, but this does not propagate into the
    # try_compile()'s:
    set(CMAKE_POLICY_VERSION_MINIMUM 3.5)
    # This CMAKE_POLICY_VERSION_MINIMUM environment variable will propagate into
    # ReferenceLAPACK's try_compile()'s, but won't create the
    # CMAKE_POLICY_VERSION_MINIMUM cache variable on its own:
    set(ENV{CMAKE_POLICY_VERSION_MINIMUM} 3.5)
endif ()

set(BLAS_LINKER_FLAGS_BACKUP "${BLAS_LINKER_FLAGS}")
set(LAPACK_LINKER_FLAGS_BACKUP "${LAPACK_LINKER_FLAGS}")

# Temporarily clear the BLAS and LAPACK link flags to prevent the Reference
# LAPACK CMake script to add them to the CMAKE_EXE_LINKER_FLAGS variable, which
# would break the following try_compile()'s.
set(BLAS_LINKER_FLAGS "")
set(LAPACK_LINKER_FLAGS "")

FetchContent_MakeAvailable(reference-lapack)

# Restore BLAS and LAPACK link flags to what they were before (32bit or 64bit)
set(BLAS_LINKER_FLAGS "${BLAS_LINKER_FLAGS_BACKUP}")
set(LAPACK_LINKER_FLAGS "${LAPACK_LINKER_FLAGS_BACKUP}")

if (CMAKE_VERSION VERSION_GREATER_EQUAL 4)
    unset(CMAKE_POLICY_VERSION_MINIMUM)
    unset(ENV{CMAKE_POLICY_VERSION_MINIMUM})
endif ()

if (BUILD_INDEX64)
    if (TARGET "blas64")
        message(FATAL_ERROR "LAPACKE failed to use BLAS provided by Accelerate and included Netlib's Reference BLAS implementation into the build.")
    endif ()

    if (TARGET "lapack64")
        message(FATAL_ERROR "LAPACKE failed to use LAPACK provided by Accelerate and included Netlib's Reference LAPACK implementation into the build.")
    endif ()

    # Set the LINK_OPTIONS of the lapacke64 target to the NEWLAPACK ILP64 ones.
    # The lapacke64 target was created with empty LAPACK_LINKER_FLAGS, which we
    # had to do to satisfy the Check... tests, that the Reference LAPACK build
    # script does.
    target_link_options(lapacke64 PRIVATE ${LAPACK64_LINKER_FLAGS})
else ()
    if (TARGET "blas")
        message(FATAL_ERROR "LAPACKE failed to use BLAS provided by Accelerate and included Netlib's Reference BLAS implementation into the build.")
    endif ()

    if (TARGET "lapack")
        message(FATAL_ERROR "LAPACKE failed to use LAPACK provided by Accelerate and included Netlib's Reference LAPACK implementation into the build.")
    endif ()

    # Set the LINK_OPTIONS of the lapacke target to the NEWLAPACK ones. The
    # lapacke target was created with empty LAPACK_LINKER_FLAGS, which we had to
    # do to satisfy the Check... tests, that the Reference LAPACK build script
    # does.
    target_link_options(lapacke PRIVATE ${LAPACK32_LINKER_FLAGS})
endif ()
