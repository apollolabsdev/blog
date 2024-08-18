---
title: "6 Things I Wish I Knew Starting with Embedded Rust"
datePublished: Mon Aug 15 2022 16:58:23 GMT+0000 (Coordinated Universal Time)
cuid: cl6v02e6905o1s0nve9ngfx4n
slug: 6-things-i-wish-i-knew-starting-with-embedded-rust
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1660582263858/wh01V5Dpe.png
tags: iot, beginners, rust, embedded

---

## Introduction 

There is no doubt in my mind that Rust is a great (if not the best) language fit for embedded. Especially since eventually a lot that newcomers can benefit as the space evolves. For one, if you are already in embedded I'm sure you spent hours as a newbie debugging an issue just to find out you misconfigured a pin. It turns out Rust can incorporate many of these checks statically. Meaning if you go and configure a pin for UART usage when it's not supported for the controller at hand you get a compile error. That is all managed through the Rust type system. This makes one feel that this is just some work of the future 😃. 

The benefits of Rust as a language also probably mainly shine at the HAL level rather than the PAC. That's where many of the features of Rust can be leveraged to increase portability and reliability. Interestingly enough, through my experience thus far, although the HAL is at a higher abstraction level, it wasn't easier to work with compared to lower abstractions. I actually found a tougher time navigating my way through the HAL to achieve what I needed compared to the PAC. At the PAC, things were much more straightforward since I would be familiar with what I needed to do exactly at the device register level.

Based on my experience thus far, in my humble opinion, embedded Rust learning content still has some ways to go to accommodate embedded newbies (even ones already familiar with Rust as a language). Be mindful as well that the HALs out there are still evolving, and as they do, I think there is more space for improvement in the documentation and commonality among HALs.

This post is sort of a summary of my perception versus reality when getting into learning embedded Rust. My hope is that it would serve as some sort of guide to set expectations for anyone about to embark on the field. It's worth noting that when I started out, my focus was on learning Rust at the HAL. Rust at the PAC was a bit more straightforward to me as one would be configuring registers like they would in another language. Given the speed at which everything Rust seems to be evolving, I'm hoping that this post would soon go out of date!

Please note here that the goal is not to deter, but rather to encourage more to get in and contribute while setting proper expectations. As a matter of fact, I loved (and am still loving) the experience as it helped me learn a ton about embedded Rust and the state of things. A lot of what I mention here is what actually prompted me to start a continuously evolving series about [developing embedded Rust at the HAL with STM32](https://apollolabsblog.hashnode.dev/series/stm32f4-embedded-rust-hal). 

%%[substart]

Without further ado, here is a list of 6 things I wish I knew starting out:

## 1. Don't Get Carried away with Tooling Setup 

![toomanytools.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1660579587730/nOR3Wh82W.jpeg align="center")

In embedded, when introduced to a new platform, typically a new learner would want to be able to plug, code, program, message log, and debug hassle-free. At that point, performance or speed would be the least concern. Ideally, it would be an integrated environment, with minimal setup challenges. I personally don't consider myself an embedded newcomer yet I recall it probably took me a day or so to set up an integrated working environment and work through issues. This was including a debug toolchain and message logging to the console. It mainly was mainly trying with different options provided out there until I got what I needed to work. 

In my search for an integrated environment, CLion was the closest, only for me to discover soon enough that embedded GDB was not supported. After that, I stuck with VSCode to provide what I need as it seemed that most of the community was using it. Now, on the programming and debugging end, I got started with probe-rs by Ferrous systems which is an awesome tool. Though its VSCode extension was not released as it is still regarded as being in the 'Alpha' state. One can go through extra steps to install the extension which was not a smooth sail. Additionally, I personally prefer stable releases.
 
On the other hand, for message logging, one can get swamped in the choices of semihosting, ITM, defmt, or UART. In addition to multiple write ups on why you should be using one over another. Granted that one should be aware that not all options are supported by all controllers (ex. ITM). Although I don't remember the details, I recall it was a struggle to get some of the options to work. I guess I got carried away with the excitement of trying different things. Still, I personally didn't want to get bogged down in dealing with tooling setup issues rather than working on embedded Rust itself.

I personally faced the least troubles going with a standard toolchain of OpenOCD and GDB along with the Cortex-M Debug VSCode Extension. This is in addition to good old UART for message logging. The rest could be left for later as one becomes more familiar with the space and the options revolving around it.

Soon enough, maybe short term future, the Ferrous Knurling-rs tools (Ex. defmt and probe-rs) would probably become the go-to tools for embedded Rust.  

## 2. Learning Resource Expectations 📚

![learnembedded.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1660580171942/eToQ-0Tx6.jpeg align="center")

As I've navigated the embedded Rust field, I noticed that most resources focus on what the language brings. This is opposed to having resources focus on learning the field (embedded) with Rust as the language of choice. Maybe, in other words, I had an expectation along the lines of "learn embedded with Rust" as opposed to "how Rust applies to embedded". This is probably a recurring pattern I observed while learning Rust itself. Meaning that most resources assumed the reader knows how to program, but the material shows how Rust is different/better. This is a similar experience I've expressed when learning Rust itself in a past blog post ["5 things I loved about learning Rust"](https://apollolabsblog.hashnode.dev/5-things-i-loved-about-learning-rust). 

Part of this feeling is probably all because I had my expectations built on past experiences. Meaning that because of my past exposure to embedded C/C++  resources, I expected material similar in nature. This means that currently, most resources have an expectation that newcomers are already familiar with the embedded field and the underlying technology (Ex. microcontrollers, and their features and peripherals). I feel that if embedded Rust were to accept more embedded newbies, more resources need to evolve in a different direction. This is maybe even to attract Rust enthusiasts that want to get into embedded. 

In summary, the point here is that one getting into embedded Rust needs to manage expectations when it comes to learning resources. If one is an embedded newbie, I think it would be better to defer to standard embedded C/C++ learning resources before diving into embedded Rust. The good news though is that many hands-on learning resources exist and keep emerging, the latest of which was the Ferrous Systems and ESP announcement of the [Rust Training on ESP32](https://www.espressif.com/en/news/ESP_RUST_training). For more resources, there is always the [Awesome Embedded Rust Repository](https://github.com/rust-embedded/awesome-embedded-rust) that contains an aggregation of almost everything embedded Rust related.

## 3. Not all HALs are Created Equal 👥

Coming into embedded Rust, one of the things that are highlighted repeatedly is how language features like traits can potentially enable a lot of portability. As a matter of fact, the embedded-hal that most (if not all) HALs out there are built on states as one of its rules that it "must erase device specific details". As such, I had this misconceived notion that there would be much more common methods/traits among device HALs than I thought. What I found instead is that there are some embedded-hal traits that span all HALs, but there were much more methods that were specific to a certain HAL. 

I personally worked with an STM32. I found that among STM32 device families there were separate HALs for each family. Within each device family HAL, there were methods that would achieve the same thing but did not necessarily have matching signatures among other STM32 HALs. It felt a bit strange as many of the STM32 devices are configured in the same manner at the low level for certain peripherals. It felt a bit weird as it was sort of opposing the portability notion.

## 4. Confusing Method/API Patterns 🤔

![apis.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1660580477987/o-zbv2NGR.jpeg align="center")

This made navigating the documentation a nightmare at first. Especially when I wanted to deal with a new peripheral. Especially since there is a huge lack of consistency in the way how different peripherals are instantiated/configured. Other than the fact that I found more than one way to instantiate some peripherals (more on this below). For example, when I first started out, when wanting to experiment with a new peripheral I would seek some sort of `new` method. It turns out that there isn't always one for all peripherals. This made it more difficult to figure out sometimes given the level of completeness of documentation. At times I would refer to examples in the same repo for the HAL but the issue is that several times (except for comments provided) the examples lack context.

One also needs to be aware of the difference between embedded HAL traits and device HAL traits. This might be specific to some HALs as I didn't see this pattern always map to other stm32 HALs (I used the stm32f4xx-hal specifically). Often I would find that there were two ways to instantiate a handle for a peripheral. One using a specific HAL method (something like a `new` method) and the other using an extension trait available within the device HAL to instantiate. I used to think that the extension traits were imported from the embedded-hal. I still wonder what the purpose behind this was since it felt redundant. 

What I describe here is quite different in the STM32-hal which I talk about more in my last point.

## 5. Feature & Documentation Completeness 📄

![documenation.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1660580951729/c5NS_G3wx.jpeg align="center")

This is something that although I knew about, I didn't realize the gap till later. My approach to learning embedded Rust was to grab a dev board I had and experiment with peripherals one by one. For that, I had to rely heavily on HAL documentation and examples. Along the line, I noticed that features and documentation were still not as complete as I expected. For example, I noticed that timers for the stm32f4xx had no input capture feature implemented, also I2C does not support interrupts yet, and so on. As a result, one has to be aware that not all controller device low-level features are exposed in all the HALs. The same applied to documentation, several methods did not really have a description of what they did. I managed to figure out how they worked around that in three ways; experimentation, looking at other HALs, or reading the source code. It's worth noting from a feature perspective, I felt that the embedded-hal itself still had a lot of space for improvement. In fairness, the embedded-hal docs do state the following:

>Expect the traits presented here to be tweaked, split or be replaced wholesale before being stabilized, i.e. before hitting the 1.0.0 release. That being said there’s a part of the HAL that’s currently considered unproven and is hidden behind an “unproven” Cargo feature.

This is understandable since Rust is an open source project that's based on contributions and all this has to do with how well the projects are maintained. To minimize the effect of what I mention here, one probably is better off starting with a device HAL that is considered more "complete" so to speak. The issue is that I couldn't really find a place where I could figure out the completeness level of the different embedded HALs. It just came along from exposure and experience. At least in the case of the STM32 I’ve noticed that the stm32f1xx and stm32f3xx were the most wholesome HALs. I think I read in many repo descriptions that most other stm32 device HALs were also inspired by the stm32f1xx-hal. If one wants to start with a different device with multiple families, it might pay off to ask in one of the communities which HAL would be the most recommended to start with.

## 6. Watch for HALs with Different Patterns 🔍

![other hals.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1660580620245/7anz1h8qy.jpeg align="center")

As I worked with the stm32, as implied earlier, the HALs that I worked with were ones built around embedded-hal traits. Nevertheless, I came across a HAL at a certain point that adopted a different approach that felt more practical and easy to understand. This was the  [stm32-hal](https://github.com/David-OConnor/stm32-hal) that I found to be more wholesome as it incroporated multiple families of the STM32 under a single HAL umbrella (my original expectation). The STM32-hal eliminates much of the trait confusion that I had encountered before. The thing is the stm32-hal does not seem to be mainstream yet. From what I understand, the HALs built with the mebedded-hal as a basis seem to be the ones mainly adopted by the embedded working group. Additionally, I am not sure if the stm32-hal has any equivalent counterparts for other manufacturer devices. 


## Conclusion

Having learned Rust the language followed by embedded Rust, I did feel a noticeable difference. Although I am already familiar with the embedded space, I found the learning curve to be steeper in embedded Rust. There was much more navigation I needed to do rather than referring to one central place. I would say that when I set out on learning embedded Rust, I probably had different expectations than the reality I encountered. Additionally, compared to programming with Rust, it felt that embedded Rust's community had a higher ratio of experts to newbies. This could be considered a good and bad thing depending on where one stands. Overall, what I would say is that if you are new to embedded altogether, picking up embedded Rust first might not be the best approach. One is probably better off learning embedded with C/C++ and then switching over to embedded Rust. The good news like everything Rust though, is that you can bet on the awesome community for things to change pretty quickly.

Do you share my opinion? Do you have a different experience? Share your thoughts in the comments below 👇.  

%%[subend]


