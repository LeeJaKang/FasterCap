# minimum version is 3.0 to be able to handle Eigen library
cmake_minimum_required (VERSION 2.8)
project (FasterCap)

# main project specific settings
#
# option to compile as headless, or with GUI
set(FASTFIELDSOLVERS_HEADLESS OFF CACHE BOOL "Compile as headless (i.e. no GUI)")

#
# Global Compiler and Linker Settings
#

#
# using only GCC, therefore configuring build only for GCC
#
# if headless, pass on the "#define FCG_HEADLESS"
if(${FASTFIELDSOLVERS_HEADLESS} MATCHES ON)
    add_definitions("-DFCG_HEADLESS -DwxUSE_GUI=0")
else()
    add_definitions("-DwxUSE_GUI=1")
endif()
#
# List of options to pass to the compiler (note this is a CMake list. Do not enclose in ")
set(FASTFIELDSOLVERS_COMPILE_OPTIONS ${FASTFIELDSOLVERS_COMPILE_OPTIONS} -Wextra -Wall -fopenmp)
# -fopenmp: use OpenMP pragmas
# -lm: link the math library
# -lstdc++: link the standard c++ library (needed for headless)
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fopenmp -lm -lstdc++")
# -s: strip all symbols from the executable
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} -s")

# wxWidgets usage
set(wxWidgets_CONFIG_OPTIONS --version=3.0 --static=no $<$<CONFIG:Debug>:--debug>)
if(${FASTFIELDSOLVERS_HEADLESS} MATCHES ON)
    find_package(wxWidgets COMPONENTS base)
else()
    find_package(wxWidgets COMPONENTS core base html)
endif()
include(${wxWidgets_USE_FILE})
set(WXWIDGETS_LIBS ${WXWIDGETS_LIBS} ${wxWidgets_LIBRARIES})

#
# Include directories
#
# Parent directory: for LinAlgebra/ and Geometry/ external deps
include_directories("..")
# Project root: for res/ includes
include_directories(".")
# Source subdirectories: allow bare includes across modules
include_directories("src")
include_directories("src/core")
include_directories("src/solver")
include_directories("src/gui")

# External libraries (LinAlgebra, Geometry) - sibling directories
include_directories ("../LinAlgebra")
add_subdirectory("../LinAlgebra" LinAlgebra)
include_directories ("../Geometry")
add_subdirectory("../Geometry" Geometry)
set (EXTRA_LIBS ${EXTRA_LIBS} LinAlgebra Geometry)

# Solver sources (included as source list, not a separate library)
include_directories ("${PROJECT_SOURCE_DIR}/src/solver")
include("./src/solver/CMakeLists.txt")

#
# Core sources (console, globals, test)
#
set(FastFieldSolvers_Sources ${FastFieldSolvers_Sources}
    src/core/FasterCapConsole.cpp
    src/core/FasterCapConsole.h
    src/core/FasterCapGlobal.cpp
    src/core/FasterCapGlobal.h
    src/test/test.cpp
    src/test/test.h)

#
# GUI sources (only when not headless)
#
if(${FASTFIELDSOLVERS_HEADLESS} MATCHES OFF)
    set(FastFieldSolvers_Sources ${FastFieldSolvers_Sources}
        src/gui/AboutBox.cpp
        src/gui/AboutBox.h
        src/gui/FasterCapApp.cpp
        src/gui/FasterCapApp.h
        src/gui/FasterCapMain.cpp
        src/gui/FasterCapMain.h
        src/gui/LicenseBox.cpp
        src/gui/LicenseBox.h
        src/gui/RunDialog.cpp
        src/gui/RunDialog.h
        src/gui/resource.rc)
    # Windows-only: COM automation
    if(CMAKE_SYSTEM_NAME STREQUAL Windows)
        set(FastFieldSolvers_Sources ${FastFieldSolvers_Sources}
            src/platform/windows/AutomationHelper.cpp
            src/platform/windows/AutomationHelper.h)
    endif()
endif()

add_executable (FasterCap ${FastFieldSolvers_Sources} ${FasterCap_Sources})
# Specify compile options to use when compiling a given target.
target_compile_options(FasterCap PUBLIC ${FASTFIELDSOLVERS_COMPILE_OPTIONS})
target_link_libraries(FasterCap ${EXTRA_LIBS} ${WXWIDGETS_LIBS} ${EIGEN_LIBS})
