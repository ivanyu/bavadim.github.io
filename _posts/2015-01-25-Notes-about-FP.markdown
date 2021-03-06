---
layout: post
title: "Заметки о функциональном программировании. Наследование и полиморфизм."
comments: true
categories: programming
---

Ранее мы разобрались [что такое](http://bavadim.me/programming/2015/01/11/Notes-about-FP.html) функциональное программирование и даже смогли организовать [инкапсуляцию](http://bavadim.me/programming/2015/01/19/Notes-about-FP.html), в том числе и при помощи понятий класса и объекта. В этот раз мы поговорим о наследовании, полиморфизме, продолжим разговор об инкапсуляции и бегло взглянем на иерархию типов и обработку ошибок в Scala. Пост получился больше про ООП и Scala, чем про функциональное программирование, но надеюсь, вы сможете найти в нем что-то полезное для себя.

## Наследование

Можно сказать что класс - это описание множества объектов, которые можно использовать одним и тем же способом.

**Наследование** - это синтаксическая конструкция, позволяющая создать новый класс, дополняющий (правильнее *расширяющий*) другой класс новым интерфейсом и реализацией. 

Можно сказать, что наследование позволяет определить подмножество объектов класса (*базового класса*) путем создания нового класса (*дочернего класса* или *подкласса*). Из определения наследования следует, что подкласс может быть использован везде, где предполагается использование базового класса, другими словами подкласс для внешнего наблюдателя является *базовым классом*. Обратное утверждение не верно. Наследуя классы друг от друга мы можем строить иерархии классов любой глубины.

Рассмотрим наследование на примере множества. Множество, как в C++ или Java, построим на двоичном дереве, правда не будем забивать себе голову мелочами типа балансировки. Наше множество будет поддерживать две операции:

- `incl` - операцию добавления целого числа в множество
- `contains` - операцию проверки вхождения числа в множество 

Состоять множество будет из элементов двух типов: 

- пустого поддерева типа `Empty`
- поддерева целых чисел типа `NonEmpty` 

Дерево `NonEmpty` состоит из целого числа-корня, левого поддерева и правого поддерева. Левое поддерево может быть `Empty` или `NonEmpty`, корень которого *меньше* корня дерева. Правое поддерево определяется также, только корень поддерева типа `NonEmpty` должен быть *больше* корня дерева. На рисунке представлено множество чисел 2, 3, 5, 8, 10, 11:

![bintree](/images/3/bintree.png)

Понятно, что классы `Empty` и `NonEmpty` сами являются множествами, точнее являются подклассами класса множества, который мы назовем `IntSet`. Начнем писать код. Для начала определим интерфейс множества `IntSet` в виде *абстрактного класса*:

{% highlight scala %}

abstract class IntSet {
	def incl(x: Int): IntSet
	def contains(x: Int): Boolean
}
	
{% endhighlight %}

**Абстрактный класс** - это класс, в котором могут присутствовать методы без реализации. Другими словами абстрактный класс определяет множество объектов с определенным интерфейсом но не обязательно определяет реализацию интерфейса. Абстрактный класс предназначен только для наследования и создать объект абстрактного класса нельзя. 

В реализации интерфейса `IntSet` *объявлены* общие для всех множеств методы `incl` и `contains`. *Определены* эти методы в каждом подклассе по-своему:

{% highlight scala %}

// extends говорит, что NonEmpty подкласс IntSet
class NonEmpty(elem: Int, left: IntSet, right: IntSet) extends IntSet {  		
	def contains(x: Int): Boolean =
		if (x < elem) left contains x 	// `.` и `()` можно опускать, 
						//если результат однозначен
		else if (x > elem) right contains x
		else true
			
	def incl(x: Int): IntSet = 
		if (x < elem) new NonEmpty(elem, left incl x, right)
		else if (x > elem) new NonEmpty(elem, left, right incl x)
		else this
			
	override def toString = "."		//override обязателен при 
						//переопределении метода.
						//Необходимость override
						//позволяет
						//избежать ошибок
} 
	
class Empty extends IntSet {
	def contains(x: Int): Boolean = false
	def incl(x: Int): IntSet  = new NonEmpty(x, new Empty, new Empty) 
		
	override def toString = "{" + left + elem + right + "}"
}

{% endhighlight %}

В Scala большинство классов унаследовано от системного класса `Object`. Новые классы неявно наследуются от этого класса. В `Object` определен ряд методов, в том числе метод `toString`. По-умолчанию этот метод возвращает строковый идентификатор объекта в JVM, например `java.lang.Object@21ba5846`. В нашем случае `IntSet` - подкласс `Object`. Так как `Empty` и `NonEmpty` - подклассы `IntSet`, то они также подклассы `Object`. Методы `toString` классов `Empty` и `NonEmpty` *переопределяют* метод `toString` класса `Object`. Возникает следующий вопрос: как выбрать правильную реализацию `toString`? В большинстве объектно-ориентированных языков для решения этой задачи используется механизм *динамического связывания*. Механизм динамического связывания работает следующим образом: во время выполнения программы, в момент вызова метода происходит поиск его реализации. Поиск происходит сначала в классе объекта и если реализация не найдена поиск продолжается в классах, располагающихся выше по иерархии, вплоть до `Object`. Таким образом для `new Empty toString`будет вызвана реализация `toString` из класса `Empty`, а для `new NonEmpty(1, Empty, Empty) toString` будет вызвана реализация из класса `NonEmpty`. Если же мы вызовем у объекта класса `Empty` метод `hashCode`, определенный в `Object`, то динамическое связывание вызовет реализацию из класса `Object`, так как в `Empty` и `IntSet` реализации этого метода нет. 

Давайте теперь разберемся как динамическое связывание влияет на нашу модель вычислений:

	new NonEmpty(1, new Empty, new Empty).contains(1) ->
	[1/elem][new Empty/left][new Empty/right][1/x][new NonEmpty(1, new Empty, new Empty)/this]
	-> 	if (1 < 1) new Empty.contains(1)
		else if (1 > 1) new Empty.contains(1)
		else true
	-> true

Видим, что для того, чтобы в ней работал механизм наследования достаточно оговорить правила работы оператора `.`. Эти правила задает алгоритм работы динамического связывания. 

С наследованием разобрались. Давайте теперь посмотрим на код множества еще раз. Понятно, что все объекты `Empty` абсолютно одинаковы и мы воспринимаем их как один и то же объект. Тем не менее мы создаем каждый объект оператором `new` а компьютер, что еще хуже, считает эти объекты различными и выделяет под каждый из них память. Для решения этой проблемы мы можем создать объект и использовать его везде, где он необходим. Такие объекты называются *одиночными объектами* (singleton object). В Scala существует специальный синтаксис для работы с одиночными объектами:

{% highlight scala %}

object Empty extends IntSet {
	def contains(x: Int): Boolean = false
	def incl(x: Int): IntSet  = new NonEmpty(x, new Empty, new Empty) 
		
	override def toString = "{" + left + elem + right + "}"
}
	
{% endhighlight %}

Объект приложения Scala еще один пример одиночного объекта. Классическая программа Hello world на Scala выглядит так:

{% highlight scala %}

object Hello {
	def main(args: Array[String]) = println("Hello world")
}

{% endhighlight %}

Теперь мы должны ссылаться на одиночный объект по имени, например `Empty incl 1`. В теории `Empty` является примитивным значением в нашей модели подстановки и не отличается от `new Empty()`. Но, практически, теперь будет выделено меньше памяти и в коде будет подчеркнута единственность объекта.

В одиночных объектах можно определять общие для всех объектов класса методы, не зависящие от конкретного объекта. Такие методы в других языках часто называются *статическими методами* или *методами класса*. Имена объектов и классов хранятся в различных пространствах имен, поэтому объекты и классы могут иметь одно и то же имя. Давайте определим метод `singletonSet`, создающий пустое множество:

{% highlight scala %}

object IntSet {
        def singleton(first: Int): IntSet = Empty incl first
}

{% endhighlight %}

Для создания множества из одного элемента 1 достаточно написать `IntSet singleton 1`

## Множественное наследование

Множественное наследование позволяет выполнить наследование интерфейса и реализации от нескольких базовых классов. Множественное наследование интерфейса позволяет использовать объект сразу в нескольких ролях. Множественное наследование реализации позволяет использовать реализацию интерфейсов из различных классов и часто используется для создания *классов-примесей*. Класс-примесь реализует независимую атомарную функциональность, например функции отрисовки объекта на экране или функции сохранения объекта в БД. Мы можем добавлять в произвольный класс нужные функции просто наследуясь от соответствующих классов-примесей. 

В Scala множественное наследование реализуется с применением трэйтов (не знаю как эта концепция называется по-русски). Трэйты аналогичны абстрактному классу, но не имеют аргументов и у класса может быть несколько базовых классов-трэйтов:

{% highlight scala %}

trait Planar {
	def height: Int
	def width: Int
	def surface = height * width
}

...
	
class Square extends Shape with Planar with Movable ...

{% endhighlight %}
	
Множественное наследование ведет к ряду проблем, самая серьезная из которых называется *ромбическое наследование*. Суть проблемы продемонстрирована на рисунке:

![множественное наследование](/images/3/sq_inh.png)

`глючный_метод` реализован в двух базовых классах и возникает неоднозначность при использовании динамического связывания. Проблема ромбического наследования решается в разных языках по-разному. В Java запрещено множественное наследование реализации (до версии 1.8), в Ruby множественное наследование ограниченно [реализовано через модули](http://rubydev.ru/2010/08/ruby_module/), а в С++ традиционно реализовано [через задницу](https://ru.wikipedia.org/wiki/%D0%92%D0%B8%D1%80%D1%82%D1%83%D0%B0%D0%BB%D1%8C%D0%BD%D0%BE%D0%B5_%D0%BD%D0%B0%D1%81%D0%BB%D0%B5%D0%B4%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5). Scala решает проблему самым очевидным путем. В этом языке определены явные правила разрешения конфликтов. Про эти правила я расскажу как-нибудь в другой раз.


## Полиморфизм

На примере множеств мы видели, что подклассы можно использовать везде, где предполагается использование базового класса. Метод, имеющий аргумент определенного класса, может работать со всеми подклассами этого класса. Это проявление свойства функций, называемого **полиморфизмом**.

**Полиморфизм** - это свойство функции, заключающееся в возможности работы с данными разных типов.

Существует [несколько типов](http://en.wikipedia.org/wiki/Polymorphism_(computer_science)) полиморфизма. Тут мы рассмотрим два:

- параметрический полиморфизм
- полиморфизм включения

С полиморфизм включения мы сталкивались ранее. Он позволяет методу принимать в качестве аргумента объект, тип которого является подклассом класса формального аргумента. С параметрическим полиморфизмом мы познакомимся на примере фундаментальной структуры данных - связного списка.

Связный список состоит из двух типов элементов:

- Nil - пустой список
- Cons - ячейка, включающая элемент-голову и список-хвост

Например `List(1, 2, 3)` будет выглядеть так:

![linked_list](/images/3/linked_list.png)

А список списков `List(List(true, false), List(3))` так:

![complex_linked_list](/images/3/complex_linked_list.png)

Давайте обобщим все сказанное выше в виде кода списка, хранящего целые числа:

{% highlight scala %}

trait IntList {
	def isEmpty: Boolean
	def head: Int
	def tail: IntList
}
	
// val позволяет прочитать значения аргументов класса
// через одноименные методы
class Cons(val head: Int, val tail: IntList) extends IntList {
	def isEmpty = false
}
	
class Nil extends IntList {
	def isEmpty: Boolean = true
	def head: Nothing = throw new NoSuchElementException("Nil.head")
	def tail: Nothing = throw new NoSuchElementException("Nil.tail")
}

{% endhighlight %}

В примере методы `head` и `tail` класса `Nil` выглядят необычно. Во-первых их реализации демонстрируют как бросать [исключения](https://ru.wikipedia.org/wiki/%D0%9E%D0%B1%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B0_%D0%B8%D1%81%D0%BA%D0%BB%D1%8E%D1%87%D0%B5%D0%BD%D0%B8%D0%B9) в Scala. Во-вторых они возвращают не тот же тип, что методы базового класса. `Nothing` - является подклассом всех возможных классов (в следующем разделе поговорим об этом). Поддержка полиморфизма в Scala распространяется и на возвращаемые значения. Тип возвращаемого значения может быть подклассом типа метода в базовом классе. Поэтому  `head` и `tail` класса `Nil` вполне законно переопределяют методы из `IntList`

Теперь давайте подумаем, как научить наш список работать с логическими значениями. Можно просто поменять `Int` на `Boolean`, алгоритмы методов от этого не изменятся! В такой ситуации лучшим решением будет метод обобщенный для всех возможных типов. Такой метод будет обладать параметрическим полиморфизмом. Scala поддерживает параметрический полиморфизм с помощью специальной синтаксической конструкции:

{% highlight scala %}

def id[T](elem: T): T = elem

{% endhighlight %}
	
В этом примере описан метод, который может быть применен к аргументу любого типа и тип возвращаемого значения аналогичен типу аргумента. Использовать этот метод можно так:

{% highlight scala %}

id[Int](1) // -> 1
id[Boolean](true) // -> true

{% endhighlight %}
	
Scala похожим образом позволяет определить полиморфный класс (*дженерик*):

{% highlight scala %}

trait List[T] {
	def isEmpty: Boolean
	def head: T
	def tail: List[T]
}
	
// val позволяет прочитать значения аргументов класса
// через одноименные методы
class Cons[T](val head: T, val tail: List[T]) extends List[T] {
	def isEmpty = false
}
	
class Nil[T] extends List[T] {
	def isEmpty: Boolean = true
	def head: Nothing = throw new NoSuchElementException("Nil.head") //Генерируется ошибка
	def tail: Nothing = throw new NoSuchElementException("Nil.tail") //Про тип Nothing смотрите ниже
}

{% endhighlight %}

Осталось выяснить, как параметрический полиморфизм влияет на модель вычислений. Ответ - никак. Во время компиляции происходит генерация соответствующих функций и классов. В момент выполнения мы имеем программу без параметрического полиморфизма. Такой механизм работы с полиморфизмом называется *стиранием типов*. Стирание типов применяется, например, в Java, Scala, Haskell, ML, OCamel. Но есть языки сохраняющие *параметр типа* до момента выполнения программы. Это, например, языки С++, С#, F#.

Пока с полиморфизмом закончим. В других постах я еще вернусь к параметрическому полиморфизму.

## Иерархия типов Scala

Закончим разговор еще одним примером. На этот раз мы рассмотрим иерархию типов Scala, но пред этим нам придется разобраться с понятием *пакета*.

Пакет схож с классом в смысле инкапсуляции и используется для разбиения программы на модули. В следующем примере все объявления в файле будут находиться в пакете `ExamplePackage`:

{% highlight scala %}

package ExamplePackage
	
object HelloWorld { ... }
class OtherClass { ... }

{% endhighlight %}

*Полное имя* объекта `HelloWorld` будет `ExamplePackage.HelloWorld`. Командой `import` мы можем *импортировать* в текущую *область видимости* объявления из класса, объекта или пакета:

{% highlight scala %}

import ExamplePackage.HelloWorld //импорт в область видимости
HelloWorld // использование. Вместо ExamplePackage.HelloWorld
	
{% endhighlight %}

Импортировать сразу все объявления можно командой `import ExamplePackage._`, а несколько объявлений - `import ExamplePackage.{HelloWorld, OtherClass}`. 

В любую программу на Scala автоматически импортируются объявления из следующих областей видимости:

- все объявления из пакета `scala`
- все объявления из пакета `java.lang`
- все объявления из одиночки `scala.Predef`

Ранее виденные нами функции и типы, такие как `Object`, `Int`, `Boolean`, `assert`, `require`, импортируются именно из этих областей видимости.

Теперь рассмотрим иерархию типов Scala.

![scala_hierarchy](/images/3/scalaHierarchy.png)

На схеме изображены основные типы Scala, отношения наследования и возможности приведения одного типа к другому.

На вершине иерархии лежат следующие классы:

- класс `Any`, содержит методы `==`, `!=`, `equals`, `hashCode`
- класс `AnyRef`, класс ссылочных типов, псевдоним `java.lang.Object`
- базовый класс для объектов-значений `AnyVal`

Классы чисел, символов, логических значений и подобные им примитивные *объекты-значения* являются подклассами `AnyVal`. Сложные *ссылочные* типы, в том числе и подавляющее большинство пользовательских классов, являются подклассами `AnyRef`. `AnyRef` на самом деле псевдоним для `Object`.

В рамках нашей теоретической модели вычислений ссылочные типы ничем не отличаются от объектов-значений. На практике же под объекты-значения не выделяется память во время выполнения программы и они [ограничены в возможностях](http://docs.scala-lang.org/overviews/core/value-classes.html). В императивных программах различия между этими типами принципиальные, но сейчас я останавливаться на этом не буду.

Внизу иерархии лежат два особенных класса: `scala.Null` и `scala.Nothing`. `scala.Nothing` является абстрактным подклассом любого другого класса. Класс не имеет ни одного объекта и может использоваться, например, в следующих случаях:

- для обозначения случаев, когда выражение не возвращает значения, например в случае исключительной ситуации
- для создания коллекций без элементов (например List[Nothing])

Давайте сейчас подробнее рассмотрим первый случай. Scala позволяет работать с исключительными ситуациями схожим с Java или С++ способом. Для прекращения выполнения программы достаточно вызвать код `throw Exception`. Будет брошено исключение `Exception` и программа продолжит выполнение с ближайшего обработчика этого исключения. Понятно, что `throw Exception` ничего не возвращает, точнее тип этого выражения `scala.Nothing`.

Класс `scala.Null` является подклассом любого ссылочного типа. У `scala.Null` существует лишь один объект - `null`. Это значение означает, что объект отсутствует.


## Заключительное слово

В этот раз мы рассмотрели полиморфизм и наследование. Полиморфизм обширная тема, за которой стоит интересная научная [теория](http://logic.pdmi.ras.ru/csclub/courses/systemsoftypedlambdacalculi) и мы еще к нему вернемся. 

К идее наследования я лично отношусь достаточно прохладно. Основной проблемой наследования, на мой взгляд, является техническая невозможность запретить подклассам реализовывать поведение, логически противоречащее поведению базового типа. Так, например, в примере `IntSet` мы могли бы реализовать метод `incl` в классе `Empty` так, чтобы он форматировал жесткий диск. Получается, что нарушение свойства подкласса быть одновременно базовым классом невозможно контролировать. Этот факт значительно усложняет программы с наследованием.

В следующей статье мы подробно поговорим о типах, продолжим разговор о полиморфизме и разберемся в сопоставлении с образцом. 
	
