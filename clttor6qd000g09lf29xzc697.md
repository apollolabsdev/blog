---
title: "Embedded Rust Bluetooth on ESP: BLE Advertiser"
datePublished: Sat Mar 16 2024 06:07:12 GMT+0000 (Coordinated Universal Time)
cuid: clttor6qd000g09lf29xzc697
slug: embedded-rust-bluetooth-on-esp-ble-advertiser
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1710568925151/b5b81f22-f099-419c-8889-06ec07f07a86.png
tags: tutorial, iot, rust, bluetooth, bluetooth-low-energy, esp32

---

> This post is the second of a multi-part series where I'm exploring the use of Bluetooth Low Energy along embedded Rust on the ESP32. Information in this post might rely on knowledge presented in past posts.

1. [Embedded Rust Bluetooth on ESP: BLE Scanner](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-scanner)
    

## Introduction

In [last week's post](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-scanner), some BLE foundations were introduced where the difference between **central** and **peripheral** devices was explained. **Central** devices initiate connections and request data or services from **peripheral** devices. **Peripheral** devices, on the other end, were ones that advertise their presence and provide services to **central** devices. Additionally, these terms were associated with devices before a connection was established. In that context, the ESP32-C3 was configured to run as a **central** device to perform a scan for advertising BLE **peripheral** devices.

In this post, we're going to remain in the pre-connection phase but do the opposite. We're going to configure the ESP32-C3 to run as an advertising **peripheral** device. The code will be built using the [**esp32-nimble**](https://crates.io/crates/esp32-nimble/0.6.0) crate.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with standard library development in Rust with the ESP.
    
* Basic knowledge of networking layering concepts/stacks (ex. OSI model).
    
* Basic knowledge of Bluetooth.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3 git repo**](https://github.com/apollolabsdev/ESP32C3). Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp align="center")
    
    #### **🔌 Connections**
    
    No connections are required for this example.
    

## **👨‍🎨 Software Design**

The idea of advertising is for a **peripheral** device to "advertise" its presence and services. However, a **peripheral** device does not necessarily have to establish a connection with a **central** device to provide data. An application example is beacons. Beacons are **peripheral** devices that broadcast data but do not establish connections with **central** devices. In this post, we are going to program the ESP as an advertising **peripheral** device. Key settings include the following:

### Advertisment Type

There are several advertisement types including the following:

* **Not connectable:** This means that a **central** device cannot connect to the **peripheral** device.
    
* **Directed connectable:** This means that the **peripheral** device advertisement packets are targeted to a specific central scanner. This type is meant for cases where the **peripheral** already knows the **central** and wants to allow for a quick reconnection.
    
* **Undirected connectable:** This means that the **peripheral** device advertisement packets are not targeted to a specific scanner.
    

### Scan Response

Advertising **peripherals** can choose to accept what is referred to as **scan requests**. **Central** devices can choose to send **scan requests** to advertisers. If a **scan request** is accepted, the **peripheral** device would respond with additional information in a **scan response**. In that context, a **peripheral** can be either **scannable** or **non-scannable**. A **scannable** **peripheral** is one that accepts **scan requests** from a **central** scanner.

Recall from [last week's post](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-scanner), in the **central** device, **scan requests** were defined through the **scan type**. The **scan type** could be configured as active or passive. Active scanning meant that the central would send **scan requests**. Additionally, the **scan type** was configured using the `active_scan` method.

### Discoverable Mode

The discoverable mode refers to the state in which a BLE device actively broadcasts its presence to nearby devices. There are two main discoverable modes:

1. **General Discoverable Mode**: In this mode, the BLE device continuously broadcasts advertising packets to advertise its presence to nearby devices.
    
2. **Limited Discoverable Mode**: Limited discoverable mode is similar to general discoverable mode but with a limited duration. This mode is typically used when the device only needs to be discoverable for a short time to establish connections with nearby devices. This mode is useful in cases relative to conserving power.
    

The **peripheral** that will be configured in this post, will only advertise its presence. As such, it will not advertise any services, will be **non-connectable** and **non-scannable** (doesn't support scan requests).

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `esp_idf_hal` crate to import `delay` abstractions.
    
* The `esp32_nimble` crate for the necessary BLE abstractions.
    
* The `esp_idf_sys` crate since its needed by the `esp32_nimble` crate.
    

```rust
use esp32_nimble::{
    enums::{ConnMode, DiscMode},
    BLEAdvertisementData, BLEDevice,
};
use esp_idf_hal::delay::FreeRtos;
use esp_idf_sys as _;
```

### **🎛 Initialization/Configuration Code**

**1️⃣ Obtain a handle for the BLE device**: Similar to the pattern we've seen in embedded Rust with peripherals, as part of the singleton design pattern, we first have to take ownership of the device peripherals. In this context, its the `BLEDevice` that we need to take ownership of. This is done using the `take()` associated method. Here I create a BLE device handler named `ble_device` as follows:

```rust
let ble_device = BLEDevice::take();
```

Although not obvious, note that `take` not only provides ownership, but behind the scenes also initializes the NimBLE stack.

**2️⃣ Create an Advertiser Instance**: After initializing the NimBLE stack we create an advertiser instance by calling `get_advertising`, this will create a `&Mutex<BLEAdvertising>` instance. Heres the code:

```rust
let ble_advertiser = ble_device.get_advertising();
```

Note how `BLEAdvertising` is wrapped in a `Mutex`. This means that every time we want to access it, we need to obtain a `lock`.

**3️⃣ Configure Device Advertising Data:** In this example, we are not going to advertise any specific data. However, at minimum we need to define what our advertiser name is. To do this, there exists the `set_data` method for `BLEAdvertising` that takes a single argument of `BLEAdvertisementData`. For that we need to create an instance of `BLEAdvertisementData` using it's `new` method, then specify a name using the `name` method. The name method takes a single `&str` argument.

```rust
 // Specify Advertiser Name
 let mut ad_data = BLEAdvertisementData::new();
 ad_data.name("ESP32-C3 Peripheral");

 // Configure Advertiser with Specified Data
 ble_advertiser.lock().set_data(&mut ad_data).unwrap();
```

**4️⃣ Configure Advertiser Settings**: With advertiser instance we created we can now configure the advertiser parameters discussed earlier; **advertisement type**, **discoverable mode**, and **scan response**. These parameters can all be configured by calling `BLEAdvertising` methods on the `ble_advertiser` instance we created.

To set the **advertisement type,** there exists the `advertisement_type` method that takes a single `ConnMode` enum argument that specifies the connection mode. We are going to choose the `Non` option which means that our device is **non-connectable**. This is because we don't want any **central** devices to connect to us.

To set the **discoverable mode** there exists the `disc_mode` method that takes a single `DiscMode` enum argument specifying the discoverable mode. We are going to choose the `Gen` option which means that our device is in **general discoverable** mode. This means our device will continuously broadcasts advertising packets to advertise its presence to nearby devices.

Finally, we can specify if we will allow **scan responses**. This is done using the `scan_response` method which takes a single `bool` argument. We'll simply set this to `false`. Note that if we were to allow **scan responses** we would have needed to specify the data that would result from a **scan response**. For that we could have used the `set_raw_scan_response_data` method that is part of `BLEAdvertising`**.** Here's the code for our configuration:

```rust
ble_advertiser
    .lock()
    .advertisement_type(ConnMode::Non)
    .disc_mode(DiscMode::Gen)
    .scan_response(false);
```

That's it for configuration!

### **📱 Application Code**

**Start Advertising**: All we have to do now is start the advertising process. This is done by calling the `BLEAdvertising` `start` method. `start` doesn't take any parameters. After that we enter a `loop` to keep the application running. A delay is included in the `loop` to avoid the watchdog from triggering due to lack of activity.

```rust
// Start Advertising
ble_advertiser.lock().start().unwrap();
println!("Advertisement Started");
loop {
    // Keep Advertising
    // Add delay to prevent watchdog from triggering
    FreeRtos::delay_ms(10);
}
```

## **📱Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo.

```rust
use esp32_nimble::{
    enums::{ConnMode, DiscMode},
    BLEAdvertisementData, BLEDevice,
};
use esp_idf_hal::delay::FreeRtos;
use esp_idf_sys as _;

fn main() {
    esp_idf_svc::sys::link_patches();

    // Take ownership of device
    let ble_device = BLEDevice::take();
    // Obtain handle for advertiser
    let ble_advertiser = ble_device.get_advertising();

    // Specify Advertiser Name
    let mut ad_data = BLEAdvertisementData::new();
    ad_data.name("ESP32-C3 Peripheral");

    // Configure Advertiser with Specified Data
    ble_advertiser.lock().set_data(&mut ad_data).unwrap();

    // Make Advertiser Non-Connectable
    // Set Discovery Mode to General
    // Deactivate Scan Responses
    ble_advertiser
        .lock()
        .advertisement_type(ConnMode::Non)
        .disc_mode(DiscMode::Gen)
        .scan_response(false);

    // Start Advertising
    ble_advertiser.lock().start().unwrap();
    println!("Advertisement Started");
    loop {
        // Keep Advertising
        // Add delay to prevent watchdog from triggering
        FreeRtos::delay_ms(10);
    }
}
```

## **Conclusion**

This post introduced how to start working with BLE on the ESP32-C3 with Rust. This was by using the `esp32-nimble` crate in a standard library development environment using the `esp-idf-hal` . In this post, the ESP32-C3 was configured as a peripheral device as an advertising BLE device. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]