---
title: "Planning a bar velocity tracker from scratch"
date: "2026-04-10"
updated: "2026-04-10"
categories:
  - "embedded"
  - "c"
  - "stm32"
  - "arm"
  - "projects"
excerpt: "A tentative design for a barbell velocity tracker: STM32L4, a high-resolution accelerometer, e-ink display, and a CR2032 coin cell. This is the optimistic version of the plan."
coverImage: "/images/posts/nucleo-l432kc.png"
coverWidth: 1200
coverHeight: 630
---

I've been working through Israel Gbati's *Bare-Metal Embedded C Programming* and writing up the [early](/blog/stm32-bare-metal-hello-world) [chapters](/blog/stm32-bare-metal-blinky-gnu-toolchain) as I go, including posts on [writing a linker script and startup code from scratch](/blog/stm32-linker-script-and-startup-code), [building a custom Makefile](/blog/chapter5gbati), and [understanding ARM CMSIS memory-mapped hardware](/blog/chapter6gbati). The book covers the fundamentals, but I need a capstone project. I want something that ties together peripherals, protocols, and power management into a device that does something useful.

This post is my plan for that project. This is a first draft. I have thought through the design, but I haven't built anything yet. I expect to hit problems once I start. The sensor might not match its datasheet. The power budget might not work as intended. Features that look simple might turn into rabbit holes. I'll iterate and cut scope as needed. This is the optimistic version.

## The idea

A device that straps to a barbell, measures concentric velocity during a set, counts reps, and buzzes when you've slowed down enough to signal you're past the hypertrophy threshold.

The training concept is straightforward: your first rep of a set is your fastest. As you fatigue, each rep gets slower. Research suggests that when concentric velocity drops below roughly 80% of your first rep, you've entered the zone where further reps yield diminishing returns and increasing recovery cost. A buzzer that fires at that threshold tells you to rack the bar.

Commercial velocity-based training devices exist, but they're expensive, phone-dependent, or designed for exercise science labs. I want something self-contained: strap it on, lift, hear the buzz, take it off.

## Hardware (tentative)

The current plan centres on three main components:

**MCU: STM32L432KC (Cortex-M4F, 80 MHz).** I picked this for the hardware FPU and power management. The core algorithm integrates accelerometer data using floating-point math. The hardware FPU handles this natively instead of burning cycles on software emulation. The L4 series also has aggressive low-power modes. The datasheet claims around 1.2 µA in deep sleep, which matters when running off a coin cell. Whether I achieve that in practice remains to be seen. I'll prototype on the NUCLEO-L432KC dev board and target the bare KCU6 package for the final PCB.

**Accelerometer: LSM6DSV16X.** This is the part I'm least certain about. I chose it based on the datasheet, not hands-on experience. On paper it looks good: low noise density, a large FIFO buffer so the sensor can collect data while the MCU sleeps, and an on-chip sensor fusion engine for estimating the gravity vector. I'll need tilt compensation when the bar path isn't perfectly vertical. I'll start with the similar LSM6DSOX on an Adafruit breakout for prototyping, since it's easier to breadboard. I'll swap to the DSV16X once the algorithm works, or I might find the DSOX is good enough and never swap.

**Display: 1.54" e-ink (SSD1681 driver).** This is probably my most unconventional choice. E-ink refreshes slowly (250 to 300 ms), and the instinct is that a workout device needs a fast display. But the timing actually works for barbell use. On bench press, the display updates at the bottom of the rep when the bar is on your chest and you cannot see the device. On squats and deadlifts, you cannot see a barbell-mounted device during the movement at all. You check it between sets when the bar is racked. The e-ink flicker happens during the eccentric pause, and by the time you look, the new rep count is static. The display also draws zero power to hold an image. That seems ideal for a coin cell.

**Power: CR2032 coin cell.** 3V nominal, fits the L432's 1.71-3.6V range directly with no regulator. The target is months of gym use on a single cell. Whether that's realistic depends on how well the power management works in practice.

**Feedback: piezo buzzer** for the slowdown alert. I'm least certain about the feedback mechanism. A piezo is simple, but gyms are loud and the device vibrates. I might try a small vibration motor or a bright LED, but audible alerts are my starting point.

## Firmware (rough sketch)

I'm still early in my embedded learning. I haven't worked with interrupts, DMA, or low-power modes yet, so this section is more "where I think the design is headed" than a concrete plan.

The broad idea is that the accelerometer buffers samples while the MCU sleeps. When enough samples accumulate, the MCU wakes up, reads them in bulk, and does the math: integrate acceleration to get velocity, detect when the bar is stationary to reset drift, count the rep, and compare velocity to the first rep to decide whether to buzz.

All the math should stay in single-precision `float` so the hardware FPU handles it directly. Using `double` anywhere, even accidentally via a literal like `0.01` instead of `0.01f`, would fall back to software emulation. That defeats the purpose of choosing a chip with an FPU.

The power strategy is "do the work as fast as possible, then sleep." The L4 has low-power sleep modes that draw microamps, and the sensor has a large FIFO so it doesn't need the MCU awake to collect data. How well this works in practice, whether the timing feels responsive, whether the sleep transitions match the datasheet claims, I don't know yet. That's what prototyping is for.

## What I'm deliberately leaving out

- **Bluetooth / phone app.** The device is self-contained. Adding wireless doubles the BOM complexity for a feature that doesn't demonstrate embedded skills any better than a buzzer. As a single person working on this project, this kind of feature is more about productizing and making it consumer-friendly. I don't intend to sell this or start a company, so it's out of scope.

- **Exercise auto-detection.** The LSM6DSV16X has a machine learning core that could theoretically classify squat vs. bench vs. deadlift. It would also be a distraction from the core signal processing work.

- **Workout history / data logging.** No external flash, no filesystem, no SD card. The device tells you about the set you're doing right now.

- **Multi-axis velocity.** Vertical only, via sensor fusion. Horizontal bar drift is interesting but out of scope. I don't think it would add much functionality or accuracy. Exercise science isn't a precise field, and the 80% velocity reduction threshold is approximate anyway.

- **RTOS.** Bare metal only. The state machine is simple enough that an RTOS would add complexity without buying anything, and bare metal is the point of this project.

- **USB charging.** Replaceable CR2032. The power story I'm aiming for is "it runs on a coin cell for months". I know if I had to recharge it frequently I wouldn't use it. The device I want to use is something you turn on, strap to the barbell, and then turn off when you're done. 

Any of these would be reasonable additions if I wanted to make a proper commercial product. None of them are worth the scope risk in v1.

## Rough development order

I don't have a detailed phase plan because I don't know enough to write one. What I have is a general order of attack:

1. **Get the sensor talking.** Wire the LSM6DSOX breakout to the NUCLEO-L432KC and read accelerometer data over I2C. This is the first milestone. If I can print raw acceleration values to a debugger, the rest is software.

2. **Get the math working.** Take those raw readings and turn them into something that resembles velocity. This means figuring out numerical integration, dealing with drift, and detecting when the bar is stationary so I can zero the velocity. I expect this to be the hardest part and the one where I'll learn the most.

3. **Add the display.** Wire up the e-ink screen over SPI and get it showing a rep count. The "update only when the bar is paused" logic depends on the zero-velocity detection from the previous step working, so this can't come first.

4. **Add the buzzer and threshold logic.** Compare each rep's velocity to the first rep and fire the buzzer when it drops below the threshold. This is conceptually simple but depends on the velocity numbers being reliable enough to make the comparison meaningful.

5. **Think about power.** Once everything works on the dev board, start figuring out how to make it sleep between sets and how much current it actually draws. This is where the low-power modes and the sensor's FIFO buffering come in.

6. **Move to custom hardware.** Design a PCB in KiCad, swap to the final sensor (LSM6DSV16X), order boards from JLCPCB, and get a friend to 3D print an enclosure. This assumes everything before it went well enough to justify the investment.

Some of these steps will probably collapse or split once I'm in the middle of it. I'm listing them in the order that makes sense now, knowing I'll adjust as I go.

## Why write this up now

I expect this plan to change. Parts of it might be naive. That's fine. I'd rather start with a clear target and adjust than try to design something perfect on paper before touching hardware. Writing it down gives me something concrete to work toward and something to look back on once the project is done. I want to see how my thinking about these problems changes over time. The next post will go back to working through Gbati's book. There's still a lot of foundation to lay before any of this leaves the planning stage.