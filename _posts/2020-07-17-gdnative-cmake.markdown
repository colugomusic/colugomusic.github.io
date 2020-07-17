---
layout: post
title:  "Building GDNative C++ plugins with CMake"
---
The [official Godot docs](https://godot-es-docs.readthedocs.io/en/latest/tutorials/plugins/gdnative/gdnative-cpp-example.html) provide an example `SConstruct` file for building GDNative plugins. As I understand it the motivations behind using SCons as a build tool over CMake are as follows:
- The SCons documentation is all hot piss.
- It's also really slow because it has to re-run the configuration step every time you do a build.
- SCons allows you to spend more time doing what you love: trying to figure out how to generate a visual studio project so you can debug.

Despite these benefits some people still prefer using CMake so here is a simple example of how to set up your project.

In this example the project is set up something like this:
```
📦example_project  
 ┣ 📂front
 ┃ ┣ 📜project.godot
 ┃ ┗ 📂bin
 ┃   ┣ 📜example_project.gdnlib
 ┃   ┣ 📜various.gdns
 ┃   ┣ 📜native.gdns
 ┃   ┗ 📜scripts.gdns
 ┗ 📂back
   ┣ 📜CMakeLists.txt
   ┣ 📂submodules
   ┃ ┗ 📂GodotNativeTools
   ┃   ┗ 📂godot-cpp (cloned from github)
   ┗ 📂src
     ┣ 📜various.cpp
     ┣ 📜source.cpp
     ┗ 📜files.cpp
```
`./back/submodules/GodotNativeTools/godot-cpp/` is cloned from the Github repository and configured as described in the offical docs.

In my project I separate the actual Godot project from the GDNative plugins into `front` and `back` directories respectively, but there's no need to organize things like this if you don't want to.

Here's the `CMakeLists.txt`. This is a stripped down version of the CMakeLists I'm using in my project. If you can't read [generator expressions](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html) then probably your brain is using that space for something less stupid so you should consider yourself blessed.
```cmake
cmake_minimum_required(VERSION 3.12)
project(example_project)

set(dir_submodules ${CMAKE_CURRENT_LIST_DIR}/submodules)
set(dir_godot_cpp ${dir_submodules}/GodotNativeTools/godot-cpp)
set(dir_front ${CMAKE_CURRENT_LIST_DIR}/../front)

list(APPEND HEADERS
    ./src/various.h
    ./src/header.h
    ./src/files.h
)

list(APPEND SRC
    ./src/various.cpp
    ./src/source.cpp
    ./src/files.cpp
)

list(APPEND BIN_SRC
    ${dir_front}/bin/example_project.gdnlib
    ${dir_front}/bin/various.gdns
    ${dir_front}/bin/native.gdns
    ${dir_front}/bin/scripts.gdns
)

source_group(TREE ${CMAKE_CURRENT_LIST_DIR}/src PREFIX headers FILES ${HEADERS})
source_group(TREE ${CMAKE_CURRENT_LIST_DIR}/src PREFIX src FILES ${SRC})
source_group(TREE ${dir_front}/bin PREFIX bin FILES ${BIN_SRC})

set(on_windows $<STREQUAL:${CMAKE_SYSTEM_NAME},Windows>)
set(on_osx $<STREQUAL:${CMAKE_SYSTEM_NAME},Darwin>)
set(on_linux $<STREQUAL:${CMAKE_SYSTEM_NAME},Linux>)
set(debug_build $<OR:$<CONFIG:Debug>,$<STREQUAL:${CMAKE_BUILD_TYPE},Debug}>>)

set(lib_godot_cpp_platform $<IF:${on_windows},windows,$<IF:${on_osx},osx,linux>>)
set(lib_godot_cpp_config $<IF:${debug_build},debug,release>)

set(out_dir ${dir_front}/bin/$<IF:${on_windows},win64,$<IF:${on_osx},osx,x11>>)

set(godot_cpp_library ${dir_godot_cpp}/bin/libgodot-cpp.${lib_godot_cpp_platform}.${lib_godot_cpp_config}.64${CMAKE_STATIC_LIBRARY_SUFFIX})

add_library(${PROJECT_NAME} SHARED ${SRC} ${HEADERS} ${BIN_SRC})

target_include_directories(${PROJECT_NAME} PRIVATE
    ${dir_godot_cpp}/godot_headers
    ${dir_godot_cpp}/include
    ${dir_godot_cpp}/include/core
    ${dir_godot_cpp}/include/gen
    ${CMAKE_CURRENT_LIST_DIR}/src
)

target_link_libraries(${PROJECT_NAME} PRIVATE
    ${godot_cpp_library}
)

set_target_properties(${PROJECT_NAME} PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY_DEBUG ${out_dir}
    RUNTIME_OUTPUT_DIRECTORY_RELEASE ${out_dir}
    RUNTIME_OUTPUT_DIRECTORY ${out_dir}
    CXX_STANDARD 17
)
```
This will output libraries to `./front/bin/`.