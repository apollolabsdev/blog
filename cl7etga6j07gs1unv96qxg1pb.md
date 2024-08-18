---
title: "STM32F4 Embedded Rust at the HAL: Timer Interrupts"
datePublished: Mon Aug 29 2022 13:48:37 GMT+0000 (Coordinated Universal Time)
cuid: cl7etga6j07gs1unv96qxg1pb
slug: stm32f4-embedded-rust-at-the-hal-timer-interrupts
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1661576204878/t8trIHqDL.png
tags: tutorial, beginner, rust, embedded

---

> This blog post is the second of a three-part series of posts where I explore interrupts for the STM32F401RE microcontroller using embedded Rust at the HAL level.

As a word of note, this post heavily depends on the previous post, [STM32F4 Embedded Rust at the HAL: GPIO Interrupts](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts). Thus I recommend that the reader refers to the previous post before reading this one.

%%[substart]

## Introduction
In the previous post, [STM32F4 Embedded Rust at the HAL: GPIO Interrupts](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts), I've implemented the button controlled blinking led application using interrupts. In that code, only the button event was configured for interrupts. However, there was a second event that was possible to encode as well, which is the timer delay. When the timer delay expires, we get an interrupt that we need to respond to. For those familiar with past posts and how we've done delay, a blocking `delay_ms` method was used. Using `delay_ms` essentially halts all operations in the controller until the delay expires. One can imagine how inefficient this kind of operation can be. Instead, with interrupts, the timer can be triggered to count while the controller goes about doing other things. The controller would only respond when the timer expires. 

This post will mostly focus on the code implementation as many of the other sections are identical to the ones in [STM32F4 Embedded Rust at the HAL: GPIO Interrupts](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts).

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
Here I want to refer quickly to the state machine I presented in the past post. Recall that the button press event/transition was configured with interrupts in the past post. The idea here is to integrate interrupts for timer delay events as well, thus having the application run completely based on interrupts. This means that two ISRs will be needed instead; one for GPIO and one for Timer, combined with a main loop that does nothing. 

![Algorithmic State Machine](https://cdn.hashnode.com/res/hashnode/image/upload/v1660913041210/D-OU4bg3J.png align="center")

**📝 Note**

> Applications that are completely driven by interrupts are ones that can provide a basic hardware scheduling environment. Typically a low-power application is one that would benefit from such an architecture. Essentially, when the application is idle it would go into low power sleep mode using the `wfi` or `wfe` instructions and wake up to process only when events occur.

Let's now jump into implementing this algorithm.

## 👨‍💻 Code Implementation

### 📥 Crate Imports 
In this implementation, the crate imports are more or less the same as before except that I added the timer types needed from the `stm32f4xx_hal`.

```rust
use core::cell::{Cell, RefCell};
use cortex_m::interrupt::Mutex;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    gpio::{self, Edge, Input, Output, PushPull},
    pac::{self, interrupt, TIM2},
    prelude::*,
    timer::{CounterMs, Event},
};
```

#### 🌍 Global Variables 

Remember that [before](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts) I created two global variables wrapped in safe abstractions, one for the button to pass around the GPIO `button` handle and another for the delay value. I'm going to keep the previous global variables but I'm going to add an additional two to handle the LED GPIO output and the Timer peripheral. This is because I need to control the LED from the timer ISR and additionally clear the timer interrupt flag also in the timer ISR.

Again here, for convenience, I create another alias for the `PA5` pin that is connected to the LED as follows:

```rust
type LedPin = gpio::PA5<Output<PushPull>>;
```
Next, exactly as before, I create additional two `static` global variables called `G_LED` and `G_TIMER` wrapping the led pin `LedPin` and millisecond timer (I chose TIM2) `CounterMs<TIM2>` in safe abstractions as follows:

```rust
static G_TIM: Mutex<RefCell<Option<CounterMs<TIM2>>>> = Mutex::new(RefCell::new(None));

static G_LED: Mutex<RefCell<Option<LedPin>>> = Mutex::new(RefCell::new(None));
```

### 🎛 Peripheral Configuration Code

**Obtain a handle for the device peripherals**: This is the first step that we always need before configuring anything, taking the PAC-level device peripherals:

```rust
let mut dp = pac::Peripherals::take().unwrap();
```
####  🕹 GPIO Configuration:

**📝 Note**

> GPIO pin and interrupt configuration in this post are exactly the same as in the previous post. I am keeping it to make the configuration code part of this post as self-contained as possible.

1️⃣ **Promote the PAC-level GPIO structs**: We need to configure the LED pin as a push-pull output and obtain a handler for the pin so that we can control it. We also need to obtain a handle for the button input pin. According to the connection details, the LED pin connection is part of `GPIOA` and the button connection is part of `GPIOC`. Before we can obtain any handles for the LED and the button we need to promote the pac-level `GPIOA` and `GPIOC` structs to be able to create handles for individual pins. We do this by using the `split()` method as follows: 
```rust
let gpioa = dp.GPIOA.split();
let gpioc = dp.GPIOC.split();
```
2️⃣ **Obtain a handle for the LED and configure it to an output**: As earlier stated, the on-board LED on the Nucleo-F401RE is connected to pin PA5 (Pin 5 Port A). As such, we need to create a handle for the LED pin that has PA5 configured to a push-pull output using the `into_push_pull_output()` method. We will name the handle `led` and configure it as follows: 
```rust
let mut led = gpioa.pa5.into_push_pull_output();
```
For those interested, [this](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/struct.Pin.html) HAL documentation page has the full list of methods that the `Pin` type supports. Also, if you find the `split()` method confusing, please refer to my blog post [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods) for more detail.

3️⃣ **Obtain a handle and configure the input button**: The on-board user push button on the Nucleo-F401RE is connected to pin PC13 (Pin 13 Port C) as stated earlier. Pins are configured to an input by default so when creating the handle for the button we don't call any special methods.
```rust
let mut button = gpioc.pc13;
``` 

#### ⏱ ⏸ Timer Configuration:

1️⃣ **Configure the system clocks**: The system clocks need to be configured as they are needed in setting up the timer peripheral. To set up the system clocks we need to first promote the RCC struct from the PAC and constrain it using the `constrain()` method (more detail on the `constrain` method [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)) to give use access to the `cfgr` struct. After that, we create a `clocks` handle that provides access to the configured (and frozen) system clocks. The clocks are configured to use an HSE frequency of 8MHz by applying the `use_hse()` method to the `cfgr` struct. The HSE frequency is defined by the reference manual of the Nucleo-F401RE development board. Finally, the `freeze()` method is applied to the `cfgr` struct to freeze the clock configuration. Note that freezing the clocks is a protection mechanism by the HAL to avoid the clock configuration changing during runtime. It follows that the peripherals that require clock information would only accept a [frozen `Clocks` configuration struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/rcc/struct.Clocks.html). 

```rust
let rcc = dp.RCC.constrain();
let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
```

2️⃣ **Obtain a handle for the timer**: I'll be using `TIM2` to create a timer handle, I also use the `counter_ms` implementation for convenience as it would allow me to provide a millisecond value directly. 

```rust
let mut timer = dp.TIM2.counter_ms(&clocks);
```

The GPIO and Timer are both now configured though not set up for interrupts yet, which is the next step.

####  ⏸ Interrupt Configuration:

The last thing that remains in the configuration is to configure and enable interrupt operation for the GPIO button and timer peripherals. This is so that when a button is pressed or the timer expires, execution switches over to the corresponding interrupt service routine. As mentioned before we need to configure the hardware in three steps:

1️⃣ **Enable global interrupts at the Cortex-M processor level**: Cortex-M processors have an architectural register named PRIMASK that contains a bit to enable/disable interrupts globally. Note that interrupts are globally enabled by default in the Cortex-M PRIMASK register. Nothing needs to be done here from a code perspective, however, I like to include this step for awareness. 

2️⃣ **Set up and enable interrupts at the peripheral level (Ex. GPIO and Timer)**: To set up the interrupts for GPIO we need access to the `SYSCFG` struct. In order to be able to use the `SYSCFG` struct, we first need to first promote it to the HAL level by constraining it using the `constrain()` method. How did I know that I need to `constrain` `SYSCFG`? I recommend you refer to [this](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods) post for more detail. `SYSCFG` contains the information for a set of registers in the STM32 that are needed to configure interrupts for peripherals and is promoted as follows: 

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

As for the timer, first, we need to initialize the timer with a timeout value and start the counting. This is done using the `start` method, I chose a timeout of 2 seconds:

```rust
timer.start(2000.millis()).unwrap();
```
Note that above I am using the `millis` method that I used in the past [Button Controlled Blinking by Timer Polling](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling) post. As a reminder, what `millis` achieves is converting a `u32` to a `Duration` because that is what the `start` method requires.

Second, just like GPIO, interrupts need to be enabled at the peripheral level. The method that does that for the timers is called `listen`. The method takes one parameter which is an `Event` enum that defines the type of event the timer generates an interrupt for. These events are ones defined by the controller datasheet. Here I chose an `Update` event:

```rust
 timer.listen(Event::Update);
```

**3️⃣ Enable interrupt source in the NVIC**: Now that the button and timer interrupts are configured, the corresponding interrupt numbers in the NVIC needs to be unmasked. This is done using the NVIC `unmask` method in the `cotrex_m::peripheral` crate. As a reminder, the `unmask` method expects that we pass it the number for the interrupt that we want to unmask. This has been done by applying the generic `Pin` `interrupt` method on the `button` handle. However, for the timer, I couldn't locate in the documentation a method that returns the interrupt number for the timer. Though, I mentioned in the past [post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts) that there is an alternative approach leveraging the [`interrupt` enum](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/enum.interrupt.html) that enumerates all interrupt sources. For the timer, the interrupt source is `TIM2` and the NVIC unmasking can be done as as follows:

```rust
    unsafe {
        cortex_m::peripheral::NVIC::unmask(interrupt::TIM2);
        cortex_m::peripheral::NVIC::unmask(button.interrupt());
    }
```

**4️⃣ Move Variable to Global Context**: I initialized three global variables in the beginning with `None`, pending initialization. Those were `G_BUTTON`, `G_TIM`, and `G_LED`. Now that all associated peripherals have been initialized, they all need to be moved to the global context. This is can be done in one critical section as follows:

```rust
    cortex_m::interrupt::free(|cs| {
        G_TIM.borrow(cs).replace(Some(timer));
        G_BUTTON.borrow(cs).replace(Some(button));
        G_LED.borrow(cs).replace(Some(led));
    });
```

Were done with the configuration! Over to the application code!

### 📱 Application Code 

#### 🔁 Application Loop

Following the design described earlier, there is no application code! Essentially all we need to do is set up an empty loop that goes on forever until it is preempted by one of our interrupts. However, I do something slightly different like this:

```rust
    loop {
        cortex_m::asm::wfi();
    }
```

I am calling the `wfi` instruction from the cortex_m library. `wfi` stands for wait for interrupt and what it does is send the processor to sleep while it's sitting idle. The processor would then wake up when any of our interrupts occur. This is a power saving mechanism, alternatively, I could have kept the `loop` empty.

#### ⏸ Interrupt Service Routine(s)

Here I need to set up the ISRs that would include the code that executes once any of the interrupts is detected. Here we're going to have two ISRs, one for the timer and one for the button. As before, we would have to use the `#[interrupt]` attribute, followed by a function name matching the interrupt name. Again, the interrupt name is obtained from the hal [documentation](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/enum.interrupt.html), and in our case for the pin PC13 its `EXTI15_10` and for the timer its `TIM2`. 

**1️⃣ Button Press ISR**
Inside the button press ISR, the first thing that needs to be done is to modify the `G_DELAYMS` delay. Similar to the previous application, I chose to start with a delay of 2 seconds and decrease it in 0.5-second decrements. If the delay value reaches zero then I reset it again to 2 seconds. Though as before, to access `G_DELAYMS`, a critical section is needed. Additionally, I am using a new `set` method that allows me to modify the contents of `G_DELAYMS`.

```rust
    cortex_m::interrupt::free(|cs| {
        G_DELAYMS
            .borrow(cs)
            .set(G_DELAYMS.borrow(cs).get() - 500_u32);
        if G_DELAYMS.borrow(cs).get() < 500_u32 {
            G_DELAYMS.borrow(cs).set(2000_u32);
        });
```

Now that the delay is different, the change needs to be propagated to the timer so that it starts counting with the new delay (that's the point of the button press after all). We'll have to use the `start` method again, but remember the timer is also in a global context wrapped in the `G_TIM` variable. 

```rust
        let mut timer = G_TIM.borrow(cs).borrow_mut();

        timer
            .as_mut()
            .unwrap()
            .start(G_DELAYMS.borrow(cs).get().millis())
            .unwrap();
```

Finally, as the interrupt pending flag for the button press in the hardware is still set. If it is not reset, then we won't be able to detect any subsequent interrupts. For that I would need a reference to the GPIO button peripheral which can be provided by the `G_BUTTON` global variable created earlier. As a result, in the same critical section started earlier the following lines can be added:

```rust
        let mut button = G_BUTTON.borrow(cs).borrow_mut();
        button.as_mut().unwrap().clear_interrupt_pending_bit();
```
The first line obtains a mutable reference to the `Option` in `G_BUTTON` using the `borrow_mut` method. In the following line, the mutable reference is unwrapped, providing access to the `button` handle. Finally, the `clear_interrupt_pending_bit` method from the [`ExtiPin` traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/trait.ExtiPin.html) is applied to clear the interrupt pending flag/bit.

**2️⃣ Timer ISR**
In the timer ISR only two things need to be done. The first is toggling the LED since the timer expired and the second is clearing the timer interrupt pending flag. This also needs to be done in a critical section since we'll need access to the global `G_LED` and `G_TIM` variables. For the timer, the interrupt flag is cleared using a `clear_interrupt` method, that also requires the type of `Event` we are clearing.

 ```rust
    cortex_m::interrupt::free(|cs| {
        let mut led = G_LED.borrow(cs).borrow_mut();
        led.as_mut().unwrap().toggle();
      
        let mut timer = G_TIM.borrow(cs).borrow_mut();
        timer.as_mut().unwrap().clear_interrupt(Event::Update);
    });
```

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
    gpio::{self, Edge, Input, Output, PushPull},
    pac::{self, interrupt, TIM2},
    prelude::*,
    timer::{CounterMs, Event},
};

// Create aliases for pins PC13 and PA5
type ButtonPin = gpio::PC13<Input>;
type LedPin = gpio::PA5<Output<PushPull>>;

// Global Variable Definitions
// Global variables are wrapped in safe abstractions.
// If you notice peripherals are wrapped in a different manner than regular global mutable data.
// In the case of peripherals we must be sure only one refrence exists at a time.
// Refer to Chapter 6 of the Embedded Rust Book for more detail.

// Create a Global Variable for the Button GPIO Peripheral that I'm going to pass around.
static G_BUTTON: Mutex<RefCell<Option<ButtonPin>>> = Mutex::new(RefCell::new(None));
// Create a Global Variable for the Timer Peripheral that I'm going to pass around.
static G_TIM: Mutex<RefCell<Option<CounterMs<TIM2>>>> = Mutex::new(RefCell::new(None));
// Create a Global Variable for the LED GPIO Peripheral that I'm going to pass around.
static G_LED: Mutex<RefCell<Option<LedPin>>> = Mutex::new(RefCell::new(None));
// Create a Global Variable for the delay value that I'm going to use to manage the delay.
static G_DELAYMS: Mutex<Cell<u32>> = Mutex::new(Cell::new(2000));

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let mut dp = pac::Peripherals::take().unwrap();

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

    // Enable the external interrupt in the NVIC for all peripherals by passing the interrupt numbers
    unsafe {
        cortex_m::peripheral::NVIC::unmask(interrupt::TIM2);
        cortex_m::peripheral::NVIC::unmask(button.interrupt());
    }

    // Now that all peripherals are configured, move them into global context
    cortex_m::interrupt::free(|cs| {
        G_TIM.borrow(cs).replace(Some(timer));
        G_BUTTON.borrow(cs).replace(Some(button));
        G_LED.borrow(cs).replace(Some(led));
    });

    // Application Loop
    loop {
        // Go to sleep
        cortex_m::asm::wfi();
    }
}

// Button Interrupt
#[interrupt]
fn EXTI15_10() {
    // When Button interrupt happens three things need to be done
    // 1) Adjust Global Delay Variable
    // 2) Update Timer with new Global Delay value
    // 3) Clear Button Pending Interrupt

    // Start a Critical Section
    cortex_m::interrupt::free(|cs| {
        // Obtain Access to Delay Global Data and Adjust Delay
        G_DELAYMS
            .borrow(cs)
            .set(G_DELAYMS.borrow(cs).get() - 500_u32);

        // Reset delay value if it drops below 500 milliseconds
        if G_DELAYMS.borrow(cs).get() < 500_u32 {
            G_DELAYMS.borrow(cs).set(2000_u32);
        }

        // Obtain access to global timer
        let mut timer = G_TIM.borrow(cs).borrow_mut();

        // Adjust and start timer with updated delay value
        timer
            .as_mut()
            .unwrap()
            .start(G_DELAYMS.borrow(cs).get().millis())
            .unwrap();

        // Obtain access to Global Button Peripheral and Clear Interrupt Pending Flag
        let mut button = G_BUTTON.borrow(cs).borrow_mut();
        button.as_mut().unwrap().clear_interrupt_pending_bit();
    });
}

// Timer Interrupt
#[interrupt]
fn TIM2() {
    // When Timer Interrupt Happens Two Things Need to be Done
    // 1) Toggle the LED
    // 2) Clear Timer Pending Interrupt

    // Start a Critical Section
    cortex_m::interrupt::free(|cs| {
        // Obtain Access to Delay Global Data and Adjust Delay
        let mut led = G_LED.borrow(cs).borrow_mut();
        led.as_mut().unwrap().toggle();

        // Obtain access to Global Timer Peripheral and Clear Interrupt Pending Flag
        let mut timer = G_TIM.borrow(cs).borrow_mut();
        timer.as_mut().unwrap().clear_interrupt(Event::Update);
    });
}
``` 
 ## 🔬 Further Experimentation/Ideas
Here the experimentation ideas remain the same, though can be adapted to the new structure.
- If you have extra buttons, try implementing additional interrupts from other input pins where each button press applies a different delay. This is instead of one button applying one delay.
- A cool mini project is capturing a human response time. Using the LED and a press button, see how long it takes you to press the button after the LED turns on. You can use a counter/timer peripheral to capture duration and UART to propagate the result. Refer to past posts for dealing with the counter and UART.

## Conclusion
In this post, an interrupt-based LED control application was created leveraging the GPIO and Timer peripherals in the STM32F401RE microcontroller on the Nucleo-F401RE development board. All code was created at the HAL level using the stm32f4xx Rust HAL. As opposed to the prior [post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-interrupts), the application is completely driven by interrupts. It can be seen that doing interrupts in Rust can be a bit verbose for all the safe abstractions that are added. In the following post, I will be porting the above code to the RTIC framework that can reduce much of the code. Have any questions/comments? Share your thoughts in the comments below 👇.  

%%[subend]