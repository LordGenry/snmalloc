cmake_minimum_required(VERSION 3.8)
project(snmalloc C CXX)

option(USE_SNMALLOC_STATS "Track allocation stats" OFF)
option(USE_MEASURE "Measure performance with histograms" OFF)
option(USE_SBRK "Use sbrk instead of mmap" OFF)

macro(subdirlist result curdir)
  file(GLOB children LIST_DIRECTORIES true RELATIVE ${curdir} ${curdir}/*)
  set(dirlist "")
  foreach(child ${children})
    if(IS_DIRECTORY ${curdir}/${child})
      list(APPEND dirlist ${child})
    endif()
  endforeach()
  set(${result} ${dirlist})
endmacro()

macro(linklibs project)
  if(NOT MSVC)
    target_link_libraries(${project} ${CMAKE_THREAD_LIBS_INIT})
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
      target_link_libraries(${project} atomic)
    endif()
  endif()
endmacro()

if(MSVC)
  add_compile_options(/WX /W4 /wd4127 /wd4324 /wd4201 /std:c++latest)
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Zi")
  set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /DEBUG")
else()
  find_package(Threads REQUIRED)
  add_compile_options(-mcx16 -march=native -Wall -Wextra -Werror -g)
endif()

set(CMAKE_CXX_STANDARD 17)

if(USE_SNMALLOC_STATS)
  add_definitions(-DUSE_SNMALLOC_STATS)
endif()

if(USE_MEASURE)
  add_definitions(-DUSE_MEASURE)
endif()

if(USE_SBRK)
  add_definitions(-DUSE_SBRK)
endif()

if(NOT MSVC)
  add_library(snmallocshim SHARED src/override/malloc.cc)
  target_link_libraries(snmallocshim -pthread)
  target_include_directories(snmallocshim PRIVATE src)
endif()

enable_testing()

set(TESTDIR ${CMAKE_CURRENT_SOURCE_DIR}/src/test)
subdirlist(TEST_CATEGORIES ${TESTDIR})
foreach(TEST_CATEGORY ${TEST_CATEGORIES})
  subdirlist(TESTS ${TESTDIR}/${TEST_CATEGORY})
  foreach(TEST ${TESTS})
    unset(SRC)
    aux_source_directory(${TESTDIR}/${TEST_CATEGORY}/${TEST} SRC)
    set(TESTNAME "${TEST_CATEGORY}-${TEST}")
    add_executable(${TESTNAME} ${SRC} src/override/new.cc)
    target_include_directories(${TESTNAME} PRIVATE src)
    linklibs(${TESTNAME})
    add_test(${TESTNAME} ${TESTNAME})
  endforeach()
endforeach()


# The clang-format tool is installed under a variety of different names.  Try
# to find a sensible one.  Only look for 6.0 and 7.0 versions explicitly - we
# don't know whether our clang-format file will work with newer versions of the
# tool
set(CLANG_FORMAT_NAMES
  clang-format-7.0
  clang-format-6.0
  clang-format70
  clang-format60
  clang-format)

# Loop over each of the possible names of clang-format and try to find one.
set(CLANG_FORMAT CLANG_FORMAT-NOTFOUND)
foreach (NAME IN ITEMS ${CLANG_FORMAT_NAMES})
  if (${CLANG_FORMAT} STREQUAL "CLANG_FORMAT-NOTFOUND")
    find_program(CLANG_FORMAT ${NAME})
  endif ()
endforeach()

# If we've found a clang-format tool, generate a target for it, otherwise emit
# a warning.
if (${CLANG_FORMAT} STREQUAL "CLANG_FORMAT-NOTFOUND")
  message(WARNING "Not generating clangformat target, no clang-format tool found")
else ()
  message(STATUS "Generating clangformat target using ${CLANG_FORMAT}")
  file(GLOB_RECURSE ALL_SOURCE_FILES *.cc *.h *.hh)
  add_custom_target(
          clangformat
          COMMAND ${CLANG_FORMAT}
          -i
          ${ALL_SOURCE_FILES}
  )
endif()
