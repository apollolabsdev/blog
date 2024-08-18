---
title: "Embedded Rust & Embassy: Analog Sensing with ADCs"
datePublished: Mon Dec 12 2022 19:45:56 GMT+0000 (Coordinated Universal Time)
cuid: clbl7g8bn000808jv7kbggirs
slug: embedded-rust-embassy-analog-sensing-with-adcs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1670873521393/xfbl3VSY8.png
tags: tutorial, iot, rust, embedded

---

> This blog post is the fourth of a multi-part series of posts about Rust embassy for the STM32. This post is going to explore reading Analog values using the embassy HAL. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.

Prior posts include (in order of publishing):

1.  [Embedded Rust & Embassy: GPIO Button Controlled Blinking](https://apollolabsblog.hashnode.dev/embedded-rust-embassy-gpio-button-controlled-blinking)
    
2.  [Embedded Rust & Embassy: UART Serial Communication](https://apollolabsblog.hashnode.dev/embedded-rust-embassy-uart-serial-communication)
    
3.  [Embedded Rust & Embassy: PWM Generation](https://apollolabsblog.hashnode.dev/embedded-rust-embassy-pwm-generation)
    

%%[substart] 

## Introduction

Apart from a few unclarities here and there, working with embassy thus far has been a joy. What I've been doing thus far is rewriting past posts in embassy to compare. Given the lesser amount of needed code, I feel that embassy is on a path to becoming the framework of choice for teaching embedded Rust. Not only because of less verbosity but also because the function interfaces are more readable. However, at least from an stm32 context, there is still some work that needs to be done at least on the documentation side to make things more accessible.

In this post, I will recreate the [analog sensor reading application](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-analog-temperature-sensing-using-the-adc) I created with the stm32f4xx-hal. The post will be self-contained so there is no need to refer back to the past post unless one is interested in comparing. We'll see that setting up a simple ADC reading is fairly straightforward in embassy, but there are a few things that one needs to be aware of.

### **📚** Knowledge Pre-requisites

To understand the content of this post, you need the following:

*   Basic knowledge of coding in Rust.
    
*   Familiarity with the basic template for creating embedded applications in Rust.
    
*   Familiarity with [UART communication basics](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter).
    
*   Familiarity with the working principles of NTC Thermistors. [This](https://www.electronics-tutorials.ws/io/thermistors.html) page is a good resource.
    

### **💾** Software Setup

All the code presented in this post in addition to instructions for the environment and toolchain setup is available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

In addition to the above, you would need to install some sort of serial communication terminal on your host PC. Some recommendations include:

**For Windows**:

*   [PuTTy](https://www.putty.org/)
    
*   [Teraterm](https://ttssh2.osdn.jp/index.html.en)
    
*   [Serial Studio](https://serial-studio.github.io/)
    

**For Mac and Linux**:

*   [minicom](https://wiki.emacinc.com/wiki/Getting_Started_With_Minicom)
    
*   [Serial Studio](https://serial-studio.github.io/)
    

Apart from Serial Studio, some detailed instructions for the different operating systems are available in the [Discovery Book](https://docs.rust-embedded.org/discovery/microbit/06-serial-communication/index.html).

For me, Serial Studio comes highly recommended. I personally came across [Serial Studio](https://serial-studio.github.io/) recently and found it to be awesome for two main reasons. First is that you can skip many of those instructions for other tools, especially in Mac and Linux systems. Second, if you are you want to graph data over UART, it has a really nice and easy-to-configure setup. It's also open-source and free to use.

### **🛠** Hardware Setup

#### **👔** Materials

*   [Nucleo-F401RE board](https://amzn.to/3yn6AIb)
    

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779876602/9SAtIAZXK.jpg align="center")

*   Seeed Studio [Grove Base Shield V2.0](https://www.seeedstudio.com/base-shield-v2.html)
    

![Base_Shield_v2-1.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779892425/x3AcSK-0N.png align="center")

*   Seeed Studio [Temperature Sensor](https://www.seeedstudio.com/Grove-Temperature-Sensor.html). The module uses the [NCP18WF104F03RC](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/NCP18WF104F03RC.pdf) NTC Thermistor.
    

![Grove_Temperature_Sensor.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779961407/NCTaek6Eg.jpeg align="center")

**🚨 Important Note:**

> I used the Grove modular system for connection ease. It is a more elegant approach and less prone to mistakes. To directly wire the NTC temperature sensor to the board, one would need to build a circuit similar to the one shown in [this](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/Grove_-_Temperature_sensor_v1.1.pdf) schematic.

#### **🔌** Connections

*   Temperature sensor signal pin connected to pin PA0 (Grove Connector A0).
    
*   The UART Tx line that connects to the PC through the onboard USB bridge is via pin PA2 on the microcontroller. This is a hardwired pin, meaning you cannot use any other for this setup. Unless you are using a different board other than the Nucleo-F401RE, you have to check the relevant documentation (reference manual or datasheet) to determine the number of the pin.
    

#### 🔬 Circuit Analysis

The temperature sensor used has a single-pin interface called "signal" that provides a voltage output. The temperature sensor is also a negative temperature coefficient (NTC) sensor. This means the resistance of the sensor increases as the temperature increases. The following figure shows the schematic of the temperature sensor circuit for the grove module utilized.

![Temperature Sensor Schematic](https://cdn.hashnode.com/res/hashnode/image/upload/v1658850715996/o4ELQnm78.png align="center")

It is shown that the [NCP18WF104F03RC](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/NCP18WF104F03RC.pdf) NTC Thermistor is connected in a voltage divider configuration with a 100k resistor. The Op-Amp only acts as a voltage follower (or buffer). As such, the voltage at the positive terminal of the op-amp \\(V_{+}\\) is equal to the voltage on the signal terminal and expressed as:

\\[V_{\text{+}} = V_{cc} * \frac{R_{1}}{R_{1} + R_{\text{NTC}}}\\]

Where \\(R_1 = 100k\Omega\\) and the resistance value of \\(R_{\text{NTC}}\\) is the one that needs to be calculated to obtain the temperature. This means that later in the code, I would need to retrieve back the value of \\(R_{\text{NTC}}\\) from the \\(V_{\text{+}}\\) value that is being read by the ADC. With some algebraic manipulation we can move all the known variables to the right-hand side of the equation to reach the following expression:

\\[R_{\text{NTC}} = \left( \frac{ V_{cc} }{ V_{\text{+}} } -1 \right) * R_{1}\\]

After extracting the value of \\(R_{\text{NTC}}\\), I would need to determine the temperature. Following the equations in the datasheet, I leverage the Steinhart-Hart NTC equation that is presented as follows:

\\[\beta = \frac{ln(\frac{R_{\text{NTC}}}{R_0})}{(\frac{1}{T}-\frac{1}{T_0})}\\]

where \\( \beta \\) is a constant and equal to 4275 for our NTC as stated by the datasheet and \\( T \\) is the temperature we are measuring. \\( T_0 \\) and \\( R_0 \\) refer to the ambient temperature (typically 25 Celcius) and resistance at ambient temperature, respectively. For the Grove module used, again from the datasheet, the value of the resistance at 25 Celcius ( \\( T_0 \\) ) is equal to \\( 100k\Omega \\) ( \\( R_0 \\) ). With more algebraic manipulation we solve for \\( T \\) to get:

\\[T = \frac{1}{\frac{1}{\beta} * ln(\frac{R_{\text{NTC}}}{R_0}) +\frac{1}{T_0}}\\]

## **👨‍🎨** Software Design

Now that we know the equations from the prior section, an algorithm needs to be developed and is quite straightforward in this case. After configuring the device (including ADC and UART peripherals), the algorithmic steps are as follows:

1.  Kick off the ADC and obtain a reading/sample.
    
2.  Calculate the temperature in Celcius.
    
3.  Send the temperature value over UART.
    
4.  Go back to step 1.
    

## **👨‍💻** Code Implementation

### **📥** Crate Imports

In this implementation, the following crates are required:

*   The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
    
*   The `libm::log` crate that is a math crate that will allow me to calculate the natural logarithm.
    
*   The `heapless::String` crate to create a fixed capacity `String.`
    
*   The `core::fmt` crate will allow us to use the `writeln!` macro for print formatting.
    
*   The `panic_halt` crate to define the panicking behavior to halt on panic.
    
*   The `embassy_stm32` crate to import the embassy STM32 series microcontroller device hardware abstractions. The needed abstractions are imported accordingly.
    
*   The `embassy_time` crate to import timekeeping capabilities.
    

```rust
use core::fmt::Write;
use heapless::String;
use libm::log;
use cortex_m_rt::entry;
use embassy_stm32::adc::Adc;
use embassy_stm32::dma::NoDma;
use embassy_stm32::usart::{Config, UartTx};
use embassy_time::Delay;
use panic_halt as _;
```

### **🎛** Peripheral Configuration Code

#### ADC Peripheral Configuration

1️⃣ **Initialize MCU and obtain a handle for the device peripherals**: A device peripheral handler `p` is created:

```rust
let p = embassy_stm32::init(Default::default());
```

2️⃣ **Configure ADC and obtain handle**: ADCs in microcontrollers typically have many configuration options. At the time of writing this post, the implementation and documentation of ADC at the embassy HAL level are a bit limited. Things that I've noticed missing from an ADC embassy HAL perspective include the following:

*   There isn't any interrupt or async support.
    
*   DMA support seems to be missing as well.
    
*   There is unclarity in some documentation aspects (for example is `delay` parameter described in `new` method).
    
*   Not all device configuration options are adjustable.
    
*   To find out the default configuration of the ADC peripheral one needs to navigate the source.
    

The ADC peripheral configuration is actually quite simple. The driver struct contains only a few methods. In the [documentation](https://docs.embassy.dev/embassy-stm32/git/stm32f423ch/adc/struct.Adc.html), there is a `new` method as part of the `Adc` struct abstraction to configure an ADC peripheral so that we can obtain a handle. `new` has the following signature:

```rust
pub fn new(
    _peri: impl Peripheral<P = T> + 'd,
    delay: &mut impl DelayUs<u32>
) -> Self
```

`new` takes two parameters where `peri` expects an argument passing in an ADC peripheral instance and `delay` that expects a `Delay` abstraction. Unfortunately, the documentation doesn't really explain the point behind the `delay` parameter. For now, it does not seem to serve any useful purpose. Following the above, an `adc` handle is created as follows:

```rust
let mut delay = Delay;
let mut adc = Adc::new(p.ADC1, &mut delay);
```

Two questions remain here though, first, where is the pin that we will be reading from defined? and second, what is the configuration?

Regarding the pin, it turns out that it will be passed as a parameter into the `read` method later. The `read` method is the one that triggers a conversion.

Regarding the configuration, one needs to navigate the source code of the documentation. From what I've found, the default \\(V_{\textit{ref}}\\) is 3.3V, the default resolution is 12 bits, and the default sample time setting is 3 clock cycles. It seems also, although not clear in the documentation, that in the current state the ADC supports only one-shot conversion. That would probably make sense anyway since DMA is still not supported either. For understanding in detail what each configuration parameter means, detail is provided in the [stm32f401re reference manual](https://www.st.com/resource/en/reference_manual/dm00096844-stm32f401xb-c-and-stm32f401xd-e-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf).

#### UART Peripheral Configuration

1️⃣ **Configure UART and obtain handle**: On the Nucleo-F401RE board pinout, the Tx line pin PA2 connects to the USART2 peripheral in the microcontroller device. Similar to what was done in the [embassy UART post](https://apollolabsblog.hashnode.dev/embedded-rust-embassy-uart-serial-communication), an instance of USART2 is attached to the `usart` handle as follows:

```rust
let mut usart = UartTx::new(p.USART2, p.PA2, NoDma, Config::default());
```

Also, similar to before a `String` type `msg` handle is created to store the formatted text that will be transmitted over UART:

```rust
let mut msg: String<64> = String::new();
```

This concludes the configuration aspect of the code.

### **📱**Application Code

Following the design described earlier, before entering the `loop`, I first need to set up a couple of static values that I will be using in the conversion calculations. This includes keying in the constant values for \\( \beta \\) and \\(R_0\\) as follows:

```rust
    static R0: f64 = 100000.0;
    static B: f64 = 4275.0; // B value of the thermistor
```

After entering the program loop, as the software design stated earlier, the first thing I need to do is kick off the ADC to obtain a sample. In the [documentation](https://docs.embassy.dev/embassy-stm32/git/stm32f423ch/adc/struct.Adc.html), I found a `read` method with the following signature:

```rust
pub fn read<P>(&mut self, pin: &mut P) -> u16
where
    P: AdcPin<T>,
    P: Pin,
```

As shown, we need to pass a reference to `pin` which will be the actual pin instance that will connect to the sensor output.

Then an ADC sample is obtained as follows.

```rust
 let sample = adc.read(&mut p.PA0);
```

Next, I convert the sample value to a temperature by implementing the earlier derived equations as follows:

```rust
let mut r: f64 = 4094.0 / sample as f64 - 1.0;
r = R0 * r;
let temperature = (1.0 / (log(r / R0) / B + 1.0 / 298.15)) - 273.15;
```

A few things to note here; first I don't convert the collected sample to value to a voltage as in the first calculation the voltage calculation is a ratio. This means I keep the `sample` in LSBs and use the equivalent LSB value for \\(V_{cc}\\). To plug in \\(V_{cc}\\) I simply calculate the maximum possible LSB value (upper reference) that can be generated by the ADC. This is why I needed to know the resolution, which was 12 because \\(V_{cc} = 2^{12} LSBs\\) . Second, recall from the `convert` signature that `sample` is a `u16`, so I had to use `as f64` to cast it as an `f64` for the calculation. Third, `log` is the natural logarithm and obtained from the `libm` library that I imported earlier. Fourth, and last, the temperature is calculated in Kelvins, the `273.15` is what converts it to Celcius.

Finally, now that the temperature is available, similar to the [embassy UART](https://apollolabsblog.hashnode.dev/embedded-rust-embassy-uart-serial-communication) example, a message is prepared and sent over UART as follows:

```rust
// Format Message
core::writeln!(&mut msg, "Temperature {:02} Celcius\r", temperature).unwrap();

// Transmit Message
usart.blocking_write(msg.as_bytes()).unwrap();

// Clear String for next message
msg.clear();
```

This is it!

## 📀 Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.

```rust
#![no_std]
#![no_main]
#![feature(type_alias_impl_trait)]

use core::fmt::Write;

use heapless::String;

use libm::log;

use cortex_m_rt::entry;
use embassy_stm32::adc::Adc;
use embassy_stm32::dma::NoDma;
use embassy_stm32::usart::{Config, UartTx};
use embassy_time::Delay;
use panic_halt as _;

#[entry]
fn main() -> ! {
    // Initialize and create handle for devicer peripherals
    let mut p = embassy_stm32::init(Default::default());

    // ADC Configuration
    let mut delay = Delay;
    // Create Handler for adc peripheral (PA0 is connected to ADC1)
    let mut adc = Adc::new(p.ADC1, &mut delay);

    //Configure UART
    let mut usart = UartTx::new(p.USART2, p.PA2, NoDma, Config::default());

    // Create empty String for message
    let mut msg: String<64> = String::new();

    static R0: f64 = 100000.0;
    static B: f64 = 4275.0; // B value of the thermistor

    // Algorithm
    // 1) Get adc reading
    // 2) Convert to temperature
    // 3) Send over Serial
    // 4) Go Back to step 1

    // Application Loop
    loop {
        // Get ADC reading
        let sample = adc.read(&mut p.PA0);

        //Convert to temperature
        let mut r: f64 = 4094.0 / sample as f64 - 1.0;
        r = R0 * r;
        let temperature = (1.0 / (log(r / R0) / B + 1.0 / 298.15)) - 273.15;

        // Format Message
        core::writeln!(&mut msg, "Temperature {:02} Celcius\r", temperature).unwrap();

        // Transmit Message
        usart.blocking_write(msg.as_bytes()).unwrap();

        // Clear String for next message
        msg.clear();
    }
}
```

## Conclusion

In this post, an analog temperature measurement application was created leveraging the ADC peripheral using Rust on the Nucleo-F401RE development board. The resulting measurement is also sent over to a host PC over a UART connection. All code was created leveraging the embassy framework for STM32. As things stand right now, the STM32 embassy HAL can only provide a simple implementation of ADCs and is behind on several features. At a minimum, interrupts do not seem to be supported yet. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]