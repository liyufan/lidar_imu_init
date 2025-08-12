cmake_minimum_required(VERSION 3.9)
project(lidar_imu_init)

SET(CMAKE_BUILD_TYPE "Release")

add_compile_options(
  -O3
  -fexceptions
)

add_definitions(-DROOT_DIR=\"${CMAKE_CURRENT_SOURCE_DIR}/\")

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

message("Current CPU architecture: ${CMAKE_SYSTEM_PROCESSOR}")
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)" )
  include(ProcessorCount)
  ProcessorCount(N)
  message("Processer number:  ${N}")
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

find_package(OpenMP QUIET)

find_package(PythonLibs REQUIRED)
find_path(MATPLOTLIB_CPP_INCLUDE_DIRS "matplotlibcpp.h")

find_package(catkin REQUIRED COMPONENTS
  geometry_msgs
  nav_msgs
  sensor_msgs
  roscpp
  rospy
  std_msgs
  pcl_ros
  tf
  message_generation
  eigen_conversions
)

find_package(catkin COMPONENTS
  livox_ros_driver2
  livox_ros_driver
)
find_package(Eigen3 REQUIRED)
find_package(PCL 1.8 REQUIRED)
find_package(Ceres REQUIRED)

message(Eigen: ${EIGEN3_INCLUDE_DIR})

add_message_files(
  FILES
  Pose6D.msg
  States.msg
)

include_directories(
  ${catkin_INCLUDE_DIRS} 
  ${EIGEN3_INCLUDE_DIR}
  ${PCL_INCLUDE_DIRS}
  ${PYTHON_INCLUDE_DIRS}
  include
)

generate_messages(
  DEPENDENCIES
  geometry_msgs
)

catkin_package(
  CATKIN_DEPENDS geometry_msgs nav_msgs roscpp rospy std_msgs message_runtime
  DEPENDS EIGEN3 PCL
  INCLUDE_DIRS
)

add_executable(li_init
  src/laserMapping.cpp
  include/ikd-Tree/ikd_Tree.cpp
  include/LI_init/LI_init.cpp
  src/preprocess.cpp
)

if(livox_ros_driver2_FOUND)
  message(STATUS "Found livox_ros_driver2: ${livox_ros_driver2_DIR}")
  target_compile_definitions(li_init PRIVATE -DLIVOX_ROS_DRIVER2)
elseif(livox_ros_driver_FOUND)
  message(STATUS "Found livox_ros_driver: ${livox_ros_driver_DIR}")
  target_compile_definitions(li_init PRIVATE -DLIVOX_ROS_DRIVER)
else()
  message(FATAL_ERROR "livox_ros_driver or livox_ros_driver2 not found")
endif()

if(OpenMP_CXX_FOUND)
  target_link_libraries(li_init OpenMP::OpenMP_CXX)
endif()

target_link_libraries(li_init
  ${catkin_LIBRARIES}
  ${PCL_LIBRARIES}
  ${PYTHON_LIBRARIES}
  ${CERES_LIBRARIES}
)
target_include_directories(li_init PRIVATE ${PYTHON_INCLUDE_DIRS})
