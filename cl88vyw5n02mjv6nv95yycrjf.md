---
title: "Embedded IoT Without Hardware: 8 Must Know Resources"
datePublished: Mon Sep 19 2022 14:52:10 GMT+0000 (Coordinated Universal Time)
cuid: cl88vyw5n02mjv6nv95yycrjf
slug: embedded-iot-without-hardware-8-must-know-resources
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1663487409766/FAlLxe8_Y.PNG
tags: learning, iot, beginners, embedded

---

## Introduction
I've always wondered how embedded development in general is wasteful and, well, unsustainable. Whether it was a beginner or a professional somehow you end up with several dev boards that you don't use or stashed under your desk. In a world suffering from chip shortages now the effect is being felt. Many on online forums express their struggle to access even the most popular dev boards. 

On a second note, given that embedded requires hardware to learn. I can imagine that acquiring and setting up hardware can be an entry barrier for many. Pure software development becomes a much more convenient and cheaper option for many. Sort of like the difference in picking up a cheap (soccer or basketball) vs. expensive (ice hockey) sport to learn. Not that one is easier than the other, but rather the setup that is involved to be able to learn.

Also for beginners, as I've expressed in my [4 Reasons Why Embedded IoT Learners Give Up Early](https://apollolabsblog.hashnode.dev/4-reasons-why-embedded-iot-learners-give-up-early) blog post, choosing incorrect development boards and incorrect wiring are two of four main reasons for quitting early. Beginners are prone to mistakes in board choice, and cycling through boards can prove to be costly. Other than the fact that different boards could require modified toolchains and setups.

One might say that "well we have simulators", and that's true. Though the issue with traditional simulators is that they are specific to a microcontroller, not necessarily providing something as close as possible to a full environment. Also when I say a full environment I mean things like programming language freedom of choice, wiring hardware interfaces, a wide variety of sensors/actuators, a wide variety of platforms/microcontrollers, and the ability to debug. This means that not only the device/uC at hand needs to be simulated, but also the surrounding components it connects to, the electrical circuits, and the physical environment it's in.

%%[substart]

Believe it or not, luckily, nowadays there are amazing solutions emerging to address much of the above. Though some solutions are in their early stages, the promise of providing full solutions without having physical hardware in hand is becoming more possible than ever. Along those lines, and without further ado, here are 8 websites every embedded developer must know:

### 1️⃣ [Wokwi](https://wokwi.com/) 


![Wokwi](https://cdn.hashnode.com/res/hashnode/image/upload/v1663488110890/22hShZ5Yz.png align="center")

I am a huge fan of Wokwi and am amused by how rapidly it's evolving to include more solutions. Wokwi is an online Electronics simulator that currently supports several popular platforms including several Arduinos, ESP32, and Rasberry Pi Pico among others. There is a wide variety of popular components to also select from. Wokwi also supports debugging and network connections. You can actually connect to WiFi and establish an MQTT or HTTP connection (Bluetooth on ESP is also being requested as a feature)! Not only that, but Wokwi also supports languages like MicroPython, CircuitPython, and Rust!

The awesomeness doesn't stop here. Users get to suggest and vote on what features come next. Take a peek at the [roadmap](https://wokwi.com/features) to see what features are coming next.

✅ **Pros**
- Rapidly evolving large library of popular platforms, sensors, and components.
- Multiple language support (uPython, Rust, and Arduino's C++).
- Multiple platform support.
- Supports GDB Debugging.
- Has a logic analyzer function.
- Supports networking applications.
- Free with optional club membership.

🚫 **Cons**
- Not working with real hardware 
- Restricted to the cloud platform (Can't use any IDE)
- I've realized that sometimes error messages don't show

### 2️⃣ [Planet Debug by MikroElektronika](https://www.mikroe.com/planet-debug)

![Planet Debug](https://cdn.hashnode.com/res/hashnode/image/upload/v1663488557195/260AuCqU3.png align="center")

Planet debug is an embedded hardware remote programming/debugging platform. With Planet Debug the user gets to interact with real hardware through a live stream as part of the Necto Studio IDE by Mikroelektronika. This allows users to view a hardware development working in real-time. Although really powerful, I've found the platform to be quite restrictive in terms of hardware and software where the IDE has a bit of a learning curve to get used to and navigate around. Also, when opening the IDE one has to reserve a specific platform with the device/sensor combination required so it's subject to availability. 

I've personally used Mikroelectronika products in teaching embedded before until eventually decided on changing. This was because I wanted to expose students to common industry environments and platforms other than sometimes facing features with paywalls. I see that probably Planet Debug is more fit for custom-targeted environments that do not mind/care about the restriction. As such, I personally won't recommend using planet debug for an embedded newcomer.

✅ **Pros**
- Get to work with real hardware w/ debug capability
- Several prewired component options
- High-quality video stream
- Good variety of sensors and components

🚫 **Cons**
- Restricted to Necto Studio IDE.
- Restricted to Microelektronica Platforms and Debug framework.
- Seems to support C programming only.
- Required device/platform combination is subject to availability.
- There is a bit of a steep learning curve getting used to the tools.

### 3️⃣ [Allhardware](https://all-hw.com/app/#/index)

![All Hardware](https://cdn.hashnode.com/res/hashnode/image/upload/v1663488437266/p-56iyn8e.png align="center")

Described as a dev board as a service platform, Allhardware provides remote access to real embedded platform hardware to program and debug. Allhardware has a similar approach to Planet Debug in that it allows remote access to hardware albeit with fewer restrictions. With allhardware you can choose your own IDE and program the platform of choice with your code over a remote secure connection. Additionally, allhardware provides a selection of common hardware platforms (mostly ST Microelectronics platforms) that you need to reserve and also subject to availability. Unfortunately, Allhardware gives access to programming a development board without any wired external components. That means Allhardware is good for running your code on different platforms/controllers but does not provide the capability to interface with external components (unless the board itself integrates some). Essentially, Allhardware seems to target CI/CD applications where an entity can verify its codebase across multiple devices. 

✅ **Pros**
- Closer to a real development set up only with remote hardware
- Remote hardware options are based on commercial platforms
- Can use a variety of popular IDEs

🚫 **Cons**
- Access is only for the development board, no sensors or actuators
- Video streaming quality isn't that great

### 4️⃣ [Arm Mbed Simulator](https://github.com/ARMmbed/mbed-simulator#readme)

![ARMmbedSimulator.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663488647423/JlgoSbLhG.png align="center")

I like to describe the ARM mbed simulator as a very restricted form of Wokwi. Arm Mbed runs only Mbed OS 5 applications using a single platform with fewer component options. The Arm mbed simulator provides access to a good amount of electronic components to connect and wire to the simulated hardware platform. The simulator also provides access to networking features like mbed-http, and the Mbed LoRaWAN stack. 

I've personally used Arm mbed simulator in learning environments and found it quite useful though with some caveats. The simulator does not always work smoothly and crashes quite often depending on what is being done in code. It did not provide a smooth experience for students at times as they had to deal with issues not directly related to their code.  Additionally, there is no debug support.

The ARM Mbed simulator has [cloud-hosted option](https://simulator.mbed.com/), however, it seems that it's being phased out in December 2022 for Docker image and native options. 

✅ **Pros**
- Good learning setup with preloadable demos using Arm mbed OS
- Supports networking applications
- Free

🚫 **Cons**
- No debug option
- Can experience crashes
- Restricted to Arm mbed OS
- Restricted to single-platform

### 5️⃣ [MicroPython](http://MicroPython.org) Online Simulator

![MicroPython Simulator](https://cdn.hashnode.com/res/hashnode/image/upload/v1663488712477/0E0K4fgJx.png align="center")

The MicroPython online simulator is probably best described as the ARM Mbed Simulator only restricted to MicroPython and the PyBoard. Additionally, only four types of peripheral external components are supported. The cool thing though is that an inline assembly option is available through the simulator. As such, the [MicroPython](http://MicroPython.org) online simulator is probably good only for test-driving MicroPython rather than building full applications.

✅ **Pros**
- Good learning setup with preloadable demos using MicroPython
- In-line assembly feature
- Free

🚫 **Cons**
- No debug option
- Can experience crashes
- Restricted to microPython
- Restricted to single-platform

### 6️⃣ [Labsland](https://labsland.com/en)

![LabsLand](https://cdn.hashnode.com/res/hashnode/image/upload/v1663488887112/KQpyGxBFx.png align="center")

LabsLand provides remote laboratories for educational purposes. Part of their offering includes, albeit very limited, embedded solutions. The solutions include the Arduino Uno and Intel/Altera DE1 and DE2 FPGA platforms. With the Arduino, there is a cool experiment where a robot can be controlled as well. In terms of usage though, the user is restricted to the LabsLand custom web-based development environment. Unfortunately, all of it is behind a paywall.

✅ **Pros**
- Provides an FPGA solution
- More interactive robot control experiment

🚫 **Cons**
Not free.
Very limited embedded offering
No debug option

### 7️⃣ [Sahara](https://saharacloud.io/)

![sahara](https://cdn.hashnode.com/res/hashnode/image/upload/v1663488965863/q00zwm-Z-.gif align="center")

Sahara a solution that seems to be promising but yet in its very early stages. Sahara seems to want to provide a hybrid of simulated and real devices in a single platform. It can be described as a Woki-Planet Debug hybrid. However, accessing real devices seems that it will be behind a paywall. While Sahara seems like a promising idea, it is still rough around the edges. The user interface is not too friendly and the device options are still very limited (only a single Arduino option).

✅ **Pros**
- Aims to provide a simulated/real device hybrid.

🚫 **Cons**
- User interface not that good.
- Very early stage, still needs quite some work.

### 8️⃣ [Makerchip](https://makerchip.com/) 

![MakerChip](https://cdn.hashnode.com/res/hashnode/image/upload/v1663489079148/LRF0TgIQg.png align="left")

Although Makerchip is specific to FPGA development and Verilog, I thought it's worth mentioning given how great of a resource it is. Makerchip is a cloud-based Verliog design environment that supports the emerging Transaction-Level Verilog standard (TL-Verilog). FPGA tools are probably famous for not being the quickest or most useable and also have many bugs. Makerchip on the other hand has quite a user-friendly interface with clear diagrams and built-in tutorials and courses that guide you along the way. There is also a tutorial on building a RISC-V core! Makerchip focuses on building and simulating designs, but doesn't deal with any external interfaces or target particular devices. 

✅ **Pros**
- Great guided learning platform for designing for FPGAs with TL-Verliog

🚫 **Cons**
- No other language options (Ex. VHDL)
- Does not target specific devices or simulate external components/interfaces

## Summary
If you'd ask me, my choice, for the time being, is Wokwi. Especially due to how fast it is evolving and the variety of choices it provides. While the promise of real remote hardware provided by other options is attractive, I don't see any living up to the needs of the community and especially beginners. Ideally, I personally would like to have access to an IDE and language of my choice, with remote hardware and several components. Sort of like Planet Debug divorced from all the tool restrictions around it. On the FPGA end, there also seems to be much potential for more solutions, especially given the cost of platforms.

## Conclusion
Doing embedded IoT without access to physical hardware would have sounded like a remote possibility not so long ago. Moreover, traditional emulator and simulator solutions are not general enough and don't cut it anymore. The good news is that several solutions have emerged to make embedded IoT without access to physical hardware a possibility. In this article, I provided 8 different resources that are doing a great job in that regard. 

Are there any platforms you use that I missed? Have you used any of the above and have an experience you'd like to share? Share your thoughts in the comments below 👇.

%%[subend]