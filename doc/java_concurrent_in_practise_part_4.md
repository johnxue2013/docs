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


