---
layout: post
title:  "Stalagmite и его скорость работы"
date:   2017-09-02 17:20:01 +0300
categories: scala stalagmite perf
---

Это лето я участвовал в проекте [Google Summer of Code](https://summerofcode.withgoogle.com/projects/#4661695229198336) в организации [Scala](https://www.scala-lang.org/). Моей задачей являлось написание библиотеки ([**stalagmite**](https://github.com/fommil/stalagmite)), которая при помощи [scala.meta](http://scalameta.org/) могла бы генерировать *эффективную* и настраиваемую замену обычным [`case class`](http://docs.scala-lang.org/tour/case-classes.html)ам с некоторыми удобными оптимизациями. Разберем подробнее это описание:
- **Эффективность**, то есть сравнимая скорость работы для функций, уже представленных в `case class`е, и более-менее быстрая реализация дополнительных
- **Настраиваемость** -- возможность включать и отключать различные режимы генерации кода
- **Замена** -- поддержка уже доступных методов и функциональности
- **Оптимизации** -- сокращение повторений кода (boilerplate), уменьшение используемой памяти и повышения скорости работы через макро-генерацию.

Этот список отлично выглядит, но сделать полноценную замену мощным и проработанным `case class`ам сложно. Есть множество ~~костылей~~ особых случаев, которые обрабатывает компилятор Scala. Целью проекта было поддержать необходимое подмножество таких "случаев" вместе с дополнительными настройками, реализация которых без использования макро-генерации потребовала бы написание похожего кода и громоздких конструкций внутри класса. Также требовалось обеспечить приемлемое время работы макро-генерирумого кода, что я и проверю дальше с помощью бенчмарков.

## Текущее состояние проекта
В настоящий момент проект предоставляет 3 главных режима работы вместе с набором небольших улучшений:
- **Дублирование поведения `case class`а**, в этом режиме генерируются все основные функции: `toString`, `hashCode`, `copy`, `apply`, `unapply`, методы `Product`, `Serializable`, также есть генерация `Generic, LabelledGeneric, Typeable` для [`Shapeless`](https://github.com/milessabin/shapeless)
- **[Мемоизация](https://ru.wikipedia.org/wiki/%D0%9C%D0%B5%D0%BC%D0%BE%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F) (memoisation), интернирование (interning)**, это техника позволяет не дублировать объекты, равные *по значению*, тем самым иметь в памяти только один объект и множество ссылок на него. Это сокращает используемую память, уменьшает время работы сравнения объектов (равенство объектов эквивалентно равенству ссылок), но создание каждого объекта требует больших ресурсов. Stalagmite поддерживает интернирование как самого класса, так и полей внутри него. Также есть два режима работы: слабая мемоизация (weak memoisation) и строгая мемоизация (strong memoiastion). Первый режим хранит объекты класса внутри пула, используя слабую ссылку (weak reference), позволяя сборщику мусора удалять неиспользуемые экземпляры, второй режим хранит обычную ссылку и запрещает удаление из памяти объектов, попавших в пул.
- **"Упаковка" полей класса** (в проекте имеет название `heap optimization`), этот режим уменьшает количество используемой памяти на каждый экземпляр класса с помощью "упаковки" `Boolean` и `Option[AnyVal]` (для примитивов) в битовую маску и преобразовании поля типа `Option[T]` в поле тип `T`, которое принимает `null` для изначального случая `None`.

Также сгенерированные классы учитывают важное свойство неизменяимости. Его неявно использует режимы мемоизации и упаковки полей. 

Работа над библиотекой еще не закончена, многие проблемы и идеи зафиксированы в разделе [репозитория](https://github.com/fommil/stalagmite) `Issues`. Заходите туда, если вам интересна судьба проекта.

## Бенчмарки
*Далее будет длинное описание бенчмарков. Для нетерпеливых и тех, кто хочет узнать только результаты тестирования производительности, в конце есть секция **TL;DR**.*

### Время работы
Сначала разберем бенчмарки на время работы, которые использовали [JMH](http://openjdk.java.net/projects/code-tools/jmh/).
Каждый из них представлял запуск некоторого метода на коллекции из 5000 кортежей (tuple) или объектов определенного класса. Считалось количество обработок такой коллекции в секунду в среднем на каждом из 15и тредов. Результаты приведены для процессора `Intel Core i5-4210U @ 1.70GHz`.

Далее я буду использовать следующие обозначения классов, на которых запускался бенчмарк:
- `caseClass` -- обычный `case class`
- `caseClassMeta` -- макро-сгенерированный класс, дублирующий функциональность `caseClass`
- `memoisedCaseClass` -- тоже обычный `case class`, созданный для сравнения с режимом мемоизации
- `memoisedMeta` -- сгенерированный класс с режимом `strong memoisation`
- `memoisedWeak` -- сгенерированный класс с режимом `weak memoisation`
- `optimizeHeapCaseClass` -- `case class` для сравнения с режимом `heap optimization`
- `optimizeHeapMeta` -- сгенерированный класс с `heap optimization`

*TL;DR* Есть три группы классов, которые использовались при тестировании. `caseClass.*` проверяли режим дублирования `case class`, `memoised.*` проверяли режим мемоизации, `optimizeHeap*` -- режим упаковки полей. Классы `.*caseClass` служили бенчмарками каждой группы и являлись простыми `case class`ами.

Первая группа бенчмарков состояла в проверке скорости основных методов для `case class`а: `apply`, `copy`, доступ к полям класса, `hashCode`, методы `Product`, сериализация, `toString`,

##### apply
Для каждого кортежа с полями класса из коллекции создавался экземпляр класса через метод `apply`.

| Класс | Результат |
| - | - |
| ` ApplyBenchmark.caseClass ` | ` 19136.554 ± 637.993  ops/s ` |
| ` ApplyBenchmark.caseClassMeta ` | ` 18331.622 ± 510.110  ops/s ` |
| ` ApplyBenchmark.memoisedCaseClass ` | ` 20007.336 ± 276.490  ops/s ` |
| ` ApplyBenchmark.memoisedMeta ` | ` 2560.862 ±  30.396  ops/s ` |
| ` ApplyBenchmark.memoisedMetaWeak ` | ` 3714.472 ±  21.630  ops/s ` |
| ` ApplyBenchmark.optimizeHeapCaseClass ` | ` 19397.820 ± 509.561  ops/s ` |
| ` ApplyBenchmark.optimizeHeapMeta ` | ` 3949.541 ±  54.173  ops/s ` |

Режимы `memoised` и `optimizeHeap` имеют накладные расходы на мемоизацию и упаковку полей, поэтому работают медленее обычного `case class`. `caseClass` показывает ту же скорость, что и `case class`. 

#### copy
Для каждого экземпляра класса в коллекции создавалось 2 копии с другими значениями в одном из двух полей.

| Класс | Результат |
| - | - |
| ` CopyBenchmark.caseClass ` | ` 5080.050 ± 284.304  ops/s ` |
| ` CopyBenchmark.caseClassMeta ` | ` 5432.313 ± 336.334  ops/s ` |
| ` CopyBenchmark.memoisedCaseClass ` | ` 4998.074 ± 173.456  ops/s ` |
| ` CopyBenchmark.memoisedMeta ` | ` 853.461 ±   8.121  ops/s ` |
| ` CopyBenchmark.memoisedMetaWeak ` | ` 1159.225 ±  63.359  ops/s ` |
| ` CopyBenchmark.optimizeHeapCaseClass ` | ` 6899.622 ± 129.428  ops/s ` |
| ` CopyBenchmark.optimizeHeapMeta ` | ` 959.581 ±  11.433  ops/s ` |

Ситуация схожа с предыдущим бенчмарком. Метод `copy` просто создает новый объект через `apply`.

#### Доступ к полям
Читалось каждое поле из экземпляра класса в коллекции.

| Класс | Результат |
| - | - |
| ` FieldAccessBenchmark.caseClass ` | ` 15390.926 ± 122.415  ops/s ` |
| ` FieldAccessBenchmark.caseClassMeta ` | ` 15433.422 ± 254.224  ops/s ` |
| ` FieldAccessBenchmark.optimizeHeapCaseClass ` | ` 22975.564 ± 803.286  ops/s ` |
| ` FieldAccessBenchmark.optimizeHeapMeta ` | ` 4535.168 ±  50.179  ops/s ` |

Режим `caseClass` не отличается от `case class`, метод доступа к полю возвращает фактическое поле класса. Режим `optimizeHeap` производит "распаковку" полей при чтении, поэтому время работы больше. Режим `memoised` не рассматривался, так как в этом контексте он не отличается от `caseClass`.

#### `.hashCode` и `.toString`
Для каждого экземпляра класса вызывались методы `.hashCode`и `.toString`.

| Класс | Результат |
| - | - |
| ` HashCodeBenchmark.caseClass ` | ` 11307.343 ± 1278.621 ops/s ` |
| ` HashCodeBenchmark.caseClassMeta ` | ` 12106.911 ± 263.225 ops/s ` |
| ` HashCodeBenchmark.memoisedCaseClass ` | ` 16266.437 ± 292.010 ops/s ` |
| ` HashCodeBenchmark.memoisedMeta ` | ` 24262.046 ± 1876.971 ops/s ` |
| ` HashCodeBenchmark.optimizeHeapCaseClass ` | ` 4208.655 ± 323.614 ops/s ` |
| ` HashCodeBenchmark.optimizeHeapMeta ` | ` 3078.390 ± 210.247 ops/s ` |

| Класс | Результат |
| - | - |
| ` ToStringBenchmark.caseClass ` | ` 1145.058 ± 34.190 ops/s ` |
| ` ToStringBenchmark.caseClassMeta ` | ` 1296.309 ± 22.632 ops/s ` |
| ` ToStringBenchmark.memoisedCaseClass ` | ` 1439.810 ± 106.486 ops/s ` |
| ` ToStringBenchmark.memoisedMeta ` | ` 26745.501 ± 1026.738 ops/s ` |
| ` ToStringBenchmark.optimizeHeapCaseClass ` | ` 548.055 ± 98.772 ops/s ` |
| ` ToStringBenchmark.optimizeHeapMeta ` | ` 646.945 ± 29.493 ops/s ` |

Для режимов `caseClass` и `optimizeHeap` ситуация повторяет доступ к полям. Методы `.hashCode` и `.toString` обращаются к полям класса для построения хешей и строк, правда делают еще много другой работы, поэтому разница небольшая. Режим `memoised` работает быстрее бейзлайна в виде `case class` из-за особых настроек: `memoisedHashCode` и `memoisedToString`. Они сохраняют значений двух методов и не пересчитывают их много раз.

#### `productElement`
Этот бенчмарк тестировал метод `productElement`.

| Класс | Результат |
| - | - |
| ` ProductElementBenchmark.caseClass ` | ` 13921.991 ± 631.164 ops/s ` |
| ` ProductElementBenchmark.caseClassMeta ` | ` 13871.084 ± 1241.714 ops/s ` |
| ` ProductElementBenchmark.optimizeHeapCaseClass ` | ` 21984.665 ± 636.940 ops/s ` |
| ` ProductElementBenchmark.optimizeHeapMeta ` | ` 4305.256 ± 73.872 ops/s ` |

Тут все одинаково с бенчмарком на доступ полей.

#### Сериализация
Вся коллекция экземпляров класса сериализовалась и затем десериализовалась.

| Класс | Результат |
| - | - |
| ` SerializationBenchmark.caseClass ` | ` 190.118 ± 19.615  ops/s ` |
| ` SerializationBenchmark.caseClassMeta ` | ` 205.555 ± 29.683  ops/s ` |
| ` SerializationBenchmark.memoisedCaseClass ` | ` 250.941 ± 34.648  ops/s ` |
| ` SerializationBenchmark.memoisedMeta ` | ` 316.477 ± 30.002  ops/s ` |
| ` SerializationBenchmark.memoisedMetaWeak ` | ` 338.591 ± 39.945  ops/s ` |
| ` SerializationBenchmark.optimizeHeapCaseClass ` | ` 93.594 ± 18.978  ops/s ` |
| ` SerializationBenchmark.optimizeHeapMeta ` | ` 66.927 ±  8.449  ops/s ` |

Здесь почти нет отличий от бейзлайнов `case class`ов. Сложные операции записи и чтения сериализованных данных затмевают быстрые чтения полей и создания объектов. Хороший вывод -- даже непростая упаковка полей в `optimizeHeap` и мемоизация не влияют на сериализацию объектов.

#### unapply
Над каждым экземпляром класса производился pattern-matching.

| Класс | Результат |
| - | - |
| ` UnapplyBenchmark.caseClass ` | ` 14816.202 ± 939.746  ops/s ` |
| ` UnapplyBenchmark.caseClassMeta ` | ` 13392.518 ± 431.781  ops/s ` |
| ` UnapplyBenchmark.optimizeHeapCaseClass ` | ` 23635.361 ± 238.981  ops/s ` |
| ` UnapplyBenchmark.optimizeHeapMeta ` | ` 4082.947 ±  61.865  ops/s ` |

Режим `optimizeHeap` показывает медленные результаты из-за распаковки полей, режим `caseClass` показывает хорошую производительность относительно `case class`а. 

Следующая группа бенчмарков тестировала особенности режимов генерации: методы для поддержки `Shapeless` и скорость работы `.equals` при мемоизации.

### Shapeless
Настройка `shapeless` генерирует объекты типов `Generic` и `LabelledGeneric`, что позволяло приводить экземпляры класса к типу `Hlist` и обратно. Такие преобразования проводились для всей коллекции.

| Класс | Результат |
| - | - |
| ` ShapelessBenchmark.caseClass ` | ` 8406.639 ± 342.925  ops/s ` |
| ` ShapelessBenchmark.caseClassMeta ` | ` 6653.700 ±  38.209  ops/s ` |

Сгенерированный класс работает медленее. Причины тоже не ясны, возможно это связано с макро-генерацией `Shapeless`. 

#### `.equals` внутри `Vector`
Этот бенчмарк тестировал время работы метода сравнения `.equals` в случае, когда данные помещены внутри `Vector`. Выбиралось множество из 1000000 случайных пар индексов для сравнения. Индексы выбирались на расстоянии не больше 10 и постепенно увеличивались, начиная с 1. Это поддерживает локальности данных, и структура данных `Vector` в состоянии ее обеспечить.

| Класс | Результат |
| - | - |
| ` EqualsVectorBenchmark.caseClass ` | ` 29.456 ± 1.851 ops/s ` |
| ` EqualsVectorBenchmark.caseClassMeta ` | ` 30.252 ± 1.707 ops/s ` |
| ` EqualsVectorBenchmark.memoisedCaseClass ` | ` 36.041 ± 3.207 ops/s ` |
| ` EqualsVectorBenchmark.memoisedMeta ` | ` 36.401 ± 1.061 ops/s ` |
| ` EqualsVectorBenchmark.memoisedWeak ` | ` 36.737 ± 2.372 ops/s ` |
| ` EqualsVectorBenchmark.optimizeHeapCaseClass ` | ` 25.922 ± 1.398 ops/s ` |
| ` EqualsVectorBenchmark.optimizeHeapMeta ` | ` 11.473 ± 0.828 ops/s ` |

Режимы `caseClass` и `memoised` работают так же как `case class`. В первом случае разницы в реализации действительно нет, во втором в сгенерированных классах происходит сравнение *по ссылке*, а не *по значению*. Должно работать быстрее, но нет. Все перекрывает время обращения к элементу `Vector`а. В случае `optimizeHeap` операции распаковки сильно замедляют `.equals`, даже по сравнению со временем доступа к элементам.

#### `.equlas` внутри `HashSet`
`HashSet` использует `.equals` и `.hashCode` для построения хеш-таблицы. Измерялось время, за которое вся коллекция экземпляров класса записывалась в пустой `HashSet`.

| Класс | Результат |
| - | - |
| ` HashSetBenchmark.caseClass ` | ` 3341.499 ± 44.082 ops/s ` |
| ` HashSetBenchmark.caseClassMeta ` | ` 3665.179 ± 77.763 ops/s ` |
| ` HashSetBenchmark.memoisedCaseClass ` | ` 3858.759 ± 49.616 ops/s ` |
| ` HashSetBenchmark.memoisedIntern ` | ` 5009.943 ± 137.088 ops/s ` |
| ` HashSetBenchmark.memoisedMeta ` | ` 6776.364 ± 39.766 ops/s ` |
| ` HashSetBenchmark.memoisedWeak ` | ` 6401.936 ± 139.723 ops/s ` |
| ` HashSetBenchmark.optimizeHeapCaseClass ` | ` 1566.175 ± 70.254 ops/s ` |
| ` HashSetBenchmark.optimizeHeapMeta ` | ` 869.202 ± 22.643 ops/s ` |

Режимы `caseClass` и `optimizeHeap` работают стандартно. Первый не отличается от бейзлайна, второй медленнее. А вот с `memoised` все намного интереснее! Здесь представлен новый класс `memoisedIntern`, который не использует настройку `memoisedHashCode`. Она кеширует `hashCode` и уменьшает время обращения к хеш-коду. Но и без нее сгенерированный класс работает быстрее `case class`а за счет ускоренного `.equals`. Вместе с кешированием хеш-кода скорость работы сильно возрастает.

## Память
Бенчмарки на использование памяти работали просто. Замерялось две характеристики: сколько памяти использовалось во время итерации и сколько использовалось с начала запуска. Вторая характеристика нужна, чтобы проверить сколько порождается данных, которые не может удалить сборщик мусора. Производилось несколько итераций и выводилась динамика использования памяти. 

Всего было 4 бенчмарка: 
- измерение памяти для режима дублирования `case class`
- измерение для упаковки полей
- измерение для мемоизации, когда данные не могут быть удалены сборщиком мусора
- и когда могут

#### `case class`
Сравнивалось сколько места занимает 500 тыс. `case class`ов и сгенерированных классов. Каждый из них состоял из следующих полей: `i: Int, b: Boolean, s: String`.

**`caseClass`**  

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 54659 kb | 54571 kb |
| 1 | 54574 kb | 54499 kb |
| 2 | 54877 kb | 54689 kb |
| 3 | 54801 kb | 54570 kb |
| 4 | 54691 kb | 54595 kb |

**`caseClassMeta`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 55032 kb | 54934 kb |
| 1 | 54740 kb | 54626 kb |
| 2 | 54896 kb | 54835 kb |
| 3 | 54687 kb | 54612 kb |
| 4 | 54920 kb | 54845 kb |

Абсолютно одинаково. 

#### Упаковка полей
Происходило похожее сравнение: класс с упаковкой полей против `case class`а. 
Они содержали поля: `i: Option[Int],
                       s: Option[String],
                       b1: Option[Boolean],
                       b2: Option[Boolean],
                       b3: Option[Boolean],
                       b4: Option[Boolean]`

**`caseClass`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 111830 kb | 111732 kb |
| 1 | 113230 kb | 112418 kb |
| 2 | 112626 kb | 112325 kb |
| 3 | 112698 kb | 112315 kb |
| 4 | 113117 kb | 112715 kb |

**`optimizeHeapMeta`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 40399 kb | 40281 kb |
| 1 | 42455 kb | 39790 kb |
| 2 | 42949 kb | 40088 kb |
| 3 | 42539 kb | 39811 kb |
| 4 | 43195 kb | 40395 kb |

Упаковка полей уменьшает потребление памяти почти в 3 раза. Этот бенчмарк ясно показывает насколько избыточны типы `Option` и `Boolean`.

#### Мемоизация без удаляемых данных
Наконец-то приступим к самым интересным и показательным бенчмаркам на память. В первом из них создаваемые экземпляры классов не могли быть удалены сборщиком мусора. Рассматривалось 4 класса: `case class`, строгая мемоизация, строгая мемоизация с мемоизацией внутренних полей и слабая мемоизация. Они содержали поля: `b: Boolean, s: String` и строковое поле `s` мемоизировалось в 3ем случае. Использовалось два вида данных для создания объектов этих классов: данные с большим количеством повторений и данные с различными элементами. 

##### Данные с повторениями
500 тысяч элементов, строки размером 2 символа из цифр и букв. Всего различных комбинаций таких строк и `Boolean` намного меньше чем размер коллекции, поэтому появляется много дубликатов. 

**`caseClass`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 45174 kb | 50249 kb |
| 1 | 46968 kb | 50343 kb |
| 2 | 47010 kb | 50478 kb |
| 3 | 46936 kb | 50540 kb |
| 4 | 46980 kb | 50646 kb |

**`memoisedMeta`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 11831 kb | 11537 kb |
| 1 | 11781 kb | 11570 kb |
| 2 | 11770 kb | 11601 kb |
| 3 | 11729 kb | 11601 kb |
| 4 | 11729 kb | 11601 kb |

**`memoisedMeta` c мемоизацией строк**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 11733 kb | 11732 kb |
| 1 | 11718 kb | 11732 kb |
| 2 | 11729 kb | 11742 kb |
| 3 | 11718 kb | 11742 kb |
| 4 | 11728 kb | 11753 kb |

**`memoisedWeak`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 11666 kb | 11659 kb |
| 1 | 11684 kb | 11680 kb |
| 2 | 11663 kb | 11680 kb |
| 3 | 11662 kb | 11680 kb |
| 4 | 11683 kb | 11700 kb |

Мемоизация делает свое дело. Все повторения данных складываются в кеш и не дублируются как в случае `case class`а. 

##### Данные без повторений
Также 500 тыс. элементов, но строки теперь имеют длину 5. Теперь данные не повторяются. 

**`caseClass`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 54937 kb | 54938 kb |
| 1 | 53696 kb | 53724 kb |
| 2 | 54271 kb | 53308 kb |
| 3 | 54590 kb | 53211 kb |
| 4 | 54667 kb | 53191 kb |

**`memoisedMeta`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 78619 kb | 77984 kb |
| 1 | 74883 kb | 140421 kb |
| 2 | 82160 kb | 210863 kb |
| 3 | 75171 kb | 273346 kb |
| 4 | 74882 kb | 336093 kb |

**`memoisedMeta` c мемоизацией строк**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 95091 kb | 94141 kb |
| 1 | 86867 kb | 168425 kb |
| 2 | 102367 kb | 259074 kb |
| 3 | 86810 kb | 333071 kb |
| 4 | 86753 kb | 407689 kb |

**`memoisedWeak`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 105562 kb | 105562 kb |
| 1 | 66527 kb | 105683 kb |
| 2 | 66392 kb | 105669 kb |
| 3 | 66423 kb | 105686 kb |
| 4 | 66406 kb | 105686 kb |

Здесь выигрывают `case class`ы. Строгая мемоизация постоянно записывает новые данные в кеш, потребляя большое количество памяти. Мемоизация строк только ухудшает положение. Слабая мемоизация лучше использует память, но имеет накладные расходы на работу с кешом. 

#### Мемоизация со сборщиком мусора
В этом бенчмарке после создания 500 тысяч объектов ссылки оставались только на 20 тыс. Это позволяло сборщику мусора сделать свою ~~грязную~~ работу. 

##### Данные с повторениями

**`caseClass`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 1914 kb | 1923 kb |
| 1 | 1871 kb | 1920 kb |
| 2 | 1852 kb | 1897 kb |
| 3 | 1856 kb | 1879 kb |
| 4 | 1859 kb | 1863 kb |

**`memoisedMeta`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 1662 kb | 1663 kb |
| 1 | 445 kb | 1376 kb |
| 2 | 459 kb | 1367 kb |
| 3 | 448 kb | 1347 kb |
| 4 | 468 kb | 1347 kb |

**`memoisedMeta` c мемоизацией строк**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 1347 kb | 1348 kb |
| 1 | 473 kb | 1351 kb |
| 2 | 458 kb | 1341 kb |
| 3 | 458 kb | 1331 kb |
| 4 | 468 kb | 1331 kb |

**`memoisedWeak`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 1583 kb | 1583 kb |
| 1 | 900 kb | 1521 kb |
| 2 | 886 kb | 1496 kb |
| 3 | 906 kb | 1494 kb |
| 4 | 909 kb | 1498 kb |

Снова все различные комбинации данных можно полностью записать в кеш. Строгая мемоизация дает хороший выигрыш по сравнению с `case class`ом, слабая чуть хуже. 

##### Данные без повторений

**`caseClass`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 2299 kb | 2299 kb |
| 1 | 2228 kb | 2341 kb |
| 2 | 2187 kb | 2341 kb |
| 3 | 2207 kb | 2341 kb |
| 4 | 2187 kb | 2341 kb |

**`memoisedMeta`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 66468 kb | 66468 kb |
| 1 | 66920 kb | 132920 kb |
| 2 | 62015 kb | 193819 kb |
| 3 | 70686 kb | 264037 kb |
| 4 | 62875 kb | 326444 kb |

**`memoisedMeta` c мемоизацией строк**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 82753 kb | 82754 kb |
| 1 | 81617 kb | 163902 kb |
| 2 | 75799 kb | 239233 kb |
| 3 | 91769 kb | 330278 kb |
| 4 | 75296 kb | 403948 kb |

**`memoisedWeak`**

| Итерация | Память на итерацию | Память после итерации |
| - | - | - |
| 0 | 41861 kb | 41861 kb |
| 1 | 3089 kb | 42294 kb |
| 2 | 2744 kb | 42382 kb |
| 3 | 2676 kb | 42402 kb |
| 4 | 2676 kb | 42423 kb |

Тут уже все намного хуже. Строгая мемоизация хранит все данные в кеше, ничего не удаляется, и кеш становится огромным. Слабая хранит в кеше только то, что нужно на итерации, но все равно получается много. Это худший случай, чтобы использовать мемоизацию. Он запросто может привести к `OutOfMemoryError`. 

### *TL;DR* 
Измерения скорости работы и потребляемой памяти показывают, что нет ситуации, когда сгенерированный класс однозначно лучше чем `case class`. Либо идет выигрыш в скорости, но при большем использовании памяти, либо наоборот. Сгенерированные классы, если нет дополнительных режимов генерации, дублируют эффективность `case class`ов. Упаковка полей потребляет меньше памяти, но ведет себя медленно при обращении к полям класса и в методе `.apply`. Мемоизация ускоряет метод `.equals`, но так же имеет накладные расходы в `.apply`. Потребление памяти зависит от контекста. При большом количестве дубликатов мемоизация кеширует их и хранит только ссылки. При данных почти без дубликатов, особенно в случае, когда данные удаляются сборщиком мусора, потребление памяти существенно увеличивается. 

## В итоге 
Вывод будет простой. Встроенные `case class`ы работают очень хорошо в большинстве случаев. Stalagmite предоставляет альтернативу им, обменивая память на скорость работы и наоборот. В планах разработки есть идеи как сократить повторения кода (boilerplate) и добавить новых оптимизаций, но полностью заменить `case class` нельзя. Поэтому используйте эту библиотеку с умом. Надеюсь я показал вам сильные и слабые стороны текущей реализации :)