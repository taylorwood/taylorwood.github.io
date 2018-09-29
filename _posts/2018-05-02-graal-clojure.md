---
layout: post
title:  "Building native Clojure images with GraalVM"
date:   2018-05-02 12:00:00
tags:   graalvm clojure native
---

[GraalVM](http://www.graalvm.org) makes it possible to compile JVM-based programs ahead-of-time into a native executable,
with significant improvements to startup time, resource usage, etc.
This post will demonstrate a CLI tool written in Clojure and compiled to a native binary, and
compare its performance to the JVM-based version.

_**Update 2018-05-25:** I created a simple Leiningen plugin [`lein-native-image`](https://github.com/taylorwood/lein-native-image)
to simplify `native-image` usage with Leiningen projects;
its README has more up-to-date info and additional [example projects](https://github.com/taylorwood/lein-native-image/tree/master/examples).
I created a similar tool [`clj.native-image`](https://github.com/taylorwood/clj.native-image)
for use with deps.edn projects.
This article was written beforehand and assumes neither is used._

This approach is similar to [Jan Stępień's article](https://www.innoq.com/en/blog/native-clojure-and-graalvm/)
but runs in macOS rather than within a Linux container.

I'll create a simple command line utility program that converts JSON to EDN,
starting with a minimal Leiningen `project.clj`:
```clojure
(defproject jadensmith "0.1.0-SNAPSHOT"
  :dependencies [[org.clojure/clojure "1.9.0"]
                 [org.clojure/data.json "0.2.6"]]
  :main ^:skip-aot jadensmith.core
  :profiles {:uberjar {:aot :all}})
```

And the implementation:
```clojure
(ns jadensmith.core
  (:require [clojure.data.json :as json]
            [clojure.edn :as edn])
  (:gen-class))

(defn -main [& args]
  (prn (json/read *in* :key-fn keyword)))
```
It reads JSON from stdin and prints it to stdout in EDN format.

[This article](https://www.innoq.com/en/blog/native-clojure-and-graalvm/)
shows how to use GraalVM's `native-image` on particular class files,
but requires manually copying/unpacking some dependency JARs into the `target` directory beforehand.
We can avoid that since `native-image` can also use JAR files, so let's build an uberjar:
```bash
$ lein uberjar
```

_Note: Building an uberjar is unnecessary when using [`lein native-image`](https://github.com/taylorwood/lein-native-image)._

## Obtaining GraalVM

Now you'll need to obtain [GraalVM](http://www.graalvm.org/downloads/).
I downloaded the Community Edition for macOS and placed it in some odd directory on my hard drive,
then set an environment variable similar to `$JAVA_HOME`:
```bash
export GRAALVM_HOME=/path/to/graalvm-ce-1.0.0-rc6/Contents/Home
```
On macOS, the contents of GraalVM's `Home` directory should look similar to a JDK installation e.g. `Home/bin/java`.

## Building a native image

We can now use `native-image` on the standalone uberjar:
```bash
$ cd target
$ $GRAALVM_HOME/bin/native-image -jar jadensmith-0.1.0-SNAPSHOT-standalone.jar
Build on Server(pid: 13184, port: 26681)*
   classlist:   2,240.42 ms
       (cap):   1,690.30 ms
       setup:   2,808.39 ms
  (typeflow):   4,677.28 ms
   (objects):   3,363.83 ms
  (features):      44.11 ms
    analysis:   8,229.62 ms
    universe:     326.01 ms
     (parse):   1,154.86 ms
    (inline):   1,200.41 ms
   (compile):   7,656.20 ms
     compile:  10,606.37 ms
       image:   1,926.50 ms
       write:   1,357.57 ms
     [total]:  27,586.92 ms
```

The resulting image:
```bash
$ la jadensmith-0.1.0-SNAPSHOT-standalone
-rwxr-xr-x  1 taylorwood  44583454   6.8M May  2 06:24 jadensmith-0.1.0-SNAPSHOT-standalone*
```
It's slightly larger than the 4.2M uberjar, but the native image has no dependency on JRE/JDK
and has some other interesting properties:

> The resulting program does not run on the Java HotSpot VM, but uses necessary components like memory management, thread scheduling from a different implementation of a virtual machine, called Substrate VM. Substrate VM is written in Java and compiled into the native executable. The resulting program has faster startup time and lower runtime memory overhead compared to a Java VM.

Let's `time` the JAR version by feeding it JSON:
```bash
$ /usr/bin/time -l sh -c 'cat test.json | java -jar jadensmith-0.1.0-SNAPSHOT-standalone.jar'
...
        1.02 real         1.79 user         0.11 sys
 117514240  maximum resident set size
8<-----------------------------------
     32345  page reclaims
8<-----------------------------------
         6  signals received
         4  voluntary context switches
      2860  involuntary context switches
```

Now let's `time` the native image for comparison:
```bash
$ /usr/bin/time -l sh -c 'cat test.json | ./jadensmith-0.1.0-SNAPSHOT-standalone'
...
        0.02 real         0.01 user         0.00 sys
  10436608  maximum resident set size
8<-----------------------------------
      3449  page reclaims
8<-----------------------------------
         4  voluntary context switches
        48  involuntary context switches
```

We can see considerable differences in startup time, CPU usage, and memory usage.
The JVM version uses ~11x more memory and takes ~10x longer.
This seems promising for writing native CLI tools in Clojure.

## Limitations

There are some [limitations](http://www.graalvm.org/docs/reference-manual/aot-compilation/#limitations-of-aot-compilation)
to GraalVM's native image building:

> To achieve such aggressive ahead-of-time optimizations, we run an aggressive static analysis that requires a closed-world assumption. We need to know all classes and all bytecodes that are reachable at run time. Therefore, it is not possible to load new classes that have not been available during ahead-of-time-compilation.

There are other limitations on reflection, documented [here](https://github.com/oracle/graal/blob/master/substratevm/LIMITATIONS.md).
Some of these issues can be worked around with hinting/configuration;
see [these projects](https://github.com/taylorwood/lein-native-image/tree/master/examples) for examples of such workarounds.

GraalVM's `native-image` has a flag to treat these issues as exceptions at run-time, instead of its default behavior of failing to build. Hopefully some of these limitations will be lifted over time, and GraalVM can support more non-trivial Clojure apps out-of-the-box.

<blockquote class="twitter-tweet" data-cards="hidden" data-lang="en"><p lang="en" dir="ltr">Excellent article by <a href="https://twitter.com/janstepien?ref_src=twsrc%5Etfw">@janstepien</a> &quot;Native Clojure with GraalVM&quot; creating a lightweight web server. We will incrementally lift restrictions for native image generation in upcoming releases. <a href="https://t.co/tsx0OBtssG">https://t.co/tsx0OBtssG</a> <a href="https://t.co/Td387PHRoa">pic.twitter.com/Td387PHRoa</a></p>&mdash; GraalVM (@graalvm) <a href="https://twitter.com/graalvm/status/990139129617928193?ref_src=twsrc%5Etfw">April 28, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 
