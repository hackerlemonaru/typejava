# AGENTS.md

## Project

TypeJava is a small TypeScript-flavoured language that compiles to JVM bytecode with ASM. The
pipeline is lexer -> parser -> checker -> ASM code generator -> generated classloader.

Keep `README.md` a short user-facing landing page. Put language semantics in `TJ_GUIDE.md`, Java
integration and tooling in `IMPLEMENTATORS_GUIDE.md`, and development, release, CI, and maintenance
details here.

## Documentation

- `README.md` — who TypeJava is for, the short pitch, install, and links to the guides.
- `TJ_GUIDE.md` — language syntax, type system, nulls, objects, collections, and TypeScript
  differences. Assumes some TypeScript familiarity.
- `IMPLEMENTATORS_GUIDE.md` — Java embedding, compile/instantiate lifecycle, exports, record
  bindings, native types, declaration generation, and boundary semantics.
- `benchmarks/README.md` — workload, methodology, current results, and benchmark commands.
- `AGENTS.md` — build, test, release, CI, architecture, and maintainer constraints.

Do not grow `README.md` into a language or integration manual; extend the appropriate guide instead.

## Modules

- `:lib` — compiler, runtime, and public `TypeJava` compile/run API. This is the only published
  module.
- `:cli` — command-line entry point, thin CLI jar, and self-contained fat jar. Depends on `:lib`.
- `:examples` — runnable public-API demos. Depends only on `:lib`.
- `:benchmarks` — JMH harness comparing TypeJava with plain Java, Groovy, BeanShell, and Rhino.
  Depends on `:lib` plus benchmark-only language runtimes.

Dependency direction is `:cli -> :lib`, `:examples -> :lib`, and `:benchmarks -> :lib`. `:lib` must
not depend on any of them. Do not publish `:cli`, `:examples`, or `:benchmarks` yet. Groovy,
BeanShell, and Rhino must stay `implementation` dependencies of `:benchmarks` and must never appear
in `:lib` or its published POM.

## Commands

Use the Nix dev shell:

```bash
nix develop --command gradle check
nix develop --command gradle build
```

`build` runs all module tests, produces the thin library jar, sources/javadoc jars, and the fat CLI
jar.

Run the CLI:

```bash
nix develop --command gradle :cli:run --args="path/to/script.tj"
```

Run all examples:

```bash
nix develop --command gradle :examples:run
```

Run one example:

