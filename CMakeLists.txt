cmake_minimum_required(VERSION 3.12)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TESTS "Build tests" ON)
option(BUILD_DOCS "Build documentation" OFF)
option(BUILD_COVERAGE "Build code coverage" OFF)

set(
  HUNTER_CACHE_SERVERS
  "https://github.com/bmstu-iu8-cpp-sem-3/hunter-cache"
  CACHE STRING "Default cache server"
)

include("cmake/HunterGate.cmake")

HunterGate(
  URL "https://github.com/cpp-pm/hunter/archive/v0.23.278.tar.gz"
  SHA1 "35fa55bc1dfcf9f4d7264f8bb26cd1759ce1ae07" FILEPATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.cmake )


project(template VERSION 0.1.0)
string(APPEND CMAKE_CXX_FLAGS " -pedantic -Werror -Wall -Wextra")
string(APPEND CMAKE_CXX_FLAGS " -Wno-unused-command-line-argument")
string(APPEND CMAKE_CXX_FLAGS " -Wshadow -Wnon-virtual-dtor")

hunter_add_package(gflags)
find_package(gflags CONFIG REQUIRED)

hunter_add_package(rocksdb)
find_package(RocksDB CONFIG REQUIRED)

hunter_add_package(Boost COMPONENTS program_options log thread system filesystem )
find_package(Boost CONFIG REQUIRED program_options log  thread system filesystem )
hunter_add_package(GTest)
find_package(GTest CONFIG REQUIRED)

add_library(${PROJECT_NAME} STATIC
  # enum your files and delete this comment
        include/dbWrapper.hpp
        sources/dbWrapper.cpp
        include/hashData.hpp
        sources/hashData.cpp
        include/ThreadPool.h
)

add_executable(demo
  ${CMAKE_CURRENT_SOURCE_DIR}/demo/main.cpp
)

target_include_directories(${PROJECT_NAME} PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
  "$<INSTALL_INTERFACE:include>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/third-party/PicoSHA2>"

        )
target_link_libraries(demo ${PROJECT_NAME} )
target_link_libraries( ${PROJECT_NAME} RocksDB::rocksdb
        ${Boost_PROGRAM_OPTIONS_LIBRARY}
Boost::log
        ${Boost_THREAD_LIBRARY}
        ${Boost_LOG_LIBRARY}
        ${Boost_FILESYSTEM_LIBRARY}
        )

if(BUILD_TESTS)
  add_executable(tests

    ${CMAKE_CURRENT_SOURCE_DIR}/tests/test.cpp
  )
  target_link_libraries(tests ${PROJECT_NAME} GTest::gtest_main)
  enable_testing()
  add_test(NAME unit_tests COMMAND tests)
endif()

if(BUILD_COVERAGE)
  set(ENABLE_COVERAGE ON CACHE BOOL "Enable coverage build." FORCE)
  list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/tools/coverage/cmake")
  find_package(codecov)
  string(APPEND CMAKE_CXX_FLAGS " --coverage")
  add_coverage(${PROJECT_NAME})
  add_coverage(tests)
  coverage_evaluate()
endif()

if(BUILD_DOCS)
  set(DOXYGEN_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/_docs)
  set(DOXYGEN_EXAMPLE_PATH examples)
  set(DOXYGEN_GENERATE_HTML NO)
  set(DOXYGEN_GENERATE_MAN YES)
  find_package(Doxygen)
  doxygen_add_docs(docs include)
  install(DIRECTORY ${DOXYGEN_OUTPUT_DIRECTORY}/man DESTINATION share)
  # add_dependencies(install docs) need to call target docs before install or package
endif()

include(CPackConfig.cmake)

install(TARGETS ${PROJECT_NAME} demo
  EXPORT "${PROJECT_NAME}-targets"
  RUNTIME DESTINATION bin
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  INCLUDES DESTINATION include
)

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/ DESTINATION include)

install(EXPORT "${PROJECT_NAME}-targets"
  NAMESPACE "${PROJECT_NAME}::"
  DESTINATION "lib/cmake"
)
