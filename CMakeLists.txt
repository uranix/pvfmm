project(pvfmm)
cmake_minimum_required(VERSION 2.8)
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})

find_package(MPI REQUIRED)
include_directories(${MPI_INCLUDE_PATH})

find_package(OpenMP REQUIRED)

find_package(FFTW)
set(PVFMM_HAVE_FFTW ${FFTW_FOUND})
find_package(FFTWF)
set(PVFMM_HAVE_FFTWF ${FFTWF_FOUND})

if (ENABLE_CUDA)
	find_package(CUDA)
else()
	message(STATUS "CUDA support disabled")
	set(CUDA_FOUND NO)
endif()

if (CUDA_FOUND)
	set(PVFMM_HAVE_CUDA)
endif()

find_package(BLAS REQUIRED)
find_package(LAPACK REQUIRED)

set(ALLTOALLV_FIX YES) # Simply does not build without
set(PVFMM_PRECOMP_DATA_PATH "\"\"")
configure_file("include/pvfmm_config.h.in" "${CMAKE_CURRENT_BINARY_DIR}/include/pvfmm_config.h" @ONLY)
include_directories("${CMAKE_CURRENT_BINARY_DIR}/include")

include_directories("include")
add_definitions("-Wall -Werror")

set(pvfmm_SOURCES
	src/cheb_utils.cpp
	src/device_wrapper.cpp
	src/fmm_gll.cpp
	src/legendre_rule.cpp
	src/math_utils.cpp
	src/mat_utils.cpp
	src/mem_mgr.cpp
	src/mortonid.cpp
	src/profile.cpp
	src/tree_node.cpp
	)

if (CUDA_FOUND)
	list(APPEND pvfmm_SOURCES src/fmm_pts_gpu.cu)
	cuda_add_library(pvfmm STATIC ${pvfmm_SOURCES})
else()
	add_library(pvfmm STATIC ${pvfmm_SOURCES})
endif()

if (MPI_COMPILE_FLAGS)
	set_target_properties(pvfmm PROPERTIES COMPILE_FLAGS "${MPI_COMPILE_FLAGS}")
endif()

if (OpenMP_CXX_FLAGS)
	set_target_properties(pvfmm PROPERTIES COMPILE_FLAGS "${OpenMP_CXX_FLAGS}")
endif()

# ========================================================== #

if (BUILD_TESTS)
foreach(example example1 example2 fmm_cheb fmm_pts)

	add_executable(${example} "examples/src/${example}.cpp")
	include_directories("examples/include")

	if (MPI_LINK_FLAGS)
		set_target_properties(${example} PROPERTIES LINK_FLAGS "${MPI_LINK_FLAGS}")
	endif()

	if (OpenMP_CXX_FLAGS)
		set_target_properties(${example} PROPERTIES COMPILE_FLAGS "${OpenMP_CXX_FLAGS}")
		# sic!
		set_target_properties(${example} PROPERTIES LINK_FLAGS "${OpenMP_CXX_FLAGS}")
	endif()

	target_link_libraries(${example} pvfmm ${MPI_LIBRARIES} ${BLAS_LIBRARIES} ${LAPACK_LIBRARIES})

	if (FFTW_FOUND)
		target_link_libraries(${example} ${FFTW_LIBRARIES})
	endif()

	if (FFTWF_FOUND)
		target_link_libraries(${example} ${FFTWF_LIBRARIES})
	endif()

	if (CUDA_FOUND)
		target_link_libraries(${example} ${CUDA_CUDART_LIBRARY})
	endif()

endforeach()
endif()
