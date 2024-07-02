cmake_minimum_required(VERSION 3.21)

project(llama_cpp)

option(LLAMA_BUILD "Build llama.cpp shared library and install alongside python package" ON)
option(LLAVA_BUILD "Build llava shared library and install alongside python package" ON)

function(llama_cpp_python_install_target target)
    install(
        TARGETS ${target}
        LIBRARY DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib
        RUNTIME DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib
        ARCHIVE DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib
        FRAMEWORK DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib
        RESOURCE DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib
        PUBLIC_HEADER DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/include
    )
    install(
        TARGETS ${target}
        LIBRARY DESTINATION ${SKBUILD_PLATLIB_DIR}/llama_cpp/lib
        RUNTIME DESTINATION ${SKBUILD_PLATLIB_DIR}/llama_cpp/lib
        ARCHIVE DESTINATION ${SKBUILD_PLATLIB_DIR}/llama_cpp/lib
        FRAMEWORK DESTINATION ${SKBUILD_PLATLIB_DIR}/llama_cpp/lib
        RESOURCE DESTINATION ${SKBUILD_PLATLIB_DIR}/llama_cpp/lib
        PUBLIC_HEADER DESTINATION ${SKBUILD_PLATLIB_DIR}/llama_cpp/include
    )
    
    if(UNIX)
        if(APPLE)
            set_target_properties(${target} PROPERTIES
                INSTALL_RPATH "@loader_path"
                BUILD_WITH_INSTALL_RPATH TRUE
            )
        else()
            set_target_properties(${target} PROPERTIES
                INSTALL_RPATH "$ORIGIN"
                BUILD_WITH_INSTALL_RPATH TRUE
            )
        endif()
    elseif(WIN32)
        # For Windows, we'll use a custom post-build command to copy dependencies
        add_custom_command(TARGET ${target} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                $<TARGET_FILE:${target}>
                ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib/
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                $<TARGET_RUNTIME_DLLS:${target}>
                ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib/
            COMMAND_EXPAND_LISTS
        )
    endif()
endfunction()

if (LLAMA_BUILD)
    set(BUILD_SHARED_LIBS "On")

    if(UNIX)
        set(CMAKE_SKIP_BUILD_RPATH FALSE)
        set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
        set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
        if(APPLE)
            set(CMAKE_INSTALL_RPATH "@loader_path")
        else()
            set(CMAKE_INSTALL_RPATH "$ORIGIN")
        endif()
        set(CMAKE_SKIP_RPATH FALSE)
    elseif(WIN32)
        # For Windows, we need to make sure the DLLs are found at runtime
        set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib)
        set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib)
        set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib)
        set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
    endif()

    # Building llama
    if (APPLE AND NOT CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
        # Need to disable these llama.cpp flags on Apple x86_64,
        # otherwise users may encounter invalid instruction errors
        set(LLAMA_AVX "Off" CACHE BOOL "llama: enable AVX" FORCE)
        set(LLAMA_AVX2 "Off" CACHE BOOL "llama: enable AVX2" FORCE)
        set(LLAMA_FMA "Off" CACHE BOOL "llama: enable FMA" FORCE)
        set(LLAMA_F16C "Off" CACHE BOOL "llama: enable F16C" FORCE)
    endif()

    if (APPLE)
        set(LLAMA_METAL_EMBED_LIBRARY "On" CACHE BOOL "llama: embed metal library" FORCE)
    endif()

    add_subdirectory(vendor/llama.cpp)
    
    # Make sure targets are created before installing
    add_library(llama ALIAS llama)
    add_library(ggml ALIAS ggml_static)
    
    llama_cpp_python_install_target(llama)
    llama_cpp_python_install_target(ggml_static)
    
    if (WIN32)
        # For Windows, we need to make sure all necessary DLLs are installed
        file(GLOB LLAMA_DLLS "${CMAKE_CURRENT_SOURCE_DIR}/llama_cpp/lib/*.dll")
        install(
            FILES ${LLAMA_DLLS}
            DESTINATION ${SKBUILD_PLATLIB_DIR}/llama_cpp/lib
        )
    endif()

    if (LLAVA_BUILD)
        if (LLAMA_CUBLAS OR LLAMA_CUDA)
            add_compile_definitions(GGML_USE_CUBLAS)
            add_compile_definitions(GGML_USE_CUDA)
        endif()

        if (LLAMA_METAL)
            add_compile_definitions(GGML_USE_METAL)
        endif()

        # Building llava
        add_subdirectory(vendor/llama.cpp/examples/llava)
        set_target_properties(llava_shared PROPERTIES OUTPUT_NAME "llava")
        # Set CUDA_ARCHITECTURES to OFF on windows
        if (WIN32)
            set_target_properties(llava_shared PROPERTIES CUDA_ARCHITECTURES OFF)
        endif()
        llama_cpp_python_install_target(llava_shared)
    endif()
endif()
