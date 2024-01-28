# 懒加载
默认情况下，ApplicationContext实现急切地创建和配置所有单例bean，作为初始化过程的一部分。通常，这种预实例化是可取的，因为可以立即发现配置或周围环境中的错误，而不是在几小时甚至几天之后发现。
当不需要这种行为时，可以通过将bean定义标记为惰性初始化来防止单例bean的预实例化。
懒加载bean告诉IoC容器在第一次请求时创建bean实例，而不是在启动时创建

在XML中，此行为由<bean/>元素上的lazy-init属性控制，如下面的示例所示：
```xml
<bean id="lazy" class="com.something.ExpensiveToCreateBean" lazy-init="true"/>
<bean name="not.lazy" class="com.something.AnotherBean"/>
```
当前面的配置被ApplicationContext使用时，当ApplicationContext启动时，lazy bean不会被急切地预实例化，not.lazy bean被急切地预实例化

然而，当懒加载bean是非懒加载单例bean的依赖项时，ApplicationContext会在启动时初始化懒加载bean，
因为它必须满足单例的依赖项。懒加载的bean被注入到其他非懒加载单例bean中。

你也可以通过在<beans/>元素上使用default-lazy-init属性来控制容器级别的惰性初始化，如下面的例子所示:
```xml
<beans default-lazy-init="true">
	<!-- no beans will be pre-instantiated... -->
</beans>
```