cmake_minimum_required(VERSION 3.12.1 FATAL_ERROR)
project("pbcvt")

#----------------------------CMAKE & GLOBAL PROPERTIES-------------------------#
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

###============= C++11 support====================================
if(${CMAKE_VERSION} VERSION_LESS "3.1")
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if (COMPILER_SUPPORTS_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif (COMPILER_SUPPORTS_CXX0X)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else ()
    message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif ()
else()
	set(CMAKE_CXX_STANDARD 11)
	set(CMAKE_CXX_STANDARD_REQUIRED ON)
endif()
#=================================================================
# PYTHON option

set(PYTHON_OPTIONS "2.X" "3.X")
set(PYTHON_DESIRED_VERSION "3.X" CACHE STRING "Choose which python version to use, options are: ${PYTHON_OPTIONS}.")
set_property(CACHE PYTHON_DESIRED_VERSION PROPERTY STRINGS ${PYTHON_OPTIONS})

#=============== Find Packages ====================================
## OpenCV
find_package(Boost COMPONENTS ALL REQUIRED)
find_package(OpenCV COMPONENTS core REQUIRED)
find_package(Python 3.5 REQUIRED COMPONENTS Interpreter Development NumPy)


#========pick python stuff========================================
if (${PYTHON_DESIRED_VERSION} STREQUAL "2.X")
    set(ARCHIVE_OUTPUT_NAME pbcvt_py2)
    set(ARCHIVE_OUTPUT_SUFFIX _py2)
else ()
    set(ARCHIVE_OUTPUT_NAME pbcvt_py3)
    set(ARCHIVE_OUTPUT_SUFFIX _py3)
endif ()

file(GLOB project_sources ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)


macro(pbcvt_add_pbcvt_library target_name STATIC)
    if(${STATIC})
        add_library(${target_name} STATIC ${project_sources} ${CMAKE_CURRENT_SOURCE_DIR}/include/pyboostcvconverter/pyboostcvconverter.hpp)
    else()
        add_library(${target_name} SHARED ${project_sources} ${CMAKE_CURRENT_SOURCE_DIR}/include/pyboostcvconverter/pyboostcvconverter.hpp)
    endif()
    target_include_directories(
        ${target_name} 
        PUBLIC
        "${CMAKE_CURRENT_SOURCE_DIR}/include"
    )

    target_link_libraries(
        ${target_name}
        PUBLIC
        Python::Python
        Python::NumPy
        Boost::boost
        opencv_core
    )

    if(CMAKE_CXX_COMPILER_ID MATCHES MSVC)
        # Provisions for typical Boost compiled on Windows
        # Unless some extra compile options are used on Windows, the libraries won't have prefixes (change as necesssary)
        target_compile_definitions(${target_name} PUBLIC -DBOOST_ALL_NO_LIB -DBOOST_SYSTEM_NO_DEPRECATED)
    endif()
endmacro()

pbcvt_add_pbcvt_library(${PROJECT_NAME} OFF)

#---------------------------   INSTALLATION    -----------------------------------------------------
#-get proper extension for python binary shared object on this platform

set(__python_ext_suffix_var "EXT_SUFFIX")
if(${PYTHON_DESIRED_VERSION} STREQUAL "2.X")
    set(__python_ext_suffix_var "SO")
endif()

execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "import distutils.sysconfig; print(distutils.sysconfig.get_config_var('${__python_ext_suffix_var}'))"
        RESULT_VARIABLE PYTHON_${PROJECT_NAME}_PY_PROCESS
        OUTPUT_VARIABLE ${PROJECT_NAME}_PY_SUFFIX
        OUTPUT_STRIP_TRAILING_WHITESPACE)
if(NOT ${PYTHON_${PROJECT_NAME}_PY_PROCESS} EQUAL 0)
    set(${PROJECT_NAME}_PY_SUFFIX ".so")
endif()


set_target_properties(${PROJECT_NAME} PROPERTIES
        ARCHIVE_OUTPUT_NAME ${ARCHIVE_OUTPUT_NAME}  # prevent name conflict for python2/3 outputs
        PREFIX ""
        OUTPUT_NAME pbcvt
        SUFFIX ${${PROJECT_NAME}_PY_SUFFIX})



if (MSVC AND NOT PYTHON_DEBUG_LIBRARIES)
    set(PYTHON_INSTALL_CONFIGURATIONS CONFIGURATIONS Release)
else ()
    set(PYTHON_INSTALL_CONFIGURATIONS "")
endif ()

if (WIN32)
    set(PYTHON_INSTALL_ARCHIVE "")
else ()
    set(PYTHON_INSTALL_ARCHIVE ARCHIVE DESTINATION ${PYTHON_PACKAGES_PATH} COMPONENT python)
endif ()

install(TARGETS ${PROJECT_NAME}
        ${PYTHON_INSTALL_CONFIGURATIONS}
        RUNTIME DESTINATION ${PYTHON_PACKAGES_PATH} COMPONENT python
        LIBRARY DESTINATION ${PYTHON_PACKAGES_PATH} COMPONENT python
        ${PYTHON_INSTALL_ARCHIVE}
        )

#---------------------------   TEST PROJECT    -----------------------------------------------------
# Test Project option
option(BUILD_TEST_PROJECT ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
pbcvt_add_pbcvt_library(static_pbcvt ON)

if (BUILD_TEST_PROJECT)
    add_subdirectory(tests)
endif()
