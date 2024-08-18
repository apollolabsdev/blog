---
title: "ESP32 Standard Library Embedded Rust: GPIO Interrupts"
datePublished: Thu Sep 07 2023 15:00:09 GMT+0000 (Coordinated Universal Time)
cuid: clm9aovhy000908l2gl9u402b
slug: esp32-standard-library-embedded-rust-gpio-interrupts
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1694066615692/b3242b63-93e3-4451-a84e-8d4977ac9eb5.png
tags: beginners, rust, embedded, esp32

---

> ***This blog post is the eighth of a multi-part series of posts where I explore various peripherals in the ESP32C3 using standard library embedded Rust and the esp-idf-hal. Please be aware that certain concepts in newer posts could depend on concepts in prior posts.***

Prior posts include (in order of publishing):

1. [**ESP32 Standard Library Embedded Rust: GPIO Control**](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-gpio-control)
    
2. [**ESP32 Standard Library Embedded Rust: UART Communication**](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-uart-communication)
    
3. [**ESP32 Standard Library Embedded Rust: I2C Communication**](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-i2c-communication)
    
4. [**ESP32 Standard Library Embedded Rust: Timers**](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-timers)
    
5. [**ESP32 Standard Library Embedded Rust: PWM Servo Motor Sweep**](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-pwm-servo-motor-sweep)
    
6. [**ESP32 Standard Library Embedded Rust: Analog Temperature Sensing using the ADC**](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-analog-temperature-sensing-using-the-adc)
    
7. [**ESP32 Standard Library Embedded Rust: SPI with the MAX7219 LED Dot Matrix**](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-spi-with-the-max7219-led-dot-matrix)
    

## Introduction

It's well established that interrupts are a tough concept to grasp for the embedded beginner. Add to that when doing it in Rust the complexity of dealing with mutable `static` variables. This is because working with shared variables and interrupts is inherently `unsafe` if proper measures are not taken. When looking at how to do interrupts using the `esp-idf-hal` I first resorted to the [Embedded Rust on Espressif book](https://esp-rs.github.io/std-training/). Interrupts are covered under the Advanced Workshop in [section 4.3](https://esp-rs.github.io/std-training/04_4_0_interrupts.html), and to be honest, I was taken aback a little at what could be an additional level of complexity for a beginner. Without too much detail, this is because the book resorts to using lower-level implementations. For those interested, by that, I mean FFI interfaces to FreeRTOS which I will be creating a separate post about later.

I figured there must be a more friendly interface at the `esp-idf-hal` level. However what got me a bit worried at first is that when I dug into the `esp-idf-hal` examples, I didn't find any with interrupts 😱. With some further digging into the `esp-idf-hal` documentation afterward, I found some interfaces that I figured could help. Although the methods didn't have any descriptions, the names were more or less self-explanatory.

The thing about such interfaces, you need to understand how interrupts work to be able to use them. The good news is that regardless of device/architecture interrupts follow more or less a similar process. As such, in this post, I will be using the `esp-idf-hal` interfaces to create an interrupt-based application. The application would detect button presses, count them, and print the count to the console.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/375095209744413697).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp align="center")
    
* Pushbutton
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689175647786/f97380f4-e55d-4517-a49e-16d0137f008f.jpeg?auto=compress,format&format=webp align="center")

#### **🔌 Connections**

**📝 Note**

> ***All connection details are also shown in the*** [***Wokwi example***](https://wokwi.com/projects/375095209744413697)***.***

Connections include the following:

* On one end, the button pin should be connected to gpio0 of the devkit. The gpio0 pin will be configured as input. On the same button end, the other pin of the switch will be connected to the devkit GND.
    

## **👨‍🎨 Software Design**

In the application software in this post, a count variable will be incremented every time the button is pressed. Upon an interrupt occurrence, the count variable will also be printed to the console. In this application, the button would be configured for interrupts.

Interrupts are software routines (or functions) that get triggered/called by hardware events. These software routines are usually referred to as interrupt service routines or ISRs. Generally, in a microcontroller, there are several hardware sources that can be configured to provide interrupts on an event. In our case, we want to configure a GPIO input pin to call an ISR based on a button press event. An input pin can be configured to detect either signal edge (positive or negative) caused by the event. Regardless of the peripheral, typically the steps to configure a hardware source to trigger an interrupt follow a similar pattern. For the input GPIO button, these are the steps:

1. Configure the button pin as input.
    
2. (Optional) Configure any internal pull-ups or pull-downs. This isn't needed if the pull-up/down resistor exists external to the pin.
    
3. Configure the interrupt type you want the pin to detect.
    
4. Attach/subscribe the interrupt service routine to the button interrupt. Here we are telling the controller which routine (ISR) to call on the occurrence of the event.
    
5. Enable the interrupt to start listening for events.
    

After that, one would need to define an Interrupt Service Routine (ISR) in the application code. As one would expect, the ISR contains the code executed in response to a specific interrupt event. Additionally, inside the ISR, it is typical that one would use values that are shared with the `main` routine. This is a bit of a challenge in Rust. In Rust, global mutable (`static`) variables, rightly so, are considered unsafe to read or write. This is because without taking special care, a race condition might emerge. To solve the challenges, in Rust, global mutable data needs to be wrapped in a safe abstraction that allow it to be shared between the ISR and the main thread. Enough explanations, lets move on to the actual code.

## **👨‍💻 Code Implementation**

### **📥 Crate Imports**

In this implementation the crates required are as follows:

* The `std::sync` to import synchronization primitives.
    
* The `esp_idf_hal` crate to import the needed device hardware abstractions.
    

```rust
use std::sync::atomic::AtomicBool;
use std::sync::atomic::Ordering;
use esp_idf_hal::gpio::*;
use esp_idf_hal::peripherals::Peripherals;
```

### **🎛 Initialization/Configuration Code**

#### **🌍 Global Variables**

In the application at hand, I'm choosing to enable interrupts for the GPIO peripheral to detect a button press. As such, I would need to create a global (`static`) shared variable to allow the ISR to indicate that an interrupt occurred. This `static` needs to be wrapped in a safe abstraction. Consequently, I create a `static` global variable called `FLAG` with type `AtomicBool`:

```rust
static FLAG: AtomicBool = AtomicBool::new(false);
```

The `AtomicBool` is a synchronization primitive from `std::sync` that makes sure that the peripheral can be safely shared among threads. It's initialized to `false` as no interrupts have been detected yet. On a side note,`std::sync` provides several synchronization primitives that one would use depending on the type that needs to be wrapped.

#### **GPIO Peripheral Configuration:**

1️⃣ **Obtain a handle for the device peripherals**: Similar to all past blog posts, in embedded Rust, as part of the singleton design pattern, we first have to take the device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` as follows:

```rust
let dp = Peripherals::take().unwrap();
```

**2️⃣ Obtain a Handle for the Button Pin and Configure it as Input:** Using the `input` instance method in `gpio::PinDriver`, we create a `button` configuring `gpio0` to an input:

```rust
let mut button = PinDriver::input(dp.pins.gpio0).unwrap();
```

**3️⃣ Configure Button Pin with an Internal Pull Up:** The `PinDriver` type contains a `set_pull` method that allows us to select the type of pull from the `Pull` enum. In this case we need an internal pull-up leading to the following line:

```rust
button.set_pull(Pull::Up).unwrap();
```

4️⃣ **Configure Button Pin Interrupt Type:** Now we need to Configure the button pin to detect interrupts. For that, we use the `set_interrupt_type` method from `PinDriver`. We have several options to choose from in the `InterruptType` enum. For this application's purpose, I am going to use positive edge detection. This means that when the pin sees a transition from low to high on the pin, it will fire an interrupt. Here's the configuration code:

```rust
button.set_interrupt_type(InterruptType::PosEdge).unwrap();
```

4️⃣ **Attach the ISR to the Interrupt:** Now that all is configured at a pin level, we still need to inform the controller where to go when the interrupt event happens. We are going to create a function called `gpio_int_callback` for that which we'll show the implementation for shortly. As such, to attach `gpio_int_callback` to the interrupt button we use the `subscribe` method from `PinDriver`. Note that `subscribe` is `unsafe` so it needs to be wrapped in an `unsafe` block. However, this doesn't mean that subscribing to an ISR is actually `unsafe` , it's only that the compiler cannot guarantee what's happening in the underlying code. Moving on, we `subscribe` to the ISR as follows:

```rust
unsafe { button.subscribe(gpio_int_callback).unwrap() }
```

5️⃣ **Enable the Interrupt:** After all the configuration has been done, believe it or not, if the button is pressed an event won't be detected. Why? because the button interrupt is not switched on (or enabled) yet. Is only configured. Sort of like buying a car with a ton of bells and whistles, Unless you turn it on, technically many of those features won't work. Again, from `PinDriver` there is the `enable_interrupt` method that would allow us to enable the button interrupt:

```rust
button.enable_interrupt().unwrap();
```

### **📱 Application Code**

**⏸️ The Interrupt Service Routine**

Recall earlier that we attached (subscribed to) an ISR called `gpio_int_callback`. As such, we still need to define what happens inside the ISR when an interrupt happens. The ISR is defined exactly in the same style as a function. Also, in `gpio_int_callback` all we need to do is set the `FLAG` variable to `true` marking the occurrence of the interrupt. Since `FLAG` is an `AtomicBool` we can use the `store` method for that:

```rust
fn gpio_int_callback() {
    // Assert FLAG indicating a press button happened
    FLAG.store(true, Ordering::Relaxed);
}
```

Note that in `AtomicBool` there are `store()` and `load()` methods that we are using. Both require an `Ordering` enum argument to be specified. The `Ordering` enum refers to the way atomic operations synchronize memory which I chose `Relaxed`. For more details, one can refer to the full list of options [**here**](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html). There is even a more detailed explanation of ordering [**here**](https://doc.rust-lang.org/nomicon/atomics.html).

**🔁 The Main Loop**

In the main loop, we are going to have to keep checking `FLAG` using the `load` method. If `FLAG` gets asserted, then we know that a press button happened. Since we are going to track how many times the press occurred, then we need some variable to do that which I created `count` for that. As such, if `FLAG` is asserted, we need reset it first to allow consecutive interrupts to happen, increment `count`, then finally print `count` to the console. Here's the full code:

```rust
// Set up a variable that keeps track of press button count
let mut count = 0_u32;

loop {
    // Check if global flag is asserted
    if FLAG.load(Ordering::Relaxed) {
        // Reset global flag
        FLAG.store(false, Ordering::Relaxed);
        // Update Press count and print
        count = count.wrapping_add(1);
        println!("Press Count {}", count);
    }
}
```

## **📱Full Application Code**

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/375095209744413697).

```rust
use std::sync::atomic::AtomicBool;
use std::sync::atomic::Ordering;

use esp_idf_hal::gpio::*;
use esp_idf_hal::peripherals::Peripherals;
use esp_idf_sys::{self as _};

static FLAG: AtomicBool = AtomicBool::new(false);

fn gpio_int_callback() {
    // Assert FLAG indicating a press button happened
    FLAG.store(true, Ordering::Relaxed);
}

fn main() -> ! {
    // It is necessary to call this function once. Otherwise some patches to the runtime
    // implemented by esp-idf-sys might not link properly. See https://github.com/esp-rs/esp-idf-template/issues/71
    esp_idf_sys::link_patches();

    // Take Peripherals
    let dp = Peripherals::take().unwrap();

    // Configure button pin as input
    let mut button = PinDriver::input(dp.pins.gpio0).unwrap();
    // Configure button pin with internal pull up
    button.set_pull(Pull::Up).unwrap();
    // Configure button pin to detect interrupts on a positive edge
    button.set_interrupt_type(InterruptType::PosEdge).unwrap();
    // Attach the ISR to the button interrupt
    unsafe { button.subscribe(gpio_int_callback).unwrap() }
    // Enable interrupts
    button.enable_interrupt().unwrap();

    // Set up a variable that keeps track of press button count
    let mut count = 0_u32;

    loop {
        // Check if global flag is asserted
        if FLAG.load(Ordering::Relaxed) {
            // Reset global flag
            FLAG.store(false, Ordering::Relaxed);
            // Update Press count and print
            count = count.wrapping_add(1);
            println!("Press Count {}", count);
        }
    }
}
```

## **Conclusion**

In this post, a GPIO interrupt application was created by configuring a GPIO pin to detect button press events. This was using the GPIO peripheral for the ESP32C3 and the `esp-idf-hal`. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]