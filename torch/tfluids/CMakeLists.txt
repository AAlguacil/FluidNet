CMAKE_MINIMUM_REQUIRED(VERSION 2.6 FATAL_ERROR)
CMAKE_POLICY(VERSION 2.6)

SET(CMAKE_BUILD_TYPE Release)
SET(CMAKE_VERBOSE_MAKEFILE false)

OPTION(WITH_OPENGL "Enable building OpenGL functions" ON)
OPTION(WITH_CUDA "Enable cuda functions (note: nvcc still required!)" ON)

FIND_PACKAGE(Torch REQUIRED)
FIND_PACKAGE(OpenMP)
IF(OPENMP_FOUND)
  MESSAGE(STATUS "Compiling with OpenMP support")
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
ENDIF(OPENMP_FOUND)

FIND_PACKAGE(CUDA 7.5 REQUIRED)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/generic)
LIST(APPEND CUDA_NVCC_FLAGS "-arch=sm_61;--use_fast_math; -D_FORCE_INLINES")
MESSAGE("Compiling for CUDA architecture 6.1")
FIND_PACKAGE(GLUT REQUIRED)
FIND_PACKAGE(OpenGL REQUIRED)

INCLUDE_DIRECTORIES("${Torch_INSTALL_INCLUDE}")
INCLUDE_DIRECTORIES("${Torch_INSTALL_INCLUDE}/TH")
IF (WITH_CUDA)
  INCLUDE_DIRECTORIES("${Torch_INSTALL_INCLUDE}/THC")
ENDIF()
LINK_DIRECTORIES("${Torch_INSTALL_LIB}")

IF (WITH_OPENGL)
  MESSAGE("Compiling with OpenGL support")
  INCLUDE_DIRECTORIES(${GLUT_INCLUDE_DIRS})
  LINK_DIRECTORIES(${GLUT_LIBRARY_DIRS})
  ADD_DEFINITIONS(${GLUT_DEFINITIONS})
  INCLUDE_DIRECTORIES(${OpenGL_INCLUDE_DIRS})
  LINK_DIRECTORIES(${OpenGL_LIBRARY_DIRS})
  ADD_DEFINITIONS(${OpenGL_DEFINITIONS})
ELSE()
  MESSAGE("Compiling without OpenGL support")
  SET(GLUT_LIBRARY "")
  SET(OPENGL_LIBRARIES "")
ENDIF()

# THIS IS FOR CUDA 7.5 and gcc 5.x
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -fPIC -D_MWAITXINTRIN_H_INCLUDED -D_FORCE_INLINE -D__STRICT_ANSI__")

IF (WITH_OPENGL)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DBUILD_GL_FUNCS")
ENDIF()

IF (WITH_CUDA)
  MESSAGE("Compiling with CUDA support.")
ELSE()
  MESSAGE("Compiling without CUDA support.")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DBUILD_WITHOUT_CUDA_FUNCS")
ENDIF()

SET(luasrc init.lua flags_to_occupancy.lua set_wall_bcs.lua test_tfluids.lua velocity_divergence.lua velocity_update.lua  volumetric_up_sampling_nearest.lua)

if (WITH_CUDA)
  SET(src init.cu generic/tfluids.cu generic/tfluids.cu.h generic/advect_type.cc)
ELSE()
  SET(src init.cu generic/advect_type.cc)
ENDIF()

ADD_TORCH_PACKAGE(tfluids "${src}" "${luasrc}" "Torch Fluids utility lib.")
TARGET_LINK_LIBRARIES(tfluids luaT TH ${OPENGL_LIBRARIES} ${GLUT_LIBRARY} ${CUDA_cusparse_LIBRARY})

