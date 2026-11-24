# Implementor Guide

This guide is for Java developers embedding TypeJava in an application. It covers the public compile
and run API, isolated program instances, exported functions, Java record bindings, custom native
types, TypeScript declaration generation, and the language semantics that matter at the Java
boundary.

The language itself is covered in the [TypeJava guide](TJ_GUIDE.md).

## Requirements and dependency

TypeJava requires Java 21. The published coordinates are:

```text
cc.kilonode:typejava:<version>
```

See [Install](README.md#install) in the README for Gradle, GitHub Packages authentication, and the
CLI jar. This guide starts with the Java API.

## Compile and run

The simplest entry point is:

```java
TypeJava.run("console.log(\"hello\");", Environment.standard());
```

`Environment.standard()` binds `TjArray` and `TjMap` and writes `console.log` output to
