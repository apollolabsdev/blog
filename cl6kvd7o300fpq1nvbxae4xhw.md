---
title: "STM32F4 Embedded Rust at the HAL: I2C  Temperature & Pressure Sensing with BMP180"
datePublished: Mon Aug 08 2022 14:49:07 GMT+0000 (Coordinated Universal Time)
cuid: cl6kvd7o300fpq1nvbxae4xhw
slug: stm32f4-embedded-rust-at-the-hal-i2c-temperature-pressure-sensing-with-bmp180
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1660760377536/C-dK9aV04.png
tags: tutorial, beginners, rust, embedded

---

> This blog post is the seventh of a multi-part series of posts where I explore various peripherals in the STM32F401RE microcontroller using embedded Rust at the HAL level. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.

Prior posts include (in order of publishing):
1. [STM32F4 Embedded Rust at the HAL: GPIO Button Controlled Blinking](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking)
2. [STM32F4 Embedded Rust at the HAL: Button Controlled Blinking by Timer Polling](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling)
3. [STM32F4 Embedded Rust at the HAL: UART Serial Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication)
4. [STM32F4 Embedded Rust at the HAL: PWM Buzzer](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-pwm-buzzer)
5. [STM32F4 Embedded Rust at the HAL: Timer Ultrasonic Distance Measurement](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-timer-ultrasonic-distance-measurement)
6. [STM32F4 Embedded Rust at the HAL: Analog Temperature Sensing using the ADC](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-analog-temperature-sensing-using-the-adc)

%%[substart]

## Introduction
In this post, I will be configuring and setting up the stm32f4xx-hal I2C peripheral to collect ambient temperature measurement data from the [BMP180](https://cdn-shop.adafruit.com/datasheets/BST-BMP180-DS000-09.pdf) Digital Pressure Sensor. Note that the BMP180 provides both temperature and pressure data. Temperature measurement will be continuously collected and sent to a PC terminal over UART. I will also be leveraging the [UART Serial Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication) application/code from a previous post. Additionally, I will not be using any interrupts.

 **🚨 Important Note:**
>  The BMP180 provides both temperature and pressure sensor data which are collected in a very similar manner. I elected to collect only temperature data since the converstion equations are less. My target in this blog post is to focus on operating I2C more than the features of the BMP180 itself. However, the provided code can be easily expanded to collect and convert pressure data as well.

### Knowledge Pre-requisites
To understand the content of this post, you need the following:
- Basic knowledge of coding in Rust.
- Familiarity with the basic template for creating embedded applications in Rust.
- Familiarity with [I2C communication basics](https://en.wikipedia.org/wiki/I%C2%B2C).
- Familiarity with [UART communication basics](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter). 

### Software Setup
All the code presented in this post in addition to instructions for the environment and toolchain setup are available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

In addition to the above, you would need to install some sort of serial communication terminal on your host PC. Some recommendations include:

**For Windows**:
- [PuTTy](https://www.putty.org/)
- [Teraterm](https://ttssh2.osdn.jp/index.html.en)

**For Mac and Linux**:
- [minicom](https://wiki.emacinc.com/wiki/Getting_Started_With_Minicom)

Some installation instructions for the different operating systems are available in the [Discovery Book](https://docs.rust-embedded.org/discovery/microbit/06-serial-communication/index.html).

### Hardware Setup
#### Materials
- [Nucleo-F401RE board](https://amzn.to/3yn6AIb)

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659627678828/dqGelXcR-.jpg align="center")

- [BMP 180 GY-68 Barometric Pressure and Temperature Sensor](https://www.amazon.com/HiLetgo-Digital-Barometric-Pressure-Replace/dp/B01F527EXS)

![bmp180.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659628305425/sOiX14W2_.jpg align="center")

#### Connections
- BMP180 module SCL pin connected to Nucleo board pin PB8.
- BMP180 module SDA pin connected to Nucleo board pin PB9.
- BMP180 module Vcc pin connected to Nucleo board 3.3V.
- BMP180 module GND pin connected to Nucleo board GND.
- The UART Tx line that connects to the PC through the onboard USB bridge is via pin PA2 on the microcontroller. This is a hardwired pin, meaning you cannot use any other for this setup. Unless you are using a different board other than the Nucleo-F401RE, you have to check the relevant documentation (reference manual or datasheet) to determine the number of the pin.

## Software Design

Luckily, the software design for this application can be obtained from the [BMP180 datasheet](https://cdn-shop.adafruit.com/datasheets/BST-BMP180-DS000-09.pdf). When digging further, the [datasheet](https://cdn-shop.adafruit.com/datasheets/BST-BMP180-DS000-09.pdf) for the BMP180 provides a full flow chart describing the algorithmic steps. The flow chart also provides a list of conversion formulas needed to calculate the compensated temperature and pressure. The display temperature value part is essentially the step where I would send the result over UART.

![Algorithm Flow Chart](https://cdn.hashnode.com/res/hashnode/image/upload/v1659629261633/915GonAvd.png align="center")

From the above flow chart I will be only be implementing the parts to do with temperature collection. The same code can be easily expanded to pressure as well. One would only have to implement the additional equations.


## Code Implementation

### Crate Imports
In this implementation, the following crates are required:
- The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
- The `core::fmt` crate will allow us to use the `writeln!` macro for easy printing.
- The `panic_halt` crate to define the panicking behavior to halt on panic.
- The `stm32f4xx_hal` crate to import the STMicro STM32F4 series microcontrollers device hardware abstractions on top of the peripheral access API.

```rust
use core::fmt::Write;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    i2c::Mode,
    pac::{self},
    prelude::*,
    serial::config::Config,
};
```

### Peripheral Configuration Code

#### I2C Peripheral Configuration:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to take the PAC level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let dp = pac::Peripherals::take().unwrap();
```
2️⃣ **Configure the system clocks**: The system clocks need to be configured as they are needed in setting up both the I2C and UART peripherals. To set up the system clocks we need to first promote the `RCC` struct from the PAC and constrain it using the `constrain()` method (more detail on the `constrain` method [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)) to give use access to the `cfgr` struct. After that, we create a `clocks` handle that provides access to the configured (and frozen) system clocks. The clocks are configured to use an HSE frequency of 8MHz by applying the `use_hse()` method to the `cfgr` struct. The HSE frequency is defined by the reference manual of the Nucleo-F401RE development board. Finally, the `freeze()` method is applied to the `cfgr` struct to freeze the clock configuration. Note that freezing the clocks is a protection mechanism by the HAL to avoid the clock configuration changing during runtime. It follows that the peripherals that require clock information would only accept a [frozen `Clocks` configuration struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/rcc/struct.Clocks.html). 

```rust
let rcc = dp.RCC.constrain();
let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
```
2️⃣ **Promote the PAC-level GPIO structs and obtain handles for SDA and SCL pins**: Here I need to configure and obtain handles for the SDA and SCL pins so that they can be controlled by the I2C peripheral. As shown earlier, the SDA and SCL pins are connected to PB9 and PB8, respectively. As such, before I can obtain any handles I need to promote the pac-level `GPIOB` struct to be able to create handles for individual pins. I do this by using the `split()` method as follows: 
```rust
let gpiob = dp.GPIOB.split();
```
Next I obtain handles for `sda` and `scl` as follows:
```rust
let scl = gpiob.pb8;
let sda = gpiob.pb9;
```

3️⃣ **Configure the I2C peripheral channel**: Looking into the [Nucleo-F401RE board pinout](https://os.mbed.com/platforms/ST-Nucleo-F401RE/), the SDA and SCL lines pins PB8 and PB9 connect to the I2C1 peripheral in the microcontroller device. As such, this means we need to configure I2C1 and somehow pass it to the handles of the pins we want to use. To configure/instantiate the serial peripheral channel we have two options as seen with some other peripherals. The first is to use the device peripheral handle `dp` to directly access I2C and instantiate a transmitter instance using the `i2c` method from the [I2C extension traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/i2c/trait.I2cExt.html). The second is to use the `new` method in the `I2c` abstraction struct to instantiate a I2C1 instance. Note that both are different ways of doing exactly the same thing!

For the first option, if we examine the `i2c` method signature in the [I2C extension traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/i2c/trait.I2cExt.html), it looks like this:

```rust
fn i2c<SCL, SDA>(
    self,
    pins: (SCL, SDA),
    mode: impl Into<Mode>,
    clocks: &Clocks
) -> I2c<Self, (SCL, SDA)>
```
The method takes three parameters, a pins instance as a tuple, a mode, and a frozen `Clocks` instance reference. As such, we can create a handle `i2c` for I2C1 as follows:

```rust
let mut i2c = dp.I2C1.i2c(
        (scl, sda),
        Mode::Standard {
            frequency: 100.kHz(),
        },
        &clocks,
    );
```
`scl`, `sda`, and `clocks` are the handles that we created earlier. `Mode` is an [enum](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/i2c/enum.Mode.html) that contains mode information needed for the I2C peripheral and has two options as follows:
 
```rust
pub enum Mode {
    Standard {
        frequency: Hertz,
    },
    Fast {
        frequency: Hertz,
        duty_cycle: DutyCycle,
    },
}
```
In my instance, I chose the `Standard` option with a frequency of 100 kHz for operation. The BMP180 datasheet states that the device can handle up to 3.4Mbit/sec so I only chose an arbitrary value under the stated limit. The second option is a `Fast` option that allows you to control the duty cycle as well.

Alternatively, the second option using the `I2C` abstraction looks like this:

```rust
let mut i2c = I2c::new(
        dp.I2C1,
        (scl, sda),
        Mode::Standard {
            frequency: 300.kHz(),
        },
        &clocks,
    );
```
You can see that the main difference here is that `new` is applied as an instance method on the `I2c` struct. It can be observed here that the `new` method also accepts a fourth parameter here which is an instance of the I2C peripheral `I2C1`. This can be observed in the signature of the `tx` instance method in the documentation which looks as follows:

```rust
pub fn new(
    i2c: I2C,
    pins: (SCL, SDA),
    mode: impl Into<Mode>,
    clocks: &Clocks
) -> Self
```
That's it for I2C configuration, now moving on to UART.

#### UART Serial Communication Peripheral Configuration:

1️⃣ **Obtain a handle and configure the serial transmit (Tx) pin**: Since the Tx button is `PA2`, I need to create a handle for `gpioa`. However, since the pin is not being used as a regular GPIO input or output it means that the pin needs to be connected to a different peripheral internal to the microcontroller. The pin can be configured as such using the `into_alternate()` method as follows:

```rust
let gpioa = dp.GPIOA.split();    
let tx_pin = gpioa.pa2.into_alternate();
```

2️⃣ **Configure the serial peripheral channel**: Looking into the [Nucleo-F401RE board pinout](https://os.mbed.com/platforms/ST-Nucleo-F401RE/), the Tx line pin PA2 connects to the USART2 peripheral in the microcontroller device. As such, this means we need to configure USART2 and somehow pass it to the handle of the pin we want to use. This is done as follows:

```rust
    let mut tx = dp
        .USART2
        .tx(
            tx_pin,
            Config::default()
                .baudrate(115200.bps())
                .wordlength_8()
                .parity_none(),
            &clocks,
        )
        .unwrap();
```

`tx_pin` and `clocks` are the handles that we created earlier. `Config` is a type [struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/serial/config/struct.Config.html) that contains the configuration information needed for configuring the UART peripheral. Here I am creating an instance of `Config` with the `default` trait first to configure default parameters. After that, I apply the `baudrate`, `wordlength_8`, and `parity_none` methods to configure the UART peripheral to the settings I need. A full list of `Config` methods can be found [here](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/serial/config/struct.Config.html). I configured the UART settings as shown to 115200 bps baud with 8 bits of data, and no parity, also commonly referred to as 8N1. Finally, since the `tx` method returns a result, we would have to unwrap it using the `unwrap` method.

** 📝 Note: **
> More detail on UART setup is available in the [UART Serial Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication) blog post.

#### Timer and Delay Peripheral Configuration:
In the algorithm, a delay will need to be introduced to wait for the ADC conversion in the BMP180 to finish. I will be using `TIM1` and create a millisecond delay handle `delay` as follows:

```rust
let mut delay = dp.TIM1.delay_ms(&clocks);
```

This is it for configuration! Let's now jump into the application code.

### Application Code

In the software design described, the first step requires that we read a bunch of calibration data from the EEPROM of BMP180. In order to that, I would need to set up and initialize some sort of a struct to save the calibration data. I called the struct type `Coeffs` and defined it as follows:

```rust
    struct Coeffs {
        ac5: i16,
        ac6: i16,
        mc: i16,
        md: i16,
    }
```
After which I instantiate `calib_coeffs` of type `Coeffs` initialized to all zeros:

```rust
    let mut calib_coeffs = Coeffs {
        ac5: 0,
        ac6: 0,
        mc: 0,
        md: 0,
    };
```

 **📝 Note:**
> In the datasheet for the BMP180 there are 11 different calibration coefficients that need to be captured. Here I am capturing only the ones needed for temperature calculation.

Next, I define a bunch of constants that reflect the addresses for the calibration coefficients in the BMP180 EEPROM, the I2C address of the BMP180 itself (`BMP180_ADDR`), and the address to retrieve the BMP180 device ID (`REG_ID_ADDR`).

```rust
    const BMP180_ADDR: u8 = 0x77;
    const REG_ID_ADDR: u8 = 0xD0;
    const AC5_MSB_ADDR: u8 = 0xB2;
    const AC6_MSB_ADDR: u8 = 0xB4;
    const MC_MSB_ADDR: u8 = 0xBC;
    const MD_MSB_ADDR: u8 = 0xBE;
    const CTRL_MEAS_ADDR: u8 = 0xF4;
    const MEAS_OUT_LSB_ADDR: u8 = 0xF7;
    const MEAS_OUT_MSB_ADDR: u8 = 0xF6;
```

I also define two variables, a `[u8; 2]` array I named `rx_buffer` and an `i16` named `rx_word`. I will be using `rx_buffer` later to buffer data I read over I2C from the BMP180. `rx_word` will be used to reconstruct the read bytes into a 16-bit value.

```rust
let mut rx_buffer: [u8; 2] = [0; 2];
let mut rx_word: i16;
```
Before doing anything, we have to understand how the BMP180 communicates over I2C. Essentially, the way the BMP180 communication works, first there is a write (control) cycle indicating what internal BMP180 EEPROM address we want to read. Second, there is a read cycle where the data in the address that was requested in the write cycle is provided. 

This first thing I need to read according to the algorithmic steps is the calibration coefficients. The datasheet recommends, however, that before reading any calibration coefficients from the BMP180, that the BMP180 device ID is read as a sanity check. The device should provide back the value of `0x55`. To do that, digging into the I2C method [documentation](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/i2c/struct.I2c.html), I found `write` and `read` methods with the following signatures:

```rust
pub fn write(&mut self, addr: u8, bytes: &[u8]) -> Result<(), Error>
```
and
```rust
pub fn read(&mut self, addr: u8, buffer: &mut [u8]) -> Result<(), Error>
```
Although there wasn't much description in the documentation, it can be seen that both are more or less similar with a minor difference. Both take two arguments and return a `Result`. The first argument `addr` is the device address in which case for us will be the BMP180 device address `BMP180_ADDR`. The second argument for the `write` method is `bytes` and is an array slice containing the bytes being written. The second argument for the `read` method is `buffer` and is a mutable array slice containing the bytes being returned back by the addressed device.

There was also another method that caught my attention that looked useful. It was the `write_read` method. Again, although there wasn't much description, I figured its workings with some experimentation. The `write_read` method does a write cycle immediately followed by a read cycle. This will be useful in many contexts and save some lines of code. We'll see that I still would need the separate `read` and `write` methods, however. The `write_read` method has the following signature:

```rust
pub fn write_read(
    &mut self,
    addr: u8,
    bytes: &[u8],
    buffer: &mut [u8]
) -> Result<(), Error>
```
The parameters are more or less the same as the earlier methods, only combined in one.

So, now that I have what I need, to do our sanity check, this is the code I wrote:
```rust
    i2c.write_read(BMP180_ADDR, &[REG_ID_ADDR], &mut rx_buffer).unwrap();
    if rx_buffer[0] == 0x55 {
        writeln!(tx, "Device ID is {}\r", rx_buffer[0]).unwrap();
    } else {
        writeln!(tx, "Device ID Cannot be Detected \r").unwrap();
    }
```
Analyzing the code, the first argument is the device address `BMP180_ADDR`, the second argument is a slice containing the device ID retrieval address `REG_ID_ADDR`, and finally the third argument is the receive buffer `rx_buffer`. Note that all the arguments I am passing I have created at an earlier point in the application. The following `if` statement checks if the received ID is correct and sends the appropriate message accordingly over UART. Note that from the BMP180 datasheet, since REG_ID_ADDR returns only a single byte, I only had to check the first index in the buffer `rx_buffer[0]`. 

Now that I've verified that the device can be detected, let's start with the first step in the algorithmic steps which is collecting the calibration coefficients. Here's the code for retrieving calibration coefficient AC5:

```rust
    i2c.write_read(BMP180_ADDR, &[AC5_MSB_ADDR], &mut rx_buffer)
        .unwrap();
    rx_word = ((rx_buffer[0] as i16) << 8) | rx_buffer[1] as i16;
    writeln!(tx, "AC5 = {} \r", rx_word).unwrap();
    calib_coeffs.ac5 = rx_word;
```
As might be noticed I used the `write_read` method exactly in the same way I did earlier. There are three differences to note here though:
1. Recall I mentioned that the BMP180 provides a 16-bit value for each calibration coefficient. Since I2C communicates in bytes, each coefficient is broken down into two bytes MSB and LSB, with each having its own address in the BMP180 EEPROM. This means that technically I would need to retrieve each address separately and reconstruct the 16-bit value. Though it turns out that by sending only the MSB address to the BMP180 (`AC5_MSB_ADDR` in this case) the device sends back both the MSB followed by the LSB without needing to address the LSB separately. The MSB will be located in the first index of the buffer and the LSB in the second index. So the point here is that I needed to call the `read_write` method only once using `AC5_MSB_ADDR` to retrieve both `AC5_MSB_ADDR` and `AC5_LSB_ADDR`.
2. Since an `i16` will be provided back, the line `rx_word = ((rx_buffer[0] as i16) << 8) | rx_buffer[1] as i16;` takes the MSB bits from `rx_buffer[0]`, casts them to an `i16` and shifts the 1 byte (8 times) to the left using the `<<` operator, then ORs the result using the `|` operator with the LSB bits in `rx_buffer[1]` that are also cast as `i16`. The result is finally stored in `rx_word` and then sent over the UART channel.
3.  The `rx_word` is stored in the `calib_coeffs` struct I had created earlier in the statement `calib_coeffs.ac5 = rx_word;`. This would also allow me to reuse rx_word for the following operations.

The previous code is repeated in the exact same manner three times to retrieve the AC6, MC, and MD coefficients. Obviously, the only differences would be the address I would send to the BMP180, and the name of the member I store in the `calib_coeffs` struct.

Now that the calibration coefficients are available, the measurement loop can be started. The first step as indicated by the software design is to kick-off the temperature measurement in the BMP180 by writing 0x2E in register with address 0xF4 ( keyed in as `CTRL_MEAS_ADDR`). This is done using two write cycles first sending `0x2E` followed by `CTRL_MEAS_ADDR`. This can be done in a single line as well. Since the `write` method accepts a slice in the `bytes` parameter in its signature, all the bytes that need to be sent can be included in the slice as follows:

```rust
i2c.write(BMP180_ADDR, &[CTRL_MEAS_ADDR, 0x2E]).unwrap();
```
This statement will send the `CTRL_MEAS_ADDR` to the BMP180 followed by the value `0x2E`. For those familiar with I2C terms, although I didn't verify at the low level, nor does the documentation specify, but here a "repeated start" should be what is occurring. 

After kicking off the BMP180 temperature measurement, the datasheet tells us we need to wait at least 4.5ms. So I wait for 5ms to stay on the safe side using the `delay` handle created earlier:

```rust
 delay.delay_ms(5_u32);
```
Now the temperature measurement should be ready to collect by reading the measurement BMP180 MSB and LSB EEPROM addresses. This can be done using the same earlier approach where I only send the MSB address. Here I show a different approach where I read the MSB and the LSB separately, just to prove that it works 😃:

```rust
        i2c.write(BMP180_ADDR, &[MEAS_OUT_MSB_ADDR]).unwrap();
        i2c.read(BMP180_ADDR, &mut rx_buffer).unwrap();
        rx_word = (rx_buffer[0] as i16) << 8;

        i2c.write(BMP180_ADDR, &[MEAS_OUT_LSB_ADDR]).unwrap();
        i2c.read(BMP180_ADDR, &mut rx_buffer).unwrap();
        rx_word |= rx_buffer[0] as i16;
```

Finally, since the temperature value received is uncompensated. The datasheet provides us a bunch of formulas to calculate the compensated temperature based on the calibration coefficients that were collected earlier. The following code implements the formulas in the datasheet to calculate the temperature followed by sending the result over UART:

```rust
        let x1 = (rx_word as i32 - calib_coeffs.ac6 as i32) * (calib_coeffs.ac5 as i32) >> 15;
        let x2 = ((calib_coeffs.mc as i32) << 11) / (x1 + calib_coeffs.md as i32);
        let b5 = x1 + x2;
        let t = ((b5 + 8) >> 4) / 10;

        // Print Temperature Value
        writeln!(tx, "Temperature = {:} \r", t).unwrap();
```

Note how values are cast as `i32` so to prevent overflow from the multiplication operations.

This is it!

## Full Application Code
Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.


```rust
#![no_std]
#![no_main]

// Imports
use core::fmt::Write;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    i2c::Mode,
    pac::{self},
    prelude::*,
    serial::config::Config,
};

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let dp = pac::Peripherals::take().unwrap();

    // I2C Config steps:
    // 1) Need to configure the system clocks
    // - Promote RCC structure to HAL to be able to configure clocks
    let rcc = dp.RCC.constrain();
    // - Configure system clocks
    // 8 MHz must be used for the Nucleo-F401RE board according to manual
    let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
    // 2) Configure/Define SCL and SDA pins
    let gpiob = dp.GPIOB.split();
    let scl = gpiob.pb8;
    let sda = gpiob.pb9;
    // 3) Configure I2C perihperal channel
    // We're going to use I2C1 since its pins are the ones connected to the I2C interface we're using
    // To configure/instantiate serial peripheral channel we have two options:
    // Use the i2c device peripheral handle and instantiate a transmitter instance using extension trait
    let mut i2c = dp.I2C1.i2c(
        (scl, sda),
        Mode::Standard {
            frequency: 100.kHz(),
        },
        &clocks,
    );
    // Or use the I2C abstraction
    // let mut i2c = I2c::new(
    //     dp.I2C1,
    //     (scl, sda),
    //     Mode::Standard {
    //         frequency: 300.kHz(),
    //     },
    //     &clocks,
    // );

    // Serial config steps:
    // 1) Need to configure the system clocks
    // Already done earlier for I2C module
    // 2) Configure/Define TX pin
    // Use PA2 as it is connected to the host serial interface
    let gpioa = dp.GPIOA.split();
    let tx_pin = gpioa.pa2.into_alternate();
    // 3) Configure Serial perihperal channel
    // We're going to use USART2 since its pins are the ones connected to the USART host interface
    // To configure/instantiate serial peripheral channel we have two options:
    // Use the device peripheral handle to directly access USART2 and instantiate a transmitter instance
    let mut tx = dp
        .USART2
        .tx(
            tx_pin,
            Config::default()
                .baudrate(9600.bps())
                .wordlength_8()
                .parity_none(),
            &clocks,
        )
        .unwrap();

    let mut delay = dp.TIM1.delay_ms(&clocks);

    struct Coeffs {
        ac5: i16,
        ac6: i16,
        mc: i16,
        md: i16,
    }

    let mut calib_coeffs = Coeffs {
        ac5: 0,
        ac6: 0,
        mc: 0,
        md: 0,
    };

    const BMP180_ADDR: u8 = 0x77;
    const REG_ID_ADDR: u8 = 0xD0;
    const AC5_MSB_ADDR: u8 = 0xB2;
    const AC6_MSB_ADDR: u8 = 0xB4;
    const MC_MSB_ADDR: u8 = 0xBC;
    const MD_MSB_ADDR: u8 = 0xBE;
    const CTRL_MEAS_ADDR: u8 = 0xF4;
    const MEAS_OUT_LSB_ADDR: u8 = 0xF7;
    const MEAS_OUT_MSB_ADDR: u8 = 0xF6;

    let mut rx_buffer: [u8; 2] = [0; 2];
    let mut rx_word: i16;

    // Read Device ID as Sanity Check
    i2c.write(BMP180_ADDR, &[REG_ID_ADDR]).unwrap();
    i2c.read(BMP180_ADDR, &mut rx_buffer).unwrap();
    // OR 
    // i2c.write_read(BMP180_ADDR, &[REG_ID_ADDR], &mut rx_buffer)
    //     .unwrap();
    if rx_buffer[0] == 0x55 {
        writeln!(tx, "Device ID is {}\r", rx_buffer[0]).unwrap();
    } else {
        writeln!(tx, "Device ID Cannot be Detected \r").unwrap();
    }

    // Read Calibration Coefficients
    // Read AC5
    i2c.write_read(BMP180_ADDR, &[AC5_MSB_ADDR], &mut rx_buffer)
        .unwrap();
    rx_word = ((rx_buffer[0] as i16) << 8) | rx_buffer[1] as i16;
    writeln!(tx, "AC5 = {} \r", rx_word).unwrap();
    calib_coeffs.ac5 = rx_word;

    // Read AC6
    i2c.write_read(BMP180_ADDR, &[AC6_MSB_ADDR], &mut rx_buffer)
        .unwrap();
    rx_word = ((rx_buffer[0] as i16) << 8) | rx_buffer[1] as i16;
    writeln!(tx, "AC6 = {} \r", rx_word).unwrap();
    calib_coeffs.ac6 = rx_word;

    // Read MC
    i2c.write_read(BMP180_ADDR, &[MC_MSB_ADDR], &mut rx_buffer)
        .unwrap();
    rx_word = ((rx_buffer[0] as i16) << 8) | rx_buffer[1] as i16;
    writeln!(tx, "MC = {} \r", rx_word).unwrap();
    calib_coeffs.mc = rx_word;

    // Read MD
    i2c.write_read(BMP180_ADDR, &[MD_MSB_ADDR], &mut rx_buffer)
        .unwrap();
    rx_word = ((rx_buffer[0] as i16) << 8) | rx_buffer[1] as i16;
    writeln!(tx, "MD = {} \r", rx_word).unwrap();
    calib_coeffs.md = rx_word;

    // Application Loop
    loop {
        // Kick off Temperature Measurement by writing 0x2E in register 0xF4
        i2c.write(BMP180_ADDR, &[CTRL_MEAS_ADDR, 0x2E]).unwrap();
        // Wait 4.5 ms for measurment to complete as specified by the datasheet
        delay.delay_ms(5_u32);

        // Collect Temperature Measurment
        // Read Measurement MSB
        // Achieving same as above using an alternate method syntax here to do a write followed by read
        i2c.write(BMP180_ADDR, &[MEAS_OUT_MSB_ADDR]).unwrap();
        i2c.read(BMP180_ADDR, &mut rx_buffer).unwrap();
        rx_word = (rx_buffer[0] as i16) << 8;
        // Read Measurement LSB
        i2c.write(BMP180_ADDR, &[MEAS_OUT_LSB_ADDR]).unwrap();
        i2c.read(BMP180_ADDR, &mut rx_buffer).unwrap();
        rx_word |= rx_buffer[0] as i16;

        // Uncomment following line to print raw uncompenstated temperature value
        //writeln!(tx, "UT = {} \r", rx_word).unwrap();

        // Calculate Temperature According to Datasheet Formulas
        let x1 = (rx_word as i32 - calib_coeffs.ac6 as i32) * (calib_coeffs.ac5 as i32) >> 15;
        let x2 = ((calib_coeffs.mc as i32) << 11) / (x1 + calib_coeffs.md as i32);
        let b5 = x1 + x2;
        let t = ((b5 + 8) >> 4) / 10;

        // Print Temperature Value
        writeln!(tx, "Temperature = {:} \r", t).unwrap();
    }
}
``` 

## Further Experimentation/Ideas:
Some ideas to experiment with include:
- Expand the code to collect pressure measurement data including calibration data and calculate barometric pressure.
- The BMP180 has different accuracy modes in which more accurate measurements can be provided but also different wait times apply as well. You can refactor the code or even create functions that can select one of the desired modes.

## Conclusion
In this post, an I2C temperature and pressure measurement application was created by controlling and collecting data from an external BMP180 sensor. This was using the I2C peripheral for the STM32F401RE microcontroller on the Nucleo-F401RE development board. The resulting measurement is also sent over to a host PC over a UART connection. All code was based on polling (without interrupts). Additionally, all code was created at the HAL level using the [stm32f4xx Rust HAL](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/index.html). Have any questions? Share your thoughts in the comments below 👇.  

%%[subend]