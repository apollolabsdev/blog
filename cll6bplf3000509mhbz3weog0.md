---
title: "ESP32 Standard Library Embedded Rust: PWM Servo Motor Sweep"
datePublished: Fri Aug 11 2023 08:25:42 GMT+0000 (Coordinated Universal Time)
cuid: cll6bplf3000509mhbz3weog0
slug: esp32-standard-library-embedded-rust-pwm-servo-motor-sweep
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691741836432/8eb67b79-1744-4e50-a399-3b6fafbe82b0.png
tags: tutorial, beginner, rust, esp32, embedded-systems

---

> ***This blog post is the fifth of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [ESP32 Standard Library Embedded Rust: GPIO Control](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-gpio-control)
    
2. [ESP32 Standard Library Embedded Rust: UART Communication](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-uart-communication)
    
3. [ESP32 Standard Library Embedded Rust: I2C Communication](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-i2c-communication)
    
4. [ESP32 Standard Library Embedded Rust: Timers](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-timers)
    

## Introduction

In this post, I will be exploring the generating PWM for the ESP32C3 using the Rust [**esp32c3-hal**](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/index.html). Implementing hardware-based PWM in the ESP32C3 is a bit non-conventional. Meaning that I expected the timer peripheral to have a PWM function similar to other microcontrollers you might use. ESP32s rather seem to have three types of application-driven peripherals that enable PWM implementation; the LED controller (LEDC) peripheral, the motor control (MCPWM) peripheral, and the Remote Control Peripheral (RMT). The ESP32C3 in particular does not have an MCPWM peripheral, so the choices come down to two. In this post, I use the LEDC peripheral. As such, I will configure and set up the LEDC peripheral to do a servo motor sweep. This will lead the servo motor to swing back and forth continuously.

%%[substart] 

### **📚** Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Basic knowledge of Servos and PWMs
    

### **💾** Software Setup

All the code presented in this post is available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [here](https://wokwi.com/projects/372530583695095809).

### **🛠** Hardware Setup

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg align="center")
    
* **SG90 Servo Motor**
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691733027185/c17f0feb-df42-4a29-86dc-c1b2ebf07686.png align="center")

#### **🔌** Connections

**📝 Note**

> All connection details are also shown in the [Wokwi example](https://wokwi.com/projects/372530583695095809).

Connections include the following:

* Gpio7 wired to the PWM pin of the servo.
    
* Servo V+ connected to ESP 5V
    
* Servo Gnd connected to ESP GND
    

## **👨‍🎨** Software Design

  
A servo motor's control involves transmitting a sequence of pulses along the signal line. This comes in the form of a PWM signal. The control signal's frequency should ideally be 50Hz, equivalent to, a period of 20 ms, or in other words, a pulse recurring every 20 ms. The width of each pulse dictates the servo's angular position, typically within a range of 180 degrees (constrained by physical limits of movement).

In general, pulses lasting 1 ms correspond to a 0-degree position, 1.5 ms to a 90-degree angle, and 2 ms to a full 180-degree rotation. However, variations may exist among different brands, leading to potential differences in the minimum and maximum pulse durations. Some servos could utilize 0.5ms for 0-degree positioning and 2.5ms for a 180-degree orientation. This typically can be addressed through calibration.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1691687865534/fa6ed242-3dc6-4b66-96a7-2f46fcda3173.png align="center")

A servo motor sweep algorithm would make a servo motor smoothly sweep its output shaft back and forth over a specified range of angles. This creates a controlled motion pattern, often used for purposes like scanning, testing, or showcasing. Here's how a servo motor sweep algorithm typically works:

1. **Define Parameters:** Determine the range of PWM duty cycles you want the servo to sweep through. These would map to actual angles. This could be from a minimum angle (e.g., 0 degrees) corresponding to 0.5 ms to a maximum angle (e.g., 180 degrees) corresponding to 2.5 ms.
    
2. **Initialization:** Position the servo motor at the starting angle of the sweep range (e.g., 0 degrees). You may also need to introduce a delay to allow the servo to reach this position smoothly.
    
3. **Sweeping Loop:** Create a loop that gradually increases the angle in increments until the maximum angle is reached. During each iteration of the loop, update the servo's position by changing the PWM duty cycle to move it to the new angle.
    
4. **Direction Change:** Once the servo reaches the maximum angle, reverse the direction of the sweep. Decrease the angle in increments back to the minimum angle.
    
5. **Loop Continuation:** Repeat the loop, incrementing and decrementing the angle, until the servo returns to its starting position (minimum angle).
    

One thing to keep in mind is delay and smoothing. To create a smooth motion, one can introduce a small delay between angle changes. This prevents rapid jerking of the motor and allows it to move at a controlled pace.

On to coding!

## **👨‍💻** Code Implementation

### **📥** Crate Imports

In this implementation, one crate is required as follows:

* The `esp_idf_hal` crate to import the needed device hardware abstractions.
    

```rust
use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::ledc::{config::TimerConfig, LedcDriver, LedcTimerDriver, Resolution};
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::prelude::*;
```

### **🎛** Peripheral Configuration Code

Ahead of our application code, peripherals are configured through the following steps:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to `take` the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

**2️⃣ Configure the LEDC Timer Driver**: The [**ESP programming guide for LEDC control**](https://docs.espressif.com/projects/esp-idf/en/v4.3/esp32/api-reference/peripherals/ledc.html) specifies the steps for configuration. Configuration is done in three steps:

1. Timer Configuration by specifying the PWM signal’s frequency and duty cycle resolution.
    
2. Channel Configuration by associating it with the timer and GPIO to output the PWM signal.
    
3. Change PWM Signal that drives the output.
    

In the esp-idf-hal API for Rust, the second and third steps are combined in one. As such, we need to configure the LEDC timer.

To configure the LEDC Timer, there exists the [`LedcTimerDriver`](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/ledc/struct.LedcTimerDriver.html) struct with a `new` method allowing us to create an instance of the driver. The `new` method has the following signature:

```rust
pub fn new<T: LedcTimer>(
    _timer: impl Peripheral<P = T> + 'd,
    config: &TimerConfig
) -> Result<Self, EspError>
```

Note that all `new` needs is a timer peripheral instance and a timer configuration as parameters. `TimerConfig` is a struct in the `ledc::config` module that allows us to define the timer frequency, resolution, and speed mode. Following that we can define a `timer_driver` handle and using the `new` method of the `LedcTimerDriver` and configure the timer as follows:

```rust
// Configure Pins that Will Read the Square Wave as Inputs
let timer_driver = LedcTimerDriver::new(
    peripherals.ledc.timer0,
    &TimerConfig::default()
        .frequency(50.Hz())
        .resolution(Resolution::Bits14),
)
.unwrap();
```

Note that I chose the `timer0` peripheral to drive the LEDC peripheral. Additionally, I adjusted the frequency to 50 Hz as its the desired frequency. Finally, I chose a resolution of 14-bits for the timer. The resolution defines how accurate the duty cycle/on time can be (for more insight refer to the [ESP IDF technical documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/ledc.html)).

3️⃣ **Obtain a handle and configure the LEDC peripheral**: One step remains in configuring the LEDC is creating an instance to drive the peripheral. There exists an [`LedcDriver`](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/ledc/struct.LedcDriver.html) struct with a `new` method allowing us to create an instance of the driver. The `new` method requires three parameters; a ledc peripheral channel, a timer driver (already created in the previous step), and a gpio pin. This results in the following code:

```rust
let mut driver = LedcDriver::new(
    peripherals.ledc.channel0,
    timer_driver,
    peripherals.pins.gpio7,
)
.unwrap();
```

That's it for configuration. On to coding the application!

### **📱** Application Code

Recovering the algorithmic steps from the software design section, the following need to take place ahead of the program loop:

**1️⃣ Define Parameters:** It was mentioned earlier that we need to determine the range of PWM duty cycles you want the servo to sweep through. Also that they would map to actual angles. This means that we would need to map the range of angles to the range of duty cycle values. As such, it would be useful to create a function that we can use later for that. After the main function, I defined a `map` function as follows:

```rust
// Function that maps one range to another
fn map(x: u32, in_min: u32, in_max: u32, out_min: u32, out_max: u32) -> u32 {
    (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min
}
```

`x` is the input value we wish to map, `in_min` and `in_max` define the minimum and maximum limits of the range for the input value, `out_min` and `out_max` define the minimum and maximum limits of the range for the output value. Consequently, the function returns a `u32` value with a value mapped to the output range.

Now using the `get_max_duty` `LedcTimerDriver` method, we can retrieve the maximum duty value and store it in `max_duty`. After that we can calculate the `min_limit` and `max_limit` that correspond to the 0.5 ms (2.5 % Duty Cycle) and 2.5 ms (12.5 % Duty Cycle) on time values, respectively. Note that I scaled the numerators and denominators to avoid floating point math.

```rust
// Get Max Duty and Calculate Upper and Lower Limits for Servo
let max_duty = driver.get_max_duty();
let min_limit = max_duty * 25 / 1000;
let max_limit = max_duty * 125 / 1000;
```

**2️⃣ Motor Position Initialization:** Now we need to position the servo motor at the starting angle of the sweep range (e.g., 0 degrees). Driving the motor to the zero angle is done using the `set_duty` `LedcTimerDriver` method and the `map` function created earlier:

```rust
// Define Starting Position
driver
    .set_duty(map(0, 0, 180, min_limit, max_limit))
    .unwrap();
// Give servo some time to update
FreeRtos::delay_ms(500);
```

Note that a 0.5s delay has also been added to allow the servo some time to adjust.

### 🔁 The Application Loop

Following the software design steps:

1. **Create a Sweeping Loop:** We need to create a loop that gradually increases the angle in increments until the maximum angle is reached. During each iteration of the loop, we can update the servo's position by using the same `set_duty` method along with the `map` function. This can be done using a `for` loop with a `0..180` range as follows:
    

```rust
// Sweep from 0 degrees to 180 degrees
for angle in 0..180 {
    // Print Current Angle for visual verification
    println!("Current Angle {} Degrees", angle);
    // Set the desired duty cycle
    driver
      .set_duty(map(angle, 0, 180, min_limit, max_limit))
      .unwrap();
    // Give servo some time to update
    FreeRtos::delay_ms(12);
}
```

Note that the `0..180` `Range` as defined in Rust, will go up to `179` . As a result, the end value can be changed to `181` if the 180 angle value is required to be achieved

1. **Change the Sweep Direction:** To change the sweep direction the code is identical to the previous step with one minor modification. This can be achieved by using the `rev` method on the `0..180` `Range`.
    

```rust
// Sweep from 180 degrees to 0 degrees
for angle in (0..180).rev() {
    // Print Current Angle for visual verification
    println!("Current Angle {} Degrees", angle);
    // Set the desired duty cycle
    driver
      .set_duty(map(angle, 0, 180, min_limit, max_limit))
      .unwrap();
    // Give servo some time to update
    FreeRtos::delay_ms(12);
}
```

## **📱** Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [here](https://wokwi.com/projects/372530583695095809).

```rust
use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::ledc::{config::TimerConfig, LedcDriver, LedcTimerDriver, Resolution};
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::prelude::*;

fn main() {
    esp_idf_sys::link_patches();

    // Take Peripherals
    let peripherals = Peripherals::take().unwrap();

    // Configure and Initialize LEDC Timer Driver
    let timer_driver = LedcTimerDriver::new(
        peripherals.ledc.timer0,
        &TimerConfig::default()
            .frequency(50.Hz())
            .resolution(Resolution::Bits14),
    )
    .unwrap();

    // Configure and Initialize LEDC Driver
    let mut driver = LedcDriver::new(
        peripherals.ledc.channel0,
        timer_driver,
        peripherals.pins.gpio7,
    )
    .unwrap();

    // Get Max Duty and Calculate Upper and Lower Limits for Servo
    let max_duty = driver.get_max_duty();
    println!("Max Duty {}", max_duty);
    let min_limit = max_duty * 25 / 1000;
    println!("Min Limit {}", min_limit);
    let max_limit = max_duty * 125 / 1000;
    println!("Max Limit {}", max_limit);

    // Define Starting Position
    driver
        .set_duty(map(0, 0, 180, min_limit, max_limit))
        .unwrap();
    // Give servo some time to update
    FreeRtos::delay_ms(500);

    loop {
        // Sweep from 0 degrees to 180 degrees
        for angle in 0..180 {
            // Print Current Angle for visual verification
            println!("Current Angle {} Degrees", angle);
            // Set the desired duty cycle
            driver
                .set_duty(map(angle, 0, 180, min_limit, max_limit))
                .unwrap();
            // Give servo some time to update
            FreeRtos::delay_ms(12);
        }

        // Sweep from 180 degrees to 0 degrees
        for angle in (0..180).rev() {
            // Print Current Angle for visual verification
            println!("Current Angle {} Degrees", angle);
            // Set the desired duty cycle
            driver
                .set_duty(map(angle, 0, 180, min_limit, max_limit))
                .unwrap();
            // Give servo some time to update
            FreeRtos::delay_ms(12);
        }
    }
}

// Function that maps one range to another
fn map(x: u32, in_min: u32, in_max: u32, out_min: u32, out_max: u32) -> u32 {
    (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min
}
```

## Conclusion

In this post, a PWM application creating a servo motor sweep effect was created. The application leverages the LEDC peripheral for the ESP32C3 microcontroller. The code was also created using an embedded `std` development environment supported by the `esp-idf-hal`. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]