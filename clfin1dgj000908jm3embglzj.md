---
title: "STM32F4 Embedded Rust at the PAC: Creating Hardware Abstractions"
datePublished: Tue Mar 21 2023 19:17:47 GMT+0000 (Coordinated Universal Time)
cuid: clfin1dgj000908jm3embglzj
slug: stm32f4-embedded-rust-at-the-pac-creating-hardware-abstractions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1679424903344/d1694868-2913-4b3a-a8b0-15263bba24c8.png
tags: tutorial, rust, embedded, embedded-systems

---

> ***This blog post is the sixth of a multi-part series of posts where I explore various peripherals in the STM32F401RE microcontroller using embedded Rust at the PAC level.***

Prior posts include (in order of publishing):

1. [**STM32F4 Embedded Rust at the PAC: svd2rust**](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-svd2rust)
    
2. [**STM32F4 Embedded Rust at the PAC: GPIO Control**](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-gpio-control)
    
3. [**STM32F4 Embedded Rust at the PAC: System Clock Configuration**](https://hashnode.com/post/cled7b634000308l179747njg)
    
4. [**STM32F4 Embedded Rust at the PAC: SysTick Delay**](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-systick-delay)
    
5. [**STM32 Embedded Rust at the PAC: UART Communication**](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-uart-communication)
    

## Introduction

In the series of posts over the last few weeks, I've been creating PAC-level examples at the register level. Obviously, a lot of low-level knowledge is required to develop at the PAC level. Additionally, the code can become really verbose. As such, adding abstractions would not only help in reducing the amount of code but also reduce the amount of required low-level knowledge.

In this post, I am going to grab existing code that I've done in the previous UART post and create an abstraction around it. This would be a step in the direction of creating a hardware abstraction layer (HAL) for a device. What will be interesting to see is that it's similar to creating component drivers I've explained in a [past post](https://apollolabsblog.hashnode.dev/4-simple-steps-for-creating-a-platform-agnostic-driver-in-rust). Ultimately, a proper HAL would have larger goals than creating abstractions over individual peripherals, instead, a HAL would try to encompass as many devices as possible. One way of doing that in Rust is using the embedded-hal crate which achieves that through traits. I will show how the embedded-hal can be incorporated into an abstraction crate in a later post.

%%[substart] 

In what follows, I will replicate the steps of the "[4 Simple Steps for Creating a Platform Agnostic Driver in Rust](https://apollolabsblog.hashnode.dev/4-simple-steps-for-creating-a-platform-agnostic-driver-in-rust)" post. I will also add additional steps that adjust the code from the [UART post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-uart-communication) to use the new abstraction.

## **1️⃣** Create a Library Binary Package that Imports the Embedded HAL 📚

Instead of creating a regular binary package with a [`main.rs`](http://main.rs) here we need a library package. I will call the library package `stm32-uart-hal` . The library package can be created with cargo in the command line as follows:

```rust
$ cargo new stm32-uart-hal --lib
```

Next, the pac needs to be imported in the [`lib.rs`](http://lib.rs):

```rust
#![no_std]
use stm32f401_pac as pac;
```

## 2️⃣ Create the Peripheral Driver Struct 🏗

This is the main part of the peripheral abstraction layer. We'd have to define the struct that through it, all the function implementations will be provided. This looks something like this:

```rust
pub struct Uart2Tx;
```

I am calling the struct `Uart2Tx` referring to the USART2 peripheral in the STM32 doing only transmit operation. Note that `Uart2Tx` is a unit struct created to associate implementations with it.

## 3️⃣ Implement the Driver & its Functions 🚘

In this part, we need to create an implementation block that includes the functions associated with the driver. Ahead of creating the driver block though, for convenience, I want to create a `Config` struct that encompasses the configuration parameters for the UART block. For now, I will include the USART clock frequency and the baud:

```rust
pub struct Config {
    pub freq: u32,
    pub baud: u32,
}
```

Consequently, the driver implementation block would look like this:

```rust
impl Uart2Tx {
    pub fn init(clocks: &pac::RCC, usart: &pac::USART2, cnfg: Config) {
        // Enable Clock to USART2
        clocks.apb1enr.write(|w| w.usart2en().set_bit());

        // Enable USART2 by setting the UE bit in USART_CR1 register
        usart.cr1.reset();
        usart.cr1.modify(|_, w| {
            w.ue().set_bit() // USART enabled
        });

        // Program the UART Baud Rate
        usart
            .brr
            .write(|w| unsafe { w.bits(cnfg.freq / cnfg.baud) });

        // Enable the Transmitter
        usart.cr1.modify(|_, w| w.te().set_bit());

        // Wait until TXE flag is set
        while usart.sr.read().txe().bit_is_clear() {}
    }

    pub fn blocking_write(usart: &pac::USART2, data: u16) {
        // Put Data in Data Register
        usart.dr.write(|w| unsafe { w.dr().bits(data) });
        // Wait for data to get transmitted
        while usart.sr.read().tc().bit_is_clear() {}
    }
}
```

I created two functions, `init` to initialize the USART block and `blocking_write` to transmit a piece of data. In the functions, I pass references to the `RCC` and `USART2` register blocks so that they can be referenced in the implementation. The encompassing code is exactly the same code I used in the [UART Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-uart-communication) post, only divided into functions.

## 4️⃣ Incorporate the Abstraction & It's Functions 🧰

This is the final step, the abstraction that was created needs to be incorporated and used. In `cargo.toml` , the path of the abstraction layer needs to be added as a dependency. Then the abstraction can be imported with a statement like `use stm32_uart_hal as hal;`. As such, to configure and initialize the UART2 peripheral, it would boil down to these lines of code:

```rust
 // Configure USART Parameters
let usartcfg = hal::Config {
     freq: 16_000_000,
     baud: 115_200,
};

// Initialize USART
hal::Uart2Tx::init(&dp.RCC, &dp.USART2, usartcfg);
```

Consequently, transmitting a word would come down to a single line as follows:

```rust
hal::Uart2Tx::blocking_write(&dp.USART2, b'A' as u16);
```

Thats it! You can see that it didn't take much to create an abstraction. However, this is not the ideal form that one would desire. Meaning that one would rather have a single abstraction to accommodate all USART peripherals in a single device. Or even ultimately accommodate all USART peripherals in a family of devices (Ex. all STM32s). Accommodating one device probably would be manageable, however, expanding beyond that would be more involved. One would need to accommodate all the potential differences between devices in a family to configure the peripheral correctly. From what I have noticed, a lot of HAL crates seem to use macros for that, which in Rust are extremely powerful.

In the following sections, you can find the library implementation code in addition to the application code that incorporates the library.

## **📀** Hardware Abstraction Library Application Code

Here is the full code for the library/abstraction implementation described in this post. You can additionally find the full project and others available on the [**apollolabsdev Nucleo-F401RE**](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.

```rust
pub struct Config {
    pub freq: u32,
    pub baud: u32,
}
pub struct Uart2Tx;

impl Uart2Tx {
    pub fn init(clocks: &pac::RCC, usart: &pac::USART2, cnfg: Config) {
        // Enable Clock to USART2
        clocks.apb1enr.write(|w| w.usart2en().set_bit());

        // Enable USART2 by setting the UE bit in USART_CR1 register
        usart.cr1.reset();
        usart.cr1.modify(|_, w| {
            w.ue().set_bit() // USART enabled
        });

        // Program the UART Baud Rate
        usart
            .brr
            .write(|w| unsafe { w.bits(cnfg.freq / cnfg.baud) });

        // Enable the Transmitter
        usart.cr1.modify(|_, w| w.te().set_bit());

        // Wait until TXE flag is set
        while usart.sr.read().txe().bit_is_clear() {}
    }

    pub fn blocking_write(usart: &pac::USART2, data: u16) {
        // Put Data in Data Register
        usart.dr.write(|w| unsafe { w.dr().bits(data) });
        // Wait for data to get transmitted
        while usart.sr.read().tc().bit_is_clear() {}
    }
}
```

## **📀** Full Application Code

Here is the full code for the implementation described in this post.

```rust
#![no_std]
#![no_main]

// Imports
use cortex_m_rt::entry;
use panic_halt as _;
use stm32_uart_hal as hal;
use stm32f401_pac as pac;

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let dp = pac::Peripherals::take().unwrap();

    // Enable HSI Oscillator
    dp.RCC.cr.modify(|_, w| w.hsion().set_bit());

    // Wait for HSI clock to become ready
    while dp.RCC.cr.read().hsirdy().bit() {}

    // Enable Clock to GPIOA
    dp.RCC.ahb1enr.write(|w| w.gpioaen().set_bit());

    // Select Alternate Function for PA2
    dp.GPIOA.afrl.modify(|_, w| unsafe { w.afrl2().bits(7) });

    // Configure PA2 as Alternate Output
    dp.GPIOA.moder.modify(|_, w| unsafe { w.moder2().bits(2) });

    // Configure USART Parameters
    let usartcfg = hal::Config {
        freq: 16_000_000,
        baud: 115_200,
    };

    // Initialize USART
    hal::Uart2Tx::init(&dp.RCC, &dp.USART2, usartcfg);

    loop {
        hal::Uart2Tx::blocking_write(&dp.USART2, b'A' as u16);
    }
}
```

## **🔬** Further Experimentation/Ideas

* Expand the driver to include a receive function
    
* Expand the driver to support different USART peripheral instances. Enums could be used for that.
    

## Conclusion

Hardware abstractions in Rust are a powerful concept that can enable the support of many devices. In this post, I attempt to create a USART abstraction layer in Rust for the STM32F4. This driver is limited in what it can do and only scratches the surface when it comes to Rust capability. Additional capabilities can include incorporating embedded-hal traits, typestate, and the use of macros to cover a wider family of devices.

%%[subend]