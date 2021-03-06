#
#   Copyright (c) 2012 250bpm s.r.o.  All rights reserved.
#   Copyright (c) 2013 GoPivotal, Inc.  All rights reserved.
#
#   Permission is hereby granted, free of charge, to any person obtaining a copy
#   of this software and associated documentation files (the "Software"),
#   to deal in the Software without restriction, including without limitation
#   the rights to use, copy, modify, merge, publish, distribute, sublicense,
#   and/or sell copies of the Software, and to permit persons to whom
#   the Software is furnished to do so, subject to the following conditions:
#
#   The above copyright notice and this permission notice shall be included
#   in all copies or substantial portions of the Software.
#
#   THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#   IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#   FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
#   THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#   LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
#   FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
#   IN THE SOFTWARE.
#

cmake_minimum_required (VERSION 2.8)
include (CheckIncludeFiles)
include (CheckSymbolExists)
include (CheckCSourceCompiles)

project (nanomsg)
enable_testing ()

#  Platform checks.

if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set (NN_HAVE_LINUX 1)
    add_definitions (-DNN_HAVE_LINUX)
endif ()

if (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set (NN_HAVE_WINDOWS 1)
    add_definitions (-DNN_HAVE_WINDOWS)
endif ()

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set (NN_HAVE_OSX 1)
    add_definitions (-DNN_HAVE_OSX)
endif ()

if (${CMAKE_SYSTEM_NAME} MATCHES "FreeBSD")
    set (NN_HAVE_FREEBSD 1)
    add_definitions (-DNN_HAVE_FREEBSD)
endif ()

if (${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
    set (NN_HAVE_NETBSD 1)
    add_definitions (-DNN_HAVE_NETBSD)
endif ()

if (${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD")
    set (NN_HAVE_OPENBSD 1)
    add_definitions (-DNN_HAVE_OPENBSD)
endif ()

if (${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
    set (NN_HAVE_SOLARIS 1)
    add_definitions (-DNN_HAVE_SOLARIS)
    list (APPEND CMAKE_REQUIRED_LIBRARIES socket)
    list (APPEND CMAKE_REQUIRED_LIBRARIES nsl)
endif ()

#  Feature checks.

find_package (Threads)

check_include_files (sys/eventfd.h NN_HAVE_EVENTFD)
if (NN_HAVE_EVENTFD)
    add_definitions (-DNN_HAVE_EVENTFD)
endif()

check_symbol_exists (pipe "unistd.h" NN_HAVE_PIPE)
if (NN_HAVE_PIPE)
    add_definitions (-DNN_HAVE_PIPE)
endif()

list (APPEND CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
check_symbol_exists (pipe2 "unistd.h" NN_HAVE_PIPE2)
list (REMOVE_ITEM CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
if (NN_HAVE_PIPE2)
    add_definitions (-DNN_HAVE_PIPE2)
    add_definitions (-D_GNU_SOURCE)
endif()

check_symbol_exists (socketpair "sys/types.h;sys/socket.h" NN_HAVE_SOCKETPAIR)
if (NN_HAVE_SOCKETPAIR)
    add_definitions (-DNN_HAVE_SOCKETPAIR)
endif()

check_symbol_exists (CLOCK_MONOTONIC time.h NN_HAVE_CLOCK_MONOTONIC)
if (NN_HAVE_CLOCK_MONOTONIC)
    add_definitions (-DNN_HAVE_CLOCK_MONOTONIC)
endif()

check_symbol_exists (gethrtime time.h NN_HAVE_GETHRTIME)
if (NN_HAVE_GETHRTIME)
    add_definitions (-DNN_HAVE_GETHRTIME)
endif ()

check_include_files (poll.h NN_HAVE_POLL)
if (NN_HAVE_POLL)
    add_definitions (-DNN_HAVE_POLL)
endif()

check_include_files (sys/epoll.h NN_HAVE_EPOLL)
if (NN_HAVE_EPOLL)
    add_definitions (-DNN_HAVE_EPOLL)
endif ()

check_symbol_exists (kqueue "sys/types.h;sys/event.h;sys/time.h" NN_HAVE_KQUEUE)
if (NN_HAVE_KQUEUE)
    add_definitions (-DNN_HAVE_KQUEUE)
endif ()

check_symbol_exists (getifaddrs "sys/types.h;ifaddrs.h" NN_HAVE_IFADDRS)
if (NN_HAVE_IFADDRS)
    add_definitions (-DNN_HAVE_IFADDRS)
endif ()

check_symbol_exists (SIOCGIFADDR sys/ioctl.h NN_HAVE_SIOCGIFADDR)
if (NN_HAVE_SIOCGIFADDR)
    add_definitions (-DNN_HAVE_SIOCGIFADDR)
endif ()

list (APPEND CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
check_symbol_exists (accept4 sys/socket.h NN_HAVE_ACCEPT4)
list (REMOVE_ITEM CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
if (NN_HAVE_ACCEPT4)
    add_definitions (-DNN_HAVE_ACCEPT4)
    add_definitions (-D_GNU_SOURCE)
endif ()

list (APPEND CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
list (APPEND CMAKE_REQUIRED_LIBRARIES anl)
check_symbol_exists (getaddrinfo_a netdb.h NN_HAVE_GETADDRINFO_A)
list (REMOVE_ITEM CMAKE_REQUIRED_LIBRARIES anl)
list (REMOVE_ITEM CMAKE_REQUIRED_DEFINITIONS -D_GNU_SOURCE)
if (NN_HAVE_GETADDRINFO_A)
    add_definitions (-DNN_HAVE_GETADDRINFO_A)
    add_definitions (-D_GNU_SOURCE)
endif ()

list (APPEND CMAKE_REQUIRED_LIBRARIES rt pthread)
check_symbol_exists (sem_wait semaphore.h NN_HAVE_SEMAPHORE)
list (REMOVE_ITEM CMAKE_REQUIRED_LIBRARIES rt pthread)
if (NN_HAVE_SEMAPHORE)
    add_definitions (-DNN_HAVE_SEMAPHORE)
endif()

check_c_source_compiles ("
    #include <stdint.h>
    int main()
    {
        volatile uint32_t n = 0;
        __sync_fetch_and_add (&n, 1);
        __sync_fetch_and_sub (&n, 1);
        return 0;
    }
    " NN_HAVE_GCC_ATOMIC_BUILTINS)
if (NN_HAVE_GCC_ATOMIC_BUILTINS)
    add_definitions (-DNN_HAVE_GCC_ATOMIC_BUILTINS)
endif ()

#  Decide which features to actually use.

if (NN_HAVE_EPOLL)
    message ("-- Using epoll for socket monitoring")
    add_definitions (-DNN_USE_EPOLL)
elseif (NN_HAVE_KQUEUE)
    message ("-- Using kqueue for socket monitoring")
    add_definitions (-DNN_USE_KQUEUE)
elseif (NN_HAVE_POLL)
    message ("-- Using poll for socket monitoring")
    add_definitions (-DNN_USE_POLL)
endif ()

if (NN_HAVE_EVENTFD)
    message ("-- Using eventfd for signaling")
    add_definitions (-DNN_USE_EVENTFD)
elseif (NN_HAVE_PIPE)
    message ("-- Using pipe for signaling")
    add_definitions (-DNN_USE_PIPE)
elseif (NN_HAVE_SOCKETPAIR)
    message ("-- Using socketpair for signaling")
    add_definitions (-DNN_USE_SOCKETPAIR)
endif ()

if (NN_HAVE_IFADDRS)
    message ("-- Using getifaddrs for NIC name resolution")
    add_definitions (-DNN_USE_IFADDRS)
elseif (NN_HAVE_SIOCGIFADDR)
    message ("-- Using SIOCGIFADDR for NIC name resolution")
    add_definitions (-DNN_USE_SIOCGIFADDR)
else ()
    message ("-- No NIC name resolution mechanism used")
    add_definitions (-DNN_USE_LITERAL_IFADDR)
endif ()

#  Optional debugging/profiling tools to switch on.

option (ALLOC_MONITOR "Add memory allocation monitoring" OFF)
if (ALLOC_MONITOR)
    add_definitions(-DNN_ALLOC_MONITOR)
endif ()

#  Version.

file (READ "${PROJECT_SOURCE_DIR}/src/nn.h" NN_H_DATA)
string (REGEX REPLACE ".*#define NN_VERSION_CURRENT ([0-9]+).*" "\\1" NN_VERSION_CURRENT "${NN_H_DATA}")
string (REGEX REPLACE ".*#define NN_VERSION_REVISION ([0-9]+).*" "\\1" NN_VERSION_REVISION "${NN_H_DATA}")
string (REGEX REPLACE ".*#define NN_VERSION_AGE ([0-9]+).*" "\\1" NN_VERSION_AGE "${NN_H_DATA}")
set (NN_VERSION_STR "${NN_VERSION_CURRENT}.${NN_VERSION_REVISION}.${NN_VERSION_AGE}")

#  Description

set (NN_DESCRIPTION "nanomsg library is a high-performance implementation of several \"scalability protocols\". Scalability protocol's job is to define how multiple applications communicate to form a single distributed application")

#  Build the library itself.

add_subdirectory (src)

#  Build unit tests.

set (all_tests "")

macro (add_libnanomsg_test NAME)
    list (APPEND all_tests ${NAME})
    add_executable (${NAME} tests/${NAME}.c)
    target_link_libraries (${NAME} nanomsg)
    add_test (${NAME} ${NAME})
endmacro (add_libnanomsg_test)

#  Transport tests.
add_libnanomsg_test (inproc)
add_libnanomsg_test (inproc_shutdown)
add_libnanomsg_test (ipc)
add_libnanomsg_test (ipc_shutdown)
add_libnanomsg_test (tcp)
add_libnanomsg_test (tcp_shutdown)

#  Protocol tests.
add_libnanomsg_test (pair)
add_libnanomsg_test (pubsub)
add_libnanomsg_test (reqrep)
add_libnanomsg_test (pipeline)
add_libnanomsg_test (survey)
add_libnanomsg_test (bus)

#  Feature tests.
add_libnanomsg_test (block)
add_libnanomsg_test (shutdown)
add_libnanomsg_test (timeo)
add_libnanomsg_test (iovec)
add_libnanomsg_test (msg)
add_libnanomsg_test (prio)
add_libnanomsg_test (poll)
add_libnanomsg_test (device)
add_libnanomsg_test (emfile)
add_libnanomsg_test (domain)
add_libnanomsg_test (trie)
add_libnanomsg_test (list)
add_libnanomsg_test (hash)
add_libnanomsg_test (symbol)
add_libnanomsg_test (separation)

#  The builtin "test" target cannot be modified to build the library and tests
#  or to show output when a test failes. Let's create a different target.
add_custom_target (check
                   COMMENT "Running tests..."
                   COMMAND ctest --force-new-ctest-process --output-on-failure
                   WORKING_DIRECTORY ${CMAKE_BINARY_DIR})

add_dependencies (check ${all_tests})

#  Build the performance tests.

macro (add_libnanomsg_perf NAME)
    add_executable (${NAME} perf/${NAME}.c)
    target_link_libraries (${NAME} nanomsg)
endmacro (add_libnanomsg_perf)

add_libnanomsg_perf (inproc_lat)
add_libnanomsg_perf (inproc_thr)
add_libnanomsg_perf (local_lat)
add_libnanomsg_perf (remote_lat)
add_libnanomsg_perf (local_thr)
add_libnanomsg_perf (remote_thr)

#  Installation.

install (FILES src/nn.h DESTINATION include/nanomsg)
install (FILES src/inproc.h DESTINATION include/nanomsg)
install (FILES src/ipc.h DESTINATION include/nanomsg)
install (FILES src/tcp.h DESTINATION include/nanomsg)
install (FILES src/pair.h DESTINATION include/nanomsg)
install (FILES src/pubsub.h DESTINATION include/nanomsg)
install (FILES src/reqrep.h DESTINATION include/nanomsg)
install (FILES src/pipeline.h DESTINATION include/nanomsg)
install (FILES src/survey.h DESTINATION include/nanomsg)
install (FILES src/bus.h DESTINATION include/nanomsg)

