############################################################################
#    Copyright (C) 2014 by Ahmed Charles - acharles@outlook.com            #
#    Copyright (C) 2015-2018, 2020, 2022, 2024 by Stephen Lyons            #
#                                                - slysven@virginmedia.com #
#                                                                          #
#    This program is free software; you can redistribute it and/or modify  #
#    it under the terms of the GNU General Public License as published by  #
#    the Free Software Foundation; either version 2 of the License, or     #
#    (at your option) any later version.                                   #
#                                                                          #
#    This program is distributed in the hope that it will be useful,       #
#    but WITHOUT ANY WARRANTY; without even the implied warranty of        #
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the         #
#    GNU General Public License for more details.                          #
#                                                                          #
#    You should have received a copy of the GNU General Public License     #
#    along with this program; if not, write to the                         #
#    Free Software Foundation, Inc.,                                       #
#    59 Temple Place - Suite 330, Boston, MA  02111-1307, USA.             #
############################################################################

############################################################################
#                                                                          #
#    NOTICE: FreeBSD, OpenBSD and GNU/Hurd are not officially supported    #
#    platforms as such; the work on getting them working has been done by  #
#    myself, and other developers, unless they have explicitly said so,    #
#    are not able to address issues relating specifically to these         #
#    Operating Systems. Nevertheless users of these operating systems are  #
#    equally welcome to contribute to the development of Mudlet - bugfixes #
#    and enhancements are welcome from all!                                #
#        Stephen Lyons, February 2018, updated March 2021 & October 2022   #
#                                                                          #
############################################################################

# Should be called before PROJECT.
cmake_minimum_required(VERSION 3.3)

if(APPLE)
  # needed for fetching Sparkle
  cmake_minimum_required(VERSION 3.18)
  # minimum supported version by Qt6 for our Qt5/Qt6 build compatibility
  set(CMAKE_OSX_DEPLOYMENT_TARGET "12.0" CACHE STRING "")
endif()

if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/vcpkg/scripts/buildsystems/vcpkg.cmake)
  # In cases where VCPkg is NOT being used we need to skip this
  set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/vcpkg/scripts/buildsystems/vcpkg.cmake
    CACHE STRING "Vcpkg toolchain file")
  message(STATUS "Using Vcpkg toolchain file: ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/vcpkg/scripts/buildsystems/vcpkg.cmake .")
else()
  message(STATUS "Vcpkg toolchain file ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/vcpkg/scripts/buildsystems/vcpkg.cmake NOT found, proceeding without it.")
endif()

project(mudlet)

if(CMAKE_CONFIGURATION_TYPES)
  add_custom_target(
    check COMMAND ${CMAKE_CTEST_COMMAND} --force-new-ctest-process
                  --output-on-failure --build-config "$<CONFIGURATION>")
else()
  add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND}
                                  --force-new-ctest-process --output-on-failure)
endif()

enable_testing()

if(POLICY CMP0020)
  cmake_policy(SET CMP0020 NEW)
endif()

if(POLICY CMP0057)
  cmake_policy(SET CMP0057 NEW)
endif()

if(WIN32)
  set(APP_TARGET mudlet.exe)
elseif(APPLE)
  set(APP_TARGET Mudlet)
else()
  set(APP_TARGET mudlet)
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(Git REQUIRED)
# Get Git SHA1 hash
execute_process(
    COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_SHA1
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

# If a CI/CB build system provides an alternative Git SHA1 to identify the build
# use that instead - reporting which is being used:
if(DEFINED ENV{BUILD_COMMIT} AND NOT $ENV{BUILD_COMMIT} STREQUAL "")
    string(TOLOWER $ENV{BUILD_COMMIT} GIT_SHA1)
    message(STATUS "Git SHA1 set from environemnt: ${GIT_SHA1}")
else()
    message(STATUS "Git SHA1 used: ${GIT_SHA1}")
endif()

# Set APP_VERSION (update in mudlet.pro as well)
set(APP_VERSION 4.18.5)

# Set APP_BUILD based on environment variable MUDLET_VERSION_BUILD or default
if(DEFINED ENV{MUDLET_VERSION_BUILD} AND NOT $ENV{MUDLET_VERSION_BUILD} STREQUAL "")
    set(BUILD_VALUE "$ENV{MUDLET_VERSION_BUILD}-${GIT_SHA1}")
else()
    set(BUILD_VALUE "-dev-${GIT_SHA1}")
endif()

# Write to app-build.txt and set APP_BUILD
file(WRITE ${CMAKE_SOURCE_DIR}/src/app-build.txt "${BUILD_VALUE}")
message(STATUS "Value written to app-build.txt file: ${BUILD_VALUE}")
set(APP_BUILD ${BUILD_VALUE} CACHE STRING "Auto-set during CMake run. Do not alter manually." FORCE)

# For release builds, comment out the above and uncomment below:
# file(WRITE ${CMAKE_SOURCE_DIR}/src/app-build.txt " ")
# message(STATUS "Value written to app-build.txt file: {nothing}")
# set(APP_BUILD "" CACHE STRING "Auto-set during CMake run. Do not alter manually." FORCE)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

include(IncludeOptionalModule)

include_optional_module(ENVIRONMENT_VARIABLE WITH_UPDATER
                        OPTION_VARIABLE USE_UPDATER
                        READABLE_NAME "updater"
                        SUPPORTED_SYSTEMS "Linux"
                                          "Windows"
                                          "Darwin")

include_optional_module(ENVIRONMENT_VARIABLE WITH_FONTS
                        OPTION_VARIABLE USE_FONTS
                        READABLE_NAME "fonts")
include_optional_module(ENVIRONMENT_VARIABLE WITH_3DMAPPER
                        OPTION_VARIABLE USE_3DMAPPER
                        READABLE_NAME "3D mapper")
include_optional_module(ENVIRONMENT_VARIABLE WITH_VARIABLE_SPLASH_SCREEN
                        OPTION_VARIABLE USE_VARIABLE_SPLASH_SCREEN
                        READABLE_NAME "build-type splash screen")
include_optional_module(ENVIRONMENT_VARIABLE WITH_OWN_QTKEYCHAIN
                        OPTION_VARIABLE USE_OWN_QTKEYCHAIN
                        READABLE_NAME "own QtKeychain library")
include_optional_module(ENVIRONMENT_VARIABLE WITH_QT6
                        OPTION_VARIABLE USE_QT6
                        READABLE_NAME "build with Qt6 (if installed)")

if(USE_QT6)
  find_package(
          Qt6 6.2.0
          QUIET)
  if(NOT Qt6_FOUND)
    message(STATUS "Qt6 >= 6.2.0 was not found. Building with Qt5.")
  endif()
endif()

# Set Qt version for src, test, translations and communi
set(WITH_QT6 ${Qt6_FOUND} CACHE BOOL "Whether to build with Qt6 or Qt5.")
# Set Qt version for edbee-lib and dblsqd
if(WITH_QT6)
  option(BUILD_WITH_QT5 "" OFF)
else()
  option(BUILD_WITH_QT5 "" ON)
endif()
# Set Qt version for qtkeychain
option(BUILD_WITH_QT6 "" ${WITH_QT6})

include(InitGitSubmodule)

git_submodule_init(
  CHECK_FILE "3rdparty/edbee-lib/CMakeLists.txt" SUBMODULE_PATH
  "3rdparty/edbee-lib" READABLE_NAME "edbee-lib editor widget")

if(USE_OWN_QTKEYCHAIN)
  git_submodule_init(
    CHECK_FILE "3rdparty/qtkeychain/CMakeLists.txt"
    SUBMODULE_PATH "3rdparty/qtkeychain"
    READABLE_NAME "QtKeychain")
  add_subdirectory(3rdparty/qtkeychain)
endif()

git_submodule_init(
  CHECK_FILE "3rdparty/lcf/lcf-scm-1.rockspec" SUBMODULE_PATH "3rdparty/lcf"
  READABLE_NAME "lua code formatter source code")

# PLACEMARKER: sample benchmarking code
# include(FetchContent)
# FetchContent_Declare(
#     nanobench
#     GIT_REPOSITORY https://github.com/martinus/nanobench.git
#     GIT_TAG v4.3.11
#     GIT_SHALLOW TRUE)
# FetchContent_MakeAvailable(nanobench)

if(USE_UPDATER)
  git_submodule_init(CHECK_FILE "3rdparty/dblsqd/CMakeLists.txt" SUBMODULE_PATH
                     "3rdparty/dblsqd" READABLE_NAME "DBLSQD updater")
endif()

if(APPLE)
  if(USE_UPDATER)
    git_submodule_init(
      CHECK_FILE "3rdparty/sparkle-glue/CMakeLists.txt" SUBMODULE_PATH
      "3rdparty/sparkle-glue" READABLE_NAME "Sparkle glue for updater")

    if(NOT EXISTS "${CMAKE_HOME_DIRECTORY}/3rdparty/cocoapods/Pods/Sparkle")
      message(
        STATUS "Sparkle is missing, fetching it..."
      )
      file(DOWNLOAD "https://github.com/sparkle-project/Sparkle/releases/download/1.25.0/Sparkle-1.25.0.tar.xz" "${CMAKE_HOME_DIRECTORY}/3rdparty/cocoapods/Pods/Sparkle-1.25.0.tar.xz"
        TIMEOUT 60  # seconds
        TLS_VERIFY ON
      )
      file(ARCHIVE_EXTRACT INPUT "${CMAKE_HOME_DIRECTORY}/3rdparty/cocoapods/Pods/Sparkle-1.25.0.tar.xz"
        DESTINATION "${CMAKE_HOME_DIRECTORY}/3rdparty/cocoapods/Pods/Sparkle")
      file(REMOVE "${CMAKE_HOME_DIRECTORY}/3rdparty/cocoapods/Pods/Sparkle-1.25.0.tar.xz")
    endif()
  endif()
endif()

find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif(CCACHE_FOUND)

add_subdirectory(3rdparty/edbee-lib/edbee-lib)
add_subdirectory(3rdparty/communi)
add_subdirectory(translations/translated)
add_subdirectory(src)
add_subdirectory(test)

if(USE_UPDATER)
  add_subdirectory(3rdparty/dblsqd)

  if(APPLE)
    add_subdirectory(3rdparty/sparkle-glue)
  endif()

endif()

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE -O3)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_DEBUG -O0)
