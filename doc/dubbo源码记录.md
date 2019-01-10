## dubbo源码记录.md
### 基础知识
#### Java SPI机制
Service Provider Interface) ，是JDK内置的一种服务提供发现机制。SPI是一种动态替换发现的机制， 比如有个接口，想运行时动态的给它添加实现，你只需要添加一个实现。

SPI接口的定义在调用方，在概念上更依赖调用方；组织上位于调用方所在的包中；实现位于独立的包中。

当接口属于实现方的情况，实现方提供了接口和实现，这个用法很常见，属于API调用。我们可以引用接口来达到调用某实现类的功能。

当服务的提供者提供了一种接口的实现之后，需要在classpath下的META-INF/services/目录里创建一个以服务接口名(接口类的full-name)命名的文件，这个文件里的内容就是这个接口的具体的实现类(实现类的full-name)。当其他的程序需要这个服务的时候，就可以通过查找这个jar包（一般都是以jar包做依赖）的META-INF/services/中的配置文件，配置文件中有接口的具体实现类名，可以根据这个类名进行加载实例化，就可以使用该服务了。JDK中查找服务实现的工具类是：java.util.ServiceLoader。

具体示例可以参考https://gitlab.com/johnxue2013/spi-service.git接口定义和https://gitlab.com/johnxue2013/spi-service-impl.git接口实现
