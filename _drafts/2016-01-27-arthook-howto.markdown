---
layout: post
title: ARTDroid HowTo
author: vaioco
categories: ART
published: false
tags: [ARTDroid, Android]
date: 2016-01-27T20:19:27+01:00
---
## Dependencies

* Android SDK
* Android NDK
* busybox installed on device/emu 
* ADBI from Collin Mulliner (included into ARTDroid as submodule)

## Project structure

* arthook/core/jni : framework core
* examples
	* arthook_demo : user-project example
	* classes.dex : "patch" code
	* app.apk
* docs
* scripts : script for auto install

## Installation

The framework is compiled as static library so it can be directly included into your NDK project. The shared library will be injected into the application's memory (or into zygote) using the tool _hijack_.

To compile ARTDroid you can use the script _build.sh_, it will produce the shared object into _arthook\_demo/libs/_. Push the following files to the device (paths are from repo root dir):

{% highlight bash linenos %}
//pushing hijack from adbi
adb push adbi/hijack/libs/armeabi/hijack /data/local/tmp
adb push examples/classes.dex /data/local/tmp
adb push examples/arthook_demo/libs/armeabi/libarthook.so /data/local/tmp
{% endhighlight %}

create the environment dirs running the following commands:

{% highlight bash linenos %}
cd /data/local/tmp
mkdir -p dex/opt
cp classes.dex dex/target.dex
touch arthook.log adbi.log
chmod -R 777 *
{% endhighlight %}


## Hooking for fun and profit

Hijack allows to inject shared library into target process. As we would like to intercept method calls early as possibile, (from the beginning of the application's life) we have to inject our library directly into the zygote process. By this all future started applications will contains a detoured version of the target methods.

As example we included a demo usage inside the repository, look in "examples/arthook_demo"

Defining new hooks is a very simple task, you have to follow these steps:

1. write your own "patch" code and compile it to DEX

2. create the configuration file defining target methods (defined into the "patch" code)

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
    {
	"class-name": "java/io/FileOutputStream",
	"method-name": "write",
	"method-sig": "([BII)V",
	"hook-cls-name": "org/sid/arthookbridge/HookCls"
    },
    {
	"class-name": "java/lang/String",
	"method-name": "toString",
	"method-sig": "()Ljava/lang/String;",
	"hook-cls-name": "org/sid/arthookbridge/HookCls"
    }]
}}
{% endhighlight %}

3. set the configuration file name before calling ARTDroid entrypoint (refer to arthook_demo.c)

4. shared library injection using hijack
