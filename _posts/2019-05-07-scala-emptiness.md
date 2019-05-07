---
layout: post
title: "Scala: Null, null, Nil, Nothing, None and Unit"
---

Scala включает в себя несколько понятий для обозначения отсутствия значения. Каждое из них используется в строго определенных ситуациях, потому хорошо бы знать, что же каждое их этих понятий означает и где применяется.

![_config.yml]({{ site.baseurl }}/images/confused-cat.jpg)

### Null and null

`Null` это трейт, у которого существует единственный экземпляр -- `null`.
`Null` является потомком всех ссылочных классов (`AnyRef`), но не примитивных (`AnyVal`).

null служит той же цели, что в Java, то есть представляет собой значение ссылочного типа, которое не указывает ни на какой объект. Ссылочные типы могут иметь значение `null`, а примитивные (например, `Int`) не могут:

```scala
scala> val n: Int = null
<console>:11: error: an expression of type Null is
ineligible for implicit conversion
       val n: Int = null
```

Если метод принимает параметр типа `Null`, у нас два варианта: передать ему непосредственно `null` или ссылку типа `Null`.

```scala
scala> def f(x: Null): Unit = println("Ok")
f: (x: Null)Unit

scala> f(null)
Ok

scala> val nullRef: Null = null
nullRef: Null = null

scala> f(nullRef)
Ok
```

При этом, если попробовать передать нулевую ссылку на `String`, это не пройдет проверку типов, получим ошибку компиляции:

```scala
scala> val stringRef: String = null
stringRef: String = null

scala> f(stringRef)
<console>:14: error: type mismatch;
 found   : String
 required: Null
       f(stringRef)
         ^
```

### Nil

`Nil` это объект, который расширяет (extends) `List[Nothing]`. То есть это пустой список нулевой длины, которые по сути не означает "пустоту" или "отсутствие значения", а является объектом, списком, просто без содержимого.

```scala
scala> Nil
res1: scala.collection.immutable.Nil.type = List()

scala> Nil.length
res2: Int = 0

scala> "ABC" :: Nil
res3: List[String] = List(ABC)

scala> Nil :: Nil
res4: List[scala.collection.immutable.Nil.type] = List(List())
```

### Nothing

`Nothing` это еще один трейт, он расширяет класс `Any`(все классы в Scala наследуются от `Any`), таким образом `Nothing` подтип *любого* класса в Scala. Важная особенность `Nothing` -- не существует экземпляров этого класса.

Возвращаясь к `Nil`, так как это `List[Nothing]`, а `Nothing` подтип любого класса, можно использовать `Nil` как пустой тип списка `String`, `Int` или любого другого класса. Удобно.

```scala
scala> val emptyStringList: List[String] = List[Nothing]()
emptyStringList: List[String] = List()

scala> val emptyIntList: List[Int] = List[Nothing]()
emptyIntList: List[Int] = List()

val emptyIntList: List[Int] = Nil
emptyIntList: List[Int] = List()
```

Еще одно важное применение `Nothing` -- в качестве типа возвращаемого значения методов, которые никогда не возвращают результат. Если это обдумать, выходит очень логично: Если тип возвращаемого значения метода `Nothing`, и не существует экземпляров типа `Nothing`, то выходит, метод никогда не возвращает результат.

Пример таких методов: методы, которые всегда выбрасывают исключения.


### None

Бывают ситуации, когда метод не может вернуть полезный результат. Но выбрасывать исключение или возвращать `null` -- не лучшие решения. Для этого в Scala существует класс `Option`. У него ровно два потомка: класс `Some[T]` и объект `None`.

Цель класса `Option` -- дать знать пользователям метода, что он может вернуть `T` в форме `Some[T]` или `None`, чтобы обозначить отсутствие результата.

```scala
scala> def getOptString(n: Int): Option[String] = if (n > 0) Some("A positive number!") else None
getOptString: (n: Int)Option[String]

scala> getOptString(1)
res9: Option[String] = Some(A positive number!)

scala> getOptString(-1)
res10: Option[String] = None

scala> def printOptString(optStr: Option[String]) = optStr match {
     | case Some (str) => println(str)
     | case None => println("none")
     | }
```

### Unit

`Unit` -- аналог `void` в Java, используется для обозначения типа результата методов, которые не возвращают значений. 

Пустые круглые скобки -- значение типа `Unit`.

```scala
scala> def iAmVoid(x: String): Unit = println(x)
iAmVoid: (x: String)Unit

scala> val x: Unit = ()
x: Unit = ()
```
