---
layout: post
title: "Type classes in Scala"
---

Type class -- паттерн, пришедший из Haskell. Тype class в Scala -- это трейт с как минимум одним параметром типа. 
Этот параметр определяет конкретные типы, для которых определены type class instances.

Type classes позволяют расширить функционал существующих классов без использования наследования и не изменяя их код непосредственно.

А что плохого в наследовании? Например, мы определили некий трейт `Greeting`. Для каждого класса, 
реализующего данное поведение, придется унаследоваться от трейта:
```
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
```
object Greeter {
  def greet(value: Any): String = value match {
    case Person(name, _) => s"$name says: hello"
    case Cat(name) => s"$name says: meow"
    case _ => ???
  }
}
```
Тут тоже есть проблемы: во-первых, теряем в типобезопасности, т. к. не можем определить хороший супертип для всех 
возможных значений и придется использовать что-то вроде Any; во-вторых, как и в предыдущем примере, невозможно 
реализовать разные варианты поведения для одного и того же типа.

Мы можем решить многие из перечисленных выше проблем следующим образом:
```
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
По сути трейт `Greeter` это type class, а `PersonGreeter` и `CatGreeter` -- type class instances.

![_config.yml]({{ site.baseurl }}/images/typeclasses/speechless.jpg)

Для классической реализации паттерна type class в Scala не хватает лишь одной важной детали –- implicits. Имплиситы 
позволяют использовать type classes с меньшим количеством boilerplate кода.

### Type class interfaces

Type class interfaces -- это обощенные методы, которые принимают type class instances в качестве неявных (implicit) 
параметров. Это можно реализовать с помощью interface objects и interface syntax:

####Interface object:

Чтобы использовать такой объект, нужно импортировать нужный type class instance и вызвать соответствующий метод.
Важно: type class instance должен быть объявлен как implicit.

```
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

Часто бывает удобнее использовать методы нужного type class instance непосредственно, не оборачивая каждый из них. Это 
достигается с помощью метода apply без параметров:
```
object GreetUtil {
  def apply[A](implicit greeter: Greeter[A]): Greeter[A] =
    greeter
}
GreetUtil[Cat].greet(Cat("Simon"))
```

####Interface syntax

Другой способ использования type classes -- interface syntax. Он так же называется extension methods, type enrichment 
или pimp-my-library pattern. Смысл в том, чтобы определить неявный класс-обертку над исходным классом с нужными 
методами, и 
импортировать этот класс в область видимости. Далее можно вызывать эти дополнительные методы у исходного 
класса без дополнительных усилий, как будто это его собственные методы. Всю необходимую работу
сделает компилятор: обнаружив вызов метода, не определенного для класса, компилятор попробует найти в implicit scope 
класс, у которого есть искомый метод и который может быть сконструирован из значения исходного класса.

```
object GreeterSyntax {
  implicit class GreeterOps[A](value: A) {
    def greet(implicit g: Greeter[A]): String =
      g.greet(value)
  }
}

import GreeterSyntax._
Cat("Simon").greet
```

###Context Bounds и implicitly

Часто, мы не используем type class instances непосредственно в теле метода, однако должны иметь их в области 
видимости, чтобы использовать в другом методе. В таких случаях для краткости используют context bounds.
Вместо:
```
def greet[A](someone: A)(implicit greeter: Greeter[A]): String = ???
```
Пишем:
```
def greet[A: Greeter](someone: A): String = ???
```
Если все же нам необходимо обратиться к type class instance, можно использовать метод из стандарной библиотеки Scala 
-- implicitly:
```
def implicitly[A](implicit value: A): A =
  value
```
Назначение implicitly довольно просто -- он призывает (summons) значение указанного типа из implicit scope.
```
implicitly[Greeter[Cat]].greet(new Cat("Simon"))
```