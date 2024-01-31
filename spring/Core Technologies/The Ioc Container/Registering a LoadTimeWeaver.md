# 注册LoadTimeWeaver

Spring使用LoadTimeWeaver在类加载到Java虚拟机(JVM)时对它们进行动态转换。

>应该是类似于javaAgent的技术,直接修改字节码

要启用加载时编织，你可以将@EnableLoadTimeWeaving添加到你的@Configuration类中，如下面的例子所示:

```java
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {
}
```

另外，对于XML配置，你可以使用context:load-time-weaver元素:

```xml
<beans>
	<context:load-time-weaver/>
</beans>
```

一旦为ApplicationContext配置好，该ApplicationContext中的任何bean都可以实现LoadTimeWeaverAware，从而接收到对加载时编织实例的引用。这在与Spring的JPA支持结合使用时特别有用，因为加载时编织可能是JPA类转换所必需的。有关更多细节，请参阅LocalContainerEntityManagerFactoryBean javadoc。有关AspectJ加载时编织的更多信息，请参阅Spring框架中使用AspectJ进行加载时编织。