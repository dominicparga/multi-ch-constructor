cmake_minimum_required(VERSION 3.5)
project(multi-ch-constructor)
add_compile_options(-Wall -Wextra -Wpedantic -std=c++17 -Wno-register)

# Set a default build type if none was specified
set(default_build_type "Release")
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake" "${CMAKE_SOURCE_DIR}/externals/sanitizers-cmake/cmake" ${CMAKE_MODULE_PATH})
include(cotire)

find_package(glpk REQUIRED)

find_package(Boost 1.58 REQUIRED COMPONENTS serialization program_options filesystem iostreams graph)
find_package (Threads)

find_package(glpk REQUIRED) 
find_package(Sanitizers) 
sanitizer_add_blacklist_file("sanitizer_blacklist.txt")


add_library(process INTERFACE)
target_include_directories(process SYSTEM INTERFACE vendor/boost-process)

# Actual Implementation of project in static library
file(GLOB lib_src src/multi_lib/*.cpp )
add_library(multi_lib STATIC ${lib_src})
target_include_directories(multi_lib PUBLIC src/multi_lib)
target_link_libraries(multi_lib ${GLPK_LIBRARIES})
target_link_libraries(multi_lib ${Boost_LIBRARIES})
target_link_libraries(multi_lib -lm)
target_link_libraries(multi_lib -ldl)
target_link_libraries(multi_lib ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(multi_lib process)
add_sanitizers(multi_lib)


# Set graph's dimension to 4 when fresh
set(GRAPH_DIM 4 CACHE STRING "The graph-metrics' dimension")
message("Using GRAPH_DIM=${GRAPH_DIM}")
add_compile_definitions(GRAPH_DIM=${GRAPH_DIM})

# Set graph's edges' cost-accuracy to 10e-6
set(COST_ACCURACY 0.000001 CACHE STRING "The graph-edges' cost-accuracy")
message("Using COST_ACCURACY=${COST_ACCURACY}")
add_compile_definitions(COST_ACCURACY=${COST_ACCURACY})

# Executable of project links against lib
add_executable(multi-ch src/main.cpp)
target_link_libraries(multi-ch multi_lib)
add_sanitizers(multi-ch)

add_executable(multi_lp src/lpsolver.cpp)
target_link_libraries(multi_lp ${GLPK_LIBRARIES})

cotire(multi_lib multi-ch)

# Definition of Testing library catch
add_library(catch INTERFACE)
target_include_directories(catch INTERFACE vendor/catch)

file(GLOB test_src test/*.cpp)
add_executable(ch_test ${test_src})
target_link_libraries(ch_test multi_lib catch)
add_sanitizers(ch_test)
