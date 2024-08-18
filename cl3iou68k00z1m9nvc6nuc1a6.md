---
title: "Embedded IoT Headaches: How Do I Size My Memory Buffers?"
datePublished: Mon May 23 2022 12:11:42 GMT+0000 (Coordinated Universal Time)
cuid: cl3iou68k00z1m9nvc6nuc1a6
slug: embedded-iot-headaches-how-do-i-size-my-memory-buffers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1653240080380/LJH0sjJtW.png
tags: c, developer, iot, embedded

---

Embedded IoT developers getting involved in more complex applications often struggle when they start dealing with different rate processes. For example, how to deal with applications that have an input peripheral or interrupt that is providing data at one rate, and a software thread that processes the data at a different typically slower rate. In this case, some sort of memory buffering would be required to accommodate the difference in rates. This allows the slower rate thread more time to process the incoming data without loss. Though often beginners tend to misinterpret the system workings or understand what they can do by jumping immediately to conclusions like changing the microcontroller to get one with larger memory or even changing the microcontroller to get a faster one.

The thing is that changing a controller is probably the easier way to go but still one cannot draw such conclusions without proper context. The context is data or information about the system. Otherwise, changing blindly would only lead to more bad decisions and a waste of money. One might end up with a more expensive controller that still cannot do the job. In this post, I will explain through examples how an embedded developer can approach the problem of sizing memory buffers to make more informed decisions when managing a certain application.

%%[substart]

## Queueing Model

Let's start by introducing a common model that replicates the behavior we want to describe. As depicted in the figure below, at the input, the model has some source that is providing us data at a certain rate \\(R\_{input}\\) and accordingly interarrival times \\( T\_{input} = \frac{1}{R\_{input}} \\). The input source could be an interrupt from a serial communication peripheral, ADC peripheral, DMA, or what have you. This input source would be feeding its data into a memory buffer (our queue) that has a length \\( L\_{buffer} \\). Finally, at the output, the model has a task or software thread that grabs data from the buffer at a rate \\( R\_{process} \\) (or processing times \\( T\_{process} = \frac{1}{R\_{process}} \\) ) to process it and push it out. It becomes obvious that if \\( R\_{process} > R\_{input} \\), meaning that the rate that the model processes data is always faster than the rate that data is coming in then we don't have an issue and we don't even need a buffer. Though the question is that if it's the other way around, wouldn't the buffered data always eventually grow out of bounds and overflow? That would be true if \\( R\_{input} \\) and \\( R\_{process} \\) are constant all the time, which is not always the case. I will demonstrate two typical scenarios utilizing this model that are discussed further in the examples that follow.

![Queue Model.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652896707304/95K5wxXfp.png align="center")

![buffovflow.jpeg](https://cdn.hashnode.com/res/hashnode/image/upload/v1653213911231/ke9yzEQlv.jpeg align="center")

Note a few things here, before we do our calculations. First, the memory size we would need in our microcontroller system would be a function of  \\( L\_{buffer} \\). Second, the speed of the CPU or the processing time is a function of  \\( R\_{process} \\). Reducing \\( R\_{process} \\) can come in the form of increasing the speed of the processor or even optimizing the code.  Finally, \\( R\_{input} \\), on the other hand, can be assumed to be constant and defined by the system requirements. This doesn't mean that  \\( R\_{input} \\) cannot change, but should any change happen it would only be if the overall system/applications requirements allow so. As a result, based on the framework presented a designer should be able to make informed decisions on what they need to (or can) change in their system.

##  Scenario 1: Bursty Data Input 💥
From a model perspective applications that operate on data bursts are ones that provide a bunch of data at a rate \\( R\_{input} > R\_{process}\\) every now and then. Additionally, the buffer then is emptied out completely before the next burst comes in. This means that a buffer is a must but the question would be how large? As such, if the buffer is not sized properly to accommodate the new data or even emptied out before the next burst of data comes in, then data loss will occur. In this case, what we need to make sure of is that \\( R\_{process} \\) empties out the buffer before the occurrence of the next burst. In such applications, we typically know what \\( R\_{input} \\) and \\( R\_{process} \\) are and can determine the length of the buffer appropriately so that no overflows occur.


![burst.webp](https://cdn.hashnode.com/res/hashnode/image/upload/v1653213508708/smYqluDbx.webp align="center")

### Burst Data Example
Let's start with a more simple and maybe common example. Let's say you have a DMA transaction (or even serial communication input) that stores a burst of data at a rate of 500 KB/s  (\\( R\_{input} \\)) lasting 100 ms every 5 seconds. Additionally, you know that your processor can process the received data at a rate of 20 KB/s ( \\( R\_{process} \\) ). The question here is how large does your buffer need to be for the processor to process all the burst data? 

Let's analyze this step by step:

First, let's think what is the total amount of data that the burst is going to generate. The input source is generating data at a 500 KB/s rate for 100 ms. As such, we simply need to multiply the two values resulting in input data size \\(D\_{input}\\) equal to:

$$
D\_{input}={R\_{input}} * 0.1 \text{ s} = 50 \text{ KBs}
$$

Second, we know that we can process data at a rate of 20 KB/s. As a result, in the time that data is generated (the 100ms duration) the processor should be able to process part of the incoming data albeit not all of it. As such, you would need to buffer what is leftover so that the data isn't lost. To know the amount of data that can be processed without overflow we can simply do a similar calculation where we multiply the processing rate \\( R\_{process} \\) by the duration of incoming data, we can denote that value to be \\(D\_{processed}\\) and is equal to:

$$
D\_{processed}=R\_{process} * 0.1 \text{ s} = 2 \text{ KBs}
$$

This means that we at least need a buffer of size:

$$
L\_{buffer} = D\_{input} - D\_{processed} = 48 \text{ KBs}
$$

We can generalize the above for any system following the same mode where we can define the buffer length as

$$
L\_{buffer} = \(R\_{input} - R\_{process}\)*T
$$
where \\(T\\) in this case is the duration of the burst.

Finally, it seems like we might have skipped some information. If you noticed we said that the bursts occur every 5 seconds. This means that we need to make sure that we would have emptied the buffer before the next burst occurs. This can be calculated by dividing the buffer size by the processing rate as follows:

$$
\frac{L\_{buffer}}{R\_{process}} = 2.4 \text{ ms}
$$

This means it takes 2.4 milliseconds to empty the buffer which is way less than the 5-second update cadence. Well, what if it wasn't? Then this means that you most likely need to alter/speedup \\(R\_{process}\\). This can be done by optimizing code, using a higher clock frequency, or if all doesn't work, changing the device to a more powerful one. Changing other parameters like let's say slowing down \\(R\_{input}\\) would be ok, only if the system requirements allow.

Keep in mind that under this same burst scenario, there are cases where the rate of the input can change based on time which I won't delve into here. In that case, the calculation becomes a bit more involved where you need to integrate over the different durations taking into consideration how the rate changes. Sometimes it might be easier to go for a worst-case scenario taking into account the worst rate possible over the different time periods and assuming that the worst-case value is constant adopting the save equations above. Afterward, you can check if your system can accommodate it, if it doesn't, then a more detailed analysis might be necessary. 

## Scenario 2: Random Input & Output Processing Rates 🎲

Another existing scenario is that at times we don't know the exact rate of the input and the output. Meaning that the input and output rates are random. However, this doesn't mean that we cannot model arrival and processing times, instead, we need to resort to probabilistic methods that describe the input and output rates using probability distributions. Consequently, all we need to do is estimate what is "on average" our input and output rates. For this, we need to adopt what is referred to as queuing theory.


![queue.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1653213357254/dz5hzjFxx.gif align="center")


Within queueing theory, a common model is referred to as the M/M/1 queue model that replicates nicely behaviour in embedded systems applications. In the M/M/1 model, the first M represents the distribution of the input rate which is modeled as a Poisson distribution. The second M represents the distribution of the output rate which is modeled as an exponential distribution. Finally, 1 represents the number of processing elements that we have which is one in this case. 

Don't worry much as we are not going to delve into the mathematical proofs behind the distributions presented here. Though to get a sense of what is happening here, all you need to know is that the Poisson and exponential distributions are common in modeling what is referred to as arrival rates. As such, here what we are trying to accomplish is to minimize the possibility of a buffer overflow as much as possible. 

To introduce our new formulae, we're now going to denote the average process rate as \\( R\_{processavg} \\) and the average input rate as \\( R\_{inputavg} \\). I'm going to first present all the formulas and then work through examples to explain usage.

A useful ratio we can calculate is the ratio of \\( R\_{processavg} \\) to \\( R\_{inputavg} \\), we are going to denote it as \\( \rho \\) and its expressed as follows:
$$
\rho = \frac{R\_{processavg}}{R\_{inputavg}}
$$
Examining \\( \rho \\), you can notice that as its value gets closer to 1 then \\( R\_{inputavg} \\) is also close to \\( R\_{processavg} \\) which means we are nearing having a queue that grows out of bound (goes to infinite size). One might think that if \\( R\_{inputavg} \\) is equal to \\( R\_{processavg} \\) then we don't need to buffer. That is true if the rates were constant. Though keep in mind in this scenario that values of \\( R\_{inputavg} \\) and \\( R\_{processavg} \\) are averages following a distirbution.

### Length of Queue Formula 📏
Obviously, this was one of our main goals, determining what our queue size should be. Using \\( \rho \\), the average length of the queue \\( L\_{bufferavg} \\) is defined as follows:
$$
L\_{bufferavg} = \frac{\rho}{1-\rho}
$$

### Time Spent in Queue (Delay) Formula ⏱
The time a message spends in the queue \\(T\_{queue}\\) is good for scenarios if we want to determine the delay. Meaning how much time a message sits inside a queue before it is processed. \\(T\_{queue}\\) is defined as:

$$
T\_{queue} = \frac{R\_{processavg}}{1-\rho}
$$


### Probability of Number of Messages in a Queue 📨
Since this scenario is all based on probability, there are no guarantees that the buffer won't overflow. However, there are some cases where we have the buffer size already defined but we need to find out the probability that it might overflow. This can be determined by the following formula:

$$
\text{Pr}[\geq m\_{\text{queued}}] = \rho^{m\_{\text{queued}}}
$$

\\( \text{Pr}[\geq m\_{\text{queued}}] \\) can be read as the probability that at least \\( m\_{\text{queued}} \\) messages exist in the queue at the same time. 

Next, I'll present some examples. You'll see that from here all you need to do is simply plug system figures into the above equations to get your answers. 

### Example: Determining Queue Length
Let's say you need to determine the buffer size for a source that is generating a certain size data element on average every 4 ms, on the other hand, it takes on average 5 ms to process each element. 

Notice here that we are dealing with average durations, not rates like we've indicated in our model. To convert to a rate, we simply take the reciprocal of the duration. So using the above equation we can calculate the buffer size we need as follows:

$$
L\_{bufferavg} = \frac{\frac{4}{5}}{1-\frac{4}{5}} = \frac{0.8}{0.2} = 4 \text{ elements}
$$

### Example: Determining Delay
For the same above example, we can determine how long messages/elements remain in a buffer on average. This becomes necessary to know for applications where you need to meet a certain deadline and cannot afford more than a certain amount of delay (Ex. Real-time applications). For the same figures from the previous example we can calculate the delay as follows:

$$
T\_{queue} = \frac{\frac{1}{5}}{1-\frac{4}{5}} = \frac{0.2}{0.2} = 1 \text{ ms}
$$

This means that each message spends on average 1 ms in the queue before it is processed.

### Example: Analyzing an Existing Queue
Let's say you have an existing system with \\( R\_{inputavg} = \frac{1}{30 \text{ ms}}\\) and \\( R\_{processavg} = \frac{1}{3 \text{ ms}}\\) and memory that allows us only to queue 5 messages. As such, we need to find out, how probable is it for our buffer to overflow. This means we need to know how probable it is to get 6 messages or more since our buffer fits only 5 messages.

Using the above equation, then:

$$
\text{Pr}[\geq 6] = \left(\frac{3}{30}\right)^{6} = 0.000001 = 0.0001%
$$

This means there is less than 0.0001% chance that 6 or more messages would appear in the buffer given the rates at hand.

## Conclusion
If you want to build more complex systems in embedded IoT, will need to manage multiple asynchronous processes at a time. The traditional approach of having the controller sit idle while waiting for new data to come wouldn't be sustainable. This means that at a certain point you would be dealing with multiple data sources providing data at different rates and a processor that is trying to keep up with processing all that data. As such, data buffering becomes a must. Without a proper framework to size buffers, you are setting yourself up for a near-impossible task. This post gives a primer on how you should tackle such problems. Have something to add? Share your thoughts in the comments below👇. 

%%[subend]