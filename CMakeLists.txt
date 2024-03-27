cmake_minimum_required(VERSION 3.2)

project(ziplib LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

file(GLOB_RECURSE SOURCES Source/ZipLib/**.cpp Source/ZipLib/**.h)

add_library(ziplib STATIC ${SOURCES})
target_include_directories(ziplib INTERFACE Source/ZipLib)
