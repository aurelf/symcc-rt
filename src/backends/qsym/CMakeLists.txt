# This file is part of the SymCC runtime.
#
# The SymCC runtime is free software: you can redistribute it and/or modify it
# under the terms of the GNU Lesser General Public License as published by the
# Free Software Foundation, either version 3 of the License, or (at your option)
# any later version.
#
# The SymCC runtime is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
# FITNESS FOR A PARTICULAR PURPOSE. See the GNU Lesser General Public License
# for more details.
#
# You should have received a copy of the GNU Lesser General Public License along
# with SymCC. If not, see <https://www.gnu.org/licenses/>.

# Build the parts of the Qsym backend that are relevant for us

set(QSYM_SOURCE_DIR "qsym/qsym/pintool")

if (NOT DEFINED LLVM_VERSION)
  message(FATAL_ERROR "LLVM version must be specified with the LLVM_VERSION variable.")
endif()

find_package(LLVM ${LLVM_VERSION} REQUIRED CONFIG)
add_definitions(${LLVM_DEFINITIONS})
include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})

# Qsym doesn't work with old versions of Z3
find_package(Z3 4.5 CONFIG)
if (NOT Z3_FOUND)
  if (NOT Z3_TRUST_SYSTEM_VERSION)
    message(FATAL_ERROR "Couldn't locate Z3. \
If you want me to trust that a suitable version is available nonetheless, \
configure CMake with -DZ3_TRUST_SYSTEM_VERSION=on (see also docs/Configuration.txt).")
  else()
    if (EXISTS "/usr/include/z3")
      set(Z3_CXX_INCLUDE_DIRS "/usr/include/z3")
    else()
      set(Z3_CXX_INCLUDE_DIRS)
    endif()
    set(Z3_LIBRARIES "z3")
  endif()
endif()

set(SymCCRtSrc
  ${QSYM_SOURCE_DIR}/expr.cpp
  ${QSYM_SOURCE_DIR}/expr_builder.cpp
  ${QSYM_SOURCE_DIR}/expr_cache.cpp
  ${QSYM_SOURCE_DIR}/expr_evaluate.cpp
  ${QSYM_SOURCE_DIR}/solver.cpp
  ${QSYM_SOURCE_DIR}/dependency.cpp
  ${QSYM_SOURCE_DIR}/logging.cpp
  ${QSYM_SOURCE_DIR}/afl_trace_map.cpp
  ${QSYM_SOURCE_DIR}/allocation.cpp
  ${QSYM_SOURCE_DIR}/call_stack_manager.cpp
  ${QSYM_SOURCE_DIR}/third_party/llvm/range.cpp
  ${QSYM_SOURCE_DIR}/third_party/xxhash/xxhash.cpp
  ${SHARED_RUNTIME_SOURCES}
  Runtime.cpp)

add_library(SymCCRtObj OBJECT
        ${SymCCRtSrc})

set_property(TARGET SymCCRtObj PROPERTY POSITION_INDEPENDENT_CODE 1)

add_library(SymCCRtShared SHARED $<TARGET_OBJECTS:SymCCRtObj>)
add_library(SymCCRtStatic STATIC $<TARGET_OBJECTS:SymCCRtObj>)

target_include_directories(SymCCRtObj PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}     # for our fake pin.H and Runtime.h
  ${SYMCC_RT_INCLUDE_DIR}) # for common runtime components

target_include_directories(SymCCRtObj SYSTEM PRIVATE
  ${Z3_CXX_INCLUDE_DIRS}        # for Z3 support
  ${QSYM_SOURCE_DIR})

# We need to get the LLVM support component for llvm::APInt.
llvm_map_components_to_libnames(QSYM_LLVM_DEPS support)

set(SymCCRtDeps ${Z3_LIBRARIES} ${QSYM_LLVM_DEPS})

# Object libraries cannot be linked directly, so we link the final libraries one by one
# https://gitlab.kitware.com/cmake/cmake/-/issues/18090
target_link_libraries(SymCCRtShared ${SymCCRtDeps})
target_link_libraries(SymCCRtStatic ${SymCCRtDeps})

# We use std::filesystem, which has been added in C++17. Before its official
# inclusion in the standard library, Clang shipped the feature first in
# libc++experimental and later in libc++fs; GCC used to require libstdc++fs. We
# make no attempt to support those older compiler versions, with one exception:
# some current LTS distributions ship a GCC that requires libstdc++fs for
# std::filesystem - we catch this case in order to enable users of such systems
# to build with the default compiler.
find_package(Filesystem COMPONENTS Final Experimental)
target_link_libraries(SymCCRtObj std::filesystem)
