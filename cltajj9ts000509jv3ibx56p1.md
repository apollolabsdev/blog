---
title: "Edge IoT with Rust on ESP: WiFi Revisited"
datePublished: Sat Mar 02 2024 20:33:27 GMT+0000 (Coordinated Universal Time)
cuid: cltajj9ts000509jv3ibx56p1
slug: edge-iot-with-rust-on-esp-wifi-revisited
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1709410739267/ffbd5275-ddf0-480f-8044-56900c1e81a0.png
tags: tutorial, iot, rust, esp32, embedded-systems

---

## Introduction

Ever since creating the [WiFi post](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi), I received several inquiries about using a custom SSID and password. In that past post, I had hardcoded the WiFi SSID and password. I figured its a sign for updating the code to demonstrate how to enter a custom access point SSID and password.

In this post, I will be updating the past [WiFi post](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi) application code to accommodate custom network SSID entry. UART will be used to acquire user entry from the terminal.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* [WiFi Blog Post](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi).
    
* [UART Blog Post](https://apollolabsblog.hashnode.dev/embassy-on-esp-uart-echo).
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/391058600462186497).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    

## **👨‍🎨 Software Design**

In the past example, the ESP32 was configured in station mode in the following steps:

1. Configure WiFi
    
2. Start WiFi
    
3. Connect WiFi
    
4. (Optional) Confirm Connection and Check Connection Configuration
    

Ahead of these steps we'll need to capture the user entry and use it in the WiFi configuration. These are the additional steps that need to be taken ahead of connecting to WiFi:

1. Instantiate and Configure UART
    
2. Ask user for SSID
    
3. Read and store SSID
    
4. Ask user for password
    
5. Read and store password
    

After that we can proceed to configure WiFi with the stored entries.

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `anyhow` crate for error handling.
    
* The `esp_idf_hal` crate to import the peripherals.
    
* The `esp_idf_svc` crate to import the device services necessary for WiFi.
    
* The `heapless` crate for the heapless `String` and `Vec` types.
    

```rust
use esp_idf_hal::delay::BLOCK;
use esp_idf_hal::gpio;
use esp_idf_hal::prelude::*;
use esp_idf_hal::uart::*;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use heapless::{String, Vec};
use std::fmt::Write;
```

### **🎛 Initialization/Configuration Code**

1️⃣ **Obtain a handle for the device peripherals**: Similar to all past blog posts, in embedded Rust, as part of the singleton design pattern, we first have to take the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

2️⃣ **Configure & Instantiate UART**: UART needs to be instantiated to use the same pins used for terminal logging. These are pins 20 and 21. This is similar to how UART was configured in the [UART post](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-uart-communication):

```rust
// Configure UART
// Create handle for UART config struct
let config = config::Config::default().baudrate(Hertz(115_200));

// Instantiate UART
let mut uart = UartDriver::new(
    peripherals.uart0,
    peripherals.pins.gpio21,
    peripherals.pins.gpio20,
    Option::<gpio::Gpio0>::None,
    Option::<gpio::Gpio1>::None,
    &config,
)
.unwrap();
```

3️⃣ **Acquire User Input**: Following the UART configuration, the user is prompted to enter the SSID as shown below. Following that, the SSID is captured by entering a loop where a character is read one at a time using the UART driver `read` method. Note the following:

* Each character entered is echoed to the console using the `write` method. This is so that the user has visual confirmation that the intended character is entered. In the case of password entry, an asterisk (ascii value 42) is echoed instead of the actual entry.
    
* Each character that is acquired is appended/buffered in the `ssid` and `password` vectors using the `extend_from_slice` `Vec` method.
    
* Every time a character is read, the code checks if its a carriage return (ascii value 13). If it is then the code breaks out of the loop.
    
* Both `ssid` and `password` are `heapless::Vec` types.
    

```rust
uart.write_str("Enter Network SSID: ").unwrap();

// Read and Buffer SSID
let mut ssid = Vec::<u8, 32>::new();
loop {
    let mut buf = [0_u8; 1];
    uart.read(&mut buf, BLOCK).unwrap();
    uart.write(&buf).unwrap();
    if buf[0] == 13 {
        break;
    }
    ssid.extend_from_slice(&buf).unwrap();
}

uart.write_str("\nEnter Network Password: ").unwrap();

// Read and Buffer Password
let mut password = Vec::<u8, 64>::new();
loop {
    let mut buf = [0_u8; 1];
    uart.read(&mut buf, BLOCK).unwrap();
    uart.write(&[42]).unwrap();
    if buf[0] == 13 {
        break;
    }
    password.extend_from_slice(&buf).unwrap();
}
```

4️⃣ **Adjust Buffered Types**: Both `ssid` and `password` are `Vec` types. The WiFi configuration however accepts a `heapless::String` type. As such, the acquired values need to be adjusted such that the types are compatible as follows:

```rust
let ssid: String<32> = String::from_utf8(ssid).unwrap();
let password: String<64> = String::from_utf8(password).unwrap();
```

5️⃣ **Obtain handle for WiFi**: this involves the same steps that were done in the WiFi [**post**.](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi)

```rust
let sysloop = EspSystemEventLoop::take()?;
let nvs = EspDefaultNvsPartition::take()?;
let mut wifi = EspWifi::new(peripherals.modem, sysloop, Some(nvs))?;
```

6️⃣ **Configure the WiFi Driver**: now that we have the ssid and password we can proceed to configure the `wifi` driver as follows:

```rust
wifi.set_configuration(&Configuration::Client(ClientConfiguration {
    ssid: ssid,
    bssid: None,
    auth_method: AuthMethod::None,
    password: password,
    channel: None,
}))?;
```

This is it for configuration! Let's now jump into the application code.

### **📱 Application Code**

**Start and Connect Wifi**: Now that wifi is configured, all we need to do is `start` it and then `connect` to a network:

```rust
// Start Wifi
wifi.start()?;

// Connect Wifi
wifi.connect()?;

// Wait until the network interface is up
wifi.wait_netif_up()?;

println!("Wifi Connected");

loop {}
```

This is it!

## **📱Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/391058600462186497).

```rust
use esp_idf_hal::delay::BLOCK;
use esp_idf_hal::gpio;
use esp_idf_hal::prelude::*;
use esp_idf_hal::uart::*;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{AuthMethod, BlockingWifi, ClientConfiguration, Configuration, EspWifi};
use heapless::{String, Vec};
use std::fmt::Write;

fn main() -> anyhow::Result<()> {
    // Take Peripherals
    let peripherals = Peripherals::take().unwrap();
    let sysloop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    // Configure UART
    // Create handle for UART config struct
    let config = config::Config::default().baudrate(Hertz(115_200));

    // Instantiate UART
    let mut uart = UartDriver::new(
        peripherals.uart0,
        peripherals.pins.gpio21,
        peripherals.pins.gpio20,
        Option::<gpio::Gpio0>::None,
        Option::<gpio::Gpio1>::None,
        &config,
    )
    .unwrap();

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sysloop.clone(), Some(nvs))?,
        sysloop,
    )?;

    // This line is for Wokwi only so that the console output is formatted correctly
    uart.write_str("\x1b[20h").unwrap();

    uart.write_str("Enter Network SSID: ").unwrap();

    // Read and Buffer SSID
    let mut ssid = Vec::<u8, 32>::new();
    loop {
        let mut buf = [0_u8; 1];
        uart.read(&mut buf, BLOCK).unwrap();
        uart.write(&buf).unwrap();
        if buf[0] == 13 {
            break;
        }
        ssid.extend_from_slice(&buf).unwrap();
    }

    uart.write_str("\nEnter Network Password: ").unwrap();

    // Read and Buffer Password
    let mut password = Vec::<u8, 64>::new();
    loop {
        let mut buf = [0_u8; 1];
        uart.read(&mut buf, BLOCK).unwrap();
        uart.write(&[42]).unwrap();
        if buf[0] == 13 {
            break;
        }
        password.extend_from_slice(&buf).unwrap();
    }

    let ssid: String<32> = String::from_utf8(ssid).unwrap();
    let password: String<64> = String::from_utf8(password).unwrap();

    wifi.set_configuration(&Configuration::Client(ClientConfiguration {
        ssid: ssid,
        bssid: None,
        auth_method: AuthMethod::None,
        password: password,
        channel: None,
    }))?;

    // Start Wifi
    wifi.start()?;

    // Connect Wifi
    wifi.connect()?;

    // Wait until the network interface is up
    wifi.wait_netif_up()?;

    println!("Wifi Connected");

    loop {}
}
```

## Conclusion

This post introduced how to configure and connect ESP Wifi in station mode using Rust and the `esp_idf_svc` crate. This code is modified from a past WiFi example allowing a user to enter an SSID and password instead of hardcoding them. This avoids having to recompile the code every time the WiFi station needs to be changed. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]