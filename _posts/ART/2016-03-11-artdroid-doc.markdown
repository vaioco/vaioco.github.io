---
layout: post
title: ARTDroid Doc
author: vaioco
modified:
categories: ART
excerpt:
tags: [ARTDroid, Android]
image:
  feature:
date: 2016-03-11T18:26:40+01:00
---

## Setup

ARTDroid requires the following dependencies:

* root privilege available
* busybox installed

Push the following files (paths are from repo root dir) to the device:

{% highlight shell linenos=table %}
adb push adbi/hijack/libs/armeabi/hijack /data/local/tmp
adb push examples/classes.dex /data/local/tmp/target.dex
adb push examples/arthook_demo/libs/armeabi/libarthook.so /data/local/tmp
adb push scripts/init.sh /data/local/tmp
adb root
{% endhighlight %}


## How to use

Include the compiled ARTDroid static library in your actually hooking code. In your code call the ARTDroid init function *arthook_init_conf* specifing as its first argument the configuration file.
Let's discuss how to use ARTDroid by means of the demo code included in the directory "example" of this repo.  

Suppose you want to hook the method *android.telephony.TelephonyManager's getDeviceId*. First, write your own patch code using the Java language, the following snippet code is extracted from the example:

{% highlight java linenos=table %}
public String getDeviceId() {
    Log.d(TAG, "FAKE GETDEVICEID ");
    Utils.saveStackTraces();
    String key = "android/telephony/TelephonyManagergetDeviceId()Ljava/lang/String;";
    //Log.d(TAG, "obj =: " + this.toString() + this.getClass().getName());
    Object[] args = {};
    return (String) callOriginalMethod(key, this, args) + "w00t!!";
{% endhighlight %}


Let's discuss the above code (we call it *patch code*). The "key" variable is used to unique identify the target methods in ARTDroid  data structures. It is composed by concatenation of the following values: the Class' name, the method's name and finally its signature. ARTDroid exposes to Java the method *callOriginalMethod*  which permits to call the original method (identified by "key", passing its "args") 
The patch code could contain checks on the input arguments or code to manipulate the object "this" and of course, it could modify the return value or simply return a static value (forever true).

Second, create a JSON formatted file containing the patch code info. In our example we use the following config file:

{% highlight json linenos=table %}

{"config": {
    "debug": 1,
    "dex": "/data/local/tmp/target.dex",
    "hooks": [
    {
        "class-name": "android/telephony/TelephonyManager",
        "method-name": "getDeviceId",
        "method-sig": "()Ljava/lang/String;",
        "hook-cls-name": "org/sid/artdroidexample/HookCls"
    }]
}}
{% endhighlight %}


Finally, to initialize the ARTDroid framework, include in your code a call to function *arthook_init_conf* specifing as its first argument the JSON configuration file. The following example shows how to init the framework specifing "/data/local/tmp/test.json" as the configuration file.

{% highlight c linenos=table %}

configT_ptr configuration =  arthook_init_conf("/data/local/tmp/test.json");
if( configuration == NULL){
    LOGG("ERROR INIT!!\n");
    return;
}
{% endhighlight %}


All calls directed to the method getDeviceId will be redirected first to the patch code, then to the original method by the ARTDroid's core engine.

The shared library produced by compiling the example code can be injected in the target process using the tool _hijack_. Put in mind to disable SELinux before to inject the library, otherwise it will prevent your stuff from working properly.
To inject the compiled library in a target app, execute the following commands in a shell on the device:

{% highlight shell linenos=table %}

cd /data/local/tmp
sh init.sh (run only once to prepare the environment)
sh runhijack.sh -h

{% endhighlight %}


You can inject the hooking library either in a running app or in the zygote master process. The latter approach permits to inject the hooking library in all the future apps forked from the zygote.
Once the hooking library is loaded in the target app, debug/internal log messages are stored in the file /data/local/tmp/artdroid.log and general logs are printed in logcat using the tag "artdroid".


## How to report bugs / suggestions
To report bugs or suggestions, you can use either the github issues system or the ARTDroid google group available at this [address](https://groups.google.com/forum/#!forum/artdroid). Otherwise you can push an email to me at valerio.costamagna\at\di.unito.it. 


