---
title: "ESP32 Embedded Rust at the HAL: PWM Buzzer"
datePublished: Fri May 12 2023 06:41:25 GMT+0000 (Coordinated Universal Time)
cuid: clhk6wz8g00110ame2uklfo43
slug: esp32-embedded-rust-at-the-hal-pwm-buzzer
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683873239086/fe328bb9-bacc-48c3-8a5e-3d76b2643487.png
tags: tutorial, rust, embedded, esp32

---

> This blog post is the fourth of a multi-part series of posts where I explore various peripherals in the ESP32C3 using embedded Rust at the HAL level. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.

Prior posts include (in order of publishing):

1. [ESP32 Embedded Rust at the HAL: GPIO Button Controlled Blinking](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-gpio-button-controlled-blinking)
    
2. [ESP32 Embedded Rust at the HAL: Button-Controlled Blinking by Timer Polling](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling)
    
3. [ESP32 Embedded Rust at the HAL: UART Serial Communication](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-uart-serial-communication)
    

%%[substart] 

## Introduction

In this post, I will be exploring the generating PWM for the ESP32C3 using the Rust [esp32c3-hal](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/index.html). I will configure and set up an LEDC peripheral to play different tones on a buzzer. The different tones will be used to generate a tune.

### Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with the basic template for creating embedded applications in Rust.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/364076171079011329).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp align="center")
    
* Piezo Buzzer/Active Buzzer
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1683799540328/31ed20da-db31-4349-90a5-11f4790f9549.jpeg align="center")

#### **🔌 Connections**

* Buzzer positive terminal connected to pin GPIO1.
    
* Buzzer negative terminal connected to GND.
    

## **👨‍🎨 Software Design**

The buzzer used is quite simple to operate. Through the buzzer-connected signal pin, various tones can be generated by the controller PWM peripheral. This occurs by changing the PWM frequency to match the needed tone. As a result, to generate a certain tune a collection of tones at a certain rate (tempo) need to be provided to the PWM peripheral. This also means that the code would need to include some data structures storing the needed information to provide to the PWM peripheral. Two data structures are needed, the first would include a mapping between notes and their associated frequencies. The second would represent a tune that includes a collection of notes each played for a certain amount of beats.

Following that information, after configuring the device, the algorithmic steps are as follows:

1. From the tune array data structure obtain a note and its associated beat
    
2. From the tones array retrieve the frequency associated with the note obtained in step 1
    
3. Play the note for the desired duration (number of beats \* tempo)
    
4. Include half a beat of silence (0 frequency) between notes
    
5. Go back to 1.
    

There are fine details in between relative to the PWM details that will be discussed in detail in the implementation.

Implementing hardware-based PWM in the ESP32C3 is a bit non-conventional. Meaning that I expected the timer peripheral to have a PWM function. ESP32s rather seem to have three types of application-driven peripherals that enable PWM implementation; the LED controller (LEDC) peripheral, the motor control (MCPWM) peripheral, and the Remote Control Peripheral (RMT). The ESP32C3 in particular does not have an MCPWM peripheral, so the choices come down to two. In this post, I use the LEDC peripheral.

📝 ***Note***

> A challenge that emerged using the LEDC is from a HAL perspective. It turns out that for now, the [esp32c3-hal](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/index.html) supports fixed-frequency output only. This means that every time the frequency needs to be changed the peripheral needs to be reconfigured. Reconfiguring the ESP32 LEDC involves several steps, and the way the code is designed some ownership issues arise in Rust. As such, making it work requires the code to become a bit verbose. The verbosity could probably be reduced by using functions but not the focus of this post.

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `esp32c3_hal` crate to import the ESP32C3 device hardware abstractions.
    
* The `esp_backtrace` crate to define the panicking behavior.
    

```rust
use esp32c3_hal::{
    clock::ClockControl,
    delay::Delay,
    ledc::{
        channel,
        timer::{self},
        LSGlobalClkSource, LowSpeed, LEDC,
    },
    peripherals::Peripherals,
    prelude::*,
    timer::TimerGroup,
    Rtc, IO,
};
use esp_backtrace as _;
```

### **🎛 Initialization (Configuration) Code**

#### ⌨️ GPIO Peripheral Configuration:

In embedded Rust, as part of the singleton design pattern, we first have to take the PAC-level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let peripherals = Peripherals::take();
```

**2️⃣ Disable the Watchdogs:** Just like earlier posts, the ESP32C3 has watchdogs enabled by default and they need to be disabled. If they are not disabled then the device would keep on resetting. To avoid this issue, the following code needs to be included:

```rust
let system = peripherals.SYSTEM.split();
let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

// Instantiate and Create Handles for the RTC and TIMG watchdog timers
let mut rtc = Rtc::new(peripherals.RTC_CNTL);
let timer_group0 = TimerGroup::new(peripherals.TIMG0, &clocks);
let mut wdt0 = timer_group0.wdt;
let timer_group1 = TimerGroup::new(peripherals.TIMG1, &clocks);
let mut wdt1 = timer_group1.wdt;
```

3️⃣ **Instantiate and Create Handle for IO**: We need to configure the LED pin as a push-pull output and obtain a handler for the pin so that we can control it. We also need to obtain a handle for the button input pin. Before we can obtain any handles for the LED and the button we need to create an `IO` struct instance. The `IO` struct instance provides a HAL-designed struct that gives us access to all gpio pins thus enabling us to create handles for individual pins. This is similar to the concept of a `split` method used in other HALs (more detail [**here**](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)). We do this by calling the `new()` instance method on the `IO` struct as follows:

```rust
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
```

3️⃣ **Obtain a handle and configure the PWM pin**: Here `gpio1` needs to be configured into an output. This is done using the `into_push_pull_output` method. I will name the handle `buzzer_pin` and configure it as follows:

```rust
let mut buzzer_pin = io.pins.gpio1.into_push_pull_output();
```

#### ⏰ PWM Timer Peripheral Configuration:

The [ESP programming guide for LEDC control](https://docs.espressif.com/projects/esp-idf/en/v4.3/esp32/api-reference/peripherals/ledc.html) specifies the steps for configuration. Configuration is done in three steps:

1. Timer Configuration by specifying the PWM signal’s frequency and duty cycle resolution.
    
2. Channel Configuration by associating it with the timer and GPIO to output the PWM signal.
    
3. Change PWM Signal that drives the output.
    

The third step has already been done at an earlier stage where `buzzer_pin` has been configured.

The [esp32c3-hal documentation](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/ledc/index.html) also gives an example of how to achieve this. I noticed there is a a bit of difference between the two. A part that is not explicitly mentioned in the above steps, and required in the esp32c3-hal, is creating an instance of LEDC and selecting the clock source.

1️⃣ **Configure the LEDC Peripheral & Set Clock Source**: the `LEDC` peripheral is instantiated using the `new` method that has the following signature:

```rust
pub fn new(
    _instance: impl Peripheral<P = LEDC> + 'd,
    clock_control_config: &'d Clocks<'_>,
    system: &mut PeripheralClockControl
) -> LEDC<'d>
```

The method requires that we pass the LEDC peripheral, the clock control configuration, and the system peripheral clock control. For the clock parameters, an instance of `system` and `clocks` has been created earlier in the watchdog disable step. As such, an `LEDC` handle `buzzer` is created as follows:

```rust
// Initialize and create handle for LEDC peripheral
let mut buzzer = LEDC::new(
    peripherals.LEDC,
    &clocks,
    &mut system.peripheral_clock_control,
);

// Set up global clock source for LEDC to APB Clk
buzzer.set_global_slow_clock(LSGlobalClkSource::APBClk);
```

Notice the clock source, `APBClk`, was chosen from an `LSGlobalClkSource` enum for low-speed clock sources.

2️⃣ **Configure a Delay:** in the algorithm, a delay must be introduced to control the tempo. Using the `Delay` struct, a `delay` handle can be simply created as follows:

```rust
let mut delay = Delay::new(&clocks);
```

3️⃣ **Configure the Timer and the Channel:**

🚨 **Important** **Note**

> For the rest of the LEDC configuration, one would expect the configuration to appear ahead of the application loop. Though the LEDC peripheral in the esp32c3-hal there arent any methods yet that support variable frequency output. As such, in order to configure PWM frequency on the fly, the peripheral needs to be reconfigured every time the frequency needs to be changed. This means that the LEDC configuration code will appear inside the application loop.

The following is the rest of the LEDC configuration code as given but the example in the HAL:

```rust
let mut lstimer0 = buzzer.get_timer::<LowSpeed>(timer::Number::Timer0);
lstimer0
   .configure(timer::config::Config {
       duty: timer::config::Duty::Duty13Bit,
       clock_source: timer::LSClockSource::APBClk,
       frequency: tone.1,
   })
   .unwrap();

let mut channel0 =
buzzer.get_channel(channel::Number::Channel0, &mut buzzer_pin);
channel0
   .configure(channel::config::Config {
       timer: &lstimer0,
       duty_pct: 50,
   })
   .unwrap();
```

the timer is configured and given the `lstimer0` handle with 13 bit resolution, the `APBClk` as clock source, and a particular frequency from the `tones` array (shown later). The channel on the other hand is given the `channel0` handle and configured to use the `lstimer0` and duty cycle of 50%.

This is it for configuration! Let's now jump into the application code.

### **📱 Application Code**

According to the software design description, two arrays are needed to store the tone and tune information. The first array `tones`, contains a collection of tuples that provide a mapping of the note letter and its corresponding frequency. The second array `tune` contains a collection of tuples that present the note that needs to be played and the number of beats per note. Note that the `tune` array contains an empty note `' '` that presents silence and does not have a corresponding mapping in the `tones` array.

```rust
    let tones = [
        ('c', 261.Hz()),
        ('d', 294.Hz()),
        ('e', 329.Hz()),
        ('f', 349.Hz()),
        ('g', 392.Hz()),
        ('a', 440.Hz()),
        ('b', 493.Hz()),
    ];

    let tune = [
        ('c', 1),
        ('c', 1),
        ('g', 1),
        ('g', 1),
        ('a', 1),
        ('a', 1),
        ('g', 2),
        ('f', 1),
        ('f', 1),
        ('e', 1),
        ('e', 1),
        ('d', 1),
        ('d', 1),
        ('c', 2),
        (' ', 4),
    ];
```

Next, before jumping into the algorithmic loop the tempo needs to be defined which will be used in the `delay` handle. A `tempo` variable is created as follows:

```rust
let tempo = 300_u32;
```

Next, the application loop looks as follows:

```rust
// Application Loop
loop {
    // Obtain a note in the tune
    for note in tune {
        // Retrieve the freqeuncy and beat associated with the note
        for tone in tones {
            // Find a note match in the tones array and update 
            if tone.0 == note.0 {
            // Play the note for the desired duration (beats*tempo)
            // Adjust period of the PWM output to match the new freq
                let mut lstimer0 = buzzer.get_timer::<LowSpeed>
                                   (timer::Number::Timer0);
                lstimer0
                .configure(timer::config::Config {
                    duty: timer::config::Duty::Duty13Bit,
                    clock_source: timer::LSClockSource::APBClk,
                    frequency: tone.1,
                })
                .unwrap();

                let mut channel0 =
                buzzer.get_channel(channel::Number::Channel0, 
                                   &mut buzzer_pin);
                channel0
                .configure(channel::config::Config {
                    timer: &lstimer0,
                    duty_pct: 50,
                })
                .unwrap();

               // Keep the output on for as long as required by note
               delay.delay_ms(note.1 * tempo);
                } else if note.0 == ' ' {
               // If ' ' tone is found disable output for one beat
                 let mut lstimer0 = buzzer.get_timer::
                                    <LowSpeed>(timer::Number::Timer0);
                 lstimer0
                 .configure(timer::config::Config {
                     duty: timer::config::Duty::Duty13Bit,
                     clock_source: timer::LSClockSource::APBClk,
                     frequency: 1_u32.Hz(),
                 })
                 .unwrap();
                let mut channel0 = 
                        buzzer.get_channel(channel::Number::Channel0,       
                        &mut buzzer_pin);

                channel0
                .configure(channel::config::Config {
                    timer: &lstimer0,
                    duty_pct: 0,
                })
                .unwrap();
                // Keep the output off for as long as required by note
                delay.delay_ms(tempo);
                }
            }
            // Silence for half a beat between notes
        let mut lstimer0 = buzzer.get_timer::
                           <LowSpeed>(timer::Number::Timer0);
        lstimer0
        .configure(timer::config::Config {
            duty: timer::config::Duty::Duty13Bit,
            clock_source: timer::LSClockSource::APBClk,
            frequency: 1_u32.Hz(),
        })
        .unwrap();
        let mut channel0 = 
                buzzer.get_channel(channel::Number::Channel0, 
                                   &mut buzzer_pin);

        channel0
        .configure(channel::config::Config {
            timer: &lstimer0,
            duty_pct: 0,
        })
        .unwrap();
        // Keep the output off for half a beat between notes
        delay.delay_ms(tempo / 2);
    }
}
```

Let's break down the loop line by line. The line

```rust
for note in tune
```

iterates over the `tune` array obtaining a note with each iteration. Within the first loop another `for` loop `for tone in tones` is nested which iterates over the `tones` array. The second loop retrieves the frequency and beat associated for each note obtained from the `tune` array. The statement

```rust
if tone.0 == note.0
```

checks if there is a match for the mapping between the `note` and the `tone`. The `.0` index is in reference to the first index in the tuple which is the note letter. Once a match is found, the note is played for the desired duration which equals the beats multiplied by the tempo. This is done over three steps:

First, reconfiguring the `lstimer0` and `channel0`, the tone frequency is adjusted to match the frequency of the found `tone`. The frequency of the `tone` corresponds to index `1` of the tuple and is configured as follows:

```rust
let mut lstimer0 = buzzer.get_timer::<LowSpeed>(timer::Number::Timer0);
lstimer0
   .configure(timer::config::Config {
       duty: timer::config::Duty::Duty13Bit,
       clock_source: timer::LSClockSource::APBClk,
       frequency: tone.1,
   })
   .unwrap();

let mut channel0 =
buzzer.get_channel(channel::Number::Channel0, &mut buzzer_pin);
channel0
   .configure(channel::config::Config {
       timer: &lstimer0,
       duty_pct: 50,
   })
   .unwrap();
```

In the third and final step the output is kept on for a period of beat\*tempo milliseconds. Here I leverage the `delay` handle created earlier as follows:

```rust
delay.delay_ms(note.1 * tempo);
```

In the case a `' '` note is found the `LEDC` channel and timer are reconfigured to eliminate (sort of disable) the output for one beat.

```rust
else if note.0 == ' ' {
           // Code disabling output
           delay.delay_ms(tempo);
}
```

Finally, after exiting the inner loop, half a beat of silence is introduced between notes in the outer loop `tune` repeating the configuration code that disables the output:

```rust
// Code disabling output
delay.delay_ms(tempo / 2);
```

## **📱 Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also the Wokwi project can be accessed [**here**](https://wokwi.com/projects/363774125451149313).

```rust
#![no_std]
#![no_main]

use esp32c3_hal::{
    clock::ClockControl,
    delay::Delay,
    ledc::{
        channel,
        timer::{self},
        LSGlobalClkSource, LowSpeed, LEDC,
    },
    peripherals::Peripherals,
    prelude::*,
    timer::TimerGroup,
    Rtc, IO,
};
use esp_backtrace as _;

#[entry]
fn main() -> ! {
    // Take Peripherals, Initialize Clocks, and Create a Handle for Each
    let peripherals = Peripherals::take();
    let mut system = peripherals.SYSTEM.split();
    let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

    // Instantiate and Create Handles for the RTC and TIMG watchdog timers
    let mut rtc = Rtc::new(peripherals.RTC_CNTL);
    let timer_group0 = TimerGroup::new(peripherals.TIMG0, &clocks);
    let mut wdt0 = timer_group0.wdt;
    let timer_group1 = TimerGroup::new(peripherals.TIMG1, &clocks);
    let mut wdt1 = timer_group1.wdt;

    // Disable the RTC and TIMG watchdog timers
    rtc.swd.disable();
    rtc.rwdt.disable();
    wdt0.disable();
    wdt1.disable();

    // Instantiate and Create Handle for IO
    let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);

    // Instantiate and Create Handle for LED output & Button Input
    let mut buzzer_pin = io.pins.gpio1.into_push_pull_output();

    // Define the notes and their frequencies
    let tones = [
        ('c', 261_u32.Hz()),
        ('d', 294_u32.Hz()),
        ('e', 329_u32.Hz()),
        ('f', 349_u32.Hz()),
        ('g', 329_u32.Hz()),
        ('a', 440_u32.Hz()),
        ('b', 493_u32.Hz()),
    ];

    // Define the notes to be played and the beats per note
    let tune = [
        ('c', 1),
        ('c', 1),
        ('g', 1),
        ('g', 1),
        ('a', 1),
        ('a', 1),
        ('g', 2),
        ('f', 1),
        ('f', 1),
        ('e', 1),
        ('e', 1),
        ('d', 1),
        ('d', 1),
        ('c', 2),
        (' ', 4),
    ];

    // Define the tempo
    let tempo = 300_u32;

    // Initialize and create handle for LEDC peripheral
    let mut buzzer = LEDC::new(
        peripherals.LEDC,
        &clocks,
        &mut system.peripheral_clock_control,
    );

    // Set up global clock source for LEDC to APB Clk
    buzzer.set_global_slow_clock(LSGlobalClkSource::APBClk);

    // Instantiate Delay handle
    let mut delay = Delay::new(&clocks);

    // Application Loop
    loop {
        // Obtain a note in the tune
        for note in tune {
            // Retrieve the freqeuncy and beat associated with the note
            for tone in tones {
                // Find a note match in the tones array and update frequency and beat variables accordingly
                if tone.0 == note.0 {
                    // Play the note for the desired duration (beats*tempo)
                    // Adjust period of the PWM output to match the new frequency
                    let mut lstimer0 = buzzer.get_timer::<LowSpeed>(timer::Number::Timer0);
                    lstimer0
                        .configure(timer::config::Config {
                            duty: timer::config::Duty::Duty13Bit,
                            clock_source: timer::LSClockSource::APBClk,
                            frequency: tone.1,
                        })
                        .unwrap();

                    let mut channel0 =
                        buzzer.get_channel(channel::Number::Channel0, &mut buzzer_pin);
                    channel0
                        .configure(channel::config::Config {
                            timer: &lstimer0,
                            duty_pct: 50,
                        })
                        .unwrap();

                    // Keep the output on for as long as required by note
                    delay.delay_ms(note.1 * tempo);
                } else if note.0 == ' ' {
                    // If ' ' tone is found disable output for one beat
                    let mut lstimer0 = buzzer.get_timer::<LowSpeed>(timer::Number::Timer0);
                    lstimer0
                        .configure(timer::config::Config {
                            duty: timer::config::Duty::Duty13Bit,
                            clock_source: timer::LSClockSource::APBClk,
                            frequency: 1_u32.Hz(),
                        })
                        .unwrap();
                    let mut channel0 =
                        buzzer.get_channel(channel::Number::Channel0, &mut buzzer_pin);

                    channel0
                        .configure(channel::config::Config {
                            timer: &lstimer0,
                            duty_pct: 0,
                        })
                        .unwrap();
                    // Keep the output off for as long as required by note
                    delay.delay_ms(tempo);
                }
            }
            // Silence for half a beat between notes
            let mut lstimer0 = buzzer.get_timer::<LowSpeed>(timer::Number::Timer0);
            lstimer0
                .configure(timer::config::Config {
                    duty: timer::config::Duty::Duty13Bit,
                    clock_source: timer::LSClockSource::APBClk,
                    frequency: 1_u32.Hz(),
                })
                .unwrap();
            let mut channel0 = buzzer.get_channel(channel::Number::Channel0, &mut buzzer_pin);

            channel0
                .configure(channel::config::Config {
                    timer: &lstimer0,
                    duty_pct: 0,
                })
                .unwrap();
            // Keep the output off for half a beat between notes
            delay.delay_ms(tempo / 2);
        }
    }
}
```

## Conclusion

In this post, a buzzer application that plays a tune was created leveraging the LEDC peripheral to create a PWM output for the ESP32C3. It turns out that creating a variable frequency output PWM in the ESP32C3 is a bit more involved. This is because the HAL in its current form only supports fixed frequency output. All code was created at the HAL level using the [**esp32c3-hal**](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/). Have any questions? Share your thoughts in the comments below 👇

%%[subend]