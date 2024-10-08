---
title: "Rust FFI and cbindgen: Integrating Embedded Rust Code in C"
datePublished: Mon Jan 23 2023 18:38:15 GMT+0000 (Coordinated Universal Time)
cuid: cld95izjh000108jheprthudz
slug: rust-ffi-and-cbindgen-integrating-embedded-rust-code-in-c
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1674497623833/eb2f0d26-7915-4b3b-983c-f938373ae8b8.png
tags: tutorial, software-development, rust, embedded

---

## Introduction

In last week's [post](https://apollolabsblog.hashnode.dev/rust-ffi-and-bindgen-integrating-embedded-c-code-in-rust), I introduced Rust FFI and `bindgen`, a tool that supports the integration of C code in Rust. I also worked through an example of integrating API from the STM32F401RE device HAL C libraries code generated by ST-Microelectronics tools. Additionally, it was mentioned in last week's [post](https://apollolabsblog.hashnode.dev/rust-ffi-and-bindgen-integrating-embedded-c-code-in-rust) that Rust FFI supports integrating code in the other direction. Meaning that Rust code can be integrated into other languages like C. Also, like `bindgen`, there is a tool called `cbindgen` that can be helpful throughout the process.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1674419584847/10fddc43-622d-4e09-8088-aea326c63401.png align="center")

In this week's post, I'm going to do something somewhat similar to last week, only integrating Rust code in C using `cbindgen` instead. At first, I actually was planning to import API from the Rust [stm32f4xx-hal](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/) into C to create a blinky program using the imported [stm32f4xx-hal](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/) functions. However, I quickly came to realize that it's not possible for the most part. As it turns out, integrating Rust into C in embedded might prove to be a sticky affair the higher the abstraction level is in Rust. Because apparently, it’s hard for an unsafe language to understand good code (sorry, I just had to 😅). The issue is that a lot of the existing HALs rely on types not supported by cbindgen (or C for that matter) take for example trait objects commonly used in embedded HALs. Additionally, many of the HAL struct types' layouts are not compatible with C in the way they're arranged in memory (unless they're using the `#[repr(C)]` attribute). Something that the Rust compiler would warn about. To get around this, I had to resort to low-level implementations at the register level i.e. the PAC. This ultimately might defy the purpose of what one is trying to achieve.

> **📝 Note:** For a full list of the supported types one can refer to the cbindgen [documentation](https://github.com/eqrion/cbindgen/blob/master/docs.md).

Before moving on, one thing to note is that compared to `bindgen` , I probably didn't find as many resources online for `cbindgen` and the process of going to Rust from C in general. Nevertheless, I came across this awesome video by Katharina Fey from 2018 that is extremely informative and a must watch.

%[https://www.youtube.com/watch?v=x9acx2zgx4Q&t=568s] 

%%[substart] 

## **Steps for Creating Rust FFI bindings in C**

### The Hardware

In this post, the reference hardware used is the [**Nucleo-F401RE board**](https://s.click.aliexpress.com/e/_DdEhurV)**.** One can use a different STM32 hardware but needs to alternate the parameters to fit the different device/board.

> ***📝 Note: The code and project folders referenced in this post are available on the*** [***apollolabsdev Nucleo-F401RE***](https://github.com/apollolabsdev/stm32-nucleo-f401re) ***git repo.***

### **Step 1 - Install** `cbindgen`

`cbindgen` can be used as an application (for command line generation) or included as a build dependency (automated as part of build). The `cbindgen` application be installed using this command:

```bash
cargo install --force cbindgen
```

Later we'll see what `cbindgen` essentially does is receive a configuration and a Rust library and then spit out a C header (`.h`) file. One might think that what `cbindgen` is doing might not be that special and can be done by hand. In which some cases that might be true if the project is simple enough. Though additionally as the `cbindgen` [documentation](https://github.com/eqrion/cbindgen/blob/master/docs.md) states:

> While you could do this by hand, it's not a particularly good use of your time. It's also much more likely to be error-prone than machine-generated headers that are based on your actual Rust code. The cbindgen developers have also worked closely with the developers of Rust to ensure that the headers we generate reflect actual guarantees about Rust's type layout and ABI.

### **Step 2 - Create a Cargo Project**

As might be obvious already, a project needs to be created, in this case, however, it's a library crate (`lib.rs`) that we need to include rather than a binary crate (`main.rs`). Additionally, I will need to configure the project for the hardware I need. To make things easier, I grabbed an existing template for a [GPIO project](https://github.com/apollolabsdev/stm32-nucleo-f401re/tree/main/Polling/gpio) I've created in the past and modified it. I simply deleted `main.rs` and replaced it with an empty `lib.rs` as a first step. Finally, `cbindgen` also requires that we add a `cbindgen.toml` the configuration file which can be empty for starts. Afterward, the project folder tree looks like this:

```bash
├── Cargo.lock
├── Cargo.toml
├── build.rs
├── cbindgen.toml
├── memory.x
├── openocd.cfg
├── openocd.gdb
└── src
    └── lib.rs
```

### **Step 3 - Expose the Rust API**

This is the part where in `lib.rs` we need to write the code that we want to be exposed. This is also a part where I spent quite some time. I "simply" wanted to create a `togglePin` function that receives one of the stm32f4xx-hal types. I was a bit naive in the beginning to think that I can grab any HAL type and plug it in and export it so to speak. Though soon enough the awesome Rust compiler warns that the type is not FFI-safe. I will get to the code and explain soon enough, though, for example, when I tried to pass a `Pin` type for pin `PA5`, I got something like this warning:

```rust
warning: `extern` fn uses type `stm32f4xx_hal::gpio::Pin<'A', 5_u8>`, which is not FFI-safe
  --> src/lib.rs:36:37
   |
36 | pub unsafe extern "C" fn somePin(p: stm32f4xx_hal::gpio::Pin<'A', 5>) {
   |                                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ not FFI-safe
   |
   = note: `#[warn(improper_ctypes_definitions)]` on by default
   = help: consider adding a `#[repr(C)]` or `#[repr(transparent)]` attribute to this struct
   = note: this struct has unspecified layout
```

Before writing the API I specified the attributes and crates needed:

```rust
#![no_std]
use panic_halt as _;
use stm32f4::stm32f401::gpioa::{afrh, afrl, bsrr, idr, lckr, moder, odr, ospeedr, otyper, pupdr};
```

The first two lines are ones we continuously see in a standard Rust embedded template. Additionally, in the last line, I am importing the low-level register types/names for GPIOA that I am going to need in defining the GPIOA `RegisterBlock` struct that I define next.

Afterward, I define the `RegisterBlock` struct:

```rust
pub struct RegisterBlock {
    #[doc = "0x00 - GPIO port mode register"]
    pub moder: stm32f4::Reg<moder::MODER_SPEC>,
    #[doc = "0x04 - GPIO port output type register"]
    pub otyper: stm32f4::Reg<otyper::OTYPER_SPEC>,
    #[doc = "0x08 - GPIO port output speed register"]
    pub ospeedr: stm32f4::Reg<ospeedr::OSPEEDR_SPEC>,
    #[doc = "0x0c - GPIO port pull-up/pull-down register"]
    pub pupdr: stm32f4::Reg<pupdr::PUPDR_SPEC>,
    #[doc = "0x10 - GPIO port input data register"]
    pub idr: stm32f4::Reg<idr::IDR_SPEC>,
    #[doc = "0x14 - GPIO port output data register"]
    pub odr: stm32f4::Reg<odr::ODR_SPEC>,
    #[doc = "0x18 - GPIO port bit set/reset register"]
    pub bsrr: stm32f4::Reg<bsrr::BSRR_SPEC>,
    #[doc = "0x1c - GPIO port configuration lock register"]
    pub lckr: stm32f4::Reg<lckr::LCKR_SPEC>,
    #[doc = "0x20 - GPIO alternate function low register"]
    pub afrl: stm32f4::Reg<afrl::AFRL_SPEC>,
    #[doc = "0x24 - GPIO alternate function high register"]
    pub afrh: stm32f4::Reg<afrh::AFRH_SPEC>,
}
```

Essentially, all I did here was go into the PAC source code and copy over the `RegisterBlock` struct code as is and then adjust the namespaces to point to the `stm32f4` registers. So the question is why did I need to do this? That is instead of just importing `RegisterBlock` earlier. It turns out, that when I did that, I noticed that `cbindgen` later did not declare the `RegisterBlock` type in the generated header file.

Moving on to the last step here, the API I wanted to create was a `togglePin` function that can toggle PA5 that controls the onboard LED. This is the resulting code I wrote:

```rust
#[no_mangle]
pub unsafe extern "C" fn togglePin(p: &mut RegisterBlock) {
    // Toggle the ODR5 bit, all the other bits will remain untouched
    p.odr.modify(|r, w| w.odr5().bit(!r.odr5().bit()));
}
```

A few things to note:

1. **Attribute** `#[no_mangle]` : This is needed because the Rust compiler "mangles" or in other words changes the function names. This makes it incompatible with C. As such, the `#[no_mangle]` attribute instructs the compiler not to mess around with names.
    
2. `extern "C"` : This makes this function adhere to the C calling convention.
    
3. `p: &mut RegisterBlock` : This makes the function accept a mutable reference to a GPIO register block.
    
4. **The function code:** The line of code is PAC-level code that toggles bit number 5 in the ODR register in a GPIO port register block that is passed to the function.
    

## **Step 4 - Setup the Build Configuration**

In this step, we need to ensure that the build generates a static library (with `.a` extension) as an output. As a result, in cargo.toml the following is added:

```rust
[lib]
name = "toggle"
crate-type = ["staticlib"] # for .a static lib
```

## **Step 5 - Setup & Run the Build Script**

Hera one can create a script (which would make sense for more complex projects). To create a script, first, we need to add `cbindgen` as build dependency in cargo.toml as follows:

```rust
[build-dependencies]
cbindgen = "0.20.0"
```

Second, we need to modify `build.rs` to include our build commands as part of the build process. The following code was appended to `build.rs`:

```rust
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();
    let mut config: cbindgen::Config = Default::default();
    config.language = cbindgen::Language::C;

    cbindgen::Builder::new()
        .with_crate(crate_dir)
        .with_config(config)
        .generate()
        .expect("Unable to generate bindings")
        .write_to_file("bindings.h");
```

The above code is pointing the builder to the crate location, configuring it for the C language, specifying the output file name, and instructing it to generate the header. This is largely similar to the approach used with the `bindgen` `Builder`. A full list of supported methods for `cbindgen::Builder` are available [here](https://docs.rs/cbindgen/latest/cbindgen/struct.Builder.html#method.new).

For the earlier specified Rust code, `cbindgen` generates the following header:

```c
#include <stdarg.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdlib.h>

typedef struct RegisterBlock RegisterBlock;

void togglePin(struct RegisterBlock *p);
```

The header contains effectively the function prototype for `togglePin` and the type definition for the `RegisterBlock` struct.

> **📝 Note:** I encountered strange behavior where if `bindings.h` was generated already from a previous build, it wouldn't update when running a new build with changes. I had to delete the `bindings.h` file and run `cargo clean` to get the updated `bindings.h` . The point here is that, as a word of caution, it would pay off to check that the header files have been updated after a build.

## **Alternative Step 5 - Command Line Approach**

As an alternative of what was done previously in step 5, we can use the command line by running the following command:

```bash
cbindgen --config cbindgen.toml --lang c --crate lib.rs --output bindings.h
```

Running the above command line produces a `bindings.h` header file (as requested in the command) for C in our root folder. The `--config` cbindgen.toml switch passes the `cbindgen` toml file, the `--lang c` switch specifies that we need a C header, `--crate lib.rs` specifies the name of the crate we want headers generated for, and finally `--output bindings.h` specifies the name of the output header file.

## **Step 6 - Generate the STM32 HAL C Project**

Before we can do anything with the files from the previous step, we need to generate/obtain the STM32 C project containing all the header and implementation files for the particular device used. This is exactly the same process I followed in the first step when in the `bindgen` post. Then I used the ST-Microelectronics tools ([**CubeMX**](https://www.st.com/en/development-tools/stm32cubemx.html) in particular) to configure and then generate C STM HAL project files for a particular board/device. Again, for the purpose of this post, since a simple blinky application is going to be created, I kept the basic configuration that exists for the Nucleo-F401RE board I'm using in CubeMX. In generating the code, for the Toolchain/IDE option, I selected Makefile here as well.

## **Step 7 - Integrate Generated Files into C Project**

As part of the build in step 5, a static library named `libtoggle.a` was generated and placed in the `./ffi_rust_project/target/thumbv7em-none-eabihf/debug/` folder. Consequently, the generated static library along with the header file needs to be moved into the appropriate locations in the C project. Moreover, the header file needs to be imported into the appropriate code files that need it. In our context, I placed `bindings.h` in `./ffi_c_project/Core/Src` . `libtoggle.a` , on the other hand, was placed in `./ffi_c_project/build` the target directory for the makefile.

## **Step 8 - Modify C Code**

In the project `main.c` we first need to include the `bindings.h` header so that we have access to the bindings we brought in from Rust:

```c
#include "bindings.h"
```

further down in the code, in the application loop the `tooglePin` function is used as follows:

```c
  while (1)
  {
    togglePin((RegisterBlock *)GPIOA_BASE);
    HAL_Delay(1000);
  }
```

Here I am type casting `GPIOA_BASE` to a pointer of type `RegisterBlock`. `RegisterBlock` is the type definition we brought in through our `bingings.h` from Rust. `GPIOA_BASE` on the other hand, is a `#define` in the C project for the base address of the Port A GPIO register in the STM32F4.

Are we done yet? Of course not 😀

## **Step 9 - Modify Makefile**

Before building the project, we need to take the linker to include the `libtoggle.a` static library that defines the implementation of the imported function. To do that, I appended the following to the end of the `LDFLAGS` environment variable:

```makefile
-L$(BUILD_DIR) -ltoggle -static
```

`-L$(BUILD_DIR)` specifies the location of the library (the build folder), `-ltoggle` specifies the name of the library (note how here the lib prefix is eliminated), and `-static` specifies that the library is static.

The full `LDFLAGS` definition ends up looking something like this:

```makefile
LDFLAGS = $(MCU) -specs=nano.specs -T$(LDSCRIPT) $(LIBDIR) $(LIBS) -Wl,-Map=$(BUILD_DIR)/$(TARGET).map,--cref -Wl,--gc-sections -L$(BUILDDIR) -llibtoggle -static
```

## **Step 10 - Build & Flash C Project**

The project is simply built by running `make` from the root of the C project, next, I used the `st-flash` tool/command to flash the code binary to the device as follows:

```bash
st-flash write ffi_c_project.bin 0x08000000
```

## Conclusion

Exporting Rust API to C can prove to be a more daunting task than it seems. Especially compared to going the other way from C to Rust. This is mainly because of the incompatible types going from Rust to C. As such, a lot of work might need to be done to achieve compatibility. In the context of embedded and in the example I show in this post, I had to dive down to low-level Rust code to achieve my goal. I'm not sure which direction of API integration is going to be more common in the future (C to Rust or Rust to C). Though it's obvious that going from Rust to C, much of the abstractions and probably advantages achieved in the code (safety or another perspective), are prone to be lost. Have any questions/comments? Share your thoughts in the comments below 👇.

%%[subend]