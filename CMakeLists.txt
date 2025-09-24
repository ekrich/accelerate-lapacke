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

set(CMAKE_MAXIMUM_SUPPORTED_VERSION 4.0)

cmake_minimum_required(VERSION 3.12...${CMAKE_MAXIMUM_SUPPORTED_VERSION})

project(AccelerateLAPACKE VERSION 1.6.1 LANGUAGES C Fortran)

include(FetchContent)

string(COMPARE EQUAL "${CMAKE_PROJECT_NAME}" "${PROJECT_NAME}" IS_TOP_LEVEL)

if (NOT IS_TOP_LEVEL)
    message(FATAL_ERROR "This project is meant to be built as a stand-alone project, only.")
endif ()

option(BUILD_INDEX64 "Link to ILP64 version of LAPACK?" OFF)

message(STATUS "Trying to link to ILP64 interface of LAPACK: ${BUILD_NEW_LAPACK}")

FetchContent_Declare(
    AccelerateLAPACK
    GIT_REPOSITORY "https://github.com/lepus2589/accelerate-lapack.git"
    GIT_TAG v1.6.0
    SYSTEM
    FIND_PACKAGE_ARGS 1.6.0 CONFIG NAMES AccelerateLAPACK
)

set(AccelerateLAPACK_INCLUDE_PACKAGING FALSE)

FetchContent_MakeAvailable(AccelerateLAPACK)

add_subdirectory(src)
