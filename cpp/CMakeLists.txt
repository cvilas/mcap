cmake_minimum_required(VERSION 3.22.1)
project(mcap LANGUAGES CXX)

#set(CMAKE_CXX_STANDARD 17)

set(MCAP_CPP_HEADERS_DIR ${PROJECT_SOURCE_DIR}/mcap/include/)

# Extract version information from one of the header files
file(READ "${MCAP_CPP_HEADERS_DIR}/mcap/types.hpp" HEADER_CONTENT)
string(REGEX MATCH "MCAP_LIBRARY_VERSION \"([0-9]+\.[0-9]+\.[0-9]+)\"" _ ${HEADER_CONTENT})
set(PROJECT_VERSION ${CMAKE_MATCH_1})

# Generate the headers-only library target
set(LIBRARY_NAME ${PROJECT_NAME})
add_library(${LIBRARY_NAME} INTERFACE)
target_include_directories(${LIBRARY_NAME}
  INTERFACE
    $<BUILD_INTERFACE:${MCAP_CPP_HEADERS_DIR}>
    $<INSTALL_INTERFACE:include>
)

# Define C++ language version
target_compile_features(${LIBRARY_NAME} INTERFACE cxx_std_17)

# Resolve dependencies
# TODO: This probably won't work if user's availability of these dependencies are different from the developer's
set(MCAP_CPP_DEPENDENCIES "")

find_package(lz4 QUIET)
if (${lz4_FOUND})
  message(STATUS "Found lz4 (${lz4_DIR})")
  list(APPEND MCAP_CPP_DEPENDENCIES LZ4::lz4)
else()
  message(STATUS "lz4 not found. Setting MCAP_COMPRESSION_NO_LZ4")
  target_compile_definitions(${LIBRARY_NAME} INTERFACE "MCAP_COMPRESSION_NO_LZ4")
endif()

find_package(zstd QUIET)
if (${zstd_FOUND})
  message(STATUS "Found zstd (${zstd_DIR})")
  list(APPEND MCAP_CPP_DEPENDENCIES zstd::zstd)
else()
  message(STATUS "zstd not found. Setting MCAP_COMPRESSION_NO_ZSTD")
  target_compile_definitions(${LIBRARY_NAME} INTERFACE "MCAP_COMPRESSION_NO_ZSTD")
endif()

target_link_libraries(${LIBRARY_NAME} INTERFACE ${MCAP_CPP_DEPENDENCIES})

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Install header files
install(
    DIRECTORY ${MCAP_CPP_HEADERS_DIR}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Install the libray target(s) and create export set
install(
    TARGETS ${LIBRARY_NAME}
    EXPORT ${PROJECT_NAME}Targets
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Generate package files
write_basic_package_version_file(
    "${PROJECT_NAME}-config-version.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# Install package files
install(
    EXPORT ${PROJECT_NAME}Targets
    FILE ${PROJECT_NAME}-targets.cmake
    NAMESPACE ${PROJECT_NAME}::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

install(
    FILES
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

# Users can use this library like this:
#find_package(mcap-cpp REQUIRED)
#target_link_libraries(their_target PRIVATE mcap-cpp::mcap-cpp)
