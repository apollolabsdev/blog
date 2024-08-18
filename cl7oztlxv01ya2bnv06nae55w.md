---
title: "STM32F4 Embedded Rust at the HAL: The RTIC Framework"
datePublished: Mon Sep 05 2022 16:44:38 GMT+0000 (Coordinated Universal Time)
cuid: cl7oztlxv01ya2bnv06nae55w
slug: stm32f4-embedded-rust-at-the-hal-the-rtic-framework
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1661699421665/1YwayD0z8.png
tags: tutorial, beginners, rust, embedded

---

> This blog post is the third of a three-part series of posts where I explore interrupts for the STM32F401RE microcontroller using embedded Rust at the HAL level.

Past posts include (in order):
1. [STM32F4 Embedded Rust at the HAL: GPIO Interrupts](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts)
2. [STM32F4 Embedded Rust at the HAL: Timer Interrupts](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-timer-interrupts)

%%[substart]

## Introduction

In the prior two posts, I walked through creating an application, completely based on interrupts, that changes the rate of blinking for an LED based on a button press. There were two interrupt service routines, one that responded to a button press event and another that responded to a timer delay expiring. It was shown how in order to create safe operations the abstractions utilized make the code quite verbose. Gladly, there is an alternative solution that can be leveraged which is the [Real-Time Interrupt-driven Concurrency (RTIC) framework](https://rtic.rs/1/book/en/). The RTIC framework provides the same safety guarantees in the prior code though is a less verbose and more elegant solution. In this post, I will be porting the code from the latest timer interrupts post to demonstrate the ease of transferring to the RTIC framework. I will be building up the RTIC application step by step explaining how each part relates to the timer interrupt application post.

The full code is also available for reference on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.

**📝 Notes:**
> 1️⃣ The code porting the GPIO interrupts post is also provided on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.

> 2️⃣ The RTIC provides much more features other than just handling interrupts (Ex. scheduling, message passing...etc.) but they will not be addressed in this post. For example, the RTIC can provide a convenient setup for a programmer that wants to create an OS. I would recommend the interested reader to refer to the [Real-Time Interrupt-driven Concurrency (RTIC) framework](https://rtic.rs/1/book/en/) documentation for details.


## 📱 The RTIC Application

The RTIC framework structure can be viewed as a combination of specialized attributes. Each attribute can define a task, an initialization function, a timer for scheduling, local and global resources, or even the application itself. Some of the attributes can also take arguments. In the following sections, I will be breaking down the structure to the individual attributes and describe the contents of each importing our application to it. 

### The `#[app]` attribute

As a start, all RTIC applications require the `#[app]` attribute. The `#[app]` attribute also requires that we pass a mandatory `device` argument that contains the path pointing to the PAC utilized. In our case, this would be `stm32f4xx_hal::pac` and the attribute is defined as follows:

```rust
#[rtic::app(device = stm32f4xx_hal::pac, peripherals = true)]
```
Note that here that `peripherals = true` makes sure that the `device` handle/field is available for use later in our code. By default, `peripherals` is set to `true` but can be set to `false` to reduce the size of the application if one does not require access to the device peripherals. Directly under the `#[app]` attribute, we include an  `app` module, `mod app` that will encapsulate the rest of the attributes and the imports needed in our application. As such, before introducing any new attributes, right at the beginning of the `mod app` body/scope, we can simply copy over the imports from the timer interrupt post. The code under `#[rtic::app(device = stm32f4xx_hal::pac, peripherals = true)]` thus expands to the following:

```rust
#[rtic::app(device = stm32f4xx_hal::pac, peripherals = true)]
mod app {

    use stm32f4xx_hal::{
        gpio::{self, Edge, Input, Output, PushPull},
        pac::TIM2,
        prelude::*,
        timer::{self, Event},
    };
```
Now we should be ready to define the rest of our application (implementation code).

## 📚 The RTIC Application Resources
In the RTIC application, there are two types of resources, shared and local. Shared resources are the resources/variables that are going to be shared/accessed among several tasks. On the other hand, local resources are ones that can be accessed by only one task. Note that all these resources are in reference to ones that we will be initialized at the beginning of our application (under the #[init] attribute shown later) to utilize later in various tasks. This means that I don't need to assign a resource for a local variable if I were to create a variable to use locally in any of the tasks. 

In other words, what I am trying to say is that both the shared and the local resources in RTIC are global variables under our prior definition. The key difference is that shared resources are for global variables that are leveraged by several tasks. However, local resources are for global variables leveraged by one task (technically two, the `#[init]` task and one other). 

### 🗄 Shared Resources
The shared resources are defined under the `#[shared]` attribute. Inside of `#[shared]`, there is a `Shared` struct where we include handles for the types that are going to be shared.  

```rust
    #[shared]
    struct Shared {     
        timer: timer::CounterMs<TIM2>,
    }
```
Note here that `timer` is the timer peripheral that I am going to be sharing between the press button interrupt and the timer expiry interrupt. As a reminder, in the button press interrupt service routine (ISR) the timer was restarted with a new delay value. On the other hand, in the timer expiry ISR, the timer handle was needed to clear the pending interrupt in the peripheral.

### 📁 Local Resources 
Similar to the shared resources, local resources are defined under the `#[local]` attribute. Inside of `#[local]`, there is also a `Local` struct where we include handles for the types that are going to be local.  

```rust
    #[local]
    struct Local {
        delayval: u32,
        button: gpio::PC13<Input>,
        led: gpio::PA5<Output<PushPull>>,
    }
```
Where `delayval` is the delay variable that is going to be modified every time the button is pressed. `button` is the handle for the GPIO input button and is needed to clear the button press interrupt pending flag. Finally, `led` is the handle for the GPIO output led and is needed to toggle the output led. Note that all of these resources are local since they will be used **only** in the button pressed ISR.

## 🤹 The RTIC Application Tasks
### The `#[init]` Task
The `#[init]` task is the one that includes the system setup code (our configuration code) and executes after system reset. For the `#[init]` task, the `#[init]` attribute is immediately followed by an `init` function that must have the signature `fn(ctx: init::Context) -> (Shared, Local, init::Monotonics)`. In the `init` task, we are able to access the device peripherals through the `device` and `core` fields in `init::Context` that is bound to the `ctx` handle. This sort of replaces what we used to do using the `take` handle. As such, in order to be able to copy the configuration code from the previous post as is, I include the following line:

```rust
let mut dp = ctx.device;
```
This will allow me to use the `dp` handle that I was using before, replacing what we used to do using the `take` method. Before showing the full code for `init`, there is one more thing. At the end of the init task, we must return the initialized values for the system-wide `#[shared]` and `#[local]` resources, in addition to the set of initialized timers used by the application. This looks as follows:

```rust
        (
            Shared { timer },
            Local { button, led, delayval: 2000_u32 },
            init::Monotonics(),
        )
```
A couple of things to note here. First, although we are not using `Monotonics` for scheduling, they still need to be returned. Second, I am initializing `delayval` with its starting value directly in the return structure.

As such, here is the code for the `#[init]` task:

```rust
#[init]
    fn init(ctx: init::Context) -> (Shared, Local, init::Monotonics) {
        let mut dp = ctx.device;

        // Configure the LED pin as a push pull ouput and obtain handle
        // On the Nucleo FR401 theres an on-board LED connected to pin PA5
        // 1) Promote the GPIOA PAC struct
        let gpioa = dp.GPIOA.split();
        // 2) Configure Pin and Obtain Handle
        let led = gpioa.pa5.into_push_pull_output();

        // Configure the button pin as input and obtain handle
        // On the Nucleo FR401 there is a button connected to pin PC13
        // 1) Promote the GPIOC PAC struct
        let gpioc = dp.GPIOC.split();
        // 2) Configure Pin and Obtain Handle
        let mut button = gpioc.pc13;

        // Configure Button Pin for Interrupts
        // 1) Promote SYSCFG structure to HAL to be able to configure interrupts
        let mut syscfg = dp.SYSCFG.constrain();
        // 2) Make button an interrupt source
        button.make_interrupt_source(&mut syscfg);
        // 3) Make button an interrupt source
        button.trigger_on_edge(&mut dp.EXTI, Edge::Rising);
        // 4) Enable gpio interrupt for button
        button.enable_interrupt(&mut dp.EXTI);

        // Configure and obtain handle for delay abstraction
        // 1) Promote RCC structure to HAL to be able to configure clocks
        let rcc = dp.RCC.constrain();
        // 2) Configure the system clocks
        // 8 MHz must be used for HSE on the Nucleo-F401RE board according to manual
        let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
        // 3) Create delay handle
        //let mut delay = dp.TIM1.delay_ms(&clocks);
        let mut timer = dp.TIM2.counter_ms(&clocks);

        // Kick off the timer with 2 seconds timeout first
        // It probably would be better to use the global variable here but I did not to avoid the clutter of having to create a crtical section
        timer.start(2000.millis()).unwrap();

        // Set up to generate interrupt when timer expires
        timer.listen(Event::Update);

        (
            // Initialization of shared resources
            Shared { timer },
            // Initialization of task local resources
            Local {
                button,
                led,
                delayval: 2000_u32,
            },
            // Move the monotonic timer to the RTIC run-time, this enables
            // scheduling
            init::Monotonics(),
        )
    }
```
Notice how our prior code practically didn't change here, cool, huh?!

### The `#[idle]` Task
The `idle` task is also known as the background task that runs when there aren't any interrupts executing and is optional. Like before, the`#[idle]` attribute is followed immediately by an `idle` function that must have a `fn(idle::Context) -> !` signature. Recall that all we did when idle is go to sleep. As such, the `idle` task code is as follows:

```rust
    #[idle]
    fn idle(_: idle::Context) -> ! {
        loop {
            cortex_m::asm::wfi();
        }
    }
```
Here, `device` and `core` are available through the context in case either needs to be accessed. However, since I am accessing neither, I use the underscore `_` identifier since I don't need to bind `idle::Context` to anything.

### The Hardware Tasks
In our code, all that remains now is defining the hardware tasks that are bound to our interrupts. To bind an interrupt we need to use the #[task] attribute with the argument `binds = InterruptName`. As such, the task becomes the interrupt handler for the bound hardware interrupt vector. In this case for us, there are two hardware tasks, one that binds to the `EXTI15_10` interrupt name and the other that binds to the `TIM2`interrupt name. Additionally, the attribute takes as arguments the resources that the task will be using. The attribute is also followed immediately by a function that you can name that has the following signature `fn func_name (func_name::Context)`, where *func_name* is a name of your choice. As such, here is the code for the button press ISR:

```rust
#[task(binds = EXTI15_10, local = [delayval, button], shared=[timer])]
    fn button_pressed(mut ctx: button_pressed::Context) {

        // Obtain a copy of the delay value from the global context
        let mut delay = *ctx.local.delayval;

        // Adjust the amount of delay
        delay = delay - 500_u32;
        if delay < 500_u32 {
            delay = 2000_u32;
        }

        // Update delay value in global context
        *ctx.local.delayval = delay;

        // Update the timeout value in the timer peripheral
        ctx.shared
            .timer
            .lock(|tim| tim.start(delay.millis()).unwrap());

        // Obtain access to Button Peripheral and Clear Interrupt Pending Flag
        ctx.local.button.clear_interrupt_pending_bit();
    }
```
As can be seen in the line ` #[task(binds = EXTI15_10, local = [button], shared=[delayval, timer])]`, I am binding the `EXTI15_10` interrupt source to this task which has the function I am calling `button_pressed`. Additionally, I am passing as arguments the previously defined local and shared resources that the ISR is going to be using. The shared resources here are the ones that are going to be shared with the timer expiry ISR, which is only the timer peripheral `timer`. 

Next, notice the lines in which I am accessing `local` resources. This includes `ctx.local.button.clear_interrupt_pending_bit();` and `let mut delay = *ctx.local.delayval;`. Access to the local resources is obtained through `local` which is a field of the task context `ctx`. This is the same as what we had in the `init` task, we were able to access the device peripherals through the `device` and `core` fields in `init::Context` that is bound to the `ctx` handle. One more thing here is that `delayval` is dereferenced because `ctx.local.delayval` on its own is a `&mut u32` but we need the `u32` value. Also, `delayval` needs to be updated in the global context so that the value is preserved for the next time the task is called. 

In the case of `shared` resources its not as straightforward. Since the resource is shared, we need to obtain a lock in which a token is provided in a closure to write the code. This is similar to our past code, albeit mush less verbose, in which we obtained a lock using `interrupt::free`.  Again, like with `local`, we can access `timer` through the `shared` field and `ctx`. However, additionally, we need to use the `lock` method the provides access to the "locked" resource through a token in a closure. This is done in the line `ctx.shared.timer.lock(|tim| tim.start(delay.millis()).unwrap());` where a new delay value is loaded to the timer shared resource. 

Finally, all that remains is defining the timer expiry ISR. The following is the timer expiry ISR:

```rust
    #[task(binds = TIM2, local=[led], shared=[timer])]
    fn timer_expired(mut ctx: timer_expired::Context) {
        ctx.local.led.toggle();
        ctx.shared
            .timer
            .lock(|tim| tim.clear_interrupt(Event::Update));
    }
```
The idea is exactly the same as the earlier `button_pressed` ISR. All that is done here is toggle the `led` which is local to the timer expiry ISR. Additionally, note that the `shared` timer resource `timer` is being locked here as well to clear the interrupt. 

## 📱 Full Application Code
Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.

```rust
#![deny(unsafe_code)]
#![deny(warnings)]
#![no_main]
#![no_std]

use panic_halt as _;

#[rtic::app(device = stm32f4xx_hal::pac, peripherals = true)]
mod app {

    use stm32f4xx_hal::{
        gpio::{self, Edge, Input, Output, PushPull},
        pac::TIM2,
        prelude::*,
        timer::{self, Event},
    };

    // Resources shared between tasks
    #[shared]
    struct Shared {
        timer: timer::CounterMs<TIM2>,
    }

    // Local resources to specific tasks (cannot be shared)
    #[local]
    struct Local {
        button: gpio::PC13<Input>,
        led: gpio::PA5<Output<PushPull>>,
        delayval: u32,
    }

    #[init]
    fn init(ctx: init::Context) -> (Shared, Local, init::Monotonics) {
        let mut dp = ctx.device;

        // Configure the LED pin as a push pull ouput and obtain handle
        // On the Nucleo FR401 theres an on-board LED connected to pin PA5
        // 1) Promote the GPIOA PAC struct
        let gpioa = dp.GPIOA.split();
        // 2) Configure Pin and Obtain Handle
        let led = gpioa.pa5.into_push_pull_output();

        // Configure the button pin as input and obtain handle
        // On the Nucleo FR401 there is a button connected to pin PC13
        // 1) Promote the GPIOC PAC struct
        let gpioc = dp.GPIOC.split();
        // 2) Configure Pin and Obtain Handle
        let mut button = gpioc.pc13;

        // Configure Button Pin for Interrupts
        // 1) Promote SYSCFG structure to HAL to be able to configure interrupts
        let mut syscfg = dp.SYSCFG.constrain();
        // 2) Make button an interrupt source
        button.make_interrupt_source(&mut syscfg);
        // 3) Make button an interrupt source
        button.trigger_on_edge(&mut dp.EXTI, Edge::Rising);
        // 4) Enable gpio interrupt for button
        button.enable_interrupt(&mut dp.EXTI);

        // Configure and obtain handle for delay abstraction
        // 1) Promote RCC structure to HAL to be able to configure clocks
        let rcc = dp.RCC.constrain();
        // 2) Configure the system clocks
        // 8 MHz must be used for HSE on the Nucleo-F401RE board according to manual
        let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
        // 3) Create delay handle
        //let mut delay = dp.TIM1.delay_ms(&clocks);
        let mut timer = dp.TIM2.counter_ms(&clocks);

        // Kick off the timer with 2 seconds timeout first
        // It probably would be better to use the global variable here but I did not to avoid the clutter of having to create a crtical section
        timer.start(2000.millis()).unwrap();

        // Set up to generate interrupt when timer expires
        timer.listen(Event::Update);

        (
            // Initialization of shared resources
            Shared { timer },
            // Initialization of task local resources
            Local {
                button,
                led,
                delayval: 2000_u32,
            },
            // Move the monotonic timer to the RTIC run-time, this enables
            // scheduling
            init::Monotonics(),
        )
    }

    // Background task, runs whenever no other tasks are running
    #[idle]
    fn idle(_: idle::Context) -> ! {
        loop {
            // Go to sleep
            cortex_m::asm::wfi();
        }
    }

    #[task(binds = EXTI15_10, local = [delayval, button], shared=[timer])]
    fn button_pressed(mut ctx: button_pressed::Context) {
        // When Button press interrupt happens three things need to be done
        // 1) Adjust Global Delay Variable
        // 2) Update Timer with new Global Delay value
        // 3) Clear Button Pending Interrupt

        // Obtain a copy of the delay value from the global context
        let mut delay = *ctx.local.delayval;

        // Adjust the amount of delay
        delay = delay - 500_u32;
        if delay < 500_u32 {
            delay = 2000_u32;
        }
        
        // Update delay value in global context
        *ctx.local.delayval = delay;

        // Update the timeout value in the timer peripheral
        ctx.shared
            .timer
            .lock(|tim| tim.start(delay.millis()).unwrap());

        // Obtain access to Button Peripheral and Clear Interrupt Pending Flag
        ctx.local.button.clear_interrupt_pending_bit();
    }

    #[task(binds = TIM2, local=[led], shared=[timer])]
    fn timer_expired(mut ctx: timer_expired::Context) {
        // When Timer Interrupt Happens Two Things Need to be Done
        // 1) Toggle the LED
        // 2) Clear Timer Pending Interrupt

        ctx.local.led.toggle();
        ctx.shared
            .timer
            .lock(|tim| tim.clear_interrupt(Event::Update));
    }
}
```
 ## 🔬 Further Experimentation/Ideas
- Refactor this code to mimic the [GPIO interrupt application](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts) behavior that was based on GPIO interrupts. An example of the refactored code is available on the git repo. You can compare your refactored code to the one available in the repo.
- If you have extra buttons, using the RTIC try implementing additional interrupts from other input pins where each button press applies a different delay and has a different source. This means that you would have to create additional interrupt tasks. 
- Convert the [Analog Temperature Sensing Application](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-analog-temperature-sensing-using-the-adc) to an interrupt based application using the RTIC. 

## Conclusion
It was shown in prior [posts](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts), examples of Rust applications that are completely driven by interrupts. It was also shown that doing interrupts in Rust can be a bit verbose for all the safe abstractions that are added. In this post, the RTIC was leveraged instead to demonstrate the process of porting code of an existing interrupt-based application created in Rust. The post also shows the power and ease in using the RTIC where safe guarantees are provided without much code verbosity. The application was created leveraging the STM32F401RE microcontroller on the Nucleo-F401RE development board. All code was created at the HAL level using the stm32f4xx Rust HAL. Have any questions/comments? Share your thoughts in the comments below 👇.

%%[subend]