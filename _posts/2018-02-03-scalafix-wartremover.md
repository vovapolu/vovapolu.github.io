---
layout: post
title:  "Wartremover porting to scalafix"
date:   2018-02-03
categories: scalafix
---

During the last two months I was working on several tasks in the [scalafix repository](https://github.com/scalacenter/scalafix)
(See [my last post](https://vovapolu.github.io/scalafix/2017/12/14/scalafix-fp-alpha.html) and 
[scalafix docs](https://scalacenter.github.io/scalafix/docs/users/rules) for more cool stuff).
Among them was porting [wartremover](http://www.wartremover.org/doc/warts.html) rules to the scalafix. 
I've created a [simple repo](https://github.com/vovapolu/ScalafixWartremover/tree/master/scalafix/input/src/main/scala/fix) with example configs and tests to illustrate the current state of porting. 
Please visit it if you're interested :)
It's still in progress, and some rules aren't implemented yet, but the existing ones are already quite impressive! Let's take a closer look at them. 

## Wartremover

Wartremover rules consist of two parts: `Unsafe` subset and ... others. 
I won't describe the behavior of every rule, you can find description in [the official doc](http://www.wartremover.org/doc/warts.html). 
Besides the names are quite expressive, it's not hard to guess what the particular rule does. 

### Unsafe

`Unsafe` rules are most common and guard you from the most obvious bugs, like using `None.get` or `.asInstanceOf` instead of pattern-matching. 
Here's a list of these rules:
- Any
- AsInstanceOf
- EitherProjectionPartial
- IsInstanceOf
- NonUnitStatements _(not implemented in the scalafix yet)_
- Null 
- OptionPartial
- Product
- Return
- Serializable 
- StringPlusAny _(partially implemented)_
- Throw 
- TraversableOps _[will be soon]_
- TryPartial
- Var

Scalafix porting lacks one rule and one is half done. 
This is due to the [absence of proper type detection of arbitrary term in the SemanticDB](https://github.com/scalameta/scalameta/issues/1212). 
After fixing this issue it will be easy to finally complete this list :)

To look at the example scalafix config and tests of Unsafe rules you can visit 
[my small repo](https://github.com/vovapolu/ScalafixWartremover/blob/master/scalafix/input/src/main/scala/fix/ScalafixWartremoverUnsafe.scala). 

### All rules

Consider the rest of wartremover rules that aren't in the `Unsafe` set:
- AnyVal
- ArrayEquals _(not implemented because of the aforementioned SemanticDB issue)_
- DefaultArguments
- Enumeration
- Equals
- ExplicitImplicitTypes _(this rule's covered by more general scalafix `ExplicitResultTypes`, but still need to be polished)_
- FinalCaseClass
- FinalVal
- ImplicitConversion
- ImplicitParameter _(not implemented yet)_
- JavaConversions _[will be soon]_
- JavaSerializable
- LeakingSealed
- MutableDataStructures _[will be soon]_
- Nothing _(not implemented yet)_
- Option2Iterable
- Overloading _(not implemented yet, arguable rule)_
- PublicInference _(not implemented yet)_
- Recursion _(not implemented yet, arguable rule)_
- ToString
- While

Yeah, this list is more pessimistic than the last. 
But these rules aren't so common and some of them should be definitely discussed before adding to the scalafix (for instance, blocking of overloading or recursion). 

Again you can go to the 
[my repo](https://github.com/vovapolu/ScalafixWartremover/blob/master/scalafix/input/src/main/scala/fix/ScalafixWartremoverAll.scala) 
to take a look at the scalafix config that contains all already implemented wartremover rules. 


### Small conclusion 
There's a lot of work to be done, as you see. 
Some rules merely aren't implemented because of bugs and lack of the time, some of them are implemented only partially and some require more thoughtful discussion. 
But nevertheless current functionality is a good reason to start using scalafix from scratch or port your wartremover config. 

