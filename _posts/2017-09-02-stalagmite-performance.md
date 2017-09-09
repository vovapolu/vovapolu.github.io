---
layout: post
title:  "Stalagmite and its speed"
date:   2017-09-02 17:20:01 +0300
categories: scala stalagmite perf
---

*Russian version of this post is [here]({% post_url 2017-09-02-stalagmite-performance-ru %})*

This summer I participated in the project [Google Summer of Code](https://summerofcode.withgoogle.com/projects/#4661695229198336) in the [Scala](https://www.scala-lang.org/) organization. My task was writing a library ([**stalagmite**](https://github.com/fommil/stalagmite)), which could generate by [scala.meta](http://scalameta.org/) *effective* and customizable replacement of conventional [`case classes`](http://docs.scala-lang.org/tour/case-classes.html) with some convenient optimizations. Let's analyze this description: 
- **Effective**, that is comparable speed of functions already provided in the `case class', and more or less fast implementation of additional ones
- **Customizable** -- the ability to enable and disable various modes of code generation
- **Replacement** -- support for already available methods and functionality
- **Optimizations** -- reducing memory usage, boilerplate, increasing speed via macro generation

This list looks great, but making complete replacement for the powerful and elaborated `case classes` is difficult. There are many ~~hacks~~ special cases that are handled by the Scala compiler. The aim of the project was to support the necessary subset of such "cases" along with additional settings, the implementation of which without the use of macro-generation would require writing boilerplate code and bulky structures within the class. It was also required to provide an acceptable working time of macro-generated code, that I will check further by benchmarks. 

## Current status of the project
Currently the project provides 3 main operation mods together with a set of small improvements:
- **The duplication of the behavior of `case class`**, in this mode all the basic functions is generated: `toString`, `hashCode`, `copy`, `apply`, `unapply`, `Product` and `Serializable` handled, generation of `Generic LabelledGeneric, Typeable` from [`Shapeless`](https://github.com/milessabin/shapeless)
 - **[Memoisation](https://ru.wikipedia.org/wiki/%D0%9C%D0%B5%D0%BC%D0%BE%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F), interning**, this technique disallows to duplicate objects that are equal *by value*, thus keeps in memory only one object and multiple references to it. It reduces the memory usage and time for comparing objects (object equality is equivalent to reference equality), but the creation of each object requires more resources. Stalagmite supports the interning of both the class and the fields inside it. There are also two sub-mods: *weak* and *strong* memoisation. The first mode stores the class objects within a pool using a *weak reference*, allowing the garbage collector to remove unused instances, the second mode keeps the normal link, and prohibits removing from memory objects, that get to the pool. 
 - **"Packaging" the class fields** (in the project called `heap optimization`), this mode reduces the amount of memory for each instance of the class using the `Boolean` and `Option[AnyVal]` (for primitives) "packaging" in the bitmask and converting fields of type `Option[T]` to the type `T` that accepts `null` for the original case of `None`. 

Generated classes with these mods also takes into account important property of immutability. Memoisation and fields packing implicitly use it. 

Work on the library is not finished yet, many problems and ideas are documented in the [repository](https://github.com/fommil/stalagmite) `Issues`. Visit it if you are interested in the fate of the project. 

## Benchmarks
*Next there will be the long description of the benchmarks. For the impatient and those who want to learn only the results of performance testing there's a **TL;DR** section at the end*

### Working time
First, let's examine the time benchmarks, that used the [JMH](http://openjdk.java.net/projects/code-tools/jmh/).
Each of them represents the applying of some method to a collection of 5000 tuples or instances of a certain class. Number of such applyings in a second was counted for collection on average on each of the 15 threads. The results are given for processor `Intel Core i5-4210U @ 1.70 GHz`.

I will use the following notation for classes used in benchmarks: 
- `caseClass` -- normal `case class`
- `caseClassMeta` -- macro-generated class that duplicates the functionality of `caseClass`
- `memoisedCaseClass` -- also an ordinary `case class`, created for comparison with memorization verison
- `memoisedMeta` -- the generated class with `strong memoisation`
- `memoisedWeak` -- the generated class with `weak memoisation`
- `optimizeHeapCaseClass` -- `case class` for comparison with `heap optimization`
- `optimizeHeapMeta` -- the generated class with `heap optimization`

*TL;DR* There are three groups of classes that were used during testing. `caseClass.*` tested duplication of `case classes`, `memoised.*` tested memorization, `optimizeHeap*` -- fields packaging. `*caseClass` served as the baselines for each group, and were simple `case classes`. 

The first group of benchmarks checks the speed of the main methods for `case class`: `apply`, `copy`, field access, `hashCode`, `Product` methods, serialization, `toString`, 

##### apply
Collection consisted of tuples with fields for creating classes. For each tuple the `apply` method from corresponding class was called. 

| Class | Result |
| - | - |
| ` ApplyBenchmark.caseClass ` | ` 19136.554 ± 637.993  ops/s ` |
| ` ApplyBenchmark.caseClassMeta ` | ` 18331.622 ± 510.110  ops/s ` |
| ` ApplyBenchmark.memoisedCaseClass ` | ` 20007.336 ± 276.490  ops/s ` |
| ` ApplyBenchmark.memoisedMeta ` | ` 2560.862 ±  30.396  ops/s ` |
| ` ApplyBenchmark.memoisedMetaWeak ` | ` 3714.472 ±  21.630  ops/s ` |
| ` ApplyBenchmark.optimizeHeapCaseClass ` | ` 19397.820 ± 509.561  ops/s ` |
| ` ApplyBenchmark.optimizeHeapMeta ` | ` 3949.541 ±  54.173  ops/s ` |

`memoised` and `optimizeHeap` modes have the overhead of memoization and fields packaging, thus work slower than regular `case class`. `caseClass` shows the same speed as `case class`.

#### copy
For each class instance in the collection 2 copies with different values in one of the two fields were created.

| Class | Result |
| - | - |
| ` CopyBenchmark.caseClass ` | ` 5080.050 ± 284.304  ops/s ` |
| ` CopyBenchmark.caseClassMeta ` | ` 5432.313 ± 336.334  ops/s ` |
| ` CopyBenchmark.memoisedCaseClass ` | ` 4998.074 ± 173.456  ops/s ` |
| ` CopyBenchmark.memoisedMeta ` | ` 853.461 ±   8.121  ops/s ` |
| ` CopyBenchmark.memoisedMetaWeak ` | ` 1159.225 ±  63.359  ops/s ` |
| ` CopyBenchmark.optimizeHeapCaseClass ` | ` 6899.622 ± 129.428  ops/s ` |
| ` CopyBenchmark.optimizeHeapMeta ` | ` 959.581 ±  11.433  ops/s ` |

The situation is similar to the previous benchmark. Method `copy` just creates a new object using `apply`. 

#### Access to the fields
Each field of a class instance in the collection was read.

| Class | Result |
| - | - |
| ` FieldAccessBenchmark.caseClass ` | ` 15390.926 ± 122.415  ops/s ` |
| ` FieldAccessBenchmark.caseClassMeta ` | ` 15433.422 ± 254.224  ops/s ` |
| ` FieldAccessBenchmark.optimizeHeapCaseClass ` | ` 22975.564 ± 803.286  ops/s ` |
| ` FieldAccessBenchmark.optimizeHeapMeta ` | ` 4535.168 ±  50.179  ops/s ` |

`caseClass` mode doesn't differ from `case class`, field access method returns the actual field from the class. `optimizeHeap` mode performs the "unpacking" of the fields while reading, so working time is longer. `memoised` mode wasn't included, because it's equivalent to `caseClass` in this context. 

#### `.hashCode` and `.toString`
For each class instance methods `.hashCode'and `.toString` were called. 

| Class | Result |
| - | - |
| ` HashCodeBenchmark.caseClass ` | ` 11307.343 ± 1278.621 ops/s ` |
| ` HashCodeBenchmark.caseClassMeta ` | ` 12106.911 ± 263.225 ops/s ` |
| ` HashCodeBenchmark.memoisedCaseClass ` | ` 16266.437 ± 292.010 ops/s ` |
| ` HashCodeBenchmark.memoisedMeta ` | ` 24262.046 ± 1876.971 ops/s ` |
| ` HashCodeBenchmark.optimizeHeapCaseClass ` | ` 4208.655 ± 323.614 ops/s ` |
| ` HashCodeBenchmark.optimizeHeapMeta ` | ` 3078.390 ± 210.247 ops/s ` |

| Class | Result |
| - | - |
| ` ToStringBenchmark.caseClass ` | ` 1145.058 ± 34.190 ops/s ` |
| ` ToStringBenchmark.caseClassMeta ` | ` 1296.309 ± 22.632 ops/s ` |
| ` ToStringBenchmark.memoisedCaseClass ` | ` 1439.810 ± 106.486 ops/s ` |
| ` ToStringBenchmark.memoisedMeta ` | ` 26745.501 ± 1026.738 ops/s ` |
| ` ToStringBenchmark.optimizeHeapCaseClass ` | ` 548.055 ± 98.772 ops/s ` |
| ` ToStringBenchmark.optimizeHeapMeta ` | ` 646.945 ± 29.493 ops/s ` |

For `caseClass` and `optimizeHeap` modes the situation is similar as in the field access. Methods `.hashCode` and `.toString` get fields of the class for building hashes and strings, though still doing a lot of other work, so the difference is small. `memoised` mode works faster than `case class` baseline because of the special settings: `memoisedHashCode` and `memoisedToString`. They save the values of the two methods, and don't count them many times. 

#### `productElement` 
This benchmark tested method `productElement`. 

| Class | Result |
| - | - |
| ` ProductElementBenchmark.caseClass ` | ` 13921.991 ± 631.164 ops/s ` |
| ` ProductElementBenchmark.caseClassMeta ` | ` 13871.084 ± 1241.714 ops/s ` |
| ` ProductElementBenchmark.optimizeHeapCaseClass ` | ` 21984.665 ± 636.940 ops/s ` |
| ` ProductElementBenchmark.optimizeHeapMeta ` | ` 4305.256 ± 73.872 ops/s ` |

Everything is the same as in benchmark on field access. 

#### Serialization
The entire collection of class instances was serialized and then deserialized. 

| Class | Result |
| - | - |
| ` SerializationBenchmark.caseClass ` | ` 190.118 ± 19.615  ops/s ` |
| ` SerializationBenchmark.caseClassMeta ` | ` 205.555 ± 29.683  ops/s ` |
| ` SerializationBenchmark.memoisedCaseClass ` | ` 250.941 ± 34.648  ops/s ` |
| ` SerializationBenchmark.memoisedMeta ` | ` 316.477 ± 30.002  ops/s ` |
| ` SerializationBenchmark.memoisedMetaWeak ` | ` 338.591 ± 39.945  ops/s ` |
| ` SerializationBenchmark.optimizeHeapCaseClass ` | ` 93.594 ± 18.978  ops/s ` |
| ` SerializationBenchmark.optimizeHeapMeta ` | ` 66.927 ±  8.449  ops/s ` |

There are almost no differences from `case class` baselines. Complex operations of writing and reading serialized data outshine quick fields accessing and objects creating. Good conclusion -- even difficult fields packing in `optimizeHeap` and memoization does not affect the objects serialization. 

#### unapply
Each class instance was pattern-matched.

| Class | Result |
| - | - |
| ` UnapplyBenchmark.caseClass ` | ` 14816.202 ± 939.746  ops/s ` |
| ` UnapplyBenchmark.caseClassMeta ` | ` 13392.518 ± 431.781  ops/s ` |
| ` UnapplyBenchmark.optimizeHeapCaseClass ` | ` 23635.361 ± 238.981  ops/s ` |
| ` UnapplyBenchmark.optimizeHeapMeta ` | ` 4082.947 ±  61.865  ops/s ` |

`optimizeHeap` mode shows slow results due to the fields unpacking, `caseClass` mode shows good performance regarding the `case class`. 

The next group of benchmarks tested the generating mods features: methods to support `Shapeless` and speed of `.equals` with memoization. 

### Shapeless
`shapeless` mod generates objects of type `Generic` and `LabelledGeneric` that allow conversion of class instances to `Hlist` and back. Such conversions were performed for the entire collection. 

| Class | Result |
| - | - |
| ` ShapelessBenchmark.caseClass ` | ` 8406.639 ± 342.925  ops/s ` |
| ` ShapelessBenchmark.caseClassMeta ` | ` 6653.700 ±  38.209  ops/s ` |

Generated class works slower. Reasons aren't clear, probably it's because of `Shapeless` macro-generation. 

#### `.equals` within `Vector`
This benchmark tested speed of `.equals` method in the case when data is placed in a `Vector`. A set of 1000000 random pairs of indices for comparisons was choosen. The indices were chosen with distance 10 or less and gradually increased, starting with 1. It allows to support data locality and data structure `Vector` is able to provide it. 

| Class | Result |
| - | - |
| ` EqualsVectorBenchmark.caseClass ` | ` 29.456 ± 1.851 ops/s ` |
| ` EqualsVectorBenchmark.caseClassMeta ` | ` 30.252 ± 1.707 ops/s ` |
| ` EqualsVectorBenchmark.memoisedCaseClass ` | ` 36.041 ± 3.207 ops/s ` |
| ` EqualsVectorBenchmark.memoisedMeta ` | ` 36.401 ± 1.061 ops/s ` |
| ` EqualsVectorBenchmark.memoisedWeak ` | ` 36.737 ± 2.372 ops/s ` |
| ` EqualsVectorBenchmark.optimizeHeapCaseClass ` | ` 25.922 ± 1.398 ops/s ` |
| ` EqualsVectorBenchmark.optimizeHeapMeta ` | ` 11.473 ± 0.828 ops/s ` |

`caseClass` and `memoised` modes work as well as `case class`. In the first case there aren't any differences in the implementations, in the second comparison *by reference*, not by *value* was used in the generated classes. It should work faster, but it doesn't. Time to access an element in `Vector` overlaps all. In the case of `optimizeHeap` unpacking operation slows down `.equals`, even compared to the accessing time to the elements. 

#### `.equlas` inner `HashSet`
`HashSet` uses `.equals` and `.hashCode` to build the hash table. The time for the entire collection of instances of class to be written into an empty `HashSet` was meausred.  

| Class | Result |
| - | - |
| ` HashSetBenchmark.caseClass ` | ` 3341.499 ± 44.082 ops/s ` |
| ` HashSetBenchmark.caseClassMeta ` | ` 3665.179 ± 77.763 ops/s ` |
| ` HashSetBenchmark.memoisedCaseClass ` | ` 3858.759 ± 49.616 ops/s ` |
| ` HashSetBenchmark.memoisedIntern ` | ` 5009.943 ± 137.088 ops/s ` |
| ` HashSetBenchmark.memoisedMeta ` | ` 6776.364 ± 39.766 ops/s ` |
| ` HashSetBenchmark.memoisedWeak ` | ` 6401.936 ± 139.723 ops/s ` |
| ` HashSetBenchmark.optimizeHeapCaseClass ` | ` 1566.175 ± 70.254 ops/s ` |
| ` HashSetBenchmark.optimizeHeapMeta ` | ` 869.202 ± 22.643 ops/s ` |

`caseClass` and `optimizeHeap` modes work normally. The first is similar to the baseline, the second is slower. But `memoised` mode here is way more interesting! The new class `memoisedIntern` is introduced here, it uses `memoisedHashCode` setting. It caches the `hashCode` and reduces the accessing time to the hash code. But even without it the generated class is faster than `case class` because of accelerated `.equals` method. With the caching of the hash code speed is greatly increased. 

## Memory
Benchmarks on memory usage worked simple. Two characteristics were measured: how much memory was used during the iteration and how much was used from the start of the run. The second characteristic is necessary to check how much memory, that cannot be removed by the garbage collector, is being producing. Several iterations was performed with output of memory usage dynamic. 

There were 4 benchmarks: 
- memory usage measurment for `case class` duplication mode
- measurement for packing fields
- measurement for memoization when the data cannot be removed by the garbage collector
- and when it can be

#### `case class`
Memory usage of 500 thousand `case classes` and generated classes was compared. Each of them consisted of the following fields: `i: Int, b: Boolean, s: String`.

**`caseClass`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 54659 kb | 54571 kb |
| 1 | 54574 kb | 54499 kb |
| 2 | 54877 kb | 54689 kb |
| 3 | 54801 kb | 54570 kb |
| 4 | 54691 kb | 54595 kb |

**Generated class**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 55032 kb | 54934 kb |
| 1 | 54740 kb | 54626 kb |
| 2 | 54896 kb | 54835 kb |
| 3 | 54687 kb | 54612 kb |
| 4 | 54920 kb | 54845 kb |

Absolutely identical. 

#### Packaging fields
A similar comparison was performed: a class with fields packaging against `case class`. 
They contain the fields: `i: Option[Int],
s: Option[String],
b1: Option[Boolean],
b2: Option[Boolean],
b3: Option[Boolean],
b4: Option[Boolean]`

**`caseClass`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 111830 kb | 111732 kb |
| 1 | 113230 kb | 112418 kb |
| 2 | 112626 kb | 112325 kb |
| 3 | 112698 kb | 112315 kb |
| 4 | 113117 kb | 112715 kb |

**The package fields**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 40399 kb | 40281 kb |
| 1 | 42455 kb | 39790 kb |
| 2 | 42949 kb | 40088 kb |
| 3 | 42539 kb | 39811 kb |
| 4 | 43195 kb | 40395 kb |

Fileds packaging reduces memory usage by almost 3 times. This benchmark clearly shows how redundant are `Option` and `Boolean` types.

#### Memoization without deletable data
Let's finally proceed to the most interesting and demonstrative memory benchmarks. In the first created instances of classes could not be removed by the garbage collector. There were 4 classes: `case class`, with strong memoization, with strong memoization and inner fields interning, and weak memoization. They contain the following fields: `b: Boolean, s: String` and the string fields `s` was interned in the 3rd case. Two types of data were used to create instances of these classes: data with a large number of repetitions and data with various elements. 

##### Data with repetitions
500 thousand elements, the 2-length strings of digits and letters. The size of all different combinations of such strings and `Boolean` is much smaller than the size of the collection, so there were many duplicates. 

**`caseClass`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 45174 kb | 50249 kb |
| 1 | 46968 kb | 50343 kb |
| 2 | 47010 kb | 50478 kb |
| 3 | 46936 kb | 50540 kb |
| 4 | 46980 kb | 50646 kb |

**`memoisedMeta`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 11831 kb | 11537 kb |
| 1 | 11781 kb | 11570 kb |
| 2 | 11770 kb | 11601 kb |
| 3 | 11729 kb | 11601 kb |
| 4 | 11729 kb | 11601 kb |

**`memoisedMeta` with string interning**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 11733 kb | 11732 kb |
| 1 | 11718 kb | 11732 kb |
| 2 | 11729 kb | 11742 kb |
| 3 | 11718 kb | 11742 kb |
| 4 | 11728 kb | 11753 kb |

**`memoisedWeak`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 11666 kb | 11659 kb |
| 1 | 11684 kb | 11680 kb |
| 2 | 11663 kb | 11680 kb |
| 3 | 11662 kb | 11680 kb |
| 4 | 11683 kb | 11700 kb |

Memoization is doing its job. All repetitions in data are added to the cache and not duplicated as in the case of `case class`. 

##### Data without repetition
Also 500 thousand elements, but the strings have length 5. The data hasn't repetitions now. 

**`caseClass`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 54937 kb | 54938 kb |
| 1 | 53696 kb | 53724 kb |
| 2 | 54271 kb | 53308 kb |
| 3 | 54590 kb | 53211 kb |
| 4 | 54667 kb | 53191 kb |

**`memoisedMeta`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 78619 kb | 77984 kb |
| 1 | 74883 kb | 140421 kb |
| 2 | 82160 kb | 210863 kb |
| 3 | 75171 kb | 273346 kb |
| 4 | 74882 kb | 336093 kb |

**`memoisedMeta` with string interning**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 95091 kb | 94141 kb  |
| 1 | 86867 kb | 168425 kb |
| 2 | 102367 kb | 259074 kb |
| 3 | 86810 kb | 333071 kb |
| 4 | 86753 kb | 407689 kb |

**`memoisedWeak`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 105562 kb | 105562 kb |
| 1 | 66527 kb | 105683 kb|
| 2 | 66392 kb | 105669 kb |
| 3 | 66423 kb | 105686 kb |
| 4 | 66406 kb | 105686 kb |

Here `case class`'s winning. Strong memoization is continuously writing new data to the cache, consuming a large amount of memory. String interning only worsens the situation. Weak memoization uses the memory better, but it has overhead in cache building. 

#### Memoization with the garbage collector
In this benchmark after creating 500 thousand instances only 20000 references to them were left. This allowed the garbage collector to do its ~~dirty~~ work. 

##### Data with repetitions

**`caseClass`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 1914 kb | 1923 kb |
| 1 | 1871 kb | 1920 kb |
| 2 | 1852 kb | 1897 kb |
| 3 | 1856 kb | 1879 kb |
| 4 | 1859 kb | 1863 kb |

**`memoisedMeta`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 1662 kb | 1663 kb |
| 1 | 445 kb | 1376 kb |
| 2 | 459 kb | 1367 kb |
| 3 | 448 kb | 1347 kb |
| 4 | 468 kb | 1347 kb |

**`memoisedMeta` with string interning**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 1347 kb | 1348 kb |
| 1 | 473 kb | 1351 kb |
| 2 | 458 kb | 1341 kb |
| 3 | 458 kb | 1331 kb |
| 4 | 468 kb | 1331 kb |

**`memoisedWeak`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 1583 kb | 1583 kb |
| 1 | 900 kb | 1521 kb |
| 2 | 886 kb | 1496 kb |
| 3 | 906 kb | 1494 kb |
| 4 | 909 kb | 1498 kb |

All the different combinations of data aggin can be fully written to the cache. Strong memoization gives a good improvement over the `case class`, weak a little worse. 

##### Data without repetition

**`caseClass`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 2299 kb | 2299 kb |
| 1 | 2228 kb | 2341 kb |
| 2 | 2187 kb | 2341 kb |
| 3 | 2207 kb | 2341 kb |
| 4 | 2187 kb | 2341 kb |

**`memoisedMeta`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 66468 kb | 66468 kb |
| 1 | 66920 kb | 132920 kb |
| 2 | 62015 kb | 193819 kb |
| 3 | 70686 kb | 264037 kb |
| 4 | 62875 kb | 326444 kb |

**`memoisedMeta` with string interning**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 82753 kb | 82754 kb |
| 1 | 81617 kb | 163902 kb |
| 2 | 75799 kb | 239233 kb |
| 3 | 91769 kb | 330278 kb |
| 4 | 75296 kb | 403948 kb |

**`memoisedWeak`**

| Iteration | Memory for iteration | Memory after iteration |
| - | - | - |
| 0 | 41861 kb | 41861 kb |
| 1 | 3089 kb | 42294 kb |
| 2 | 2744 kb | 42382 kb |
| 3 | 2676 kb | 42402 kb |
| 4 | 2676 kb | 42423 kb |

Here everything is much worse. Strong memoization stores all the data in the cache, nothing is deleted, and the cache is becoming huge. Weak memoisation stores only what is needed for iteration in the cache, but it still takes a lot of memory. This is the worst case to use memoization. It could easily lead to `OutOfMemoryError`. 

### *TL;DR* 
The speed and memory consumption measurements show that there's no such situation when the generated class is definitely better than the `case class`. There's either the gain in speed, but with more memory usage, or vice versa. Generated classes, if no additional generation modes applied, duplicate the effectiveness of the `case class`. Field packaging consumes less memory, but behaves slowly in accessing to class fields and `.apply` method. Memoization speeds up the `.equals` method, but also has overhead in `.apply`. Memory usage depends on the context. With a large number of duplicates memoization caches them and stores only references. When data has almost no duplicates, especially in the case when data is being cleaned by the garbage collector, the memory consumption increases significantly.

## In the end 
The conclusion will be simple. Built-in `case classes` work very well in most cases. Stalagmite provides an alternative to them, exchanging memory for speed and vice versa. There are some ideas on how to reduce the boilerplate and add new optimizations in developing plans, but complete replacement the `case class` is impossible. So use this library wisely. I hope I showed you the strengths and weaknesses of the current implementation :)
