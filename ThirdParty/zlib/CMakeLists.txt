option(ZIPLIB_BUNDLED_ZLIB "Use bundled zlib" ON)

if (ZIPLIB_BUNDLED_ZLIB)
    message(STATUS "ZipLib: Using bundled zlib, use option ZIPLIB_BUNDLED_ZLIB")
    cmake_policy(PUSH)
    set(CMAKE_POLICY_DEFAULT_CMP0077 NEW) # cmake_policy(SET CMP0077 NEW) - not working
    set(ZLIB_BUILD_EXAMPLES OFF)
    add_subdirectory(zlib-1.3.1)
    cmake_policy(POP)
    add_library(ziplib::external_zlib ALIAS zlibstatic)
elseif (TARGET zlibstatic)
    add_library(ziplib::external_zlib ALIAS zlibstatic)
elseif (TARGET zlib)
    add_library(ziplib::external_zlib ALIAS zlib)
else ()
    find_package(ZLIB REQUIRED)
    add_library(ziplib::external_zlib ALIAS ZLIB::ZLIB)
endif ()
