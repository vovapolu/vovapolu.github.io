---
layout: post
title:  "Disable your non FP code by Scalafix (installation guide)"
date:   2017-12-14
categories: scalafix
---

_This is a short installation guide for the new `DisableUnless` rule that I recently added to [scalafix](https://scalacenter.github.io/scalafix/)._

### DisableUnless
The `DisableUnless` rule bans usages of "disabled" symbols unless in a "safe" block. 

Let's look at a simple example.


With the given config:
```
DisableUnless.symbols = [
    {
        unless = "test.DisableUnless.IO"
        symbols = [
            {
                symbol = "scala.Predef.println"
                message = "println has side-effects"
            }
        ]
    }
]
```

We get several linter errors in the following code:
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

```sbt
// If using sbt-scalafix, add to project/plugins.sbt
resolvers += Resolver.sonatypeRepo("snapshots")
addSbtPlugin("ch.epfl.scala" % "sbt-scalafix" % "0.5.7-70-7c4efe55-SNAPSHOT")

// If using scalafix-cli, launch with coursier
coursier launch ch.epfl.scala:scalafix-cli_2.12.4:0.5.7-70-7c4efe55-SNAPSHOT -r sonatype:snapshots --main scalafix.cli.Cli -- --help
```

And then create `.scalafix.conf` file in the root of our project. See more [here](https://scalacenter.github.io/scalafix/docs/users/configuration).

## Configuration

By default, all symbols are permitted. To disallow a symbol in a block:
```
DisableUnless.symbols = [
    {
        unless = "scala.Option"
        symbols = [
            {
                symbol = "test.DisableUnless.dangerousFunction"
                message = "the function may return null"
            }
            "test.DisableUnless.anotherFunction"
        ]
    }
]
```
You can use objects or regular strings to specify blocked symbols, 
`message` is optional parameter and could be used to provide custom errors. 

## Quick-start config
To quickly start using this rule and scalafix overall I suggest a very simple config. 
It blocks some "not safe" stuff that usually indicates that program has side-effects/nullable values. 

```
rules = [DisableUnless]

DisableUnless.symbols = [
    {
        unless = "Your.IO.Class"
        symbols = {
            "scala.Predef.println"
            "java.lang.System.currentTimeMillis"
            "scala.io.Source.fromFile"
            "java.io.FileInputStream.read"
        }
    }
]
```

But to use the full power of `DisableUnless` rule this config should be project specific.
_We hope that the community will share per-library banlists as the tool gains users._
