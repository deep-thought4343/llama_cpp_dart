cmake_minimum_required(VERSION 3.22.1)

project(llama_cpp_dart_wrapper)

# 1. Setup OpenCL paths BEFORE adding llama.cpp (only when GGML_OPENCL is ON)
set(OPENCL_HEADERS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/OpenCL-Headers")

if (ANDROID AND GGML_OPENCL)
    message(STATUS "Configuring for Android (OpenCL)...")
    message(STATUS "CMAKE_ANDROID_ARCH_ABI = ${CMAKE_ANDROID_ARCH_ABI}")

    # Define the path to the prebuilt library
    set(OPENCL_LIB_PATH "${CMAKE_CURRENT_SOURCE_DIR}/opencl-libs/android/${CMAKE_ANDROID_ARCH_ABI}/libOpenCL.so")

    # Create the imported target for your wrapper to use later
    add_library(opencl-lib SHARED IMPORTED)
    set_target_properties(opencl-lib PROPERTIES IMPORTED_LOCATION "${OPENCL_LIB_PATH}")

    # CRITICAL: Force llama.cpp (and ggml) to use these paths instead of searching system paths
    set(OpenCL_FOUND TRUE CACHE BOOL "OpenCL found" FORCE)
    set(OpenCL_INCLUDE_DIR "${OPENCL_HEADERS_DIR}" CACHE PATH "OpenCL include dir" FORCE)
    set(OpenCL_LIBRARY "${OPENCL_LIB_PATH}" CACHE FILEPATH "OpenCL library" FORCE)
endif()

# Disable unnecessary llama.cpp build targets
set(LLAMA_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(LLAMA_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(LLAMA_BUILD_SERVER OFF CACHE BOOL "" FORCE)
set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build static libs for embedding" FORCE)

# 2. Add llama.cpp (This builds llama, ggml)
add_subdirectory(llama.cpp)

# Force symbol visibility so that Dart FFI can look up llama/ggml symbols
# when they are linked into our shared libmtmd.so via --whole-archive.
# Without this, LLAMA_API / GGML_API are empty (hidden) in the static build.
if (ANDROID)
    set_target_properties(llama PROPERTIES POSITION_INDEPENDENT_CODE ON)
    target_compile_definitions(llama PRIVATE LLAMA_BUILD)
    target_compile_definitions(llama PUBLIC  LLAMA_SHARED)

    # ggml targets
    foreach(_tgt ggml ggml-base ggml-cpu)
        if (TARGET ${_tgt})
            set_target_properties(${_tgt} PROPERTIES POSITION_INDEPENDENT_CODE ON)
            target_compile_definitions(${_tgt} PRIVATE GGML_BUILD)
            target_compile_definitions(${_tgt} PUBLIC  GGML_SHARED)
        endif()
    endforeach()
endif()

# 3. Define your wrapper library (mtmd)
if (ANDROID)
    find_library(log-lib log)

    file(GLOB MTMD_MODEL_SOURCES llama.cpp/tools/mtmd/models/*.cpp)

    add_library(mtmd SHARED
        llama.cpp/tools/mtmd/mtmd.cpp
        llama.cpp/tools/mtmd/mtmd-audio.cpp
        llama.cpp/tools/mtmd/clip.cpp
        llama.cpp/tools/mtmd/mtmd-helper.cpp
        ${MTMD_MODEL_SOURCES}
    )

    # Include directories
    target_include_directories(mtmd PUBLIC
        "${CMAKE_CURRENT_SOURCE_DIR}/llama.cpp/include"
        "${CMAKE_CURRENT_SOURCE_DIR}/llama.cpp/ggml/include"
        "${CMAKE_CURRENT_SOURCE_DIR}/llama.cpp/common"
        "${CMAKE_CURRENT_SOURCE_DIR}/llama.cpp/tools/mtmd"
        "${CMAKE_CURRENT_SOURCE_DIR}"
    )

    # Link libraries — use --whole-archive so ALL symbols from the static
    # llama/ggml libs are exported in the shared library (Dart FFI needs them).
    target_link_libraries(mtmd
        PUBLIC
            -Wl,--whole-archive
            llama
            ggml
            -Wl,--no-whole-archive
            ${log-lib}
    )

    if (GGML_OPENCL)
        target_include_directories(mtmd PUBLIC "${OPENCL_HEADERS_DIR}")
        target_link_libraries(mtmd PUBLIC opencl-lib)
    endif()

    find_package(Threads REQUIRED)
    target_link_libraries(mtmd PRIVATE Threads::Threads)
endif()