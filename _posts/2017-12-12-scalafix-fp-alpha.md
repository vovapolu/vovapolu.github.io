---
layout: post
title:  "Disable your non FP code by Scalafix (installation guide)"
date:   2017-09-02 17:20:01 0300
categories: scala scalafix install disable unless
---

_This is a short installation guide for the new `DisableUnless` rule that I recently added to the [scalafix](https://scalacenter.github.io/scalafix/)._

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

### DisableUnless
The `DisableUnless` rule bans usages of "disabled" symbols unless in a "safe" block. 
 
Any inner blocks (e.g. anonymous functions or classes) 
within the given "safe" blocks are banned again, to avoid leakage. 

## Configuration

By default, this rule does allows all symbols. To disallow a symbol in a block:
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

## Example
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

## Quick-start config
To quickly start using this rule and scalafix overall I suggest a simple config. 
It blocks basic "not safe" stuff that usually indicates that program has side-effects/nullable values. 

```
rules = [DisableUnless]

DisableUnless.symbols = [
    {
        block = "test.Test.IO"
        symbol = "scala.Predef.println"
    }
]
```