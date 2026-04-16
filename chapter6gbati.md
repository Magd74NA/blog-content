---
title: "Understanding ARM CMSIS: Memory-Mapped Hardware"
date: "2026-04-16"
updated: "2026-04-16"
categories:
  - "embedded"
  - "stm32"
  - "cmsis"
excerpt: "How I learned to work with ARM CMSIS standard by rewriting a blinky program and understanding memory-mapped peripheral structures."
coverImage: "/images/posts/chapter5gbati-2-cover.svg"
coverWidth: 1200
coverHeight: 630
---

# Understanding ARM CMSIS: Memory-Mapped Hardware

I'm working through chapter 6 of Gbati's *Bare-Metal Embedded C Programming*, which covers the ARM CMSIS standard. [The previous chapter](/blog/chapter5gbati) taught me to automate the build with a Makefile. This chapter taught me the industry standard approach to hardware definitions in embedded C.

## Rewriting Hardware Definitions

I started by rewriting [the original blinky program](/blog/stm32-bare-metal-blinky-gnu-toolchain) to follow the CMSIS structure pattern. Instead of working with [the raw register macros from the hello world post](/blog/stm32-bare-metal-hello-world), I defined hardware using structured types that mirror the actual memory layout:

```c
typedef struct {
    volatile uint32_t MODER;    //offset: 0x00
    volatile uint32_t OTYPER;   //offset: 0x04
    volatile uint32_t OSPEEDR;  //offset: 0x08
    volatile uint32_t PUPDR;    //offset: 0x0C
    volatile uint32_t IDR;      //offset: 0x10
    volatile uint32_t ODR;      //offset: 0x14
    volatile uint32_t BSRR;     //offset: 0x18
    volatile uint32_t LCKR;     //offset: 0x1C
    volatile uint32_t AFRL;     //offset: 0x20
    volatile uint32_t AFRH;     //offset: 0x24
} GPIO_TypeDef;

typedef struct {
    volatile uint32_t DUMMY[12];
    volatile uint32_t AHB1ENR;  //offset: 0x30
} RCC_TypeDef;

#define RCC_BASE    0x40023800
#define GPIOA_BASE  0x40020000

#define RCC         ((RCC_TypeDef*)   RCC_BASE)
#define GPIOA       ((GPIO_TypeDef*)GPIOA_BASE)
```

## The Moment of Confusion

When I first flashed this code to my STM32F446RE, I was genuinely confused. We're defining structs and manipulating their members without ever initializing any values. How did I access `GPIOA->MODER` when we never created a `GPIO_TypeDef` instance? Where's the initialization?

## The Breakthrough: Memory-Mapped Hardware

The breakthrough came when I connected this to what I'd already learned about the [linker script and memory layout](/blog/stm32-linker-script-and-startup-code). I stepped back and thought about what the code was actually doing. We're not creating struct instances. Instead, we're casting memory addresses to struct pointers, overlaying our struct layout onto hardware registers.

The key insight: **memory-mapped hardware exists in contiguous blocks**. When I write:

```c
#define GPIOA ((GPIO_TypeDef*)GPIOA_BASE)
```

We're telling the compiler that starting at address `GPIOA_BASE`, the memory layout follows our `GPIO_TypeDef` structure. Each `uint32_t` member occupies the next 4 bytes in sequence. The hardware designers built the GPIO peripheral this way, and our struct simply mirrors that layout.

## Handling Non-Sequential Registers

The `RCC_TypeDef` structure demonstrates a clever technique for handling cases where we don't need all registers:

```c
typedef struct {
    volatile uint32_t DUMMY[12];  // Skip first 48 bytes
    volatile uint32_t AHB1ENR;    // The register we actually need
} RCC_TypeDef;
```

The `DUMMY[12]` array skips the first 48 bytes (12 × 4 bytes) because we don't care about those registers for this simple blinky program. We jump straight to `AHB1ENR` at offset 0x30. This technique lets us define only the registers we need while maintaining the correct memory layout.

## Clean Hardware Manipulation

With these definitions in place, the actual hardware manipulation becomes clean and readable:

```c
int main(void)
{
    // Enable clock access to GPIOA
    RCC->AHB1ENR |= GPIOAEN;

    // Set PA5 to general output mode
    GPIOA->MODER |=  (1U<<10);
    GPIOA->MODER &= ~(1U<<11);

    while(1){
        // Toggle PA5 (LED)
        GPIOA->ODR ^= LED_PIN;
        
        // Simple delay
        volatile int i;
        for (i = 0; i < 400000; i++) {
            __asm__ volatile ("nop");
        }
    }
}
```

## Moving to Official CMSIS Headers

The second half of the chapter introduces the official CMSIS headers from STMicrolectronics. Instead of manually defining structs based on reference manual documentation, I can now use the official header files that follow this exact same pattern for all peripherals.

The CMSIS standard uses this technique across all ARM Cortex-M microcontrollers, providing a standardized way to interact with memory-mapped hardware that's much more readable than the method I had been using previously.