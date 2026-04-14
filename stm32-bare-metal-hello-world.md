---
title: "STM32 bare metal hello world"
date: "2026-04-09"
updated: "2026-04-09"
categories:
  - "embedded"
  - "c"
  - "stm32"
  - "arm"
excerpt: "First embedded foray: configuring a Nucleo F446 by reading the datasheet directly and turning on the LED 2 light, following Israel Gbati's Bare-Metal Embedded C Programming."
coverImage: "/images/posts/nucleo-f446-led-on.jpeg"
coverWidth: 1000
coverHeight: 1160
---

I've recently been working through Israel Gbati's *[Bare-Metal Embedded C Programming](https://www.packtpub.com/en-us/product/bare-metal-embedded-c-programming-9781835467688)* (Packt) as my first foray into embedded systems. Chapters 1 and 2 cover getting the environment set up and understanding the hardware. By the end of Chapter 2 I had the LED on my Nucleo-F446RE actually on.

<img
	src="/images/posts/bare-metal-embedded-c-cover.jpg"
	alt="Cover of Bare-Metal Embedded C Programming by Israel Gbati, published by Packt"
	width="200"
/>

The code is unidiomatic (no HAL, no abstractions, just raw register addresses from the reference manual) but that's the point.

```c
#define GPIOA_BASE          0x40020000
#define RCC_BASE            0x40023800
#define RCC_AHB1ENR_OFFSET  0x30
#define GPIOAEN             0x0   /* bit 0: enables GPIOA clock */
#define GPIOx_ODR_OFFSET    0x14  /* output data register */

#define GPIOA_MODER  (* (volatile unsigned long *) (GPIOA_BASE + 0x00))
#define RCC_AHB1ENR  (* (volatile unsigned long *) (RCC_BASE + RCC_AHB1ENR_OFFSET))
#define GPIOx_ODR    (* (volatile unsigned long *) (GPIOA_BASE + GPIOx_ODR_OFFSET))

int main(void)
{
    RCC_AHB1ENR |= (1 << GPIOAEN);   /* enable GPIOA clock */

    GPIOA_MODER &= ~(0x3 << 10);     /* clear MODER5 */
    GPIOA_MODER |=  (0x1 << 10);     /* set MODER5 to general-purpose output */

    GPIOx_ODR |= (1 << 5);           /* set PA5 high, LED on */

    for (;;) {}
}
```

Three things are happening here:

1. **Enable the GPIOA clock.** Peripherals on STM32 are clock-gated by default to save power. Writing to `RCC_AHB1ENR` turns the clock on before any GPIO register access, otherwise the writes just disappear.

2. **Configure PA5 as output.** `MODER` is a 2-bit-per-pin field. Bits 10-11 control pin 5. Clearing them first and then writing `01` sets the pin to general-purpose push-pull output.

3. **Drive the pin high.** Writing bit 5 of the output data register sources current through the LED to ground.

That's it. The [follow-up post](/blog/stm32-bare-metal-blinky-gnu-toolchain) adds a toggle loop and walks through the GNU ARM toolchain manually.
