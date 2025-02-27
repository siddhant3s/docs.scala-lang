---
title: Classpath Level
type: section
description: This section describes the compatibility between Scala 2.13 and Scala 3 class files.
num: 3
previous-page: compatibility-source
next-page: compatibility-runtime
---

In your code you can use public types and terms, and call public methods that are defined in a different module or library.
It works well as long as the type checker, which is the compiler phase that validates the semantic consistency of the code, is able to read the signatures of those types, terms and methods, from the class files containing them.

In Scala 2 the signatures are stored in a dedicated format called the Pickle format.
In Scala 3 the story is a bit different because it relies on the TASTy format which is a lot more than a signature layout.
But, for the purpose of moving from Scala 2.13 to Scala 3, only the signatures are useful.

## The Scala 3 Unpickler

The first piece of good news is that the Scala 3 compiler is able to read the Scala 2.13 Pickle format and thus it can type check code that depends on modules or libraries compiled with Scala 2.13.

The Scala 3 unpickler has been extensively tested in the community build for many years now. It is safe to use.

### A Scala 3 module can depend on a Scala 2.13 artifact

![Scala 3 module depending on a Scala 2.13 artifact](/resources/images/scala3-migration/compatibility-3-to-213.svg)

As an sbt build it can be illustrated by (sbt 1.5.0 or higher is required):

```scala
lazy val foo = project.in(file("foo"))
  .settings(scalaVersion := "3.0.0-RC3")
  .dependsOn(bar)

lazy val bar = project.in(file("bar"))
  .settings(scalaVersion := "2.13.5)
```

Or, in case bar is a published Scala 2.13 library, we can have:

```scala
lazy val foo = project.in(file("foo"))
  .settings(
    scalaVersion := "3.0.0-RC3",
    libraryDependencies += ("org.bar" %% "bar" % "1.0.0").cross(CrossVersion.for3Use2_13)
  )
```

We use `CrossVersion.for3Use2_13` in sbt to resolve `bar_2.13` instead of `bar_3`.

### The Standard Library

One notable example is the Scala 2.13 library.
We have indeed decided that the Scala 2.13 library is the official standard library for Scala 3.

Let's note that the standard library is automatically provided by the build tool, you should not need to configure it manually.

## The Scala 2.13 TASTy Reader

The second piece of good news is that the Scala 2.13 TASTy reader, which enables consuming Scala 3 libraries has been shipped into Scala 2.13.5.

> The TASTy reader is very new. That's why it is  only available under the `-Ytasty-reader` flag.

### Supported Features

The TASTy reader supports all the traditional language features as well as the following brand-new features:
- [Enumerations](http://dotty.epfl.ch/docs/reference/enums/enums.html)
- [Intersection Types](http://dotty.epfl.ch/docs/reference/new-types/intersection-types.html)
- [Opaque Type Aliases](http://dotty.epfl.ch/docs/reference/other-new-features/opaques.html)
- [Type Lambdas](http://dotty.epfl.ch/docs/reference/new-types/type-lambdas.html)
- [Contextual Abstractions](http://dotty.epfl.ch/docs/reference/contextual/motivation.html) (new syntax)
- [Open Classes](http://dotty.epfl.ch/docs/reference/other-new-features/open-classes.html) (and inheritance of super traits)
- [Export Clauses](http://dotty.epfl.ch/docs/reference/other-new-features/export.html)

We have limited support on:
- [Top-Level Definitions](http://dotty.epfl.ch/docs/reference/dropped-features/package-objects.html)
- [Extension Methods](http://dotty.epfl.ch/docs/reference/contextual/extension-methods.html)

More exotic features are not supported:
- [Context Functions](http://dotty.epfl.ch/docs/reference/contextual/context-functions.html)
- [Polymorphic Function Types](http://dotty.epfl.ch/docs/reference/new-types/polymorphic-function-types.html)
- [Trait Parameters](http://dotty.epfl.ch/docs/reference/other-new-features/trait-parameters.html)
- `@static` Annotation
- `@alpha` Annotation
- [Functions and Tuples larger than 22 parameters](https://dotty.epfl.ch/docs/reference/dropped-features/limit22.html)
- [Match Types](https://dotty.epfl.ch/docs/reference/new-types/match-types.html)
- [Union Types](https://dotty.epfl.ch/docs/reference/new-types/union-types.html)
- [Multiversal Equality](https://dotty.epfl.ch/docs/reference/contextual/multiversal-equality.html) (unless explicit)
- [Inline](https://dotty.epfl.ch/docs/reference/metaprogramming/inline.html) (including Scala 3 macros)
- [Kind Polymorphism](https://dotty.epfl.ch/docs/reference/other-new-features/kind-polymorphism.html) (the `scala.AnyKind` upper bound)

### A Scala 2.13 module can depend on a Scala 3 artifact

By enabling the TASTy reader with `-Ytasty-reader`, a Scala 2.13 module can depend on a Scala 3 artifact.

![Scala 2 module depending on a Scala 3 artifact](/resources/images/scala3-migration/compatibility-213-to-3.svg)

As an sbt build it can be illustrated by:

```scala
lazy val foo = project.in.file("foo")
  .settings(
    scalaVersion := "2.13.5",
    scalacOptions += "-Ytasty-reader"
  )
  .dependsOn(bar)

lazy val bar = project.in(file("bar"))
  .settings(scalaVersion := "3.0.0-RC3")
```

Or, in case `bar` is a published Scala 3 library:

```scala
lazy val foo = project.in.file("foo")
  .settings(
    scalaVersion := "3.0.0-RC3",
    scalacOptions += "-Ytasty-reader",
    libraryDependencies += ("org.bar" %% "bar" % "1.0.0").cross(CrossVersion.for2_13Use3)
  )
```

Similarly to `CrossVersion.for2_13Use3`, we use `CrossVersion.for3Use2_13` in sbt to resolve `bar_2.13` instead of `bar_3`.

## Interoperability Overview

In short, we have backward and forward compatibility and so **migration can happen gradually**.

You can port a big Scala application one module at a time, even if its library dependencies have not yet been ported (excepting the macro libraries).

During the transition period, you can have a Scala 3 module layered in between two 2.13 modules.

![Sandwich pattern](/resources/images/scala3-migration/compatibility-sandwich.svg)

This is permitted as long as all libraries are resolved to a single binary version: you can have `lib-foo_3` and `lib-bar_2.13` in the same classpath, but you cannot have `lib-foo_3` and `lib-foo_2.13`.

The inverted pattern, with a 2.13 module in the middle, is also possible.

> #### Disclaimer for library maintainers
> 
> Using the interoperability between Scala 2.13 and Scala 3 in a published library is generally not safe for your end-users.
> 
> Unless you know exactly what you are doing, it is discouraged to publish a Scala 3 library that depends on a Scala 2.13 library (the scala-library being excluded) or vice versa.
> The reason is to prevent library users from ending up with two conflicting versions `foo_2.13` and `foo_3` of the same foo library in their classpath, this problem being unsolvable in some cases.
