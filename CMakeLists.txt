# required cmake version
cmake_minimum_required(VERSION 3.17)

# autodetect vcpkg CMAKE_TOOLCHAIN_FILE if VCPKG_ROOT is defined
# this needs to be configured before the project() directive
if((CMAKE_GENERATOR MATCHES "Ninja") AND DEFINED ENV{VCPKG_ROOT} AND NOT $ENV{VCPKG_ROOT} STREQUAL "" AND NOT DEFINED CMAKE_TOOLCHAIN_FILE)
    set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
endif((CMAKE_GENERATOR MATCHES "Ninja") AND DEFINED ENV{VCPKG_ROOT} AND NOT $ENV{VCPKG_ROOT} STREQUAL "" AND NOT DEFINED CMAKE_TOOLCHAIN_FILE)
set(BUILTIN_SOCKET ON CACHE BOOL "") # for static Python

# configure basic project information
project(osslsigncode
    VERSION 2.10
    DESCRIPTION "OpenSSL based Authenticode signing for PE, CAB, CAT, MSI, APPX and script files"
    HOMEPAGE_URL "https://github.com/mtrojnar/osslsigncode"
    LANGUAGES C)

# force nonstandard version format for development packages
set(DEV "-dev")
set(PROJECT_VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}${DEV}")

# version and contact information
set(PACKAGE_STRING "${PROJECT_NAME} ${PROJECT_VERSION}")
set(PACKAGE_BUGREPORT "Michal.Trojnara@stunnel.org")

# specify the C standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# load CMake library modules
include(FindOpenSSL)
if(OPENSSL_VERSION VERSION_LESS "1.1.1")
    message(FATAL_ERROR "OpenSSL version must be at least 1.1.1")
endif()
if(OPENSSL_VERSION VERSION_LESS "3.0.0")
    include(FindCURL)
endif(OPENSSL_VERSION VERSION_LESS "3.0.0")
include(FindZLIB)

# load CMake project modules
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake")
include(SetBashCompletion)
include(FindHeaders)

# define the target
add_executable(osslsigncode)

# add compiler/linker flags
include(SetCompilerFlags)

# create and use config.h
configure_file(Config.h.in config.h)
target_compile_definitions(osslsigncode PRIVATE HAVE_CONFIG_H=1)

# set sources
target_sources(osslsigncode PRIVATE osslsigncode.c helpers.c utf.c msi.c pe.c cab.c cat.c appx.c script.c)
if(NOT UNIX)
    target_sources(osslsigncode PRIVATE applink.c)
endif(NOT UNIX)

# set include directories
target_include_directories(osslsigncode PRIVATE "${PROJECT_BINARY_DIR}")

# set OpenSSL includes/libraries
if(NOT OPENSSL_FOUND)
    message(FATAL_ERROR "OpenSSL library not found")
endif(NOT OPENSSL_FOUND)
target_include_directories(osslsigncode PRIVATE ${OPENSSL_INCLUDE_DIR})
target_link_libraries(osslsigncode PRIVATE ${OPENSSL_LIBRARIES})

# set cURL includes/libraries
if(OPENSSL_VERSION VERSION_LESS "3.0.0")
if(CURL_FOUND)
    target_compile_definitions(osslsigncode PRIVATE ENABLE_CURL=1)
    target_include_directories(osslsigncode PRIVATE ${CURL_INCLUDE_DIRS})
    target_link_libraries(osslsigncode PRIVATE ${CURL_LIBRARIES})
    message(STATUS "cURL support enabled")
else(CURL_FOUND)
    message(STATUS "cURL support disabled (library not found)")
endif(CURL_FOUND)
endif(OPENSSL_VERSION VERSION_LESS "3.0.0")

if(NOT ZLIB_FOUND)
    message(FATAL_ERROR "Zlib library not found")
endif(NOT ZLIB_FOUND)
target_include_directories(osslsigncode PRIVATE ${ZLIB_INCLUDE_DIR})
target_link_libraries(osslsigncode PRIVATE ${ZLIB_LIBRARIES})

if(NOT UNIX)
# https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-shutdown
target_link_libraries(osslsigncode PRIVATE ws2_32.lib crypt32.lib)
endif(NOT UNIX)

# add paths to linker search and installed rpath
set_target_properties(osslsigncode PROPERTIES INSTALL_RPATH_USE_LINK_PATH TRUE)

# testing with CTest
include(CMakeTest)

# installation rules for a project
set(BINDIR "${CMAKE_INSTALL_PREFIX}/bin")
install(TARGETS osslsigncode RUNTIME DESTINATION ${BINDIR})
if(UNIX)
    include(CMakeDist)
else(UNIX)
    install(
        DIRECTORY ${PROJECT_BINARY_DIR}/ DESTINATION ${BINDIR}
        FILES_MATCHING
        PATTERN "*.dll"
        PATTERN "vcpkg_installed" EXCLUDE
        PATTERN "CMakeFiles" EXCLUDE
        PATTERN "Testing" EXCLUDE)
endif(UNIX)

#[[
Local Variables:
    c-basic-offset: 4
    tab-width: 4
    indent-tabs-mode: nil
End:
    vim: set ts=4 expandtab:
]]
