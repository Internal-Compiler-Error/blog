---
title: "Web Is the Best UI Architecture"
date: 2021-12-19T14:30:05-05:00
draft: true
---
# Background
Recently, I had the misfortune of using Java Swing for building a GUI application from an university OOP course.
Since I've written GUI apps before using other languages and frameworks , in true university fashion, I just vomited 
code out the day before it's due and submitted the next day with minimum 
testing. However, I still feel disgusting after the architecture afterwards and generated some thoughts that led to 
me realizing desktop apps are either moving to Electron and React Native or dying a slow death. 

Why are developing UI so much more enjoyable on the browser? The follow will hopefully offer one explanation. 


# Foreword
If you're new to the concepts of concurrency, async, and network protocols, the following will be daunting to you. Fear
not, if you don't get it now, you will get it sooner or later. I often read one concepts explained by multiple different
people/tutorial until it hits me in the right way. Computing is difficult, we all have torn out our hair trying to
understand a concept once 😊. 

It might be several months later when looking at another concept when you finally have the eureka moment, the feeling is
much sweeter than understanding it in first go 🤗.

# Losing Control
One the greatest advantages of console applications (ignoring TUI) in lowering difficulty is you get to control when 
does *input point* appear. By *input point* I refer to sections of your application where you give up control of the 
software and yield control to the user to let them perform certain task. For example, if you prompt a message to the user
and ask them to input something, I would consider that as an input point. 

By only selectively yielding control to the user, we can reason about code much more reasonably. In a lot of cases, 
our apps consists of one giant loop[^1] that asks for inputs and some logics to handle them (which are usually defined) 
separately. We can easily reason about object lifetimes and when the output will be presented to the user since we have
complete control.

Contrasting that to a GUI application, we lose all control over when the control is yield to the user. In a GUI application,
a user may decide to click a button at any time, we can only respond to them rather than proactively carving sections that
are dedicated for handling inputs. In a lot of GUI frameworks, this is done by registering a callback through 
function pointers/listener classes/lambdas. When some action is performed, say a user clicks a button, the callback is 
called **on the GUI thread**. As it turns out, this has huge consequences in keeping UI responsive.

## Don't Block!
If you have user used GUI apps before, doubtless you have encountered sluggish UI interfaces. Yet have you realized most
recent apps feel much smoother and doesn't have the jittery that most early 2010 apps have? 

One huge advancement in realizing how best architecting GUI apps comes from the realization that we should almost do nothing
on the GUI thread. If you have ever written a GUI with handlers, just an interface, you will quickly realize it's insanely
smooth. Updating the UI is a very cheap operation! The sluggishness only comes with we add logic that have nothing to do 
with UI into our UI callbacks.

For example, if you have a button to compute the fibonacci number of 600[^2] on demand, you will see the "down" animation
for the button play, then nothing happens. You can't click any other buttons and the entire apps looks frozen until a short
while when the computation is completed. The reason is the GUI thread whose purpose is only supposed to update the UI is 
forced to do heavy computation, while it's calculating the the `fib(600)`, it can't respond to any user actions. 

We say the thread is ***blocked*** on getting computation. It can't continue until we have the value of `fib(600)` in hand.
Now substitute calcuting the fibonacci number with making a network call, the sluggishness will be entirely unacceptable. 

To write GUI that doesn't feel sluggish, we must *never* block on the GUI thread for non-Ui work.

## One Thread is not Enough
To let our UI thread to only handle UI updates, we must offload the work of calculation to somewhere somehow. The naive
solution is just spin up another thread to do the computation, and tell the GUI thread to update the value when the 
computation is completed. 

This strategy is unsuitable apps with lots of inputs, but the general idea for apps with high inputs is the same---we offload
the work to another thread. 

Now the idea of a *frontend* where it only knows about UI and how to mutate them and a *backend* where all the heavy 
computation or manipulating the states of resources will start screaming to you if you've ever done any web development.
In web apps, the UI and the computation unit are not on the same machine, which implies they are on different threads. 
This is just one of the many advantages of separating frontend from backend.

## How to Talk
By separating our apps to frontend and backend, we reject thinking of monolithic applications and start modularize. 
However, this raises a question---how do we communicate from our UI to inform our request to compute and conversely, 
inform our UI that the result is here to display it? Previously, since our apps is a monolith, the UI callbacks can just 
call the function directly (or spawn a thread to do so).

This is actually a non-trivial problem, and non-trivial problems have non-trivial solutions. To communicate between threads,
you can use message passing or shared memory, to communicate between processes, each operating system provides something different,
we use HTTP with predefined endpoints to exchange information. 

If any of the previous solutions sounded complicated, they are. *Communicating is difficult* (insert CS people joke). 
Of those three, HTTP has the widest support of library make it painless while using channel and Inter-Process-Communication
are all advanced topics requiring lots of experience with working with concurrency and threads. 

In essence, we should all use HTTP to convey request. That's what we use for web apps already! Or as the business people
would call it, the skills transfer!

# A High Performance Event Loop
Now that I have justified why separating UI and logic into not just different classes but different apps are logical, I 
shall also make the claim that JavaScript being a language based on event loops is actually the best choice for UI. 

As the first section states, in GUI apps, we lose out the ability to control when does inputs appear. Since we don't know
when will the results from our backend be delivered, we lose control of that either. 

JavaScript savvy people will already realize these are async events. However, if you have never done JavaScript before,
think of any async events to be an event that:
1. if you started it, you don't know when it will be done
2. if somebody elses started it, you don't know when it will be appeared

Then you'll come to the realization that the ***real world is async***. You don't know when a phone might appear, 
instead of watching for your phone 24/7, you respond to any calls that appear. You don't know when exactly your coffee is
ready, you just wait for a signal. 

Javascript is the right language to model this since all expensive operations (which we mostly care about network calls)
are async! We **don't** want to block, which means we don't want a language that performs all operations on the same
thread. 

Instead of hand writing our poor quality event loop (see footnote 1), we should just piggyback off the huge amount of
engineering that made JavaScript event loop so performant on almost all implementations! This is the entire raison d'être
of JavaScript; there are many things that JavaScript is horrible for but from day one, it was designed for first class
support for assigning functions to variable for easy[^3] callbacks for async operations. AJAX (Asynchronous JavaScript and XML)
first appeared in 1999, when most of the world was still running on Apache HTTP servers which spun up threads for each request[^4].

# Conclusion/Tl;Dr

In the request of finding out what should the optimal architecture for GUI applications be, we naturally end up at writing
a front end using web technologies and communicate to a backend using HTTP for resource exchange.

While this will be resource expensive than a monolithic applications, your sanity is worth more than RAM. Note that the
backend doesn't need to be on a different machine, you can loop back your network requests using `localhost` as the destination; the 
infamous sizes of electron apps mostly comes with bundling a chromium kernel rather than the actual bloat of your software.

[^1]: In effect, this is our handwritten event loop. If you have never heard of event loops before, it shouldn't hurt in understanding the rest
[^2]: Ignore all fancy techniques such as caching, dynamic programming or the Binet's Formula 
[^3]: Yes callback hell *was* real, but with promise or better yet aysnc/await (which is just syntax suger for promises), the feeling is almost natural
[^4]: Spring (which uses Tomcat as the default container) still uses this model 🙃. Although, see [Project Loom](https://openjdk.java.net/projects/loom/)
introducing coroutines into JVM, or use [Kotlin](https://kotlinlang.org/). I hope to make a post about explaining the rationals and implementations of async/await/coroutines in the future
