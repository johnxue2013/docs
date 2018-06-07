# Java New I/O
## 第二章 缓冲区
## 缓冲区(Buffer)
属性
- 容量(Capacity)
缓冲区能够容纳的数据元素的最大数量，这一容量在缓冲区创建时被设定，并且永远不能被改变。
- 上界(Limit)
缓冲区第一个不能被读或写的元素。或者说，缓冲区中现存元素的计数。
- 位置(Position)
下一个要被读或者写的元素的索引。位置会自动由相应的get()和put()函数更新
- 标记(Mark)
一个备忘位置。调用mark()来设定mark=position。调用reset()设定position=mark。标记在设定前是未定义的

四个属性遵循以下关系：
0 <= mark <= position <= limit <= capacity

> 所有缓冲区都是可读的，但并非所有都可写。

### 存取
缓冲区中的put和get可以是相对的或者是绝对的。相对方案是不带有索引参数的函数。当相对函数被调用时，位置(position)在返回时前进1，如果位置前进过多，相对运算就会抛出异常。绝对存取不会影响缓冲区的位置属性，但是如果调用时提供的索引超过范围（负数或不小于上界）也会抛出异常。
### 填充
看个例子将代表"Hello"字符串的ASCII骂载入一个名为buffer的ByteBuffer对象中

```java
buffer.put((byte)'H').put((byte)'e')put((byte)'l')put((byte)'l')put((byte)'o');
```
五次调用put()之后的缓冲区

![image](https://github.com/johnxue2013/docs/blob/master/images/2_3.jpg)




> 在java中字符以Unicode码表示,每个Unicode字符占16位。
### 翻转
当写满了缓冲区，现在我们必须准备将其清空。把缓冲区传递给一个通道，以使内容能被全部写出。但如果通道现在在缓冲区上执行get()，那么它将从我们刚刚插入有用数据之外取出未定义的数据。如果我们将位置设置为0，通道就会从正确的位置开始获取，但是是怎样知道何时到达我们插入数据的末端的呢？这就是上界属性被引入的目的。上界属性指明了缓冲区有效内容的末端。我们需要将上界(limit)属性设置为当前位置(position)，然后将位置重置为0。可以人工用下面代码实现
```java
buffer.limit(position).position(0);
```
但这种从填充状态到释放状态的缓冲区翻转是API设计者预先设计好的，他们为我们提供一个非常便利的函数"
```java
buffer.flip()
```
`flip()`函数将一个能够继续添加数据元素的填充状态的缓冲区翻转成一个准备读出元素的释放状态。

### 释放
`hasRemaining()`函数会在释放缓冲区时告诉您是否已经达到缓冲区的上界。以下是一种将元素从缓冲区释放到一个数组的方法。
```java
for(int i = 0; buffer.hasRemaining(); i++) {
  myByteArray[i] = buffer.get();
}
```

作为选择，remaining()函数将返回从当前位置到上界(limit)还剩余的元素个数，也可以通过下面的循环来释放缓冲区
```java
int count = buffer.remaining();
for(int i = 0; i < count; i++) {
  myByteArray[i] = buffer.get();
}
```
一旦缓冲区对象完成填充并释放，它就可以被重新使用了。clear()函数将缓冲区重置为空的状态。它并不改变缓冲区中的任何元素，而是仅仅将limit设置为容量的值，并把position设置回0，这使得缓冲区可以被重新装入。
> 缓冲区不是线程安全的，如果想以多线程同时存取，则需要进行同步

```java
import java.nio.CharBuffer;

/**
 * demo 2.1 填充和释放缓冲区
 * @author han.xue
 * @since 2018-05-11 20:06:06
 */
public class BufferFillDrain {
	private static int index = 0;
	private static String[] strings = {
			"A random string value",
			"The product of an infinite number of monkeys",
			"Hey hey we're the Monkees",
			"Opening act for the Monkees: Jimi Hendrix",
			"'Scuse me while I kiss this fly", // Sorry Jimi ;-)
			"Help Me! Help Me!"
	};

	public static void main(String[] args) {
		CharBuffer buffer = CharBuffer.allocate(100);

		while (fillBuffer(buffer)) {
			//读之前先翻转状态
			buffer.flip();
			drainBuffer(buffer);
			//读完后重置，以便再次写
			buffer.clear();
		}
	}

	private static void drainBuffer(CharBuffer buffer) {
		while (buffer.hasRemaining()) {
			System.out.print(buffer.get());
		}
		System.out.println("");
	}

	private static boolean fillBuffer(CharBuffer buffer) {
		if (index >= strings.length) {
			return false;
		}

		String string = strings[index++];

		for(int i = 0; i < string.length(); i++) {
			buffer.put(string.charAt(i));
		}

		return true;
	}
}
```

### 压缩
有事，想从缓冲区中释放一部分区域，而不是全部，然后重新填充。此时可以使用compact()函数，这个函数比自己是使用get()和put()函数实现压缩高效的多。
调用compact的作用是丢弃已经释放的数据，保留未被释放的数据，并使缓冲区对重新填充做好准备

> 当全部元素都读取完毕时，调用compact()和clear()效果是一样的。

![压缩前](https://github.com/johnxue2013/docs/blob/master/images/2_6.png)
调用compact()方法后
![压缩前](https://github.com/johnxue2013/docs/blob/master/images/2_7.png)

此时数据元素2-5被复制到0-3的位置，位置4和5不受影响，再次写入时将被覆盖。最后上界limit被设置为容量大小，并做好被再次填充的准备。

> 如果想在压缩后clear数据，还是需要先翻转

### 标记
标识使缓冲区能够记住一个位置(position)，并在之后将其返回。缓冲区的mark属性值在调用mark()之前是未定义的(默认值为-1)，调用时将被设置为当前position的值。reset()函数将position设为当前的标记值，调用reset()时如果mark值为-1将导致`InvalidMarkException`异常.一些缓冲区函数总是(rewind() clear()以及flip())会抛弃已经设定的标记。

>reset()不要和clear()弄混淆

### 比较
提供了两个函数equals和compareTo

### 创建缓冲区
以CharBuffer为例，对其他六种主要的缓冲区也是通用的：
```java
public abstract class CharBuffer extends Buffer implements CharSequence,Comparable {
  public static CharBuffer allocate(int capacity);

  public static CharBuffer wrap(char[] array);
  public static CharBuffer wrap(char[] array, int offset, length);

  //标识是否有可存取的备份数组
  public final static boolean hasArray();

  //如果hashArray()返回true，则array函数会返回这个缓冲区对象所使用的数组存储空间的引用。如果hasArray返回false，不要调用array或者arrayOffset函数，调用会得到一个异常
  public final static char[] array();
  public final int arrayOffset();
}
```

新的的缓冲区可通过分配或者包装创建。分配操作创建一个缓冲区对象并分配要给私有的空间来存储数据元素，包装操作一个缓冲区但是不分配任何空间来存储数据元素。
```java
CharBuffer buffer = CharBuffer.allocate(100);

//wrap的方式创建意味着调用put函数造成对缓冲区的改动会直接影响数组，而且对数组的改动也会对这个缓冲区课件。带有offset和length作为参数的wrap会构建一个position值为offset，limit值为length + offset，容量为array.length的缓冲区
char array = new char[100];
CharBuffer buffer = CharBuffer.wrap(array);
```
> 通过allocate()或者wrap()函数创建的缓冲区通常都是间接的，间接的缓冲区使用备份数组。

### 复制缓冲区

当一个管理其他其他缓冲器所包含的元素的缓冲器被创建时，这个缓冲器被称为视图缓冲器。

以CharBuffer为例展示如何创建一个视图缓冲器
```java
public abstract class CharBuffer extends Buffer implements CharSequence,Comparable {
  /*创建一个与原始缓冲区相似的新缓冲区。两个缓冲区共享数据元素，拥有同样的容量，
  但每个缓冲区有自己的位置、上界、和标记属性。对一个缓冲器内的数据所做的改变会反映在爱另外一个缓冲区上。这一副本具有与原始缓冲区同样的数据视图，如果原始缓冲区为只读，或者为直接缓冲区，新的缓冲区将继承这些属性。
  */
  public abstract CharBuffer duplicate();

//可以使用asReadOnlyBuffer()函数生成一个只读的缓冲区视图。对只读缓冲区调用put将抛出异常
  public abstract CharBuffer asReadOnlyBuffer();

  //与复制相似，但slice创建一个从原始缓冲区当前位置开始的新缓冲区，且其容量是原始缓冲区的剩余元素数量(limit-position).
  //这个新缓冲区与原始缓冲区共享一段数据元素子序列。分割出来的换冲过去也会继承只读和直接属性。
  public abstract CharBuffer slice();
}
```



> 复制一个缓冲区会创建一个新的Buffer对象，但并不复制数据。原始缓冲区和副本都会操作同样的数据元素

如下代码将产生如图所示的效果
```java
CharBuffer buffer = CharBuffer.allocate(8);
buffer.position(3).limit(6).mark().position(5);
CharBuffer buffer = buffer.duplicate();
buffer.clear();
```
![复制](https://github.com/johnxue2013/docs/blob/master/images/2_12.png)


> 如果一个只读的缓冲区与一个可写的缓冲区共享数据，或者有包装好的备份数组，那么对这个可写的缓冲区或直接对这个数组的改变将反映在所有关联的缓冲区上，包括只读缓冲区。

要映射到数组位置12-20(9个元素)的buffer对象，应使用下面的代码实现
```java
char[] myBuffer = new char[100];
CharBuffer cb = CharBuffer.wrap(myBuffer);
cb.position(12).limit(21);
CharBuffer sliced = cb.slice();
```

## 字节缓冲区
### 字节顺序
如果一个缓冲区被创建为要给ByteBuffer对象的视图，那么order()返回的数值就是视图被创建时其创建源头的ByteBuffer字节顺序设定。视图的字节顺序在创建后不能被改变，而且如果原始的字节缓冲区的字节顺序在之后被改变，它也不会受到影响。
### 直接缓冲区
直接缓冲区通常是I/O操作最好的选择。在设计方面，它们支持JVM可用的最高效I/O机制。非直接字节缓冲区可以被传递给通道，但是这样可能导致性能损耗。

直接缓冲区是I/O的最佳选择，但可能比创建非直接缓冲区要花费更高的成本。直接缓冲区使用的内存是通过调用本地操作系统方面的代码分配的，绕过了JVM堆栈。

直接字节缓冲区ByteBuffer是通过调用allocateDirect()函数产生的。注意使用一个wrap()函数所创建的被包装的缓冲区总是非直接的。
