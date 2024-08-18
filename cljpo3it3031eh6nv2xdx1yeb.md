---
title: "Innovation Made Easy: 7 Hidden Features to Harness the Power of ESP in Wokwi"
datePublished: Wed Jul 05 2023 12:00:39 GMT+0000 (Coordinated Universal Time)
cuid: cljpo3it3031eh6nv2xdx1yeb
slug: innovation-made-easy-7-hidden-features-to-harness-the-power-of-esp-in-wokwi
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1687856693206/01097093-716b-4304-b481-fe90e29f1c18.png
tags: features, rust, esp32, embedded-systems

---

It's no secret that I am a huge fan of Wokwi. I think it's one of the greatest tools introduced in recent times for embedded enthusiasts and learners. This is mainly because it relieves you of the hardware part. Especially if you have difficulty accessing hardware or don't want to splash out the money as a beginner. Other than the part that it's actually a more sustainable alternative. I can only count the number of dev boards that I bought and are sitting around after projects concluded. Wokwi is also rapidly evolving to include even more helpful features, devices, and languages. For more about Wokwi and other hardware-free resources, there's a [past post](https://apollolabsblog.hashnode.dev/embedded-iot-without-hardware-8-must-know-resources) I wrote discussing the pros and cons.

One of the cool things about Wokwi right now is that it supports Rust, particularly for ESP devices. Behind the beginner-friendly looks, there are sort of what I like to call "hidden features" that are not particularly obvious and are really powerful.

Without further ado, here's a list of those features:

%%[substart] 

## 1) Flash Directly From Browser ⚡️

Have you ever tried to press the F1 key in Wokwi? Well, a world of features behind that list pops up. One of them is the ability to directly download a program image to ESP hardware. Meaning if you have a physical device on hand connected to your PC you can flash your device directly from the browser. No IDEs, toolchains, or anything else.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687856078370/d9ab0189-61cd-4737-9856-cf65e8b6496c.gif align="center")

## 2) Pause for Pin Configuration 📌

Not sure if you configured your ESP device pins correctly? In Wokwi, if you pause a live simulation, a label will appear beside each pin showing what the pin configuration is. This is really also useful at times to make sure you connected the pins to the correct circuitry.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687855460962/fa102e1c-55d6-4856-a181-10624ecdab35.gif align="center")

## 3) Develop Using VSCode 👨‍💻

Do you prefer using VSCode as your editor? Wokwi actually has an extension allowing you to develop directly from VSCode and simulate there as well. Wokwi for VS Code works also with Zehpyr Project, PlatformIO, ESP-IDF, Pi Pico SDK, NuttX, Rust, and Arduino CLI, among others. You can get started by following the instructions [here](https://docs.wokwi.com/vscode/getting-started).

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687855646802/ab3d3f0c-ad7c-448d-8b68-dfd477c16fcc.gif align="center")

## 4) Code Profiling 🏎️

You can actually profile your ESP code using Wokwi. How? You might have guessed, it's an option behind that F1 menu. If you select the profile program option Wokwi will compile and run the simulation.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687856096950/59672f3d-d517-44c1-a4b7-886ab1d5229f.gif align="center")

When you pause (not stop) the simulation a `profile.json` file will be downloaded. After that, you'll need to go to [https://profiler.wokwi.com/](https://profiler.wokwi.com/) and upload the profile file. You'll get something like this showing you how much time was spent in different parts of your code:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687769144229/b3bb8a4d-8330-4681-bfa9-219d20e5e926.png align="center")

## 5) Interactive Debug 👾

Behind that F1 menu resides a **"Start Web GDB Session (debug build)"** option. Though there is something cooler, it's the built-in [interactive debugger](https://docs.wokwi.com/guides/debugger). Through the interactive debugger, you can step through your code, view your call stack, inspect variables, and even set breakpoints. You can access the debugger by clicking on the three dots in the simulation window:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687854444804/02f54fbe-ad56-44c1-83c1-3cb67e9cd6bb.png align="center")

Currently, only AVR microcontrollers are supported from the web interface. For ESP devices, there is support by using the integrated debugger in Wokwi for VSCode. Here there is an extra step involved where you need first to configure the Wokwi debugger for VSCode, and second, to start the debugger, its back to the F1 menu selecting "**Wokwi: Start Simulator and Wait for Debugger**".

## 6) Continuous Integration 🔄

Continuous Integration (CI) is a software development practice that aims to enhance the efficiency and reliability of the development process. It involves the frequent integration of code changes from multiple developers into a shared repository, followed by an automated build and testing process. The fundamental principle of CI is to detect and address integration issues as early as possible, ensuring that the software remains in a consistently functional state.

Building and testing your code requires a server. You can build and test updates locally before pushing code to a repository, or you can use a CI server that checks for new code commits in a repository. For that, you can use GitHub Actions workflows to build code in your repository and run your tests. What is really cool is that you can integrate the Wokwi Embedded Systems Simulator in your GitHub actions CI workflow. This would enable you to verify that your code is not broken with every new commit.

[This](https://github.com/wokwi/wokwi-ci-action) is a link to the GitHub action that you can integrate into your workflow. There is also a Wokwi CI ESP32 example [here](https://github.com/wokwi/esp32-hello-wokwi-ci) that shows how to use GitHub Actions to build and test ESP32 projects.

## 7) Packet Tracing ✏️

Did you run an ESP32 project that uses the WiFi in the simulator? You can actually view the packet capture of the simulation. Again, behind that F1 menu choose **Download WiFi Packet Capture (PCAP) file**. Your browser will download a file called `wokwi.pcap` which you can open with a tool like Wireshark.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1688539223402/4b009c1e-d562-4405-813e-4d602d47babd.png align="center")

## Conclusion

Wokwi is an embedded hardware simulator that has been replacing much of the need for physical hardware. Additionally, Wokwi provides integrations for popular development boards including the ESP and languages like Rust. In the realm of Wokwi, there reside many hidden features that might not seem obvious to many with many new emerging. This post list 7 powerful features that can help one unlock many more possibilities. Have any questions/comments? Share your thoughts in the comments below 👇.

%%[subend]