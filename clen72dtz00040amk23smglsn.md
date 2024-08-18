---
title: "STM32F4 Embedded Rust at the PAC: SysTick Delay"
datePublished: Mon Feb 27 2023 19:09:49 GMT+0000 (Coordinated Universal Time)
cuid: clen72dtz00040amk23smglsn
slug: stm32f4-embedded-rust-at-the-pac-systick-delay
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1677524803004/080f1b4b-d1da-4293-8566-afd72c1687be.png
tags: tutorial, rust, embedded, beginnersguide

---

> This blog post is the fourth of a multi-part series of posts where I explore various peripherals in the STM32F401RE microcontroller using embedded Rust at the PAC level.

Prior posts include (in order of publishing):

1. [STM32F4 Embedded Rust at the PAC: svd2rust](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-svd2rust)
    
2. [STM32F4 Embedded Rust at the PAC: GPIO Control](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-gpio-control)
    
3. [STM32F4 Embedded Rust at the PAC: System Clock Configuration](https://hashnode.com/post/cled7b634000308l179747njg)
    

%%[substart] 

## 🎬 Introduction

In the series of PAC posts thus far, I haven't used any delays yet. I could have done something in software but it would not be too deterministic as program parameters change. More accurate measurable delay is better generated through hardware timers. Though to do that the timer clock source needs to be known. In that regard, generating and configuring clocks was covered in the [past post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-system-clock-configuration) which allows us now to play around with timers.

In this post, I will use the STM32 core Cortex-M SysTick timer to generate a measurable delay. I will be creating a blinky example that will use the delay measured by the timer. The SysTick timer is a good point to start with as it is simpler to deal with compared to other STM32 timers. Other timer peripherals tend to have many configuration options and advanced functionality that are not needed for the time being.

### **📚** Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with the basic template for creating embedded applications in Rust.
    

### **💾** Software Setup

All the code presented in this post in addition to instructions for the environment and toolchain setup is available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any Rust updates.

### 🛠 Hardware Setup

#### 🧰 Materials

* [Nucleo-F401RE board](https://amzn.to/3yn6AIb)
    

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779074203/HjlMouvt1.jpg align="center")

#### **🔌** Connections

There will be no need for external connections. On-board connections will be utilized and include the following:

* An LED is connected to pin PA5 on the microcontroller. The pin will be used as an output.
    

## **👨‍🎨** Software Design

The application in this post is blinky. An LED will be turned on and off using a measurable delay via the SysTick timer. For that, we need to understand the basic structure of the timer and how it can be configured. Something to also note is that the SysTick timer is part of the core peripherals. In the case of the STM32F4 device, the core is an ARM Cortex-M4.

### Hardware Timers

Hardware timers are basically electronic components that keep track of time in a digital system. They are used for a wide range of purposes, such as scheduling tasks, generating periodic interrupts, measuring time intervals, and synchronizing different components in a system. The figure below shows the basic structure of a timer.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677523467971/6ecd0767-88ad-4f9c-8b74-a87556b05658.png align="center")

The core unit of the timer is a counter that is *n*\-bits wide and supplied with a clock that has a certain frequency. The counter would then either increment or decrement its value at each clock instant. The value in the counter itself is often referred to as the ***current value***. When the timer reaches its end value, either \\(0\\) if counting down or \\(2^n -1\\) if counting up, the timer will overflow and indicate the event through some flag. This event is commonly referred to as a ***timeout***. In the event of a timeout, the timer can either cycle around to its original starting value or load a predetermined ***reload*** value. The reload value typically reflects the desired timeout/delay period.

To achieve a particular delay, the starting/reload value needs to be calculated to reflect the amount of delay needed. Meaning if for example a 1-second delay is desired, we need to determine the count value that triggers the timeout flag every 1 second. For that we can leverage the following formula:

$$T_{count} = f_{clk} \times T_{delay}$$

Where \\(T_{count}\\) is the starting and reload value (in counts), \\(f_{clk}\\) is the timer clock frequency (in counts/seconds) and \\(T_{delay}\\) is the amount of desired delay (in seconds). As an example, if we need a delay of 1 second and the clock frequency is 2 MHz that means we need a starting count/reload value of 2 million.

According to the STM32F4 datasheet, the Cortex-M SysTick timer runs on the processor clock (HCLK). As such, this clock frequency can be configured in the same manner as the [last post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-system-clock-configuration). Additionally, the SysTick timer counts in a countdown manner and the reload and current values are undefined at reset. Programmatically, the initialization sequence is as follows:

1. Program reload value in the `SYST_RVR` register.
    
2. Clear/Reset the current value in the `SYST_CVR` register.
    
3. Program the `SYST_CSR` control and status register.
    

After the timer is enabled in the final step the software needs to keep monitoring the timer for the timeout event.

## **👨‍💻** Code Implementation

### 📥 Crate Imports

In this implementation, three crates are required as follows:

* The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
    
* The `cortex_m` crate to import the Cortex-m peripheral API.
    
* The `panic_halt` crate to define the panicking behavior to halt on panic.
    
* The `stm32f4xx_pac` crate to import the STM32F401 microcontroller device PAC API that was created in the [first post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-svd2rust) in this series.
    

```rust
use cortex_m_rt::entry;
use cortex_m;
use panic_halt as _;
use stm32f401_pac as pac;
```

### 🎛 Peripheral Configuration Code

**1- Obtain a Handle for the Device and Core Peripherals**: As we always do and part of the singleton pattern in embedded Rust, this needs to be done before accessing any peripheral or core register. Here I create a device peripheral handler named `dp` and core peripheral handler named `cp` as follows:

```rust
let dp = pac::Peripherals::take().unwrap();
let mut cp = cortex_m::Peripherals::take().unwrap();
```

Using this handle, I will be accessing the peripherals of the device.

**2- Configure System Clocks:** I simplfiy the clock configuration from the [last post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-system-clock-configuration) where I only use the HSE clock. The HSE clock is fed by a crystal on the board and has a value of 8 MHz. The main takeaway here is that the clock tree is configured to provide a value of HCLK (the processor clock) at 8 MHz.

```rust
// Enable HSE Clock
dp.RCC.cr.write(|w| w.hseon().set_bit());
// Wait for HSE clock to become ready
while dp.RCC.cr.read().hserdy().bit() {}
```

**3- Configure the LED Output:** The details of configuring GPIO were discussed in detail in a [prior post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-gpio-control) as well resulting in the following code:

```rust
//Enable Clock to GPIOA
dp.RCC.ahb1enr.write(|w| w.gpioaen().set_bit());
//Configure PA5 as Output
dp.GPIOA.moder.write(|w| unsafe { w.moder5().bits(0b01) });
```

**4- Configure the SysTick Reload Value:** The SysTick reload value needs to be loaded in the `SYST_RVR` register which is 24-bits wide.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677492408030/04ff14bc-22e3-4d56-acaa-bdbb9a81f54f.png align="center")

According to our earlier formula, since our HCLK is 8 MHz, to get a one-second delay, the calculation would result in a count of 8000000 counts. This is the value we would need to load into the `SYST_RVR` register as follows:

```rust
unsafe { cp.SYST.rvr.write(8000000) };
```

**5- Clear the Current Value:** The current value resides in the `SYST_CVR` register and can be cleared or set to the same as the reload value as shown below. This is to know what the starting value of the register will be. `SYST_CVR` is also 24-bits wide.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677492958553/a5f09cf4-9132-40a4-b77a-eb219d45e45d.png align="center")

```rust
unsafe { cp.SYST.cvr.write(8000000) };
```

**6- Configure the SysTick Control and Status Register and Enable the Timer:** This is the final step before running the timer. This is also done through the `SYST_CSR` register.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677493103155/cb5f388c-f868-48f7-b5b1-d7b05211c94e.png align="center")

`COUNTFLAG` is our ***timeout*** flag. We will be reading it later in our application to check if the timer expired. As for the rest, the snapshot below describes their function.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677493484577/15acd5ca-86bc-40c6-8cb1-c4ff7443fdbf.png align="center")

Here we need to:

1. Assert `CLKSOURCE` so that it uses the processor clock (HCLK).
    
2. Clear `TICKINT` since we're going to poll the timer (not going to use interrupts).
    
3. Assert `ENABLE` to enable the timer so that it starts counting.
    

The above results in a hexadecimal value of `0x0105` resulting in the following code:

```rust
unsafe { cp.SYST.csr.write(0x0105) };
```

This is it for configuration.

### 📱 Application Code

Since our application is a blinky, all we need to do is keep polling the `countflag` in the `SYST_CSR` register to see if it gets asserted. If `countflag` goes high, that means that 1 second has elapsed and we need to toggle the output LED. To check the `countflag`, in the `cortex_m` crate provides a `has_wrapped()` method that allows us to check if a timeout occurred. The code below reads the `countflag` and and if asserted toggles the output using the `modify` method on the gpio `odr` register. Note in toggling that we are reading the `odr5` bit using the read `r` token, inverting the read value and writing it back using the `w` write token.

```rust
loop {
    // Check if Timer Expired
    if cp.SYST.has_wrapped(){
        // Toggle Output Pin
        dp.GPIOA.odr.modify(|r, w| w.odr5().bit(!r.odr5().bit()));
    }
}
```

## 📀 Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.

```rust
#![no_std]
#![no_main]

// Imports
use cortex_m;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f401_pac as pac;

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let dp = pac::Peripherals::take().unwrap();
    let mut cp = cortex_m::Peripherals::take().unwrap();

    // Initlialize Clocks
    // Enable HSE Clock
    dp.RCC.cr.write(|w| w.hseon().set_bit());

    // Wait for HSE clock to become ready
    while dp.RCC.cr.read().hserdy().bit() {}

    //Enable Clock to GPIOA
    dp.RCC.ahb1enr.write(|w| w.gpioaen().set_bit());

    //Configure PA5 as Output
    dp.GPIOA.moder.write(|w| unsafe { w.moder5().bits(0b01) });

    // Set PA5 Output to High signalling end of configuration
    dp.GPIOA.odr.write(|w| w.odr5().set_bit());

    // Set the Reload Value Reflecting 1 second for a 8 MHz Clock
    unsafe { cp.SYST.rvr.write(8000000) };

    // Reset the Current Value to match the Reload Value
    unsafe { cp.SYST.cvr.write(8000000) };

    // Configure the CSR register and Enable Timer
    unsafe { cp.SYST.csr.write(0x0105) };

    loop {
        // Check if Timer Expired
        if cp.SYST.has_wrapped() {
            // Toggle Output Pin
            dp.GPIOA.odr.modify(|r, w| w.odr5().bit(!r.odr5().bit()));
        }
    }
}
```

## 🔬 Further Experimentation/Ideas

* Try blinking using different durations.
    
* Create a delay function that calculates the counts based on duration.
    

## Conclusion

In this post, Rust code was developed to use a timer to create delay in a STM32 device exclusively at the peripheral access crate (PAC) level. The application was developed for an STM32F401RE microcontroller deployed on the Nucleo-F401RE development board. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]