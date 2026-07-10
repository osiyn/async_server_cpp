cmake_minimum_required(VERSION 3.19)
project(AsyncServer VERSION 0.1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_INCLUDE_CURRENT_DIR ON)

if(NOT DEFINED CMAKE_COLOR_DIAGNOSTICS)
    set(CMAKE_COLOR_DIAGNOSTICS ON)
endif()

option(ENABLE_SANITIZERS "Enable AddressSanitizer and UndefinedBehaviorSanitizer" OFF)
option(ENABLE_ASAN_EXTRA "Enable extra ASAN checks (leak, use-after-return)" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer instead of ASAN/UBSAN" OFF)
option(ENABLE_WERROR "Treat warnings as errors" OFF)
option(ENABLE_TESTS "Build unit tests" ON)
option(ENABLE_BENCHMARKS "Build benchmarks" OFF)

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(-Wall -Wextra -Wpedantic)
    add_compile_options(-Wshadow)
    add_compile_options(-Wnon-virtual-dtor)
    add_compile_options(-Wold-style-cast)
    add_compile_options(-Wcast-align)
    add_compile_options(-Wunused)
    add_compile_options(-Wsign-conversion)
    add_compile_options(-Wconversion)
    
    if(ENABLE_WERROR)
        add_compile_options(-Werror)
    endif()
    
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        add_compile_options(-O0 -g3)
        add_compile_options(-fno-omit-frame-pointer)
    elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
        add_compile_options(-O2 -g)
    elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
        add_compile_options(-O3 -DNDEBUG)
    endif()
    
    if(ENABLE_SANITIZERS AND (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang") AND CMAKE_SYSTEM_NAME STREQUAL "Linux")
        set(SANITIZER_FLAGS "")
        
        if(ENABLE_TSAN)
            set(SANITIZER_FLAGS "-fsanitize=thread")
            message(STATUS "ThreadSanitizer enabled")
            add_compile_options(-fno-omit-frame-pointer)
            add_compile_options(-g)
            set(ENV{TSAN_OPTIONS} "history_size=7:second_deadlock_stack=1")
        else()
            set(SANITIZER_FLAGS "-fsanitize=address,undefined")
            
            if(ENABLE_ASAN_EXTRA)
                set(SANITIZER_FLAGS "${SANITIZER_FLAGS},leak")
                set(SANITIZER_FLAGS "${SANITIZER_FLAGS},pointer-compare")
                set(SANITIZER_FLAGS "${SANITIZER_FLAGS},pointer-subtract")
                message(STATUS "Extra ASAN checks enabled")
            endif()
            
            add_compile_options(-fno-omit-frame-pointer)
            add_compile_options(-fno-optimize-sibling-calls)
            
            message(STATUS "Sanitizers enabled: ${SANITIZER_FLAGS}")
        endif()
        
        add_compile_options(${SANITIZER_FLAGS})
        add_link_options(${SANITIZER_FLAGS})
        add_compile_options(-fno-omit-frame-pointer)
        
    elseif(ENABLE_SANITIZERS)
        message(WARNING "Sanitizers are only supported on Linux with GCC/Clang, disabling...")
        set(ENABLE_SANITIZERS OFF)
    endif()
endif()

if(ENABLE_TESTS)
    include(FetchContent)
    FetchContent_Declare(
        googletest
        URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.tar.gz
    )

    # set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    # set(BUILD_GMOCK ON CACHE BOOL "" FORCE)

    FetchContent_MakeAvailable(googletest)

    if(ENABLE_WERROR)
        target_compile_options(gtest PRIVATE -w)
        target_compile_options(gmock PRIVATE -w)
        target_compile_options(gtest_main PRIVATE -w)
        target_compile_options(gmock_main PRIVATE -w)
    endif()
endif()

if(ENABLE_BENCHMARKS)
    include(FetchContent)
    FetchContent_Declare(
        googlebenchmark
        URL https://github.com/google/benchmark/archive/refs/tags/v1.8.3.tar.gz
    )
    FetchContent_MakeAvailable(googlebenchmark)

    if(ENABLE_WERROR)
        target_compile_options(benchmark PRIVATE -w)
        target_compile_options(benchmark_main PRIVATE -w)
    endif()
endif()

set(SOURCES
    src/core/dummy.cpp
)

add_library(async_core STATIC ${SOURCES})

target_include_directories(async_core PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}/src/core
    ${CMAKE_CURRENT_SOURCE_DIR}/src/http
)

add_executable(async_server src/main.cpp)

target_link_libraries(async_server
    async_core
    pthread
    dl
)

if(ENABLE_SANITIZERS AND CMAKE_SYSTEM_NAME STREQUAL "Linux" AND NOT ENABLE_TSAN)
    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        target_link_libraries(async_server
            asan
            ubsan
        )
    endif()
endif()

if(ENABLE_TESTS)
    enable_testing()
    
    file(GLOB_RECURSE TEST_SOURCES
        "${CMAKE_CURRENT_SOURCE_DIR}/tests/*.cpp"
    )
    
    foreach(TEST_FILE ${TEST_SOURCES})
        get_filename_component(TEST_NAME ${TEST_FILE} NAME_WE)
        add_executable(${TEST_NAME} ${TEST_FILE})
        target_link_libraries(${TEST_NAME}
            async_core
            GTest::gtest_main
            pthread
        )
        add_test(NAME ${TEST_NAME} COMMAND ${TEST_NAME})
    endforeach()
endif()

if(ENABLE_BENCHMARKS)
    file(GLOB_RECURSE BENCH_SOURCES
        "${CMAKE_CURRENT_SOURCE_DIR}/benchmarks/*.cpp"
    )
    
    foreach(BENCH_FILE ${BENCH_SOURCES})
        get_filename_component(BENCH_NAME ${BENCH_FILE} NAME_WE)
        add_executable(${BENCH_NAME} ${BENCH_FILE})
        target_link_libraries(${BENCH_NAME}
            async_core
            benchmark::benchmark
            pthread
        )
    endforeach()
endif()

find_program(CLANG_FORMAT clang-format)
if(CLANG_FORMAT)
    add_custom_target(format
        COMMAND ${CLANG_FORMAT} -i ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp ${CMAKE_CURRENT_SOURCE_DIR}/src/*.hpp
        COMMAND ${CLANG_FORMAT} -i ${CMAKE_CURRENT_SOURCE_DIR}/tests/*.cpp
        COMMENT "Formatting code with clang-format"
    )
endif()

find_program(CLANG_TIDY clang-tidy)
if(CLANG_TIDY)
    add_custom_target(tidy
        COMMAND ${CLANG_TIDY} ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp -- -std=c++17 -I${CMAKE_CURRENT_SOURCE_DIR}/src
        COMMENT "Running clang-tidy"
    )
endif()

add_custom_target(run
    COMMAND ./async_server
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS async_server
    COMMENT "Running async_server"
)

message(STATUS "")
message(STATUS "==========================================")
message(STATUS "AsyncServer Configuration:")
message(STATUS "   Generator:      ${CMAKE_GENERATOR}")
message(STATUS "   Build type:     ${CMAKE_BUILD_TYPE}")
message(STATUS "   C++ Standard:   ${CMAKE_CXX_STANDARD}")
message(STATUS "   Compiler:       ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
if(ENABLE_SANITIZERS)
    if(ENABLE_TSAN)
        message(STATUS "   Sanitizer:      ThreadSanitizer")
    else()
        message(STATUS "   Sanitizer:      Address + Undefined")
    endif()
else()
    message(STATUS "   Sanitizer:      OFF")
endif()
message(STATUS "   Werror:         ${ENABLE_WERROR}")
message(STATUS "   Tests:          ${ENABLE_TESTS}")
message(STATUS "   Benchmarks:     ${ENABLE_BENCHMARKS}")
message(STATUS "==========================================")
message(STATUS "")