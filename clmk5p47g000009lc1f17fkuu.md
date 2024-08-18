---
title: "The Embedded Rust ESP Development Ecosystem"
datePublished: Fri Sep 15 2023 05:25:50 GMT+0000 (Coordinated Universal Time)
cuid: clmk5p47g000009lc1f17fkuu
slug: the-embedded-rust-esp-development-ecosystem
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1694352037032/c49e2da5-8705-456e-8ec3-b672d238c0ca.png
tags: iot, rust, embedded, esp32

---

## Introduction

Espressif with ESP controllers has a more diverse support ecosystem compared to other embedded controllers. Many controller crates support `no-std` development only. On the other hand, ESPs also add support for `std` development. This is significant as it allows non-embedded Rustaceans to ease into embedded development.

The increased diversity while really beneficial, typically can have a significant disadvantage: confusion. All the added crates interfaces and examples mixing the different levels of abstraction can make one's head spin. This was one of the reasons that drove me to start learning series at a certain abstraction level and just stick to that one level. With `std` development, this effect might be amplified as there can potentially be more levels of abstraction introduced. As such, it would be important for one to understand what lies beneath.

In this post, I would like to add some insight into the Rust ESP ecosystem. I figured it would be a good topic to tackle as it wasn't always clear-cut to me from the start.

%%[substart] 

## `std` vs. `no-std` 🧐

In embedded development, there are two main development approaches; bare-metal and non-bare-metal or high-level. The difference lies in how they interact with the underlying hardware. Bare-metal programming involves writing code that directly interfaces with and controls the hardware, offering maximum control and minimal overhead. It's ideal for resource-constrained systems, real-time applications, and situations where precise hardware control is crucial. In contrast, a non-bare-metal approach, often using an operating system or higher-level libraries, adds layers of abstraction, simplifying development but potentially introducing additional overhead and sacrificing some level of control over the hardware.

It is worth noting that bare metal does not necessarily mean no OS. There are OSs that are closer to bare metal than they are to non-bare metal. Notably, real-time operating systems or RTOSs. Also, in the context of Rust, bare metal development is often referred to as `no-std` or "core library" development whilst non-bare metal is referred to as `std` of "standard library" development.

The choice between these approaches depends on the project's specific requirements, balancing control, efficiency, and development ease. As such, one might choose to use the Standard Library (`std`) in embedded systems when they require rich functionality, and portability across different platforms, and desire rapid development without delving into low-level details. The `std` library provides a wide range of pre-built functionality, and standardized APIs, and abstracts many low-level concerns. Also `std` development typically involves the usage of higher-performance higher-resource controllers.

On the other hand, one might opt for the Core Library (`no_std`) when dealing with embedded systems with limited memory resources, needing direct hardware control, facing real-time constraints, or having custom requirements. The no\_std approach allows for a smaller memory footprint, direct hardware interaction, predictable real-time performance, and greater customization. Consequently, no-`std` development typically involves the usage of lower-performance resource-constrained controllers.

## The ESP Ecosystem 🏡

Espressif provides support (an ecosystem) for both `std` and `no-std` development. Under both ecosystems, Espressif offers a variety of crates for different purposes. I would say that the `std` ecosystem is probably harder to navigate than the `no-std` ecosystem, however. Obviously, that is expected granted the natural higher level of complexity. It is worth noting that all of the Rust on ESP efforts are unified under the umbrella of [`esp-rs`](https://github.com/esp-rs) on github. Moreover, probably until this point in time, this is what makes ESP devices unique such that it's the only ecosystem that provides `std` support. Many, if not all, existing approaches for other devices are `no-std` based. The following sections will look at each ecosystem separately to help note the differences, the idea is to clarify how the ecosystems look like to help beginners understand the different abstractions.

## The ESP `no-std` Rust Ecosystem 🦶🎸

The no-std ecosystem follows the same layering approach that exists within embedded Rust. In embedded Rust, there are several levels of abstraction that are introduced on top of microcontroller hardware as shown in the figure below. The first level is the peripheral access crate (PAC) which gives us access to low-level microcontroller registers at the bit level. It's also worth noting that the PAC is specific to a particular microcontroller series. For ESP devices the different PACs are captured in the [esp-pacs repository](https://github.com/esp-rs/esp-pacs). The microarchitecture crate is at a similar abstraction level to the PAC but specific to processor core (Ex. RISC-V) functions.

Going another level up the chain, we then have the hardware abstraction layer (HAL) crate. HAL crates are supposed to offer more portability and user-friendly API for a particular processor. For ESP devices the different hals are captured in the [esp-hal repository](https://github.com/esp-rs/esp-hal). This occurs by implementing some common traits defined in what is referred to as the [embedded-hal](https://docs.rs/embedded-hal/latest/embedded_hal/). Additionally, the device HAL attempts to incorporate mechanisms, or wrappers around lower-level functions, that are part of the Rust safety model.

![rust embedded layers.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1650963194624/LJHBDqlVQ.png?auto=compress,format&format=webp align="center")

Among these several abstractions, we can program a microcontroller device at any level we like. Additionally, we can develop code with a mix of low-level and high-level abstractions. Obviously, to make code more portable it's better to stick to higher-level abstractions. Also in addition to the above, there exists other crates supporting other functions in `no-std` development. These include wifi services in the [esp-wifi repository](https://github.com/esp-rs/esp-wifi), heap allocators in the [esp-alloc repository](https://github.com/esp-rs/esp-alloc), logging features in the [esp-println repository](https://github.com/esp-rs/esp-println), exception handlers in the [esp-backtrace repository](https://github.com/esp-rs/esp-backtrace), and finally embedded storage traits in the [esp-storage repository](https://github.com/esp-rs/esp-storage).

## The ESP `std` Rust Ecosystem 🏘️

Rust `std` library development using ESP devices is built on the [ESP-IDF](https://idf.espressif.com/#:~:text=ESP%2DIDF%20is%20Espressif's%20official,as%20C%20and%20C%2B%2B.). The ESP-IDF (Espressif IoT Development Framework) is an open-source software development framework created by Espressif Systems for programming ESP32 and ESP8266 microcontrollers. It offers many features and components to simplify the development process, including peripheral drivers for hardware interfaces, a robust Wi-Fi and Bluetooth stack, Real-Time Operating System (RTOS) support, networking libraries, and command-line tools for building and flashing applications. The framework also provides extensive documentation and examples to aid developers in creating a variety of IoT and embedded projects.

The RTOS that ESP-IDF incorporates is FreeRTOS, which along with other components and abstractions, is written in C. To support Rust std development Espressif did not rewrite the whole new code base, but rather elected to introduce Rust layers on top of the existing framework. Rust can support that through the foreign function interface (FFI). For those interested, you can check [this](https://apollolabsblog.hashnode.dev/rust-ffi-and-bindgen-integrating-embedded-c-code-in-rust) post and [this](https://apollolabsblog.hashnode.dev/rust-ffi-and-cbindgen-integrating-embedded-rust-code-in-c) post about using the FFI.

The figure below shows the layering/dependencies of the Rust crates within the ESP-IDF ecosystem. In the lower abstraction layers of the ESP-IDF framework written in C, there exists the ESP-IDF hardware abstraction layer (HAL). On top of the ESP-IDF HAL sits the FreeRTOS kernel alongside other components. Components include, but are not limited to, network stacks, protocols, and wireless stacks. This is where the existing popular framework for C developers stops.

The Rust part was developed on top of the existing C code. Using the Rust FFI, bindings were created to utilize the underlying existing ESP-IDF framework. These first-layer bindings are accessed through the `esp-idf-sys` crate. Through the `esp-idf-sys` one can essentially call many of the existing underlying API by using Rust. However, many `esp-idf-sys` bindings are considered `unsafe` since they do not apply Rust safety mechanisms/features.

For Rust safety mechanisms, come in the `esp-idf-hal` crate which is a crate that applies Rust safe wrappers to the `esp-idf-sys` bindings. The `esp-idf-hal` in turn supports many ESP-IDF drivers such as GPIO, SPI, I2C, Timers, and UART, among others. The `esp-idf-hal` also implements the `embedded-hal` traits including `async`.

The last piece of the puzzle is the `esp-idf-svc` where it supports ESP-IDF services. Examples include Wifi, Ethernet, HTTP client & server, and MQTT, among others. Additionally, the `esp-idf-svc` implements the `embedded-svc` traits.

Finally, note within the figure that at an application layer we can have access to all the interfaces from the different crates. Meaning if I want to use `esp-idf-sys` bindings alongside some `esp-idf-svc` interface it would be possible.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1694270244438/afbe7bfe-f8bb-4637-b04e-ead67ef18b62.png align="center")

## Conclusion

Navigating the Rust-embedded ecosystem could be confusing for a newbie. Understanding the different options available is key to unlocking a lot of powerful options. In this post, particular focus is given to the options available in the espressif ESP ecosystem. The ESP ecosystem is one of the diverse embedded Rust ecosystems as the manufacturer Espressif officially supports Rust. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]