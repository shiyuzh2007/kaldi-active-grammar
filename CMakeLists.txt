cmake_minimum_required(VERSION 3.13.0)
project(kaldi_binaries)

include(ExternalProject)
include(ProcessorCount)

ProcessorCount(NCPU)
if(NOT NCPU EQUAL 0)
  set(MAKE_FLAGS -j${NCPU})
endif()

set(DST ${PROJECT_SOURCE_DIR}/kaldi_active_grammar/exec)
if ("${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Darwin")
  set(DST ${DST}/macos/)
elseif("${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Linux")
  set(DST ${DST}/linux/)
else()
  set(DST ${DST}/windows/)
endif()

set(BINARIES
  tools/openfst/bin/fstarcsort${CMAKE_EXECUTABLE_SUFFIX}
  tools/openfst/bin/fstcompile${CMAKE_EXECUTABLE_SUFFIX}
  tools/openfst/bin/fstinfo${CMAKE_EXECUTABLE_SUFFIX}
  src/fstbin/fstaddselfloops${CMAKE_EXECUTABLE_SUFFIX}
  src/dragonflybin/compile-graph-agf${CMAKE_EXECUTABLE_SUFFIX})
set(LIBRARIES
  tools/openfst/lib/libfst${CMAKE_SHARED_LIBRARY_SUFFIX}
  tools/openfst/lib/libfstscript${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-base${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-chain${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-cudamatrix${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-decoder${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-dragonfly${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-feat${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-fstext${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-gmm${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-hmm${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-ivector${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-lat${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-lm${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-matrix${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-nnet2${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-nnet3${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-online2${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-transform${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-tree${CMAKE_SHARED_LIBRARY_SUFFIX}
  src/lib/libkaldi-util${CMAKE_SHARED_LIBRARY_SUFFIX})

if(DEFINED ENV{INTEL_MKL_DIR})
  set(INTEL_MKL_DIR $ENV{INTEL_MKL_DIR})
else()
  set(INTEL_MKL_DIR "/opt/intel/mkl/")
endif()
if(EXISTS "${INTEL_MKL_DIR}")
  set(KALDI_CONFIG_FLAGS --shared --static-math --mkl-root=${INTEL_MKL_DIR})
else()
  message("WARNING: cannot find INTEL_MKL_DIR: ${INTEL_MKL_DIR}")
  set(KALDI_CONFIG_FLAGS --shared --static-math)
endif()

set(ENV{CXXFLAGS} -O2)

find_program(MAKE_EXE NAMES make gmake nmake)

message("   MAKE_EXE              = ${MAKE_EXE}")
message("   PYTHON_EXECUTABLE     = ${PYTHON_EXECUTABLE}")
message("   PYTHON_INCLUDE_DIR    = ${PYTHON_INCLUDE_DIR}")
message("   PYTHON_LIBRARY        = ${PYTHON_LIBRARY}")
message("   PYTHON_VERSION_STRING = ${PYTHON_VERSION_STRING}")
message("   SKBUILD               = ${SKBUILD}")

if("${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Windows")
  message(FATAL_ERROR "CMake build not supported on Windows")
endif()

if(NOT "${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Windows")
  if("${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Darwin")
    set(STRIP_COMMAND find ${DST} -type f | xargs strip -x)
  else()
    set(STRIP_COMMAND find ${DST} -type f | xargs strip)
  endif()
  ExternalProject_Add(kaldi
    GIT_CONFIG        advice.detachedHead=false
    GIT_REPOSITORY    git@github.com:daanzu/kaldi-fork-active-grammar.git
    GIT_TAG           3f56242df335792756ca17f46a348ca2b36238d7
    CONFIGURE_COMMAND mkdir -p python && touch python/.use_default_python && tools/extras/check_dependencies.sh
    BUILD_IN_SOURCE   TRUE
    BUILD_COMMAND     cd tools && ${MAKE_EXE} ${MAKE_FLAGS} && cd openfst && autoreconf && cd ../../src && ./configure ${KALDI_CONFIG_FLAGS} && ${MAKE_EXE} ${MAKE_FLAGS} depend && ${MAKE_EXE} ${MAKE_FLAGS} dragonfly dragonflybin bin fstbin lmbin
    LIST_SEPARATOR    " "
    INSTALL_COMMAND   mkdir -p ${DST} && cp ${BINARIES} ${LIBRARIES} ${DST} && ${STRIP_COMMAND})
endif()

# Fix dynamic libraries loading paths on macOS.  The libraries and
# executables are built with RPATH settings embedded in them, pointing
# to the locations in temporary directories used to build the
# binaries.  After package installation is done, these directories are
# deleted and the dynamic libraries cannot be loaded.  The following
# commands generate a shell script that fixes the paths to the dynamic
# libraries in the built executables and the libraries themselves.
# Also the commands add a custom target to invoke the generated script
# after the external project (kaldi) has been built.  An alternative
# would be to change the kaldi engine build system to accept a path to
# where the binaries would be placed and point RPATH to that location.
if("${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Darwin")
  string (REPLACE ";" " " BINARIES_STR "${BINARIES}")
  string (REPLACE ";" " " LIBRARIES_STR "${LIBRARIES}")
  file(GENERATE OUTPUT name_fixer
    CONTENT
    "for a in ${BINARIES_STR} ; do
       a_bare=$\{a##*/\}
       install_name_tool -change ${PROJECT_BINARY_DIR}/kaldi-prefix/src/kaldi/tools/openfst-1.6.7/lib/libfst.10.dylib \"@loader_path/libfst.dylib\" ${DST}$a_bare
       install_name_tool -change ${PROJECT_BINARY_DIR}/kaldi-prefix/src/kaldi/tools/openfst-1.6.7/lib/libfstscript.10.dylib \"@loader_path/libfstscript.dylib\" ${DST}$a_bare
       for b in ${LIBRARIES_STR} ; do
          b_bare=$\{b##*/\}
         install_name_tool -change \"@rpath/$b_bare\" \"@loader_path/$b_bare\" ${DST}$a_bare
       done
     done
     for a in ${LIBRARIES_STR} ; do
       a_bare=$\{a##*/\}
       install_name_tool -id \"@loader_path/$a_bare\" ${DST}$a_bare
       install_name_tool -change ${PROJECT_BINARY_DIR}/kaldi-prefix/src/kaldi/tools/openfst-1.6.7/lib/libfst.10.dylib \"@loader_path/libfst.dylib\" ${DST}$a_bare
       install_name_tool -change ${PROJECT_BINARY_DIR}/kaldi-prefix/src/kaldi/tools/openfst-1.6.7/lib/libfstscript.10.dylib \"@loader_path/libfstscript.dylib\" ${DST}$a_bare
       for b in ${LIBRARIES_STR} ; do
          b_bare=$\{b##*/\}
         install_name_tool -change \"@rpath/$b_bare\" \"@loader_path/$b_bare\" ${DST}$a_bare
       done
     done")
  add_custom_target(fixer ALL COMMAND /bin/sh name_fixer)
  add_dependencies(fixer kaldi)
endif()

install(CODE "MESSAGE(\"Installed kaldi engine binaries.\")")
