cmake_minimum_required(VERSION 3.21)

project (amg2023 C)

# Version information
set(AMG2023_MAJOR_VERSION 1)
set(AMG2023_MINOR_VERSION 0)
set(AMG2023_PATCH_VERSION 0)
set(AMG2023_VERSION "${AMG2023_MAJOR_VERSION}.${AMG2023_MINOR_VERSION}.${AMG2023_PATCH_VERSION}")

## Fail if someone tries to config an in-source build.
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
   message(FATAL_ERROR "In-source builds are not supported. Please remove "
                       "CMakeCache.txt from the 'src' dir and configure an "
                       "out-of-source build in another directory.")
endif()

add_definitions(-DAMG_CMAKE_BUILD=1)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

option(AMG_WITH_MPI "Use MPI" TRUE) 
option(AMG_WITH_CUDA "Use CUDA" TRUE)
option(AMG_WITH_HIP "Use HIP" FALSE)
option(AMG_WITH_CALIPER "Enable Caliper" FALSE)
option(AMG_WITH_OMP "Enable OpenMP" FALSE)
option(AMG_WITH_UMPIRE "Enable Umpire support (requires Umpire support in HYPRE)" FALSE)
option(AMG_WITH_CASSINI "Enable CassiniPapi performance counters" TRUE)

find_library(HYPRE_LIBRARIES
    NAMES HYPRE
    HINTS ${HYPRE_PREFIX}/lib)
find_path(HYPRE_INCLUDE_DIR
    NAMES HYPRE.h
    HINTS ${HYPRE_PREFIX}/include)

include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(HYPRE DEFAULT_MSG
    HYPRE_LIBRARIES
    HYPRE_INCLUDE_DIR)

if (NOT HYPRE_FOUND)
    message(FATAL_ERROR "Hypre not found")
endif()

add_definitions(-DHYPRE_TIMING)
include_directories(${HYPRE_INCLUDE_DIR})

list(APPEND AMG_DEPENDENCIES
  ${HYPRE_LIBRARIES})

if (AMG_WITH_CUDA)
  find_package(CUDAToolkit REQUIRED)
  list(APPEND AMG_DEPENDENCIES
    CUDA::cudart
    CUDA::cublas
    CUDA::cusparse
    CUDA::curand)
endif()

if (AMG_WITH_HIP)
  enable_language(CXX)
  find_package(hip REQUIRED)
  find_package(rocsparse REQUIRED)
  find_package(rocrand REQUIRED)
  list(APPEND AMG_DEPENDENCIES
    hip::device
    hip::host
    roc::rocsparse
    roc::rocrand)
endif()

if (AMG_WITH_CALIPER)
  find_package(caliper REQUIRED)
  list(APPEND AMG_DEPENDENCIES 
    caliper)
  set(USE_CALIPER TRUE)
  find_package(adiak REQUIRED)
  list(APPEND AMG_DEPENDENCIES
    adiak::adiak)
endif()

if (AMG_WITH_OMP)
  find_package(OpenMP REQUIRED)
  list(APPEND AMG_DEPENDENCIES
    OpenMP::OpenMP_C)
endif()

set(MPIP_LIB_PATH /pscratch/sd/c/cunyang/software/lib)
set(MPIP_INCLUDE_PATH /pscratch/sd/c/cunyang/software/include)

include_directories(${MPIP_INCLUDE_PATH})

if (AMG_WITH_MPI)
  find_package(MPI REQUIRED)
  list(APPEND AMG_DEPENDENCIES
    MPI::MPI_C)
endif()

if (AMG_WITH_UMPIRE)
  find_package(umpire REQUIRED)
  list(APPEND AMG_DEPENDENCIES
    umpire)
endif()

if (AMG_WITH_CASSINI)
  # CassiniPapi library paths
  set(CASSINI_DIR "/pscratch/sd/c/cunyang/CassiniPapi/src")
  set(CASSINI_INCLUDE_DIR "${CASSINI_DIR}/include")
  set(CASSINI_LIB_DIR "${CASSINI_DIR}/lib")
  
  # Find CassiniPapi library
  find_library(CASSINI_LIBRARY
    NAMES cassini
    PATHS ${CASSINI_LIB_DIR}
    REQUIRED)
  
  # Find PAPI library (required by CassiniPapi)
  find_library(PAPI_LIBRARY
    NAMES papi
    REQUIRED)
  
  # Add include directory
  include_directories(${CASSINI_INCLUDE_DIR})
  
  # Add libraries to dependencies
  list(APPEND AMG_DEPENDENCIES
    ${CASSINI_LIBRARY}
    ${PAPI_LIBRARY}
    pthread)
  
  # Add compile definition
  add_definitions(-DUSE_CASSINI)
  
  message(STATUS "CassiniPapi support enabled")
  message(STATUS "  CASSINI_LIBRARY: ${CASSINI_LIBRARY}")
  message(STATUS "  PAPI_LIBRARY: ${PAPI_LIBRARY}")
endif()

configure_file(
  "${CMAKE_SOURCE_DIR}/amg-config.h.in"
  "${CMAKE_BINARY_DIR}/amg-config.h"
)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O2")


add_executable(amg amg.c)
target_link_libraries(amg PRIVATE stdc++)

target_link_libraries(amg PUBLIC -L${MPIP_LIB_PATH} -lmpiP   ${AMG_DEPENDENCIES} m)
target_include_directories(amg PUBLIC  ${CMAKE_BINARY_DIR})

install(TARGETS amg DESTINATION ${CMAKE_INSTALL_BINDIR})
