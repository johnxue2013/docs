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

shutdown hook可以用于服务或应用程序的清理。比如清理临时文件，或清楚OS不能自动清理的资源

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

一个稳妥的资源管理策略是使用有限队列，比如`ArrayBlockingQueue`或者有限的`LinkedBlockingQueue`以及`PriorityBlockingQueue`。有界队列有助于避免资源耗尽的情况发生，但是当队列满了之后，新的任务怎么办？有很多的**饱和策略**可以处理这个问题。对于有界队列，队列的长度必须与池的长度一起调解。一个大对内加一个小池，可以控制对内存和CPU的使用，还可减少上下文切换，但是要接收潜在的吞吐量约束的开销.

### 饱和策略
当一个有限队列充满后，**饱和策略**开始起作用。ThreadPoolExecutor的饱和策略可通过调用`setRejectedExecutionHandler`来修改（如果任务提交到一个一进来被关闭的Executor时，也会用到饱和策略）。JDK提供了几种不同的`RejectedExecutionHandler`实现，每一个都实现了不同的饱和策略:
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
