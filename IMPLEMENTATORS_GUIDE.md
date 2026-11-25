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

| API | Purpose |
|---|---|
| `TypeJava.run(source, environment)` | Compile and run one instance. |
| `TypeJava.compile(source, environment)` | Parse, check, and generate bytecode once. |
| `TypeJava.check(source, environment)` | Validate without running; returns diagnostics. |
| `TypeJava.checkFile(path, environment)` | Read and validate a `.tj` file; returns diagnostics. |
| `TypeJava.runFile(path, environment)` | Read, compile, and run a `.tj` file. |
| `CompiledProgram.run()` | Instantiate and run, discarding the instance. |
| `CompiledProgram.instantiate()` | Create an isolated `ProgramInstance`. |
| `CompiledProgram.exports()` | Inspect exported function signatures. |
| `ProgramInstance.invoke(name, args...)` | Call an export without a typed interface. |
| `ProgramInstance.exports(Api.class)` | Get a typed proxy over exports. |
| `Environment.typeScriptDeclarations()` | Generate ambient `.d.ts` contents. |

## Exported functions

Exported functions are top-level functions marked with `export`. They are the supported bridge from
Java into a program instance.

### Java records as TypeJava types

Bind a Java record with `record(Class<?>)`:

```java
public record Payload(double incrementMe) {}

public interface LogApi {
  Payload analyzeLog(Payload payload);

  double invocationCount();
}
```

```java
Environment environment =
    TypeJava.environment()
        .record(Payload.class)
        .exports(LogApi.class)
        .build();
```

The record's simple name becomes the TypeJava type name. TypeJava source can use `Payload` directly,
without declaring it:

```ts
let invocations: number = 0;

export function analyzeLog(payload: Payload): Payload {
  invocations = invocations + 1;
  return { incrementMe: payload.incrementMe + 1 };
};

export function invocationCount(): number {
  return invocations;
};
```

The source may also redeclare the record as an identical readonly object type. A different
redeclaration is a compile error; the bound Java record remains authoritative.

### Call exports

```java
CompiledProgram program = TypeJava.compile(source, environment);
ProgramInstance instance = program.instantiate();

LogApi api = instance.exports(LogApi.class);
Payload result = api.analyzeLog(new Payload(0));

Object dynamic = instance.invoke("analyzeLog", new Payload(1));
```

`exports(LogApi.class)` returns a JDK proxy. Every abstract method on the interface must have a
matching exported function with a compatible name, arity, and host signature. The proxy call is
not cached; each method call goes through the same invocation path as `invoke`.

`program.exports()` returns a map of exported names to `ExportSignature` values. This is useful for
diagnostics or building dynamic adapters.

### Export contracts

`exports(Class<?>...)` records expected interfaces. During compilation, every abstract method in the
interface must have a matching `export function`; otherwise compilation fails. Extra exported
functions are allowed.

Exported functions must use types that can cross the host boundary:

- `number`, `string`, `boolean`, and `void`.
- Nullable variants using the Java wrapper for primitives and `String` for strings.
- Bound record classes.
- Bound native types such as `TjArray` and `TjMap`.
