Log4j2
- 性能上是log4j 1.x和logback的18倍
- 像logback一样可以自动重新加载配置
- 像logback一样提供基于上下文数据的过滤器，

获取 root logger
```java
Logger logger = LogManager.getLog(LogManager.ROOT_LOGGER_NAME);

//更简单的方法
Logger logger = LogManager.getRootLogger();
```
其他Loggers可以通过使用`LogManager.getLogger`静态方法传入响应的日志名就可以获取

日志级别
TRACE
DEBUG
INFO
WARN
ERROR
FATAL

- Appender
可以理解为一个输出地，log4j支持输出到console,files, remote socket, servers,Apache Flume,JMS,remote UNIX syslog daemons和多种数据库，一个Logger可以有不止一个Appender。默认情况下子Logger会继承父Logger的Appender，也就是说子Logger会在父Logger的appender里输出，若设置子类`additivity=false`，则子Logger只会在自己的appender里输出，而不会在父Logger的appender里输出。


### Async Loggers
#### Asynchronous Loggers for Low-Latency Logging
- `异步日志`是log4j2中新增的功能，你可以选择将所有的logger都设置成异步的或者使用同步/异步混合模式。设置全部日志为异步的将带来更高的性能，而混合模式则更加的灵活

#### 是否使用异步logger的考量点
- 优点
  - 高吞吐
  - 更低的打日志响应时间
- 缺点
  - 错误处理：
    在异步日志如果在记录日志的过程中发生异常，则将更不容易将此异常通知到我们的应用程序。这个可以通过配置`ExceptionHandler`去缓解该问题，但仍然不能覆盖所有的场景。因此，如果记录日志是你的程序的一部分逻辑的话，我们推荐你使用同步的日志。
  - 在一些罕见的场景啊中，应该记录不可变的message。大部分时间你不需要担心这个问题。log4j会保证记录这个message时的数据和调用logger.debug("My Object is {}", myObject)一样，myObject就算在调用debug方法后发生修改也不会影响。但有一写异常情况，有些可变对象将无法保证。
  - 如果你的应用程序运行在一个CPU资源很稀缺的环境里,比如只有一个CPU且是单核的，此时开启另一个线程将不会得到更好的性能的改善。


  #### 设置所有的Loggers为异步
  > log4j-2.9或以上需要disruptor-3.3.4.jar或者更高的版本。Log4j-2.9之前的版本需要disruptor-3.0.0或者更好的版本，

  首先添加对应disruptor的jar到class path，然后设置系统属性`log4j2.contextSelector to org.apache.logging.log4j.core.async.AsyncLoggerContextSelector`
  By default, location is not passed to the I/O thread by asynchronous loggers. If one of your layouts or custom filters needs location information, you need to set "includeLocation=true" in the configuration of all relevant loggers, including the root logger.
  A configuration that does not require location might look like:

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!-- Don't forget to set system property
  -Dlog4j2.contextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
       to make all loggers asynchronous. -->
  <Configuration status="WARN">
    <Appenders>
      <!-- Async Loggers will auto-flush in batches, so switch off immediateFlush. -->
      <RandomAccessFile name="RandomAccessFile" fileName="async.log" immediateFlush="false" append="false">
        <PatternLayout>
          <Pattern>%d %p %c{1.} [%t] %m %ex%n</Pattern>
        </PatternLayout>
      </RandomAccessFile>
    </Appenders>
    <Loggers>
      <Root level="info" includeLocation="false">
        <AppenderRef ref="RandomAccessFile"/>
      </Root>
    </Loggers>
  </Configuration>
  ```
> 可以在主程序开头，加一句系统属性代码做到设置系统属性
```java
System.setProperty("Log4jContextSelector", "org.apache.logging.log4j.core.async.AsyncLoggerContextSelector");

```
