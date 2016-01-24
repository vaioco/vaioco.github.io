---
layout: post
title: ART Part II - Hooking
modified:
categories: ART
excerpt:
tags: []
image:
  feature:
date: 2016-01-25T00:38:11+01:00
---

### Hooking virtual methods for profit ###

What do you need for hooking Java virtual methods? Suppose you want to hook the method **A** within class **Z** and divert the control flow
to the new method **A'**, the "patch" method. You have to:

1. load **A'** method into application's memory 
2. retrive **A**'s reference from memory and its position inside the **Z**'s vtable array
3. change the pointer to **A** inside **Z**'s vtable and make it pointing to **A'**

Using the _GetMethodID_ function of java native Interface (JNI) we can get the **A**'s memory reference. The _GetMethodID_ function returns an _jobject_ type, it is just an alias for the **ArtMethod** data structure.

Using relative offset we can access to elements inside the returned ArtMethod for retriving the information needed for locating **A**'s memory reference within the **Z**'s **vtable** array. Relative offsets can be obtained from a debbugging session with symbols. ART's internals are very unliked the subject of OEM modifications.

To achieve point 1 we use the DexClassLoader API, the method **A'** is defined by the user and loaded from a DEX file.

Once we have got the information scanning the memory we can get point 3 using just simple memory operation from native code.


{% highlight C linenos %}
754  bool ShouldHaveEmbeddedImtAndVTable() SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
755    return IsInstantiable();
756  }

454  bool IsInstantiable() SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
455    return (!IsPrimitive() && !IsInterface() && !IsAbstract()) ||
456        (IsAbstract() && IsArrayClass());
457  }
458
{% endhighlight %}