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

```java
CompiledProgram program = TypeJava.compile(source, environment);

program.run();  // creates and runs one instance
program.run();  // creates a separate instance with fresh top-level state
```

`CompiledProgram` is immutable and can be shared between threads. `run()` is a convenience that
instantiates a program and discards the instance.

Use `instantiate()` when you need to call into the program:

```java
ProgramInstance first = program.instantiate();
ProgramInstance second = program.instantiate();
```

Each instance has its own copy of top-level variables. A `ProgramInstance` is not thread-safe;
create one instance per thread, or synchronize access. Instances are reclamation-managed; there is
no explicit close operation.

`runFile(Path, Environment)` is a convenience for reading a source file and running it. It throws
`IOException` if the file cannot be read.

### Check without running

Use `check` to validate source without creating or running a program instance. This is useful for
editor diagnostics and save-time validation:

```java
List<Diagnostic> diagnostics = TypeJava.check(source, environment);
for (Diagnostic diagnostic : diagnostics) {
  System.out.println(
      diagnostic.line() + ":" + diagnostic.column() + " " + diagnostic.message());
}
```

An empty list means the program passed the full compile pipeline and would compile. The current
compiler is fail-fast, so the list contains at most one diagnostic, but callers should treat it as a
list because later versions may report more. A successful check does not run top-level statements,
and the program may still fail at runtime.

`checkFile(Path, Environment)` reads and checks a `.tj` file. It also returns diagnostics and throws
`IOException` if the file cannot be read.

### Public API at a glance
