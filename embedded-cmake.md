---
title: "STM32F446 + CMake + CLion Integration"
date: "2026-04-16"
updated: "2026-04-17"
categories:
  - "embedded"
  - "stm32"
  - "c"
excerpt: "Setting up one-click build and flash for STM32F446 bare-metal development with CMake and CLion on Arch Linux."
coverImage: "/images/posts/embedded-cmake-cover.svg"
coverWidth: 1200
coverHeight: 630
---

## Intro

After [automating my build with a Makefile](/blog/chapter5gbati), I wanted to take a detour and get CLion properly configured for STM32 development. The goal was simple: one-click builds and flashing. What followed was a frustrating exercise in CMake configuration that took way longer than expected.

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
Binary was suspiciously small. Disassembly showed `main()` at `0x08000000` instead of vector table. This was the same class of bug I'd fought before in [the linker script post](/blog/stm32-linker-script-and-startup-code), where a section name mismatch sent the vector table to the wrong address.

### Diagnostic Commands
```bash
# Check if vector table section exists
arm-none-eabi-objdump -s -j .isr_vector cmake-build-stm32debug/stm32f446_baremetal.elf
# Result: Section not found

# Check symbols
arm-none-eabi-nm cmake-build-stm32debug/stm32f446_baremetal.elf | head -20
# Shows: 08000000 T main (wrong!)
```

### Issue: Garbage Collection Removing Vector Table
With `--gc-sections`, the linker removed `vector_table[]` because nothing references it. The proper fix is adding `KEEP(*(.isr_vector))` to the linker script, which tells the linker to preserve that section even if nothing references it:
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

I had run into this exact class of bug before in [the linker script post](/blog/stm32-linker-script-and-startup-code), where a section name mismatch between the C attribute and the linker script caused the vector table to land in the wrong place.

### Workaround: Remove --gc-sections
I punted on the root cause and just removed `--gc-sections` from the linker flags:
```cmake
# Before (vector table gets garbage collected):
set(CMAKE_EXE_LINKER_FLAGS_INIT "${MCU_FLAGS} -Wl,--gc-sections")

# After (works, but no dead code elimination):
set(CMAKE_EXE_LINKER_FLAGS_INIT "${MCU_FLAGS} -Wl,-Map=5_makefile_project.map")
```

This works but it's not ideal. Without `--gc-sections`, unused functions and variables stay in the binary, wasting flash. For a blinky program it doesn't matter, but for real projects it might. I need to come back and figure out the `KEEP()` directive properly. 

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

This was an exercise in suffering but I got CMake working and CLion properly one-click building, flashing and running code on my STM32f446RE. The payoff came immediately in [the GPIO driver post](/blog/chap7-gpio), where CLion's peripherals view let me spot a typo by watching register values change in real time.

## Conclusion

I got what I came for: one-click build and flash in CLion for STM32F446 bare-metal C. Cross-compilation toolchain file, linker script integration, OpenOCD config on Arch, and a working debug workflow.

## The Payoff

The first time I hit a breakpoint on `GPIOA->ODR ^= LED_PIN` and pressed Continue, the LED toggled. Pressed it again, toggled back. I was stepping through code running on the bare metal from inside a proper IDE. Just click, step, watch the hardware respond. That's when the whole painful detour felt worth it and justified my decision to settle on CLion as my IDE of choice.