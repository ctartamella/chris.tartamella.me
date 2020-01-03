---
layout:     post
title:      What Is Node.js?
date:       2016-10-12 22:57:00
summary:    Cutting through the lingo....
tags:       node javascript backend server
thumbnail:  jekyll
---

*The following is cross-posted with permission of my employer and is an example of some of the content*
*we are producing over at [Anexinet](http://www.anexinet.com/blog/).  Please check it out.*

It's not uncommon in the IT industry to hear people rave about the next big thing and subsequently to 
sit there wondering, "Just what the heck is it and why do I care?"  Node.js certainly seems to be one 
of those technologies, and frankly, their own answer to this question doesn't help:

> "Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine. Node.js uses an event-driven, 
> non-blocking I/O model that makes it lightweight and efficient. Node.js' package ecosystem, npm, is the 
> largest ecosystem of open source libraries in the world." - nodejs.org

And we still wonder....
So, let's break it down.  There seems to be 3 main pieces to the above statement.
1.	"… is a JavaScript runtime built on Chrome's V8 JavaScript engine...."
2.	"… uses an event-driven, non-blocking I/O model...."
3.	"… package ecosystem, npm, is the largest ecosystem of open source libraries in the world."

JavaScript meets Server
===============================
Unlike many computer languages which are compiled, JavaScript is an interpreted language.  This means 
that rather than feeding code through a program that generates a binary, JavaScript code runs through 
an interpreter which executes the code in real time. Any application which wises to run JavaScript 
code needs such an interpreter.  There are many available, but one of the most popular ones is Chrome's 
V8 engine.  This engine's SOLE job is to execute JavaScript code.  As such, it is not JavaScript code 
itself (for the most part) but rather compiled C++ code.  This helps to ensure it can run as fast as 
possible on the hardware it is compiled for.

To implement JavaScript, applications (like web browsers) include an engine as a library.  For example, 
Google's Chrome browser includes the V8 engine.  When the web browser needs to execute JavaScript, 
they pass it on to the V8 engine to do so.

What does this mean for node.js?  Well node.js is simply another program which includes the V8 engine 
as a library.  That gives it the ability to execute JavaScript code.  Like V8, node is written in C++ 
to help with performance.

One of the main benefits of doing this is the ability to modify the JavaScript language.  V8 provides 
extension points allowing implementers to add features to the language as they wish.  Node.js takes 
advantage of this to create its own API that adds many platform-specific functions to a language 
that normally would not allow this.  Node has the ability to access the file system, initiate and 
receive HTTP requests, and manage OS processes.  JavaScript normally does not allow for this, but 
by extending V8 node is able to allow this.

By including the V8 engine, or rather building on top of it as the website states, node.js becomes 
a viable method of running JavaScript directly on hardware.  This opens up the ability to do so as
a server backend, and indeed that is its chief purpose.

But servers need to scale?
================================
JavaScript by itself is a synchronous language.  It allows for callbacks to be used easily, but code 
is still executed synchronously within the interpreter.  This is problematic in a server environment 
where your app needs to handle many requests simultaneously.  This is where the second point from 
above comes into play.

Node.js uses the libuv library to implement an event loop.  This loop is constantly monitoring an 
event queue and processing entries off that queue in a synchronous matter.  However, items can be 
pushed onto that queue in an asynchronous manner from the C++ portions of node and the V8 engine.  
So, incoming network requests coming from the operating system can be processed asynchronously, 
put on to the event queue where node processes them one at a time.

![Node.js event queue](/images/nodeEventQueue.png)

A package manager to die for.....
=================================
You may think I’m exaggerating but you couldn’t be more wrong.  Npm is the package manager developers 
wish they had on every platform.  A single file (package.json) contains everything you need to define 
a project and developers can easily add and remove dependencies.  It even enables simple project 
scripting to define your build process.  Npm is so well designed, in fact, that its not uncommon to 
see non-node Java-Script projects using it.  If you’re doing JavaScript, and you aren’t using npm 
you are doing it wrong.