cmake_minimum_required(VERSION 3.24)
include(ExternalProject)

project(triton-runner)

#
# Generating the Capstone engine project, which is required by Triton
#
ExternalProject_Add(
	capstone_ep
	GIT_REPOSITORY		https://github.com/capstone-engine/capstone.git
	GIT_TAG				1d230532840a37ac032c6ab80128238fc930c6c1 # 4.0.2
	CMAKE_ARGS			-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
						-DCAPSTONE_BUILD_SHARED=OFF
						-DCAPSTONE_BUILD_TESTS=OFF
						-DCAPSTONE_BUILD_CSTOOL=OFF
						-DCMAKE_POLICY_DEFAULT_CMP0091:STRING=NEW
						-DCMAKE_MSVC_RUNTIME_LIBRARY:STRING=MultiThreaded$<$<CONFIG:Debug>:Debug>
	PREFIX				"${CMAKE_BINARY_DIR}/capstone"
	BUILD_BYPRODUCTS	"<BINARY_DIR>/capstone.lib"
)

#
# Generating the triton project
#
ExternalProject_Add(
	triton_ep
	GIT_REPOSITORY	https://github.com/JonathanSalwan/Triton.git
	GIT_TAG			b9fb2e7b586e7e7ee39ef41e2afed8de1a63f29a
	CMAKE_ARGS		-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
					-DZ3_LIBRARY_DIRS=${CMAKE_INSTALL_PREFIX}/lib/libz3.lib
					-DZ3_INCLUDE_DIRS=${CMAKE_INSTALL_PREFIX}/include
					-DCAPSTONE_INCLUDE_DIRS=${CMAKE_INSTALL_PREFIX}/include
					-DCAPSTONE_LIBRARIES=${CMAKE_INSTALL_PREFIX}/lib/capstone.lib
					-DBOOST_INTERFACE=OFF
					-DBUILD_EXAMPLES=OFF
					-DPYTHON_BINDINGS=OFF
					-DENABLE_TEST=OFF
					-DBUILD_SHARED_LIBS=OFF
					-DMSVC_STATIC=OFF
					-DCMAKE_POLICY_DEFAULT_CMP0091:STRING=NEW
					-DCMAKE_MSVC_RUNTIME_LIBRARY:STRING=MultiThreaded$<$<CONFIG:Debug>:Debug>
	PREFIX			"${CMAKE_BINARY_DIR}/triton"
	BUILD_BYPRODUCTS "<BINARY_DIR>/triton.lib"
)

#
# Generating Z3 project, which is required by triton
#
ExternalProject_Add(
	z3_ep
	GIT_REPOSITORY				https://github.com/Z3Prover/z3.git
	GIT_TAG						19da3c7086dcf5cd0c78a82db9f99f5ef295b8ff # 4.11.0
	CMAKE_GENERATOR				"Visual Studio 17 2022"
	CMAKE_GENERATOR_PLATFORM	"x64"
	CMAKE_ARGS					-DZ3_ENABLE_EXAMPLE_TARGETS=OFF
								-DZ3_BUILD_LIBZ3_SHARED=OFF
								-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
								-DCMAKE_POLICY_DEFAULT_CMP0091:STRING=NEW
								-DCMAKE_MSVC_RUNTIME_LIBRARY:STRING=MultiThreaded$<$<CONFIG:Debug>:Debug>
	PREFIX						"${CMAKE_BINARY_DIR}/z3"
	BUILD_BYPRODUCTS			"<BINARY_DIR>/libz3.lib"
)

# We don't generate triton.lib until all other targets are installed
ExternalProject_Add_StepDependencies(triton_ep install z3_ep capstone_ep)

# Retrieving the platform lib.exe (ar on linux, libtool on macOS. Unsupported at the moment)
cmake_path(GET CMAKE_LINKER PARENT_PATH LINKER_PARENT_PATH)
message(STATUS "Lib.exe directory: ${LINKER_PARENT_PATH}")

# Merging all libraries together for easier linking in external projects.
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang")
	set(LIB_EXE ${LINKER_PARENT_PATH}/llvm-lib.exe)
	add_custom_target(triton_all ALL
		COMMAND ${LIB_EXE} /OUT:${CMAKE_INSTALL_PREFIX}/lib/triton_all.lib ${CMAKE_INSTALL_PREFIX}/lib/capstone.lib ${CMAKE_INSTALL_PREFIX}/lib/libz3.lib ${CMAKE_INSTALL_PREFIX}/lib/triton.lib
		BYPRODUCTS ${CMAKE_INSTALL_PREFIX}/lib/triton_all.lib
		DEPENDS triton_ep z3_ep capstone_ep)
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "MSVC")
	set(LIB_EXE ${LINKER_PARENT_PATH}/lib.exe)
	add_custom_target(triton_all ALL
		COMMAND ${LIB_EXE} /OUT:${CMAKE_INSTALL_PREFIX}/lib/triton_all.lib ${CMAKE_INSTALL_PREFIX}/lib/capstone.lib ${CMAKE_INSTALL_PREFIX}/lib/libz3.lib ${CMAKE_INSTALL_PREFIX}/lib/triton.lib
		BYPRODUCTS ${CMAKE_INSTALL_PREFIX}/lib/triton_all.lib
		DEPENDS triton_ep z3_ep capstone_ep)
else()
	# Extend here if required for other platforms
	message(WARNING "No suitable lib.exe found, skipping merging...")
endif()