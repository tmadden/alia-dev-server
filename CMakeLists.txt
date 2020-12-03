cmake_minimum_required(VERSION 3.14)
project(alia-dev-server)

# Download/include CMake/Conan integration.
if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/conan.cmake")
    message(STATUS "Downloading conan.cmake from https://github.com/conan-io/cmake-conan")
    file(DOWNLOAD "https://github.com/conan-io/cmake-conan/raw/v0.15/conan.cmake"
                  "${CMAKE_CURRENT_BINARY_DIR}/conan.cmake"
                  TLS_VERIFY ON)
endif()
include(${CMAKE_CURRENT_BINARY_DIR}/conan.cmake)

# Invoke Conan.
conan_cmake_run(
    REQUIRES
        asio/1.18.0
        websocketpp/0.8.2
        catch2/2.13.3
    BASIC_SETUP
    BUILD missing)

# Set build options for instrumenting test coverage.
enable_testing()
if(IS_CLANG AND CMAKE_BUILD_TYPE STREQUAL "Debug")
    message(STATUS "Enabling gcov support")
    add_compile_options(-DLLVM_USE_LINKER=gold -fprofile-instr-generate
                        -fcoverage-mapping)
    string(APPEND CMAKE_EXE_LINKER_FLAGS
           " -fprofile-instr-generate -fcoverage-mapping")
endif()

# Create an object library with the src/ directory.
file(GLOB_RECURSE srcs CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")
file(GLOB_RECURSE headers CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.hpp")
add_library(dev-server-lib OBJECT ${srcs} ${headers})
target_include_directories(dev-server-lib PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/src)
target_compile_definitions(
    dev-server-lib PUBLIC ${CONAN_DEFINES} ${CONAN_COMPILE_DEFINITIONS})
target_link_directories(dev-server-lib PUBLIC ${CONAN_LIB_DIRS})
target_link_libraries(dev-server-lib PUBLIC ${CONAN_LIBS})

# Add the server executable.
add_executable(alia-dev-server server.cpp)
target_link_libraries(alia-dev-server PUBLIC dev-server-lib)

# Add the unit test runner.
file(GLOB_RECURSE UNIT_TEST_FILES CONFIGURE_DEPENDS "unit_tests/*.cpp")
add_executable(unit-test-runner ${UNIT_TEST_FILES})
target_link_libraries(unit-test-runner PUBLIC dev-server-lib)
target_include_directories(unit-test-runner
    PRIVATE ${PROJECT_SOURCE_DIR}/unit_tests)

# Add a target for running the unit tests.
add_custom_target(
    unit-tests
    # Create a fresh 'unit-testing' directory within the build dir and run the
    # tests with that.
    COMMAND ${CMAKE_COMMAND} -E remove_directory unit-testing
    COMMAND ${CMAKE_COMMAND} -E make_directory unit-testing
    COMMAND ${CMAKE_COMMAND} -E chdir unit-testing ${CMAKE_COMMAND}
                             $<TARGET_FILE:unit-test-runner>
    WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
    DEPENDS unit-test-runner)

# On Linux debug builds, the proper CMake test associated with the unit tests
# includes test coverage reporting.
if(IS_CLANG AND CMAKE_BUILD_TYPE STREQUAL "Debug")
    string(REPLACE clang llvm-profdata LLVM_PROFDATA $ENV{CC})
    string(REPLACE clang llvm-cov LLVM_COV $ENV{CC})
    add_custom_target(
        unit-test-coverage
        COMMAND ${CMAKE_COMMAND} --build . --target unit-tests
        COMMAND ${LLVM_PROFDATA} merge -sparse unit-testing/default.profraw
                -o default.profdata
        COMMAND ${LLVM_COV} show -instr-profile=default.profdata
                $<TARGET_FILE:unit-test-runner>
                >${PROJECT_SOURCE_DIR}/coverage.txt
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
    add_test(
        NAME unit-test-coverage
        COMMAND ${CMAKE_COMMAND} --build . --target unit-test-coverage
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
else()
    add_test(
        NAME unit-tests
        COMMAND ${CMAKE_COMMAND} --build . --target unit-tests
        WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
endif()
