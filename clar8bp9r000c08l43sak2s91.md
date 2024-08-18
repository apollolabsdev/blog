---
title: "Embedded Rust & Embassy: GPIO Button Controlled Blinking"
datePublished: Mon Nov 21 2022 20:17:19 GMT+0000 (Coordinated Universal Time)
cuid: clar8bp9r000c08l43sak2s91
slug: embedded-rust-embassy-gpio-button-controlled-blinking
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1669061457909/YHXPakdWh.png
tags: tutorial, beginner, iot, rust

---

> This blog post is the first one of a multi-part series of posts where I will explore various peripherals of the STM32 microcontroller using the embedded Rust embassy framework.

%%[substart]

## Introduction
Until recently, I've found out that I had a misconception of what embassy is in relation to embedded Rust. Along with various internet descriptions I came along, the official description states that: "Embassy is a project to make async/await a first-class option for embedded development". As such, I thought embassy was a framework similar to the idea of RTIC but for async. Though it turns out that embassy is much more than that. As a matter of fact, what I found out is that part of embassy is a HAL implementation that is more of a well-rounded HAL compared to other HALs I've dealt with thus far. What do I mean by that? Ideally, a HAL layer would be as portable as possible. However, in the case of embedded-hal HALs, although they were portable to an extent, the way they were designed made them more specific to particular families of devices. For example, the stm32f4xx family had a separate HAL compared to the stm32f3xx or other stm32 families. Instead, with embassy, there's actually one stm32 HAL that rules them all.

In this blog post series, I'm going to experiment with different stm32f4xx peripherals using embassy. I will be working with the STM32F401 microcontroller to present and walk through examples for different peripherals. I will be exclusively working at the [embassy-stm32 HAL](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/index.html) level. In this post, I will be starting out with the GPIO peripheral. We'll see how we can configure GPIO, read inputs, and manipulate output. The example here is a bit more advanced version of blinky.

> 📝 Its worth noting that at the time of writing this post, embassy still leverages nightly features of Rust. This means that a stable version of embassy has not been released yet and is sort of still experimental. Nevertheless, the amount of features embassy provides is quite advanced in nature. Also as of now, the embassy HAL support is available only the stm32 and nrf devices.

### 📚 Knowledge Pre-requisites
To understand the content of this post, you need the following:
- Basic knowledge of coding in Rust.
- Knowledge of `async/await` and `Futures`.
- Familiarity with the basic template for creating embedded applications in Rust.
- Familiarity with interrupts in Cortex-M processors.

### 💾 Software Setup
All the code presented in this post in addition to instructions for the environment and toolchain setup are available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

### 🛠 Hardware Setup
#### Materials
- [Nucleo-F401RE board](https://s.click.aliexpress.com/e/_DdEhurV)

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779074203/HjlMouvt1.jpg align="center")

#### 🔌 Connections
There will be no need for external connections. On-board connections will be utilized and the include the following:
- LED is connected to pin PA5 on the microcontroller. The pin will be used as an output.
- User button connected to pin PC13 on the microcontroller. The pin will be used as an input.

## 👨‍🎨 Software Design
In the application developed in this post, I will to cycle through several LED blinking frequencies based on a button press. Meaning, that every time I press the on-board button, I want to see the LED turning on and off at a different rate. This is similar to past posts I've done both [with](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts) and [without](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling) interrupts using the [stm32f4xx-hal](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/). At that time, I left the interrupt-based approach for a separate post given that implementing it was more involved.  In this post, with embassy, I will be using interrupts from the get-go. In fact, what will become obvious, is that implementing interrupts in embassy is nothing but a breeze. 

For algorithm design purposes, I will assume that interrupts are configured already which will be detailed in the configuration steps in the implementation section. The algorithm representation remains the same to what I've done in the stm32f4xx-hal [interrupts post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts):

![Algorithmic State Machine](https://cdn.hashnode.com/res/hashnode/image/upload/v1660913041210/D-OU4bg3J.png align="center")

As before, the application will start in a state where the LED is turned on (or off). In this state, a timer would also be started. While in the "LED on" state two events could happen to make the system transition to another state; either a button press or a timer delay expiration. If the delay expires the application would transition to an "LED off" state that turns off the LED and resets the delay timer. On the other hand, if a button press is detected the application would transition to a "Button Pressed" state. The "Button Pressed" state would in turn adjust the amount of timer delay and then return to the state it transitioned from. The "LED off" state has exactly the same transition conditions as the "LED on" state. 

Note an interesting difference here is that we will be servicing an interrupt without really having the traditional service routine. Instead, there is this executor that will be running in the background that will be informed of the interrupt occurring and execute code in an `async` block accordingly. As such, we'll create code with two `async` tasks, one that `await`s on a delay to change the LED state and one that `await`s on a button press to change the delay state. I will explain more in the implementation section. Also, more detail on the executor can be found [here](https://embassy.dev/dev/runtime.html).

Let's now jump into implementing this algorithm.

## 👨‍💻 Code Implementation

> 📝 Although I followed the documentation to set up my first project, it wasn't all smooth sailing. It was mainly had to do with configurations/settings of configuration (toml) files. I figure this has to do with embassy still being considered to be experimental. Set up I managed to get working is available up in my [git repo](https://github.com/apollolabsdev/stm32-nucleo-f401re).

### 📥 Crate Imports 
In this implementation the crates required are as follows:
- The `core::sync::atomic` crate to import the `Atomic` sharing type.
- The `embassy_executor` crate to import the embassy embedded async/await executor.
- The `panic_halt` crate to define the panicking behavior to halt on panic.
- The `embassy_stm32` crate to import the embassy STM32 series microcontroller device hardware abstractions.

```rust
use core::sync::atomic::{AtomicU32, Ordering};
use embassy_executor::Spawner;
use embassy_stm32::exti::ExtiInput;
use embassy_stm32::gpio::{AnyPin, Input, Level, Output, Pin, Pull, Speed};
use embassy_time::{Duration, Timer};
use panic_halt as _;
```
#### 🌍 Global Variables 

In the application at hand, there will be two tasks that share a delay value. The button press detection task will adjust the delay and the LED control task will use it. As such, I create a global variable `BLINK_MS` to carry the delay value that I'm going to pass around. Here I use the `AtomicU32` type which is an integer type that can be safely shared between threads. The `AtomicU32` type has the same in-memory representation as the underlying integer type, `u32` but is considered safe to share between threads. 

```rust
static BLINK_MS: AtomicU32 = AtomicU32::new(0);
```
 As such, I would need to create a global shared variable to access the GPIO peripheral (remember singleton pattern).

### 💡 The LED Blinking Task
Ahead of defining the main task, it would be beneficial to define the LED blinking task as it would be kicked off/spawned by the main task. The LED task is expected to accept a GPIO pin as input and loop forever toggling the pin with a certain delay. 

1️⃣ **Create a blinking task and handle for the LED**: Tasks are marked by the `#[embassy_executor::task]` macro followed by a `async` function implementation. The task created is referred to as `led_task` defined as follows:

```rust
#[embassy_executor::task]
async fn led_task(led: AnyPin)
```

`AnyPin` marks a generic pin type and will allow us to configure the pin in the task. This means we need to obtain a handle for the LED pin and configure it to an output. As earlier stated, the on-board LED on the Nucleo-F401RE is connected to pin PA5 (Pin 5 Port A). As such, we need to create a handle for the LED pin that has PA5 configured to an output using the `gpio::Output` gpio output driver. In the [documentation](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/gpio/struct.Output.html), we can find that `gpio::Output` has a `new` constructor method with the following signature: 

```rust
pub fn new(
    pin: impl Peripheral<P = T> + 'd,
    initial_output: Level,
    speed: Speed
) -> Self
```
Where `pin` expects an argument of type `Pin` passing in one of the `Peripheral` pins,  `initial_output` expects an initial output level setting selected through the `Level` enum, and `speed` expects a pin speed selected through the `Speed` enum. Given this information, we will name the handle `led` and configure it as follows: 

```rust
let mut led = Output::new(p.PA5, Level::High, Speed::Low);
```
In addition to `new`, [this](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/gpio/struct.Output.html) documentation page has the full list of methods that the `Output` driver supports.

2️⃣ **Define the task loop**: Next enter the task loop. First we `load` the blink delay value from the global context as follows:

```rust
let del = BLINK_MS.load(Ordering::Relaxed);
```
Then we establish a timer delay using the following code:

```rust
Timer::after(Duration::from_millis(del_var.into())).await;
```
`Timer` comes from the [`embassy_time` crate](https://docs.rs/embassy-time/0.1.0/embassy_time/struct.Timer.html). `after` is a `Timer` instance method that accepts a `Duration` and returns a `Future`. As such, `await` allows us to yield execution to the executor such that the task can be polled later to check if the delay expired.

Once the delay expires, the LED is toggled using the [`toggle` method](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/gpio/struct.Output.html) for the `Output` type:

```rust
led.toggle();
```

Here's the full loop code:

```rust
    loop {
        let del = BLINK_MS.load(Ordering::Relaxed);
        Timer::after(Duration::from_millis(del.into())).await;
        led.toggle();
    }

```

### 📱 The Main Task (Button Press Task)

The start of the main task is marked by the following code:

```rust
#[embassy_executor::main]
async fn main(spawner: Spawner) 
```
As the documentation states: "The main entry point of an Embassy application is defined using the `#[embassy_executor::main]` macro. The entry point is also required to take a `Spawner` argument." As we'll see, `Spawner` is what will allow us to spawn or kick off `led_task`.

As indicated before, the main task will also be the one we manage the button press in. The following steps will mark the tasks performed in the main task.

1️⃣ **Initialize MCU and obtain a handle for the device peripherals**: As part of the embassy-stm32 crate an `init` function exists at the crate level with the following signature:

```rust
pub fn init(config: Config) -> Peripherals
```

The `init` function takes a `Config` configuration struct parameter and returns a `Peripherals` struct instance. The [`Config` parameter](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/struct.Config.html) allows us to configure the RCC and whether to enable debug during sleep. The [documentation](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/fn.init.html) states that this function is to "Initialize embassy". Looking into the source code, what the `init` function does is initialize device system clocks in addition to a variety of peripherals including timers, dma, and gpio. Here I create a peripheral handler named `p` as follows:

```rust
 let p = embassy_stm32::init(Default::default());
```
Here we're just passing the default value for the `Config` type.

2️⃣ **Obtain a handle and configure the input button**: The on-board user push button on the Nucleo-F401RE is connected to pin PC13 (Pin 13 Port C) as stated earlier. Similar to the `Output`, there exists an `Input` driver with a `new` constructor with the following signature:

```rust
pub fn new(pin: impl Peripheral<P = T> + 'd, pull: Pull) -> Self
```
Where `pin` here is same as before with `Output` and `pull` expects an internal resistor setting (pull-up, pull-down, or none) selected through the `Pull` enum. As such, we end up with the following line of code:

```rust
let button = Input::new(p.PC13, Pull::None);
``` 

3️⃣ **Attach the interrupt to the input button**: 
At this point, the `button` instance is just an input pin. In order to make the device respond to push button interrupts, `button` needs to be attached to one of the device's external interrupt inputs. The embassy-stm32 crate has an `exti` external interrupt module that has an `EXTI` [external interrupt input driver](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/exti/struct.ExtiInput.html). Like `Input` and `Output`, `EXTI` also has a `new` constructor function that has the following signature:

```rust
pub fn new(
    pin: Input<'d, T>,
    _ch: impl Peripheral<P = T::ExtiChannel> + 'd
) -> Self
```
`pin` expects an `Input` instance and `ch` expects an `ExtiChannel` which is one of the `Peripheral` exti channels. In this case, `ch` would be the exti channel that attaches to PC13 which is EXTI13. This results in the following line of code:

```rust
let mut button = ExtiInput::new(button, p.EXTI13);
```

4️⃣ **Publish Delay Value and Spawn LED Blinking Task**: before entering the button press loop, we're going to need to kick off our LED blinking task. Though ahead of that, we need to provide an initial value for the delay and update it to the global context as follows: 

```rust
    let mut del_var = 2000;
    BLINK_MS.store(del_var, Ordering::Relaxed);
```
Then `led_task` can be kicked off using the `spawn` method as follows:

```rust
    spawner.spawn(led_task(p.PA5.degrade())).unwrap();
```
Note that `degrade()` is a method that, well, "degrades" the pin type to `AnyPin` that the task parameter requires as observed earlier.

Next, we can move on to the application Loop.

#### 🔁 Main Task Loop

Following the design described earlier in the state machine, the button press will be the only event that is interrupt-based. However, using embassy, events from interrupts or otherwise are managed through `Futures`. This means that execution can be yielded to allow other code to progress until the event attached to a `Future` occurs. There is an executor that manages all of this in the background and more detail about it is provided [in the documentation](https://embassy.dev/dev/runtime.html). 

Given that, in the main task loop all we need to do is wait for a button press to occur. This is done through the following line: 

```rust
button.wait_for_rising_edge().await;
```

`wait_for_rising_edge` is an [`ExtiInput` method](https://docs.embassy.dev/embassy-stm32/git/stm32f030c6/exti/struct.ExtiInput.html) that returns a `Future` as well. This is a `Future` that is attached to the interrupt event of a button press. This can be read as the task yielding execution to the executor until a rising edge occurs on the button. 

This means that once a button press occurs, the code following the above line will execute. The following code adjusts the delay value and updates the global context in the following lines:

```rust
del_var = del_var - 300_u32;
   if del_var < 500_u32 {
       del_var = 2000_u32;
   }
BLINK_MS.store(del_var, Ordering::Relaxed);
```

This concludes the code for the full application.

## 📱 Full Application Code
Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev git repo](https://github.com/apollolabsdev/stm32-nucleo-f401re).


```rust
#![no_std]
#![no_main]
#![feature(type_alias_impl_trait)]

use core::sync::atomic::{AtomicU32, Ordering};
use embassy_executor::Spawner;
use embassy_stm32::exti::ExtiInput;
use embassy_stm32::gpio::{AnyPin, Input, Level, Output, Pin, Pull, Speed};
use embassy_time::{Duration, Timer};
use panic_halt as _;

static BLINK_MS: AtomicU32 = AtomicU32::new(0);

#[embassy_executor::task]
async fn led_task(led: AnyPin) {
    // Configure the LED pin as a push pull ouput and obtain handler.
    // On the Nucleo FR401 theres an on-board LED connected to pin PA5.
    let mut led = Output::new(led, Level::Low, Speed::Low);

    loop {
        let del = BLINK_MS.load(Ordering::Relaxed);
        Timer::after(Duration::from_millis(del.into())).await;
        led.toggle();
    }
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    // Initialize and create handle for devicer peripherals
    let p = embassy_stm32::init(Default::default());

    // Configure the button pin (if needed) and obtain handler.
    // On the Nucleo FR401 there is a button connected to pin PC13.
    let button = Input::new(p.PC13, Pull::None);
    let mut button = ExtiInput::new(button, p.EXTI13);

    // Create and initialize a delay variable to manage delay loop
    let mut del_var = 2000;

    // Publish blink duration value to global context
    BLINK_MS.store(del_var, Ordering::Relaxed);

    // Spawn LED blinking task
    spawner.spawn(led_task(p.PA5.degrade())).unwrap();

    loop {
        // Check if button got pressed
        button.wait_for_rising_edge().await;

        // If button pressed decrease the delay value
        del_var = del_var - 300;
        // If updated delay value drops below 300 then reset it back to starting value
        if del_var < 500 {
            del_var = 2000;
        }
        // Publish updated delay value to global context
        BLINK_MS.store(del_var, Ordering::Relaxed);
    }
}

``` 

## 👨‍⚖️ The Verdict

From a first look, what is probably already obvious is that the code is much less verbose. It takes much less code to achieve the same thing in embassy. Tasks we'd do in non-embassy HALs including taking and configuring the peripherals, setting up the system clocks, and promoting the pac-level structures are all done in a single line here. Attaching timers are also a breeze. There are no ISRs, no interrupt enabling, or NVIC configuration. Finally, when it comes to using timers, the built-in timer abstraction provides for a really convenient way to set up delays.

In summary, from a first look, and aside from `async`, embassy seems to provide a smoother approach and frankly lower entry barrier to embedded Rust than what I've encountered thus far. Granted that I haven't delved too deep into embassy yet, it took me a while to acquaint myself with thinking `async`. I figure as I experiment further I am going to encounter more challenges. Nevertheless, embassy still seems rough around the edges as it continues to develop towards a more stable release. I found various gaps in the documentation where I needed to ask to get an answer, setting up the environment was a bit of a hassle, and I did have issues trying to debug. At times, although I got some application code to work just fine, during debug I saw strange behavior that didn't really add up. I figure embassy still has some work to be done to prove its viability from various aspects so it can become a mainstream choice.

## Conclusion
In this post, a GPIO interrupt-based LED control application was created for the STM32F401RE microcontroller on the Nucleo-F401RE development board. All code was created leveraging the embassy framework HAL for stm32. It can be seen that leveraging embassy provides for a smoother sail compared to existing approaches. Still, embassy is not yet in a phase as stable as existing solutions. Additionally, embassy leverages Rust's async framework which implies that embedded application programmers need to adjust their thinking approach. Have any questions/comments? Share your thoughts in the comments below 👇. 

%%[subend]