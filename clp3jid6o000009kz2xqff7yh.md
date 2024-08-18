---
title: "Edge IoT with Rust on ESP: MQTT Publisher"
datePublished: Sat Nov 18 2023 04:19:32 GMT+0000 (Coordinated Universal Time)
cuid: clp3jid6o000009kz2xqff7yh
slug: edge-iot-with-rust-on-esp-mqtt-publisher
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1700280418276/0260899a-53e1-436e-bd7f-7e1653bcdee8.png
tags: tutorial, iot, beginners, rust, esp32

---

> ***This blog post is the sixth of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [**Edge IoT with Rust on ESP: Connecting WiFi**](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi)
    
2. [**Edge IoT with Rust on ESP: HTTP Client**](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-http-client)
    
3. [**Edge IoT with Rust on ESP: HTTP Server**](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-http-server)
    
4. [Edge IoT with Rust on ESP: NTP](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-ntp)
    
5. [Edge IoT with Rust on ESP: MQTT Subscriber](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-mqtt-subscriber)
    

## Introduction

In last week's post, MQTT was used to subscribe to a topic published by another client. This was achieved in part using the `mqtt` abstractions in the ESP Rust `esp_idf_svc` crate. Using the same crate, we can leverage abstractions to publish MQTT topics as well. In this post, we're going to look at how to configure an ESP using Rust to publish a topic to an MQTT broker.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Basic familiarity with WiFi & MQTT.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/381659210200887297).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    

## **👨‍🎨 Software Design**

> **Note:** The code in this post is largely similar to last week's post. Also please refer to the same section of the [prior post](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-mqtt-subscriber) for additional background.

In this post, we are going to configure the ESP to publish a topic to an MQTT broker. The steps include the following:

1. Configure and Connect to WiFi
    
2. Configure MQTT
    
3. Create Client Instance and Define Event Behaviour
    
4. Wait for Broker Connection
    
5. Publish a Topic
    

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation, the following crates are required:

* The `anyhow` crate for error handling.
    
* The `esp_idf_hal` crate to import the peripherals.
    
* The `esp_idf_svc` crate to import the device services (wifi in particular).
    
* The `embedded_svc` crate to import the needed `wifi` and `mqtt::client service` traits.
    
* The `std::thread` and `std::time` for sleep behavior and time measurement.
    

```rust
use anyhow;
use embedded_svc::mqtt::client::Event;
use embedded_svc::mqtt::client::QoS;
use embedded_svc::wifi::{AuthMethod, ClientConfiguration, Configuration};
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::mqtt::client::{EspMqttClient, MqttClientConfiguration};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};
use std::{thread::sleep, time::Duration};
```

### **🎛 Initialization/Configuration Code**

1️⃣ **Obtain a handle for the device peripherals**: Similar to all past blog posts, in embedded Rust, as part of the singleton design pattern, we first have to take the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

2️⃣ **Configure and Connect to WiFi**: this involves the same steps that were done in the [wifi post](https://apollolabsblog.hashnode.dev/edge-iot-with-rust-on-esp-connecting-wifi).

3️⃣ **Create the MQTT Configuration Handle:** Within `esp_idf_svc::mqtt::Client` there exists an `MqttClientConfiguration` abstraction. This is the abstraction needed to configure MQTT. `MqttClientConfiguration` contains a `default` method allowing us to configure MQTT with a `default` configuration. Following that we create an `mqtt_config` handle as follows:

```rust
let mqtt_config = MqttClientConfiguration::default();
```

Note that if you were to use MQTT with a more advanced configuration, then this is the abstraction you need to update. This includes things like adding certificates for secure connections, defining client IDs, hostnames, and passwords among other things. Refer to the [documentation](https://esp-rs.github.io/esp-idf-svc/esp_idf_svc/mqtt/client/struct.MqttClientConfiguration.html) for the full list of members.

That's it for Configuration!

### **📱 Application Code**

1️⃣ **Create Client Instance and Define Event Behaviour:** Within the `EspMqttClient` abstraction, there is a `new` method that is used to create an instance of `EspMqttClient`. `new` has the following signature:

```rust
pub fn new(
    url: &str,
    conf: &MqttClientConfiguration<'_>,
    callback: impl for<'b> FnMut(&'b Result<Event<EspMqttMessage<'b>>, EspError>) + Send + 'a
) -> Result<Self, EspError>
```

Note there are three parameters a `url`, a `MqttClientConfiguration` configuration, and a `callback`. The `url` as would be expected is the `url` of the broker and `conf` is the configuration we created earlier. `callback` is a closure where we would need to define `Event` behavior. Meaning, every time there is a broker `Event`, we need to know what it is and react accordingly. `Event` is an enum with the following signature:

```rust
pub enum Event<M> {
    BeforeConnect,
    Connected(bool),
    Disconnected,
    Subscribed(MessageId),
    Unsubscribed(MessageId),
    Published(MessageId),
    Received(M),
    Deleted(MessageId),
}
```

We're not going to react to all of the events but rather only the ones we care about. Those would be `Connected` that conveys that we are connected to a broker, `Suscribed` which means we are successfully subscribed to a topic, and `Recieved` that conveys we received a message from the broker. Other than that, we can simply print the message event.

For this application, we are going to use the [HiveMQ broker service](https://www.hivemq.com/demos/websocket-client/). The service broker `url` is `mqtt://`[`broker.mqttdashboard.com`](http://broker.mqttdashboard.com) . Before running this code, you would need to go to the [HiveMQ](https://www.hivemq.com/demos/websocket-client/) page and click connect. While connected, on the same page, you can define a topic to publish and a message then click publish. This would publish the message for any subscriber to read.

Following all of the above, a `client` handle is created as follows:

```rust
let mut client = EspMqttClient::new(
    "mqtt://broker.mqttdashboard.com",
    &mqtt_config,
    move |message_event| {
        match message_event.as_ref().unwrap() {
            Event::Connected(state) => println!("Connected"),
            Event::Subscribed(id) => println!("Subscribed to {} id", id),
            Event::Received(msg) => {
                if msg.data() != [] {
                    println!("Recieved {}", std::str::from_utf8(msg.data()).unwrap())
                }
            }
            _ => println!("{:?}", message_event.as_ref().unwrap()),
        };
    },
)?;
```

Note how the match statement matches what we mentioned earlier. The only additional thing to mention here is how we are handling the received message. `Recieve` wraps a `EspMqttMessage` type. `EspMqttMessage` has [several methods](https://esp-rs.github.io/esp-idf-svc/esp_idf_svc/mqtt/client/struct.EspMqttMessage.html) to process the incoming message. One of the methods is `data` that recovers the received data in a `&[u8]` . To print it out the received message it needs to be converted to a string using `std::str::from_utf8`.

2️⃣ **Subscribe to Topic:** While we've defined the event behavior, we haven't subscribed to a topic yet. For that, we need to call the `subscribe` method on `client` that is of type `EspMqttClient`. Here's the code:

```rust
client.subscribe("testtopic/1", QoS::AtLeastOnce)?;
```

`testtopic/1` is the default topic on [HiveMQ](https://www.hivemq.com/demos/websocket-client/). If you modify the topic there then you need to update it here.

When running the code, don't forget to connect the broker and then hit publish to see messages received by the ESP.

3️⃣ **Publish a Topic:** To publish a topic, we need to call the `publish` method in `client` that is of type `EspMqttClient`. `publish` has 4 parameters; a `&str` topic parameter, a `QoS` type qos parameter, a `bool` retain parameter, and a `&[u8]` payload parameter. In the code we can create a `count` variable that is incremented and published under a topic `mytopic/1` every second. Here's the code:

```rust
let mut count:u8 = 0;

loop {
  let msg = format!("Counter {}", count);
    // Keep waking up device to avoid watchdog reset
    client.publish("mytopic/1", QoS::AtLeastOnce, true, msg.as_bytes())?;
    count += 1;
    sleep(Duration::from_millis(1000));
}
```

## **📱Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/381659210200887297).

```rust
use anyhow;
use embedded_svc::mqtt::client::Event;
use embedded_svc::mqtt::client::QoS;
use embedded_svc::wifi::{AuthMethod, ClientConfiguration, Configuration};
use embedded_svc::mqtt::client::Connection;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_svc::eventloop::EspSystemEventLoop;
use esp_idf_svc::mqtt::client::{EspMqttClient, MqttClientConfiguration};
use esp_idf_svc::nvs::EspDefaultNvsPartition;
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};
use std::{thread::sleep, time::Duration};

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

    // Set up handle for MQTT Config
    let mqtt_config = MqttClientConfiguration::default();

    // Create Client Instance and Define Behaviour on Event
    let mut client = EspMqttClient::new(
        "mqtt://broker.mqttdashboard.com",
        &mqtt_config,
        move |message_event| {
            match message_event.as_ref().unwrap() {
                Event::Connected(_) => println!("Connected"),
                Event::Subscribed(id) => println!("Subscribed to {} id", id),
                Event::Received(msg) => {
                    if msg.data() != [] {
                        println!("Recieved {}", std::str::from_utf8(msg.data()).unwrap())
                    }
                }
                _ => println!("{:?}", message_event.as_ref().unwrap()),
            };
        },
    )?;

    // Subscribe to MQTT Topic
    client.subscribe("testtopic/1", QoS::AtLeastOnce)?;
    
    let mut count:u8 = 0;

    loop {
      let msg = format!("Counter {}", count);
        // Keep waking up device to avoid watchdog reset
        client.publish("mytopic/1", QoS::AtLeastOnce,true,msg.as_bytes())?;
        count += 1;
        sleep(Duration::from_millis(1000));
    }
}
```

## Conclusion

MQTT is a popular protocol in IoT applications due to its lightweight nature. As such, there is widespread support for MQTT implementations in many embedded devices. This post introduced how to set up MQTT publisher on ESP using Rust and the `esp_idf_svc`. Have any questions? Share your thoughts in the comments below 👇.

%%[substart]