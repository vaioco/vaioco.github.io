---
layout: post
title: ART Part III - ARTHook Framework
author: vaioco
modified:
categories: ART
excerpt:
tags: [ART, Android, ARTHook]
image:
  feature:
date: 2016-01-25T16:40:36+01:00
---

What is [ARTHook](https://github.com/vaioco/art-hooking-vtable)? It is an easy-to-use framework for hooking virtual methods call under ART runtime. The thing is: you can override any virtual methods with your own one and thus hooking on every method call. 
Imagine you want to intercept calls to a virtual method. You have to define your own Java method and by using ARTHook API you can override the target method. All future calls to the target method will be intercepted and they will go to your own method.

What are the differences from state of the art?
ARTHook has various advantages respects to other projects like "APIMonitor", "DroidBox", "Cuckoo-Droid", "Xposed" :

1. you don't have to modify the target application's code
2. you don't have to modify the Android framework
3. it works on real-world devices as well
4. it works with Marshmallow
5. there is no race-windows where calls can bypass the hook

The first permits to analyze also applications which uses tampering protection. This kind of protections allows an application to discover if it was tampered by an attacker, modify the classes.dex by inserting bytecode will be defeated by that protections.

Moreover, you don't need to modify the Android framework or recompiling AOSP from source.
Finally you are able to implement the analysis system on real-world devices as well.

ARTHook is based on library injection and hooking virtual methods by vtable tampering.

All the things you need are:

* "patch" method implementation
* root privilege on the target device

## How does ARTHook works? ##

So far I introduced how the ART internals works and what is the code-path called for lookup virtual-methods. Let's go into the framework details.

First challenge I had to solve was how to retrive the target method's reference from virtual memory. It was achieved using the Java Native Interfaces (JNI) facilities calling the function _GetMethodID_. Once we have the target method memory reference we can parse the _ArtMethod_ structure from memory and access to its useful elements, like **declaring_class_**. 

Next question was, how do I retrive the _vtable\__ from within the Class data structure saved in virtual memory? Well, I got it using relative offsets starting from the Class' memory address. We have seen before the data structure layout, using a debugger (or reading the sources) we can retrive the offset for accessing the variable inside the **Class** data structure.


The last step is load our own method implementation into the application's memory. We use the DexClassLoader using a DEX file as container. Finally we just need to change the target method's pointer make pointing it to your own method.

ARTHook is composed by three elements:

* core
* Java API bridge
* Java "patch" code

The core is written in C and the other parts are in Java. The API bridge permits the communication with the core from the user-defined Java patch code. Users can defines they own "patch" methods using Java language, this facility permits to use Android API inside the "patch" method code.

Let's start explaining the Java API bridge.
At this time there is only one exposed API: _callOriginalMethod_ calls the original method. Its signature is:

{% highlight Java linenos %}
public static native Object callOriginalMethod(String key, Object thiz, Object[] args);
{% endhighlight %}

The first argument is the 'unique key' used to identify the hooked method's original implementation to call, the second one is the _thiz_ object and the last one is the arguments array. Suppose the hooked method is _GetDeviceID_ within _TelephonyManager_ class, the unique key used by the framework to identify that method is: XXX
Basically, it is the concatenation of classname, methodname and method signature.

The "patch" code contains the alternative code to execute when a call is hooked, users can define they own "patch" code and use any Android API (also methods which are hooked).

{% highlight Java linenos %}
package org.test.patchcode

public class MyPatchCode{
	public String getDeviceId(){
		String key = ""; //dictionary key
		Object[] args = {}; //method's args
		// adding custom string to original result
		// note: "this" will be taken from stackframe, it will be a "TelephonyManager" obj
		return (String) callOriginalMethod(key, this, arg) + "w00tw00t!!" ;
	}
}
{% endhighlight %}

To define target methods to hook, users have to write a JSON formatted configuration file like the following one:

{% highlight json linenos %}
<config>
</config>	
{% endhighlight %}

Click [here]() for an easy to follow step-by-step instructions for howto build and use ARTHook.