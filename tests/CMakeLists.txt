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
  FIND_VCVARS_COMPONENTS
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

if("FunctionsOnly" IN_LIST FIND_VCVARS_COMPONENTS)
  set(FUNCTIONS_ONLY_COMPONENT_REQUESTED TRUE)
else()
  set(FUNCTIONS_ONLY_COMPONENT_REQUESTED FALSE)
endif()

# Set Vcvars_<varname> based of REQUESTED_<varname>
set(requested_vars)

if(DEFINED REQUESTED_MSVC_ARCH AND NOT FUNCTIONS_ONLY_COMPONENT_REQUESTED)
  if(REQUESTED_MSVC_ARCH STREQUAL "64")
    set(CMAKE_SIZEOF_VOID_P 8)
  elseif(REQUESTED_MSVC_ARCH STREQUAL "32")
    set(CMAKE_SIZEOF_VOID_P 4)
  else()
    message(FATAL_ERROR "Variable REQUESTED_MSVC_ARCH is invalid. Expected value 32 or 64")
  endif()
  list(APPEND requested_vars REQUESTED_MSVC_ARCH)
endif()

if(DEFINED REQUESTED_MSVC_VERSION AND NOT FUNCTIONS_ONLY_COMPONENT_REQUESTED)
  set(Vcvars_MSVC_VERSION ${REQUESTED_MSVC_VERSION})
  list(APPEND requested_vars REQUESTED_MSVC_VERSION)
endif()

if(DEFINED REQUESTED_FIND_VCVARSALL AND NOT FUNCTIONS_ONLY_COMPONENT_REQUESTED)
  set(Vcvars_FIND_VCVARSALL ${REQUESTED_FIND_VCVARSALL})
  list(APPEND requested_vars REQUESTED_FIND_VCVARSALL)
endif()

foreach(var_suffix IN LISTS
  FIND_VCVARSALL
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

if(FUNCTIONS_ONLY_COMPONENT_REQUESTED)
  find_package(Vcvars COMPONENTS FunctionsOnly REQUIRED)
else()
  find_package(Vcvars ${components_arg} REQUIRED)
endif()

message(STATUS "")

function(check_function_defined func_name)
  if(NOT COMMAND "${func_name}")
    message(FATAL_ERROR "Function ${func_name} is not defined in the calling scope")
  endif()
endfunction()

function(check_var_defined var_name)
  if(NOT DEFINED "${var_name}")
    message(FATAL_ERROR "Variable ${var_name} is not defined in the calling scope")
  endif()
endfunction()

function(check_var_not_defined var_name)
  if(DEFINED "${var_name}")
    message(FATAL_ERROR "Variable ${var_name} is defined in the calling scope")
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

check_var_equals("Vcvars_FOUND" "TRUE")

check_function_defined("Vcvars_ConvertMsvcVersionToVsVersion")
check_function_defined("Vcvars_GetVisualStudioPaths")
check_function_defined("Vcvars_FindFirstValidMsvcVersion")

# VS 2022
check_var_equals(Vcvars_TOOLSET_143_MSVC_VERSIONS "1949;1948;1947;1946;1945;1944;1943;1942;1941;1940;1939;1938;1937;1936;1935;1934;1933;1932;1931;1930")
# VS 2019
check_var_equals(Vcvars_TOOLSET_142_MSVC_VERSIONS "1929;1928;1927;1926;1925;1924;1923;1922;1921;1920")
# VS 2017
check_var_equals(Vcvars_TOOLSET_141_MSVC_VERSIONS "1916;1915;1914;1913;1912;1911;1910")
check_var_equals(Vcvars_TOOLSET_140_MSVC_VERSIONS "1900") # VS 2015
check_var_equals(Vcvars_TOOLSET_120_MSVC_VERSIONS "1800") # VS 2013
check_var_equals(Vcvars_TOOLSET_110_MSVC_VERSIONS "1700") # VS 2012
check_var_equals(Vcvars_TOOLSET_100_MSVC_VERSIONS "1600") # VS 2010
check_var_equals(Vcvars_TOOLSET_90_MSVC_VERSIONS "1500") # VS 2008
check_var_equals(Vcvars_TOOLSET_80_MSVC_VERSIONS "1400") # VS 2005
check_var_equals(Vcvars_TOOLSET_71_MSVC_VERSIONS "1310") # VS 2003
check_var_equals(Vcvars_TOOLSET_70_MSVC_VERSIONS "1300") # VS 2002
check_var_equals(Vcvars_TOOLSET_60_MSVC_VERSIONS "1200") # VS 6.0

check_var_equals(Vcvars_VS17_MSVC_VERSIONS "${Vcvars_TOOLSET_143_MSVC_VERSIONS}") # VS 2022
check_var_equals(Vcvars_VS16_MSVC_VERSIONS "${Vcvars_TOOLSET_142_MSVC_VERSIONS}") # VS 2019
check_var_equals(Vcvars_VS15_MSVC_VERSIONS "${Vcvars_TOOLSET_141_MSVC_VERSIONS}") # VS 2017
check_var_equals(Vcvars_VS14_MSVC_VERSIONS "${Vcvars_TOOLSET_140_MSVC_VERSIONS}") # VS 2015
check_var_equals(Vcvars_VS12_MSVC_VERSIONS "${Vcvars_TOOLSET_120_MSVC_VERSIONS}") # VS 2013
check_var_equals(Vcvars_VS11_MSVC_VERSIONS "${Vcvars_TOOLSET_110_MSVC_VERSIONS}") # VS 2012
check_var_equals(Vcvars_VS10_MSVC_VERSIONS "${Vcvars_TOOLSET_100_MSVC_VERSIONS}") # VS 2010
check_var_equals(Vcvars_VS9_MSVC_VERSIONS "${Vcvars_TOOLSET_90_MSVC_VERSIONS}") # VS 2008
check_var_equals(Vcvars_VS8_MSVC_VERSIONS "${Vcvars_TOOLSET_80_MSVC_VERSIONS}") # VS 2005
check_var_equals(Vcvars_VS71_MSVC_VERSIONS "${Vcvars_TOOLSET_71_MSVC_VERSIONS}") # VS 2003
check_var_equals(Vcvars_VS7_MSVC_VERSIONS "${Vcvars_TOOLSET_70_MSVC_VERSIONS}") # VS 2002
check_var_equals(Vcvars_VS6_MSVC_VERSIONS "${Vcvars_TOOLSET_60_MSVC_VERSIONS}") # VS 6.0

function(check_msvc_version_to_vs_version_convert msvc_versions_var expected_vs_version)
  foreach(msvc_version IN LISTS ${msvc_versions_var})
    Vcvars_ConvertMsvcVersionToVsVersion("${msvc_version}" output_var)
    if(NOT "${output_var}" STREQUAL "${expected_vs_version}")
      message(FATAL_ERROR "Vcvars_ConvertMsvcVersionToVsVersion failed for msvc_version [${msvc_version}]
       current_value [${output_var}]
      expected_value [${expected_vs_version}]
      ")
    endif()
  endforeach()
endfunction()

check_msvc_version_to_vs_version_convert(Vcvars_VS17_MSVC_VERSIONS "17")
check_msvc_version_to_vs_version_convert(Vcvars_VS16_MSVC_VERSIONS "16")
check_msvc_version_to_vs_version_convert(Vcvars_VS15_MSVC_VERSIONS "15")
check_msvc_version_to_vs_version_convert(Vcvars_VS14_MSVC_VERSIONS "14.0")
check_msvc_version_to_vs_version_convert(Vcvars_VS12_MSVC_VERSIONS "12.0")
check_msvc_version_to_vs_version_convert(Vcvars_VS11_MSVC_VERSIONS "11.0")
check_msvc_version_to_vs_version_convert(Vcvars_VS10_MSVC_VERSIONS "10.0")
check_msvc_version_to_vs_version_convert(Vcvars_VS9_MSVC_VERSIONS "9.0")
check_msvc_version_to_vs_version_convert(Vcvars_VS8_MSVC_VERSIONS "8.0")
check_msvc_version_to_vs_version_convert(Vcvars_VS71_MSVC_VERSIONS "7.1")
check_msvc_version_to_vs_version_convert(Vcvars_VS7_MSVC_VERSIONS "7.0")
check_msvc_version_to_vs_version_convert(Vcvars_VS6_MSVC_VERSIONS "6.0")

if(FUNCTIONS_ONLY_COMPONENT_REQUESTED)
  check_var_not_defined("Vcvars_FIND_VCVARSALL")
  check_var_not_defined("Vcvars_MSVC_ARCH")
  check_var_not_defined("Vcvars_MSVC_VERSION")
  check_var_not_defined("Vcvars_LAUNCHER")
  check_var_not_defined("Vcvars_LAUNCHER")
else()
  check_var_equals("Vcvars_FIND_VCVARSALL" "${EXPECTED_FIND_VCVARSALL}")
  check_var_equals("Vcvars_MSVC_ARCH" "${EXPECTED_MSVC_ARCH}")
  check_var_equals("Vcvars_MSVC_VERSION" "${EXPECTED_MSVC_VERSION}")

  check_file_exists("Vcvars_BATCH_FILE")
  check_filename_matches("Vcvars_BATCH_FILE" "${EXPECTED_BATCH_FILENAME}")

  check_file_exists("Vcvars_LAUNCHER")
  check_filename_matches("Vcvars_LAUNCHER" "${EXPECTED_LAUNCHER_FILENAME}")
endif()
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

  function(_add_find_vcvars_test component)
    if(component)
      set(test_name_suffix "${component}-${arg_NAME}")
    else()
      set(test_name_suffix "${arg_NAME}")
    endif()
    list(APPEND arg_OPTIONS
      -DFIND_VCVARS_COMPONENTS=${component}
      )

    # script-mode
    add_test(
      NAME "script-mode-${test_name_suffix}"
      COMMAND ${CMAKE_COMMAND} ${arg_OPTIONS} -P ${script_mode_script}
      )

    # project-mode
    set(src_dir "${project_mode_src_dir}")
    set(bld_dir "${project_mode_src_dir}-${test_name_suffix}-build")
    add_test(
      NAME "project-mode-${test_name_suffix}"
      COMMAND ${CMAKE_COMMAND} ${arg_OPTIONS} -S ${src_dir} -B ${bld_dir}
      )
    set_tests_properties("project-mode-${test_name_suffix}"  PROPERTIES FIXTURES_REQUIRED "fixture-${test_name_suffix}")
    add_test(
      NAME "project-mode-${test_name_suffix}-setup"
      COMMAND ${CMAKE_COMMAND} -E rm -rf ${bld_dir}
      )
    set_tests_properties("project-mode${suffix}-${test_name_suffix}-setup"  PROPERTIES FIXTURES_SETUP "fixture-${test_name_suffix}")
  endfunction()

  _add_find_vcvars_test("")
  _add_find_vcvars_test("FunctionsOnly")
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
    -DEXPECTED_FIND_VCVARSALL:BOOL=FALSE
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
    -DEXPECTED_FIND_VCVARSALL:BOOL=FALSE
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
    -DEXPECTED_FIND_VCVARSALL:BOOL=FALSE
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
    -DEXPECTED_FIND_VCVARSALL:BOOL=FALSE
  )

add_find_vcvars_test(
  NAME "find-vcvarsall-ON"
  OPTIONS
    # Requested
    -DREQUESTED_FIND_VCVARSALL:BOOL=ON
    # Expected
    -DEXPECTED_MSVC_ARCH:STRING=${EXPECTED_MSVC_ARCH}
    -DEXPECTED_MSVC_VERSION:STRING=${EXPECTED_MSVC_VERSION}
    -DEXPECTED_BATCH_FILENAME:STRING=vcvarsall.bat
    -DEXPECTED_WRAPPER_FILENAME:STRING=vcvarsall_wrapper.bat
    -DEXPECTED_FIND_VCVARSALL:BOOL=ON
  )

add_find_vcvars_test(
  NAME "find-vcvarsall-OFF"
  OPTIONS
    # Requested
    -DREQUESTED_FIND_VCVARSALL:BOOL=OFF
    # Expected
    -DEXPECTED_MSVC_ARCH:STRING=${EXPECTED_MSVC_ARCH}
    -DEXPECTED_MSVC_VERSION:STRING=${EXPECTED_MSVC_VERSION}
    -DEXPECTED_BATCH_FILENAME:STRING=${EXPECTED_BATCH_FILENAME}
    -DEXPECTED_WRAPPER_FILENAME:STRING=${EXPECTED_WRAPPER_FILENAME}
    -DEXPECTED_FIND_VCVARSALL:BOOL=OFF
  )

