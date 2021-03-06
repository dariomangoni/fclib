# =================================================================
# cmake configuration to compile and install fclib library
# =================================================================

#
# Global cmake Settings 
#

# Set minimum version for cmake
cmake_minimum_required(VERSION 2.8)

# Set policy
cmake_policy(VERSION 2.8)

# Set cmake modules directory (i.e. the one which contains all
# user-defined FindXXX.cmake files among other things)
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/CMake)

# debug
macro(display V)
  message(STATUS "${V} = ${${V}}")
endmacro(display V)

MACRO(ASSERT VAR)
  IF (NOT DEFINED ${VAR})
    MESSAGE( FATAL_ERROR "ASSERTION ERROR : ${VAR} UNSET" )
  ENDIF()
ENDMACRO()

IF(APPLE)
  SET (CMAKE_OSX_ARCHITECTURES "i386;x86_64" )
ENDIF(APPLE)

# User defined options
option(VERBOSE_MODE "enable verbose mode for cmake exec. Default = on" ON)
option(USE_MPI "compile and link fclib with mpi when this mode is enable. Default = on." OFF)
option(BUILD_SHARED_LIBS "Enable dynamic library build, default = ON" ON)
option(WITH_TESTS "Enable testing. Default = on" ON)
option(FORCE_SKIP_RPATH "Do not build shared libraries with rpath. Useful only for packaging. Default = OFF" OFF)

# cmake project name
set(PROJECT_NAME fclib)

# This name will be used to install FCLib (library, headers, ...) and
# when another lib or soft will need to search for FCLib.
set(PACKAGE_NAME "FCLib")

# --- Set a version number for the package ---
set(${PACKAGE_NAME}_version 1.1.0)

# --- The name (without extension) of the lib to be created ---
set(PROJECT_LIBRARY_NAME ${PROJECT_NAME})

# The list of all dirs containing sources to be compiled for the fclib lib
# Any file in those dirs will be used to create libfclib
set(${PROJECT_LIBRARY_NAME}_SRCDIRS "src;externals/SuiteSparse/CSparse")

# Matching expr for files to be compiled. 
set(EXTS *.c)

# Matching expr for headers (install purpose)
set(EXTS_HDRS *.h)

# ============= The project =============
# Set project name and project languages 
# => this automatically defines:
#   - ${PROJECT_NAME}_BINARY_DIR : where you have run cmake, i.e. the place for compilation
#   - ${PROJECT_NAME}_SOURCE_DIR : where sources (.f and .h and this CMakeLists.txt) are located
# Note that because of OutOfSourceBuild, binary_dir and source_dir must be different. 
project(${PROJECT_NAME} C)

# Install lib directory 32, 64 etc. on Fedora, Debian 
# http://public.kitware.com/Bug/view.php?id=11964
# See also http://www.cmake.org/cmake/help/v3.0/module/GNUInstallDirs.html?highlight=gnuinstalldirs
include(GNUInstallDirs)

# ============= Doc and website =============
# Doxygen documentation
if(WITH_DOCUMENTATION)
  include(FCLibDoc)
endif(WITH_DOCUMENTATION)


include(FeatureSummary)

# ============= Search for libraries  =============
# set the compile/link conf (-I and -L opt)

# --- hdf5 --- what a mess!, cf this thread
# http://www.cmake.org/pipermail/cmake/2011-June/044723.html
find_package(HDF5 REQUIRED COMPONENTS HL)

#set(HDF5_INCLUDE_DIRS "/opt/local/include/")
include_directories(${HDF5_INCLUDE_DIRS})

set(LIBS ${LIBS} ${HDF5_LIBRARIES} ${HDF5_HL_LIBRARIES})
display(HDF5_INCLUDE_DIRS)

# --- MPI ---
if(USE_MPI)
  # Find MPI for C++ and fortran.
  find_package(MPI REQUIRED)
  if(MPI_COMPILER)
    set(CMAKE_CXX_COMPILER mpic++)
    set(CMAKE_C_COMPILER mpicc)
  elseif(MPI_COMPILER)
    # -I
    include_directories(${MPI_INCLUDE_PATH})
    # Add compilation flags
    append_c_flags(${MPI_COMPILE_FLAGS})
    set(${PROJECT_NAME}_LINK_FLAGS ${${PROJECT_NAME}_LINK_FLAGS} ${MPI_LINK_FLAGS})
  endif(MPI_COMPILER)
  set(LIBS ${LIBS} ${MPI_LIBRARIES} )
endif(USE_MPI)

# ============= Prepare compilation =============
# Force a default build type if not provided by user
# CMAKE_BUILD_TYPE = empty, Debug, Release, RelWithDebInfo or MinSizeRel.
if (NOT CMAKE_BUILD_TYPE)
  set (CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None, Debug, Release, RelWithDebInfo or MinSizeRel." FORCE)  
endif (NOT CMAKE_BUILD_TYPE)

if(MSVC)
	option(USING_HDF5_DLLs "WIN ONLY: turn on if the shared library of HDSL is linked" ON)
	IF (USING_HDF5_DLLs)
		ADD_DEFINITIONS("-DH5_BUILT_AS_DYNAMIC_LIB")
	ENDIF(USING_HDF5_DLLs)
	ADD_DEFINITIONS("-DFCLIB_APICOMPILE=__declspec( dllexport )")
	ADD_DEFINITIONS("-D_CRT_SECURE_NO_WARNINGS")
endif(MSVC)

# ============= Source and header files list =============
# We scan all files with matching extension in directories 
# containing sources.
# Source and header files list:
foreach(_DIR ${${PROJECT_LIBRARY_NAME}_SRCDIRS})
  set(_DIR_FILES)
  foreach(_EXT ${EXTS}) # Source files
    file(GLOB _DIR_FILES_EXT ${_DIR}/${_EXT})
    if(_DIR_FILES_EXT)
      list(APPEND ${PROJECT_LIBRARY_NAME}_SRC ${_DIR_FILES_EXT})
    endif()
  endforeach()
  foreach(_EXT ${EXTS_HDRS}) # Headers
    file(GLOB _DIR_FILES_EXT ${_DIR}/${_EXT})
    if(_DIR_FILES_EXT)
      list(APPEND ${PROJECT_LIBRARY_NAME}_HDRS ${_DIR_FILES_EXT})
    endif()
  endforeach()
endforeach()

# We add headers to source files
list(APPEND ${PROJECT_LIBRARY_NAME}_SRC ${${PROJECT_LIBRARY_NAME}_HDRS})

# Add directories to those searched by compiler ...
# -I
include_directories(${${PROJECT_LIBRARY_NAME}_SRCDIRS})

# ============= Creates the library =============
if(BUILD_SHARED_LIBS) # shared library
  add_library(${PROJECT_LIBRARY_NAME} SHARED ${${PROJECT_LIBRARY_NAME}_SRC})
else() # static library
  add_library(${PROJECT_LIBRARY_NAME} STATIC ${${PROJECT__LIBRARY_NAME}_SRC})
endif()
# Libs to link with PROJECT__LIBRARY_NAME
target_link_libraries(${PROJECT_LIBRARY_NAME} ${LIBS})

set_target_properties(${PROJECT_LIBRARY_NAME} PROPERTIES
  OUTPUT_NAME "${PROJECT_LIBRARY_NAME}"
  VERSION "${${PACKAGE_NAME}_version}"
  CLEAN_DIRECT_OUTPUT 1 # no clobbering
  LINKER_LANGUAGE C)

# ============== Add tests ==============

FILE(GLOB_RECURSE _DATA_FILES 
  RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/${_D}
  *.hdf5
  *.mat 
  *.dat
  *.xml
  *.DAT
  *.INI)

FOREACH(_F ${_DATA_FILES})
  IF (NOT "${_F}" MATCHES "fclib-library")
    GET_FILENAME_COMPONENT(_BASENAME_F "${_F}" NAME )
    MESSAGE(STATUS "basename = ${_BASENAME_F}")
    MESSAGE(STATUS "copy  ${CMAKE_CURRENT_SOURCE_DIR}/${_D}/${_F} in ${CMAKE_CURRENT_BINARY_DIR}/${_BASENAME_F}")
    CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/${_D}/${_F} ${CMAKE_CURRENT_BINARY_DIR}/${_BASENAME_F} COPYONLY)
  ENDIF()
ENDFOREACH(_F ${_DATA_FILES})


add_executable(fctest1 src/tests/fctst.c)
add_executable(fctest_merit src/tests/fctst_merit.c)
target_link_libraries(fctest1 ${PROJECT_LIBRARY_NAME})
target_link_libraries(fctest_merit ${PROJECT_LIBRARY_NAME})
add_test(fctest1 fctest1)
add_test(fctest_merit fctest_merit)
enable_testing()

# # =========== RPATH stuff ===========
# # we follow recommendation of https://cmake.org/Wiki/CMake_RPATH_handling

# =========== RPATH stuff ===========
# do not skip the full RPATH for the build tree
if(FORCE_SKIP_RPATH)
  set(CMAKE_SKIP_BUILD_RPATH TRUE)
else()
  set(CMAKE_SKIP_BUILD_RPATH FALSE)
endif()

# when building, don't use the install RPATH already
# (but later on when installing)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE) 

# when building a binary package, it makes no sense to add this rpath
if(NOT FORCE_SKIP_RPATH)
  # the RPATH to be used when installing
  ASSERT(CMAKE_INSTALL_LIBDIR)
  set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}")
endif(NOT FORCE_SKIP_RPATH)

# don't add the automatically determined parts of the RPATH
# which point to directories outside the build tree to the install RPATH
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# ============= Prepare install =============

# The library, the headers and mod files, the cmake generated files
# will be install in CMAKE_INSTALL_PREFIX/lib include and share
display(${PROJECT_LIBRARY_NAME}_HDRS)
set(${PROJECT_LIBRARY_NAME}_INSTALL_HDRS "${CMAKE_SOURCE_DIR}/src/fclib.h;")
include(InstallPackage)
install_package(${PACKAGE_NAME} ${PROJECT_LIBRARY_NAME} ${PROJECT_LIBRARY_NAME}_INSTALL_HDRS)

# ============= Summary =============
if(VERBOSE_MODE)
  message(STATUS "====================== Summary ======================")
  message(STATUS " Compiler : ${CMAKE_C_COMPILER}")
  message(STATUS " Sources are in : ${CMAKE_SOURCE_DIR}")
  message(STATUS " Project uses MPI : ${USE_MPI}")
  message(STATUS " Project uses HDF5 : ${HDF5_LIBRARIES}")
  message(STATUS " Project will be installed in ${CMAKE_INSTALL_PREFIX}")
  message(STATUS "====================== ======= ======================")
endif()


feature_summary(WHAT ALL)
