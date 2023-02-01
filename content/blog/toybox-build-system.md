+++
title = "The Toybox Build System"
date = 2023-01-31
+++

I've been working on a little game engine I've been calling [toybox](https://github.com/HoneybunchBuilder/toybox). It's been a fun exercise in building a small piece tech for crafting the kinds of games I want to build, the way I want to build them. It's not a perfect project by any stretch of the imagination but the CMake based build system that I have driving my work with it has reached a point where I'd like to show it off a bit.

I'll be honest: I'm a CMake apologist. I don't want to pretend that it's perfect or even that it's best at doing what I'm trying to do with it. I also think that CMake is extremely powerful and I'm pretty happy with the way that `toybox` builds. As you read through this if you find yourself saying "I could do this way better in X environment" (`zig`, `rust`, `meson`, etc.) please feel free to write up your own version of what I have here.

But let'st start off with what actually needs accomplishing and then we can get to implementation details. For any game project I want to:
 * Compile whatever core language (in this case C)
 * Compile shaders
 * Compile/cook/process assets into a game-ready format
 * Reliably depend on 3rd party libraries
 * Compile for as many platforms as possible
 * Compile with as many compilers as possible
 * Be editor agnostic
 * Not close any doors on development for game consoles
 * Compile on at least Windows, Mac and Linux host OSes
 * Be able to setup CI
 * Be able to deploy to Steam from CI
 * Be as easy to get building on a fresh machine as possble

That's a lot to ask so let's also set some clear boundaries. Here's what I don't care about:
 * iOS / Android are nice to haves but not necessary
 * CI doesn't have to be perfectly generic (I manage my own self-hosted runners)
 * `toybox` does not have to be able to be hosted on a package manager
 * Projects built on `toybox` don't have to be as effortless to setup as an off-the-shelf engine

Easy, right? Well it actually hasn't been so bad. I've managed to get `toybox` to a place where it really does achieve all the goals I've outlined here. All in less than 500 lines of CMake as of the time of writing. There's some other caveats too which I'll detail as I highlight interesting parts of the implementation.

The best way to show off that implementation is to start from a sample and work backwards. There are a couple samples inside of the `toybox` repo that I use for debugging. The big one is `tb_bistro`. Here's how its CMakeLists.txt file looks:

```CMake
file(GLOB_RECURSE bistro_source "source/*.c")

tb_add_app(tb_bistro "${bistro_source}")
target_link_libraries(tb_bistro PRIVATE tb_samplecore)
```

My samples do share a base `tb_samplecore` library for convenience but I promise there isn't any trickery there. It's just:
```CMake
file(GLOB_RECURSE samplecore_source "source/*.c")

add_library(tb_samplecore STATIC ${samplecore_source})

target_include_directories(tb_samplecore PUBLIC "include")
target_link_libraries(tb_samplecore PUBLIC toybox)
```

The magic is all coming from that call to `tb_add_app`. It's just a function in the top level CMakeLists.txt of the repo. That may be considered cheating but I have a standalone project in a different repo as well. I call it [the high seas](https://github.com/HoneybunchBuilder/thehighseas) and its CMakeLists.txt is a bit more complex.

```CMake
cmake_minimum_required(VERSION 3.20)

option(TB_BUILD_SAMPLES "" OFF)

if(NOT TB_SOURCE_PATH)
  message(FATAL_ERROR "Please specify TB_SOURCE_PATH as a source path to the toybox engine")
endif()

message("Including: ${TB_SOURCE_PATH}/CMakeLists.txt")
include(${TB_SOURCE_PATH}/CMakeLists.txt)

# Determine version from the vcpkg json manifest
file(READ vcpkg.json VCPKG_MANIFEST)
string(JSON VCPKG_VERSION GET ${VCPKG_MANIFEST} "version")

project(thehighseas
        VERSION ${VCPKG_VERSION}
        DESCRIPTION "A simple game built on the toybox engine"
        LANGUAGES C CXX)

set(GAME_NAME "thehighseas")
set(GAME_VERSION_MAJOR ${CMAKE_PROJECT_VERSION_MAJOR})
set(GAME_VERSION_MINOR ${CMAKE_PROJECT_VERSION_MINOR})
set(GAME_VERSION_PATCH ${CMAKE_PROJECT_VERSION_PATCH})

option(FINAL "Compile with the intention to redistribute" OFF)
option(PROFILE_TRACY "Compile with support for the tracy profiler" ON)
option(COOK_ASSETS "Process assets for runtime loading" ON)

file(GLOB_RECURSE source "source/*.c" )

tb_add_app(thehighseas "${source}")
```
This is kind of cheating. You'll notice that this project requires on having a path to the `toybox` engine repo. It's not elegant but it allows a game project to have much more control over the exact options it builds the engine with. I did have a version of `toybox` that could be built as a vcpkg port but there were a couple problems I couldn't solve. If I wanted to build with `tracy` instrumentation at the engine level, how would that work? It doesn't make sense for it to be a "feature" of the vcpkg port. And what if I wanted to skip cooking engine assets because I just want CI to build code and not bother compressing `glb` files? If you have answers to these questions please e-mail me but after slamming my head on these questions for a bit I decided to settle for imperfection and keep moving forward with this slightly hacky solution.

Of course setting up `TB_SOURCE_PATH` by hand on the CLI or through editor configuration would be a huge pain. My solution has been to rely heavily on `CMakePresets.json`. I didn't know about this until I was stumbling through the CMake documentation (as I find myself often doing). Check out [the docs](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html) as there's even more in there that I haven't found myself utilizing yet. Here's an example of my most-used configuration preset from `thehighseas`:
```json
{
      "name": "x64-windows-ninja-llvm",
      "displayName": "x64 Windows Ninja LLVM",
      "generator": "Ninja Multi-Config",
      "binaryDir": "${sourceDir}/build/x64/windows",
      "toolchainFile": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
      "cacheVariables": {
        "VCPKG_TARGET_TRIPLET": "x64-windows",
        "CMAKE_C_COMPILER": "clang",
        "CMAKE_CXX_COMPILER": "clang++",
        "CMAKE_RC_COMPILER": "llvm-rc",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
        "TB_SOURCE_PATH": "../toybox"
      },
      "condition": {
        "type": "equals",
        "lhs": "${hostSystemName}",
        "rhs": "Windows"
      }
    },
```
What's excellent about this is that this file is in source control. So all that's needed is to have `toybox` and `thehighseas` repos next to each other and from `thehighseas`'s repo you can simply run `cmake --preset x64-windows-ninja-llvm` and you're off to the races. I also have build presets so you can build with `cmake --build --preset debug-x64-windows-ninja-llvm`.  The only real requirement here is that the `VCPKG_ROOT` environment variable needs to be set.

Some downsides to this approach include:
 * CMakePresets.json is large and requires occasional maintainance
 * Presets cannot easily be shared between `toybox` and dependant projects like `thehighseas` so they must be duplicated.
 * If `VCPKG_ROOT` isn't defined the error about an invalid toolchain path isn't particularly helpful to an end user

Speaking of `vcpkg` now's a good time to bring up how I'm using it. There are quite a few package management solutions for C/C++ but `vcpkg` is my favorite. I've found it to largely be "how I would build a cmake based package manager" and as such I'm a pretty active maintainer on the project. A number of dependencies such as `ktx` and `mimalloc` have required some extra effort to get working but I've found it pretty effortless to contribute changes I've needed upstream. The documentation is good and the entire project is really well thought out. As such I've relied upon it heavily for managing all build dependencies. Of which I have several! It's also been straightforward to occasionally point my dependencies at my own custom forks as I had to test things before upstreaming them.

I've been so happy working with `vcpkg` that, as noted, I use the vcpkg toolchain as my primary build toolchain and most of my build centers around vcpkg triplets. This might be limiting if I wanted to bring my work to a game console but with overlay-triplets I could add on my own custom triplets just for `toybox`. In fact I did a proof of concept of this about a year back getting all of my core work building with the switch homebrew SDK. I got as far as getting everything but rendering working! Getting SDL2 and other libraries working was possible because of vcpkg's overlay-ports concept. If I took the time to build a vulkan driver for the switch homebrew environment I could have gotten it working on my hacked switch but that was a bit out-of-scope. It makes me confident in saying that this build system currently at least *appears* to be capable of being fitted to compile to consoles.

The only dependency I currently *don't* have managed by `vcpkg` is the DirectXShaderCompiler or `dxc`. That I currently require to be found on the environment's PATH. There is a port of `dxc` on vcpkg but it downloads a binary and the only pre-packaged solutions that it can provide are for Windows and Ubuntu; macOS still needs the Vulkan SDK so I may as just require it for everything. Plus the Vulkan SDK is currently the best way to get ahold of the vulkan Validation Layers. I did look into getting a port of that onto vcpkg but considering I'd also have to get `dxc` building through a port to completely ditch the Vulkan SDK I've decided against it (for now).

Cooking assets was another area that I spent a lot of time getting streamlined. My engine primarily consumes `gltf` files exported from blender. So my project simply pulls in the `gltfpack` tool from vcpkg and links `meshoptimizer` to be able to read the compressed `glb`s. All that "cooking" entails in CMake is creating a custom target that invokes the tool on the assets:
```CMake
  # snippet from CMake function cook_assets
  file(GLOB_RECURSE source_scenes "${CMAKE_CURRENT_LIST_DIR}/assets/scenes/*.glb")
  foreach(scene ${source_scenes})
    file(RELATIVE_PATH relpath ${CMAKE_CURRENT_LIST_DIR}/assets ${scene})
    get_filename_component(relpath ${relpath} DIRECTORY)
    get_filename_component(filename ${scene} NAME_WE)
    # $<CONFIG> is a generator expression which will evaluate to Debug/Release/etc when the actual build-tool evaluates this path
    set(packed_scene ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>/assets/scenes/${filename}.glb)

    # ${CMAKE_COMMAND} is a built-in variable that points to the cmake executable itself. The -E flag runs cmake in a mode where it can run helper routines. Here we use -E make_directory to ensure that the relative path to the cooked scene is created before gltfpack tries to output to it
    # Parameters here like OUTPUT and MAIN_DEPENDENCY are used to tell CMake what this command operates on so if the ${scene} file changes or ${packed_scene} is missing CMake will notice and know that it needs to run this command
    add_custom_command(
      OUTPUT ${packed_scene}
      COMMAND ${CMAKE_COMMAND} -E make_directory assets/${relpath}
      COMMAND ${GLTFPACK} -cc -kn -km -ke -tc -i ${scene} -o ${packed_scene}
      MAIN_DEPENDENCY ${scene}
      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>
    )

    list(APPEND packed_scenes ${packed_scene})
  endforeach()
  list(APPEND assets ${packed_scenes})

  # ${target_name} being a parameter for this function eg. "tb_bistro"
  # Effectively this adds a target tb_bistro_scenes that when built will produce ${packed_scenes} which the above custom commands have marked as their outputs
  add_custom_target(${target_name}_scenes DEPENDS ${packed_scenes})
```
Compiling shaders is a similar story:
```CMake
set(shader_include_dir "${CMAKE_CURRENT_LIST_DIR}/include")
  file(GLOB shader_includes "${shader_include_dir}/*.hlsli")

  file(GLOB shader_files "${CMAKE_CURRENT_LIST_DIR}/source/*.hlsl")
  foreach(shader ${shader_files})
    get_filename_component(filename ${shader} NAME_WLE)
    set(shader_out_path ${CMAKE_CFG_INTDIR}/shaders)

    set(vert_out_path "${shader_out_path}/${filename}_vert.h")
    set(frag_out_path "${shader_out_path}/${filename}_frag.h")
    # Putting a semicolon here means that out_paths is a list with two elements
    set(out_paths "${vert_out_path};${frag_out_path}")

    # Here we have more complex generator expressions
    # $<$<CONFIG:Debug>:-O0> is a way to say "if Debug config pass -O0 flag" so we can have unoptimized shaders
    # Overly verbose and hard to read but it gets the job done
    add_custom_command(
        OUTPUT ${out_paths}
        COMMAND ${CMAKE_COMMAND} -E make_directory ${shader_out_path}
        COMMAND ${DXC} -T vs_6_5 -E vert -Vn ${filename}_vert $<$<CONFIG:Debug>:-O0> $<$<CONFIG:Debug>:-Zi> -I ${shader_include_dir} $<$<CONFIG:Debug>:-Qembed_debug> -fspv-target-env=vulkan1.1 -spirv ${shader} -Fh ${vert_out_path}
        COMMAND ${DXC} -T ps_6_5 -E frag -Vn ${filename}_frag $<$<CONFIG:Debug>:-O0> $<$<CONFIG:Debug>:-Zi> -I ${shader_include_dir} $<$<CONFIG:Debug>:-Qembed_debug> -fspv-target-env=vulkan1.1 -spirv ${shader} -Fh ${frag_out_path}
        MAIN_DEPENDENCY ${shader}
        DEPENDS ${shader_includes}
    )
    list(APPEND shader_headers ${out_paths})
  endforeach()
  list(APPEND shader_sources ${shader_files})
```

Which brings us back to `tb_add_app` which is mostly:
```CMake
  # Looks for all shaders in the source directory and outputs a list of all input hlsl files to be compiled to ${target_name}_shader_sources
  # An a list of all output headers that will be produced to ${target_name}_shader_headers
  cook_shaders(${target_name}_shader_sources ${target_name}_shader_headers)
  if(out_shader_headers)
    # A project may have no custom shaders, in which case don't even bother
    add_custom_target(${target_name}_shaders ALL DEPENDS ${target_name}_shader_headers)
    # target_sources ensures that if you generate a project for an IDE that the shaders will show up as part of the project
    target_sources(${target_name}_shaders PRIVATE ${target_name}_shader_sources)
    add_dependencies(${target_name} ${target_name}_shaders)
  endif()

  if(COOK_ASSETS)
    # ${target_name}_assets_path is another output variable; unused by projects but used to track the path to engine assets
    cook_assets(${target_name} ${target_name}_assets_path)
  endif()
```
Some additional, uninteresting CMake cruft has been elided for your reading pleasure but if you check out [the full CMakeLists.txt](https://github.com/HoneybunchBuilder/toybox/blob/main/CMakeLists.txt) I think you'll find that it's not entirely awful.

Ok, yes, I admit that the CMake scripting language is... not great. I don't want to call it *bad* because I think that's uncharitable. CMake is trying to accomplish a lot here and, at least for my needs so far, it's succeeding. So I want to be nice. But this is a language that's hard to grok. I've shown colleagues what you can do with modern CMake and they all come away with the same "I didn't know you could do that". Custom targets are extremely powerful and I've done whacky stuff like building custom targets that use maya as a "compiler" to automatically export fbx files with desired settings. I love what I can do with this but the language here is absolutely god awful and I wish it was better but it's not.

I still remain a CMake apologist though. It's imperfect but I currently have this project building across a variety of host operating systems targeting 5 major OSes (Android and iOS build but don't pacakge or run). I have my self-hosted github actions runners deploying to Steam for Windows, Mac and Linux every time I commit to `thehighseas` repo. Shaders compile, meshes and textures compress and it's *fast*.

Configuration takes a hot minute (maybe 10 minutes) to build all dependencies from scratch but after that I find myself iterating extremely quickly. HLSL shaders compile like any other source file. I can export a glb from blender and have it loaded in the runtime in no time flat. My CI job for Windows builds all 6 of my configurations [in less than 5 minutes](https://github.com/HoneybunchBuilder/toybox/actions/runs/4059779139) and I get coverage over:
 * windows-ninja-llvm
 * windows-static-ninja-llvm
 * mingw-ninja
 * mingw-static-ninja
 * windows-vs2022-llvm
 * windows-static-vs2022-llvm

That's right I have this building with LLVM and GCC; with ninja and Visual Studio. Super useful because sometimes I do like to use the VS debugger for things like memory breakpoints. That and I just like to build with as many setups as possible; multi-compiler coverage has caught bugs for me in the past.

Maybe this is all old hat or boring but I've been spending most of my career in Unity and Unreal Engine where builds and CI are painful, complicated and expensive to setup. Maybe this is something that I could have done way easier in some other environment but I've tried a lot of them and I like to think I take the time to RTFM but CMake managed to get me all the way to a system where I can go from commit to Steam build in minutes and I'm pretty happy with it.

My only real goal with this post is to get you interested in digging in and attempting to grok my full CMakeLists.txt setup. If this post even gets one person to go and check out the CMake documentation for custom targets I'd consider it a success. Or maybe someone will get mad at me for not knowing the obvious way I could do this with `x` tool or `y` setup and they'll wind up pointing me or someone else at a better solution. Heck, if you got this far try going through the [README](https://github.com/HoneybunchBuilder/toybox/blob/main/readme.md) for toybox and attempt to build it from source. If you have any ideas on how I could improve my CMake or vcpkg usage please feel free to open an issue.

There's still a lot of work to do on `toybox` and I have no idea how many years it'll take me to get it where I want it but at least for now I don't find this build system getting in my way. At the very least I hope it's a good exercise in not letting "perfect" be the enemy of "good enough". And I guess "good enough" for me is getting my silly little toy project on Steam :)