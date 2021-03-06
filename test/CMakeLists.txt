CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
PROJECT(ArrayFire-Tests)

# Find CUDA and OpenCL
SET(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules")
FIND_PACKAGE(CUDA QUIET)
FIND_PACKAGE(OpenCL QUIET)

OPTION(BUILD_SINGLE_TEST_FILE "Build tests in a single file" OFF)

# If the tests are not being built at the same time as ArrayFire,
# we need to first find the ArrayFire library
IF(TARGET afcpu OR TARGET afcuda OR TARGET afopencl OR TARGET af)
    SET(ArrayFire_CPU_FOUND False)
    SET(ArrayFire_CUDA_FOUND False)
    SET(ArrayFire_OpenCL_FOUND False)
    SET(ArrayFire_Unified_FOUND False)
ELSE()
    SET_PROPERTY(GLOBAL PROPERTY USE_FOLDERS ON)
    FIND_PACKAGE(ArrayFire REQUIRED)
    INCLUDE_DIRECTORIES(${ArrayFire_INCLUDE_DIRS})
    OPTION(BUILD_NONFREE "Build Tests for nonfree algorithms" OFF)

    IF(${BUILD_NONFREE})
        MESSAGE(WARNING "Building With NONFREE ON requires the following patents")
        SET(BUILD_NONFREE_SIFT ON CACHE BOOL "Build ArrayFire with SIFT")
    ELSE(${BUILD_NONFREE})
        UNSET(BUILD_NONFREE_SIFT CACHE) # BUILD_NONFREE_SIFT cannot be built without BUILD_NONFREE
    ENDIF(${BUILD_NONFREE})

    IF(${BUILD_NONFREE_SIFT})
      ADD_DEFINITIONS(-DAF_BUILD_NONFREE_SIFT)

      MESSAGE(WARNING "Building with SIFT requires the following patents")

      MESSAGE("Method and apparatus for identifying scale invariant features"
        "in an image and use of same for locating an object in an image,\" David"
        "G. Lowe, US Patent 6,711,293 (March 23, 2004). Provisional application"
        "filed March 8, 1999. Asignee: The University of British Columbia. For"
        "further details, contact David Lowe (lowe@cs.ubc.ca) or the"
        "University-Industry Liaison Office of the University of British"
        "Columbia.")
    ENDIF(${BUILD_NONFREE_SIFT})

    # ENABLE_TESTING is required when building only tests
    # When building from source, enable_testing is picked from from the main
    # CMakeLists.txt
    ENABLE_TESTING()
ENDIF()

REMOVE_DEFINITIONS(-std=c++11)

MACRO(CREATE_TESTS BACKEND AFLIBNAME GTEST_LIBS OTHER_LIBS)
    STRING(TOUPPER ${BACKEND} DEF_NAME)

    # For some reason passing FILES/UNIFIED_FILES to macro doesn't work
    IF(${BACKEND} STREQUAL "unified")
      SET(TEST_FILES ${UNIFIED_FILES})
    ELSE(${BACKEND} STREQUAL "unified")
      SET(TEST_FILES ${FILES})
    ENDIF(${BACKEND} STREQUAL "unified")

    IF (${BUILD_SINGLE_TEST_FILE})
      SET(TEST_NAME test_${BACKEND})
      SET(TEST_NAME_BASIC test_basic_${BACKEND})
      ADD_EXECUTABLE(${TEST_NAME} ${CPP_FILES})
      ADD_EXECUTABLE(${TEST_NAME_BASIC} basic_c.c)

      TARGET_LINK_LIBRARIES(${TEST_NAME}  PRIVATE ${AFLIBNAME}
        PRIVATE ${THREAD_LIB_FLAG}
        PRIVATE ${GTEST_LIBS}
        PRIVATE ${OTHER_LIBS})

      TARGET_LINK_LIBRARIES(${TEST_NAME_BASIC}  PRIVATE ${AFLIBNAME}
        PRIVATE ${THREAD_LIB_FLAG}
        PRIVATE ${GTEST_LIBS}
        PRIVATE ${OTHER_LIBS})

      SET_TARGET_PROPERTIES(${TEST_NAME_BASIC}
        PROPERTIES
        COMPILE_FLAGS -DAF_${DEF_NAME}
        FOLDER "Tests/${BACKEND}")

    ELSE()
      FOREACH(FILE ${TEST_FILES})
        GET_FILENAME_COMPONENT(FNAME ${FILE} NAME_WE)
        SET(TEST_NAME ${FNAME}_${BACKEND})

        IF(NOT ${BUILD_NONFREE} AND "${FILE}" MATCHES ".nonfree.")
          MESSAGE(STATUS "Removing ${FILE} from ctest")
        ELSEIF("${FILE}" MATCHES ".manual.")
          MESSAGE(STATUS "Removing ${FILE} from ctest")
        ELSE()
          ADD_TEST(Test_${TEST_NAME} ${TEST_NAME})
        ENDIF()

        FILE(GLOB TEST_FILE "${FNAME}.cpp" "${FNAME}.c")
        ADD_EXECUTABLE(${TEST_NAME} ${TEST_FILE})
        TARGET_LINK_LIBRARIES(${TEST_NAME}  PRIVATE ${AFLIBNAME}
          PRIVATE ${THREAD_LIB_FLAG}
          PRIVATE ${GTEST_LIBS}
          PRIVATE ${OTHER_LIBS})

        SET_TARGET_PROPERTIES(${TEST_NAME}
          PROPERTIES
          COMPILE_FLAGS -DAF_${DEF_NAME}
          FOLDER "Tests/${BACKEND}")
      ENDFOREACH()
    ENDIF()

ENDMACRO(CREATE_TESTS)

MACRO(CHECK_AND_CREATE_TESTS BACKEND AFLIBNAME GTEST_LIBS OTHER_LIBS)
    STRING(TOUPPER ${BACKEND} BACKEND_NAME_UPPER)
    MESSAGE(STATUS "TESTS: ${BACKEND_NAME_UPPER} backend is ${BUILD_${BACKEND_NAME_UPPER}}.")
    IF(${BUILD_${BACKEND_NAME_UPPER}})
        CREATE_TESTS(${BACKEND} ${AFLIBNAME} "${GTEST_LIBS}" "${OTHER_LIBS}")
    ENDIF()
ENDMACRO(CHECK_AND_CREATE_TESTS)

FIND_PACKAGE(Threads REQUIRED)
IF(CMAKE_USE_PTHREADS_INIT AND NOT "${APPLE}")
    SET(THREAD_LIB_FLAG "-pthread")
ELSE()
    SET(THREAD_LIB_FLAG ${CMAKE_THREAD_LIBS_INIT})
ENDIF()

OPTION(USE_RELATIVE_TEST_DIR "Use relative paths for the test data directory(For continious integration(CI) purposes only)" OFF)

IF(${USE_RELATIVE_TEST_DIR})
    # RELATIVE_TEST_DATA_DIR is a User-visible option with default value of test/data directory
    SET(RELATIVE_TEST_DATA_DIR "${CMAKE_CURRENT_SOURCE_DIR}/data" CACHE STRING "Relative Test Data Directory")
    SET(TESTDATA_SOURCE_DIR ${RELATIVE_TEST_DATA_DIR})
ELSE(${USE_RELATIVE_TEST_DIR})  # Not using relative test data directory
    SET(TESTDATA_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/data")
ENDIF(${USE_RELATIVE_TEST_DIR})

IF (${CMAKE_GENERATOR} STREQUAL "Xcode")
    ADD_DEFINITIONS("-D TEST_DIR=\"\\\\\"${TESTDATA_SOURCE_DIR}\\\\\"\"")
ELSE (${CMAKE_GENERATOR} STREQUAL "Xcode")
    ADD_DEFINITIONS("-D TEST_DIR=\"\\\"${TESTDATA_SOURCE_DIR}\\\"\"")
ENDIF (${CMAKE_GENERATOR} STREQUAL "Xcode")

IF(NOT ${USE_RELATIVE_TEST_DIR})
    # Check if data exists
    IF (EXISTS "${TESTDATA_SOURCE_DIR}" AND IS_DIRECTORY "${TESTDATA_SOURCE_DIR}"
        AND EXISTS "${TESTDATA_SOURCE_DIR}/README.md")
        # Test data is available
        # Do Nothing
    ELSE (EXISTS "${TESTDATA_SOURCE_DIR}" AND IS_DIRECTORY "${TESTDATA_SOURCE_DIR}"
        AND EXISTS "${TESTDATA_SOURCE_DIR}/README.md")
        MESSAGE(STATUS "Test submodules unavailable. Updating submodules.")
        EXECUTE_PROCESS(
            COMMAND git submodule update --init --recursive
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            OUTPUT_QUIET
        )
    ENDIF()
ENDIF(NOT ${USE_RELATIVE_TEST_DIR})

OPTION(USE_SYSTEM_GTEST "Use GTEST from system libraries" OFF)
IF(USE_SYSTEM_GTEST)
    FIND_PACKAGE(GTest REQUIRED)
ELSE(USE_SYSTEM_GTEST)
    INCLUDE("${CMAKE_MODULE_PATH}/build_gtest.cmake")
ENDIF(USE_SYSTEM_GTEST)

INCLUDE_DIRECTORIES(${GTEST_INCLUDE_DIRS})

INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})
FILE(GLOB FILES "*.cpp" "*.c")
FILE(GLOB CPP_FILES "*.cpp")
LIST(SORT FILES)  # Tests execute in alphabetical order

# We only build backend.cpp for Unified backend
SET(UNIFIED_FILES "backend.cpp;main.cpp")
LIST(SORT UNIFIED_FILES)  # Tests execute in alphabetical order

# Next we build each example using every backend.
IF(${ArrayFire_CPU_FOUND})  # variable defined by FIND(ArrayFire ...)
    OPTION(BUILD_CPU "Build ArrayFire Tests for CPU backend" ON)
    CHECK_AND_CREATE_TESTS(cpu ${ArrayFire_CPU_LIBRARIES} "${GTEST_LIBRARIES}" "")
ELSEIF(TARGET afcpu)        # variable defined by the ArrayFire build tree
    CHECK_AND_CREATE_TESTS(cpu afcpu "${GTEST_LIBRARIES}" "")
ELSE()
    MESSAGE(STATUS "TESTS: CPU backend is OFF. afcpu was not found.")
ENDIF()

# CUDA Backend
IF (${CUDA_FOUND})
    IF(${ArrayFire_CUDA_FOUND})  # variable defined by FIND(ArrayFire ...)
        FIND_LIBRARY( CUDA_NVVM_LIBRARY
          NAMES "nvvm"
          PATH_SUFFIXES "nvvm/lib64" "nvvm/lib"
          PATHS ${CUDA_TOOLKIT_ROOT_DIR}
          DOC "CUDA NVVM Library"
          )
        MARK_AS_ADVANCED(CUDA_NVVM_LIBRARY)
        # If OSX && CLANG && CUDA < 7
        IF("${APPLE}" AND ${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" AND ${CUDA_VERSION_MAJOR} VERSION_LESS 7)
            OPTION(BUILD_CUDA "Build ArrayFire Tests for CUDA backend" ON)
            CHECK_AND_CREATE_TESTS(cuda ${ArrayFire_CUDA_LIBRARIES} "${GTEST_LIBRARIES_STDLIB}" "${CUDA_CUBLAS_LIBRARIES};${CUDA_LIBRARIES};${CUDA_cusolver_LIBRARY};${CUDA_CUFFT_LIBRARIES};${CUDA_NVVM_LIBRARY};${CUDA_CUDA_LIBRARY}")

            FOREACH(FILE ${FILES})
                GET_FILENAME_COMPONENT(FNAME ${FILE} NAME_WE)
                SET(TEST_NAME ${FNAME}_cuda)
                SET_TARGET_PROPERTIES(${TEST_NAME} PROPERTIES COMPILE_FLAGS -stdlib=libstdc++)
                SET_TARGET_PROPERTIES(${TEST_NAME} PROPERTIES LINK_FLAGS -stdlib=libstdc++)
            ENDFOREACH()

        # ELSE OSX && CLANG && CUDA < 7
        ELSE("${APPLE}" AND ${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" AND ${CUDA_VERSION_MAJOR} VERSION_LESS 7)
            OPTION(BUILD_CUDA "Build ArrayFire Tests for CUDA backend" ON)
            CHECK_AND_CREATE_TESTS(cuda ${ArrayFire_CUDA_LIBRARIES} "${GTEST_LIBRARIES}" "${CUDA_CUBLAS_LIBRARIES};${CUDA_LIBRARIES};${CUDA_cusolver_LIBRARY};${CUDA_CUFFT_LIBRARIES};${CUDA_NVVM_LIBRARY};${CUDA_CUDA_LIBRARY}")

        ENDIF("${APPLE}" AND ${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" AND ${CUDA_VERSION_MAJOR} VERSION_LESS 7)

    ELSEIF(TARGET afcuda)        # variable defined by the ArrayFire build tree
        # If OSX && CLANG && CUDA < 7
        IF("${APPLE}" AND ${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" AND ${CUDA_VERSION_MAJOR} VERSION_LESS 7)
            CHECK_AND_CREATE_TESTS(cuda afcuda "${GTEST_LIBRARIES_STDLIB}" "${CUDA_CUBLAS_LIBRARIES};${CUDA_LIBRARIES};${CUDA_cusolver_LIBRARY};${CUDA_CUFFT_LIBRARIES};${CUDA_NVVM_LIBRARY};${CUDA_CUDA_LIBRARY}")

            FOREACH(FILE ${FILES})
                GET_FILENAME_COMPONENT(FNAME ${FILE} NAME_WE)
                SET(TEST_NAME ${FNAME}_cuda)
                SET_TARGET_PROPERTIES(${TEST_NAME} PROPERTIES COMPILE_FLAGS -stdlib=libstdc++)
                SET_TARGET_PROPERTIES(${TEST_NAME} PROPERTIES LINK_FLAGS -stdlib=libstdc++)
            ENDFOREACH()

        # ELSE OSX && CLANG && CUDA < 7
        ELSE("${APPLE}" AND ${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" AND ${CUDA_VERSION_MAJOR} VERSION_LESS 7)
            CHECK_AND_CREATE_TESTS(cuda afcuda "${GTEST_LIBRARIES}" "${CUDA_CUBLAS_LIBRARIES};${CUDA_LIBRARIES};${CUDA_cusolver_LIBRARY};${CUDA_CUFFT_LIBRARIES};${CUDA_NVVM_LIBRARY};${CUDA_CUDA_LIBRARY}")

        ENDIF("${APPLE}" AND ${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang" AND ${CUDA_VERSION_MAJOR} VERSION_LESS 7)
    ELSE()
        MESSAGE(STATUS "TESTS: CUDA backend is OFF. afcuda was not found")
    ENDIF()
ELSE()
    MESSAGE(STATUS "TESTS: CUDA backend is OFF. CUDA was not found")
ENDIF()

# OpenCL Backend
IF (${OpenCL_FOUND})
    INCLUDE_DIRECTORIES(${OpenCL_INCLUDE_DIRS})
    IF(${ArrayFire_OpenCL_FOUND})  # variable defined by FIND(ArrayFire ...)
        OPTION(BUILD_OPENCL "Build ArrayFire Tests for OpenCL backend" ON)
        MESSAGE(${OpenCL_LIBRARIES})
        CHECK_AND_CREATE_TESTS(opencl ${ArrayFire_OpenCL_LIBRARIES} "${GTEST_LIBRARIES}" "${OpenCL_LIBRARIES}")
    ELSEIF(TARGET afopencl)        # variable defined by the ArrayFire build tree
        CHECK_AND_CREATE_TESTS(opencl afopencl "${GTEST_LIBRARIES}" "${OpenCL_LIBRARIES}")
    ELSE()
        MESSAGE(STATUS "TESTS: OpenCL backend is OFF. afopencl was not found")
    ENDIF()
ELSE()
    MESSAGE(STATUS "TESTS: OpenCL backend is OFF. OpenCL was not found")
ENDIF()

# Unified Backend
IF(${ArrayFire_Unified_FOUND})  # variable defined by FIND(ArrayFire ...)
  OPTION(BUILD_UNIFIED "Build ArrayFire Tests for Unified backend" ON)
    CHECK_AND_CREATE_TESTS(unified ${ArrayFire_Unified_LIBRARIES} "${GTEST_LIBRARIES}" "${CMAKE_DL_LIBS}")
ELSEIF(TARGET af)        # variable defined by the ArrayFire build tree
    CHECK_AND_CREATE_TESTS(unified af "${GTEST_LIBRARIES}" "${CMAKE_DL_LIBS}")
ELSE()
    MESSAGE(STATUS "TESTS: UNIFIED backend is OFF. af was not found.")
ENDIF()
