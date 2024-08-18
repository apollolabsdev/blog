---
title: "STM32F4 Embedded Rust at the HAL: Timer Ultrasonic Distance Measurement"
datePublished: Tue Jul 26 2022 04:16:36 GMT+0000 (Coordinated Universal Time)
cuid: cl61o1p8r05en3lnvafkge43d
slug: stm32f4-embedded-rust-at-the-hal-timer-ultrasonic-distance-measurement
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1660760041225/TlxX6c7yw.png
tags: tutorial, beginners, rust, embedded

---

> This blog post is the fifth of a multi-part series of posts where I explore various peripherals in the STM32F401RE microcontroller using embedded Rust at the HAL level. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.

Prior posts include (in order of publishing):

1. [STM32F4 Embedded Rust at the HAL: GPIO Button Controlled Blinking](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking)
    
2. [STM32F4 Embedded Rust at the HAL: Button Controlled Blinking by Timer Polling](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling)
    
3. [STM32F4 Embedded Rust at the HAL: UART Serial Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication)
    
4. [STM32F4 Embedded Rust at the HAL: PWM Buzzer](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-pwm-buzzer)
    

%%[substart] 

## Introduction

In this post, I will be configuring and setting up stm32f4xx-hal timer and GPIO peripherals with an ultrasonic sensor to measure obstacle distance. A distance measurement will be continuously collected and sent to a PC terminal over UART. I will be leveraging the [UART Serial Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication) application from a previous post. Additionally, I will not be using any interrupts and the example will be set up as a simplex system that transmits in one direction only (towards the PC).

**🚨 Important Note:**

> For the purpose of this post, ideally I would have wanted to leverage the timer peripheral input capture mode. I came to discover later that input capture is yet not supported for the stm32f4xx-hal. As a result, I resorted to a different approach that achieves the same thing but is considered less efficient. One can still leverage input capture at the PAC level but for the purpose of this post, I wanted to stick with the HAL.

### Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with the basic template for creating embedded applications in Rust.
    
* Familiarity with [UART communication basics](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter).
    
* Familiarity with working principles of Ultrasonic sensors. [This](https://www.microcontrollertips.com/principle-applications-limitations-ultrasonic-sensors-faq/) page is a good resource.
    

### Software Setup

All the code presented in this post in addition to instructions for the environment and toolchain setup are available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

In addition to the above, you would need to install some sort of serial communication terminal on your host PC. Some recommendations include:

**For Windows**:

* [PuTTy](https://www.putty.org/)
    
* [Teraterm](https://ttssh2.osdn.jp/index.html.en)
    

**For Mac and Linux**:

* [minicom](https://wiki.emacinc.com/wiki/Getting_Started_With_Minicom)
    

Some installation instructions for the different operating systems are available in the [Discovery Book](https://docs.rust-embedded.org/discovery/microbit/06-serial-communication/index.html).

### Hardware Setup

#### Materials

* [Nucleo-F401RE board](https://amzn.to/3yn6AIb)
    

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779686047/l7QDlfxSv.jpg align="center")

* Seeed Studio [Grove Base Shield V2.0](https://www.seeedstudio.com/base-shield-v2.html)
    

![Base_Shield_v2-1.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779703842/yKiMqnmcx.png align="center")

* Seeed Studio [Grove Ultrasonic Distance Sensor](https://www.seeedstudio.com/grove-ultrasonic-distance-sensor.html). The module uses the [NU40C16T/R-1 Ultrasonic Sensor](https://files.seeedstudio.com/wiki/Grove_Ultrasonic_Ranger/res/NU40C16T-R-1.pdf).
    

![ultrasonicgrove.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779793153/gG-cdzdP-.jpg align="center")

**🚨 Important Note:**

> I used the Grove modular system for connection ease. It is a more elegant approach and less prone to mistakes. One can directly wire the ultrasonic sensor to the board if need be.

#### Connections

* Ultrasonic echo pin connected to pin PA8 (Grove Connector D7).
    
* The UART Tx line that connects to the PC through the onboard USB bridge is via pin PA2 on the microcontroller. This is a hardwired pin, meaning you cannot use any other for this setup. Unless you are using a different board other than the Nucleo-F401RE, you have to check the relevant documentation (reference manual or datasheet) to determine the number of the pin.
    

## Software Design

The ultrasonic sensor used is a single-pin interface sensor. The single pin, referred to as the echo pin, operates in a bidirectional mode. The echo pin, first operating as an input, should be triggered by a pulse that is at least 10us wide. This would cause the sensor to emit a series of ultrasonic pulses that it measures the propagation delay of. After that, the echo pin switches to an output providing a pulse width proportional to the distance of the obstacle.

![Ultrasonic Pulse](https://cdn.hashnode.com/res/hashnode/image/upload/v1657901598049/axfh0Ir4B.png align="center")

The obstacle distance is calculated as: distance(cm)=echo pulse width(us)29∗2

The algorithm is quite straightforward in this case. After configuring the device, the algorithmic steps are as follows:

1. Set PA8 pin output to low for 5 us to get a clean low pulse
    
2. Set PA8 pin output to high (trigger) for 10us
    
3. Switch PA8 to an input
    
4. Keep polling PA8 input until it goes high
    
5. Once PA8 input goes high kick-off counter/timer
    
6. Keep polling PA8 input until it goes low
    
7. Obtain pulse duration measurement from counter/timer
    
8. Calculate distance and send the result to UART serial channel
    
9. Go back to 1
    

## Code Implementation

### Crate Imports

In this implementation, the following crates are required:

* The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
    
* The `core::fmt` crate will allow us to use the `writeln!` macro for easy printing.
    
* The `panic_halt` crate to define the panicking behavior to halt on panic.
    
* The `stm32f4xx_hal` crate to import the STMicro STM32F4 series microcontrollers device hardware abstractions on top of the peripheral access API.
    

```rust
use core::fmt::Write;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    gpio::PinState,
    pac::{self},
    prelude::*,
    serial::config::Config,
};
```

### Peripheral Configuration Code

#### GPIO Peripheral Configuration:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to take the PAC level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let dp = pac::Peripherals::take().unwrap();
```

2️⃣ **Promote the PAC-level GPIO structs**: I need to configure the echo pin as input in the beginning and obtain a handler for the pin so that I can control it. I also need to obtain a handle for the UART Tx pin. Both pins are part of `GPIOA`. Before I can obtain any handles I need to promote the pac-level `GPIOA` struct to be able to create handles for individual pins. I do this by using the `split()` method as follows: `rust let gpioa = dp.GPIOA.split();` 3️⃣ **Obtain a handle for the echo pin and configure it to an input**: As earlier stated, the echo pin is connected to pin PA8 (Pin 8 Port A). As such, I need to create a handle for the echo pin that has PA8 configured to an input. I will name the handle `echo` and configure it as follows: `rust let mut echo = gpioa.pa8;` **📝 Note:**

> For more detail on GPIO control, please refer to my past post [GPIO Button Controlled Blinking](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking).

#### Serial Communication Peripheral Configuration:

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

3️⃣ **Configure the serial peripheral channel**: Looking into the [Nucleo-F401RE board pinout](https://os.mbed.com/platforms/ST-Nucleo-F401RE/), the Tx line pin PA2 connects to the USART2 peripheral in the microcontroller device. As such, this means we need to configure USART2 and somehow pass it to the handle of the pin we want to use. This is done as follows:

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

**📝 Note:**

> More detail on UART setup is available in the [UART Serial Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication) blog post.

#### Timer and Delay Peripheral Configuration:

In the algorithm, there is a step where I have to provide a pulse trigger that is 10us wide. For that, I would need to use some delay method to keep the echo pin high for that duration. Additionally, in another step, I have to also use a timer to determine how long the pulse width is. For that, I need to configure two peripherals as follows:

1️⃣ **Configure a timer for delay and obtain handle**: I will be using `TIM1` to provide a blocking delay. I will call the handle `delay` and create it as follows:

```rust
let mut delay = dp.TIM1.delay_us(&clocks);
```

2️⃣ **Configure a timer for pulse measurement and obtain handle**: I will be using `TIM2` to provide a counter I can leverage to obtain a `Duration`. I will call the handle `counter` and create it as follows:

```rust
let mut counter = dp.TIM2.counter_us(&clocks);
```

**📝 Note:**

> More detail on timers/counters and their setup is available in the [Button Controlled Blinking by Timer Polling](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling) blog post.

This is it for configuration! Let's now jump into the application code.

### Application Code

Following the design described earlier, I first need to set the `echo` pin output to low for 5 us to get a clean low pulse. The issue now is that the `echo` pin is configured as an input. As a result, if one would examine the generic `Pin` [methods](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/struct.Pin.html), there is a `with_push_pull_output_in_state` method that, according to its description, temporarily configures a pin as a push-pull output and has the following signature:

```rust
pub fn with_push_pull_output_in_state<R>(
    &mut self,
    state: PinState,
    f: impl FnOnce(&mut Pin<P, N, Output<PushPull>>) -> R
) -> R
```

Note here that the method has a closure `f` that is called with the reconfigured pin. After the closure returns, the pin will be configured back to its original configuration. Addtionally, the method has a `state` parameter that allows me to assign a certain state to the output pin (high or low) when its reconfigured. As such, I can achieve what I want as follows:

```rust
echo.with_push_pull_output_in_state(PinState::Low, |_f| delay.delay_us(5_u32));
```

what is happening here is that the `echo` pin is reconfigured to push pull output, with the output being low. In the closure, I am introducing the 5us delay using the `delay` handle. This means that the pin is going to remain as an output in the low state for 5us, and then return to being an input again.

Steps 2 and 3 in the algorithm require that I set the `echo` pin output to high (trigger) for 10us and then switch `echo` back to an input. This can be done exactly in the same manner as the previous step as follows:

```rust
echo.with_push_pull_output_in_state(PinState::High, |_f| delay.delay_us(10_u32));
```

The main differences here is that the `state` argument is `High` and that the closure has a 10us delay instead.

Next I need to keep polling the `echo` pin until it goes high marking the start of the echo pulse. This is done as follows:

```rust
while !(echo.is_high()) {}
```

Using the `while` loop and the `is_high` `Pin` method, the code is going around this same line until the `echo` pin input goes high.

Afterwards a timer needs to be kicked-off. Using the `counter` handle created earlier and the `start` `Counter` [method](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/timer/counter/struct.Counter.html) the counter is kicked-off as follows:

```rust
counter.start(1000.millis()).unwrap();
```

Here a `timeout` `Duration` is provided as an argument which presents the maximum duration the counter would run for. The `start` method also returns a `Result` which is why I had to `unwrap` it. I chose a duration of `1000` milliseconds as it corresponds to the longest distance that can be measured.

Now that the timer is kicked off, next step requires that I keep polling the `echo` pin input until it goes low. This is done exactly as before but rather using the `is_low` method instead as follows:

```rust
while !(echo.is_low()) {}
```

Once the `echo` pin goes low, the pulse duration measurement needs to be collected by the counter/timer as follows: `rust let duration = counter.now().duration_since_epoch(); counter.cancel().unwrap();` Here the `now` method is leveraged to obtain the current `Instance` and the `duration_since_epoch` method to provide back the a `Duration` value. I am also cancelling/stopping the timer using the `cancel` `Counter` method and unwrapping it.

**📝 Note:**

> Again, if any clarity is lacking relative to counter methods I would recommend referring to the [Button Controlled Blinking by Timer Polling](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling) blog post as it digs into more detail.

Now that the pulse duration is available, a distance can be calculated. Using the earlier presented formula, the distance in centimeters is calculated using the following code:

```rust
let distance_cm = duration.to_micros() / 2 / 29;
```

The `to_micros` [method](https://docs.rs/fugit/0.3.5/fugit/struct.Duration.html#method.to_micros) converts the `Duration` to an integer number of microseconds.

Finally, the result is sent over UART using the `writeln!` macro:

```rust
writeln!(tx, "Distance {:02} cm\r", distance_cm).unwrap();
```

If you have noticed, `writeln!` takes three parameters and in the first parameter of `writeln!`, I am passing the `tx` serial handler as an argument. Additionally, the `writeln!` macro needs to be unwrapped since it returns a `Result`. The third parameter of `writeln!` also contains the `distance_cm` variable that was created in the previous line to store the result of the distance calculation.

## Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.

```rust
#![no_std]
#![no_main]

// Imports
use core::fmt::Write; // allows use to use the WriteLn! macro for easy printing
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    gpio::PinState,
    pac::{self},
    prelude::*,
    serial::config::Config,
};

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let dp = pac::Peripherals::take().unwrap();

    // Configure the ultasonic device echo pin as input and obtain handler.
    let gpioa = dp.GPIOA.split();
    let mut echo = gpioa.pa8;

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

    // Delay Configuration
    // Set up a microsecond delay handler
    let mut delay = dp.TIM1.delay_us(&clocks);

    // Counter/timer congig
    // Set up a microsecond counter handler
    let mut counter = dp.TIM2.counter_us(&clocks);

    // Algorithim
    // 1) Set pin ouput to low for 5 us to get clean low pulse
    // 2) Set pin output to high (trigger) for 10us
    // 3) Switch back to input
    // 4) Keep checking if pin goes high
    // 5) Once pin goes high start kick off counter/timer
    // 6) Wait for Pin to go low
    // 7) Obtain pulse measurement from timer
    // 8) Print out measurement on Serial
    // 9) Go back to 1)

    // Application Loop
    loop {
        // 1) Set pin ouput to low for 5 us to get clean low pulse
        echo.with_push_pull_output_in_state(PinState::Low, |_f| delay.delay_us(5_u32));

        // 2) Set pin output to high (trigger) for 10us
        // 3) Switch back to input
        echo.with_push_pull_output_in_state(PinState::High, |_f| delay.delay_us(10_u32));

        // 4) Wait until pin goes high
        while !(echo.is_high()) {}

        // 5) Kick off timer measurement with a max timeout Duration of 100ms?? defined by data sheet (longest distance that can be measured)
        counter.start(1000.millis()).unwrap();

        // 6) Wait until pin goes low.
        while !(echo.is_low()) {}

        // 7) Stop timer and collect elapsed time
        let duration = counter.now().duration_since_epoch();
        counter.cancel().unwrap();

        // 8) Calculate the distance in cms using formula in datasheet
        let distance_cm = duration.to_micros() / 2 / 29;

        // 8) Send calculated distance to serial interface
        writeln!(tx, "Distance {:02} cm\r", distance_cm).unwrap();
    }
}
```

## Conclusion

In this post, an ultrasonic distance measurement application was created leveraging the GPIO and Counter peripherals for the STM32F401RE microcontroller on the Nucleo-F401RE development board. The resulting measurement is also sent over to a host PC over a UART connection. All code was based on polling (without interrupts). Additionally, all code was created at the HAL level using the [stm32f4xx Rust HAL](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/index.html). Have any questions? Share your thoughts in the comments below 👇.

%%[subend]