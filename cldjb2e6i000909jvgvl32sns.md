---
title: "What the HAL? The Quest for Finding a Suitable Embedded Rust HAL"
datePublished: Mon Jan 30 2023 21:11:01 GMT+0000 (Coordinated Universal Time)
cuid: cldjb2e6i000909jvgvl32sns
slug: what-the-hal-the-quest-for-finding-a-suitable-embedded-rust-hal
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1675112514352/090c4308-6729-4071-8056-aac8f6df46e3.png
tags: iot, rust, embedded

---

## Introduction

When starting out with embedded Rust, I used to naively think that all existing hardware abstraction layers (HALs) adopt more or less the same approach. Probably something closer to what one would see in other languages. Soon after I came to realize that I was mistaken. In fact, for a beginner, this probably gets even more confusing. Making the choice of a HAL to start out with can become really tough. Especially for one that may not have much background in embedded.

In this post, I attempt to categorize and explain the differences between different HALs that exist right now. As the embedded Rust space is continuously evolving, this post is not one expected to last for the ages. I only figure that anybody starting out would like to have a better understanding of the different options out there before picking a HAL.

%%[substart] 

Though first, there is something I want to get out of the way.

## Embedded-HAL is not a HAL 🤨

Well, maybe not in the sense of what is commonly known for a HAL to do. Consequently, the naming comes as a big source of confusion, and it's important to clarify before moving forward. The embedded-hal can be thought of as a crate that sits on top of an existing HAL crate to define common behavior through traits. Meaning that the embedded-hal cannot and does not operate as a standalone HAL but rather is adopted by existing device HALs to define common behavior. This is a really powerful concept as it enables the creation of platform-agnostic drivers.

Moving on.

## HAL Capabilities 🧰

Different HALs offer different features/capabilities. Some of these capabilities include the following:

**Device Support Level**

Ideally, one would like a single HAL to support all controllers. Meaning that one codebase is portable among all devices. Obviously, that is not the case, though there are HALs that have a support level more than others. For example, there are HALs that support a full series of controllers (ex. STM32), only a family in a series (ex. STM32F1xx), or even only a single device.

**Typestate Adoption**

Certain HALs leverage the Rust type system to manage pin configuration and check pin compatibility at compile time. This is a nice feature to have to ensure that you are configuring pins correctly. For example, the compiler would generate an error if one tries to configure a UART peripheral connection to a pin that does not support UART.

I personally think that the adoption of typestate might be a desirable feature for a beginner. Incorrectly configuring pins can lead to a frustrating debugging experience for a beginner if typestate isn't adopted.

**Embedded-hal Support**

This is about the level of support of embedded-hal traits. The more traits supported the better compatibility the HAL would have.

`std` **Support**

For the most part, given resource limitations, embedded Rust HALs do not support `std` libraries. However, some devices with more resources and advanced features (Ex. networking or wireless) would require `std` support.

## Some Other Considerations 🛠

**Documentation**

Needless to say, how well a HAL is documented is a really important aspect. Having many APIs with poor descriptions or even outdated signatures can send one into a spin.

**Ease of Use**

This could mean several things, and it all goes to the steepness of the learning curve, especially in Rust. One is how friendly the API is. Another is how verbose the code can get. Finally, how complicated is it to implement things like interrupts?

**API Coverage**

This has to do with how much of the device features are supported in a HAL. Meaning that some HALs do not necessarily implement all features in a controller (or family of controllers). This would result in one having to potentially leverage a different crate (Ex. PAC-level crate) to achieve a particular implementation.

## Categorizing HALs 📗

In the below table, I compare some of the HALs I came across according to the earlier-mentioned parameters. While the table is not exhaustive, most HALs fall within similar categories that are going to be discussed right after.

| **Crate Examples** 	| esp-hal variants 	| esp-idf-hal 	| Embassy HALs<br>(Ex. embassy-nrf, embassy-stm, & embassy-rp) 	| STM32-HAL<br>&<br>nRF-HAL 	| Various HALs for the STM32 (Ex. stm32f4xx-hal, stm32f1xx-hal...etc.) 	| nRF Device HALs 	|
|---	|---	|---	|---	|---	|---	|---	|
| **Device Support Level** 	| Family Support through individual HALs (esp32-hal, esp32c2-hal...etc.) 	| Series Support (Various ESP32 variants) 	| Series Support 	| Family Support 	| Family Support 	| Family and Device-Level Support Depending on Crate 	|
| **Adopts Typestate** 	| Yes 	| Yes 	| No, but still ensures that pin configurations are correct in an alternative manner. 	| No 	| Yes 	| Yes 	|
| **embedded-hal Style API** 	| Yes, for the most part. 	| Yes, for low-level hardware access. 	| Does not adopt embedded-hal style API, however, supports embedded-hal integration. 	| Does not adopt embedded-hal style API, however, supports embedded-hal integration. 	| Yes. There are small variations among crates for devices in the same series. 	| Yes. There are small variations among crates for devices in the same series. 	|
| **`std` Support** 	| No 	| Yes 	| No 	| No 	| No 	| No 	|
| **Documentation** 	| Well documented by espressif. 	| Well documented by espressif. 	| Can be outdated in some areas. Need to refer to the source code at times. 	| Not all aspects are accurate for all variants. 	| Depends on Crate 	| Depends on Crate 	|
| **Ease of use** 	| Dealing with interrupts and DMAs can be difficult. Code can get relatively verbose. 	| Requires a somewhat different approach than other HALs. The learning curve can be somewhat steep. Set up can be more involved. 	| Very easy-to-use friendly API. Non-verbose code. Need to get into async to do multi-threaded. 	| Really friendly API. Strips out a lot of the annoyances of trait-based HALs. 	| Dealing with interrupts and DMAs can be difficult. PAC struct promotion to HAL can be confusing. Code can get relatively verbose without a framework like RTIC. 	| Dealing with interrupts and DMAs can be difficult. PAC struct promotion to HAL can be confusing. Code can get relatively verbose without a framework like RTIC. 	|
| **API Coverage** 	| Very good coverage. There is a lot of consistency among different ESP HALs. 	| Very good coverage.  	| The nrf HAL is probably the most complete. One can find missing implementations. 	| Better coverage for STM devices than nRF. 	| Depends on Crate 	| Depends on Crate 	|

From what can be observed in the table, Rust-based HALs seem to all fall within four categories:

* **embedded-hal trait-based HALs:** There could be a better description than this. However, this category has the widest base of implementations with more options than can be mentioned here. A more comprehensive list can be found on the [awesome embedded Rust repository](https://github.com/rust-embedded/awesome-embedded-rust).
    
* **Embassy HALs:** Current HALs provide support only for the [stm32](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/index.html), [rp](https://docs.embassy.dev/embassy-rp/git/rp2040/index.html), and [nRF](https://docs.embassy.dev/embassy-nrf/git/nrf52805/index.html).
    
* **HALs with** `std` **support:** This is exclusive to [ESP32](https://github.com/esp-rs/esp-idf-hal) devices right now.
    
* **Typestate-free HALs:** This is in exchange for better ergonomics as the author claims. Only two HALs fall in this category right now which are the [STM32-HAL](https://github.com/David-OConnor/stm32-hal) & [nRF-HAL](https://github.com/placrosse/nrf-hal).
    

## So What Route Should I Take? 🤔

That's the million-dollar question. I often think if I were to do things all over, would I pick the same route? Below I analyze the different routes and give my personal opinion on each.

**The Embassy Route**

Embassy has the friendliest API and configuration is a breeze. Even setting up interrupts and DMA is quite straightforward. However, at some point, one would have to get involved with `async` . This is not a bad thing really, on the contrary, it probably is better to adopt going forward. However, as a personal preference, I'd rather avoid any underlying frameworks if beginning with embedded. I like to understand how to interact directly with a controller without any intermediaries.

**embedded-hal Trait-based HAL Route**

While API in this route is not as friendly as embassy and certain things like interrupts can be a bit painful, I think it still is worth picking. It is a route I personally took and it helped me understand certain concepts better. Though this is a route where most options of HALs exist to choose from. The challenge here is picking a HAL with a good level of support and documentation. Obviously, this is not easy to figure out for a beginner. The great part right now is the emergence of the ESP HALs with official support from Espressif. As such, an ecosystem is quickly growing around them.

At the time I started, Espressif support for ESP HALs was still non-existent. However, if I would start over, one of the ESP HALs would certainly be my choice.

**HALs with** `std` **Support Route**

I would avoid this route in the beginning due to some of the same reasons I would in embassy. Add to it that the APIs are not nearly as friendly as embassy's and the code can get quite verbose.

**Typestate-free Route**

For the HALs that exist in this route, the nice part is that they are relatively easy to work with. Though I figure typestate would be helpful for a beginner to not shoot themselves in the foot. Additionally, it introduces a nice feature in Rust not available in other languages commonly used in embedded like C. Finally, HALs in this route don't seem to be as popular as other HALs.

## Conclusion

Navigating the HAL space in embedded Rust can be a rough experience. This post analyzes the Rust HAL space and looks at the different options that exist. Have any questions/comments? Share your thoughts in the comments below 👇.

%%[subend]