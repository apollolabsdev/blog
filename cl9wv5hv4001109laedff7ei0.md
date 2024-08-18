---
title: "4 Simple Steps for Creating a Platform Agnostic Driver in Rust"
datePublished: Mon Oct 31 2022 14:15:29 GMT+0000 (Coordinated Universal Time)
cuid: cl9wv5hv4001109laedff7ei0
slug: 4-simple-steps-for-creating-a-platform-agnostic-driver-in-rust
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1667225238021/7lnXWXdJh.png
tags: tutorial, iot, rust, internet-of-things

---

For those following my blog posts, I have recently written several separate posts working through creating platform-agnostic drivers in Rust. Those posts worked through the details for particular device implementations, documentation, and crates.io publication. Although it took several posts before, it was more to show how powerful the idea is and how it's enabled by Rust. However, I wouldn't want to establish the notion that creating platform-agnostic drivers in Rust is complicated. This post is sort of the TL;DR version as I figured it would be only appropriate if one is interested in getting the gist of it. 

%%[substart]

Essentially all that needs to be done can be broken down into the following 4 steps:

## 1️⃣ Create a Library Binary Package that Imports the Embedded HAL 📚

Instead of creating a regular binary package with a `main.rs` here we need a library package that will be uploaded to crates.io. The library package can be created with cargo in the command line as follows:

```rust
$ cargo new [driver name] --lib
```

Next, the embedded-hal needs to be imported in the `lib.rs` along with any modules required for the driver. Here's an example of imports we would need if we are to use GPIO Output and SPI:

```rust
use embedded_hal as hal;

use hal::blocking::spi::Write;
use hal::digital::v2::OutputPin;
use hal::spi::{Mode, Phase, Polarity};
```
To identify the modules one would refer to the embedded-hal [documentation](https://docs.rs/embedded-hal/latest/embedded_hal/).

## 2️⃣ Create the Driver Struct 🏗

This is the main part of the driver. We'd have to define the struct that through it, all the function implementations will be provided. This looks something like this:

```rust
pub struct DriverName<T, U> {
    peripheral1: T,
    peripheral2: U,
    // Any other peripherals
}
```

Both `T` and `U` here are generics that will be bound to particular traits in the implementation of the struct.

## 3️⃣ Implement your Driver 🚘
Create an implementation block that defines the generic types and includes at a minimum a function to allow for instantiating the driver. The driver implmentation block would look like this:

```rust
impl<T, U> DriverName<T, U>
where
    T: Write<u8>,
    U: OutputPin,
{
// Implmemented Fucntions
}
```
Here the generic type `T` is being bound to implement the `Write<u8>` trait in `hal::blocking::spi::Write` from the embedded-hal crate. Additionally, `U` bound to the `OutputPin` trait in `hal::digital::v2::OutputPin` from the embedded-hal crate. Keep in mind here that depending on what you want to do the trait bound would be different. One can find the appropriate bound from the [embedded-hal documentation](https://docs.rs/embedded-hal/latest/embedded_hal/). For example, here the `Write<u8>` trait supports only writing in a single direction using SPI. However, if one would need to both write and read (full-duplex transaction) instead would use the [`Transfer` trait](https://docs.rs/embedded-hal/latest/embedded_hal/blocking/spi/trait.Transfer.html). Or, alternatively, as another example, if one would want to implement an SPI slave, then the `InputPin` trait would be more appropriate instead. 

Now inside the `impl` block, an instantiating function needs to be created and can be called `new`. Here's how it would look like:

```rust
pub fn new(peripheral1: T, peripheral2: U) -> Result<Self, DriverError> {
     let driver = DriverName { peripheral1: T, peripheral2: U };
     Ok(driver)
}
```
In `new` a `Result` is being returned to confirm that the driver has been instantiated successfully. Optionally the error can be ignored but that would be poor design. Also, here `DriverError` is an enum that is defined in the same library file and enumerates the errors that can occur. The `DriverError` enum looks as follows:

```rust
pub enum DriverError {
    SpiError,
    PinError,
}
```

Here also two error types are defined. `SpiError` indicates that an error in the SPI peripheral occurred, and `PinError` indicates that an error in the GPIO peripheral occurred. These could have all been bundled in one type of error but it's up to the designer. At least this way some differentiation is helpful for debugging the source of the error.

## 4️⃣ Implement the Driver Functions 🧰
This is the final step, in the `impl` block, in addition to the `new` function, we would add any other driver-associated function. These would be functions that one would have planned ahead of time and define what things the driver can do. Overall, the `impl` block would resemble something like this:

```rust
impl<T, U> DriverName<T, U>
where
    T: Write<u8>,
    U: OutputPin,
{
     
 pub fn new(peripheral1: T, peripheral2: U) -> Result<Self, DriverError> {
        let driver = DriverName { peripheral1: T, peripheral2: U };
        Ok(driver)
 }

pub fn driverFunc1(&self, optionalParam: [paramType]) -> () {
        // Function 1 Implementation
 }

pub fn driverFunc2(&self, optionalParam: [paramType]) -> Result<Self, DriverError> {
        // Function 2 Implementation
 }

// Remaining function implementations

}
```

If one would has the need to use enums in their driver, one would define any enums outside of the `impl` block though still inside the `impl` block.

## Conclusion

The implementation of a platform-agnostic driver in Rust might sound intimidating at first. However, going through the various steps it turns out that it's less challenging than one might think. Probably the most challenging part is dealing with the aspect of using generics and traits. Nevertheless, in this post, I attempt to simplify the process of creating platform gnostic drivers in Rust by breaking it down into 4 steps. This should be all that one needs to create a simple driver.

Have any questions or thoughts? Please post them in the comments below 👇. 

%%[subend]