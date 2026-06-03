cmake_minimum_required(VERSION 3.20)

project(FluidX3D LANGUAGES CXX)

if(NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(FLUIDX3D_DEFINES_DIR "${CMAKE_CURRENT_SOURCE_DIR}/examples/benchmark" CACHE PATH "Directory containing the FluidX3D defines.hpp configuration header")
option(FLUIDX3D_BUILD_EXAMPLES "Build FluidX3D example executables" ON)

if(NOT EXISTS "${FLUIDX3D_DEFINES_DIR}/defines.hpp")
	message(FATAL_ERROR "FLUIDX3D_DEFINES_DIR must point to a directory containing defines.hpp")
endif()

find_package(Threads REQUIRED)

add_library(fluidx3d_opencl INTERFACE)
target_include_directories(fluidx3d_opencl INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/src/OpenCL/include")

find_package(OpenCL QUIET)
if(OpenCL_FOUND)
	target_link_libraries(fluidx3d_opencl INTERFACE OpenCL::OpenCL)
elseif(WIN32)
	set(_fluidx3d_opencl_library "${CMAKE_CURRENT_SOURCE_DIR}/src/OpenCL/lib/OpenCL.lib")
	if(NOT EXISTS "${_fluidx3d_opencl_library}")
		message(FATAL_ERROR "OpenCL was not found and bundled library is missing: ${_fluidx3d_opencl_library}")
	endif()
	target_link_libraries(fluidx3d_opencl INTERFACE "${_fluidx3d_opencl_library}")
elseif(APPLE)
	target_link_libraries(fluidx3d_opencl INTERFACE "-framework OpenCL")
elseif(ANDROID)
	target_link_libraries(fluidx3d_opencl INTERFACE OpenCL)
else()
	set(_fluidx3d_opencl_library "${CMAKE_CURRENT_SOURCE_DIR}/src/OpenCL/lib/libOpenCL.so")
	if(NOT EXISTS "${_fluidx3d_opencl_library}")
		message(FATAL_ERROR "OpenCL was not found and bundled library is missing: ${_fluidx3d_opencl_library}")
	endif()
	target_link_libraries(fluidx3d_opencl INTERFACE "${_fluidx3d_opencl_library}")
endif()

add_library(fluidx3d_backend STATIC
	src/info.cpp
	src/kernel.cpp
	src/lbm.cpp
	src/lodepng.cpp
	src/runtime.cpp
	src/shapes.cpp
)
add_library(FluidX3D::backend ALIAS fluidx3d_backend)

target_compile_features(fluidx3d_backend PUBLIC cxx_std_17)
target_include_directories(fluidx3d_backend
	PUBLIC
		"${FLUIDX3D_DEFINES_DIR}"
		"${CMAKE_CURRENT_SOURCE_DIR}/src"
)
target_compile_options(fluidx3d_backend
	PRIVATE
		$<$<CXX_COMPILER_ID:GNU,Clang,AppleClang>:-Wno-comment>
)
target_link_libraries(fluidx3d_backend
	PUBLIC
		Threads::Threads
		fluidx3d_opencl
)

add_library(fluidx3d_visualization STATIC
	src/graphics.cpp
)
add_library(FluidX3D::visualization ALIAS fluidx3d_visualization)

target_compile_features(fluidx3d_visualization PUBLIC cxx_std_17)
target_link_libraries(fluidx3d_visualization PUBLIC fluidx3d_backend)

if(WIN32)
	target_link_libraries(fluidx3d_visualization PUBLIC user32 gdi32)
elseif(UNIX AND NOT ANDROID)
	find_package(X11 QUIET)
	if(X11_FOUND)
		target_link_libraries(fluidx3d_visualization PUBLIC ${X11_LIBRARIES})
		if(X11_Xrandr_FOUND)
			target_link_libraries(fluidx3d_visualization PUBLIC ${X11_Xrandr_LIB})
		endif()
	elseif(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/src/X11/lib/libX11.so")
		target_link_directories(fluidx3d_visualization PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/src/X11/lib")
		target_link_libraries(fluidx3d_visualization PUBLIC X11 Xrandr)
	endif()
endif()

if(FLUIDX3D_BUILD_EXAMPLES)
	add_executable(fluidx3d_benchmark
		examples/benchmark/main.cpp
		examples/benchmark/setup.cpp
	)
	target_link_libraries(fluidx3d_benchmark PRIVATE fluidx3d_visualization)
	set_target_properties(fluidx3d_benchmark
		PROPERTIES
			OUTPUT_NAME FluidX3D
			RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
	)
endif()
