---
title: "4-Step Primer on Navigating Embedded Rust HAL Documentation"
datePublished: Mon Nov 14 2022 17:46:26 GMT+0000 (Coordinated Universal Time)
cuid: clah2uplc000j08jqb3wn88he
slug: 4-step-primer-on-navigating-embedded-rust-hal-documentation
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1668445106503/yzv3JUqUW.png
tags: tutorial, iot, beginners, rust

---

## The Problem 😱

Starting out with embedded Rust, I found that reading embedded HAL documentation was really confusing. As I navigated around trying to find things I recall the feeling at times that things were going around in circles. Also, quite frankly, whenever I asked in different Rust circles, I couldn’t really get a clear explanation of how I should go about it. With some time and practice, at a certain point, I did manage to find my way around to reach what I need. Though if one would ask me to describe how I got to where I need, I wouldn’t have been able to answer. 

My eureka moment came along when I learned how to write device drivers in embedded Rust (I created a whole series about it [here](https://apollolabsblog.hashnode.dev/rust-embedded-graphics-with-the-max7219)). This was in addition to the significant time I spent navigating HAL source code. Interestingly enough, it turns out that the design of a HAL is quite similar in nature to a driver. From that point on, everything started clicking.  

In this post, I'll describe how one should go about navigating embedded HAL documentation. I also broke down the process into 4 main steps. Also, although this guide is referencing embedded HAL documentation, it should be generic enough to work for other documentation (Ex. drivers, libraries..etc.). 

I should caution that there still can be a certain level of vagueness that hangs around certain items. This can be attributed to documentation not being in perfect shape. Still, it's not something that would prevent one from proceeding. However, I would say that it's also part of the open source effort/responsibility of each of us to contribute their share in filling the gaps as we grow in the space.

%%[substart]

Ok, let's get started.

## 👣 🕐 Step 1 :  Identifying Tasks ✅​

When writing program code for a microcontroller, there are generally two tasks that would be involved. The first is configuring the peripherals and the device, followed by writing the application code that uses the peripherals. However, using crates based on the embedded Rust HAL, takes somewhat a slightly different approach.

A general theme adopted in the embedded Rust HALs is that PAC-level peripherals need to be, so to speak, "promoted" to the HAL-level before they can be used. Recall in Rust, a common line of code observed in a standard template looks as follows:

```rust
let dp = pac::Peripherals.take().unwrap();
```
Through this line, a handle, `dp`,  is created to obtain access to all PAC-level peripherals, albeit not yet usable at the HAL level. By promoting the peripherals, a new handle is then attached to the newly created HAL-level peripheral which HAL-level methods can be applied to them. As such, relative to peripherals and the embedded HAL, three tasks need to be performed as follows:

1. Promote 
2. Configure 
3. Use

Depending on the peripheral, tasks 1 and 2 are sometimes combined together. Additionally, a configuration can be done separately by applying methods to a handle, created in the first task, or alternatively by passing a configuration struct when instantiating the HAL-level handle in the first task.

Well, why am I talking about this? We're supposed to be talking about documentation, right? 

I am stating this because it's important to understand what our goal is in finding what we want from documentation before starting to read it. Especially that here the case is slightly different than usual. This means that for task 1 we need to find how to create an instance of the promoted PAC-level peripheral. For, task 2, we need to find out what methods, structs, or even enums can be used to configure a peripheral. Finally, for task 3 we need to find the functions/methods we can apply to an instance.

## 👣 🕑 Step 2 : Seek the Core Driver Struct 🧐

Our initial goal is to find the method that allows us to promote the PAC level peripheral to a HAL-level. As such, if you open any of the Rust embedded-HAL-based documentation, on the main page, the device peripherals are all organized in modules. The following screenshot is an example from the [stm32f4xx HAL](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/): 

![Main Documentation Page](https://cdn.hashnode.com/res/hashnode/image/upload/v1668445549242/vo0SqMad9.png align="center")

When clicking on a particular peripheral module name, the peripheral module page would typically include some description followed by a list of structs, traits, types, enums, functions, and even other modules in the current module. The following is a screenshot example of the ADC peripheral module from the [stm32f4xx HAL](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/): 


![Adc Documentation Page](https://cdn.hashnode.com/res/hashnode/image/upload/v1668445563460/jkoWTVdQj.png align="center")

Note here that in the ADC module there is only a modules and structs section with a bit of description on the side. Some other peripheral modules would contain much more than this. I chose the ADC module given that it's more compact to show in a single screenshot.

On this peripheral module page, and for any peripheral, the main goal is to **identify the core driver struct**. This core driver struct is your main starting point for everything related to the peripheral! It is the center of the peripheral universe. It is what everything else in the peripheral module revolves around (I guess I made my point 😁). This one struct is the one that contains all the method implementations that you need for a peripheral. There are some small exceptions where there are cases where you might find more than one driver struct, but this isn't typically the norm. 

In the case of the ADC, you'll find that it's the struct named `Adc`.

So how about all those other structs, enums, types...etc. that appear on a peripheral module page? Those are all used for different purposes by the driving struct methods where a common case is passing different configurations.

## 👣 🕒 Step 3 : Seek the Constructor Method or Trait 🏗

Now that we've identified the core driver struct, in this step we need to promote and then configure the peripheral. Or even do both at the same time. As such, promoting and/or configuring a peripheral can take place by looking for one of the following:

- **A constructor method in the driver struct** 🛠️​: This would be a method that accepts the PAC-level peripheral as a parameter. However, this method would normally also accept configuration parameters in the form of structs, enums, or other types as well. This means that we are both promoting and configuring the peripheral at the same time.
- **An extension trait** 🧬​: Sometimes in addition to a constructor, going back to the module level page, a trait (or traits) exists that can be applied to the PAC-level peripheral directly. This trait achieves exactly the same thing that the constructor method does albeit in a different manner. One other thing to note is that sometimes in cases like GPIO and clocks, a constructor does not exist in the core struct and an extension trait is all that exists. In those traits, one would find something like the `split` or `constrain` methods (refer to [this](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods) post for more detail). These traits promote the peripheral, albeit without configuring it. However, the configuration is then applied through the core struct methods.

As an example of a constructor method, we can look at the [`spi` module](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/spi/index.html) in the [stm32f4xx HAL](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/). The core driver struct is the `Spi` struct, clicking on it, on the next page, you would find a long list of methods including a `new` method with the following signature:

```rust
pub fn new(
    spi: SPI,
    pins: (SCK, MISO, MOSI),
    mode: impl Into<Mode>,
    freq: Hertz,
    clocks: &Clocks
) -> Self
```
Note that the first parameter is the PAC-level `SPI` peripheral that you would pass via the `dp` handle. Also, the remaining parameters are configuration parameters. Meaning that this constructor promotes and configures `SPI` and returns a HAL-level instance.

Going back to the [`spi` module](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/spi/index.html) page, scroll down to the Traits section, you can also find a `SpiExt` trait. Clicking on it, you'll find several traits of which one has the following signature:

```rust
fn spi<SCK, MISO, MOSI>(
        self, 
        pins: (SCK, MISO, MOSI), 
        mode: impl Into<Mode>, 
        freq: Hertz, 
        clocks: &Clocks
    ) -> Spi<Self, (SCK, MISO, MOSI), TransferModeNormal, Master>
```
This is a trait that can be applied directly to a PAC-level SPI peripheral achieving the same as before though in a different manner. 

 **📝​ Note:**
> What about those other modules that exist inside a peripheral module? I realized that often modules inside of peripheral modules are used to contain configuration structs. I've seen this with peripherals like ADCs or others that have several configuration parameters. As a result, it makes it more convenient to package configuration parameters in structs with their own methods. In code, these structs are often configured using a separate handle and then the handle is passed as a parameter into a peripheral constructor. It makes the code more organized. 

## 👣 🕓 Step 4 : Back to the Core Driver Struct 🚗

After configuring a peripheral I would say that the rest of your time would be all in the peripheral core driver struct. In the driver struct, you'll find a long list of methods that you can apply to the peripheral to read the status or control the peripheral in your application. Unless one would want to change the configuration again, all methods that one needs exist in the peripheral core driver struct.

Note that given the state of embedded Rust, it is not all uncommon to find out that a certain method isn't implemented. There are several ways to address this, you could either submit an issue in the GitHub repo of the hal and hope that the method is implemented sometime. Or you could do a pull request and update the HAL code with your own implementation. Finally, you can skate around this by implementing that particular feature at the PAC level manipulating registers directly. 

## Conclusion

I feel that navigating documentation is a necessary skill that is often overlooked. Given how to code up a certain application (ex. in a training course) versus trying to figure out how to do it yourself are totally two different experiences. In the latter, you need to stick yourself into the weeds of documentation trying to figure out how to do it for something you haven't seen before. This is something beginners often encounter and it's no different when learning embedded Rust. In this post, I give a primer on navigating Rust-embedded HAL documentation by breaking the process down into 4 main steps. This post is meant to give an initial direction for somebody starting out but is by no means comprehensive. Still, to become more comfortable, one would need to spend significant time navigating the docs and trying out different things.

Any thoughts? Please comment below 👇

%%[subend]