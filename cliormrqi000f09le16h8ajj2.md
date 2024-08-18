---
title: "ESP32 Embedded Rust at the HAL: SPI Communication"
datePublished: Fri Jun 09 2023 16:12:08 GMT+0000 (Coordinated Universal Time)
cuid: cliormrqi000f09le16h8ajj2
slug: esp32-embedded-rust-at-the-hal-spi-communication
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1686327030308/b1966911-8690-4f93-ba3f-3734ac2b6853.png
tags: tutorial, iot, rust, embedded, esp32

---

> ***This blog post is the eighth of a multi-part series of posts where I explore various peripherals in the ESP32C3 using embedded Rust at the HAL level. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [**ESP32 Embedded Rust at the HAL: GPIO Button Controlled Blinking**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-gpio-button-controlled-blinking)
    
2. [**ESP32 Embedded Rust at the HAL: Button-Controlled Blinking by Timer Polling**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling)
    
3. [**ESP32 Embedded Rust at the HAL: UART Serial Communication**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-uart-serial-communication)
    
4. [**ESP32 Embedded Rust at the HAL: PWM Buzzer**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-pwm-buzzer)
    
5. [**ESP32 Embedded Rust at the HAL: Timer Ultrasonic Distance Measurement**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-timer-ultrasonic-distance-measurement)
    
6. [**ESP32 Embedded Rust at the HAL: Analog Temperature Sensing using the ADC**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-analog-temperature-sensing-using-the-adc)
    
7. [ESP32 Embedded Rust at the HAL: GPIO Interrupts](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-gpio-interrupts)
    

%%[substart] 

## Introduction

Serial communication interfaces can be really hard to debug when running into an issue. Running blind to what is happening at the electrical level just leads to more guesswork. Though with the proper tools, a lot more insight can make things much easier. As a result, when it comes to serial debugging, logic analyzers are worth their weight in gold.

This post is going to take a slightly different approach from prior ones. Although the post is about SPI it would be a Wokwi logical analyzer tutorial. To elaborate, I will be using SPI in a loop-back configuration to explore the logic analyzer feature in Wokwi.

### 📚 Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with the basic template for creating embedded applications in Rust.
    
* Familiarity with the SPI interface.
    

### 💾 Software Setup

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/366980653780379649).

For this post, you need to download [PulseView](https://sigrok.org/wiki/Downloads) from the sigrok website. PulseView is an open-source logic analyzer software that is going to be used to view the output signals.

### 🛠 Hardware Setup

#### **Materials**

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    
* [The Wokwi Logic Analyzer](https://docs.wokwi.com/parts/wokwi-logic-analyzer)
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686306479599/bafe75e3-6955-4ea2-b3ba-ab2bd758ef72.png align="center")

#### 🔌 Connections

Connections include the following:

* Gpio7 (SCLK) to D0 on the Wokwi logic analyzer
    
* Gpio6 (MISO) to D1 on the Wokwi logic analyzer
    
* Gpio5 (MOSI) to D2 on the Wokwi logic analyzer
    
* Gpio4 (CS) to D3 on the Wokwi logic analyzer
    
* (MISO) Gpio6 to (MOSI) Gpio5 (loopback configuration)
    

## 👨‍🎨 Software Design

In the code introduced in this post, there isn't much of an application. Its mostly configuration since the hardware is configured in loopback mode (mosi connected to miso). The SPI peripheral will keep transferring the same message over and checking if it was received correctly.

Let's now jump into implementing this algorithm.

## 👨‍💻 Code Implementation

### 📥 Crate Imports

In this implementation the crates required are as follows:

* The `esp32c3_hal` crate to import the ESP32C3 device hardware abstractions.
    
* The `esp_backtrace` crate to define the panicking behavior.
    
* The `esp_println` crate to provide `println!` implementation.
    

```rust
use esp32c3_hal::{
    clock::ClockControl,
    gpio::IO,
    peripherals::Peripherals,
    prelude::*,
    spi::{Spi, SpiMode},
    timer::TimerGroup,
    Delay,
    Rtc,
};
use esp_backtrace as _;
use esp_println::println;
```

### 🎛 Peripheral Configuration Code

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to take the PAC-level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let peripherals = Peripherals::take();
```

**2️⃣ Disable the Watchdogs:** The ESP32C3 has watchdogs enabled by default and they need to be disabled. If they are not disabled then the device would keep on resetting. I'm not going to go into much detail, however, watchdogs require the application software to periodically "kick" them to avoid resets. This is out of the scope of this example, though to avoid this issue, the following code needs to be included:

```rust
let mut system = peripherals.SYSTEM.split();
let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

let mut rtc = Rtc::new(peripherals.RTC_CNTL);
let timer_group0 = TimerGroup::new(
        peripherals.TIMG0,
        &clocks,
        &mut system.peripheral_clock_control,
);
let mut wdt0 = timer_group0.wdt;
let timer_group1 = TimerGroup::new(
        peripherals.TIMG1,
        &clocks,
        &mut system.peripheral_clock_control,
);
let mut wdt1 = timer_group1.wdt;

rtc.swd.disable();
rtc.rwdt.disable();
wdt0.disable();
wdt1.disable();
```

3️⃣ **Instantiate and Create Handle for IO**: We need to configure the LED pin as a push-pull output and obtain a handler for the pin so that we can control it. We also need to obtain a handle for the button input pin. Before we can obtain any handles for the LED and the button we need to create an `IO` struct instance. The `IO` struct instance provides a HAL-designed struct that gives us access to all gpio pins thus enabling us to create handles for individual pins. This is similar to the concept of the `split` method used in other HALs (more detail [**here**](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)). We do this by calling the `new()` instance method on the `IO` struct as follows:

```rust
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
```

Note how the `new` method requires passing the `GPIO` and `IO_MUX` peripherals.

4️⃣ **Obtain handles for the SPI pins**: I will be creating these handles for convenience since the SPI configuration will take care of configuring the pins. This would help me remember what pins were assigned to which function:

```rust
let sclk = io.pins.gpio7;
let miso = io.pins.gpio6;
let mosi = io.pins.gpio5;
let cs = io.pins.gpio4;
```

5️⃣ **Obtain a handle and configure the SPI peripheral**: to create a SPI instance, the `Spi` type in the `esp32c3-hal` has a `new` method that takes 9 parameters, the first parameter is the SPI peripheral that we want to instantiate, the next 4 are the SPI pins that we created handles for, then the 5th parameter is the desired SPI frequency. The 6th parameter is the SPI mode which takes an `SpiMode` `enum`. The final two parameters are the clock handles.

```rust
let mut spi = Spi::new(
    peripherals.SPI2,
    sclk,
    mosi,
    miso,
    cs,
    25u32.kHz(),
    SpiMode::Mode0,
    &mut system.peripheral_clock_control,
    &clocks,
);
```

6️⃣ **Obtain a handle for the delay**: I'll be using the hal `Delay` type to create a delay handle as follows:

```rust
let mut delay = Delay::new(&clocks);
```

### 📱 Application Code

#### 🔁 Application Loop

I'm going to try two things here. The first is sending three bytes in three consecutive transactions. I will be printing the received byte to the console to confirm that the same transmitted byte is received:

```rust
loop {
    // Individual arrays multiple transfers
    let mut data = [0xde];
    spi.transfer(&mut data).unwrap();
    println!("{:x?}", data);
    let mut data = [0xca];
    spi.transfer(&mut data).unwrap();
    println!("{:x?}", data);
    let mut data = [0xad];
    spi.transfer(&mut data).unwrap();
    println!("{:x?}", data);
    delay.delay_ms(100u32);
}
```

You can see that I'm using the `transfer` method which takes a single `&[u8]` parameter. The `transfer` method transmits the data in a `data` array and stores the received data in the same `data` array. [This](https://wokwi.com/projects/366980653780379649) is a link to the Wokwi simulation. Once you run the simulation on Wokwi samples will be captured. After you stop the simulation a `wokwi-logic.vcd` fill will automatically be downloaded. To open it, you need to fire up [Pulseview](https://sigrok.org/wiki/Downloads) and click on the open icon and select "Import Value Change Dump data" as shown below:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686308325110/4a2c5850-d8ea-4908-ace6-26dbae13fefa.png align="center")

Once opened, you'll see the below options before the signals are loaded. You'll only need to increase the downsampling factor to reduce memory usage. A factor of 50 would be sufficient for most serial communication applications:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686308509026/4d30da2a-dda7-4f03-8585-57dffbe2848a.png align="center")

When the signals appear in the view, you can choose a decoder to make signal analysis easier. You can do that by clicking on the green and yellow icon in the upper right corner in which the menu shown below appears. Search for SPI and double click.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686313694847/75191e55-8702-4cf6-b9c1-a9195b124389.png align="center")

Once doing that, an additional SPI signal line will appear in the window. The signals will remain red until you assign the signals. To do that you need to click on the SPI tag/label on the left, and a menu will appear for you to identify the signals.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686313838278/6bfb01be-1f3e-4a40-bcd4-f21570778801.png align="center")

For our case, the signals are as shown below:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686313962871/4d6b3a48-c27a-4302-95e2-894dc7fd3c39.png align="center")

Notice how the signals appear as small lines or pulses. This is because we need to zoom in further. You can zoom in either by double-clicking on the signal in the window or using the plus icon on top. After zooming, you should see something like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686314110045/0283841d-31a2-49de-a59c-fd33b8e7af31.png align="center")

This is exactly what we expect. Notice that `0xde` is transmitted (and consequently received), followed by `0xce`, and finally `0xde`.

#### 🔁 Second Experiment

In SPI it's actually possible that we pack the three bytes shown earlier in one array and a single transaction. This is shown in the code below:

```rust
loop {
        // One array single transfer
        let mut data = [0xde, 0xca, 0xad];
        spi.transfer(&mut data).unwrap();
        println!("{:x?}", data);
}
```

Following the earlier steps, the corresponding logic analyzer signal is as follows:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686314430430/d3db3588-9e57-47a1-84cf-fef428edb8f8.png align="center")

I wanted to highlight this case since the decoder, in this case, seems to act weirdly. Although the SPI signal is correct, only the first byte seems to be decoded correctly. Additionally, not all bits seem to be decoded either. The point here is that you probably shouldn't rely completely on the decoders assuming they are generating correct answers.

## 📱 Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also the Wokwi project can be accessed [**here**](https://wokwi.com/projects/366980653780379649).

```rust
#![no_std]
#![no_main]

use esp32c3_hal::{
    clock::ClockControl,
    gpio::IO,
    peripherals::Peripherals,
    prelude::*,
    spi::{Spi, SpiMode},
    timer::TimerGroup,
    Delay,
    Rtc,
};
use esp_backtrace as _;
use esp_println::println;

#[entry]
fn main() -> ! {
    let peripherals = Peripherals::take();
    let mut system = peripherals.SYSTEM.split();
    let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

    // Disable the watchdog timers
    let mut rtc = Rtc::new(peripherals.RTC_CNTL);
    let timer_group0 = TimerGroup::new(
        peripherals.TIMG0,
        &clocks,
        &mut system.peripheral_clock_control,
    );
    let mut wdt0 = timer_group0.wdt;
    let timer_group1 = TimerGroup::new(
        peripherals.TIMG1,
        &clocks,
        &mut system.peripheral_clock_control,
    );
    let mut wdt1 = timer_group1.wdt;

    rtc.swd.disable();
    rtc.rwdt.disable();
    wdt0.disable();
    wdt1.disable();

    let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
    let sclk = io.pins.gpio7;
    let miso = io.pins.gpio6;
    let mosi = io.pins.gpio5;
    let cs = io.pins.gpio4;

    let mut spi = Spi::new(
        peripherals.SPI2,
        sclk,
        mosi,
        miso,
        cs,
        25u32.kHz(),
        SpiMode::Mode0,
        &mut system.peripheral_clock_control,
        &clocks,
    );

    let mut delay = Delay::new(&clocks);

    loop {
        // One array single transfer
        //let mut data = [0xde, 0xca, 0xad];
        // spi.transfer(&mut data).unwrap();
        // println!("{:x?}", data);

        // Individual arrays multiple transfers
        let mut data = [0xde];
        spi.transfer(&mut data).unwrap();
        println!("{:x?}", data);
        let mut data = [0xca];
        spi.transfer(&mut data).unwrap();
        println!("{:x?}", data);
        let mut data = [0xad];
        spi.transfer(&mut data).unwrap();
        println!("{:x?}", data);
        delay.delay_ms(100u32);
    }
}
```

## Conclusion

In this post, a simple SPI loopback application was created leveraging the SPI peripheral for the ESP32C3 and the Wokwi logic analyzer. The SPI code was created at the HAL level using the Rust esp32c3-hal. Additionally, a walkthrough of setting up the Wokwi logic analyzer and decoding signals was also provided. Have any questions/comments? Share your thoughts in the comments below 👇.

%%[subend]