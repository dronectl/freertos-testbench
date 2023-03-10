# 3.22 ubuntu-latest standard
cmake_minimum_required(VERSION 3.22)
# Load toolchain files
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_SOURCE_DIR}/cmake/arm-none-eabi-gcc.cmake)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
# For debug, no change detected ...
# set(CMAKE_VERBOSE_MAKEFILE ON)
# global project name
set(EXECUTABLE "freertostb")
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

# cmake build guard
if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
  message(FATAL_ERROR "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there.")
endif()

# set default build type if none specified
if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "No build type was set. Setting build type to 'Release'")
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose cmake build type: Debug Release" FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif()

# Options for DEBUG build
set(CMAKE_C_FLAGS_DEBUG "-DFREERTOSTB_DEBUG -O0 -g" CACHE INTERNAL "C Compiler options for debug build type")
set(CMAKE_ASM_FLAGS_DEBUG "-g" CACHE INTERNAL "ASM Compiler options for debug build type")
set(CMAKE_EXE_LINKER_FLAGS_DEBUG "" CACHE INTERNAL "Linker options for debug build type")

# Options for RELEASE build
set(CMAKE_C_FLAGS_RELEASE "-Os" CACHE INTERNAL "C Compiler options for release build type")
set(CMAKE_ASM_FLAGS_RELEASE "" CACHE INTERNAL "ASM Compiler options for release build type")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "" CACHE INTERNAL "Linker options for release build type")

project(
  ${EXECUTABLE}
  LANGUAGES C ASM
  VERSION 0.1.0
  DESCRIPTION "Hardware in the loop test executor and reporting aggregation system."
)

message(STATUS "Configuring config.h header")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h.in config.h @ONLY)

# global include for "config.h"
include_directories(${CMAKE_CURRENT_BINARY_DIR})

include(FetchContent)
message(STATUS "Fetching STM32F4xx headers ...")
FetchContent_Declare(
  stm32f4
  GIT_REPOSITORY https://github.com/STMicroelectronics/STM32CubeF4.git
  GIT_TAG v1.27.1
)

FetchContent_MakeAvailable(stm32f4)
# set STM32F4xx include paths
set(
  STM32CUBEF4_INCLUDE
  ${stm32f4_SOURCE_DIR}/Drivers/CMSIS/Device/ST/STM32F4xx/Include
  ${stm32f4_SOURCE_DIR}/Drivers/CMSIS/Core/Include
)
# Include the FreeRTOS source code
message(STATUS "Fetching FreeRTOS kernel ...")
set(FREERTOS_CONFIG_FILE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include CACHE STRING "")
FetchContent_Declare(
  freertos_kernel
  GIT_REPOSITORY https://github.com/FreeRTOS/FreeRTOS-Kernel.git
  GIT_TAG V10.5.1
)
add_library(freertos_config INTERFACE)
target_include_directories(freertos_config SYSTEM INTERFACE include)
target_compile_definitions(freertos_config INTERFACE projCOVERAGE_TEST=0)
set(FREERTOS_HEAP "4" CACHE STRING "" FORCE)
# Select the native compile PORT
set(FREERTOS_PORT "GCC_POSIX" CACHE STRING "" FORCE)
# Select the cross-compile PORT
if (CMAKE_CROSSCOMPILING)
  set(FREERTOS_PORT "GCC_ARM_CM4F" CACHE STRING "" FORCE)
endif()
FetchContent_MakeAvailable(freertos_kernel)

# set compile flags for STM32F411XE 
list(APPEND STM32F411XE_C_FLAGS -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard)
list(APPEND STM32F411XE_COMPILE_DEFINITIONS -D${MCU} -DSTM32F411VETx -DSTM32F411E_DISCO)
list(APPEND STM32F411XE_LINKER_FLAGS -T${LINKER_SCRIPT} -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard)


add_executable(
  ${EXECUTABLE}
  src/main.c
  src/system_stm32f4xx.c
  src/stm32f4xx_it.c
  startup/startup_stm32f411vetx.s
)
target_link_libraries(${EXECUTABLE} LINK_PUBLIC freertos_kernel)
set_target_properties(${EXECUTABLE} PROPERTIES OUTPUT_NAME ${EXECUTABLE}.elf)
# for config.h inclusion
target_include_directories(${EXECUTABLE} PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_include_directories(${EXECUTABLE} PRIVATE include)
target_include_directories(${EXECUTABLE} PRIVATE ${STM32CUBEF4_INCLUDE})

message(STATUS "Configuring config.h header")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h.in config.h @ONLY)


# find code generation tooling
message(STATUS "Discovering cgt executables.")
find_program(ARM_OBJCPY NAMES ${TOOLCHAIN_PREFIX}-objcopy${TOOLCHAIN_EXT} PATHS ${TOOLCHAIN_DIR} NO_DEFAULT_PATH)
find_program(ARM_OBJDUMP NAMES ${TOOLCHAIN_PREFIX}-objdump${TOOLCHAIN_EXT} PATHS ${TOOLCHAIN_DIR} NO_DEFAULT_PATH)
find_program(ARM_SIZE NAMES ${TOOLCHAIN_PREFIX}-size${TOOLCHAIN_EXT} PATHS ${TOOLCHAIN_DIR} NO_DEFAULT_PATH)
message(STATUS "Finding OpenOCD executable")
# Find the host's openocd program
find_program(
  OPENOCD_EXECUTABLE
  NAMES openocd
  PATHS "/usr/local/bin"
        "/usr/bin"
        "$ENV{PROGRAMFILES}/OpenOCD"
  DOC "Path to the openocd program"
)

# Check if the program was found
if (NOT OPENOCD_EXECUTABLE)
  message(FATAL_ERROR "Could not find openocd program")
endif()

message(STATUS "Adding custom targets")
add_custom_target(size ALL ${ARM_SIZE} -B -d --target=binary ${EXECUTABLE}.elf DEPENDS ${EXECUTABLE})
add_custom_target(hex ALL ${ARM_OBJCPY} -O ihex ${EXECUTABLE}.elf ${EXECUTABLE}.hex DEPENDS ${EXECUTABLE})
add_custom_target(bin ALL ${ARM_OBJCPY} -O binary -S ${EXECUTABLE}.elf ${EXECUTABLE}.bin DEPENDS ${EXECUTABLE})
add_custom_target(objdump ${ARM_OBJDUMP} -S ${EXECUTABLE}.elf DEPENDS ${EXECUTABLE})
add_custom_target(flash DEPENDS size COMMAND ${OPENOCD_EXECUTABLE} -finterface/stlink.cfg -ftarget/stm32f4x.cfg -c "program ${EXECUTABLE}.elf verify reset exit")