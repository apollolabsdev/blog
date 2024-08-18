---
title: "Edge IoT with Rust on ESP: NTP"
datePublished: Fri Nov 03 2023 05:06:04 GMT+0000 (Coordinated Universal Time)
cuid: cloi5kfrm000509l52ewl36qu
slug: edge-iot-with-rust-on-esp-ntp
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1698987477369/6bc6b891-ae10-45a2-a9cc-6251d68dd70d.png
tags: tutorial, iot, rust, esp32, embedded-systems

---

> ***This blog post is the fourth of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [Edge IoT with Rust on ESP: Connecting WiFi](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi)
    
2. [Edge IoT with Rust on ESP: HTTP Client](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-http-client)
    
3. [Edge IoT with Rust on ESP: HTTP Server](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-http-server)
    

## Introduction

In the last post, we understood how to create an HTTP server on an ESP device using the Rust ESP-IDF framework. That also required a connection to WiFi that was established and explained in an earlier post. In this post, we are going to move on to a different application protocol to synchronize device system time with network time using the ESP and Rust. For that, we'll need to use the SNTP protocol that will allow us to retrieve time from the network.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Basic familiarity with WiFi & NTP.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/380110069926213633).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    

## **👨‍🎨 Software Design**

SNTP, which stands for Simple Network Time Protocol, is a protocol used to synchronize computer clocks over a network. In simpler terms, it helps devices like computers, routers, and servers to make sure they all have the same accurate time. Here's how it works:

1. **Requesting the Time:**
    
    * A device (Ex. the ESP) sends a request to a time server on the network asking for the current time.
        
2. **Time Server Responds:**
    
    * The time server receives the request and responds by sending back the current time, usually down to the millisecond.
        
3. **Adjusting the Clock:**
    
    * Your device receives the time from the server and adjusts its own clock to match the received time. This ensures that all devices connected to the network are using the same time reference.
        
4. **Regular Updates:**
    
    * SNTP can be configured to periodically sync the time. Devices can send requests at regular intervals to make sure their clocks stay accurate.
        

SNTP is a simplified version of the Network Time Protocol (NTP). While NTP provides more advanced features and accuracy, SNTP is lightweight and suitable for most basic time synchronization needs, especially in situations where high precision is not critical.

SNTP operates over UDP (User Datagram Protocol) at the transport layer. UDP is a connectionless protocol that provides a lightweight way to exchange data between devices on a network. SNTP uses UDP because it is faster and requires less overhead compared to connection-oriented protocols like TCP. Since time synchronization typically involves sending small packets of data with minimal delay, using UDP makes SNTP a suitable choice for this purpose.

UTC, or Coordinated Universal Time, serves as the global time standard, providing a uniform reference point for timekeeping across different regions. SNTP relies on UTC to provide accurate and standardized time information to devices. When a device using SNTP sends a request for the current time, it receives UTC time from a time server.

In this post, we are going to configure the ESP to obtain the UTC time from an NTP server and print it to the console. The steps include the following:

1. Configure and Connect to WiFi
    
2. Configure SNTP
    
3. Synchronize Time
    
4. Obtain System Time
    
5. Convert Time to UTC time
    
6. Print Time
    
7. Delay and Go Back to Step 4
    

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `anyhow` crate for error handling.
    
* The `esp_idf_hal` crate to import the peripherals.
    
* The `esp_idf_svc` crate to import the device services (wifi in particular).
    
* The `embedded_svc` crate to import the needed `wifi` service traits.
    
* The `std::SystemTime` to obtain system time.
    
* The `chrono` crate to perform `DateTime` conversions and time formatting.
    

```rust
use anyhow;
use chrono::{DateTime, Utc};
use embedded_svc::wifi::{AuthMethod, ClientConfiguration, Configuration};
use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::sntp::{EspSntp, SyncStatus};
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};
use std::time::SystemTime;
```

### **🎛 Initialization/Configuration Code**

1️⃣ **Obtain a handle for the device peripherals**: Similar to all past blog posts, in embedded Rust, as part of the singleton design pattern, we first have to take the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

2️⃣ **Configure and Connect to WiFi**: this involves the same steps that were done in the [wifi post](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi).

3️⃣ **Create the SNTP Connection Handle:** Within `esp_idf_svc::sntp` there exists an `EspSntp` abstraction. This is the abstraction needed to set up and configure SNTP. `EspSntp` contains a `new_default` method allowing us to configure SNTP with a `default` configuration. Following that we create an `ntp` handle as follows:

```rust
let ntp = EspSntp::new_default().unwrap();
```

You can alternatively configure the SNTP with different stnp servers, operating modes, or synchronization modes. For that, there exists a `new` method and a `SntpConf` struct in the `esp_idf_svc::sntp` module.

That's it for Configuration!

### **📱 Application Code**

1️⃣ **Synchronize SNTP:** Within the `EspSntp` abstraction, there is a `get_sync_status` method that is used to obtain the synchronization status. `get_sync_status` returns a `SyncStatus` enum indicating whether the `SystemTime` has been synced. As such, we can ensure synchronization is performed before proceeding as follows:

```rust
println!("Synchronizing with NTP Server");
while ntp.get_sync_status() != SyncStatus::Completed {}
println!("Time Sync Completed");
```

2️⃣ **Obtain and Print Time:** To obtain the current time we'll need to use the `now` method in the `SystemTime` abstraction. The time obtained, however, is not in a desirable format. For that, we can use the `chrono` crate that allows us to convert to a different type. We'll be converting to the `DateTime<Utc>` type. Within `DateTime` is a `format` method that allows us also format the `DateTime` `String`. Here's the full code:

```rust
loop {
    // Obtain System Time
    let st_now = SystemTime::now();
    // Convert to UTC Time
    let dt_now_utc: DateTime<Utc> = st_now.clone().into();
    // Format Time String
    let formatted = format!("{}", dt_now_utc.format("%d/%m/%Y %H:%M:%S"));
    // Print Time
    println!("{}", formatted);
    // Delay
    FreeRtos::delay_ms(1000);
}
```

## **📱Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/380110069926213633).

```rust
use anyhow;
use chrono::{DateTime, Utc};
use embedded_svc::wifi::{AuthMethod, ClientConfiguration, Configuration};
use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::sntp::{EspSntp, SyncStatus};
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};
use std::time::SystemTime;

fn main() -> anyhow::Result<()> {
    // It is necessary to call this function once. Otherwise some patches to the runtime
    // implemented by esp-idf-sys might not link properly. See https://github.com/esp-rs/esp-idf-template/issues/71
    esp_idf_sys::link_patches();

    let peripherals = Peripherals::take().unwrap();
    let sysloop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sysloop.clone(), Some(nvs))?,
        sysloop,
    )?;

    wifi.set_configuration(&Configuration::Client(ClientConfiguration {
        ssid: "Wokwi-GUEST".into(),
        bssid: None,
        auth_method: AuthMethod::None,
        password: "".into(),
        channel: None,
    }))?;

    // Start Wifi
    wifi.start()?;

    // Connect Wifi
    wifi.connect()?;

    // Wait until the network interface is up
    wifi.wait_netif_up()?;

    // Print Out Wifi Connection Configuration
    while !wifi.is_connected().unwrap() {
        // Get and print connection configuration
        let config = wifi.get_configuration().unwrap();
        println!("Waiting for station {:?}", config);
    }

    println!("Wifi Connected");

    // Create Handle and Configure SNTP
    let ntp = EspSntp::new_default().unwrap();

    // Synchronize NTP
    println!("Synchronizing with NTP Server");
    while ntp.get_sync_status() != SyncStatus::Completed {}
    println!("Time Sync Completed");

    loop {
        // Obtain System Time
        let st_now = SystemTime::now();
        // Convert to UTC Time
        let dt_now_utc: DateTime<Utc> = st_now.clone().into();
        // Format Time String
        let formatted = format!("{}", dt_now_utc.format("%d/%m/%Y %H:%M:%S"));
        // Print Time
        println!("{}", formatted);
        // Delay
        FreeRtos::delay_ms(1000);
    }
}
```

## Conclusion

NTP is a useful protocol in Internet applications used for synchronizing time. As a result, it can be useful in many projects that require time synchronization. This post introduced how to obtain time from an NTP server on ESP using Rust and the `esp_idf_svc` crate. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]