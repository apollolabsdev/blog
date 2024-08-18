---
title: "Embassy on ESP: UART Transmitter"
datePublished: Sat Dec 09 2023 06:44:16 GMT+0000 (Coordinated Universal Time)
cuid: clpxoxe30000g08l78fmo2smp
slug: embassy-on-esp-uart-transmitter
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1702103963148/976f57a7-8af1-4617-82cc-d7544b3a808e.png
tags: tutorial, iot, rust, embedded, esp32

---

> ***This blog post is the third of a multi-part series of posts where I will explore various peripherals of the ESP32 using the embedded Rust embassy framework.***

Prior posts include (in order of publishing):

1. [Embassy on ESP: Getting Started](https://apollolabsblog.hashnode.dev/embassy-on-esp-getting-started)
    
2. [Embassy on ESP: GPIO](https://apollolabsblog.hashnode.dev/embassy-on-esp-gpio)
    

## Introduction

Setting up UART serial communication remains useful for several types of device-to-device (point-to-point) communication. One of the common examples is interacting with a host PC. UART implementations are also still found in some industrial and wireless applications. In this post, using embassy and the ESP32C3, I will be configuring and setting up UART communication with a host PC. UART will be configured as a transmitter where the application will detect and send button press counts over the communication channel.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Knowledge of how the embassy executor works.
    
* Basic knowledge of UART Communication.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/383561090802699265).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp align="center")
    

#### **🔌 Connections**

No special connections are required for this implementation. The UART lines on the ESP32-C3 in DevKitM are connected on-board to the UART bridge.

## **👨‍🎨 Software Design**

In the application developed in this post, button presses will be detected and counted. The button press count will be sent over UART to the host PC. A button press input will be configured for interrupts and increment a `count` each time the button is pressed. Consequently, every time a button press is detected the UART interface will transmit the `count`. The button press logic and the UART logic will each be implemented in their own tasks.

Figure 1 below expresses the states/logic of the button press task. Additionally, Figure 2 expresses the states/logic the UART task will alternate between. The button task will be implemented in the main task. Note that both tasks are being observed as independent each running their own logic. In reality, each will treated by the embassy executor as such. However, the UART task is dependent on a `count` variable being updated by the button press state machine. For that, we will need to use a special abstraction that allows safe exchange of data among tasks.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702053857193/00cd524f-dc05-499b-bbdf-24e85bf8853b.png align="center")

Let's now jump into implementing this algorithm.

## **👨‍💻 Code Implementation**

### 📥 Crate Imports

In this implementation the crates required are as follows:

* The `embassy_sync` crate to import `Signal`, a special abstraction needed for synchronization.
    
* The `embassy_executor` crate to import the embassy executor.
    
* The `embedded-hal-async` crate to import the GPIO abstractions to detect button presses.
    
* The `esp32c3-hal` crate to import the needed ESP32C3 abstractions.
    
* The `esp_backtrace` crate needed to define panic behavior.
    

```rust
use embassy_executor::Spawner;
use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;
use embassy_sync::signal::Signal;
use embedded_hal_async::digital::Wait;
use esp32c3_hal::{
    clock::ClockControl,
    embassy, interrupt,
    peripherals::{Interrupt, Peripherals, UART0},
    prelude::*,
    Uart, UartTx, IO,
};
use esp_backtrace as _;
```

#### 🌍 Global Variables

In the application at hand, there will be two tasks that share a count value. The button press detection task will increment a count the UART writer task will use it. As such, we can create a global variable `MYSIGNAL` to "signal" the count value that is going to be passed around. We declare `MYSIGNAL` as `static` and define it as follows:

```rust
static MYSIGNAL: Signal<CriticalSectionRawMutex, u32> = Signal::new();
```

> 📝 Note: Global variables shared among tasks in Rust is a very sticky topic. This is because sharing global data is `unsafe` since it can cause race conditions. You can read more about it [here](https://doc.rust-lang.org/beta/embedded-book/concurrency/index.html). Embassy offers several synchronization primitives that provide safe abstractions depending on what needs to be accomplished. There is a prior post where I go into detail about these primitives [here](https://apollolabsblog.hashnode.dev/sharing-data-among-tasks-in-rust-embassy-synchronization-primitives).

### 🕹️ The UART Writer Task

The UART writer task is expected to accept a UART instance as input and `loop` constantly check/`await` if `MYSIGNAL` gets updated. Though ahead of the `loop` it might be beneficial to indicate that the task has started. These are the required steps:

1️⃣ **Create a UART Writer Task**: Tasks are marked by the `#[embassy_executor::task]` macro followed by a `async` function implementation. The task created is referred to as `uart_writer` task defined as follows:

```rust
#[embassy_executor::task]
async fn uart_writer(mut tx: UartTx<'static, UART0>)
```

`UartTx` marks a UART transmitted type that is configured with an instance of `UART0`. This means when spawning the task, we need pass a handle for a UART transmitter configured with an instance of `UART0`. This will be done in the `main` task before spawning the `uart_writer` task.

2️⃣ **Print a Message**: As part of the `embedded_io_async` crate we can use the `write` implementation in the `Write` trait to asynchronously write a message. `write` has the following signature:

```rust
async fn write(&mut self, buf: &[u8]) -> Result<usize, Self::Error>
```

As such, we need to pass a mutable reference of a type (`UartTx` in our case) that implements `Write` and our message as a slice of `u8`. Since `write` is an `async` function, the program needs to `await` for the write operation to complete. This results in the following code:

```rust
embedded_io_async::Write::write(
    &mut tx,
    b"UART Task Spawned. Waiting for Button Press...\r\n",
)
.await
.unwrap();
```

> 📝 Note: I could have easily also used `esp_println` which provides more convenient abstractions to print to the console. While in this context it does not make much difference, I did this for two reasons. First, since this is a UART example, we need to demonstrate how to use UART abstractions. Second, the previous code is a good demonstration of the use of the `embedded-io-async` traits. Using the `embedded-io-async` traits allows for more portable code in some contexts.

3️⃣ **Define the task loop**: Next enter the task `loop`. The first thing we need to do is `await` a change on `MYSIGNAL`. For that, there exists a `wait` method for the `Signal` type. `wait_for_rising_edge` is an `async` function that resolves into a `Future` if its waiting on a signal. Once the `Future` resolves, we get back a value corresponding to the `press_count`. We then print the `press_count`:

```rust
loop {
    let press_count = MYSIGNAL.wait().await;
    esp_println::println!("Button Pressed {} time(s)", press_count);
}
```

### **📱 The Main Task**

The start of the main task is marked by the following code:

```rust
#[main]
async fn main(spawner: Spawner)
```

As the documentation states: "The main entry point of an Embassy application is defined using the `#[main]` macro. The entry point is also required to take a `Spawner` argument." As we'll see, `Spawner` is what will allow us to spawn or kick-off the `button_press` task.

The following steps will mark the tasks performed in the main task.

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

> ***📝 Note:*** *At the time of writing this post, I couldn't really locate the* `init` function [**docs.rs**](http://docs.rs) documentation. It didn't seem easily accessible through any of the current HAL implementation documentation. Nevertheless, I reached the signature of the function through the source [***here***](https://github.com/esp-rs/esp-hal/blob/ece40abaed0e642b751a8752ce6406740efa4af6/esp-hal-common/src/embassy/mod.rs#L100)*.*

3️⃣ **Instantiate and Create Handle for IO**: We need to configure the LED pins as a push-pull output and obtain a handler for the pin so that we can control it. Similarly, we need to obtain a handle for the button input pin. Before we can obtain any handles for the LEDs and the button we need to create an `IO` struct instance. The `IO` struct instance provides a HAL-designed struct that gives us access to all gpio pins thus enabling us to create handles for individual pins. This is similar to the concept of a `split` method used in other HALs (more detail [**here**](https://apollolabsblog.hashnode.dev/demystifying-rust-embedded-hal-split-and-constrain-methods)). We do this by calling the `new()` instance method on the `IO` struct as follows:

```rust
let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
```

Note how the `new` method requires passing the `GPIO` and `IO_MUX` peripherals.

4️⃣ **Obtain a handle and configure the input button**: The push button is connected to pin 2 (`gpio2`) as stated earlier. Additionally, in the pressed state, the button pulls to ground. Consequently, for the button unpressed state, a pull-up resistor needs to be included so the pin goes high. An internal pull-up can be configured for the pin using the `into_pull_up_input()` method as follows:

```rust
let mut button = io.pins.gpio2.into_pull_up_input();
```

5️⃣ **Obtain Handle and Configure UART**: to create an instance of `UART0`, there exists a `new` instance method under `esp32c3_hal::Uart` that requires two parameters; a `UART` peripheral type and a `Clocks` type. Since we need only an instance of a UART transmitted, there also exists a `split` method that allows us to "split" the `uart0` instance into separate transmitter and receiver instances. `split` returns a tuple with two instances. Since we don't need the receiver instance an underscore `_` is used.

```rust
let uart0 = Uart::new(peripherals.UART0, &clocks);
let (tx, _) = uart0.split();
```

`UART0` is chosen since it is the one that ties to the UART logging pins on board. Additionally, `new` configures UART with the default 8N1 setup.

6️⃣ **Enable GPIO Interrupts**: At this point, the `button` instance is just an input pin. In order to make the device respond to push button events, interrupts need to be enabled for `button`. This results in the following lines of code:

```rust
esp32c3_hal::interrupt::enable(
    esp32c3_hal::peripherals::Interrupt::GPIO,
    esp32c3_hal::interrupt::Priority::Priority1,
)
```

4️⃣ **Spawn UART Writer Task**: before entering the button press loop, we're going to need to kick off our `uart_writer` task. This is done using the `spawn` method as follows:

```rust
spawner.spawn(uart_writer(tx)).unwrap();
```

Next, we can move on to the task `loop`.

#### 🔁 Main Task Loop

Following the design described earlier, in the main task (button press task), we will `await` a button press to occur. Once a press has occurred, `press_count` is incremented and signaled. Here's the code:

```rust
let mut press_count = 0;
loop {
    // Await Button Press
    button.wait_for_rising_edge().await.unwrap();
    // Increment Count
    press_count += 1;
    // Signal Press Count
    MYSIGNAL.signal(press_count);
}
```

Note here the use of the `signal` method on `MYSIGNAL` to update its value and "signal" it to the `uart_writer` task.

This concludes the code for the full application.

## 📱 Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/383561090802699265).

```rust
#![no_std]
#![no_main]
#![feature(type_alias_impl_trait)]

use embassy_executor::Spawner;
use embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex;
use embassy_sync::signal::Signal;
use embedded_hal_async::digital::Wait;
use esp32c3_hal::{
    clock::ClockControl,
    embassy, interrupt,
    peripherals::{Interrupt, Peripherals, UART0},
    prelude::*,
    Uart, UartTx, IO,
};
use esp_backtrace as _;

static MYSIGNAL: Signal<CriticalSectionRawMutex, u32> = Signal::new();

#[embassy_executor::task]
async fn uart_writer(mut tx: UartTx<'static, UART0>) {
    embedded_io_async::Write::write(
        &mut tx,
        b"UART Task Spawned. Waiting for Button Press...\r\n",
    )
    .await
    .unwrap();
    loop {
        let press_count = MYSIGNAL.wait().await;
        esp_println::println!("Button Pressed {} time(s)", press_count);
    }
}

#[main]
async fn main(spawner: Spawner) {
    let peripherals = Peripherals::take();
    let system = peripherals.SYSTEM.split();
    let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

    // Initilize Embassy Timers
    embassy::init(
        &clocks,
        esp32c3_hal::timer::TimerGroup::new(peripherals.TIMG0, &clocks).timer0,
    );

    // Configure UART
    let uart0 = Uart::new(peripherals.UART0, &clocks);
    let (tx, _) = uart0.split();

    // Configure GPIO
    let io = IO::new(peripherals.GPIO, peripherals.IO_MUX);
    let mut button = io.pins.gpio2.into_pull_up_input();

    // Enable Interrupts for GPIO
    interrupt::enable(Interrupt::GPIO, interrupt::Priority::Priority1).unwrap();

    spawner.spawn(uart_writer(tx)).ok();

    let mut press_count = 0;
    loop {
        // Detect and Count Button Presses
        button.wait_for_rising_edge().await.unwrap();
        press_count += 1;
        // Signal Press Count
        MYSIGNAL.signal(press_count);
    }
}
```

## Conclusion

In this post, a UART application transmitting to a host was created for the ESP32C3 microcontroller. The code combined the use of Signals, GPIO, and interrupts with the embassy async framework. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]