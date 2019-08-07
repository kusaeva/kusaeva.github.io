---
layout: post
title: "Type classes in Scala"
---

Тайп класс -- паттерн, пришедший из Haskell.
В Scala тайп класс это трейт с хотя бы одним параметром типа. 
Этот параметр определяет конкретные типы, для которых определены экземпляры тайп класса.

Тайп классы позволяют расширить функционал существующих классов без использования наследования и не изменяя их код непосредственно.

А что плохого в наследовании? Например, мы определили некий трейт `Greeting`. Для каждого класса, 
реализующего данное поведение, придется унаследоваться от трейта:
```scala
trait Greeting {
  def greet: String
}

case class Person(name: String) extends Greeting[Person] {
  def greet: String = s"$name says: hello"
}

case class Cat(name: String) extends Greeting[Cat] {
  def greet: String = s"$name says: meow"
}
```
Если необходимо добавить поведение только собственным классам, такой способ вполне имеет правно на существование, но 
как быть, если 
нужно расширить функциональность класса, написанного не нами?

Можно попробовать написать так:
```scala
object Greeter {
  def greet(value: Any): String = value match {
    case Person(name, _) => s"$name says: hello"
    case Cat(name) => s"$name says: meow"
    case _ => ???
  }
}
```
Тут тоже есть проблемы: во-первых, теряем в типобезопасности, т. к. не можем определить хороший супертип для всех 
возможных значений и придется использовать что-то вроде `Any`; во-вторых, как и в предыдущем примере, невозможно 
реализовать разные варианты поведения для одного и того же типа.

Мы можем решить многие из перечисленных выше проблем следующим образом:
```scala
trait Greeter[A] {
  def greet(in: A): String
}

object PersonGreeter extends Greeter[Person] {
  def greet(person: Person) = s"${person.name} says: hello"
}

object CatGreeter extends Greeter[Cat] {
  def greet(cat: Cat) = s"${cat.name} says: meow"
}
```
Этот способ позволяет добавить нужное поведение чужим классам, а также различные варианты поведения для одного и того
 же класса.
По сути трейт `Greeter` это тайп класс, а `PersonGreeter` и `CatGreeter` -- экземпляры тайп класса.

![_config.yml]({{ site.baseurl }}/images/typeclasses/speechless.jpg)

Для классической реализации паттерна type class в Scala не хватает лишь одной важной детали –- implicits. Имплиситы 
позволяют использовать тайп классы с меньшим количеством boilerplate кода.

## Type class interfaces

Type class interfaces -- это обощенные методы, которые принимают экземпляры тайп классов в качестве неявных (implicit) 
параметров. Это можно реализовать с помощью interface objects и interface syntax:

### Interface object:

Чтобы использовать такой объект, нужно импортировать нужный экземпляр тайп класса и вызвать соответствующий метод.
Важно: экземпляр тайп класса должен быть объявлен как implicit.

```scala
object GreetUtil {
  def greet[A](someone: A)(implicit greeter: Greeter[A]): String =
    greeter.greet(someone)
}

object GreeterInstances {
  implicit val catGreeter: Greeter[Cat] = new Greeter[Cat] {
    def greet(cat: Cat): String = s"${cat.name} says: meow" 
  }
}

import GreeterInstances._
GreetUtil.greet(Cat("Simon"))
```

При вызове метода с неявными параметрами, компилятор попробует найти недостающие значения в implicit scope.
Implicit scope состоит из значений соответствующего типа и отмеченных как неявные из локальной области видимости, 
импортов и объектов-компаньонов всех типов, упомянутых в вызове метода (в данном примере, `Cat` и `Greeter`).
Если существует несколько инстансов максимального приоритета или же ни одного не найдено -- будет ошибка компиляции.

Часто бывает удобнее использовать методы нужного экземпляра тайп класса непосредственно, не оборачивая каждый из них. Это 
достигается с помощью метода apply без параметров:
```scala
object GreetUtil {
  def apply[A](implicit greeter: Greeter[A]): Greeter[A] =
    greeter
}
GreetUtil[Cat].greet(Cat("Simon"))
```

### Interface syntax

Другой способ использования тайп классов -- interface syntax (extension methods, type enrichment, pimp-my-library pattern). Смысл в том, чтобы определить неявный класс-обертку над исходным классом с нужными методами, и импортировать этот класс в область видимости. Далее можно вызывать эти дополнительные методы у исходного 
класса без дополнительных усилий, как будто это его собственные методы. Всю необходимую работу
сделает компилятор: обнаружив вызов метода, не определенного для класса, компилятор попробует найти в implicit scope 
класс, у которого есть искомый метод и который может быть сконструирован из значения исходного класса.

```scala
object GreeterSyntax {
  implicit class GreeterOps[A](value: A) {
    def greet(implicit g: Greeter[A]): String =
      g.greet(value)
  }
}

import GreeterSyntax._
Cat("Simon").greet
```

## Context Bounds, implicitly

Часто, мы не используем экземпляры тайп классов непосредственно в теле метода, однако должны иметь их в области 
видимости, чтобы использовать в другом методе. В таких случаях для краткости используют context bounds.
Вместо:
```scala
def greet[A](someone: A)(implicit greeter: Greeter[A]): String = ???
```
Пишем:
```scala
def greet[A: Greeter](someone: A): String = ???
```
Если все же нам необходимо обратиться к экземпляру тайп класса, можно использовать метод из стандарной библиотеки Scala 
-- implicitly:
```scala
def implicitly[A](implicit value: A): A =
  value
```
Назначение implicitly довольно просто -- он призывает (summons) значение указанного типа из implicit scope.
```scala
implicitly[Greeter[Cat]].greet(new Cat("Simon"))
```
