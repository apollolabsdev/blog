---
title: "34 Must Know Terms for Embedded Rust Newbies"
datePublished: Mon Nov 07 2022 18:35:50 GMT+0000 (Coordinated Universal Time)
cuid: cla74ja1z000f08l2apvacfnw
slug: 34-must-know-terms-for-embedded-rust-newbies
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1667742933685/Ly4nsff8p.png
tags: iot, beginners, rust, embedded

---

When I first started out in embedded Rust and engaged with the community, I found myself in the midst of some specific jargon and acronyms. At the time, some of that jargon flew ✈ over my head 👱‍♂️ faster than a speeding bullet 🔫.

In this post, I try to aggregate many of those terms that I saw/read frequently in the hope to create a quick reference for someone new to the space. Note not all are specific to embedded-Rust and some are already common terms, however, one would see most mentioned ever so often in association.

%%[substart]

I used the following icons in an attempt to somewhat categorize the different terms:
- 🛠 Tools (Debug, Flash, Protocol, or Utility)
- 🖼 Framework/Runtime
- 🏗 Repository/Project
- 📁 File Extension
- 🦀 Rust Keyword/Trait/Type
- 📦 Library/Crate


📝 **Note**:
> If you want to read more about a particular term, many of the terms have embedded links if you click on the term itself.

Here it goes:

1. 🖼 **[RTIC](https://rtic.rs/1/book/en/)** - Short for Real-Time Interrupt-driven Concurrency which is a framework (Not an Operating System) for building real-time systems. As a starter, you'd probably get introduced to this framework as an alternative for implementing interrupt-based applications. RTIC provides a less verbose and more structured way of dealing with interrupts in Rust.

2. 🖼 **RTFM** - Short for Real-Time for the Masses, this was the older naming for the RTIC framework.

3. 🦀 **[`async`](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)** - Used in the context of multi-threaded applications, `async` is a keyword used in Rust ahead of a function to make it return a `Future` or promise to resolve. 

4. 🦀 **[`await`](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)**  - Also used in the context of multi-threaded applications, `await` is a keyword used inside an `async` block/function that makes the application wait until the `Future` or promise resolves. 

 📝 **Note:**
> `async` and `await` are commonly mentioned in conjunction with each other as `async/await`.

5. 🦀 **[`Future`](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)** - Is used in asynchronous programming and is a trait representing a value that might not have finished computation yet. A `Future` makes it possible for a thread to continue doing useful work until a value becomes available.

6. 🖼 **[Tokio](https://tokio.rs/)** - Tokio is a Rust runtime for writing multi-threaded asynchronous applications. Tokio builds on Rust's asynchronous features to provide a runtime, APIs (networking, filesystem operations..etc.), and asynchronous task tools among others. 

7. 🖼 **[Embassy](https://embassy.dev/)** - Can be considered as the embedded version of Tokio albeit more compact and with fewer features. Embassy is a more encompassing HAL and also can serve as an alternative to the RTIC framework. 

8. 🖼 **[uAMP (microAMP)](https://github.com/rtic-rs/microamp)** - A (micro) framework for building bare-metal AMP (Asymmetric Multi-Processing) applications. It's the core foundation of the multi-core version of (RTIC).

9. 🦀 **[`Cow`](https://doc.rust-lang.org/std/borrow/enum.Cow.html)** - This one is probably a bit out of context but I saw it way too often that I had to mention it. The naming is quite confusing obviously 🐄. It turns out that `Cow` is a type of smart pointer similar to `Cell`, `RefCell`, or `Arc`, where it stands for clone-on-write.

10. 🏗 **[Knurling](https://knurling.ferrous-systems.com/)** - The term is used in reference to the Knurling-rs project created by [Ferrous Systems](https://ferrous-systems.com/), a consulting firm specialized in Rust. The goal of the Knurling project is to enhance bare metal development using Rust by providing training material and tools. 

11. 🛠 **[Probe-run](https://ferrous-systems.com/blog/knurling-changelog-1/)** - is one of the Knurling tools introduced to easily flash and run embedded applications on baremetal devices. Probe-run also provides stack backtraces that mimic Rust panicking behavior to see where things go wrong. Additional detail [here](https://knurling.ferrous-systems.com/tools/).

12. 🛠 **[Defmt](https://ferrous-systems.com/blog/knurling-changelog-1/)** - is also part of the Knurling tools and is logging framework for microcontrollers. Sort of an efficient alternative for the traditional serial monitor using UART. defmt stands for "deferred formatting". Additional detail [here](https://knurling.ferrous-systems.com/tools/).

13. 🛠 **[Flip-Link](https://ferrous-systems.com/blog/knurling-changelog-1/)** - Another Kunrling tool that is a linker wrapper that adds stack overflow protection to embedded applications. Additional detail [here](https://knurling.ferrous-systems.com/tools/).

14. 🦀 **[FFI](https://doc.rust-lang.org/nomicon/ffi.html)** - Short for Foreign Function Interface which is a function interface that allows calls to C library functions from within Rust.

15. 🛠 **[RTT](https://www.segger.com/products/debug-probes/j-link/technology/about-real-time-transfer/)** - Short for Real-Time Transfer and refers to a protocol created by J-Link used in debugging microcontroller devices.

16. 📦 **[rtt_target](https://docs.rs/rtt-target/latest/rtt_target/)** - Crate that provides the target side implementation of the RTT protocol.

17. 📦 **[defmt-rtt](https://docs.rs/defmt-rtt/latest/defmt_rtt/)** - Crate that supports transmitting defmt log messages over the RTT (Real-Time Transfer) protocol.

18. 📦 **[PAC](https://docs.rust-embedded.org/book/start/registers.html)** - Short for peripheral access crate and is a lower layer of abstraction that provides a wrapper around microcontroller peripherals memory mapped registers. Typically each microcontroller has its own PAC.

19. 📦 **[HAL](https://docs.rust-embedded.org/book/start/registers.html)** - Short for Hardware Abstraction Layer which provides a higher level of abstraction and is the layer that sits on top of the PAC. Several microcontrollers can be bundled under one HAL.

20. 📁 **[SVD](https://www.keil.com/pack/doc/CMSIS/SVD/html/svd_Format_pg.html)** - Short for System View Description and is a file format that formalizes the description of the system contained in microcontrollers, in particular, the memory mapped registers of peripherals. The detail contained in system view descriptions is comparable to the data in device reference manuals.

21. 🛠 **[SVD2Rust](https://docs.rs/svd2rust/latest/svd2rust/)** - Is a command line tool that transforms SVD files into crates that expose a type safe API to access the peripherals of the device.

22. 🛠 **[cargo-embed](https://probe.rs/docs/tools/cargo-embed/)** - Part of probe-rs, cargo-embed is a cargo subcommand that supports flashing and debug logging of embedded targets.

23. 🛠 **[cargo-flash](https://probe.rs/docs/tools/cargo-flash/)** - Also part probe-rs, and is a cargo subcommand for flashing embedded targets.

24. 🛠 **[OpenOCD](https://openocd.org/doc/html/About.html)** - Short for Open On-Chip  Debugger and is a program that provides an interface between a debug adapter and a host computer. OpenOCD provides debug, test, and programming capabilities for microcontrollers.

25. 🛠 **[GDB](https://www.sourceware.org/gdb/)** - Part of a debugging toolchain and short for GNU Debugger. GDB is a popular platform used in debugging applications. It provides a user interface to debug microcontroller applications. GDB typically connects to a microcontroller through OpenOCD.

26. 🛠 **[ITM](https://developer.arm.com/documentation/ddi0337/e/System-Debug/ITM)** - Short for Instrumentation Trace Macrocell and is a debug feature/tool available in particular on ARM Cortex-M devices and is an application driven microcontroller trace source for debugging embedded applications. Also one of the possible options to replace traditional serial monitoring.

27. 🛠 **[Semihosting](https://wiki.segger.com/Semihosting)** - is another logging mechanism/framework for embedded systems application debugging. Also another alternative for traditional serial communication logging.

 📝 **Note:**
> It's probably obvious at this point that there are many logging mechanisms/frameworks. The choice typically would depend on the speed and features of a mechanism. Some are considered slower than others and some also consume more application overhead than others. [Here](https://blog.japaric.io/itm/) is a blog post from 2017 that nicely explains the differences.

28. 📦 **[nalgebra](https://nalgebra.org/)** - is the name for a linear algebra library written using Rust. This library is commonly used with the Rust-embedded graphics library.

29. 🖼 **[WASM](https://webassembly.org/)** - is short for Web Assembly and provides a standard for writing applications in a particular format known as a binary instruction format. Technically, application code in any programming language can be compiled into WASM. The generated bytecode, as a result, needs to run in a virtualized environment that can execute the WASM bytecode. This means that WASM can also technically execute on any platform. WASM is commonly referred to as a build target for a conceptual machine.
 
30. 🖼 **[WASI](https://wasi.dev/)** - is short for web assembly system interface and enables the usage of WASM outside the context of the web. Essentially WASI provides a mechanism (or standard to be more accurate) for WASM to access system functions. It is often mentioned that WASI provides the view of a conceptual operating system withing WASM. For more on the difference between WASM and WASI [here](https://stackoverflow.com/questions/59609013/wasi-vs-web-assembly) is a useful Stackoverflow post. 

31. 📁 **witx** is a file format based on the WASM text format. witx is also supposed to provide a set of specifications for embedded device interfaces.

32. 🏗 **[esp-rs](https://github.com/esp-rs)** - Name of the project containing libraries, crates, and examples for using Rust on Espressif SoC's

33. 🏗 **[rp-rs](https://github.com/rp-rs)** - Name of the project containing libraries, crates, and examples for using Rust on the rasberry pi series of microcontrollers.

34. 🏗 **[stm32-rs](https://github.com/stm32-rs)** - Name of the project containing libraries, crates, and examples for using Rust on STM32 microcontrollers

## Conclusion

Monitoring embedded Rust community chatter can be challenging for a noob that isnt familiar with many terms. As I've progressed in embedded Rust myself, I've made a list of terms that I found useful and summarized in this post. I hope for the new learners to be able to leverage this list and find it useful. What was your experience like? Are there different terms you encountered? Share your thoughts in the comments 👇.

%%[subend]

