# Copyright 2018 Universidad Carlos III de Madrid
# 
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
# 
#     http://www.apache.org/licenses/LICENSE-2.0
# 
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# GOOGLE TEST----------------------------------------------------------------------------
option(GRPPI_UNIT_TEST_ENABLE "Activate unit tests" OFF)

if(GRPPI_UNIT_TEST_ENABLE)
  # Set policy to support newer version of CMake and GTest
  # We will remove this when minimum version of CMake for GrPPI is move to 3.3 or
  # higher
  if(POLICY CMP0057)
    cmake_policy(SET CMP0057 NEW)
  endif()
  find_package(GTest)
  if(GTEST_FOUND)
    message(STATUS "GTest found")
  else()
    set(GRPPI_UNIT_TEST_ENABLE OFF CACHE BOOL "Activate unit tests" FORCE)
    message(STATUS "GTest not found")
    message(STATUS "  GRPPI_UNIT_TEST_ENABLE has been set to OFF")
    message(STATUS "  To enable unit tests, please, download and instal Google Test")
    message(STATUS "  https://github.com/google/googletest")
  endif()
endif()

if(GRPPI_UNIT_TEST_ENABLE)
  include_directories(${GTEST_INCLUDE_DIRS})

  if ( NOT (${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang") )
    # Add coverage options. Needs to be done before add_executable
    add_compile_options(--coverage)
  endif()

  set(PROJECT_TEST_NAME utest_${PROJECT_NAME_STR})
  file(GLOB TEST_SRC_FILES
      ${CMAKE_CURRENT_SOURCE_DIR}/*cpp)
  add_executable(${PROJECT_TEST_NAME} ${TEST_SRC_FILES})
  
  # Unit testing should be made in debug mode
  set(CMAKE_BUILD_TYPE "Debug")

  # Do not emit deprecated warnings for _legacy unit tests
  file(GLOB TEST_SRC_FILES_LEGACY
      ${CMAKE_CURRENT_SOURCE_DIR}/*_legacy.cpp)
  set_source_files_properties(${TEST_SRC_FILES_LEGACY} 
      PROPERTIES COMPILE_FLAGS "-Wno-deprecated-declarations")

  target_link_libraries(${PROJECT_TEST_NAME} 
      ${GTEST_BOTH_LIBRARIES}
      gcov
      ${TBB_LIBRARIES}
      ${CMAKE_THREAD_LIBS_INIT}
  )
  
  GTEST_ADD_TESTS(${PROJECT_TEST_NAME} "" ${TEST_SRC_FILES})

  # Coverage options
  if ( NOT (${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang") )

    find_program(LCOV_CMD lcov)
    find_program(GENHTML_CMD genhtml)

    ADD_CUSTOM_TARGET(coverage
        ${LCOV_CMD} --directory . --zerocounters
        COMMAND ${PROJECT_TEST_NAME}
        COMMAND ${LCOV_CMD} --directory . --capture --output-file mycov.info
        COMMAND ${LCOV_CMD} --remove mycov.info '/usr/*' 'unit_tests/*' 
              --output-file mycov.info.cleaned
        COMMAND ${GENHTML_CMD} -o mycov mycov.info.cleaned --legend -s 
        COMMAND ${CMAKE_COMMAND} -E remove mycov.info mycov.info.cleaned
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Computing coverage and generating report"
    )
  endif()

endif(GRPPI_UNIT_TEST_ENABLE)
