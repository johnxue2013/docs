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
