---
title: "From Zero to Rust: Simplified Embedded Systems Programming"
datePublished: Mon May 20 2024 20:54:38 GMT+0000 (Coordinated Universal Time)
cuid: clwfg2tb8000008jzcnb9bq8g
slug: from-zero-to-rust-simplified-embedded-systems-programming
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716237290838/39404d58-13c2-4e30-b877-a1ecad9c67b7.png
tags: books, beginner, learning, iot, rust, embedded

---

## Introduction 🚀

On Friday, May 17th, 2024, the release of *Simplified Embedded Rust* was announced marking a new kind of book in the embedded Rust learning space. I'm happy to say that this book, designed to address the needs of learners, has been received with great excitement and enthusiasm. Following its release, I received an overwhelming number of responses and questions. Common queries included:

* 📚 When is the paperback going to be available?
    
* 💳 Why the subscription model?
    
* 📖 Which edition should I choose?
    
* 🦀 What's Rust?
    
* 🤔 How different is the book from [*The Embedded Rustacean*](http://blog.theembeddedrustacean.com/) blog posts?
    

In this post, I aim to address these questions and provide further insights into the motivations and goals behind the book.

%%[substart] 

## Motivation Behind the Book 🎓

Curiosity is a vital component of learning 🐈. As educators, we strive to nurture this curiosity to help students discover their passion. However, without engaging and exciting material, we risk losing their interest and attention. Hands-on learning is crucial, but what is required to get started can vary significantly across different fields. Beginners are in some cases expected to deal with complex tools they encounter for the first time. For instance, coding used to require relatively involved setups, whereas now, it can be done in seconds on several web-based platforms. Embedded adds a twist to this -&gt; hardware 😱.

*Simplified Embedded Rust* is as much about transforming the approach to embedded systems learning as it is about embedded Rust. Learning embedded systems can be daunting due to the complexities of toolchains and setups, especially for beginners. Rust, often seen as a challenging language, adds another layer of difficulty. This book rises to the challenge by simplifying Rust's introduction within the context of embedded systems.

## Goals 🎯

The primary goal of *Simplified Embedded Rust* is to make embedded Rust accessible and straightforward. By lowering the barrier to entry, the book aims to increase accessibility and provide a coherent overview of embedded Rust, minimizing the challenges many of us faced early on. The book gradually builds up the necessary knowledge, making it easier for readers to grasp more complex concepts. Practical material for each peripheral is included, allowing readers to apply their learning effectively without needing physical hardware. Thanks to modern tools like [Wokwi](https://wokwi.com/) we can replicate the software learning experience in embedded more effectively. The book also supports programming any supported ESP device in Rust, leveraging the powerful Espressif crates.

## Why Rust? 🦀

Rust is a modern systems programming language designed for performance, safety, and concurrency. Unlike many traditional languages, Rust provides memory safety without requiring a garbage collector, making it a great fit for low-level programming where direct hardware access and real-time constraints are crucial.

Key features of Rust include:

* **Memory Safety**: Rust ensures memory safety through its infamous borrow checker preventing common bugs such as null pointer dereferencing and buffer overflows by using a strict ownership system. 🔒
    
* **Concurrency**: Rust's concurrency model ensures that data races are caught at compile time, making it easier to write safe, concurrent code. 🏎️
    
* **Performance**: Rust's performance is comparable to C and C++, making it suitable for systems programming and other performance-critical applications. 🚀
    
* **Tooling**: Rust comes with a rich set of tools, including a package manager (Cargo), a build system, and integrated testing support. 🛠️
    

Rust's unique combination of memory-safety, performance, and modern tooling makes it an excellent choice for embedded systems programming, where reliability and efficiency are paramount.

## Why ESP? 🌐

The choice of ESP was deliberate. ESP offers the lowest entry barrier and has official Rust support, making it an ideal starting point. Espressif was one of the first system-on-chip (SoC) hardware companies to announce official support for Rust. Espressif has also a dedicated Rust team that contributes to their open source project to support embedded crates for ESPs. Through this effort, and among many other things, espressif also supports Rust on Wokwi. The espressif Rust team was also instrumental in helping me bring this book to life.

## The Review Process 🔍

*Simplified Embedded Rust* is self-published, but ensuring its technical accuracy and effectiveness was paramount. To achieve this, both editions of the book was sent for review by a diverse group of 20 reviewers, who provided invaluable feedback. The group included both experienced and beginner developers. Their insights were invaluable in helping ensure the book meets its goals and maintains technical correctness.

## **How is This Book Different from The Blog?** ✍️

A question I received is how the book differs from the content on [*The Embedded Rustacean*](https://blog.theembeddedrustacean.com/) blog (formerly apollolabs tech blog). While *Simplified Embedded Rust* builds on several examples from the blog, it offers core differences that make it a more comprehensive learning resource. Unlike the blog, the book is self-contained with exercises, aims to be ESP agnostic, offers a clear flow, and provides a conceptual background on embedded systems and the Rust ecosystem. The book is also regularly updated and includes pre-wired exercises, ensuring readers have a seamless learning experience from start to finish.

## What the Book Covers 📘

The book starts with a generic explanation of microcontroller systems that drive embedded systems, providing essential background knowledge for programming embedded systems. This is followed by an in-depth explanation of the ESP and Rust embedded ecosystems. The remainder of the book is programming-oriented, covering all core peripherals and how to code each in Rust on ESP. Each peripheral has its own chapter that includes:

1. **Conceptual Background**: An overview of the peripheral and how it operates.
    
2. **Configuration and Coding Steps**: Detailed steps to configure and program the peripheral.
    
3. **Application Example(s)**: A practical example (or examples) to demonstrate the peripheral in action.
    
4. **Exercises**: Exercises to reinforce learning and provide hands-on experience.
    

## **Who is This Book For?** 🌟

*Simplified Embedded Rust* is targeted at individuals familiar with Rust who are eager to explore embedded systems for the first time, as well as embedded developers at all experience levels who are new to Rust. Whether you're looking to expand your skills or embark on a new learning journey, this book aims to meet your needs and offer valuable insights into the world of embedded Rust.

## Choosing the Right Edition 📚

Espressif offers two approaches for embedded development on their SoCs; `std` and `no-std`. `std,` also known as the standard library, which is based on the ESP-IDF framework. `no-std`, also known as the core library, which is for bare-metal development. For those seeking more details, please check a more detailed overview [here](https://docs.esp-rs.org/book/overview/index.html). As such, the book is available in two editions supporting each approach: the [standard library edition](https://www.theembeddedrustacean.com/c/ser-std) and the [core library edition](https://www.theembeddedrustacean.com/c/ser-no-std). The choice depends on the depth of knowledge you seek. The standard library edition is easier to start with especially if coming from a Rust background, while the core library edition delves a tad deeper. If still confused, the following table might help you decide:

| Embedded Systems Knowledge | Rust Programming Knowledge | Recommended Edition |
| --- | --- | --- |
| Intermediate/Advanced | Intermediate/Advanced | Core |
| Intermediate/Advanced | Beginner | Core or Standard |
| Beginner | Intermediate/Advanced | Standard |
| Beginner | Beginner | Standard |

To acquire the Core Edition [click here](https://www.theembeddedrustacean.com/c/ser-std).

To acquire the Standard Edition [click here](https://www.theembeddedrustacean.com/c/ser-std).

## Why a Subscription Model? 💡

A frequent question is why the book is subscription-based. While this model might not be ideal for everyone, it reflects the ongoing effort required to keep the book current in a dynamic ecosystem. There is a significant amount of work that would be needed to keep the book up to date. For example, during the review process, significant changes occurred to the `esp-hal` which necessitated updates, delaying the publication to ensure that the book is up to date.

Still, for those who prefer a one-time purchase, there are two options:

1. Keep the subscription only if you want updates beyond the minimum duration. You can still purchase once and cancel anytime before renewal, maintaining access to the latest material until the end of the billing cycle. If you need an updated copy at any time later, simply resubscribe.
    
2. Wait for the paperback version, which will be available on Amazon within a month.
    

## Upcoming Updates 🛠️

Although the book is published, I view this as a start rather than an end to an exciting journey. The ecosystem, as mentioned before, is evolving quickly. There are still a lot of exciting things to come that I hope to incorporate. Some short-term milestones include:

* Publishing additional formats. This includes Mobi, Epub, and Paperback. These formats should be available within a month.
    
* Since the book is meant to be ESP agnostic, pre-wired examples and templates for all other ESPs will be provided. Also, if necessary, the content would be modified to make it more overarching.
    
* Added projects. The idea is to incorporate projects for readers so that they can apply their knowledge from all peripherals.
    
* Added examples. Some peripherals might benefit from more examples. Also, some more peripherals might be added, notably SPI.
    
* Incorporate community crates. Some might notice that the core edition does not have a connectivity/networking chapter. This is because I am waiting on the crate involved, `esp-wifi`, to become more stable
    

If you have any ideas or suggestions, please feel free to submit a feature request to either of the book's GitHub repositories. You can submit [here](https://github.com/theembeddedrustacean/ser-std/issues/new?assignees=&labels=enhancement&projects=&template=feature_request.md&title=) for the Standard edition and [here](https://github.com/theembeddedrustacean/ser-no-std/issues/new?assignees=&labels=enhancement&projects=&template=feature_request.md&title=) for the Core edition.

## Acknowledgments 🙏

Finally, this work would not have been possible without all the support that I have received. I extend my heartfelt thanks to many individuals and groups. This experience has underscored how special the Rust community is. I am grateful to the reviewers, the Espressif Rust team, and all my students who have been a constant source of inspiration. Thank you for your support and contributions to making *Simplified Embedded Rust* a reality.

## **Conclusion** ✨

*Simplified Embedded Rust* is more than just a book—it's a gateway to a transformative journey to becoming an embedded Rust developer. Whether you're a seasoned developer or just starting your journey, this book aims to make embedded Rust accessible, engaging, and practical. I invite you to dive in, explore the world of embedded Rust, and start your transformation.

Thank you for your support, and I look forward to your feedback and contributions!