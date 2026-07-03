---
title: "Chapter 11 of Bare-Metal Embedded C Programming"
date: "2026-07-03"
updated: "2026-07-03"
categories:
  - "c"
  - "embedded"
  - "stm32"
  - "projects"
excerpt: "First ADC driver on the STM32F446RE — configuring the analog peripheral, getting printf working again, and pointing it at an ambient light sensor."
coverImage: "/images/posts/gbati-chapter-11-cover.svg"
coverWidth: 1200
coverHeight: 630
---

## Part 1: From Digital to Analog

Chapter 10 left the MCU talking to a Raspberry Pi over UART, which felt like a real milestone the chip was finally shouting at another computer. But every signal so far in this project has been digital: a pin is high or low, a byte goes out the wire, a timer ticks. Chapter 11 is the first time the STM32 listens to the world through an Analog-to-Digital Converter.

The goal of the chapter is modest: configure ADC1 to read a voltage on **PA1** (ADC1_IN1), keep converting continuously, and dump the 12-bit result out over UART. I pointed the driver at an **ambient light sensor**.

Here's the driver in full:

`adc.c`

```c
#include "adc.h"

#define GPIOAEN     (1U<<0)
#define ADC1EN      (1U<<8)
#define ADC_CH1     (1U<<0)
#define ADC_SEQ_LEN_1 0x00
#define CR2_ADCON   (1U<<0)
#define CR2_CONT    (1U<<1)
#define CR2_SWSTART (1U<<30)
#define SR_EOC      (1U<<1)

void pa1_adc_init(void)
{
    /****Configure the ADC GPIO Pin****/
    /*Enable clock access to GPIOA*/
    RCC->AHB1ENR |= GPIOAEN;

    /*Set PA1 mode to analog mode*/
    GPIOA->MODER |= (1U<<2);
    GPIOA->MODER |= (1U<<3);

    /****Configure the ADC module****/
    /*Enable clock access to the ADC module*/
    RCC->APB2ENR |= ADC1EN;
    /*Set conversion sequence start*/
    ADC1->SQR3 = ADC_CH1;
    /*Set conversion sequence length*/
    ADC1->SQR1 = ADC_SEQ_LEN_1;
    /*Enable ADC module*/
    ADC1->CR2 |= CR2_ADCON;
}

void start_conversion(void)
{
    /*Enable continuous conversion*/
    ADC1->CR2 |= CR2_CONT;
    /*Start ADC conversion*/
    ADC1->CR2 |= CR2_SWSTART;
}

uint32_t adc_read(void)
{
    /*Wait for conversion to be complete*/
    while (!(ADC1->SR & SR_EOC)) {}
    /*Read converted value*/
    return (ADC1->DR);
}
```

`adc.h`

```c
#ifndef STM32F446_BAREMETAL_ADC_H
#define STM32F446_BAREMETAL_ADC_H

#include "stm32f4xx.h"

void pa1_adc_init(void);
void start_conversion(void);
uint32_t adc_read(void);

#endif //STM32F446_BAREMETAL_ADC_H
```

There are three things going on, and none of them are complicated once you see the model ST picked for this peripheral:

1. **GPIO in analog mode.** PA1's mode bits (`MODER[3:2]`) get set to `11`. Every other chapter has used `00` (input), `01` (output), or `10` (alternate function). `11` is the analog one — it disconnects the digital input Schmitt trigger and the pull-up/pull-down resistors so the pin becomes a high-impedance feed straight into the ADC mux. Forgetting this is the classic "I get a flat 0 or a flat 4095" bug.

2. **The sequence registers.** The F4's ADC is built around a programmable sequence: you tell it *which channels* to convert and *in what order*, up to 16 of them. `SQR3` holds the first conversion in its lowest nibble, `SQR1[23:20]` holds "sequence length minus one." Since I only care about one channel, `SQR3 = ADC_CH1` (channel 1) and `SQR1 = 0` (length 1). The whole sequence machinery looks like overkill for a single reading, and it is — but it's why this same driver scales up to scanning multiple sensors later without restructuring.

3. **Start, and keep going.** `CR2.SWSTART` kicks off a conversion; `CR2.CONT` makes the ADC immediately restart when one finishes, so `DR` always holds a fresh value. Then `adc_read()` just spins on the End-Of-Conversion flag (`SR.EOC`), reads `DR` (which conveniently clears EOC), and returns. Blocking polling is fine here; interrupts and DMA are chapters 14 and 17.

## Part 2: printf Is Back

In chapter 10 I gave up on `printf` entirely and called `uart_write()` by hand, because linking newlib's `printf` against a 512-byte heap was hanging the chip. This chapter I actually wanted formatted output (`"Sensor Value: %d"`) and didn't feel like hand-rolling an itoa, so I went back and fixed the toolchain properly.

The fix is in `arm-none-eabi-gcc.cmake`:

```cmake
# -specs=nano.specs : use newlib-nano (small printf/malloc)
# -specs=nosys.specs: provide no-op syscall stubs (_write, _sbrk, ...) so the
#                      link resolves; _write is overridden below in uart.c so
#                      printf actually goes out the UART.
set(CMAKE_EXE_LINKER_FLAGS_INIT
    "${MCU_FLAGS} -specs=nano.specs -specs=nosys.specs -Wl,-Map=5_makefile_project.map,--gc-sections")
```

Two things changed from the chapter 10 setup:

- **`nano.specs`** pulls in newlib-nano, which has a much smaller `printf` that doesn't need megabytes of heap. This is the actual fix for the hang.
- **`nosys.specs`** supplies default stubs for the syscalls newlib needs (`_write`, `_sbrk`, `_close`, ...). I also added the CubeIDE-generated `syscalls.c`, which defines a real `_write()` that loops over the buffer and calls `__io_putchar()` — the same weak symbol my `uart.c` overrides. So the chain is `printf → newlib's _write → __io_putchar → uart_write → USART1->DR`.

I retired the hand-written C startup I'd been nursing since the blinky chapter and switched to the CubeIDE assembly file (`startup_stm32f411retx.s`). 

The net result is that this line now just works:

```c
printf("Sensor Value: %d\r\n", sensor_value);
```

## Part 3: The Ambient Light Sensor

The book's example uses a potentiometer as a variable voltage source. I had an ambient light sensor breakout handy, so I wired that to PA1 instead. These modules are about the simplest analog sensor you can buy: three pins — `VCC`, `GND`, and `OUT` — where `OUT` is a voltage that rises with illuminance. Internally it's a phototransistor (the common TEMT6000-style breakouts work this way) feeding a resistor, so brighter light → more collector current → higher voltage on the output pin, which the ADC reads directly.

Wiring:

| Sensor | Nucleo-F446RE |
|--------|---------------|
| `VCC`  | `3V3`         |
| `GND`  | `GND`         |
| `OUT`  | `PA1` (ADC1_IN1) |

That's it. No pull-ups, no level shifting — `OUT` already swings between 0 and 3.3 V, which is exactly the ADC's full-scale range on this board. With a 12-bit result register, a reading of `0` maps to 0 V and `4095` maps to 3.3 V, so each count is about **0.806 mV** (`3300 mV / 4096`).

The conversion is just:

```c
voltage_mv = (sensor_value * 3300UL) / 4096;
```

I didn't bother calibrating it to lux — that needs the sensor's datasheet responsivity and a known reference light source, and frankly "is it brighter or darker than before" is all I wanted. The raw counts are already a perfectly usable relative brightness reading.

## Part 4: Button-Gated Readings

The book's `main()` reads the ADC in a tight loop and spams the terminal as fast as the UART can ship bytes:

```c
while (1) {
    sensor_value = adc_read();
    printf("Sensor Value: %d\r\n", sensor_value);
}
```

That works, but at 115200 baud with a continuous-conversion ADC it's a wall of numbers scrolling past faster than you can read any of them — not useful when you're trying to wave your hand over the sensor and watch the value change. So I reused the PC13 user-button driver from the GPIO chapter and gated the reading on the button:

`main.c`

```c
#include "gpio.h"
#include "uart.h"
#include <stdio.h>
#include "adc.h"

bool push_button_state;
int sensor_value;

int main(void)
{
    pa1_adc_init();

    //Initialize UART
    uart_init();

    //Initialize LED
    led_init();

    //Initialize Button
    button_init();

    start_conversion();

    while (1) {
        //Get Push Button State
        push_button_state = get_btn_state();

        if (push_button_state) {
            led_on();
            sensor_value = adc_read();
            printf("Sensor Value: %d\r\n", sensor_value);
        } else {
            led_off();
        }
    }
}
```

Now hold the blue button and the LED lights up and the terminal fills with readings only while you're actively sampling — point the sensor at a window, cover it with your thumb, point it at a lamp, and you get a clean stream of values for each scenario instead of an unreadable blur. The ADC itself is still converting continuously in the background (I never disabled `CR2_CONT`); the button just gates whether I bother reading and printing `DR`.

## What's Next

This is the first peripheral in the book where the input isn't a clean digital edge — it's a number that means something about the physical world, and the same three-register dance (`MODER` → `SQR` → `CR2`) generalizes to every analog sensor I'll ever wire up. The obvious next experiments are plotting the values instead of printing them, and converting counts back to a real lux figure with the sensor datasheet. But the chapter did its job: the STM32 can now listen.
