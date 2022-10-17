+++
title = "CMake: Global Build Configuration"
date = 2022-10-16

[taxonomies]
tags = ["c++", "cmake"]
+++

Hotarubi uses CMake for cross-platform targets including Windows and Linux. While CMake isn't my first choice, it is the most obvious one for C++ projects for many reasons outside the scope of this post.

What sucks about CMake is the documentation is very lacking in examples and the examples out in the wild are usually not "modern" CMake but legacy stuff that you really should avoid if you don't want to drive yourself crazy.

So today we're going to look at how Hotarubi uses CMake and hopefully this will be useful for anyone trying to start a new CMake project.

Hotarubi is a very modular project. It's divided up into a number of components:

- `src/common`: A library that contains code common to all projects. This includes things like the logger, various bit manipulation, string, and file helpers among other things.

- `src/core`: The core emulation logic.

- `src/frontend`: Frontend code that is common to all user interfaces including the terminal.

- `src/gui`: Contains code for graphical user interfaces and the entry point.

- `src/version`: A special project that interfaces with Git to automatically generate headers that will be used for presenting version information to the user.

- `src/video`: Code for rendering.

- `depends`: Git submodules and build files for all of the projects external dependencies.

Each of these directories and sub-directories maintains it's own `CMakeLists.txt` so we need to first include all of them in a top-level `CMakeLists.txt`:

```cmake
# ./CMakeLists.txt
# set this to whatever suits your project
cmake_minimum_required(VERSION 3.16)
# set the project name
project(Hotaru C CXX)
# bring in the dependencies
add_subdirectory(depends)
# bring in the source tree
add_subdirectory(src)
```
Next we include our dependencies:

```cmake
# ./depends/CMakeLists.txt
# https://github.com/fmtlib/fmt
add_subdirectory(fmt)
# ...
```
And finally our individual modules:
```cmake
# ./src/CMakeLists.txt
# static libraries
add_subdirectory(common)
add_subdirectory(core)
add_subdirectory(frontend)
add_subdirectory(version)
add_subdirectory(video)
# executable
add_subdirectory(gui)
```
As you can imagine, having all these individual modules and external libraries makes it difficult to manage compiler settings and especially so when they share settings in common. A very WET solution would be simply to repeat these settings in every module:
```cmake
# ./src/frontend/CMakeLists.txt
add_library(frontend
  # ...
)

target_compile_options(frontend PRIVATE
  # MSVC compiler options
  $<$<CXX_COMPILER_ID:MSVC>:/W4 /WX /utf-8>
  # Clang and GCC
  $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall -Wextra -Werror>
)

target_include_directories(frontend PRIVATE
  "${PROJECT_SOURCE_DIR}/src"
)

target_link_libraries(frontend PRIVATE
  fmt::fmt
  # ...
)
```
But now if I want to enable any additional settings I have to edit 6 `CMakeLists.txt` and that sucks. Instead it would be nicer to define some global settings and then add any module-specific settings on a per-project basis. For exmaple, perhaps I'd like to link `fmt` with every project while also having private dependencies for managing the dependency chain of the various `src/` projects. 

You might have noticed that `PRIVATE` is peppered throughout those settings. We obviosly don't want these settings to be private. The next obvious choice is `PUBLIC` and if we set `target_include_directories` to `PUBLIC` it might have the effect of making that directory visible to all projects but we still need to associate it with a project that includes source files (`frontend` in this case) so it's not a very elegant solution.

Instead we can use a slightly lesser known setting `INTERFACE`:
> Creates an Interface Library. An INTERFACE library target does not compile sources and does not produce a library artifact on disk. However, it may have properties set on it and it may be installed and exported.
> -[source](https://cmake.org/cmake/help/latest/command/add_library.html#interface-libraries)

Creating an interface library is useful to us because unlike normal libraries we don't need to associate any files with it:
```cmake
# creates an interface library called "ProjectConfiguration"
add_library(ProjectConfiguration INTERFACE)
```
We could just include this in our top-level `CMakeLists.txt` but I like to keep it in a separate file in `.cmake/build_configuration.cmake`. This isn't required, it's just how I choose to organize things (actually I believe convention is to call the directory `CMake` but I like to hide these directories from an `ls` command because they're mostly noise to me).

If you do decide to put it in a separate place then just modify the top-level `CMakeLists.txt` to include this file before our modules:
```cmake
# ./CMakeLists.txt
# set this to whatever suits your project
cmake_minimum_required(VERSION 3.16)
# set the project name
project(Hotaru C CXX)
# bring in our global build configuration
include(.cmake/project_configuration.cmake)
# bring in the dependencies
add_subdirectory(depends)
# bring in the source tree
add_subdirectory(src)
```
Then we can just create `.cmake/build_configuration.cmake` and populate it with our interface library just like any normal library (minus the source files, of course):
```cmake
# ./.cmake/build_configuration.cmake
# https://cmake.org/cmake/help/latest/variable/CMAKE_CXX_STANDARD_REQUIRED.html
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# create an interface that will be used for the entire project
add_library(ProjectConfiguration INTERFACE)

# global compiler settings
target_compile_options(ProjectConfiguration INTERFACE
  # MSVC compiler options
  $<$<CXX_COMPILER_ID:MSVC>:/W4 /WX /utf-8>
  # Clang and GCC
  $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall -Wextra -Werror>
)

# global include directories
target_include_directories(ProjectConfiguration INTERFACE 
  "${PROJECT_SOURCE_DIR}/src"
)

# global features
target_compile_features(ProjectConfiguration INTERFACE
  cxx_std_20
)

# global linked libraries
target_link_libraries(ProjectConfiguration INTERFACE
  fmt::fmt
)
```
Finally we can just link our build configuration library with all our projects and put it in the back of our minds where it belongs:
```cmake
# create the frontend library
add_library(frontend
  # ...
)

# link our dependencies
target_link_libraries(frontend PRIVATE
  ProjectConfiguration
  common
  core
  version
)
```