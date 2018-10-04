---
layout: post
title:  "GraalVM Native Image HTTPS Support"
date:   2018-10-04 12:00:00
tags:   graalvm clojure native
---

GraalVM 1.0.0-RC7 adds HTTPS as a supported protocol, and this is a brief walkthrough
for using it in a Clojure project with GraalVM Community Edition for macOS.
See [this example project](https://github.com/taylorwood/clojurl) for a demo.

## How To

1. Enable HTTPS protocol support with `native-image` options:
   `--enable-https` or `--enable-url-protocols=https`
1. Configure path to `libsunec` _(Sun Elliptic Curve crypto)_

   This shared object comes with the GraalVM distribution and can be found at
   `$GRAALVM_HOME/jre/lib/libsunec.[so|dylib]`. GraalVM uses
   `System.loadLibrary` to load it at run-time whenever it's first used. The file must
   either be in the current working directory, or in a path specified in Java system
   property `java.library.path`.

   I set the Java system property inside my application at run-time before first usage:
   ```clojure
   (System/setProperty "java.library.path"
                       (str (System/getenv "GRAALVM_HOME") "/jre/lib"))
   ```

   See [this](https://github.com/oracle/graal/blob/master/substratevm/JCA-SECURITY-SERVICES.md#native-implementations)
   and [this](https://github.com/oracle/graal/blob/master/substratevm/URL-PROTOCOLS.md#https-support)
   for more information on HTTPS support in GraalVM and native images. If you're distributing
   a native image, you'll need to include libsunec. If it's in the same directory as your image
   you don't need to set `java.library.path`.

   You'll see a [warning](https://github.com/oracle/graal/blob/e3ef4f3f741d171a83c2dd2a0390dbede6b2c62d/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/jdk/SecuritySubstitutions.java#L204)
   at run-time if this hasn't been properly configured:
   ```
   WARNING: The sunec native library could not be loaded.
   ```

1. Use more complete certificate store

   GraalVM comes with a smaller set of CA certificates. For [_reasons_](https://github.com/oracle/graal/issues/378#issuecomment-384245987)
   they cannot yet distribute the Oracle JDK root certificates. You can workaround this
   by replacing GraalVM's `cacerts`. I renamed the file and replaced it with a symbolic link
   to `cacerts` from the JRE that comes with macOS Mojave:
   ```bash
   $ mv $GRAALVM_HOME/jre/lib/security/cacerts $GRAALVM_HOME/jre/lib/security/cacerts.bak
   $ ln -s $(/usr/libexec/java_home)/jre/lib/security/cacerts $GRAALVM_HOME/jre/lib/security/cacerts
   ```

   If you don't do this, you might see such horrors as this when attempting HTTPS connections:
   ```
   Exception in thread "main" javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
   8<------------------------
   Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
   8<------------------------
   Caused by: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
   ```
