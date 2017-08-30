CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
PROJECT(RpolyPlusPlus C CXX)
INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR})
LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# System info ==========================================================
message("=============================================================")
cmake_host_system_information(RESULT HOST_HOSTNAME QUERY HOSTNAME)
cmake_host_system_information(RESULT HOST_NUMBER_OF_LOGICAL_CORES QUERY NUMBER_OF_LOGICAL_CORES)
cmake_host_system_information(RESULT HOST_NUMBER_OF_PHYSICAL_CORES QUERY NUMBER_OF_PHYSICAL_CORES)
cmake_host_system_information(RESULT HOST_TOTAL_VIRTUAL_MEMORY QUERY TOTAL_VIRTUAL_MEMORY)
cmake_host_system_information(RESULT HOST_AVAILABLE_VIRTUAL_MEMORY QUERY AVAILABLE_VIRTUAL_MEMORY)
cmake_host_system_information(RESULT HOST_TOTAL_PHYSICAL_MEMORY QUERY TOTAL_PHYSICAL_MEMORY)
cmake_host_system_information(RESULT HOST_AVAILABLE_PHYSICAL_MEMORY QUERY AVAILABLE_PHYSICAL_MEMORY)

message(STATUS "HOSTNAME: ${HOST_HOSTNAME}")
message(STATUS "NUMBER_OF_LOGICAL_CORES: ${HOST_NUMBER_OF_LOGICAL_CORES}")
message(STATUS "NUMBER_OF_PHYSICAL_CORES: ${HOST_NUMBER_OF_PHYSICAL_CORES}")
message(STATUS "TOTAL_VIRTUAL_MEMORY: ${HOST_TOTAL_VIRTUAL_MEMORY}")
message(STATUS "AVAILABLE_VIRTUAL_MEMORY: ${HOST_AVAILABLE_VIRTUAL_MEMORY}")
message(STATUS "TOTAL_PHYSICAL_MEMORY: ${HOST_TOTAL_PHYSICAL_MEMORY}")
message(STATUS "AVAILABLE_PHYSICAL_MEMORY: ${HOST_AVAILABLE_PHYSICAL_MEMORY}")

# Compilation at Maur ==================================================
if(${HOST_HOSTNAME} MATCHES login.* OR ${HOST_HOSTNAME} MATCHES compute.*)
  option(COMPILATION_AT_MAUR " " ON)
else()
  option(COMPILATION_AT_MAUR " " OFF)
endif()

# Maur =================================================================
message("=============================================================")
if(COMPILATION_AT_MAUR)

  OPTION(BUILD_TESTING "Enable testing" OFF)

  # EIGEN
  set(EIGEN3_INCLUDE_DIR $ENV{HOME}/git/libraries/eigen3.3-beta1-2/Eigen)
  include_directories($ENV{HOME}/git/libraries/eigen3.3-beta1-2)
  include_directories(${EIGEN3_INCLUDE_DIR})
  message(STATUS "EIGEN3_INCLUDE_DIR: ${EIGEN3_INCLUDE_DIR}")

else()

  # By default, the unit tests will be built.
  OPTION(BUILD_TESTING "Enable testing" ON)

  # Set Minimum Eigen version and find the installed package.
  SET(MIN_EIGEN_VERSION 3.2.0)
  FIND_PACKAGE(Eigen ${MIN_EIGEN_VERSION} REQUIRED)

endif()

IF (EIGEN_FOUND)
  MESSAGE("-- Found Eigen version ${EIGEN_VERSION}: ${EIGEN_INCLUDE_DIRS}")
ENDIF (EIGEN_FOUND)


# Use a larger inlining threshold for Clang, since it hobbles Eigen,
# resulting in an unreasonably slow version of the blas routines. The
# -Qunused-arguments is needed because CMake passes the inline
# threshold to the linker and clang complains about it and dies.
IF (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  SET(CMAKE_CXX_FLAGS
    "${CMAKE_CXX_FLAGS} -Qunused-arguments -mllvm -inline-threshold=600")
  # Older versions of Clang (<= 2.9) do not support the 'return-type-c-linkage'
  # option, so check for its presence before adding it to the default flags set.
  INCLUDE(CheckCXXCompilerFlag)
  CHECK_CXX_COMPILER_FLAG("-Wno-return-type-c-linkage"
    HAVE_RETURN_TYPE_C_LINKAGE)
  IF (HAVE_RETURN_TYPE_C_LINKAGE)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-return-type-c-linkage")
  ENDIF(HAVE_RETURN_TYPE_C_LINKAGE)
ENDIF ()

# NOTE: This fix came from Ceres solver with the following comment:
#
# Eigen SparseQR generates various compiler warnings related to unused and
# uninitialised local variables, which prevents Ceres compilation as we use
# -Werror.  To avoid having to individually suppress these warnings around
# the #include statments for Eigen headers across all GCC/Clang versions, we
# tell CMake to treat Eigen headers as system headers.  This results in all
# compiler warnings from them being suppressed.
#
# Note that this is *not* propagated to clients, ie CERES_INCLUDE_DIRS
# used by clients after find_package(Ceres) does not identify Eigen as
# as system headers.
#INCLUDE_DIRECTORIES(SYSTEM ${EIGEN_INCLUDE_DIRS})
INCLUDE_DIRECTORIES(SYSTEM ${EIGEN3_INCLUDE_DIR})

SET(RPOLY_PLUS_PLUS_SRCS
  src/polynomial.cc
  src/find_polynomial_roots_jenkins_traub.cc)

ADD_LIBRARY(rpoly_plus_plus SHARED ${RPOLY_PLUS_PLUS_SRCS})

IF (CMAKE_BUILD_TYPE STREQUAL "Debug")
  MESSAGE("\n=================================================================================")
  MESSAGE("\n-- Build type: Debug. Performance will be terrible!")
  MESSAGE("-- Add -DCMAKE_BUILD_TYPE=Release to the CMake command line to get an optimized build.")
  MESSAGE("\n=================================================================================")
ENDIF (CMAKE_BUILD_TYPE STREQUAL "Debug")

IF (BUILD_TESTING)
  ADD_SUBDIRECTORY(gtest)
  INCLUDE_DIRECTORIES("${CMAKE_SOURCE_DIR}/gtest/include")
  MACRO (GTEST FILENAME)
    STRING(REPLACE "/" ";" TEST_NAME_LIST ${FILENAME})
    LIST(REVERSE TEST_NAME_LIST)
    LIST(GET TEST_NAME_LIST 0 TEST_NAME)
    ADD_EXECUTABLE(${TEST_NAME}_test test/test_main.cc test/${FILENAME}_test.cc)
    TARGET_LINK_LIBRARIES(${TEST_NAME}_test
      gtest
      rpoly_plus_plus)
    ADD_TEST(NAME ${TEST_NAME}_test
      COMMAND ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${TEST_NAME}_test)
  ENDMACRO (GTEST)

  GTEST(find_polynomial_roots_jenkins_traub)
ENDIF (BUILD_TESTING)
