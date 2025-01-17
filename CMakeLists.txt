cmake_minimum_required(VERSION 2.8)
project(ReweightedARAP)

set(CMAKE_COLOR_MAKEFILE ON)
include(CheckCXXCompilerFlag)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

if(NOT TARGET uninstall)
  configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake IMMEDIATE @ONLY)

  add_custom_target(
    uninstall COMMAND ${CMAKE_COMMAND} -P
                      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif()

export(TARGETS FILE ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}Targets.cmake)

file(WRITE ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}Config.cmake
     "include(CMakeFindDependencyMacro)\n"
     "include(\${CMAKE_CURRENT_LIST_DIR}/${CMAKE_PROJECT_NAME}Targets.cmake)\n")

# Set a default build configuration if none is specified.
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
    "MinSizeRel" "RelWithDebInfo")
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  CHECK_CXX_COMPILER_FLAG("-std=c++11" HAS_CPP11_FLAG)

  if (HAS_CPP11_FLAG)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  else()
    message(FATAL_ERROR "Unsupported compiler -- ReweightedARAP requires C++11 support!")
  endif()

  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-result -Wno-deprecated-declarations -fPIC")
endif()

if (MSVC)
  # Enable parallel compilation for Visual Studio
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP /bigobj") 

  # Enable folders for projects in Visual Studio
  set_property(GLOBAL PROPERTY USE_FOLDERS ON)
endif()

set(CMAKE_MACOSX_RPATH 1)

# Check to see if PARDISO is available
find_library(PARDISO
  pardiso500-INTEL1301-X86-64
  pardiso500-GNU461-X86-64
  pardiso500-GNU472-X86-64
  pardiso500-GNU481-X86-64
  pardiso500-MPI-INTEL1301-X86-64
  pardiso500-MPI-GNU450-X86-64
  pardiso500-MPI-GNU461-X86-64
  pardiso500-MPI-GNU463-X86-64
  pardiso500-MPI-GNU472-X86-64
  pardiso500-WIN-X86-64
  pardiso500-MACOS-X86-64
  PATHS ${CMAKE_CURRENT_SOURCE_DIR}/ext/pardiso
)

set(ADDITIONAL_LIBRARIES "")

if("${CMAKE_SYSTEM}" MATCHES "Linux")
  set(ADDITIONAL_LIBRARIES rt pthread dl)
endif()

if (${PARDISO} MATCHES "NOTFOUND")
  message("\n*NOT* using PARDISO -- note that the performance will be considerably slower than what is shown in the paper.\n")
else()
  message("Using PARDISO library at ${PARDISO}")
  add_definitions(-DUSE_PARDISO)
  set(ADDITIONAL_LIBRARIES ${PARDISO})
  set(ADDITIONAL_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/src/PardisoSolver.cpp
  )
  set(ADDITIONAL_INCLUDES
	  ${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/PardisoSolver.h
  )
endif()

set(SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/src/main.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/eigen_stl_utils.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/parametrization_utils.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/Param_State.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/LocalWeightedArapParametrizer.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/GlobalLocalParametrization.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/LinesearchParametrizer.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/SymmetricDirichlet.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/FastLsBuildUtils.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/StateManager.cpp
  ${ADDITIONAL_SOURCES}
)

set(INCLUDES
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/eigen_stl_utils.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/parametrization_utils.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/Param_State.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/LocalWeightedArapParametrizer.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/GlobalLocalParametrization.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/LinesearchParametrizer.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/SymmetricDirichlet.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/FastLsBuildUtils.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/StateManager.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/Energy.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/StateManager.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/AlgorithmStage.h
	${ADDITIONAL_INCLUDES}
)

find_package(Eigen3 3.3 REQUIRED NO_MODULE)
find_package(libigl REQUIRED)
find_package(TBB REQUIRED)

add_library(${PROJECT_NAME} SHARED ${SOURCES} ${INCLUDES})

target_include_directories(
	${PROJECT_NAME} PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
                         $<INSTALL_INTERFACE:${CMAKE_INSTALL_PREFIX}/include>)

target_include_directories(${PROJECT_NAME} SYSTEM PUBLIC ${TBB_INCLUDE_DIRS})

target_link_libraries(${PROJECT_NAME} ${TBB_LIBRARIES} Eigen3::Eigen igl::core igl::common ${ADDITIONAL_LIBRARIES})

install(
  TARGETS ${PROJECT_NAME}
  EXPORT ${CMAKE_PROJECT_NAME}Targets
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin)

install(FILES
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/eigen_stl_utils.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/parametrization_utils.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/Param_State.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/LocalWeightedArapParametrizer.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/GlobalLocalParametrization.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/LinesearchParametrizer.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/SymmetricDirichlet.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/FastLsBuildUtils.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/StateManager.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/Energy.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/StateManager.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}/AlgorithmStage.h
	${ADDITIONAL_INCLUDES}
	DESTINATION include/${PROJECT_NAME})

export(TARGETS ${PROJECT_NAME} APPEND
       FILE ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}Targets.cmake)

file(APPEND ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}Config.cmake
     "find_package(Eigen3 3.3 REQUIRED NO_MODULE)\n"
     "find_package(libigl REQUIRED)\n"
     )

install(FILES ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}Config.cmake
        DESTINATION lib/cmake/${CMAKE_PROJECT_NAME})

install(EXPORT ${CMAKE_PROJECT_NAME}Targets NAMESPACE ReweightedARAP::
        DESTINATION lib/cmake/${CMAKE_PROJECT_NAME})
