# 使用CustomAutowireConfigurer
CustomAutowireConfigurer是一个BeanFactoryPostProcessor，它允许您注册自己的自定义限定符注解类型，即使它们没有使用Spring的@Qualifier注解。下面的例子展示了如何使用customautowireconfigiler:
```xml
<bean id="customAutowireConfigurer"
		class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
	<property name="customQualifierTypes">
		<set>
			<value>example.CustomQualifier</value>
		</set>
	</property>
</bean>
```
AutowireCandidateResolver通过以下方式确定自动候选对象:
- 每个bean定义的自动连接候选值
- <beans/>元素上可用的任何默认自动连接的候选模式
- @Qualifier注解和CustomAutowireConfigurer注册的任何自定义注解的存在，当多个bean符合自动候选条件时，确定“primary”的方法如下:如果候选中只有一个bean定义的主属性设置为true，则选择它。



