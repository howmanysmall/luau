# This file is part of the Luau programming language and is licensed under MIT License; see LICENSE.txt for details
if(EXT_PLATFORM_STRING)
    include(EXTLuau.cmake)
    return()
endif()

cmake_minimum_required(VERSION 3.0)
project(Luau LANGUAGES CXX)

option(LUAU_BUILD_CLI "Build CLI" ON)
option(LUAU_BUILD_TESTS "Build tests" ON)

add_library(Luau.Ast STATIC)
add_library(Luau.Compiler STATIC)
add_library(Luau.Analysis STATIC)
add_library(Luau.VM STATIC)

if(LUAU_BUILD_CLI)
    add_executable(Luau.Repl.CLI)
    if(NOT EMSCRIPTEN)
        add_executable(Luau.Analyze.CLI)
    else()
        # add -fexceptions for emscripten to allow exceptions to be caught in C++
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions")
    endif()

    # This also adds target `name` on Linux/macOS and `name.exe` on Windows
    set_target_properties(Luau.Repl.CLI PROPERTIES OUTPUT_NAME luau)

    if(NOT EMSCRIPTEN)
        set_target_properties(Luau.Analyze.CLI PROPERTIES OUTPUT_NAME luau-analyze)
    endif()
endif()

if(LUAU_BUILD_TESTS AND NOT EMSCRIPTEN)
    add_executable(Luau.UnitTest)
    add_executable(Luau.Conformance)
endif()

include(Sources.cmake)

target_compile_features(Luau.Ast PUBLIC cxx_std_17)
target_include_directories(Luau.Ast PUBLIC Ast/include)

target_compile_features(Luau.Compiler PUBLIC cxx_std_17)
target_include_directories(Luau.Compiler PUBLIC Compiler/include)
target_link_libraries(Luau.Compiler PUBLIC Luau.Ast)

target_compile_features(Luau.Analysis PUBLIC cxx_std_17)
target_include_directories(Luau.Analysis PUBLIC Analysis/include)
target_link_libraries(Luau.Analysis PUBLIC Luau.Ast)

target_compile_features(Luau.VM PRIVATE cxx_std_11)
target_include_directories(Luau.VM PUBLIC VM/include)

set(LUAU_OPTIONS)

if(MSVC)
    list(APPEND LUAU_OPTIONS /D_CRT_SECURE_NO_WARNINGS) # We need to use the portable CRT functions.
    list(APPEND LUAU_OPTIONS /WX) # Warnings are errors
    list(APPEND LUAU_OPTIONS /MP) # Distribute single project compilation across multiple cores
else()
    list(APPEND LUAU_OPTIONS -Wall) # All warnings
    list(APPEND LUAU_OPTIONS -Werror) # Warnings are errors
endif()

target_compile_options(Luau.Ast PRIVATE ${LUAU_OPTIONS})
target_compile_options(Luau.Analysis PRIVATE ${LUAU_OPTIONS})
target_compile_options(Luau.VM PRIVATE ${LUAU_OPTIONS})

if(LUAU_BUILD_CLI)
    target_compile_options(Luau.Repl.CLI PRIVATE ${LUAU_OPTIONS})

    if(NOT EMSCRIPTEN)
        target_compile_options(Luau.Analyze.CLI PRIVATE ${LUAU_OPTIONS})
    endif()

    target_include_directories(Luau.Repl.CLI PRIVATE extern)
    target_link_libraries(Luau.Repl.CLI PRIVATE Luau.Compiler Luau.VM)

    if(UNIX)
        target_link_libraries(Luau.Repl.CLI PRIVATE pthread)
    endif()

    if(NOT EMSCRIPTEN)
        target_link_libraries(Luau.Analyze.CLI PRIVATE Luau.Analysis)
    endif()

    if(EMSCRIPTEN)
        # declare exported functions to emscripten
        target_link_options(Luau.Repl.CLI PRIVATE -sEXPORTED_FUNCTIONS=['_executeScript'] -sEXPORTED_RUNTIME_METHODS=['ccall','cwrap'] -fexceptions)

        # custom output directory for wasm + js file
        set_target_properties(Luau.Repl.CLI PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/docs/assets/luau)
    endif()
endif()

if(LUAU_BUILD_TESTS AND NOT EMSCRIPTEN)
    target_compile_options(Luau.UnitTest PRIVATE ${LUAU_OPTIONS})
    target_include_directories(Luau.UnitTest PRIVATE extern)
    target_link_libraries(Luau.UnitTest PRIVATE Luau.Analysis Luau.Compiler)

    target_compile_options(Luau.Conformance PRIVATE ${LUAU_OPTIONS})
    target_include_directories(Luau.Conformance PRIVATE extern)
    target_link_libraries(Luau.Conformance PRIVATE Luau.Analysis Luau.Compiler Luau.VM)
endif()
