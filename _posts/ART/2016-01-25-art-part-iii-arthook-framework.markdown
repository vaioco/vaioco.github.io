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

{% comment %}
	la ciccia in culo
{% endcomment %}

What is [ARTHook](https://github.com/vaioco/art-hooking-vtable)? It is an easy-to-use framework for hooking virtual methods calls under ART runtime. 

The thing is: you can override any virtual methods with your own one and thus hooking on every method call. ARTHook is based on library injection and hooking virtual methods by vtable tampering. 
Imagine you want to intercept calls to a virtual method. You have to define your own Java method and by using ARTHook API you can override the target method. All future calls to the target method will be intercepted and they will go to your own Java method.

What are the differences from state of the art?
ARTHook has various advantages respects to other projects like "APIMonitor", "DroidBox", "Cuckoo-Droid", "Xposed" :

1. you don't have to modify the target application's code
2. you don't have to modify the Android framework
3. it works on real-world devices as well
4. it works on Marshmallow

Target application's code is unmodified allowing to analyze applications which uses _integrity checking_ techniques, without needing of a manual reverse-engineering effort to disable integrity checks. 

Moreover, you don't need to modify the Android framework or rebuild AOSP or any of its components.
Lastly you are able to implement the analysis system on real-world devices as well. This feature is realy useful, expecially when dealing with malware. As the techniques for Android malware detection are progressing, malware also fights back through deploying advanced code encryption and emulation detection techniques ([Rage against the virtual machine: hindering dynamic analysis of android malware.](http://www.cs.columbia.edu/~mikepo/papers/ratvm.eurosec14.pdf))

The ARTHook furter supports loading "patch" code (code which override target method) from DEX file. This enable the "patch" code to be written in Java and thus semplifies interacting with the target application and the Android framework (Context, etc...).

ARTHook uses [ADBI]() from Collin Mulliner and requires the root privilege on the device.

## How does ARTHook works? ##

So far I introduced how the ART internals works and what is the code-path called for lookup virtual-methods. Let's go into the framework details.

First challenge I had to solve was how to retrive the target method's reference from virtual memory. It was achieved using the Java Native Interfaces (JNI) facilities calling the function _GetMethodID_. Once we have the target method memory reference we can parse the _ArtMethod_ structure from memory and access to its useful elements, like **declaring_class_**. 

Next question was, how do I retrive the _vtable\__ from within the Class data structure saved in virtual memory? Well, I got it using relative offsets starting from the Class' memory address. We have seen before the data structure layout, using a debugger (or reading the sources) we can retrive the offset for accessing the variable inside the **Class** data structure.


The last step is load our own method implementation into the application's memory. We use the DexClassLoader using a DEX file as container. Finally we just need to change the target method's pointer make pointing it to your own method.

ARTHook consists of three component:

* core
* Java API bridge
* Java "patch" code (defined by users)

The core is written in C. The API bridge permits the communication with the core from the user-defined Java "patch" code. Users can defines they own "patch" methods using Java language, this facility permits to use Android API inside the "patch" method code.

Let's start explaining the Java API bridge.

* _callOriginalMethod_: allows to call the original method. Its signature is:

{% highlight Java linenos %}
public static native Object callOriginalMethod(String key, Object thiz, Object[] args);
{% endhighlight %}

First argument is the 'unique key' used to identify original method from dictionary data structure, the second one is the _this_ object and the last one is the arguments array. 

Suppose the method you want to hook is _GetDeviceID_ within _TelephonyManager_ class, the key used by the framework to identify that method is: XXX
Basically, it is the concatenation of classname, methodname and method signature.

The "patch" code contains the alternative code to execute when a call is hooked, users can define they own "patch" code. The following is an example of a "patch" method for _getDeviceId_ API method :

{% highlight Java linenos %}
package org.test.patchcode

// import Java API bridge
import org.sid.arthook.javabridge.*

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

Target methods are defined by JSON configuration file:

{% highlight json linenos %}
{"config": {
    "debug": 1,
    "dex": "target.dex",
    "hooks": [
    {
	"class-name": "android/telephony/TelephonyManager",
	"method-name": "getDeviceId",
	"method-sig": "()Ljava/lang/String;",
	"hook-cls-name": "org/sid/arthookbridge/HookCls"
    },
	[...]	
{% endhighlight %}

Line 2 switch on debug logs, line 3 is the DEX file with the "patch" methods and the "hooks" element is a list of target methods, where:

* class-name : class name which contains target virtual method
* method-name : target method name
* method-sig : target method signature
* hook-cls-name : "patch" code class name

Click [here!]({% post_url /ART/2016-01-27-arthook-howto %}) for an easy to follow step-by-step instructions for howto build and use ARTHook.