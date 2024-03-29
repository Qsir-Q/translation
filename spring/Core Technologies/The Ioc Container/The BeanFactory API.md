# BeanFactory API
BeanFactory API为Spring的IoC功能提供了底层基础。它的特定接口主要用于与Spring的其他部分和相关的第三方框架集成，它的DefaultListableBeanFactory实现是高级GenericApplicationContext容器中的关键委托。

BeanFactory和相关接口(如BeanFactoryAware、InitializingBean、DisposableBean)是其他框架组件的重要集成点。由于不需要任何注释甚至反射，它们允许容器与其组件之间进行非常有效的交互。应用程序级bean可以使用相同的回调接口，但通常更喜欢通过注释或编程配置进行声明性依赖注入。

请注意，核心BeanFactory API级别及其DefaultListableBeanFactory实现没有对要使用的配置格式或任何组件注释做出假设。所有这些风格都是通过扩展(如XmlBeanDefinitionReader和AutowiredAnnotationBeanPostProcessor)实现的，并将共享BeanDefinition对象作为核心元数据表示进行操作。这是使Spring的容器如此灵活和可扩展的本质。



# BeanFactory or ApplicationContext?

本节将解释BeanFactory和ApplicationContext容器级别之间的区别，以及它们对引导的影响。

你应该使用ApplicationContext，除非你有很好的理由不这样做，使用GenericApplicationContext和它的子类AnnotationConfigApplicationContext作为自定义引导的通用实现。这些是Spring核心容器的主要入口点，用于所有常见目的:加载配置文件、触发类路径扫描、以编程方式注册bean定义和带注释的类，以及(从5.0开始)注册功能bean定义。

因为ApplicationContext包含了BeanFactory的所有功能，所以除了需要完全控制bean处理的场景外，通常建议使用它而不是普通的BeanFactory。在ApplicationContext(例如GenericApplicationContext实现)中，按照约定(即按bean名称或bean类型—特别是后处理器)检测几种bean，而普通的DefaultListableBeanFactory不知道任何特殊的bean。

对于许多扩展的容器特性，比如注释处理和AOP代理，BeanPostProcessor扩展点是必不可少的。如果只使用普通的DefaultListableBeanFactory，默认情况下不会检测和激活此类后处理器。这种情况可能令人困惑，因为您的bean配置实际上没有任何问题。相反，在这种情况下，需要通过额外的设置来完全引导容器。

下表列出了BeanFactory和ApplicationContext接口和实现提供的特性。
ApplicationContext相对于BeanFactory多了以下功能：
1. 集成生命周期管理
2. 自动BeanPostProcessor注册
3. 自动BeanFactoryPostProcessor注册
4. 方便的MessageSource访问(用于国际化)
5. 内置的ApplicationEvent发布机制

要显式地用DefaultListableBeanFactory注册一个bean后处理器，你需要通过编程方式调用addBeanPostProcessor，如下面的例子所示:
```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// populate the factory with bean definitions

// now register any needed BeanPostProcessor instances
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
factory.addBeanPostProcessor(new MyBeanPostProcessor());

// now start using the factory
```
要将BeanFactoryPostProcessor应用到一个普通的DefaultListableBeanFactory，你需要调用它的postProcessBeanFactory方法，如下面的例子所示:
```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new FileSystemResource("beans.xml"));

// bring in some property values from a Properties file
PropertySourcesPlaceholderConfigurer cfg = new PropertySourcesPlaceholderConfigurer();
cfg.setLocation(new FileSystemResource("jdbc.properties"));

// now actually do the replacement
cfg.postProcessBeanFactory(factory);
```

在这两种情况下，显式注册步骤都不方便，这就是为什么在spring支持的应用程序中，各种ApplicationContext变体比普通的DefaultListableBeanFactory更受欢迎，特别是在典型的企业设置中依赖BeanFactoryPostProcessor和BeanPostProcessor实例来扩展容器功能时。

一个AnnotationConfigApplicationContext已经注册了所有通用的注释后处理器，并且可以通过配置注释(如@EnableTransactionManagement)在底层引入额外的处理器。在Spring基于注释的配置模型的抽象层，bean后处理器的概念变成了仅仅是内部容器细节。