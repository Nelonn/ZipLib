cmake_minimum_required(VERSION 3.5)

project(ziplib LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_subdirectory(ThirdParty/bzip2)
add_subdirectory(ThirdParty/zlib)

file(GLOB_RECURSE SOURCES Source/**.cpp Source/**.h)

add_library(ziplib STATIC ${SOURCES})

target_include_directories(ziplib PUBLIC Source)

target_link_libraries(ziplib PUBLIC ziplib::external_bzip2 ziplib::external_zlib)

if (PROJECT_IS_TOP_LEVEL)
    add_executable(ziplib_sample Sample/Main.cpp)
    target_link_libraries(ziplib_sample PRIVATE ziplib)
endif ()
