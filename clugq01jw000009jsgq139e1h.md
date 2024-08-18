---
title: "Embedded Rust Bluetooth on ESP: BLE Client"
datePublished: Mon Apr 01 2024 09:00:46 GMT+0000 (Coordinated Universal Time)
cuid: clugq01jw000009jsgq139e1h
slug: embedded-rust-bluetooth-on-esp-ble-client
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711909287026/046a366b-cbda-40cb-affc-d637bdedfd6a.png
tags: tutorial, iot, rust, bluetooth, embedded, esp32

---

> ***This post is the fourth of a multi-part series where I'm exploring the use of Bluetooth Low Energy along embedded Rust on the ESP32. Information in this post might rely on knowledge presented in past posts.***

1. [**Embedded Rust Bluetooth on ESP: BLE Scanner**](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-scanner)
    
2. [**Embedded Rust Bluetooth on ESP: BLE Advertiser**](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-advertiser)
    
3. [**Embedded Rust Bluetooth on ESP: BLE Server**](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-server)
    

## **Introduction**

In this post, we're going to build on the [previous post](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-server) to instead create a BLE client. We're going to create a **central** device that will assume the role of a **client** upon connection establishment. Similar to past posts, the code will be built using the [**esp32-nimble** crate.](https://crates.io/crates/esp32-nimble/0.6.0)

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
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    
    #### **🔌 Connections**
    
    No connections are required for this example.
    

## **👨‍🎨 Software Design**

In our example, we'll be creating a **central client** with one **characteristic** that can be **read** from. In that context, after configuring our device, the code will take the following steps:

1. Scan and find a particular advertiser
    
2. Establish a connection
    
3. Read the characteristic value every second
    

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `esp_idf_hal` crate to import delays and blocking abstractions.
    
* The `esp_idf_sys` crate since its needed.
    
* The `esp32_nimble` crate for the BLE abstractions.
    

```rust
use esp32_nimble::{uuid128, BLEClient, BLEDevice};
use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::task::block_on;
use esp_idf_sys as _;
```

### **🎛 Initialization/Configuration Code**

**1️⃣ Obtain a handle for the BLE device**: Similar to the pattern we've seen in embedded Rust with peripherals, as part of the singleton design pattern, we first have to take ownership of the device peripherals. In this context, its the `BLEDevice` that we need to take ownership of. This is done using the `take()` associated method. Here I create a BLE device handler named `ble_device` as follows:

```rust
let ble_device = BLEDevice::take();
```

**2️⃣ Create a Scan Instance**: After initializing the NimBLE stack we create a scan instance by calling [`get_scan`](https://taks.github.io/esp32-nimble/esp32_nimble/struct.BLEDevice.html#method.get_scan), this wi[ll creat](https://taks.github.io/esp32-nimble/esp32_nimble/struct.BLEDevice.html#method.get_scan)e a [`BLEScan`](https://taks.github.io/esp32-nimble/esp32_nimble/struct.BLEScan.html) instance[. This](https://taks.github.io/esp32-nimble/esp32_nimble/struct.BLEScan.html) instance would allow us to start looking for advertising servers. Heres the code:

```rust
let ble_scan = ble_device.get_scan();
```

**3️⃣ Configure Scan Parameters and Callback**: Now that we have a scan instance we can configure the scan parameters as done previously in the past [BLE scanner post](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-scanner). One difference here is that the `on_result` method has been replaced by a `find_device` `async` method. In this case, to make things easier, rather than printing all advertising devices, we're going to look for and connect one particular peripheral device. `find_device` has two parameters; a `ms` duration defining how long to look for a particular device, and a closure passing a `BLEAdvertisedDevice` as a token. In this closure, we are extracting the advertising device names to see if they match the `name` we're looking for. `DEVICE_NAME` is a `const` defined earlier in the code reflecting the `&str` name.

Upon completion of the scan process, note that the device would be a handle for a `BLEAdvertisedDevice` type wrapped in an `Option`. Note also how the **scan interval** chosen is 100ms and the **scan window** is 99ms.

```rust
let device = ble_scan
    .active_scan(true)
    .interval(100)
    .window(99)
    .find_device(10000, |device| device.name() == DEVICE_NAME)
    .await
    .unwrap();
```

4️⃣ **Instantiate Client and Define Behaviour On Connection to a Peripheral:** Upon identifying a device to connect to the `Option` should return a `Some` wrapping a `BLEAdvertisedDevice` . For the obtained device, we need to instantiate the `Client` and define connection behavior. We instantiate a `client` using the `BLEClient` `new` method.

Using the `client` instance there exists a `on_connect` method for `BLEClient`. `on_connect` has one argument which is a closure that passes a handle for a `BLEClient` that contains the connected client information. In the closure body, upon connect, we'll print that we are connected and update the connection parameters. To update connection parameters, the `BLEClient` `update_conn_params` method is used and has the following signature:

```rust
pub fn update_conn_params(
    &mut self,
    min_interval: u16,
    max_interval: u16,
    latency: u16,
    timeout: u16
) -> Result<(), BLEError>
```

`min_interval` is a value for the minimum **connection interval** in ms, `max_interval` is a value for the maximum **connection interval** in ms, `latency` is expressed as the number of intervals to skip if theres no data to transmit, and `timeout` is the **supervision timeout** time in 10ms units. Here's the code:

```rust
if let Some(device) = device {
    // Create Client Handle
    let mut client = BLEClient::new();

    // Define Connect Behaviour
    client.on_connect(|client| {    
    // Update Connect Parameters on Connect
        client.update_conn_params(120, 250, 0, 60).unwrap();
        println!("Connected to {}", DEVICE_NAME);
    });

   // Remainder of code
}
```

5️⃣ **Establish a Connection:** Now that we've identified the device we want to connect to and the connection behavior, we can establish the connection. This is done by calling the `connect` method for the identified `device`. All we need to do is pass the device address as an argument which can be accessed using the `BLEAdvertisedDevice` `addr` method.

```rust
if let Some(device) = device {
    // Prior code from step 4

   // Connect to advertising device using its address
   client.connect(device.addr()).await.unwrap();

   // Remainder of code
}
```

That's it for configuration!

### **📱 Application Code**

**Identify Services and Characteristics available:** Since we are operating as a **Client**, we need to get the "remote" **services** and **characteristics** that are available at the **server**. The UUIDs for these services should be known beforehand. Meaning, the **client** should be familiar with the **services** the **server** offers and their UUIDs to identify them.

In the first step, we need to obtain a **service** and then follow it by obtaining the **characteristic(s)** corresponding to that **service**. For the connected `client` of `BLEClient` type, there exists a `get_service` method that takes a UUID as a parameter. `get_service` is an `async` method returning a `Result` wrapping a `BLERemoteService`. In turn, `BLERemoteService` has a `get_characteristic` method with a similar signature that returns a `BLERemoteCharacteristic`. Heres the corresponding code:

```rust
// Seek and Create Handle for the BLE Remote Server Service corresponding to the UUID
let service = client
        .get_service(uuid128!("9b574847-f706-436c-bed7-fc01eb0965c1"))
        .await
        .unwrap();

// Seek and Create Handle for the BLE Remote Server Characteristic corresponding to the UUID
let uuid = uuid128!("681285a6-247f-48c6-80ad-68c3dce18585");
let characteristic = service.get_characteristic(uuid).await.unwrap();
```

**Read From Characteristic(s):** Now that we have access to the remote **characteristic** through the `characteristic` handle, we can perform any supported GATT operations. For this case, for a server we create we know that the characteristic would at least support a read operation. Using the `BLERemoteCharacteristic` `read_value` method we receive a `Result` wrapping a `Vec<u8>`. As indicated earlier we mentioned that we want to update the read value every 1 sec as follows:

```rust
loop {
    // Read & Print Characteristic Value
    let value = characteristic.read_value().await.unwrap();
    println!("Read Value: {:?}", value);

    // Wait 1 second before reading again
    FreeRtos::delay_ms(1000);
}
```

> 📝 **Note**: While not incorporated in this code, it would be beneficial to identify the supported GATT operations for a characteristic beforehand. For that, there exists `BLERemoteCharacteristic` methods like `can_read`, `can_notify`, `can_write`, and so on that return a `bool` indicating if the operation is supported.

## 🧪 Testing

In order to test this code, you can use [nRF connect](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile) mobile app or the [bluefruit connect](https://learn.adafruit.com/bluefruit-le-connect/ios-setup) mobile app. In either app, you will be able to create an advertising **peripheral** as a **server** and define **services** and **characteristics**. Be mindful that the services you create need to have the same UUID defined in our code.

> 📝 **Note**: When testing, make sure to change `DEVICE_NAME` accordingly.

## **📱Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo.

```rust
use esp32_nimble::{uuid128, BLEClient, BLEDevice};
use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::task::block_on;
use esp_idf_sys as _;

const DEVICE_NAME: &str = "Device Name";

fn main() {
    esp_idf_sys::link_patches();

    block_on(async {
        // Acquire BLE Device Handle
        let ble_device = BLEDevice::take();
        // Acquire Scan Handle
        let ble_scan = ble_device.get_scan();
        // Scan and Find Device/Server
        let device = ble_scan
            .active_scan(true)
            .interval(100)
            .window(99)
            .find_device(10000, |device| device.name() == DEVICE_NAME)
            .await
            .unwrap();

        if let Some(device) = device {
            // Create Client Handle
            let mut client = BLEClient::new();

            // Define Connect Behaviour
            client.on_connect(|client| {
                // Update Connect Parameters on Connect
                client.update_conn_params(120, 250, 0, 60).unwrap();
                println!("Connected to {}", DEVICE_NAME);
            });

            // Connect to advertising device using its address
            client.connect(device.addr()).await.unwrap();

            // Seek and Create Handle for the BLE Remote Server Service corresponding to the UUID
            let service = client
                .get_service(uuid128!("9b574847-f706-436c-bed7-fc01eb0965c1"))
                .await
                .unwrap();

            // Seek and Create Handle for the BLE Remote Server Characteristic corresponding to the UUID
            let uuid = uuid128!("681285a6-247f-48c6-80ad-68c3dce18585");
            let characteristic = service.get_characteristic(uuid).await.unwrap();

            loop {
                // Read & Print Characteristic Value
                let value = characteristic.read_value().await.unwrap();
                println!("Read Value: {:?}", value);

                // Wait 1 second before reading again
                FreeRtos::delay_ms(1000);
            }
        }
    });
}
```

## **Conclusion**

This post introduced how to create a BLE client on the ESP32-C3 with Rust. This was by using the `esp32-nimble` crate in a standard library development environment using the `esp-idf-hal` . In this post, the ESP32-C3 was configured as a central device scanning for a peripheral server device containing a service. The device also assumes a client role after a connection is established. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]