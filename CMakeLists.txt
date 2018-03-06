cmake_minimum_required(VERSION 3.1)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

set(CMAKE_TOOLCHAIN_FILE "${CMAKE_CURRENT_SOURCE_DIR}/cmake/toolchain.cmake"
    CACHE FILEPATH
    "Toolchain to use for building this package and dependencies")

include(HunterGate)
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.20.14.tar.gz"
    SHA1 "18cdbf5c47abdda437c73bf5437b7c3c65afe47c"
)

project(cctz VERSION 2.2)

include(CTest)
include(FeatureSummary)

option(BUILD_TOOLS "Whether or not to build tools" ON)
option(BUILD_EXAMPLES "Whether or not to build examples" ON)

if (BUILD_TESTING)
  hunter_add_package(benchmark)
  find_package(benchmark CONFIG REQUIRED)
  hunter_add_package(GTest)
  find_package(GMock CONFIG REQUIRED)
  find_package(GTest CONFIG REQUIRED)
endif()

add_library(cctz
  src/civil_time_detail.cc
  src/time_zone_fixed.cc
  src/time_zone_fixed.h
  src/time_zone_format.cc
  src/time_zone_if.cc
  src/time_zone_if.h
  src/time_zone_impl.cc
  src/time_zone_impl.h
  src/time_zone_info.cc
  src/time_zone_info.h
  src/time_zone_libc.cc
  src/time_zone_libc.h
  src/time_zone_lookup.cc
  src/time_zone_posix.cc
  src/time_zone_posix.h
  src/tzfile.h
  src/zone_info_source.cc)

target_include_directories(cctz PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>)

if (BUILD_TOOLS)
  add_executable(time_tool src/time_tool.cc)
  target_link_libraries(time_tool cctz)
endif()

if (BUILD_EXAMPLES)
  add_subdirectory(examples)
endif()

if (BUILD_TESTING)
  add_executable(civil_time_test src/civil_time_test.cc)
  target_link_libraries(civil_time_test PUBLIC
    cctz
    GTest::main)
  add_test(civil_time_test civil_time_test)

  add_executable(time_zone_lookup_test src/time_zone_lookup_test.cc)
  target_link_libraries(time_zone_lookup_test PUBLIC
    cctz
    GTest::main)
  add_test(time_zone_lookup_test time_zone_lookup_test)

  add_executable(time_zone_format_test src/time_zone_format_test.cc)
  target_link_libraries(time_zone_format_test PUBLIC
    cctz GMock::main)
  add_test(time_zone_format_test time_zone_format_test)

  # tests runs on testdata
  set_property(
    TEST
      civil_time_test
      time_zone_format_test
      time_zone_lookup_test
    PROPERTY
      ENVIRONMENT "TZDIR=${CMAKE_CURRENT_SOURCE_DIR}/testdata/zoneinfo"
    )

  add_executable(benchmarks src/benchmarks.cc)
  target_link_libraries(benchmarks cctz benchmark::benchmark)
endif()

feature_summary(WHAT ALL FATAL_ON_MISSING_REQUIRED_PACKAGES)

# Installation (https://github.com/forexample/package-example)

# Layout. This works for all platforms:
#   * <prefix>/lib/cmake/<PROJECT-NAME>
#   * <prefix>/lib/
#   * <prefix>/include/
set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Configuration
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

# Include module with fuction 'write_basic_package_version_file'
include(CMakePackageConfigHelpers)

# Configure '<PROJECT-NAME>ConfigVersion.cmake'
# Use:
#   * PROJECT_VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Configure '<PROJECT-NAME>Config.cmake'
# Use variables:
#   * TARGETS_EXPORT_NAME
#   * PROJECT_NAME
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

# Targets:
#   * <prefix>/lib/libcctz.a
#   * header location after install: <prefix>/include/cctz/time_zone.h
#   * headers can be included by C++ code `#include <cctz/time_zone.h>`
install(
    TARGETS cctz
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
)

# Headers:
#   * include/cctz/time_zone.h -> <prefix>/include/cctz/time_zone.h
install(
    DIRECTORY "include/cctz"
    DESTINATION "${include_install_dir}"
    FILES_MATCHING PATTERN "*.h"
)

# Config
#   * <prefix>/lib/cmake/cctz/cctzConfig.cmake
#   * <prefix>/lib/cmake/cctz/cctzConfigVersion.cmake
install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

# Config
#   * <prefix>/lib/cmake/cctz/cctzTargets.cmake
install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
