# Spring框架总览
Spring可以让创建企业级应用变得很简单，Spring提供了企业级Java应用的任何东西，同时支持了Grovvy和Kotlin这种适配JVM的语言。

同时可以灵活的创建多种架构；Spring6.0需要java17+的支持

Spring提供了非常广泛的使用场景，在大型企业应用场景中，在大型企业中，应用程序通常存在很长时间，并且必须在JDK和应用程序服务器上运行，其升级周期超出开发人员的控制范围
其他可能作为嵌入服务器的单个jar运行，可能在云环境中。还有一些可能是不需要服务器的独立应用程序(如批处理或集成工作负载)。

Spring是开源的，同时Spring拥有非常庞大且活跃的社区。社会可以提供基于真实世界中使用过程中的反馈；这帮助Spring在长时间的良好的进化

# Spring是什么意思
Spring关键字在不同上下文中有不同的含义，Spring可以被用在Spring框架本身。这是所有东西的起点；随着事件推移，有其他Spring的附属项目构建在Spring之上；
通常来说，当我们说Spring的时候，指向的一整个系列的成员，这个文档聚焦于基础：Spring框架本身；

关于模块的说明:Spring的框架jar允许部署到JDK 9的模块路径(“Jigsaw
为了在支持jigsaw的应用程序中使用，Spring Framework 5 jar附带了“Automatic-Module-Name”清单项，
它定义了稳定的语言级模块名称(“spring.core”、“spring.context”等)，独立于jar构件名称(jar遵循相同的命名模式，使用“-”而不是“。”，例如。“spring-core”和“spring-context”)。当然，在JDK 8和JDK 9+的类路径下，Spring的框架jar都能很好地工作。

# Spring的历史
Spring于2003年出现，作为对早期J2EE规范复杂性的一种解决方案。虽然有些人认为Java EE及其现代继承者Jakarta EE是Spring的竞争对手，实际上它们是一种互补的关系；Spring编程模型不支持Jakarta EE平台规范；相反，它集成了传统EE中精心挑选的特性
列表如下：

 - Servlet API (JSR 340)
 - WebSocket API (JSR 356)
 - Concurrency Utilities (JSR 236)
 - JSON Binding API (JSR 367)
 - Bean Validation (JSR 303)
 - JPA (JSR 338)
 - JMS (JSR 914)
 - as well as JTA/JCA setups for transaction coordination, if necessary.

Spring框架同时也支持了依赖注入(Dependency Injection)以及通用注解(Common Annotations),开发人员可以选择使用这些规范，而不是Spring框架提供的特定于Spring的机制 
从Spring6.0开始，Spring已经升级到 Jakarta EE 9级别(例如Servlet 5.0+， JPA 3.0+)，基于Jakarta名称空间而不是传统的javax包；
以EE 9为最低标准，并且已经支持EE 10，Spring准备为Jakarta EE api的进一步发展提供开箱即用的支持，Spring Framework 6.0与Tomcat 10.1、Jetty 11和Undertow 2.3作为web服务器完全兼容，也与Hibernate ORM 6.1兼容

随着时间的推移，Java/Jakarta EE在应用程序开发中的角色不断发展。在J2EE和Spring的早期，创建应用程序是为了部署到应用服务器上。如今，在Spring Boot的帮助下，应用程序可以以一种对devops和云友好的方式创建，并嵌入Servlet容器，更改非常简单。

从Spring Framework 5开始，WebFlux应用程序甚至不直接使用Servlet API，可以在不是Servlet容器的服务器(比如Netty)上运行

# 设计哲学
当学习这个框架的时候，不只是要弄明白这个框架是干什么的，更要知道框架设计遵循着什么原则。这里有一些Spring设计过程中的指导性原则
- 在每一个层级都提供选择；Spring让你尽可能晚地推迟设计决策; 例如，您可以通过配置切换持久性提供程序，而无需更改代码;对于许多其他基础设施问题和与第三方api的集成也是如此。
- 包容不同的观点。Spring拥抱灵活性，并且不固执于应该如何做事情。它支持具有不同视角的广泛应用程序需求
- 保持强大的向后兼容性。Spring的演变得到了精心的管理，在版本之间很少强制进行突破性的更改。Spring支持一系列精心挑选的JDK版本和第三方库，以方便对依赖Spring的应用程序和库的维护
- 关注API设计。Spring团队投入了大量的精力和时间来制作直观的api，这些api可以跨越许多版本和许多年
- 为代码质量设定高标准。Spring框架非常强调有意义的、最新的和准确的javadoc。它是少数几个可以宣称代码结构干净且包之间没有循环依赖的项目之一

# 反馈和贡献
 推荐使用 Stack Overflow 和 Github

# 开干
如果您刚刚开始使用Spring，您可能希望通过创建一个基于Spring boot的应用程序来开始使用Spring框架。
Spring Boot提供了一种快速的方法来创建生产就绪的基于Spring的应用程序。它基于Spring框架，支持约定大于配置， 

> 约定大于配置的意思就是：有一些东西是默认的，比如：模型中有个名为Sale的类，那么数据库中对应的表就会默认命名为sales。只有在偏离这一约定时，例如将该表命名为“products_sold”，才需写有关这个名字的配置

旨在让您尽可能快地启动和运行。 你可以使用start.spring.io来生成一个基本的项目，或者按照“入门”指南中的一个来做，比如入门构建RESTful Web服务。

这些指南不仅易于理解，且目标明确；其中大多数都基于Spring Boot。它们还涵盖了在解决特定问题时可能需要考虑的Spring组合中的其他项目