cmake_minimum_required(VERSION 3.19)
cmake_policy(SET CMP0114 NEW)
cmake_policy(SET CMP0116 NEW)
if (POLICY CMP0147)
    # Parallel build custom build steps
    cmake_policy(SET CMP0147 NEW)
endif()

find_package(Vulkan COMPONENTS glslc REQUIRED)

if (NOT Vulkan_GLSLC_EXECUTABLE OR Vulkan_GLSLC_EXECUTABLE MATCHES "NOTFOUND")
    message(STATUS "Vulkan_GLSLC_EXECUTABLE not found, searching in NDK: ${ANDROID_NDK}")
    find_program(Vulkan_GLSLC_EXECUTABLE
        NAMES glslc glslc.exe
        PATHS "${ANDROID_NDK}/shader-tools/windows-x86_64"
              "${ANDROID_NDK}/shader-tools/linux-x86_64"
              "${ANDROID_NDK}/shader-tools/darwin-x86_64"
        NO_DEFAULT_PATH
    )
    if (Vulkan_GLSLC_EXECUTABLE)
        message(STATUS "Found glslc in NDK: ${Vulkan_GLSLC_EXECUTABLE}")
        set(Vulkan_GLSLC_EXECUTABLE "${Vulkan_GLSLC_EXECUTABLE}" CACHE FILEPATH "Path to glslc" FORCE)
    endif()
endif()

if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    # Parallel build object files
    add_definitions(/MP)
endif()

function(detect_host_compiler)
    if (CMAKE_HOST_SYSTEM_NAME STREQUAL "Windows")
        # Clear cache variables to ensure fresh search for absolute paths
        unset(HOST_C_COMPILER CACHE)
        unset(HOST_CXX_COMPILER CACHE)
        
        # 0. If we found vcvars64.bat, we can assume MSVC (cl.exe) is available
        # and will be accessible via our environment wrapper script.
        if (VC_VARS_PATH)
            message(STATUS "Using MSVC via environment wrapper (VC_VARS_PATH=${VC_VARS_PATH})")
            set(HOST_C_COMPILER "cl.exe" PARENT_SCOPE)
            set(HOST_CXX_COMPILER "cl.exe" PARENT_SCOPE)
            return()
        endif()

        # 1. Try to find a compiler that is NOT in the Android/NDK paths
        # We prefer gcc (MinGW) or clang (LLVM) as they usually don't require environment injection like cl.exe
        # but we must avoid the NDK's clang which is for cross-compiling.
        find_program(HOST_C_COMPILER 
            NAMES gcc.exe clang.exe cl.exe
            HINTS "C:/msys64/mingw64/bin" "C:/Program Files/LLVM/bin" "C:/Strawberry/c/bin"
            NO_CMAKE_FIND_ROOT_PATH
        )
        
        # 2. If the found compiler is in the NDK, it's likely incorrect for host tools
        if (HOST_C_COMPILER MATCHES ".*[Aa]ndroid.*" OR HOST_C_COMPILER MATCHES ".*[Nn][Dd][Kk].*")
            set(NDK_COMPILER "${HOST_C_COMPILER}")
            unset(HOST_C_COMPILER CACHE)
            # Try to find again, but excluding NDK paths this time
            # We can't easily exclude paths in find_program, but we can look in common system paths first
            find_program(HOST_C_COMPILER 
                NAMES gcc.exe cl.exe clang.exe 
                PATHS "C:/Windows/System32" "C:/msys64/mingw64/bin" "C:/Program Files/LLVM/bin"
                NO_DEFAULT_PATH
            )
            if (NOT HOST_C_COMPILER)
                # Fallback to standard find but if it's still NDK, we might be in trouble
                find_program(HOST_C_COMPILER NAMES cl.exe NO_CMAKE_FIND_ROOT_PATH)
            endif()
        endif()

        # Repeat for CXX
        find_program(HOST_CXX_COMPILER 
            NAMES g++.exe clang++.exe cl.exe
            HINTS "C:/msys64/mingw64/bin" "C:/Program Files/LLVM/bin" "C:/Strawberry/c/bin"
            NO_CMAKE_FIND_ROOT_PATH
        )
        if (HOST_CXX_COMPILER MATCHES ".*[Aa]ndroid.*" OR HOST_CXX_COMPILER MATCHES ".*[Nn][Dd][Kk].*")
            unset(HOST_CXX_COMPILER CACHE)
            find_program(HOST_CXX_COMPILER 
                NAMES g++.exe cl.exe clang++.exe 
                PATHS "C:/Windows/System32" "C:/msys64/mingw64/bin" "C:/Program Files/LLVM/bin"
                NO_DEFAULT_PATH
            )
            if (NOT HOST_CXX_COMPILER)
                find_program(HOST_CXX_COMPILER NAMES cl.exe NO_CMAKE_FIND_ROOT_PATH)
            endif()
        endif()
    else()
        find_program(HOST_C_COMPILER NAMES gcc clang NO_CMAKE_FIND_ROOT_PATH)
        find_program(HOST_CXX_COMPILER NAMES g++ clang++ NO_CMAKE_FIND_ROOT_PATH)
    endif()

    if (NOT HOST_C_COMPILER OR NOT HOST_CXX_COMPILER)
        message(WARNING "Host compiler not found or is pointing to NDK. Shader generation might fail.")
    endif()

    set(HOST_C_COMPILER "${HOST_C_COMPILER}" PARENT_SCOPE)
    set(HOST_CXX_COMPILER "${HOST_CXX_COMPILER}" PARENT_SCOPE)
    message(STATUS "Detected host C compiler: ${HOST_C_COMPILER}")
    message(STATUS "Detected host CXX compiler: ${HOST_CXX_COMPILER}")
endfunction()

# Function to test shader extension support
# Parameters:
#  EXTENSION_NAME - Name of the extension to test (e.g., "GL_EXT_integer_dot_product")
#  TEST_SHADER_FILE - Path to the test shader file
#  RESULT_VARIABLE - Name of the variable to set (ON/OFF) based on test result
function(test_shader_extension_support EXTENSION_NAME TEST_SHADER_FILE RESULT_VARIABLE)
    execute_process(
        COMMAND ${Vulkan_GLSLC_EXECUTABLE} -o - -fshader-stage=compute --target-env=vulkan1.3 "${TEST_SHADER_FILE}"
        OUTPUT_VARIABLE glslc_output
        ERROR_VARIABLE glslc_error
    )

    if (${glslc_error} MATCHES ".*extension not supported: ${EXTENSION_NAME}.*")
        message(STATUS "${EXTENSION_NAME} not supported by glslc")
        set(${RESULT_VARIABLE} OFF PARENT_SCOPE)
    else()
        message(STATUS "${EXTENSION_NAME} supported by glslc")
        set(${RESULT_VARIABLE} ON PARENT_SCOPE)
        add_compile_definitions(${RESULT_VARIABLE})

        # Ensure the extension support is forwarded to vulkan-shaders-gen
        list(APPEND VULKAN_SHADER_GEN_CMAKE_ARGS -D${RESULT_VARIABLE}=ON)
        set(VULKAN_SHADER_GEN_CMAKE_ARGS "${VULKAN_SHADER_GEN_CMAKE_ARGS}" PARENT_SCOPE)
    endif()
endfunction()

if (Vulkan_FOUND)
    message(STATUS "Vulkan found")

    ggml_add_backend_library(ggml-vulkan
                             ggml-vulkan.cpp
                             ../../include/ggml-vulkan.h
                            )

    set(VULKAN_SHADER_GEN_CMAKE_ARGS "")

    # Test all shader extensions
    test_shader_extension_support(
        "GL_KHR_cooperative_matrix"
        "${CMAKE_CURRENT_SOURCE_DIR}/vulkan-shaders/feature-tests/coopmat.comp"
        "GGML_VULKAN_COOPMAT_GLSLC_SUPPORT"
    )

    test_shader_extension_support(
        "GL_NV_cooperative_matrix2"
        "${CMAKE_CURRENT_SOURCE_DIR}/vulkan-shaders/feature-tests/coopmat2.comp"
        "GGML_VULKAN_COOPMAT2_GLSLC_SUPPORT"
    )

    test_shader_extension_support(
        "GL_EXT_integer_dot_product"
        "${CMAKE_CURRENT_SOURCE_DIR}/vulkan-shaders/feature-tests/integer_dot.comp"
        "GGML_VULKAN_INTEGER_DOT_GLSLC_SUPPORT"
    )

    test_shader_extension_support(
        "GL_EXT_bfloat16"
        "${CMAKE_CURRENT_SOURCE_DIR}/vulkan-shaders/feature-tests/bfloat16.comp"
        "GGML_VULKAN_BFLOAT16_GLSLC_SUPPORT"
    )

    target_link_libraries(ggml-vulkan PRIVATE Vulkan::Vulkan)
    target_include_directories(ggml-vulkan PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

    # Workaround to the "can't dereference invalidated vector iterator" bug in clang-cl debug build
    # Posssibly relevant: https://stackoverflow.com/questions/74748276/visual-studio-no-displays-the-correct-length-of-stdvector
    if (MSVC AND CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        add_compile_definitions(_ITERATOR_DEBUG_LEVEL=0)
    endif()

    if (GGML_VULKAN_CHECK_RESULTS)
        add_compile_definitions(GGML_VULKAN_CHECK_RESULTS)
    endif()

    if (GGML_VULKAN_DEBUG)
        add_compile_definitions(GGML_VULKAN_DEBUG)
    endif()

    if (GGML_VULKAN_MEMORY_DEBUG)
        add_compile_definitions(GGML_VULKAN_MEMORY_DEBUG)
    endif()

    if (GGML_VULKAN_SHADER_DEBUG_INFO)
        add_compile_definitions(GGML_VULKAN_SHADER_DEBUG_INFO)
        list(APPEND VULKAN_SHADER_GEN_CMAKE_ARGS -DGGML_VULKAN_SHADER_DEBUG_INFO=ON)
    endif()

    if (GGML_VULKAN_VALIDATE)
        add_compile_definitions(GGML_VULKAN_VALIDATE)
    endif()

    if (GGML_VULKAN_RUN_TESTS)
        add_compile_definitions(GGML_VULKAN_RUN_TESTS)
    endif()

    # Set up toolchain for host compilation whether cross-compiling or not
    if (CMAKE_CROSSCOMPILING)
        if (GGML_VULKAN_SHADERS_GEN_TOOLCHAIN)
            set(HOST_CMAKE_TOOLCHAIN_FILE ${GGML_VULKAN_SHADERS_GEN_TOOLCHAIN})
        else()
            # Search for vcvars64.bat in common locations first to inform compiler detection
            if (WIN32 AND NOT VC_VARS_PATH)
                message(WARNING "DEBUG: Searching for Visual Studio host compiler environment...")
                
                # 1. Try using where.exe to see if cl.exe is ALREADY in the path
                execute_process(COMMAND where cl.exe OUTPUT_VARIABLE WHERE_CL OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
                if (WHERE_CL)
                    message(WARNING "DEBUG: Found cl.exe in PATH: ${WHERE_CL}")
                    set(HOST_C_COMPILER "cl.exe" CACHE INTERNAL "")
                    set(HOST_CXX_COMPILER "cl.exe" CACHE INTERNAL "")
                endif()

                # 2. Try using vswhere.exe
                if (NOT HOST_C_COMPILER)
                    set(VSWHERES "C:/Program Files (x86)/Microsoft Visual Studio/Installer/vswhere.exe" "$ENV{ProgramFiles\(x86\)}/Microsoft Visual Studio/Installer/vswhere.exe")
                    foreach(V ${VSWHERES})
                        if (EXISTS "${V}")
                            message(WARNING "DEBUG: Found vswhere at: ${V}")
                            execute_process(COMMAND "${V}" -latest -products * -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath
                                            OUTPUT_VARIABLE VS_INSTALL_PATH
                                            OUTPUT_STRIP_TRAILING_WHITESPACE)
                            if (VS_INSTALL_PATH)
                                message(WARNING "DEBUG: vswhere reported installation path: ${VS_INSTALL_PATH}")
                                set(POTENTIAL_VARS "${VS_INSTALL_PATH}/VC/Auxiliary/Build/vcvars64.bat")
                                if (EXISTS "${POTENTIAL_VARS}")
                                    message(WARNING "DEBUG: Confirmed vcvars64.bat exists at: ${POTENTIAL_VARS}")
                                    set(VC_VARS_PATH "${POTENTIAL_VARS}" CACHE INTERNAL "Path to vcvars64.bat")
                                    break()
                                endif()
                            endif()
                        endif()
                    endforeach()
                endif()

                # 3. Last resort: Hardcoded common paths
                if (NOT HOST_C_COMPILER AND NOT VC_VARS_PATH)
                    set(VC_VARS_LOCATIONS
                        "C:/Program Files (x86)/Microsoft Visual Studio/2019/BuildTools/VC/Auxiliary/Build/vcvars64.bat"
                        "C:/Program Files (x86)/Microsoft Visual Studio/2017/BuildTools/VC/Auxiliary/Build/vcvars64.bat"
                        "C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Auxiliary/Build/vcvars64.bat"
                        "C:/Program Files (x86)/Microsoft Visual Studio/2019/Community/VC/Auxiliary/Build/vcvars64.bat"
                        "C:/Program Files/Microsoft Visual Studio/2022/BuildTools/VC/Auxiliary/Build/vcvars64.bat"
                    )
                    foreach(LOC ${VC_VARS_LOCATIONS})
                        if(EXISTS "${LOC}")
                            message(WARNING "DEBUG: Found hardcoded VS environment script at: ${LOC}")
                            set(VC_VARS_PATH "${LOC}" CACHE INTERNAL "Path to vcvars64.bat")
                            break()
                        endif()
                    endforeach()
                endif()

                if (NOT HOST_C_COMPILER AND NOT VC_VARS_PATH)
                    # Try one more time with backslashes just in case
                    set(LOC_BS "C:\\Program Files (x86)\\Microsoft Visual Studio\\2019\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat")
                    if (EXISTS "${LOC_BS}")
                        message(WARNING "DEBUG: Found VS environment script using backslashes: ${LOC_BS}")
                        set(VC_VARS_PATH "${LOC_BS}" CACHE INTERNAL "Path to vcvars64.bat")
                    endif()
                endif()

                if (NOT HOST_C_COMPILER AND NOT VC_VARS_PATH)
                    message(WARNING "DEBUG: Could not find Visual Studio environment. Host tools might fail to build.")
                endif()
            endif()

            detect_host_compiler()
            if (NOT HOST_C_COMPILER OR NOT HOST_CXX_COMPILER)
                message(FATAL_ERROR "Host compiler not found. Please ensure a host C/C++ compiler (gcc, clang, or MSVC) is in your PATH or Visual Studio is installed.")
            else()
                message(STATUS "Host compiler: ${HOST_C_COMPILER} ${HOST_CXX_COMPILER}")
            endif()
            configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/host-toolchain.cmake.in ${CMAKE_BINARY_DIR}/host-toolchain.cmake @ONLY)
            set(HOST_CMAKE_TOOLCHAIN_FILE ${CMAKE_BINARY_DIR}/host-toolchain.cmake)
        endif()
    else()
        # For non-cross-compiling, use empty toolchain (use host compiler)
        set(HOST_CMAKE_TOOLCHAIN_FILE "")
    endif()

    include(ExternalProject)

    if (CMAKE_CROSSCOMPILING)
        list(APPEND VULKAN_SHADER_GEN_CMAKE_ARGS -DCMAKE_TOOLCHAIN_FILE=${HOST_CMAKE_TOOLCHAIN_FILE})
        message(STATUS "vulkan-shaders-gen toolchain file: ${HOST_CMAKE_TOOLCHAIN_FILE}")
    endif()

    set(VSG_CMAKE_ARGS
        "-DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}/$<CONFIG>"
        "-DCMAKE_INSTALL_BINDIR=."
        "-DCMAKE_BUILD_TYPE=$<CONFIG>"
        "-DCMAKE_MAKE_PROGRAM=${CMAKE_MAKE_PROGRAM}"
        "-G${CMAKE_GENERATOR}"
        ${VULKAN_SHADER_GEN_CMAKE_ARGS}
    )

    # Use the discovered VC_VARS_PATH to create a wrapper
    if (CMAKE_HOST_SYSTEM_NAME STREQUAL "Windows" AND VC_VARS_PATH AND EXISTS "${VC_VARS_PATH}")
        file(TO_NATIVE_PATH "${VC_VARS_PATH}" VC_VARS_PATH_NATIVE)
        set(WRAPPER_SCRIPT "${CMAKE_CURRENT_BINARY_DIR}/vsg_env_wrapper.bat")
        file(WRITE "${WRAPPER_SCRIPT}" "@echo off\r\ncall \"${VC_VARS_PATH_NATIVE}\" >nul\r\n%*\r\n")
        message(STATUS "Created VS environment wrapper at: ${WRAPPER_SCRIPT}")
        set(VSG_WRAPPER cmd /c "${WRAPPER_SCRIPT}")
    else()
        set(VSG_WRAPPER "")
    endif()

    ExternalProject_Add(
        vulkan-shaders-gen
        SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/vulkan-shaders
        CMAKE_ARGS ${VSG_CMAKE_ARGS}
        CONFIGURE_COMMAND ${VSG_WRAPPER} ${CMAKE_COMMAND} ${VSG_CMAKE_ARGS} ${CMAKE_CURRENT_SOURCE_DIR}/vulkan-shaders
        BUILD_COMMAND ${VSG_WRAPPER} ${CMAKE_COMMAND} --build . --config $<CONFIG>
        INSTALL_COMMAND ${VSG_WRAPPER} ${CMAKE_COMMAND} -E env --unset=DESTDIR ${CMAKE_COMMAND} --install . --config $<CONFIG>
        BUILD_ALWAYS TRUE
    )

    set (_ggml_vk_host_suffix $<IF:$<STREQUAL:${CMAKE_HOST_SYSTEM_NAME},Windows>,.exe,>)
    set (_ggml_vk_genshaders_cmd "${CMAKE_BINARY_DIR}/$<CONFIG>/vulkan-shaders-gen${_ggml_vk_host_suffix}")
    set (_ggml_vk_header     "${CMAKE_CURRENT_BINARY_DIR}/ggml-vulkan-shaders.hpp")
    set (_ggml_vk_input_dir  "${CMAKE_CURRENT_SOURCE_DIR}/vulkan-shaders")
    set (_ggml_vk_output_dir "${CMAKE_CURRENT_BINARY_DIR}/vulkan-shaders.spv")

    file(GLOB _ggml_vk_shader_files CONFIGURE_DEPENDS "${_ggml_vk_input_dir}/*.comp")

    # Because external projects do not provide source-level tracking,
    # the vulkan-shaders-gen sources need to be explicitly added to
    # ensure that changes will cascade into shader re-generation.

    file(GLOB _ggml_vk_shaders_gen_sources
              CONFIGURE_DEPENDS "${_ggml_vk_input_dir}/*.cpp"
                                "${_ggml_vk_input_dir}/*.h")

    add_custom_command(
        OUTPUT ${_ggml_vk_header}
        COMMAND ${VSG_WRAPPER} "${_ggml_vk_genshaders_cmd}"
            --output-dir ${_ggml_vk_output_dir}
            --target-hpp ${_ggml_vk_header}
        DEPENDS ${_ggml_vk_shaders_gen_sources}
                vulkan-shaders-gen
        COMMENT "Generate vulkan shaders header"
    )
    target_sources(ggml-vulkan PRIVATE ${_ggml_vk_header})

    foreach (file_full ${_ggml_vk_shader_files})
        get_filename_component(file ${file_full} NAME)
        set (_ggml_vk_target_cpp "${CMAKE_CURRENT_BINARY_DIR}/${file}.cpp")

        add_custom_command(
            OUTPUT  ${_ggml_vk_target_cpp}
            DEPFILE ${_ggml_vk_target_cpp}.d
            COMMAND ${VSG_WRAPPER} "${_ggml_vk_genshaders_cmd}"
                --glslc      ${Vulkan_GLSLC_EXECUTABLE}
                --source     ${file_full}
                --output-dir ${_ggml_vk_output_dir}
                --target-hpp ${_ggml_vk_header}
                --target-cpp ${_ggml_vk_target_cpp}
            DEPENDS ${file_full}
                    ${_ggml_vk_shaders_gen_sources}
                    vulkan-shaders-gen
            COMMENT "Generate vulkan shaders for ${file}"
        )
        target_sources(ggml-vulkan PRIVATE ${_ggml_vk_target_cpp})
    endforeach()

else()
    message(WARNING "Vulkan not found")
endif()
