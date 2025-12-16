cmake_minimum_required(VERSION 3.0.2)
project(dji_psdk_vendor)

add_definitions(-D_GNU_SOURCE)

## Compile as C++11, supported in ROS Kinetic and newer
# add_compile_options(-std=c++11)

## Find catkin macros and libraries
## if COMPONENTS list like find_package(catkin REQUIRED COMPONENTS xyz)
## is used, also find other catkin packages
find_package(catkin REQUIRED)
find_package(PkgConfig REQUIRED)

set(PSDK_PLATFORM "" CACHE STRING "Target platform: raspberry_pi or nvidia_jetson")
if (NOT PSDK_PLATFORM AND DEFINED ENV{CI})
    set(PSDK_PLATFORM "raspberry_pi")
    set(PSDK_PLATFORM_DIR "raspberry_pi")
    message(STATUS "CI detected: defaulting PSDK_PLATFORM to raspberry_pi")
elseif (NOT PSDK_PLATFORM)
    message(FATAL_ERROR "PSDK_PLATFORM is not set. Please set it to your target platform (e.g., nvidia_jetson, raspberry_pi).")
elseif(PSDK_PLATFORM STREQUAL "nvidia_jetson")
    set(PSDK_PLATFORM_DIR "nvidia_jetson")
elseif(PSDK_PLATFORM STREQUAL "raspberry_pi")
    set(PSDK_PLATFORM_DIR "raspberry_pi")
    message(STATUS "C++ PSDK platform not available on Raspberry Pi; psdk_vendor_cpp will not be built")
else()
    message(FATAL_ERROR "Unsupported PSDK_PLATFORM: ${PSDK_PLATFORM}. Supported platforms are 'nvidia_jetson' and 'raspberry_pi'.")
endif()

# C source directories
set(PSDK_CORE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_core/psdk_lib/include")
set(PSDK_C_MODULES_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_platform/C/module_sample")
set(PSDK_C_COMMON_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_platform/C/platform/common")
set(PSDK_C_HAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_platform/C/platform/${PSDK_PLATFORM_DIR}/hal")
set(PSDK_C_APP_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_platform/C/platform/${PSDK_PLATFORM_DIR}/application")
# C source files
file(GLOB_RECURSE PSDK_C_MODULES_SOURCES
    "${PSDK_C_MODULES_DIR}/utils/*.c"
)
file(GLOB_RECURSE PSDK_C_COMMON_SOURCES
    "${PSDK_C_COMMON_DIR}/*.c"
)
file(GLOB_RECURSE PSDK_C_HAL_SOURCES
    "${PSDK_C_HAL_DIR}/*.c"
)
file(GLOB_RECURSE PSDK_C_APP_SOURCES
    "${PSDK_C_APP_DIR}/*.c"
)

# C++ source directories
set(PSDK_CPP_MODULES_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_platform/C++/module_sample")
set(PSDK_CPP_COMMON_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_platform/C++/platform/common")
set(PSDK_CPP_HAL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_platform/C++/platform/${PSDK_PLATFORM_DIR}/hal")
set(PSDK_CPP_APP_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_platform/C++/platform/${PSDK_PLATFORM_DIR}/application")
# C++ source files
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    file(GLOB_RECURSE PSDK_CPP_MODULES_SOURCES
        "${PSDK_CPP_MODULES_DIR}/utils/*.c"
        "${PSDK_CPP_MODULES_DIR}/utils/*.cpp"
    )
    file(GLOB_RECURSE PSDK_CPP_COMMON_SOURCES
        "${PSDK_CPP_COMMON_DIR}/*.c"
        "${PSDK_CPP_COMMON_DIR}/*.cpp"
    )
    file(GLOB_RECURSE PSDK_CPP_HAL_SOURCES
        "${PSDK_CPP_HAL_DIR}/*.c"
        "${PSDK_CPP_HAL_DIR}/*.cpp"
    )
    file(GLOB_RECURSE PSDK_CPP_APP_SOURCES
        "${PSDK_CPP_APP_DIR}/*.c"
        "${PSDK_CPP_APP_DIR}/*.cpp"
    )
endif()

# Create static libraries
add_library(psdk_vendor_c STATIC
    ${PSDK_C_MODULES_SOURCES}
    ${PSDK_C_COMMON_SOURCES}
    ${PSDK_C_HAL_SOURCES}
    ${PSDK_C_APP_SOURCES}
)
# add_dependencies(psdk_vendor_c ${catkin_EXPORTED_TARGETS})
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    add_library(psdk_vendor_cpp STATIC
        ${PSDK_CPP_MODULES_SOURCES}
        ${PSDK_CPP_COMMON_SOURCES}
        ${PSDK_CPP_HAL_SOURCES}
        ${PSDK_CPP_APP_SOURCES}
    )
    # add_dependencies(psdk_vendor_cpp ${catkin_EXPORTED_TARGETS})
endif()

# Include directories
target_include_directories(psdk_vendor_c
    PRIVATE
        ${PSDK_C_MODULES_DIR}
        ${PSDK_C_COMMON_DIR}
        ${PSDK_C_HAL_DIR}
        ${PSDK_C_APP_DIR}
    PUBLIC
        ${PSDK_CORE_DIR}
)
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    target_include_directories(psdk_vendor_cpp
        PRIVATE
            ${PSDK_CPP_MODULES_DIR}
            ${PSDK_CPP_COMMON_DIR}
            ${PSDK_CPP_HAL_DIR}
            ${PSDK_CPP_APP_DIR}
        PUBLIC
            ${PSDK_CORE_DIR}
    )
endif()

# Imported libraries
if (CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
    set(PSDK_CORE_LIB_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_core/psdk_lib/lib/aarch64-linux-gnu-gcc")
elseif (CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
    set(PSDK_CORE_LIB_DIR "${CMAKE_CURRENT_SOURCE_DIR}/psdk_core/psdk_lib/lib/x86_64-linux-gnu-gcc")
else()
    message(FATAL_ERROR "Unsupported architecture: ${CMAKE_SYSTEM_PROCESSOR}")
endif()

if (NOT EXISTS ${PSDK_CORE_LIB_DIR}/libpayloadsdk.a)
    message(FATAL_ERROR
        "DJI Payload SDK core library not found at: ${PSDK_CORE_LIB_DIR}/libpayloadsdk.a\n"
    )
endif()

add_library(payloadsdk STATIC IMPORTED)
set_target_properties(payloadsdk PROPERTIES
    IMPORTED_LOCATION "${PSDK_CORE_LIB_DIR}/libpayloadsdk.a"
)
target_link_libraries(psdk_vendor_c
    PRIVATE
        payloadsdk
)
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    target_link_libraries(psdk_vendor_cpp
        PRIVATE
            payloadsdk
    )
endif()

# Link system libraries
target_link_libraries(psdk_vendor_c
    PRIVATE
        pthread
        m
        usb-1.0
)
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    target_link_libraries(psdk_vendor_cpp
        PRIVATE
            pthread
            m
            usb-1.0
    )
endif()

pkg_check_modules(FFMPEG REQUIRED
    libavcodec
    libavformat
    libavutil
    libswscale
)

# find_package(FFMPEG REQUIRED)
message(STATUS "Found FFMPEG installed in the system")
message(STATUS " - Includes: ${FFMPEG_INCLUDE_DIRS}")
message(STATUS " - Libraries: ${FFMPEG_LIBRARIES}")

execute_process(
    COMMAND ffmpeg -version
    OUTPUT_VARIABLE FFMPEG_VERSION_RAW
)

# Extract major version, allowing for 'n4.x.x' or '4.x.x'
string(REGEX MATCH "ffmpeg version[ \t]+n?([0-9]+)" _ "${FFMPEG_VERSION_RAW}")

if (NOT CMAKE_MATCH_1)
    message(FATAL_ERROR
        "Failed to parse ffmpeg version from:\n${FFMPEG_VERSION_RAW}"
    )
endif()

set(FFMPEG_MAJOR_VERSION "${CMAKE_MATCH_1}")

if (NOT FFMPEG_MAJOR_VERSION STREQUAL "4")
    message(FATAL_ERROR
        "Unsupported FFMPEG version: ${FFMPEG_MAJOR_VERSION}.x — DJI requires 4.x.x"
    )
else()
    message(STATUS " - FFMPEG version: ${FFMPEG_MAJOR_VERSION}.x")
endif()

target_link_libraries(psdk_vendor_c PRIVATE ${FFMPEG_LIBRARIES})
target_include_directories(psdk_vendor_c PRIVATE ${FFMPEG_INCLUDE_DIRS})
target_compile_definitions(psdk_vendor_c PRIVATE FFMPEG_INSTALLED)
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    target_link_libraries(psdk_vendor_cpp PRIVATE ${FFMPEG_LIBRARIES})
    target_include_directories(psdk_vendor_cpp PRIVATE ${FFMPEG_INCLUDE_DIR})
    target_compile_definitions(psdk_vendor_cpp PRIVATE FFMPEG_INSTALLED)
endif ()

pkg_check_modules(OPUS REQUIRED opus)
message(STATUS "Found OPUS installed in the system")
message(STATUS " - Includes: ${OPUS_INCLUDE_DIRS}")
message(STATUS " - Libraries: ${OPUS_LIBRARY}")

target_link_libraries(psdk_vendor_c PRIVATE ${OPUS_LIBRARY})
target_include_directories(psdk_vendor_c PRIVATE ${OPUS_INCLUDE_DIRS})
target_compile_definitions(psdk_vendor_c PRIVATE OPUS_INSTALLED)
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    target_link_libraries(psdk_vendor_cpp PRIVATE ${OPUS_LIBRARY})
    target_include_directories(psdk_vendor_cpp PRIVATE ${OPUS_INCLUDE_DIRS})
    target_compile_definitions(psdk_vendor_cpp PRIVATE OPUS_INSTALLED)
endif ()

pkg_check_modules(LIBUSB REQUIRED libusb-1.0)
message(STATUS "Found LIBUSB installed in the system")
message(STATUS " - Includes: ${LIBUSB_INCLUDE_DIRS}")
message(STATUS " - Libraries: ${LIBUSB_LIBRARY}")

target_link_libraries(psdk_vendor_c PRIVATE ${LIBUSB_LIBRARY})
target_include_directories(psdk_vendor_c PRIVATE ${LIBUSB_INCLUDE_DIRS})
target_compile_definitions(psdk_vendor_c PRIVATE LIBUSB_INSTALLED)
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    target_link_libraries(psdk_vendor_cpp PRIVATE ${LIBUSB_LIBRARY})
    target_include_directories(psdk_vendor_cpp PRIVATE ${LIBUSB_INCLUDE_DIR})   
    target_compile_definitions(psdk_vendor_cpp PRIVATE LIBUSB_INSTALLED)
endif ()

# Catkin package
if (PSDK_PLATFORM STREQUAL "raspberry_pi")
    catkin_package(
        INCLUDE_DIRS psdk_core/psdk_lib/include
        LIBRARIES psdk_vendor_c
        # EXPORTED_TARGETS psdk_vendor_c
    )
endif()
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    catkin_package(
        INCLUDE_DIRS psdk_core/psdk_lib/include
        LIBRARIES 
            psdk_vendor_c 
            psdk_vendor_cpp
    )
endif()

# Install static libraries
if (PSDK_PLATFORM STREQUAL "nvidia_jetson")
    install(TARGETS
        psdk_vendor_c
        psdk_vendor_cpp
        ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
    )
else()
    install(TARGETS
        psdk_vendor_c
        ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
    )
endif()

# Install include files
install(DIRECTORY
    psdk_core/psdk_lib/include/
    DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
)