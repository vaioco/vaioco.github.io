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

```
adb push adbi/hijack/libs/armeabi/hijack /data/local/tmp
adb push examples/classes.dex /data/local/tmp/target.dex
adb push examples/arthook_demo/libs/armeabi/libarthook.so /data/local/tmp
adb push scripts/init.sh /data/local/tmp
adb root
```

## How to use

Include the compile ARTDroid static library in your actually hooking code. In your code call the ARTDroid init function arthook_init_conf specifing as first argument the configuration file.
Let's discuss how to use ARTDroid by means of the demo code included in the directory "example" of this repo.  

Suppose you want to hook the method android.telephony.TelephonyManager's getDeviceId. First, write your own patch code using the Java language, the following snippet code is extracted from the example:

```
public String getDeviceId() {
    Log.d(TAG, "FAKE GETDEVICEID ");
    Utils.saveStackTraces();
    String key = "android/telephony/TelephonyManagergetDeviceId()Ljava/lang/String;";
    //Log.d(TAG, "obj =: " + this.toString() + this.getClass().getName());
    Object[] args = {};
    return (String) callOriginalMethod(key, this, args) + "w00t!!";
```

Let's discuss the above code (we call it patch code). The variable named "key" is used to unique identify the target methods from ARTDroid internal data structures. It is composed by a concatenation of the following values: the Class' name, the method's name and finally its signature. ARTDroid exposes to Java the method callOriginalMethod which permits to call the original method, identified by "key", passing the specified "args". 
The patch code could contain checks on the input arguments or code to manipulate the object "this" and of course, it could modify the return value or simply return a static value (forever true).

Second, create a JSON formatted file containing the patch code info. In our example we use the following config file:

```
{"config": {
    "debug": 1,
    "dex": "/data/local/tmp/target.dex",
    "hooks": [
    {
        "class-name": "java/lang/reflect/Method",
        "method-name": "invoke",
        "method-sig": "(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;",
        "hook-cls-name": "org/sid/artdroidexample/HookCls"
    }]
}}
``

Finally, to initialize the ARTDroid framework include in your code a call the function arthook_init_conf, its first argument is the JSON configuration file. In our example we initialize the framework specifing the /data/local/tmp/test.json configuration file.

```
configT_ptr configuration =  arthook_init_conf("/data/local/tmp/test.json");
if( configuration == NULL){
    LOGG("ERROR INIT!!\n");
    return;
}
```

Now, all calls directed to the method getDeviceId will be redirected first to the patch code, then to the original method by the ARTDroid's core engine.

The shared library produced by building the example code can be injected in the target process using the tool _hijack_. Put in mind to disable SELinux before hooking, otherwise it will prevent your stuff from working properly.
Running the following command in a shell locally on the device, the hooking library is injected in the target process.

```
cd /data/local/tmp
sh init.sh (run only once to set the environment)
sh runhijack.sh -h

```

You can inject the hooking library either in a running app or in the zygote master process. The latter approach permits to inject the hooking library in all the future apps forked from the zygote.
Once the hooking library is loaded in the target app, debug/internal log messages are stored in the file /data/local/tmp/artdroid.log and general logs are printed in logcat using the tag "artdroid".


## How to report bugs / suggestions
To report bugs or consigli, you can use either the github issues system or the ARTDroid google group available at this address. Otherwise you can push an email to me at valerio.costamagna<at>di.unito.it. 


