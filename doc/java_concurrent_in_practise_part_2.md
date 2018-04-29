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

类库提供了一个领会的线程池实现和一些有用的预设设置。可以通过`Executors`中的某个静态工厂方法来创建一个线程池：

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

`Executor`执行的任务的生命周期:`创建`、`提交`、`开始`、`完成`。任务的状态(尚未开始，运行中，完成)觉得了get方法的行为。

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
