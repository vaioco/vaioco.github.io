---
layout: post
title: ART Part III - ARTHook Framework
modified:
categories: ART
excerpt:
tags: [ART, Android, ARTHook]
image:
  feature:
date: 2016-01-25T16:40:36+01:00
---

### ARTHook ###

What is ARTHook? It is an easy-to-use library for hooking virtual methods call under ART runtime. The thing is: you can override any virtual methods with your own one and thus hooking on every method call. 
Imagine you want to intercept calls to a virtual method. You have to define your own Java method and by using ARTHook API you can override the target method. All future calls to the target method will be intercepted and they will go to your own method.

What is the differences with state of the art?
ARTHook has various advantages respects to other projects like "APIMonitor", "Xposed", "DDI" :

1. you don't have to modify the target application's code
2. you don't have to modify the Android framework
3. it works on real-world devices as well
4. it works with Marshmallow
5. there is no race-windows where calls can bypass the hook

The first permits to analyze also applications which uses tampering protection. This kind of protections allows an application to discover if it was tampered by an attacker, modify the classes.dex by inserting code will be defeated by that protections.

Moreover, you don't need to modify the Android framework recompiling AOSP from source or similar things.
Finally you are able to implement the analysis system on real-world devices as well.

All the things you need are:

* your own method implementation
* root privilege on the target device

## How does ARTHook works? ##

So far I introduced how the ART internals works and what is the lookuping method used for calling virtual-methods. Let's go into the deep details of the framework.

First challenges I had to solve was how to retrive the target method's reference from virtual memory. It was achieved using the Java Native Interfaces (JNI) facilities calling the function _GetMethodID_.
Next question was, how do I retrive the _vtable\__ from within the Class data structure saved in virtual memory? Well, I got it using relative offsets starting from the Class' memory address. We have seen before the data structure layout, using a debugger (or reading the sources) we can retrive the offset for accessing the variable inside the data structure.
Once we have the target method memory reference we can parse the _ArtMethod_ structure from memory and access to its useful elements. The last step is loading into the application's memory our own method implementation using a DEX file as container. Finally we just need to change the target method's pointer make pointing it to your own method.

ARTHook is composed by three elements:

* core
* Java API bridge
* Java patch code

The core code is written in C and the other parts are in Java. The API bridge permits the communication with the core from the user-defined Java patch code.
