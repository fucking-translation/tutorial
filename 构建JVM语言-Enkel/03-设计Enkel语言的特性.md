# 构建JVM语言 - Enkel

<h2 align="center">【第四节】：设计Enkel语言的特性</h2>

</br>

[原文](http://jakubdziworski.github.io/enkel/2016/03/22/enkel_4_design.html)

</br>

鉴于我已经实现了Enkel编译器第一个原型版本，是时候花点时间去真正的设计在后续迭代中要添加的特性。

在Java中，有很多冗长的特性。他一方面通常会保护你免于出错，并最小化不被预期的“魔法”隐式操作。但是在另一方面，他会让我们写了很多重复的代码。

我们的主要目标是使Enkel与之相反 - 尽量避免冗长的代码。这既有优点也有缺点，但是对于原型制作来说绝对是很棒的。这里有一些将在下轮迭代中实现的特性：


|特性|示例|
|:-:|:-:|
|一个文件对应一个类，也就无需指明“class”关键字 - 只需在导入类库后的第一行提供类名即可| `Car {}`|
| 继承 | `Car : Vehicle {}` |
|可选的自动生成getters，setters，builder，equals，hashcode方法| `Car(getters, setters, hashcode, equals, builder) : Vehicle {}` |
|类型推断| `var x = 5` |
|默认参数| `fun createPoint(Int x = 0, Int y = 0)` |
|当调用方法时，可选的参数名称指定，这使得可读性更好，支持任意顺序的参数列表| `createPoint(5, 0)` </br> `createPoint(x->5, y->0)` </br> `createPoint(y->0, x->5)` |
|方法可以作为对象| `const f = (Int x = 0, Int y = 0) => x*y` |
|没有静态的上下文| ~~`static void x()`~~ |
|默认用 == 取代 equals| object1 == object2 |


</br></br></br>

<div align="left"><a href="./02-Hello Enkel.md">上一节</a></div>

<div align="left"><a href="./03-设计Enkel语言的特性.md">下一节</a></div>
