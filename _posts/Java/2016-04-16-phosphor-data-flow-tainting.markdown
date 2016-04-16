---
layout: post
title: PhosPhor - Data Flow Tainting
author: vaioco
categories: Java
excerpt:
tags: [java,tainting]
image:
  feature:
date: 2016-04-16T15:53:40+02:00
---

# ENVIRONMENT

{% highlight shell linenos %}
$ echo $JAVA_HOME
/usr/lib/jvm/java-8-oracle
$ java -version
java version "1.8.0_77"
Java(TM) SE Runtime Environment (build 1.8.0_77-b03)
Java HotSpot(TM) 64-Bit Server VM (build 25.77-b03, mixed mode)
{% endhighlight %}

{% highlight shell linenos %}
$ cat taint-sinks
java/io/PrintStream.println(Ljava/lang/String;)V
java/io/PrintStream.println(Ljava/lang/Object;)V
{% endhighlight %}

{% highlight shell linenos %}
$ cat taint-sources
java/lang/Integer.toString(I)Ljava/lang/String;
java/io/ObjectInputStream.readObject()Ljava/io/Object;
{% endhighlight %}


# INSTRUMENTING JRE

1. using int tag:

{% highlight Java linenos %}
java -jar Phosphor-0.0.2-SNAPSHOT.jar -forceUnboxAcmpEq -withEnumsByValue -taintSources taint-sources -taintSinks taint-sinks /usr/lib/jvm/java-8-oracle/jre/ jre-inst
{% endhighlight %}

2. using obj tag (multiTaint):

{% highlight Java linenos %}
java -jar Phosphor-0.0.2-SNAPSHOT.jar -forceUnboxAcmpEq -withEnumsByValue -taintSources taint-sources -taintSinks taint-sinks -multiTaint /usr/lib/jvm/java-8-oracle/jre/ jre-inst-obj
{% endhighlight %}

3. using obj tag + implicit flow:

{% highlight shell linenos %}
java -jar Phosphor-0.0.2-SNAPSHOT.jar -controlTrack -forceUnboxAcmpEq -withEnumsByValue -taintSources taint-sources -taintSinks taint-sinks -multiTaint /usr/lib/jvm/java-8-oracle/jre/ jre-inst-implicit
{% endhighlight %}


# INT TAG mode

First, I tested the instrumented int-tag JRE (jre-inst) on both phosphortests.jar, test-app.jar  and jenkins.

phosphortests:

{% highlight Java linenos %}
java -jar Phosphor-0.0.2-SNAPSHOT.jar -taintSources taint-sources -taintSinks taint-sinks phosphortests.jar phosphortests-inst-int

$ jre-inst/bin/java -Xbootclasspath/p:Phosphor-0.0.2-SNAPSHOT.jar -javaagent:Phosphor-0.0.2-SNAPSHOT.jar -cp phosphortests-inst-int/phosphortests.jar -ea phosphor.test.DroidBenchTest
testFieldSensitivity1
testFieldSensitivity2
testFieldSensitivity3
[...]
{% endhighlight %}

test-app:

{% highlight Java linenos %}
java -jar Phosphor-0.0.2-SNAPSHOT.jar -taintSources taint-sources -taintSinks taint-sinks test-app.jar testapp-inst-int

$ jre-inst/bin/java -Xbootclasspath/p:Phosphor-0.0.2-SNAPSHOT.jar -javaagent:Phosphor-0.0.2-SNAPSHOT.jar -cp testapp-inst-int/test-app.jar -ea com.test.Main
haha
plugin started
Phosphor plugin: Tainted Object ==> 123
12312
phosphor plugin: # of tainted objects 1
phosphor plugin: # of tainted objects 1
phosphor plugin: # of tainted objects 1
[...] 
{% endhighlight %}

(I see only 1 tainted object)

I don't know if it is a feature, but actually the plugin's code starts an endless loop.

I also tested against jenkins:

{% highlight Java linenos %}
$ java -jar Phosphor-0.0.2-SNAPSHOT.jar jenkins.war -taintSources taint-jenkins-sources jenkins
Data flow tracking: enabled
Control flow tracking: disabled
Taints will be combined with logical-or.
Starting analysis
Analysis Completed: Beginning Instrumentation Phase
Using taint sources file: taint-sources
Using taint sinks file: taint-sinks
Loaded 0 sinks and 1 sources
[...]
{% endhighlight %}

During the instrumentation step, i got several exceptions like the following one:

{% highlight Java linenos %}
java.util.zip.ZipException: duplicate entry: javax/jmdns/impl/tasks/state/Renewer.class
	at java.util.zip.ZipOutputStream.putNextEntry(ZipOutputStream.java:232)
	at java.util.jar.JarOutputStream.putNextEntry(JarOutputStream.java:109)
	at edu.columbia.cs.psl.phosphor.Instrumenter.processJar(Instrumenter.java:631)
	at edu.columbia.cs.psl.phosphor.Instrumenter.processZip(Instrumenter.java:814)
	at edu.columbia.cs.psl.phosphor.Instrumenter._main(Instrumenter.java:512)
	at edu.columbia.cs.psl.phosphor.Instrumenter.main(Instrumenter.java:380)
{% endhighlight %}

Besides the above expections, the instrumented version of jenkins is successfully created. Then i ran jenkins, and the following box shows the plugin messagges:

{% highlight Java linenos %}
sid@artdroid:~/PSU/deserialization/phosphor-mod/Phosphor/target$ ./jre-inst/bin/java -Xbootclasspath/a:Phosphor-0.0.2-SNAPSHOT.jar  -javaagent:Phosphor-0.0.2-SNAPSHOT.jar -jar jenkins/jenkins.war
Running from: /home/sid/PSU/deserialization/phosphor-mod/Phosphor/target/jenkins/jenkins.war
webroot: $user.home/.jenkins
plugin started
Phosphor plugin: Tainted Object ==> 800
phosphor plugin: # of tainted objects 1
[...]
{% endhighlight %}

Then i ran the jenkins.py from JavaUnserializeExploits and the file in /tmp is successfully created. The following
is the log printed by jenkins:

{% highlight Java linenos %}
INFO: Accepted connection #2 from /127.0.0.1:33640
Phosphor plugin: Tainted Object ==> 0
Phosphor plugin: Tainted Object ==> 0
Phosphor plugin: Tainted Object ==> -1
Phosphor plugin: Tainted Object ==> 0
Exception in thread "TCP slave agent connection handler #2 with /127.0.0.1:33640" java.lang.ClassCastException: java.lang.UNIXProcess cannot be cast to java.util.Set
	at com.sun.proxy.$Proxy37.entrySet(Unknown Source)
	at sun.reflect.annotation.AnnotationInvocationHandler.readObject(AnnotationInvocationHandler.java:452)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at java.io.ObjectStreamClass.invokeReadObject(ObjectStreamClass.java:1058)
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:1900)
	at java.io.ObjectInputStream.readOrdinaryObject$$PHOSPHORTAGGED(ObjectInputStream.java:1801)
	at java.io.ObjectInputStream.readObject0$$PHOSPHORTAGGED(ObjectInputStream.java:1351)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:371)
	at hudson.remoting.Capability.read(Capability.java:125)
	at hudson.remoting.ChannelBuilder.negotiate(ChannelBuilder.java:355)
	at hudson.remoting.ChannelBuilder.build(ChannelBuilder.java:282)
	at hudson.cli.CliProtocol$Handler.runCli(CliProtocol.java:77)
	at hudson.cli.CliProtocol$Handler.run(CliProtocol.java:64)
	at hudson.cli.CliProtocol.handle(CliProtocol.java:40)
	at hudson.TcpSlaveAgentListener$ConnectionHandler.run(TcpSlaveAgentListener.java:156)
phosphor plugin: # of tainted objects 50
[...]
{% endhighlight %}


# USING MULTITAINT

I'm going to show the results of intrumented JRE with multiTaint enabled. 
First, I created an instrumented JRE with multiTaint enable using the following command:

{% highlight Java linenos %}
java -jar Phosphor-0.0.2-SNAPSHOT.jar -taintSources taint-sources -taintSinks taint-sinks -multiTaint phosphortests.jar phosphortests-inst-multi
{% endhighlight %}

Then, i ran the phosphortests in multiTaint mode:

{% highlight Java linenos %}
$ jre-inst-obj/bin/java -Xbootclasspath/p:Phosphor-0.0.2-SNAPSHOT.jar -javaagent:Phosphor-0.0.2-SNAPSHOT.jar -cp phosphortests-inst-multi/phosphortests.jar -ea phosphor.test.DroidBenchTest
testFieldSensitivity1
java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at phosphor.test.DroidBenchTest.main(DroidBenchTest.java:521)
Caused by: java.lang.NoSuchMethodError: edu.columbia.cs.psl.phosphor.runtime.Tainter.taintedCharArray$$PHOSPHORTAGGED([Ledu/columbia/cs/psl/phosphor/runtime/Taint;[CLedu/columbia/cs/psl/phosphor/runtime/Taint;ILedu/columbia/cs/psl/phosphor/struct/TaintedCharArrayWithObjTag;)Ledu/columbia/cs/psl/phosphor/struct/TaintedCharArrayWithObjTag;
[...]
{% endhighlight %}

.. but the infamous java.lang.NoSuchMethodError arise again ...

Same story with test-app, but with different exception:

{% highlight Java linenos %}
java -jar Phosphor-0.0.2-SNAPSHOT.jar -multiTaint -taintSources taint-sources -taintSinks taint-sinks test-app.jar testapp-inst-multi
{% endhighlight %}

{% highlight Java linenos %}
$ jre-inst-obj/bin/java -Xbootclasspath/p:Phosphor-0.0.2-SNAPSHOT.jar -javaagent:Phosphor-0.0.2-SNAPSHOT.jar -cp testapp-inst-multi/test-app.jar -ea com.test.Main
haha
Exception in thread "main" java.lang.IllegalAccessError: Argument carries taints:  Taint [lbl=java/lang/Integer.toString(I)Ljava/lang/String;  deps = []]
	at edu.columbia.cs.psl.phosphor.runtime.TaintChecker.checkTaint(TaintChecker.java:76)
	at edu.columbia.cs.psl.phosphor.runtime.TaintChecker.checkTaint(TaintChecker.java:64)
	at java.io.PrintStream.write$$PHOSPHORTAGGED(PrintStream.java)
	at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:221)
	at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:291)
	at sun.nio.cs.StreamEncoder.flushBuffer(StreamEncoder.java:104)
	at java.io.OutputStreamWriter.flushBuffer(OutputStreamWriter.java:185)
	at java.io.PrintStream.write(PrintStream.java:527)
	at java.io.PrintStream.print(PrintStream.java:669)
	at java.io.PrintStream.println(PrintStream.java:806)
	at com.test.Main.main(Main.java:7)
{% endhighlight %}

Even the Jenkins' turn is not happy for me:

{% highlight Java linenos %}
$ ./jre-inst-obj/bin/java -Xbootclasspath/a:Phosphor-0.0.2-SNAPSHOT.jar  -javaagent:Phosphor-0.0.2-SNAPSHOT.jar -jar jenkins-multi/jenkins.war
Running from: /home/sid/PSU/deserialization/phosphor-mod/Phosphor/target/jenkins-multi/jenkins.war
webroot: $user.home/.jenkins
Exception in thread "main" java.lang.IllegalAccessError: Argument carries taints:  Taint [lbl=java/io/InputStream.read([BII)I  deps = []]
	at edu.columbia.cs.psl.phosphor.runtime.TaintChecker.checkTaint(TaintChecker.java:76)
	at edu.columbia.cs.psl.phosphor.runtime.TaintChecker.checkTaint(TaintChecker.java:64)
	at java.io.FileOutputStream.write$$PHOSPHORTAGGED(FileOutputStream.java)
	at Main.copyStream(Main.java:379)
	at Main.extractFromJar(Main.java:404)
	at Main._main(Main.java:190)
	at Main.main(Main.java:98)
{% endhighlight %}


# TODO


- Actually, all tests against jre-inst-obj are failing

- while instrumenting JRE with "java/io/InputStream.read()I" as taint-source, I see the following messages:

{% highlight Java linenos %}
Source: com/sun/javafx/runtime/async/AbstractRemoteResource$ProgressInputStream.read$$PHOSPHORTAGGED(Ledu/columbia/cs/psl/phosphor/struct/TaintedIntWithIntTag;)Ledu/columbia/cs/psl/phosphor/struct/TaintedIntWithIntTag; Label: 1
Processed: 5000/31516
Processed: 6000/31516
Source: com/sun/webkit/SimpleSharedBufferInputStream.read$$PHOSPHORTAGGED(Ledu/columbia/cs/psl/phosphor/struct/TaintedIntWithIntTag;)Ledu/columbia/cs/psl/phosphor/struct/TaintedIntWithIntTag; Label: 1
{% endhighlight %}

I get no messages while instrumenting the JRE with "java/io/ObjectInputStream.readObject()Ljava/io/Object;" as single taint-source. It looks strange.

