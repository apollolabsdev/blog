---
title: "4 Reasons Why Embedded IoT Learners Give Up Early"
datePublished: Mon Jun 20 2022 18:19:18 GMT+0000 (Coordinated Universal Time)
cuid: cl4n2arho01hqvenve94iek58
slug: 4-reasons-why-embedded-iot-learners-give-up-early
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1655704275084/ijaS1j9p8.png
tags: iot, beginners, embedded

---

Embedded IoT is considered a relatively challenging field since it involves both software and hardware. I would say that giving up early is often due to accumulated bad habits. Unfortunately, avoiding such habits is something that I don't see emphasized enough when in embedded systems courses. Often courses put too much focus on how to do things without trying to introduce a proper structure. The unstructured approach might be ok, and probably necessary, for a beginner, but as soon as complexity grows a tad, lots of things start going wrong leading to many frustrations and eventually giving up. Based on my experience working with and teaching embedded, I noticed recurring patterns that lead to giving up early. Those patterns fall under 4 main categories as follows:

## 1. **Code without Design:** 

![Code Design](https://cdn.hashnode.com/res/hashnode/image/upload/v1655704722653/JV-ZK6DD1.jpg align="center")

Beginning embedded developers, once presented a problem, might always be tempted to hook up a circuit and start coding right away. While this approach might work with simple problems and experimenting, going forward, it does not bear much fruit (hello interrupts!). Moreover, I personally have seen that most developers start getting stuck on issues where they fail to distinguish whether the source of the problem is a flaw in the program logic, the code itself, or even the hardware. Also when debugging, the developer is not necessarily sure what to look for. This ends up frustrating newcomers quite quickly and leads to quitting early. Having a framework to build on will help developers tackle problems with confidence and will make a world of difference.  

In the context of developing applications in the industry. Unless the idea is extremely simple, seasoned engineers do not usually just start to code immediately. The process typically can be broken down into two steps:

1. Code Design
2. Code Implementation 

As you grow into this mindset you will find that doing this will particularly make application development much easier. The idea is that youd want to separate the logic that goes behind the program from how it’s implemented in a certain programming language. As such this means that when you are doing application design you will be developing the logic behind the program. This can be done in many forms graphically or otherwise. In the implementation part, you will focus on how to implement the design steps in the language of your choice. 

If you take a step back and think about this briefly, you start to notice that by taking this approach, while step 1 is programming language agnostic, to a certain level can be device agnostic, or even at least combine a family of devices under it.

> Software engineers leverage a variety of tools for code design that are both graphical and non-graphical to help model the behavior of an algorithm/system. An example of a non-graphical tool that is quite common is [pseudo-code or algorithmic step definition](https://www.khanacademy.org/computing/ap-computer-science-principles/algorithms-101/building-algorithms/a/expressing-an-algorithm). For graphical tools, there are finite state machines, flow charts, and sequence diagrams among others. One might think then when do I use which? The answer is that it’s really up to the developer. Whichever tool seems more appropriate to apply in the context of the problem is the one that the programmer would choose. In some cases even for the same algorithm, a programmer can combine different tools to describe what the algorithm should do. The interested reader can look into a popular modeling tool referred to as the [Unified Modeling Language (UML)](https://www.uml.org/what-is-uml.htm) for more examples of modeling diagrams.

To get one used to this approach, I would encourage one to practice with code they already built and think about how to model it. Start with something as simple as the Blinky program and build up from there.

On a side note, I'm a solid believer that the strength of coders is not in how good they are in a language, but rather in their ability to design complex algorithms. As is obvious nowadays, new programming languages pop up quite so often that it's hard to keep up. As a result, having the strong ability to design algorithms is more crucial than building up a list of programming languages you are familiar with.

## 2. Choosing an Unsuitable Development Board

![Development Boards](https://cdn.hashnode.com/res/hashnode/image/upload/v1655705011245/WfPb4Pyrr.jpg align="center")

A beginner taking some course or starting out would probably be given the board they need to work on. Alternatively, it is common for many beginners to start out with Arduino Unos. After gaining some knowledge and throwing a developer into the wild to create their own projects, things can get really confusing. An embedded developer trying to purchase a board can get really unsettled by the number of options presented to them. There are many choices of which many seem similar. Sometimes after many hours of searching and finally purchasing, one might discover that the board acquired does not meet their needs. Again, leading to early frustration. Frankly, the number of choices out there makes selecting a board confusing even for those experienced in embedded. 

While this can be attributed to part of the learning experience, you do not want to keep on wasting your money on things that don't work out. To mitigate this, you need to adopt a structured way to select a board. This simply starts by identifying in as much detail as possible what you want to achieve. Check out my other blog post [3 steps for choosing a development board](https://apollolabsblog.hashnode.dev/learning-embedded-iot-3-steps-for-choosing-a-development-board) for more detail.

## 3. Incorrectly Wiring Circuits

![Wiring Board](https://cdn.hashnode.com/res/hashnode/image/upload/v1655705143248/Dlp-nQTBM.jpg align="center")

I know this is probably easier said than done. I've seen a lot of students/learners wire a circuit incorrectly and when things don't work out they would question almost everything except for their own wiring. I've seen this happen far too often it's not even funny. This involves cases where students/learners go on wild tangents blaming the underpinnings of breadboards, wires, and even microcontrollers but not their own circuits. 

The idea is to eliminate the guesswork here and have more confidence in one's circuits, especially when scaling. Here I typically recommend the usage of plug n play platforms for beginners. A list of platforms I recommend can be found in my blog post [5 Must Know Plug n Play Platforms to Relieve Embedded IoT Wiring Headaches](https://apollolabsblog.hashnode.dev/5-must-know-plug-n-play-platforms-to-relieve-embedded-iot-wiring-headaches).

## 4. Incorrect Debugging Approach/Skipping Debugging

![bug.jpg](https://cdn.hashnode.com/res/hashnode/image/upload/v1655705720505/QxSG643cT.jpg align="center")

One of the most frustrating parts in embedded is figuring out the source of the problem when things don't work. It's more challenging than regular software debugging because the source of the issue can be either hardware or software. When I'm consulted about an issue in a system, typically my first go-to advice is to isolate the source of the issue, hardware or software. Additionally, without the proper tools to do that, you would only be guessing, essentially failing to isolate the source of the issue and leading yourself to more frustration. 

One of the simplest forms of software debugging is printing messages out to a console. Though this is a very limited form of debugging that does not give detailed insight into a microcontroller system and the software (one of the reasons I haven't been a huge fan of Arduino). It might be useful for simple systems, but again, with growing complexity, I render it quite useless. Setting up a debug toolchain with solutions like gdb and OpenOCD would give you low-level access to many of the things happening in a microcontroller. Although it might be time-consuming and painful at times to get a toolchain to work, it can save you a lot of time identifying the source of a problem 

On the hardware end, if skipping plug n play solutions, sometimes it would pay off to have some debugging tools as well. One would feel blind debugging applications involving serial communications if they can't view what is happening on the bus. Typical tools to debug hardware include logic analyzers and oscilloscopes. While these are tools that are considered costly, there are solutions nowadays fit for embedded applications that are quite affordable like the [Digilent Analog Discovery](https://digilent.com/shop/analog-discovery-2-100ms-s-usb-oscilloscope-logic-analyzer-and-variable-power-supply/) and [Salae logic analyzers](https://www.saleae.com/) .

## Conclusion
Embedded IoT is a tricky field to tackle if not done right. Consequently, how a learner decides to approach embedded can make the difference between finding it really exciting or really frustrating. Simplicity is key in the beginning, but often learners tend to find themselves lost the most when going beyond introductory levels. 
Many would tell you that you are guaranteed to face problems in many applications you build. As a result, not having the proper habits in tackling problems is going to lead to frustration. In this post, I summarized the 4 most common reasons l found that lead to giving up early. Do you share my opinion? Do you have a different experience? Share your thoughts in the comments below 👇.  If you found this useful, make sure you subscribe to the newsletter [here](https://subscribepage.io/apollolabsnewsletter) to stay informed about new blog posts.

