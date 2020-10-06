cmake_minimum_required(VERSION 3.7)
project(libcsp C)

# Compiler requirements
set(CMAKE_C_STANDARD_REQUIRED 99)

# Version
set(libcsp_VERSION_MAJOR 1) 
set(libcsp_VERSION_MINOR 6) 
set(libcsp_VERSION_PATCH 0) 
set(libcsp_VERSION ${libcsp_VERSION_MAJOR}.${libcsp_VERSION_MINOR}.${libcsp_VERSION_PATCH})

set(valid_os no-os;posix;windows;freertos;macosx)
set(valid_loglevel error;warn;info;debug)
set(valid_rtable static;cidr)

# CSP Options
set(includes "" CACHE STRING "Add additional include paths, separate with comma")

option(install-csp              "Installs CSP headers and lib"          true)
option(disable-output           "Disable CSP output"                    true)
option(disable-stlib            "Build objects only"                    true)
option(enable-shlib             "Build shared library"                  true)
option(enable-rdp               "Enable RDP support"                    true)
option(enable-rdp-fast-close    "Enable fast close of RDP connections"  true)
option(enable-qos               "Enable Quality of Service support"     true)
option(enable-promisc           "Enable promiscuous support"            true)
option(enable-crc32             "Enable CRC32 support"                  true)
option(enable-hmac              "Enable HMAC-SHA1 support"              true)
option(enable-xtea              "Enable XTEA support"                   true)
option(enable-python3-bindings  "Enable Python3 bindings"               true)
option(enable-examples          "Enable examples"                       true)
option(enable-dedup             "Enable packet deduplicator"            true)
option(enable-external-debug    "Enable external debug API"             true)
option(enable-debug-timestamp   "Enable timestamps on debug/log"        true)

# Drivers and interfaces (requires external dependencies)
option(enable-if-zmqhub         "Enable ZMQ interface"                  false)
option(enable-can-socketcan     "Enable Linux socketcan driver"         false)
set(with-driver-usart "None" CACHE STRING "Build USART driver. [windows, linux, None]")

# Macro for creating a cache variable accepting enum-like values
macro(enum_option name help options)
    list(GET ${options} 0 defaultValue)
    set(${name} ${defaultValue} CACHE STRING "${help}")
    set_property(CACHE ${name} PROPERTY STRINGS ${${options}})
    list(FIND ${options} "${${name}}" index)
    if (index EQUAL -1)
        message(FATAL_ERROR "The value option ${name} must be [${${options}}]")
    endif()
endmacro()

# Operating system
enum_option(with-os "Set operating system. Must be one of: ${valid_os}" valid_os)

# Log level
enum_option(with-loglevel "Set minimum compile time log level. Must be one of: ${valid_loglevel}" valid_loglevel)

# Routing Table
enum_option(with-rtable "Set routing table type: ${valid_rtable}" valid_rtable)

###################   Build   ###################

# Get GIT revision at build time
add_custom_target(GetGITRev COMMAND "git describe --long --always" $<TARGET_FILE:csp> )

if (with-rtable)
    message(STATUS "rtable: ${with-rtable}")
else()
    message(FATAL_ERROR "variable with-rtable not set")
endif()

file(GLOB FILES_CSP
    "src/*.c"
    "src/external/**/*.c"
    "src/transport/**/*.c"
    "src/crypto/**/*.c"
    "src/interfaces/**/*.c"
    "src/arch/*.c"
    "src/arch/${with_os}/*.c"
    "src/rtable/csp_rtable.c"
    "src/rtable/csp_rtable_${with-rtable}.c"
)

# TODO Add socketcan
# TODO Add uart
# TODO Add zmq
# TODO Add examples

add_library(csp STATIC ${FILES_CSP})

target_include_directories(csp PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}/include
)

target_compile_options(csp PUBLIC
        -Wall -Wextra -Wshadow -Wcast-align
        -Wwrite-strings -Wno-unused-parameter -Werror)

if(${with-os} STREQUAL "posix")
    message(STATUS "Configuring project for posix")
    target_link_libraries(csp PUBLIC rt pthread util)
    set(CSP_POSIX 1)
endif()

if(${with-os} STREQUAL "macosx")
    message(STATUS "Configuring project for macosx")
    target_link_libraries(csp PUBLIC pthread)
    set(CSP_MACOSX 1)
endif()

if(${with-os} STREQUAL "windows")
    message(STATUS "Configuring project for windows")
    target_compile_options(csp PUBLIC -D_WIN32_WINNT=0x0600)
    set(CSP_WINDOWS 1)
endif()

if(${with-os} STREQUAL "freertos")
    message(STATUS "Configuring project for freertos")
    set(CSP_FREERTOS 1)
endif()

if(NOT ${disable-output})
    set(CSP_DEBUG 1)
endif()

if(${enable_debug_timestamp})
    set(CSP_DEBUG_TIMESTAMP 1)
endif()

if(${enable_rdp})
    set(CSP_USE_RDP 1)
endif()

# Configure logger
message(STATUS "Loglevel: ${with-loglevel}")
if(${with-loglevel} STREQUAL "debug")
    set(CSP_LOG_LEVEL_DEBUG 1)
    set(CSP_LOG_LEVEL_INFO 1)
    set(CSP_LOG_LEVEL_WARN 1)
    set(CSP_LOG_LEVEL_ERROR 1)
elseif(${with-loglevel} STREQUAL "info")
    set(CSP_LOG_LEVEL_INFO 1)
    set(CSP_LOG_LEVEL_WARN 1)
    set(CSP_LOG_LEVEL_ERROR 1)
elseif(${with-loglevel} STREQUAL "warn")
    set(CSP_LOG_LEVEL_WARN 1)
    set(CSP_LOG_LEVEL_ERROR 1)
elseif(${with-loglevel} STREQUAL "error")
    set(CSP_LOG_LEVEL_ERROR 1)
endif()

# Test big or little endianness
include (TestBigEndian)
TEST_BIG_ENDIAN(IS_BIG_ENDIAN)
if(IS_BIG_ENDIAN)
    set(CSP_BIG_ENDIAN 1)
else()
    set(CSP_LITTLE_ENDIAN 1)
endif()

# Generate the configuration file
configure_file(csp_autoconfig.h.in ${CMAKE_CURRENT_BINARY_DIR}/include/csp/csp_autoconfig.h @ONLY)
target_sources(csp PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/include/csp/csp_autoconfig.h)

#

