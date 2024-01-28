# 容器总览

org.springframework.context.ApplicationContext 接口是 Spring IoC container 的实现代表之一，负责实例化，配置和组装Beans，容器通过获取配置，元数据来实例化对象；配置元数据的类型有XML，java注解，和java代码等。这些配置可以让你创建对象并组装应用，并且，可以和多种方式和第三方组件桥接使用；

在Spring中有很多种ApplicationContext的实现，在独立的应用中，通常使用的ClassPathXmlApplicationContext和FileSystemXmlApplicationContext；虽然XML一直是定义配置元数据的传统格式，但可以通过提供少量XML配置来声明地启用对这些附加元数据格式的支持，从而指示容器使用Java注解或代码作为元数据格式

在绝大部分应用场景中，不需要用户使用显式代码来实例化一个或多个Spring IoC容器实例。例如，在web应用的场景中，XML文件中简单的八行(或左右)样板web描述符XML通常就足够了(参见web应用程序的方便应用程序上下文实例化)。如果使用Spring Tools for Eclipse(基于Eclipse的开发环境)，只需单击几下鼠标或敲击几下键盘，就可以轻松地创建这个样板配置

下面的这个图片从高层级的角度解释了Spring是怎么工作的，应用程序类与配置元数据相结合，在ApplicationContext创建和完成初始化之后，就可以创建出一个完全配置的可执行系统或应用程序。

![container magic](../The%20Ioc%20Container/assets/container-magic.png)



# 配置元数据

如上图所示，Spring IoC容器使用一种形式的配置元数据。此配置元数据表示作为应用程序开发人员，告诉Spring容器如何实例化、配置和组装应用程序中的对象。
传统上，配置元数据以简单直观的XML格式提供，本章大部分内容都使用这种格式来传达Spring IoC容器的关键概念和特性。

基于xml的元数据并不是唯一允许的配置元数据形式。Spring IoC容器本身与实际编写配置元数据的格式完全解耦。现在，许多开发人员为他们的Spring应用程序选择基于java注解的配置。

有关在Spring容器中使用其他形式的元数据的信息，请参见: 
基于注解的配置:使用基于注解的配置元数据定义bean。 
基于Java的配置:通过使用Java而不是XML文件来定义应用程序类外部的bean。要使用这些特性，请参阅@Configuration、@Bean、@Import和@DependsOn注解

Spring配置由容器必须管理的至少一个(通常是多个)bean定义组成。
基于xml的配置元数据将这些bean配置为顶层<beans/>元素中的<bean/>元素。
Java配置通常在@Configuration类中使用带有@bean注解的方法

这些bean定义对应于构成应用程序的实际对象。
通常，您定义服务层对象、持久化层对象(如存储库或数据访问对象)、表示对象(如Web控制器)、基础设施对象(如JPA EntityManagerFactory)、JMS队列等。
一般不需要在容器中配置细粒度的域对象，因为创建和加载域对象通常是存储库和业务逻辑的责任

下面的示例展示了基于xml的配置元数据的基本结构：
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="..." class="...">  
		<!-- collaborators and configuration for this bean go here -->
	</bean>

	<bean id="..." class="...">
		<!-- collaborators and configuration for this bean go here -->
	</bean>

	<!-- more bean definitions go here -->

</beans>

```
* id属性是标识单个bean定义的字符串。 
* class属性定义bean的类型并使用完全限定类名。