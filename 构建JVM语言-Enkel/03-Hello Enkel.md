# 构建JVM语言 - Enkel

<h2 align="center">【第三节】：Hello Enkel</h2>

</br>

[原文](http://jakubdziworski.github.io/enkel/2016/03/16/enkel_3_hello_enkel.html)

</br>

## 自顶向下的方式

由于构建一门语言不是一个短暂的任务，在项目的开发期间我将非常依赖自顶向下的方式。我将以最简洁的方式一次性描述所有的模块，而不是面面俱到，分别介绍每个模块的细节。每次迭代后，我都会在项目中添加一些新功能，并对其进行详细说明。

## 源码

这个项目的源码可以从[Github仓库](https://github.com/JakubDziworski/Enkel-JVM-language)中进行克隆。

## 特性

这一节我将会给Enkel语言添加以下特性：

- 用整型（`int`）或字符串类型（`string`）来描述变量
- 打印变量
- 简单的类型引用

因此，让我们从最简洁的实现开始，让Enkel的第一行代码在JVM上运行。这是将在一天结束时执行的代码。

```enk
var five = 5
print five
var dupa = "duap"
print dupa
```

## 使用Antlr4进行词法分析与语法解析

我本来是想从头开始实现一个词法分析器的，但是这似乎看起来是个非常具有重复性的工作。在浏览网页一段时间后我发现了一个很棒的工具交“Antlr”。你只需用描述你的语言规则的类似组织结构的文件来“喂”它，它就会为你提供一种遍历抽象语法树（也叫解析树）的方式。让我们来为Enkel语言创建一个非常简单的规则

```anltr
//header
grammar Enkel;

//parser rules
compilationUnit : ( variable | print )* EOF; //root rule - globally code consist only of variables and prints (see definition below)
variable : VARIABLE ID EQUALS value; //requires VAR token followed by ID token followed by EQUALS TOKEN ...
print : PRINT ID ; //print statement must consist of 'print' keyword and ID
value : NUMBER
      | STRING ; //must be NUMBER or STRING value (defined below)

//lexer rules (tokens)
VARIABLE : 'var' ; //VARIABLE TOKEN must match exactly 'var'
PRINT : 'print' ;
EQUALS : '=' ; //must be '='
NUMBER : [0-9]+ ; //must consist only of digits
STRING : '"'.*'"' ; //must be anything in qutoes
ID : [a-zA-Z0-9]+ ; //must be any alphanumeric value
WS: [ \t\n\r]+ -> skip ; //special TOKEN for skipping whitespaces
```