cmake_minimum_required(VERSION 3.13)
set(PROJECT_NAME freertos_pico_template)

set(CMAKE_C_STANDARD 23)
set(CMAKE_CXX_STANDARD 20)

set(PICO_SDK_PATH /root/repos/pico-sdk)

include(pico_sdk_import.cmake)
if (PICO_SDK_VERSION_STRING VERSION_LESS "2.2.0")
    message(FATAL_ERROR "Require at least Raspberry Pi Pico SDK version 2.2.0")
endif()
pico_sdk_init()

project(${PROJECT_NAME} C CXX ASM)

set(FREERTOS_PORT GCC_ARM_CM0 CACHE STRING "")
set(FREERTOS_KERNEL_PATH "" CACHE PATH "Path to FreeRTOS Kernel (leave empty to fetch automatically)")

# freertos_config target is needed by freertos-kernel upstream
add_library(freertos_config INTERFACE)
target_include_directories(freertos_config INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/rtos_config)

if(NOT FREERTOS_KERNEL_PATH)
    include(FetchContent)
    FetchContent_Declare(
        FreeRTOS-Kernel
        GIT_REPOSITORY https://github.com/FreeRTOS/FreeRTOS-Kernel.git
        GIT_TAG main )
    FetchContent_MakeAvailable(FreeRTOS-Kernel)

    set(FREERTOS_KERNEL_PATH ${freertos-kernel_SOURCE_DIR})
    message(STATUS "FreeRTOS Kernel fetched automatically to: ${FREERTOS_KERNEL_PATH}")
else()
    message(STATUS "Using user-specified FreeRTOS Kernel path: ${FREERTOS_KERNEL_PATH}")
endif()

if(NOT EXISTS "${FREERTOS_KERNEL_PATH}/include/FreeRTOS.h")
    message(FATAL_ERROR "FreeRTOS kernel not found in ${FREERTOS_KERNEL_PATH}")
endif()

set(FREERTOS_HEAP_IMPL "heap_4" CACHE STRING "Select FreeRTOS heap implementation (heap_1, heap_2, heap_3, heap_4, heap_5)")
set_property(CACHE FREERTOS_HEAP_IMPL PROPERTY STRINGS heap_1 heap_2 heap_3 heap_4 heap_5)

set(HEAP_IMPL_FILE "${FREERTOS_KERNEL_PATH}/portable/MemMang/${FREERTOS_HEAP_IMPL}.c")
if(NOT EXISTS "${HEAP_IMPL_FILE}")
    message(FATAL_ERROR "Selected heap implementation file does not exist: ${HEAP_IMPL_FILE}")
endif()
message(STATUS "Using heap implementation: ${FREERTOS_HEAP_IMPL}")

add_library(freertos_core STATIC
    ${FREERTOS_KERNEL_PATH}/croutine.c
    ${FREERTOS_KERNEL_PATH}/event_groups.c
    ${FREERTOS_KERNEL_PATH}/list.c
    ${FREERTOS_KERNEL_PATH}/queue.c
    ${FREERTOS_KERNEL_PATH}/stream_buffer.c
    ${FREERTOS_KERNEL_PATH}/tasks.c
    ${FREERTOS_KERNEL_PATH}/timers.c
    ${CMAKE_CURRENT_SOURCE_DIR}/pico_src/port.c
    ${HEAP_IMPL_FILE})
    
target_link_libraries(freertos_core 
    freertos_config
    pico_stdlib
    pico_base_headers
    hardware_clocks
    hardware_exception
    pico_multicore)

target_compile_definitions(freertos_core INTERFACE
    LIB_FREERTOS_KERNEL=1
    FREE_RTOS_KERNEL_SMP=1)

target_include_directories(freertos_core PUBLIC
    ${FREERTOS_KERNEL_PATH}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/pico_inc)

add_executable(${PROJECT_NAME} src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE freertos_core)