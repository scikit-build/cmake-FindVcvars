cmake_minimum_required(VERSION 3.20.6...3.22.6 FATAL_ERROR)

project(FindVcvarsTests LANGUAGES NONE)

foreach(var_name IN ITEMS
  EXPECTED_MSVC_ARCH
  EXPECTED_MSVC_VERSION
  EXPECTED_BATCH_FILENAME
  EXPECTED_WRAPPER_FILENAME
  )
  if(NOT DEFINED ${var_name})
    message(FATAL_ERROR "Variable ${var_name} is not specified using -D${var_name}=<value>")
  endif()
endforeach()

set(FindVcvars_MODULE_PATH "${CMAKE_SOURCE_DIR}/..")

include(CTest)

# Testing logic common to both script and project modes
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/test-find-vcvars.cmake.in [==[
cmake_minimum_required(VERSION @CMAKE_VERSION@ FATAL_ERROR)

if(CMAKE_ROLE STREQUAL "PROJECT")
  project(TestFindVcvars)
endif()

list(INSERT CMAKE_MODULE_PATH 0 "@FindVcvars_MODULE_PATH@")

# Check expected defined variables
set(expected_vars
  EXPECTED_MSVC_ARCH
  EXPECTED_MSVC_VERSION
  EXPECTED_BATCH_FILENAME
  EXPECTED_WRAPPER_FILENAME
  )
foreach(var_name IN LISTS expected_vars)
  if(NOT DEFINED ${var_name})
    message(FATAL_ERROR "Variable ${var_name} is not defined using -D${var_name}=<value>")
  endif()
endforeach()

# Set Vcvars_<varname> based of REQUESTED_<varname>
set(requested_vars)

if(DEFINED REQUESTED_MSVC_ARCH)
  if(REQUESTED_MSVC_ARCH STREQUAL "64")
    set(CMAKE_SIZEOF_VOID_P 8)
  elseif(REQUESTED_MSVC_ARCH STREQUAL "32")
    set(CMAKE_SIZEOF_VOID_P 4)
  else()
    message(FATAL_ERROR "Variable REQUESTED_MSVC_ARCH is invalid. Expected value 32 or 64")
  endif()
  list(APPEND requested_vars REQUESTED_MSVC_ARCH)
endif()

if(DEFINED REQUESTED_MSVC_VERSION)
  set(Vcvars_MSVC_VERSION ${REQUESTED_MSVC_VERSION})
  list(APPEND requested_vars REQUESTED_MSVC_VERSION)
endif()

foreach(var_suffix IN LISTS
  MSVC_ARCH
  MSVC_VERSION
  )
  if(DEFINED REQUESTED_${var_suffix})
    set(Vcvars_${var_suffix} ${REQUESTED_${var_suffix}})
    list(APPEND requested_vars REQUESTED_${var_suffix})
  endif()
endforeach()

# Display input variables
foreach(var_name IN LISTS requested_vars expected_vars)
  if(DEFINED ${var_name})
    message(STATUS "Tests: ${var_name} [${${var_name}}]")
  endif()
endforeach()

message(STATUS "")

find_package(Vcvars REQUIRED)

message(STATUS "")

function(check_var_defined var_name)
  if(NOT DEFINED ${var_name})
    message(FATAL_ERROR "Variable ${var_name} is not defined in the calling scope")
  endif()
endfunction()

function(check_var_equals var_name expected_value)
  check_var_defined(${var_name})

  set(var_value ${${var_name}})
  if(NOT var_value STREQUAL expected_value)
    message(FATAL_ERROR "Variable ${var_name} value does not match expected value
       current_value [${var_value}]
      expected_value [${expected_value}]
    ")
  endif()
endfunction()

function(check_file_exists var_name)
  check_var_defined(${var_name})
  set(var_value ${${var_name}})
  if(NOT EXISTS "${var_value}")
    message(FATAL_ERROR "Variable ${var_name} is defined but corresponds to nonexistent file [${var_value}]")
  endif()
endfunction()

function(check_filename_matches var_name expected_filename)
  check_var_defined(${var_name})
  set(var_value ${${var_name}})
  if(NOT ${var_value} MATCHES "${expected_filename}$")
    message(FATAL_ERROR "Variable ${var_name} is defined but os not set to '${expected_filename}' file [${var_value}]")
  endif()
endfunction()

# Display output variables
set(output_vars
  Vcvars_MSVC_ARCH
  Vcvars_MSVC_VERSION
  Vcvars_BATCH_FILE
  Vcvars_LAUNCHER
  )
foreach(var_name IN LISTS output_vars)
  message(STATUS "Tests: ${var_name} is '${${var_name}}'")
endforeach()

# Check output values
message(STATUS "")

check_var_equals("Vcvars_MSVC_ARCH" "${EXPECTED_MSVC_ARCH}")
check_var_equals("Vcvars_MSVC_VERSION" "${EXPECTED_MSVC_VERSION}")

check_file_exists("Vcvars_BATCH_FILE")
check_filename_matches("Vcvars_BATCH_FILE" "${EXPECTED_BATCH_FILENAME}")

check_file_exists("Vcvars_LAUNCHER")
check_filename_matches("Vcvars_LAUNCHER" "${EXPECTED_LAUNCHER_FILENAME}")
]==]
)

# script-mode
set(script_mode_script "${CMAKE_CURRENT_BINARY_DIR}/test-script-mode.cmake")
configure_file(
  ${CMAKE_CURRENT_BINARY_DIR}/test-find-vcvars.cmake.in
  ${script_mode_script}
  @ONLY
  )

# project-mode
set(project_mode_src_dir "${CMAKE_CURRENT_BINARY_DIR}/test-find-vcvars")
configure_file(
  ${CMAKE_CURRENT_BINARY_DIR}/test-find-vcvars.cmake.in
  ${project_mode_src_dir}/CMakeLists.txt
  @ONLY
  )

function(add_find_vcvars_test)
  set(options )
  set(oneValueArgs
    NAME
    )
  set(multiValueArgs
    OPTIONS
    )
  cmake_parse_arguments(PARSE_ARGV 0 arg
    "${options}" "${oneValueArgs}" "${multiValueArgs}"
    )

  # script-mode
  add_test(
    NAME "script-mode-${arg_NAME}"
    COMMAND ${CMAKE_COMMAND} ${arg_OPTIONS} -P ${script_mode_script}
    )

  # project-mode
  set(src_dir "${project_mode_src_dir}")
  set(bld_dir "${project_mode_src_dir}-${arg_NAME}-build")
  add_test(
    NAME "project-mode-${arg_NAME}"
    COMMAND ${CMAKE_COMMAND} ${arg_OPTIONS} -S ${src_dir} -B ${bld_dir}
    )
  set_tests_properties("project-mode-${arg_NAME}"  PROPERTIES FIXTURES_REQUIRED "fixture-${arg_NAME}")
  add_test(
    NAME "project-mode-${arg_NAME}-setup"
    COMMAND ${CMAKE_COMMAND} -E rm -rf ${bld_dir}
    )
  set_tests_properties("project-mode-${arg_NAME}-setup"  PROPERTIES FIXTURES_SETUP "fixture-${arg_NAME}")
endfunction()

add_find_vcvars_test(
  NAME "defaults"
  OPTIONS
    # Requested
    # NA
    # Expected
    -DEXPECTED_MSVC_ARCH:STRING=${EXPECTED_MSVC_ARCH}
    -DEXPECTED_MSVC_VERSION:STRING=${EXPECTED_MSVC_VERSION}
    -DEXPECTED_BATCH_FILENAME:STRING=${EXPECTED_BATCH_FILENAME}
    -DEXPECTED_WRAPPER_FILENAME:STRING=${EXPECTED_WRAPPER_FILENAME}
  )

add_find_vcvars_test(
  NAME "set-arch-64"
  OPTIONS
    # Requested
    -DREQUESTED_MSVC_ARCH:STRING=64
    # Expected
    -DEXPECTED_MSVC_ARCH:STRING=64
    -DEXPECTED_MSVC_VERSION:STRING=${EXPECTED_MSVC_VERSION}
    -DEXPECTED_BATCH_FILENAME:STRING=vcvars64.bat
    -DEXPECTED_WRAPPER_FILENAME:STRING=vcvars64_wrapper.bat
  )

add_find_vcvars_test(
  NAME "set-arch-32"
  OPTIONS
    # Requested
    -DREQUESTED_MSVC_ARCH:STRING=32
    # Expected
    -DEXPECTED_MSVC_ARCH:STRING=32
    -DEXPECTED_MSVC_VERSION:STRING=${EXPECTED_MSVC_VERSION}
    -DEXPECTED_BATCH_FILENAME:STRING=vcvars32.bat
    -DEXPECTED_WRAPPER_FILENAME:STRING=vcvars32_wrapper.bat
  )

add_find_vcvars_test(
  NAME "set-version-${EXPECTED_MSVC_ARCH}"
  OPTIONS
    # Requested
    -DREQUESTED_MSVC_VERSION:STRING=${EXPECTED_MSVC_VERSION}
    # Expected
    -DEXPECTED_MSVC_ARCH:STRING=${EXPECTED_MSVC_ARCH}
    -DEXPECTED_MSVC_VERSION:STRING=${EXPECTED_MSVC_VERSION}
    -DEXPECTED_BATCH_FILENAME:STRING=${EXPECTED_BATCH_FILENAME}
    -DEXPECTED_WRAPPER_FILENAME:STRING=${EXPECTED_WRAPPER_FILENAME}
  )

