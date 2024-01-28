# 容器总览

org.springframework.context.ApplicationContext 接口是 Spring IoC container 的实现代表之一，负责实例化，配置和组装Beans，容器通过获取配置，元数据来实例化对象；配置元数据的类型有XML，java注解，和java代码等。这些配置可以让你创建对象并组装应用，并且，可以和多种方式和第三方组件桥接使用；

在Spring中有很多种ApplicationContext的实现，在独立的应用中，通常使用的ClassPathXmlApplicationContext和FileSystemXmlApplicationContext；虽然XML一直是定义配置元数据的传统格式，但可以通过提供少量XML配置来声明地启用对这些附加元数据格式的支持，从而指示容器使用Java注解或代码作为元数据格式

在绝大部分应用场景中，不需要用户使用显式代码来实例化一个或多个Spring IoC容器实例。例如，在web应用的场景中，XML文件中简单的八行(或左右)样板web描述符XML通常就足够了(参见web应用程序的方便应用程序上下文实例化)。如果使用Spring Tools for Eclipse(基于Eclipse的开发环境)，只需单击几下鼠标或敲击几下键盘，就可以轻松地创建这个样板配置

下面的这个图片从高层级的角度解释了Spring是怎么工作的，应用程序类与配置元数据相结合，在ApplicationContext创建和完成初始化之后，就可以创建出一个完全配置的可执行系统或应用程序。

![container magic](../The%20Ioc%20Container/assets/container-magic.png)



# 配置元数据

如上图所示，Spring IoC容器使用一种来源形式的配置元数据。此配置元数据表示作为应用程序开发人员告诉Spring容器如何实例化、配置和组装应用程序中的对象。传统上，配置元数据以简单直观的XML格式提供，本章大部分内容都使用这种格式来传达Spring IoC容器的关键概念和特性。

基于xml的元数据并不是唯一允许的配置元数据形式。Spring IoC容器本身与实际编写配置元数据的格式完全解耦。现在，许多开发人员为他们的Spring应用程序选择基于java注解的配置。

有关在Spring容器中使用其他形式的元数据的信息，请参见: 

- 基于注解的配置：使用基于注解的配置元数据定义bean。 
- 基于Java的配置：通过使用Java而不是XML文件来定义应用程序类外部的bean。要使用这些特性，请参阅@Configuration、@Bean、@Import和@DependsOn注解

Spring配置由容器必须管理的至少一个(通常是多个)bean定义组成。基于xml的配置元数据将这些bean配置为顶层<beans/>元素中的<bean/>元素。Java配置通常在@Configuration类中使用带有@bean注解的方法。

这些bean定义对应于构成应用程序的实际对象。通常，您定义服务层对象、持久化层对象(如存储库或数据访问对象)、表示对象(如Web控制器)、基础设施对象(如JPA EntityManagerFactory)、JMS队列等。
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
* __id属性是标识单个bean定义的字符串。__ 
* class属性定义bean的类型并使用的完全限定类名。



# 实例化Spring容器

提供给ApplicationContext构造函数的位置路径是资源字符串，它允许容器从各种外部资源(如本地文件系统、Java CLASSPATH等)加载配置元数据
```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```
在了解了Spring的IoC容器之后，您可能希望更多地了解Spring的资源抽象(如参考资料中所述)，它提供了一种方便的机制，用于从URI语法中定义的位置读取InputStream。特别是，资源路径用于构建应用程序上下文，如应用程序上下文和资源路径中所述

下面的示例显示了服务层对象(services.xml)配置文件：
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<!-- services -->

	<bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
		<property name="accountDao" ref="accountDao"/>
		<property name="itemDao" ref="itemDao"/>
		<!-- additional collaborators and configuration for this bean go here -->
	</bean>

	<!-- more bean definitions for services go here -->

</beans>
```

下面的示例显示了数据访问对象daos.xml文件
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="accountDao"
		class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
		<!-- additional collaborators and configuration for this bean go here -->
	</bean>

	<bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
		<!-- additional collaborators and configuration for this bean go here -->
	</bean>

	<!-- more bean definitions for data access objects go here -->

</beans>
```
在前面的示例中，服务层由PetStoreServiceImpl类和两个类型为JpaAccountDao和JpaItemDao的数据访问对象(基于JPA对象-关系映射标准)组成。

__属性name元素引用JavaBean属性的名称，ref元素引用另一个bean定义的名称。id和ref元素之间的这种链接表达了协作对象之间的依赖关系。__有关配置对象依赖项的详细信息，请参见依赖项



# 组合基于xml的配置元数据

让bean定义跨多个XML文件是很有用的。通常，每个单独的XML配置文件表示体系结构中的一个逻辑层或模块
可以使用应用程序上下文构造函数从所有这些XML片段加载bean定义。这个构造函数接受多个Resource位置，如前一节所示。或者，使用<import/>元素的一次或多次出现来从另一个或多个文件加载bean定义。下面的示例展示了如何这样做

```xml
<beans>
	<import resource="services.xml"/>
	<import resource="resources/messageSource.xml"/>
	<import resource="/resources/themeSource.xml"/>

	<bean id="bean1" class="..."/>
	<bean id="bean2" class="..."/>
</beans>
```
在前面的示例中，外部bean定义是从三个文件加载的:services.xml、messageSource.xml和themeSource.xml。
所有位置路径都相对于执行导入的定义文件，因此services.xml必须与执行导入的文件位于相同的目录或类路径位置，而messageSource.xml和themeSource.xml必须位于导入文件位置下方的资源位置。

如您所见，前导斜杠会被忽略。然而，考虑到这些路径是相对的，最好不要使用斜杠。根据Spring Schema，被导入的文件的内容，包括顶级的<beans/>元素，必须是有效的XML bean定义

可以(但不推荐)使用相对的".. "来引用父目录中的文件。/”路径。这样做会创建对当前应用程序外部文件的依赖。特别地，不建议对classpath: URLs(例如，classpath:../services.xml)使用这个引用，因为运行时解析过程会选择“最近的”类路径根，然后查看它的父目录。类路径配置更改可能导致选择不同的、不正确的目录

您始终可以使用完全限定的资源位置，而不是相对路径:例如，file:C:/config/services.xml或classpath:/config/services.xml。但是，请注意，您正在将应用程序的配置与特定的绝对位置耦合。对于这样的绝对位置，通常最好保留一个间接的位置——例如，通过在运行时根据JVM系统属性解析的“${…}”占位符

命名空间本身提供了导入指令特性。除了普通bean定义之外，Spring还提供了一系列XML名称空间来提供更多的配置特性——例如，上下文和util名称空间



#  Groovy Bean定义DSL
略



# 使用容器
ApplicationContext是高级工厂的接口，该工厂能够维护不同bean及其依赖项的注册表。通过使用方getBean(字符串名称，类<T> requiredType)，您可以检索bean的实例

ApplicationContext允许您读取bean定义并访问它们，如下面的示例所示：
```java
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```


最灵活的变体是GenericApplicationContext与阅读器委托结合使用——例如，与XML文件的XmlBeanDefinitionReader结合使用，如下面的示例所示:
```java
GenericApplicationContext context = new GenericApplicationContext();

new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");

context.refresh();
```
您可以在同一个ApplicationContext上混合和匹配这样的读取器委托，从不同的配置源读取bean定义

然后可以使用getBean来检索bean的实例。ApplicationContext接口有一些其他的方法来检索bean，但理想情况下，应用程序代码不应该使用它们。

实际上，您的应用程序代码根本不应该调用getBean()方法，因此根本不依赖于Spring api。例如，Spring与web框架的集成为各种web框架组件(如控制器和jsf托管bean)提供了依赖注入，允许您通过元数据(如自动装配注解)声明对特定bean的依赖



