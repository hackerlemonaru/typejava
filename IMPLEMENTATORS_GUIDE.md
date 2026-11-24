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
`System.out`. Build a custom environment when you need different output, records, export contracts,
or native types.

### Custom environment and output

`Io` receives already formatted console lines. It has a default `println` method, so implementations
are usually anonymous classes:

```java
List<String> lines = new ArrayList<>();

Environment environment =
    TypeJava.environment()
        .io(
            new Io() {
              @Override
              public void println(String line) {
                lines.add(line);
              }
            })
        .withStd()
        .build();

TypeJava.run("console.log(\"hello\", 42);", environment);
```

`withStd()` is shorthand for `withArray().withMap()`. Without it, `TjArray` and `TjMap` are unknown
types. `withArray()` and `withMap()` can be used separately.

### Compile once, instantiate many times
