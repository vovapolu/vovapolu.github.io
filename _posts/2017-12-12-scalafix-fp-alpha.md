---
layout: post
title:  "Disable your non FP code by Scalafix (installation guide)"
date:   2017-12-14
categories: scalafix
---

_This is a short installation guide for the new `DisableUnless` rule that I recently added to the [scalafix](https://scalacenter.github.io/scalafix/)._

### DisableUnless
The `DisableUnless` rule bans usages of "disabled" symbols unless in a "safe" block. 

Let's look at a simple example.


With the given config:
```
DisableUnless.symbols = [
    {
        block = "test.Test.IO"
        symbol = "scala.Predef.println"
        message = "println has side-effects"
    }
]
```

We got several linter errors in the following code:
```scala
package test

object Test {
    object IO {
        def apply[T](run: => T) = ???
    }


    println("hi") // not ok

    IO {
        println("hi") // ok
    }

    IO {
        def sideEffect(i: Int) = println("not good!") // not ok
        (i: Int) => println("also not good!") // not ok
    }
}
```

## Snaphost installation 
Currently rule isn't in the official release, so you should install a snapshot version to use it. 

Scalafix CI publishes a snapshot release to Sonatype on every merge into master. Each snapshot release has a unique version number, jars don’t get overwritten. To find the latest snapshot version number, go <https://oss.sonatype.org/content/repositories/snapshots/ch/epfl/scala/scalafix-core_2.12/> and select the version number at the bottom, the one with the latest “Last Modified”. Once you have found the version number, adapting the version number

```sbt
// If using sbt-scalafix, add to project/plugins.sbt
resolvers += Resolver.sonatypeRepo("snapshots")
addSbtPlugin("ch.epfl.scala" % "sbt-scalafix" % "<SNAPSHOT-version>")

// If using scalafix-cli, launch with coursier
coursier launch ch.epfl.scala:scalafix-cli_2.12.4:<SNAPSHOT-version> -r sonatype:snapshots --main scalafix.cli.Cli -- --help
```

And then create `.scalafix.conf` file in the root of our project. See more [here](https://scalacenter.github.io/scalafix/docs/users/configuration).

## Configuration

By default, this rule does allow all symbols. To disallow a symbol in a block:
```
DisableUnless.symbols = [
    {
        block = "scala.Option"
        symbol = "dangerousFunction"
        message = "the function may return null"
    }
]
```
Message is optional parameter and could be used to provide custom errors. 


## Quick-start config
To quickly start using this rule and scalafix overall I suggest a very simple config. 
It blocks some "not safe" stuff that usually indicates that program has side-effects/nullable values. 
But to use the full power of `DisableUnless` rule this config should be project specific.

```
rules = [DisableUnless]

DisableUnless.symbols = [
    {
        block = "Your.IO.Class"
        symbol = "scala.Predef.println"
    }
    {
        block = "Your.IO.Class"
        symbol = "java.lang.System.currentTimeMillis"
    }
    {
        block = "Your.IO.Class"
        symbol = "scala.io.Source.fromFile"
    }
    {
        block = "Your.IO.Class"
        symbol = "java.io.FileInputStream.read"
    }
]
```