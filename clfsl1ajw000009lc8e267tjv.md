---
title: "STM32F4 Embedded Rust at the PAC: Creating Hardware Abstractions with embedded-hal"
datePublished: Tue Mar 28 2023 18:19:26 GMT+0000 (Coordinated Universal Time)
cuid: clfsl1ajw000009lc8e267tjv
slug: stm32f4-embedded-rust-at-the-pac-creating-hardware-abstractions-with-embedded-hal
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1680027167005/36445fd0-179d-4cea-8317-896b4e6ff28d.png
tags: tutorial, programming-blogs, rust, embedded

---

## Introduction

In last week's post, I demonstrated the creation of a simple UART abstraction used to hide register-level details. What is obvious from the prior code is that its behavior is limited to a single UART peripheral. Meaning, that as we create more abstractions for other UART instances, we'll need to duplicate code. Rust provides an alternative through generics and traits. Through generics, a type can be determined at compile time. Traits, on the other hand, provide a way of defining common behavior. Thinking in an embedded context, peripherals among devices share a lot of common traits. For example, serial peripherals always have a `Write` or `Read` function. This is where the `embedded-hal` comes in. It's a community effort crate that was created to define common behavior among controllers through traits.

`embedded-hal` is a really powerful concept. The existence of `embedded-hal` enables the implementation of platform-agnostic drivers. This is done by creating a component driver that uses the `embedded-hal` traits and then deploying the driver on any platform that supports `embedded-hal` . From a PAC context, to implement the `embedded-hal` one would have to import the `embedded-hal` crate and then implement the particular behavior for the different peripherals.

In this post, I will be adjusting the code from [last week's post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-creating-hardware-abstractions) such that the USART abstraction would instead implement the `embedded-hal` for it's `Write` function.

%%[substart] 

## 1- Import the `embedded-hal` and `nb` Crates 📥

In the imports need to add `embedded_hal` and `nb` . `embedded_hal` includes all the required traits and `nb` is a crate that adds blocking operation support (more on this later).

```rust
#![no_std]
use embedded_hal as ehal;
use nb;
use stm32f401_pac as pac;
```

## 2- Update the Driver Struct 📲

In the previous code, recall the abstraction unit struct `Uart2Tx` :

```rust
pub struct Uart2Tx;
```

Which now changes to the following:

```rust
pub struct SerUart<USART> {
    usart: USART,
}
```

Note that I modified the struct name to `SerUart` such that the name is not particular to UART2, but more importantly, I introduced a generic type `UART`. This potentially would allow the struct to be instantiated to different UART instances.

## 3- Implement the `Write` Trait ✍️

Defining the `Write` trait behavior starts with `impl ehal::serial::Write<u8> for SerUart<pac::USART2>` . This line of code essentially says that we want to define the behavior of the `ehal::serial::Write<u8>` trait for the `SerUart` struct that implements the type `pac::USART2` . Within `ehal::serial::Write<u8>` there are two functions that need to be defined; `write` and `flush` .

The `seril::Write` trait requires us to implement an Error type that is associated with the serial interface. To do that one can enumerate all the possible errors and associate them with the type. Ideally, these would be used to identify the types of errors occurring. For now, I didn't want to implement any error functionality so I created an empty `enum` and associated it with the `Error` type:

```rust
pub enum Errors {}
impl ehal::serial::Write<u8> for SerUart<pac::USART2> {
    type Error = Errors;
    fn write(&mut self, word: u8) -> nb::Result<(), Self::Error> {
        // Implementation of write 
    }
    fn flush(&mut self) -> nb::Result<(), Self::Error> {
        // Implementation of flush
    }
```

Following that, I copied over the prior implementation of `write` to transmit a byte over the UART channel. Additionally, I left the flush implementation empty since I have no use for it in this particular implementation.

```rust
pub enum Errors {}

impl ehal::serial::Write<u8> for SerUart<pac::USART2> {
    type Error = Errors;
    fn write(&mut self, word: u8) -> nb::Result<(), Self::Error> {
        // Put Data in Data Register
        self.usart.dr.write(|w| unsafe { w.dr().bits(word as u16) });
        // Wait for data to get transmitted
        while self.usart.sr.read().tc().bit_is_clear() {}
        Ok(())
    }
    fn flush(&mut self) -> nb::Result<(), Self::Error> {
        Ok(())
    }
}
```

Note how the `write` and `flush` functions have a `nb::Result` type returned. `nb` is a special type of crate that allows the implementation of the `Result` enum with additional variants. The purpose of the additional variants is to allow the implementation of blocking (or non-blocking) operations if required. This is mainly to support blocking asynchronous programming models if required.

## 4- Update the Initialization Function Implementation 💾

Initialization of the UART peripheral remains more or less the same as before. The only minor difference is that the type for `SerUart` is specified in the implementation:

```rust
impl SerUart<pac::USART2> {
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
}
```

Thats it! now we have a hardware abstraction that supports `embedded-hal`!

## Conclusion

The `embedded-hal` is commonly mistaken to be a hardware abstraction layer (HAL) itself although it is not. On the contrary, `embedded-hal` is a crate that provides a common API that can be adopted among multiple platforms. This enables powerful concepts like platform-agnostic drivers. The `embedded-hal` crate is used alongside a HAL implementation to define the behavior of particular peripherals through traits. In this post, I modify a [previously created USART abstraction layer](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-pac-creating-hardware-abstractions) in Rust for the STM32F4 to adopt the `embedded-hal` traits.

%%[subend]