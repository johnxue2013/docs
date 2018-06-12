# Java New I/O
## 第三章 通道
Channel用于在字节缓冲区和位于通道另一侧的实体（通常是一个文件或套接字）之间有效地传输数据。

### 打开通道
I/O可以分为广义的File I/O和Stream I/O，那么相应的有两种类型的通道也就不足为怪了，它们是文件(file)通道和套接字(socket)通道。因此有`FileChannel`和三个Socket通道类：`SocketChannel`、`ServerSocketChannel`和`DatagramChannel`

`SocketChannel`有可以直接创建新socket通道的工厂方法。但是一个`FileChannel`通道只能通过在一个打开的`RandomAccessFile`
、`FileInputStream`或`FileOutputStream`对象上调用`getChannel()`方法来获取。不可以直接创建一个FileChannel对象.

```java

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.Channels;
import java.nio.channels.ReadableByteChannel;
import java.nio.channels.WritableByteChannel;

/**
 * 从一个通道复制数据到另一个通道
 * @author han.xue
 * @since 2018-05-12 23:20:20
 */
public class ChannelCopy {

	/**
	this code copies data from stdio to stout. like the 'cat' command,but without any useful options
	*/
	public static void main(String[] args) throws IOException {
		ReadableByteChannel rc = Channels.newChannel(System.in);
		WritableByteChannel wc = Channels.newChannel(System.out);

		channelCopy1(rc, wc);

		rc.close();
		wc.close();
	}


	/**
    * Channel copy method 1. This method copies data from the src
    * channel and writes it to the dest channel until EOF on src.
    * This implementation makes use of compact( ) on the temp buffer
    * to pack down the data if the buffer wasn't fully drained. This
    * may result in data copying, but minimizes system calls. It also
    * requires a cleanup loop to make sure all the data gets sent.
  */
	private static void channelCopy1(ReadableByteChannel src, WritableByteChannel dest) throws IOException {
		ByteBuffer buffer = ByteBuffer.allocateDirect(16 * 1024);
		while (src.read(buffer) != -1) {
			//prepare the buffer to be drained
			buffer.flip();
			//write to the channel; may block
			//某些通道可能不会一次写入buffer中的所有数据
			dest.write(buffer);

			//if partial transfer,shift remainder down
			//if buffer is empty,same as doing clear()
			buffer.compact();
		}

		buffer.flip();

		while (buffer.hasRemaining()) {
			dest.write(buffer);
		}
	}

	/**
  * Channel copy method 2. This method performs the same copy, but
  * assures the temp buffer is empty before reading more data. This
  * never requires data copying but may result in more systems calls.
  * No post-loop cleanup is needed because the buffer will be empty
  * when the loop is exited.
  */
	private static void channelCopy2(ReadableByteChannel src, WritableByteChannel dest) throws IOException {
		ByteBuffer buffer = ByteBuffer.allocateDirect(16 * 1024);

		while (src.read(buffer) != -1) {
			// Prepare the buffer to be drained
			buffer.flip();
			// make sure that buffer was fully drained
			while (buffer.hasRemaining()) {
				//某些通道可能不会一次写入buffer中的所有数据
				dest.write(buffer);
			}
			//make the buffer empty,ready for filling
			buffer.clear();
		}
	}
}
```

通道可以以阻塞(blocking)或者非阻塞(non-blocking)模式运行。非阻塞模式的通道永远不会让调用的线层休眠。请求的操作要么立即完成，要么返回一个结果表明未进行任何操作。只有面向流的(stream-oriented)的通道，如sockets和pips才能使用非阻塞模式

### 关闭通道
调用通道的close()方法时，可能会导致在通道关闭底层I/O服务的过程中线程暂时阻塞，那怕该通道处于非阻塞模式，通道的关闭时的阻塞行为（如果有的话）是高度取决于操作系统或者文件系统的。在一个通道上多次调用close()方法时没有坏处的，但是如果第一个线程在close()方法中阻塞，那么在它完成关闭通道之前，其他任何调用close()方法都会阻塞。后续在该已关闭的通道上调用close()不会产生任何操作，只会立即返回。

此外，如果一个已经被中断的线程(interrupt status被设置为中断状态)视图访问一个通道，那么这个通道将立即被关闭，同时将抛出异常

```java
public interface Channel {
	public boolean isOpen();
	public void close() throws IOException;
}
```

> 线程的中断状态可以通过Thread.interrupted()方法清除。

### Scatter/Gather
通道提供了一种被称为Scatter/Gather的重要功能（有时也称为矢量I/O），是指在多个缓冲区上实现一个简单的I/O操作。对于一个write操作而言，数据是从几个缓冲区按顺序抽取(称为gather)并沿着通道发送的。缓冲区本身并不需要具备这种能力（通常它们也没有此能力）。该gather过程的效果就好比全部缓冲区的内容被链接起来，并在发送数据前存放到一个大的缓冲区中。

对于read操作而言，从通道读取的数据会按顺序被散布（称为scatter）到多个缓冲区，将每个缓冲区填满直至通道中的数据或者缓冲区的最大空间被消耗完。

```java
//接口定义
public interface ScatteringByteChannel
    extends ReadableByteChannel
{

    public long read(ByteBuffer[] dsts, int offset, int length)
        throws IOException;

    public long read(ByteBuffer[] dsts) throws IOException;

}
```

```java

public interface GatheringByteChannel
    extends WritableByteChannel
{

    public long write(ByteBuffer[] srcs, int offset, int length)
        throws IOException;

    public long write(ByteBuffer[] srcs) throws IOException;

}
```


```java
scatter使用方法demo
ByteBuffer header = ByteBuffer.allocateDirect(10);
ByteBuffer body = ByteBuffer.allocateDirect(80);

ByteBuffer[] buffers = {header, body};

//假定channel链接到了一个有48字节数据等待读取的socket上
//当read方法返回时，bytesRead就被赋值48，header缓冲区将包含前10个从通道
//连续读取的字节，而body则包含接下来的38个字节，通道会自动的将数据scatter到这两个缓冲区中。缓冲区
//已经被填充了(尽管此例子中body缓冲区还有空间去填充更多数据),
int bytesRead = channel.read(buffers);
```

```java
//使用gather操作将多个缓冲区的数据组合并发送出去
body.clear( );
body.put("FOO".getBytes()).flip( ); // "FOO" as bytes
header.clear( );
header.putShort (TYPE_FILE).putLong (body.limit()).flip( );
long bytesWritten = channel.write (buffers);
````

使用得当的话，Scatter/Gather 会是一个极其强大的工具。它允许您委托操作系统来完成辛苦 活:将读取到的数据分开存放到多个存储桶(bucket)或者将不同的数据区块合并成一个整体。这 是一个巨大的成就，因为操作系统已经被高度优化来完成此类工作了。它节省了您来回移动数据的工作，也就避免了缓冲区拷贝和减少了您需要编写、调试的代码数量。既然您基本上通过提供数据 容器引用来组合数据，那么按照不同的组合构建多个缓冲区阵列引用，各种数据区块就可以以不同 的方式来组合了.

```java
package com.weimob.soa.amasscard.constant.nio;

import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.GatheringByteChannel;
import java.util.LinkedList;
import java.util.List;
import java.util.Random;

/**
 * demo gathering write using many buffers
 * 以Gather 写操作集合多个缓冲区数据，将其写入到文件
 * @author han.xue
 * @since 2018-06-08 16:34:34
 */
public class Marketing {

	private static final String DEMOGRAPHIC = "blahblah.txt";
	private static Random rand = new Random();

	private static String[] col1 = {
			"Aggregate", "Enable", "Leverage",
			"Facilitate", "Synergize", "Repurpose",
			"Strategize", "Reinvent", "Harness"
	};

	private static String[] col2 = {
			"cross-platform", "best-of-breed", "frictionless",
			"ubiquitous", "extensible", "compelling",
			"mission-critical", "collaborative", "integrated"
	};
	private static String[] col3 = {
			"methodologies", "infomediaries", "platforms",
			"schemas", "mindshare", "paradigms",
			"functionalities", "web services", "infrastructures"
	};

	private static String newline = System.getProperty("line.separator");

	public static void main(String[] args) throws Exception {
		int reps = 10;
		if (args.length > 0) {
			reps = Integer.parseInt(args[0]);
		}

		FileOutputStream fos = new FileOutputStream(DEMOGRAPHIC);
		GatheringByteChannel gatheringByteChannel = fos.getChannel();
		ByteBuffer[] bs = utterBS(reps);

		//Deliver the message to the waiting market
		while (gatheringByteChannel.write(bs) > 0) {
			// Empty body
			// Loop until write() returns zero
		}

		System.out.println("Mindshare paradigms synergized to " + DEMOGRAPHIC);

		fos.close();

	}


	private static ByteBuffer[] utterBS(int howMany) throws Exception {
		List<ByteBuffer> list = new LinkedList<>();
		for (int i = 0; i < howMany; i++) {
			list.add(pickRandom(col1, " "));
			list.add(pickRandom(col2, " "));
			list.add(pickRandom(col3, newline));
		}

		ByteBuffer[] bufs = new ByteBuffer[list.size()];
		list.toArray(bufs);
		return bufs;
	}

	// Pick one, make a buffer to hold it and the suffix, load it with
	// the byte equivalent of the strings (will not work properly for
	// non-Latin characters), then flip the loaded buffer so it's ready
	// to be drained
	private static ByteBuffer pickRandom(String[] strings, String suffix) throws Exception {
		String string = strings[rand.nextInt(strings.length)];
		int total = string.length() + suffix.length();

		ByteBuffer buf = ByteBuffer.allocate(total);
		buf.put(string.getBytes("UTF-8"));
		buf.put(suffix.getBytes("UTF-8"));
		buf.flip();
		return buf;
	}
}
```

## 文件通道(FileChannel)

![FileChannel](https://github.com/johnxue2013/docs/blob/master/images/3_7.png)

文件通道总是阻塞式的，因此不能被置于非阻塞模式。`FileChannel`对象是线程安全(thread-safe)的。多个进程可以在同一个实例上并发调用方法而 不会引起任何问题，不过并非所有的操作都是多线程的(multithreaded)。影响通道位置或者影响 文件大小的操作都是单线程的(single-threaded)。如果有一个线程已经在执行会影响通道位置或文 件大小的操作，那么其他尝试进行此类操作之一的线程必须等待。并发行为也会受到底层的操作系 统或文件系统影响。
