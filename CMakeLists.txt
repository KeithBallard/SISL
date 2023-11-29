cmake_minimum_required(VERSION 3.8)

cmake_policy(SET CMP0048 NEW)
project(SISL VERSION 1.0.0 LANGUAGES C CXX)

# Organize the project in folders (VS only?)
SET_PROPERTY(GLOBAL PROPERTY USE_FOLDERS ON)

# Include directories

INCLUDE_DIRECTORIES(
  ${sisl_SOURCE_DIR}/include
  )

#########################################################################################
# Make the sisl library

# Set the variables for SISL
set( SISL_INCLUDE     ${CMAKE_SOURCE_DIR}/include )
set( SISL_SRC         ${CMAKE_SOURCE_DIR}/src )

file( GLOB_RECURSE SISL_HEADER_LIST "${SISL_INCLUDE}/*.h" )
file( GLOB_RECURSE SISL_SOURCE_LIST "${SISL_SRC}/*.h" "${SISL_SRC}/*.c" )
	
# Create target
add_library(sisl STATIC)
#set_target_properties(sisl PROPERTIES LINKER_LANGUAGE CXX)
# Add the header includes and headers to the library
target_include_directories(sisl PUBLIC
    $<BUILD_INTERFACE:${SISL_INCLUDE}>
    $<INSTALL_INTERFACE:include>)
# Add headers in the include folder as PUBLIC so downstream
# targets have access to them
target_sources(sisl PUBLIC ${SISL_HEADER_LIST})
# Add headers and sources in the source folder as PRIVATE as they
# will not be needed downstream
target_sources(sisl PRIVATE ${SISL_SOURCE_LIST})
# Add an alias so that a library can be used inside the build tree, eg when testing
add_library(Sisl::Sisl ALIAS sisl)

IF(CMAKE_COMPILER_IS_GNUCC)
    TARGET_LINK_LIBRARIES(sisl PUBLIC m)
ENDIF(CMAKE_COMPILER_IS_GNUCC)

SET_PROPERTY(TARGET sisl
  PROPERTY FOLDER "sisl/Libs")

# This line may be needed only on AMD 64 bit...
#IF(NOT WIN32)
#  SET_TARGET_PROPERTIES(sisl PROPERTIES COMPILE_FLAGS "-fPIC")
#ENDIF(NOT WIN32)

# Set various compiler flags
IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-unused-but-set-variable -fPIC")
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wno-unused-but-set-variable -fPIC")
ENDIF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
IF(WIN32)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP8 -D_SCL_SECURE_NO_WARNINGS -D_CRT_SECURE_NO_WARNINGS")
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /MP8 -D_SCL_SECURE_NO_WARNINGS -D_CRT_SECURE_NO_WARNINGS")
ENDIF(WIN32)


# Apps, examples, tests, ...?

option(sisl_COMPILE_APPS "Compile apps?" OFF)

option(sisl_COMPILE_EXAMPLES "Compile examples?" OFF)

option(sisl_COMPILE_VIEWER "Compile viewer?" OFF)

IF(sisl_COMPILE_APPS)
  FILE(GLOB sisl_APPS app/*.[cC])
  FOREACH(app ${sisl_APPS})
    GET_FILENAME_COMPONENT(appname ${app} NAME_WE)
    ADD_EXECUTABLE(${appname} ${app})
    TARGET_LINK_LIBRARIES(${appname} sisl sisl_streaming)
    SET_TARGET_PROPERTIES(${appname}
      PROPERTIES RUNTIME_OUTPUT_DIRECTORY app)
  ENDFOREACH(app)
ENDIF()

IF(sisl_COMPILE_EXAMPLES OR sisl_COMPILE_VIEWER OR sisl_COMPILE_APPS)
  INCLUDE_DIRECTORIES(
    ${sisl_SOURCE_DIR}/streaming/include
    )

  FILE(GLOB sisl_go_SRCS streaming/src/*.cpp streaming/include/*.h)
  ADD_LIBRARY(sisl_streaming ${sisl_go_SRCS})
  # IF(CMAKE_COMPILER_IS_GNUCC)
  #     TARGET_LINK_LIBRARIES(sisl_go m)
  # ENDIF(CMAKE_COMPILER_IS_GNUCC)
ENDIF()

IF(sisl_COMPILE_EXAMPLES)
  FILE(GLOB sisl_APPS2 examples/*.cpp)
  FOREACH(app ${sisl_APPS2})
    GET_FILENAME_COMPONENT(appname ${app} NAME_WE)
    ADD_EXECUTABLE(${appname} ${app})
    TARGET_LINK_LIBRARIES(${appname} sisl sisl_streaming)
    SET_TARGET_PROPERTIES(${appname}
      PROPERTIES RUNTIME_OUTPUT_DIRECTORY examples)
  ENDFOREACH(app)
ENDIF()

IF(sisl_COMPILE_VIEWER)

  find_package(OpenGL REQUIRED)

  find_path(OPENGL_INCLUDE_DIR
            gl.h
            PATH_PREFIXES GL
            PATHS
            /usr/local/include
            /usr/include
            "$ENV{HOME}\\Install\\include" # For some reason HOME is not found in VS 2019.
            "C:\\Users\\$ENV{USERNAME}\\Install\\include"
	    "C:\\local\\include"
            )

  if(WIN32)

    find_path(GLUT_INCLUDE_DIR
              glut.h
              PATH_PREFIXES GL
              PATHS
              "$ENV{PROGRAMW6432}/Microsoft SDKs/Windows/v6.0A/Include"
              "$ENV{HOME}\\Install\\include" # For some reason HOME is not found in VS 2019.
              "C:\\Users\\$ENV{USERNAME}\\Install\\include"
	      "C:\\local\\include"
              )
    if(${MSVC_VERSION} EQUAL 1900)
      set(MSVC_NAME "msvc2015_")
      # MESSAGE("Visual Studio 2015!")
    elseif((${MSVC_VERSION} GREATER_EQUAL 1920) AND (${MSVC_VERSION} LESS 1930))
      # MESSAGE("Visual Studio 2019!")
      set(MSVC_NAME "msvc2019_")
    elseif((${MSVC_VERSION} GREATER_EQUAL 1930) AND (${MSVC_VERSION} LESS 1940))
      set(MSVC_NAME "msvc2022_")
    else()
      message("MSVC version not supported or not installed!")
    endif()
    if(CMAKE_CL_64)
      set(WIN_LIB_TYPE "64")
    else()
      set(WIN_LIB_TYPE "32")
    endif()

    find_library( GLUT_LIBRARY_RELEASE NAMES freeglut
    		  PATHS
  		  "~/Install/${MSVC_NAME}lib${WIN_LIB_TYPE}/Release"
  		  "C:/local/${MSVC_NAME}lib${WIN_LIB_TYPE}/Release"
                  ${GLUT_ROOT_PATH}/Release
                  #"~/Install/${MSVC_NAME}lib${WIN_LIB_TYPE}/Release"
                 )

    find_library( GLUT_LIBRARY_DEBUG NAMES freeglutd
    		  PATHS
                  ${GLUT_ROOT_PATH}/Debug
		  "~/Install/${MSVC_NAME}lib${WIN_LIB_TYPE}/Debug"
		  "C:/local/${MSVC_NAME}lib${WIN_LIB_TYPE}/Debug"
                  #"~/Install/${MSVC_NAME}lib${WIN_LIB_TYPE}/Debug"
                 )
    # message("GLUT_LIBRARY_RELEASE: ${GLUT_LIBRARY_RELEASE}")
    # message("GLUT_LIBRARY_DEBUG: ${GLUT_LIBRARY_DEBUG}")
    
    set(GLUT_glut_LIBRARY "")
    # message("GLUT_glut_LIBRARY: ${GLUT_glut_LIBRARY}")
    if(GLUT_LIBRARY_DEBUG)
      set(GLUT_glut_LIBRARY ${GLUT_glut_LIBRARY} debug ${GLUT_LIBRARY_DEBUG})
    endif()
    if(GLUT_LIBRARY_RELEASE)
      set(GLUT_glut_LIBRARY ${GLUT_glut_LIBRARY} optimized ${GLUT_LIBRARY_RELEASE})
    endif()
    # message("GLUT_glut_LIBRARY: ${GLUT_glut_LIBRARY}")
  else()
    find_package(GLUT REQUIRED)  
  endif()

  INCLUDE_DIRECTORIES(
    ${sisl_SOURCE_DIR}/viewer/include
    ${OPENGL_INCLUDE_DIR}
    ${GLUT_INCLUDE_DIR}
    )

  # message("GLUT_INCLUDE_DIR: ${GLUT_INCLUDE_DIR}")
  # message("GLUT_INCLUDE_PATH: ${GLUT_INCLUDE_PATH}")
  # message("GLUT_glut_LIBRARY_RELEASE: ${GLUT_glut_LIBRARY_RELEASE}")
  # message("GLUT_glut_LIBRARY_DEBUG: ${GLUT_glut_LIBRARY_DEBUG}")
  # message("GLUT_glut_LIBRARY: ${GLUT_glut_LIBRARY}")
  # message("OPENGL_LIBRARIES: ${OPENGL_LIBRARIES}")
  # message("MSVC_NAME lib WIN_LIB_TYPE: ${MSVC_NAME}lib${WIN_LIB_TYPE}")

  FILE(GLOB sisl_viewer_SRCS viewer/src/*.cpp viewer/include/*.h)
  ADD_LIBRARY(sisl_viewer ${sisl_viewer_SRCS})
  IF(CMAKE_COMPILER_IS_GNUCC)
    TARGET_LINK_LIBRARIES(sisl_streaming sisl)
  ENDIF(CMAKE_COMPILER_IS_GNUCC)

  FILE(GLOB sisl_APPS3 viewer/*.cpp)
  FOREACH(app ${sisl_APPS3})
    GET_FILENAME_COMPONENT(appname ${app} NAME_WE)
    ADD_EXECUTABLE(${appname} ${app})
    TARGET_LINK_LIBRARIES(${appname} sisl sisl_streaming sisl_viewer ${OPENGL_LIBRARIES} ${GLUT_glut_LIBRARY})
    SET_TARGET_PROPERTIES(${appname}
      PROPERTIES RUNTIME_OUTPUT_DIRECTORY viewer)
  ENDFOREACH(app)
ENDIF()

##################################################################################################
# 'install' target

include(GNUInstallDirs)
set(INSTALL_CONFIGDIR ${CMAKE_INSTALL_LIBDIR}/cmake/Sisl)

# Install the sisl target (library and headers)
install(
    TARGETS sisl
    EXPORT SislTargets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    INCLUDES DESTINATION include
)
# This is required so that the exported target has the name Sisl and not sisl
set_target_properties(sisl PROPERTIES EXPORT_NAME Sisl)

# Create a ConfigVersion.cmake file
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/SislConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

# Create the package config file for find_package(Sisl)
configure_package_config_file(
    ${CMAKE_CURRENT_LIST_DIR}/cmake/SislConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/SislConfig.cmake
    INSTALL_DESTINATION ${INSTALL_CONFIGDIR}
)

# Install the config and config version files
# Note: if you write custom find modules, add them here
# Refer to https://github.com/pabloariasal/modern-cmake-sample/blob/cfda7b51ff915409fd61d4f2e75b63032bedb182/libjsonutils/CMakeLists.txt
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/SislConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/SislConfigVersion.cmake
	  ${CMAKE_CURRENT_BINARY_DIR}/SislTargets.cmake
    DESTINATION ${INSTALL_CONFIGDIR}
)

# CPack stuff
IF(NOT GoTools_ALL_COMPILE)
  SET(CPACK_SOURCE_PACKAGE_FILE_NAME "sisl-4.6.0")
  SET(CPACK_SOURCE_IGNORE_FILES "/CVS/;/\\.svn/;\\.swp$;\\.#;/#;.*~")
  INCLUDE(CPack)
ENDIF(NOT GoTools_ALL_COMPILE)

SET(CPACK_STRIP_FILES ${CPACK_STRIP_FILES} libsisl.so)

#########################################################################################
# Exports

# Export just the sisl target (not tests, etc.)
export(
    TARGETS sisl
    FILE ${CMAKE_CURRENT_BINARY_DIR}/SislTargets.cmake
    NAMESPACE Sisl::
)
