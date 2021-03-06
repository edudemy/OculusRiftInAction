# Required for statically linking the MSVC runtime
set(CMAKE_USER_MAKE_RULES_OVERRIDE
   ${CMAKE_CURRENT_SOURCE_DIR}/cmake/c_flag_overrides.cmake)
set(CMAKE_USER_MAKE_RULES_OVERRIDE_CXX
   ${CMAKE_CURRENT_SOURCE_DIR}/cmake/cxx_flag_overrides.cmake)

project(OculusRiftExamples)
cmake_minimum_required(VERSION 2.8)
include (GenerateExportHeader)

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# First for the generic no-config case (e.g. with mingw)
set( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/output )
set( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/output )
set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/output )

# Second, for multi-config builds (e.g. msvc)
foreach( OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES} )
    string( TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG )
    set( CMAKE_RUNTIME_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_BINARY_DIR}/output )
    set( CMAKE_LIBRARY_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_BINARY_DIR}/output )
    set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_BINARY_DIR}/output )
endforeach( OUTPUTCONFIG CMAKE_CONFIGURATION_TYPES )

# Comment out to disable the debug build
set(CMAKE_BUILD_TYPE DEBUG)
set(CMAKE_DEBUG_POSTFIX "d")
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER "CMakeTargets")


macro(ADD_MSVC_PRECOMPILED_HEADER PrecompiledHeader PrecompiledSource SourcesVar)
  IF(MSVC)
    GET_FILENAME_COMPONENT(PrecompiledBasename ${PrecompiledHeader} NAME_WE)
    SET(PrecompiledBinary "${CMAKE_CURRENT_BINARY_DIR}/${PrecompiledBasename}.pch")
    LIST(REMOVE_ITEM ${SourcesVar} "${CMAKE_CURRENT_SOURCE_DIR}/${PrecompiledSource}")
    SET(Sources ${${SourcesVar}})

    SET_SOURCE_FILES_PROPERTIES(${PrecompiledSource}
                                PROPERTIES COMPILE_FLAGS "/Yc\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\""
                                           OBJECT_OUTPUTS "${PrecompiledBinary}")
    SET_SOURCE_FILES_PROPERTIES(${Sources}
                                PROPERTIES COMPILE_FLAGS "/Yu\"${PrecompiledHeader}\" /FI\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\""
                                           OBJECT_DEPENDS "${PrecompiledBinary}")  
    # Add precompiled header to SourcesVar
    LIST(APPEND ${SourcesVar} ${PrecompiledSource})
  ENDIF(MSVC)
ENDMACRO(ADD_MSVC_PRECOMPILED_HEADER)


option(RIFT_BUILD_SCRATCH_EXAMPLES OFF)

string( TOLOWER "${CMAKE_BUILD_TYPE}" BUILD_TYPE_LOWER )

if(BUILD_TYPE_LOWER STREQUAL "debug")
   set(RIFT_DEBUG 1)
endif()

if(RIFT_DEBUG)
   message("Debug build, resources will be loaded from disk")
   set(OVR_DEBUG_BUILD 1)
   add_definitions(-DOVR_DEBUG_BUILD)
else() 
   message("Release build, resources will be loaded from executable/bundle")
endif()

# Verbosity is good, especially if a command fails
set(CMAKE_VERBOSE_MAKEFILE ON)
set(TARGET_ARCHITECTURE "x64")

if(CMAKE_COMPILER_IS_GNUCXX)
    # Ensure we use C++ 11 
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
    # Ensure we generate position independent code 
    if("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86_64")
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fPIC"  )
        set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fPIC"  )    
    endif()
endif()

if(APPLE)
    # Ensure we use C++ 11 
    set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LANGUAGE_STANDARD "c++0x")
    set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LIBRARY "libc++")
    set(CMAKE_XCODE_ATTRIBUTE_GCC_VERSION "com.apple.compilers.llvm.clang.1_0")
#    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x -stdlib=libc++ -g -Wall")
endif()

if(WIN32)
    add_definitions(-DUNICODE -D_UNICODE)
    set(TARGET_ARCHITECTURE "x86")
endif()

###############################################################################
#
# Oculus VR SDK dependency 
#
set(OVR_DIR ${CMAKE_SOURCE_DIR}/libraries/OculusSDK/LibOVR)
# Check for the existence of the LibOVR project.  If it's missing, the 
# submodule probably hasn't been checked out.
if (NOT(EXISTS ${OVR_DIR} AND IS_DIRECTORY ${OVR_DIR}))
    message(FATAL_ERROR "Can't find the OVR library at ${OVR_DIR}.  Perhaps you forgot to check out the git submodules?") 
endif()

# Add debug output from the Oculus SDK
add_definitions(-DOVR_BUILD_DEBUG)

# The primary SDK artifact, a static library for Oculus access 
add_subdirectory(libraries/OculusSDK/LibOVR)
set_target_properties(OculusVR PROPERTIES FOLDER "3rdparty")

# Make sure to set up the OVR include paths
include_directories(libraries/OculusSDK/LibOVR/Include)
include_directories(libraries/OculusSDK/LibOVR/Src)
include_directories(libraries/OculusSDK/LibOVR/Src/Kernel)
include_directories(libraries/OculusSDK/LibOVR/Src/Util)

set(EXAMPLE_LIBS OculusVR )

###############################################################################
#
# Non-Oculus third party dependencies 
#

# Cross platform access to shader functionality
add_definitions(-DGLEW_STATIC)
add_subdirectory(libraries/glew)
set_target_properties(glew PROPERTIES FOLDER "3rdparty")
include_directories(${CMAKE_SOURCE_DIR}/libraries/glew/include)
list(APPEND EXAMPLE_LIBS glew)

# GLFW - Cross platform OpenGL window creation and input handling
set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "Build the GLFW example programs")
set(GLFW_BUILD_TESTS OFF CACHE BOOL "Build the GLFW test programs")
set(GLFW_BUILD_DOCS OFF CACHE BOOL "Build the GLFW documentation")
set(GLFW_INSTALL OFF CACHE BOOL "Generate installation target")
add_subdirectory(libraries/glfw)
set_target_properties(glfw PROPERTIES FOLDER "3rdparty")
include_directories(${CMAKE_SOURCE_DIR}/libraries/glfw/include)
list(APPEND EXAMPLE_LIBS glfw ${GLFW_LIBRARIES})

# OpenCTM - a 3D mesh compression
add_subdirectory(libraries/OpenCTM)
set_target_properties(OpenCTM PROPERTIES FOLDER "3rdparty")
include_directories(${CMAKE_SOURCE_DIR}/libraries/OpenCTM)
list(APPEND EXAMPLE_LIBS OpenCTM)

# Json parsing library
set(JSONCPP_WITH_TESTS OFF CACHE BOOL "Compile and run JsonCpp test executables")
set(JSONCPP_WITH_POST_BUILD_UNITTEST OFF CACHE BOOL "Automatically run unit-tests as a post build step")
add_subdirectory(libraries/jsoncpp)
set_target_properties(jsoncpp PROPERTIES FOLDER "3rdparty")
include_directories(${CMAKE_SOURCE_DIR}/libraries/jsoncpp)
list(APPEND EXAMPLE_LIBS jsoncpp)

# OpenCV is a computer vision library with advanced
# image loading and manipulation functionality
find_package( OpenCV )
if (OpenCV_FOUND) 
    # Augmented reality tracking library
    add_subdirectory(libraries/ARToolkit/lib)
    set_target_properties(ARToolkit PROPERTIES FOLDER "3rdparty")
    include_directories(${CMAKE_SOURCE_DIR}/libraries/ARToolkit/lib/include)
    list(APPEND EXAMPLE_LIBS ARToolkit)

    include_directories(${OpenCV_INCLUDE_DIRS})
    list(APPEND EXAMPLE_LIBS ${OpenCV_LIBS})
    set(HAVE_OPENCV 1)
else()
    # Without OpenCV, we have to fall back on libpng
    # which requires zlib.  For windows and apple we
    # build the library.  For Unix systems we locate 
    # the native package
    if((WIN32 OR APPLE))
        # zlib
        add_subdirectory(libraries/zlib)
        set(ZLIB_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/libraries/zlib)
        include_directories(${CMAKE_BINARY_DIR}/libraries/zlib)
        include_directories(${CMAKE_SOURCE_DIR}/libraries/zlib)
        list(APPEND EXAMPLE_LIBS zlib)

        # png
        set(PNG_STATIC ON CACHE BOOL "Build PNG static library")
        set(PNG_TESTS OFF CACHE BOOL "Build PNG tests")
        set(PNG_SHARED OFF CACHE BOOL "Build PNG shared library")
        add_subdirectory(libraries/libpng)
        include_directories(${CMAKE_SOURCE_DIR}/libraries/libpng)
        include_directories(${CMAKE_BINARY_DIR}/libraries/libpng)
        list(APPEND EXAMPLE_LIBS png)
        set_target_properties(zlib PROPERTIES FOLDER "3rdparty")
        set_target_properties(png PROPERTIES FOLDER "3rdparty")
    else()
        find_package(PNG REQUIRED)
        include_directories(${PNG_INCLUDE_DIR})
        list(APPEND EXAMPLE_LIBS ${PNG_LIBRARIES})
    endif()
endif()

# Vector / matrix header only math library based on
# the syntax of GLSL
include_directories(${CMAKE_SOURCE_DIR}/libraries/glm)

# Boost is a general purpose C++ library
#set(Boost_DEBUG 1)
if (NOT APPLE)
    set(Boost_USE_STATIC_LIBS ON)
    set(Boost_USE_MULTITHREADED ON)
    set(Boost_USE_STATIC_RUNTIME ON)
    find_package( Boost QUIET COMPONENTS filesystem system thread )
    if(Boost_FOUND)
        include_directories(${Boost_INCLUDE_DIRS})
        link_directories(${Boost_LIBRARY_DIR}) 
        list(APPEND EXAMPLE_LIBS ${Boost_LIBRARIES})
        set(HAVE_BOOST 1)
    endif()
endif()

find_package(Sixense)
if(SIXENSE_FOUND)
    if (WIN32)
        file(COPY ${SIXENSE_BINARY}
            DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
        file(COPY ${SIXENSE_UTIL_BINARY}
            DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
    endif()
    include_directories(${SIXENSE_INCLUDE_DIR}) 
    list(APPEND EXAMPLE_LIBS ${SIXENSE_LIBRARIES})
    set(HAVE_SIXENSE 1)
endif()

find_package( SpaceNavigator QUIET)
if(SPNAV_FOUND)
    include_directories(${SPNAV_INCLUDE_DIR}) 
    list(APPEND EXAMPLE_LIBS ${SPNAV_LIBRARIES})
    set(HAVE_SPNAV 1)
endif()

find_package(Threads)
list(APPEND EXAMPLE_LIBS ${CMAKE_THREAD_LIBS_INIT} )

###############################################################################
#
# Common GL code, using only headers.  
#
add_subdirectory(libraries/gl)
set_target_properties(gl PROPERTIES FOLDER "Examples/Shared")
include_directories(${CMAKE_SOURCE_DIR}/libraries/gl)

###############################################################################
#
# Non-C++ resources that are shared by the examples
# (shaders, meshes, images, fonts, etc)
#
# On Windows these will be pulled from a common resource DLL.  
#
# On OSX, they will be embedded in each application bundle, owing to CMakes
# lack of support for shared frameworks
# 
# On Linux or if you enable the debug build, they will be pulled from the
# source location at runtime.  
#
add_subdirectory(resources)
set_target_properties(ExampleResources PROPERTIES FOLDER "Examples/Shared")
include_directories(resources)
include_directories(${CMAKE_BINARY_DIR}/resources)
list(APPEND EXAMPLE_LIBS ExampleResources)

###############################################################################
#
# Shared codebase for all the examples and demos
#
add_subdirectory(source/common)
set_target_properties(ExampleCommon PROPERTIES FOLDER "Examples/Shared")
include_directories(source/common)
include_directories(${CMAKE_BINARY_DIR}/source/common)


###############################################################################
#
# The examples themselves
#
add_subdirectory(source)

