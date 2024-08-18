---
title: "5 Must Know Plug n Play Platforms to Relieve Embedded IoT Wiring Headaches"
datePublished: Mon Jun 13 2022 15:15:23 GMT+0000 (Coordinated Universal Time)
cuid: cl4cvn9r1073jjpnv4kbe20nh
slug: 5-must-know-plug-n-play-platforms-to-relieve-embedded-iot-wiring-headaches
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1655132828717/7DxNOX1UM.png
tags: beginner, iot, hardware, embedded

---

Probably one of the most daunting debugging aspects for embedded IoT learners is wiring. Especially those new to electronics, they often think that they've wired everything correctly only to discover after hours of debugging that one wire was placed incorrectly. This includes that sometimes a learner doesn't have enough knowledge to configure the circuit of a particular sensor or device properly. Additionally, when prototyping, and especially increasing the number of circuits, wires become hard to manage. Any wire can get loose and things start going out of whack. 

![wiresmeme.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1654755921330/M2cxvG-3a.jpeg align="center")

Is there a better alternative? Absolutely! I typically recommend plug n play platforms for prototyping and even learning. The idea behind plug n play platforms is that they provide a standard interface for a large variety of sensors and actuators. This relieves the user from worrying about wiring and also designing the electronics. It eliminates a lot of the guesswork and saves a lot of time in debugging and prototyping. 

The idea behind all of the plug n play platforms is that they provide a modular interfacing approach for microcontroller development boards. This is done by providing some sort of standard interface enabling connection to a wide variety of sensors and actuators. The standard interfaces would typically support at a minimum GPIO, analog, and some sort of serial communication.

## Plug n Play Platform Structure

For all plug n play platforms, the structure is similar and can be broken down into four layers as shown below.

![Plug n Play.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1655013496955/797PdbVki.png align="center")

The adapter that connects to the development board provides the compatible receptacles to connect the upper modules. Sometimes also, these modules are connected through a standard wiring system to the sensor and actuator modules. Additionally, some interface standards allow connection directly to the board or adapter without wires. The adapter is also listed as optional since there are some boards that directly integrate the receptacles. 

Below I list the most popular plug n play platforms available in the market and mention the pros and cons I found in each.

## 1) Seeed Studio Grove

![grove_Kit.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1654933761974/zEiME7WFg.jpeg align="center")


**Pros**
- Adapter support for Arduino and Rasberry Pi. Has also custom boards with adapter-free solutions.
- 100's of sensor and actuator options.
- Tons of guides and resources.

**Cons**
- Serial interface options do not support SPI (Only I2C)

**Where to Start?**
Starter kits available on [SeeedStudio website](https://www.seeedstudio.com/category/Grove-c-1003.html) or from the Seeed Studio Store on [Amazon](https://amzn.to/3aVVwZC).

## 2) Mikroelektronika MikroBUS

![arduino-uno-click-shield-large_default-42x.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1654934240186/C8HTEyW3E.jpeg align="left")

**Pros**
- Probably has the widest variety of interface options through "click boards" (>1000 click board options).
- Offers the widest variety of adapters for Arduino, RasberryPi, Beaglebone, STM Discovery, and TI Launchpad among others. Full list [here](https://www.mikroe.com/click/click-shields).
- "Click Boards" connect directly without wires.
- Direct MikroBUS interface supported by various supplier development boards in the market. Full list [here](https://www.mikroe.com/mikrobus)

**Cons**
- Mikroelectronika development boards themselves are large and costly.
- Adapters and boards that integrate MikroBUS do not offer many Click Board interfaces. Probably a maximum of 2-4 click boards can be connected in parallel.

**Where to Start?**
Identify the board or adapter needed. If you want a board with an integrated Mikrobus interface you can select an appropriate board [here](https://www.mikroe.com/mikrobus).
Otherwise, for standard adapters, clickboards, and mickroelektronika development boards you can directly go to the [Mikroelectronika store](https://www.mikroe.com/shop).

## 3) NightShade Electronics Treo

![TREO-EVALKIT-ARDUINO.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1654933950853/X6hxnW9eC.jpeg align="center")

**Pros**
- Adapter support for Arduino and Rasberry Pi. 
- Mechanical mounting is available for elegant design solutions.

**Cons**
- Limited module options for interfacing
- Interface connector is a bit heavy with wires

**Where to Start?**
Available directly through the [Nightshade website](https://nightshade.net/treo-system/).

## 4) Osoyoo Magic I/O Shield

![Magic-Board.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1654933864156/uTgBFXmpB.jpg align="center")

**Pros**
- Utilizes standard Arduino R3 interface utilized for shields.
- Has built-in motor driver targeting robotics designs.

**Cons**
- Design is not too polished.
- Adapter solutions are limited to Arduino.
- Does not utilize one standard wire interface.
- Limited application space.
- The adapter comes as a part of a robot car kit and does not seem to be sold separately. 

**Where to Start?**
Kits are available directly through the Osoyoo store on [Amazon](https://amzn.to/3Qkzi3p). 


## 5) Digilent Pmod


![pmod.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1654962540665/UmWqLXLGl.jpg align="center")

**Pros**
- Has adapter solutions for Arduino R3 interface, Rasberry Pi, and NI myRIO.
- Interface boards (modules) connect directly without wires.

**Cons**
- Pmod Ardunio compatible shield is being retired.
- Considered a more specialized solution for scientific applications and also Digilent Xilinx-based FPGA boards.

**Where to Start?**
Adapter boards and Pmods are all available directly through the [Digilent store](https://digilent.com/shop/all-products/).

## How to Choose?
Given the above, which platform I would use depends on what I want to do. If I was a beginner Grove is probably the best fit given the cost, options, and resources available, with the Nightshade Treo coming right after. If I were looking for a teaching solution then  Mikroelectronika would be a good choice to pick up for a lab. Finally, the others like the Osoyoo and Pmod would probably be a better fit if I were working on more specialized applications.

## Conclusion
Hardware wiring in embedded IoT is probably one of the most annoying aspects for a beginner or even a prototype engineer. The accumulation of wires makes for a less portable setup and one vunerable to many mistakes. Gladly, the plug n play solutions available in the market and listed in this post can ease much of that pain. Are there any platforms you use that I missed? Have you used any of the above and have an experience you'd like to share? Share your thoughts in the comments below 👇.  If you found this useful, make sure you subscribe to the newsletter [here](https://subscribepage.io/apollolabsnewsletter) to stay informed about new blog posts.
