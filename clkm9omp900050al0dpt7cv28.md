---
title: "ESP32 Standard Library Embedded Rust: I2C Communication"
datePublished: Fri Jul 28 2023 07:33:34 GMT+0000 (Coordinated Universal Time)
cuid: clkm9omp900050al0dpt7cv28
slug: esp32-standard-library-embedded-rust-i2c-communication
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1690526950517/91c4486f-1e94-4f4d-b592-71ce219863b3.png
tags: tutorial, rust, embedded, esp32

---

> ***This blog post is the third of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [ESP32 Standard Library Embedded Rust: GPIO Control](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-gpio-control)
    
2. [ESP32 Standard Library Embedded Rust: UART Communication](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-uart-communication)
    

## Introduction

I2C is a popular serial communication protocol in embedded systems. Another common name used is also Two-Wire Interface (TWI), since, well, it is. I2C is commonly found in sensors and interface boards as it offers a compact implementation with decent speeds reaching Mbps. Counter to UART, I2C allows multiple devices to tap into the same bus.

In this post, I will be configuring and setting up I2C communication for the ESP32C3 to communicate with a real-time clock (RTC) module. In particular, I will be sending and retrieving time and date data to and from a DS1307 device. The retrieved results will be printed on the console.

%%[substart] 

### **📚** Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with [**I2C communication basics**](https://learn.sparkfun.com/tutorials/i2c/all).
    

### **💾** Software Setup

All the code presented in this post is available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [here](https://wokwi.com/projects/370570140823146497).

### **🛠** Hardware Setup

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg align="center")
    
* [Adafruit DS1307 Real-Time Clock](https://www.adafruit.com/product/3296)
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690478164257/363b83cb-d6a8-4542-81bf-c99707c64675.png align="center")

#### **🔌** Connections

**📝 Note**

> All connection details are also shown in the [Wokwi example](https://wokwi.com/projects/370570140823146497).

Connections include the following:

* Gpio2 wired to the SCL pin on DS1307.
    
* Gpio3 wired to the SDA pin on DS1307.
    
* 5V pin on the Dev kit to the 5V pin on DS1307.
    
* Gnd pin on the Dev kit to the gnd pin on DS1307.
    

## **👨‍🎨** Software Design

In the application developed in this post, the DS1307 time and date data will be configured first and then the date/time data will be read back afterward every second. The DS1307 provides the following table in its datasheet:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690479123682/317aa9d4-328e-499e-8686-06efd0407053.png align="center")

The table above provides an address mapping of the internal memory of the DS1307 that contains the time/date data. Addresses 0x00 to 0x06 are all we need for the desired application. The table also specifies the valid range of values for the data in each address. All values are BCD (Binary Coded Decimal) encoded.

The DS1307 communicates over I2C where we will be reading from it and writing to it. In order to achieve what we want here are the main points we need to know:

* The address of the device is 0x68.
    
* On reset, the CH bit in address 0x00 needs to be set to 0 to kick off the DS1307 internal oscillator.
    
* The date/time of the device can be configured by writing to the different addresses.
    
* The date/time of the device can be retrieved by reading from different addresses.
    
* To communicate with the DS1307 a starting address needs to be provided followed by the data read/written from/to the device. The device will keep on writing/reading to/from consecutive addresses until the master (ESP32) decides to halt.
    

Let's now jump into implementing this algorithm.

## **👨‍💻** Code Implementation

### **📥** Crate Imports

In this implementation, one crate is required as follows:

* The `esp_idf_hal` crate to import the needed device hardware abstractions.
    
* The `no_bcd` crate to provide abstractions to convert to BCD from decimal and vice versa.
    

```rust
use esp_idf_hal::delay::{FreeRtos, BLOCK};
use esp_idf_hal::i2c::*;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::prelude::*;

use nobcd::BcdNumber;
```

### **🎛** Peripheral Configuration Code

Ahead of our application code, peripherals are configured through the following steps:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to `take` the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let peripherals = Peripherals::take().unwrap();
```

**2️⃣ Obtain handles for the sda and scl pins**: for convenience, we obtain handles for the pins that are going to be configured as `sda` and `scl` in the I2C peripheral. In the ESP32C3, any pin can be used for `sda` and `scl`. As such, I chose `gpio3` to be used as the `sda` pin and `gpio2` will be used as the `scl` pin:

```rust
let sda = peripherals.pins.gpio3;
let scl = peripherals.pins.gpio2;
```

Note that at this point the pins are not configured yet, we merely created handles for them. They yet need to be configured to be connected to the internal I2C peripheral of the ESP32.

3️⃣ **Obtain a handle and configure the I2C peripheral**: In order to configure I2C, there is a `I2cDriver` abstraction in the [esp-idf-hal](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/uart/struct.UartDriver.html) that contains a `new` method to create a I2C instance. The `new` method has the following signature:

```rust
pub fn new<I2C: I2c>(
    _i2c: impl Peripheral<P = I2C> + 'd,
    sda: impl Peripheral<P = impl InputPin + OutputPin> + 'd,
    scl: impl Peripheral<P = impl InputPin + OutputPin> + 'd,
    config: &Config
) -> Result<Self, EspError>
```

as shown, the `new` method has 4 parameters. The first `i2c` parameter is a `I2C` peripheral type. The second `sda` parameter is a bidirectional pin combining `InputPin` and `OutputPin` types. The third `scl` parameter is also a bidirectional pin combining `InputPin` and `OutputPin` types. Finally, the fourth parameter is a I2C configuration type `Config`. As such, we create an instance for `i2c` with handle name `ds1307` as follows:

```rust
let config = I2cConfig::new().baudrate(100.kHz().into());
let mut ds1307 = I2cDriver::new(i2c, sda, scl, &config).unwrap();
```

`Config` is a [`i2c::config`](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/uart/config/struct.Config.html) type that provides configuration parameters for I2C. `Config` contains a method `new` to create an instance with default parameters. Afterward, there are configuration parameters adjustable through various methods. A full list of those methods can be found in the [documentation](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/i2c/config/struct.Config.html). In this case, I only used the `baudrate` method to change the baud rate configuration.

That's it for configuration.

### **📱** Application Code

Following the design described earlier, there are a few things we can do before building the application to make things neater. the first is to set up a constant that reflects the `0x68` address of the DS1307:

```rust
// DS1307 Address
const DS1307_ADDR: u8 = 0x68;
```

Next, I create a `Ds1307Addr` enum reflecting the address mappings of the DS1307:

```rust
#[repr(u8)]
enum Ds1307Addr {
    Seconds,
    Minutes,
    Hours,
    Day,
    Date,
    Month,
    Year,
}
```

This enum would be useful to access particular addresses of the DS1307. The order of members is aligned with the address map in the DS1307 table shown earlier.

Next, an enum `DAY` to create a mapping for the days of the week:

```rust
enum DAY {
    Sun = 1,
    Mon = 2,
    Tues = 3,
    Wed = 4,
    Thurs = 5,
    Fri = 6,
    Sat = 7,
}
```

The DS1307 defines the range of values for the day or the week from 1-7. The mapping is left to the user and doesn't matter what value is attached to a day as long as it remains consistent.

Next, I create a `DateTime` struct that contains members that store date and time data. Additionally, I create a `start_dt` variable of `DateTime` type that contains data I want to initialize the DS1307 with.

```rust
struct DateTime {
    sec: u8,
    min: u8,
    hrs: u8,
    day: u8,
    date: u8,
    mnth: u8,
    yr: u8,
}

let start_dt = DateTime {
    sec: 0,
    min: 0,
    hrs: 0,
    day: DAY::Thurs as u8,
    date: 27,
    mnth: 7,
    yr: 23,
};
```

### ⏰ Setting Time

Before sending a message over I2C to the DS1307, recall that the DS1307 data is BCD encoded. For that, we need to convert the numbers we want to send to BCD ahead of transmission. Performing a search on crates.io, I came across the `nobcd` crate that would help. I won't go into much detail here as I would defer to the crate [documentation](https://docs.rs/nobcd/latest/nobcd/struct.BcdNumber.html), but the first line below converts the second's data `start_dt.sec` to its BCD equivalent. Afterward, using the `write` method available `I2cDriver` we write the data to the DS1307:

```rust
let secs: [u8; 1] = BcdNumber::new(start_dt.sec).unwrap().bcd_bytes();
Ds1307Addr
    .write(
        Ds1307Addr,
        &[Ds1307Addr::Seconds as u8, secs[0]],
        BLOCK,
    )
    .unwrap();
```

The first argument of the `write` method is the address of the device, the second argument is the `&[u8]` data that needs to be sent, and the third argument is a delay value (please refer to the [UART post](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-uart-communication) for more on `BLOCK`). Regarding the data, note how two bytes are being sent as part of the array. First is the starting address `Ds1307Addr::Seconds` followed by the data itself `secs[0]`.

The above code needs to be repeated for each of the addresses in the DS1307. Meaning we'd still need to set the minute, hour, day of week, day of month, month, and year data in the same manner. The full code is shown in the latter part of this post.

### ⏳ Reading Date and Time

In order to read all the date and time data from the DS1307, it needs two operations. Meaning we need to provide a `DS1307` starting address using the `write` method first. Second, we use the `read` method passing a slice with a size large enough to fit the data read. As such, the I2C peripheral `read` method will keep on reading data from the addressed device until the buffer is filled and then stops. The following is the code:

```rust
let mut data: [u8; 7] = [0_u8; 7];

Ds1307Addr.write(Ds1307Addr_ADDR, &[0_u8], BLOCK).unwrap();
Ds1307Addr.read(Ds1307Addr_ADDR, &mut data, BLOCK).unwrap();
```

Note here that the DS1307 starting address provided is zero propagated through the `&[0_u8]` argument. Also, we read 7 bytes of data implied through the size of the `data` array passed as an argument in the `read` method.

### 🖨️ Printing to Console

After this, a date/time reading will be available in `data` that needs to be extracted for printing. Using `println!` with some formatting, the data is printed as follows:

```rust
let secs = BcdNumber::from_bcd_bytes([data[0] & 0x7f])
    .unwrap()
    .value::<u8>();
let mins = BcdNumber::from_bcd_bytes([data[1]]).unwrap().value::<u8>();
let hrs = BcdNumber::from_bcd_bytes([data[2] & 0x3f])
    .unwrap()
    .value::<u8>();
let dom = BcdNumber::from_bcd_bytes([data[4]]).unwrap().value::<u8>();
let mnth = BcdNumber::from_bcd_bytes([data[5]]).unwrap().value::<u8>();
let yr = BcdNumber::from_bcd_bytes([data[6]]).unwrap().value::<u8>();
let dow = match BcdNumber::from_bcd_bytes([data[3]]).unwrap().value::<u8>() {
    1 => "Sunday",
    2 => "Monday",
    3 => "Tuesday",
    4 => "Wednesday",
    5 => "Thursday",
    6 => "Friday",
    7 => "Saturday",
    _ => "",
};

println!(
    "{}, {}/{}/20{}, {:02}:{:02}:{:02}",
    dow, dom, mnth, yr, hrs, mins, secs
);
```

A few things to note:

* The retrieved data is converted back from BCD to decimal before printing.
    
* The day of the week is recovered is mapped to a string using the match statement
    
* The `secs` and `hrs` data is masked as data recovered from those two addresses have additional non-necessary data (shown in DS1307 table above).
    

## 🔧 Code Optimization

Looking at the code, it appears quite verbose. However, there are opportunities for making it more concise. I typically do not optimize from the get-go to clarify what the code tries to achieve. Sometimes, optimizations introduce several ideas at once which makes it harder to grasp the core concept. As such, here, I am going to state some ideas and leave it to the interested reader to experiment. As such, some opportunities include:

* In the code setting the time, the same code is repeated for each piece of data written to the DS1307. Instead, the date/time data can be iterated over in an array using a `for` loop and sent to the DS1307.
    
* In reading the time from the DS1307, I used a `write` method followed by a `read` method. Instead, there exists a `write_read` method that can compact the two methods in a single statement.
    
* Converting the read data from BCD to decimal for printing to the console also involves repetitive code. This can also be made much more compact through a loop that buffers the converted BCD data and then prints.
    

## **📱** Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [here](https://wokwi.com/projects/370570140823146497).

```rust
use esp_idf_hal::delay::{FreeRtos, BLOCK};
use esp_idf_hal::i2c::*;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_hal::prelude::*;

use nobcd::BcdNumber;

const Ds1307Addr_ADDR: u8 = 0x68;

fn main() {
    esp_idf_sys::link_patches();

    let peripherals = Peripherals::take().unwrap();

    let sda = peripherals.pins.gpio3;
    let scl = peripherals.pins.gpio2;

    let i2c = peripherals.i2c0;
    let config = I2cConfig::new().baudrate(100.kHz().into());
    let mut Ds1307Addr = I2cDriver::new(i2c, sda, scl, &config).unwrap();

    #[repr(u8)]
    enum Ds1307Addr {
        Seconds,
        Minutes,
        Hours,
        Day,
        Date,
        Month,
        Year,
    }

    #[repr(u8)]
    enum DAY {
        Sun = 1,
        Mon = 2,
        Tues = 3,
        Wed = 4,
        Thurs = 5,
        Fri = 6,
        Sat = 7,
    }

    struct DateTime {
        sec: u8,
        min: u8,
        hrs: u8,
        day: u8,
        date: u8,
        mnth: u8,
        yr: u8,
    }

    let start_dt = DateTime {
        sec: 0,
        min: 0,
        hrs: 0,
        day: DAY::Thurs as u8,
        date: 27,
        mnth: 7,
        yr: 23,
    };

    // Set Time
    // Set Seconds -> Also Activates Oscillator
    let secs: [u8; 1] = BcdNumber::new(start_dt.sec).unwrap().bcd_bytes();
    Ds1307Addr
        .write(
            Ds1307Addr_ADDR,
            &[Ds1307Addr::Seconds as u8, secs[0]],
            BLOCK,
        )
        .unwrap();
    // Set Minutes
    let mins: [u8; 1] = BcdNumber::new(start_dt.min).unwrap().bcd_bytes();
    Ds1307Addr
        .write(
            Ds1307Addr_ADDR,
            &[Ds1307Addr::Minutes as u8, mins[0]],
            BLOCK,
        )
        .unwrap();
    // Set Hours
    let hrs: [u8; 1] = BcdNumber::new(start_dt.hrs).unwrap().bcd_bytes();
    Ds1307Addr
        .write(Ds1307Addr_ADDR, &[Ds1307Addr::Hours as u8, hrs[0]], BLOCK)
        .unwrap();
    // Set Day of Week
    let dow: [u8; 1] = BcdNumber::new(start_dt.day).unwrap().bcd_bytes();
    Ds1307Addr
        .write(Ds1307Addr_ADDR, &[Ds1307Addr::Day as u8, dow[0]], BLOCK)
        .unwrap();
    // Set Day of Month
    let dom: [u8; 1] = BcdNumber::new(start_dt.date).unwrap().bcd_bytes();
    Ds1307Addr
        .write(Ds1307Addr_ADDR, &[Ds1307Addr::Date as u8, dom[0]], BLOCK)
        .unwrap();
    // Set Month
    let mnth: [u8; 1] = BcdNumber::new(start_dt.mnth).unwrap().bcd_bytes();
    Ds1307Addr
        .write(Ds1307Addr_ADDR, &[Ds1307Addr::Month as u8, mnth[0]], BLOCK)
        .unwrap();
    // Set Year
    let yr: [u8; 1] = BcdNumber::new(start_dt.yr).unwrap().bcd_bytes();
    Ds1307Addr
        .write(Ds1307Addr_ADDR, &[Ds1307Addr::Year as u8, yr[0]], BLOCK)
        .unwrap();

    loop {
        // Initialize Array that will buffer data read from the Ds1307Addr
        let mut data: [u8; 7] = [0_u8; 7];

        // Provide Starting Address (zero) to Read Data from Ds1307Addr
        // Optionally can use the wrte_read method that performs both in a single line
        Ds1307Addr.write(Ds1307Addr_ADDR, &[0_u8], BLOCK).unwrap();
        Ds1307Addr.read(Ds1307Addr_ADDR, &mut data, BLOCK).unwrap();

        println!("{:?}", data);

        let secs = BcdNumber::from_bcd_bytes([data[0] & 0x7f])
            .unwrap()
            .value::<u8>();
        let mins = BcdNumber::from_bcd_bytes([data[1]]).unwrap().value::<u8>();
        let hrs = BcdNumber::from_bcd_bytes([data[2] & 0x3f])
            .unwrap()
            .value::<u8>();
        let dom = BcdNumber::from_bcd_bytes([data[4]]).unwrap().value::<u8>();
        let mnth = BcdNumber::from_bcd_bytes([data[5]]).unwrap().value::<u8>();
        let yr = BcdNumber::from_bcd_bytes([data[6]]).unwrap().value::<u8>();
        let dow = match BcdNumber::from_bcd_bytes([data[3]]).unwrap().value::<u8>() {
            1 => "Sunday",
            2 => "Monday",
            3 => "Tuesday",
            4 => "Wednesday",
            5 => "Thursday",
            6 => "Friday",
            7 => "Saturday",
            _ => "",
        };

        println!(
            "{}, {}/{}/20{}, {:02}:{:02}:{:02}",
            dow, dom, mnth, yr, hrs, mins, secs
        );

        FreeRtos::delay_ms(1000_u32);
    }
}
```

## Conclusion

In this post, a I2C serial communication application talking to the DS1307 was created. The application leverages the I2C peripheral for the ESP32C3 microcontroller. The code was created using an embedded `std` development environment supported by the `esp-idf-hal`. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]