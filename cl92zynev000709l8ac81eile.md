---
title: "Platform Agnostic Drivers in Rust: The MAX7219 Driver"
datePublished: Mon Oct 10 2022 16:37:02 GMT+0000 (Coordinated Universal Time)
cuid: cl92zynev000709l8ac81eile
slug: platform-agnostic-drivers-in-rust-the-max7219-driver
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1665419477393/lWEAjKRti.png
tags: tutorial, beginners, rust, embedded

---

## Introduction

In this post, I continue the work I started in an attempt to create a platform-agnostic driver for the MAX7219 LED Driver IC. To reach that goal, as a reminder, here are the steps I had laid out for the series:

1. Create simple code to configure and test the MAX7219 with a simple application. [Link to Post](https://apollolabsblog.hashnode.dev/stm32f4-embedded-rust-at-the-hal-spi-with-the-max7219-led-dot-matrix).
2. Refactor and optimize the code in the first step by adding functions. This step would also create a driver that isn't platform agnostic. [Link to post](https://apollolabsblog.hashnode.dev/platform-agnostic-drivers-in-rust-max7219-naive-code-refactoring).
3. **Refactor code in the second step to incorporate embedded-hal traits and create a platform-agnostic driver.**
4. Register the driver in [crates.io](https://crates.io/) and publish its documentation.
5. Add advanced features to the driver and introduce a new version of the crate.

Step 3 in **bold** is where we stand now in the series. So this means that right now the goal is to refactor (again) the code in step 2 from the [previous post](https://apollolabsblog.hashnode.dev/platform-agnostic-drivers-in-rust-max7219-naive-code-refactoring) with functions that can be utilized to drive the MAX7219 instead. 

%%[substart]

To recap quickly from the past post, there were three issues we faced:

1. The Driver Code was Verbose: We had to include the SPI peripheral and CS pin as parameters in every function call.
2. The Driver Code Restricted to One Peripheral Instance: We were stuck to a single instance of the SPI peripheral and the CS pin. Meaning, if we were to use a different SPI peripheral, the driver code itself needed to be changed.
3. Driver Code Restricted to One Platform: Similar to the previous issue,  the code was specific only to the stm32f4xx-hal and no other. 

With that being said let's get started.

## Creating a Library 📚

Before writing the driver there is something slightly different that needs to be done in creating the cargo package. All packages that we wrote code in before were binary packages that had a `main.rs` file. When creating a driver with the goal of publishing to crates.io we need to create a library package instead of a binary. This is done simply by navigating to your target folder, invoking cargo, and passing the file name with `--lib` as follows:

```
$ cargo new max7219 --lib
```
Now you'll have a file structure in your target folder that encapsulates `cargo.toml` file and a `src` folder. Inside the `src` folder, there should be a `lib.rs` file where we will be inserting our code. You can refer to the [git repo](https://github.com/apollolabsdev/stm32-nucleo-f401re) to see what the structure looks like.

If you recall, the way the functions were implemented in the past post was as standalone functions. Meaning that the functions weren't associated with a particular entity in the same code. However, recall from Rust that we could implement functions that are associated with a particular struct. Once we do, we can create instances (copies) of the same struct and each instance will have access to all the associated functions. This is somehow analogous to creating classes in OOP.

Let's take a step back and digest what was mentioned a little. Because from what we also know in Rust, we can use generics to remove type limitations. However, we also know from embedded Rust that peripherals exist as types, this means that it's possible to create a struct that removes the limitations experienced in the last post by using generics. Though if we use a generic, an issue remains. How would we restrict the generic type to the peripheral we want? Meaning, how do we make sure that when the struct is instantiated, the correct peripheral is instantiated as a member? This is where the embedded-hal traits come in. We can restrict the generic type to implement only the peripheral we want. As such, if the user tries to instantiate with a different peripheral, they'd get a compile error. Let's walk through some code to make this clearer.

## Where it All Starts 🎬

Let's start out by creating the following struct in our newly created `lib.rs` file:

```rust
pub struct MAX7219<SPI, CS> {
    spi: SPI,
    cs: CS,
}
```

As simple as this struct looks, believe it or not, this is the struct that the whole driver will revolve around. Note here what was done. A struct called `MAX7219` has been created with `SPI` and `CS` as generic placeholders. Be aware that `SPI` and `CS` at this point are still nothing but names, I could have called them `Bob` and `Amy` for that matter and it wouldn't make a difference. There is nothing yet in the code that says that `SPI` and `CS` should be something in particular. They are named this way for convenience. So if that's the case, how do we make sure that `SPI` and `CS` are actually SPI and Pin peripherals? This is where the `MAX7219` struct `impl` implementation block comes in.

## Where it all Goes 🥅

The implementation block starts out like this:

```rust
impl<SPI, CS> MAX7219<SPI, CS>
where
    SPI: Write<u8>,
    CS: OutputPin,
{
// ... Struct-associated functions go here
}
```
The `impl` block, as defined in Rust, is where we include all struct-associated functions. Note here that we brought up the `SPI` and `CS` generics again. However, we added the `where` keyword in which we associated each generic type with a trait. The line `SPI: Write<u8>` for example restricts the `SPI` generic type to the `Write` embedded-hal trait. Additionally, the line `CS: OutputPin` restricts the `CS` generic type to the `OutputPin` embedded-hal trait.

So where would one find these trait signatures anyway? there are actually two places, either the embedded-hal documentation or the documentation of whichever device hal you are using. Though you need to be careful to scroll down to the "trait implementations" section for the particular peripheral to see the implemented embedded-hal traits. 

** 🚨 Important Note **
> One thing to be aware of is that for this to work, the hal for the device you are using should be compatible with the emebedded-hal to start with. Otherwise, the implementation for the behavior of the trait you are trying to restrict your driver to might not even exist. 

## The Grand Finale 🏁

Essentially all that is left right now is to copy over code we created last time. Be mindful that there are four things to take care of here: 

**1️⃣ Copy over all the enums:**

These are the enums that we used for configuration. Note that these need to be added **outside** the implementation block!

**2️⃣ Add a `new` function:**

In order to use the driver, for every max7219 device we connect to a SPI interface, an instance of the `MAX7219` struct needs to be created. Before that, we cant do anything with the driver or access the driver functions. Right now, we do not have a function that does that. This looks as follows:

```rust 
    pub fn new(spi: SPI, cs: CS) -> Result<Self, DriverError> {
        let max7219 = MAX7219 { spi: spi, cs: cs };
        Ok(max7219)
    }
```
what is happening here, is that when the `new` instance function is called, an instance of the  `MAX7219` is created with `spi`, and `cs` passed as types for its two members. Also `spi` and `cs` are already insured to be the correct types based on the implementation constraints applied earlier. This is similar to the idea of a class constructor in object-oriented languages.

**3️⃣ Copy over and modify the existing functions:**

Remember all the driver functions that were created in the last post? All you have to do is copy them over to the implementation block with some minor modifications. To see what the modifications look like, let's do a comparison of the `transmit_raw_data` before and after modification.  The old implementation of `transmit_raw_data` looked like this:

```rust
fn transmit_raw_data(
    arr: &[u8],
    per: &mut Spi<
        SPI1,
        (
            Pin<'A', 5_u8, Alternate<5_u8>>,
            NoPin,
            Pin<'A', 7_u8, Alternate<5_u8>>,
        ),
        TransferModeNormal,
    >,
    cs: &mut Pin<'A', 6_u8, Output>,
) -> Result<(), stm32f4xx_hal::spi::Error> {
    cs.set_low();
    let transfer = per.write(&arr);
    cs.set_high();
    transfer
}
```

The new implementation now looks like this:

```rust
    pub fn transmit_raw_data(&mut self, arr: &[u8]) -> Result<(), DriverError> {
        self.cs.set_low().map_err(|_| DriverError::PinError)?;
        let transfer = self.spi.write(&arr).map_err(|_| DriverError::SpiError);
        self.cs.set_high().map_err(|_| DriverError::PinError)?;
        transfer
    }
```
If you notices, we got rid of the part we were complaining about that was limiting us! Which was the following portion of code:

```rust
    per: &mut Spi<
        SPI1,
        (
            Pin<'A', 5_u8, Alternate<5_u8>>,
            NoPin,
            Pin<'A', 7_u8, Alternate<5_u8>>,
        ),
        TransferModeNormal,
    >,
    cs: &mut Pin<'A', 6_u8, Output>,
```
this all got reduced to `&mut self`! Why because now `self` refers to the `MAX7219` driver struct instantiated that already has `spi` and `cs`  as part of it! Two other differences to note is, first, the addition of the `self` keyword to access `spi` and `cs`. Second, is the use of `map_err(|_| DriverError::PinError)`. What we're doing here is that now by using the embedded-hal traits, both `Write<u8>` and `OutputPin` return a `Result`. As such, the error of the `Result` needs to be propagated in case one happens. What I've done is add a `DriverError` enum in the code to differentiate between SPI errors and Pin errors. The driver error enum looks as such:

```rust
pub enum DriverError {
    /// An error occurred when working with SPI
    SpiError,
    /// An error occurred when working with a PIN
    PinError,
}
```
One can choose to ignore propagating/handling the errors altogether, however that would be considered bad practice. 

**4️⃣ Make the structs, enums, and functions public:** 

Everything that needs to be utilized outside of the scope of the lib file needs to be made public. This is by adding the `pub` keyword ahead of all the structs, enums, and functions that need to be public. You can note the usage of `pub` in the earlier step.

This is it, we're done! There's much less to it than initially one would expect! 

## Some Additional Notes/Tips 💵

1️⃣ If you'd like to see how the driver is used. I refactored the application code from the last post to demonstrate the usage of the driver. You can check it out [here](https://github.com/apollolabsdev/stm32-nucleo-f401re/tree/main/Drivers/max7219-driver-ex). Of course, now the application would have to import the new driver crate. If the driver crate is still on your device (not published to crates.io) then in the toml file you need to do something like this:

```rust
[dependencies]
max7219 = {path = "/Path to Driver"}
```
However, if the driver is up on crates.io all you need is the crate name and its version. Just like when importing any other crate. This is a step that we'll be doing in the next post. The next post will focus on showing how to document and publish the driver crate to crates.io.

2️⃣ There is one final thing I'd like to mention, that may make things easier. I found [this](https://github.com/ryankurte/rust-embedded-driver) embedded driver template by user [ryankurte](https://github.com/ryankurte) on github as all drivers follow the same footprint. One can start with the template, remove the parts not needed and then copy in all their function code and enums. 

## Conclusion
In this post, a Rust platform agnostic driver for the MAX7219 device was created. This was done by refactoring the code (again) from the [previous post](https://apollolabsblog.hashnode.dev/platform-agnostic-drivers-in-rust-max7219-naive-code-refactoring). In summary, although platform-agnostic drivers in Rust might sound like a scary thing to approach. In reality, it turns out that, once one feels comfortable with certain Rust concepts, there isn't much to it. I would say that the key is in understanding how generics and traits work and how everything ties to the embedded-hal. Also looking further into other embedded Rust code, understanding drivers could serve also as a good precursor to creating platform HALs. In the following post, the code will be documented properly and published to crates.io. Have any questions or thoughts? Please post them in the comments below 👇.

%%[subend]