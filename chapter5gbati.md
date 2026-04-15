---
title: "Building a Custom Makefile for STM32 Bare-Metal Development"
date: "2026-04-15"
updated: "2026-04-15"
categories:
  - "embedded"
  - "c"
  - "stm32"
  - "makefile"
excerpt: "I wrote a custom Makefile for my STM32F446 bare-metal C project, learning about target-prerequisite relationships and Make's reserved keywords along the way."
coverImage: "/images/posts/chapter5gbati-cover.svg"
coverWidth: 1200
coverHeight: 630
---

I'm working through Chapter 5 of Gbati's *Bare-Metal Embedded C Programming*, which focuses on the make build system. Having previously written my own linker script and initialized the C runtime on STM32 using weak and alias attributes, I was ready to automate the build process that I'd been running manually with gcc, openOCD, and GDB commands.

## Understanding Makefile Structure

A Makefile consists of rules that define how a project compiles. Each rule follows this pattern:

```makefile
# target : prerequisite
#     recipe (must begin with tab)
# Example:
main.o : main.c
    arm-none-eabi-gcc main.c -o main.o
```

The target is the file you want generated, the prerequisite is the input required, and the recipe contains the shell commands to generate the target. Makefiles support variables and built-in macros like `$@` (target name) and `$^` (prerequisite name):

```makefile
CC = arm-none-eabi-gcc

main.o : main.c
	$(CC) $^ -o $@
```

## Writing My Custom Makefile

Rather than use Gbati's provided version, I decided to write my own. My setup differs slightly: I'm using an STM32F446 instead of the F411 Nucleo, and I wanted to try C23 instead of C11. I know C11 is more standard but GNU's arm-none compiler supports C23 now which has a few features I would like to be able to use in the future like underscores in numbers and being able to use binary literals with the 0b notation which I could see making embedded development a lot more convenient. I intend to write the barbell velocity tracker project I detailed in another post with C23 so it's good to just get used to using that as I work through the book.

My first attempt failed with this cryptic error:
```
Makefile:17: :: cannot open shared object file: No such file or directory
Makefile:17: *** :: failed to load.  Stop.
```

The problematic Makefile:
```makefile
CC = arm-none-eabi-gcc
mcpu = -mcpu=cortex-m4
thumb = -mthumb
std = -std=gnu23

final : 5_makefile_project.elf

main.o : main.c
	$(CC) -c $(mcpu) $(thumb) $(std) $^ -o $@

stm32f466_startup.o : stm32f446_startup.c
	$(CC) -c $(mcpu) $(thumb) $(std) $^ -o $@

5_makefile_project.elf : main.o stm32f466_startup.o
	$(CC) $(mcpu) $(thumb) -nostdlib -T stm32_ls.ld *.o -o 5_makefile_project.elf -W1, -Map=5_makefile_project.map

load :
	openocd -f board/st_nucleo_f4.cfg
clean:
	del -f *.o *.elf *.map
```

After some research, I discovered that Make 4.0+ reserves the term `load` for special use. I fixed this along with a few other issues in my second attempt:

```makefile
CC = arm-none-eabi-gcc
mcpu = -mcpu=cortex-m4
thumb = -mthumb
std = -std=gnu23

final : 5_makefile_project.elf

main.o : main.c
	$(CC) -c $(mcpu) $(thumb) $(std) $^ -o $@

stm32f446_startup.o : stm32f446_startup.c
	$(CC) -c $(mcpu) $(thumb) $(std) $^ -o $@

5_makefile_project.elf : main.o stm32f446_startup.o
	$(CC) $(mcpu) $(thumb) -nostdlib -T stm32_ls.ld *.o -o 5_makefile_project.elf -Wl,-Map=5_makefile_project.map

flash :
	openocd -f board/st_nucleo_f4.cfg
clean:
	rm -f *.o *.elf *.map
```

This version worked correctly. The key fixes were:
- Renaming `load` to `flash` to avoid the reserved keyword
- Correcting the linker flag from `-W1,` to `-Wl,`
- Changing `del` to `rm` for Unix compatibility Gbati's book used del as he's using windows

## Testing the Flash Command

The `make flash` command successfully starts openOCD, streamlining the flashing process. I still need to manually connect with GDB and run the flash commands, but this is already much more convenient. Here's the two-terminal workflow:

**Terminal 1** - Start openOCD:
```shell
❯ make flash
openocd -f board/st_nucleo_f4.cfg
# ... openOCD output showing successful connection and GDB server on port 3333
```

**Terminal 2** - Connect with GDB and flash:
```shell
arm-none-eabi-gdb
(gdb) target remote localhost:3333
(gdb) monitor reset init
(gdb) monitor flash write_image erase 5_makefile_project.elf
(gdb) monitor reset init
(gdb) monitor resume
```

The flashing process works as expected, writing 16384 bytes to the STM32F446. For future iterations, I'd like to explore automating the GDB commands as well, since we're just connecting to localhost:3333 served by openOCD. There should be a way to script these commands directly into the Makefile.

This exercise reinforced the power of build systems and was a great introduction to the C flavor of build systems. What used to be a series of manual compiler and flashing commands is now reduced to `make final` and `make flash`. Although the book doesn't cover it, I'll probably dive into CMake eventually because my preferred IDE right now is CLion with IdeaVIM, and CLion really leverages CMake beautifully.