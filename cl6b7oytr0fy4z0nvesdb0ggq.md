---
title: "STM32F4 Embedded Rust at the HAL: Analog Temperature Sensing using the ADC"
datePublished: Mon Aug 01 2022 20:36:29 GMT+0000 (Coordinated Universal Time)
cuid: cl6b7oytr0fy4z0nvesdb0ggq
slug: stm32f4-embedded-rust-at-the-hal-analog-temperature-sensing-using-the-adc
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1660760161365/8fjtt9qRA.png
tags: tutorial, beginners, rust, embedded

---

> This blog post is the sixth of a multi-part series of posts where I explore various peripherals in the STM32F401RE microcontroller using embedded Rust at the HAL level. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.

Prior posts include (in order of publishing):
1. [STM32F4 Embedded Rust at the HAL: GPIO Button Controlled Blinking](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-gpio-button-controlled-blinking)
2. [STM32F4 Embedded Rust at the HAL: Button Controlled Blinking by Timer Polling](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-button-controlled-blinking-by-timer-polling)
3. [STM32F4 Embedded Rust at the HAL: UART Serial Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication)
4. [STM32F4 Embedded Rust at the HAL: PWM Buzzer](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-pwm-buzzer)
5. [STM32F4 Embedded Rust at the HAL: Timer Ultrasonic Distance Measurement](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-timer-ultrasonic-distance-measurement)

%%[substart]

## Introduction
In this post, I will be configuring and setting up an stm32f4xx-hal ADC to measure ambient temperature using the [NCP18WF104F03RC](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/NCP18WF104F03RC.pdf) NTC Thermistor. Temperature measurement will be continuously collected and sent to a PC terminal over UART. I will also be leveraging the [UART Serial Communication](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication) application/code from a previous post. Additionally, I will not be using any interrupts and the example will be set up as a simplex system that transmits in one direction only (towards the PC).


### Knowledge Pre-requisites
To understand the content of this post, you need the following:
- Basic knowledge of coding in Rust.
- Familiarity with the basic template for creating embedded applications in Rust.
- Familiarity with [UART communication basics](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter). 
- Familiarity with working principles of NTC Thermistors. [This](https://www.electronics-tutorials.ws/io/thermistors.html) page is a good resource.

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

![nucleof401re.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779876602/9SAtIAZXK.jpg align="center")
- Seeed Studio [Grove Base Shield V2.0](https://www.seeedstudio.com/base-shield-v2.html)

![Base_Shield_v2-1.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779892425/x3AcSK-0N.png align="center")
- Seeed Studio [Temperature Sensor](https://www.seeedstudio.com/Grove-Temperature-Sensor.html). The module uses the [NCP18WF104F03RC](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/NCP18WF104F03RC.pdf) NTC Thermistor.

![Grove_Temperature_Sensor.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1659779961407/NCTaek6Eg.jpeg align="center")

 **🚨 Important Note:**
> I used the Grove modular system for connection ease. It is a more elegant approach and less prone to mistakes. To directly wire the NTC temperature sensor to the board, one would need to build a circuit similar to the one shown in [this](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/Grove_-_Temperature_sensor_v1.1.pdf) schematic.

#### Connections
- Temperature sensor signal pin connected to pin PA0 (Grove Connector A0).
- The UART Tx line that connects to the PC through the onboard USB bridge is via pin PA2 on the microcontroller. This is a hardwired pin, meaning you cannot use any other for this setup. Unless you are using a different board other than the Nucleo-F401RE, you have to check the relevant documentation (reference manual or datasheet) to determine the number of the pin.

#### Circuit Analysis
The temperature sensor used has a single-pin interface called "signal" that provides a voltage output. The temperature sensor is also a negative temperature coefficient (NTC) sensor. This means the resistance of the sensor increases as the temperature increases. The following figure shows the schematic of the temperature sensor circuit for the grove module utilized.

![Temperature Sensor Schematic](https://cdn.hashnode.com/res/hashnode/image/upload/v1658850715996/o4ELQnm78.png align="center")

It is shown that the [NCP18WF104F03RC](https://files.seeedstudio.com/wiki/Grove-Temperature_Sensor_V1.2/res/NCP18WF104F03RC.pdf) NTC Thermistor is connected in a voltage divider configuration with a 100k resistor. The Op-Amp only acts as a voltage follower (or buffer). As such, the voltage at the positive terminal of the op-amp \\( V_{\+} \\) is equal to the voltage on the signal terminal and expressed as:

$$
 V\_{\text{+}}  = V\_{cc} * \frac{R\_{1}}{R\_{1} + R\_{\text{NTC}}}
$$

Where \\( R\_1 = 100k\Omega \\) and the resistance value of \\( R\_{\text{NTC}} \\) is the one that needs to be calculated to obtain the temperature. This means that later in the code, I would need to retrieve back the value of \\( R\_{\text{NTC}} \\) from the \\( V\_{\text{+}} \\) value that is being read by the ADC. With some algebraic manipulation we can move all the known variables to the right hand side of the equation to reach the following expression:

$$
R\_{\text{NTC}} = \left( \frac{ V\_{cc} }{ V\_{\text{+}}  }  -1 \right) * R\_{1}
$$

After extracting the value of \\( R\_{\text{NTC}} \\), I would need to determine the temperature. Following the equations in the datasheet, I leverage the Steinhart-Hart NTC equation that is presented as follows:

\\[
\beta = \frac{ln(\frac{R\_{\text{NTC}}}{R_0})}{(\frac{1}{T}-\frac{1}{T_0})}
\\]

where \\( \beta \\) is a constant and equal to 4275 for our NTC as stated by the datasheet and \\( T \\) is the temperature we are measuring. \\( T_0 \\) and \\( R_0 \\) refer to the ambient temperature (typically 25 Celcius) and resistance at ambient temperature, respectively. For the Grove module used, again from the datasheet, the value of the resistance at 25 Celcius (\\( T_0 \\)) is equal to \\( 100k\Omega \\) (\\( R_0 \\)). With more algebraic manipulation we solve for \\( T \\) to get:

\\[
T = \frac{1}{\frac{1}{\beta} * ln(\frac{R\_{\text{NTC}}}{R_0}) +\frac{1}{T_0}}
\\]

## Software Design

Now that we know the equations from the prior section, an algorithm needs to be developed and is quite straightforward in this case. After configuring the device (including ADC and UART peripherals), the algorithmic steps are as follows:

1. Kick off the ADC and obtain a reading/sample.
2. Calculate the temperature in Celcius.
3. Send the temperature value over UART.
4. Go back to step 1.

## Code Implementation

### Crate Imports
In this implementation, the following crates are required:
- The `cortex_m_rt` crate for startup code and minimal runtime for Cortex-M microcontrollers.
- The `libm::log` crate that is a math crate that will allow me to calculate the natural logarithm. 
- The `core::fmt` crate will allow us to use the `writeln!` macro for easy printing.
- The `panic_halt` crate to define the panicking behavior to halt on panic.
- The `stm32f4xx_hal` crate to import the STMicro STM32F4 series microcontrollers device hardware abstractions on top of the peripheral access API.

```rust
use core::fmt::Write; // allows use to use the WriteLn! macro for easy printing
use cortex_m_rt::entry;
use libm::log;
use panic_halt as _;
use stm32f4xx_hal::{
    adc::{config::AdcConfig, config::SampleTime, Adc},
    pac,
    prelude::*,
    serial::config::Config,
};
```

### Peripheral Configuration Code

#### GPIO Peripheral Configuration:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to take the PAC level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let dp = pac::Peripherals::take().unwrap();
```
2️⃣ **Promote the PAC-level GPIO structs**: I need to configure the signal pin as input in the beginning and obtain a handler for the pin so that I can control it. I also need to obtain a handle for the ADC signal pin. The pin PA0 is part of `GPIOA`. Before I can obtain any handles I need to promote the pac-level `GPIOA` struct to be able to create handles for individual pins. I do this by using the `split()` method as follows: 
```rust
let gpioa = dp.GPIOA.split();
```
3️⃣ **Obtain a handle for the signal pin and configure it to an analog pin**: As earlier stated, the signal pin is connected to pin PA0 (Pin 0 Port A). As such, I need to create a handle for the signal pin that has PA0 configured to an analog pin. I will name the handle `temperature_pin` and configure it as follows: 
```rust
let temperature_pin = gpioa.pa0.into_analog();
```
Note here that the `temperature_pin ` handle here does not need to be mutable since we will only be reading it.

#### ADC Peripheral Configuration:

**Obtain a handle and configure the signal pin**: ADCs in microcontrollers typically have many configuration options. At the time of writing this post, only (the most important) two were implemented for the stm32f4xx family. Please refer to the [documentation](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/adc/struct.Adc.html) for the latest updates. The implemented two are one-shot and sequence conversions. Sequence conversions as the name implies do a sequence of conversions based on a single trigger. For one-shot, a sample will be collected per trigger. One-shot is what I will be using. 

The ADC peripheral configuration actually turns out to be quite simple and can be done in a single line. According to the stm32f401re datasheet, `PA0` is connected to ADC1, and digging into the HAL [documentation](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/adc/struct.Adc.html), we find an `adc1` method as part of the `ADC` struct abstraction to configure an ADC peripheral so that we can obtain a handle. `adc1` has the following signature:

```rust
pub fn adc1(adc: ADC1, reset: bool, config: AdcConfig) -> Adc<ADC1>
```
From the method description for the `adc1` method the documentation states:
> Enables the ADC clock, resets the peripheral (optionally), runs calibration and applies the supplied config

As such, the first parameter requires an instance of the `ADC1`, the second parameter defines if we want to reset the ADC or not, and the third applies the configuration using the `AdcConfig` struct. From the examples in the documentation, it turns out also that there is a `default` configuration that can be applied. However, similar to my past struggles with finding out the default configuration in my [UART post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-uart-serial-communication) I also had to do the same thing here and dig into the source code to find out what the default config is:

```rust
impl Default for AdcConfig {
        fn default() -> Self {
            Self {
                clock: Clock::Pclk2_div_2,
                resolution: Resolution::Twelve,
                align: Align::Right,
                scan: Scan::Disabled,
                external_trigger: (TriggerMode::Disabled, ExternalTrigger::Tim_1_cc_1),
                continuous: Continuous::Single,
                dma: Dma::Disabled,
                end_of_conversion_interrupt: Eoc::Disabled,
                default_sample_time: SampleTime::Cycles_480,
                vdda: None,
            }
        }
    }
```
The default configuration configures the ADC in one-shot mode as indicated by the `continuous` parameter, but what was really important for me to know here as well, is the resolution. This is because I would need the resolution value to use in the calculation of the temperature later. As shown in the `resolution` parameter, the ADC is configured to a 12-bit resolution by default.

After having all the information needed, I create an ADC handle `adc`, as follows:

```rust
let mut adc = Adc::adc1(dp.ADC1, true, AdcConfig::default());
```

** 📝 Note: **
> As opposed to other peripherals I've configured in past posts, the ADC has only one approach to be configured. Some other peripherals (ex. UART) could be configured by using the device peripheral handle `dp` to directly access the peripheral and instantiate an instance using one of the methods from the extension traits. The second was to use a method in the peripheral abstraction struct to instantiate an instance. In the stm32f4xx-hal the ADC only supports the latter.

#### Serial Communication Peripheral Configuration:

1️⃣ **Configure the system clocks**: The system clocks need to be configured as they are needed in setting up the UART peripheral. To set up the system clocks we need to first promote the RCC struct from the PAC and constrain it using the `constrain()` method (more detail on the `constrain` method [here](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)) to give use access to the `cfgr` struct. After that, we create a `clocks` handle that provides access to the configured (and frozen) system clocks. The clocks are configured to use an HSE frequency of 8MHz by applying the `use_hse()` method to the `cfgr` struct. The HSE frequency is defined by the reference manual of the Nucleo-F401RE development board. Finally, the `freeze()` method is applied to the `cfgr` struct to freeze the clock configuration. Note that freezing the clocks is a protection mechanism by the HAL to avoid the clock configuration changing during runtime. It follows that the peripherals that require clock information would only accept a [frozen `Clocks` configuration struct](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/rcc/struct.Clocks.html). 

```rust
let rcc = dp.RCC.constrain();
let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
```
** 🚨 Important Note: **
> Using a frequency different than 8 MHz for HSE on the Nucleo-F401RE board will cause the UART to output erroneous characters. This value needs to be adjusted to what the individual board settings are.

2️⃣ **Obtain a handle and configure the serial transmit (Tx) pin**: Since the Tx button is `PA2`, earlier I had already created a handle for `gpioa` that I have to leverage. However, now that we are not using the pin as a regular GPIO input or output it means that the pin needs to be connected to a different peripheral internal to the microcontroller. The pin can be configured as such using the `into_alternate()` method as follows.

```rust    
let tx_pin = gpioa.pa2.into_alternate();
```
3️⃣ **Configure the serial peripheral channel**: Looking into the [Nucleo-F401RE board pinout](https://os.mbed.com/platforms/ST-Nucleo-F401RE/), the Tx line pin PA2 connects to the USART2 peripheral in the microcontroller device. As such, this means we need to configure USART2 and somehow pass it to the handle of the pin we want to use. This is done as follows:

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

This is it for configuration! Let's now jump into the application code.

### Application Code

Following the design described earlier, before entering my `loop`, I first need to set up a couple of statics that I will be using in my calculations. This includes keying in the constant values for \\( \beta \\) and \\( R\_0 \\) as follows:

```rust
    static R0: f64 = 100000.0;
    static B: f64 = 4275.0; // B value of the thermistor
```

After entering the program loop, as the software design stated earlier, first thing I need to do is kick off the ADC to obtain a sample. With some digging into the [documentation](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/adc/struct.Adc.html#), I found a `convert` method with the following signature:

```rust
pub fn convert<PIN>(&mut self, pin: &PIN, sample_time: SampleTime) -> u16
```
so where we need to pass a reference to `PIN` which will be the `temperature_pin` handle I created earlier and also a `SampleTime`. `SampleTime` is simply an enum that specifies the number of cycles to sample a given channel for. The options available are defined as follows:

```rust
pub enum SampleTime {
    Cycles_3,
    Cycles_15,
    Cycles_28,
    Cycles_56,
    Cycles_84,
    Cycles_112,
    Cycles_144,
    Cycles_480,
}
```
For understanding in detail what each means, detail is provided in the [stm32f401re reference manual](https://www.st.com/resource/en/reference_manual/dm00096844-stm32f401xb-c-and-stm32f401xd-e-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf). For the purposes of this post, I chose `Cycles_480`. Following the above detail, I obtain a single ADC sample (one-shot conversion) as follows.

```rust
 let sample = adc.convert(&temperature_pin, SampleTime::Cycles_480);
```
Next, I convert the sample value to a temperature by implementing the earlier derived equations as follows:

```rust
let mut r: f64 = 4094.0 / sample as f64 - 1.0;
r = R0 * r;
let temperature = (1.0 / (log(r / R0) / B + 1.0 / 298.15)) - 273.15;
```
A few things to note here; first I don't convert the collected sample to value to a voltage as in the first calculation the voltage calculation is a ratio. This means I keep the `sample` in LSBs and use the equivalent LSB value for \\( V\_{cc} \\). To plug in \\( V\_{cc} \\) I simply calculate the maximum possible LSB value (upper reference) that can be generated by the ADC. This is why I needed to know the resolution, which was 12 because \\( V\_{cc} = 2^{12} LSBs \\). Second, recall from the `convert` signature that `sample` is a `u16`, so I had to use `as f64` to cast it as an `f64` for the calculation. Third, `log` is the natural logarithm and obtained from the `libm` library that I imported earlier. Fourth, and last, the temperature is calculated in Kelvins, the `273.15` is what converts it to Celcius.

Finally, now that the temperature is available, I send it over UART using the `writeln!` macro as follows:

```rust
writeln!(tx, "Temperature {:02} Celcius\r", temperature).unwrap();
```
This is it!

## Full Application Code
Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabsdev Nucleo-F401RE](https://github.com/apollolabsdev/stm32-nucleo-f401re) git repo.


```rust
#![no_std]
#![no_main]

// Imports
use core::fmt::Write; // allows use to use the WriteLn! macro for easy printing
use cortex_m_rt::entry;
use libm::log;
use panic_halt as _;
use stm32f4xx_hal::{
    adc::{config::AdcConfig, config::SampleTime, Adc},
    pac,
    prelude::*,
    serial::config::Config,
};

#[entry]
fn main() -> ! {
    // Setup handler for device peripherals
    let dp = pac::Peripherals::take().unwrap();

    // ADC Configuration Steps:
    // 1) Configure the temperature sensor temperature pin into analog and obtain handler.
    let gpioa = dp.GPIOA.split();
    let temperature_pin = gpioa.pa0.into_analog();
    // 2) Create Handler for adc peripheral (PA0 is connected to ADC1)
    // Configure ADC for single shot conversion
    let mut adc = Adc::adc1(dp.ADC1, true, AdcConfig::default());

    // Serial config steps:
    // 1) Need to configure the system clocks
    // - Promote RCC structure to HAL to be able to configure clocks
    let rcc = dp.RCC.constrain();
    // - Configure system clocks
    // 8 MHz must be used for the Nucleo-F401RE board according to manual
    let clocks = rcc.cfgr.use_hse(8.MHz()).freeze();
    // 2) Configure/Define TX pin
    // Note that we already split port A earlier for the led pin
    // Use PA2 as it is connected to the host serial interface
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
                .baudrate(115200.bps())
                .wordlength_8()
                .parity_none(),
            &clocks,
        )
        .unwrap();

    static R0: f64 = 100000.0;
    static B: f64 = 4275.0; // B value of the thermistor

    // Algorithim
    // 1) Get adc reading
    // 2) Convert to temperature
    // 3) Send over Serial
    // 4) Go Back to step 1

    // Application Loop
    loop {
        // Get ADC reading
        let sample = adc.convert(&temperature_pin, SampleTime::Cycles_480);

        //Convert to temperature
        let mut r: f64 = 4094.0 / sample as f64 - 1.0;
        r = R0 * r;
        let temperature = (1.0 / (log(r / R0) / B + 1.0 / 298.15)) - 273.15;

        // Send temperature to serial interface
        writeln!(tx, "Temperature {:02} Celcius\r", temperature).unwrap();
    }
}
``` 

## Further Experimentation/Ideas:
Some ideas to experiment with include:
- Refactor code to obtain a different resolution measurement that the stm32f401re supports (ex. 10-bits or 8-bits).
- Note how in my code I didn't use voltages in my calculations but rather LSBs. For that, I had to know the resolution. If you dig into documentation, you'll notice that in the ADC [documentation](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/adc/struct.Adc.html#) there are several methods that are extremely useful. You will actually find methods that provide you a voltage (even for the reference voltage) immediately without worrying about what the resolution is. An idea would be to refactor the above code to leverage methods that retrieve voltages immediately. 
- The stm32 has an internal temperature sensor that measures the temperature of the device. Dig into the reference manual and see if you can configure the ADC to read the internal temperature of the device. Note that here GPIO would no longer be required as well.

## Conclusion
In this post, an analog temperature measurement application was created leveraging the ADC peripheral for the STM32F401RE microcontroller on the Nucleo-F401RE development board. The resulting measurement is also sent over to a host PC over a UART connection. All code was based on polling (without interrupts). Additionally, all code was created at the HAL level using the [stm32f4xx Rust HAL](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/index.html). Have any questions? Share your thoughts in the comments below 👇. 

%%[subend]