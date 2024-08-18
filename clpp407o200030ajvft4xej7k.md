---
title: "Embassy on ESP: GPIO"
datePublished: Sun Dec 03 2023 06:36:27 GMT+0000 (Coordinated Universal Time)
cuid: clpp407o200030ajvft4xej7k
slug: embassy-on-esp-gpio
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1701585332175/163136e6-29dc-4591-830f-26d73a39cc09.png
tags: iot, rust, embedded, internet-of-things, esp32

---

> ***This blog post is the second of a multi-part series of posts where I will explore various peripherals of the ESP32 using the embedded Rust embassy framework.***

Prior posts include (in order of publishing):

1. [Embassy on ESP: Getting Started](https://apollolabsblog.hashnode.dev/embassy-on-esp-getting-started)
    

## Introduction

In the first post from last week, a basic application was built in embassy on the ESP32C3. This was mainly to introduce basic operations and also a template to build on. One of the things that is amazing about embassy and async is how much easier it is to implement interrupt-driven code. There are two prior blog posts that you can compare to for ESP. Both for [std](https://apollolabsblog.hashnode.dev/esp32-standard-library-embedded-rust-gpio-interrupts) and [no-std](https://apollolabsblog.hashnode.dev/esp32-embedded-rust-at-the-hal-gpio-interrupts) implementation of interrupts. I recommend revisiting those prior posts to draw comparisons.

In this post, we'll get to start experimenting with GPIO interrupts in embassy. We'll see how we can configure GPIO, read inputs, and manipulate output. We'll be developing an application that uses a 10 LED bar graph to circulate a light at different speeds. The speed is altered by a button press.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Knowledge of how the embassy executor works
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/382810046853433345).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp align="center")
    
* 10 Segment LED Bar Graph
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689175567250/bbe8cf6f-a46c-4913-a61e-2e334af5e8eb.jpeg?auto=compress,format&format=webp align="center")

* Pushbutton
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1689175647786/f97380f4-e55d-4517-a49e-16d0137f008f.jpeg?auto=compress,format&format=webp align="center")

#### **🔌 Connections**

**📝 Note**

> ***All connection details are also shown in the*** [***Wokwi example***](https://wokwi.com/projects/382810046853433345)***.***

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
    

## **👨‍🎨 Software Design**

In the application developed in this post, I want to cycle through turning on LEDs on an LED bar. A button will also be used to change how fast the light is cycling. Meaning, that every time I press the button, I want to see the LED cycling at a different speed. Obviously, the device pins would need to be configured first, which I will cover in the next section. In this section, I will focus on the design of the application algorithm.

The design will use interrupts to detect button press events. The button press will modify a delay variable shared with the main task. We can consider that the application has two tasks, an LED cycling task and a button-press task. Both applications share a `del` variable that is adjusted according to button presses.

In the LED cycling (`main`) task, starting with the first LED in the sequence on the LED bar, here are the steps the algorithm would go through:

1. Turn on LED.
    
2. `await` for a delay of `del` to expire.
    
3. Turn off the LED.
    
4. `await` for a delay of 100ms to expire.
    
5. Repeat steps 1-5 for the next LED in sequence.
    
6. Once all LEDs are done, loop back to the first LED in sequence.
    

In the button-pressed task, here are the steps the algorithm would go through:

1. `await` for a button press to occur (ex. rising edge to occur).
    
2. Adjust `del`
    
3. Go back to step 1.
    

Note that every time we have an `await` the task is yielding to the executor to determine what to do next.

For the delay adjusting procedure in the button-pressed task, the delay value is changed so that the rate of LED cycling decreases. However, we need to make sure that the new delay value does not go negative. As such, if the `del` drops below a certain threshold we'd want to reset it to the original value we started with.

For step 4 in the `main` task, note the 100 ms delay. This is to make sure that the current LED is off (visually) before turning on the next one in the sequence. You can experiment with this and see that if removed, you would notice an effect that the previous LED is still on when the current one turns on. You could probably live with a smaller delay as long as your eye does not notice it, I just used 100ms to stay on the safe side.

Let's now jump into implementing this algorithm.

## **👨‍💻 Code Implementation**

> ***📝 Although I followed the documentation to set up my first project, it wasn't all smooth sailing. It was mainly had to do with configurations/settings of configuration (toml) files. I figure this has to do with embassy still being considered to be experimental. Set up I managed to get working is available up in my*** [***git repo***](https://github.com/apollolabsdev/stm32-nucleo-f401re)***.***

### 📥 Crate Imports

In this implementation the crates required are as follows:

* The `core::sync::atomic` and `portable_atomic` crates to import `Atomic` and `Ordering` that will be needed for syncronization.
    
* The `embassy_executor` crate to import the embassy executor.
    
* The `embassy_time` crate to import `Timer` abstractions for delays.
    
* The `embedded-hal-async` crate to import the GPIO abstractions to detect button presses.
    
* The `esp32c3-hal` crate to import the needed ESP32C3 abstractions.
    
* The `esp_backtrace` crate needed to define panic behavior.
    

```rust
use core::sync::atomic::Ordering;
use portable_atomic::AtomicU32;
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embedded_hal_async::digital::Wait;
use esp32c3_hal::{clock::ClockControl, embassy, peripherals::Peripherals, prelude::*, IO};
use esp32c3_hal::gpio::{AnyPin, Input, PullUp};
use esp_backtrace as _;
```

#### 🌍 Global Variables

In the application at hand, there will be two tasks that share a delay value. The button press detection task will adjust the delay and the LED control task will use it. As such, we can create a global variable `BLINK_DELAY` to carry the delay value that is going to be passed around. Here the `AtomicU32` type is used which is an integer type that can be safely shared between threads. The `AtomicU32` type has the same in-memory representation as the underlying integer type, `u32` but is considered safe to share between threads.

```rust
static BLINK_DELAY: AtomicU32 = AtomicU32::new(200_u32);
```

> 📝 Note: Global variables shared among tasks in Rust is a very sticky topic. This is because sharing global data is `unsafe` since it can cause race conditions. You can read more about it [here](https://doc.rust-lang.org/beta/embedded-book/concurrency/index.html). Embassy offers several synchronization primitives that provide safe abstractions depending on what needs to be accomplished. There is a prior post about these primitives [here](https://apollolabsblog.hashnode.dev/sharing-data-among-tasks-in-rust-embassy-synchronization-primitives).

### 🕹️ The Button Press Task

The button press task is expected to accept a GPIO pin as input and loop forever checking if the button is pressed. These are the required steps:

1️⃣ **Create a blinking task and handle for the button**: Tasks are marked by the `#[embassy_executor::task]` macro followed by a `async` function implementation. The task created is referred to as `press_button` task defined as follows:

```rust
#[embassy_executor::task]
async fn press_button(mut button: AnyPin<Input<PullUp>>)
```

`AnyPin` marks a generic pin type that is configured as a `PullUp` `Input`. This means we need to obtain a handle for the button pin and configure it to an input with a pull-up. This will be done in the `main` task before spawning the `button_press` task

2️⃣ **Define the task loop**: Next enter the task `loop`. The first thing we need to do is `await` a button press. For that, there exists a `wait_for_rising_edge` method implementation for the `Wait` trait in the [`embedded-hal-async`](https://docs.rs/embedded-hal-async/1.0.0-rc.2/embedded_hal_async/digital/trait.Wait.html). `wait_for_rising_edge` is an `async` function that resolves into a `Future` if its waiting on a condition. Otherwise, we get a `Result` . We call `wait_for_rising_edge` on `button` as follows:

```rust
button.wait_for_rising_edge().await.unwrap();
```

Using embassy, `async` events from interrupts or otherwise are managed through `Futures`. This means that execution can be yielded using `await` to allow other code to progress until the event attached to a `Future` occurs. The executor manages all of this in the background and more detail about it is provided [**in the embassy documentation**](https://embassy.dev/dev/runtime.html).

3️⃣ **Retrieve the delay**: Next we `load` the delay value from the global context as follows:

```rust
let del = BLINK_DELAY.load(Ordering::Relaxed);
```

`load` is a method part of the `AtomicU32` synchronization abstraction.

**4️⃣ Adjust the Delay:** This is the final step in the task and involves adjusting the delay according to our desired logic. Here we are doing decrements of 50 ms and if we reach a value less than 50, we reset back to the starting value of 200.

```rust
if del <= 50_u32 {
  BLINK_DELAY.store(200_u32,Ordering::Relaxed);
  esp_println:: println!("Delay is now 200ms");
} else {
  BLINK_DELAY.store(del - 50_u32,Ordering::Relaxed);
  esp_println:: println!("Delay is now {}ms", del - 50_u32);
}
```

### 📱 The Main Task (LED Cycling Task)

The start of the main task is marked by the following code:

```rust
#[embassy_executor::main]
async fn main(spawner: Spawner)
```

As the documentation states: "The main entry point of an Embassy application is defined using the `#[embassy_executor::main]` macro. The entry point is also required to take a `Spawner` argument." As we've seen in last week's post, `Spawner` is what will allow us to spawn or kick-off `button_task`.

As indicated before, the main task will also be where we manage the LED cycling logic. The following steps will mark the tasks performed in the main task.

1️⃣ **Obtain a handle for the device peripherals & system clocks**: In embedded Rust, as part of the singleton design pattern, we first have to take the PAC-level device peripherals. This is done using the `take()` method. Here I create a device peripheral handler named `peripherals` , a system peripheral handler `system`, and a system clock handler `clocks` as follows:

```rust
let peripherals = Peripherals::take();
let system = peripherals.SYSTEM.split();
let clocks = ClockControl::boot_defaults(system.clock_control).freeze();
```

**2️⃣ Initialize Embassy Timers for the ESP32C3:**

In embassy, there exists an `init` function that takes two parameters. The first is system clocks and the second is an instance of a timer. Under the hood, what this function does is initialize the embassy timers. As such, we can initialize the embassy timers as follows:

```rust
embassy::init(
    &clocks,
    esp32c3_hal::timer::TimerGroup::new(peripherals.TIMG0, &clocks).timer0,
);
```

> ***📝 Note:*** *At the time of writing this post, I couldn't really locate the* `init` function [**docs.rs**](http://docs.rs/) documentation. It didn't seem easily accessible through any of the current HAL implementation documentation. Nevertheless, I reached the signature of the function through the source [***here***](https://github.com/esp-rs/esp-hal/blob/ece40abaed0e642b751a8752ce6406740efa4af6/esp-hal-common/src/embassy/mod.rs#L100)*.*

3️⃣ **Instantiate and Create Handle for IO**: We need to configure the LED pins as a push-pull output and obtain a handler for the pin so that we can control it. Similarly, we need to obtain a handle for the button input pin. Before we can obtain any handles for the LEDs and the button we need to create an `IO` struct instance. The `IO` struct instance provides a HAL-designed struct that gives us access to all gpio pins thus enabling us to create handles for individual pins. This is similar to the concept of a `split` method used in other HALs (more detail [**here**](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)). We do this by calling the `new()` instance method on the `IO` struct as follows:

```rust
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
```

Note how the `new` method requires passing the `GPIO` and `IO_MUX` peripherals.

4️⃣ **Obtain a handle and configure the input button**: The push button is connected to pin 2 (`gpio2`) as stated earlier. Additionally, in the pressed state, the button pulls to ground. Consequently, for the button unpressed state, a pull-up resistor needs to be included so the pin goes high. An internal pull-up can be configured for the pin using the `into_pull_up_input()` method as follows:

```rust
let del_but = io.pins.gpio2.into_pull_up_input().degrade();
```

Note that as opposed to the LED outputs, the `button` handle here does not need to be mutable since we will only be reading it. Additionally, here we are using the `degrade` method which "degrades" the pin type into a generic `AnyPin` type that is required to pass to the `button_press` task.

5️⃣ **Obtain handles for the LEDs and configure them to output**: We have 10 LEDs that we need to activate individually. One approach is to configure each separately and activate it separately. However, in order to make things efficient, we can combine the LED pin handles all in one array. This will allow us to iterate over the individual pins using a `for` loop. However, there is a challenge here. Each pin will have a different type and arrays require that all elements are of a similar type. This is another good example for usage of `degrade`. Using `degrade` we can make all the pins of the same `AnyPin` type. This is how it looks like:

```rust
let mut leds = [
    io.pins.gpio1.into_push_pull_output().degrade(),
    io.pins.gpio10.into_push_pull_output().degrade(),
    io.pins.gpio19.into_push_pull_output().degrade(),
    io.pins.gpio18.into_push_pull_output().degrade(),
    io.pins.gpio4.into_push_pull_output().degrade(),
    io.pins.gpio5.into_push_pull_output().degrade(),
    io.pins.gpio6.into_push_pull_output().degrade(),
    io.pins.gpio7.into_push_pull_output().degrade(),
    io.pins.gpio8.into_push_pull_output().degrade(),
    io.pins.gpio9.into_push_pull_output().degrade(),
];
```

3️⃣ **Enable GPIO Interrupts**: At this point, the `button` instance is just an input pin. In order to make the device respond to push button events, interrupts need to be enabled for `button`. In the `interrupt` module in the [esp32c3-hal](https://docs.rs/esp32c3-hal/latest/esp32c3_hal/interrupt/fn.enable.html), there exists an `enable` method that allows the enabling of different interrupts. The `enable` method has the following signature:

```rust
pub fn enable(interrupt: Interrupt, level: Priority) -> Result<(), Error>
```

`interrupt` expects an `Interrupt` type which is an enumeration of all interrupts for the esp32c3. Also level expects a `priority` which is also an enumeration of all priorites. This results in the following line of code:

```rust
esp32c3_hal::interrupt::enable(
    esp32c3_hal::peripherals::Interrupt::GPIO,
    esp32c3_hal::interrupt::Priority::Priority1,
)
```

I chose a priority of one since there are no other interrupts. this would only make a difference when you have several interrupts that might compete for processor time.

4️⃣ **Spawn Button Press Task**: before entering the LED cycling loop, we're going to need to kick off our `button_press` task. Then `button_press` task can be kicked off using the `spawn` method as follows:

```rust
spawner.spawn(press_button(del_but)).unwrap();
```

Next, we can move on to the application Loop.

#### 🔁 Main Task Loop

Following the design described earlier, in the main task, we will cycle through turning on and off LEDs. Along the way we should `await` a `BLINK_DELAY` amount of time before going to the next LED. Since we packaged all LEDs in an array this is done in a `for` loop as follows:

```rust
loop {
    for led in &mut leds {
        led.set_high().unwrap();
        Timer::after(Duration::from_millis(BLINK_DELAY.load(Ordering::Relaxed) as u64)).await;
        led.set_low().unwrap();
        Timer::after(Duration::from_millis(100)).await;
    }
}
```

Note the usage of `Timer` that comes from the [`embassy_time` crate](https://docs.rs/embassy-time/0.1.0/embassy_time/struct.Timer.html). `after` is a `Timer` instance method that accepts a `Duration` and returns a `Future`. As such, `await` allows us to yield execution to the executor such that the task can be polled later to check if the delay expired. Note that we are delaying `BLINK_DELAY` amount of delay after setting the LED to high. This is the shared value that is adjusted by the `button_press` task.

This concludes the code for the full application.

## 📱 Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/382346257842181121).

```rust
#![no_std]
#![no_main]
#![feature(type_alias_impl_trait)]

use core::sync::atomic::Ordering;
use portable_atomic::AtomicU32;
use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embedded_hal_async::digital::Wait;
use esp32c3_hal::{clock::ClockControl, embassy, peripherals::Peripherals, prelude::*, IO};
use esp32c3_hal::gpio::{AnyPin, Input, PullUp};
use esp_backtrace as _;

// Global Variable to Control LED Rotation Speed
static BLINK_DELAY: AtomicU32 = AtomicU32::new(200_u32);

#[main]
async fn main(spawner: Spawner) {
    // Take Peripherals
    let peripherals = Peripherals::take();
    let system = peripherals.SYSTEM.split();
    let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

    // Initilize Embassy Timers
    embassy::init(
        &clocks,
        esp32c3_hal::timer::TimerGroup::new(peripherals.TIMG0, &clocks).timer0,
    );

    // Acquire Handle to IO
    let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
    // Configure Delay Button to Pull Up input
    let del_but = io.pins.gpio2.into_pull_up_input().degrade();
    // Configure LED Array Pins to Output & Store in Array
    let mut leds = [
        io.pins.gpio1.into_push_pull_output().degrade(),
        io.pins.gpio10.into_push_pull_output().degrade(),
        io.pins.gpio19.into_push_pull_output().degrade(),
        io.pins.gpio18.into_push_pull_output().degrade(),
        io.pins.gpio4.into_push_pull_output().degrade(),
        io.pins.gpio5.into_push_pull_output().degrade(),
        io.pins.gpio6.into_push_pull_output().degrade(),
        io.pins.gpio7.into_push_pull_output().degrade(),
        io.pins.gpio8.into_push_pull_output().degrade(),
        io.pins.gpio9.into_push_pull_output().degrade(),
    ];
    // Enable GPIO Interrupts
    esp32c3_hal::interrupt::enable(
        esp32c3_hal::peripherals::Interrupt::GPIO,
        esp32c3_hal::interrupt::Priority::Priority1,
    )
    .unwrap();
    // Spawn Button Press Task
    spawner.spawn(press_button(del_but)).unwrap();

    // This line is for Wokwi only so that the console output is formatted correctly
    esp_println::print!("\x1b[20h");

    // Enter Application Loop Blinking on LED at a Time
    loop {
        for led in &mut leds {
            led.set_high().unwrap();
            Timer::after(Duration::from_millis(BLINK_DELAY.load(Ordering::Relaxed) as u64)).await;
            led.set_low().unwrap();
            Timer::after(Duration::from_millis(100)).await;
        }
    }
}


#[embassy_executor::task]
async fn press_button(mut button: AnyPin<Input<PullUp>>) {
    loop {
      // Wait for Button Press
      button.wait_for_rising_edge().await.unwrap();
      esp_println:: println!("Button Pressed!");
      // Retrieve Delay Global Variable
      let del = BLINK_DELAY.load(Ordering::Relaxed);
      // Adjust Delay Accordingly
      if del <= 50_u32 {
        BLINK_DELAY.store(200_u32,Ordering::Relaxed);
        esp_println:: println!("Delay is now 200ms");
      } else {
        BLINK_DELAY.store(del - 50_u32,Ordering::Relaxed);
        esp_println:: println!("Delay is now {}ms", del - 50_u32);
      } 
    }
}
```

## Conclusion

In this post, an LED control application was created leveraging the GPIO peripheral for the ESP32C3 microcontroller. The code was created using interrupts and the embassy async framework. It shows how embassy really simplifies the development of interrupt-based code. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]