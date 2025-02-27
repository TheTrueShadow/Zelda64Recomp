cmake_minimum_required(VERSION 3.20)
project(Zelda64Recompiled)
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
# set(CMAKE_CXX_VISIBILITY_PRESET hidden)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-everything -Wall -Wextra")

# Avoid warning about DOWNLOAD_EXTRACT_TIMESTAMP in CMake 3.24:
if (CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
    cmake_policy(SET CMP0135 NEW)
endif()

if(UNIX AND NOT APPLE)
    set(LINUX TRUE)
endif()

set (CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

if (WIN32)
    set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/lib/")
endif()

set(RT64_STATIC TRUE)
add_subdirectory(${CMAKE_SOURCE_DIR}/lib/rt64 ${CMAKE_BINARY_DIR}/rt64)

# set(BUILD_SHARED_LIBS_SAVED "${BUILD_SHARED_LIBS}")
set(BUILD_SHARED_LIBS OFF)
SET(LUNASVG_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
add_subdirectory(${CMAKE_SOURCE_DIR}/lib/lunasvg)
# set(BUILD_SHARED_LIBS "${BUILD_SHARED_LIBS_SAVED}")
SET(ENABLE_SVG_PLUGIN ON CACHE BOOL "" FORCE)
add_subdirectory(${CMAKE_SOURCE_DIR}/lib/RmlUi)

target_include_directories(rt64 PRIVATE ${CMAKE_BINARY_DIR}/rt64/src)

# RecompiledFuncs - Library containing the primary recompiler output
add_library(RecompiledFuncs STATIC)

target_compile_options(RecompiledFuncs PRIVATE
    # -Wno-unused-but-set-variable
    -fno-strict-aliasing
)

target_include_directories(RecompiledFuncs PRIVATE
    ${CMAKE_SOURCE_DIR}/include
)

file(GLOB FUNC_C_SOURCES ${CMAKE_SOURCE_DIR}/RecompiledFuncs/*.c)
file(GLOB FUNC_CXX_SOURCES ${CMAKE_SOURCE_DIR}/RecompiledFuncs/*.cpp)

target_sources(RecompiledFuncs PRIVATE ${FUNC_C_SOURCES} ${FUNC_CXX_SOURCES})

# PatchesLib - Library containing the recompiled output for any custom function patches
add_library(PatchesLib STATIC)

target_compile_options(PatchesLib PRIVATE
    # -Wno-unused-but-set-variable
    -fno-strict-aliasing
)

target_include_directories(PatchesLib PRIVATE
    ${CMAKE_SOURCE_DIR}/include
)

target_sources(PatchesLib PRIVATE
    ${CMAKE_SOURCE_DIR}/RecompiledPatches/patches.c
    ${CMAKE_SOURCE_DIR}/RecompiledPatches/patches_bin.c
)

set_source_files_properties(${CMAKE_SOURCE_DIR}/RecompiledPatches/patches.c PROPERTIES COMPILE_FLAGS -fno-strict-aliasing)

# Build patches elf
add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/patches/patches.bin
    COMMAND make
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/patches
    BYPRODUCTS ${CMAKE_SOURCE_DIR}/patches/patches.bin}
)

# Generate patches_bin.c from patches.bin
add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/RecompiledPatches/patches_bin.c
    COMMAND file_to_c ${CMAKE_SOURCE_DIR}/patches/patches.bin mm_patches_bin ${CMAKE_SOURCE_DIR}/RecompiledPatches/patches_bin.c ${CMAKE_SOURCE_DIR}/RecompiledPatches/patches_bin.h 
    DEPENDS ${CMAKE_SOURCE_DIR}/patches/patches.bin
)

# Generate mm_shader_cache.c from the MM shader cache
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/mm_shader_cache.c ${CMAKE_CURRENT_BINARY_DIR}/mm_shader_cache.h
    COMMAND file_to_c ${CMAKE_SOURCE_DIR}/shadercache/mm_shader_cache.bin mm_shader_cache_bytes ${CMAKE_CURRENT_BINARY_DIR}/mm_shader_cache.c ${CMAKE_CURRENT_BINARY_DIR}/mm_shader_cache.h
    DEPENDS ${CMAKE_SOURCE_DIR}/shadercache/mm_shader_cache.bin
)

# Recompile patches elf into patches.c
add_custom_command(OUTPUT
                       ${CMAKE_SOURCE_DIR}/RecompiledPatches/patches.c
                       ${CMAKE_SOURCE_DIR}/RecompiledPatches/recomp_overlays.inl
                       ${CMAKE_SOURCE_DIR}/RecompiledPatches/funcs.h
                       ${CMAKE_SOURCE_DIR}/src/recomp/patch_loading.cpp
                   COMMAND ./N64Recomp patches.toml && ${CMAKE_COMMAND} -E touch ${CMAKE_SOURCE_DIR}/src/recomp/patch_loading.cpp
                   WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
                   DEPENDS ${CMAKE_SOURCE_DIR}/patches/patches.bin
)

# Main executable
add_executable(Zelda64Recompiled)

set (SOURCES
    ${CMAKE_SOURCE_DIR}/ultramodern/audio.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/events.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/mesgqueue.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/misc_ultra.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/port_main.c
    ${CMAKE_SOURCE_DIR}/ultramodern/scheduling.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/threadqueue.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/task_win32.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/threads.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/timer.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/ultrainit.cpp
    ${CMAKE_SOURCE_DIR}/ultramodern/rt64_layer.cpp

    ${CMAKE_SOURCE_DIR}/src/recomp/ai.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/cont.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/dp.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/eep.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/euc-jp.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/flash.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/math_routines.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/overlays.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/patch_loading.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/pak.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/pi.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/ultra_stubs.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/ultra_translation.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/print.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/recomp.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/sp.cpp
    ${CMAKE_SOURCE_DIR}/src/recomp/vi.cpp

    ${CMAKE_SOURCE_DIR}/src/main/main.cpp
    
    ${CMAKE_SOURCE_DIR}/src/game/input.cpp
    ${CMAKE_SOURCE_DIR}/src/game/controls.cpp
    ${CMAKE_SOURCE_DIR}/src/game/config.cpp
    ${CMAKE_SOURCE_DIR}/src/game/scene_table.cpp
    ${CMAKE_SOURCE_DIR}/src/game/debug.cpp
    ${CMAKE_SOURCE_DIR}/src/game/quicksaving.cpp
    ${CMAKE_SOURCE_DIR}/src/game/recomp_api.cpp

    ${CMAKE_SOURCE_DIR}/src/ui/ui_renderer.cpp
    ${CMAKE_SOURCE_DIR}/src/ui/ui_launcher.cpp
    ${CMAKE_SOURCE_DIR}/src/ui/ui_config.cpp
    ${CMAKE_SOURCE_DIR}/src/ui/ui_color_hack.cpp
    ${CMAKE_SOURCE_DIR}/src/ui/ui_rml_hacks.cpp

    ${CMAKE_SOURCE_DIR}/rsp/aspMain.cpp
    ${CMAKE_SOURCE_DIR}/rsp/njpgdspMain.cpp

    ${CMAKE_SOURCE_DIR}/lib/RmlUi/Backends/RmlUi_Platform_SDL.cpp

    ${CMAKE_CURRENT_BINARY_DIR}/mm_shader_cache.c
)

target_include_directories(Zelda64Recompiled PRIVATE
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/lib/concurrentqueue
    ${CMAKE_SOURCE_DIR}/lib/GamepadMotionHelpers
    ${CMAKE_SOURCE_DIR}/lib/RmlUi/Include
    ${CMAKE_SOURCE_DIR}/lib/RmlUi/Backends
    ${CMAKE_SOURCE_DIR}/lib/rt64/src/contrib
    ${CMAKE_SOURCE_DIR}/lib/rt64/src/contrib/hlslpp/include
    ${CMAKE_SOURCE_DIR}/lib/rt64/src/contrib/dxc/inc
    ${CMAKE_SOURCE_DIR}/lib/rt64/src
    ${CMAKE_SOURCE_DIR}/lib/rt64/src/rhi
    ${CMAKE_SOURCE_DIR}/lib/rt64/src/render
    ${CMAKE_SOURCE_DIR}/lib/freetype-windows-binaries/include
    ${CMAKE_SOURCE_DIR}/lib/rt64/src/contrib/nativefiledialog-extended/src/include
    ${CMAKE_BINARY_DIR}/shaders
    ${CMAKE_CURRENT_BINARY_DIR}
)

target_compile_options(Zelda64Recompiled PRIVATE
    -march=nehalem
    -fno-strict-aliasing
    -fms-extensions
)

if (WIN32)
    include(FetchContent)
    # Fetch SDL2 on windows
    FetchContent_Declare(
        sdl2
        URL https://github.com/libsdl-org/SDL/releases/download/release-2.28.5/SDL2-devel-2.28.5-VC.zip
        URL_HASH MD5=d8173db078e54040c666f411c5a6afff
    )
    FetchContent_MakeAvailable(sdl2)
    target_include_directories(Zelda64Recompiled PRIVATE
        ${sdl2_SOURCE_DIR}/include
    )
    target_link_directories(Zelda64Recompiled PRIVATE
        ${sdl2_SOURCE_DIR}/lib/x64
    )

    # Copy SDL2 and dxc DLLs to output folder as post build step
    add_custom_command(TARGET Zelda64Recompiled POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${sdl2_SOURCE_DIR}/lib/x64/SDL2.dll"
            "${CMAKE_SOURCE_DIR}/lib/rt64/src/contrib/dxc/bin/x64/dxil.dll"
            "${CMAKE_SOURCE_DIR}/lib/rt64/src/contrib/dxc/bin/x64/dxcompiler.dll"
            $<TARGET_FILE_DIR:Zelda64Recompiled>)
            
    set_target_properties(
        Zelda64Recompiled
        PROPERTIES
            LINK_FLAGS_DEBUG "/SUBSYSTEM:CONSOLE"
            LINK_FLAGS_RELEASE "/SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup"
            LINK_FLAGS_RELWITHDEBINFO "/SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup"
            LINK_FLAGS_MINSIZEREL "/SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup"
    )

    target_sources(Zelda64Recompiled PRIVATE ${CMAKE_SOURCE_DIR}/icons/app.rc)
endif()

if (LINUX)
    find_package(SDL2 REQUIRED)
    find_package(X11 REQUIRED)
    
    # Generate icon_bytes.c from the app icon PNG.
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/icon_bytes.c ${CMAKE_CURRENT_BINARY_DIR}/icon_bytes.h
        COMMAND file_to_c ${CMAKE_SOURCE_DIR}/icons/512.png icon_bytes ${CMAKE_CURRENT_BINARY_DIR}/icon_bytes.c ${CMAKE_CURRENT_BINARY_DIR}/icon_bytes.h 
        DEPENDS ${CMAKE_SOURCE_DIR}/icons/512.png
    )
    target_sources(Zelda64Recompiled PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/icon_bytes.c)

    message(STATUS "SDL2_FOUND = ${SDL2_FOUND}")
    message(STATUS "SDL2_INCLUDE_DIRS = ${SDL2_INCLUDE_DIRS}")

    target_include_directories(Zelda64Recompiled PRIVATE ${SDL2_INCLUDE_DIRS})

    message(STATUS "X11_FOUND = ${X11_FOUND}")
    message(STATUS "X11_Xrandr_FOUND = ${X11_Xrandr_FOUND}")
    message(STATUS "X11_INCLUDE_DIR = ${X11_INCLUDE_DIR}")
    message(STATUS "X11_LIBRARIES = ${X11_LIBRARIES}")

    target_include_directories(Zelda64Recompiled PRIVATE ${X11_INCLUDE_DIR} ${X11_Xrandr_INCLUDE_PATH})
    target_link_libraries(Zelda64Recompiled PRIVATE ${X11_LIBRARIES} ${X11_Xrandr_LIB})

    find_package(Freetype REQUIRED)

    message(STATUS "FREETYPE_FOUND = ${FREETYPE_FOUND}")
    message(STATUS "FREETYPE_INCLUDE_DIRS = ${FREETYPE_INCLUDE_DIRS}")
    message(STATUS "FREETYPE_LIBRARIES = ${FREETYPE_LIBRARIES}")

    include_directories(${FREETYPE_LIBRARIES})
    target_link_libraries(Zelda64Recompiled PRIVATE ${FREETYPE_LIBRARIES})

    set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
    set(THREADS_PREFER_PTHREAD_FLAG TRUE)
    find_package(Threads REQUIRED)
    
    target_link_libraries(Zelda64Recompiled PRIVATE "-latomic -static-libstdc++" ${CMAKE_DL_LIBS} Threads::Threads)
endif()

target_link_libraries(Zelda64Recompiled PRIVATE
    PatchesLib
    RecompiledFuncs
    SDL2
    rt64
    RmlCore
    RmlDebugger
    nfd
    lunasvg
)

# TODO fix the rt64 CMake script so that this doesn't need to be duplicated here
# For DXC
set (DXC_COMMON_OPTS "-I${PROJECT_SOURCE_DIR}/src")
set (DXC_DXIL_OPTS "-Wno-ignored-attributes")
set (DXC_SPV_OPTS "-spirv" "-fspv-target-env=vulkan1.0" "-fvk-use-dx-layout")
set (DXC_PS_OPTS "${DXC_COMMON_OPTS}" "-E" "PSMain" "-T ps_6_0" "-D DYNAMIC_RENDER_PARAMS")
set (DXC_VS_OPTS "${DXC_COMMON_OPTS}" "-E" "VSMain" "-T vs_6_0" "-D DYNAMIC_RENDER_PARAMS" "-fvk-invert-y")
set (DXC_CS_OPTS "${DXC_COMMON_OPTS}" "-E" "CSMain" "-T cs_6_0")
set (DXC_GS_OPTS "${DXC_COMMON_OPTS}" "-E" "GSMain" "-T gs_6_0")
set (DXC_RT_OPTS "${DXC_COMMON_OPTS}" "-D" "RT_SHADER" "-T" "lib_6_3" "-fspv-target-env=vulkan1.1spirv1.4" "-fspv-extension=SPV_KHR_ray_tracing" "-fspv-extension=SPV_EXT_descriptor_indexing")

if (${WIN32})
    set (DXC "${PROJECT_SOURCE_DIR}/lib/rt64/src/contrib/dxc/bin/x64/dxc.exe")
    add_compile_definitions(NOMINMAX)
else()
    set (DXC "LD_LIBRARY_PATH=${PROJECT_SOURCE_DIR}/lib/rt64/src/contrib/dxc/lib/x64" "${PROJECT_SOURCE_DIR}/lib/rt64/src/contrib/dxc/bin/x64/dxc")
endif()

build_vertex_shader(Zelda64Recompiled "shaders/InterfaceVS.hlsl" "shaders/InterfaceVS.hlsl")
build_pixel_shader (Zelda64Recompiled "shaders/InterfacePS.hlsl" "shaders/InterfacePS.hlsl")

target_sources(Zelda64Recompiled PRIVATE ${SOURCES})

set_property(TARGET Zelda64Recompiled PROPERTY VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}")

