cmake_minimum_required(VERSION 3.0)
project(OpenZWave VERSION  1.4.0 LANGUAGES C CXX)
# Get the current working branch
execute_process(
  COMMAND git rev-parse --abbrev-ref HEAD
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_BRANCH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)
# Get the latest abbreviated commit hash of the working branch
execute_process(
  COMMAND git log -1 --format=%h
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_COMMIT_HASH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)
set(OpenZWave_GITVERSION ${GIT_COMMIT_HASH})

include(CheckIncludeFiles)
if (${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    CHECK_INCLUDE_FILES(libudev.h HAVE_LIBUDEV_DEV)
    if(NOT HAVE_LIBUDEV_DEV)
        message(SEND_ERROR "Need libudev-dev installed on Linux")
    endif()
endif()


set(OPENZWAVE_INCLUDE_DIRS)

set(AES_BASE cpp/src/aes)
list(APPEND OPENZWAVE_INCLUDE_DIRS ${AES_BASE})
set(AES_SRCS
${AES_BASE}/aes_modes.c
${AES_BASE}/aescrypt.c
${AES_BASE}/aeskey.c
${AES_BASE}/aestab.c
)


set(COMMAND_CLASS_BASE cpp/src/command_classes)
list(APPEND OPENZWAVE_INCLUDE_DIRS ${COMMAND_CLASS_BASE})
#TODO: Don't glob this
file(GLOB COMMAND_CLASS_SRCS ${COMMAND_CLASS_BASE}/*.cpp)

set(VALUE_CLASS_BASE cpp/src/value_classes)
list(APPEND OPENZWAVE_INCLUDE_DIRS ${VALUE_CLASS_BASE})
#TODO: Don't glob this
file(GLOB VALUE_CLASS_SRCS ${VALUE_CLASS_BASE}/*.cpp)

set(TINY_XML_BASE cpp/tinyxml)
list(APPEND OPENZWAVE_INCLUDE_DIRS ${TINY_XML_BASE})
set(TINY_XML_SRCS
${TINY_XML_BASE}/tinystr.cpp
${TINY_XML_BASE}/tinyxml.cpp
${TINY_XML_BASE}/tinyxmlerror.cpp
${TINY_XML_BASE}/tinyxmlparser.cpp
)

set(INEP_BASE cpp/src)
list(APPEND OPENZWAVE_INCLUDE_DIRS ${INEP_BASE})
#TODO: Don't glob this
file(GLOB INEP_SRCS ${INEP_BASE}/*.cpp)

set(HID_SRCS)
if(UNIX)
    list(APPEND OPENZWAVE_INCLUDE_DIRS cpp/hidapi/hidapi)
    if(APPLE)
        set(HID_SRCS cpp/hidapi/mac/hid.c)
    elseif(${CMAKE_SYSTEM_NAME} STREQUAL "FreeBSD")
        set(HID_SRCS cpp/hidapi/libusb/hid.c)
    else()
        set(HID_SRCS cpp/hidapi/linux/hid.c)
    endif(APPLE)
else()
    message(STATUS "Not a unix system so not building HID")
endif(UNIX)

set(PLATFORM_BASE cpp/src/platform)
list(APPEND OPENZWAVE_INCLUDE_DIRS ${PLATFORM})
#TODO: Don't glob this
file(GLOB PLATFORM_SRCS ${PLATFORM_BASE}/*.cpp)
if(UNIX)
    message(STATUS "Adding unix platform sources")
    set(PLATFORM_UNIX_BASE cpp/src/platform/unix)
    list(APPEND OPENZWAVE_INCLUDE_DIRS ${PLATFORM_UNIX_BASE})
    file(GLOB PLATFORM_UNIX_SRCS ${PLATFORM_UNIX_BASE}/*.cpp)
    list(APPEND PLATFORM_SRCS ${PLATFORM_UNIX_SRCS})
else()
    message(STATUS "No platform specifiec sources")
endif(UNIX)

set(OPENZWAVE_SRCS
${AES_SRCS}
${COMMAND_CLASS_SRCS}
${VALUE_CLASS_SRCS}
${INEP_SRCS}
${HID_SRCS}
${PLATFORM_SRCS}
${TINY_XML_SRCS}
)

configure_file(cpp/version/vers.cpp ${CMAKE_CURRENT_BINARY_DIR}/vers.cpp)
list(APPEND OPENZWAVE_SRCS ${CMAKE_CURRENT_BINARY_DIR}/vers.cpp)

add_library(openzwave SHARED ${OPENZWAVE_SRCS})
add_library(openzwave-static STATIC ${OPENZWAVE_SRCS})

target_include_directories(openzwave PUBLIC ${OPENZWAVE_INCLUDE_DIRS})
target_include_directories(openzwave-static PUBLIC ${OPENZWAVE_INCLUDE_DIRS})


set(minozw_SRCS cpp/examples/MinOZW/Main.cpp)
add_executable(minozw ${minozw_SRCS})
target_link_libraries(minozw openzwave-static)
if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
    target_link_libraries(minozw udev pthread)
    target_compile_options(minozw PRIVATE -pthread)
endif()
