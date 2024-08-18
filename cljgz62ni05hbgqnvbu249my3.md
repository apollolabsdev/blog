---
title: "Unlocking Possibilities: 4 Reasons Why ESP32 and Rust Make a Winning Combination"
datePublished: Thu Jun 29 2023 10:00:39 GMT+0000 (Coordinated Universal Time)
cuid: cljgz62ni05hbgqnvbu249my3
slug: unlocking-possibilities-4-reasons-why-esp32-and-rust-make-a-winning-combination
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1687710057090/6351a0cc-d1a9-4d31-a27b-a72538337fc1.png
tags: features, rust, esp32, embedded-systems

---

Rust has gained significant attention and popularity among developers due to its robustness, memory safety guarantees, and emphasis on performance. However, beyond being a language, Rust is also a thriving programming project with a vibrant community and an extensive ecosystem of tools and libraries. The Rust project encompasses not only the language's development and maintenance but also various related initiatives that aim to enhance its usability, foster collaboration, and promote its adoption. From the Rust compiler to package managers, build systems, and testing frameworks, the Rust project embodies a collective effort to empower developers with a reliable and efficient programming environment, enabling them to tackle complex software projects with confidence.

This has also carried over into embedded development with the Embedded Working Group. The Rust Embedded Working Group plays a crucial role by focusing specifically on embedded systems development. Embedded systems, characterized by their resource-constrained environments and real-time requirements, pose unique challenges for software development. The Rust Embedded Working Group has two main tasks, one is to work with the community to improve the embedded ecosystem\*.\* The other is to serve as a bridge between the Rust teams and the embedded community. Within the embedded group there are several teams working on different aspects including developing crates, resources, and tools.

Typically, if one wants to transition to embedded Rust from regular Rust there are several differences or challenges to consider. First is that most embedded devices cannot support `std` libraries which means resorting to `no_std` implementations. The immediate effect one might notice is reduced access to data structures commonly used in `std` Rust. Second, is the need for physical hardware. Third is the toolchain setup. Here one would have to configure the compile and debug toolchains for the target microcontroller device. The fourth, and final aspect, is device support and ecosystem. While the embedded working groups work to make this better, in embedded doing the same thing among two devices can take two totally different forms.

%%[substart] 

Due to the above, this is where the ESP is in a unique position. Here are the 4 reasons why:

## 1) Want to Ease into Embedded? ESP has `std` support 🏗️

Alongside the existing `no_std` support, espressif chips provide `std` support. This probably makes a big difference for somebody wanting to transition into embedded. It would really smoothen the learning curve and allow one to understand the constraints before trying to optimize thier code.

Good places to get started with std Rust on ESP include the [Rust on ESP book](https://esp-rs.github.io/book/), [Embedded Rust on Espressif](https://esp-rs.github.io/std-training/) by Ferrous Systems. There's also the [Awesome ESP Rust GitHub repository](https://github.com/esp-rs/awesome-esp-rust) that contains a lot of useful material and project examples.

## 2) No hardware? No problem. The Magic of Wokwi 🪄

[Wokwi](https://wokwi.com/) is an online platform and virtual simulation environment designed for electronics enthusiasts, hobbyists, and students. It provides a browser-based interface where individuals can design, build, and test electronic circuits using a wide range of components and microcontrollers. With Wokwi, users can simulate their projects in real-time, experiment with different configurations, and debug their designs. One can do all that without physical hardware!

Wowki was mainly an Arduino simulator, but later added support for Micropython, and then Rust. Currently, for Rust, Wokwi only supports ESP devices. You can have a look at [https://wokwi.com/rust](https://wokwi.com/rust) for example projects and starter templates. You can also check [here](https://apollolabsblog.hashnode.dev/series/esp32c3-embedded-rust-hal) for a series of blog posts for the ESP32C3 with Wokwi examples.

## 3) Dev Containers, No More Toolchain Pain 🫙

If you are unfamiliar, dev containers are lightweight, self-contained environments for software development that leverage containerization technology. They encapsulate all the necessary tools, libraries, and dependencies within a container image, eliminating the need for manual setup.

Setting up toolchains in embedded could be a major pain, especially if you are a beginner with embedded. As such, dev containers provide for a great option to relieve much of that pain. Espressif has provided several options in which one can use dev containers to ease setup. Dev container support is offered for VS Code, Gitpod, and GitHub Codespaces, resulting in a fully working environment to develop ESP boards in Rust, flash, and simulate projects with Wokwi from the container.

[This](https://github.com/georgik/rustzx-esp32) repo has a nice example of a project set up with Gitpod. Also, Dev Container support can be added when creating project templates (more detail [here](https://esp-rs.github.io/book/writing-your-own-application/generate-project/index.html?highlight=dev%20container#using-dev-containers-in-the-templates)).

## 4) Official Support 👮‍♂️

Many of the crates maintained publicly for different embedded devices/microcontrollers have no official support from the manufacturers. Espressif on the other hand, has teams invested in supporting Rust for their devices. This makes a significant difference in the speed of development, documentation, and tooling mainly. Many of the Espressif team members are also available on the esp-rs matrix channel interacting with community members. In addition to the points mentioned, there have been really useful efforts like [esp-template](https://github.com/esp-rs/esp-template) and [espflash](https://github.com/esp-rs/espflash) that make setting up an environment a breeze. This is all in addition to supporting development crates like the different device hals.

## Conclusion

Rust is a language on the rise in the embedded space. Also, for the most part, efforts in developing device ecosystems have been mostly driven by the community. The ESP ecosystem in particular, however, introduces unique solutions that would make embedded development smoother. This unique position is driven by support from Espressif. This post covers the four aspects that make developing Rust for ESPs unique. Have any questions/comments? Share your thoughts in the comments below 👇.

%%[subend]