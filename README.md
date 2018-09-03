FindVcvars
==========

Finds a "vcvars" batch script.

This [CMake](https://cmake.org) module can be used when configuring a project or when running
in cmake -P script mode.

## Why this repository ?

Waiting the module is integrated in upstream CMake (most likely CMake 3.13), this repository allows project 
to easily integrate the module by either copying its content or downloading it.

For reference, the associated merge request is https://gitlab.kitware.com/cmake/cmake/merge_requests/899

## Integrating the module in your project

There are few possible approaches:

### Approach 1: Download

* Add file `cmake/CMakeLists.txt` with the following code used to download the module:

```cmake
# Download FindVcvars.cmake
set(dest_file "${CMAKE_CURRENT_BINARY_DIR}/FindVcvars.cmake")
set(expected_hash "9072eac4ca2d7a06c6a69cefc315338d322954184a7410892e9afdb2486d9fb7")
set(url "https://raw.githubusercontent.com/scikit-build/cmake-FindVcvars/v1.0/FindVcvars.cmake")
if(NOT EXISTS ${dest_file})
  file(DOWNLOAD ${url} ${dest_file} EXPECTED_HASH SHA256=${expected_hash})
else()
  file(SHA256 ${dest_file} current_hash)
  if(NOT ${current_hash} STREQUAL ${expected_hash})
    file(DOWNLOAD ${url} ${dest_file} EXPECTED_HASH SHA256=${expected_hash})
  endif()
endif()
```

* Update top-level `CMakeLists.txt` with:

```cmake
add_subdirectory(cmake)
```

* Update `CMAKE_MODULE_PATH`:

```cmake
list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_CURRENT_BINARY_DIR}/cmake)
```


### Approach 2: Copy

* Copy `FindVcvars.cmake` into your source tree making sure you reference the tag (or SHA) in the associated
  commit message.

* Update `CMAKE_MODULE_PATH`:

```cmake
list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
```

### Approach 3: Git submodule

Add this repository as a git submodule.

