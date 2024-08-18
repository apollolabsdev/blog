---
title: "Embassy on ESP: UART Echo"
datePublished: Fri Jan 12 2024 17:38:28 GMT+0000 (Coordinated Universal Time)
cuid: clrax9nku000309lb9wpee6wm
slug: embassy-on-esp-uart-echo
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1705080332954/920df4d5-54d3-4d80-9c73-563fb9c055e0.png
tags: tutorial, iot, rust, embedded, esp32

---

> ***This blog post is the fourth of a multi-part series of posts where I will explore various peripherals of the ESP32 using the embedded Rust embassy framework.***

Prior posts include (in order of publishing):

1. [Embassy on ESP: Getting Started](https://apollolabsblog.hashnode.dev/embassy-on-esp-getting-started)
    
2. [Embassy on ESP: GPIO](https://apollolabsblog.hashnode.dev/embassy-on-esp-gpio)
    
3. [Embassy on ESP: UART Transmitter](https://apollolabsblog.hashnode.dev/embassy-on-esp-uart-transmitter)
    

## Introduction

It is probably more common that UART is used in simplex one-direction communication Ex. to log device messages. As such, there might be less mention of UART receiver functionality. With receive functionality UART can be set up to receive and process incoming messages. For embedded applications, it can be useful if one wants to enable user input from a serial terminal. In this post, using embassy and the ESP32C3, I will be expanding on the last [UART Transmitter](https://apollolabsblog.hashnode.dev/embassy-on-esp-uart-transmitter) post to include receive functionality as well. In the example in this post, I will create a UART echo application. The application will receive user input from the terminal and "echo" it back to the terminal.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Knowledge of how the embassy executor works.
    
* Basic knowledge of UART Communication.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different then it means that it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/386561280544436225).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp align="center")
    

#### **🔌 Connections**

No special connections are required for this implementation. The UART lines on the ESP32-C3 in DevKitM are connected on-board to the UART bridge.

> 🗒️ **Note:** The ESP uses the UART0 peripheral for `esp_println` . Be mindful of that if you want to modify the code in this example.

## **👨‍🎨 Software Design**

In the application developed in this post, the ESP will wait for user input from the serial monitor and echo it back to the monitor. As such, the ESP will wait for incoming messages on the UART receiver end. Once a new message is entered, the ESP will keep buffering input characters until a valid end of transmission (EOT) character is received. After that the ESP will send the same buffered characters back to the monitor using its transmitter.

For this functionality there will be two tasks, a `uart_reader` (receive) task and a `uart_writer` (transmit) task. In between the tasks there will be a shared character (`u8`) buffer that I'll call `DATAPIPE`.

On a read event (incoming characters from monitor/terminal), the `uart_reader` task executes in taking the following steps:

1. `read` characters from UART receive buffer until a valid EOT character.
    
2. On the successful completion of a `read`, share the received characters with the `uart_writer` task by writing them to `DATAPIPE`.
    

The `uart_writer` task will kick in once new data appears in `DATAPIPE` taking the following steps:

1. `write` the `DATAPIPE` contents to the UART `write` buffer.
    
2. Transmit a new line (carriage return `0x0D` followed by line feed `0x0A` characters)
    
3. Flush the transmit buffer.
    

Let's now jump into the implementation.

## **👨‍💻 Code Implementation**

### 📥 Crate Imports

In this implementation the crates required are as follows:

* The `embassy_sync` crate to import `Pipe`, a special abstraction needed for synchronization.
    
* The `embassy_executor` crate to import the embassy executor.
    
* The `esp32c3-hal` crate to import the necessary ESP32C3 abstractions.
    
* The `esp_backtrace` crate needed to define panic behavior.
    

```rust
use embassy_executor::Spawner;
use embassy_sync::{blocking_mutex::raw::CriticalSectionRawMutex, pipe::Pipe};
use esp32c3_hal::{
    clock::ClockControl,
    embassy,
    peripherals::{Peripherals, UART0},
    prelude::*,
    uart::{config::AtCmdConfig, UartRx, UartTx},
    Uart,
};
use esp_backtrace as _;
```

#### 🌍 Global Variables & Constants

In the application at hand, there will be two tasks that share a character buffer. The `uart_reader` task will write to the shared buffer and the `uart_writer` task will read from this buffer. As such, we can create a global buffer `DATAPIPE` to "pipe" the characters are going to be passed around. We declare `DATAPIPE` as `static` and define it as follows:

```rust
static DATAPIPE: Pipe<CriticalSectionRawMutex, READ_BUF_SIZE> = Pipe::new();
```

> 📝 **Note:** Global variables shared among tasks in Rust could be a sticky topic. This is because sharing global data is `unsafe` since it can cause race conditions. You can read more about it [here](https://doc.rust-lang.org/beta/embedded-book/concurrency/index.html). Embassy offers several synchronization primitives that provide safe abstractions depending on what needs to be accomplished. There is a prior post where I go into detail about these primitives [here](https://apollolabsblog.hashnode.dev/sharing-data-among-tasks-in-rust-embassy-synchronization-primitives).

In addition to the above, it would be convenient to define the following constants:

```rust
// Read Buffer Size
const READ_BUF_SIZE: usize = 64;
// End of Transmission Character (Carrige Return -> 13 or 0x0D in ASCII)
const AT_CMD: u8 = 0x0D;
```

`READ_BUF_SIZE` will be used to define the sizes of the buffers we declare. Additionally `AT_CMD` defines the character that we will use for detecting an EOT sequence.

### 🤓 The UART Reader Task

The UART reader task is expected to accept a UART instance as input and `loop` constantly check/`await` if `MYSIGNAL` gets updated. Though ahead of the `loop` it might be beneficial to indicate that the task has started. These are the required steps:

1️⃣ **Create a UART Reader Task**: Tasks are marked by the `#[embassy_executor::task]` macro followed by a `async` function implementation. The task created is referred to as `uart_reader` task defined as follows:

```rust
#[embassy_executor::task]
async fn uart_reader(mut tx: UartRx<'static, UART0>)
```

`UartRx` marks a UART receiver type that is configured with an instance of `UART0`. This means when spawning the task, we need pass a handle for a UART reciever configured with an instance of `UART0`. This will be done in the `main` task before spawning the `uart_reader` task.

**2️⃣ Declare a Read Buffer:** Before entering the task loop, we need to declare read buffer `rbuf` to store characters that will be received. `rbuf` will later store characters written to `DATAPIPE` .

```rust
let mut rbuf: [u8; READ_BUF_SIZE] = [0u8; READ_BUF_SIZE];
```

#### 🔁 Task Loop

**1️⃣ Read Characters from Monitor/Terminal into Buffer:**

As part of the `embedded_io_async` crate we can use the `read` implementation of the `Read` trait to asynchronously read a message. `read` has the following signature:

```rust
async fn read(&mut self, buf: &mut [u8]) -> Result<usize, Self::Error>
```

As such, we need to pass a mutable reference of a type (`UartRx` in our case) that implements `Read` and our message as a slice of `u8`. Since `read` is an `async` function, the program needs to `await` for the read operation to complete. Using `read` we can receive characters into our buffer `rbuf` as follows:

```rust
 let r = embedded_io_async::Read::read(&mut rx, &mut rbuf[0..]).await;
```

`read` will keep on reading characters into `rbuf` until reaching a valid EOT character. The EOT character will be defined later in out `main` when we configure the `UART0` peripheral. `read` also returns a `Result` that contains the amount of characters that were read.

**2️⃣ Write Buffered Characters to Pipe:** The `Pipe` we need to write to is `DATAPIPE`. For that, we need to write all of the characters in `rbuf` to `DATAPIPE` There exists a `write_all` method for the `Pipe` type that has the following signature:

```rust
pub async fn write_all(&self, buf: &[u8])
```

`write_all` will write all bytes from `buf` into the `Pipe`. This results in the following code for us:

```rust
match r {
    Ok(len) => {
        // If read succeeds then write recieved characters to pipe
        DATAPIPE.write_all(&rbuf[..len]).await;
    }
    Err(e) => esp_println::println!("RX Error: {:?}", e),
}
```

The `match` is used to process the `Result` returned from the `read` method. If a receive error occurs, then `esp_println` is used to log the error.

### 🕹️ The UART Writer Task

The UART writer task is expected to accept a UART instance as input and `loop` constantly check/`await` if `DATAPIPE` has new data. These are the required steps:

1️⃣ **Create a UART Writer Task**: Tasks are marked by the `#[embassy_executor::task]` macro followed by a `async` function implementation. The task created is referred to as `uart_writer` task defined as follows:

```rust
#[embassy_executor::task]
async fn uart_writer(mut tx: UartTx<'static, UART0>)
```

`UartTx` marks a UART transmitter type that is configured with an instance of `UART0`. This means when spawning the task, we need pass a handle for a UART transmitter configured with an instance of `UART0`. This will be done in the `main` task before spawning the `uart_writer` task.

**2️⃣ Declare a Write Buffer:** Before entering the task loop, we need to declare write buffer `wbuf` to store characters that will be transmitted. `wbuf` will later store characters read from `DATAPIPE` .

```rust
let mut wbuf: [u8; READ_BUF_SIZE] = [0u8; READ_BUF_SIZE];
```

#### 🔁 Task Loop

**1️⃣ Read Characters from Pipe into Write Buffer:** In the task `loop`, the first thing we need to do is `await` a change on `DATAPIPE`. For that, there exists a `read` method for the `Pipe` type. `read` has the following signature:

```rust
pub fn read<'a>(&'a self, buf: &'a mut [u8]) -> ReadFuture<'a, M, N>
```

`read` will read a nonzero amount of bytes from the pipe into `buf`. If it is not possible to read a nonzero amount of bytes because the pipe’s buffer is empty, this method will wait until it isn’t. Consequently, we'll read `DATAPIPE` contents into `wbuf` as follows:

```rust
DATAPIPE.read(&mut wbuf).await;
```

2️⃣ **Complete Write Operations:**

As part of the `embedded_io_async` crate we can use the `write` implementation of the `Write` trait to asynchronously write a message. `write` has the following signature:

```rust
async fn write(&mut self, buf: &[u8]) -> Result<usize, Self::Error>
```

As such, we need to pass a mutable reference of a type (`UartTx` in our case) that implements `Write` and our message as a slice of `u8`. Since `write` is an `async` function, the program needs to `await` for the write operation to complete. Using `write` we'll need to transmit the buffer contents, transmit a new line, and flush the transmit buffer. This results in the following code:

```rust
// Transmit Buffer Contents
embedded_io_async::Write::write(&mut tx, &wbuf)
    .await
    .unwrap();
// Transmit a new line
embedded_io_async::Write::write(&mut tx, &[0x0D, 0x0A])
    .await
    .unwrap();
// Flush transmit buffer
embedded_io_async::Write::flush(&mut tx).await.unwrap();
```

`flush` will flush the output stream and ensure that all the buffer contents reach their destination.

> 📝 Note: I could have easily also used `esp_println` which provides more convenient abstractions to print to the console. While in this context it does not make much difference, I did this for two reasons. First, since this is a UART example, we need to demonstrate how to use UART abstractions. Second, the previous code is a good demonstration of the use of the `embedded-io-async` traits. Using the `embedded-io-async` traits allows for more portable code in some contexts.

### **📱 The Main Task**

The start of the main task is marked by the following code:

```rust
#[main]
async fn main(spawner: Spawner)
```

As the documentation states: "The main entry point of an Embassy application is defined using the `#[main]` macro. The entry point is also required to take a `Spawner` argument." As we'll see, `Spawner` is what will allow us to spawn or kick-off the `uart_writer` task.

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
    let timer_group0 = esp32c3_hal::timer::TimerGroup::new(peripherals.TIMG0, &clocks);
    embassy::init(&clocks, timer_group0.timer0);
```

> ***📝 Note:*** *At the time of writing this post, I couldn't really locate the* `init` function [**docs.rs**](http://docs.rs) documentation. It didn't seem easily accessible through any of the current HAL implementation documentation. Nevertheless, I reached the signature of the function through the source [***here***](https://github.com/esp-rs/esp-hal/blob/ece40abaed0e642b751a8752ce6406740efa4af6/esp-hal-common/src/embassy/mod.rs#L100)*.*

3️⃣ **Obtain Handle and Configure UART**: to create an instance of `UART0`, there exists a `new` instance method under `esp32c3_hal::Uart` that requires two parameters; a `UART` peripheral type and a `Clocks` type. There also exists a `split` method that allows us to "split" the `uart0` instance into separate transmitter and receiver instances with handles `tx` and `rx`. `split` returns a tuple with two instances. We also will be using `UART` methods `set_at_cmd` and `set_rx_fifo_full_threshold` to configure the `AT_CMD` character and the read buffer size.

```rust
let uart0 = Uart::new(peripherals.UART0, &clocks);
uart0.set_at_cmd(AtCmdConfig::new(None, None, None, AT_CMD, None));
uart0
    .set_rx_fifo_full_threshold(READ_BUF_SIZE as u16)
    .unwrap();
let (tx, rx) = uart0.split();
```

Note the following:

* `UART0` is chosen since it is the one that ties to the UART logging pins on board. Additionally, `new` configures UART with the default 8N1 setup.
    
* `set_at_cmd` takes a `AtCmdConfig` parameter that is a struct with configuration options for the AT-CMD detection functionality. We only need to configure the EOT character. The rest of the members can be viewed in the [documentation](https://docs.rs/esp32c3-hal/0.13.0/esp32c3_hal/uart/config/struct.AtCmdConfig.html).
    

4️⃣ **Spawn UART Reader and Writer Tasks**: before entering the button press loop, we're going to need to kick off our `uart_writer` and `uart_reader` tasks. This is done using the `spawn` method as follows:

```rust
spawner.spawn(uart_writer(tx)).unwrap();\
spawner.spawn(uart_reader(tx)).unwrap();
```

Next, we can move on to the task `loop`.

This concludes the code for the full application.

## 📱 Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/386561280544436225).

```rust
#![no_std]
#![no_main]
#![feature(type_alias_impl_trait)]

use embassy_executor::Spawner;
use embassy_sync::{blocking_mutex::raw::CriticalSectionRawMutex, pipe::Pipe};
use esp32c3_hal::{
    clock::ClockControl,
    embassy,
    peripherals::{Peripherals, UART0},
    prelude::*,
    uart::{config::AtCmdConfig, UartRx, UartTx},
    Uart,
};
use esp_backtrace as _;

// Read Buffer Size
const READ_BUF_SIZE: usize = 64;

// End of Transmission Character (Carrige Return -> 13 or 0x0D in ASCII)
const AT_CMD: u8 = 0x0D;

// Declare Pipe sync primitive to share data among Tx and Rx tasks
static DATAPIPE: Pipe<CriticalSectionRawMutex, READ_BUF_SIZE> = Pipe::new();

#[embassy_executor::task]
async fn uart_writer(mut tx: UartTx<'static, UART0>) {
    // Declare write buffer to store Tx characters
    let mut wbuf: [u8; READ_BUF_SIZE] = [0u8; READ_BUF_SIZE];
    loop {
        // Read characters from pipe into write buffer
        DATAPIPE.read(&mut wbuf).await;
        // Transmit/echo buffer contents over UART
        embedded_io_async::Write::write(&mut tx, &wbuf)
            .await
            .unwrap();
        // Transmit a new line
        embedded_io_async::Write::write(&mut tx, &[0x0D, 0x0A])
            .await
            .unwrap();
        // Flush transmit buffer
        embedded_io_async::Write::flush(&mut tx).await.unwrap();
    }
}

#[embassy_executor::task]
async fn uart_reader(mut rx: UartRx<'static, UART0>) {
    // Declare read buffer to store Rx characters
    let mut rbuf: [u8; READ_BUF_SIZE] = [0u8; READ_BUF_SIZE];
    loop {
        // Read characters from UART into read buffer until EOT
        let r = embedded_io_async::Read::read(&mut rx, &mut rbuf[0..]).await;
        match r {
            Ok(len) => {
                // If read succeeds then write recieved characters to pipe
                DATAPIPE.write_all(&rbuf[..len]).await;
            }
            Err(e) => esp_println::println!("RX Error: {:?}", e),
        }
    }
}

#[main]
async fn main(spawner: Spawner) {
    let peripherals = Peripherals::take();
    let system = peripherals.SYSTEM.split();
    let clocks = ClockControl::boot_defaults(system.clock_control).freeze();

    // Initialize Embassy with needed timers
    let timer_group0 = esp32c3_hal::timer::TimerGroup::new(peripherals.TIMG0, &clocks);
    embassy::init(&clocks, timer_group0.timer0);

    // Initialize and configure UART0
    let mut uart0 = Uart::new(peripherals.UART0, &clocks);
    uart0.set_at_cmd(AtCmdConfig::new(None, None, None, AT_CMD, None));
    uart0
        .set_rx_fifo_full_threshold(READ_BUF_SIZE as u16)
        .unwrap();
    // Split UART0 to create seperate Tx and Rx handles
    let (tx, rx) = uart0.split();

    // Spawn Tx and Rx tasks
    spawner.spawn(uart_reader(rx)).ok();
    spawner.spawn(uart_writer(tx)).ok();
}
```

## Conclusion

In this post, a UART application receiving and transmitting from/to a host was created for the ESP32C3 microcontroller. The code leveraged the `Pipe` synchronization primitive with the embassy async framework to enable a UART echo application. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]