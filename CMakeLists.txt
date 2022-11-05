cmake_minimum_required(VERSION 3.24)
include(ExternalProject)

project(triton-runner)

message("Installing to ${CMAKE_INSTALL_PREFIX}")

#
# Generating the LLVM project, which is required by Triton
#
ExternalProject_Add(
  llvm_ep
  GIT_REPOSITORY     https://github.com/llvm/llvm-project.git
  GIT_TAG            llvmorg-15.0.4
  GIT_SHALLOW        1
  SOURCE_SUBDIR      llvm
  CMAKE_ARGS         -DLLVM_BUILD_TOOLS=OFF
                     -DLLVM_BUILD_EXAMPLES=OFF
                     -DLLVM_BUILD_BENCHMARKS=OFF
                     -DBUILD_SHARED_LIBS=OFF
                     -DLLVM_USE_CRT_RELEASE=MT
                     -DLLVM_USE_CRT_DEBUG=MTd
                     -DLLVM_USE_CRT_RELWITHDEBINFO=MT
                     -DLLVM_USE_CRT_MINSIZEREL=MT
                     -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
                     -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
  PREFIX             "${CMAKE_BINARY_DIR}/llvm-build"
)

#
# Generating the Capstone engine project, which is required by Triton
#
ExternalProject_Add(
    capstone_ep
    GIT_REPOSITORY      https://github.com/capstone-engine/capstone.git
    GIT_TAG             1d230532840a37ac032c6ab80128238fc930c6c1 # 4.0.2
    CMAKE_ARGS          -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
                        -DCAPSTONE_BUILD_SHARED=OFF
                        -DCAPSTONE_BUILD_TESTS=OFF
                        -DCAPSTONE_BUILD_CSTOOL=OFF
                        -DCMAKE_POLICY_DEFAULT_CMP0091:STRING=NEW
                        -DCMAKE_MSVC_RUNTIME_LIBRARY:STRING=MultiThreaded$<$<CONFIG:Debug>:Debug>
                        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    PREFIX             "${CMAKE_BINARY_DIR}/capstone"
    BUILD_BYPRODUCTS    "<BINARY_DIR>/capstone.lib"
)

#
# Generating the triton project
#
ExternalProject_Add(
    triton_ep
    GIT_REPOSITORY   https://github.com/JonathanSalwan/Triton.git
    GIT_TAG          339f16076381f6efdde1a4a156631e0de71e2026
    CMAKE_ARGS       -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
                     #-DZ3_LIBRARY_DIRS=${CMAKE_INSTALL_PREFIX}/lib/libz3.lib
                     #-DZ3_INCLUDE_DIRS=${CMAKE_INSTALL_PREFIX}/include
                     #-DCAPSTONE_INCLUDE_DIRS=${CMAKE_INSTALL_PREFIX}/include
                     #-DCAPSTONE_LIBRARIES=${CMAKE_INSTALL_PREFIX}/lib/capstone.lib
                     -DBOOST_INTERFACE=OFF
                     -DBUILD_EXAMPLES=OFF
                     -DPYTHON_BINDINGS=OFF
                     -DENABLE_TEST=OFF
                     -DBUILD_SHARED_LIBS=OFF
                     -DMSVC_STATIC=ON
                     -DCMAKE_POLICY_DEFAULT_CMP0091:STRING=NEW
                     -DCMAKE_MSVC_RUNTIME_LIBRARY:STRING=MultiThreaded$<$<CONFIG:Debug>:Debug>
                     -DCMAKE_PREFIX_PATH=${CMAKE_INSTALL_PREFIX}
                     -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
                     -DLLVM_INTERFACE=ON
                     -DZ3_INTERFACE=ON
    PREFIX	         "${CMAKE_BINARY_DIR}/triton"
    BUILD_BYPRODUCTS "<BINARY_DIR>/triton.lib"
)

#
# Generating Z3 project, which is required by triton
#
ExternalProject_Add(
    z3_ep
    GIT_REPOSITORY              https://github.com/Z3Prover/z3.git
    GIT_TAG                     8e6f17ebd011a352b3e67fcec03dcd4378a1472c # 4.11.2
    GIT_SHALLOW                 1
    CMAKE_GENERATOR             "Ninja"
    CMAKE_ARGS                  -DZ3_ENABLE_EXAMPLE_TARGETS=OFF
                                -DZ3_BUILD_LIBZ3_SHARED=OFF
                                -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
                                -DCMAKE_POLICY_DEFAULT_CMP0091:STRING=NEW
                                -DCMAKE_MSVC_RUNTIME_LIBRARY:STRING=MultiThreaded$<$<CONFIG:Debug>:Debug>
                                -DCMAKE_CXX_COMPILER=cl # Z3 behaves weirdly with clang-cl...
                                -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
                                -DCMAKE_CXX_FLAGS="/D WIN32 /D _WINDOWS"
    PREFIX                      "${CMAKE_BINARY_DIR}/z3"
    BUILD_BYPRODUCTS            "<BINARY_DIR>/libz3.lib"
)

# We don't generate triton.lib until all other targets are installed
ExternalProject_Add_StepDependencies(triton_ep install z3_ep capstone_ep llvm_ep)