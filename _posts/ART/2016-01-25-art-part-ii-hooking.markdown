---
layout: post
title: ART Part II - Hooking
author: vaioco
modified:
categories: ART
excerpt:
tags: [android, ART]
image:
  feature:
date: 2016-01-25T00:38:11+01:00
---

### Hooking virtual methods for fun and profit ###

What do you need for hooking Java virtual methods? 
Suppose you want to hook the method **A** within class **Z** and divert the control flow
to the new method **B** ( the "patch" method ) within class **X**. At least you have to:

1. load **X** into application's memory 
2. retrive **A** reference from memory and its position inside the **Z** vtable
3. modify the **A** entry within **Z** vtable with **B** address


Using the _GetMethodID_ function of Java Native Interface (JNI) we can obtain the **A** memory reference (suppose **A** is already loaded in memory,true for Android APIs). The _GetMethodID_ function returns an _jobject_ type, which is just an alias to **ArtMethod** data structure.

Using relative offset, we can access to elements inside the ArtMethod object returned by _GetMethodID_ for retriving the information needed to retrive **A** index value inside **Z** **vtable**. Relative offset approach is stable and reliable because ART internals are very unlikely to be subject of OEM modifications.

To achieve point 1 we use the _DexClassLoader_ API, the "patch" method **B** is defined by the user and loaded from a DEX file.

Once we have got the information scanning the memory, we can achieve point 3 using just simple memory operations by native code.

Using the **A** memory reference returned by _GetMethodID_ , we can parse the _ArtMethod_ structure and access to its following elements:

1. **declaring_class_**
2. **method_index_**

The former is a reference to **Z** (a **Class** object). This class contains the method **A**. The latter is the **A**'s index value inside **Z** vtable.

Following the _declaring\_class\__ pointer, we can parse the **Class** object from memory to access its following elements:

* **vtable_**
* **virtual_methods_**

## How does the runtime is looking up virtual methods? ##

First let's try to follow the flow of a Java method called by JNI. As example we take in account the _CallObjectMethod_ function, defined in [jniinternals.c](http://androidxref.com/6.0.1_r10/xref/art/runtime/jni_internal.cc#680). This function is used for calling Java methods from native code, exactly only methods which returns an Object type.

(All following source code is taken from [androidxref](https://androidxref.com) Android 6.0.1_r10 )

{% highlight C %}
680  static jobject CallObjectMethod(JNIEnv* env, jobject obj, jmethodID mid, ...) {
681    va_list ap;
682    va_start(ap, mid);
683    CHECK_NON_NULL_ARGUMENT(obj);
684    CHECK_NON_NULL_ARGUMENT(mid);
685    ScopedObjectAccess soa(env);
686    JValue result(InvokeVirtualOrInterfaceWithVarArgs(soa, obj, mid, ap));
687    va_end(ap);
688    return soa.AddLocalReference<jobject>(result.GetL());
689  }
{% endhighlight %}

The called function _InvokeVirtualOrInterfaceWithVarArgs_ (line 686) is defined in [reflection.cc](http://androidxref.com/6.0.1_r10/xref/art/runtime/reflection.cc#529):


{% highlight C %}
529JValue InvokeVirtualOrInterfaceWithVarArgs(const ScopedObjectAccessAlreadyRunnable& soa,
530                                           jobject obj, jmethodID mid, va_list args) {
	 [...]
539  mirror::Object* receiver = soa.Decode<mirror::Object*>(obj);
540  ArtMethod* method = FindVirtualMethod(receiver, soa.DecodeMethod(mid));
541  bool is_string_init = method->GetDeclaringClass()->IsStringClass() && method->IsConstructor();
	 [...]
547  uint32_t shorty_len = 0;
548  const char* shorty = method->GetInterfaceMethodIfProxy(sizeof(void*))->GetShorty(&shorty_len);
549  JValue result;
550  ArgArray arg_array(shorty, shorty_len);
551  arg_array.BuildArgArrayFromVarArgs(soa, receiver, args);
552  InvokeWithArgArray(soa, method, &arg_array, &result, shorty);
     [...]
557  return result;
558}
{% endhighlight %}

At line 540 is retrived the _ArtMethod_ identified by the _jmethodID_ passed as third argument, the _FindVirtualMethod_ function is defined in _reflection.cc_
The real method invocation is the call to _InvokeWithArgArray_ at line 552.

The _FindVirtualMethod_ function is used to look up a virtual method inside its _receiver_ object, following box shows the code.

{% highlight C %}
420static ArtMethod* FindVirtualMethod(mirror::Object* receiver, ArtMethod* method)
421    SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
422  return receiver->GetClass()->FindVirtualMethodForVirtualOrInterface(method, sizeof(void*));
423}
{% endhighlight %}

The above code calls function _FindVirtualMethodForVirtualOrInterface_ (line 422) defined in [class-inl.h](http://androidxref.com/6.0.1_r10/xref/art/runtime/mirror/class-inl.h#399) to look up the target _method_.
The following box show _FindVirtualMethodForVirtualOrInterface_ code

{% highlight C %}
399 inline ArtMethod* Class::FindVirtualMethodForVirtualOrInterface(ArtMethod* method,
400                                                                size_t pointer_size) {
401  if (method->IsDirect()) {
402    return method;
403  }
404  if (method->GetDeclaringClass()->IsInterface() && !method->IsMiranda()) {
405    return FindVirtualMethodForInterface(method, pointer_size);
406  }
407  return FindVirtualMethodForVirtual(method, pointer_size);
408 }
{% endhighlight %}

_FindVirtualMethodForVirtual_, defined in _class-inl.h_, returns the _method_ scanning the **vtable_** array.

{% highlight C %}
387 inline ArtMethod* Class::FindVirtualMethodForVirtual(ArtMethod* method, size_t pointer_size) {
388  DCHECK(!method->GetDeclaringClass()->IsInterface() || method->IsMiranda());
389  // The argument method may from a super class.
390  // Use the index to a potentially overridden one for this instance's class.
391  return GetVTableEntry(method->GetMethodIndex(), pointer_size);
392 }
{% endhighlight %}

Analyzing the _GetVtableEntry_ code, still defined in _class-inl.h_, we can see a call to function _ShouldHaveEmbeddedImtAndVTable_. If the returned result is _True_ the vtable is embedded within the class, otherwise it is retrived calling the function _GetVTable()_ (line 184)

{% highlight C %}
180inline ArtMethod* Class::GetVTableEntry(uint32_t i, size_t pointer_size) {
181  if (ShouldHaveEmbeddedImtAndVTable()) {
182    return GetEmbeddedVTableEntry(i, pointer_size);
183  }
184  auto* vtable = GetVTable();
185  DCHECK(vtable != nullptr);
186  return vtable->GetElementPtrSize<ArtMethod*>(i, pointer_size);
187}
{% endhighlight %}

So, what do the function _ShouldHaveEmbeddedImtAndVTable_ do?

{% highlight C %}
754  bool ShouldHaveEmbeddedImtAndVTable() SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
755    return IsInstantiable();
756  }

454  bool IsInstantiable() SHARED_LOCKS_REQUIRED(Locks::mutator_lock_) {
455    return (!IsPrimitive() && !IsInterface() && !IsAbstract()) ||
456        (IsAbstract() && IsArrayClass());
457  }
{% endhighlight %}

The _GetVTable_ code is showed in following box:

{% highlight C %}
138 inline PointerArray* Class::GetVTable() {
139   DCHECK(IsResolved() || IsErroneous());
140   return GetFieldObject<PointerArray>(OFFSET_OF_OBJECT_MEMBER(Class, vtable_));
141 }
{% endhighlight %}

Ok now we know that virtual methods called by JNI are looked up using the **vtable_** array (embedded or not). 

Let's me try to figure out how the runtime invoke methods called by reflection.
The function involved in reflection calls is _InvokeMethod_ defined in [reflection.cc](http://androidxref.com/6.0.1_r10/xref/art/runtime/reflection.cc#560). The target method to invoke is looked up using the following code (line 600):

{% highlight C %}
586  if (!m->IsStatic()) {
587    // Replace calls to String.<init> with equivalent StringFactory call.
588    if (declaring_class->IsStringClass() && m->IsConstructor()) {
589      jmethodID mid = soa.EncodeMethod(m);
590      m = soa.DecodeMethod(WellKnownClasses::StringInitToStringFactoryMethodID(mid));
591      CHECK(javaReceiver == nullptr);
592    } else {
593      // Check that the receiver is non-null and an instance of the field's declaring class.
594      receiver = soa.Decode<mirror::Object*>(javaReceiver);
595      if (!VerifyObjectIsClass(receiver, declaring_class)) {
596        return nullptr;
597      }
598
599      // Find the actual implementation of the virtual method.
600      m = receiver->GetClass()->FindVirtualMethodForVirtualOrInterface(m, sizeof(void*));
601    }
602  }
{% endhighlight %}

Finally, the looked up method is invoked at line 640:

{% highlight C %}
630  // Invoke the method.
631  JValue result;
632  uint32_t shorty_len = 0;
633  const char* shorty = np_method->GetShorty(&shorty_len);
634  ArgArray arg_array(shorty, shorty_len);
635  if (!arg_array.BuildArgArrayFromObjectArray(receiver, objects, np_method)) {
636    CHECK(soa.Self()->IsExceptionPending());
637    return nullptr;
638  }
639
640  InvokeWithArgArray(soa, m, &arg_array, &result, shorty);
{% endhighlight %}

Once we know how the runtime is lookuping virtual methods, we can exploit this mechanism to achieve Java virtual methods hooking.

Next post introduces ARTHook, an easy-to-use library for hooking virtual methods calls under ART runtime.
