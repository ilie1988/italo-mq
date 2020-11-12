cmake_minimum_required(VERSION 3.7)

# Has to be set before `project()`, and ignored on non-macos:
set(CMAKE_OSX_DEPLOYMENT_TARGET 10.12 CACHE STRING "macOS deployment target (Apple clang only)")

project(libitalomq CXX C)

include(GNUInstallDirs)

set(ITALOMQ_VERSION_MAJOR 1)
set(ITALOMQ_VERSION_MINOR 2)
set(ITALOMQ_VERSION_PATCH 1)
set(ITALOMQ_VERSION "${ITALOMQ_VERSION_MAJOR}.${ITALOMQ_VERSION_MINOR}.${ITALOMQ_VERSION_PATCH}")
message(STATUS "italomq v${ITALOMQ_VERSION}")

set(ITALOMQ_LIBVERSION 0)


option(BUILD_SHARED_LIBS "Build shared libraries instead of static ones" ON)


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

configure_file(italomq/version.h.in italomq/version.h @ONLY)
configure_file(libitalomq.pc.in libitalomq.pc @ONLY)

add_library(italomq
    italomq/address.cpp
    italomq/auth.cpp
    italomq/bt_serialize.cpp
    italomq/connections.cpp
    italomq/jobs.cpp
    italomq/italomq.cpp
    italomq/proxy.cpp
    italomq/worker.cpp
)
set_target_properties(italomq PROPERTIES SOVERSION ${ITALOMQ_LIBVERSION})

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(italomq PRIVATE Threads::Threads)

# libzmq is nearly impossible to link statically from a system-installed static library: it depends
# on a ton of other libraries, some of which are not all statically available.  If the caller wants
# to mess with this, so be it: they can set up a libzmq target and we'll use it.  Otherwise if they
# asked us to do things statically, don't even try to find a system lib and just build it.
set(italomq_build_static_libzmq OFF)
if(TARGET libzmq)
    target_link_libraries(italomq PUBLIC libzmq)
elseif(BUILD_SHARED_LIBS)
    include(FindPkgConfig)
    pkg_check_modules(libzmq libzmq>=4.3 IMPORTED_TARGET)

    if(libzmq_FOUND)
        # Debian sid includes a -isystem in the mit-krb package that, starting with pkg-config 0.29.2,
        # breaks cmake's pkgconfig module because it stupidly thinks "-isystem" is a path, so if we find
        # -isystem in the include dirs then hack it out.
        get_property(zmq_inc TARGET PkgConfig::libzmq PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
        list(FIND zmq_inc "-isystem" broken_isystem)
        if(NOT broken_isystem EQUAL -1)
          list(REMOVE_AT zmq_inc ${broken_isystem})
          set_property(TARGET PkgConfig::libzmq PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${zmq_inc})
        endif()

        target_link_libraries(italomq PUBLIC PkgConfig::libzmq)
    else()
        set(italomq_build_static_libzmq ON)
    endif()
else()
    set(italomq_build_static_libzmq ON)
endif()

if(italomq_build_static_libzmq)
    message(STATUS "libzmq >= 4.3 not found or static build requested, building bundled version")
    list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/local-libzmq")
    include(LocalLibzmq)
    target_link_libraries(italomq PUBLIC libzmq_vendor)
endif()

target_include_directories(italomq
    PUBLIC
        $<INSTALL_INTERFACE:>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/cppzmq>
)

target_compile_options(italomq PRIVATE -Wall -Wextra -Werror)
set_target_properties(italomq PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
    POSITION_INDEPENDENT_CODE ON
)

function(link_dep_libs target linktype libdirs)
    foreach(lib ${ARGN})
        find_library(link_lib-${lib} NAMES ${lib} PATHS ${libdirs})
        message(STATUS "FIND ${lib} FOUND ${link_lib-${lib}}")
        if(link_lib-${lib})
            target_link_libraries(${target} ${linktype} ${link_lib-${lib}})
        endif()
    endforeach()
endfunction()

# If the caller has already set up a sodium target then we will just link to it, otherwise we go
# looking for it.
if(TARGET sodium)
    target_link_libraries(italomq PUBLIC sodium)
    if(italomq_build_static_libzmq)
        target_link_libraries(libzmq_vendor INTERFACE sodium)
    endif()
else()
    pkg_check_modules(sodium REQUIRED libsodium IMPORTED_TARGET)

    if(BUILD_SHARED_LIBS)
        target_link_libraries(italomq PUBLIC PkgConfig::sodium)
        if(italomq_build_static_libzmq)
            target_link_libraries(libzmq_vendor INTERFACE PkgConfig::sodium)
        endif()
    else()
        link_dep_libs(italomq PUBLIC "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
        target_include_directories(italomq PUBLIC ${sodium_STATIC_INCLUDE_DIRS})
        if(italomq_build_static_libzmq)
            link_dep_libs(libzmq_vendor INTERFACE "${sodium_STATIC_LIBRARY_DIRS}" ${sodium_STATIC_LIBRARIES})
            target_link_libraries(libzmq_vendor INTERFACE ${sodium_STATIC_INCLUDE_DIRS})
        endif()
    endif()
endif()

add_library(italomq::italomq ALIAS italomq)

export(
    TARGETS italomq
    NAMESPACE italomq::
    FILE italomqTargets.cmake
)
install(
    TARGETS italomq
    EXPORT italomqConfig
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(
    FILES italomq/address.h
          italomq/auth.h
          italomq/base32z.h
          italomq/base64.h
          italomq/batch.h
          italomq/bt_serialize.h
          italomq/bt_value.h
          italomq/connections.h
          italomq/hex.h
          italomq/italomq.h
          italomq/message.h
          italomq/string_view.h
          italomq/variant.h
          ${CMAKE_CURRENT_BINARY_DIR}/italomq/version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/italomq
)

option(ITALOMQ_INSTALL_CPPZMQ "Install cppzmq header with italomq/ headers" ON)
if(ITALOMQ_INSTALL_CPPZMQ)
  install(
    FILES cppzmq/zmq.hpp
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/italomq
  )
endif()


install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/libitalomq.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
  set(italomq_IS_TOPLEVEL_PROJECT TRUE)
else()
  set(italomq_IS_TOPLEVEL_PROJECT FALSE)
endif()

option(ITALOMQ_BUILD_TESTS "Building and perform italomq tests" ${italomq_IS_TOPLEVEL_PROJECT})
if(ITALOMQ_BUILD_TESTS)
    add_subdirectory(tests)
endif()

