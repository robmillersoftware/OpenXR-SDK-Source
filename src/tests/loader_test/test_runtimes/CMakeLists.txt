# Copyright (c) 2017 The Khronos Group Inc.
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
#
# Author:
#

# Force all compilers to output to binary folder without additional output (like Windows adds "Debug" and "Release" folders)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
foreach(OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES})
    string(TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_CURRENT_BINARY_DIR})
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_CURRENT_BINARY_DIR})
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_CURRENT_BINARY_DIR})
endforeach(OUTPUTCONFIG CMAKE_CONFIGURATION_TYPES)

add_library(test_runtime SHARED
    runtime_test.cpp
)
set_target_properties(test_runtime PROPERTIES FOLDER ${TESTS_FOLDER})

add_dependencies(test_runtime
    xr_global_generated_files
    generate_openxr_header
    generated_rt_json_files
)
target_include_directories(test_runtime
    PRIVATE ${PROJECT_SOURCE_DIR}/src
    PRIVATE ${PROJECT_SOURCE_DIR}/src/common
    PRIVATE ${PROJECT_BINARY_DIR}/include
)
if(VulkanHeaders_FOUND)
    target_include_directories(test_runtime
        PRIVATE ${Vulkan_INCLUDE_DIRS}
    )
endif()

macro(gen_xr_runtime_json filename libfile)
    add_custom_command(OUTPUT ${filename}
        COMMAND
            ${CMAKE_COMMAND} -E env
                ${PYTHON_EXECUTABLE}
                    ${PROJECT_SOURCE_DIR}/src/scripts/generate_runtime_manifest.py -f ${filename} -l ${libfile} ${ARGN}
    DEPENDS
        ${PROJECT_SOURCE_DIR}/src/scripts/generate_runtime_manifest.py
    COMMENT "Generating Runtime JSON ${filename} using -f ${filename} -l ${libfile} ${ARGN}"
    )
endmacro()

if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    target_compile_definitions(test_runtime PRIVATE _CRT_SECURE_NO_WARNINGS)
    # Turn off transitional "changed behavior" warning message for Visual Studio versions prior to 2015.
    # The changed behavior is that constructor initializers are now fixed to clear the struct members.
    target_compile_options(test_runtime PRIVATE "$<$<AND:$<CXX_COMPILER_ID:MSVC>,$<VERSION_LESS:$<CXX_COMPILER_VERSION>,19>>:/wd4351>")
    gen_xr_runtime_json(
        ${PROJECT_BINARY_DIR}/src/tests/loader_test/resources/runtimes/test_runtime.json
        ${CMAKE_CURRENT_BINARY_DIR}/test_runtime.dll
        -b
    )
    FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/test_runtime.def DEF_FILE)
    add_custom_target(copy-test_runtime-def-file ALL
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${DEF_FILE} ${CMAKE_CURRENT_BINARY_DIR}/test_runtime.def
        VERBATIM
    )
	set_target_properties(copy-test_runtime-def-file PROPERTIES FOLDER ${HELPER_FOLDER})
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_compile_options(test_runtime PRIVATE -Wpointer-arith -Wno-unused-function -Wno-sign-compare)
    set_target_properties(test_runtime PROPERTIES LINK_FLAGS "-Wl,-Bsymbolic,--exclude-libs,ALL")
    gen_xr_runtime_json(
        ${PROJECT_BINARY_DIR}/src/tests/loader_test/resources/runtimes/test_runtime.json
        ${CMAKE_CURRENT_BINARY_DIR}/libtest_runtime.so
        -b
    )
endif()

add_custom_target(generated_rt_json_files DEPENDS
    ${PROJECT_BINARY_DIR}/src/tests/loader_test/resources/runtimes/test_runtime.json
)
set_target_properties(generated_rt_json_files PROPERTIES FOLDER ${CODEGEN_FOLDER})

