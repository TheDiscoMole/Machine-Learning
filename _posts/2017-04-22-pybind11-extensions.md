---
layout: post
title: "PyBind11 CUDA Extensions"
date: 2017-04-22
excerpt: "A better way to compile header-only CUDA extensions"
tag:
comments: false
---
During my [general intelligence]() projects I figured out there is an easier, cleaner and more efficient way to write CUDA Python extensions with pybind11 than described in the PyTorch [documentation](https://pytorch.org/tutorials/advanced/cpp_extension.html). Assuming the extension can be expressed as a header-only lib.

with an extension.cu:

```cpp
// extension includes
#include <cuda.h>
#include <curand.h>
...

// pybind includes
#include <pybind11/pybind11.h>
#include <pybind11/numpy.h> // used pybind11 functionality needs to be included manually
...

// project merging includes
#include "a.h"
#include "b.h"
...

// pybind module creation
namespace py = pybind11;

PYBIND11_MODULE(bindings, m) {}
```

and a CMakeLists.txt:

```make
cmake_minimum_required(VERSION 3.12 FATAL_ERROR)
project(extension_name VERSION 0.0.1 LANGUAGES CXX CUDA)

# set compiler flags
set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -std=c++11 -arch=sm_61 --use_fast_math ...")

# get python <-> c++ interface headers
find_package(PythonLibs REQUIRED)
include_directories(${PYTHON_INCLUDE_DIRS})

# set source dir list
set(sources PROJECT lib/pybind11/include)

# genreate python module
add_library (bindings MODULE extension.cu)

# link header directories and python resources
target_include_directories(bindings PRIVATE ${sources})
target_link_libraries(bindings ${PYTHON_LIBRARIES})
```
