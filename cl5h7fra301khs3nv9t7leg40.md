---
title: "STM32F4 Embedded Rust at the HAL: UART Serial Communication"
datePublished: Mon Jul 11 2022 20:36:15 GMT+0000 (Coordinated Universal Time)
cuid: cl5h7fra301khs3nv9t7leg40
slug: stm32f4-embedded-rust-at-the-hal-uart-serial-communication
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1660759112467/nPcRhcdHu.png
tags: tutorial, iot, rust, embedded

---

> This blog post is the third one of a multi-part series of posts where I explore various peripherals in the STM32F401RE microcontroller using embedded Rust at the HAL level. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.

Prior posts include (in order of publishing):
1. [STM32F4 Embedded Rust at the HAL: GPIO Button Controlled Blinking](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking)
2. [STM32F4 Embedded Rust at the HAL: Button Controlled Blinking by Timer Polling](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling)

%%[substart]

## Introduction
Setting up UART serial communication is useful for any type of device-to-device (point-to-point) communication. One of the common past use cases for UART was in development to print output to a PC. However, for that particular use case, nowadays some microcontrollers have advanced debug features like instruction trace macrocell (aka ITM) that don't leverage the device's own peripheral resources. Obviously, this does not make UART obsolete, as it has other use cases and some controllers don't support advanced debug to start with. In this post, I will be configuring and setting up UART communication with a PC terminal for the Nucleo-F401RE board. I will be leveraging the [GPIO button-controlled blinking project](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking) from a previous post to print to the console how many times the button has been pressed. Additionally, I will not be using interrupts and the example will be set up as a simplex system that transmits in one direction only (towards the PC).

### Knowledge Pre-requisites
To understand the content of this post, you need the following:
- Basic knowledge of coding in Rust.
- Familiarity with the basic template for creating embedded applications in Rust.
- Familiarity with [UART communication basics](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter). 

### Software Setup
All the code presented in this post in addition to instructions for the environment and toolchain setup are available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

In addition to the above, you would need to install some sort of serial communication terminal on your host PC. Some recommendations include:

**For Windows**:
- [PuTTy](https://www.putty.org/)
- [Teraterm](https://ttssh2.osdn.jp/index.html.en)

**For Mac and Linux**:
- [minicom](https://wiki.emacinc.com/wiki/Getting_Started_With_Minicom)

Some installation instructions for the different operating systems are available in the [Discovery Book](https://docs.rust-embedded.org/discovery/microbit/06-serial-communication/index.html).

### Hardware Setup
#### Materials
- [Nucleo-F401RE board](https://amzn.to/3yn6AIb)

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779179125/kB0juI3OD.jpg align="center")

#### Connections
There will be no need for external connections. On-board Nucleo-F401RE connections will be utilized and include the following:
- LED is connected to pin PA5 on the microcontroller. The pin will be used as an output.
- User button connected to pin PC13 on the microcontroller. The pin will be used as an input.
- The UART Tx line that connects to the PC through the onboard USB bridge is via pin PA2 on the microcontroller. This is a hardwired pin, meaning you cannot use any other for this setup. Unless you are using a different board other than the Nucleo-F401RE, you have to check the relevant documentation (reference manual or datasheet) to determine the number of the pin.

## Software Design
The application in this post adopts the same algorithmic approach as my [previous post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking). Here, however, I will be adding two things:

1. I will be leveraging the [debouncr](https://docs.rs/debouncr/latest/debouncr/) crate (yes, that's how the crate name is spelled 😀) to eliminate the effect of button debouncing.
2. I will be sending/printing a value that tracks the number of times the button is pressed to the PC terminal.

The updated flow diagram would look as follows:

![Flow Chart](https://cdn.hashnode.com/res/hashnode/image/upload/v1657001444071/is1tg2bU-.png align="center")

## Code Implementation

### Crate Imports
In this implementation, the following crates are required:
- The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
- The `core::fmt` crate will allow us to use the `writeln!` macro for easy printing.
- The `debouncr` crate to debounce the button press.
- The `panic_halt` crate to define the panicking behavior to halt on panic.
- The `stm32f4xx_hal` crate to import the STMicro STM32F4 series microcontrollers device hardware abstractions on top of the peripheral access API.

```rust
use core::fmt::Write; 
use cortex_m_rt::entry;
use debouncr::{debounce_3, Edge};
use panic_halt as _;
use stm32f4xx_hal::{
    pac::{self},
    prelude::*,
    serial::{Config, Serial},
};
```

### Peripheral Configuration Code

#### LED & Button GPIO Configuration:

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

#### Serial Communication Configuration:

1️⃣ **Configure the system clocks**: The system clocks need to be configured as they are needed in setting up the UART peripheral. To set up the system clocks we need to first promote the RCC struct from the PAC and constrain it using the `constrain()` method (more detail on the `constrain` method [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)) to give use access to the `cfgr` struct. After that, we create a `clocks` handle that provides access to the configured (and frozen) system clocks. The clocks are configured to use an HSE frequency of 8MHz by applying the `use_hse()` method to the `cfgr` struct. The HSE frequency is defined by the reference manual of the Nucleo-F401RE development board. Finally, the `freeze()` method is applied to the `cfgr` struct to freeze the clock configuration. Note that freezing the clocks is a protection mechanism by the HAL to avoid the clock configuration changing during runtime. It follows that the peripherals that require clock information would only accept a [frozen `Clocks` configuration struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/rcc/struct.Clocks.html). 

```rust
let rcc = dp.RCC.constrain();
let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
```
**🚨 Important Note:**
> Using a frequency different than 8 MHz for HSE on the Nucleo-F401RE board will cause the UART to output erroneous characters. This value needs to be adjusted to what the individual board settings are.

2️⃣ **Obtain a handle and configure the serial transmit (Tx) pin**: Since the Tx button is `PA2`, earlier I had already created a handle for `gpioa` that I have to leverage. However, now that we are not using the pin as a regular GPIO input or output it means that the pin needs to be connected to a different peripheral internal to the microcontroller. The pin can be configured as such using the `into_alternate()` method as follows.

```rust    
let tx_pin = gpioa.pa2.into_alternate();
```
3️⃣ **Configure the serial peripheral channel**: Looking into the [Nucleo-F401RE board pinout](https://os.mbed.com/platforms/ST-Nucleo-F401RE/), the Tx line pin PA2 connects to the USART2 peripheral in the microcontroller device. As such, this means we need to configure USART2 and somehow pass it to the handle of the pin we want to use. To configure/instantiate the serial peripheral channel we have two options. The first is to use the device peripheral handle `dp` to directly access USART2 and instantiate a transmitter instance using the `tx` method from the [serial extension traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/serial/trait.SerialExt.html). The second is to use the `tx` method in the `Serial` abstraction struct to instantiate a transmitter instance. Note that both are different ways of doing exactly the same thing!

For the first option, if we examine the `tx` method signature in the [serial extension traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/serial/trait.SerialExt.html), it looks like this:

```rust
fn tx<TX, WORD>(
    self,
    tx_pin: TX,
    config: impl Into<Config>,
    clocks: &Clocks
) -> Result<Tx<Self, WORD>, InvalidConfig>
```
The method takes three parameters, a pin instance, a configuration, and a frozen `Clocks` instance reference. As such, we can create a handle `tx` for the UART transmitter as follows:

```rust
    let mut tx = dp
        .USART2
        .tx(
            tx_pin,
            Config::default()
                .baudrate(115200.bps())
                .wordlength_8()
                .parity_none(),
            &clocks,
        )
        .unwrap();
```
`tx_pin` and `clocks` are the handles that we created earlier. `Config` is a type [struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/serial/config/struct.Config.html) that contains the configuration information needed for configuring the UART peripheral. Here I am creating an instance of `Config` with the `default` trait first to configure default parameters. After that, I apply the `baudrate`, `wordlength_8`, and `parity_none` methods to configure the UART peripheral to the settings I need. A full list of `Config` methods can be found [here](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/serial/config/struct.Config.html). I configured the UART settings as shown to 115200 bps baud with 8 bits of data, and no parity, also commonly referred to as 8N1. Finally, since the `tx` method returns a result, we would have to unwrap it using the `unwrap` method.

**🚨 Important Note:**
> To figure out what the default configuration in the `Config` struct entailed, I had to always go into the source code of the `default` trait implementation. Unfortunately, the HAL documentation itself does not make it easily obvious what the default configuration is. 

Alternatively, the second option using the `Serial` abstraction looks like this:

```rust
let mut tx = Serial::tx(
      dp.USART2,
      tx_pin,
      Config::default()
                .baudrate(115200.bps())
                .wordlength_8()
                .parity_none(),
      &clocks,
      )
    .unwrap();
```
You can see that the main difference here is that `tx` is applied as an instance method on the `Serial` struct. It can be observed here that the tx method also accepts a fourth parameter here which is an instance of the UART peripheral USART2. This can be observed in the signature of the `tx` instance method in the documentation which looks as follows:

```rust
pub fn tx(
    usart: USART,
    tx_pin: TX,
    config: impl Into<Config>,
    clocks: &Clocks
) -> Result<Tx<USART, WORD>, InvalidConfig>
```
Note that in both cases, we used a `tx` method. This is because earlier I mentioned that I will be doing simplex communication. Meaning that I will be transmitting in one direction only. If you delve into the documentation, you'll find that there are also methods to support receive only and also bidirectional transmit and receive.

This is it for configuration! Let's now jump into the application code.

### Application Code

Following the design described earlier, I first need to initialize a delay variable `del_var` and initialize the output of the LED. `del_var` needs to be mutable as it will be modified by the delay loop. I also choose to set the initial `led` output level to low by default. Using the same `Pin` methods mentioned [earlier](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/struct.Pin.html), there is a `set_low()` method that I use to achieve that.
```rust
    // Create and initialize a delay variable to manage delay loop
    let mut del_var = 7_0000_i32;

    // Initialize LED to on or off
    led.set_low();
```
I also want to initialize a variable `value` that I want to use to track how many times the button has been pressed:
```rust
let mut value: u8 = 0;
```
Afterward, I have one small thing remaining. I mentioned that I will be using the `debouncr` crate to debounce button presses. This means I need to create some sort of handler to utilize the crate methods. In the crate [documentation](https://docs.rs/debouncr/latest/debouncr/), to instantiate, I first need to determine how many states need to be detected. In a way, it sort of means how many state changes you want to see before a press is determined? Truth be told I experimented a bit with this and found that 3 states was the most suitable for my application.

```rust
    let mut debouncer = debounce_3(false);
```
The reason I initialized `debouncer` to `false` is that the documentation mentioned that I have to do that if the button is active low. 

Next inside the program loop, I first start by calling a delay function where inside of it I check if the button is pressed. After the delay completes, I toggle led using the toggle() method, again part of the methods available for the Pin type. This is the complete application loop:

```rust
    loop {
        for _i in 1..del_var {
            if debouncer.update(button.is_low()) == Some(Edge::Falling) {
                writeln!(tx, "Button Press {:02}\r", value).unwrap();
                value = value.wrapping_add(1);
                del_var = del_var - 3_0000_i32;
                if del_var < 1_0000 {
                    del_var = 7_0000_i32;
                }
                break;
            }
        }
        led.toggle();
    }
```
The general structure is exactly the same as the application loop in the regular polled [button-controlled blinking application](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking). The outer `for` loop is the one that keeps track of the delay through`del_var`. A difference here is the `if` statement condition. For the condition, I am leveraging the `update` method from the `debouncr` crate. When polling the pin, the `update` method is repeatedly called. The `update` method returns an `Option` enum that keeps providing a `None` when no press is detected. As soon as a (debounced) press is detected, the `update` method returns either a `Some(Edge::Falling)` or `Some(Edge::Rising)`. Since the Nucleo-F401RE baord has an active-low button, a press is detected with a falling edge and a `Some(Edge::Falling)` is returned on a successful debounce. 

Once the button detect is completed, I use the `writeln!` macro provided through `core::fmt::Write` that I imported earlier. The usage is exactly the same as the formatted print using `println!` in Rust with a couple of small exceptions. Examining the statement,

```rust
writeln!(tx, "Button Press {:02}\r", value).unwrap();
```

If you have noticed, `writeln!` takes three parameters and in the first parameter of `writeln!`, I am passing the `tx` serial handler as an argument. Additionally, the `writeln!` macro needs to be unwrapped since it returns a `Result`. The third parameter of `writeln!` also contains the `value` variable initialized earlier that is being incremented by the following line as follows:
```rust
value = value.wrapping_add(1);
```
The `wrapping_add` method, as the name implies performs a wrapping add on `value` adding `1` every time the method is called and wraps around if needed. The remaining code takes care of decreasing the value of `del_var` to reduce the delay and make sure that it does not drop below zero. Finally, outside of the delay loop the led is toggled using the `Pin` `toggle` method.

## Full Application Code
Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.


```rust
#![no_std]
#![no_main]

// Imports
use core::fmt::Write; // allows use to use the WriteLn! macro for easy printing
use cortex_m_rt::entry;
use debouncr::{debounce_3, Edge};
use panic_halt as _;
use stm32f4xx_hal::{
    pac::{self},
    prelude::*,
    serial::{Config, Serial},
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

    // Serial config steps:
    // 1) Need to configure the system clocks
    // - Promote RCC structure to HAL to be able to configure clocks
    let rcc = dp.RCC.constrain();
    // - Configure system clocks
    // 8 MHz must be used for the Nucleo-F401RE board according to manual
    let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();

    // 2) Configure/Define TX pin
    // Note that we already split port A earlier for the led pin
    // Use PA2 as it is connected to the host serial interface
    let tx_pin = gpioa.pa2.into_alternate();

    // 3) Configure Serial perihperal channel
    // We're going to use USART2 since its pins are the ones connected to the USART host interface
    // To configure/instantiate serial peripheral channel we have two options:
    // Use the device peripheral handle to directly access USART2 and instantiate a transmitter instance
    let mut tx = dp
        .USART2
        .tx(
            tx_pin,
            Config::default()
                .baudrate(115200.bps())
                .wordlength_8()
                .parity_none(),
            &clocks,
        )
        .unwrap();
    // or
    // Use the Serial abstraction to instantiate a transmitter instance
    // let mut tx = Serial::tx(
    //     dp.USART2,
    //     tx_pin,
    //     Config::default()
    //         .baudrate(115200.bps())
    //         .wordlength_8()
    //         .parity_none(),
    //     &clocks,
    // )
    // .unwrap();

    // Create and initialize a delay variable to manage delay loop
    let mut del_var = 7_0000_i32;

    // Initialize LED to on or off
    led.set_low();

    // Initialize debouncer to false because button is active low
    // Chose 3 consecutive states based on testing
    let mut debouncer = debounce_3(false);

    // Variable to keep track of how many button presses occured
    let mut value: u8 = 0;

    // Application Loop
    loop {
        // Enter Delay Loop
        for _i in 1..del_var {
            // Keep checking if button got pressed
            if debouncer.update(button.is_low()) == Some(Edge::Falling) {
                // If button is pressed print to derial and decrease the delay value
                writeln!(tx, "Button Press {:02}\r", value).unwrap();
                // Increment value keeping track of button presses
                value = value.wrapping_add(1);
                // Decrement the amount of delay
                del_var = del_var - 3_0000_i32;
                // If updated delay value drops below threshold then reset it back to starting value
                if del_var < 1_0000 {
                    del_var = 7_0000_i32;
                }
                // Exit delay loop since button was pressed
                break;
            }
        }

        // Delay loop without button debouncing
        // for _i in 1..del_var {
        //     // Keep checking if button got pressed
        //     if button.is_low() {
        //         // If button is pressed print to derial and decrease the delay value
        //         writeln!(tx, "Button Pressed\r").unwrap();
        //         del_var = del_var - 3_0000_i32;
        //         // If updated delay value drops below threshold then reset it back to starting value
        //         if del_var < 1_0000 {
        //             del_var = 7_0000_i32;
        //         }
        //         // Exit loop
        //         break;
        //     }
        // }

        // Toggle LED
        led.toggle();
    }
}
``` 

## Further Experimentation/Ideas
- Notice how for led pin `pa5` we've always separated the configuration of the pin as an output using `into_push_pull_output` from the code to set its initial state to low using `set_low`. Check out the [documentation](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/struct.Pin.html) for the generic `Pin` type and see if there is a method that does both in one shot.
- Instead of doing only transmission, experiment with receive as well. Instead of having a button press to control the speed, replace it with a message received over UART from a PC terminal.

## Conclusion
In this post, an LED control application was created leveraging the GPIO and UART peripherals for the STM32F401RE microcontroller on the Nucleo-F401RE development board. The UART peripheral sends to a host PC a status update every time a GPIO button is pressed. All code was based on polling (without interrupts). All code was created at the HAL level using the [stm32f4xx Rust HAL](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/index.html). Have any questions? Share your thoughts in the comments below 👇. 

%%[subend]