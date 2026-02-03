# Using tinylog with GraalVM Native Image (Changelog Notes)

> Environment:
> GraalVM 25 (Native Image installed)
> 
> Gradle Wrapper 9.3.1
> 
> macOS (Apple Silicon arm64)

---


## Background: JVM vs Native Image 

On the JVM:
- Classes are loaded at runtime 
- Reflection, ServiceLoader, and resource loading work dynamically 
- Startup is relatively heavy (class loading, JIT warm-up)

With GraalVM Native Image:
- Everything is compiled ahead of time (AOT)
- The world must be “closed” at build time
- Startup is much faster and memory usage is lower

Trade-off:
Native images require extra configuration because dynamic JVM features must be declared explicitly.

The above was summarized by ChatGPT.

References:

- https://jabref.github.io/GSoC/posts/native-image/
- https://www.graalvm.org/latest/reference-manual/native-image/
- https://tutorials.jenkov.com/tutorials/graalvm/index.html#graalvm-website

---


## 1. The Big Picture: agent → config → native-image

The native image pipeline consists of three conceptual phases:

    JVM execution (with agent)
    ↓
    Reachability configuration (JSON metadata)
    ↓
    native-image compilation

Why is this needed?

When building a native image, GraalVM cannot “discover things at runtime” like the JVM does.
Everything has to be known ahead of time, during compilation.

This means GraalVM needs to be told explicitly about anything that is accessed dynamically,
for example:
classes used via reflection, resources loaded from the classpath, services discovered with
ServiceLoader, or other dynamic features such as proxies or JNI.

If this information is not provided, GraalVM will assume those parts are never used and
remove them during compilation. As a result, the native executable may fail at runtime,
even though the same application works correctly on the JVM.
---

## 2. Step-by-step Changelog

Upstream project:

- https://github.com/tinylog-org/tinylog-graal-example

This repository documents the changes required to build a working native image using a modern toolchain (GraalVM 25 + Gradle 9).

### 2.1 Run on the JVM (Baseline)

Before anything native-related, the application must work on the JVM:
```
./gradlew run
```

### 2.2 Local Toolchain Setup

The following tools must be available before running a task:

- GraalVM 25 (JDK)
- native-image installed
- Gradle Wrapper 9.3.1

Follow [the official GraalVM installation guide](https://www.graalvm.org/latest/getting-started/) to install GraalVM.

If the wrapper needs to be regenerated:
`gradle wrapper --gradle-version 9.3.1 --distribution-type bin`

Verify the local toolchain points to GraalVM 25:

```
which java
java -version

which native-image
native-image --version
```

Expected (example):

- ~/.sdkman/candidates/java/25-graal/...


If not:

```
source ~/.sdkman/bin/sdkman-init.sh
sdk use java 25-graal
```


References:

- https://www.graalvm.org/latest/getting-started/
- https://docs.gradle.org/current/userguide/gradle_wrapper.html
- https://docs.gradle.org/current/userguide/compatibility.html


### 2.3 Generate Native Image Configuration (Agent Phase)

This project defines a custom Gradle task:

```
./gradlew generateConfiguration
```

Internally, it runs:
```
java \
-agentlib:native-image-agent=config-output-dir=build/native-image-agent/META-INF/native-image \
-cp <runtime classpath> \
demo.Application
```
The native-image agent observes the running JVM application and records:
- Reflection usage 
- Resource loading 
- Service loader usage

These are written as JSON files like `reachability-metadata.json` (Use `ls build/native-image-agent/META-INF/native-image` to inspect.)

This directory contains reachability metadata used by native-image.

### 2.4 Build the Native Image

The second custom task is:

`./gradlew nativeImage`

This is a Gradle command, not a GraalVM command. Inside that task, Gradle invokes GraalVM’s native-image tool

This project uses classpath-based compilation, not -jar:
```
native-image \
-cp <runtime classpath> \
demo.Application \
-H:ConfigurationFileDirectories=build/native-image-agent/META-INF/native-image \
-o build/graal/demo
```

Reason:

The default Gradle JAR does not include runtime dependencies, which makes `java -jar`
unsuitable for this project and leads to `NoClassDefFoundError`. Using a classpath-based
invocation (`-cp`) ensures that both the JVM run and the native-image build operate on
the same set of dependencies.

---

### 2.5 Run the executable



The native executable is generated at:

```
build/graal/demo
```

Run and measure:

```
time ./build/graal/demo
```

JVM (baseline)

`./gradlew run`

---

### Next steps

At the moment, native image generation is handled via explicit custom Gradle tasks.

This decision was made to avoid compatibility issues observed with the
`com.palantir.graal` plugin.

A potential follow-up step would be to evaluate the [GraalVM Native Build Tools](https://github.com/graalvm/graalvm-demos/tree/master/native-image/native-build-tools/gradle-plugin)
plugin.
