cmake_minimum_required(VERSION 2.8.3)
project(lidar_imu_init)

# =============================
# Build type & compiler options
# =============================
set(CMAKE_BUILD_TYPE "Release")
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Keep flags tidy
set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS} -fexceptions")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -pthread -fexceptions")

add_definitions(-DROOT_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/\")

# =============================
# CPU detection & MP defines
# =============================
message("Current CPU architecture: ${CMAKE_SYSTEM_PROCESSOR}")
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)")
  include(ProcessorCount)
  ProcessorCount(N)
  message("Processor number: ${N}")
  if(N GREATER 4)
    add_definitions(-DMP_EN)
    add_definitions(-DMP_PROC_NUM=3)
    message("core for MP: 3")
  elseif(N GREATER 3)
    add_definitions(-DMP_EN)
    add_definitions(-DMP_PROC_NUM=2)
    message("core for MP: 2")
  else()
    add_definitions(-DMP_PROC_NUM=1)
  endif()
else()
  add_definitions(-DMP_PROC_NUM=1)
endif()

# =========
# OpenMP
# =========
find_package(OpenMP QUIET)
if(OpenMP_CXX_FOUND)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()
if(OpenMP_C_FOUND)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
endif()

# ==========================================
# Python (3.x) & NumPy + matplotlib-cpp opts
# ==========================================
option(USE_MATPLOTLIBCPP "Build with matplotlib-cpp plotting" ON)

# Old-CMake friendly discovery (works with 2.8.x)
find_package(PythonInterp 3 REQUIRED)
find_package(PythonLibs   3 REQUIRED)

if(USE_MATPLOTLIBCPP)
  # Locate matplotlibcpp.h (fallback to local include/)
  find_path(MATPLOTLIB_CPP_INCLUDE_DIRS
    NAMES matplotlibcpp.h
    HINTS
      ${CMAKE_CURRENT_SOURCE_DIR}/include
      ${CMAKE_CURRENT_SOURCE_DIR}/third_party
      /usr/local/include
      /usr/include
    PATH_SUFFIXES matplotlib-cpp matplotlibcpp
  )
  if(NOT MATPLOTLIB_CPP_INCLUDE_DIRS AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/include/matplotlibcpp.h")
    set(MATPLOTLIB_CPP_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/include")
  endif()
  if(NOT MATPLOTLIB_CPP_INCLUDE_DIRS)
    message(FATAL_ERROR
      "matplotlibcpp.h not found. Place it under ${CMAKE_CURRENT_SOURCE_DIR}/include "
      "or disable plotting with -DUSE_MATPLOTLIBCPP=OFF")
  endif()

  # Ask Python where NumPy's C headers are
  execute_process(
    COMMAND ${PYTHON_EXECUTABLE} -c "import sys; import numpy; sys.stdout.write(numpy.get_include())"
    OUTPUT_VARIABLE NUMPY_INCLUDE_DIR
    RESULT_VARIABLE NUMPY_RESULT
  )
  if(NOT NUMPY_RESULT EQUAL 0 OR NUMPY_INCLUDE_DIR STREQUAL "")
    message(FATAL_ERROR
      "NumPy headers not found for ${PYTHON_EXECUTABLE}. Install python3-numpy or pip3 install numpy.")
  endif()
endif()

message(STATUS "Using Python: ${PYTHON_EXECUTABLE}")
message(STATUS "Python include: ${PYTHON_INCLUDE_DIRS}")
if(USE_MATPLOTLIBCPP)
  message(STATUS "NumPy include: ${NUMPY_INCLUDE_DIR}")
  message(STATUS "matplotlibcpp include: ${MATPLOTLIB_CPP_INCLUDE_DIRS}")
endif()

# =======================
# Catkin & ROS dependencies
# =======================
find_package(catkin REQUIRED COMPONENTS
  geometry_msgs
  nav_msgs
  sensor_msgs
  roscpp
  rospy
  std_msgs
  pcl_ros
  tf
  livox_ros_driver
  message_generation
  eigen_conversions
)

# Messages
add_message_files(
  FILES
  Pose6D.msg
  States.msg
)

generate_messages(
  DEPENDENCIES
  geometry_msgs
)

# =======================
# Third-party libraries
# =======================
find_package(Eigen3 REQUIRED)
find_package(PCL 1.8 REQUIRED)
find_package(Ceres REQUIRED)

message(STATUS "Eigen include: ${EIGEN3_INCLUDE_DIR}")

# ==============
# catkin package
# ==============
catkin_package(
  CATKIN_DEPENDS geometry_msgs nav_msgs roscpp rospy std_msgs message_runtime
  DEPENDS EIGEN3 PCL
  INCLUDE_DIRS
)

# ==================
# Include directories
# ==================
# Use absolute devel include to avoid CMP0021 warning
include_directories(
  ${CATKIN_DEVEL_PREFIX}/include
  ${catkin_INCLUDE_DIRS}
  ${EIGEN3_INCLUDE_DIR}
  ${PCL_INCLUDE_DIRS}
  ${PYTHON_INCLUDE_DIRS}
  include
)

if(USE_MATPLOTLIBCPP)
  include_directories(
    ${NUMPY_INCLUDE_DIR}
    ${MATPLOTLIB_CPP_INCLUDE_DIRS}
  )
endif()

# =======
# Target
# =======
add_executable(li_init
  src/laserMapping.cpp
  include/ikd-Tree/ikd_Tree.cpp
  include/LI_init/LI_init.cpp
  src/preprocess.cpp
)

# If you prefer to keep these include dirs target-local (optional)
if(USE_MATPLOTLIBCPP)
  target_include_directories(li_init PRIVATE
    ${NUMPY_INCLUDE_DIR}
    ${MATPLOTLIB_CPP_INCLUDE_DIRS}
  )
endif()
target_include_directories(li_init PRIVATE
  ${PYTHON_INCLUDE_DIRS}
)

# Link libraries
target_link_libraries(li_init
  ${catkin_LIBRARIES}
  ${PCL_LIBRARIES}
  ${PYTHON_LIBRARIES}
)

# Ceres can be available as imported target or old-style vars; handle both
if(TARGET Ceres::ceres)
  target_link_libraries(li_init Ceres::ceres)
elseif(DEFINED CERES_LIBRARIES)
  target_link_libraries(li_init ${CERES_LIBRARIES})
endif()
