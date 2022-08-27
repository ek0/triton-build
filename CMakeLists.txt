cmake_minimum_required(VERSION 3.24)
include(FetchContent)
include(ExternalProject)

project(triton-runner)

#
# Generating the Capstone engine project, which is required by Triton
#
set(CAPSTONE_BUILD_SHARED OFF CACHE BOOL "Disable shared library build" FORCE)
set(CAPSTONE_BUILD_TESTS OFF CACHE BOOL "Disable test build" FORCE)
set(CAPSTONE_BUILD_CSTOOL OFF CACHE BOOL "Disable tools build" FORCE)
# TODO: Fine tune architectures?
FetchContent_Declare(
  capstone
  GIT_REPOSITORY https://github.com/capstone-engine/capstone.git
  GIT_TAG        1d230532840a37ac032c6ab80128238fc930c6c1 # 4.0.2
  OVERRIDE_FIND_PACKAGE
)

FetchContent_MakeAvailable(capstone)

set(CAPSTONE_INCLUDE_DIRS "${capstone_SOURCE_DIR}/include")

set_property(TARGET capstone-static PROPERTY
  MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")


#
# Generating the triton project
#
ExternalProject_Add(
	triton_ep
	GIT_REPOSITORY https://github.com/JonathanSalwan/Triton.git
	GIT_TAG        b9fb2e7b586e7e7ee39ef41e2afed8de1a63f29a
	CMAKE_ARGS     -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR> -DZ3_LIBRARY_DIRS=${CMAKE_BINARY_DIR}/z3/lib/libz3.lib -DZ3_INCLUDE_DIRS=${CMAKE_BINARY_DIR}/z3/include -DCAPSTONE_INCLUDE_DIRS=${CAPSTONE_INCLUDE_DIRS} -DCAPSTONE_LIBRARIES=${capstone_BUILD_DIR}/capstone.lib
	CMAKE_ARGS     -DBOOST_INTERFACE=OFF -DBUILD_EXAMPLES=OFF -DPYTHON_BINDINGS=OFF -DENABLE_TEST=OFF -DBUILD_SHARED_LIBS=OFF -DMSVC_STATIC=OFF
	PREFIX "${CMAKE_BINARY_DIR}/triton"
	BUILD_BYPRODUCTS "<BINARY_DIR>/triton.lib"
)

add_library(tritonlib STATIC IMPORTED GLOBAL)

set_property(TARGET tritonlib PROPERTY
  MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

set_property(TARGET tritonlib PROPERTY
  IMPORTED_LOCATION "${CMAKE_BINARY_DIR}/triton/bin/triton.lib") # FIXME

set_property(TARGET tritonlib PROPERTY
  INCLUDE_DIRECTORIES "${CMAKE_BINARY_DIR}/triton/src/triton/src/libtriton/includes")

#
# Generating Z3 project, which is required by triton
#
ExternalProject_Add(
	z3_ep
	GIT_REPOSITORY https://github.com/Z3Prover/z3.git
	GIT_TAG        19da3c7086dcf5cd0c78a82db9f99f5ef295b8ff # 4.11.0
	CMAKE_GENERATOR "Visual Studio 17 2022"
	CMAKE_GENERATOR_PLATFORM "x64"
	CMAKE_ARGS -DZ3_ENABLE_EXAMPLE_TARGETS=OFF -DZ3_BUILD_LIBZ3_SHARED=OFF -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
	PREFIX "${CMAKE_BINARY_DIR}/z3"
	BUILD_BYPRODUCTS "<BINARY_DIR>/libz3.lib"
)

add_library(z3lib STATIC IMPORTED GLOBAL)

set_property(TARGET z3lib PROPERTY
  IMPORTED_LOCATION "${CMAKE_BINARY_DIR}/z3/lib/libz3.lib")

set_property(TARGET z3lib PROPERTY
  INCLUDE_DIRECTORIES "${CMAKE_BINARY_DIR}/z3/include")

set_property(TARGET z3lib PROPERTY
  MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

#ExternalProject_Add_StepDependencies(triton build z3)
ExternalProject_Add_StepDependencies(triton_ep install z3_ep)

add_library(triton-all INTERFACE)
target_link_libraries(triton-all INTERFACE z3lib tritonlib capstone-static)