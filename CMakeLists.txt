# qwen_asr — Qwen3-ASR Pure C Inference Engine
# CMakeLists.txt

cmake_minimum_required(VERSION 3.16)
project(qwen_asr LANGUAGES C)

# ---------------------------------------------------------------------------
# Default build type: Release (matches Makefile's -O3 default)
# ---------------------------------------------------------------------------
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE
        PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")
endif()

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# ---------------------------------------------------------------------------
# Options
# ---------------------------------------------------------------------------
option(QWEN_BLAS "Enable BLAS acceleration (Accelerate on macOS, OpenBLAS on Linux)" ON)

# ---------------------------------------------------------------------------
# Windows guard — C code uses mmap, pthreads, sysconf, dirent.h
# ---------------------------------------------------------------------------
if(WIN32)
    message(WARNING
        "qwen_asr uses POSIX APIs (mmap, pthreads, sysconf, dirent.h) "
        "that are not available on Windows. The build will likely fail.")
endif()

# ---------------------------------------------------------------------------
# Library sources (OBJECT library — mirrors Makefile .o compilation)
# ---------------------------------------------------------------------------
add_library(qwen_asr_lib OBJECT
    qwen_asr.c
    qwen_asr_kernels.c
    qwen_asr_kernels_generic.c
    qwen_asr_kernels_neon.c
    qwen_asr_kernels_avx.c
    qwen_asr_audio.c
    qwen_asr_encoder.c
    qwen_asr_decoder.c
    qwen_asr_tokenizer.c
    qwen_asr_safetensors.c
)

# ---------------------------------------------------------------------------
# Executable
# ---------------------------------------------------------------------------
add_executable(qwen_asr main.c)
target_link_libraries(qwen_asr PRIVATE qwen_asr_lib)

# ---------------------------------------------------------------------------
# Compiler flags (applied to both targets)
# ---------------------------------------------------------------------------
set(_all_targets qwen_asr_lib qwen_asr)

foreach(_tgt IN LISTS _all_targets)
    # Warnings (all build types)
    target_compile_options(${_tgt} PRIVATE -Wall -Wextra)

    # Release flags — match Makefile: -O3 -march=native -ffast-math
    if(NOT CMAKE_CROSSCOMPILING)
        target_compile_options(${_tgt} PRIVATE
            $<$<CONFIG:Release>:-O3 -march=native -ffast-math>
            $<$<CONFIG:RelWithDebInfo>:-march=native -ffast-math>
        )
    endif()

    # Debug flags — match Makefile: -O0 -fsanitize=address -DDEBUG
    target_compile_options(${_tgt} PRIVATE
        $<$<CONFIG:Debug>:-O0 -fsanitize=address>
    )
    target_compile_definitions(${_tgt} PRIVATE
        $<$<CONFIG:Debug>:DEBUG>
    )
endforeach()

# Debug linker flags (sanitizer must also be linked)
target_link_options(qwen_asr PRIVATE
    $<$<CONFIG:Debug>:-fsanitize=address>
)

# ---------------------------------------------------------------------------
# Threading & math
# ---------------------------------------------------------------------------
find_package(Threads REQUIRED)
target_link_libraries(qwen_asr PRIVATE Threads::Threads)
if(UNIX)
    target_link_libraries(qwen_asr PRIVATE m)
endif()

# ---------------------------------------------------------------------------
# BLAS detection
# ---------------------------------------------------------------------------
if(QWEN_BLAS)
    if(APPLE)
        message(STATUS "BLAS: Apple Accelerate")
        foreach(_tgt IN LISTS _all_targets)
            target_compile_definitions(${_tgt} PRIVATE USE_BLAS ACCELERATE_NEW_LAPACK)
        endforeach()
        target_link_libraries(qwen_asr PRIVATE "-framework Accelerate")

    elseif(UNIX)
        # Try pkg-config first, then fall back to find_library + find_path
        find_package(PkgConfig QUIET)
        set(_openblas_found FALSE)

        if(PkgConfig_FOUND)
            pkg_check_modules(_OPENBLAS QUIET openblas)
            if(_OPENBLAS_FOUND)
                set(_openblas_found TRUE)
                set(_openblas_include_dirs "${_OPENBLAS_INCLUDE_DIRS}")
                set(_openblas_libraries "${_OPENBLAS_LIBRARIES}")
                set(_openblas_library_dirs "${_OPENBLAS_LIBRARY_DIRS}")
            endif()
        endif()

        if(NOT _openblas_found)
            find_library(_openblas_lib NAMES openblas)
            find_path(_openblas_inc NAMES cblas.h PATHS /usr/include/openblas)
            if(_openblas_lib AND _openblas_inc)
                set(_openblas_found TRUE)
                set(_openblas_include_dirs "${_openblas_inc}")
                set(_openblas_libraries "${_openblas_lib}")
                set(_openblas_library_dirs "")
            endif()
        endif()

        if(_openblas_found)
            message(STATUS "BLAS: OpenBLAS")
            foreach(_tgt IN LISTS _all_targets)
                target_compile_definitions(${_tgt} PRIVATE USE_BLAS USE_OPENBLAS)
                target_include_directories(${_tgt} PRIVATE ${_openblas_include_dirs})
            endforeach()
            if(_openblas_library_dirs)
                target_link_directories(qwen_asr PRIVATE ${_openblas_library_dirs})
            endif()
            target_link_libraries(qwen_asr PRIVATE ${_openblas_libraries})
        else()
            message(WARNING
                "OpenBLAS not found — building without BLAS.\n"
                "  Install: sudo apt install libopenblas-dev")
        endif()

    elseif(WIN32)
        message(WARNING "BLAS support on Windows is not yet implemented.")
    endif()
endif()

# ---------------------------------------------------------------------------
# Convenience targets
# ---------------------------------------------------------------------------
add_custom_target(download_model
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/download_model.sh
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    USES_TERMINAL
    COMMENT "Downloading Qwen3-ASR model weights"
)

# ---------------------------------------------------------------------------
# Tests
# ---------------------------------------------------------------------------
enable_testing()
add_test(
    NAME regression
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/asr_regression.py
            --binary $<TARGET_FILE:qwen_asr>
            --model-dir ${CMAKE_CURRENT_SOURCE_DIR}/qwen3-asr-1.7b
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

# ---------------------------------------------------------------------------
# Install
# ---------------------------------------------------------------------------
install(TARGETS qwen_asr RUNTIME DESTINATION bin)
