# CMake config file to build AER
#
# For Linux and Mac, we can build both statically or dynamically. The latter is
# the default. If you want to build an static executable/library, you need to set
# STATIC_LINKING to True, example:
#     out$ cmake -DSTATIC_LINKING=True ..
#
# For Mac, statically linking only happens with user libraries, system libraries cannot
# be linked statically per Apple's indications.

cmake_minimum_required(VERSION 3.6)
file(STRINGS "qiskit/providers/aer/VERSION.txt" VERSION_NUM)
project(qasm_simulator VERSION ${VERSION_NUM} LANGUAGES CXX C)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)
list(APPEND CMAKE_PREFIX_PATH ${CMAKE_SOURCE_DIR}/cmake)

# Warning: Because of a bug on CMake's FindBLAS or (it's not clear whoes fault is)
# libopenblas.a for Ubuntu (maybe others) we need to copy the file:
# cmake/FindBLAS.cmake.fix-static-linking, to the directory were CMake Modules are
# installed in the system, but with the name: FindBLAS.cmake
option(STATIC_LINKING "Specify if we want statically link the executable (for
						redistribution mainly)" FALSE)
option(BUILD_TESTS "Specify whether we want to build tests or not" FALSE)

include(CTest)
include(compiler_utils)
include(Linter)
include(findBLASInSpecificPath)

# Get version information
get_version(${VERSION_NUM})
configure_file("${PROJECT_SOURCE_DIR}/contrib/standalone/version.hpp.in"
               "${PROJECT_SOURCE_DIR}/contrib/standalone/version.hpp")

set(AER_SIMULATOR_CPP_SRC_DIR "${PROJECT_SOURCE_DIR}/src")
set(AER_SIMULATOR_CPP_MAIN
    "${PROJECT_SOURCE_DIR}/contrib/standalone/qasm_simulator.cpp")
set(AER_SIMULATOR_CPP_EXTERNAL_LIBS
	"${AER_SIMULATOR_CPP_SRC_DIR}/third-party/headers"
	"${AER_SIMULATOR_CPP_SRC_DIR}/third-party/macos/lib"
	"${AER_SIMULATOR_CPP_SRC_DIR}/third-party/win64/lib"
	"${AER_SIMULATOR_CPP_SRC_DIR}/third-party/linux/lib"
	"${USER_LIB_PATH}")

set(AER_COMPILER_DEFINITIONS "")

# TODO: We may want to change the prefix path for all the environments
if(WIN32)
	set(CMAKE_PREFIX_PATH "${AER_SIMULATOR_CPP_EXTERNAL_LIBS} ${CMAKE_PREFIX_PATH}")
endif()

# Adding support for CCache
find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
	set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
	set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif(CCACHE_FOUND)

# Set default build type to Release with Debug Symbols
IF(NOT CMAKE_BUILD_TYPE)
	SET(CMAKE_BUILD_TYPE Release CACHE STRING
		"Choose the type of build, options are: Debug Release"
		FORCE)
ENDIF(NOT CMAKE_BUILD_TYPE)

if(APPLE AND CMAKE_CXX_COMPILER_ID MATCHES "Clang")
	# In order to build for MacOSX 10.9 and above with Clang, we need to force the "deployment target" to 10.9
	# and force using libc++ instead of the default for this target: libstdc++ otherwise we could not
	# use C++11/14
	set(CMAKE_OSX_DEPLOYMENT_TARGET "10.9" CACHE STRING "" FORCE)
	enable_cxx_compiler_flag_if_supported("-stdlib=libc++")
endif()

if(STATIC_LINKING)
	if(APPLE AND CMAKE_CXX_COMPILER_ID MATCHES "Clang")
		message(WARNING "Clang on MacOS doesn't support some -static-* flags. Switching to dyn compilation...")
		unset(STATIC_LINKING)
	else()
	    # MacOS compilers don't support -static flag either
	    if(NOT APPLE)
	        enable_cxx_compiler_flag_if_supported("-static")
	    endif()
	    # This is enough to build a semi-static executable on Mac
	    enable_cxx_compiler_flag_if_supported("-static-libgcc")
	    enable_cxx_compiler_flag_if_supported("-static-libstdc++")
	endif()
endif()

if(NOT MSVC)
	enable_cxx_compiler_flag_if_supported("-ffast-math")
	if(CMAKE_SYSTEM_PROCESSOR MATCHES "ppc64le")
		# PowerPC builds are not meant to be redistributable, we build them
		# in place, so we can have CPU = native.
		enable_cxx_compiler_flag_if_supported("-mcpu=native")
	endif()
	# Warnings and Errors
	enable_cxx_compiler_flag_if_supported("-pedantic")
	enable_cxx_compiler_flag_if_supported("-Wall")
	enable_cxx_compiler_flag_if_supported("-Wfloat-equal")
	enable_cxx_compiler_flag_if_supported("-Wundef")
	enable_cxx_compiler_flag_if_supported("-Wcast-align")
	enable_cxx_compiler_flag_if_supported("-Wwrite-strings")
	enable_cxx_compiler_flag_if_supported("-Wmissing-declarations")
	enable_cxx_compiler_flag_if_supported("-Wredundant-decls")
	enable_cxx_compiler_flag_if_supported("-Wshadow")
	enable_cxx_compiler_flag_if_supported("-Woverloaded-virtual")
endif()

if(STATIC_LINKING)
    SET(CMAKE_FIND_LIBRARY_SUFFIXES .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
    if(WIN32)
        SET(CMAKE_FIND_LIBRARY_SUFFIXES .lib ${CMAKE_FIND_LIBRARY_SUFFIXES})
    endif()
endif()

#
# Looking for external libraries
#
message(STATUS "Looking for OpenMP support...")
find_package(OpenMP QUIET)
# This is a hack for building with Apple's LLVM, which doesn't support OpenMP yet
# so we need to link with an external library: libomp.
# NOTE: CMake >= 3.12.0 doesn't need this hack. It will just find OpenMP in the
# first find_package(OpenMP) call
if(NOT "${OpenMP_FOUND}" OR NOT "${OpenMP_CXX_FOUND}")
	if(APPLE)
		set(OpenMP_CXX_FLAGS "-Xpreprocessor -fopenmp")
		set(OpenMP_CXX_LIB_NAMES "omp")
		set(OpenMP_omp_LIBRARY "${AER_SIMULATOR_CPP_SRC_DIR}/third-party/macos/lib/libomp.dylib")
		include_directories("${AER_SIMULATOR_CPP_SRC_DIR}/third-party/macos/lib")
		set(OPENMP_FOUND TRUE)
	endif()
endif()

if(OPENMP_FOUND)
	set(AER_COMPILER_FLAGS "${AER_COMPILER_FLAGS} ${OpenMP_CXX_FLAGS}")
	set(AER_LINKER_FLAGS "${AER_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS} ${OpenMP_CXX_FLAGS}")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
	if(APPLE)
		# On Apple and clang, we do need to link against the library unless we are building
		# the Terra Addon, see issue: https://github.com/Qiskit/qiskit-aer/issues/1
		if(NOT SKBUILD)
			set(OPENMP_EXTERNAL_LIB "${OpenMP_${OpenMP_CXX_LIB_NAMES}_LIBRARY}")
			message(STATUS "Adding Clang: ${OPENMP_EXTERNAL_LIB}")
		endif()
	endif()
	message(STATUS "OpenMP found!")
	message(STATUS "OpenMP_CXX_FLAGS = ${OpenMP_CXX_FLAGS}")
	message(STATUS "OpenMP_EXE_LINKER_FLAGS = ${OpenMP_EXE_LINKER_FLAGS}")
else()
	message(STATUS "WARNING: No OpenMP support found!")
endif()

message(STATUS "Looking for NLOHMANN Json library...")
set(NLOHMANN_JSON_PATH ${AER_SIMULATOR_CPP_EXTERNAL_LIBS})
find_package(nlohmann_json REQUIRED)

if(STATIC_LINKING)
	message(STATUS "Using static linking with Threads...")
	set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
	set(THREADS_PREFER_PTHREAD_FLAG True)
endif()
find_package(Threads)

if(STATIC_LINKING)
	message(STATUS "Setting BLA_STATIC")
	set(BLA_STATIC TRUE)
endif()

if(BLAS_LIB_PATH)
	find_BLAS_in_specific_path(${BLAS_LIB_PATH})
else()
	if(APPLE)
		message(STATUS  "Looking for Apple BLAS library...")
		set(BLA_VENDOR "Apple")
	else()
		message(STATUS "Looking for OpenBLAS library...")
		set(BLA_VENDOR "OpenBLAS")
	endif()
	if(WIN32)
		message(STATUS "Uncompressing OpenBLAS static library...")
		execute_process(COMMAND ${CMAKE_COMMAND} -E tar "xvfj" "${AER_SIMULATOR_CPP_SRC_DIR}/third-party/win64/lib/openblas.7z" WORKING_DIRECTORY  "${AER_SIMULATOR_CPP_SRC_DIR}/third-party/win64/lib/")
	endif()
	find_package(BLAS QUIET)
	if(NOT BLAS_FOUND)
		message(STATUS "OpenBLAS not found. Looking for any other BLAS library...")
		unset(BLA_VENDOR)
		find_package(BLAS REQUIRED)
	endif()
endif()

message(STATUS "BLAS library found: ${BLAS_LIBRARIES}")

message(STATUS "Looking for spdlog library...")
find_package(spdlog QUIET)
if(spdlog_FOUND)
	message(STATUS "spdlog found.")
	set(SPDLOG_LIB spdlog::spdlog)
else()
	message(STATUS "spdlog not found.")
	set(SPDLOG_LIB "")
endif()

# Windows don't have a DL library (dlopen, dlclose, etc...)
if(NOT MSVC)
	find_library(DL_LIB NAMES dl)
	if(${DL_LIB} MATCHES "DL_LIB-NOTFOUND")
		message(FATAL_ERROR "No dl lib found")
	endif()
endif()

# GPU support through Thrust library (CPU fallbacks)
# Defaults to TRUE, but if no Thrust backend has been defined, will set to FALSE
set(AER_THRUST_SUPPORTED TRUE)
if(AER_THRUST_SUPPORTED)
	if(AER_THRUST_BACKEND STREQUAL "CUDA")
		message(STATUS "Thrust library: Looking for CUDA backend...")
		find_package(CUDA REQUIRED)
		message(STATUS "Thrust library: CUDA found!")
		cuda_select_nvcc_arch_flags(AER_CUDA_ARCH Auto)
		set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -ccbin ${CMAKE_CXX_COMPILER} ${AER_CUDA_ARCH} -DAER_THRUST_CUDA -std=c++14 -I${AER_SIMULATOR_CPP_SRC_DIR} -isystem ${AER_SIMULATOR_CPP_SRC_DIR}/third-party/headers -use_fast_math --expt-extended-lambda")
		set_source_files_properties(${AER_SIMULATOR_CPP_MAIN} PROPERTIES
			CUDA_SOURCE_PROPERTY_FORMAT OBJ)
		set(AER_COMPILER_DEFINITIONS ${AER_COMPILER_DEFINITIONS} THRUST_DEVICE_SYSTEM=THRUST_DEVICE_SYSTEM_CUDA)
		set(THRUST_DEPENDANT_LIBS "")
	elseif(AER_THRUST_BACKEND STREQUAL "TBB")
		message(STATUS "Thrust library: Looking for TBB backend...")
		find_package(TBB REQUIRED)
		message(STATUS "TBB Support found!")
		get_thrust_source_code()
		set(THRUST_DEPENDANT_LIBS ${TBB_LIBRARIES})
		set(AER_COMPILER_DEFINITIONS ${AER_COMPILER_DEFINITIONS} THRUST_DEVICE_SYSTEM=THRUST_DEVICE_SYSTEM_TBB AER_THRUST_CPU=TRUE)
	elseif(AER_THRUST_BACKEND STREQUAL "OMP")
		message(STATUS "Thrust library: Setting OMP backend")
		if(NOT OPENMP_FOUND)
			message(FATAL_ERROR "There's no OMP support. We cannot set Thrust backend to OMP!!")
		endif()
		get_thrust_source_code()
		set(AER_COMPILER_DEFINITIONS ${AER_COMPILER_DEFINITIONS} THRUST_DEVICE_SYSTEM=THRUST_DEVICE_SYSTEM_OMP AER_THRUST_CPU=TRUE)
		# We don't need to add OMP because it's already an AER dependency
		set(THRUST_DEPENDANT_LIBS "")
	else()
		message(STATUS "No Thrust supported backend")
		set(AER_THRUST_SUPPORTED FALSE)
	endif()
endif()

if(AER_THRUST_SUPPORTED)
	set(AER_COMPILER_DEFINITIONS ${AER_COMPILER_DEFINITIONS} AER_THRUST_SUPPORTED=TRUE)
else()
	message(STATUS "No Thrust support enabled")
endif()



# Set dependent libraries
set(AER_LIBRARIES
	${OPENMP_EXTERNAL_LIB}
	${BLAS_LIBRARIES}
	nlohmann_json
	Threads::Threads
	${SPDLOG_LIB}
	${DL_LIB}
	${THRUST_DEPENDANT_LIBS})

# Cython build is only enabled if building through scikit-build.
if(SKBUILD) # Terra Addon build
	add_muparserx_lib()

	add_subdirectory(qiskit/providers/aer/pulse/qutip_extra_lite/cy)
	add_subdirectory(qiskit/providers/aer/backends/wrappers)
	add_subdirectory(src/open_pulse)
else() # Standalone build
	set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
	if(CUDA_FOUND)
		cuda_add_executable(qasm_simulator ${AER_SIMULATOR_CPP_MAIN})
		target_link_libraries(qasm_simulator ${AER_LIBRARIES})
	else(CUDA_FOUND)
		add_executable(qasm_simulator ${AER_SIMULATOR_CPP_MAIN})
		target_link_libraries(qasm_simulator PRIVATE ${AER_LIBRARIES})
	endif(CUDA_FOUND)
	set_target_properties(qasm_simulator PROPERTIES
		LINKER_LANGUAGE CXX
		CXX_STANDARD 14
		COMPILE_FLAGS ${AER_COMPILER_FLAGS}
		LINK_FLAGS ${AER_LINKER_FLAGS}
		RUNTIME_OUTPUT_DIRECTORY_DEBUG Debug
		RUNTIME_OUTPUT_DIRECTORY_RELEASE Release)
	target_include_directories(qasm_simulator
		PRIVATE ${AER_SIMULATOR_CPP_SRC_DIR}
		PRIVATE ${AER_SIMULATOR_CPP_EXTERNAL_LIBS})
	target_compile_definitions(qasm_simulator
		PRIVATE ${AER_COMPILER_DEFINITIONS})
	install(TARGETS qasm_simulator DESTINATION bin)

	# Linter
	# This will add the linter as part of the compiling build target
	#add_linter(qasm_simulator)
endif()

# Tests
if(BUILD_TESTS)
	# These tests were meant to be as an example, or template for C++ specific
	# code, but they're not being maintained, so we are disabling them until we
	# have real C++ tests in the codebase.
	# add_subdirectory(test)
endif()
