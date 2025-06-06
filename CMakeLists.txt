cmake_minimum_required(VERSION 3.15)

project(tdscpp VERSION 20250301)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
include(CheckIncludeFileCXX)

option(BUILD_SAMPLE        "Build sample program" ON)
option(WITH_OPENSSL        "Use OpenSSL for encryption" OFF)

if(NOT WIN32)
    option(ENABLE_KRB5         "Enable Kerberos support" ON)
endif()

find_package(nlohmann_json REQUIRED)

if(ENABLE_KRB5)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(KRB5 REQUIRED IMPORTED_TARGET mit-krb5-gssapi)
    set(HAVE_GSSAPI 1)
endif()

if(NOT WIN32)
    find_package(ICU COMPONENTS uc REQUIRED)
endif()

if(WITH_OPENSSL)
    find_package(OpenSSL REQUIRED)
endif()

if(WIN32)
    option(ONECORE "Use OneCore libraries" OFF)
endif()

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

CHECK_INCLUDE_FILE_CXX("format" HAS_STD_FORMAT)

if(NOT HAS_STD_FORMAT)
    find_package(fmt REQUIRED)
endif()

add_definitions(-DTDSCPP_EXPORT)

# ----------------------------------------

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

set(SRC_FILES src/tdscpp.cpp
    src/bcp.cpp
    src/value.cpp
    src/ssl.cpp
    src/ringbuf.cpp
    src/rpc.cpp
    src/batch.cpp
    src/collation.cpp
    src/ver80coll.cpp
    src/ver90coll.cpp
    src/ver100coll.cpp
    src/ver140coll.cpp
    src/tests.cpp)

add_library(tdscpp ${SRC_FILES})

set_property(TARGET tdscpp PROPERTY CMAKE_EXPORT_FIND_PACKAGE_NAME "tdscpp")

add_library(tdscpp::tdscpp ALIAS tdscpp)

if(WIN32)
	add_definitions(-DNOMINMAX)
endif()

target_link_libraries(tdscpp PUBLIC nlohmann_json::nlohmann_json)

if(WITH_OPENSSL)
    target_link_libraries(tdscpp PUBLIC OpenSSL::Crypto OpenSSL::SSL)
endif()

if(NOT WIN32)
    target_link_libraries(tdscpp PUBLIC ICU::uc)
endif()

target_compile_options(tdscpp PRIVATE
     $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:GNU>>:
          -Wall>
     $<$<CXX_COMPILER_ID:MSVC>:
          /W4>)

set_target_properties(tdscpp PROPERTIES PUBLIC_HEADER src/tdscpp.h)

if(WIN32)
    set_target_properties(tdscpp PROPERTIES PREFIX "")
endif()

if(NOT MSVC)
    target_link_options(tdscpp PUBLIC -static-libgcc -static-libstdc++)

    if(ONECORE)
        target_link_options(tdscpp PUBLIC -nodefaultlibs)
    endif()
endif()

if(NOT HAS_STD_FORMAT)
    target_link_libraries(tdscpp INTERFACE fmt::fmt-header-only)
endif()

if(NOT MSVC)
	target_compile_options(tdscpp PRIVATE -Wall -Werror=cast-function-type -Wno-expansion-to-defined -Wunused-parameter -Wtype-limits -Wextra -Wconversion -fstrict-aliasing)
endif()

target_include_directories(tdscpp PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if(WIN32)
    if(ONECORE)
        target_compile_options(tdscpp PRIVATE -Wl,-Bstatic -lstdc++ -Wl,-Bdynamic)
        target_link_libraries(tdscpp PUBLIC gcc mingwex mingw32 gcc_eh ucrt onecore_apiset)
    else()
        target_link_libraries(tdscpp PUBLIC ws2_32 secur32 crypt32)
    endif()
elseif(ENABLE_KRB5)
	target_link_libraries(tdscpp PUBLIC PkgConfig::KRB5)
endif()

install(TARGETS tdscpp
    EXPORT tdscpp-targets
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    PRIVATE_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

install(EXPORT tdscpp-targets
  NAMESPACE tdscpp::
  DESTINATION lib/cmake/tdscpp
)

#install(EXPORT tdscpp-targets DESTINATION lib/cmake/tdscpp)

if(MSVC AND BUILD_SHARED_LIBS)
    install(FILES $<TARGET_PDB_FILE:tdscpp> DESTINATION bin OPTIONAL)
endif()

configure_package_config_file(
    "tdscppConfig.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/tdscppConfig.cmake"
    INSTALL_DESTINATION "lib/cmake/tdscpp"
)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/tdscppConfig.cmake" DESTINATION "lib/cmake/tdscpp")

# ----------------------------------------

if(BUILD_SAMPLE)
    set(TEST_SRC_FILES src/tdscpp-test.cpp)

    add_executable(tdscpp-test ${TEST_SRC_FILES})
    target_link_libraries(tdscpp-test PUBLIC tdscpp)

    if(NOT HAS_STD_FORMAT)
        target_link_libraries(tdscpp-test INTERFACE fmt::fmt-header-only)
    endif()

    install(TARGETS tdscpp-test DESTINATION "${CMAKE_INSTALL_BINDIR}")

    add_executable(colltest src/colltest.cpp)
    target_link_libraries(colltest PUBLIC tdscpp)

    if(NOT HAS_STD_FORMAT)
        target_link_libraries(colltest INTERFACE fmt::fmt-header-only)
    endif()

    install(TARGETS colltest DESTINATION "${CMAKE_INSTALL_BINDIR}")

    if(MSVC)
        install(FILES $<TARGET_PDB_FILE:tdscpp-test> DESTINATION bin OPTIONAL)
        install(FILES $<TARGET_PDB_FILE:colltest> DESTINATION bin OPTIONAL)
    endif()
endif()

set(CPACK_PACKAGE_NAME tdscpp-dev)
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Mark Harmstone")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libicu-dev, nlohmann-json3-dev, libssl-dev, libkrb5-dev")
include(CPack)
