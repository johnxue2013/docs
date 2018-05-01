## 并发编程实践 笔记
## 第一部分
### 可见性
`synchronized`不仅仅用于原子操作或者划定"临界区"，同步同样还有一个重要、微妙的方面：内存可见性。这样可以避免一个线程修改其他线程正在使用的对象的状态，而且希望确保当一个线程修改了对象的状态后，其他线程能够真正看到改变。也就是说为了保证跨线程写入的内存可见性，必须使用同步机制。

**加锁可以保证可见性与原子性；volatile变量只能保证可见性**

只有满足下了下面的所有标准后，你才能使用volatile变量:
- 写入变量时并不依赖变量当前的值；或者能够确保只有单一的线程修改变量的值
- 变量不需要与其他的状态变量共同参与不变约束
- 而且访问变量时，没有其他的原因需要加锁

### 发布(publishing)和逃逸(escape)

一个对象尚未准备好就将它发布，这种情况称为逃逸。this逃逸经常发生在构造函数中启动线程或者注册监听器时，如:
```java
public class ThisEscape {  
    public ThisEscape() {  
        new Thread(new EscapeRunnable()).start();  
        // ...  
    }  

    private class EscapeRunnable implements Runnable {  
        @Override  
        public void run() {  
            // 通过ThisEscape.this就可以引用外围类对象, 但是此时外围类对象可能还没有构造完成, 即发生了外围类的this引用的逃逸  
        }  
    }  
}
```

解决办法
```java
public class ThisEscape {  
    private Thread t;  
    public ThisEscape() {  
        t = new Thread(new EscapeRunnable());  
        // ...  
    }  

    public void init() {  
        t.start();  
    }  

    private class EscapeRunnable implements Runnable {  
        @Override  
        public void run() {  
            // 通过ThisEscape.this就可以引用外围类对象, 此时可以保证外围类对象已经构造完成  
        }  
    }  
}  
```

### 对象的不可变性
只有满足如下状态，一个对象才是不可变的
- 他的状态不能在创建后再被修改
- 所有域都是final类型；并且它被正确创建(创建期间没有发生this引用的逃逸)

### 阻塞队列
类库中包含一些`BlockingQueue`的实现，其中`LinkedBlockingQueue`和`ArrayBlockingQueue`是FIFO队列，与`LinkedList`和`ArrayList`相似,但是却拥有比同步List更好的并发性。`PrioriyBlockingQueue`是一个按优先级顺序排序的队列,当你不希望按照FIFO的顺序处理元素时，这个`PrioriyBlockingQueue`是非常有用的。正如其他排序的容易一样，`PrioriyBlockingQueue`可以比较元素本身的自然顺序（如果它们实现了`Comparable`），也可使用一个`Comparator`进行排序

最后一个`BlockingQueue`的实现是`SynchronousQueue`，它根本上不是一个真正的队列，因为它不会为队列元素维护任何存储空间。不过，它维护一个排队的线程清单,这些线程等待把元素加入(enquere)队列或者移出(dequeue)队列。因为`SynchronousQueue`没有存储能力，所以除非另一个线程已经准备好参与移交工作，否则`put`和`take`会一直 阻止。`SynchronousQueue`这类队列只在消费者充足的时候比较合适，他们总能为下一个任务做好准备

### 双端队列
java 6新增了两个容器类型`Deque`和`BlockingDeque`，他们分别拓展了`Queue`和`BlockingQueue`，`Deque`是一个双端队列，允许高效的在头和尾分别进行插入和移出，实现他们的是`ArrayDeque`和`LinkedBlockingDeque`。


### 线程中断
中断是一种**协作**机制。
当你在代码调用了一个会抛出`InterruptedException`的方法时，你自己的方法就成为了一个阻塞的方法，要为响应中断做好准备

- 传递InteruptedException
如果你能侥幸避开异常的话，这通常是明智的策略--只需要把`InterruptedException`传递给你的调用者。这可能包括不捕获`InteruptedException`，也可能是先捕获，然后对其中特定的活动进行简洁的清理后，再抛出。
- 恢复中断
有时候你不能抛出`InteruptedException`，比如你的代码是Runnable的一部分时。在这样的情况下，你必须捕获`InteruptedException`，并且，在当前线程中通过调用interrupt从中断中恢复，这样调用栈中更高的代码可以发现中断已经发生。如

```java
public class TaskRunnable implements Runnable {
  BlockingQueue<Task> queue;
  ...
  public void run() {
    try {
      processTask(queue.take());
    }catch(InterruptedException e) {
      //因为在抛出InterruptedException，因为抛出InterruptedException异常后,中断
      //标识位会被清除，调用如下方法重新设置中断状态。否则下方检测中断状态的代码将识别不到该线程被中断过
      Thread.currentThread().interrupt();
    }

    //检测中断状态
    boolean flag = Thread.currentThread().isInterrupted();
  }
}
```
### Synchronizer
它根据本身的状态调节线程的控制流。阻塞队列可以扮演一个`Synchronizer`的角色；其他类型的`Synchronizer`包括信号量(semaphore)、关卡(barrier)以及闭锁(latch)

- 闭锁(latch)
  是一种`Synchronizer`，它可以延迟线程的进度直到线程到达终止状态。闭锁可以用来确保特定活动直到其他的活动完成后才发生。闭锁是一次性对象，一旦进入最终状态，就不能被重置了。
  `CountDownLatch`是一个灵活的闭锁实现

- 信号量
  计数信号量(Counting Semaphore)用来控制能够同时访问某特定资源的活动的数量，或者同时执行某一给定操作的数量。计数信号量可以用来实现资源池或者给一个容器限定边界。

  ```java
  import java.util.Collections;
  import java.util.HashSet;
  import java.util.Set;
  import java.util.concurrent.Semaphore;

  /**
   * 使用Semaphore实现带边界的容器
   *
   * @author han.xue
   * @since 2018-04-18 18:11:11
   */
  public class BoundedHashSet<T> {
  	private final Set<T> set;
  	private final Semaphore semaphore;

  	public BoundedHashSet(int bound) {
  		this.set = Collections.synchronizedSet(new HashSet<T>());
  		semaphore = new Semaphore(bound);
  	}

  	public boolean add(T o) throws InterruptedException {
  		semaphore.acquire();
  		boolean wasAdded = false;
  		try {
  			wasAdded = set.add(o);
  			return wasAdded;
  		} finally {
  			//如果没添加进去，则释放信号量
  			if (!wasAdded) {
  				semaphore.release();
  			}
  		}
  	}

  	public boolean remove(Object o) {
  		boolean wasRemoved = set.remove(o);
  		if (wasRemoved) {
  			semaphore.release();
  		}
  		return wasRemoved;
  	}
  }
  ```
- 关卡（barrier）
  关卡类似闭锁，能够则赛一组线程，直到某些事件发生。
  ```java
  public class CyclicBarrierTest {

  	static CyclicBarrier c = new CyclicBarrier(2);

  	public static void main(String[] args) {
  		new Thread(new Runnable() {

  			@Override
  			public void run() {
  				try {
  					c.await();
  				} catch (Exception e) {

  				}
  				System.out.println(1);
  			}
  		}).start();

  		try {
  			c.await();
  		} catch (Exception e) {

  		}
  		System.out.println(2);
  	}
  }
  ```
  输出
  ```java
  2
  1
  ```
  或者
  ```java
  1
  2
  ```
  `CyclicBarrier`还提供一个更高级的构造函数`CyclicBarrier(int parties, Runnable barrierAction)``，用于在线程到达屏障时，优先执行`barrierAction`，方便处理更复杂的业务场景。代码如下：
  ```java
  public class CyclicBarrierTest2 {

  	static CyclicBarrier c = new CyclicBarrier(2, new A());

  	public static void main(String[] args) {
  		new Thread(new Runnable() {

  			@Override
  			public void run() {
  				try {
  					c.await();
  				} catch (Exception e) {

  				}
  				System.out.println(1);
  			}
  		}).start();

  		try {
  			c.await();
  		} catch (Exception e) {

  		}
  		System.out.println(2);
  	}

  	static class A implements Runnable {

  		@Override
  		public void run() {
  			System.out.println(3);
  		}

  	}

  }
  ```
  输出
    ```java
    3
    1
    2
    ```
    或者
    ```java
    3
    2
    1
    ```

    不同之处在于闭锁CountDownLatch计数器只能使用一次。而CyclicBarrier的计数器可以使用reset()重置。所以CyclicBarrier能处理更为复杂的业务场景，比如如果计算发生错误，可以重置计数器，并让线程们重新执行一次。

    CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得CyclicBarrier阻塞的线程数量。isBroken方法用来知道阻塞的线程是否被中断。比如以下代码执行完之后会返回true。  

    ```java
    import java.util.concurrent.BrokenBarrierException;
    import java.util.concurrent.CyclicBarrier;

    public class CyclicBarrierTest3 {

        static CyclicBarrier c = new CyclicBarrier(2);

        public static void main(String[] args) throws InterruptedException, BrokenBarrierException {
            Thread thread = new Thread(new Runnable() {

                @Override
                public void run() {
                    try {
                        c.await();
                    } catch (Exception e) {
                    }
                }
            });
            thread.start();
            thread.interrupt();
            try {
                c.await();
            } catch (Exception e) {
                System.out.println(c.isBroken());
            }
        }
    }
    ```

    ```java
    //获取此时可用的cpu个数
    int count = Runtime.getRuntime().availableProcessors();
    ```

    Exchanger是关卡的另一种形式，它是一种两步关卡，在关卡点会交换数据。当两方进行的活动不对等时，Exchanger是非常有用的.

    ```java

    import java.util.ArrayList;
    import java.util.List;
    import java.util.concurrent.Exchanger;

    /**
     * 生产者
     * @author han.xue
     * @since 2018-04-19 14:15:15
     */
    public class Producer implements Runnable {
    	private List<String> buffer;
    	private final Exchanger<List<String>> exchanger;

    	public Producer(Exchanger<List<String>> exchanger) {
    		this.buffer = new ArrayList<>();
    		this.exchanger = exchanger;
    	}

    	@Override
    	public void run() {
    		for(int i = 0; i < 10; i++) {
    			System.out.println("Producer cycle:" + (i + 1));

    			//生产数据
    			for(int j = 0; j < 10; j++) {
    				String message = "message " + ((i * 10) + j);
    				buffer.add(message);
    				System.out.println("producer message:" + message);
    			}

    			try {
    				buffer = exchanger.exchange(buffer);
    			}catch (Exception e) {
    				e.printStackTrace();
    			}

    			System.out.println("Producer buffer size: " + buffer.size());

    		}
    	}
    }
    ```

    ```java
    import java.util.ArrayList;
    import java.util.List;
    import java.util.concurrent.Exchanger;

    /**
     * 消费者
     * @author han.xue
     * @since 2018-04-19 14:26:26
     */
    public class Consumer implements Runnable{

    	private List<String> buffer;
    	private final Exchanger<List<String>> exchanger;

    	public Consumer(Exchanger<List<String>> exchanger) {
    		this.buffer = new ArrayList<>();
    		this.exchanger = exchanger;
    	}

    	@Override
    	public void run() {
    		for(int i = 0; i < 10; i++) {
    			System.out.println("consumer cycle:" + (i + 1));

    			//阻塞，等待另一个线程到达交换点
    			try {
    				buffer = exchanger.exchange(buffer);
    			}catch (Exception e) {
    				e.printStackTrace();
    			}

    			System.out.println("Consumer buffer size:" + buffer.size());

    			for(int j = 0; j < buffer.size(); j++) {
    				String message = buffer.get(j);
    				System.out.println("Consumer:" + message);
    				buffer.remove(j);
    			}
    		}
    	}
    }
    ```

    ```java
    import java.io.IOException;
    import java.util.List;
    import java.util.concurrent.Exchanger;

    /**
     * 客户端使用
     * @author han.xue
     * @since 2018-04-19 10:39:39
     */
    public class Client {
    	public static void main(String[] args) throws IOException {
    		Exchanger<List<String>> exchanger = new Exchanger<>();

    		Producer producer = new Producer(exchanger);
    		Consumer consumer = new Consumer(exchanger);

    		Thread producerThread = new Thread(producer);
    		Thread consumerThread = new Thread(consumer);
    		producerThread.start();
    		consumerThread.start();
    	}
    }
    ```

  此注释注释在方法上，可以将父类或者实现类上的注解，显示在此方法上
  ```java
  {@inheritDoc}
  ```
