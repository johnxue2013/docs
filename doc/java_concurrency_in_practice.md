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

## 第三部分
调用`interrupt`并不意味着必然停止目标线程正在进行的工作，它仅仅传递了请求中断的信息。

> 理解：interrupt方法不会真正中断一个正在运行的线程；它仅仅发出**中断请求**,线程自己会在下一个方便的时刻中断(这些时刻被称为**取消点(cancellation point)**，)。有一些方法对这样的请求很重视，比如wait、sleep和join方法。当它们接到中断请求时，会抛出一个异常，或者进入时中断状态就已经被设置了。

中断通常是实现取消任务最明智的选择。
> 通过violate变量设置标识位取消任务的实现，当与可阻塞的库函数同时使用时，可能并不能取消任务，因为组塞住不能去检查violate变量的值。

### 只执行一次的服务
如果一个方法需要处理一批任务，并在所有任务结束前不会返回，那么它可以通过使用私有的`Executors`来简化服务的生命周期管理。

如：以下代码中的checkMail方法同时在多个服务器上并行地检查新邮件。它创建一个私有的`Executors`，并向每一个主机提交任务：在这之后，当所有检查邮件的任务完成后，它会关闭`Executor`,并等待结束。

```java
boolean checkMail(Set<String> hosts, long timeout, TimeUnit unit) throws interruptedException {
  ExecutorService exec = Executors.newCachedThreadPool();
  final AtomicBoolean hasNewMail = new AtomicBoolean(false);

  try {
    for(final String host : hosts) {
      exec.executor(new Runnable() {
        public void run() {
          if(checkMail(host)) {
            hasNewMail.set(true);
          }
        }
      });
    }
  }finally {
    exec.shutdown();
    exec.awaitTermination(timeout, unit);
  }

  return hasNewMail.get();
}
```

### 未捕获的异常处理
可以使用`UncaughtExceptionHandler`工具，能够检测到线程因为未捕获的异常引起的"死亡"。

```java
/**
 * @author han.xue
 * @since 2018-04-20 16:12:12
 */
public class UserDao {
	public static void main(String[] args) {

		final Thread thread = new Thread(new Runnable() {
			@Override
			public void run() {
				System.out.println("run...");
				throw new RuntimeException("非法访问");
			}
		});

		thread.setUncaughtExceptionHandler(new ThreadExceptionHandler());
		thread.start();
	}

	private static class ThreadExceptionHandler implements Thread.UncaughtExceptionHandler {

		@Override
		public void uncaughtException(Thread t, Throwable e) {
			System.out.println("thread terminated with exception: " + t.getName());
			e.printStackTrace();
		}
	}
}
```

除此以外，我们还可以通过

```java
Thread.setDefaultUncaughtExceptionHandler(UncaughtExceptionHandler)
```
设置全局的默认异常处理机制。此外，ThreadGroup 也实现了 UncaughtExceptionHandler 接口，所以通过 ThreadGroup 还可以为一组线程设置默认的异常处理机制。

> 在线程池中设置UncaughtExceptionHandler。只有通过execute提交的任务，才能将它抛出的异常送交给异常的处理器；而通过submit提交的任务，抛出的异常。无论是否为受检查的，都被认为是任务返回状态的一部分。如果一个由submit提交的任务以异常结束，这个异常会被Future.get重抛出，包装在ExecutionException中.

## JVM关闭
可以通过Runtime.addShutdownHook注册尚未开始的线程

```java
Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
			@Override
			public void run() {
				System.out.println("退出hooker");
			}
		}));

//此行代码执行完毕，上面的钩子线程将得到执行
System.exit(-1);
```

> 在正常的关闭中JVM将启动所有已注册的shutdown hook,JVM不保证shutdown hook的执行顺序，将并发执行。在强行关闭中，JVM不会运行shutdown hook

shutdown hook可以用于服务或应用程序的清理。比如清理临时文件，或清除OS不能自动清理的资源

### 精灵线程
线程分为两种:普通线程和精灵线程。JVM启动的时候创建的所有线程，除了主线程以外，其他的都是精灵线程（比如垃圾回收期和其他类似的线程）。当一个新的线程创建时，新线程集成了创建它的线程的后台状态，所以默认情况下，任何主线程创建的线程都是普通线程。

> 精灵线程和普通线程的差别仅仅在于退出时会发生什么。当一个线程退出时，JVM会检查一个运行中线程的详细清单，如果仅剩下精灵线程，它会发起正常的退出。当JVM停止时，所有仍然存在的精灵线程都会被抛弃--不会执行finally块，也不会释放栈--JVM直接退出。

## 使用线程池

为了正确的定制线程池的大小，需要理解计算环境、资源预算和任务自身的特性。部署系统中安装了多少个CPU？多少内存？任务主要是计算、I/O还是一些混合操作？

对于计算密集型的任务，一个有N<sub>cpu</sbu>个处理器的系统通常通过使用 N<sub>cpu</sbu>个线程的线程池来获得最优的利用率(计算密集型的线程恰好在某时因为发生一个页错误或者因为其他原因而暂停，刚好有一个"额外"的线程，可以确保在这种情况下CPU周期不会中断工作)。

对于包含了I/O和其他阻塞操作的任务。不是所有的线程都会在所有的时间被调度，因此你需要一个更大的池。为了正确设置线程池的长度，可以不是十分精确的估算得出。
给出如下定义:
N<sub>cpu</sub> = CPU的数量
U<sub>cpu</sub> = 目标CPU使用率, 0 <=  U<sub>cpu</sub> <= 1
W/C = 等待时间与计算时间的比率
为了保持处理器达到期望的使用率，最优池的大小等于:
N<sub><thread/sub> = N<sub>cpu</sub> x U<sub>cpu</sub> x (1 + W/C)

> 可以通过
```java
int N_CPUS = Runtime.getRuntime.availableProcessors();
```
获取CPU数目

## 配置ThreadPoolExecutor
```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler)
```

- corePoolSize： 核心池大小是目标线程大小；线程池的实现试图维护池的大小，即时没有任务执行，池的大小也等于核心池的大小，并且直到工作队列满前，池都不会创建更多的线程。
- maximumPoolSize： 最大池的大小是可同时活动的线程数的上线。如果一个线程已经闲置的时间超过了存活时间，它将成为一个被回收的候选者，如果当前池大小超过了核心池大小，线程池将终止它。

> 如果新请求到达的频率超过了线程池能够处理它们的速度，请求将在队列中等候，而不是竞争CPU资源，随着队列的越来越大，仍然有耗尽资源的风险。

一个稳妥的资源管理策略是使用有限队列，比如`ArrayBlockingQueue`或者有限的`LinkedBlockingQueue`以及`PriorityBlockingQueue`。有界队列有助于避免资源耗尽的情况发生，但是当队列满了之后，新的任务怎么办？有很多的**饱和策略**可以处理这个问题。对于有界队列，队列的长度必须与池的长度一起调解。一个大队列加一个小池，可以控制对内存和CPU的使用，还可减少上下文切换，但是要接收潜在的吞吐量约束的开销.

### 饱和策略
当一个有限队列充满后，**饱和策略**开始起作用。ThreadPoolExecutor的饱和策略可通过调用`setRejectedExecutionHandler`来修改（如果任务提交到一个已经被关闭的Executor时，也会用到饱和策略）。JDK提供了几种不同的`RejectedExecutionHandler`实现，每一个都实现了不同的饱和策略:
- AbortPolicy:
默认的终止策略会引起executor抛出未检查的`RejectedExecutionException`；调用者可以捕获这个异常，然后编写能满足自己需求的处理代码
- CallerRunsPolicy:
既不会丢弃哪个任务，也不会抛出任何异常。它会把一些任务推回到调用者那里，以此减缓新任务流。它不会在线程池中执行最新的提交的任务，但是它会在一个调用了execute的线程中执行。
- DiscardPolicy:
默认放弃这个任务
- DiacardOldestPolicy:
该策略丢弃的任务，是本应该接下来就执行的任务，该策略还会尝试去重新提交新任务

### 线程工厂
无论何时，线程池需要创建一个线程，都要通过一个**线程工厂(thread factory)**。ThreadFactory接口只有一个方法
```java
public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}
```

**使用场景**
- 为线程池指明一个`UncaughtExceptionHandler`
- 实例化一个定制的Thread类的实例，或者只是希望给线程池一个更有意义的名称，来简化对线程转出和错误日志的解释。

以下为自定义工厂的例子

```java
import java.util.concurrent.ThreadFactory;

/**
 * @author han.xue
 * @since 2018-04-29 18:47:47
 */
public class MyThreadFactory implements ThreadFactory {

	private final String poolName;

	public MyThreadFactory(String poolName) {
		this.poolName = poolName;
	}

	@Override
	public Thread newThread(Runnable r) {
		return new MyAppThread(r, poolName);
	}
}

```

```java
import sun.rmi.runtime.Log;

import java.util.concurrent.atomic.AtomicInteger;
import java.util.logging.Level;

/**
 * @author han.xue
 * @since 2018-04-29 17:21:21
 */
public class MyAppThread extends Thread{
	public static final String DEFAULT_NAME = "MyAPPThread";
	private static volatile boolean debugLifecycle = false;
	//该变量当线程池创建重用该线程时，此值将变化
	private static final AtomicInteger created = new AtomicInteger();
	//该变量当线程池创建重用该线程时，此值将变化
	private static final AtomicInteger alive = new AtomicInteger();
	private static final Log log = Log.getLog("d", "s1", 1);

	public MyAppThread(Runnable runnable) {
		this(runnable, DEFAULT_NAME);
	}

	public MyAppThread(Runnable runnable, String name) {
		super(runnable, name + "-" + created.incrementAndGet());
		setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {

			@Override
			public void uncaughtException(Thread t, Throwable e) {
				log.log(Level.SEVERE, "UNCAUGHT in thread " + t.getName());
			}
		});
	}

	@Override
	public void run() {
		boolean debug = debugLifecycle;
		if(debug) {
			log.log(Level.ALL, "Created " + getName());
		}

		try {
			alive.incrementAndGet();
			super.run();
		}finally {
			alive.decrementAndGet();
			if(debug) {
				log.log(Level.ALL, "Exiting " + getName());
			}
		}
	}
}
```

### 扩展ThreadPoolExecutor
ThreadPoolExecutor的设计是可扩展的，它提供了几个"钩子"让子类去复写--`beforeExecute`、`afterExecute`和`terminate`。这些可以用来扩展ThreadPoolExecutor行为。


## 第三部分
## 死锁
### 顺序死锁
发生原因：两个线程试图通过不同的顺序获得多个相同的锁，如果所有线程通过相同的顺序获得锁，程序就不会出现锁顺序死锁的问题。
如:
```java
public class LeftRightDeadLock {
  private final Object left = new Object();
  private final Object right = new Object();

  public void leftRight() {
    synchronized(left) {
      synchronized(right) {
        doSomething();
      }
    }
  }

  public void rightLeft() {
    synchronized (right) {
      synchronized (left) {
        doSomethingElse();
      }
    }
  }

}

```

### 动态的锁顺序死锁
有时，并不是能一目了然的看清是否已经对锁有足够的控制，如把资金从一个账户转入另一个账户，在执行之前要获得`Account`对象的锁（为了简化操作，忽略账户余额不能为负数的情况）。
```java
public void transferMoney(Account fromAccount, Account toAccount, DollerAmount amount) throws InsufficientFundException {
  synchronized (toAccount) {
    synchronized(fromAccount) {
      if(fromAccount.getBalance.compareTo(amount) < 0) {
        throw new InsufficientFundException();
      }else {
        fromAccount.debit(amount);
        toAccount.credit(amount);
      }
    }
  }
}
```

如果两个线程同时调用`transferMoney`，一个从X向Y转账，一个从Y想X转账，那么就会发生死锁。
> 死锁原因：在偶发的时序中，A线程会获得X的锁，并等待Y锁，B线程会获得Y锁，并等待X锁。

解决方法：查看是否有嵌套，因为参数的顺序是超出我们控制的，可以通过制定锁的顺序，并在整个应用中获得锁都必须遵循这个顺序。

在指定对象顺序的时候，可以使用`System.identityHashCode(Object obj)`这样一种方式，它会返回obj的hashcode。在极少数情况下，两个对象具有相同的hashcode，此时引入"**加时赛(tie-breaking)**"锁，在获得两个Account锁之前，就要获得这个加时赛锁，这样就保证一次只有一个线程执行这个有风险的操作，从而减少死锁发生的可能性。
>如果经常休闲hash冲突，那么这个技术可能会成为并发的瓶颈，但`System.identityHashCode(Object obj)`的hash冲突出现频率很低，所以这个技术以最小的代价，换来最大的安全性。

```java
//指定锁的顺序来避免死锁
private static final Object tieLock = new Object();

public void transferMoney(final Account fromAccount, final toAccount, final DollarAmount amount) {
  class Helper {
    public void transfer () throws InsufficientFundException {
      if(fromAccount.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundException();
      }else {
        fromAccount.debit(amount);
        toAccount.credit(amount);
      }
    }
  }

  int fromHash = System.identityhashCode(fromAccount);

  int toHash = System.identityHashCode(toAccount);

  if(fromHash < toHash) {
    synchronized(fromAccount) {
      synchronized (toAccount) {
        new Helper().transfer();
      }
    }
  }else if(fromHash > toHash) {
    synchronized (toAccount) {
      synchronized(fromAccount) {
        new Helper().transfer();
      }
    }
  }else {
    synchronized(tieLock) {
      synchronized (fromAccount) {
        synchronized (toAccount) {
          new Helper.transfer();
        }
      }
    }
  }
}
```

### 开放调用
当调用的方法不需要持有锁时，这被称为"开放调用(open call)"

### 避免和诊断死锁
如果过必须获得多个锁，那么锁的顺序必须是你设计工作的一部分：尽量减少潜在锁之间的交互数量，遵守并文档化该锁顺序协议，这些缺一不可。

- 使用显式的可超时的锁
使用每个显式的Lock类中定时的tryLock特性，显式锁可以定时timeout时间，在规定实时间没有获得锁就返回失败。

活锁(live lock)是线程中活跃度失败的另一种形式，尽管没有被阻塞，线程却仍然不能继续，因为它不断重试相同的操作，却总是失败。

## 性能和可伸缩性
### 减少锁的竞争
- 减少持有锁的时间
- 减少请求锁的频率
- 或者用协调机制取代独占锁，从而允许更强的并发性  

#### 缩小锁的范围(快进快出)
减小竞争发生的可能性的有效方式是尽可能缩短把持锁的时间。可以通过把通过与锁无关的代码溢出synchronized块来实现。
#### 减小锁粒度
减小持有锁的时间比例的另外一种方式是**让线程减少调用它的频率**（因此减小发生竞争的可能性），这可以通过**分拆锁**(lock split)和**分离锁**(lock striping)来实现。即采用互相独立的锁，守卫多个独立的状态变量。
> 这些技术减少了锁发生时的粒度，潜在实现了更好的可伸缩性--但是使用更多的锁同样会增加死锁是风险

```java
//应当拆分锁的代码
public class ServerStatus {
  public final Set<String> users;
  public final Set<String> queries;

  ...

  public synchronized void addUser(String u) {
    users.addUser(u);
  }

  public synchronized void addQuery(String q) {
    queries.add(q);
  }

  public synchronized void removeUser(String u) {
    users.remove(u);
  }

  public synchronized void removeQuery(String q) {
    queries.removeQuery(q)
  }
}
```

```java
//使用分拆锁重构ServerStatus
public class ServerStatus {
  public final Set<String> users;
  public final Set<String> queries;


  public void addUser(String u) {
    synchronized(users) {
      users.addUser(u);
    }
  }

  public void addQuery(String q) {
    synchronized(queries) {
      queries.add(q);
    }
  }

  //使用类似技术对remove进行重构
}
```

### 分离锁
把一个竞争激烈的锁分拆成两个，很可能形成两个竞争激烈的锁（尽管这样可以使得两个线程并发执行，从而对伸缩性有一些小的改进）。单这仍然不能大幅的提高多个处理器在同一系统中并发性
> 上述的ServerStatus类没有提供明显的机会进行进一步分拆

**分拆锁** 有时候可以被扩展，分成可大可小的加锁的块集合，并且它们归属于相互独立的对象，这样的情况就是**分离锁**

> ConcurrentHashMap是使用的分离锁

分离锁的一个负面作用是：对容器加锁，进行独占访问更加困难，并且更加昂贵了。

```java
public class StripedMap {
  //同步策略:buckets[n]由locks[n%N_LOCKS]保护
  private static final int N_LOCKS = 16;
  private static final Node[] buckets[];
  private final Object[] locks;

  private static class Node {};

  public StripedMap(int numBuckets) {
    buckets = new Node[numBuckets];
    locks = new Object[N_LOCKS];
    for(int i = 0; i < N_NOCKS; i++)
      locks[i] = new Object();
  }

  private final int hash(Object key) {
    return Math.abs(key.hashcode() % buckets.length);
  }

  public Object get(Object key) {
    int hash = hash(key);
    synchronized (locks[hash % N_LOCKS]) {
      for(Node m = buckets[hash]; m != null; m = m.next()) {
        if(m.key.equals(key)) {
          return m.value;
        }
      }
      return null;
    }
  }

  /*这个方法不是原子操作，不排除当StripedMap为空时，其他线程正并发的向其加入元素；使操作原子化需要一次获得所有锁*/
  public void clear() {
    for(int i = 0; i < buckets.length; i++) {
        synchronized (locks[i % N_LOCKS]) {
          buckets[i] = null;
        }
    }
  }
}
```
## 测试并发程序
并发测试基本分为两类: 安全性的和活跃度

与活跃度相关的是性能测试。性能可以通过很多方式来测量，其中包括：
- 吞吐量:在一个并发任务里，已经完成任务所占的比例。
- 响应性:从请求到完成一些动作之间的延迟(也被称作**等待时间**)
- 可伸缩性:增加更多的资源(通常是CPU)，就能提高(或者缓解短缺)吞吐量

## 第四部分 高级主题
java5.0之前，用于调节共享对象访问的机制只有`synchronized`和`volatile`。java5.0提供了新的选择：`ReentrantLock`。`ReentrantLock`并不是作为内部锁机制的替代，而是当内部锁被证明受到局限时，提供可选择的高级特性。

### Lock和ReentrantLock
`Lock`接口定义了一些抽象的锁操作。与内部加锁机制不同，Lock提供了无条件的、可轮询的、定时的、可中断的锁获取操作，所有加锁和解锁的方法都是显式的。

Lock接口定义如下
```java
public interface Lock {
  void lock();
  void lockInterruptibly() throws InterruptedException;
  boolean tryLock();
  boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
  void unlock();
  Condition newCondition();
}
```

使用Lock的通用规范
```java
Lock lock = new ReentrantLock();
...

lock.lock();
try {
  //更新对象状态
  //捕获异常，必要时恢复到原来的不变约束
}finally {
  lock.unlock();
}
```
> 锁必须在finally块中释放，忘记使用finally释放Lock是一个定时炸弹。将很难追踪到错误发生点。

`ReentrantLock`提供的**竞争上的性能**要与远远优于内部锁（synchronized）

## 公平性
`ReentrantLock`构造函数提供了两种公平性选择：创建**非公平锁**(默认)和**公平锁**

线程按照顺序请求获得公平所，然而一个非公平锁允许“闯入”：当请求这样的锁时，如果锁状态变为可用，线程的请求可以在等待的线程队列中向前跳跃，获得该锁。

当发生加锁的时候，公平会因为挂起和重新开始线程带来巨大的性能开销。**在多数情况下，非公平锁性能远大于公平锁**，但大体按照如下标准衡量使用公平锁还是非公平锁：

- **在激烈竞争的情况下，非公平锁比公平锁性能好**的原因之一是：挂起和重新开始，与它真正开始运行，两者之间会产生严重的延迟。假设线程A持有一个锁，线程B请求该锁，因为此时锁正在使用，所以B会被挂起。当A释放锁后，B重新开始。于此同时，如果C请求这个锁，那么C得到很好的机会获得这个锁，使用它，并且甚至可能B在换新前就已经释放该锁了。在这样的情况下，各方面都获得了成功：B并没有比其他任何线程晚得到锁，C更早的得到了锁，吞吐量得到了改进。

- **当持有锁的时间相对较长，或者请求锁的平均时间间隔比较长**，那么使用公平锁比较好。这种情况下，闯入锁带来的优势情况--在线程正在换新的过程中，还没得到锁，不太容易出现。

## `synchronized`和`ReentrantLock`选择
除非对`ReentrantLock`的可伸缩性有切实的需要，否则就性能的原因选择`ReentrantLock`而不是`synchronized`，这不是正确的决定。

## 读写锁(read-write lock)
读写锁允许一个资源能够被多个读者访问，或者被一个写着访问，两个不能同时进行。

```java
//读写锁接口定义
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading.
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing.
     */
    Lock writeLock();
}
```

读写锁的设计是用来进行性能改进的，使得特定情况下能够有更好的并发性。在实践中，当多处理器系统，频繁的访问主要为读取数据结构的时候，读-写锁能够改进性能；在其他情况下运行的情况比独占的锁要稍差一些，因为它更大的复杂性。

`ReentrantReadWriteLock`实现了`ReadWriteLock`为读写锁提供了可重进入的加锁语义。`ReentrantReadWriteLock`能够被构造成非公平（默认）或者是公平的。由写者降级为读者是允许的，由读者升级为写者是不允许的（会导致死锁）
```java
//使用读写锁包装的map
public class ReadWriteMap<K, V> {
  private final Map<K, V> map;
  private final ReadWriteLock lock = new ReentrantReadWriteLock();
  private Lock r = lock.readLock();
  private Lock w = lock.writeLock();

  public ReadWriteMap(Map<K, V> map) {
    this.map = map;
  }

  public V put(K key, V value) {
    w.lock();
    try {
      return map.put(key, value);
    }finally {
      w.unLock();
    }
  }

  //remove(),putAll(),clear()也完全类似

  public V get(Object key) {
    r.lock();
    try {
      return map.get(key);
    }finally {
      r.unlock();
    }
  }
}
```

## 构建自定义的同步工具
### 管理状态依赖性
一个可阻塞的状态依赖活动表现为如下所示的形式
```java
//伪代码
 void blockingAction() throws InterruptedException {
   acquire lock on object state
   while(precondition does not hold) {
     release lock
     wait until precondition might hold
     optionally fail if interrupted or might timeout expires
     reacquire lock
   }

   perform action
 }
```
> 锁是在操作过程中被释放与重获的，这也让这种加锁的模式略显与众不同。组成先验条件的状态变量必须由UI想的锁保护起来，这样他们能在测试先验条件的过程中保持不变。如果先验条件尚未满足，就必须释放锁，让其它线程可以修改对象的状态--否则，先验条件就永远无法变成真了。再一次验证先验条件之前，必须要重新获得锁。

如果依赖于状态的操作在处理先验条件不满足时，可以抛出异常或者返回错误状态（把问题留给调用者），也可以保持阻塞知道对象状态转入正确的状态。

## 使用条件队列
**条件队列**可以让一组线程,称作`等待集`,以某种方式等待相关条件变成真。
> 不同与传统的队列，他们的元素是数据项；条件队列的元素是等待相关条件的线程

就像每个Java对象都能当作锁一样，每个对象也能当作条件队列，Object中的wait、notify、notifyAll方法构成了内部条件队列的API。Object的wait会自动释放锁，并请求OS挂起当前线程，让其他线程获得锁进而修改对象状态。当它被唤醒时，它会在返回前重新获得锁。

> 注意：如果不能通过“轮询和休眠”完成的事情，用条件队列也无法完成，但是条件队列使得表达和管理状态的依赖性变得更加简单和高效。

```java
/**
 * 有限缓存基类
 * @author han.xue
 * @since 2018-05-05 20:20:20
 */
public abstract class BaseBundedBuffer<V> {
	private final V[] buf;
	private int tail;
	private int head;
	private int count;

	public BaseBundedBuffer(int capacity) {
		this.buf = (V[]) new Object[capacity];
	}

	protected synchronized final void doPut(V v) {
		buf[tail] = v;
		if(++tail == buf.length) {
			tail = 0;
		}
		++count;
	}

	protected synchronized final V doTake() {
		V v = buf[head];
		buf[head] = null;
		if(++head == buf.length) {
			head = 0;
		}
		--count;
		return v;
	}

	protected synchronized final boolean isFull() {
		return count == buf.length;
	}

	public synchronized final boolean isEmpty() {
		return count == 0;
	}
}



/**
 * 有限缓存使用条件队列实现跟高效
 *
 * @author han.xue
 * @since 2018-05-05 22:24:24
 */
public class BoundBuffer<V> extends BaseBundedBuffer<V> {
  //条件谓词: not-full (!isFull())
  //条件谓词: not-empty(!isEmpty())

	public BoundBuffer(int capacity) {
		super(capacity);
	}


	public synchronized void put(V v) throws InterruptedException {
		while (isFull()) {
			//阻塞，知道未满
			wait();
		}

		doPut(v);
		notifyAll();
	}

	public synchronized V take() throws InterruptedException {
		while (isEmpty()) {
			//阻塞，直到不为空
			wait();
		}

		V v = doTake();
		notifyAll();
		return v;
	}
}
```

> wait会在返回前重新请求锁，一个从wait方法中唤醒的线程，在重新请求锁的过程中没有任何特殊的优先级，它会像任何其他尝试进入synchronized块的线程一样去争夺锁。

当使用条件等待时(Object.wait或者Condition.wait)要严格遵循以下几点:
- 永远设置一个条件谓词（即一些对象状态的验证），线程执行前 必须满足它
- 永远在在调用wait验证条件谓词，并且从wait中返回后再次验证
- 即永远在循环中调用wait
- 确保构成条件谓词的状态变量被锁保护，而这个锁真实与条件队列相关联的
- 当调用wait、notify或者notifyAll时，要持有与条件队列相关的锁；并且在在验证条件谓词之后、开始执行被保护的逻辑之前，不要释放锁。

man许上述条件的通用代码为:
```java
void stateDependentMethod() throws InterruptedException{
  //条件谓词必须被锁守护
  synchronized(lock) {
    while(!conditionPredicate()) {
      lock.wait();
    }
    //现在，对象处于期望的状态中，可以进行后续的逻辑处理
  }
}
```

> 在条件队列中有两个通知方法`notify`和`notifyAll`。由于无论调用哪个方法时都必须持有条件队列对象的锁，这导致等待线程此时不能重新获得锁，无法从wait返回，因此通知线程应该尽快释放锁，以确保等待线程尽可能快的解除阻塞。

### 显式的Condition对象
一个`Condition`和一个单独的`Lock`相关联，就像条件队列和单独的内部锁相关联一样：调用与Condition相关联的Lock.newCondition方法，可以创建一个Condition。
```java
//Condition接口定义
public interface Condition {
  void await() throws InterruptedException;
  void awaitUninterruptibly();
  long awaitNanos(long nanosTimeout) throws InterruptedException;
  boolean await(long time, TimeUnit unit) throws InterruptedException;
  boolean awaitUntil(Date deadline) throws InterruptedException;
  void signal();
  void signalAll();
}
```

不同于内部队列，你可以让每个Lock都有任意数量的Condition对象。Condition对象集成了与之相关的锁的公平性：如果锁时公平的锁，线程会按照FIFO的顺序从Condition.await中被释放。

> 注意：Condition继承了Object，这也意味着它也有wait、notifyAll和notify方法。wait和notify在Condition中对等的是await、signal和signalAll，一定要确保使用了正确版本的await和signal

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 有限缓存使用显示的条件变量
 * @author han.xue
 * @since 2018-05-06 12:50:50
 */
public class ConditionBoundBuffer<T> {
	protected final Lock lock = new ReentrantLock();
	//条件谓词notFull (count < items.length)
	private final Condition notFull = lock.newCondition();
	//条件谓词notEmpty(count > 0)
	private final Condition notEmpty = lock.newCondition();
	private final T[] items;

	public ConditionBoundBuffer(int capacty) {
		this.items = (T[]) new Object[capacty];
	}

	private int tail, head, count;

	//阻塞直到:notFull
	public void put(T t) throws InterruptedException {
		lock.lock();
		try {
			while (count == items.length) {
				notFull.await();
			}
			items[tail] = t;
			if(++tail == items.length) {
				tail = 0;
			}
			++count;
			notEmpty.signal();
		}finally {
			lock.unlock();
		}
	}

	//阻塞，直到notEmpty
	public T take() throws InterruptedException{
		lock.lock();
		try {
			while (count == 0) {
				notEmpty.await();
			}
			T t = items[head];
			if(++head == items.length) {
				head = 0;
			}
			--count;
			return t;
		}finally {
			lock.unlock();
		}
	}
}
```

### 剖析Synchronized
Semaphore和ReentrantLock有很多相同点，他们都使用到了一个共同的基类，`AbstractQueuedSynchronizer`（即AQS）。
> 基于AQS能够简单且高效的构造出应用广泛的大量的Synchronizer。如`CountDownLatch`、`ReentrantReadWriteLock`、`SynchronousQueue`和`FutureTask`

## 原子变量和非阻塞同步机制
### 比较并交换（CAS）
CAS有三个操作数，内存位置V，旧的预期值和新值B。当且仅当V符合旧预期值A时，CAS用新值B原子操作的更新V的值，否则什么都不做，无论是否更新了旧值，都会返回V的真实值（即返回更新之前的值）
> CAS是一种乐观锁技术

```java
/**
 * 模拟CAS操作（并非CAS的真实的实现）
 * @author han.xue
 * @since 2018-05-06 19:27:27
 */
public class SimulatCAS {
	private int value;

	public synchronized int get() {
		return value;
	}

	public synchronized int compareAndSwap(int exepectValue, int newValue) {
		int oldValue = value;
		if(value == exepectValue) {
			value = newValue;
		}
		return oldValue;
	}

	public synchronized boolean compareAndSet(int expectValue, int newValue) {
		return expectValue == compareAndSwap(expectValue, newValue);
	}
}
```

```java
/**
 * 使用CAS实现非阻塞的计数器
 *
 * @author han.xue
 * @since 2018-05-06 19:38:38
 */
public class CASCounter {
	private SimulatCAS value;

	public int getValue() {
		return value.get();
	}

	public int increment() {
		int v;
		do {
			v = value.get();
			//如果此线程中的v和内存中的实际值不相等，则一直更新，直到相等
		} while (v != value.compareAndSwap(v, v + 1));
		return v + 1;
	}
}
```
java提供了AtomicXXX对数字提供原子化操作和`AtomicReference<T>`对对象的原子化操作

#### AtomicReference
An object reference that may be updated atomically.The AtomicReference class provides reference objects that may be read and written atomically, so when multiple threads try to reach them at the same time, only one will be able to do so.
提供方法如下

- `compareAndSet(V expect, V update)`：Atomically sets the value to the given updated value if the current value == the expected value.
- `getAndSet(V newValue)`：Atomically sets to the given value and returns the old value.
- `lazySet(V newValue)`：Eventually sets to the given value.
- `set(V newValue)`：Sets to the given value.
- `get()`：Gets the current value.

假设有一个类 Person，定义如下：
```java
class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String toString() {
        return "[name: " + this.name + ", age: " + this.age + "]";
    }
}
```

如果使用普通的对象引用，在多线程情况下进行对象的更新可能会导致不一致性。例如：
一个对象的初始状态为 name=Tom, age = 18。
在 线程1 中将 name 修改为 Tom1，age + 1。
在 线程2 中将 name 修改为 Tom2，age + 2。

我们认为只会产生两种结果：

若 线程1 先执行，线程2 后执行，则中间状态为 name=Tom1, age = 19，结果状态为 name=Tom2, age = 21
若 线程2 先执行，线程1 后执行，则中间状态为 name=Tom2, age = 20，结果状态为 name=Tom1, age = 21

```java
// 普通引用
private static Person person;

public static void main(String[] args) throws InterruptedException {
    person = new Person("Tom", 18);

    System.out.println("Person is " + person.toString());

    Thread t1 = new Thread(new Task1());
    Thread t2 = new Thread(new Task2());

    t1.start();
    t2.start();

    t1.join();
    t2.join();

    System.out.println("Now Person is " + person.toString());
}

static class Task1 implements Runnable {
    public void run() {
        person.setAge(person.getAge() + 1);
        person.setName("Tom1");

        System.out.println("Thread1 Values "
                + person.toString());
    }
}

static class Task2 implements Runnable {
    public void run() {
        person.setAge(person.getAge() + 2);
        person.setName("Tom2");

        System.out.println("Thread2 Values "
                + person.toString());
    }
}
```

但是可能的输出如下：
Person is [name: Tom, age: 18]
Thread2 Values [name: Tom1, age: 21]
Thread1 Values [name: Tom1, age: 21]
Now Person is [name: Tom1, age: 21]
上述结果表明并发情况下可能导致错误结果.

如果使用原子性对象引用，在多线程情况下进行对象的更新可以确保一致性。例如：

```java
// 普通引用
private static Person person;
// 原子性引用
private static AtomicReference<Person> aRperson;

public static void main(String[] args) throws InterruptedException {
    person = new Person("Tom", 18);
    aRperson = new AtomicReference<Person>(person);

    System.out.println("Atomic Person is " + aRperson.get().toString());

    Thread t1 = new Thread(new Task1());
    Thread t2 = new Thread(new Task2());

    t1.start();
    t2.start();

    t1.join();
    t2.join();

    System.out.println("Now Atomic Person is " + aRperson.get().toString());
}

static class Task1 implements Runnable {
    public void run() {
        aRperson.getAndSet(new Person("Tom1", aRperson.get().getAge() + 1));

        System.out.println("Thread1 Atomic References "
                + aRperson.get().toString());
    }
}

static class Task2 implements Runnable {
    public void run() {
        aRperson.getAndSet(new Person("Tom2", aRperson.get().getAge() + 2));

        System.out.println("Thread2 Atomic References "
                + aRperson.get().toString());
    }
}
```
可能结果如下
Atomic Person is [name: Tom, age: 18]
Thread1 Atomic References [name: Tom1, age: 19]
Thread2 Atomic References [name: Tom2, age: 21]
Now Atomic Person is [name: Tom2, age: 21]

### ABA问题
更新一对值即更新引用和版本号，而不是仅仅更新引用。
> 使用类库提供的`AtomicStampedReference`（以及它同系的`AtomicMarkableReference`）

## java存储模型（JMM）

### 不安全的发布
在缺少happen-before关系的情况下会导致不正确的发布
```java
//不安全的惰性初始化
public class UnsafeInitialzation {
  private static Resource resource;

  public static Resource getInstance() {
    if(null == resource) {
      resource = new Resource();
    }
    return resource;
  }
}
```

假设第一次调用getInstance的是线程A，它会看到resource是null，接着初始化一个新的Resource，然后设置resource引用这几个新实例。随后线程B调用getInstance，它可能看到resource已经有了一个非空的值，于是就是用这个已经创建的Resource。这看起来无害，但线程A写入resource与线程B读取resource之间不是按照happen-before进行排序的，发布对象时存在数据竞争，因此V并不能保证就可以看到Resource的正确状态。
> 除了不可变对象以外，使用被另一个线程初始化的对象，是不安全的，除非对象的发布是happen-before与对象的消费线程使用它。

```java
//线程安全的惰性初始化
public class UnsafeInitialzation {
  private static Resource resource;

  public synchronized static Resource getInstance() {
    if(null == resource) {
      resource = new Resource();
    }
    return resource;
  }
}
```
或者主动初始化
```java
//线程安全的惰性初始化
public class UnsafeInitialzation {
  private static Resource resource = new Resource();

  public  static Resource getInstance() {
    return resource;
  }
}
```

或者使用一个专门用来初始化Resource的类。JVM将ResourceHolder的初始化被延迟到真正使用它的时刻
```java
//线程安全的惰性初始化
public class UnsafeInitialzation {
	private static class ResourceHolder {
		public static Resource resource = new Resource();
	}
  private static Resource resource = new Resource();

  public  static Resource getInstance() {
    return ResourceHolder.resource;
  }
}
```


