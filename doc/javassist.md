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
静态方法`ClassPool.getDefault（）`返回的默认`ClassPool`搜索与底层JVM（Java虚拟机）具有相同的路径。**如果程序在诸如JBoss和Tomcat之类的Web应用程序服务器上运行，则`ClassPool`对象可能无法找到用户类**，因为这样的Web应用程序服务器使用多个类加载器以及系统类加载器。在这种情况下，必须在`ClassPool`中注册其他类路径。假设pool引用`ClassPool`对象：
```Java
  pool.insertClassPath(new ClassClassPath(this.getClass()));
```  

此语句注册了用于加载此引用的对象的类的类路径。你可以使用任何Class对象作为参数而不是this.getClass（）。用于加载由该Class对象表示的类的类路径已注册。

你可以将目录名称注册为类搜索路径。例如，以下代码将目录/usr/local/javalib添加到搜索路径：
```Java
  ClassPool pool = ClassPool.getDefault();
  pool.insertClassPath("/usr/local/javalib");
```  
用户可以添加的搜索路径不仅是目录，还包括URL：
```Java
  ClassPool pool = ClassPool.getDefault();
  ClassPath cp = new URLClassPath("www.javassist.org", 80, "/java/", "org.javassist.");
  pool.insertClassPath(cp);
```  
该程序将“http://www.javassist.org:80/java/”添加到类搜索路径中。此URL仅用于搜索属于包org.javassist的类。例如，要加载类org.javassist.test.Main，其类文件将从以下位置获取： http://www.javassist.org:80/java/org/javassist/test/Main.class

此外，您可以直接向ClassPool对象提供一个字节数组，并从该数组构造一个CtClass对象。为此，请使用ByteArrayClassPath。例如，
```Java
  ClassPool cp = ClassPool.getDefault();
  byte[] b = a byte array;
  String name = class name;
  cp.insertClassPath(new ByteArrayClassPath(name, b));
  CtClass cc = cp.get(name);
```
获取的`CtClass`对象表示由b指定的类文件定义的类。如果调用`get（）`并且给予`get（）`的类名等于name指定的类名，则`ClassPool`从给定的`ByteArrayClassPath`读取类文件。

如果您不知道该类的完全限定名称，则可以在`ClassPool`中使用`makeClass（）`：
```Java
  ClassPool cp = ClassPool.getDefault();
  InputStream ins = an input stream for reading a class file;
  CtClass cc = cp.makeClass(ins);
```
`makeClass（）`返回从给定输入流构造的`CtClass`对象。 您可以使用`makeClass（）`将类文件急切地提供给`ClassPool`对象。 如果搜索路径包含大型jar文件，这可能会提高性能。 由于`ClassPool`对象按需读取类文件，因此它可能会重复搜索整个jar文件中的每个类文件。 `makeClass（）`可用于优化此搜索。 由`makeClass（）`构造的`CtClass`保存在`ClassPool`对象中，并且永远不会再读取类文件。  

用户可以扩展类搜索路径。他们可以定义一个实现`ClassPath`接口的新类，并将该类的实例提供给`ClassPool`中的`insertClassPath（）`。这允许将非标准资源包括在搜索路径中。

### 2.ClassPool
ClassPool对象是CtClass对象的容器。 创建CtClass对象后，它将永远记录在ClassPool中。 这是因为编译器在编译引用该CtClass表示的类的源代码时可能需要稍后访问CtClass对象。

例如，假设将一个新方法getter（）添加到表示Point类的CtClass对象中。 稍后，程序尝试编译源代码，包括在Point中调用getter（）的方法，并使用编译的代码作为方法的主体，将其添加到另一个类Line。 如果表示Point的CtClass对象丢失，则编译器无法将方法调用编译为getter（）。 请注意，原始类定义不包括getter（）。 因此，要正确编译这样的方法调用，ClassPool必须始终包含程序执行的所有CtClass实例。

#### 避免内存溢出
如果CtClass对象的数量变得非常大，那么ClassPool的这种规范可能会导致巨大的内存消耗（这很少发生，因为Javassist试图以各种方式减少内存消耗）。 要避免此问题，可以从ClassPool中显式删除不必要的CtClass对象。 如果在CtClass对象上调用detach（），则会从ClassPool中删除该CtClass对象。 例如，
```Java
  CtClass cc = ... ;
  cc.writeFile();
  cc.detach();
```
调用detach（）后，不得在该CtClass对象上调用任何方法。 但是，您可以在ClassPool上调用get（）以使CtClass的新实例表示相同的类。 如果调用get（），ClassPool会再次读取一个类文件并重新创建一个CtClass对象，该对象由get（）返回。

另一个方法是偶尔用新的ClassPool替换ClassPool并丢弃旧的ClassPool。 如果旧的ClassPool被垃圾收集，那么ClassPool中包含的CtClass对象也会被垃圾收集。 要创建ClassPool的新实例，请执行以下代码段：
```Java
ClassPool cp = new ClassPool(true);
// if needed, append an extra search path by appendClassPath()
```
这将创建一个ClassPool对象，这效果与ClassPool.getDefault（）返回的默认ClassPool相同。 请注意，为方便起见，ClassPool.getDefault（）是一个单独的工厂方法。 它以与上面所示相同的方式创建一个ClassPool对象，尽管它保留了ClassPool的单个实例并重用它。 getDefault（）返回的ClassPool对象没有特殊角色。 getDefault（）是一种方便的方法。
请注意，新的ClassPool（true）是一个方便的构造函数，它构造一个ClassPool对象并将系统搜索路径附加到它。调用该构造函数等效于以下代码：
```Java
ClassPool cp = new ClassPool();
cp.appendSystemPath();  // or append another path by appendClassPath()
```

#### 级联的ClassPool
**如果程序正在Web应用程序服务器上运行**,创建ClassPool的多个实例可能是必要的;应为每个类加载器（即容器）创建一个ClassPool实例。程序应该不调用getDefault（）而是调用ClassPool的构造函数来创建ClassPool对象。
多个ClassPool对象可以像`java.lang.ClassLoader`一样级联。例如，
```Java
ClassPool parent = ClassPool.getDefault();
ClassPool child = new ClassPool(parent);
child.insertClassPath("./classes");
```
如果调用了child.get（），则子类ClassPool首先委托父类ClassPool。如果父ClassPool无法找到类文件，则子ClassPool会尝试在./classes目录下查找类文件。  

如果child.childFirstLookup为true，则子ClassPool会在委派给父ClassPool之前尝试查找类文件。例如，
```Java
ClassPool parent = ClassPool.getDefault();
ClassPool child = new ClassPool(parent);
child.appendSystemPath();         // the same class path as the default one.
child.childFirstLookup = true;    // changes the behavior of the child.
```

#### 更改类名以定义新类
  可以将新类定义为现有类的副本。以下程序可以：
  ```Java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("Point");
  cc.setName("Pair");
````
该程序首先获取Point类的CtClass对象。 然后它调用setName（）为该CtClass对象赋予一个新的名称Pair。 在此调用之后，由该CtClass对象表示的类定义中出现的所有类名都将从Point更改为Pair。 类定义的其他部分不会改变。

请注意，CtClass中的setName（）更改了ClassPool对象中的记录。 从实现的角度来看，ClassPool对象是CtClass对象的哈希表。 setName（）更改与哈希表中的CtClass对象关联的键。 键已从原始类名更改为新类名。

因此，如果稍后再次在ClassPool对象上调用get（“Point”），则它永远不会返回变量cc引用的CtClass对象。 ClassPool对象再次读取一个类文件Point.class，它为类Point构造一个新的CtClass对象。 这是因为与名称Point关联的CtClass对象不再存在。 请参阅以下内容：
```Java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
CtClass cc1 = pool.get("Point");   // cc1 is identical to cc.
cc.setName("Pair");
CtClass cc2 = pool.get("Pair");    // cc2 is identical to cc.
CtClass cc3 = pool.get("Point");   // cc3 is not identical to cc.
```
cc1和cc2指的是与cc相同的CtClass实例，而cc3则不是。请注意，在执行`cc.setName("Pair")`之后，cc和cc1引用的`CtClass`对象表示`Pair`类。

ClassPool对象用于维护类和CtClass对象之间的一对一映射。 除非创建了两个独立的ClassPool，否则Javassist永远不会允许两个不同的CtClass对象表示同一个类。 这是一致的程序转换的重要特征。

要创建ClassPool.getDefault（）返回的ClassPool默认实例的另一个副本，请执行以下代码片段（此代码已在上面显示）：
```Java
ClassPool cp = new ClassPool(true);
```
如果您有两个ClassPool对象，则可以从每个ClassPool中获取表示同一类文件的不同CtClass对象。您可以不同地修改这些CtClass对象以生成该类的不同版本。

#### 重命名冻结类以定义新类
一旦CtClass对象被writeFile（）或toBytecode（）转换为类文件，Javassist就会拒绝对该CtClass对象的进一步修改。 因此，在将表示Point类的CtClass对象转换为类文件之后，您无法将Pair类定义为Point的副本，因为在Point上执行setName（）会被拒绝。 以下代码段错误：
```Java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("Point");
  cc.writeFile();
  cc.setName("Pair");    // wrong since writeFile() has been called.
```
要避免此限制，您应该在ClassPool中调用getAndRename（）。例如，
```Java
  ClassPool pool = ClassPool.getDefault();
  CtClass cc = pool.get("Point");
  cc.writeFile();
  CtClass cc2 = pool.getAndRename("Point", "Pair");
```

如果调用了getAndRename（），则ClassPool首先读取Point.class以创建表示Point类的新CtClass对象。 但是，在将CtClass对象记录在哈希表中之前，它会将该CtClass对象从Point重命名为Pair。 因此，在表示Point类的CtClass对象上调用writeFile（）或toBytecode（）之后，可以执行getAndRename（）。

### 3.类加载器
如果事先知道需要修改哪些类，则修改类最简单的方法如下:
 1. 调用ClassPool.get()获取CtClass对象
 2. 修改它，然后
 3. 调用CtClass对象的`writeFile()`或者`toBytecode()`方法获得一个修改后的类文件

如果在加载时确定是否修改了类，则用户必须使Javassist与类加载器协作。 Javassist可以与类加载器一起使用，以便可以在加载时修改字节码。 Javassist的用户可以定义他们自己的类加载器版本，但他们也可以使用Javassist提供的类加载器。

#### 3.1 CtClass中的toClass方法
`CtClass`提供了一个简便的`toClass()`方法，它请求当前线程的上下文类加载器加载由CtClass对象表示的类。 要调用此方法，调用者必须具有适当的权限; 否则，可能会抛出SecurityException。

以下代码展示如何使用`toClass()`方法:
```Java
public class Hello {
    public void say() {
        System.out.println("Hello");
    }
}

public class Test {
    public static void main(String[] args) throws Exception {
        ClassPool cp = ClassPool.getDefault();
        CtClass cc = cp.get("Hello");
        CtMethod m = cc.getDeclaredMethod("say");
        m.insertBefore("{ System.out.println(\"Hello.say():\"); }");
        Class c = cc.toClass();
        Hello h = (Hello)c.newInstance();
        h.say();
    }
}
```
Test.main（）在Hello中的say（）方法体中插入对println（）的调用。然后它构造一个修改过的Hello类的实例，并在该实例上调用say（）。

请注意，上面的程序取决于在调用toClass（）之前从不加载Hello类的事实。 如果不是，JVM将在`toClass()`请求加载被修改的Hello类之前请求加载原始的Hello类。 因此，加载修改后的Hello类将失败（抛出LinkageError）。 例如，如果Test中的main（）是这样的：

```Java
public static void main(String[] args) throws Exception {
    Hello orig = new Hello();
    ClassPool cp = ClassPool.getDefault();
    CtClass cc = cp.get("Hello");
        :
}
```
然后原始的Hello类在main的第一行加载，对toClass（）的调用抛出异常，因为类加载器不能同时加载两个不同版本的Hello类。

如果程序在某些应用程序服务器（如JBoss和Tomcat）上运行，则toClass（）使用的上下文类加载器可能不合适。 在这种情况下，您会看到意外的ClassCastException。 要避免此异常，必须为toClass（）显式提供适当的类加载器。 例如，如果bean是您的会话bean对象，那么以下代码：
```Java
CtClass cc = ...;
Class c = cc.toClass(bean.getClass().getClassLoader());
```
将会起作用,你应该给toClass（）加载你的程序的类加载器（在上面的例子中，bean对象的类）。

提供toClass（）是为了方便起见。如果您需要更复杂的功能，则应编写自己的类加载器。

#### 3.2 Java中的类加载器
在Java中，多个类加载器可以共存，每个类加载器都可以创建自己的名称空间。 不同的类加载器可以加载具有相同类名的不同类文件。 加载的两个类被视为不同的类。 此功能使我们能够在单个JVM上运行多个应用程序，即使这些程序包含具有相同名称的不同类。

注意：JVM不允许动态重新加载类。 一旦类加载器加载了一个类，它就无法在运行时重新加载该类的修改版本。 因此，在JVM加载类之后，你无法更改类的定义。 但是，JPDA（Java平台调试器体系结构）提供了重新加载类的有限能力。 见3.6节。

如果同一个类文件由两个不同的类加载器加载，则JVM会生成两个具有相同名称和定义的不同类。 这两个类被视为不同的类。 由于这两个类不相同，因此一个类的实例不能分配给另一个类的变量。 两个类之间的强制转换操作失败并抛出`ClassCastException`。

例如，以下代码段会引发异常
```Java
MyClassLoader myLoader = new MyClassLoader();
Class clazz = myLoader.loadClass("Box");
Object obj = clazz.newInstance();
Box b = (Box)obj;    // this always throws ClassCastException.
```
Box类由两个类加载器加载。 假设类加载器CL加载包含此代码片段的类。 由于此代码段引用了MyClassLoader，Class，Object和Box，因此CL还会加载这些类（除非它委托给另一个类加载器）。 因此，变量b的类型是CL加载的Box类。 另一方面，myLoader也加载Box类。 对象obj是myLoader加载的Box类的实例。 因此，最后一个语句总是抛出一个ClassCastException，因为obj的类是Box类的一个不同的版本，而不是用作变量b的类型。

多个类加载器形成树结构。 除引bootstrap loader之外的每个类加载器都有一个父类加载器，它通常加载了该子类加载器的类。 由于可以沿着类加载器的这个层次结构委托加载类的请求，因此可以通过不请求类加载的类加载器加载类。 因此，请求加载类C的类加载器可能与实际加载类C的加载器不同。为了区分，我们将前加载器称为C的发起者，并将后者加载器称为C的真实加载器。

此外，如果一个类加载器CL请求加载一个类C（C的发起者）委托给父类加载器PL，那么类加载器CL永远不会被请求加载在类C的定义中引用的任何类.CL 不是这些类的发起者。 相反，父类加载器PL成为它们的启动器，并要求加载它们。 类C的定义所引用的类由C的真实加载器加载。

为了了解此行为，请考虑以下示例。

```Java
  public class Point {    // loaded by PL
      private int x, y;
      public int getX() { return x; }
          :
  }

  public class Box {      // the initiator is L but the real loader is PL
      private Point upperLeft, size;
      public int getBaseX() { return upperLeft.x; }
          :
  }

  public class Window {    // loaded by a class loader L
      private Box box;
      public int getBaseX() { return box.getBaseX(); }
  }
```
假设一个类`Window`由类加载器L加载.`Window`的启动器和实际加载器都是L.由于`Window`引用了`Box`，JVM将请求L加载`Box`。 这里，假设L将此任务委托给父类加载器PL。 Box的发起者是L，但真正的加载器是PL。 在这种情况下，`Point`的发起者不是L而是PL，因为它与`Box`的真实加载器相同。 因此，L永远不会被请求加载`Point`。

接下来，让我们考虑一个稍微修改过的示例。
```Java
public class Point {
    private int x, y;
    public int getX() { return x; }
        :
}

public class Box {      // the initiator is L but the real loader is PL
    private Point upperLeft, size;
    public Point getSize() { return size; }
        :
}

public class Window {    // loaded by a class loader L
    private Box box;
    public boolean widthIs(int w) {
        Point p = box.getSize();
        return w == p.getX();
    }
}
```
现在，`Window`的定义也引用`Point`。 在这种情况下，如果请求加载Point，则类加载器L也必须委托给PL。 **你必须避免让两个类加载器加倍加载同一个类**。 两个装载机中的一个必须委托给另一个。

如果L在加载Point时没有委托给PL，则widthIs（）会抛出ClassCastException。 由于Box的真实加载器是PL，因此Box中引用的Point也由PL加载。 因此，getSize（）的结果值是由PL加载的Point的实例，而widthIs（）中的变量p的类型是由L加载的Point.JVM将它们视为不同的类型，因此它因类型而抛出异常不匹配。

这种行为有点不方便但必要。如果声明如下：
```Java
Point p = box.getSize();
```
没有抛出异常，那么Window的程序员可以打破Point对象的封装。例如，字段x在由PL加载的Point中是私有的。但是，如果L加载以下定义的Point，则Window类可以直接访问x的值：
```Java
public class Point {
    public int x, y;    // not private
    public int getX() { return x; }
        :
}
```
有关Java中类加载器的更多详细信息，以下文章将有所帮助:
Sheng Liang and Gilad Bracha, "Dynamic Class Loading in the Java Virtual Machine",
ACM OOPSLA'98, pp.36-44, 1998.

#### 3.3 使用javassist.Loader
Javassist提供了一个类加载器javassist.Loader。此类加载器使用javassist.ClassPool对象来读取类文件。 例如，javassist.Loader可用于加载使用Javassist修改的特定类。

```Java
import javassist.*;
import test.Rectangle;

public class Main {
  public static void main(String[] args) throws Throwable {
     ClassPool pool = ClassPool.getDefault();
     Loader cl = new Loader(pool);

     CtClass ct = pool.get("test.Rectangle");
     ct.setSuperclass(pool.get("test.Point"));

     Class c = cl.loadClass("test.Rectangle");
     Object rect = c.newInstance();
         :
  }
}
```
该程序修改了一个类test.Rectangle。 test.Rectangle的超类设置为test.Point类。然后，该程序加载修改后的类，并创建test.Rectangle类的新实例。

如果用户希望在加载时按需修改类，则用户可以向javassist.Loader添加事件侦听器。 当类加载器加载类时，会通知添加的事件侦听器。 事件监听器类必须实现以下接口：
```Java
public interface Translator {
    public void start(ClassPool pool) throws NotFoundException, CannotCompileException;
    public void onLoad(ClassPool pool, String classname) throws NotFoundException, CannotCompileException;
}
```
当javassist.Loader中的addTranslator（）将此事件侦听器添加到javassist.Loader对象时，将调用方法start（）。 在javassist.Loader加载类之前调用onLoad（）方法。 onLoad（）可以修改加载类的定义。
例如，以下事件侦听器在加载之前将所有类更改为公共类。
```Java
public class MyTranslator implements Translator {
    void start(ClassPool pool)
        throws NotFoundException, CannotCompileException {}
    void onLoad(ClassPool pool, String classname)
        throws NotFoundException, CannotCompileException
    {
        CtClass cc = pool.get(classname);
        cc.setModifiers(Modifier.PUBLIC);
    }
}
```
请注意，onLoad（）不必调用toBytecode（）或writeFile（），因为javassist.Loader调用这些方法来获取类文件。

要使用MyTranslator对象运行应用程序类MyApp，请按如下方式编写主类：
```Java
import javassist.*;

public class Main2 {
  public static void main(String[] args) throws Throwable {
     Translator t = new MyTranslator();
     ClassPool pool = ClassPool.getDefault();
     Loader cl = new Loader();
     cl.addTranslator(pool, t);
     cl.run("MyApp", args);
  }
}
```
要运行此程序，请执行：
% java Main2 arg1 arg2...
MyApp和其他应用程序类由MyTranslator转换。

请注意，像MyApp这样的应用程序类无法访问Main2，MyTranslator和ClassPool等加载器类，因为它们由不同的加载器加载。 应用程序类由javassist.Loader加载，而加载器类（如Main2）由默认的Java类加载器加载。

javassist.Loader以与java.lang.ClassLoader不同的顺序搜索类。 ClassLoader首先将加载操作委托给父类加载器，然后仅在父类加载器找不到它们时才尝试加载类。 另一方面，javassist.Loader尝试在委托给父类加载器之前加载类。 它仅在以下情况下委托：
- 在ClassPool对象上调用get（）找不到类，或者
- 通过使用delegateLoadingOf（）来指定类由父类加载器加载。

此搜索顺序允许Javassist加载修改的类。 但是，如果由于某种原因无法找到修改的类，它会委托给父类加载器。 一旦类由父类加载器加载，该类中引用的其他类也将由父类加载器加载，因此它们永远不会被修改。 回想一下，C类中引用的所有类都由C的实际加载器加载。如果您的程序无法加载修改后的类，则应确保使用该类的所有类是否已由javassist.Loader加载。

#### 3.4 写一个类加载器
使用Javassist的简单类加载器如下：
```Java
import javassist.*;

public class SampleLoader extends ClassLoader {
    /* Call MyApp.main().
     */
    public static void main(String[] args) throws Throwable {
        SampleLoader s = new SampleLoader();
        Class c = s.loadClass("MyApp");
        c.getDeclaredMethod("main", new Class[] { String[].class })
         .invoke(null, new Object[] { args });
    }

    private ClassPool pool;

    public SampleLoader() throws NotFoundException {
        pool = new ClassPool();
        pool.insertClassPath("./class"); // MyApp.class must be there.
    }

    /* Finds a specified class.
     * The bytecode for that class can be modified.
     */
    protected Class findClass(String name) throws ClassNotFoundException {
        try {
            CtClass cc = pool.get(name);
            // modify the CtClass object here
            byte[] b = cc.toBytecode();
            return defineClass(name, b, 0, b.length);
        } catch (NotFoundException e) {
            throw new ClassNotFoundException();
        } catch (IOException e) {
            throw new ClassNotFoundException();
        } catch (CannotCompileException e) {
            throw new ClassNotFoundException();
        }
    }
}
```
MyApp类是一个应用程序。 要执行此程序，首先将类文件放在./class目录下，该目录不能包含在类搜索路径中。 否则，MyApp.class将由默认的系统类加载器加载，该加载器是SampleLoader的父加载器。 目录名./class由构造函数中的insertClassPath（）指定。 如果需要，你可以选择其他名称替换./class。 然后执行以下操作：
```Java
% java SampleLoader
```
类加载器加载类MyApp（./class/MyApp.class）并使用命令行参数调用MyApp.main（）。

这是使用Javassist的最简单方法。 但是，如果编写更复杂的类加载器，则可能需要详细了解Java的类加载机制。 例如，上面的程序将MyApp类放在与SampleLoader类所属的名称空间分开的名称空间中，因为这两个类由不同的类加载器加载。 因此，MyApp类无法直接访问SampleLoader类。

#### 3.5 修改系统类
系统类（如java.lang.String）不能由系统类加载器以外的类加载器加载。因此，上面显示的SampleLoader或javassist.Loader无法在加载时修改系统类。

如果你的应用程序需要这样做，则必须对系统类进行静态修改。例如，以下程序向java.lang.String添加一个新字段hiddenValue：
```Java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("java.lang.String");
CtField f = new CtField(CtClass.intType, "hiddenValue", cc);
f.setModifiers(Modifier.PUBLIC);
cc.addField(f);
cc.writeFile(".");
```
This program produces a file "./java/lang/String.class".
要使用此已修改的String类运行程序MyApp，请执行以下操作：

假设MyApp的定义如下：
```Java
public class MyApp {
    public static void main(String[] args) throws Exception {
        System.out.println(String.class.getField("hiddenValue").getName());
    }
}
```
如果正确加载了修改后的String类，MyApp将打印hiddenValue。

注意：不应部署使用此技术来覆盖rt.jar中的系统类的应用程序，因为这样做会违反Java 2 Runtime Environment二进制代码许可证。

#### 3.6 在运行时重新加载类
如果在启用JPDA（Java平台调试器体系结构）的情况下启动JVM，则可以动态地重新加载类。 在JVM加载类之后，可以卸载旧版本的类定义，并且可以再次重新加载新版本。 也就是说，可以在运行时动态修改该类的定义。 但是，新类定义必须与旧类定义兼容。 JVM不允许两个版本之间的架构更改。 他们有相同的方法和领域。
Javassist提供了一个方便的类，用于在运行时重新加载类。有关更多信息，请参阅javassist.tools.HotSwapper的API文档。
















[1]:http://www.javassist.org/tutorial/tutorial2.html#intro



> 1. 译者注:系统类加载器是由 Sun的 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。它负责将系统类路径java -classpath或-Djava.class.path变量所指的目录下的类库加载到内存中。开发者可以直接使用系统类加载器。一般来说，Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader()来获取它。

> 本文本基于Javassist version 3.24.0-GA翻译
