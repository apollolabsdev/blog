---
title: "STM32F4 Embedded Rust at the HAL: SPI with the MAX7219 LED Dot Matrix"
datePublished: Mon Sep 26 2022 14:27:13 GMT+0000 (Coordinated Universal Time)
cuid: cl8iv5sit00hwr0nvetx1ghip
slug: stm32f4-embedded-rust-at-the-hal-spi-with-the-max7219-led-dot-matrix
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1664201738104/p3RswtcfB.png
tags: tutorial, beginners, rust, embedded

---

## Introduction
This blog post is the start of a new series where I explore the creation of platform agnostic embedded-hal drivers. As a start, I'll be working with the [MAX7219](https://datasheets.maximintegrated.com/en/ds/MAX7219-MAX7221.pdf) 8-digit LED display driver. For the purpose of testing, I will also be using the STM32F401RE microcontroller. To make things more digestible, I will be breaking the steps into several posts as follows:

1. **Create simple code to configure and test the MAX7219 with a simple application. [Link to Post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-spi-with-the-max7219-led-dot-matrix).**
2. Refactor and optimize the code in the first step by adding functions. This step would also create a driver that isn't platform agnostic. [Link to post](https://apollolabsblog.hashnode.dev/platform-agnostic-drivers-in-rust-max7219-naive-code-refactoring).
3. Refactor code in the second step to incorporate embedded-hal traits and create a platform-agnostic driver. [Link to post](https://apollolabsblog.hashnode.dev/platform-agnostic-drivers-in-rust-the-max7219-driver).
4. Register the driver in [crates.io](https://crates.io/) and publish its documentation.
5. Add advanced features to the driver and introduce a new version of the crate.

%%[substart]

### Knowledge Pre-requisites
To understand the content of this post, you need the following:
- Basic knowledge of coding in Rust.
- Familiarity with the basic template for creating embedded applications in Rust.
- Familiarity with [SPI communication basics](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface).
- Having read the MAX7219 8-digit LED display driver [datasheet](https://datasheets.maximintegrated.com/en/ds/MAX7219-MAX7221.pdf).

### Software Setup
All the code presented in this post in addition to instructions for the environment and toolchain setup are available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

### Hardware Setup
#### Materials
- [Nucleo-F401RE board](https://amzn.to/3yn6AIb)

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659627678828/dqGelXcR-.jpg align="center")

- [MAX7219 LED 8x8 Dot Matrix Module](https://www.amazon.com/ALAMSCN-MAX7219-Display-Raspberry-Microcontroller/dp/B08SQT9GGB/ref=sr_1_13?crid=3HUOC2Q9Q7JMQ&keywords=max7219&qid=1664032988&qu=eyJxc2MiOiI0Ljc2IiwicXNhIjoiNC41MSIsInFzcCI6IjQuMTUifQ%3D%3D&sprefix=max7219%2Caps%2C322&sr=8-13)

![max7219-led-driver.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1664033038052/liiP8gcWj.jpeg align="center")

#### Connections
- MAX7219 module CLK pin connected to Nucleo board pin PA5 (SPI SCLK).
- MAX7219 module DIN pin connected to Nucleo board pin PA7 (SPI MOSI).
- MAX7219 module CS pin connected to Nucleo board pin PA6 (SPI CS).
- MAX7219 module Vcc pin connected to Nucleo board 3.3V or 5V.
- MAX7219 module GND pin connected to Nucleo board GND.

## Software Design

All that the application software will do is draw a diagonal line on the 8x8 LED matrix then erase it and draw it again. Ahead of that all the steps involved will be to initialize and configure the MAX7219 so that it can be used. What steps need to be taken for configuration all come from the datasheet. Ahead of that though, lets take a look at a couple of things. First is the internal block diagram of the MAX7219:

![max7219 block diagram](https://cdn.hashnode.com/res/hashnode/image/upload/v1664034317499/qDrikBr2R.png align="center")

I will leave the intimate details to the reader to figure out from the datasheet. However, there are a few things I want to highlight here. There are 16-bits of data that get clocked in MSB first using SPI while CS is low. After that, CS is driven high to latch the data. A portion of those latched-in bits are used as an address (D8-D11) and another portion as data or command (D0-D7). The rest of the bits (D12-D15) are simply ignored. The MAX7219 also allows for chaining multiple display matrices side by side by connecting the DOUT to the next DIN. When showing data the address part selects what digit we want to show and the data controls which segments are turned on. As shown in the diagram, there are 7-segment driver pins (and a decimal point) and 8 digit driver pins.

Second lets take a look at the schematic showing how the LED dot matrix is connected to the MAX7219:

![max7219schematic.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664033795046/X_UpOYdpa.png align="center")

If you notice, the segment pins of the MAX7219 are connected to the rows and the digit pins are connected to the columns. As such, the way the MAX7219 operates, it will allow us to cycle through digits (or columns) turning on/off any segment (or LEDs) in any digit. 

#### Initialization Sequence
After initializing the required peripherals in the controller, reading into the MAX7219 datasheet, the following steps need to be taken to initialize the device for usage:

1.  Write '0x01' to address '0x0C' to power up the MAX7219
2. Write '0x00' to address '0x09' to put the MAX7219 in "No Decode" mode (see notes)
3. Write '0x07' to address '0x0B' removing any scan limit (see notes)
4. Write '0x07' to address '0x0A' choosing a medium light intensity for the LEDs

**📝 Notes**
> The "No Decode" mode is the one necessary for usage with the LED matrix. Other modes are ones used for usage with seven segment displays.

> The scan limit controls how many digits are activated (displayed) in the case of hooking up the MAX7219 to seven segment displays. In the case of the dot matrix, however, the scan limit affects how many rows can be activated. Obviously, in the case of the 8x8 dot matrix, all rows need to be shown. However, this is a feature when connecting a series of seven segment displays, in some cases, some digits need not be used.

#### LED Control Sequence
Like I had mentioned earlier all that the code will be doing is activate the LEDs diagonally one at a time then deactivate them again. These are the steps:

1. `x` = 1 
2. Activate LED `x` in row `x`
3. Delay 500 ms
4. `x = x + 1`
5. If `x = 8` reset `x` to 1 else go back to step 2
6. Deactivate all LEDs in row `x`
7. Delay 500 ms
8. `x = x + 1`
9. If `x=8` reset `x` go back to step 1

It becomes clear that it would be convenient to wrap this implementation in some sort of loop. Lets move on to the implementation to see how the code looks like.

## Code Implementation

### Crate Imports
In this implementation, the following crates are required:
- The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
- The `panic_halt` crate to define the panicking behavior to halt on panic.
- The `stm32f4xx_hal` crate to import the STMicro STM32F4 series microcontrollers device hardware abstractions on top of the peripheral access API. The `spi` module and associated types that will be used are also impoted.

```rust
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    pac::{self},
    prelude::*,
    spi::{Mode, NoMiso, Phase, Polarity},
};
```

### Peripheral Configuration Code

#### SPI Peripheral Configuration:

1️⃣ **Obtain a handle for the device peripherals**: Similar to all past blog posts, in embedded Rust, as part of the singleton design pattern, we first have to take the PAC level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let dp = pac::Peripherals::take().unwrap();
```
2️⃣ **Configure the system clocks**: The system clocks need to be configured as they are needed in setting up both the SPI peripheral. To set up the system clocks we need to first promote the `RCC` struct from the PAC and constrain it using the `constrain()` method (more detail on the `constrain` method [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)) to give use access to the `cfgr` struct. After that, we create a `clocks` handle that provides access to the configured (and frozen) system clocks. The clocks are configured to use an HSE frequency of 8MHz by applying the `use_hse()` method to the `cfgr` struct. The HSE frequency is defined by the reference manual of the Nucleo-F401RE development board I am using. Finally, the `freeze()` method is applied to the `cfgr` struct to freeze the clock configuration. Note that freezing the clocks is a protection mechanism by the HAL to avoid the clock configuration changing during runtime. It follows that the peripherals that require clock information would only accept a [frozen `Clocks` configuration struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/rcc/struct.Clocks.html). 

```rust
let rcc = dp.RCC.constrain();
let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
```
2️⃣ **Promote the PAC-level GPIO structs and obtain handles for SCLK, MOSI, and CS pins**: Here I need to configure and obtain handles for the SCLK, MOSI, and CS pins so that they can be controlled by the SPI peripheral. As shown earlier, the SCLK, MOSI, and CS pins are connected to PA5, PA7, and PA6, respectively. As such, before I can obtain any handles I need to promote the pac-level `GPIOA` struct to be able to create handles for individual pins. I do this by using the `split()` method as follows: 
```rust
let gpioa = dp.GPIOA.split();
```
Next I obtain handles for `sclk`, `mosi` and `cs` as follows:
```rust
let sclk = gpioa.pa5.into_alternate();
let mosi = gpioa.pa7.into_alternate();
let mut cs = gpioa.pa6.into_push_pull_output();
```
Note how `sclk` and `mosi` are configured using the `into_alternate()` method. This is because these pins will be routed internal to the microcontroller to the SPI peripheral. As such, the reference manual requires that the pins are configured into alternate so that they can be routed appropriately.

3️⃣ **Configure the SPI peripheral channel**: Looking into the [Nucleo-F401RE board pinout](https://os.mbed.com/platforms/ST-Nucleo-F401RE/), the MOSI and SCL lines pins PA7 and PA5 connect to the SPI1 peripheral in the microcontroller device. As such, this means we need to configure SPI1 and somehow pass it to the handles of the pins we want to use. To configure/instantiate the serial peripheral channel we have two options as seen with some other peripherals. The first is to use the device peripheral handle `dp` to directly access SPI1 and instantiate a transmitter instance using the `spi` method from the [SPI extension traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/spi/trait.SpiExt.html). The second is to use the `new` method in the `Spi` abstraction struct to instantiate a SPI1 instance. Note that both are different ways of doing exactly the same thing!

For the first option, if we examine the `SPI` method signature in the [SPI extension traits](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/spi/trait.SpiExt.html), it looks like this:

```rust
    fn spi<SCK, MISO, MOSI>(
        self, 
        pins: (SCK, MISO, MOSI), 
        mode: impl Into<Mode>, 
        freq: Hertz, 
        clocks: &Clocks
    ) -> Spi<Self, (SCK, MISO, MOSI), TransferModeNormal, Master>
    where
        (SCK, MISO, MOSI): Pins<Self>;
```
The method takes four parameters, a pins instance as a tuple, a mode, an operation frequency, and a frozen `Clocks` instance reference. As such, we can create a handle `spi` for SPI1 as follows:

```rust
    let mut spi = dp.SPI1.spi(
        (sclk, NoMiso {}, mosi),
        Mode {
            polarity: Polarity::IdleLow,
            phase: Phase::CaptureOnFirstTransition,
        },
        2.MHz(),
        &clocks,
    );
```
`sclk`, `mosi`, and `clocks` are the handles that we created earlier. Also `NoMiso{}` is a filler type since we are not doing bidirectional SPI and don't need a MISO line. `Mode` is a [struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/spi/struct.Mode.html) that contains mode information needed for the SPI peripheral and has two options as follows:
 
```rust
pub struct Mode {
    pub polarity: Polarity,
    pub phase: Phase,
}
```
Both `Polarity` and `Phase` are enums with different options. In my instance, I chose the `Polarity::IdleLow` option and `Phase::CaptureOnFirstTransition` option. The choices correspond to what is known as SPI Mode 0 which is what the MAX7219 datasheet defines as the mode of operation.  Additionally, the MAX7219 datasheet states that the device can handle up to 10 MHz as a maximum rate so I only chose an arbitrary value of 2 MHz under the stated limit. 

Alternatively, the second option instantiating SPI is using the `SPI` abstraction looks like this:

```rust
    let mut spi = Spi::new(
        dp.SPI1,
        (sclk, NoMiso {}, mosi),
        Mode {
            polarity: Polarity::IdleLow,
            phase: Phase::CaptureOnFirstTransition,
        },
        2.MHz(),
        &clocks,
    );
```
You can see that the main difference here is that `new` is applied as an instance method on the `Spi` struct. It can be observed here that the `new` method also accepts a fifth parameter here which is an instance of the I2C peripheral `SPI1`. This can be observed in the signature of the instance method in the documentation which looks as follows:

```rust
pub fn new(
    spi: SPI,
    pins: (SCK, MISO, MOSI),
    mode: impl Into<Mode>,
    freq: Hertz,
    clocks: &Clocks
) -> Self
```

This is it for configuration! Let's now jump into the application code.

### Application Code
In the software design described, the first step requires that we write '0x01' to address '0x0C' to power up the MAX7219. As explained earlier, the way data is written to the MAX7219 is by clocking in 16 bits MSB first while the CS line is low. After that the CS line needs to be asserted to latch the data. To send data over SPI  there is a `write` method that has the following signature:

```rust
fn write(&mut self, words: &[u8]) -> Result<(), Self::Error>
```
As can be observed, the `write` method takes a slice of bytes sending everything the slice contains byte after byte. As such, to achieve the described earlier, the following code was written:

```rust
// Prepare Data
let data: u8 = 0x01;
let addr: u8 = 0x0C;
let send_array: [u8; 2] = [addr, data];

// Send Data
cs.set_low();
spi.write(&send_array).unwrap();
cs.set_high();
```

As can be seen, the 16-bit word is packaged in a single array that is passed into the `write` method so that it can be transmitted while CS is low. `data` refers to data that is being written to the `addr` address in the MAX7219. After that CS is asserted to latch the transmitted array. This code is repeated exactly in the same manner with different `addr` and `data` contents for the remaining steps 2, 3, and 4 required to initialize the MAX7219.

All that is left now is to write the code for drawing the diagonal line on the LED matrix. Looking at the steps described earlier, essentially all that needs to be done, is cycle through addresses 1 to 8 that represent the individual columns sending 8-bit data that represents which LEDs are lit. Since we are drawing a diagonal line then one LED needs to be lit up in each row at a time. The LED that is lit will shift one bit to the left as we cycle through the rows. This can be wrapped all in a `for` loop as follows:

```rust
let mut data: u8 = 1;
for addr in 1..9 {
      let send_array: [u8; 2] = [addr, data];
      data = data << 1;

      cs.set_low();
      spi.write(&send_array).unwrap();
      cs.set_high();

      delay.delay_ms(500_u32);
}
```
Note also the delay that is introduced in the end of the loop so that the LEDs can be noticed as they turn on and off. Next we want to clear the rows one by one so a second similar loop can be introduced as follows:

```rust
for addr in 1..9 {
      let send_array: [u8; 2] = [addr, data];
      cs.set_low();
      spi.write(&send_array).unwrap();
      cs.set_high();
      delay.delay_ms(500_u32);
}
```
Keep in mind here that `data` already has a zero value from the previous loop, so it does not need to be updated.

This is it!

## Full Application Code
Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.


```rust
#![no_std]
#![no_main]

// Imports
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    pac::{self},
    prelude::*,
    spi::{Mode, NoMiso, Phase, Polarity, Spi},
};

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let dp = pac::Peripherals::take().unwrap();

    // Configure the system clocks
    // - Promote RCC structure to HAL to be able to configure clocks
    let rcc = dp.RCC.constrain();
    // - Configure system clocks
    // 8 MHz must be used for the Nucleo-F401RE board according to manual
    let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();

    // Configure the SPI pins
    // Don't need miso since communication is simplex (single direction)
    // PA5 and PA7 use SPI1 in the STM32F401RE
    let gpioa = dp.GPIOA.split();
    let sclk = gpioa.pa5.into_alternate();
    let mosi = gpioa.pa7.into_alternate();
    let mut cs = gpioa.pa6.into_push_pull_output();

    // Initialize SPI
    // Theres also two other methods to instantiate 'new' and 'init'!
    let mut spi = dp.SPI1.spi(
        (sclk, NoMiso {}, mosi),
        Mode {
            polarity: Polarity::IdleLow,
            phase: Phase::CaptureOnFirstTransition,
        },
        2.MHz(),
        &clocks,
    );

    // OR you can do something like this
    // let mut spi = Spi::new(
    //     dp.SPI1,
    //     (sclk, NoMiso {}, mosi),
    //     Mode {
    //         polarity: Polarity::IdleLow,
    //         phase: Phase::CaptureOnFirstTransition,
    //     },
    //     2.MHz(),
    //     &clocks,
    // );

    // Application Loop

    // 1) Initalizing Matrix Display

    // 1.a) Power Up Device

    // - Prepare Data to be Sent
    // 8-bit Data/Command Corresponding to Matrix Power Up
    let data: u8 = 0x01;
    // 4-bit Address of Shutdown Mode Command
    let addr: u8 = 0x0C;
    // Package into array to pass to SPI write method
    // Write method will grab array and send all data in it
    let send_array: [u8; 2] = [addr, data];

    // - Send Data
    // Set CS to low to shift/clock bits into max7219 (datasheet requirement)
    cs.set_low();
    // Shift in 16 bits by passing send_array (bits will be shifted MSB first)
    spi.write(&send_array).unwrap();
    // Set CS to high to latch shifted bits into max7219 (datasheet requirement)
    cs.set_high();

    // 1.b) Set up Decode Mode

    // - Prepare Information to be Sent
    // 8-bit Data/Command Corresponding to No Decode Mode
    let data: u8 = 0x00;
    // 4-bit Address of Decode Mode Command
    let addr: u8 = 0x09;
    // Package into array to pass to SPI write method
    // Write method will grab array and send all data in it
    let send_array: [u8; 2] = [addr, data];

    // - Send Data
    // Set CS to low to shift/clock bits into max7219 (datasheet requirement)
    cs.set_low();
    // Shift in 16 bits by passing send_array (bits will be shifted MSB first)
    spi.write(&send_array).unwrap();
    // Set CS to high to latch shifted bits into max7219 (datasheet requirement)
    cs.set_high();

    // 1.c) Configure Scan Limit

    // - Prepare Information to be Sent
    // 8-bit Data/Command Corresponding to Scan Limit Displaying all digits
    let data: u8 = 0x07;
    // 4-bit Address of Scan Limit Command
    let addr: u8 = 0x0B;
    // Package into array to pass to SPI write method
    // Write method will grab array and send all data in it
    let send_array: [u8; 2] = [addr, data];

    // - Send Data
    // Set CS to low to shift/clock bits into max7219 (datasheet requirement)
    cs.set_low();
    // Shift in 16 bits by passing send_array (bits will be shifted MSB first)
    spi.write(&send_array).unwrap();
    // Set CS to high to latch shifted bits into max7219 (datasheet requirement)
    cs.set_high();

    // 1.c) Configure Intensity

    // - Prepare Information to be Sent
    // 8-bit Data/Command Corresponding to (15/32 Duty Cycle) Medium Intensity
    let data: u8 = 0x07;
    // 4-bit Address of Intensity Control Command
    let addr: u8 = 0x0A;
    // Package into array to pass to SPI write method
    // Write method will grab array and send all data in it
    let send_array: [u8; 2] = [addr, data];

    // - Send Data
    // Set CS to low to shift/clock bits into max7219 (datasheet requirement)
    cs.set_low();
    // Shift in 16 bits by passing send_array (bits will be shifted MSB first)
    spi.write(&send_array).unwrap();
    // Set CS to high to latch shifted bits into max7219 (datasheet requirement)
    cs.set_high();

    let mut delay = dp.TIM2.delay_ms(&clocks);

    loop {
        let mut data: u8 = 1;
        // Iterate over all rows of LED matrix
        for addr in 1..9 {
            // addr refrences the row data will be sent to
            let send_array: [u8; 2] = [addr, data];
            // Shift a 1 with evey loop
            data = data << 1;

            // Send data just like earlier
            cs.set_low();
            spi.write(&send_array).unwrap();
            cs.set_high();

            // Delay for 500ms to show effect
            delay.delay_ms(500_u32);
        }

        // Clear the LED matrix row by row with 500ms delay in between
        for addr in 1..9 {
            let send_array: [u8; 2] = [addr, data];
            cs.set_low();
            spi.write(&send_array).unwrap();
            cs.set_high();
            delay.delay_ms(500_u32);
        }
    }
}
``` 

## Conclusion
In this post, a LED dot matrix display simple application was created by configuring and controlling the MAX7219 LED driver. This was using the SPI peripheral for the STM32F401RE microcontroller on the Nucleo-F401RE development board. This code will be used as a base for a blog series about creating platform-agnostic drivers in Rust. Additionally, all code was created at the HAL level using the [stm32f4xx Rust HAL](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/index.html). Have any questions? Share your thoughts in the comments below 👇. 

%%[subend]