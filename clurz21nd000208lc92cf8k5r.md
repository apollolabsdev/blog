---
title: "Embedded Rust Bluetooth on ESP: Secure BLE Client"
datePublished: Tue Apr 09 2024 05:59:44 GMT+0000 (Coordinated Universal Time)
cuid: clurz21nd000208lc92cf8k5r
slug: embedded-rust-bluetooth-on-esp-secure-ble-client
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712585919445/eb243d8d-2f3d-400d-add3-eb1ebad4a793.png
tags: tutorial, iot, rust, esp32, embedded-systems

---

> ***This post is the fourth of a multi-part series where I'm exploring the use of Bluetooth Low Energy along embedded Rust on the ESP32. Information in this post might rely on knowledge presented in past posts.***

1. [**Embedded Rust Bluetooth on ESP: BLE Scanner**](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-scanner)
    
2. [**Embedded Rust Bluetooth on ESP: BLE Advertiser**](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-advertiser)
    
3. [**Embedded Rust Bluetooth on ESP: BLE Server**](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-server)
    
4. [**Embedded Rust Bluetooth on ESP: BLE Client**](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-client)
    

## **Introduction**

In this post, we're going to build on the [previous post](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-client) to instead create a secure BLE client. We're going to create a **central** device that will assume the role of a **client** upon connection establishment. Afterward, the connection will be secured. Similar to past posts, the code will be built using the [**esp32-nimble** crate.](https://crates.io/crates/esp32-nimble/0.6.0)

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

After a connection is established is can be secured by encrypting the connection. This is such that data exchanged is not in plain text. Encryption requires that keys are exchanged as part of the process in securing a connection. There are two terms involved in this process, **pairing** and **bonding**. **Pairing** refers to the process of exchanging and keys for encryption purposes. **Bonding** on the other hand is in reference to encryption for future reconnections. Additionally, **bonding** always happens after **pairing** is completed.

There are several steps that a BLE connection goes through to secure a channel:

1. **Establish I/O Capabilities:** As part of the **pairing** process both devices have to establish what their I/O capabilities are. In addition, among other things, devices exchange supported security features and whether or not bonding is requested. I/O capabilites include the following options:
    
    * **DisplayOnly**: The peer only has a display
        
    * **DisplayYesNo**: The peer has a display and the option to select “yes” or “no”
        
    * **KeyboardOnly**: The peer has keyboard only
        
    * **NoInputNoOutput**: The peer has no input and no output capabilities
        
    * **KeyboardDisplay**: The peer has keyboard and display capabilities
        
2. **Choose Pairing Method:** In order to complete **pairing**, devices need to agree on a pairing method to exchange keys. The **pairing** method chosen is decided based on a Out of Bound (OOB) flag, the Man-In-The-Middle (MITM) flag, and the I/O capability chosen earlier. Pairing methods include (least secure to most secure):
    
    * **Just Works**: This method is the simplest and most convenient, where devices automatically exchange keys without any user interaction. However, it provides the lowest level of security and is vulnerable to man-in-the-middle attacks.
        
    * **Passkey Entry**: In this method, one device displays a randomly generated passkey, and the user must enter the same passkey on the other device. This ensures that the devices are physically present and aware of each other during the **pairing** process.
        
    * **Out-of-Band (OOB)**: OOB pairing involves using an external channel, such as NFC or QR codes, to exchange **pairing** information securely. This method provides a high level of security by leveraging an additional communication channel.
        
    * **Numeric Comparison**: Numeric Comparison pairing provides a high level of security by ensuring that both devices are aware of each other's identities. This method also makes sure that there is no interception or tampering during the **pairing** process. It requires active participation from the users to verify the displayed numeric values, which adds an extra layer of security against unauthorized access. In this method, the devices display a 6-digit number and the user selects “yes” or “no” to confirm the display.
        
3. **Bonding (Optional)**: After successful pairing, the devices have the option to enter into a **bonding** relationship. This involves securely storing the security information exchanged during **pairing**, such as the encryption keys and device identities, in persistent memory. As such, when the devices come into proximity again, they can use the stored security information to quickly establish a secure connection.
    

Now that the connection is secured, we further have the option to secure particular attributes. In the [BLE Server post](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-server), GATT operations were explained in which the supported operations by an attribute were defined through its properties. As such, there are additional properties available related to security for choice of authentication, authorization, and encryption. Authentication guarantees that a message has originated from a trusted party while authorization is a confirmation by the user to continue with the procedure. These attribute properties, however, are set by a server as the client would be accessing the attributes remotely.

Note that attributes can be associated with different levels of security. The level of security associated with an attribute, like GATT operations, is captured through its properties. This is something we'll be demonstrating in a secure server example in next week's post. This means that there could be several attributes available applying different security levels. A client in turn would only be able to access the attributes matching (or less than) the level of security is achieves.

In this post, we'll be appending the code in the [central client post](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-client) to make it's connection secure. As such, we'll be creating a **secure central client** with one **characteristic**. In that context, the code will take the following steps:

1. **(Added Step)** Configure device security capabilities and connection method for pairing
    
2. Scan and find a particular advertiser
    
3. Establish a connection
    
4. **(Added Step)** Secure the connection
    
5. Read a characteristic value every second
    

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `esp_idf_hal` crate to import delays and blocking abstractions.
    
* The `esp_idf_sys` crate since its needed.
    
* The `esp32_nimble` crate for the BLE abstractions.
    

```rust
use esp32_nimble::{enums::*, uuid128, BLEClient, BLEDevice};
use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::task::block_on;
use esp_idf_sys as _;
```

### **🎛 Initialization/Configuration Code**

**1️⃣ Obtain a handle for the BLE device**: Similar to the pattern we've seen in embedded Rust with peripherals, as part of the singleton design pattern, we first have to take ownership of the device peripherals. In this context, its the `BLEDevice` that we need to take ownership of. This is done using the `take()` associated method. Here I create a BLE device handler named `ble_device` as follows:

```rust
let ble_device = BLEDevice::take();
```

**2️⃣ Configure Device Security:** In this step we need to configure the device I/O capabilities for pairing and set the authorization mode. The authorization mode would determine the pairing method and whether bonding is required or not. The authorization mode is captured through several flags mentioned earlier configurable through the `esp32_nimble::enums::AuthReq` enum. From what it seems, the NimBLE crate supports at most the passkey entry method.

To configure security parameters, we first need to call the `security` method on the `BLEDevice` instance. This would return a `BLESecurity` type that renders access to the security configuration methods. The authorization mode is configured through the `set_auth` method where we pass a the `AuthReq` enum choice. `AuthReq::all` means setting all the flags and implies the passkey entry method and and bonding. We also need to set the IO capabilities through the `set_io_cap` method. For that, we need to pass a `SecurityIOCap` enum option which we pass `KeyboardOnly`. Now although we do not have a keyboard, there exists a method we'll see later where we can pass a passkey that we will hardcode. Here's the code:

```rust
// Configure Device Security
ble_device
    .security()
    .set_auth(AuthReq::all())
    .set_io_cap(SecurityIOCap::KeyboardOnly);
```

**3️⃣ Create a Scan Instance**: After initializing the NimBLE stack we create a scan instance by calling [`get_scan`](https://taks.github.io/esp32-nimble/esp32_nimble/struct.BLEDevice.html#method.get_scan), this will create a [`BLEScan`](https://taks.github.io/esp32-nimble/esp32_nimble/struct.BLEScan.html) instance. This instance would allow us to start looking for advertising servers. Heres the code:

```rust
let ble_scan = ble_device.get_scan();
```

**4️⃣ Configure Scan Parameters and Callback**: Now that we have a scan instance we can configure the scan parameters as done previously in the past [BLE scanner post](https://apollolabsblog.hashnode.dev/embedded-rust-bluetooth-on-esp-ble-scanner). One difference here is that the `on_result` method has been replaced by a `find_device` `async` method. In this case, to make things easier, rather than printing all advertising devices, we're going to look for and connect one particular peripheral device. `find_device` has two parameters; a `ms` duration defining how long to look for a particular device, and a closure passing a `BLEAdvertisedDevice` as a token. In this closure, we are extracting the advertising device names to see if they match the `name` we're looking for. `DEVICE_NAME` is a `const` defined earlier in the code reflecting the `&str` name.

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

5️⃣ **Instantiate Client and Define Behaviour On Connection to a Peripheral:** Upon identifying a device to connect to the `Option` should return a `Some` wrapping a `BLEAdvertisedDevice` . For the obtained device, we need to instantiate the `Client` and define connection behavior. We instantiate a `client` using the `BLEClientnew` method.

Using the `client` instance there exists a `on_connect` method for `BLEClient`. `on_connect` has one argument which is a closure that passes a handle for a `BLEClient` that contains the connected client information. In the closure body, upon connect, we'll print that we are connected and update the connection parameters. To update connection parameters, the `BLEClient` `update_conn_params` method is used. Here's the code:

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

6️⃣ **Establish a Connection:** Now that we've identified the device we want to connect to and the connection behavior, we can establish the connection. This is done by calling the `connect` method for the identified `device`. All we need to do is pass the device address as an argument which can be accessed using the `BLEAdvertisedDeviceaddr` method.

```rust
if let Some(device) = device {
    // Prior code from step 4

   // Connect to advertising device using its address
   client.connect(device.addr()).await.unwrap();

   // Remainder of code
}
```

**7️⃣ Secure the Connection:** After connection establishment, we need to secure the connection. To secure the connection we need to provide the passkey that will be entered in the process. This is done through the `on_passkey_request` method where a 6-digit passkey needs to be provided. This passkey should match the key entered at the other peer. Afterward, `secure_connection` an `async` method is called to establish a secure connection:

```rust
// Enter 123456 passkey on request
client.on_passkey_request(|| 123456);

// Secure Connection
client.secure_connection().await.unwrap();
```

That's it for configuration!

### **📱 Application Code**

The code in this section is identical to what has been developed in the BLE Client example. However, note what differs is that the remote characteristic security level can be different depending on its properties. To be able to access the characteristic, our client would need to have a matching security level.

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

**Read From Characteristic(s):** Now that we have access to the remote **characteristic** through the `characteristic` handle, we can perform any supported GATT operations. For this case, for a server we create we know that the characteristic would at least support a read operation. Using the `BLERemoteCharacteristicread_value` method we receive a `Result` wrapping a `Vec<u8>`. As indicated earlier we mentioned that we want to update the read value every 1 sec as follows:

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
use esp32_nimble::{enums::*, uuid128, BLEClient, BLEDevice};
use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::task::block_on;
use esp_idf_sys as _;

const DEVICE_NAME: &str = "iPhone";

fn main() {
    esp_idf_sys::link_patches();

    block_on(async {
        // Acquire BLE Device Handle
        let ble_device = BLEDevice::take();

        // Configure Device Security
        ble_device
            .security()
            .set_auth(AuthReq::all())
            .set_io_cap(SecurityIOCap::KeyboardOnly);

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

            // Enter 123456 passkey on request
            client.on_passkey_request(|| 123456);

            // Secure Connection
            client.secure_connection().await.unwrap();

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

This post introduced how to create a secure BLE client on the ESP32-C3 with Rust. This was by using the `esp32-nimble` crate in a standard library development environment using the `esp-idf-hal` . In this post, the ESP32-C3 was configured as a secure central device scanning for a peripheral server device containing a service. The device also assumes a client role after a connection is established and secures the connection with a passkey. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]