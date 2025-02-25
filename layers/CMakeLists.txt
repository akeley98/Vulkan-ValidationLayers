# ~~~
# Copyright (c) 2014-2021 Valve Corporation
# Copyright (c) 2014-2021 LunarG, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# ~~~

if(WIN32)
    add_definitions(-DVK_USE_PLATFORM_WIN32_KHR -DVK_USE_PLATFORM_WIN32_KHX -DWIN32_LEAN_AND_MEAN)
    add_custom_target(mk_layer_config_dir ALL
                      COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>)
    set_target_properties(mk_layer_config_dir PROPERTIES FOLDER ${LAYERS_HELPER_FOLDER})
elseif(ANDROID)
    add_definitions(-DVK_USE_PLATFORM_ANDROID_KHR -DVK_USE_PLATFORM_ANDROID_KHX)
elseif(APPLE)
    add_definitions(-DVK_USE_PLATFORM_MACOS_MVK -DVK_USE_PLATFORM_METAL_EXT)
    if(CMAKE_GENERATOR MATCHES "^Xcode.*")
        add_custom_target(mk_layer_config_dir ALL
                          COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>)
    endif()
elseif(UNIX AND NOT APPLE) # i.e. Linux
    if(BUILD_WSI_XCB_SUPPORT)
        add_definitions(-DVK_USE_PLATFORM_XCB_KHR -DVK_USE_PLATFORM_XCB_KHX)
    endif()

    if(BUILD_WSI_XLIB_SUPPORT)
        add_definitions(-DVK_USE_PLATFORM_XLIB_KHR -DVK_USE_PLATFORM_XLIB_KHX -DVK_USE_PLATFORM_XLIB_XRANDR_EXT)
    endif()

    if(BUILD_WSI_WAYLAND_SUPPORT)
        add_definitions(-DVK_USE_PLATFORM_WAYLAND_KHR -DVK_USE_PLATFORM_WAYLAND_KHX)
    endif()
else()
    message(FATAL_ERROR "Unsupported Platform!")
endif()

# Configure installation of source files that are dependencies of other repos.
if(BUILD_LAYER_SUPPORT_FILES)
    set(LAYER_UTIL_FILES
        cast_utils.h
        hash_util.h
        hash_vk_types.h
        vk_format_utils.h
        vk_format_utils.cpp
        vk_layer_config.h
        vk_layer_config.cpp
        vk_layer_data.h
        vk_layer_extension_utils.h
        vk_layer_extension_utils.cpp
        vk_layer_logging.h
        vk_layer_utils.h
        vk_layer_utils.cpp
        vk_loader_platform.h
        xxhash.h
        xxhash.c
        generated/vk_validation_error_messages.h
        generated/vk_layer_dispatch_table.h
        generated/vk_dispatch_table_helper.h
        generated/vk_safe_struct.h
        generated/vk_safe_struct.cpp
        generated/vk_enum_string_helper.h
        generated/vk_object_types.h
        generated/vk_extension_helper.h
        generated/vk_typemap_helper.h)
    install(FILES ${LAYER_UTIL_FILES} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/vulkan)
endif()

set(TARGET_NAMES
    VkLayer_khronos_validation)

# System-specific macro to create a library target.
macro(AddVkLayer target LAYER_COMPILE_DEFINITIONS)
    add_library(VkLayer_${target} SHARED ${ARGN})
    set_target_properties(VkLayer_${target} PROPERTIES CXX_STANDARD ${VVL_CPP_STANDARD})
    target_compile_definitions(VkLayer_${target} PUBLIC ${LAYER_COMPILE_DEFINITIONS})
    target_link_libraries(VkLayer_${target} PRIVATE VkLayer_utils)

    if(WIN32)
        target_sources(VkLayer_${target} PRIVATE VkLayer_${target}.def)
        target_compile_definitions(VkLayer_${target} PUBLIC NOMINMAX)
    elseif(APPLE)
        set_target_properties(VkLayer_${target}
                              PROPERTIES LINK_FLAGS
                                         "-Wl"
                                         INSTALL_RPATH
                                         "@loader_path/")
    else(UNIX AND NOT APPLE) # i.e.: Linux
        set_target_properties(VkLayer_${target} PROPERTIES LINK_FLAGS "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/libVkLayer_${target}.map,-Bsymbolic,--exclude-libs,ALL")
    endif()

    install(TARGETS VkLayer_${target} DESTINATION ${CMAKE_INSTALL_LIBDIR})
endmacro()

include_directories(${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/generated ${VulkanHeaders_INCLUDE_DIR})

if(MSVC)
    # Applies to all configurations
    add_definitions(-D_CRT_SECURE_NO_WARNINGS -DNOMINMAX)
    # Avoid: fatal error C1128: number of sections exceeded object file format limit: compile with /bigobj
    add_compile_options("/bigobj")
    # Allow Windows to use multiprocessor compilation
    add_compile_options(/MP)
    # Turn off transitional "changed behavior" warning message for Visual Studio versions prior to 2015. The changed behavior is
    # that constructor initializers are now fixed to clear the struct members.
    add_compile_options("$<$<AND:$<CXX_COMPILER_ID:MSVC>,$<VERSION_LESS:$<CXX_COMPILER_VERSION>,19>>:/wd4351>")
else()
    if(MINGW)
        add_compile_options("-Wa,-mbig-obj")
    endif()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wpointer-arith -Wno-unused-function -Wno-sign-compare")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wpointer-arith -Wno-unused-function -Wno-sign-compare")
endif()


if(ANNOTATED_SPEC_LINK)
    message("-- ANNOTATED_SPEC_LINK is ${ANNOTATED_SPEC_LINK}")
    add_definitions(-DANNOTATED_SPEC_LINK=${ANNOTATED_SPEC_LINK})
endif()

# Clang warns about unused const variables. Generated files may purposely contain unused consts, so silence this warning in Clang
if(CMAKE_C_COMPILER_ID MATCHES "Clang")
    set_source_files_properties(parameter_validation.cpp PROPERTIES COMPILE_FLAGS "-Wno-unused-const-variable")
endif()

set(CHASSIS_LIBRARY_FILES
    generated/chassis.cpp
    generated/layer_chassis_dispatch.cpp
    generated/command_counter_helper.cpp
    generated/vk_safe_struct.cpp
    generated/vk_safe_struct.h
    layer_options.cpp
    state_tracker.cpp
    state_tracker.h
    image_layout_map.cpp
    image_layout_map.h
    range_vector.h
    vk_layer_settings_ext.h
    subresource_adapter.cpp
    subresource_adapter.h
    sync_utils.cpp
    sync_utils.h)

set(CORE_VALIDATION_LIBRARY_FILES
    core_validation.cpp
    core_validation.h
    core_validation_error_enums.h
    core_error_location.h
    core_error_location.cpp
    core_validation_types.h
    drawdispatch.cpp
    convert_to_renderpass2.cpp
    descriptor_sets.cpp
    descriptor_sets.h
    buffer_validation.cpp
    buffer_validation.h
    shader_module.cpp
    shader_module.h
    shader_validation.cpp
    shader_validation.h
    sync_vuid_maps.cpp
    sync_vuid_maps.h
    generated/spirv_validation_helper.cpp
    generated/synchronization_validation_types.cpp
    gpu_validation.cpp
    generated/corechecks_optick_instrumentation.cpp
    xxhash.c)

set(OBJECT_LIFETIMES_LIBRARY_FILES
    generated/object_tracker.cpp
    generated/object_tracker.h
    object_tracker_utils.cpp
    object_lifetime_validation.h)

set(THREAD_SAFETY_LIBRARY_FILES
    generated/thread_safety.cpp
    generated/thread_safety.h)

set(STATELESS_VALIDATION_LIBRARY_FILES
    generated/parameter_validation.cpp
    generated/parameter_validation.h
    parameter_validation_utils.cpp
    stateless_validation.h)

set(BEST_PRACTICES_LIBRARY_FILES
    best_practices_utils.cpp
    generated/best_practices.cpp
    generated/best_practices.h
    best_practices_validation.h
    best_practices_error_enums.h)

set(GPU_ASSISTED_LIBRARY_FILES
    gpu_validation.cpp
    gpu_validation.h)

set(DEBUG_PRINTF_LIBRARY_FILES
    debug_printf.cpp
    debug_printf.h)

set(GPU_UTILITY_LIBRARY_FILES
    gpu_utils.cpp
    gpu_utils.h)

set(SYNC_VALIDATION_LIBRARY_FILES
    synchronization_validation.cpp
    synchronization_validation.h)

# Validation Layer performance instrumentation support using Optick.
# https://optick.dev/ https://github.com/bombomby/optick
# To include Optick instrumentation:
# 1) Download the Optick 1.3.1 release: https://github.com/bombomby/optick/releases/tag/1.3.1.0
# 2) Copy the files from the src directory to external/optick.
# 3) Run CMake configure with -DINSTRUMENT_OPTICK.
set(OPTICK_SOURCE_DIR "${PROJECT_SOURCE_DIR}/external/optick")
if(INSTRUMENT_OPTICK)
    # Optick Instrumentation currently supported only on Microsoft toolchain.
    if(MSVC)
        if(IS_DIRECTORY "${OPTICK_SOURCE_DIR}")
            file(GLOB OPTICK_SOURCE_FILES "${OPTICK_SOURCE_DIR}/*.*")
            source_group("Optick Lib" FILES ${OPTICK_SOURCE_FILES})
            # Disable GPU performance measurements (timestamp queries).
            # Not needed for CPU Validation Layer measurements, but would be interesting to try someday.
            # We'll need to map Vulkan API calls into the layer stack API entry points in order to build.
            set(KHRONOS_LAYER_COMPILE_DEFINITIONS
                -DINSTRUMENT_OPTICK
                -DOPTICK_ENABLE_GPU_D3D12=0
                -DOPTICK_ENABLE_GPU_VULKAN=0)
        else()
            message(WARNING "Optick sources not found at ${OPTICK_SOURCE_DIR} - continuing without instrumentation.")
            set(INSTRUMENT_OPTICK OFF)
        endif()
    else()
        message(WARNING "Optick instrumentation not supported on this platform - continuing without instrumentation.")
        set(INSTRUMENT_OPTICK OFF)
    endif()
endif()

if(BUILD_LAYERS)
    AddVkLayer(khronos_validation "${KHRONOS_LAYER_COMPILE_DEFINITIONS}"
        ${CHASSIS_LIBRARY_FILES}
        ${CORE_VALIDATION_LIBRARY_FILES}
        ${OBJECT_LIFETIMES_LIBRARY_FILES}
        ${THREAD_SAFETY_LIBRARY_FILES}
        ${STATELESS_VALIDATION_LIBRARY_FILES}
        ${BEST_PRACTICES_LIBRARY_FILES}
        ${GPU_UTILITY_LIBRARY_FILES}
        ${GPU_ASSISTED_LIBRARY_FILES}
        ${DEBUG_PRINTF_LIBRARY_FILES}
        ${SYNC_VALIDATION_LIBRARY_FILES}
        ${OPTICK_SOURCE_FILES})

    # Force generation of the PDB file for Release builds.
    # Note that CMake reduces inlining optimization levels for RelWithDebInfo builds.
    if(MSVC)
        target_compile_options(VkLayer_khronos_validation PRIVATE "$<$<CONFIG:Release>:/Zi>")
        if (USE_ROBIN_HOOD_HASHING)
            # This warning produces what look like false positives in robin_hood.h with Visual Studio 2015
            target_compile_options(VkLayer_khronos_validation PRIVATE "$<$<AND:$<CXX_COMPILER_ID:MSVC>,$<VERSION_LESS:$<CXX_COMPILER_VERSION>,19.1>>:/wd4996>")
        endif()
        # Need to use this instead of target_link_options() for older versions of CMake.
        target_link_libraries(VkLayer_khronos_validation PRIVATE "$<$<CONFIG:Release>:-DEBUG:FULL>")
    endif()

    # Khronos validation additional dependencies
    target_include_directories(VkLayer_khronos_validation PRIVATE ${GLSLANG_SPIRV_INCLUDE_DIR})
    target_include_directories(VkLayer_khronos_validation PRIVATE ${SPIRV_TOOLS_INCLUDE_DIR})
    target_include_directories(VkLayer_khronos_validation PRIVATE ${SPIRV_HEADERS_INCLUDE_DIR})
    if(INSTRUMENT_OPTICK)
        target_include_directories(VkLayer_khronos_validation PRIVATE ${OPTICK_SOURCE_DIR})
    endif()
    if (USE_ROBIN_HOOD_HASHING)
        target_include_directories(VkLayer_khronos_validation PRIVATE ${ROBIN_HOOD_HASHING_INCLUDE_DIR})
    endif()
    target_link_libraries(VkLayer_khronos_validation PRIVATE ${SPIRV_TOOLS_LIBRARIES})

    # The output file needs Unix "/" separators or Windows "\" separators On top of that, Windows separators actually need to be doubled
    # because the json format uses backslash escapes
    file(TO_NATIVE_PATH "./" RELATIVE_PATH_PREFIX)
    string(REPLACE "\\"
                   "\\\\"
                   RELATIVE_PATH_PREFIX
                   "${RELATIVE_PATH_PREFIX}")

    # Run each .json.in file through the generator We need to create the generator.cmake script so that the generator can be run at
    # compile time, instead of configure time Running at compile time lets us use cmake generator expressions (TARGET_FILE_NAME and
    # TARGET_FILE_DIR, specifically)
    file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/generator.cmake" "configure_file(\"\${INPUT_FILE}\" \"\${OUTPUT_FILE}\")")
    foreach(TARGET_NAME ${TARGET_NAMES})
        set(CONFIG_DEFINES -DINPUT_FILE="${CMAKE_CURRENT_SOURCE_DIR}/json/${TARGET_NAME}.json.in" -DVK_VERSION=1.2.${vk_header_version})
        # If this json file is not a metalayer, get the needed properties from that target
        if(TARGET ${TARGET_NAME})
            set(CONFIG_DEFINES
                ${CONFIG_DEFINES}
                -DOUTPUT_FILE="$<TARGET_FILE_DIR:${TARGET_NAME}>/${TARGET_NAME}.json"
                -DRELATIVE_LAYER_BINARY="${RELATIVE_PATH_PREFIX}$<TARGET_FILE_NAME:${TARGET_NAME}>")
            # If this json file is a metalayer, make the output path match core validation, and there is no layer binary file
        else()
            set(CONFIG_DEFINES ${CONFIG_DEFINES} -DOUTPUT_FILE="$<TARGET_FILE_DIR:VkLayer_khronos_validation>/${TARGET_NAME}.json")
        endif()
        add_custom_target(${TARGET_NAME}-json ALL
                          COMMAND ${CMAKE_COMMAND} ${CONFIG_DEFINES} -P "${CMAKE_CURRENT_BINARY_DIR}/generator.cmake")
        if(CMAKE_GENERATOR MATCHES "^Visual Studio.*")
            set_target_properties(${TARGET_NAME}-json PROPERTIES FOLDER ${LAYERS_HELPER_FOLDER})
        endif()
    endforeach()

    # For UNIX-based systems, `library_path` should not contain a relative path (indicated by "./") before installing to system
    # directories, so we do not include it in the staging-json files which are used for installation
    if(UNIX)
        foreach(TARGET_NAME ${TARGET_NAMES})
            set(INSTALL_DEFINES
                -DINPUT_FILE="${CMAKE_CURRENT_SOURCE_DIR}/json/${TARGET_NAME}.json.in"
                -DOUTPUT_FILE="${CMAKE_CURRENT_BINARY_DIR}/staging-json/${TARGET_NAME}.json"
                -DVK_VERSION=1.2.${vk_header_version})
            # If this json file is not a metalayer, get the needed properties from that target
            if(TARGET ${TARGET_NAME})
                set(INSTALL_DEFINES ${INSTALL_DEFINES} -DRELATIVE_LAYER_BINARY="$<TARGET_FILE_NAME:${TARGET_NAME}>")
            endif()
            add_custom_target(${TARGET_NAME}-staging-json ALL
                              COMMAND ${CMAKE_COMMAND} ${INSTALL_DEFINES} -P "${CMAKE_CURRENT_BINARY_DIR}/generator.cmake")
        endforeach()
    endif()

    # Install the layer json files
    foreach(TARGET_NAME ${TARGET_NAMES})
        if(WIN32)
            install(FILES $<TARGET_FILE_DIR:${TARGET_NAME}>/${TARGET_NAME}.json
                    DESTINATION ${CMAKE_INSTALL_LIBDIR})
        elseif(UNIX)
            install(FILES ${CMAKE_CURRENT_BINARY_DIR}/staging-json/${TARGET_NAME}.json
                    DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/vulkan/explicit_layer.d)
        endif()
    endforeach()
endif()
