---
title: "Is the Microprocessor Device Dead?"
datePublished: Mon May 16 2022 19:35:51 GMT+0000 (Coordinated Universal Time)
cuid: cl394mdpr02v4hcnv860levrt
slug: is-the-microprocessor-device-dead
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1652729376472/22eHSkaBr.png
tags: iot, hardware, embedded

---

Maybe a question to ask first instead is what defines a microprocessor device? It actually comes as a surprise to me how in recent times there is still debate or even question about the definition of processing devices. There are various terms out there that are often used interchangeably, albeit incorrectly as each term refers to a different thing. This includes the three main terms; microprocessor, microcontroller, and system on chip (SoC). Now what needs to be clear is that in between the three terms the naming is all in reference to a processing device and what it entails on a single die inside a single package or integrated circuit (IC). In other words, the terms define what is part of a device IC that resembles something like shown in the figure below 👇.


![Untitled design.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652728951603/mFksVdLQt.png align="center")

In this post, I will be explaining the differences between the three terms and we’ll see why I believe that microprocessor devices are headed toward extinction, although as it seems the term used incorrectly might live on for a while.

%%[substart]

## The Microprocessor

A microprocessor contains a CPU were inside of it live the processing elements including the control unit (CU), the Arithmetic logic unit (ALU), and the register file. Additionally, sometimes cache memory is also integrated within the same device. Although the terms CPU and microprocessor are often used interchangeably, it is not always correct. This is because some microprocessors can have multiple CPUs, which makes them multi-core processors. Following that, microprocessor devices do not include blocks like main memory (Ex. DRAM), I/O interfaces/peripherals, graphics unit, or permanent memory in the same IC. Any additional functions need to be added via external devices that connect to the microprocessor device. The figure below shows an example of the contents of a microprocessor device. 


![uP.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652728119037/nvtlNglQP.png align="center")

What we need to know also is that microprocessors are usually high-performing processing devices that consume significant power. Microprocessors typically target general-purpose type applications such as desktop computing. I would say that probably all older Intel devices (Ex. Pentium processors) and even some recent ones fall under the microprocessor category. 

## The Microcontroller

The microcontroller essentially emerged from the application needs in the world of embedded systems and is crucial to the operation of IoT edge devices. A microcontroller device actually contains a microprocessor as part of it in addition to other elements that allow it to operate without reliance on any memory or external interfacing elements. At a minimum, a microcontroller contains a CPU (Ex. ARM Cortex-M), permanent memory (Flash ROM), temporary memory (SRAM), general-purpose I/O interfaces, timer/counter/PWM peripherals, serial communication interface, and some form of analog-digital interface. Compared to a microprocessor, a microcontroller has much less processing performance and memory. Microcontrollers can also be battery operated as they consume much less power. The idea is that for embedded systems, the applications are mission-specific and not general-purpose in nature. This means that several cores along with gigabytes of memory and a high-performing core with deep pipelines are not necessary. On the contrary, it is quite common to have a microcontroller with flash memory less than 512kB or less and a single-core processor running on less than a 100MHz clock that consumes current on the level of several milliamps. Additionally, microcontrollers are not manufactured with the latest process technology. This is not necessary as processing power is not the main goal and it also makes controllers more reliable and more cost-efficient. 

There are many examples of microcontrollers to mention but some popular ones include the ST Microelectronics STM32 devices, Atmel (Now Microchip) AVR devices made popular by the Arduino ecosystem, and Texas Instruments MSP controllers. The figure below shows an example block diagram of what a microcontroller device incorporates.


![uC.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652728160523/keat5SzKD.png align="center")

## The System on Chip (SoC)

Often you might find microcontroller manufacturers refer to microcontrollers as SoCs. This is a sort of abuse of the term in my opinion and where much of the confusion comes from. The confusion behind the SoC term in my opinion has some to do with how manufacturers want to market their devices. The rationale behind the usage is that a microcontroller can house a complete "System" on a single chip. Although true operationally, what SoC and microcontroller devices house from the inside is different. SoCs target a wider variety of applications compared to microcontrollers including embedded systems, compute modules, and desktop computers among others. 

In simple terms, an SoC can be thought of as a microcontroller on steroids. This means that an SoC still houses many of the same functions a microcontroller does, however, it has more powerful processing (often multi-core), and sometimes additional hardware acceleration blocks (ex. GPU or FPGA fabric), and/or integrated radio (ex. WiFi). Though in some cases, SoCs don't necessarily integrate ROM and RAM memory. Often SoCs do require an operating system that microcontrollers do not. A great example describing SoCs is how the Intel processors evolved. In the past, Intel used to manufacture microprocessors that were placed in a separate chip on the motherboard side by side with a chipset. The chipset is a device that integrates many of the I/O peripheral functions that the microprocessor requires. Later on, in many  Intel devices, the chipset and the microprocessor have been integrated on the same die and placed in a single device that is referred to as an SoC. More information can be found [here](https://www.intel.com/content/www/us/en/support/articles/000056236/intel-nuc.html).

Good examples of SoC include the [Xilinx Zynq](https://www.xilinx.com/products/silicon-devices/soc/zynq-7000.html), [Apple M1](https://en.wikipedia.org/wiki/Apple_M1), and the [Nvidia Tegra X1](https://en.wikipedia.org/wiki/Tegra).


![soc.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652728193961/Reaii4bPC.png align="left")

## Comparision Summary Table

Here I summarize the differences between the terms in a single table for reference.

|        Parameter        | Microprocessor | Microcontroller | System on Chip |
|:-----------------------:|:----------------:|:---------------:|:----------------:|
|     Processing Power    |      High      |       Low       |   Medium/High  |
|    Power Consumption    |      High      |       Low       |   Application Dependent   |
|     Number of Cores     | Single or More |  Mostly Single  | Single or More |
|     Integrated ROM?     |       No       |       Yes       |  Most of the Time  |
|     Integrated RAM?     |       No       |       Yes       |  Most of the Time  |
|  Integrated Peripherals? |       No       |       Yes       |       Yes      |
|  Acceleration Hardware  |       No       |        No       |       Yes      |
|  Cost | High | Low | Medium | 
|    Example Device(s)    |   [Intel Pentium](https://en.wikipedia.org/wiki/Pentium)   |  [TI MSP430](https://en.wikipedia.org/wiki/TI_MSP430) [STM32](https://en.wikipedia.org/wiki/STM32)                |   [Xilinx Zynq](https://www.xilinx.com/products/silicon-devices/soc/zynq-7000.html), [Apple M1](https://en.wikipedia.org/wiki/Apple_M1), [Nvidia Tegra X1](https://en.wikipedia.org/wiki/Tegra)             |

## Conclusion

From what we have seen in this post is that microprocessors in their original pure form are probably not as common anymore, or as the title of this post suggests are almost dead. From what is obvious, IC technology is headed towards more integration into a single package. This means that we are probably going to end up with only SoCs and microcontrollers for a while before another term is keyed in. Additionally, while interchangeable term usage for non-technical individuals might not be a big deal, it is for an individual in the field. The choice between one device or another makes a huge difference in what you are trying to achieve technically. Is this what you thought the differences are? Or did you have a different understanding? Share your thoughts in the comments below👇.  

%%[subend]