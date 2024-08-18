---
title: "58 Rust Resources Every Learner Should Know in 2023"
datePublished: Wed Apr 05 2023 19:00:52 GMT+0000 (Coordinated Universal Time)
cuid: clg421dzs000i0amvd544gcoh
slug: 58-rust-resources-every-learner-should-know-in-2023
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1680678875988/7664b376-9c39-4fff-b167-07d8db8ebf58.png
tags: resources, beginners, rust, learning-journey, rust-lang

---

Last year, I set out to learn Rust mainly for applying it to embedded systems. Back then I also shared an aggregation of all that learning material I came across in the post "[35 Rust Learning Resources Every Beginner Should Know in 2022](https://apollolabsblog.hashnode.dev/35-rust-learning-resources-every-beginner-should-know-in-2022)". Ever since I haven't really stopped learning Rust as I keep working on solidifying my background. As such, I keep on looking out for new learning resources.

In this post, I will be expanding on the [post from last year](https://apollolabsblog.hashnode.dev/35-rust-learning-resources-every-beginner-should-know-in-2022) by adding more resources and thoughts on some paid resources that I didn't get to leverage before. As a reminder, back then, I split the resources into 6 main categories: **1) reading material**, **2) practice resources**, **3) youtube videos**, **4) git resources**, **5) cheat sheets**, and **6) question forums**. This year I add 3 more categories which are **7) courses**, **8) podcasts,** and **9) newsletters**. Where relevant, I am also adding an annotation for the level of the resource whether it's **beginner (👶)**, **intermediate (👨)**, or **advanced (👴)** level.

I would like to state again that my learning journey started with the official Rust language resource known as "the book" and then I practiced mainly by doing the amazing Rustlings exercises. All other resources I mention in this post, I came across along my learning journey to clarify concepts I read in "the book" or when I got stuck in a certain rustling exercise and wanted to explore more. Lastly, when I am asked if I would do anything differently learning Rust, if anything, based on the resources at hand, I probably would have started with the quick summary resources mentioned below first and then tackled "the book". If I were to go for a paid resource, the "Beginning Rust" book would be the one.

### 1) Reading Material 📚

#### Quick Summaries

**1.** 👶 [Tour of Rust](https://tourofrust.com/) is a step-by-step guide for the Rust programming language. It gives a nice overview of the language and allows the learner to also modify the code examples to experiment.

**2.** 👶 [A half-hour to learn Rust](https://fasterthanli.me/articles/a-half-hour-to-learn-rust) is a nice and quick summary that goes over a lot of concepts of the language with quick examples. Everything is on a single webpage page! This resource is also very useful for refreshing quickly on various concepts.

**3.** 👶 [A Gentle Introduction to Rust](https://stevedonovan.github.io/rust-gentle-intro/readme.html) is sort of a short book that summarizes the main Rust concepts. The idea, as mentioned by the author, is to "try before you buy" to get to know enough about the language before digging into the more involved resources like "the book".

#### 🆓 Free Online Books:

**4.** 👶 [Rust Book "The Book"](https://doc.rust-lang.org/book/): This is the official book on the Rust programming language, written by the Rust development team at the Mozilla Foundation. ITs where I started and one of my absolute favorites. I find myself going back to it often as I write code in Rust.

**5.** 👶 [Rust By Example](https://doc.rust-lang.org/rust-by-example/): Rust by example has a collection of runnable examples for the various concepts in Rust. I found it useful to refer to alongside "the book" whenever I learned a concept there.

**6.** 👶 [Easy Rust](https://dhghomon.github.io/easy_rust/): This is a resource that tries to focus on using "easy English" for learning Rust for non-native speakers of English. I personally found it useful to clarify concepts along the way. The nice thing about this book also is that there is a youtube video linked in the book and associated with each chapter.

**7.** 👴 [Rust Atomics and Locks:](https://marabos.nl/atomics/) This book is written by the awesome Mara Bos and is available for free. As the book's name implies, it talks about Rust atomics and locks. As such, the content of this book is largely relevant for writing concurrent application code. Typically not a beginner feat, however, a must-know resource as you advance your knowledge in Rust.

**8.** 👴 [Rust Design Patterns](https://rust-unofficial.github.io/patterns/intro.html): If you want to get into writing more idiomatic Rust this is a good resource. This is not a book to pick up as a beginner for sure but rather when you advance in Rust. If you come from a different language, the book would also help you navigate differences in design patterns. Reading parts of this book actually helped me as I got more into reading other people's codes.

#### 💲 Paid In-Print Books:

I personally relied completely on the free material to learn Rust, however, the books listed here are ones I acquired later in my learning journey. Additionally, some might prefer a more tangible form of free books of which "the book" itself is also available as a hard copy [here](https://amzn.to/3Km3jwl).

**9\. 👨** [Rust in Action](https://amzn.to/3DRu0qh): I purchased a copy of this book last year. Rust in Action is a great read and offers many examples, but not one that I would recommend for a beginner. The author (Tim Mcnamara) himself also recommends the book to those already familiar with some Rust. My personal experience is that the book helped me clarify various concepts in Rust that were still unclear to me. On the flip side, deeper into the book there are some chapters that I skipped. I found those chapters more specific to particular application areas. Finally, on a side note, the author, Tim Mcnamara, also maintains a youtube channel relative to Rust listed in the video section below.

**10\. 👨** [Programming Rust: Fast, Safe Systems Development](https://amzn.to/3Jfxbtb): This book is rated as an Amazon bestseller. I got my hands on one last year as well. The book goes into a lot of detail and different ways to do the same things in Rust. I would say it is a very good type of reference book as I felt it covers every aspect of the language. I wouldn't recommend it for a beginner though, for one reason, it may be a bit overwhelming at times.

**11.** 👶 [Beginning Rust: From Novice to Professional](https://amzn.to/3DS1pBi): If you would like to pay for books, and you are a beginner, I would say this is the one to go for. I found this book to be one of the most digestible books teaching Rust. The nice thing about this book is that its material also ties to practice material on [Educative](https://www.educative.io/courses/ultimate-guide-to-rust-programming).

**12.** 👴 [Rust for Rustaceans](https://nostarch.com/rust-rustaceans): This is a popular book among advanced users and has great reviews, though one that I haven't got my hands on yet. I'm also recommending it because, the author, Jon Gjengset, has a [youtube channel](https://www.youtube.com/channel/UC_iD0xppBwwsrM9DegC5cQQ) I absolutely love (more details in video section below).

### 2) Practice Material 🏋️‍♂️

#### 🆓 Free Resources

**13.** 👶 [Rustlings](https://github.com/rust-lang/rustlings) is a git project with small exercises to learn Rust. I really loved this resource as it covered each topic in "the book" and was challenging at times. I worked through the whole set of exercises and felt quite accomplished every time I got through a tough one. My favorite was figuring out how to write a function to find the factorial of a number in a single line using iterators.

**14.** 👶 [Exercism](https://exercism.org/tracks/rust) has a hands-on Rust learning track. I found the exercises interesting in the way they are presented. Though I found the exercises to be a bit more simplistic. This could be because I came about the resource later in my learning.

**15.** 👶 [Codewars](https://www.codewars.com/?language=rust) allows you to do coding challenges with peers. It is really cool and helps hone your skills. I found it useful to figure out if I understood certain concepts correctly.

**16.** 👶 [Coding Game](https://www.codingame.com/start) is a game-based learning how to code website that also supports Rust. I didn't get to delve too much into it though Coding Game has a lot of good reviews as it helps make learning fun.

**17.** 👶 [StackUp](https://app.stackup.dev/) is a developer community that allows you to learn and also earn money for participating in our campaigns and quests.

**18\. 👨**👴 [Advent of Code](https://adventofcode.com/) is a yearly event where you can solve small (but high-quality) programming puzzles in any language you want. It can be applied to any language that you are learning. You might find [this](https://github.com/fspoettel/advent-of-code-rust), [this](https://github.com/nickyvanurk/advent-of-code-rust-template), and [this](https://github.com/timvisee/advent-of-code-2022) repo useful as well where they provide templates and solutions for prior years.

**19.** 👶**👨**👴 [LeetCode](https://leetcode.com/) is a popular platform for interview prep with various levels of code problems to solve. There are also various repositories and videos for developers that solved the code problems in Rust if you get stuck.

#### 💲 Paid Resources

**20.** 👶 [Egghead.io](https://egghead.io/q/rust) is more project-oriented and helps developers build portfolios through projects for jobs. There are various interesting options, though one of the nice ones is a course that walks through the Rustlings exercises.

**21.** 👶 [Educative](https://www.educative.io/) has several resources to learn Rust hands-on of which one ties to the [Beginning Rust from Novice to Professional](https://amzn.to/3DS1pBi) book.

### 3) YouTube Videos 📹

**23.** 👶 [Rust Tutorial](https://www.youtube.com/playlist?list=PLLqEtX6ql2EyPAZ1M2_C0GgVd4A-_L4_5), by Doug Milford was one of my favorite resources. Doug does a great job of explaining a lot of the sticky concepts in Rust. The videos also go over different code examples and scenarios to explain.

**24.** 👶 [CS Honors @ Illinois](https://www.youtube.com/channel/UCRA18QWPzB7FYVyg0WFKC6g/featured) was another favorite go-to channel to clarify concepts. The series of videos is based on the CS196 (now CS124) freshman course at the University of Illinois at Urbana Champaign which is run by students. The course videos are not entirely about Rust, however, for the ones that do cover Rust concepts, I found them to be extremely helpful.

**25.** 👶 [Lets Get Rusty](https://www.youtube.com/c/LetsGetRusty/featured) is a useful channel that provides weekly videos about Rust. It has a nice playlist that goes over "the book" chapters one by one. Though I did find the videos to be fast-paced at times and struggled a bit with some of the explanations.

**26.** 👶 [Rust Crash Course](https://www.youtube.com/watch?v=zF34dRivLOw), by Brad Traversy is a great quick tour that introduces various concepts to the absolute beginner.

**27.** 👶 [Ryan Levick](https://www.youtube.com/channel/UCpeX4D-ArTrsqvhLapAHprQ) is a former Microsoft engineer that works on Rust and has a series of videos with useful explanations for certain concepts. There is also a cool project where Ryan builds an SDK for Microsoft Flight Simulator 2020 using Rust.

**28.** **👨** [timClicks](https://www.youtube.com/channel/UClny6qj9Mv7uFo9XGUGYQBA) is a channel by Tim Mcnamara that covers various topics in Rust and also offers live coding streams. Tim is also the author of the [Rust in Action](https://amzn.to/3DRu0qh) book mentioned earlier.

**29.** 👶 [Free Code Camp](https://www.youtube.com/watch?v=MsocPEZBd-M&t=1820s) provides a beginner tutorial for Rust that works through a CLI application. Although somewhat useful, I didn't spend much time on this one as I struggled with the audio quality and the video was hard to view on Mobile.

**30.** 👴 [Jon Gjengset](https://www.youtube.com/channel/UC_iD0xppBwwsrM9DegC5cQQ) provides a series of videos about Rust concepts, however, these videos are not really entry-level as they assume a certain level of knowledge. These videos would probably be better when you become more comfortable with Rust concepts. That being said, after I got more into Rust, this is a channel that I absolutely adore. In the videos, Jon would pick up a certain topic in Rust and tackle it from almost every angle. Watch out though, the videos can get very long! Probably nothing less than an hour per video. The longest [video](https://www.youtube.com/watch?v=zGS-HqcAvA4&t=40s) is at **7 hours and 20 mins** long!

**31.** 👶 [Code to the Moon](https://www.youtube.com/@codetothemoon/featured) offers nice videos clarifying particular sticky concepts in Rust.

**32.** 👶 [Noboilerplate](https://www.youtube.com/@NoBoilerplate/featured) offers interesting fast technical videos about Rust.

**33.** 👶 [Dcode](https://www.youtube.com/watch?v=vOMJlQ5B-M0&list=PLVvjrrRCBy2JSHf9tGxGKJ-bYAN_uDCUL) offers a playlist for some high-quality Rust Programming tutorials.

**34.** 👶 [300 seconds of Rust](https://www.youtube.com/@300secondsofrust) provides really nice shorts of different Rust concepts. It is great if you want a quick refresher or insight into a particular concept.

**35.** 👶 [The dev method](https://www.youtube.com/watch?v=pGh-0cMvH5g&list=PLAJ-sYO1aGdxQ_skPPtJ7PlSAjTXM-atv) offers a list of Rust videos explaining particular concepts. I've found explanations to be quite clear and enjoyable to follow.

### 4) Git Resources 💾

**36.** [The Rust Programming Language](https://github.com/rust-lang) is where you would find most documentation and guides related to the language itself and its crates/libraries.

**37.** [Awesome Rust](https://github.com/rust-unofficial/awesome-rust) is a great repo with a huge curated list of plenty with Rust code and resources. You can find complete applications in different areas that were built based on Rust.

**38.** [Ferrous Systems](https://github.com/ferrous-systems) is a Rust consultancy and one of the leading contributors to the Rust ecosystem. In their repos, Ferrous provides publicly all the training material (code examples and slides) in their paid private training courses. This also includes their embedded training.

**39.** [The Algorithms](https://github.com/TheAlgorithms/Rust) Rust git repo offers code examples for many essential standard algorithms for data structures, sorting, and strings, among many others. This is a very good resource if you are trying to practice for job interviews.

### 5) Courses 👨‍🏫

**40.** 👶 [First Steps in Rust](https://learn.microsoft.com/en-us/training/paths/rust-first-steps/?WT.mc_id=academic-29077-cxa) is a free course provided by Microsoft providing an introductory view into Rust.

**41.** 👶 [Rust in Motion](https://www.manning.com/livevideo/rust-in-motion) is a paid short video course by Carol Nichols and Jake Goulding. Carol Nichols is the co-author of *The Rust Programming Language* book mentioned earlier. The videos provide some introductory coverage of the basic aspects of Rust.

**42.** 👶 [Rust Essential Training](https://www.linkedin.com/learning/rust-essential-training) is a paid course available on LinkedIn with good reviews. It covers basic aspects of Rust essential for a beginner.

### 6) Cheat Sheets 📃

There are various options, just pick what suits you most

**43.** [The Rust language cheat sheet](https://cheats.rs/)

**44.** [Rust beginner cheat sheet](https://quickref.me/rust)

**45.** [Rust cheat sheet](https://phaiax.github.io/rust-cheatsheet/)

**46.** Let's get rusty cheat sheet which you'd have to sign up for a newsletter [here](https://letsgetrusty.com/) to get.

### 7) Question Forums 🙋‍♂️

Asking Questions is obviously a crucial part of learning. What made it better is how helpful and friendly Rustaceans are. Also for the resources listed below, the awesome part was that you get your questions answered really quick (almost immediately most of the time).

**47.** [The Rust Programming Language Discord](https://discord.com/invite/rust-lang)

**48.** [The Rust Programming Language Community Discord](https://discord.com/invite/rust-lang-community)

**49.** [The Rust Programming Language Subreddit](https://www.reddit.com/r/rust/)

**50.** There is the [Rust Language Forum](https://users.rust-lang.org/) which I personally didn't leverage at all as I found what I need in the previous.

### 8) Podcasts 📻

The following are some useful podcasts that I came across over time that discuss Rust concepts and interesting topics.

**51.** [New Rustacean](https://newrustacean.com/)

**52.** [Rustacean Station](https://rustacean-station.org/)

**53.** [Building with Rust](https://podcasts.apple.com/gb/podcast/building-with-rust/id1553513574)

**54.** [Rust Game Dev](https://rustgamedev.com/)

### 9) Newsletters 📰

To stay informed with everything Rust, these are some newsletters that would be beneficial to subscribe to. The newsletters provide an aggregation every week of everything going around relative to Rust.

**55.** [Awesome Rust Weekly](https://rust.libhunt.com/newsletter?ref=inboxreads)

**56.** [This week in Rust](https://this-week-in-rust.org/?ref=inboxreads)

**57.** [Rust Weekly](https://discu.eu/weekly/rust/?ref=inboxreads)

**58.** [Rust in Blockchain](https://rustinblockchain.org/?ref=inboxreads)

### Conclusion

Learning Rust is no easy feat, as such leveraging as many resources as possible becomes necessary to assert acquired knowledge. In my journey, I've come across many resources that I found useful and summarized in this post. I hope that learners can find it useful. What was your experience like? What were your favorite Rust learning resources? Share your thoughts in the comments 👇.

%%[subend]