---
title: "Learning Rust: My 6 Key Moments"
datePublished: Mon Mar 21 2022 18:47:02 GMT+0000 (Coordinated Universal Time)
cuid: cl1127wsx0076kenv7kqc76vv
slug: learning-rust-my-6-key-moments
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1646857880249/kxXox7mBc.png
tags: beginner, beginners, programmer, rust, programming-languages

---

If you’ve tried learning Rust or looked into it at least, you might have read somewhere how it has a steep learning curve. For me personally coming from a background of writing code in several other languages (mainly C), I found that to be somewhat true. Actually, I also think that some of my background in computer systems at a low level helped me grasp Rust faster. Regardless of all that, it was still an amazing journey. I found myself quite often thinking how much Rust makes sense. Especially regarding how the language handles matters from a system perspective. You can read more about what I liked in my last blog post ["5 Things I Loved About Learning Rust"](https://apollolabsblog.hashnode.dev/5-things-i-loved-about-learning-rust).

It’s worth pointing out that my personal goal in learning Rust was for using it in Embedded development. For my purposes, probably going up to chapter 6 or 7 in [“The Book”](https://doc.rust-lang.org/book/) would have been sufficient. Still, for me, I really wanted to go further given how much I found myself attracted to the language and curious in understanding its depth.

With that being said, below is my list of 6 things that, throughout my Rust learning journey, I felt I had a key moment once understanding. Mind that these concepts are probably not the toughest to understand in Rust, moreover, most were probably mentioned at some point in the resources I leveraged. However, I felt these concepts had a key impact on my understanding because they often were subtly explained though commonly used.

### 1) It’s all about the References!
Almost any Rust learner (and probably developer) can tell you about their struggles dealing with the borrow checker. Ownership and borrowing are really powerful concepts in Rust yet they take some effort and focus to understand properly. I would add that its probably even one of the things that when you consistently nail, you feel like you have become some sort of invincible programmer. Well, thanks to a series of videos about Rust by [Doug Milford](https://www.youtube.com/channel/UCmBgC0JN41HjyjAXfkdkp-Q) I found on YouTube, I got much-needed clarification. In Doug's video ["Rust Ownership and Borrowing"](https://www.youtube.com/watch?v=lQ7XF-6HYGc&list=PLLqEtX6ql2EyPAZ1M2_C0GgVd4A-_L4_5&index=13) he demonstrates quite a few examples about borrowing. My initial understanding was that all values can be owned by one variable, and cannot be passed around unless they are cloned or borrowed. If not borrowed, they would be dropped at the end of the scope they are used. It turned out that it's not always true, the exception being fixed-size variables that are created in the stack (i.e. non-pointer local variables). In Rust terminology, fixed-size variables (ones that are created in the stack) implement a copy Trait that allows them to behave in such a manner. This made a lot of sense at that point because when passing non-pointer local variables that reside on the stack, they are passed by value (copied/cloned) to the called function, via the activation record, and are erased when the function ends anyway (so why worry about borrowing?).  To demonstrate a simple example, let's say we have something as follows:
```
fn main() {
    let some_string = String::from("hello");
    let other_string = some_string;
    println!("{}", some_string);
}
```
This is something that you learn you cannot do in Rust. You get a compile error because there can be only one owner to the `String`.  If Rust had allowed this it would mean that we would have two variables `some_string` and `other_string` pointing at the same `String`, a big NO-NO. You might ask why? because if it's allowed it means that potentially two different variables can change the same location, a disaster waiting to happen in parallel programming. So, based on that, now you would think that the following code would also generate a compile error:
 ```
fn main() {
    let a = 10;
    let b = a;
    println!("{}", a);
}
```
Interestingly enough it does not. But, why?! It turns out because there are no pointers involved and both variables are on the stack, we are simply creating a copy of the value of `a` in `b`. We don't have multiple pointers pointing at one location that can change one value, and thus no potential crisis. Well, this makes a lot of sense!

In a way, while it made sense the latter is the type of code a non-Rust programmer would be used to. When I went back to check if I missed anything, it turns out that this exception is mentioned at some point in Chapter 4 of [“The Book”](https://doc.rust-lang.org/book/). I felt it wasn’t emphasized enough. The thing is that at the onset of the chapter, the ownership rules are mentioned from the get-go and somehow my thinking got consumed about applying the rules to everything going forward. However, as explained, more or less, while ownership still applies there is a "wrinkle" when it comes to fixed-size variables in the stack.

I guess this might be one of the steep learning curve contributors IMO. The part that most coders are used to as the norm comes as an exception. A better approach could probably starting by showing the things that are the same and then bringing in the Rust dealings.

### 2) `::` Operator vs. `.` Operator
While navigating through the concepts and writing new code, I often encountered the `::` and `.` operators in a way that felt they were used interchangeably. I couldn’t really figure out right away the pattern of when I should use which. I would say that the `.` operator was easier to grasp as it was used in a similar way to traditional programming languages I was used to, essentially to call methods and access struct members.
The `::` operator was more confusing though. From my not so vivid memory about C++, I recall that a similar operator was used in namespaces for scope resolution (not that I ever was fond of C++ namespaces to start with 😁). Though the main question I had is that `::` sometimes was used in a manner where I would instead expect a `.` to be used. It turns out that there are two types of methods; methods that belonged to an instance of a type and methods that belonged to the type itself. So for example, if the type is instantiated then we would use the `.` operator for all the instance methods. Though if the type is not instantiated there are methods that we would call using the `::` operator. Examples of usage can be seen commonly in strings as follows:
```
 let h = String::from("Hello world!");
``` 
so here `from`, is a method that belongs to the `String` type itself, thus using the `::` operator. Following that, after we have instantiated `h`, we can now call instance methods using the `.` operator. For example:
```
println!("{} is {} letters long", h, h.len());
```
In this case, `len()` is an instance method.
Interestingly enough this is still not where things stopped for the `::` operator. `::` is also used to reference module paths when importing libraries with the `use` keyword. For example, we can import the `PI` identifier by referencing its module paths with the `use` keyword as follows:
```
use std::f64::consts::PI;

fn main() {
        println!("Pi = {}", PI);
}
```
Was this the end of it for the `::` operator? You might have guessed it, the answer is still no 😆 Check point number 4 about turbofish for one more use. Though in that case, it combines with another operator to form something new.

###  3) The Exclamation Mark `!` 
Early on in learning Rust, I encountered the `!` often, starting with `println!`, followed by `panic!` and later `vec!`. Although the resources referred to `println!`, `panic!`, and `vec!` as macros, I did not realize until later that all identifiers that end with an `!`, are by definition macros. It turns out that the `!`  is part of the invocation syntax required to distinguish a macro from an ordinary function. Macro names if you aren’t familiar are replaced by code that is generated statically at compile time rather than called dynamically during runtime like a regular function. Makes for faster execution.

### 4) The Turbofish Operator
Generics are quite an interesting and powerful concept in Rust. In short, generics allow you to declare a general type for an enum or struct that can then be inferred by the compiler at compile time. They look something like this:

```
struct MyStruct<T> {
    item: T,
}
```
Here in the definition of `MyStruct`, we are not defining a type. We are saying that it is a generic type `T`. After that, at compile time the Rust compiler would figure out on its own from a declaration of `MyStruct` what the type is and fill it in. 
My issue was that in some cases I saw odd-looking declaration examples that looked something like this:
```
let var = MyStruct::<i32> { item: 3 };
```
It made me wonder where the `::<T>` operator business is coming from (adding more confusion to point number 2 with the `::`). It turns out that this is called a turbofish operator. It is used when the compiler isn't sure about the type you want to infer and needs your help as a programmer to help inform about the type.

### 5) The `?` Operator
Regarding points 5 and 6, these are actually some of the things I think contribute to a steep learning curve in any language. The part where shorthand notation is brought into the picture early on in the learning process. I truly believe that in beginner learning resources shorthand shouldn't be used frequently or at least used alongside non-shorthand notation with constant reminders of how it works. Better yet, I would go as far as to say that shorthand is better left till much later as a tip to enhance coding style.
Now that I'm done with my short rant this brings us to the `?` operator. In certain instances, I would encounter code that looked like this:
```
let x = function_call()?;
```
It turns out that this is directly related to the `Result` enum. I'm not going to get into too much detail, but in learning Rust you'll know that `Result` is a built-in generic enum that allows a programmer to return a value that has the possibility of failing. It is the way the programming language does error handling. So after some searching, what the `?` operator turned out to be is shorthand for pattern matching a `Result` and is the equivalent of doing this:
```
let x = match function_call() {
    Ok(x) => x,
    Err(e) => return Err(e),
}
```
So as a result, after deconstruction, `x` would either contain the Ok value from the Result of the function call, or the value from the Err variant is returned and x is never assigned.

### 6) `if let` and `while let`
While `if let` was introduced in section 6.3 of "The Book", it took a while for me to digest. Somehow it felt like it got lost in the grand scheme of things (already done with my shorthand rant 😄). What it turned out to be in simple terms is a more concise way for pattern matching an `Option` enum. Similar to the  `Result` enum idea and errors, Rust represents nullable values without using `null` but rather the generic enum `Option`. Typically, in deconstructing an `Option`, we would do something like this:
```
    match res {
        Some(x) => println!("value is {}", x),
        None ()=> (),
    }
```
With `if let` we can instead do this:
```
    if let Some(x) = res {
         println!("value is {}", x);
    }
```
The way this reads is that, if `res` is equal to (or matches) `Some(x)`, then execute the `println!`, otherwise, do nothing. Note here that, when going for `if let`, we are doing nothing if `res` is equal to `None`. On the other hand, `while let` follows a similar approach, though the difference is introducing a condition that you want to loop as long as a value matches a certain pattern.

## Conclusion
Learning Rust is not the easiest feat, though I personally found it to be most satisfying and really enjoyable. Especially when you try to understand what happens at the low level, you start having a lot of these "Well, that makes a lot of sense!" moments. Still, sometimes I found things to be a bit overwhelming and not clarified in a way I expected. It could be that I'm always trying to relate in my mind to other languages I learned in the past. Though the 6 key moments I mentioned, were probably the most transforming in my journey. What was your experience like? What were your key moments with Rust? Share your thoughts in the comments 👇.  If you found this useful, make sure you subscribe to the newsletter [here](https://subscribepage.io/apollolabsnewsletter) to stay informed about new blog posts. Also, make sure to check out our social channels [here](https://linktr.ee/apollolabs.bin)