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

The chapter starts with USART2 on PA2/PA3, but I ended up moving to USART1 on PB6 after running into ST-LINK interference (more on that in Part 4). Here's the final working driver:

`uart.c`

```c
#include "uart.h"

#define GPIOBEN     (1U<<1)
#define USART1EN    (1U<<4)

#define DBG_UART_BAUDRATE 115200
#define SYS_FREQ 16000000
#define APB2_CLK SYS_FREQ
#define CR1_TE (1U<<3)
#define CR1_UE (1U<<13)
#define SR_TXE (1U<<7)

static void uart_set_baudrate(uint32_t periph_clk, uint32_t baudrate);

int __io_putchar(int ch) {
    uart_write(ch);
    return ch;
}

void uart_init(void) {
    //Enable clock access to GPIOB
    RCC->AHB1ENR |= GPIOBEN;

    //Set the mode of PB6 to alternate function mode (bits 13:12 = 10)
    GPIOB->MODER &=~(1U<<12);
    GPIOB->MODER |=(1U<<13);

    //Set alternate function type to AF7 (USART1_TX) for PB6 (bits 27:24 = 0111)
    GPIOB->AFR[0] |= (1U<<24);
    GPIOB->AFR[0] |= (1U<<25);
    GPIOB->AFR[0] |= (1U<<26);
    GPIOB->AFR[0] &=~(1U<<27);

    //Enable Clock Access to USART1
    RCC->APB2ENR |= USART1EN;

    //Configure uart baudrate
    uart_set_baudrate(APB2_CLK, DBG_UART_BAUDRATE);

    //Configure transfer direction
    USART1->CR1 = CR1_TE;

    //Enable UART module
    USART1->CR1 |= CR1_UE;
}

void uart_write(const int ch) {
    //Make sure transmit data register is empty
    while (!(USART1->SR & SR_TXE)){}

    //Write to transmit data register
    USART1->DR =(ch & 0xFF);
}

static uint16_t compute_uart_bd(uint32_t periph_clk, uint32_t baudrate) {
    return ((periph_clk + (baudrate/2U))/baudrate);
}

static void uart_set_baudrate(uint32_t periph_clk, uint32_t baudrate) {
    USART1->BRR = compute_uart_bd(periph_clk, baudrate);
}
```

The baud rate computation uses the standard formula: `(periph_clk + baudrate/2) / baudrate`. With APB2 clocked at 16 MHz and a target of 115200 baud, that gives a BRR of 139.

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

In the end, `printf` was causing the program to hang on bare metal. Looking back, the culprit I think is the heap size. My linker script reserves only 512 bytes for the heap (`__max_heap_size = 0x200`). newlib's `printf` needs heap space for formatting buffers, and 512 bytes is not enough. Bumping the heap or switching to a minimal printf implementation like `tinyprintf` would likely fix it, but for now I just called `uart_write` directly and moved on.

## Part 3: My Pi Zero Challenge

Going beyond the book, I set myself an extra challenge: get the STM32 talking to a Raspberry Pi Zero 2 W over UART. This wasn't part of the actual chapter. I wanted to see if I could make the microcontroller talk to a real Linux system, since that's the kind of thing you'd actually do in a project, not just loop back to your own desktop.

The Pi was a fresh headless install. I flashed the SD card with Raspberry Pi OS and dropped my SSH keys into the `user-data` cloud-init file on the boot partition so I could get in over Wi-Fi without ever plugging in a monitor or keyboard. Then it was a matter of freeing up the hardware UART:

1. **Enabling the GPIO serial port** via `raspi-config` (Interface Options → Serial Port → enable hardware, disable login shell).
2. **Disabling the serial getty** (`serial-getty@ttyS0.service`) that was claiming the port.

The Pi uses `/dev/ttyAMA0` (PL011 UART) on GPIO 14/15 (pins 8 and 10 on the header) with the pins in ALT0 function mode.

## Part 4: The Debugging Saga — ST-LINK vs GPIO Serial

The STM32 code was confirmed working. Connecting the Nucleo board to my desktop via USB, I could see `"Hello from STM32..."` coming through on `/dev/ttyACM0` without any issues.

However, when I connected PA2 (STM32 TX) to GPIO15/Pin 10 (Pi RXD) and shared GND, nothing arrived at the Pi. Here's what I did to debug:

1. **Pi UART loopback test** — I jumpered Pi pin 8 (TX) to pin 10 (RX) and used a Python script with pyserial to confirm the Pi's UART could successfully send and receive its own data. The Pi hardware was fine.

2. **Serial getty conflict** — The Pi had a serial login shell running on the UART that was claiming the port. Disabling `serial-getty@ttyS0.service` and removing `console=serial0,115200` from `/boot/firmware/cmdline.txt` freed the port.

3. **The real culprit: ST-LINK** — On the Nucleo-F446RE, PA2 and PA3 are physically connected to both the STM32 and the on-board ST-LINK debugger chip through solder bridges SB62/SB63. The ST-LINK's UART receiver circuitry was loading the signal line, preventing the Pi from seeing any data (I think not 100% sure on this part). The STM32 was transmitting correctly (confirmed via USB), but the ST-LINK's receiver was loading the PA2 trace, and the Pi at the end of a jumper wire wasn't seeing enough of a signal edge to register the data. The fix was simple: use a pin the ST-LINK isn't tied to.

## Part 5: The Fix — PB6 and USART1

The fix was to move to **PB6** (USART1_TX, AF7), a free GPIO on the Nucleo header that goes straight to the pin with no ST-LINK in the path. The full code for this version is shown in Part 1 above.

Key changes from the book's USART2 setup:
- GPIO port changed from **GPIOA** to **GPIOB** (`GPIOBEN`)
- USART changed from **USART2** (APB1) to **USART1** (APB2)
- Pin changed from **PA2** to **PB6** (MODER bits 13:12, AFRL bits 27:24)
- Clock enable moved from `RCC->APB1ENR` to `RCC->APB2ENR`

After flashing and moving the jumper from PA2 to PB6, the Pi immediately started receiving data:

```
Listening on /dev/ttyAMA0...
Received: 'Hello from STM32 ... \r\nHello from STM32 ... \r\n...'
```