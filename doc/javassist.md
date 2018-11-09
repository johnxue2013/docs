## Javassist
Javassist(Java Programming Assistant) 使得操作Java字节码变得简单。它是一个用于在Java中编辑字节码的类库；它使Java程序能够在运行时定义新类，并在JVM加载时修改类文件。与其他类似的字节码编辑器不同，Javassist提供两个级别的API：源级别和字节码级别。如果用户使用源级API，他们可以编辑类文件而不需要了解Java字节码的规范。整个API仅使用Java语言的风格进行设计。您甚至可以以源文本的形式指定插入的字节码; Javassist将即时编译它。另一方面，字节码级别API允许用户像其他编辑器一样直接编辑类文件(class file)。

### 1. 读写字节码
Javassist 是一个处理Java字节码的类库。Java字节码存储在一个叫做类文件的二进制文件中。每个类文件包含一个Java类(class)或者接口(interface)。
`Javassist.CtClass`类是类文件的抽象表示。一个`CtClass(compile-time class)`对象是用于处理一个类文件的句柄.下面的程序是一个非常简单的例子:

```Java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("test.Rectangle");
  cc.setSuperclass(pool.get("test.Point"));
  cc.writeFile();
  ```
这段程序首先获取一个在Javassist中用来控制字节码修改的`ClassPool`对象。`ClassPool`对象是表示类文件的`CtClass`对象的容器,它根据需要读取类文件以构造`CtClass`对象，并记录构造的对象以响应以后的访问。要修改类的定义，用户必须首先从`ClassPool`对象获取对表示该类的`CtClass`对象的引用。 `ClassPool`中的`get()`用于此目的。上述代码表示类`test.Rectangle`的`CtClass`对象是从ClassPool对象获得的，并且它被赋值给变量cc。`getDefault()`返回的`ClassPool`对象搜索默认的系统搜索路径<sup>1</sup>。

从实现的角度来看，`ClassPool`是`CtClass`对象的哈希表，它使用类名作为键。 `ClassPool`中的`get()`搜索此哈希表以查找与指定键关联的`CtClass`对象。如果没有找到，`get()`将读取类文件来构造一个新的`Ctclass`对象并存储在在哈希表中然后返回该对象。

从`ClassPool`对象中获取的`Ctclass`是可以修改的([details of how to modify a CtClass][1] will be presented later)。在上面的示例中，对其进行了修改，以便将`test.Rectangle`的超类更改为类`test.Point`。当最终调用`CtClass()`中的`writeFile()`时，此更改将反映在原始类文件中。

`writeFile()`将`CtClass`对象转换为类文件并将其写入本地磁盘。Javassist还提供了一种直接获取修改后的字节码的方法。要获取字节码，请调用`toBytecode()`：  

```Java
  byte[] b = cc.toBytecode();
```
你也可以直接直接加载`CtClass`:
```Java
  Class clazz = cc.toClass();
```
`toClass()`请求当前的上下文加载器加载`CtClass`代表的类文件。它返回一个代表被加载的类的`java.lang.Class`对象。

#### 定义一个新类
要从头开始定义新类，必须在`ClassPool`上调用`makeClass()`。
```Java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.makeClass("Point");
```
这段代码定义了一个没有成员属性的`Point`类。`Point`的成员方法可以使用在`CtNewMethod`中声明的工厂方法创建，并使用`CtClass`中的`addMethod()`添加到`Point`类。

`makeClass()`无法创建新接口; `ClassPool`中的`makeInterface()`可以做到。可以使用`CtNewMethod`中的`abstractMethod()`创建接口中的成员方法。注意，接口方法是一种抽象方法。

#### 冻结类
如果通过`writeFile（）``，`toClass（）`或`toBytecode（）`将`CtClass`对象转换为类文件，Javassist将冻结该`CtClass`对象。不允许对该CtClass对象进行进一步修改。这是为了在开发人员尝试修改已加载的类文件时警告开发人员，因为JVM不允许重新加载类。

冻结的`CtClass`可以解冻，以便允许修改类定义。举例
```Java
  CtClasss cc = ...;
      :
  cc.writeFile();
  cc.defrost();
  cc.setSuperclass(...);    // OK since the class is not frozen.
```
当调用`defrost()`之后，`CtClass`对象将再次可以被修改。

如果`ClassPool.doPruning`设置为`true`，那么当Javassist冻结该对象时，Javassist会修剪CtClass对象中包含的数据结构。 为了减少内存消耗，修剪会丢弃该对象中不必要的属性（attribute_info结构）。例如，丢弃Code_attribute结构（方法体）。 因此，在修剪`CtClass`对象之后，除方法名称，签名和注释外，不能访问方法的字节码。 修剪过的`CtClass`对象无法再次解冻。 `ClassPool.doPruning`的默认值为`false`。

要禁止修剪特定的`CtClass`，必须事先在该对象上调用`stopPruning（）`:
```Java
  CtClasss cc = ...;
  cc.stopPruning(true);
      :
  cc.writeFile();     // convert to a class file.
  // cc is not pruned.
```
`CtClass`对象cc未被修剪。因此，在调用`writeFile（）`之后可以解冻。

> Note:在调试时，你可能希望暂时停止修剪和冻结，并将修改后的类文件写入磁盘。 `debugWriteFile（）``是一种方便的方法。 它停止修剪，写一个类文件，解冻它，并再次修剪（如果它最初打开）。

#### 类搜索路径


























[1]:http://www.javassist.org/tutorial/tutorial2.html#intro



> 1. 译者注:系统类加载器是由 Sun的 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。它负责将系统类路径java -classpath或-Djava.class.path变量所指的目录下的类库加载到内存中。开发者可以直接使用系统类加载器。一般来说，Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader()来获取它。

> 本文本基于Javassist version 3.24.0-GA翻译
