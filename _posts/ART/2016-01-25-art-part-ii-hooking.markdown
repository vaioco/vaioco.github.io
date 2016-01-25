---
layout: post
title: ART Part II - Hooking
modified:
categories: ART
excerpt:
tags: [android, ART]
image:
  feature:
date: 2016-01-25T00:38:11+01:00
---

### Hooking virtual methods for profit ###

What do you need for hooking Java virtual methods? Suppose you want to hook the method **A** within class **Z** and divert the control flow
to the new method **B**, the "patch" method. You have to:

1. load **B** method into application's memory 
2. retrive **A**'s reference from memory and its position inside the **Z**'s vtable array
3. change the pointer to **A** within **Z**'s vtable and make it pointing to **B**

Using the _GetMethodID_ function of java native Interface (JNI) we can get the **A**'s memory reference. The _GetMethodID_ function returns an _jobject_ type, it is just an alias for the **ArtMethod** data structure.

Using relative offset, we can access to elements inside the object returned by _GetMethodID_ for retriving the information needed for locating **A**'s memory reference within the **Z**'s **vtable** array. ART's internals are very unliked the subject of OEM modifications.

To achieve point 1 we use the DexClassLoader API, the method **B** is defined by the user and loaded from a DEX file.

Once we have got the information scanning the memory, we can get point 3 using just simple memory operations by native code.

Starting from the **A**'s memory reference we can parse the _ArtMethod_ structure from memory and get the following elements:

1. declaring_class_
2. method_index_

The first is a reference to **Z** (a **Class** object), that is the class which contains the method **A**. The latter is the **A**'s index value inside the vtable.

Using the _declaring\_class\__ reference we can parse the **Class** structure from memory and get the following information:

* vtable_
* virtual_methods_