# 介绍 Spring IoC容器 和 Bean

本章涵盖了控制反转(IoC)原理的Spring框架实现。

依赖注入(DI)是IoC的一种特殊形式，对象仅通过__构造参数__、__工厂方法的参数__或__构造方法创建的对象__或__工厂方法__返回后在对象实例上设置的属性来定义它们的依赖项(即与它们一起工作的其他对象)。然后 IoC 容器在创建bean时注入这些依赖项。这个过程基本上是__bean本身的逆过程(因此得名为控制反转)__，通过使用类的直接构造或Service Locator模式等机制来控制其依赖项的实例化或位置

beans和context包是Spring Framework的IoC容器的基础。BeanFactory接口提供了一种高级配置机制，能够管理任何类型的对象。ApplicationContext是BeanFactory的子接口，它增加了以下功能：
 - 更容易地与Spring的AOP特性集成
 - 消息资源处理(用于国际化)
 - 事件发布
 - 应用程序层特定的上下文，例如用于web应用程序的WebApplicationContext。

简而言之，BeanFactory提供了配置框架和基本功能，而ApplicationContext则添加了更多特定于企业的功能。ApplicationContext是BeanFactory的一个完整的超集，并且在本章中专门用于描述Spring的IoC容器。有关使用BeanFactory而不是ApplicationContext的更多信息，请参阅有关BeanFactory API的部分。 

在Spring中，构成应用程序框架并由Spring IoC容器管理的对象称为bean。__bean是由Spring IoC容器实例化、组装和管理的对象__。否则，bean只是应用程序中众多对象中的一个。bean以及它们之间的依赖关系反映在容器使用的配置元数据中。

