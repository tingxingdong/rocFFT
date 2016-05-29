# ########################################################################
# Copyright 2016 Advanced Micro Devices, Inc.
# ########################################################################

# We require 2.8.12 for linking libraries we specify PRIVATE, INTERFACE and
# PUBLIC keywords
cmake_minimum_required( VERSION 2.8.12 )

if( CMAKE_GENERATOR MATCHES "NMake" )
	option( NMAKE_COMPILE_VERBOSE "Print VERBOSE compile/link msgs to the console" OFF )
	if( NMAKE_COMPILE_VERBOSE )
		set( CMAKE_START_TEMP_FILE "" )
		set( CMAKE_END_TEMP_FILE "" )
		set( CMAKE_VERBOSE_MAKEFILE 1 )
	endif( )
endif( )

# This has to be initialized before the project() command appears
# Set the default of CMAKE_BUILD_TYPE to be release, unless user specifies with -D.  MSVC_IDE does not use CMAKE_BUILD_TYPE
if( NOT MSVC_IDE AND NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." )
endif()

# Check if cmake supports the new VERSION tag for project() commands
# rocfft becomes the name of the project with a particular version
if( POLICY CMP0048 )
	cmake_policy( SET CMP0048 NEW )
	project( rocfft VERSION 0.0.0.1 LANGUAGES CXX C )
else( )
	project( rocfft CXX C )
	# Define a version for the code
	if( NOT DEFINED rocfft_VERSION_MAJOR )
		set( rocfft_VERSION_MAJOR 0 )
	endif( )

	if( NOT DEFINED rocfft_VERSION_MINOR )
		set( rocfft_VERSION_MINOR 0 )
	endif( )

	if( NOT DEFINED rocfft_VERSION_PATCH )
		set( rocfft_VERSION_PATCH 0 )
	endif( )

	if( NOT DEFINED rocfft_VERSION_TWEAK )
		set( rocfft_VERSION_TWEAK 1 )
	endif( )

	set( rocfft_VERSION "${rocfft_VERSION_MAJOR}.${rocfft_VERSION_MINOR}.${rocfft_VERSION_PATCH}.${rocfft_VERSION_TWEAK}")
endif( )

if(PLATFORM_NAME STREQUAL "NVIDIA")
    find_package(CUDA REQUIRED)
endif( )

# MACOSX_RPATH is enabled by default policy
if( POLICY CMP0042 )
	cmake_policy( SET CMP0042 NEW )
endif( )

message( STATUS "rocfft_VERSION= ${rocfft_VERSION}" )
message( STATUS "CMAKE_BUILD_TYPE= ${CMAKE_BUILD_TYPE}" )

# This is incremented when the ABI to the library changes
set( rocfft_SOVERSION 1 )

list( APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake )

# This option only works for make/nmake and the ninja generators, but no reason it shouldn't be on all the time
# This tells cmake to create a compile_commands.json file that can be used with clang tooling or vim
set( CMAKE_EXPORT_COMPILE_COMMANDS ON )

# On windows, it's convenient to change the default install prefix such that it does NOT point to 'program files'
# Need to check out CMAKE_RUNTIME_OUTPUT_DIRECTORY variable, and see if that eliminates the need to modify install path
if( WIN32 AND CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT )
	set( CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}/package" CACHE PATH "Install path prefix, prepended onto install directories" FORCE )
endif( )

include( cmake/build-options.cmake )
include( GNUInstallDirs )

# Set common compile and link options
if( MSVC )
	# Following options for nMake
	message( STATUS "Detected MSVS Ver: " ${MSVC_VERSION} )

	# CMake uses huge stack frames for windows, we would like to remove.
	string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}" )
	string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS}" )
	string( REGEX REPLACE "/STACK:[0-9]+" "" CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS}" )

elseif( CMAKE_COMPILER_IS_GNUCXX OR ( CMAKE_CXX_COMPILER_ID MATCHES "Clang" ) )
	message( STATUS "Detected GNU or Clang compiler collection" )

	if( BUILD_64 )
		set( CMAKE_CXX_FLAGS "-m64 ${CMAKE_CXX_FLAGS}" )
		set( CMAKE_C_FLAGS "-m64 ${CMAKE_C_FLAGS}" )
	else( )
		set( CMAKE_CXX_FLAGS "-m32 ${CMAKE_CXX_FLAGS}" )
		set( CMAKE_C_FLAGS "-m32 ${CMAKE_C_FLAGS}" )
	endif( )
else( )
	message( AUTHOR_WARNING "Compiler not recognized.  Using default flags." )
endif( )

# Print out compiler flags for viewing/debug
message( STATUS "CMAKE_CXX_COMPILER flags: " ${CMAKE_CXX_FLAGS} )
message( STATUS "CMAKE_CXX_COMPILER debug flags: " ${CMAKE_CXX_FLAGS_DEBUG} )
message( STATUS "CMAKE_CXX_COMPILER release flags: " ${CMAKE_CXX_FLAGS_RELEASE} )
message( STATUS "CMAKE_CXX_COMPILER relwithdebinfo flags: " ${CMAKE_CXX_FLAGS_RELWITHDEBINFO} )
message( STATUS "CMAKE_EXE_LINKER link flags: " ${CMAKE_EXE_LINKER_FLAGS} )

# configure a header file to pass the CMake version settings to the source, and package the header files in the output archive
configure_file( "${PROJECT_SOURCE_DIR}/include/rocfft-version.h.in" "${PROJECT_BINARY_DIR}/include/rocfft-version.h" )

set( rocfft_headers_public
  include/rocfft.h
  include/rocfft_transpose.h
  ${PROJECT_BINARY_DIR}/include/rocfft-version.h
)

source_group( "Header Files\\Public" FILES ${rocfft_headers_public} )

# Build into subdirectories
add_subdirectory( library )

install( FILES ${rocfft_headers_public}
	DESTINATION
		${CMAKE_INSTALL_INCLUDEDIR} )

# Depending on whether we are building for 64 or 32 bit, construct common paths and names that subdirectories can reference for their use
if( BUILD_64 )
	set( CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${rocfft_VERSION}-${CMAKE_HOST_SYSTEM_NAME}-x64" )
else( )
	set( CPACK_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${rocfft_VERSION}-${CMAKE_HOST_SYSTEM_NAME}-x32" )
endif( )

# The following code is setting variables to control the behavior of CPack to generate our
if( WIN32 )
	set( CPACK_SOURCE_GENERATOR "ZIP" )
	set( CPACK_GENERATOR "ZIP" )
else( )
	set( CPACK_SOURCE_GENERATOR "TGZ" )
	set( CPACK_GENERATOR "TGZ" )
endif( )

set( CPACK_SOURCE_PACKAGE_FILE_NAME "${CMAKE_PROJECT_NAME}-${rocfft_VERSION}-${CMAKE_HOST_SYSTEM_NAME}-Source")
set( CPACK_PACKAGE_VERSION_MAJOR ${rocfft_VERSION_MAJOR} )
set( CPACK_PACKAGE_VERSION_MINOR ${rocfft_VERSION_MINOR} )
set( CPACK_PACKAGE_VERSION_PATCH ${rocfft_VERSION_PATCH} )
set( CPACK_PACKAGE_VERSION_TWEAK ${rocfft_VERSION_TWEAK} )
set( CPACK_PACKAGE_DESCRIPTION_SUMMARY "Radeon Open Compute BLAS library package")
set( CPACK_PACKAGE_VENDOR "AMD")
set( CPACK_SOURCE_IGNORE_FILES "/\\\\.git/;/\\\\.hg/;/\\\\.svn/;" )

# Define all variables that influence CPack before including CPack, such as install targets
include( CPack )