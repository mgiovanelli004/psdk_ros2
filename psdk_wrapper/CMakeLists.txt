cmake_minimum_required(VERSION 3.14)
project(psdk_wrapper)

SET(BUILD_SHARED_LIBS ON)

# Default to C99
if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

# Default to C++17
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()


if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
add_definitions(-DSYSTEM_ARCH_LINUX)
# The PSDK OSAL uses GNU extensions (e.g. pthread_setname_np). Without this
# define, recent GCC versions (>= 14) fail with an implicit declaration error.
# DJI's own Manifold 3 samples define it as well.
add_definitions(-D_GNU_SOURCE)

# ---------------------------------------------------------------------------
# Payload SDK version and target platform
# ---------------------------------------------------------------------------
# PSDK_VERSION: git tag of https://github.com/dji-sdk/Payload-SDK to fetch.
#   Matrice 4E/4T need >= 3.10.0, Matrice 4D/4TD >= 3.11.0, Matrice 400 and
#   Manifold 3 >= 3.12.0. To build offline (e.g. inside the Manifold 3
#   container), pass -DFETCHCONTENT_SOURCE_DIR_PAYLOAD_SDK=/path/to/Payload-SDK
set(PSDK_VERSION "3.18.0" CACHE STRING "Payload-SDK git tag to fetch")

# PSDK_PLATFORM selects the hardware abstraction layer (HAL):
#   manifold2 (default): generic Linux HAL (UART + USB bulk / network), as
#                        used with M300/M350/M30 and third-party boards.
#   manifold3          : DJI Manifold 3 HAL (USB bulk device mode over
#                        FunctionFS, no UART). Required for M4/M4D/M400.
set(PSDK_PLATFORM "manifold2" CACHE STRING "PSDK HAL platform: manifold2 | manifold3")
set_property(CACHE PSDK_PLATFORM PROPERTY STRINGS manifold2 manifold3)
if(NOT PSDK_PLATFORM MATCHES "^(manifold2|manifold3)$")
  message(FATAL_ERROR "Unsupported PSDK_PLATFORM '${PSDK_PLATFORM}'. Use manifold2 or manifold3.")
endif()
message(STATUS "PSDK_VERSION: ${PSDK_VERSION}")
message(STATUS "PSDK_PLATFORM: ${PSDK_PLATFORM}")
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_lifecycle REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(std_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(psdk_interfaces REQUIRED)
find_package(std_srvs REQUIRED)
find_package(nlohmann_json REQUIRED)

# FetchContent module
include(FetchContent)

# Declare the content to fetch
FetchContent_Declare(
  Payload_SDK
  GIT_REPOSITORY https://github.com/dji-sdk/Payload-SDK.git
  GIT_TAG ${PSDK_VERSION}
  GIT_SHALLOW TRUE
)

FetchContent_GetProperties(Payload_SDK)
if(NOT Payload_SDK_POPULATED)
  FetchContent_Populate(Payload_SDK)
  message(STATUS "Payload_SDK_SOURCE_DIR: ${payload_sdk_SOURCE_DIR}")
endif()

# Set the path to the fetched content
set(PSDK_PATH ${payload_sdk_SOURCE_DIR})

message(STATUS "PSDK_PATH: ${PSDK_PATH}")

# Add path to FindFFMPEG.cmake and FindLIBUSB.cmake files
set(CMAKE_MODULE_PATH ${PSDK_PATH}/samples/sample_c++/platform/linux/common/3rdparty)

find_package(FFMPEG REQUIRED)
find_package(LIBUSB REQUIRED)

set(DJI_OSAL_PATH ${PSDK_PATH}/samples/sample_c++/platform/linux/common/osal)
# The generic UART and network HALs always come from manifold2. On Manifold 3
# only the USB bulk HAL changes (FunctionFS endpoints bulk2/bulk6/bulk3).
set(DJI_HAL_PATH ${PSDK_PATH}/samples/sample_c++/platform/linux/manifold2/hal)
if(PSDK_PLATFORM STREQUAL "manifold3")
  set(DJI_USB_BULK_HAL_PATH ${PSDK_PATH}/samples/sample_c++/platform/linux/manifold3/hal)
  if(NOT EXISTS ${DJI_USB_BULK_HAL_PATH}/hal_usb_bulk.c)
    message(FATAL_ERROR "Manifold 3 HAL not found in PSDK ${PSDK_VERSION}. Use PSDK >= 3.12.0.")
  endif()
  add_definitions(-DPSDK_PLATFORM_MANIFOLD3=1)
else()
  set(DJI_USB_BULK_HAL_PATH ${DJI_HAL_PATH})
endif()
set(DJI_STREAMING_PATH ${PSDK_PATH}/samples/sample_c++/module_sample/liveview)
set(DJI_UTILS_PATH ${PSDK_PATH}/samples/sample_c/module_sample)
set(DJI_HMS_RETURN_CODES_PATH ${PSDK_PATH}/samples/sample_c/module_sample/hms/data)

# Find toolchain name needed to select the right PSDK lib
execute_process(COMMAND uname -m
  OUTPUT_VARIABLE DEVICE_SYSTEM_ID)

if(DEVICE_SYSTEM_ID MATCHES x86_64)
  set(TOOLCHAIN_NAME x86_64-linux-gnu-gcc)
  add_definitions(-DPLATFORM_ARCH_x86_64=1)
elseif(DEVICE_SYSTEM_ID MATCHES aarch64)
  set(TOOLCHAIN_NAME aarch64-linux-gnu-gcc)
  add_definitions(-DPLATFORM_ARCH_aarch64=1)
else()
  message(FATAL_ERROR "FATAL: Please confirm your platform.")
endif()

# Check if PSDK lib is found
set(PSDK_LIB_PATH ${PSDK_PATH}/psdk_lib/lib/${TOOLCHAIN_NAME})
find_library(PSDK_LIB NAMES libpayloadsdk.a PATHS ${PSDK_LIB_PATH})
if (PSDK_LIB)
  message(STATUS "PSDK library found: ${PSDK_LIB}")
else()
  message(FATAL_ERROR "PSDK library NOT found")
endif()

if (FFMPEG_FOUND)
    message(STATUS "Found FFMPEG installed in the system")
    message(STATUS " - Includes: ${FFMPEG_INCLUDE_DIR}")
    message(STATUS " - Libraries: ${FFMPEG_LIBRARIES}")

    EXECUTE_PROCESS(COMMAND ffmpeg -version
            OUTPUT_VARIABLE ffmpeg_version_psdk_libput
            OUTPUT_STRIP_TRAILING_WHITESPACE
            )
    string(REGEX MATCH "version.*Copyright" ffmpeg_version_line "${ffmpeg_version_psdk_libput}")
    string(REGEX MATCH " .* " ffmpeg_version "${ffmpeg_version_line}")
    # The PSDK liveview decoder (dji_camera_stream_decoder.cpp) uses APIs that
    # were removed in FFmpeg 5 (avcodec_decode_video2, avcodec_register_all...).
    # Only FFmpeg 4.x works: Ubuntu 20.04 (4.2) and 22.04 (4.4) are fine,
    # Ubuntu 24.04 (6.1) is not. The original check only rejected 5.x.
    string(REGEX MATCH "^ *n?([0-9]+)" _ffmpeg_major_match "${ffmpeg_version}")
    set(ffmpeg_major_version "${CMAKE_MATCH_1}")

    if (ffmpeg_major_version STREQUAL "" OR ffmpeg_major_version LESS 5)
        message(STATUS " - Version: ${ffmpeg_version}")
    else ()
        message(FATAL_ERROR " - Not supported FFMPEG version: ${ffmpeg_version}, please install 4.x.x instead.")
    endif ()
    add_definitions(-DFFMPEG_INSTALLED)
else ()
    message(STATUS "Cannot Find FFMPEG")
endif (FFMPEG_FOUND)

if (LIBUSB_FOUND)
    message(STATUS "Found LIBUSB installed in the system")
    message(STATUS " - Includes: ${LIBUSB_INCLUDE_DIR}")
    message(STATUS " - Libraries: ${LIBUSB_LIBRARY}")

    add_definitions(-DLIBUSB_INSTALLED)
else ()
    message(STATUS "Cannot Find LIBUSB")
endif (LIBUSB_FOUND)

if (NOT EXECUTABLE_OUTPUT_PATH)
    set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)
endif ()

# Include PSDK header files
include_directories(${FFMPEG_INCLUDE_DIR})
include_directories(${DJI_OSAL_PATH})
# USB bulk HAL first, so that on Manifold 3 its hal_usb_bulk.h takes precedence
include_directories(${DJI_USB_BULK_HAL_PATH})
include_directories(${DJI_HAL_PATH})
include_directories(${DJI_STREAMING_PATH})
include_directories(${DJI_UTILS_PATH})
include_directories(${PSDK_PATH}/psdk_lib/include)
include_directories(include)

set(dependencies
  rclcpp
  rclcpp_lifecycle
  tf2
  tf2_ros
  sensor_msgs
  geometry_msgs
  std_msgs
  nav_msgs
  psdk_interfaces
  std_srvs
)

add_executable(psdk_wrapper_node
  src/main.cpp
)

add_library(psdk_wrapper_libs SHARED
  src/psdk_wrapper.cpp
  src/modules/telemetry.cpp
  src/modules/flight_control.cpp
  src/modules/camera.cpp
  src/modules/gimbal.cpp
  src/modules/liveview.cpp
  src/modules/hms.cpp
  src/modules/perception.cpp
  src/utils/node_thread.cpp
  ${DJI_OSAL_PATH}/osal_fs.c
  ${DJI_OSAL_PATH}/osal_socket.c
  ${DJI_OSAL_PATH}/osal.c
  ${DJI_HAL_PATH}/hal_network.c
  ${DJI_USB_BULK_HAL_PATH}/hal_usb_bulk.c
  ${DJI_HAL_PATH}/hal_uart.c
  ${DJI_STREAMING_PATH}/dji_camera_image_handler.cpp
  ${DJI_STREAMING_PATH}/dji_camera_stream_decoder.cpp
  ${DJI_UTILS_PATH}/utils/util_misc.c
  ${DJI_UTILS_PATH}/utils/util_file.c
  ${DJI_UTILS_PATH}/utils/cJSON.c
  ${DJI_UTILS_PATH}/utils/dji_config_manager.c
)

ament_target_dependencies(psdk_wrapper_libs
  ${dependencies}
)

target_link_libraries(
  psdk_wrapper_libs
  ${PSDK_LIB}
  nlohmann_json::nlohmann_json
)

target_link_libraries(
  psdk_wrapper_node
  psdk_wrapper_libs
  ${FFMPEG_LIBRARIES}
  -lusb-1.0
  -lavcodec
  -lswscale
)

install(TARGETS
  psdk_wrapper_node
  psdk_wrapper_libs
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY 
  include/
  ${PSDK_PATH}/psdk_lib/include/
  ${DJI_STREAMING_PATH}/
  ${DJI_OSAL_PATH}/
  ${DJI_UTILS_PATH}/
  DESTINATION include/
  FILES_MATCHING
    PATTERN "*.h"
    PATTERN "*.hpp"
)

# HAL headers: UART/network from manifold2, USB bulk from the selected
# platform. The manifold2 hal_usb_bulk.h is excluded on Manifold 3, because
# install() skips files whose timestamp matches an already installed one.
if(PSDK_PLATFORM STREQUAL "manifold3")
  install(DIRECTORY ${DJI_HAL_PATH}/
    DESTINATION include/
    FILES_MATCHING
      PATTERN "*.h"
      PATTERN "hal_usb_bulk.h" EXCLUDE
  )
  install(FILES ${DJI_USB_BULK_HAL_PATH}/hal_usb_bulk.h DESTINATION include/)
else()
  install(DIRECTORY ${DJI_HAL_PATH}/
    DESTINATION include/
    FILES_MATCHING
      PATTERN "*.h"
  )
endif()

install(DIRECTORY
  cfg
  launch
  DESTINATION share/${PROJECT_NAME}/
)

install(DIRECTORY
  ${DJI_HMS_RETURN_CODES_PATH}/
  DESTINATION share/${PROJECT_NAME}/cfg
)

ament_export_dependencies(
  "rclcpp"
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)

  # the following line skips the linter which checks for copyrights
  # uncomment the line when a copyright and license is not present in all source files
  # set(ament_cmake_copyright_FOUND TRUE)
  # the following line skips cpplint (only works in a git repo)
  # uncomment the line when this package is not in a git repo
  # set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

ament_export_include_directories(include)
ament_export_libraries(psdk_wrapper_libs)
ament_export_dependencies(${dependencies})

ament_package()
