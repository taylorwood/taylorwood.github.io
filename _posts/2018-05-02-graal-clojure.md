---
layout: post
title:  "Building native Clojure images with GraalVM"
date:   2018-05-02 12:00:00
tags:   graalvm clojure native
---

With the recent release of GraalVM it's now possible to compile programs ahead-of-time into a native executable for JVM-based languages. Along with this comes pretty radical implications with regard to startup time, artifact size, runtime performance, etc. Read [this article](https://www.innoq.com/en/blog/native-clojure-and-graalvm/) first for more information. This post demonstrates the same idea, but directly on macOS rather than within a Linux container.

Similar to the linked article above, I'll create a simple command line utility program that converts JSON to EDN, starting with a minimal Leiningen `project.clj`:
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
It just reads JSON from stdin and `prn`'s it to stdout.

The linked article covers how to use GraalVM's `native-image` on particular class files, but requires manually copying/unpacking some dependency JARs into the `target` directory beforehand. Fortunately we can avoid that since `native-image` can also use JAR files, so let's just build an uberjar:
```bash
$ lein uberjar
```

## Obtaining GraalVM

Now you'll need to obtain [GraalVM EE for macOS](http://www.graalvm.org/downloads/), which is free for evaluation purposes. The CE is not yet available for macOS for technical reasons according to Oracle, but should be available eventually. I downloaded the EE and placed it in some odd directory on my hard drive, then set an environment variable similar to `$JAVA_HOME`:
```bash
export GRAAL_HOME=/path/to/graalvm-1.0.0-rc1/Contents/Home
```
On macOS, the contents of GraalVM's `Home` directory should look similar to a JDK installation e.g. `Home/bin/java`.

## Building a native image

We can now use `native-image` on the standalone uberjar:
```bash
$ cd target
$ $GRAAL_HOME/bin/native-image -jar jadensmith-0.1.0-SNAPSHOT-standalone.jar
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
Only slightly larger than the 4.2M uberjar, but the native image requires no dependencies and has some other interesting properties.

> The resulting program does not run on the Java HotSpot VM, but uses necessary components like memory management, thread scheduling from a different implementation of a virtual machine, called Substrate VM. Substrate VM is written in Java and compiled into the native executable. The resulting program has faster startup time and lower runtime memory overhead compared to a Java VM.

Let's try out the JAR version by feeding it a big JSON input:
```bash
$ time cat /path/to/big.json | java -jar jadensmith-0.1.0-SNAPSHOT-standalone.jar
...
java -jar jadensmith-0.1.0-SNAPSHOT-standalone.jar  2.16s user 0.15s system 198% cpu 1.164 total
```

Versus the native image:
```bash
$ time cat /path/to/big.json | ./jadensmith-0.1.0-SNAPSHOT-standalone
...
/jadensmith-0.1.0-SNAPSHOT-standalone  0.02s user 0.01s system 67% cpu 0.040 total
```

Those are dramatic differences in startup time and CPU usage. Already we can see immediate potential for writing command line utilities in Clojure.

## Limitations

There are currently some [limitations](http://www.graalvm.org/docs/reference-manual/aot-compilation/#limitations-of-aot-compilation) to GraalVM's native image building:

> To achieve such aggressive ahead-of-time optimizations, we run an aggressive static analysis that requires a closed-world assumption. We need to know all classes and all bytecodes that are reachable at run time. Therefore, it is not possible to load new classes that have not been available during ahead-of-time-compilation.

There are other limitations on reflection, documented [here](https://github.com/oracle/graal/blob/master/substratevm/LIMITATIONS.md).

GraalVM's `native-image` has a flag to treat these issues as exceptions at run-time, instead of its default behavior of failing to build. Hopefully some of these limitations will be lifted over time, and GraalVM can support more non-trivial Clojure apps out-of-the-box.

<blockquote class="twitter-tweet" data-cards="hidden" data-lang="en"><p lang="en" dir="ltr">Excellent article by <a href="https://twitter.com/janstepien?ref_src=twsrc%5Etfw">@janstepien</a> &quot;Native Clojure with GraalVM&quot; creating a lightweight web server. We will incrementally lift restrictions for native image generation in upcoming releases. <a href="https://t.co/tsx0OBtssG">https://t.co/tsx0OBtssG</a> <a href="https://t.co/Td387PHRoa">pic.twitter.com/Td387PHRoa</a></p>&mdash; GraalVM (@graalvm) <a href="https://twitter.com/graalvm/status/990139129617928193?ref_src=twsrc%5Etfw">April 28, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 
