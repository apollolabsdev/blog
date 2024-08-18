---
title: "Identifying Required Skills for Jobs in Embedded IoT"
datePublished: Tue May 03 2022 00:37:44 GMT+0000 (Coordinated Universal Time)
cuid: cl2pf8osr01aw9mnv92tx6qq3
slug: identifying-required-skills-for-jobs-in-embedded-iot
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1646998674757/lVnvijiKk.png
tags: iot, jobs, career, embedded, job-search

---

## Introduction
I often get asked or see questions from embedded job seekers about what type of knowledge or expertise is required for certain job titles. Some search in job forums could help, but given the variety of titles between companies, it can become overwhelming at times. Especially if the seeker wants to pick a field to specialize in or even hone their skills in a particular area. 

I personally have a structured way in which I break down any form of expertise down to 3 knowledge areas. This is all derived from the development process adopted in companies that develop embedded solutions. Though before that, let's establish something first. You have to know that for the same job title between different companies the core knowledge needed is more or less identical. The differences lie in specific tools or programming languages one company might use vs. the other. Obviously, a good strategy would be to focus on acquiring knowledge in the most common tools, but don't attempt to pile on tool knowledge. At the end of the day, the core foundational knowledge and execution is what counts and what a good company would be looking for. Tools are tools, they exist to make the tasks you need to achieve around that foundational knowledge easier. If you have strong foundations, picking up tool knowledge shouldn't be a problem.

## The V-Model

Before diving in, one aspect important to understand is that almost all companies that develop embedded solutions/products adopt some form of development model. A development model is essentially a series of steps and/or processes that a product goes through from concept until it is released to production. One of the most popular development models in embedded is referred to as the V-model (shown below).


![V-model.svg.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646940869072/LINhaG6sm.png)


If it isn't obvious already, it's referred to as a V-model because it looks like the letter V 😄. On the left side of the V, that is where the development process starts, and it is where all the requirement capture, concept definition, and design tasks take place. Once all design and requirement tasks are completed, the product is implemented at the bottom of the V. At the completion of the implementation, the product needs to be tested that it meets the requirements it was developed for on the left side of the V. As such, all the testing to verify requirements created at design time takes place on the right side of the V. Testing happens at many levels of the product as well.

## The 3 Core Areas of Knowledge

Given we now know about the V-model, in Embedded Systems, almost all job titles come down to three core areas of knowledge; software, hardware, and systems. 

![Job Tree.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646939922515/Bjc1mA0jd.png)

These three areas all map to different parts of the V-model. For example, systems engineering tasks are focused typically on the upper left and right sides of the V-model. On the other hand, software and hardware engineering tasks are focused on the bottom part of the V-model. Within the three core areas of knowledge, a standard job title exists for each i.e. hardware engineer, software engineer, and systems engineer. There are specialization areas as well, however, each of those three titles requires some standard background regardless of specialization. 

Obviously, to increase job chances, one almost has to pick an area of specialization in either one of the 3 core areas. While some specialization areas are constantly needed (Ex. firmware development), there are new ones that keep appearing as technology advances (Ex. machine learning). Getting ahead of the curve and gaining knowledge in trending areas, no question puts you at a huge advantage over other applicants. Read my other blog posts [here](https://apollolabsblog.hashnode.dev/what-should-you-learn-next-in-tech) and [here](https://apollolabsblog.hashnode.dev/6-trending-topics-every-embedded-iot-learner-must-know-in-2022) relative to identifying in-demand/ trending areas of specializations. Now, back to our titles.

### Hardware Engineering 🛠

Hardware engineering is focused on building/implementing embedded hardware. This requires solid electronics circuit design understanding, both digital and analog. As a hardware engineer, you will design, build, test, and debug hardware. Reading and understanding datasheets is or prime importance. Also, knowing different types of ICs, components, their packages and when to use which. Understanding power dissipation and cooling methods is required as well. Tool knowledge typically revolves around schematic capture and printed circuit board (PCB) layout. Debug/test knowledge is important as well. This means knowing your way around tools like oscilloscopes and logic analyzers. Finally, hardware engineers are also typically involved in the production process. Knowing how to optimize your hardware for testing and production is always a plus. You might have heard terms like design for test (DFT) and design for manufacturing (DFM).

### Software Engineering 👨‍💻

Software engineering in embedded is about creating software for the hardware created by the hardware engineers! As any software engineer in non-embedded, you should be strong in coding and algorithm development. What is different from a non-embedded software engineer is that you will be developing at a low level (ex. bare metal). As such, understanding development and optimization (for speed and/or power) for different microcontrollers is important. This also means knowledge in assembly programming (ARM architecture being the most popular). You are expected to know how to create simple schedulers and bootloaders as well. Additionally, you need a solid understanding in compile and debugging toolchains. Possibly the most common languages in embedded are C/C++, though if you want to jump on a trend then Rust is a rising star as well.

### Systems Engineering ⚙️

This is not to be confused with the common title of systems engineering that is used in networks (IT sector) as well. That is a totally different type of job. Systems engineering in embedded requires a more holistic view of an embedded system. Essentially understanding both software, hardware, sometimes even mechanical, and how they interact with each other. Systems engineers typically capture requirements from customers and propagate them into more detailed requirements for the engineering teams internally. Writing solid requirements and having solid analysis skills is crucial as a result. Systems engineers are also expected to maintain and track those requirements ensuring they are being met by communicating with all other engineering teams. This means that systems engineers architect a whole system and partition areas of the product between software and hardware. This means understanding upper-level system and customer requirements to make sure the system at a functional level delivers what it is required to. This is all while being efficient in terms of cost, power, and performance. 

Systems engineering is different to other core areas in which your expertise might also map to a certain industry. For example, if you are interested in automotive, you are required to understand what automotive systems requirements are so that you can optimize your product towards those requirements. Knowledge in diagramming with modeling tools like UML becomes essential as requirement documentation relies heavily on such diagrams. Finally, as with all other core areas, testing and debugging skills are also important, though at a functional level of a product. System engineers are required to do integration tests at times to ensure that the software and hardware are interacting correctly with each other. This presents a challenge in debugging at times to isolate the source of the problem (software or hardware).

## Dissecting Further

In the V-model, we can see that there are more types of areas (position titles) possible in a development cycle. In essence, the tasks that are required along the V-model can all be combined under one of the previously mentioned 3 core areas, but it might end up being a lot to do under one job description. Typically, as companies grow larger, certain elements/tasks are extracted from the core area tasks to create more focused jobs. Examples of those elements include design, testing, architecture, and well, quality (although not explicitly spelled in the model). Now we can expand our previous tree to map those new elements to each of the core areas, generating new titles as follows:

![job tree ext.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1646994511299/eD9w3vEp_.png)

### Design Engineering and Architect Jobs

Regarding design engineering and architecture jobs, the specific experience they delve into becomes specializations in the core areas. For example, a firmware engineer is a type of software design engineer that focuses on developing firmware. Another example is a Hardware Safety architect, which is a hardware engineer specialized in architecting hardware safety designs. There are some more examples of titles below. You would have to read the specific descriptions for each of those titles to understand what they need as there can be many variations. Again, the core knowledge mentioned before still applies, it’s only some of the abilities that become more specific.

Some example titles in hardware may include:
- RF Design Engineer/Architect
- Mixed-Signal Design Engineer/Architect
- FPGA Design Engineer/Architect
- Hardware Safety Design Engineer/Architect

Some example titles in software may include:
- Firmware Design Engineer/architect
- Image Processing Software Design Engineer/Architect
- Machine Learning Software Design Engineer/Architect
- Real-time (or RTOS) Software Design Engineer/Architect
- Internet of Things (IoT) Software Design Engineer/Architect
- Software Security Design Engineer/Architect

For systems engineering the titles would be mostly similar as the ones just listed, though mapped to systems context. You only would need to replace the terms hardware and software, with systems. For example, a hardware safety design engineer becomes a system safety design engineer. Though be mindful that in some cases it might not work, as for example there typically aren’t any “systems firmware engineers”.

### Test Engineering Jobs

Looking at test engineering, the knowledge area becomes more involved within the core area as the engineer would have a focus on mainly developing test sequences for various projects, so scripting knowledge becomes important. Additionally, being able to develop specialized test systems or for example create and run Hardware in the Loop (HIL) systems. Nowadays, knowledge in CI/CD pipelines, version control (Ex. Git), and containers are also becoming (or already have become) essential knowledge for test engineers in embedded. If interested in this path you might find [this](https://jamesmunns.com/blog/hardware-ci-overview/) blog post by the great James Munns really insightful.

### Quality Engineering Jobs

In bigger companies, quality engineering is more common as expertise is needed in making sure that processes and design standards are followed. Additionally, having the personnel to handle quality customer quality issues/returns and ensure that corrective action is taken in the company processes is taken. Another aspect that quality engineers are involved in is process design and optimization. Background in topics like six-sigma become relevant for quality engineers.

## Conclusion
While job titles in embedded might be many, any job title in embedded can be broken down systematically to map to a specific core area. Within each of the core areas, there are clear foundations that you need to know to qualify for that type of job. Want to gain an extra edge, or advance your career? then you need to specialize in a trending area where there's little supply and increasing demand. How would you identify a trend like that? Read my other blog posts [here](https://apollolabsblog.hashnode.dev/what-should-you-learn-next-in-tech) and [here](https://apollolabsblog.hashnode.dev/6-trending-topics-every-embedded-iot-learner-must-know-in-2022) relative to identifying in-demand/ trending areas of specializations. Have any questions or comments? Please share your thoughts👇. If you found this useful, make sure you subscribe to the newsletter [here](https://subscribepage.io/apollolabsnewsletter) to stay informed about new blog posts. 