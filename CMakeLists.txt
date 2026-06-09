cmake_minimum_required(VERSION 3.20)

# Enable CUDA only if requested so the project can be configured without a GPU
option(WITH_CUDA "Build openmoc.cuda CUDA extension" ON)
if(WITH_CUDA)
  project(OpenMOC CXX CUDA)
else()
  project(OpenMOC CXX)
endif()

# ── Options ───────────────────────────────────────────────────────────────────
option(WITH_TESTS "Include CTest regression tests" OFF)
set(FP "single" CACHE STRING "Floating-point precision: single | double")

# ── Python ────────────────────────────────────────────────────────────────────
# Point at a pixi/conda env with -DPython3_ROOT_DIR=/path/to/.pixi/envs/default
find_package(Python3 REQUIRED COMPONENTS Interpreter Development NumPy)

# EXT_SUFFIX (.cpython-314-x86_64-linux-gnu.so) — fall back to asking Python
if(NOT Python3_EXT_SUFFIX)
  execute_process(
    COMMAND "${Python3_EXECUTABLE}" -c
      "import sysconfig; print(sysconfig.get_config_var('EXT_SUFFIX'), end='')"
    OUTPUT_VARIABLE Python3_EXT_SUFFIX
  )
endif()

# ── Language standards & PIC ──────────────────────────────────────────────────
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# ── Compile-time macros ───────────────────────────────────────────────────────
if(FP STREQUAL "double")
  set(FP_DEFS FP_PRECISION=double CMFD_PRECISION=double "LINALG_TOL=1e-15")
  set(VEC_LEN 4)
else()
  set(FP_DEFS FP_PRECISION=float CMFD_PRECISION=float "LINALG_TOL=1e-7" SINGLE)
  set(VEC_LEN 8)
endif()
set(COMMON_DEFS ${FP_DEFS} "VEC_ALIGNMENT=64" "VEC_LENGTH=${VEC_LEN}" OPENMP SWIG)

# ── CPU sources ───────────────────────────────────────────────────────────────
set(CPU_SRCS
  src/Cell.cpp              src/Cmfd.cpp              src/CPUSolver.cpp
  src/CPULSSolver.cpp       src/ExpEvaluator.cpp      src/Geometry.cpp
  src/linalg.cpp            src/LocalCoords.cpp       src/log.cpp
  src/Material.cpp          src/Matrix.cpp            src/Mesh.cpp
  src/MOCKernel.cpp         src/Point.cpp             src/Progress.cpp
  src/Quadrature.cpp        src/Region.cpp            src/RunTime.cpp
  src/Solver.cpp            src/Surface.cpp           src/Timer.cpp
  src/Track.cpp             src/Track3D.cpp           src/TrackGenerator.cpp
  src/TrackGenerator3D.cpp  src/TrackTraversingAlgorithms.cpp
  src/TraverseSegments.cpp  src/Universe.cpp          src/Vector.cpp
)

# ── _openmoc (CPU Python extension) ──────────────────────────────────────────
# Built as SHARED (not MODULE) so _openmoc_cuda can link against it.
# PREFIX/SUFFIX are overridden to match Python's expected naming convention.
# The SWIG-generated openmoc_wrap.cpp is already committed to the source tree.
add_library(_openmoc SHARED
  openmoc/swig/openmoc_wrap.cpp
  ${CPU_SRCS}
)
set_target_properties(_openmoc PROPERTIES
  PREFIX ""
  SUFFIX "${Python3_EXT_SUFFIX}"
)
target_compile_options(_openmoc PRIVATE -O3 -ffast-math -fopenmp -march=native)
target_compile_definitions(_openmoc PRIVATE GCC ${COMMON_DEFS})
target_include_directories(_openmoc PRIVATE
  src
  ${Python3_INCLUDE_DIRS}
  ${Python3_NumPy_INCLUDE_DIRS}
)
target_link_options(_openmoc PRIVATE -fopenmp)

# ── _openmoc_cuda (GPU Python extension) ─────────────────────────────────────
if(WITH_CUDA)
  find_package(CUDAToolkit REQUIRED)
  # CCCL ships its own CMake config inside the CUDA toolkit; derive the path
  # from CUDAToolkit_LIBRARY_DIR so we don't hardcode /usr/local/cuda-X.Y.
  find_package(CCCL CONFIG REQUIRED
    PATHS "${CUDAToolkit_LIBRARY_DIR}/cmake/cccl" NO_DEFAULT_PATH)

  # openmoc_cuda_wrap.cpp includes GPUSolver.h → Thrust/CCCL headers that only
  # nvcc can parse. LANGUAGE CUDA makes CMake route this .cpp file through nvcc
  # instead of g++, which is the entire reason setup.py's routing was needed.
  set_source_files_properties(
    openmoc/cuda/openmoc_cuda_wrap.cpp
    PROPERTIES LANGUAGE CUDA
  )

  add_library(_openmoc_cuda MODULE
    openmoc/cuda/openmoc_cuda_wrap.cpp
    src/accel/cuda/GPUQuery.cu
    src/accel/cuda/clone.cu
    src/accel/cuda/GPUSolver.cu
  )
  set_target_properties(_openmoc_cuda PROPERTIES
    PREFIX                     ""
    SUFFIX                     "${Python3_EXT_SUFFIX}"
    CUDA_SEPARABLE_COMPILATION ON   # enables --relocatable-device-code per TU
    CUDA_RESOLVE_DEVICE_SYMBOLS ON  # adds nvcc --dlink step before host link
    # At install time, find _openmoc.so one directory up (openmoc/ vs openmoc/cuda/)
    INSTALL_RPATH              "$ORIGIN/.."
    BUILD_RPATH                "${CMAKE_BINARY_DIR}"
  )
  target_compile_options(_openmoc_cuda PRIVATE
    $<$<COMPILE_LANGUAGE:CUDA>:-O3 --use_fast_math -Xcompiler=-fpic>
  )
  target_compile_definitions(_openmoc_cuda PRIVATE NVCC ${COMMON_DEFS})
  target_include_directories(_openmoc_cuda PRIVATE
    src
    ${Python3_INCLUDE_DIRS}
    ${Python3_NumPy_INCLUDE_DIRS}
  )
  # Links against _openmoc for CPUSolver/Solver/Geometry symbols,
  # mirroring setup.py's linker_flags['nvcc'] = ['-shared', get_openmoc()].
  # cudadevrt is linked automatically by CMake when CUDA_SEPARABLE_COMPILATION
  # is ON; only cudart needs to be listed explicitly.
  # CCCL::CCCL is the proper CMake target for Thrust/CUB/libcu++ headers
  # (preferred over manually adding /usr/local/cuda/include/cccl).
  target_link_libraries(_openmoc_cuda PRIVATE
    _openmoc
    CUDA::cudart
    CCCL::CCCL
  )
endif()

# ── Install ───────────────────────────────────────────────────────────────────
set(PKG_DEST "${Python3_SITEARCH}/openmoc")

install(TARGETS _openmoc LIBRARY DESTINATION "${PKG_DEST}")
install(DIRECTORY openmoc/ DESTINATION "${PKG_DEST}"
  FILES_MATCHING PATTERN "*.py"
  PATTERN "swig"        EXCLUDE
  PATTERN "cuda"        EXCLUDE
  PATTERN "__pycache__" EXCLUDE
)
if(WITH_CUDA)
  install(TARGETS _openmoc_cuda LIBRARY DESTINATION "${PKG_DEST}/cuda")
  install(DIRECTORY openmoc/cuda/ DESTINATION "${PKG_DEST}/cuda"
    FILES_MATCHING PATTERN "*.py"
    PATTERN "__pycache__" EXCLUDE
  )
endif()

# ── Tests (optional, matches original CMakeLists.txt behaviour) ───────────────
if(WITH_TESTS)
  include(CTest)
  file(GLOB_RECURSE TESTS ${CMAKE_CURRENT_SOURCE_DIR}/tests/test_*.py)
  foreach(test ${TESTS})
    get_filename_component(TEST_NAME ${test} NAME)
    get_filename_component(TEST_PATH ${test} PATH)
    add_test(NAME ${TEST_NAME}
      WORKING_DIRECTORY ${TEST_PATH}
      COMMAND ${Python3_EXECUTABLE} ${TEST_NAME}
    )
  endforeach()
endif()
