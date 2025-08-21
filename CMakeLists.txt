cmake_minimum_required(VERSION 3.16)

project(tensorflow-micro LANGUAGES C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 11)

#if(NOT CMAKE_BUILD_TYPE)
#  message(STATUS "Setting build type to Release, for debug builds use"
#  "'-DCMAKE_BUILD_TYPE=Debug'.")
# TODO: Force release build. Fix debug compile error.
set(CMAKE_BUILD_TYPE "Release")
#endif()

option(TFLITE_BUILD_TESTING "Build project tests." OFF)
option(TFLITE_STRIP_ERROR_STRINGS "Remove error strings to reduce binary size." ON)

if(DEFINED PICO_SDK_PATH)
    message("Build tensorflow-micro for rpi-pico")
    set(TFLITE_ENABLE_PICO TRUE)
    set(TFLITE_ADDITIONAL_LIBS pico_stdlib)
endif()

set(TFLITE_ROOT ${CMAKE_CURRENT_SOURCE_DIR})
set(TFLITE_DIR "${TFLITE_ROOT}/tensorflow/lite")
set(SIGNAL_DIR "${TFLITE_ROOT}/signal")
set(TFMICRO_DIR "${TFLITE_DIR}/micro")
set(TFMICRO_FRONTEND_DIR "${TFLITE_DIR}/experimental/microfrontend/lib")
set(TFMICRO_KERNELS_DIR "${TFMICRO_DIR}/kernels")
set(TFMICRO_TOOLS_DIR "${TFMICRO_DIR}/tools/make")
set(TFMICRO_DOWNLOAD_DIR "${TFMICRO_TOOLS_DIR}/downloads")

file(GLOB SRC_MICRO
    "${TFMICRO_DIR}/*.cc"
    "${TFMICRO_DIR}/*.c")

file(GLOB SRC_MICRO_FRONTEND
    "${TFMICRO_FRONTEND_DIR}/*.c"
    "${TFMICRO_FRONTEND_DIR}/*.cc")

file(GLOB SRC_KERNELS
    "${TFMICRO_KERNELS_DIR}/*.cc"
    "${TFMICRO_KERNELS_DIR}/*.c")

file(GLOB SRC_TFLITE_KERNELS
    "${TFLITE_DIR}/kernels/*.cc"
    "${TFLITE_DIR}/kernels/*.c"
    "${TFLITE_DIR}/kernels/internal/*.cc"
    "${TFLITE_DIR}/kernels/internal/*.c"
)

file(GLOB SRC_TFLITE_BRIDGE
    "${TFLITE_DIR}/micro/tflite_bridge/*.cc"
    "${TFLITE_DIR}/micro/tflite_bridge/*.c"
)

file(GLOB SRC_SIGNAL_MICRO_KERNELS
    "${SIGNAL_DIR}/micro/kernels/*.c"
    "${SIGNAL_DIR}/micro/kernels/*.cc")

file(GLOB SRC_SIGNAL
    "${SIGNAL_DIR}/src/*.c"
    "${SIGNAL_DIR}/src/*.cc")

file(GLOB SRC_SIGNAL_KISS_FFT
    "${SIGNAL_DIR}/src/kiss_fft_wrappers/*.c"
    "${SIGNAL_DIR}/src/kiss_fft_wrappers/*.cc")

set(SRC_SIGNAL_ALL
    "${SRC_SIGNAL_MICRO_KERNELS}"
    "${SRC_SIGNAL}"
    "${SRC_SIGNAL_KISS_FFT}")

list(REMOVE_ITEM SRC_MICRO_FRONTEND "${TFMICRO_FRONTEND_DIR}/frontend_memmap_main.c")

set(SRC_ALL
    "${SRC_MICRO}"
    "${SRC_KERNELS}"
    "${SRC_TFLITE_KERNELS}"
    "${SRC_MICRO_FRONTEND}"
    "${SRC_TFLITE_BRIDGE}"
    "${SRC_SIGNAL_ALL}"
    "${TFLITE_DIR}/kernels/kernel_util.cc"
    "${TFMICRO_DIR}/memory_planner/greedy_memory_planner.cc"
    "${TFMICRO_DIR}/memory_planner/linear_memory_planner.cc"
    "${TFMICRO_DIR}/memory_planner/non_persistent_buffer_planner_shim.cc"
    "${TFMICRO_DIR}/arena_allocator/non_persistent_arena_buffer_allocator.cc"
    "${TFMICRO_DIR}/arena_allocator/persistent_arena_buffer_allocator.cc"
    "${TFMICRO_DIR}/arena_allocator/recording_single_arena_buffer_allocator.cc"
    "${TFMICRO_DIR}/arena_allocator/single_arena_buffer_allocator.cc"
    "${TFLITE_DIR}/core/c/common.cc"
    "${TFLITE_DIR}/core/api/error_reporter.cc"
    "${TFLITE_DIR}/core/api/flatbuffer_conversions.cc"
    "${TFLITE_DIR}/core/api/tensor_utils.cc"
    "${TFLITE_DIR}/kernels/internal/quantization_util.cc"
    "${TFLITE_DIR}/schema/schema_utils.cc"
)

set(TFLITE_TEST_INCLUDE_DIRS )

if(TFLITE_BUILD_TESTING)
    set(SRC_TEST "${SRC_ALL}")
    list(FILTER SRC_TEST INCLUDE REGEX ".*_test\\.cc$")

    set(disabled_tests
        "${TFMICRO_DIR}/memory_arena_threshold_test.cc"
        "${TFMICRO_DIR}/micro_allocator_test.cc"
        "${TFMICRO_DIR}/recording_micro_allocator_test.cc"
        "${TFMICRO_DIR}/kernels/comparisons_test.cc"
        "${TFMICRO_DIR}/kernels/conv_test.cc"
        "${TFMICRO_DIR}/kernels/lstm_eval_test.cc"
        "${TFMICRO_DIR}/kernels/unidirectional_sequence_lstm_test.cc"
        "${TFMICRO_DIR}/kernels/while_test.cc"
    )

    list(REMOVE_ITEM SRC_TEST ${disabled_tests})
    list(APPEND SRC_ALL "${SIGNAL_DIR}/testdata/fft_test_data.cc")
    set(TFLITE_TEST_INCLUDE_DIRS "${SIGNAL_DIR}/testdata")
endif()

# Remove all test sources from the library
list(FILTER SRC_ALL EXCLUDE REGEX ".*_test\\.cc$")

# replace default logging with pico implementation
if(TFLITE_ENABLE_PICO)
    list(REMOVE_ITEM SRC_ALL REMOVE_ITEM
        "${TFMICRO_DIR}/debug_log.cc"
        "${TFMICRO_DIR}/micro_time.cc")
    list(APPEND SRC_ALL
        "${TFMICRO_DIR}/rp2/debug_log.cc"
        "${TFMICRO_DIR}/rp2/micro_time.cc")
endif()

set(FLATBUFFERS_HEADERS "${TFMICRO_DOWNLOAD_DIR}/flatbuffers/include")
set(KISSFFT_HEADERS "${TFMICRO_DOWNLOAD_DIR}/kissfft")

macro(add_thirdparty_dependency target_name outfile)
    add_custom_command(
        OUTPUT ${outfile}
        WORKING_DIRECTORY "${TFLITE_ROOT}"
        COMMAND ${CMAKE_COMMAND} -E make_directory "${TFMICRO_DOWNLOAD_DIR}"
        COMMAND "${TFMICRO_TOOLS_DIR}/${target_name}_download.sh" "${TFMICRO_DOWNLOAD_DIR}" "${PROJECT_SOURCE_DIR}/"
        DEPENDS "${TFMICRO_TOOLS_DIR}/${target_name}_download.sh"
    )
    add_custom_target(${target_name} DEPENDS ${outfile})
endmacro()

add_thirdparty_dependency(flatbuffers "${TFMICRO_DOWNLOAD_DIR}/flatbuffers")
add_thirdparty_dependency(kissfft ${KISSFFT_HEADERS})

include(FetchContent)
FetchContent_Populate(ruy
    URL https://github.com/google/ruy/archive/d37128311b445e758136b8602d1bbd2a755e115d.zip
    URL_HASH MD5=abf7a91eb90d195f016ebe0be885bb6e
    SOURCE_DIR ruy
)

set(RUY_HEADERS "${ruy_SOURCE_DIR}")

FetchContent_Populate(gemmlowp
    URL https://github.com/google/gemmlowp/archive/719139ce755a0f31cbf1c37f7f98adcc7fc9f425.zip
    URL_HASH MD5=7e8191b24853d75de2af87622ad293ba
    SOURCE_DIR gemmlowp
)

set(GEMMLOWP_HEADERS "${gemmlowp_SOURCE_DIR}")

add_library(tensorflow-micro STATIC ${SRC_ALL})
add_dependencies(tensorflow-micro flatbuffers kissfft)

target_include_directories(tensorflow-micro
    PUBLIC
    "."
    "signal/micro/kernels"
    "signal/src"
    "signal/src/kiss_fft_wrappers"
    ${FLATBUFFERS_HEADERS}
    ${KISSFFT_HEADERS}
    ${GEMMLOWP_HEADERS}
    ${RUY_HEADERS}
    ${TFLITE_TEST_INCLUDE_DIRS}
)

if(TFLITE_STRIP_ERROR_STRINGS)
    set(TF_LITE_STRIP_ERROR_STRINGS TF_LITE_STRIP_ERROR_STRINGS=1)
endif()

target_compile_definitions(tensorflow-micro
    PUBLIC
    TF_LITE_STATIC_MEMORY=1
    TF_LITE_DISABLE_X86_NEON=1
    CMSIS_NN=1
    ${TF_LITE_STRIP_ERROR_STRINGS}
)

set(BUILD_WARNINGS
    -Wsign-compare
    -Wdouble-promotion
    -Wunused-variable
    -Wunused-function
    -Wswitch
    -Wvla
    -Wall
    -Wextra
    -Wmissing-field-initializers
    -Wstrict-aliasing
    -Wno-unused-parameter
)

set(COMMON_BUILD_FLAGS
    -Werror
    -fno-unwind-tables
    -ffunction-sections
    -fdata-sections
    -fmessage-length=0
)

set(BUILD_FLAGS_FOR_CXX
    -fno-rtti
    -fno-exceptions
    -fno-threadsafe-statics
    -Wnon-virtual-dtor
    ${COMMON_BUILD_FLAGS}
    ${BUILD_WARNINGS}
)

set(BUILD_FLAGS_FOR_C
    -Wimplicit-function-declaration
    ${COMMON_BUILD_FLAGS}
    ${BUILD_WARNINGS}
)

target_compile_options(tensorflow-micro PRIVATE
    $<$<COMPILE_LANGUAGE:CXX>:${BUILD_FLAGS_FOR_CXX}>
    $<$<COMPILE_LANGUAGE:C>:${BUILD_FLAGS_FOR_C}>
)

target_link_libraries(tensorflow-micro
    PRIVATE
    -lm
    ${TFLITE_ADDITIONAL_LIBS}
)

if(TFLITE_BUILD_TESTING)
    message("Enable testing for tensorflow-micro ...")
    include(CTest)
    enable_testing()

    foreach(test_file ${SRC_TEST})
        get_filename_component(test_name ${test_file} NAME)
        string(REPLACE ".cc" "" test_name ${test_name})

        if(TARGET ${test_name})
            get_filename_component(directory_name ${test_file} DIRECTORY)
            get_filename_component(parent_directory ${directory_name} NAME)
            set(test_name ${parent_directory}_${test_name})
        endif()

        add_executable(${test_name} "${test_file}")
        target_link_libraries(${test_name} tensorflow-micro)
        add_test(NAME ${test_name} COMMAND ${test_name})
    endforeach()
endif()
