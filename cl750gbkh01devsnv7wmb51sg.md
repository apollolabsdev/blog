---
title: "STM32F4 Embedded Rust at the HAL: GPIO Interrupts"
datePublished: Mon Aug 22 2022 17:06:54 GMT+0000 (Coordinated Universal Time)
cuid: cl750gbkh01devsnv7wmb51sg
slug: stm32f4-embedded-rust-at-the-hal-gpio-interrupts
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1661186862942/rWQ6KcHlu.png
tags: tutorial, beginners, rust, embedded

---

> This blog post is the first one of a three-part series of posts where I explore interrupts for the STM32F401RE microcontroller using embedded Rust at the HAL level.

%%[substart]

## Introduction
Dealing with interrupts on its own from an embedded microcontroller perspective is more complex than polled code. As if that weren't enough, I would say the use of Rust adds another level of complexity. This is understandable because interrupts are not safe by definition since they introduce race conditions. As a result, Rust being Rust, adds abstractions to make interrupt operations safe and prevent these race conditions. These additional abstractions might not be easy to digest for a beginner. Still, the good news is that there is an alternative out there to make using Rust with interrupts smoother. This alternative is the Real-Time Interrupt-driven Concurrency (RTIC) framework which I will be tackling it in a future post.

In this post, I will be refactoring the [GPIO button-controlled blinking](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking) application I created before and transforming it to use interrupts. I will be sticking strictly to the HAL. Along the way, I will try my best to explain the elements introduced by Rust. Additionally, I will be evolving the application over three separate posts, finishing up with the RTIC, which I will explain in more detail in the software design section.

### 📚 Knowledge Pre-requisites
To understand the content of this post, you need the following:
- Basic knowledge of coding in Rust.
- Familiarity with the basic template for creating embedded applications in Rust.
- Familiarity with interrupts in Cortex-M processors.

### 💾 Software Setup
All the code presented in this post in addition to instructions for the environment and toolchain setup are available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

### 🛠 Hardware Setup
#### Materials
- [Nucleo-F401RE board](https://amzn.to/3yn6AIb)

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779074203/HjlMouvt1.jpg align="center")

#### 🔌 Connections
There will be no need for external connections. On-board connections will be utilized and the include the following:
- LED is connected to pin PA5 on the microcontroller. The pin will be used as an output.
- User button connected to pin PC13 on the microcontroller. The pin will be used as an input.

## 👨‍🎨 Software Design
In the application developed in this post, I want to cycle through several LED blinking frequencies based on a button press. Meaning, that every time I press the on-board button, I want to see the LED turning on and off at a different rate. In this section, I will focus on the design of the application algorithm itself rather than the configuration aspects. For algorithm design purposes, I will assume that interrupts are configured already. I will be detailing the configuration steps in the implementation section.

I've decided it would be best to represent the algorithm using a state machine. Here's one possible approach:


![Algorithmic State Machine](https://cdn.hashnode.com/res/hashnode/image/upload/v1660913041210/D-OU4bg3J.png align="center")

**📝 Note**

> Although I use state machine representation. I will not be encoding any states in the code. Additionally, the representation assumes that configuration is already completed.

Let's analyze what is happening here. The application will start in a state where the LED is turned on. In this state, a timer would also be started. While in the "LED on" state two events could happen to make the system transition to another state; either a button press or a timer delay expiration. If the delay expires the application would transition to an "LED off" state that turns off the LED and resets the delay timer. On the other hand, if a button press is detected the application would transition to a "Button Pressed" state. The "Button Pressed" state would in turn adjust the amount of timer delay and then return to the state it transitioned from. The "LED off" state has exactly the same transition conditions as the "LED on" state. 

In essence, interrupts are software routines that are triggered by hardware events. As such, this application can be programmed to run entirely on interrupts. This means that both hardware events causing transitions (button press and timer expiry) can be configured with interrupts. However, I won't be doing that all at once. Instead, I will be doing two separate steps, each in a separate post, as follows:

1. Configure the application to use GPIO interrupts for the button press (this post).
2. Integrate interrupts for timer delay event (next post).

The idea behind the separation is not to introduce much all at once. Especially since interrupt configuration methods for GPIO are slightly different than the ones for timers. Additionally, as an extra step, in a third post, I will also transfer the whole application to the RTIC framework. This will show how it would be smoother (and less verbose) to use the RTIC framework instead if one would want to integrate interrupts in their application.

Let's now jump into implementing this algorithm.

## 👨‍💻 Code Implementation

Before jumping in, I'd like to provide some context on what needs to be done for configuring interrupts in hardware and software. The hardware part is more or less the same across controllers that use a certain architecture (ex. ARM). For ARM-based controllers (the STM32F401RE being one) typically the following steps need to be done:
- Enable global interrupts at the Cortex-M processor level (enabled by default).
- Enable interrupts at the nested vectored interrupt controller (NVIC) level.
- Configure and enable interrupts at the peripheral level (Ex. GPIO or ADC).

After that, one would need to define an Interrupt Service Routine (ISR) in the application code. As one would expect, the ISR contains the code executed in response to a specific interrupt event. Additionally, inside the ISR, it is typical that one would use values that are shared with the `main` routine. Also in the ISR, one would have to clear the hardware pending interrupt flag to allow consecutive interrupts to happen. This is a bit of a challenge in Rust for two reasons; First, to clear the pending flag one would need to access the peripheral through its handle. This is an issue because if you recall, Rust follows a singleton pattern and we cannot have more than one reference to a peripheral. Second, in Rust, global mutable variables, rightly so, are considered unsafe to read or write. This is because without taking special care, a race condition might be triggered. To solve both challenges, in Rust, global mutable data and peripherals need to be wrapped in safe abstractions that allow them to be shared between the ISR and the main thread. Enough explanations, lets move on to the actual code.

### 📥 Crate Imports 
In this implementation the crates required are as follows:
- The `core` crate to import the `Cell` and `RefCell` pointer constructs.
- The `cortex_m` crate to import the `Mutex` construct.
- The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
- The `panic_halt` crate to define the panicking behavior to halt on panic.
- The `stm32f4xx_hal` crate to import the STMicro STM32F4 series microcontrollers device hardware abstractions on top of the peripheral access API.

```rust
use core::cell::{Cell, RefCell};
use cortex_m::interrupt::Mutex;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    gpio::{self, Edge, Input},
    pac::{self, interrupt},
    prelude::*,
};
```

#### 🌍 Global Variables 

In the application at hand, I'm choosing to enable interrupts for the GPIO peripheral to detect a button press. As such, I would need to create a global shared variable to access the GPIO peripheral (remember singleton pattern). This is because I would need to subsequently disable the interrupt pending flag in the ISR. In particular, I will be using `PC13` as the GPIO input pin that I want to enable interrupts for. For convenience, I first create an alias for the `PC13` pin as follows:

```rust
type ButtonPin = gpio::PC13<Input>;
```
Next I create a `static` global variable called `G_BUTTON` wrapping `ButtonPin` in a safe abstraction as follows:

```rust
static G_BUTTON: Mutex<RefCell<Option<ButtonPin>>> = Mutex::new(RefCell::new(None));
```
So here the peripheral is wrapped in an `Option` that is wrapped in a `RefCell`, that is wrapped in a `Mutex`. The `Mutex` makes sure that the peripheral can be safely shared among threads. It would require that we use a critical section to be able to access the peripheral. The `RefCell` is used to be able obtain a mutable reference to the peripheral. Finally, the `Option` is used to allow for lazy initialization as one would not be able to initialize the variable until later (after I configure the GPIO button).

Next I create a global variable `G_DELAYMS` to carry the delay value that I'm going to pass around for determining the blinking delay. This is a variable that will be used by the main thread and modified by the ISR. I initialize the delay to `2000_u32` which will correspond to two seconds.

```rust
static G_DELAYMS: Mutex<Cell<u32>> = Mutex::new(Cell::new(2000_u32));
```

Note two differences here from the peripheral global variable. First, I am not using an `Option` since I am directly initializing with a value. Second, I am using a `Cell` rather than a `RefCell`. This is because `Cell` only permits taking a copy of the current value or replacing it, not taking a reference which is not needed with a `u32`. 

**📝 Notes:**
> 1️⃣ These global variables can be viewed as entities that exist in a global context where access is obtained at runtime by the thread that needs them. This is why `RefCell` is needed. Compared to a `Box`, `RefCell` allows for checking during runtime that only one mutable reference exists to a variable. A `Box` makes sure statically at compile time that only one mutable reference exists. Obviously `Box` cannot be used with interrupts as multiple threads would require mutable access during runtime. 

> 2️⃣ I would strongly recommend referring to [Chapter 6 of the Embedded Rust Book](https://docs.rust-embedded.org/book/concurrency/index.html) for more detail on this topic.

### 🎛 Peripheral Configuration Code

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to take the PAC level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let mut dp = pac::Peripherals::take().unwrap();
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
let mut button = gpioc.pc13;
``` 

#### ⏱ Timer Delay Configuration:

1️⃣ **Configure the system clocks**: The system clocks need to be configured as they are needed in setting up the timer peripheral. To set up the system clocks we need to first promote the RCC struct from the PAC and constrain it using the `constrain()` method (more detail on the `constrain` method [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)) to give use access to the `cfgr` struct. After that, we create a `clocks` handle that provides access to the configured (and frozen) system clocks. The clocks are configured to use an HSE frequency of 8MHz by applying the `use_hse()` method to the `cfgr` struct. The HSE frequency is defined by the reference manual of the Nucleo-F401RE development board. Finally, the `freeze()` method is applied to the `cfgr` struct to freeze the clock configuration. Note that freezing the clocks is a protection mechanism by the HAL to avoid the clock configuration changing during runtime. It follows that the peripherals that require clock information would only accept a [frozen `Clocks` configuration struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/rcc/struct.Clocks.html). 

```rust
let rcc = dp.RCC.constrain();
let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
```

2️⃣ **Obtain a handle for the delay**: I'll be using `TIM1` to create a delay handle as follows:

```rust
let mut delay = dp.TIM1.delay_ms(&clocks);
```

####  ⏸ Interrupt Configuration Code

The last thing that remains in the configuration is to configure and enable interrupt operation for the GPIO button peripheral. This is so that when a button is pressed, execution switches over to the interrupt service routine. As mentioned earlier we need to configure the hardware in three steps:

1️⃣ **Enable global interrupts at the Cortex-M processor level**: Cortex-M processors have an architectural register named PRIMASK that contains a bit to enable/disable interrupts globally. Note that interrupts are globally enabled by default in the Cortex-M PRIMASK register. Technically nothing needs to be done here from a code perspective, however I like to include this step for awareness. 

2️⃣ **Set up and enable interrupts at the peripheral level (Ex. GPIO or ADC)**: To set up the interrupts for GPIO we need access to the `SYSCFG` struct. In order to be able to use the `SYSCFG` struct, we first need to first promote it to the HAL level by constraining it using the `constrain()` method. How did I know that I need to `constrain` `SYSCFG`? I recommend you refer to [this](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods) post for more detail. `SYSCFG` contains the information for a set of registers in the STM32 that are needed to configure interrupts for peripherals and promoted as follows: 

```rust
    let mut syscfg = dp.SYSCFG.constrain();
```
Now the gpio button needs to be configured as an interrupt source in the STM32 device. In the STM32, GPIO interrupts are considered a type of external interrupt. In the stm32f4xx-hal documentation for GPIO I've found a bunch of external pin [`ExtiPin` traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/trait.ExtiPin.html) that would allow me to configure GPIO pin interrupts. As such, the GPIO pin is set up first by using the `make_interrupt_source` method to make it an interrupt source in the STM32 device. The `make_interrupt_source` method has the below signature, of which all it needs is a mutable reference to `SysCfg`.

```rust
fn make_interrupt_source(&mut self, syscfg: &mut SysCfg)
```
Next, we would need to determine how the interrupt triggers (Ex. on a rising, falling, or both edges). Under the same list of [`ExtiPin` traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/trait.ExtiPin.html), there is a `trigger_on_edge` method with the following signature:

```rust 
fn trigger_on_edge(&mut self, exti: &mut EXTI, level: Edge)
```
The method requires two parameters, a mutable reference to the pac external interrupt/event controller `EXTI`, and the interrupt trigger level. `Edge` is an enumeration with the different types of interrupt triggers for GPIO and looks as follows:

```rust
pub enum Edge {
    Rising,
    Falling,
    RisingFalling,
}
```
Finally, one more step remains which is enabling the interrupt at the peripheral level. This is done via the `enable_interrupt` method: 

```rust
fn enable_interrupt(&mut self, exti: &mut EXTI)
```
Here the method only requires a mutable reference to the pac external interrupt/event controller `EXTI`.

Using the above methods, the GPIO button pin can now be configured for an interrupt as follows:

```rust
    button.make_interrupt_source(&mut syscfg);
    button.trigger_on_edge(&mut dp.EXTI, Edge::Rising);
    button.enable_interrupt(&mut dp.EXTI);
```

**3️⃣ Enable interrupt source in the NVIC**: Now that the button interrupt is configured, the corresponding interrupt number in the NVIC needs to be unmasked. This is done using the NVIC `unmask` method in the `cotrex_m::peripheral` crate. Also it must be noted that unmasking interrupts in the NVIC is considered `unsafe` in Rust. However, in this case it's fine since we know that we aren't doing any `unsafe` behavior. The `unmask` method expects that we pass it the number for the interrupt that we want to unmask. This could be done by applying the generic `Pin` `interrupt` method on the `button` handle as follows:

```rust
    unsafe {
        cortex_m::peripheral::NVIC::unmask(button.interrupt());
    }
```
An alternative approach is to leverage the [`interrupt` enum](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/enum.interrupt.html) that enumerates all the interrupts in the stm32f4xx-hal. You would need to find out what the button interrupt source name is in the STM32 by looking into the reference manual. It would turn out that the interrupt name is `EXTI15_10`. As such, the interrupt can be also unmasked in the NVIC as follows:

```rust
    unsafe {
        cortex_m::peripheral::NVIC::unmask(stm32f4xx_hal::interrupt::EXTI15_10);
    }
```

**🚨 Important Note:**
> During the implementation of this project I kept getting a `the trait 'Nr' is not implemented for 'stm32f4xx_hal:: interrupt` type error from the compiler for the above line.  It turns out that the crate cortex-m 0.7 started using trait `InterruptNumber` for interrupts instead of `Nr` from `bare-metal`. To get rid of this issue, you need to make sure that the version of the `cortex-m` the hal you are using imports is the same as the one in your `cargo.toml`.

**4️⃣ Move Button to Global Context**: Recall how earlier a global variable `G_BUTTON` was introduced to move around the GPIO peripheral between contexts. However, `G_BUTTON` was initialized with `None` pending the configuration of the GPIO button that is now available. This means that we can now move `button` to the global context in which it can be shared by threads. This is done as follows:

```rust
    cortex_m::interrupt::free(|cs| {
        G_BUTTON.borrow(cs).replace(Some(button));
    });
```
Here we are introducing a critical section of code enclosed in the closure `cortex_m::interrupt::free`. In this critical section of code, preemption from interrupts is disabled to ensure that the accessing the global variable does not introduce any race conditions. This is required because `G_BUTTON` is wrapped in a `Mutex`. The closure passes a token `cs` that allows us to `borrow` a mutable reference the global variable and replace the `Option` inside of with `Some(button)`.

Note that from this point on in code, every time we want to access `G_BUTTON` (or any other `Mutex` global variable we would need to introduce a critical section using `cortex_m::interrupt::free`. 

This was a lot, though its over and we can move on to the application code.

### 📱 Application Code 

#### 🔁 Application Loop

Following the design described earlier in the state machine, the button press will be the only event that is interrupt based. Meaning that the delay expiry events will be managed in the application loop. Consequently, in the application loop, all I will be doing is switching between the "LED on" and "LED off" states with a delay in between them. However, the delay value would depend on the value in `G_DELAYMS` that exists in the global context. `G_DELAYMS` is modified by the ISR overtime a button press is detected. 

In the application loop, I switch between the "LED on" and "LED off" states using the `set_high()` and `set_low()` `Pin` methods. In between the "LED on" and "LED off" states, a delay is required which I used the `delay` handle I created earlier. In order to obtain access to `G_DELAYMS` I would need to introduce a critical section as done earlier since it's wrapped in a `Mutex`. Inside the critical section, I obtain access by borrowing `G_DELAYMS` using the `borrow` method and then get its value using the `get` method. The application loop code looks as follows:

```rust
     loop {
        led.set_high();
        delay.delay_ms(cortex_m::interrupt::free(|cs| G_DELAYMS.borrow(cs).get()));
        
        led.set_low();
        delay.delay_ms(cortex_m::interrupt::free(|cs| G_DELAYMS.borrow(cs).get()));
    }
```
#### ⏸ Interrupt Service Routine(s)

Next, I need to setup the ISR that would include the code the executes once the interrupt is detected. To define the interrupt in Rust, first one would need to use the `#[interrupt]` attribute, followed by a function definition that has the interrupt name as an identifier. The interrupt name is obtained from the hal [documentation](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/enum.interrupt.html) and in our case for the pin PC13 its `EXTI15_10`. This looks as follows:

```rust
#[interrupt]
fn EXTI15_10() {
  // Interrupt Service Routine Code
}
```
Inside the ISR, first thing that needs to be done is modifying the `G_DELAYMS` delay. In this application, I chose to start with a delay of 2 seconds and decrease it in 0.5 second decrements. If the delay value reaches zero then I reset it again to 2 seconds. Though as before, to access `G_DELAYMS`, a critical section is needed. Additionally, I am using a new `set` method that allows me to modify the contents of `G_DELAYMS`.

```rust
    cortex_m::interrupt::free(|cs| {
        G_DELAYMS
            .borrow(cs)
            .set(G_DELAYMS.borrow(cs).get() - 500_u32);
        if G_DELAYMS.borrow(cs).get() < 500_u32 {
            G_DELAYMS.borrow(cs).set(2000_u32);
        });
```
We are not done yet, as the interrupt pending flag for the button press in the hardware is still set. If it is not reset, then we won't be able to detect any subsequent interrupts. For that I would need a reference to the GPIO button peripheral which can be provided by the `G_BUTTON` global variable created earlier. As a result, in the same critical section started earlier the following lines can be added:

```rust
        let mut button = G_BUTTON.borrow(cs).borrow_mut();
        button.as_mut().unwrap().clear_interrupt_pending_bit();
```
The first line obtains a mutable reference to the `Option` in `G_BUTTON` using the `borrow_mut` method. In the following line, the mutable reference is unwrapped, providing access to the `button` handle. Finally, the `clear_interrupt_pending_bit` method from the [`ExtiPin` traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/trait.ExtiPin.html) is applied to clear the interrupt pending flag/bit.

## 📱 Full Application Code
Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.


```rust
#![no_std]
#![no_main]

// Imports
use core::cell::{Cell, RefCell};
use cortex_m::interrupt::Mutex;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    gpio::{self, Edge, Input},
    pac::{self, interrupt},
    prelude::*,
};

// Create an alias for pin PC13
type ButtonPin = gpio::PC13<Input>;

// Global Variable Definitions
// Global variables are wrapped in safe abstractions.
// Peripherals are wrapped in a different manner than regular global mutable data.
// In the case of peripherals we must be sure only one refrence exists at a time.
// Refer to Chapter 6 of the Embedded Rust Book for more detail.

// Create a Global Variable for the GPIO Peripheral that I'm going to pass around.
static G_BUTTON: Mutex<RefCell<Option<ButtonPin>>> = Mutex::new(RefCell::new(None));
// Create a Global Variable for the delay value that I'm going to pass around for delay.
static G_DELAYMS: Mutex<Cell<u32>> = Mutex::new(Cell::new(2000_u32));

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let mut dp = pac::Peripherals::take().unwrap();

    // Configure and obtain handle for delay abstraction
    // 1) Promote RCC structure to HAL to be able to configure clocks
    let rcc = dp.RCC.constrain();
    // 2) Configure the system clocks
    // 8 MHz must be used for HSE on the Nucleo-F401RE board according to manual
    let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
    // 3) Create delay handle
    let mut delay = dp.TIM1.delay_ms(&clocks);

    // Configure the LED pin as a push pull ouput and obtain handle
    // On the Nucleo FR401 theres an on-board LED connected to pin PA5
    // 1) Promote the GPIOA PAC struct
    let gpioa = dp.GPIOA.split();
    // 2) Configure Pin and Obtain Handle
    let mut led = gpioa.pa5.into_push_pull_output();

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

    // Enable the external interrupt in the NVIC by passing the button interrupt number
    unsafe {
        cortex_m::peripheral::NVIC::unmask(button.interrupt());
    }

    // Now that button is configured, move button into global context
    cortex_m::interrupt::free(|cs| {
        G_BUTTON.borrow(cs).replace(Some(button));
    });

    // Application Loop
    loop {
        // Turn On LED
        led.set_high();
        // Obtain G_DELAYMS and delay
        delay.delay_ms(cortex_m::interrupt::free(|cs| G_DELAYMS.borrow(cs).get()));
        // Turn off LED
        led.set_low();
        // Obtain G_DELAYMS and delay
        delay.delay_ms(cortex_m::interrupt::free(|cs| G_DELAYMS.borrow(cs).get()));
    }
}

#[interrupt]
fn EXTI15_10() {
    // Start a Critical Section
    cortex_m::interrupt::free(|cs| {
        // Obtain Access to Delay Global Data and Adjust Delay
        G_DELAYMS
            .borrow(cs)
            .set(G_DELAYMS.borrow(cs).get() - 500_u32);
        if G_DELAYMS.borrow(cs).get() < 500_u32 {
            G_DELAYMS.borrow(cs).set(2000_u32);
        }
        // Obtain access to Global Button Peripheral and Clear Interrupt Pending Flag
        let mut button = G_BUTTON.borrow(cs).borrow_mut();
        button.as_mut().unwrap().clear_interrupt_pending_bit();
    });
}
``` 
 ## 🔬 Further Experimentation/Ideas
- If you have extra buttons, try implementing additional interrupts from other input pins where each button press applies a different delay. This is instead of one button applying one delay.
- A cool mini project is capturing a human response time. Using the LED and a press button, see how long it takes you to press the button after the LED turns on. You can use a counter/timer peripheral to capture duration and UART to propagate the result. Refer to past posts for dealing with the counter and UART.

## Conclusion
In this post, an interrupt-based LED control application was created leveraging the GPIO peripheral for the STM32F401RE microcontroller on the Nucleo-F401RE development board. All code was created at the HAL level using the stm32f4xx Rust HAL. It can be seen that doing interrupts in Rust can be a bit verbose for all the safe abstractions that are added. For that, one can instead resort to RTIC that can reduce much of the code and will be covered in a following post. Have any questions/comments? Share your thoughts in the comments below 👇. 

%%[subend]