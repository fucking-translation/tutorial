# 构建JVM语言 - Enkel

<h2 align="center">【第十七节】：引用其他的类（包括Java API）</h2>

</br>

[原文](http://jakubdziworski.github.io/enkel/2016/05/12/enkel_17_calling_other_classes_and_java.html)

</br>

## 源码

这个项目的源码可以从[Github仓库](https://github.com/JakubDziworski/Enkel-JVM-language)中进行克隆。

## 字节码 - JVM语言的共同点

所有的JVM语言都可以被编译成字节码，由虚拟机解释。这意味着编译器根本不知道引用的类是由什么语言编译的。不管是什么语言，只要类在CLASSPATH上就可以被使用。

这就给我们带来了很大的可能性。所有的Java类库，工具集和框架 都可以被Enkel使用。

## 查询类方法和结构体

当你要引用在不同的class文件中的类时，你有两个选择：

- 运行时 - 相信开发者，不校验CLASSPATH上的签名就生成字节码。如果签名在CLASSPATH上不可用，可能会抛出运行时异常。

- 编译时 - 在生成字节码之前校验CLASSPATH上的签名。如果一些引用的签名不可用，将停止编译的过程。

在Enkel中，我决定使用第二种方式 - 主要是为了安全。这可以使用反射的api实现。

```java
public class ClassPathScope {

 public Optional<FunctionSignature> getMethodSignature(Type owner, String methodName, List<Type> arguments) {
     try {
         Class<?> methodOwnerClass = owner.getTypeClass();
         Class<?>[] params = arguments.stream()
                 .map(Type::getTypeClass).toArray(Class<?>[]::new);
         Method method = methodOwnerClass.getMethod(methodName,params);
         return Optional.of(ReflectionObjectToSignatureMapper.fromMethod(method));
     } catch (Exception e) {
         return Optional.empty();
     }
 }

 public Optional<FunctionSignature> getConstructorSignature(String className, List<Type> arguments) {
     try {
         Class<?> methodOwnerClass = Class.forName(className);
         Class<?>[] params = arguments.stream()
                 .map(Type::getTypeClass).toArray(Class<?>[]::new);
         Constructor<?> constructor = methodOwnerClass.getConstructor(params);
         return Optional.of(ReflectionObjectToSignatureMapper.fromConstructor(constructor));
     } catch (Exception e) {
         return Optional.empty();
     }
 }
}
```

如果没有发现相关的方法（或结构体），将会在编译的过程中抛出异常并终止。

```java
// Scope.java
return new ClassPathScope().getMethodSignature(owner.get(), methodName, argumentsTypes).orElseThrow(() -> new MethodSignatureNotFoundException(this,methodName,arguments));
```

这种方式看起来更加安全，但是也更慢。当编译时使用反射，所有的依赖必须都要加载完。

## 示例

### 调用其他Enkel类

让我们尝试从`Client`类中调用`Library`类：

```groovy
Client {
     start {
         print "Client: Calling my own 'Library' class:"
         var myLibrary = new Library()
         var addition = myLibrary.add(5,2)
         print "Client: Result returned from 'Library.add' = " + addition
     }
 }
```

```groovy
Library {
    int add(int x,int y) {
        print "Library: add() method called"
        return x+y
    }
}
```

首先我们需要编译`Library`类（现在还不支持多文件编译）。如果我们不这么做，`Client`类的编译将会失败，因为无法正常的从`Library`中进行引用。

```shell
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java -classpath compiler/target/compiler-1.0-SNAPSHOT-jar-with-dependencies.jar:. com.kubadziworski.compiler.Compiler EnkelExamples/ClassPathCalls/Library.enk 
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java -classpath compiler/target/compiler-1.0-SNAPSHOT-jar-with-dependencies.jar:. com.kubadziworski.compiler.Compiler EnkelExamples/ClassPathCalls/Client.enk 
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java Client 
Client: Calling my own 'Library' class:
Library: add() method called
Client: Result returned from 'Library.add' = 7
```

### 调用Java API！

```java
Client {
    start {
        var someString = "someString"
        print someString + " to upper case : " +  someString.toUpperCase()
    }
}
```

```shell
kuba@kuba-laptop:~/repos/Enkel-JVM-language$ java Client 
cos to upper case = COS
```

</br></br></br>

<div align="left"><a href="./15-埋葬静态属性.md">上一节</a></div>

<div align="left"><a href="./16-引用其他的类(包括Java Api).md">下一节</a></div> 