cmake_minimum_required(VERSION 3.20.6...3.22.6 FATAL_ERROR)

project(FindVcvarsTests LANGUAGES NONE)

set(FindVcvars_MODULE_PATH "${CMAKE_SOURCE_DIR}/..")

include(CTest)

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/test-script-mode.cmake "\
cmake_minimum_required(VERSION ${CMAKE_VERSION} FATAL_ERROR)
list(INSERT CMAKE_MODULE_PATH 0 \"${FindVcvars_MODULE_PATH}\")
find_package(Vcvars REQUIRED)
")

add_test(
  NAME script-mode
  COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/test-script-mode.cmake
  )

