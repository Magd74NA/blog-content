---
title: "STM32 bare metal: writing a linker script and startup code from scratch"
date: "2026-04-12"
updated: "2026-04-12"
categories:
  - "embedded"
  - "c"
  - "stm32"
  - "arm"
excerpt: "Chapter 4 of Gbati's Bare-Metal-C: replacing the auto-generated startup assembly and linker script with hand-written C equivalents, then debugging through five attempts before the LED blinked."
coverImage: "/images/posts/nucleo-f446-led-on.jpeg"
coverWidth: 1000
coverHeight: 1160
---

Chapters 2 and 3 (covered in [the hello world post](/blog/stm32-bare-metal-hello-world) and [the blinky toolchain post](/blog/stm32-bare-metal-blinky-gnu-toolchain)) relied on STM32CubeIDE's auto-generated startup assembly and linker script. Chapter 4 of Israel Gbati's *Bare-Metal Embedded C Programming* rips all of that away: you write the linker script yourself, write the startup code in C instead of assembly, and piece together enough of the C runtime for `main()` to actually run. ([Chapter 5](/blog/chapter5gbati) builds on this by automating the build with a Makefile, and [Chapter 6](/blog/chapter6gbati) rewrites the hardware access using the CMSIS standard.)

## The Memory Map

Before writing any code, I confirmed the STM32F446RE's memory layout from the reference manual:

<img
	src="/images/posts/stm32f446-memory-map-reference.png"
	alt="STM32F446RE memory map from the reference manual showing FLASH at 0x08000000 and SRAM at 0x20000000"
	width="500"
/>

Two regions matter:

- **FLASH**: starts at `0x08000000`, 512 KiB. Where the program lives.
- **SRAM**: starts at `0x20000000`, 128 KiB. Volatile working memory.

## The Linker Script

The linker script tells the linker three things: what the entry point is, what memory exists, and which sections go where.

```ld
/* Specifying the firmware's entry point */
ENTRY(Reset_Handler)

/* Detailing available memory */
MEMORY
{
    FLASH(rx):ORIGIN =0x08000000,LENGTH =512K
    SRAM(rwx):ORIGIN =0x20000000,LENGTH =128K
}

_estack = ORIGIN(SRAM)+LENGTH(SRAM);

/* Specifying the necessary heap and stack sizes */
__max_heap_size = 0x200;
__max_stack_size = 0x400;

/* Defining output sections */
SECTIONS{
    .text :
    {
    . = ALIGN(4);
     *(.isr_vector_table) /* Merge all .isr_vector_table sections */
     *(.text)             /* Merge all .text sections */
     *(.rodata)           /* Merge all .rodata sections */
     . = ALIGN(4);
     _etext = .;          /* Global symbol: end of text section */
    }>FLASH

    .data :
    {
    . = ALIGN(4);
    _sdata = .;           /* Global symbol: start of data */
     *(.data)
     . = ALIGN(4);
    _edata = .;           /* Global symbol: end of data */
    } > SRAM AT > FLASH   /* VMA in SRAM, LMA in FLASH */

    .bss :
    {
    . = ALIGN(4);
    _sbss = .;
    *(.bss)
     . = ALIGN(4);
    _ebss = .;
    }> SRAM
}
```

The key parts:

1. **`ENTRY(Reset_Handler)`**: tells the linker which symbol is the program's entry point. The debugger uses this, and it also determines which section gets placed first.

2. **`MEMORY` block**: defines the two physical memory regions with their start addresses and sizes. `rx` means flash is readable and executable; `rwx` means SRAM is fully accessible.

3. **`_estack`**: initialized to the top of SRAM (`0x20000000 + 128K = 0x20020000`). The Cortex-M4 loads the stack pointer from the first word of the vector table on reset, so this symbol needs to be there.

4. **`.text` section → FLASH**: code, read-only data, and critically the `.isr_vector_table` all go here. The vector table **must** be at the very start of flash (`0x08000000`) because that's where the processor looks for it on boot.

5. **`.data` section → SRAM `AT > FLASH`**: this is the subtle one. `.data` contains initialized global/static variables. Their *runtime* address (VMA) is in SRAM, but they can't live there at power-on (SRAM is volatile). The `AT > FLASH` tells the linker to store the initial values at the end of flash (LMA), and the startup code copies them to SRAM before `main()` runs.

6. **`.bss` section → SRAM**: uninitialized variables. No flash copy needed; the startup code just zeroes this region.

7. **Linker symbols** (`_etext`, `_sdata`, `_edata`, `_sbss`, `_ebss`): these are the boundary markers the startup code uses to know how much `.data` to copy and how much `.bss` to zero.

## The Startup Code

Instead of the traditional assembly startup file, this chapter writes it in C:

```c
#include <stdint.h>
extern uint32_t _estack;
extern uint32_t _etext;
extern uint32_t _sdata;
extern uint32_t _edata;
extern uint32_t _sbss;
extern uint32_t _ebss;

void Reset_Handler(void);
int main(void);
void NMI_Handler(void)__attribute__((weak, alias("Default_Handler")));
void HardFault_Handler(void)__attribute__((weak, alias("Default_Handler")));
void memManage_Handler(void)__attribute__((weak, alias("Default_Handler")));


uint32_t vector_table[] __attribute__((section(".isr_vector_table"))) = {
  (uint32_t)&_estack,
  (uint32_t)&Reset_Handler,
  (uint32_t)&NMI_Handler,
  (uint32_t)&HardFault_Handler,
  (uint32_t)&memManage_Handler,
};

void Default_Handler(void) {
  while (1) {

  }
}

void Reset_Handler(void) {
  // Divide by 4 (sizeof(uint32_t))
  uint32_t data_mem_size = ((uint32_t)&_edata - (uint32_t)&_sdata) / 4;
  uint32_t bss_mem_size = ((uint32_t)&_ebss - (uint32_t)&_sbss) / 4;

  uint32_t *p_src_mem = (uint32_t *)(&_etext);
  uint32_t *p_dest_mem = (uint32_t *)(&_sdata);

  for (uint32_t i = 0; i < data_mem_size; i++) {
    *p_dest_mem++ = *p_src_mem++;
  }

  p_dest_mem = (uint32_t *)(&_sbss);
  for (uint32_t i = 0; i < bss_mem_size; i++) {
    *p_dest_mem++ = 0;
  }

  main();

  while(1);
}
```

Three things are happening:

1. **Vector table**: placed in the `.isr_vector_table` section so the linker script can position it at the start of flash. The first entry is the initial stack pointer value (`_estack`), the second is `Reset_Handler`, and the rest are exception handlers. On reset, the Cortex-M4 reads the first two words from address `0x08000000`: SP and PC.

2. **`.data` copy**: `Reset_Handler` copies `data_mem_size` words from the end of flash (`_etext`) to the start of SRAM (`_sdata`). This is what bridges the gap between the linker's "store initial values in flash" and the C runtime's "initialized globals live in RAM."

3. **`.bss` zeroing**: fills the `.bss` region with zeros. C guarantees that uninitialized static/global variables start as zero, and this is where that guarantee is fulfilled.

The `__attribute__((weak, alias("Default_Handler")))` pattern means that if you don't provide your own `HardFault_Handler` (etc.), it defaults to an infinite loop. Which is exactly what you want for a bare-metal fault: halt rather than run off into undefined behaviour.

## Building and Flashing

Compile both files with `-ffreestanding` (no standard library assumptions), link with `-nostdlib` and the custom linker script:

```sh
arm-none-eabi-gcc -c -mcpu=cortex-m4 -mthumb -std=gnu23 main.c -o main.o -ffreestanding
arm-none-eabi-gcc -c -mcpu=cortex-m4 -mthumb -std=gnu23 stm32f446_startup.c -o stm32f446_startup.o -ffreestanding
arm-none-eabi-gcc -nostdlib -mcpu=cortex-m4 -mthumb -T stm32_ls.ld *.o -o 3_LinkerAndStartup.elf
```

Then inspect the ELF to verify sections landed at the right addresses:

```sh
arm-none-eabi-objdump -h 3_LinkerAndStartup.elf | head -20

# 3_LinkerAndStartup.elf:     file format elf32-littlearm
#
# Sections:
# Idx Name          Size      VMA       LMA       File off  Algn
#   0 .text         000000fc  08000000  08000000  00001000  2**2
#                   CONTENTS, ALLOC, LOAD, READONLY, CODE
#   1 .data         00000000  20000000  080000fc  00002000  2**0
#                   CONTENTS, ALLOC, LOAD, DATA
#   2 .bss          00000000  20000000  080000fc  00002000  2**0
#                   ALLOC
```

`.text` at `0x08000000` with the vector table first, `.data` and `.bss` pointing at `0x20000000`. Flash via OpenOCD + GDB the same way as [the previous post](/blog/stm32-bare-metal-blinky-gnu-toolchain):

```sh
# Terminal 1
openocd -f board/st_nucleo_f4.cfg

# Terminal 2
arm-none-eabi-gdb
(gdb) target remote localhost:3333
(gdb) monitor reset init
(gdb) monitor flash write_image erase 3_LinkerAndStartup.elf
(gdb) monitor reset init
(gdb) monitor resume
```

## Five Attempts

This did not work the first time. Or the second. Or the third, or the fourth.

### Attempt 1: Typo in startup code

The startup code had `_ebsss` instead of `_ebss`. The compiler helpfully suggested the fix:

```
stm32f446_startup.c:33:38: error: '_ebss' undeclared (first use in this function);
   did you mean '_ebsss'?
```

Fixed the spelling, recompiled, linked, flashed. The LED didn't blink. The processor was stuck in a `HardFault` (`xPSR: 0x01000003` in the OpenOCD output).

### Attempts 2-4: Vector table in the wrong place

After fixing the typo, the `objdump` output revealed the real problem. The `.isr_vector_table` section was showing up as a *separate* output section at `0x20000000` (SRAM), not inside `.text` at `0x08000000`:

```
Idx Name               Size      VMA       LMA
  0 .text              000000e8  08000000  08000000
  1 .data              00000000  20000000  080000e8
  2 .isr_vector_table  00000014  20000000  080000e8   <-- WRONG
  3 .bss               00000000  20000014  080000fc
```

On reset, the Cortex-M4 reads the first words at `0x08000000`, but that was just `.text` code, not the vector table. The processor loaded garbage into SP and PC and immediately faulted.

The root cause was a misspelled section name in the linker script. The startup code places the table in `.isr_vector_table`, and the linker script had a slightly different name. Since no input section matched, the linker created a new output section and placed it according to a default rule, which happened to target SRAM.

### Attempt 5: Success

Once the section names matched exactly, `objdump` showed the clean layout:

```
Idx Name          Size      VMA       LMA
  0 .text         000000fc  08000000  08000000
  1 .data         00000000  20000000  080000fc
  2 .bss          00000000  20000000  080000fc
```

No orphan `.isr_vector_table` section. It was correctly merged into `.text`. Flashed, resumed, and the LED blinked.

## What Stuck

- **The linker script is a contract.** The section names in your startup code must match the section names in the linker script *exactly*. A single character mismatch silently creates an orphan section that goes somewhere wrong.
- **`objdump -h` is your best friend.** Checking section addresses before flashing saves a lot of head-scratching. If the vector table isn't at `0x08000000`, nothing else matters.
- **VMA vs LMA.** The `> SRAM AT > FLASH` syntax is the linker's way of saying "this lives in RAM at runtime, but store the initial values in flash." The startup code bridges that gap.
- **Startup code in C works.** The traditional approach is assembly, but for Cortex-M everything the startup needs to do (copy `.data`, zero `.bss`, call `main`) is straightforward in C. The compiler attributes (`section`, `weak`, `alias`) handle the parts that would normally require assembly directives.
