---
title: "7 Step Learning Path for Embedded IoT Beyond Arduino"
datePublished: Mon May 30 2022 12:20:12 GMT+0000 (Coordinated Universal Time)
cuid: cl3sp82fd00mt0tnvbtue611f
slug: 7-step-learning-path-for-embedded-iot-beyond-arduino
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1653809698097/pNuE-ja-s.png
tags: arduino, iot, embedded

---

It is common for embedded IoT enthusiasts to start out tinkering with platforms like Arduino and maybe to some extent other platforms like ESP or Raspberry Pi. I find it all too common that at a certain point when the learner would have found their footing that they wonder what should they do next to expand their knowledge in the field. What I've noticed, is that probably compared to web technologies, learners don't find as much guidance online on the learning path they should follow. If anything, more common than not, you would find individuals on forums arguing about what the best path should be. 

While tinkering platforms are good entry-level vehicles for embedded IoT, they certainly abstract away a ton of fundamentals. You cannot expect to jump from a tinkering platform to a specialized industry all at once. In this post, I first tackle why tinkering platforms like Arduino might not deliver all the necessary fundamental knowledge to build expertise in embedded IoT and then I cover what knowledge you need to build to grow beyond tinkering platforms. 

To make it clear, I'm not trying to say that you cannot use Arduino code in production. However, if an individual wants to, understanding the low-level detail becomes necessary to understand the tradeoffs. 

In this post also, I focus more on embedded software development aspects and assume that the learner already has a certain knowledge level in electronics. 

%%[substart]

## Why Can't I Always Stick with Arduino?

![wait what.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1653846106233/tcWsk7heu.jpeg align="center")

From a software perspective, you have to keep in mind that platforms like Arduino are extremely abstracted. This is done by adding software layers on top of lower-level code to remove a lot of the low-level detail. Additionally, Arduino platforms include low-level firmware to make their application software programmable on several devices. Rightly so, the idea behind it is to ease the learning curve for embedded hardware beginners. Though for somebody wanting to eventually create production-grade devices things to keep in mind include:

- **Does the code overhead impact my program size?** embedded devices use low-cost microcontrollers that often do not have lots of memory space. Given the application, you have at hand, you'll need to understand if the Arduino library and firmware overhead might be adversely affecting your application.
- **Does the Arduino code give me access to all the microcontroller features I need?** Existing Arduino libraries do not necessarily expose the full power of the underlying hardware. Microcontrollers often come with special features that allow for faster/better performance. Arduino code does not necessarily always give the flexibility to fully customize all microcontroller-specific features. If you need to squeeze out some specifics then you'd need to understand how to access the lower levels of hardware.

While I'd like to focus more on embedded software, another important aspect to keep in mind is the hardware. Arduino hardware is quite generic in nature, for production, the among the things to consider include:
- Arduino hardware adds component overhead that is not used in production. This incurs additional hardware costs in a cost-sensitive mass-produced industry. [This](https://predictabledesigns.com/from-arduino-prototype-to-manufacturable-product/) blog post gives a nice overview of the Arduino hardware extras. 
- The Arduino hardware (PCB and components soldered on it) is not qualified for all application areas/industries. Often depending on the industry you are targeting for your product, you would have to obtain qualified components for that industry. Additionally, the PCB itself sometimes needs to be qualified for the industry/area of usage like consumer electronics, automotive, industrial, or what have you. This makes sure that the hardware itself can withstand the environment it will be exposed to.

## So What Makes Platforms Like Arduino Special?

![im special.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1653846610152/hJK7nZ5oM.jpeg align="center")
I personally think that the greatness of platforms like Arduino is in how they managed to get many people interested in hardware. It's already established that Arduino eases the entry-level curve for beginners that don't have much-embedded hardware background. If anything, learners develop a sense of building logic under embedded constraints. This has numerous advantages later when developing at a lower level. Last but not least, Arduino is also a great vehicle for quick prototyping of ideas even for experienced professionals.

## So Where Do I Start?

![let the fun begin.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1653846346875/Gl862h6k7.jpeg align="center")
In the end of the day, it all comes down to fundamentals. Some tend to look to far ahead towards what they want to specialize in and what industry they want to work in. While its good to have aspirations where you want to go, understand that where ever you end up, the fundamentals are the same. As a result, it would be worthwhile focusing on building upon embedded fundamentals as they would apply to any industry you are interested in like automotive, aerospace, consumer electronics, or what have you. 

So to take a structured approach, let's revise what facts we know about embedded and then address how we can build up our fundamentals beyond tinekring platforms. Now embedded IoT applications are by definition mission-specific, part of a bigger network, and driven by constraints like power, cost, and size among others. As a result, this leads embedded IoT applications to: 

- Leverage system programming languages like C and C++
- Prioritize the use of microcontrollers  
- Prioritize Bare-metal development unless you really need an RTOS 
- Have some connection to a network (wired or wireless)

Each one of these points contributes to the body of knowledge you need or fundamentals you need to acquire to become  Let's look at each point and see how it contributes.

### Systems Programming Languages
Systems programming languages are often compiled languages that are really efficient and generate really optimized/fast code. Systems programming languages provide the necessary control that a developer needs to both configure and squeeze out the performance they need from their system. While there are some other systems languages like Rust and micropython on the rise, in the embedded field right now, the most common two languages utilized in the industry are C and C++. Most Arduino users often are familiar with the syntax as Arduino already leverages C++. As a result, it might be good to start out with a C or C++ course to build a full picture of the language. This also helps a learner understand the meaning behind a lot of the obscured code in Arduino. If anything, it would be important to gain knowledge in areas like pointers, structs, and dynamic memory allocation at a minimum. Going further, knowledge in certain data structures like queues and ring buffers would also be beneficial.

Some good resources for learning C or C++ include the following:
- The [Introductory C Programming Specialization](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=spzn%3A2mb84fnoEealcA6o860tEA&u=https%3A%2F%2Fwww.coursera.org%2Fspecializations%2Fc-programming&intsrc=PUI2_9419) on Coursera is a well reviewed course if you are interested in C.
- The [Coding for Everyone: C and C++ Specialization](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=spzn%3AlxXNdeGQEeq1Mwqrp-pWyQ&u=https%3A%2F%2Fwww.coursera.org%2Fspecializations%2Fcoding-for-everyone&intsrc=PUI2_9419) on Coursera as well that gives you a background in both C and C++/

After expanding your C or C++ knowledge, I also hugely recommend the [A Dive into Systems](https://diveintosystems.org/book/preface.html) free online book. I find this book to give a great overview of computer systems from top to bottom. It is a great precursor to all forthcoming topics. 

### Use of Microcontrollers
Microcontrollers (MCUs) emerged from the need in the embedded industry. MCUs pack several features and peripherals that are targeted for embedded use cases. If you are not too familiar with what an MCU entails or the difference between an MCU and a microprocessor or SoC, I urge you to read my other blog post [here](https://apollolabsblog.hashnode.dev/is-the-microprocessor-device-dead). Common MCU peripherals include at a minimum general-purpose I/O (GPIO), analog to digital converters (ADCs), timers/counters, and some sort of serial communication. Additionally, MCUs pack special mechanisms like interrupts or low power modes. What you also need to know is that MCU variety is HUGE. There are so many companies (probably tens to hundreds) manufacturing so many different flavors of MCUs. So in terms of fundamental knowledge, what does this require us to know/learn? At a minimum you need to understand:

- The mechanics of embedded CPU architectures and their instruction set. You can pick up one among several architectures including ARM Cortex-M, RISC-V, and AVR among others.
- The basic theory behind common peripherals. For example, how does a timer or counter work? how does an ADC work? how does GPIO work? This helps you understand what you can change in a certain peripheral to achieve what you want. Furthermore, you will see that there are several ways to achieve the same goal.
- How to manage and configure microcontroller and CPU core special features. This includes concepts like interrupts and low power operation.
- How to configure microcontroller peripherals and their features to meet your application requirements or even optimize your application. 
- How to debug code at the low level. This also entails understanding debug toolchains and how to configure them.

In summary, you need to be able to understand a microcontroller system inside out and how to configure it at a low level through registers. This requires knowledge of the microcontroller architecture and the fundamentals of operating different peripherals. This also means that based on the above knowledge, ultimately your goal should be to be able to grab any MCU datasheet or reference manual to extract the information you need to configure, program, and debug that MCU. Other things this knowledge is beneficial for include designing hardware for your MCU and also picking an MCU for your application (understanding performance, power, and cost tradeoffs).

In terms of resources, I've found the two-part course [Embedded Systems - Shape The World: Microcontroller Input/Output](https://tidd.ly/3wVIdzn) followed by the [Embedded Systems - Shape The World: Multi-Threaded Interfacing](https://tidd.ly/3wWdNNG) by Jonathan Valvano to cover most of what you need here. There are also the courses offered by ARM in the [Embedded Systems with ARM professional certificate](https://tidd.ly/3lRu80R) on edX, though these courses might be considered still a bit abstracted in some aspects. 

### Bare-metal Development & RTOS
Embedded systems prioritize bare-metal development. In bare-metal applications, the application code executes directly on the hardware device (the MCU). In other words, the application code has direct access to silicon (hardware) without any intermediaries (ex. operating system) as the figure below shows. 

![baremetal.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1653654236015/tPv0L9I59.png align="center")

As might be obvious already, bare metal development involves the least overhead and allows developers to optimize resources. One might wonder though how the code is managed as the application becomes more complex. Some sort of scheduler becomes necessary, though it would not be necessary to resort to an operating system all at once. There are some simpler scheduling approaches that can be adopted.

Up to this point, a learner would have built enough foundational knowledge for many different application areas. A learner could already expand on this knowledge to specifics of a certain industry. Though going further to build a comprehensive foundation is important in my view. 

In some cases, as application and hardware resources grow in complexity, resorting to an operating system becomes necessary. Typically it would be a real-time operating system that can provide some sort of timing guarantees for executing tasks. As such, there would be some sort of kernel underlying the application with integrated drivers that talk to the low-level hardware as the figure below shows.

![rtos.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1653654958286/jUZt8Jf9_.png align="center")

Regarding resources for Bare Metal, here its important for one to familiarize themselves with scheduling techniques. I've found the book ["An Embedded Software Primer"](https://amzn.to/3a2STET) by David E. Simon quite useful and easy to read. Here the data structure background from the first point becomes also essential. If you prefer separate resources on learning data structures there's the [Data Structures by UC San Diego Course on Coursera](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=crse%3AyOZEQ3lwEeWb-BLhFdaGww&u=https%3A%2F%2Fwww.coursera.org%2Flearn%2Fdata-structures&intsrc=PUI2_9419), or the full [Data Structures and Algorithms Specialization](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=crse%3AyOZEQ3lwEeWb-BLhFdaGww&u=https%3A%2F%2Fwww.coursera.org%2Flearn%2Fdata-structures&intsrc=PUI2_9419), there is also the [Data Structures and Algorithms Using C++ on edX](https://tidd.ly/3PKAdtR) which focuses particularly on C++. Finally, [The Algorithms](https://the-algorithms.com/) is a great resource beyond these courses that provide open source code in several languages for popular data structures and algorithms.

The knowledge a learner would have built thus far would give enough idea of what RTOS scheduling might look like. Though, the choice to deploy an RTOS gets into more than just managing the scheduling of tasks. Here, it would be beneficial for a learner to practice deploying a lightweight RTOS like [FreeRTOS](https://www.freertos.org/) or [Mbed OS](https://os.mbed.com/mbed-os/). You would find the [Shawn Hymel Introduction to RTOS YouTube Playlist](https://youtube.com/playlist?list=PLEBQazB0HUyQ4hAPU1cJED6t3DU0h34bz) quite useful here.

Beyond a lightweight RTOS, it is also common nowadays to find Linux utilized in embedded single board computers. This is for applications that become more general-purpose in nature and require many resources. For such applications, learning how to build a custom Linux image is essential. There is also a [Shawn Hymel Introduction to Embedded Linux YouTube Playlist](https://youtube.com/playlist?list=PLEBQazB0HUyTpoJoZecRK6PpDG31Y7RPB) that is very useful and beginner-friendly. The playlist, however, requires that you have some foundation in Linux command line. I've personally found the [Introduction to Linux](https://training.linuxfoundation.org/training/introduction-to-linux/) free training course by the Linux Foundation to be quite useful.

### Network Connection
As embedded devices are often part of a larger network, early on, depending on the industry a device is deployed, devices would often be part of a specialized wired network. This didn't necessarily mean that the devices in the network would be exposed to the outer world (ex. the internet/cloud), but rather talk to each other in isolation (Ex. in a car). Later on, as IoT applications became more prevalent, wireless networking also became more common (Ex. WiFi, Bluetooth). As such, in order to build foundational knowledge in embedded, it would be worth a learner's time to acquire some basic networking foundation. At a minimum, understand the different layers of a networking model and the different protocols at hand.

Here a learner can pick up a standard computer networks course or even one specialized in embedded IoT. The [Internet of Things micromasters program](https://tidd.ly/3wYzUCY) on edX has a variety of courses that cover IoT networking foundations. Additionally, an example of a standard computer networks course is the [computer communications specialization](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=spzn%3AaffCE3liEee7WwqcQWCWNA&u=https%3A%2F%2Fwww.coursera.org%2Fspecializations%2Fcomputer-communications&intsrc=PUI2_9419) on Coursera.

## 7 Step Learning Path Summary

To summarize the above, one's learning path beyond Ardunio consists of the following:

**1. Gather in-depth knowledge in C or C++ including topics like structs and pointers** 👨‍💻

 📚 **Resources:** [The Introductory C Programming Specialization on Coursera](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=spzn%3A2mb84fnoEealcA6o860tEA&u=https%3A%2F%2Fwww.coursera.org%2Fspecializations%2Fc-programming&intsrc=PUI2_9419), or for both C and C++: [The Coding for Everyone: C and C++ Specialization on Coursera](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=spzn%3AlxXNdeGQEeq1Mwqrp-pWyQ&u=https%3A%2F%2Fwww.coursera.org%2Fspecializations%2Fcoding-for-everyone&intsrc=PUI2_9419) 

**2. Gather Knowledge of Data Structures and Algorithms** 🧮

 📚**Resources:** [Data Structures by UC San Diego Course on Coursera](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=crse%3AyOZEQ3lwEeWb-BLhFdaGww&u=https%3A%2F%2Fwww.coursera.org%2Flearn%2Fdata-structures&intsrc=PUI2_9419), if interested there is a full [Data Structures and Algorithms Specialization](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=crse%3AyOZEQ3lwEeWb-BLhFdaGww&u=https%3A%2F%2Fwww.coursera.org%2Flearn%2Fdata-structures&intsrc=PUI2_9419), there is also the [Data Structures and Algorithms Using C++ on edX](https://tidd.ly/3PKAdtR). Finally, [The Algorithms](https://the-algorithms.com/) is a great resource beyond these courses that provide open source code in several languages for popular data structures and algorithms.

**3. Build a Computer Systems Knowledge Background** 💻
 
 📚**Resources:** The ["A Dive into Systems"](https://diveintosystems.org/book/preface.html) free online book provides very digestable, beginner-friendly material.

**4. Understand Microcontrollers at a Low Level (MCUs)** 🤖
 
 📚**Resources:** The two-part course combining the [Embedded Systems - Shape The World: Microcontroller Input/Output](https://tidd.ly/3wVIdzn) followed by the [Embedded Systems - Shape The World: Multi-Threaded Interfacing](https://tidd.ly/3wWdNNG) course. There is also the [Embedded Systems with ARM professional certificate](https://tidd.ly/3lRu80R) on edX, though it might be considered abstracted in some aspects. 

**5. Learn Bare Metal Software Design Principles and Scheduling Techniques in Embedded** ⠞

 📚**Resources:** Resources from point 4 might cover some of this knowledge, though the ["An Embedded Software Primer" Book](https://amzn.to/3a2STET) by David E. Simon gives a more complete picture with example code as well. The book also provides background into the underlying workings of an RTOS.

**6. Learn deploying an RTOS and/or Embedded Linux** ⌨️

 📚**Resources:** The [Shawn Hymel Introduction to RTOS YouTube Playlist](https://youtube.com/playlist?list=PLEBQazB0HUyQ4hAPU1cJED6t3DU0h34bz). The [Shawn Hymel Introduction to Embedded Linux YouTube Playlist](https://youtube.com/playlist?list=PLEBQazB0HUyTpoJoZecRK6PpDG31Y7RPB). For building up a foundation in Linux command line there's the [Introduction to Linux](https://training.linuxfoundation.org/training/introduction-to-linux/) course by the Linux Foundation.

**7. Pick up Computer Networks Knowledge** 📡

 📚**Resources:** The [Internet of Things micromasters program](https://tidd.ly/3wYzUCY) for specialized embedded IoT knowledge. In addition, the [computer communications specialization](https://imp.i384100.net/c/3405914/1242836/14726?prodsku=spzn%3AaffCE3liEee7WwqcQWCWNA&u=https%3A%2F%2Fwww.coursera.org%2Fspecializations%2Fcomputer-communications&intsrc=PUI2_9419) for general computer networks knowledge.

## Conclusion
Embedded IoT learners that start out with Arduino often find themselves lost navigating their path beyond Arduino. Most learners have aspirations to eventually build expertise for a certain embedded application area or industry like automotive, aerospace, industrial, or what have you. This makes things even more confusing as a learner starts seeing all these different job requirements and terms they're not familiar with that make their heads spin. The thing is, no matter what industry the learner eventually picks, it all comes down to one thing, **foundations**. Beyond that, the learner can pick specialization areas where they can build expertise. In this post, I lay out 7 steps every Arduino learner should take to build such foundations to enable the next step in their career.  Have any questions? Are there any other topics you think a learner should cover? Share your thoughts in the comments below 👇. 

%%[subend]