---
title: "ESP32 Standard Library Embedded Rust: Timers"
datePublished: Thu Aug 03 2023 14:25:46 GMT+0000 (Coordinated Universal Time)
cuid: clkv91ttb000909kygcbjg6gy
slug: esp32-standard-library-embedded-rust-timers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691072281692/470369fa-24f3-4c8e-b272-00fba41100d0.png
tags: tutorial, beginner, rust, embedded, esp32

---

> ***This blog post is the fourth of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [ESP32 Standard Library Embedded Rust: GPIO Control](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-gpio-control)
    
2. [ESP32 Standard Library Embedded Rust: UART Communication](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-uart-communication)
    
3. [ESP32 Standard Library Embedded Rust: I2C Communication](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-i2c-communication)
    

## Introduction

Hardware timers while relatively simple circuits are really effective in several applications in embedded. Timer peripherals are effective in timing both software and hardware events. Timers also have features that allow the generation of hardware waveforms (Ex. PWM). In this post, I'll configure and setup an ESP timer peripheral to measure the width of the pulse for two different signals. The resulting pulse width value will be printed on the console.

The square waves used in this post will be generated using the Wokwi custom external block. In general, this type of measurement would be useful as it emulates the behavior of applications that include tachometers or anemometers. Tachometers and anemometers generate square wave signals that are proportional to the speed of rotation. With some calculations, and depending on the application, the provided code can be expanded to provide frequency and/or rpm values.

%%[substart] 

### **📚** Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    

### **💾** Software Setup

All the code presented in this post is available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [here](https://wokwi.com/projects/371571666102490113).

### **🛠** Hardware Setup

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg align="center")
    
* **Square Wave Generator/Pulse Generator**: This can take many forms in real hardware though since I'm using Wokwi, there is the custom chip feature that allows me to generate square waves.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690962699761/0a69c59f-4495-41ed-9b5c-b849f0d9786b.png align="center")
    

#### **🔌** Connections

**📝 Note**

> All connection details are also shown in the [Wokwi example](https://wokwi.com/projects/371571666102490113).

Connections include the following:

* Gpio0 wired to the top pin of the breakout custom chip.
    
* Gpio1 wired to the bottom pin of the breakout custom chip.
    

## **👨‍🎨** Software Design

The square wave signal is going to be fed into the ESP32 as an input. For the purpose of this post, the code will measure the widths of the pulses in each square wave. The custom chip is designed such that one wave generates pulses that are 10ms wide and another that are 25ms wide. To measure the pulse width, the algorithm in this post needs to determine the time elapsed between every positive edge and the negative edge that follows.

Following the configuration of the pins and the device, the algorithm will work as follows:

Assuming that the existing (a.k.a `old`) level of the signal at the pin is `High`:

1. Poll/read the `current` (new) level at the input pin
    
2. If a positive edge transition occurs then reset/start the timer and update the `old` pin value.
    
3. If a negative edge transition occurs then capture the timer value and update the `old` pin value.
    
4. Calculate and print the duration of the pulse.
    
5. Loop back to step 1
    

### Identifying Edge Transitions

Note how in steps 2 and 3 we are checking for positive and negative edges. Programmatically, a transition is determined by checking if the pin `current` level is different than the `old` level. However, that still doesn't tell us if the transition is positive or negative. To check for positive edges, if the `current` level is `High`, this means that the transition is a positive edge transition. Otherwise, if it is `Low` this means the transition is negative.

Let's now jump into implementing this algorithm.

**🚨 *Important Note***

> The custom chip block in Wokwi has its own internal code that generates the square waves/pulses. There is a tab in the project where one can look at the source code. However, how to create and code a custom block is not part of this post. For the interested in creating custom chips in Wokwi I recommend checking out the [chips api documentation](https://docs.wokwi.com/chips-api/getting-started).

## **👨‍💻** Code Implementation

### **📥** Crate Imports

In this implementation, one crate is required as follows:

* The `esp_idf_hal` crate to import the needed device hardware abstractions.
    

```rust
use esp_idf_hal::gpio::*;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::timer::config::Config;
use esp_idf_hal::timer::TimerDriver;
```

### **🎛** Peripheral Configuration Code

Ahead of our application code, peripherals are configured through the following steps:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to `take` the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

**2️⃣ Obtain handles for the input pins**: As explained earlier, there are two signals that will be attached to input pins so that they can be measured. The pins are `gpio0` and `gpio1` that are given handles `pin1` and `pin2` respectively:

```rust
// Configure Pins that Will Read the Square Wave as Inputs
let pin1 = PinDriver::input(peripherals.pins.gpio0).unwrap();
let pin2 = PinDriver::input(peripherals.pins.gpio1).unwrap();
```

3️⃣ **Obtain a handle and configure the Timer peripheral**: In order to configure I2C, there is a `TimerDriver` abstraction in the [esp-idf-hal](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/timer/struct.TimerDriver.html) that contains a `new` method to create a `Timer` instance. The `new` method has the following signature:

```rust
pub fn new<TIMER: Timer>(
    _timer: impl Peripheral<P = TIMER> + 'd,
    config: &Config
) -> Result<Self, EspError>
```

as shown, the `new` method has 2 parameters. The first `timer` parameter is a `TIMER` peripheral type. The second is a timer configuration type `Config`. The ESP32C3 has two general timer peripherals named `timer00` and `timer10`. As such, we create an instance for two timers (one for each pin) `timer00` and `timer10` with handle names `timer1` and `timer2` as follows:

```rust
    let config = Config::new();
    let mut timer1 = TimerDriver::new(peripherals.timer00, &config).unwrap();
    let mut timer2 = TimerDriver::new(peripherals.timer10, &config).unwrap();
```

`Config` is a [`timer::config`](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/timer/config/struct.Config.html) type that provides configuration parameters for the Timer. `Config` contains a method `new` to create an instance with default parameters. Afterward, there are configuration parameters adjustable through various methods. A full list of those methods can be found in the [documentation](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/timer/config/struct.Config.html). For the purpose of this application, the default configuration should be fine.

That's it for configuration.

### **📱** Application Code

Before entering the program loop, the following initialization steps need to take place:

**1️⃣ Declare and initialize variables that will track the pin levels:**

Recall from the software design that we need to track both the `current` and `old` pin levels to determine the occurrence of transitions. For that, the pin methods return a `Level` enum which the tracking variables have to have the same type. For each pin two variables were created, a `current` and `old` variable. Additionally, the `old` variables need to be initialized with a known signal level. The starting state of `old` being `High` or `Low` doesn't matter as it's an assumed state and will be adjusted accordingly:

```rust
 let mut pin1_current_level: Level;
 let mut pin1_old_level: Level = Level::Low;

 let mut pin2_current_level: Level;
 let mut pin2_old_level: Level = Level::High;
```

**2️⃣ Reset Timer Value and Enable Timer**

For both timers their values can be reset to zero using the `set_counter` method and then enabled using the `enable` method:

```rust
timer1.set_counter(0_u64).unwrap();
timer2.set_counter(0_u64).unwrap();

timer1.enable(true).unwrap();
timer2.enable(true).unwrap();
```

**3️⃣ Declare and initialize variables that will store the timer values:**

The timer counts captured on negative edges will need to be retained in a variable. The `counter` method that captures the count returns a `u64`. As such, two `u64` variables are created `count1` and `count2`:

```rust
let mut count1: u64 = 0;
let mut count2: u64 = 0;
```

### 🔁 The Application Loop

Following the software design steps:

1. **Poll/read the** `current` **(new) level at the input pin**
    
    This is done using the `get_level` methods on `pin1` and `pin2` :
    
    ```rust
    // Get Level of pin 1
    pin1_current_level = pin1.get_level();
    // Get Level of pin 2
    pin2_current_level = pin2.get_level();
    ```
    
2. **If a positive edge transition occurs then reset/start the timer and update the** `old` **pin value**
    
    Recall, a transition is determined by checking if the pin `current` level is different than the `old` level. Additionally, for positive edges, we check if the `current` level is `High`. The counter is then reset using the `set_counter` used earlier and the pin `current_level` is copied over to the `old_level` to keep track of the signal level. In the current and next steps, I'm showing the code for only one pin for brevity.
    
    ```rust
    // If pin 1 level changed from Low to High then reset count
    if (pin1_current_level != pin1_old_level) & (pin1_current_level == Level::High) {
        timer1.set_counter(0).unwrap();
        pin1_old_level = pin1_current_level;
     }
    ```
    
3. **If a negative edge transition occurs then capture the timer value and update the** `old` **pin value**
    
    This step is more or less the same as the previous with two differences. First, is that we are checking for a negative edge transition using the `Low` variant. Second, we are capturing the timer level using the `counter` method.
    
    ```rust
    // If pin 1 level changed from High to Low then capture count
    if (pin1_current_level != pin1_old_level) & (pin1_current_level == Level::Low) {
        count1 = timer1.counter().unwrap();
        pin1_old_level = pin1_current_level;
    }
    ```
    
4. **Calculate and print the duration of the pulse**
    

Using `println` the pulse width duration is printed to the console. The count value is divided by `1000` to convert it to milliseconds since according to the documentation the timer clock frequency is 1 MHz.

```rust
println!("Sq Wave 1 Pulse Width is {}ms", count1 / 1000);
```

## **🔧 Enhancements & Optimizations**

There are two areas where the presented code can be enhanced. First is the continuous printing. Note how the width is printed in every loop regardless of whether the counter value was updated or not. Second is the repetitive logic/code when an edge is detected. The first area can be enhanced such that the printing happens only when the counter value is updated in a negative edge transition. The second, on the other hand, can be enhanced by introducing an edge detection function (which can include the second area too). Thank you to one of our readers, @[Christian Foucher](@Valda60) for suggesting these optimizations. Here is a suggested alternative implementation incorporating these enhancements:

```rust
use esp_idf_hal::gpio::*;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::timer::config::Config;
use esp_idf_hal::timer::TimerDriver;

fn main() {
    esp_idf_sys::link_patches();

    let peripherals = Peripherals::take().unwrap();

    // Configure and Initialize Timer Drivers
    let config = Config::new();
    let mut timer1 = TimerDriver::new(peripherals.timer00, &config).unwrap();
    let mut timer2 = TimerDriver::new(peripherals.timer10, &config).unwrap();

    // Configure Pins that Will Read the Square Wave as Inputs
    let pin1 = PinDriver::input(peripherals.pins.gpio0.downgrade_input()).unwrap();
    let pin2 = PinDriver::input(peripherals.pins.gpio1.downgrade_input()).unwrap();

    // Declare and Init Variables that will Track Pin level
    let mut pin1_old_level: Level = Level::Low;
    let mut pin2_old_level: Level = Level::High;

    // Set Counter Start Value to Zero
    timer1.set_counter(0_u64).unwrap();
    timer2.set_counter(0_u64).unwrap();

    // Enable Counter
    timer1.enable(true).unwrap();
    timer2.enable(true).unwrap();

    loop {
        pin1_old_level = measure_pin(&mut timer1, &pin1, 1, pin1_old_level);
        pin2_old_level = measure_pin(&mut timer2, &pin2, 2, pin2_old_level);
    }
}

fn measure_pin<T: Pin, MODE: esp_idf_hal::gpio::InputMode>(timer: &mut TimerDriver, pin: &PinDriver<T, MODE>, index: u8, pin_previous_level: Level) -> Level {
    // Get Level of pin
    let pin_current_level = pin.get_level();

    if pin_current_level != pin_previous_level {
        match pin_current_level {
            // If pin level changed from Low to High then reset count
            Level::High => timer.set_counter(0).unwrap(),
            // If pin level changed from High to Low then capture count
            Level::Low => {
                let count: u64 = timer.counter().unwrap();

                //**********integer version (no decimal digit)************
                //println!("Sq Wave {} Pulse Width is {}ms", index, count);
                //**********float version************

                //**********float version************
                //println!("Sq Wave {} Pulse Width is {:.1}ms", index, count as f32 / 1000f32);
                //**********float version************

                //**********remainder version********
                let integer_part  = count / 1000;
                let decimal_part  = (count % 1000) / 100;
                // Calculate and Print Out the Pulse Width
                // Clock Frequency is 1 MHz According to Code
                println!("Sq Wave {} Pulse Width is {}.{}ms", index, integer_part, decimal_part);
                //**********remainder version********

                //*****rust_decimal crate version********
                //let decimal_count = rust_decimal::Decimal::new(count as i64, 3); // decimal of 3 digits
                //println!("Sq Wave {} Pulse Width is {:.1}ms", index, decimal_count);
                //*****rust_decimal crate version********
            }
        }
    }

    pin_current_level
}
```

Note that there are options for printing in different formats. In the original code, only integer math was used.

## **📱** Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [here](https://wokwi.com/projects/371571666102490113).

```rust
use esp_idf_hal::gpio::*;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::timer::config::Config;
use esp_idf_hal::timer::TimerDriver;

fn main() {
    esp_idf_sys::link_patches();

    let peripherals = Peripherals::take().unwrap();

    // Configure and Initialize Timer Drivers
    let config = Config::new();
    let mut timer1 = TimerDriver::new(peripherals.timer00, &config).unwrap();
    let mut timer2 = TimerDriver::new(peripherals.timer10, &config).unwrap();

    // Configure Pins that Will Read the Square Wave as Inputs
    let pin1 = PinDriver::input(peripherals.pins.gpio0).unwrap();
    let pin2 = PinDriver::input(peripherals.pins.gpio1).unwrap();

    // Declare and Init Variables that will Track Pin level
    let mut pin1_current_level: Level;
    let mut pin1_old_level: Level = Level::Low;

    let mut pin2_current_level: Level;
    let mut pin2_old_level: Level = Level::High;

    // Set Counter Start Value to Zero
    timer1.set_counter(0_u64).unwrap();
    timer2.set_counter(0_u64).unwrap();

    // Enable Counter
    timer1.enable(true).unwrap();
    timer2.enable(true).unwrap();

    // Declare and Init Variables that will Track Count Value
    let mut count1: u64 = 0;
    let mut count2: u64 = 0;

    loop {
        // Get Level of pin 1
        pin1_current_level = pin1.get_level();
        // // Get Level of pin 2
        pin2_current_level = pin2.get_level();

        // If pin 1 level changed from Low to High then reset count
        if (pin1_current_level != pin1_old_level) & (pin1_current_level == Level::High) {
            timer1.set_counter(0).unwrap();
            pin1_old_level = pin1_current_level;
        }

        // If pin 1 level changed from High to Low then capture count
        if (pin1_current_level != pin1_old_level) & (pin1_current_level == Level::Low) {
            count1 = timer1.counter().unwrap();
            pin1_old_level = pin1_current_level;
        }

        // If pin 2 level changed from Low to High then reset count
        if (pin2_current_level != pin2_old_level) & (pin2_current_level == Level::High) {
            timer2.set_counter(0).unwrap();
            pin2_old_level = pin2_current_level;
        }

        // If pin 2 level changed from High to Low then capture count
        if (pin2_current_level != pin2_old_level) & (pin2_current_level == Level::Low) {
            count2 = timer2.counter().unwrap();
            pin2_old_level = pin2_current_level;
        }

        // Calculate and Print Out the Pulse Width
        // Clock Frequency is 1 MHz According to Code
        println!("Sq Wave 1 Pulse Width is {}ms", count1 / 1000);
        println!("Sq Wave 2 Pulse Width is {}ms", count2 / 1000);
    }
}
```

## Conclusion

In this post, a timer application measuring square wave pulse durations was created. The application leverages the Timer peripherals for the ESP32C3 microcontroller. The code was created using an embedded `std` development environment supported by the `esp-idf-hal`. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]