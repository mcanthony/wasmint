cmake_minimum_required(VERSION 2.8)
project(wasmint)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${wasmint_SOURCE_DIR}/cmake")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")

add_subdirectory(wasm-module)

###########################
#       Plugins           #
###########################

####################
#       SDL        #
####################

find_package(SDL2)
find_package(SDL2_Image)

if (SDL2_FOUND AND SDL2_IMAGE_FOUND)
    include_directories(${SDL2_INCLUDE_DIR})
    add_definitions(-DWASMINT_HAS_SDL)
endif()

###########################
#      Common code        #
###########################

include_directories(wasm-module/src)
include_directories (src)

add_library(core
    src/interpreter/Thread.cpp
    src/interpreter/FunctionState.cpp
    src/interpreter/InstructionState.cpp
    src/interpreter/MachineState.cpp
    src/interpreter/InstructionExecutor.cpp
    src/interpreter/Heap.cpp
    src/interpreter/StepResult.cpp
    src/builtins/StdioModule.cpp
    src/builtins/SDLModule.cpp
        src/builtins/AssertModule.cpp src/builtins/AssertModule.h)


###########################
#      Interpreter        #
###########################

add_executable(wasmint main.cpp)
target_link_libraries(wasmint core wasm-module)

if (SDL2_FOUND AND SDL2_IMAGE_FOUND)
    target_link_libraries(wasmint ${SDL2_LIBRARY}
                                  ${SDL2_IMAGE_LIBRARIES})
endif()

###########################
#        Tests            #
###########################
enable_testing()


####################
#    Unit tests    #
####################

file(GLOB TEST_FILES "tests/*Test.cpp")
foreach(TEST_FILE ${TEST_FILES})
  get_filename_component(BASENAME ${TEST_FILE} NAME_WE)
  add_executable(${BASENAME} ${TEST_FILE})
  target_link_libraries(${BASENAME} core wasm-module)
  add_test(${BASENAME} ${BASENAME})
endforeach()

####################
#  Test programs   #
####################

file(GLOB TEST_WAST_FILES "tests/wast_tests/*.wast")
foreach(TEST_WAST_FILE ${TEST_WAST_FILES})
    get_filename_component(BASENAME ${TEST_WAST_FILE} NAME)
    add_test(NAME ${BASENAME} COMMAND ./wasmint ${TEST_WAST_FILE})
endforeach()

####################
#   Fuzzing libs   #
####################

add_library(fuzzing_common
    tests/fuzzing/common/SExprExecutor.cpp
    tests/fuzzing/common/DataRescuer.cpp
    tests/fuzzing/generation_based/SmartSExprGenerator.cpp
)


####################
# Mutation fuzzer  #
####################

add_executable(mutation_fuzzer tests/fuzzing/mutation_based/mutation_fuzzer.cpp)
target_link_libraries(mutation_fuzzer core wasm-module fuzzing_common)
add_test(NAME mutation_fuzzing COMMAND ./mutation_fuzzer)

####################
# Test suite       #
####################

configure_file(runTestsuite.sh runTestsuite.sh)
configure_file(testValgrind.sh testValgrind.sh COPYONLY)