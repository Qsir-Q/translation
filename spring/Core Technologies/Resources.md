# Resources
本章将介绍Spring如何处理资源以及如何在Spring中使用资源。



# 介绍

不幸的是，Java的标准java.net.URL类和各种URL前缀的标准处理程序并不足以满足对低级资源的所有访问。例如，没有标准化的URL实现可用于访问需要从类路径或相对于ServletContext获得的资源。虽然可以为专门的URL前缀注册新的处理程序(类似于现有的http:前缀处理程序)，但这通常非常复杂，并且URL接口仍然缺乏一些理想的功能，例如检查所指向的资源是否存在的方法。



# Resource 接口

Spring的Resource接口位于org.springframework.core.io包.是一个更有能力的接口，用于抽象对低级资源的访问。下面的清单提供了Resource接口的概述。有关详细信息，请参阅参考资料javadoc。

```java
public interface Resource extends InputStreamSource {

	boolean exists();

	boolean isReadable();

	boolean isOpen();

	boolean isFile();

	URL getURL() throws IOException;

	URI getURI() throws IOException;

	File getFile() throws IOException;

	ReadableByteChannel readableChannel() throws IOException;

	long contentLength() throws IOException;

	long lastModified() throws IOException;

	Resource createRelative(String relativePath) throws IOException;

	String getFilename();

	String getDescription();
}
```
正如Resource接口的定义所示，它继承了InputStreamSource接口。下面的清单显示了InputStreamSource接口的定义:
```java
public interface InputStreamSource {

	InputStream getInputStream() throws IOException;
}
```
来自Resource接口的一些最重要的方法是:
1. getInputStream():定位并打开资源，返回用于从资源中读取的InputStream。预计每次调用都会返回一个新的InputStream。关闭流是调用者的责任。
2. exists():返回一个布尔值，指示该资源是否以物理形式实际存在。
3. isOpen():返回一个布尔值，指示此资源是否表示具有打开流的句柄。如果为true，则不能多次读取InputStream，必须只读取一次，然后关闭以避免资源泄漏。除InputStreamResource外，对所有常见的资源实现返回false。
4. getDescription():返回该资源的描述，用于处理该资源时的错误输出。这通常是完全限定的文件名或资源的实际URL。其他方法让您获得表示资源的实际URL或File对象(如果底层实现兼容并支持该功能)。

Resource接口的一些实现还为支持向其写入的资源实现了扩展的writablerresource接口。

Spring本身广泛地使用Resource抽象，在需要资源时作为许多方法签名的参数类型。一些Spring api中的其他方法(例如各种ApplicationContext实现的构造函数)接受一个字符串，该字符串以朴素或简单的形式用于创建适合该上下文实现的资源，或者通过字符串路径上的特殊前缀，让调用者指定必须创建和使用特定的资源实现。

虽然Resource接口在Spring和Spring中被大量使用，但实际上，在您自己的代码中单独使用它作为通用实用程序类来访问资源是非常方便的，即使您的代码不知道或不关心Spring的任何其他部分。虽然这将您的代码与Spring耦合在一起，但它实际上只是将其与一小部分实用程序类耦合在一起，这些实用程序类可以作为URL的更有能力的替代品，并且可以视为等同于用于此目的的任何其他库。

Resource不能取代functionality。它在可能的地方包装它。例如，UrlResource包装了一个URL，并使用包装后的URL来完成它的工作。



# 内置Resource实现

Spring包括几个内置的Resource实现:
1. UrlResource
2. ClassPathResource
3. FileSystemResource
4. PathResource
5. ServletContextResource
6. InputStreamResource
7. ByteArrayResource

有关Spring中可用的资源实现的完整列表，请参阅Resource javadoc的“所有已知实现类”部分。



# UrlResource

UrlResource包装了一个java.net.URL，并可用于访问通常可以通过URL访问的任何对象，例如文件、HTTPS目标、FTP目标等。所有的URL都有一个标准化的String表示，这样就可以使用适当的标准化前缀来表示不同的URL类型。这包括file:用于访问文件系统路径，https:用于通过https协议访问资源，ftp:用于通过ftp访问资源，等等。

UrlResource是由Java代码显式地使用UrlResource构造函数创建的，但通常是在调用API方法时隐式地创建的，该方法接受一个表示路径的String参数。对于后一种情况，JavaBeans PropertyEditor最终决定要创建哪种类型的资源。如果路径字符串包含一个众所周知的(即属性编辑器)前缀(如classpath:)，它将为该前缀创建一个适当的专门化资源。但是，如果它不识别前缀，它就假定字符串是标准URL字符串并创建UrlResource。



# ClassPathResource

该类表示应该从类路径获得的资源。它使用线程上下文类装入器、给定的类装入器或给定的类装入资源。

如果类路径资源存在文件系统中，则此资源实现支持作为java.io.File进行解析，但对于驻留在jar中的类路径资源则不支持(由servlet引擎或其他环境)扩展到文件系统。为了解决这个问题，各种资源实现总是支持解析为java.net.URL。

ClassPathResource是由Java代码显式地使用ClassPathResource构造函数创建的，但是通常是在调用API方法时隐式地创建的，该API方法接受一个表示路径的String参数。对于后一种情况，JavaBeans PropertyEditor识别字符串路径上的特殊前缀classpath:，并在这种情况下创建一个ClassPathResource。



# FileSystemResource

这是java.io.File句柄的资源实现。它还支持java.nio.file.Path句柄，应用Spring标准的基于字符串的路径转换，但通过java.nio.file.Files API执行所有操作。对于纯基于java. io.path. path的支持，请使用PathResource代替。FileSystemResource支持解析为File和URL。



# PathResource

这是java.nio.file.Path句柄的资源实现，通过Path API执行所有操作和转换。它支持作为File和URL的解析，还实现了扩展的writablerresource接口。PathResource实际上是一个纯粹的基于java. io.path. path的FileSystemResource替代方案，具有不同的createRelative行为。



# ServletContextResource

这是ServletContext资源的一个资源实现，它解释了相关web应用程序根目录中的相对路径。它始终支持流访问和URL访问，但只有当web应用程序归档扩展并且资源物理上位于文件系统中时才允许java.io.File访问。它是扩展到文件系统上，还是直接从JAR或其他地方(如数据库)访问(这是可以想象的)，实际上取决于Servlet容器。



# InputStreamResource

InputStreamResource是给定InputStream的资源实现。只有在没有特定的Resource实现适用的情况下才应该使用它。特别是，在可能的情况下，首选ByteArrayResource或任何基于文件的Resource实现。



# ByteArrayResource

这是一个给定字节数组的资源实现。它为给定的字节数组创建一个ByteArrayInputStream。它对于从任何给定的字节数组加载内容非常有用，而不必求助于单一使用的InputStreamResource。



# ResourceLoader接口

ResourceLoader接口是由可以返回(即加载)资源实例的对象实现的。下面的清单显示了ResourceLoader接口定义:
```java
public interface ResourceLoader {

	Resource getResource(String location);

	ClassLoader getClassLoader();
}
```
所有应用程序上下文都实现ResourceLoader接口。因此，可以使用所有应用程序上下文来获取Resource实例。

当您在特定的应用程序上下文中调用getResource()，并且指定的位置路径没有特定的前缀时，您将返回适合于该特定应用程序上下文中的Resource类型。例如，假设下面的代码片段是针对ClassPathXmlApplicationContext实例运行的:
```java
Resource template = ctx.getResource("some/resource/path/myTemplate.txt");
```
对于ClassPathXmlApplicationContext，该代码返回一个ClassPathResource。如果对FileSystemXmlApplicationContext实例运行相同的方法，它将返回一个FileSystemResource。
对于WebApplicationContext，它将返回一个ServletContextResource。类似地，它将为每个上下文返回适当的对象。因此，您可以以适合特定应用程序上下文的方式加载资源。

另一方面，你也可以强制使用ClassPathResource，而不考虑应用程序上下文类型，通过指定特殊的classpath:前缀，如下面的例子所示:
```java
Resource template = ctx.getResource("classpath:some/resource/path/myTemplate.txt");
```
类似地，您可以通过指定任何标准java.net.URL前缀来强制使用UrlResource。使用file和https前缀的示例如下:
```java
Resource template = ctx.getResource("file:///some/resource/path/myTemplate.txt");
```
```java
Resource template = ctx.getResource("https://myhost.com/resource/path/myTemplate.txt");
```
下表总结了将String对象转换为Resource对象的策略:
1. classpath: 从类路径加载。
2. file:作为URL从文件系统加载。另请参见FileSystemResource注意事项。
3. https: Loaded as a URL.
4. (none) 依赖于底层的ApplicationContext。



# ResourcePatternResolver接口

ResourcePatternResolver接口是对ResourceLoader接口的扩展，该接口定义了将位置模式(例如ant风格的路径模式)解析为资源对象的策略。
```java
public interface ResourcePatternResolver extends ResourceLoader {

	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

	Resource[] getResources(String locationPattern) throws IOException;
}
```
如上所示，该接口还为类路径中的所有匹配资源定义了一个特殊的classpath*: resource前缀。
注意，在这种情况下，资源位置应该是一个没有占位符的路径—例如，classpath*:/config/beans.xml。
JAR文件或类路径中的不同目录可以包含具有相同路径和相同名称的多个文件。请参阅应用程序上下文构造函数资源路径中的通配符及其子部分，以获得有关使用classpath*: Resource前缀支持通配符的更多详细信息。

可以检查传入的ResourceLoader(例如，通过ResourceLoaderAware语义提供的一个)是否也实现了这个扩展接口。

PathMatchingResourcePatternResolver是一个独立的实现，可以在ApplicationContext之外使用，也可以被ResourceArrayPropertyEditor用于填充Resource[] bean属性。
PathMatchingResourcePatternResolver能够将指定的资源位置路径解析为一个或多个匹配的资源对象。
源路径可以是一个与目标资源一对一映射的简单路径，也可以包含特殊的类路径*:前缀和/或内部ant风格的正则表达式(使用Spring的org.springframework.util.AntPathMatcher实用程序进行匹配)。后两者都是有效的通配符。

任何标准ApplicationContext中的默认ResourceLoader实际上是PathMatchingResourcePatternResolver的一个实例，它实现了ResourcePatternResolver接口。ApplicationContext实例本身也是如此，它也实现了ResourcePatternResolver接口，并委托给默认的PathMatchingResourcePatternResolver。



# ResourceLoaderAware接口

ResourceLoaderAware接口是一个特殊的回调接口，用于标识希望被提供ResourceLoader引用的组件。下面的清单显示了ResourceLoaderAware接口的定义:
```java
public interface ResourceLoaderAware {

	void setResourceLoader(ResourceLoader resourceLoader);
}
```
当一个类实现ResourceLoaderAware并被部署到应用程序上下文中(作为spring管理的bean)时，应用程序上下文将其识别为ResourceLoaderAware。然后，应用程序上下文调用setResourceLoader(ResourceLoader)，将自身作为参数提供(记住，Spring中的所有应用程序上下文都实现了ResourceLoader接口)。

由于ApplicationContext是一个ResourceLoader, bean也可以实现ApplicationContextAware接口，并使用提供的应用程序上下文直接加载资源。但是，一般来说，如果您只需要专用的ResourceLoader接口，则最好使用该接口。代码将只耦合到资源加载接口(可以认为是一个实用程序接口)，而不是耦合到整个Spring ApplicationContext接口。

在应用程序组件中，您也可以依赖于自动装配ResourceLoader，作为实现ResourceLoaderAware接口的替代方案。传统的构造函数和byType自动装配模式(如自动装配协作器中所述)能够分别为构造函数参数或setter方法参数提供ResourceLoader。为了获得更大的灵活性(包括自动装配字段和多参数方法的能力)，可以考虑使用基于注释的自动装配特性。在这种情况下，只要有问题的字段、构造函数或方法带有@Autowired注释，那么ResourceLoader就会自动连接到一个需要使用ResourceLoader类型的字段、构造函数参数或方法参数中。
有关更多信息，请参见使用@Autowired。

要为包含通配符的资源路径加载一个或多个Resource对象，或者使用特殊的classpath*: Resource前缀，请考虑在应用程序组件中自动连接一个ResourcePatternResolver实例，而不是ResourceLoader。

### resources作为依赖项
如果bean本身将通过某种动态过程确定并提供资源路径，那么bean使用ResourceLoader或ResourcePatternResolver接口来加载资源可能是有意义的。
例如，考虑加载某种类型的模板，其中所需的特定资源取决于用户的角色。如果资源是静态的，那么完全消除ResourceLoader接口(或ResourcePatternResolver接口)的使用是有意义的，让bean公开它需要的Resource属性，并期望它们被注入到它中。

使注入这些属性变得简单的是，所有应用程序上下文都注册并使用一个特殊的JavaBeans PropertyEditor，它可以将字符串路径转换为资源对象。例如，下面的MyBean类有一个Resource类型的模板属性。
```java
public class MyBean {

	private Resource template;

	public setTemplate(Resource template) {
		this.template = template;
	}

	// ...
}
```

在XML配置文件中，可以用一个简单的字符串来配置模板属性，如下面的例子所示:
```xml
<bean id="myBean" class="example.MyBean">
	<property name="template" value="some/resource/path/myTemplate.txt"/>
</bean>
```

注意，资源路径没有前缀。因此，由于应用程序上下文本身将被用作ResourceLoader，因此资源将通过ClassPathResource、FileSystemResource或ServletContextResource加载，具体取决于应用程序上下文的确切类型。如果需要强制使用特定的Resource类型，可以使用前缀。下面两个例子展示了如何强制ClassPathResource和UrlResource(后者用于访问文件系统中的文件):
```xml
<property name="template" value="classpath:some/resource/path/myTemplate.txt">
```

```xml
<property name="template" value="file:///some/resource/path/myTemplate.txt"/>
```
如果MyBean类被重构以使用注释驱动的配置，myTemplate.txt的路径可以存储在一个名为template的键下。
路径——例如，在Spring环境可用的属性文件中(参见环境抽象)。
然后可以使用属性占位符通过@Value注释引用模板路径(参见使用@Value)。Spring将以字符串的形式检索模板路径的值，一个特殊的PropertyEditor将把字符串转换为一个Resource对象，然后注入到MyBean构造函数中。下面的示例演示了如何实现这一点。
```java
@Component
public class MyBean {

	private final Resource template;

	public MyBean(@Value("${template.path}") Resource template) {
		this.template = template;
	}

	// ...
}
```
如果我们希望支持在类路径的多个位置的同一路径下发现多个模板——例如，在类路径的多个jar中——我们可以使用特殊的classpath*:前缀和通配符来定义模板。路径键为classpath*:/config/templates/*.txt。如果我们像下面这样重新定义MyBean类，Spring将把模板路径模式转换为一个可以注入到MyBean构造函数中的Resource对象数组。
```java
@Component
public class MyBean {

	private final Resource[] templates;

	public MyBean(@Value("${templates.path}") Resource[] templates) {
		this.templates = templates;
	}

	// ...
}
```



### 应用程序上下文和资源路径

本节介绍如何使用资源创建应用程序上下文，包括使用XML的快捷方式、如何使用通配符以及其他细节。



# 构建应用程序上下文

应用程序上下文构造函数(针对特定的应用程序上下文类型)通常使用字符串或字符串数组作为资源的位置路径，例如组成上下文定义的XML文件。当这样的位置路径没有前缀时，从该路径构建并用于加载bean定义的特定Resource类型取决于特定的应用程序上下文，并且适合于该应用程序上下文。例如，考虑下面的例子，它创建了一个ClassPathXmlApplicationContext:
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("conf/appContext.xml");
```
bean定义是从类路径加载的，因为使用了ClassPathResource。但是，考虑下面的例子，它创建了一个FileSystemXmlApplicationContext:
```java
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("conf/appContext.xml");
```
现在，bean定义从文件系统位置加载(在本例中，相对于当前工作目录)。

注意，在位置路径上使用特殊的类路径前缀或标准URL前缀会覆盖为加载bean定义而创建的Resource的默认类型。考虑下面的例子:
```java
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("classpath:conf/appContext.xml");
```
使用FileSystemXmlApplicationContext从类路径加载bean定义。
然而，它仍然是一个FileSystemXmlApplicationContext。如果它随后被用作ResourceLoader，则任何未加前缀的路径仍被视为文件系统路径。



# 构造ClassPathXmlApplicationContext实例-快捷方式
ClassPathXmlApplicationContext公开了许多构造函数来实现方便的实例化。
基本思想是，您可以只提供一个字符串数组，其中只包含XML文件本身的文件名(不包含前导路径信息)，还可以提供一个Class。
然后，ClassPathXmlApplicationContext从提供的类中派生路径信息。
考虑以下目录:
```txt
com/
  example/
    services.xml
    repositories.xml
    MessengerService.class
```
下面的例子展示了如何实例化一个由services.xml和repository .xml文件中定义的bean组成的ClassPathXmlApplicationContext实例(它们位于类路径上):
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext(
	new String[] {"services.xml", "repositories.xml"}, MessengerService.class);
```
有关各种构造函数的详细信息，请参阅ClassPathXmlApplicationContext javadoc。



# 应用程序上下文构造函数资源路径中的通配符

应用程序上下文构造函数值中的资源路径可以是简单路径(如前面所示)，每个路径都有到目标资源的一对一映射，或者可以包含特殊的类路径*:前缀或内部ant风格模式(通过使用Spring的PathMatcher实用程序进行匹配)。
后两者都是有效的通配符。

这种机制的一个用途是需要进行组件样式的应用程序组装。所有组件都可以将上下文定义片段发布到一个已知的位置路径，并且，当最终的应用程序上下文使用以classpath*:为前缀的相同路径创建时，所有组件片段都会被自动拾取。

注意，这种通配符特定于在应用程序上下文构造函数中使用资源路径(或者当您直接使用PathMatcher实用程序类层次结构时)，并在构造时解决。
它与Resource类型本身无关。你不能使用类路径*:前缀来构造一个实际的资源，因为一个资源一次只指向一个资源。

### Ant-style 	
路径位置可以包含ant风格的模式，如下例所示:
```txt
/WEB-INF/*-context.xml
com/mycompany/**/applicationContext.xml
file:C:/some/path/*-context.xml
classpath:com/mycompany/**/applicationContext.xml
```
当路径位置包含ant样式模式时，解析器将遵循一个更复杂的过程来尝试解析通配符。
它为直到最后一个非通配符段的路径生成一个Resource，并从中获取一个URL。
如果此URL不是jar: URL或特定于容器的变体(例如WebLogic中的zip:、WebSphere中的wsjar等等)，则从中获取java.io.File，并通过遍历文件系统来解析通配符。
在jar URL的情况下，解析器要么从中获取java.net.JarURLConnection，要么手动解析jar URL，然后遍历jar文件的内容以解析通配符。

#### 对可移植性的影响
如果指定的路径已经是一个文件URL(要么隐式地，因为基本的ResourceLoader是一个文件系统URL，要么显式地)，通配符保证以完全可移植的方式工作。

如果指定的路径是一个类路径位置，解析器必须通过调用Classloader.getResource()来获取最后一个非通配符路径段URL。
由于这只是路径的一个节点(而不是最后的文件)，因此实际上(在ClassLoader javadoc中)在这种情况下返回的URL类型是未定义的。
在实践中，它总是一个java.io.File，表示目录(类路径资源在其中解析为文件系统位置)或某种jar URL(类路径资源在其中解析为jar位置)。但是，这个操作存在可移植性问题。

如果获得了最后一个非通配符段的jar URL，则解析器必须能够从中获得java.net.JarURLConnection或手动解析jar URL，以便能够遍历jar的内容并解析通配符。
这在大多数环境中都可以工作，但在其他环境中会失败，我们强烈建议在依赖它之前，在您的特定环境中彻底测试来自jar的资源的通配符解析。

### classpath*: 前缀
在构造基于xml的应用程序上下文时，位置字符串可以使用特殊的classpath*:前缀，如下例所示:
```java
ApplicationContext ctx =
	new ClassPathXmlApplicationContext("classpath*:conf/appContext.xml");
```
这个特殊的前缀指定必须获得与给定名称匹配的所有类路径资源(在内部，这实际上是通过调用ClassLoader.getResources(…)来实现的)，然后合并形成最终的应用程序上下文定义。

通配符类路径依赖于底层ClassLoader的getResources()方法。
由于现在大多数应用服务器都提供自己的ClassLoader实现，因此行为可能有所不同，特别是在处理jar文件时。
检查classpath*是否工作的一个简单测试是使用ClassLoader从classpath上的jar中加载文件:getClass(). getclassloader (). getresources ("<someFileInsideTheJar>")。
使用具有相同名称但位于两个不同位置的文件尝试此测试—例如，具有相同名称和相同路径的文件，但在类路径上的不同jar中。
如果返回不适当的结果，请检查应用服务器文档中可能影响ClassLoader行为的设置。

您还可以在位置路径的其余部分(例如，classpath*:META-INF/*-beans.xml)中结合使用classpath*:前缀和PathMatcher模式。
在这种情况下，解析策略相当简单:在最后一个非通配符路径段上使用ClassLoader.getResources()调用来获取类加载器层次结构中的所有匹配资源，然后在每个资源之外，对通配符子路径使用前面描述的相同PathMatcher解析策略。



# 关于通配符的其他注意事项
注意，classpath*:与ant样式模式结合使用时，在模式启动之前只能可靠地与至少一个根目录一起工作，除非实际的目标文件位于文件系统中。
这意味着像classpath*:*.xml这样的模式可能不会从jar文件的根目录检索文件，而只能从扩展目录的根目录检索文件。

Spring检索类路径条目的能力源于JDK的ClassLoader.getResources()方法，该方法仅为空字符串返回文件系统位置(指示要搜索的潜在根)。
Spring也会评估URLClassLoader运行时配置和jar文件中的java.class.path清单，但这不能保证会导致可移植的行为。

扫描类路径包需要在类路径中存在相应的目录项。当您使用Ant构建JAR时，不要激活JAR任务的仅文件开关。
此外，在某些环境中，基于安全策略，类路径目录可能不会被公开——例如，JDK 1.7.0_45及更高版本上的独立应用程序(需要在清单中设置“Trusted-Library”)。
见stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources)。



# FileSystemResource警告

没有附加到FileSystemApplicationContext的FileSystemResource(也就是说，当FileSystemApplicationContext不是实际的ResourceLoader时)按照您所期望的方式处理绝对路径和相对路径。相对路径是相对于当前工作目录的，而绝对路径是相对于文件系统根目录的。

但是，由于向后兼容性(历史)原因，当FileSystemApplicationContext是ResourceLoader时，这种情况会发生变化。FileSystemApplicationContext强制所有附加的FileSystemResource实例将所有位置路径视为相对路径，无论它们是否以斜杠开头。
在实践中，这意味着以下示例是等效的:

```java
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("conf/context.xml");
```

```java
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("/conf/context.xml");
```

下面的例子也是等价的(即使它们不同也有意义，因为一种情况是相对的，另一种情况是绝对的):
```java
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("some/resource/path/myTemplate.txt");
```

```java
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("/some/resource/path/myTemplate.txt");
```

在实践中，如果您需要真正的绝对文件系统路径，您应该避免与FileSystemResource或FileSystemXmlApplicationContext一起使用绝对路径，并通过使用file: URL前缀强制使用UrlResource。下面的例子展示了如何这样做:
```java
// actual context type doesn't matter, the Resource will always be UrlResource
ctx.getResource("file:///some/resource/path/myTemplate.txt");
```

```java
// force this FileSystemXmlApplicationContext to load its definition via a UrlResource
ApplicationContext ctx =
	new FileSystemXmlApplicationContext("file:///conf/context.xml");
```