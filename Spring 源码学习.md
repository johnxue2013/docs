# Spring源码学习

## bean初始化的一个大体流程
首先入口肯定是在org.springframework.context.support.AbstractApplicationContext#refresh() -> finishBeanFactoryInitialization() -> beanFactory.preInstantiateSingletons() -> getBean() -> doGetBean() -> createBean() -> doCreateBean() ->populateBean()完成bean的初始化

## `BeanDefinition`和`BeanFactoryPostProcessor`、`BeanPostProcessor`
### BeanDefinition
`BeanDefinition`是Spring对bean的一个抽象，比如bean的作用域，bean的注入模型，bean是否懒加载等等信息，这些信息是`Class`对象无法抽象出来的，因此Spring没有使用Class而是使用了`BeanDefinition`

### `BeanFactoryPostProcessor`
接口的定义如下:
```java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;

/**
 * Allows for custom modification of an application context's bean definitions,
 * adapting the bean property values of the context's underlying bean factory.
 *
 * <p>Application contexts can auto-detect BeanFactoryPostProcessor beans in
 * their bean definitions and apply them before any other beans get created.
 *
 * <p>Useful for custom config files targeted at system administrators that
 * override bean properties configured in the application context.
 *
 * <p>See PropertyResourceConfigurer and its concrete implementations
 * for out-of-the-box solutions that address such configuration needs.
 *
 * <p>A BeanFactoryPostProcessor may interact with and modify bean
 * definitions, but never bean instances. Doing so may cause premature bean
 * instantiation, violating the container and causing unintended side-effects.
 * If bean instance interaction is required, consider implementing
 * {@link BeanPostProcessor} instead.
 *
 * @author Juergen Hoeller
 * @since 06.07.2003
 * @see BeanPostProcessor
 * @see PropertyResourceConfigurer
 */
@FunctionalInterface
public interface BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```
Spring会先执行Spring内部提供的实现，然后执行开发者的实现类（如果提供了的话），在该实现中，开发者可以修改bean的一些属性。

实例化(instance)：是对象创建的过程。比如使用构造方法new对象，为对象在内存中分配空间。

初始化(initialization)：是为对象中的属性赋值的过程。


### `BeanPostProcessor`
spring提供的bean的后置处理器，允许开发者修改spring bean工厂创建出来的bean的信息(如属性等)。接口定义如下:
```java
public interface BeanPostProcessor {

	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```  
spring自身的很多功能也是通过这个接口实现的。如XXXAware接口的处理逻辑`ApplicationContextAwareProcessor`就是通过实现`BeanPostProcessor`接口实现的

## `FactoryBean`

```java
public interface FactoryBean<T> {
	@Nullable
	T getObject() throws Exception;

	@Nullable
	Class<?> getObjectType();


	default boolean isSingleton() {
		return true;
	}
```
Interface to be implemented by objects used within a {@link BeanFactory} which are themselves factories for individual objects. If a bean implements this interface, it is used as a factory for an object to expose, not directly as a bean instance that will be exposed itself.

在实际应用中如mybatis和spring的整合时，会进行如下配置
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
</bean>
```
`SqlSessionFactoryBean`这个类其实就实现了`org.springframework.beans.factory.FactoryBean`.这就向容器中注册了一个SqlSessionFactoryBean。从类名我们就能知道这是一个FactoryBean。当我们单独使用Mybatis时，需要创建一个SqlSessionFactory，然而当MyBatis和Spring整合时，却需要一个SqlSessionFactoryBean，所以我们可以猜测，是不是SqlSessionFactoryBean通过FactoryBean的特殊性，向Spring容器中注册了一个SqlSessionFactory。查看SqlSessionFactoryBean的源代码发现，它果然实现了FactoryBean接口，并且重写了getObejct方法，通过getObject()方法向容器中注册了一个SqlSessionFactory。

## Spring的Environment
`Environment`是spring提供的一个接口,提供了对一些环境配置变量提供了统一的访问方式。类的继承关系如下所示
![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/StandardEnvironment.png)

其中比较重要的是`AbstractEnvironment`类,该类提供了resolvePlaceholders方法，可以解析配置中的占位符${}并提供了默认值的功能,如${name:zhangsan},如果启动或者配置文件中没有提供name变量的值，那么此方法会将zhangsan赋值给name。

另外`AbstractEnvironment`类是不启动spring的情况可以单独使用的（因为它就是个简单的java类）。如
```java
/**
 * @author han.xue
 * @date 2021-06-05 21:20:20
 */
public class Test extends AbstractEnvironment {
    private static final String MY_PROPERTIES = "MyProperties";
    private static final String MY_PROPERTIES2 = "MyProperties2";

    @Override
    protected void customizePropertySources(MutablePropertySources propertySources) {

        Properties properties = new Properties();
        properties.put("name", "zhangsan");
        properties.put("age", "22");
        propertySources.addLast(
                new PropertiesPropertySource(MY_PROPERTIES, properties));


        Properties properties2 = new Properties();
        properties2.put("name", "wangwu");

        propertySources.addLast(
                new PropertiesPropertySource(MY_PROPERTIES2, properties2));



    }

    public static void main(String[] args) {
        Test enTest = new Test();

        System.out.println(enTest.resolvePlaceholders("hello: ${name:lisi}"));
        System.out.println(enTest.resolveRequiredPlaceholders("your age is: ${age:-1}..."));

        //这个会抛出异常,因为找不到test
//        System.out.println(enTest.resolveRequiredPlaceholders("This is a placeholder : ${test33}..."));
        
        // 也可以获取所有的PropertySource
        MutablePropertySources propertySources = enTest.getPropertySources();
        PropertySource<?> myProperties = propertySources.get(MY_PROPERTIES);
        // 直接取出这个配置中的属性值
        Object name = myProperties.getProperty("name");
        System.out.println(name);
    }
}
```

解析${}是在`org.springframework.util.PropertyPlaceholderHelper#parseStringValue`类中完成。

那么Spring在启动的时候是在什么时机去解析占位符${}的呢？答案就是`PlaceholderConfigurerSupport`。先看下`PlaceholderConfigurerSupport`类图
![](https://data-repository-01.oss-cn-shanghai.aliyuncs.com/img/PlaceholderConfigurerSupport.png)
显而易见这个类实现了`BeanFactoryPostProcessor`，那接下来就和其他的`BeanFactoryPostProcessor`实现类没什么区别了,也就是调用实际是在org.springframework.context.support.AbstractApplicationContext#refresh方法中的`invokeBeanFactoryPostProcessors()`方法中

## BeanDefinitionReader

## 循环依赖
需要阅读org.springframework.beans.factory.support.DefaultSingletonBeanRegistry，且类中有三个属性，也就是常说的三级缓存

```java

	/** Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
```
当创建bean的时候，先从一级缓存查看是否存在，如果存在，直接返回，不会执行创建bena的逻辑，如果不存在会查找二级，如果再不存在，会查找三级。

- 一级缓存放成品对象
- 二级缓存放半成品对象
- 三级缓存放lambda表达式，来完成创建代理对象

### Q：三级缓存解决循环依赖的关键是什么？为什么通过提前暴露对象能解决？
A：实例化和初始化分开操作，在中间过程中给引用的对象赋值的时候，并不是一个完整的对象，而是把“半成品”（只完成实例化，未完成初始化的对象）赋值给了引用的对象

### Q:如果只使用一级缓存能不能解决?
A: 不能，在整个处理过程中，缓存中存放的有半成品成品对象，如果只有一个缓存，那么成品和半成品会放入同一个缓存。这样就有可能在获取过程中取到半成品对象，此时半成品对象是无法使用的，不能直接进行使用，因此半成品对象和成品对象要分开存放

### Q：如果只使用两级缓存行不行？
A: 如果可以保证所有的bean对象都不调用getEarlyBeanReference此方法，使用二级缓存是可以的，使用三级缓存的本质是为了解决AOP的代理问题(在`getEarlyBeanReference`方法中会进行if判断看是否需要创建bean的代理对象，如果需要，就将本来要返回的bean重新赋值成代理的对象，也就是说如果一个bean被创建了代理对象，其自身也会被创建)

> 'getEarlyBeanReference()'中如果bean需要代理，也会创建代理，具体接口为org.springframework.aop.framework.AopProxy#getProxy(java.lang.ClassLoader)其实现有`CglibAopProxy`和`JdkDynamicAopProxy`

### Q: 

> 查看源码时，一定不要纠结与具体的细节，要抓住方法骨架，也就是终点方法，例如: getBean -> doGetBean,createBean()-> doCreateBean()

## 配置文件加载

lookup-method
replace-method

spring bean的生命周期



# Spring Boot
run方法中会读取classpath中文件路径为META-INF/spring.factories的内容，可以获取到初始化器和监听器的全路径名称
```java
// 定义文件路径(文件名)
org.springframework.core.io.support.SpringFactoriesLoader#FACTORIES_RESOURCE_LOCATION
```

BeanFactory和ApplicationContext的关系:继承, ApplicationContext实现了BeanFactory接口

## 自动装配的原理
`org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass()`这个方法是在`org.springframework.context.support.AbstractApplicationContext#refresh()`方法中的'invokeBeanFactoryPostProcessors()方法调到的'

## 自定义Starter
在资源文件路径下新建spring.factories，然后内容写上
