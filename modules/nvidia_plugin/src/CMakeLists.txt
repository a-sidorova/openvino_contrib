# Copyright (C) 2018-2021 Intel Corporation
# SPDX-License-Identifier: Apache-2.0
#

set(TARGET_NAME openvino_nvidia_gpu_plugin)
set(OBJ_NAME ${TARGET_NAME}_obj)

set(SOURCE_MASKS *.cpp *.cu)
set(HEADER_MASKS *.hpp *.cuh)

file(GLOB_RECURSE
	SOURCES
	CONFIGURE_DEPENDS
	${SOURCE_MASKS}
)
list(FILTER SOURCES EXCLUDE REGEX ".*cuda_create_plugin\\.cpp$")
list(FILTER SOURCES EXCLUDE REGEX ".*cuda_create_extensions\\.cpp$")
list(FILTER SOURCES EXCLUDE REGEX ".*ops/examples/.*$")
file(GLOB_RECURSE
	HEADERS
	${HEADER_MASKS}
)

set_source_files_properties(*.cu *.cuh PROPERTIES LANGUAGE CUDA)

add_library(${OBJ_NAME} STATIC ${SOURCES})
target_compile_definitions(${OBJ_NAME} PRIVATE IMPLEMENT_INFERENCE_ENGINE_PLUGIN IMPLEMENT_OPENVINO_EXTENSION_API)

if(ENABLE_PROXY)
    set(PLUGIN_DEFAULT_CONFIG "PROXY_CONFIGURATION_ALIAS:GPU;PROXY_CONFIGURATION_PRIORITY:1;PROXY_CONFIGURATION_FALLBACK:GPU")
endif()

# Adds a shared library with plugin
ov_add_plugin(NAME ${TARGET_NAME}
              DEVICE_NAME "NVIDIA"
              SOURCES ${HEADERS} cuda_create_plugin.cpp cuda_create_extensions.cpp
              DEFAULT_CONFIG ${PLUGIN_DEFAULT_CONFIG}
              VERSION_DEFINES_FOR cuda_create_plugin.cpp
              ADD_CLANG_FORMAT)

# Enable support of CC for the plugin
ov_mark_target_as_cc(${TARGET_NAME})

set_property(TARGET ${OBJ_NAME} PROPERTY CUDA_ARCHITECTURES ${CMAKE_CUDA_ARCHITECTURES})

ieTargetLinkWholeArchive(${TARGET_NAME} ${OBJ_NAME})

target_include_directories(${OBJ_NAME}
                           SYSTEM PUBLIC
                           "${CUDAToolkit_INCLUDE_DIRS}"
)
target_include_directories(${OBJ_NAME}
                           PUBLIC
                           "${CUTENSOR_INCLUDE_DIR}"
                           "${CMAKE_CURRENT_SOURCE_DIR}"
                           "${IE_MAIN_CUDA_PLUGIN_SOURCE_DIR}/include"
                           PRIVATE $<TARGET_PROPERTY:openvino::conditional_compilation,INTERFACE_INCLUDE_DIRECTORIES>
)

target_link_libraries(${OBJ_NAME}
                      PUBLIC
                      GSL
                      fmt-header-only
                      openvino::runtime
                      openvino::runtime::dev
                      ov_core_dev
                      PRIVATE
                      CUDA::cudart
                      CUDA::cublas
                      CUDA::cudnn
                      CUDA::cutensor
                      ${NGRAPH_LIBRARIES}
)

# set_target_properties(${OBJ_NAME} PROPERTIES INTERPROCEDURAL_OPTIMIZATION_RELEASE ${ENABLE_LTO})


# ATTENTION: uncomment to register a plugin in the plugins.xml file
if(ENABLE_NVIDIA)
    ov_register_plugins(MAIN_TARGET ${TARGET_NAME}
                        POSSIBLE_PLUGINS ${TARGET_NAME})
endif()
