cmake_minimum_required(VERSION 2.8.12) # ubuntu 14 version

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

if(WIN32)
    set(VCPKG_DEPS protobuf libwebsockets)
else()
    set(VCPKG_DEPS protobuf)
endif()

include(Set-Toolchain-vcpkg)

project(BitcoinArmory C CXX)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "release build with debug info" FORCE)
endif()

if(POLICY CMP0077)
    cmake_policy(SET CMP0077 NEW)
endif()

include(ArmorySupport)
include(PrettyCompilerColors)

use_cxx11()

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# use ccache if available, and not already enabled on the command line
# but not with ninja and msys ccache on msys2
if(NOT (WIN32 AND (NOT $ENV{MSYSTEM} STREQUAL "") AND CMAKE_GENERATOR STREQUAL Ninja))
    if(NOT CMAKE_CXX_COMPILER_LAUNCHER AND NOT CMAKE_C_COMPILER_LAUNCHER)
        find_program(CCACHE_FOUND ccache)
        if(CCACHE_FOUND)
            set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
            set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK    ccache)
        endif(CCACHE_FOUND)
    endif()
endif()

# build for host CPU if desired (default yes)
option(WITH_HOST_CPU_FEATURES "support the CPU features of the build host, gcc only" ON)

if(WITH_HOST_CPU_FEATURES AND CMAKE_CXX_COMPILER_ID STREQUAL GNU)
    check_x86_cpu_features()

    add_compile_options(-march=native ${X86_CPU_FEATURES_COMPILER_FLAGS})
endif()

if(MSVC)
    add_compile_definitions(NOMINMAX _WINSOCKAPI_)
endif()

# we are bundling libwebsockets for the time being, at least until ubuntu
# updates their version

# this is the right way to set options for sub-projects
# we allow these to be overridden by the user
if(NOT DEFINED LWS_SSL_CLIENT_USE_OS_CA_CERTS)
    set(LWS_SSL_CLIENT_USE_OS_CA_CERTS OFF CACHE BOOL "disable using OS CA certs for libwebsockets" FORCE)
endif()

if(NOT DEFINED LWS_WITH_SHARED)
    set(LWS_WITH_SHARED OFF CACHE BOOL "disable building libwebsockets as shared library" FORCE)
endif()

if(NOT DEFINED LWS_WITH_SSL)
    set(LWS_WITH_SSL OFF CACHE BOOL "disable SSL support in libwebsockets" FORCE)
endif()

if(NOT WIN32)
    add_subdirectory(libwebsockets EXCLUDE_FROM_ALL)
    set(LIBWEBSOCKETS_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/libwebsockets/include ${CMAKE_BINARY_DIR}/libwebsockets/include CACHE PATH "libwebsockets include dirs" FORCE)
endif()

# this is the main project
add_subdirectory(cppForSwig)
