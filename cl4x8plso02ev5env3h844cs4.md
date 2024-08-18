---
title: "STM32F4 Embedded Rust at the HAL: GPIO Button Controlled Blinking"
datePublished: Mon Jun 27 2022 21:16:30 GMT+0000 (Coordinated Universal Time)
cuid: cl4x8plso02ev5env3h844cs4
slug: stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1660758722061/I-cuXfZpS.png
tags: tutorial, iot, rust, embedded

---

> This blog post is the first one of a multi-part series of posts where I explore various peripherals in the STM32f401RE microcontroller using embedded Rust at the HAL level.

%%[substart] 

## Introduction

In embedded development with Rust, the layers in which an individual can program a microcontroller include the peripheral access layer. Peripheral access layer features can be accessed via the peripheral access crate (PAC). On top of the PAC sits the hardware abstraction layer (HAL), accessed via the HAL crate. Finally, at the highest layer sits the board layer crate (more detail in the [Embedded Rust Book](https://docs.rust-embedded.org/book/start/registers.html)). The PAC is much closer to the register level and allows for much control but at the cost of portability. The HAL, on the other hand, probably allows for less control over fine details in a particular microcontroller but is much more portable. Board crates, although sit at the highest layer, are more specific to a particular development board.

Interestingly enough, when I set out to work with embedded Rust, I found that dealing with the PAC was easier for me to achieve what I want. After some time and work with the HAL, I feel that there were two main contributors to that experience. Firstly, the HAL probably is more involved in using certain features of Rust like generics and traits. Through leveraging the language features, the HAL is constructed in a way that incorporates techniques to ensure the safe operation of embedded devices. Secondly, the HAL documentation was hard to navigate at first, and in some cases, it was not as complete as I would have expected. It used to happen that for certain methods where proper descriptions were missing, I would either find myself experimenting to understand or looked at similar HALs to figure out how they work.

Reflecting on my experience, in this post, and the upcoming series of posts, I will be working with the STM32F401 microcontroller to present and walk through examples for different peripherals. I will be exclusively working at the HAL level (the [stm32f4xx\_hal](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/) in particular). I this post, I will be starting out with the GPIO peripheral. We'll see how we can configure GPIO, read inputs, and manipulate output at the HAL. The example here is a bit more advanced version of blinky.

> 📝 At the time of writing this post, it came to my attention that there is an additional HAL that targets STM32 device families (the [stm32-hal](https://github.com/David-OConnor/stm32-hal)). From what I figure, right now there seem to be two approaches for developing HALs. The first approach is trait driven so to speak where the [embedded-hal](https://github.com/rust-embedded/embedded-hal) is used as a foundation. The second approach is more application-driven and provides a high-level API that targets several families of a device. However, this exists only for the stm32 through the [stm32-hal](https://github.com/David-OConnor/stm32-hal). Right now, the first approach is what I found to be more widespread as it covers different microcontrollers and what this post is based on.

### Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with the basic template for creating embedded applications in Rust.
    

### Software Setup

All the code presented in this post in addition to instructions for the environment and toolchain setup are available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

### Hardware Setup

#### Materials

* [Nucleo-F401RE board](https://amzn.to/3yn6AIb)
    

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779074203/HjlMouvt1.jpg align="center")

#### Connections

There will be no need for external connections. On-board connections will be utilized and the include the following:

* LED is connected to pin PA5 on the microcontroller. The pin will be used as an output.
    
* User button connected to pin PC13 on the microcontroller. The pin will be used as an input.
    

## Software Design

In the application developed in this post, I want to cycle through several LED blinking frequencies based on a button press. Meaning, that every time I press the on-board button, I want to see the LED turning on and off at a different speed. In this section, I will focus on the design of the application algorithm itself rather than the configuration aspects.

Some assumptions I am taking in this design:

* Only GPIO peripherals are going to be used. Even for delay purposes, I will not be using any timer peripherals.
    
* The design will use a polling-based approach (rather than interrupts) to detect button press events. This is going to make things a bit tricky algorithmically (more explanation later).
    

For the second assumption, note that interrupts would have made things really convenient. Though I will not be using interrupts because of two reasons; the first is that interrupts are generally a more advanced concept, and the second is that interrupts in Rust are a bit more challenging to implement compared to the traditional approach in C or C++. As a result, I'd like to keep this post as simple as possible to grasp fundamental concepts first. In the future, I probably will create a separate post for an interrupt-based approach for the same application.

Moving on, let's try to represent our algorithm. I am going to use a flow chart as it would be helpful for this. Here's one possible approach:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714408271323/5d9d5c44-2049-4e80-96e6-7c3b2342c901.png align="center")

Let's analyze what is happening here. After configuring the GPIO pins, I am initializing a delay value (or variable). This is a value I am going to use to algorithmically create a delay. In addition, I am going to initialize the LED output (to on or off). Consequently, I initialize a count variable to zero and enter a loop that keeps on incrementing the count until it reaches the delay value I selected. Inside the delay loop, I am also polling the button to check if it gets pressed. If the button is pressed at any point, I need to decrease the delay value so that I increase the frequency of blinking. However, I have to also check that the new delay value does not go negative. As such, if the delay value drops below a certain threshold I want to reset it to the original value I started with. Once the check is complete I can toggle the LED output and go back to initialize the delay loop all over again.

**🚨 Important Notes:**

> 1️⃣ Note how I had to check for the button press in the delay loop. This is because if I wait till after I would potentially be missing button presses, especially when the delay is long. This is why earlier, I was mentioning that interrupts would be more convenient. This is because through interrupts I would be pausing operations to respond to the button pressing event immediately.

> 2️⃣ Since I am algorithmically creating a delay, note that this code is not portable between different devices and is not scalable. This means that you would see different delays depending on the device parameters and code responsiveness. How is this addressed? Typically delays are not created using software but rather hardware sources like timers/counters.

Let's now jump into implementing this algorithm.

## Code Implementation

### Crate Imports

In this implementation, three crates are required as follows:

* The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
    
* The `panic_halt` crate to define the panicking behavior to halt on panic.
    
* The `stm32f4xx_hal` crate to import the STMicro STM32F4 series microcontrollers device hardware abstractions on top of the peripheral access API.
    

```rust
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    gpio::Pin,
    pac::{self},
    prelude::*,
};
```

### Peripheral Configuration Code

Ahead of our application code, peripherals are configured through the following steps:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to take the PAC level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let dp = pac::Peripherals::take().unwrap();
```

2️⃣ **Promote the PAC-level GPIO structs**: We need to configure the LED pin as a push-pull output and obtain a handler for the pin so that we can control it. We also need to obtain a handle for the button input pin. According to the connection details, the LED pin connection is part of `GPIOA` and the button connection is part of `GPIOC`. Before we can obtain any handles for the LED and the button we need to promote the pac-level `GPIOA` and `GPIOC` structs to be able to create handles for individual pins. We do this by using the `split()` method as follows:

```rust
let gpioa = dp.GPIOA.split();
let gpioc = dp.GPIOC.split();
```

3️⃣ **Obtain a handle for the LED and configure it to an output**: As earlier stated, the on-board LED on the Nucleo-F401RE is connected to pin PA5 (Pin 5 Port A). As such, we need to create a handle for the LED pin that has PA5 configured to a push-pull output using the `into_push_pull_output()` method. We will name the handle `led` and configure it as follows:

```rust
let mut led = gpioa.pa5.into_push_pull_output();
```

For those interested, [this](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/struct.Pin.html) HAL documentation page has the full list of methods that the `Pin` type supports. Also, if you find the `split()` method confusing, please refer to my blog post [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods) for more detail.

4️⃣ **Obtain a handle and configure the input button**: The on-board user push button on the Nucleo-F401RE is connected to pin PC13 (Pin 13 Port C) as stated earlier. Pins are configured to an input by default so when creating the handle for the button we don't call any special methods.

```rust
let button = gpioc.pc13;
```

Note that as opposed to the LED output, the `button` handle here does not need to be mutable since we will only be reading it.

### Application Code

Following the design described earlier, I first need to initialize a delay variable `del_var` and initialize the output of the LED. `del_var` needs to be mutable as it will be modified by the delay loop. I also choose to set the initial output level to low by default. Using the same `Pin` methods mentioned [earlier](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/struct.Pin.html), there is a `set_low()` method that I use to achieve that.

```rust
    // Create and initialize a delay variable to manage delay loop
    let mut del_var = 10_0000_u32;

    // Initialize LED to on or off
    led.set_low();
```

Next inside the program loop, I first start by calling a delay function where inside of it I check if the button is pressed (detail following). After the delay completes, I toggle `led` using the `toggle()` method, again part of the methods available for the `Pin` type.

```rust
    // Application Loop
    loop {
        // Call delay function and update delay variable once done
        del_var = loop_delay(del_var, &button);

        // Toggle LED
        led.toggle();
    }
```

Here I've decided to abstract the delay into a function that takes two arguments, the delay value `del`, and the button handler `but`. Note that this is not necessary, but useful if you end up using the delay function more than once. The function `loop_delay` implementation looks as follows:

```rust
// Delay Function
fn loop_delay<const P: char, const N: u8>(mut del: u32, but: &Pin<P, N>) -> u32 {
    // Loop for until value of del
    for _i in 1..del {
        // Check if button got pressed
        if but.is_low() {
            // If button pressed decrease the delay value
            del = del - 2_5000_u32;
            // If updated delay value reaches zero then reset it back to starting value
            if del < 2_5000 {
                del = 10_0000_u32;
            }
            // Exit function returning updated delay value if button pressed
            return del;
        }
    }
    // Exit function returning original delay value if button no pressed (for loop ending naturally)
    del
}
```

Here you can see that we have created a loop that keeps going around until it reaches the value of `del`. As indicated in the design section, if the loop ends naturally then `del` remains unchanged and is returned to the calling function. Otherwise, at any point in time while delaying, if the button is pressed I can detect it using the `is_low()` method. At which point I will be decreasing `del` by `2_5000_u32`. If I end up with a `del` value less than `2_5000_u32` then I am restoring the original value I started with of `10_0000_u32`.

Why am I using the values `10_0000_u32` and `2_5000_u32`? It was actually trial and error. I kept trying values until I found ones that flash the LED in a satisfactory manner. As mentioned earlier since I am creating delays algorithmically, the duration of delays really depends on the platform in use.

**🚨 Important Notes:**

> Once you run the code, you'll see the LED flashing but you'll notice some weird behavior. The flashing frequencies would seem to keep changing in random order. This is because of an effect called "bouncing" on the mechanical button. Check the experimentation ideas section below for more detail.

## Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.

```rust
#![no_std]
#![no_main]

// Imports
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    gpio::Pin,
    pac::{self},
    prelude::*,
};

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let dp = pac::Peripherals::take().unwrap();

    // Configure the LED pin as a push pull ouput and obtain handler.
    // On the Nucleo FR401 theres an on-board LED connected to pin PA5.
    let gpioa = dp.GPIOA.split();
    let mut led = gpioa.pa5.into_push_pull_output();

    // Configure the button pin (if needed) and obtain handler.
    // On the Nucleo FR401 there is a button connected to pin PC13.
    // Pin is input by default
    let gpioc = dp.GPIOC.split();
    let button = gpioc.pc13;

    // Create and initialize a delay variable to manage delay loop
    let mut del_var = 10_0000_u32;

    // Initialize LED to on or off
    led.set_low();

    // Application Loop
    loop {
        // Call delay function and update delay variable once done
        del_var = loop_delay(del_var, &button);

        // Toggle LED
        led.toggle();

    }
}

// Delay Function
fn loop_delay<const P: char, const N: u8>(mut del: u32, but: &Pin<P, N>) -> u32 {
    // Loop for until value of del
    for _i in 1..del {
        // Check if button got pressed
        if but.is_low() {
            // If button pressed decrease the delay value
            del = del - 2_5000_u32;
            // If updated delay value reaches zero then reset it back to starting value
            if del < 2_5000 {
                del = 10_0000_u32;
            }
            // Exit function returning updated delay value if button pressed
            return del;
        }
    }
    // Exit function returning original delay value if button no pressed (for loop ending naturally)
    del
}
```

## Further Experimentation/Ideas

* Most mechanical press buttons require what is called debouncing. Buttons when pressed have a "bouncing" effect that can lead to multiple presses being detected. As a result, debouncing is required and can be achieved through hardware or software. The effect is best viewed by using an oscilloscope on the output of the pin. Check out [this](http://www.ganssle.com/debouncing.htm) page by Jack Ganssle for more detail about button bouncing and algorithms to eliminate the effect. If you look hard enough, you might even find a crate you can leverage for debouncing the button 😉.
    
* For some Rust practice, write the same code, eliminating the function and integrating the `loop_delay` body in the application loop.
    
* If you have access to an oscilloscope, determine the amount of delay generated by the delay function for a particular value on the Nucleo-F401RE.
    
* Connect multiple LED outputs and create different LED lighting patterns. You can use the button to switch between patterns.
    
* Instead of the LED, connect a buzzer to the output and generate different tones. You can use multiple button inputs to increase/decrease the frequency of the tone.
    

## Conclusion

In this post, an LED control application was created leveraging the GPIO peripheral for the STM32F401RE microcontroller on the Nucleo-F401RE development board. All code was created at the HAL level using the stm32f4xx Rust HAL. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]