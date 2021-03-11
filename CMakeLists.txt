cmake_minimum_required(VERSION 3.9)
project(ov2slam)

set(ov2slam_VERSION "1.0.2")

set(LIBRARY_NAME ov2slam)

################################################################################
# Options
################################################################################

#######
# NOTE!
#######
# Set to OFF if you did not compile OpenCV w. opencv_contrib
set(WITH_OPENCV_CONTRIB OFF)

# iBOW-LCD
#
# First check if the related thridparty libs have been
# built. Only set the use of iBoW-LCD to true if built.
set(WITH_IBOW_LCD OFF)

################################################################################
# Paths
################################################################################

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

if(UNIX)
  include(GNUInstallDirs)

  set(ov2slam_INC_INSTALL_DIR "/usr/local/include/${PROJECT_NAME}")
  set(ov2slam_RUNTIME_INSTALL_DIR "${CMAKE_INSTALL_BINDIR}")
  set(ov2slam_LIBRARY_INSTALL_DIR "/usr/local/lib")
  set(ov2slam_ARCHIVE_INSTALL_DIR "/usr/local/lib")
  set(ov2slam_FRAMEWORK_INSTALL_DIR "/usr/local/lib")

  set(ov2slam_CMAKE_CONFIG_INSTALL_DIR "/usr/local/lib/cmake/ov2slam")
  set(ov2slam_ADDITIONAL_FILES_INSTALL_DIR "${CMAKE_INSTALL_DATADIR}/ov2slam")
endif()

################################################################################
# Include CMake dependencies
################################################################################

include(CheckCXXCompilerFlag)
include(CheckLanguage)
include(CMakePackageConfigHelpers)

################################################################################
# Compiler specific configuration
################################################################################

if(POLICY CMP0022)
  cmake_policy(SET CMP0022 NEW)
endif()

if(POLICY CMP0028)
  cmake_policy(SET CMP0028 NEW)
endif()

if(CMAKE_BUILD_TYPE)
  message(STATUS "Build type specified as ${CMAKE_BUILD_TYPE}")
else()
  message(STATUS "Build type not specified, using Release")
  set(CMAKE_BUILD_TYPE Release)
  set(IS_DEBUG OFF)
endif()

if( ${CMAKE_VERSION} VERSION_LESS "3.8.2" )
  set(CMAKE_CXX_STANDARD 14)
else()
  set(CMAKE_CXX_STANDARD 17)
endif()

set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

################################################################################
# Flags
################################################################################

#######
# NOTE!
#######
# You might need to remove the -march=native if you experience
# exceptions related to ceres or opengv.
# opengv comes natively with the march=native option while
# ceres requires to specify it at build time.
# If you experience exceptions you can either re-compile
# opengv and ov2slam without -march=native or re-compile
# ceres with it
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -Wall")

LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules)

################################################################################
# Dependencies
################################################################################

if(WITH_OPENCV_CONTRIB)
    message("\nGoing to use OpenCV contrib! (WITH_OPENCV_CONTRIB is set to : ${WITH_OPENCV_CONTRIB}.
Set it to OFF is OpenCV was not compiled with the opencv_contrib modules)\n")
    add_definitions(-DOPENCV_CONTRIB)
endif()

if( EXISTS "${PROJECT_SOURCE_DIR}/Thirdparty/ibow_lcd/build/liblcdetector.so" )
  #add_definitions(-DIBOW_LCD)
  #set(WITH_IBOW_LCD ON)
  message(STATUS "iBoW-LCD found!  Going to use Loop Closer!")
else ( )
  message(STATUS "iBoW-LCD NOT found!  Loop Closer will not be enabled!")
endif ( )

## System dependencies are found with CMake's conventions

# OpenCV
find_package(OpenCV REQUIRED)

if(OpenCV_VERSION_MAJOR LESS 3)
  message( FATAL_ERROR "OpenCV 3 or 4 is required!  Current version : ${OpenCV_VERSION}" )
endif()

if(OpenCV_VERSION VERSION_LESS "3.3.0")
  message("ENABLE Internal ParallelLoopBodyLambdaWrapper Class definition (OpenCV < 3.3.0)")
  add_definitions(-DOPENCV_LAMBDA_MISSING)
endif()

#Eigen3
find_package(Eigen3 3.3.0 REQUIRED NO_MODULE)

if (Eigen3_FOUND)
  message(STATUS "Found Eigen3 lib : ${Eigen3_VERSION} - ${EIGEN3_INCLUDE_DIRS}")
endif (Eigen3_FOUND)

# add_definitions(-DEIGEN_DONT_VECTORIZE -DEIGEN_DISABLE_UNALIGNED_ARRAY_ASSERT)

#OpenGV
find_package(opengv QUIET)
# Bug include CMake 1.17
get_target_property(opengv_INCLUDE_DIR opengv INTERFACE_INCLUDE_DIRECTORIES)

if (opengv_FOUND)
  message(STATUS "Found Opengv lib : ${opengv_INCLUDE_DIR} - ${opengv_DIR}")
  add_definitions(-DUSE_OPENGV)
endif (opengv_FOUND)

# Ceres
#find_package(Ceres NO_DEFAULT_PATH PATHS ${PROJECT_SOURCE_DIR}/Thirdparty/ceres-solver/install/)
find_package(Ceres REQUIRED)

if (Ceres_FOUND)
  message(STATUS "Found Ceres lib : ${CERES_VERSION} - ${Ceres_DIR}")
endif (Ceres_FOUND)

find_package(PCL 1.7 REQUIRED)
list(REMOVE_ITEM PCL_LIBRARIES "vtkproj4")

if (PCL_FOUND)
  message(STATUS "Found PCL lib : ${PCL_VERSION} - ${PCL_DIR}")
endif (PCL_FOUND)

#Sophus
# First search for a custom build of Sophus within this project
#find_package(Sophus NO_DEFAULT_PATH PATHS ${PROJECT_SOURCE_DIR}/Thirdparty/Sophus/build/)
#find_package(Sophus NO_DEFAULT_PATH PATHS ${PROJECT_SOURCE_DIR}/Thirdparty/Sophus/install)
find_package(Sophus REQUIRED HINTS ${PROJECT_SOURCE_DIR}/Thirdparty/Sophus/install)

if (Sophus_FOUND)
  message(STATUS "Found Sophus lib : ${Sophus_VERSION} - ${Sophus_DIR}")
endif (Sophus_FOUND)

################################################################################
# Sources
################################################################################

include_directories(
        ${PROJECT_SOURCE_DIR}/include
        ${catkin_INCLUDE_DIRS}
        ${PROJECT_SOURCE_DIR}/include/ceres_parametrization
        ${Sophus_INCLUDE_DIRS}
        ${PCL_INCLUDE_DIRS}
        ${PCL_LIBRARY_DIRS}
)

set(HEADERS
        include/camera_calibration.hpp
        include/estimator.hpp
        include/feature_extractor.hpp
        include/feature_tracker.hpp
        include/frame.hpp
        include/logger.hpp
        include/loop_closer.hpp
        include/map_manager.hpp
        include/map_point.hpp
        include/mapper.hpp
        include/multi_view_geometry.hpp
        include/optimizer.hpp
        include/ov2slam.hpp
        include/profiler.hpp
        include/slam_params.hpp
        include/visual_front_end.hpp
        include/ceres_parametrization/ceres_parametrization.hpp
        include/ceres_parametrization/ceres_parametrization/intrinsic_param_block.hpp
        include/ceres_parametrization/ceres_parametrization/inverse_depth_param_block.hpp
        include/ceres_parametrization/ceres_parametrization/pointxyz_param_block.hpp
        include/ceres_parametrization/ceres_parametrization/se3_param_block.hpp
        include/ceres_parametrization/ceres_parametrization/se3left_parametrization.hpp
)

set(SOURCES
        src/ov2slam.cpp
        src/visual_front_end.cpp
        src/frame.cpp
        src/slam_params.cpp
        src/camera_calibration.cpp
        src/feature_extractor.cpp
        src/feature_tracker.cpp
        src/map_manager.cpp
        src/map_point.cpp
        src/mapper.cpp
        src/multi_view_geometry.cpp
        src/ceres_parametrization.cpp
        src/optimizer.cpp
        src/estimator.cpp
        src/loop_closer.cpp
)

add_library(${LIBRARY_NAME} SHARED
        ${HEADERS}
        ${SOURCES}
)

target_include_directories(${LIBRARY_NAME} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}> # for headers when building
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> # for headers when building
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/ceres_parametrization> # for headers when building
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include/ceres_parametrization/ceres_parametrization> # for headers when building
        $<INSTALL_INTERFACE:${ov2slam_INC_INSTALL_DIR}> # for client in install mode
)

# Add optional dependencies if they are found
if (WITH_IBOW_LCD)
  target_include_directories(${LIBRARY_NAME} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Thirdparty/obindex2/lib/include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Thirdparty/ibow_lcd/include>
        $<INSTALL_INTERFACE:${ov2slam_INC_INSTALL_DIR}> # for client in install mode
  )

  target_link_libraries(${LIBRARY_NAME} PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/Thirdparty/ibow_lcd/build/liblcdetector.so>
        $<INSTALL_INTERFACE:${ov2slam_INC_INSTALL_DIR}>
  )
endif (WITH_IBOW_LCD)

if (opengv_FOUND)
  target_link_libraries(${LIBRARY_NAME} PRIVATE
        opengv
  )
endif (opengv_FOUND)

target_link_libraries(${LIBRARY_NAME}
        PRIVATE
          ${OpenCV_LIBS}
          Ceres::ceres
          ${PCL_LIBRARIES}
        PUBLIC
          ${EIGEN3_LIBS}
          ${Sophus_LIBS}
)

################################################################################
# Create Targets
################################################################################

# Library
install(TARGETS ${LIBRARY_NAME} EXPORT ov2slamTargets
        RUNTIME       DESTINATION ${ov2slam_RUNTIME_INSTALL_DIR}
        LIBRARY       DESTINATION ${ov2slam_LIBRARY_INSTALL_DIR}
        ARCHIVE       DESTINATION ${ov2slam_ARCHIVE_INSTALL_DIR}
        FRAMEWORK     DESTINATION ${ov2slam_FRAMEWORK_INSTALL_DIR})

# Headers
install(
        FILES ${HEADERS}
        DESTINATION ${ov2slam_INC_INSTALL_DIR}
)


################################################################################
# Install All
################################################################################

set_target_properties(${LIBRARY_NAME} PROPERTIES OUTPUT_NAME "ov2slam-${ov2slam_VERSION}")

# Create and install version file
set(version_file "${CMAKE_CURRENT_BINARY_DIR}/ov2slamConfigVersion.cmake")
write_basic_package_version_file(${version_file}
        VERSION ${ov2slam_VERSION}
        COMPATIBILITY ExactVersion
        )
install(FILES ${version_file}
        DESTINATION ${ov2slam_CMAKE_CONFIG_INSTALL_DIR})

# install readme and license
install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/README.md"
        DESTINATION ${ov2slam_ADDITIONAL_FILES_INSTALL_DIR}
        )

install(
        EXPORT ov2slamTargets
        DESTINATION ${ov2slam_CMAKE_CONFIG_INSTALL_DIR}
        FILE ov2slamConfig.cmake
)
