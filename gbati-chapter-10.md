---
title: "Chapter 10 of Bare-Metal Embedded C Programming"
date: "2026-05-21"
updated: "2026-05-21"
categories:
  - "c"
  - "embedded"
  - "stm32"
  - "projects"
excerpt: "UART driver on the STM32, printf build issues, and how the ST-LINK debugger was hijacking my serial output."
coverImage: "/images/posts/gbati-chapter-10-cover.svg"
coverWidth: 1200
coverHeight: 630
---

## Part 1: UART Driver and Initial Setup

Chapter 10 was about getting UART serial output working on the STM32. The chapter walks through configuring USART2 on PA2/PA3 of the STM32F446RE Nucleo board, setting up GPIO for alternate function mode, configuring the baud rate, and implementing a blocking transmit function.

`uart.c` (initial version using USART2 on PA2)

```c
#include "uart.h"

#define GPIOAEN     (1U<<0)
#define UART2EN     (1U<<17)

#define DBG_UART_BAUDRATE 115200
#define SYS_FREQ 16000000
#define APB1_CLK SYS_FREQ
#define CR1_TE (1U<<3)
#define CR1_UE (1U<<13)
#define SR_TXE (1U<<7)

static void uart_set_baudrate(uint32_t periph_clk, uint32_t baudrate);
static void uart_write(int ch);

int __io_putchar(int ch) {
    uart_write(ch);
    return ch;
}

void uart_init(void) {
    //Enable clock access
    RCC->AHB1ENR |= GPIOAEN;

    //Set the mode of PA2 to alternate function mode
    GPIOA->MODER &=~(1U<<4);
    GPIOA->MODER |=(1U<<5);

    //Set alternate function type to AF7(UART2_TX)
    GPIOA->AFR[0] |= (1U<<8);
    GPIOA->AFR[0] |= (1U<<9);
    GPIOA->AFR[0] |= (1U<<10);
    GPIOA->AFR[0] &=~ (1U<<11);

    //Enable Clock Access to UART2
    RCC->APB1ENR |= UART2EN;

    //Configure uart baudrate
    uart_set_baudrate(APB1_CLK,DBG_UART_BAUDRATE);

    //Configure transfer direction
    USART2->CR1 = CR1_TE;

    //Enable UART module
    USART2->CR1 |= CR1_UE;
}

static void uart_write(int ch) {
    //Make sure transmit data register is empty
    while (!(USART2->SR & SR_TXE)){}

    //Write to transmit data register
    USART2->DR =(ch & 0xFF);
}

static void uart_set_baudrate(uint32_t periph_clk, uint32_t baudrate)
{
    (void)periph_clk;
    (void)baudrate;
}
```

The main loop sends a message over UART once per second, toggling the LED between transmissions using the TIM2 timer from chapter 9.

`main.c`

```c
while(1){
    led_toggle();
    const char *msg = "Hello from STM32...\r\n";
    while (*msg) {
        uart_write(*msg++);
    }
    while (!(TIM2->SR & SR_UIF)) {}
    TIM2->SR &=~SR_UIF;
}
```

## Part 2: Build Issues — printf and nostdlib

The original code used `printf()` to send messages, which required overriding `__io_putchar()` to redirect output to the UART. However, the project was linking with `-nostdlib`, which meant there was no standard library and no `printf`.

The fix was to replace `-nostdlib` with `-specs=nosys.specs` in the CMakeLists.txt linker options. This links newlib-nosys, which provides the standard C library functions and routes `printf` through `__io_putchar`. However, this also required adding the `end` symbol to the linker script (`stm32_ls.ld`) so that `_sbrk` (the heap allocator used by newlib) knows where the heap starts:

```ld
.bss :
{
    . = ALIGN(4);
    _sbss = .;
    *(.bss)
    . = ALIGN(4);
    _ebss = .;
} > SRAM

. = ALIGN(4);
end = .;
_end = .;
```

In the end, `printf` was causing the program to hang on bare metal. I didn't end up figuring out how to get printf to work, so I put a pin on that for now and just used the write function.

## Part 3: My Pi Zero Challenge

Going beyond the book, I set myself an extra challenge: get the STM32 talking to a Raspberry Pi Zero 2 W over UART. This wasn't part of the actual chapter - I just wanted to see if I could make it work as a real-world application.

Setting up the Pi involved:

1. **Flashing the SD card** with Raspberry Pi OS and adding SSH keys via the `user-data` cloud-init file on the boot partition.
2. **Enabling the GPIO serial port** via `raspi-config` (Interface Options → Serial Port → enable hardware, disable login shell).
3. **Disabling the serial getty** (`serial-getty@ttyS0.service`) that was claiming the port.

The Pi uses `/dev/ttyAMA0` (PL011 UART) on GPIO 14/15 (pins 8 and 10 on the header) with the pins in ALT0 function mode.

## Part 4: The Debugging Saga — ST-LINK vs GPIO Serial

The STM32 code was confirmed working connecting the Nucleo board to my desktop via USB, I could see `"Hello from STM32..."` coming through on `/dev/ttyACM0` without any issues.

However, when I connected PA2 (STM32 TX) to GPIO15/Pin 10 (Pi RXD) and shared GND, nothing arrived at the Pi. Here's what I did to debug:

1. **Pi UART loopback test** — I jumpered Pi pin 8 (TX) to pin 10 (RX) and used a Python script with pyserial to confirm the Pi's UART could successfully send and receive its own data. The Pi hardware was fine.

2. **Serial getty conflict** — The Pi had a serial login shell running on the UART that was claiming the port. Disabling `serial-getty@ttyS0.service` and removing `console=serial0,115200` from `/boot/firmware/cmdline.txt` freed the port.

3. **The real culprit: ST-LINK** — On the Nucleo-F446RE, PA2 and PA3 are physically connected to both the STM32 and the on-board ST-LINK debugger chip through solder bridges SB62/SB63. The ST-LINK's UART receiver circuitry was loading the signal line, preventing the Pi from seeing any data. The STM32 was transmitting correctly (confirmed via USB), but the electrical signal wasn't strong enough or was being pulled by the ST-LINK. Simply switching pins was enough.

## Part 5: The Fix — PB6 and USART1

The solution was to move the UART output to a pin that isn't shared with the ST-LINK. I rewrote the driver to use **PB6** (USART1_TX, AF7), which is a free GPIO on the Nucleo header that goes directly to the pin without any ST-LINK interference.

Key changes from USART2 to USART1:
- GPIO port changed from **GPIOA** to **GPIOB** (`GPIOBEN`)
- USART changed from **USART2** (APB1) to **USART1** (APB2)
- Pin changed from **PA2** to **PB6** (alternate function mode on MODER bits 13:12, AFRL bits 27:24)
- Clock enable moved from `RCC->APB1ENR` to `RCC->APB2ENR`

After flashing the updated code and moving the jumper from PA2 to PB6, the Pi immediately started receiving data:

```
Listening on /dev/ttyAMA0...
Received: 'Hello from STM32 ... \r\nHello from STM32 ... \r\n...'
```