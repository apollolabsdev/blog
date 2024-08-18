---
title: "ESP32 Standard Library Embedded Rust: UART Communication"
datePublished: Thu Jul 20 2023 21:22:54 GMT+0000 (Coordinated Universal Time)
cuid: clkbnsc7w000309lf517shdf9
slug: esp32-standard-library-embedded-rust-uart-communication
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1689802897346/511faf0e-0303-4e74-8db4-9cc23ba4cacb.png
tags: tutorial, iot, rust, embedded, rust-lang

---

> ***This blog post is the second of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [ESP32 Standard Library Embedded Rust: GPIO Control](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-gpio-control)
    

## Introduction

UART serial communication is a valuable protocol for facilitating direct device-to-device communication. In the past, it was commonly employed in development scenarios to log output to a PC. However, contemporary microcontrollers now incorporate sophisticated debug functionalities such as the instruction trace macrocell (also known as ITM), which do not necessarily rely on the device's peripheral resources. Nevertheless, UART still finds its way into several serial communication applications.

In this post, I will be configuring and setting up UART communication for the ESP32C3 to create a loopback application with a bit of a spin. I will be incorporating a simple encoding scheme (XOR Cipher) that garbles the message before its sent. Upon receiving the garbled message, it will then be decoded and printed to the console.

%%[substart] 

### **📚** Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with [**UART communication basics**](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter).
    
* Familiarity with [XOR Ciphers](https://en.wikipedia.org/wiki/XOR_cipher).
    

### **💾** Software Setup

All the code presented in this post is available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [here](https://wokwi.com/projects/370479605507228673).

### **🛠** Hardware Setup

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg align="center")
    

#### **🔌** Connections

**📝 Note**

> All connection details are also shown in the [Wokwi example](https://wokwi.com/projects/370479605507228673).

Connections include the following:

* gpio5 wired to gpio6 on the devkit.
    

## **👨‍🎨** Software Design

In the application developed in this post, a text message will be sent from the ESP32 back to itself. In order to to achieve this, the ESP32 transmit pin is wired externally to the receive pin (a.k.a. loopback). However, ahead of sending the text, I will be encoding the message using an [XOR Cipher](https://en.wikipedia.org/wiki/XOR_cipher). The XOR cipher encrypts a text string by applying the bitwise XOR operator to each character using a specified key. To decrypt the encrypted output and revert it back to its original form, one simply needs to reapply the XOR operation with the same key. This process effectively removes the cipher and restores the plaintext. As such, the key is a value that is shared between the transmitter and the receiver.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689783603858/a8e73705-d959-405d-ab3d-02bc163bd392.png align="center")

The figure above demonstrates the algorithm flow. As such, here are the steps the algorithm would go through:

1. Encode the message to transmit using a bitwise XOR operator.
    
2. Transmit the encoded message over UART
    
3. Recover the encoded received UART message
    
4. Decode the received message to recover the plaintext
    

The above might seem straightforward, but there is a caveat. While UART supports different data lengths in a single frame ([5 bits - 8 bits](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/uart/config/enum.DataBits.html)), byte-sized data will be the most convenient to deal with. As such, the code will be developed to accommodate byte-sized data.

Let's now jump into implementing this algorithm.

## **👨‍💻** Code Implementation

### **📥** Crate Imports

In this implementation, one crate is required as follows:

* The `esp_idf_hal` crate to import the needed device hardware abstractions.
    

```rust
use esp_idf_hal::delay::BLOCK;
use esp_idf_hal::gpio;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::prelude::*;
use esp_idf_hal::uart::*;
```

**📝 Note**

> Each of the crate imports needs to have a corresponding dependency in the Cargo.toml file. These are typically provided by the template.

### **🎛** Peripheral Configuration Code

Ahead of our application code, peripherals are configured through the following steps:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to `take` the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

**📝 Note**

> While I didn't mention it explicitily, in the `std` template for the `esp_idf` a `esp_idf_sys::link_patches();` line exists at the beginning of the `main` function. This line is necessary to make sure patches to the ESP-IDF are linked to the final executable.

**2️⃣ Obtain handles for the tx and rx pins**: for convenience, we obtain handles for the pins that are going to be configured as `tx` and `rx` in the UART peripheral. In the ESP32C3, any pin can be used for `rx` and `tx`. As such, I chose `gpio5` to be used as the `tx` pin and `gpio6` will be used as the `rx` pin:

```rust
let tx = peripherals.pins.gpio5;
let rx = peripherals.pins.gpio6;
```

Note that at this point the pins are not configured yet, we merely created handles for them. They yet need to be configured to be connected to the internal UART peripheral of the ESP32.

3️⃣ **Obtain a handle and configure the UART peripheral**: In order to configure UART, there is a `UartDriver` abstraction in the [esp-idf-hal](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/uart/struct.UartDriver.html) that contains a `new` method to create a UART instance. The `new` method has the following signature:

```rust
pub fn new<UART: Uart>(
    uart: impl Peripheral<P = UART> + 'd,
    tx: impl Peripheral<P = impl OutputPin> + 'd,
    rx: impl Peripheral<P = impl InputPin> + 'd,
    cts: Option<impl Peripheral<P = impl InputPin> + 'd>,
    rts: Option<impl Peripheral<P = impl OutputPin> + 'd>,
    config: &Config
) -> Result<Self, EspError>
```

as shown, the `new` method has 6 parameters. The first `uart` parameter is a `UART` peripheral type. The second `tx` parameter is a transmit `OutputPin` type. The third `rx` parameter is a receive `OutputPin` type. The fourth `cts` and fifth `rts` parameters are for pins used for control flow which we won't be using. Finally, the sixth parameter is a UART configuration type `Config`. As such, we create an instance for `uart1` with handle name `uart` as follows:

```rust
let config = config::Config::new().baudrate(Hertz(115_200));
let uart = UartDriver::new(
    peripherals.uart1,
    tx,
    rx,
    Option::<gpio::Gpio0>::None,
    Option::<gpio::Gpio1>::None,
    &config,
)
.unwrap();
```

A few things to note:

* Notice that `uart1` is being used. This is because `uart0` is typically used for loading firmware and logging. Therefore `uart1` is the one recommended to use in an application.
    
* The `cts` and `rts` parameters require an `Option`, here [turbofish syntax](https://matematikaadit.github.io/posts/rust-turbofish.html) is used to help the compiler infer a type. Any `gpio` type can be used here since the `None` option is selected. Alternatively, one can also use the `AnyIOPin` generic pin type in the `gpio` module.
    
* `Config` is a [`uart::config`](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/uart/config/struct.Config.html) type that provides configuration parameters for UART. `Config` contains a method `new` to create an instance with default parameters. Afterward, there are configuration parameters adjustable through various methods. A full list of those methods can be found in the [documentation](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/uart/config/struct.Config.html). In this case, I only used the `baudrate` method to change the baud rate configuration.
    

That's it for configuration.

### **📱** Application Code

Following the design described earlier, there is a couple of things we need to do before building the application. First, is to set up constants for the `MESSAGE` string that is going to be sent and the `KEY` used for the cipher:

```rust
// Message to Send
const MESSAGE: &str = "Hello";
// Key
const KEY: u8 = 212;
```

For the `KEY` any value between 1 and 255 would work. I randomly picked `212`. Also, since we're going to be sending and receiving byte-size data, the receiver would need to buffer incoming bytes until a full transmission is complete. As such, we need to instantiate a Vector that I gave the handle name `rec` as follows:

```rust
// Create a Vector Instance to buffer the recieved bytes
let mut rec = Vec::new();
```

Next, I print to the console the message that will be sent in text and also encoded values for each of the characters using the `as_bytes` method:

```rust
// Print Message to be Sent
println!("Sent Message in Text: {}", MESSAGE);
println!("Sent Message in Values: {:?}", MESSAGE.as_bytes());
```

Before sending the message over UART, recall that that XOR cipher needs to be applied. For that, I used iterators to break down the message into bytes and the `map` method to apply a bitwise XOR with the `KEY` for each of the bytes. The resulting `u8` Vector is bound to the `gmsg` handle and the new "garbled" values print to the console:

```rust
// Garble Message
let gmsg: Vec<u8> = MESSAGE.as_bytes().iter().map(|m| m ^ KEY).collect();
// Print Garbled Message
println!("Sent Garbled Message Values: {:?}", gmsg);
```

Now we can send the garbled values over UART using the `uart` handle instantiated earlier. However, we need to sent the message one byte at a time until the full message is received. As such, we're going to need to iterate over `gmsg` transmit each byte `u8` value using the `uart` `write` method. Then the `read` method can be used to receive the transmitted byte. However, the received byte needs to be buffered in the `rec` Vector until the transmission of the full message is complete. This is the code that achieves that:

```rust
// Send Garbled Message u8 Values One by One until Full Array is Sent
for letter in gmsg.iter() {
    // Send Garbled Message Value
    uart.write(&[*letter]).unwrap();

    // Recieve Garbled Message Value
    let mut buf = [0_u8; 1];
    uart.read(&mut buf, BLOCK).unwrap();

    // Buffer Recieved Message Value
    rec.extend_from_slice(&buf);
}
```

Some notes:

* The `read` method requires a mutable `u8` slice and a `u32` delay value as arguments. The delay value reflects how long the `read` method needs to block operations before the `read` operation is complete. `buf` is the handle created for the slice. On the other hand, `BLOCK` is passed for the delay. `BLOCK` is essentially a `u32` `const` that reflects a really large value to keep blocking operations until the `read` operation is completed.
    
* `extend_from_slice` is a `Vec` method that allows us to append slice to `rec` .
    

Following the complete transmit/receive operation, to confirm that the message was received correctly, I print it to the console:

```rust
// Print Recieved Garbled Message Values
println!("Recieved Garbled Message Values: {:?}", rec);
```

Then the encoded message needs to be decoded in the same manner it was encoded:

```rust
// UnGarble Message
let ugmsg: Vec<u8> = rec.iter().map(|m| m ^ KEY).collect();
println!("Ungarbled Message in Values: {:?}", ugmsg);
```

In the final step, the recovered plaintext message is printed:

```rust
// Print Recovered Message
if let Ok(rmsg) = std::str::from_utf8(&ugmsg) {
    println!("Recieved Message in Text: {:?}", rmsg);
};
```

here the `from_utf8` method in `std::str` is used. `from_utf8` allows us to convert `Vec<u8>` type or slice of bytes to a string slice `&str`. Also the `if let` syntax is used as an alternative concise option for `match` . Here we're pattern-matching only the `Ok` option from the `Result` enum returned by `from_utf8` . For the interested, more on `if let` [here](https://doc.rust-lang.org/rust-by-example/flow_control/if_let.html).

## **📱** Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [here](https://wokwi.com/projects/370479605507228673).

```rust
use esp_idf_hal::delay::BLOCK;
use esp_idf_hal::gpio;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::prelude::*;
use esp_idf_hal::uart::*;

// Message to Send
const MESSAGE: &str = "Hello";
// Key Value (Can be any value from 1 to 255)
const KEY: u8 = 212;

fn main() {
    esp_idf_sys::link_patches();

    let peripherals = Peripherals::take().unwrap();
    let tx = peripherals.pins.gpio5;
    let rx = peripherals.pins.gpio6;

    let config = config::Config::new().baudrate(Hertz(115_200));
    let uart = UartDriver::new(
        peripherals.uart1,
        tx,
        rx,
        Option::<gpio::Gpio0>::None,
        Option::<gpio::Gpio1>::None,
        &config,
    )
    .unwrap();

    let mut rec = Vec::new();

    // Print Message to be Sent
    println!("Sent Message in Text: {}", MESSAGE);
    println!("Sent Message in Values: {:?}", MESSAGE.as_bytes());

    // Garble Message
    let gmsg: Vec<u8> = MESSAGE.as_bytes().iter().map(|m| m ^ KEY).collect();

    // Print Garbled Message
    println!("Sent Garbled Message Values: {:?}", gmsg);

    // Send Garbled Message u8 Values One by One until Full Array is Sent
    for letter in gmsg.iter() {
        // Send Garbled Message Value
        uart.write(&[*letter]).unwrap();

        // Recieve Garbled Message Value
        let mut buf = [0_u8; 1];
        uart.read(&mut buf, BLOCK).unwrap();

        // Buffer Recieved Message Value
        rec.extend_from_slice(&buf);
    }

    // Print Recieved Garbled Message Values
    println!("Recieved Garbled Message Values: {:?}", rec);

    // UnGarble Message
    let ugmsg: Vec<u8> = rec.iter().map(|m| m ^ KEY).collect();
    println!("Ungarbled Message in Values: {:?}", ugmsg);

    // Print Recovered Message
    if let Ok(rmsg) = std::str::from_utf8(&ugmsg) {
        println!("Recieved Message in Text: {:?}", rmsg);
    };

    loop {}
}
```

## Conclusion

In this post, a UART serial communication application with a simple XOR cipher was created. The application leverages the UART peripheral for the ESP32C3 microcontroller. The code was created using an embedded `std` development environment supported by the `esp-idf-hal`. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]