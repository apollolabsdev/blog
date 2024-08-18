---
title: "ESP32 Standard Library Embedded Rust: GPIO Control"
datePublished: Thu Jul 13 2023 11:46:41 GMT+0000 (Coordinated Universal Time)
cuid: clk134dex00010amj6v5x1ftl
slug: esp32-standard-library-embedded-rust-gpio-control
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1689248448284/e79643d1-3ffa-49cf-ba4b-0e833f055fb9.png
tags: tutorial, rust, embedded, esp32, embedded-systems

---

## Introduction

Embedded programming can generally be classified into two areas; hosted and bare-metal. Bare-metal programming assumes a clean slate, meaning that the target hardware or environment includes no prior software. This is opposed to the convention one might be used to doing programming on conventional PCs. On the other hand, hosted programming provides an environment close to one on a conventional PC (Ex. an operating system a.k.a. OS).

**Why is there more than one area?** bare-metal programming is common in embedded given the resource constraints of a microcontroller. In certain applications, the overhead that comes along with a hosted environment is deemed unnecessary. On the other hand, more powerful devices are emerging in embedded. These device are higher in performance and possess much more internal hardware. As such, it becomes hard to avoid a hosted environment.

**So, what does this mean?** In terms of embedded Rust, the environment governs what libraries we have access to. For example, Rust by default loads the `std` libraries that provide many of the constructs like `String` and `Hashmap` which requires software provided by an OS. This would be ok in a hosted environment, but would not be in a bare-metal one. As such, we can prevent Rust from doing that by using the `#![no_std]` attribute.

For the ESP32, espressif provides support for developing code either in bare-metal (`no_std`) or using the standard library (`std`). Espressif has an existing development framework called `esp-idf` that supports all ESP32s. `esp-idf` in turn, provides a hosted environment that can provide `std` support. As a result, via the `std` library we have access to a lot of features that exist in `esp-idf`. I will be breaking this down further as I progress through the posts, however for more detail you can check the [Rust on ESP Book](https://esp-rs.github.io/book/overview/using-the-standard-library.html).

A prior [series of blog posts](https://apollolabsblog.hashnode.dev/series/esp32c3-embedded-rust-hal) I developed covers different aspects of bare-metal development for the ESP32. This post is a start of a new series using the `std` library and ESP32. As the posts progress some similarities with `no_std` will emerge in some areas. However, there are some significant differences in other areas. I will still be using [Wokwi](https://wokwi.com/) to eliminate hardware dependence.

In this post, I will be developing an application that uses a 10 LED bar graph to circulate an LED bar light at different speeds. The speed is altered by a button press.

%%[substart] 

**📝 Notes**

> * To get started with environments and hardware the [Rust on ESP Book](https://esp-rs.github.io/book/) provides a great starting point. However, if using Wokwi, much of that is not needed.
>     
> * Relative to the `esp-idf-hal` , as far as material goes, there exists [training material](https://esp-rs.github.io/std-training/) that is open sourced by Ferrous systems. The training material takes a bit of a different approach where it starts with high-level IoT exercises followed by low-level control. Additionally, the training is based on the awesome [Rust ESP board](https://github.com/esp-rs/esp-rust-board/tree/v1.2) hardware.
>     

### **📚** Knowledge Pre-requisites

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    

### **💾** Software Setup

All the code presented in this post is available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [here](https://wokwi.com/projects/369877335340108801).

### **🛠** Hardware Setup

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg align="center")
    
* 10 Segment LED Bar Graph
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689175567250/bbe8cf6f-a46c-4913-a61e-2e334af5e8eb.jpeg align="center")

* Pushbutton
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689175647786/f97380f4-e55d-4517-a49e-16d0137f008f.jpeg align="center")

#### **🔌** Connections

**📝 Note**

> All connection details are also shown in the [Wokwi example](https://wokwi.com/projects/369877335340108801).

Connections include the following:

* LED Bar Graph Anode A10 to gpio1 on the devkit.
    
* LED Bar Graph Anode A9 to gpio10 on the devkit.
    
* LED Bar Graph Anode A8 to gpio19 on the devkit.
    
* LED Bar Graph Anode A7 to gpio18 on the devkit.
    
* LED Bar Graph Anode A6 to gpio4 on the devkit.
    
* LED Bar Graph Anode A5 to gpio5 on the devkit.
    
* LED Bar Graph Anode A4 to gpio6 on the devkit.
    
* LED Bar Graph Anode A3 to gpio7 on the devkit.
    
* LED Bar Graph Anode A2 to gpio8 on the devkit.
    
* LED Bar Graph Anode A1 to gpio9 on the devkit.
    
* All LED Bar Graph Cathodes C1-C10 are connected to each other and the devkit GND.
    
* On one end, the button pin should be connected to gpio3 of the devkit. The gpio3 pin will be configured as input. On the same button end, the other pin of the switch will be connected to the devkit GND.
    

## **👨‍🎨** Software Design

In the application developed in this post, I want to cycle through turning on LEDs on an LED bar. A button will also be used to change how fast the light is cycling. Meaning, that every time I press the button, I want to see the LED cycling at a different speed. Obviously, the device pins would need to be configured first, which I would cover in the next section. In this section, I will focus on the design of the application algorithm.

The design will use a polling-based approach (rather than interrupts) to detect button press events. This is going to make things a bit tricky algorithmically (more explanation later). Interrupts would have made things really convenient. Though I will not be using interrupts because of two reasons; the first is that interrupts are generally a more advanced concept, and the second is that implementing interrupts using the `esp-idf` is more challenging compared to `no_std` Rust. As a result, I'd like to keep this post as simple as possible to grasp fundamental concepts first. In the future, I probably will create a separate post for an interrupt-based approach for the same application.

Moving on, let's try to represent our algorithm. Starting with the first LED in the sequence on the LED bar, here are the steps the algorithm would go through:

1. Turn on LED.
    
2. Check if the button has been pressed and determine the delay to be applied.
    
3. Delay by the amount determined in step 2.
    
4. Turn off the LED.
    
5. Delay for 100ms.
    
6. Repeat steps 1-5 for the next LED in sequence.
    
7. Once all LEDs are done, loop back to the first LED in sequence.
    

I'd like to point out a couple of things here. The first is step 2 and the second is step 5. In step 2, note how the button press is being checked with every LED turned on. Also, behind the button press is a procedure that would adjust the delay. This is necessary so that the button press is not missed in between all the delays. For example, if I were to have the button press checked at the end of the sequence, it's likely that several button press events will be missed. This is the issue with polling, ideally, interrupts (presented in a later post) would relieve us of this case.

For the delay adjusting procedure in step 2, a delay variable is maintained for the whole application. As such, if the button is pressed at any point, I need to decrease the delay value so that I increase the rate of LED cycling. However, I have to also check that the new delay value does not go negative. As such, if the delay value drops below a certain threshold I want to reset it to the original value I started with.

For step 5, note how I am introducing a 100 ms delay. This is to make sure that the current LED is off (visually) before turning on the next one in the sequence. You can experiment with this and see that if removed, you would notice an effect that the previous LED is still on when the current one turns on. You could probably live with a smaller delay as long as your eye does not notice it, I just used 100ms to stay on the safe side.

Let's now jump into implementing this algorithm.

## **👨‍💻** Code Implementation

### **📥** Crate Imports

In this implementation, three crates are required as follows:

* The `esp_idf_sys` crate that provides the `esp-idf` entry-point. This import exists as part of the basic template for the `std` development framework for the ESP32.
    
* The `esp_idf_hal` crate to import the device hardware abstractions.
    

```rust
use esp_idf_sys as _;

use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::gpio::*;
use esp_idf_hal::peripherals::Peripherals;
```

**📝 Note**

> Each of the crate imports needs to have a corresponding dependency in the Cargo.toml file. These are typically provided by the template.

### **🎛** Peripheral Configuration Code

Ahead of our application code, peripherals are configured through the following steps:

1️⃣ **Obtain a handle for the device peripherals**: In embedded Rust, as part of the singleton design pattern, we first have to `take` the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `dp` as follows:

```rust
let dp = Peripherals::take().unwrap();
```

**📝 Note**

> While I didn't mention it explicitily, in the `std` template for the `esp_idf` a `esp_idf_sys::link_patches();` line exists at the beginning of the `main` function. This line is necessary to make sure patches to the ESP-IDF are linked to the final executable.

**2️⃣ Obtain handles for the LEDs and configure them to output**: In the `no_std` template, configuring and obtaining handles for pins, happened typically in more than a single step. We first would obtain a handle to the `IO` struct and then configure and obtain a handle to each pin individually. In `esp_idf` via the `gpio` module we have access to the `PinDriver` struct that allows us, in a single step, to create configured instances to drive pins. To create an instance for an output pin, `PinDriver` has an instance method `output` that takes one argument, which would be one of the device's peripheral pins. The following shows how the 10 LEDs have been instantiated:

```rust
let mut led1 = PinDriver::output(dp.pins.gpio1).unwrap();
let mut led2 = PinDriver::output(dp.pins.gpio10).unwrap();
let mut led3 = PinDriver::output(dp.pins.gpio19).unwrap();
let mut led4 = PinDriver::output(dp.pins.gpio18).unwrap();
let mut led5 = PinDriver::output(dp.pins.gpio4).unwrap();
let mut led6 = PinDriver::output(dp.pins.gpio5).unwrap();
let mut led7 = PinDriver::output(dp.pins.gpio6).unwrap();
let mut led8 = PinDriver::output(dp.pins.gpio7).unwrap();
let mut led9 = PinDriver::output(dp.pins.gpio8).unwrap();
let mut led10 = PinDriver::output(dp.pins.gpio9).unwrap();
```

Note that I named the LEDs in a sequence in the order that they are connected, to make it easier to code.

3️⃣ **Obtain a handle and configure the input button**: The push button is connected to pin 3 (`gpio3`) as stated earlier. Configureing an input takes exactly the same form as the previous step for output, instead we use the `input` instance method as follows:

```rust
let mut button = PinDriver::input(dp.pins.gpio3).unwrap();
```

Additionally, in the pressed state, the button pulls to ground. This means that for the button's unpressed state, a pull-up resistor needs to be included internal to the ESP so the pin defaults to high when the button is unpressed. The button is configured as `Input` a `set_pull` method exists that allows us to select the `Pull` type from a `Pull` enum. In our application we need a pull-up that would be configured as follows:

```rust
button.set_pull(Pull::Up).unwrap();
```

### **📱** Application Code

Following the design described earlier, I first need to initialize a delay variable `blinkdelay`. `blinkdelay` needs to be mutable as it will be modified in the code.

```rust
// Initialize variable with starting delay
let mut blinkdelay = 200_u32;
```

Next inside the program loop, starting with `led1`, for each of the individual LED pins, the following code is executed:

```rust
// 1. Turn on LED
led1.set_high().unwrap();
// 2. Retrieve adjusted delay based on button press
blinkdelay = button_pressed(&button, &blinkdelay);
// 3. Delay with adjusted value
FreeRtos::delay_ms(blinkdelay);
// 4. Turn off LED
led1.set_low().unwrap();
// 5. Delay for 100ms (to make sure LED is turned off)
FreeRtos::delay_ms(100_u32);
```

The individual lines follow steps 1-5 highlighted earlier in this post in the software design section. There are two things to note here first is the use of a `FreeRtos` type for creating a delay and second is the `button_pressed` function.

Starting with `FreeRtos`, in order to create a delay, the `delay` module in the `esp-idf` provides us with several options highlighted in the [documentation](https://esp-rs.github.io/esp-idf-hal/esp_idf_hal/delay/index.html). Which you choose depends on the duration of the delay you want. For delays larger than 10ms, the doumcnetion states that we should use the `FreeRtos` type. The `FreeRtos` type has a `delay_ms` method that we only need to specify the amount of delay in ms and it will pause execution for that duration.

**📝 Note**

> The `FreeRtos` naming convention might come as a confusion here. FreeRTOS is an open source real-time operating system kernel. FreeRTOS also happens to be the underlying operating system used in the ESP-IDF framework. As such, this particular delay abstraction seems to me built on a FreeRtos abstraction provided by the `esp-idf` framework.

Next, regarding the `button_pressed function`, to avoid repetitive code, I created the `button_pressed` function to check for a button press and adjust the delay accordingly. The following is the implmentation of the `button_pressed` function:

```rust
fn button_pressed(but: &PinDriver<'_, Gpio3, Input>, del: &u32) -> u32 {
    // Check if Button has been pressed
    // If not pressed, return the delay value unchanged
    if but.is_low() {
        // if the value of the delay passed is less of equal to 50 then reset it to initial value
        // else subtract 50 from the passed delay
        println!("Button Pressed!");
        if del <= &50_u32 {
            return 200_u32;
        } else {
            return del - 50_u32;
        }
    } else {
        return *del;
    }
}
```

Note that the function has two parameters, `but` which is a reference to the `Gpio3` pin configured as an `Input`, and a reference to `del` which is the current delay that we will be passing. In the fucntion, if the button is pressed it is detected using the `is_low()` method. At which point I print using `println!` indicating the button press and decrease `del` by `50_u32`. If I end up with `del` value less than `50_u32` then I restor the original value I started with of `200_u32`. If a button press is not detected then `del` is returned unchanged.

**🚨 *Important Note***

> Keep in mind that the longer the delays are the less sensitive the appliciaton becomes to a button press. This means that you'd have to probably press the button for longer or try a few times for a press to be detected. For example, with the starting configuration we have in this applicaiton a 200ms delay is applied along with the 100ms buffer in between blinks. This means that we are checking for a button press at 300ms intervals. The loneger this duration is, the less frequent the system is checking for button presses. This is something to be mindful of if you care about how responsive your application should be.

## 🔧 An Optimized Approach

Looking at the code developed, it is somewhat verbose as the same code is repeated for each LED. One might figure that using an array with a for loop is the way to go, and it's true, however, there is somewhat of a challenge. Note in the way the pins are configured each pin has a different type. This is because the type system in Rust is typically used to encode the state of the pin into the type. This presents a challenge as arrays require that all elements are of the same type. Thanks to a tip from reader @[Christian Foucher](@Valda60) there exists a method `downgrade_output()` that can help with that. `downgrade_output()` converts or "downgrades" the pin type to a generic type. As such, when configuring the pins the all the pins can be placed in the same array as follows:

```rust
let mut leds = [
    PinDriver::output(dp.pins.gpio1.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio10.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio19.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio18.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio4.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio5.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio6.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio7.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio8.downgrade_output()).unwrap(),
    PinDriver::output(dp.pins.gpio9.downgrade_output()).unwrap(),
];
```

Additionally, the algorithm code would reduce down to the following:

```rust
loop {
    for mut led in &mut leds {
        led.set_high().unwrap();
        blinkdelay = button_pressed(&button, &blinkdelay);
        FreeRtos::delay_ms(blinkdelay);
        led.set_low().unwrap();
        FreeRtos::delay_ms(100_u32);
    }
}
```

## **📱** Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [apollolabs ESP32C3](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [here](https://wokwi.com/projects/369877335340108801).

```rust
use esp_idf_sys as _; // If using the `binstart` feature of `esp-idf-sys`, always keep this module imported

use esp_idf_hal::delay::FreeRtos;
use esp_idf_hal::gpio::*;
use esp_idf_hal::peripherals::Peripherals;

fn main() {
    // It is necessary to call this function once. Otherwise some patches to the runtime
    // implemented by esp-idf-sys might not link properly. See https://github.com/esp-rs/esp-idf-template/issues/71
    esp_idf_sys::link_patches();

    // Take Peripherals
    let dp = Peripherals::take().unwrap();

    // Configure all LED pins to digital outputs
    // let mut led1 = PinDriver::output(dp.pins.gpio1).unwrap();
    // let mut led2 = PinDriver::output(dp.pins.gpio10).unwrap();
    // let mut led3 = PinDriver::output(dp.pins.gpio19).unwrap();
    // let mut led4 = PinDriver::output(dp.pins.gpio18).unwrap();
    // let mut led5 = PinDriver::output(dp.pins.gpio4).unwrap();
    // let mut led6 = PinDriver::output(dp.pins.gpio5).unwrap();
    // let mut led7 = PinDriver::output(dp.pins.gpio6).unwrap();
    // let mut led8 = PinDriver::output(dp.pins.gpio7).unwrap();
    // let mut led9 = PinDriver::output(dp.pins.gpio8).unwrap();
    // let mut led10 = PinDriver::output(dp.pins.gpio9).unwrap();

    // Config Option 2 (Optimized Appraoch)
    let mut leds = [
        PinDriver::output(dp.pins.gpio1.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio10.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio19.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio18.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio4.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio5.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio6.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio7.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio8.downgrade_output()).unwrap(),
        PinDriver::output(dp.pins.gpio9.downgrade_output()).unwrap(),
    ];

    // Configure Button pin to input with Pull Up
    let mut button = PinDriver::input(dp.pins.gpio3).unwrap();
    button.set_pull(Pull::Up).unwrap();

    // Initialize variable with starting delay
    let mut blinkdelay = 200_u32;

    loop {
        // Algo:
        // Starting with first LED in sequence
        // 1. Turn on LED
        // 2. Retrieve adjusted delay based on button press
        // 3. Delay with adjusted value
        // 4. Turn off LED
        // 5. Delay for 100ms (to make sure LED is turned off)
        // 6. Repeat steps 1-5 for next LED in sequence
        // 7. Once all LEDs are done loop back to first LED in sequence

        // // LED 1
        // led1.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led1.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 2
        // led2.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led2.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 3
        // led3.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led3.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 4
        // led4.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led4.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 5
        // led5.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led5.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 6
        // led6.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led6.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 7
        // led7.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led7.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 8
        // led8.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led8.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 9
        // led9.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led9.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // // LED 10
        // led10.set_high().unwrap();
        // blinkdelay = button_pressed(&button, &blinkdelay);
        // FreeRtos::delay_ms(blinkdelay);
        // led10.set_low().unwrap();
        // FreeRtos::delay_ms(100_u32);

        // Option 2 (Optimized)
        for mut led in &mut leds {
            led.set_high().unwrap();
            blinkdelay = button_pressed(&button, &blinkdelay);
            FreeRtos::delay_ms(blinkdelay);
            led.set_low().unwrap();
            FreeRtos::delay_ms(100_u32);
        }
    }
}

fn button_pressed(but: &PinDriver<'_, Gpio3, Input>, del: &u32) -> u32 {
    // Check if Button has been pressed
    // If not pressed, return the delay value unchanged
    if but.is_low() {
        // if the value of the delay passed is less of equal to 50 then reset it to initial value
        // else subtract 50 from the passed delay
        println!("Button Pressed!");
        if del <= &50_u32 {
            return 200_u32;
        } else {
            return del - 50_u32;
        }
    } else {
        return *del;
    }
}
```

## Conclusion

In this post, an LED control application was created leveraging the GPIO peripheral for the ESP32C3 microcontroller. The code was created using an embedded `std` development environment supported by the `esp-idf-hal`. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]