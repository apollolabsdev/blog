---
title: "ESP Embedded Rust: Multithreading with FreeRTOS Bindings"
datePublished: Fri Sep 22 2023 19:33:19 GMT+0000 (Coordinated Universal Time)
cuid: clmv01xyw00010al49vuf60he
slug: esp-embedded-rust-multithreading-with-freertos-bindings
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1695409997321/9b5eb9d2-a9bf-49d3-9a2b-4ab45ad3aa51.png
tags: tutorial, rust, esp32, embedded-systems

---

## Introduction

In the Rust ESP `std` ecosystem, different levels of abstraction allow for different levels of access. In [last week's post](https://apollolabsblog.hashnode.dev/the-embedded-rust-esp-development-ecosystem), the different levels of Rust layers in the ESP `std` ecosystem were explained in detail. One of those layers was the `esp-idf-sys` which is the first layer on top of the existing ESP-IDF framework. The `esp-idf-sys` provides (`unsafe`) bindings to the underlying ESP-IDF framework which also includes FreeRTOS interfaces.

In this post, I'll be demonstrating how to access the lower-level `esp-idf-sys` crate that provides a direct interface (via bindings) to the ESP-IDF framework. As such, I'll be creating a multitasking application using FreeRTOS interfaces from the ESP-IDF framework. The FreeRTOS interfaces will be Rust interfaces imported from the `esp-idf-sys` crate.

%%[substart] 

### **📚 Knowledge Pre-requisites**

To understand the content of this post, you need the following:

* Basic knowledge of coding in Rust.
    
* Familiarity with multitasking in FreeRTOS and its functions.
    

### **💾 Software Setup**

All the code presented in this post is available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Note that if the code on the git repo is slightly different, it was modified to enhance the code quality or accommodate any HAL/Rust updates.

Additionally, the full project (code and simulation) is available on Wokwi [**here**](https://wokwi.com/projects/374959713217594369).

### **🛠 Hardware Setup**

#### Materials

* [**ESP32-C3-DevKitM**](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/hw-reference/esp32c3/user-guide-devkitm-1.html)
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681796942083/da81fb7b-1f90-4593-a848-11c53a87821d.jpeg?auto=compress,format&format=webp&auto=compress,format&format=webp align="center")
    

## **👨‍🎨** The Application

The main goal of this post is to demonstrate the usage of the ESP IDF FFI functions in Rust. Note that the post is not as much about the technical details of how FreeRTOS multitasking works and more about how to leverage the underlying layers in the Espressif Rust framework. For the reader interested in learning more about FreeRTOS, the [ESP FreeRTOS documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/freertos.html) is a recommended starting point.

As a short description, the application in this post will have two tasks run in parallel, in addition to `main`. To demonstrate multitasking, each task is going to print its own message after some delay. As such, only two tasks are going to be defined and created.

## **👨‍💻** Code Implementation

### **📥 Crate Imports**

In this implementation the crates required are as follows:

* The `std::sync` to import synchronization primitives.
    
* The `esp_idf_hal` crate to import the `FreeRtos` delay abstraction.
    
* `std::ffi::CString` is imported to provide a type that represents a C-compatible null-terminated string.
    
* The `esp_idf_sys` crate importing the necessary FFI function interfaces.
    

```rust
use esp_idf_hal::delay::FreeRtos;
use esp_idf_sys::{self as _};
use esp_idf_sys::xTaskCreatePinnedToCore;
use std::ffi::CString;
```

### 📝 Define the Tasks

To define new tasks there is a `xTaskCreatePinnedToCore()` function that exists within the FreeRTOS core for ESP32. As the function's name implies, we are allowed to choose which processor core to pin the task to. Keep in mind, that this is an FFI function coming from C made compatible with Rust, so its parameters have to be Rust-compatible types. Also, the function needs to be wrapped in an `unsafe` block since the Rust compiler cannot guarantee its safety (it doesn't mean the function is necessarily unsafe). Moving on, `xTaskCreatePinnedToCore()` takes seven parameters in the following order:

1. This is an `Option` containing the name of the function to run as a parallel task.
    
2. A string describing the task. This is mainly for debugging and could be the same as the task identifier. This would be a Rust equivalent of a C-style string.
    
3. An integer value representing the stack size for the task. The value represents the number of bytes needed.
    
4. A pointer to any parameter being passed to the new task. We won't be passing anything here so this will be the Rust equivalent of a C `NULL`.
    
5. The priority of the task. I'm going to be setting it to `0` for both.
    
6. A task handle to keep track of the created task. It's a handle that can be used to invoke the task. Similar to the 4th parameter, we won't be passing anything here so this will be the Rust equivalent of a C `NULL` as well.
    
7. The ID of the processor core to run the task on. Core indices start from 0. For a two-core processor, the numbers will be `0` and `1`.
    

Following the above, we define two tasks, `task1` and `task2` as follows:

```rust
unsafe {
    xTaskCreatePinnedToCore(
        Some(task1),
        CString::new("Task 1").unwrap().as_ptr(),
        1000,
        std::ptr::null_mut(),
        10,
        std::ptr::null_mut(),
        1,
    );
}

unsafe {
    xTaskCreatePinnedToCore(
        Some(task2),
        CString::new("Task 2").unwrap().as_ptr(),
        1000,
        std::ptr::null_mut(),
        9,
        std::ptr::null_mut(),
        1,
    );
}
```

### 👨‍💻 Create the Functions

We still have to create the functions that we created tasks for. We named the functions `task1` and `task2` . These functions need to adhere to the C calling convention. As a result, the `extern "C"` is used to achieve that. Normally using FreeRTOS in C the task function definition has the following signature:

```c
void task1( void * pvParameters ){
  for(;;){
    // Code will go here
  }
}
```

Note that the function takes a single argument which is a C `void` pointer type. The name `pvParameters` itself is irrelevant and could be anything. Given the above, to create our tasks this translates to the following in Rust:

```rust
unsafe extern "C" fn task1(_: *mut core::ffi::c_void) {
    loop {
        println!("Task 1 Entered");
        FreeRtos::delay_ms(1000);
    }
}
unsafe extern "C" fn task2(_: *mut core::ffi::c_void) {
    loop {
        println!("Task 2 Entered");
        FreeRtos::delay_ms(2000);
    }
}
```

In each of the functions as soon as we enter, a message is printed indicating which task has been entered. A few things to note:

* `*mut core::ffi::c_void` is the equivalent Rust type to a C `void` pointer.
    
* Both tasks have infinite loops inside them. However, using the `FreeRtos` abstraction we can block the task. This hands execution back to the kernel to run the next task with the highest priority.
    
* `task1` is executed every 1 second and `task2` is executed every 2 seconds.
    

### 🔄 The Main Loop

There still remains the main thread that we can print statements from. As such, I add another `loop` and `println` statement to print a message from the `main` thread every 500 ms.

```rust
loop {
    println!("Hello From Main");
    FreeRtos::delay_ms(500);
}
```

That's it for code!

## 📱Full Application Code

Here is the full code for the implementation described in this post. You can additionally find the full project and others available on the [**apollolabs ESP32C3**](https://github.com/apollolabsdev/ESP32C3) git repo. Also, the Wokwi project can be accessed [**here**](https://wokwi.com/projects/374959713217594369).

```rust
use esp_idf_hal::delay::FreeRtos;
use esp_idf_sys::{self as _};
use esp_idf_sys::{esp, vTaskDelay, xPortGetTickRateHz, xTaskCreatePinnedToCore, xTaskDelayUntil};
use std::ffi::CString;

unsafe extern "C" fn task1(_: *mut core::ffi::c_void) {
    loop {
        println!("Task 1 Entered");
        FreeRtos::delay_ms(1000);
    }
}

unsafe extern "C" fn task2(_: *mut core::ffi::c_void) {
    loop {
        println!("Task 2 Entered");
        FreeRtos::delay_ms(2000);
    }
}

fn main() -> anyhow::Result<()> {
    // It is necessary to call this function once. Otherwise some patches to the runtime
    // implemented by esp-idf-sys might not link properly. See https://github.com/esp-rs/esp-idf-template/issues/71
    esp_idf_sys::link_patches();

    unsafe {
        xTaskCreatePinnedToCore(
            Some(task1),
            CString::new("Task 1").unwrap().as_ptr(),
            1000,
            std::ptr::null_mut(),
            10,
            std::ptr::null_mut(),
            1,
        );
    }

    unsafe {
        xTaskCreatePinnedToCore(
            Some(task2),
            CString::new("Task 2").unwrap().as_ptr(),
            1000,
            std::ptr::null_mut(),
            9,
            std::ptr::null_mut(),
            1,
        );
    }
    loop {
        println!("Hello From Main");
        FreeRtos::delay_ms(500);
    }
}
```

## **Conclusion**

This post showed how to leverage the ESP IDF framework Rust FFI bindings. In the demonstration, a multitasking application was created on the ESP32C3. The application leveraged the FreeRTOS core FFI bindings in the `esp-idf-sys` crate. Have any questions? Share your thoughts in the comments below 👇.

%%[subend]