---
---
title: "Bare-Metal Embedded C Programming: Chapter 8 - SysTick Timer"
date: "2026-04-18"
updated: "2026-04-18"
categories:
  - "embedded"
  - "c"
  - "stm32"
excerpt: "Implementing a SysTick driver for millisecond delays on the STM32F446RE, then using it to debounce a button press that toggles an LED."
coverImage: "/images/posts/chapter-8-gbati-cover.svg"
coverWidth: 1200
coverHeight: 630
---

Chapter 8 of Gbati's book covers the SysTick Timer. This chapter was fairly straightforward.

## Part 1: Implementing a SysTick Driver

First we created the `SysTick.c` and `SysTick.h` files with the following code.

**systick.c:**

```c
#include "systick.h"

#define CTRL_ENABLE     (1U<<0)
#define CTRL_CLCKSRC    (1U<<2)
#define CTRL_COUNTFLAG  (1U<<16)

// By default, MCU frequency is 16MHz on reset
#define ONE_MSEC_LOAD   16000

// STM32F446RE max speed
#define F446_ONE_MSEC_LOAD  180000

void systick_msec_delay(const uint32_t delay) {
    // Load the number of clock cycles per millisecond
    SysTick->LOAD = ONE_MSEC_LOAD - 1;

    // Clear systick current value register
    SysTick->VAL = 0;

    // Select internal clock source
    SysTick->CTRL = CTRL_CLCKSRC;

    // Enable Systick
    SysTick->CTRL |= CTRL_ENABLE;

    for (volatile int i = 0; i < delay; i++) {
        while ((SysTick->CTRL & CTRL_COUNTFLAG) == 0) {}
    }

    // Disable SysTick
    SysTick->CTRL = 0;
}
```

**systick.h:**

```c
#ifndef STM32F446_BAREMETAL_SYSTICK_H
#define STM32F446_BAREMETAL_SYSTICK_H

#include <stdint.h>
#include "stm32f4xx.h"

void systick_msec_delay(uint32_t delay);

#endif // STM32F446_BAREMETAL_SYSTICK_H
```

## Part 2: Using the SysTick Delay Function

Gbati uses the SysTick delay for the blinky code I've been working with thus far. I decided to do something a little different and use it with the button press code instead. First I added a toggle LED function to gpio.c and gpio.h.

```c
void led_toggle(void) {
    // Toggle PA5
    GPIOA->ODR ^= LED_PIN;
}
```

Then I used the toggle plus delay in main.c:

```c
int main(void)
 {
    // Initialize LED
    led_init();

    // Initialize Button
    button_init();

    /* Loop forever */
    while(1)
    {
        // Get Push Button State
        push_button_state = get_btn_state();

        if (push_button_state) {
            led_on();
            systick_msec_delay(1000);
            led_off();
        }
    }
}
```

That's it for the SysTick timer for now. In the next chapter we will be working with General Purpose Timers.