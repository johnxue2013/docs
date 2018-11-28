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
类库中包含一些`BlockingQueue`的实现，其中`LinkedBlockingQueue`和`ArrayBlockingQueue`是FIFO队列，与`LinkedList`和`ArrayList`相似,但是却拥有比同步List更好的并发性。`PrioriyBlockingQueue`是一个按优先级顺序排序的队列,当你不希望按照FIFO的顺序处理元素时，这个`PrioriyBlockingQueue`是非常有用的。正如其他排序的容器一样，`PrioriyBlockingQueue`可以比较元素本身的自然顺序（如果它们实现了`Comparable`），也可使用一个`Comparator`进行排序

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
      //标识位会被清除，调用如下方法重新设置中断状态
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
是一种`Synchronizer`，它可以延迟线程的进度直到线程到达终止状态。闭锁可以用来确保特定活动直到其他的活动完成后才发生。闭锁是一次性对象，一旦进入最终状态，就不能被重置了。`CountDownLatch`是一个灵活的闭锁实现

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
关卡类似闭锁，能够阻塞一组线程，直到某些事件发生。
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

## 第二部分
### 构建并发应用程序
大多数并发应用程序围绕执行**任务(task)** 进行管理的。任务就是抽象、离散的工作单元。  

任务是逻辑上的工作单元，线程是使用任务异步执行的机制。在java类库中，任务执行的首要抽象不是`Thread`，而是`Executor`，
```java
public void Executor {
  void executor(Rannable command);
}
```
`Executor`基于生产者-消费者模式。提交任务的执行者是生产者(产生待完成的工作单元)，执行任务的线程就是消费者(消耗掉这些工作单元)。

类库提供了一个灵活的线程池实现和一些有用的预设设置。可以通过`Executors`中的某个静态工厂方法来创建一个线程池：

```java
Executors.newFixedThreadPool
```
创建一个定长的线程池，`每当提交一个任务就创建一个线程`，直到达到池的最大长度，这时线程池会保持长度不再变化（如果一个线程由于非预期的Exception而结束，线程池会补充一个新的线程）。


```java
Executors.newCachedThreadPool
```
创建一个可缓存的线程池，如果当前线程池超过了处理的需要时，它可以灵活的回收空闲的线程，当需求增加时，它可以灵活的添加新的线程，而并不会对池的长度作任何限制。


```java
Executors.newSingleThreadExecutor
```
创建一个单线程化的executor，它只创建唯一的工作者线程来执行任务，如果这个线程异常结束，会有另一个取代她，executor会保证任依照务队列锁规定的顺序（FIFO,LIFO，优先级）执行。

```java
Executors.newScheduledThreadPool
```
创建一个定长的线程池，而且支持定时的以及周期性的任务执行，类似于Timer

`newFixedThreadPool`和`newCachedThreadPool`两个工厂方法返回通用目的的`ThreadPoolExecutor`实例。直接使用`ThreadPoolExecutor`也能创建更满足某些专有领域的executor


> 如果在足够长的时间内，任务到达的速度总是超过任务执行的速度，服务器仍然可能（只是更不易）耗尽内存，因为等待执行的Runnable队列会不断的增长，使用一个有限的工作队列，可以在Executor框架内部解决这个问题。

### Executor的生命周期
为了解决生命周期的问题，`ExecutorService`接口扩展了`Executor`，并添加了一些用于生命周期管理的方法。

生命周期共有三个状态: 运行(runnning)、关闭(shutting down)、终止(terminated)

在关闭后提交到`ExecutorService`中的任务，会被拒绝执行处理器(rejected execution handler)处理。`拒绝执行处理器`(拒绝执行处理器是ExecutorService的一种实现，ThreadPoolExecutor提供的)可能只是简单的放弃任务，也可能引起execute抛出一个未检查的`RejectedExecutionException`。所有任务完成后`ExecutorService`会转入`终止`状态。

`Executor`执行的任务的生命周期:`创建`、`提交`、`开始`、`完成`。任务的状态(尚未开始，运行中，完成)决定了get方法的行为。

有多种方法创建一个`Future`，因此可以将一个`Runnable`或者一个`Callable`提交给executor,然后得到一个`Future`。**也可以显式的为给你的`Runnable`或者`Callable`实例化一个FutureTask(FutureTask实现了Runnable，所以即可以提交给Executor来执行，又可以直接调用run方法运行)**

> 多线程情况下的执行效果受限于I/O和CPU

`Callable`和`Future`可以表述所有协同工作的任务之间的交互。

## CompletionService
使用ExecutorCompletionService 管理线程池处理任务的返回结果

在我们日常使用线程池的时候，经常会有需要获得线程处理结果的时候。此时我们通常有两种做法。
1. 使用并发容器将callable.call() 的返回Future存储起来。然后使用一个消费者线程去遍历这个并发容器，调用Future.isDone()去判断各个任务是否处理完毕。然后再处理响应的业务。

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.LinkedBlockingQueue;

public class ExecutorResultManager {

    public static void main(String[] args) {
        // 队列
        BlockingQueue<Future<String>> futures = new LinkedBlockingQueue<>();

        // 生产者
        new Thread() {
            @Override
            public void run() {
                ExecutorService pool = Executors.newCachedThreadPool();

                for (int i=0; i< 10; i++) {
                    int index = i;
                    Future<String> submit = pool.submit(new Callable<String>() {
                        @Override
                        public String call() throws Exception {
                            return "task done" + index;
                        }
                    });
                    try {
                        futures.put(submit);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();



        // 消费者
        new Thread() {
            @Override
            public void run() {
                while(true) {
                    for (Future<String> future : futures) {
                        if(future.isDone()) {
                            // 处理业务
                            // .............


                        };
                    }
                }
            }
        }.start();
    }

}
```

2. 使用jdk 自带线程池结果管理器：ExecutorCompletionService。它将BlockingQueue 和Executor 封装起来。然后使用ExecutorCompletionService.submit()方法提交任务。

submit 方法如下：

```java
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorCompletionService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class ExecutorCompletionServiceManager {

    public static void main(String[] args) {

        ExecutorCompletionService<String> service = new ExecutorCompletionService<String>(
                Executors.newCachedThreadPool());

        // 生产者
        new Thread() {
            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    int index = i;
                    service.submit(new Callable<String>() {
                        @Override
                        public String call() throws Exception {
                            return "task done" + index;
                        }
                    });
                }
            }
        }.start();


        // 消费者
        new Thread() {
            @Override
            public void run() {
                try {
                  while(true) {
                    //谁先完成，谁先返回
                    Future<String> take = service.take();
                    // do some thing........
                  }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }.start();

    }
}
```

> 使用多线程，首先要划分任务执行的边界，如从一个公司获得报价的过程和从其他公司获得的报价是过程无关的，所以获取单一报价是一个明显的任务边界，它允许并发的处理获得的报价。为此创建n个任务。

### invokeAll

ExecutorService的invokeAll方法有两种用法：

1. exec.invokeAll(tasks)

2. exec.invokeAll(tasks, timeout, unit)

其中tasks是任务集合，timeout是超时时间，unit是时间单位

两者都会堵塞，必须等待所有的任务执行完成后统一返回，一方面内存持有的时间长；另一方面响应性也有一定的影响，毕竟大家都喜欢看看刷刷的执行结果输出，而不是苦苦的等待；

但是方法二增加了超时时间控制，这里的超时时间是针对的所有tasks，而不是单个task的超时时间。如果超时，会取消没有执行完的所有任务，并抛出超时异常。相当于将每一个future的执行情况用一个list集合保存，当调用future.get()方法取值时和设置的timeout比较，是否超时。

InvokeAll方法处理一个任务的容器（collection），并返回一个Future的容器。两个容器具有相同的结构；

**这里提交的任务容器列表和返回的Future列表存在顺序对应的关系。**

invokeAll将Future添加到返回容器中，这样可以使用任务容器的迭代器，从而调用者可以将它表现的Callable与Future关联起来。

当所有任务都完成时、调用线程被中断时或者超过时限时，限时版本的invokeAll都会返回结果。超过时限后，任何尚未完成的任务都会被取消。

作为invokeAll的返回值，每个任务要么正常地完成，要么被取消。

```java

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Random;
import java.util.concurrent.*;

/**
 * @author han.xue
 * @since 2018-04-21 13:47:47
 */
public class InvokeAllThread {

	static ExecutorService executorService = Executors.newFixedThreadPool(5);

	/**
	 * 计算价格的任务
	 *
	 * @author hadoop
	 */
	private class QuoteTask implements Callable<BigDecimal> {
		public final double price;
		public final int num;

		public QuoteTask(double price, int num) {
			this.price = price;
			this.num = num;
		}

		@Override
		public BigDecimal call() throws Exception {
			Random r = new Random();
			long time = (r.nextInt(10) + 1) * 1000;
			Thread.sleep(time);

			BigDecimal d = BigDecimal.valueOf(price * num).setScale(2);
			System.out.println("耗时：" + time / 1000 + "s,单价是：" + price + ",人数是："
					+ num + "，总额是：" + d);
			return d;
		}
	}

	/**
	 * 在预定时间内请求获得旅游报价信息
	 *
	 * @return
	 */
	public void getRankedTravelQuotes() throws InterruptedException {
		List<QuoteTask> tasks = new ArrayList<QuoteTask>();
		// 模拟10个计算旅游报价的任务
		for (int i = 1; i <= 20; i++) {
			tasks.add(new QuoteTask(200, i));
		}

		/**
		 * 使用invokeAll方法批量提交限时任务任务 预期15s所有任务都执行完,没有执行完的任务会自动取消
		 *
		 */
		List<Future<BigDecimal>> futures = executorService.invokeAll(tasks, 15, TimeUnit.SECONDS);
		// 报价合计集合
		List<BigDecimal> totalPriceList = new ArrayList<BigDecimal>();

		Iterator<QuoteTask> taskIter = tasks.iterator();

		for (Future<BigDecimal> future : futures) {
			QuoteTask task = taskIter.next();
			try {
				totalPriceList.add(future.get());
			} catch (ExecutionException e) {
				// 返回计算失败的原因
				// totalPriceList.add(task.getFailureQuote(e.getCause()));
				totalPriceList.add(BigDecimal.valueOf(-1));
				System.out.println("任务执行异常,单价是" + task.price + "，人数是：" + task.num);
			} catch (CancellationException e) {
				// totalPriceList.add(task.getTimeoutQuote(e));
				totalPriceList.add(BigDecimal.ZERO);
				System.out.println("任务超时，取消计算,单价是" + task.price + "，人数是：" + task.num);
			}
		}
		for (BigDecimal bigDecimal : totalPriceList) {
			System.out.println(bigDecimal);
		}
		executorService.shutdown();
	}


	public static void main(String[] args) {
		try {
			InvokeAllThread it = new InvokeAllThread();
			it.getRankedTravelQuotes();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```
