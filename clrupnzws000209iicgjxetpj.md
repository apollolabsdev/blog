---
title: "ESP32 Embedded Rust at the HAL: I2C Scanner"
datePublished: Fri Jan 26 2024 14:01:04 GMT+0000 (Coordinated Universal Time)
cuid: clrupnzws000209iicgjxetpj
slug: esp32-embedded-rust-at-the-hal-i2c-scanner
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1706277060056/ede81f34-90d0-4f58-a146-2a33160846d3.png
tags: tutorial, iot, rust, embedded-systems, rust-lang

---

> ***This blog post is the eleventh of a multi-part series of posts where I explore various peripherals in the ESP32C3 using embedded Rust at the HAL level. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [**ESP32 Embedded Rust at the HAL: GPIO Button Controlled Blinking**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-gpio-button-controlled-blinking)
    
2. [**ESP32 Embedded Rust at the HAL: Button-Controlled Blinking by Timer Polling**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling)
    
3. [**ESP32 Embedded Rust at the HAL: UART Serial Communication**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-uart-serial-communication)
    
4. [**ESP32 Embedded Rust at the HAL: PWM Buzzer**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-pwm-buzzer)
    
5. [**ESP32 Embedded Rust at the HAL: Timer Ultrasonic Distance Measurement**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-timer-ultrasonic-distance-measurement)
    
6. [**ESP32 Embedded Rust at the HAL: Analog Temperature Sensing using the ADC**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-analog-temperature-sensing-using-the-adc)
    
7. [**ESP32 Embedded Rust at the HAL: GPIO Interrupts**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-gpio-interrupts)
    
8. [**ESP32 Embedded Rust at the HAL: SPI Communication**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-spi-communication)
    
9. [**ESP32 Embedded Rust at the HAL: Random Number Generator**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-random-number-generator)
    
10. [**ESP32 Embedded Rust at the HAL: Remote Control Peripheral**](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-remote-control-peripheral)
    

%%[substart] 

## Introduction

I2C is a popular serial communication protocol in embedded systems. Another common name used is also Two-Wire Interface (TWI), since, well, it is. I2C is commonly found in sensors and interface boards as it offers a compact implementation with decent speeds reaching Mbps. Counter to UART, I2C allows multiple devices (up to 127) to tap into the same bus. As a result, it becomes often useful if there is a way to scan for devices that are connected.

In this post, I will be configuring and setting up I2C communication for the ESP32C3 to scan the bus for devices. The code would detect if there is a device connected to a particular address. The retrieved results will also be printed on the console.

### 📚 Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with the basic template for creating embedded applications in Rust.
    
* Familiarity with [**I2C communication basics**](https://learn.sparkfun.com/tutorials/i2c/all).
    

### 💾 Software Setup

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/387823866026280961).

### 🛠 Hardware Setup

#### **Materials**

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    
* #### Any I2C-based device (Ex. [**Adafruit DS1307 Real-Time Clock**](https://www.adafruit.com/product/3296)).
    

#### 🔌 Connections

Connections include the following:

* Gpio2 wired to the SCL pin of all I2C devices.
    
* Gpio3 wired to the SDA pin of all I2C devices.
    
* Power and Gnd wired to all I2C devices on the bus.
    

## 👨‍🎨 Software Design

I2C communication involves a master device initiating communication with a slave device through a start condition, followed by sending the slave's address and a read/write bit. The slave responds with an acknowledgment, and data transfer occurs with subsequent acknowledgment or non-acknowledgment after each byte. Finally, the communication is concluded with a stop condition. Furthermore, the address field of an I2C frame is 7-bits wide which supports up to 127 devices on a connected single bus.

Note how after a slave address is sent, a slave device has to respond with an acknowledgment. If a slave does not respond, it means that it does not exist on the bus. As such, to create an I2C scanner the following steps need to be taken:

1. Initialize `address` to 0.
    
2. Send a read frame to `address`
    
3. If ack received, record that device detected at `address` , otherwise, record that no device is connected.
    
4. Increment `address` .
    
5. If `address`&lt; 127 Go back to step 2. Otherwise, terminate the application.
    

## 👨‍💻 Code Implementation

### 📥 Crate Imports

In this implementation the crates required are as follows:

* The `esp32c3_hal` crate to import the ESP32C3 device hardware abstractions.
    
* The `esp_backtrace` crate to define the panicking behavior.
    
* The `esp_println` crate to provide `println!` implementation.
    

```rust
use esp32c3_hal::{clock::ClockControl, 
    i2c::I2C, 
    peripherals::Peripherals, 
    prelude::*, 
    IO,
};
use esp_backtrace as _;
use esp_println::println;
```

### 🎛 Peripheral Configuration Code

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to take the PAC-level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let peripherals = Peripherals::take();
```

**2️⃣ Instantiate and obtain handle to system clocks:** The system clocks need to be configured as they are needed in setting up the `I2C` peripheral. To set up the system clocks we need to first make the `SYSTEM` struct compatible with the HAL using the `split` method (more insight on this [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)). After that, the system clock control information is passed to `boot_defaults` in `ClockControl`. `ClockControl` controls the enablement of peripheral clocks providing necessary methods to enable and reset specific peripherals. Finally, the `freeze()` method is applied to freeze the clock configuration. Note that freezing the clocks is a protection mechanism by the HAL to avoid the clock configuration changing during runtime.

```rust
    let system = peripherals.SYSTEM.split();
    let clocks = ClockControl::boot_defaults(system.clock_control).freeze();
```

3️⃣ **Instantiate and Create Handle for IO**: When creating an instance for I2C, we'll be required to pass it instances of the SDA and SCL pins. As such, we'll need to create an `IO` struct instance. The `IO` struct instance provides a HAL-designed struct that gives us access to all gpio pins thus enabling us to create handles/instances for individual pins. We do this like prior posts, by calling the `new()` instance method on the `IO` struct as follows:

```rust
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
```

Note how the `new` method requires passing two parameters; the `GPIO` and `IO_MUX` peripherals.

**4️⃣ Configure and Obtain Handle for the I2C peripheral**: To create an instance of an I2C peripheral we need to use the `new` instance method in `i2c::I2c` in the esp32c3-hal:

```rust
pub fn new<SDA, SCL>(
    i2c: impl Peripheral<P = T> + 'd,
    sda: impl Peripheral<P = SDA> + 'd,
    scl: impl Peripheral<P = SCL> + 'd,
    frequency: Rate<u32, 1, 1>,
    clocks: &Clocks<'_>
) -> I2C<'d, T>
where
    SDA: OutputPin + InputPin,
    SCL: OutputPin + InputPin,
```

The `i2c` parameter expects and instance of an i2c peripheral, the `sda` and `scl` parameters expect instances of pins configured as input/output, `frequency` is the desired bus frequency, and `clocks` is an instance of the system clocks. As such, I create an instance for the `pulse` handle as follows:

```rust
let mut i2c0 = I2C::new(
    peripherals.I2C0,
    io.pins.gpio3,
    io.pins.gpio2,
    100u32.kHz(),
    &clocks,
);
```

I've chosen `I2C0` wth `gpio3` for SDA, `gpio2` for SCL, and 100kHz for the bus frequency.

Off to the application!

### 📱 Application Code

In the application, we are going to run a scan once for all possible addresses from 1 to 127. As such, a `for` loop can be utilized. In each iteration, we're first going to print the address being scanned followed by doing a `i2c` `read`. The `I2C` `read` method takes two parameters, a `u8` address and a `&[u8]` buffer to store the read data. Also, in our case, we don't really care about any data received.

`read` returns a `Result`, in which case, if the `Result` is `Ok` that means that a device was found. Otherwise, if the `Result` is `Err`, it means that an ACK was not received and a device was not found. The following is the application code:

```rust
// Start Scan at Address 1 going up to 127
for addr in 1..=127 {
    println!("Scanning Address {}", addr as u8);

    // Scan Address
    let res = i2c0.read(addr as u8, &mut [0]);

    // Check and Print Result
    match res {
        Ok(_) => println!("Device Found at Address {}", addr as u8),
        Err(_) => println!("No Device Found"),
    }
}
```

Finally, since the `main` returns a `!`, then an empty `loop` is required.

```rust
loop{}
```

## 📱 Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also the Wokwi project can be accessed [**here**](https://wokwi.com/projects/387823866026280961).

```rust
#![no_std]
#![no_main]
#![feature(type_alias_impl_trait)]

use esp32c3_hal::{clock::ClockControl, i2c::I2C, peripherals::Peripherals, prelude::*, IO};
use esp_backtrace as _;
use esp_println::println;

#[entry]
fn main() -> ! {
    let peripherals = Peripherals::take();
    let system = peripherals.SYSTEM.split();
    let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

    // Obtain handle for GPIO
    let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);

    // Initialize and configure I2C0
    let mut i2c0 = I2C::new(
        peripherals.I2C0,
        io.pins.gpio3,
        io.pins.gpio2,
        100u32.kHz(),
        &clocks,
    );

    // This line is for Wokwi only so that the console output is formatted correctly
    esp_println::print!("\x1b[20h");

    // Start Scan at Address 1 going up to 127
    for addr in 1..=127 {
        println!("Scanning Address {}", addr as u8);

        // Scan Address
        let res = i2c0.read(addr as u8, &mut [0]);

        // Check and Print Result
        match res {
            Ok(_) => println!("Device Found at Address {}", addr as u8),
            Err(_) => println!("No Device Found"),
        }
    }

    // Loop Forever
    loop {}
}
```

## Conclusion

In this post, a I2C scanner application was created leveraging the I2C peripheral for the ESP32C3. The I2C code was created at the HAL level using the Rust `no-std` `esp32c3-hal`. Have any questions/comments? Share your thoughts in the comments below 👇.

%%[subend]