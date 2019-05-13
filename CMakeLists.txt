#----------------------------------------------------------------------------

# Setup the project
#
cmake_minimum_required(VERSION 2.6 FATAL_ERROR)
project(G4HRS)

#MAC specific variable
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(CMAKE_MACOSX_RPATH ON)
    set(CMAKE_SKIP_BUILD_RPATH FALSE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
    set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
    set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
    list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
    if("${isSystemDir}" STREQUAL "-1")
        set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
    endif()
endif()

enable_language (Fortran)

#ROOT path
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules")
find_package(ROOT REQUIRED)

# Update header file
add_custom_target( gitinfo ALL COMMAND ${PROJECT_SOURCE_DIR}/pullgitinfo.py ${PROJECT_SOURCE_DIR} COMMENT "Creating gitinfo.hh" )


# this is to ensure we find the header we need
include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)

ROOT_GENERATE_DICTIONARY(
    g4hrsDict                         # path to dictionary to generate
    "include/g4hrsRunData.hh"         # list of classes to process
    "include/g4hrsTextFile.hh"
    LINKDEF "include/g4hrsLinkDef.h"  # ROOT linkDef file
)


#----------------------------------------------------------------------------
# Find Geant4 package, activating all available UI and Vis drivers by default
# You can set WITH_GEANT4_UIVIS to OFF via the command line or ccmake/cmake-gui
# to build a batch mode only executable
#
option(WITH_GEANT4_UIVIS "Build example with Geant4 UI and Vis drivers" ON)
if(WITH_GEANT4_UIVIS)
  find_package(Geant4 REQUIRED ui_all vis_all)
else()
  find_package(Geant4 REQUIRED)
endif()

#----------------------------------------------------------------------------
# Setup Geant4 include directories and compile definitions
# Setup include directory for this project
#
include(${Geant4_USE_FILE})


#debugging symbols, complains
#set(CMAKE_CXX_FLAGS "-Wall -pg")
#set(CMAKE_CXX_FLAGS "-g -Wall -g -Woverloaded-virtual")
set(CMAKE_CXX_FLAGS "-ggdb -pg -O0 -g3 -Wall -ggdb")
set(CMAKE_Fortran_FLAGS "-pg -O0 -ggdb")

# C++11 standard if avialable
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
if(COMPILER_SUPPORTS_CXX11)
        message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has C++11 support.")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)
        message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has C++0x support.")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
        message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support.")
endif()

#----------------------------------------------------------------------------
# Generate dictionaries as well (so we can write objects to ROOT files

include_directories(${PROJECT_SOURCE_DIR}/include ${ROOT_INCLUDE_DIR} ${CMAKE_CURRENT_BINARY_DIR}/include 
     ${PROJECT_SOURCE_DIR}/src/fields
     ${PROJECT_SOURCE_DIR}/HRSTransport
     ${PROJECT_SOURCE_DIR}/HRSTransport/hamcPREXTrans
     ${PROJECT_SOURCE_DIR}/HRSTransport/hamcCREXTrans
     ${PROJECT_SOURCE_DIR}/HRSTransport/HRSTransSTD )


#----------------------------------------------------------------------------
# Locate sources and headers for this project
# NB: headers are included so they will show up in IDEs
#
file(GLOB sources ${PROJECT_SOURCE_DIR}/src/g4hrs*.cc ${PROJECT_SOURCE_DIR}/src/fields/*.cc src/fortran/*.f)
file(GLOB headers ${PROJECT_SOURCE_DIR}/include/*.hh ${PROJECT_SOURCE_DIR}/src/fields/*.hh ${CMAKE_CURRENT_BINARY_DIR}/include/gitinfo.hh)

file(GLOB transsources ${PROJECT_SOURCE_DIR}/HRSTransport/*.cc 
    ${PROJECT_SOURCE_DIR}/HRSTransport/hamcPREXTrans/*.cc
    ${PROJECT_SOURCE_DIR}/HRSTransport/hamcPREXTrans/*.f

    ${PROJECT_SOURCE_DIR}/HRSTransport/hamcCREXTrans/*.cc
    ${PROJECT_SOURCE_DIR}/HRSTransport/hamcCREXTrans/*.f

    ${PROJECT_SOURCE_DIR}/HRSTransport/HRSTransSTD/*.cc
    ${PROJECT_SOURCE_DIR}/HRSTransport/HRSTransSTD/*.f
    )

file(GLOB transheaders ${PROJECT_SOURCE_DIR}/HRSTransport/*.hh
    ${PROJECT_SOURCE_DIR}/HRSTransport/hamcPREXTrans/*.hh
    ${PROJECT_SOURCE_DIR}/HRSTransport/hamcCREXTrans/*.hh
    ${PROJECT_SOURCE_DIR}/HRSTransport/HRSTransSTD/*.hh
    )


file(GLOB libsources ${PROJECT_SOURCE_DIR}/src/g4hrsRunData.cc ${PROJECT_SOURCE_DIR}/src/g4hrsTextFile.cc)
file(GLOB libheaders ${PROJECT_SOURCE_DIR}/include/*.hh ${CMAKE_CURRENT_BINARY_DIR}/include/gitinfo.hh)

#----------------------------------------------------------------------------
# Add the executable, and link it to the Geant4 libraries
#
add_executable(g4hrs g4hrs.cc ${sources} ${headers} g4hrsDict.cxx)
add_library(g4hrsroot SHARED ${libsources} ${libheaders} g4hrsDict.cxx)
add_library(HRSTransport ${transsources} ${transheaders})

target_link_libraries(g4hrs ${Geant4_LIBRARIES} ${ROOT_LIBRARIES} HRSTransport )
target_link_libraries(g4hrsroot ${ROOT_LIBRARIES} )

#----------------------------------------------------------------------------
# Copy all scripts to the build directory, i.e. the directory in which we
# build g4hrs. This is so that we can run the executable directly because it
# relies on these scripts being in the current working directory.
#

set(G4HRS_SCRIPTS
    vis/OIX.mac
    vis/Qt.mac
    vis/vis.mac
    vis/X.mac
    macros/runexample.mac
    macros/gui.mac
    macros/vis.mac
    scripts/BField_Septum.ini
    rootlogon.C 
    horpb.dat
    horpb1.dat
    ca48_fsu.dat
    ca48_fsu_stretched.dat
    ca40_fsu.dat
    ca40_fsu_stretched.dat
    ca48_fsu.dat
    ca48_fsu_stretched.dat
    c12_fsu.dat
  )

foreach(_script ${G4HRS_SCRIPTS})
  configure_file(
    ${PROJECT_SOURCE_DIR}/${_script}
    ${PROJECT_BINARY_DIR}/${_script}
    COPYONLY
    )
endforeach()

#----------------------------------------------------------------------------
# For internal Geant4 use - but has no effect if you build this
# example standalone
#
add_custom_target(G4HRS DEPENDS g4hrs)

#----------------------------------------------------------------------------
# Install the executable to 'bin' directory under CMAKE_INSTALL_PREFIX
#
install(TARGETS g4hrs DESTINATION bin)


