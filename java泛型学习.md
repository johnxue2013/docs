## 泛型的使用
泛型有三种使用方式，分别为：泛型类、泛型接口、泛型方法

### 泛型类
List<Object> l1和List<string> l2不存在继承关系，这完全是两个类型，尽管Object是String的父类。

### 关于泛型中的通配符 ?

当泛型类型不确定的时候，可以使用通配符。

泛型如果是通配符，则可以理解为所有泛型类型的父类，如
```java
List<String> str = new ArrayList<>();
//以下语句报错，也证明了List<String>不是List<Object>的子类
//List<Object> obj = str;

//以下语句编译通过，则证明List<?>是List<String>的父类
List<?> qu = str;
```

**在Java集合框架中，对于参数值是未知类型的容器类，只能读取其中元素，不能向其中添加元素， 因为，其类型是未知，所以编译器无法识别添加元素的类型和容器的类型是否兼容，唯一的例外是null(可以向集合中添加null元素，无法添加其他任何元素)**
