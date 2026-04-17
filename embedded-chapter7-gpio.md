---
title: "STM32F446 + CMake + CLion Integration"
date: "2026-04-17"
updated: "2026-04-17"
categories:
  - "embedded"
  - "stm32"
  - "c"
excerpt: "Setting up one-click build and flash for STM32F446 bare-metal development with CMake and CLion on Arch Linux."
coverImage: "/images/posts/embedded-chapter7-gpio-cover.svg"
coverWidth: 1200
coverHeight: 630
---

## Intro

I wanted to take a detour and get CLion properly configured for STM32 development. The goal was simple: one-click builds and flashing. What followed was a frustrating exercise in CMake configuration that took way longer than expected.

## Phase 1: Initial CMake Setup

### Toolchain File (`arm-none-eabi-gcc.cmake`)
Created cross-compilation toolchain file:

```cmake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_SIZE arm-none-eabi-size)

# STM32F446 specific flags (Cortex-M4 + FPU)
set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")

set(CMAKE_C_FLAGS_INIT "${MCU_FLAGS} -ffunction-sections -fdata-sections")
set(CMAKE_CXX_FLAGS_INIT "${MCU_FLAGS} -ffunction-sections -fdata-sections")
set(CMAKE_ASM_FLAGS_INIT "${MCU_FLAGS}")

# CRITICAL ISSUE: Initial linker flags included --gc-sections
set(CMAKE_EXE_LINKER_FLAGS_INIT "${MCU_FLAGS} -Wl,--gc-sections")

set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)
```

### Main CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.20)
project(stm32f446_baremetal C ASM)

set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_compile_definitions(STM32F446xx)

include_directories(${CMAKE_SOURCE_DIR}/src)

set(SOURCES
    src/main.c
    src/stm32f446_startup.c
)

add_executable(${PROJECT_NAME}.elf ${SOURCES})

set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/stm32_ls.ld)
target_link_options(${PROJECT_NAME}.elf PRIVATE 
    -T${LINKER_SCRIPT}
    -Wl,-Map=${PROJECT_NAME}.map
    -nostdlib
)

add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary ${PROJECT_NAME}.elf ${PROJECT_NAME}.bin
    COMMAND ${CMAKE_SIZE} ${PROJECT_NAME}.elf
)
```

## Phase 2: OpenOCD Configuration Issues (Arch Linux)

### Problem: CLion Cannot Find Board Configs
**Error:** `Wrong OpenOCD installation. Board configs folder not found.`

**Root Cause:** Arch Linux installs OpenOCD scripts to `/usr/share/openocd/scripts/`, but CLion expects them relative to the binary (`/bin/scripts`).

### Solution:
I tried symlinking the right location but I couldn't get that to work for some reason. I just ended up copying the scripts to where it expects to find them at `/bin/scripts`.

## Phase 3: The Vector Table Issues

### Symptom: Build Succeeds (224 bytes), No Blinky
Binary was suspiciously small. Disassembly showed `main()` at `0x08000000` instead of vector table.

### Diagnostic Commands
```bash
# Check if vector table section exists
arm-none-eabi-objdump -s -j .isr_vector cmake-build-stm32debug/stm32f446_baremetal.elf
# Result: Section not found

# Check symbols
arm-none-eabi-nm cmake-build-stm32debug/stm32f446_baremetal.elf | head -20
# Shows: 08000000 T main (wrong!)
```

### Issue 1: Garbage Collection Removing Vector Table
With `--gc-sections`, the linker removed `vector_table[]` because nothing references it.

**Attempted Fix:** Added `KEEP(*(.isr_vector))` to linker script:
```ld
SECTIONS{
    .text :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))  /* Prevent GC */
        *(.text)
        *(.rodata)
        . = ALIGN(4);
        _etext = .;
    }>FLASH
    /* ... */
}
```
Tried this but couldn't get it to work right.

### Actual Working Fix
**Replaced** `--gc-sections` with `-Map` generation in toolchain file:
```cmake
# Before (broken):
set(CMAKE_EXE_LINKER_FLAGS_INIT "${MCU_FLAGS} -Wl,--gc-sections")

# After (working):
set(CMAKE_EXE_LINKER_FLAGS_INIT "${MCU_FLAGS} -Wl,-Map=5_makefile_project.map")
```

**Note:** `--gc-sections` was the fix.

### Verification
```bash
arm-none-eabi-objdump -s -j .text --start-address=0x08000000 --stop-address=0x08000020 cmake-build-stm32debug/stm32f446_baremetal.elf

# Output:
# 8000000 00000220 79000008 71000008 71000008
# Decodes to:
# 0x20020000 (stack pointer)
# 0x08000079 (Reset_Handler with Thumb bit)
```

## Working Configuration

### Final Toolchain File
```cmake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_SIZE arm-none-eabi-size)

set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")

set(CMAKE_C_FLAGS_INIT "${MCU_FLAGS} -ffunction-sections -fdata-sections")
set(CMAKE_CXX_FLAGS_INIT "${MCU_FLAGS} -ffunction-sections -fdata-sections")
set(CMAKE_ASM_FLAGS_INIT "${MCU_FLAGS}")

# No --gc-sections to prevent vector table removal
set(CMAKE_EXE_LINKER_FLAGS_INIT "${MCU_FLAGS} -Wl,-Map=5_makefile_project.map")

set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)
```

### Startup File (`stm32f446_startup.c`)
```c
#include <stdint.h>

extern uint32_t _estack;
// ... other externs

void Reset_Handler(void);
void Default_Handler(void);

const uint32_t vector_table[] __attribute__((section(".isr_vector"))) = {
  (uint32_t)&_estack,
  (uint32_t)&Reset_Handler,
  // ... other handlers
};

void Reset_Handler(void) {
  // Copy .data from FLASH to SRAM
  // Zero .bss
  main();
  while(1);
}
```

This was an exercise in suffering but I got CMake working and CLion properly one-click build, flash and run code on my STM32F446RE.

## Conclusion

I need to learn more about CMake. This whole experience showed me that while I can hack things together until they work, I don't really understand why certain things behave the way they do.