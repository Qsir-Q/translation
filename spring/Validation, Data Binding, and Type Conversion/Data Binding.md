### 数据绑定
数据绑定对于将用户输入绑定到目标对象非常有用，其中用户输入是一个映射，属性路径作为键，遵循JavaBeans约定。DataBinder是支持此功能的主要类，它提供了两种绑定用户输入的方法:

- 构造器绑定 将用户输入绑定到公共数据构造函数，在用户输入中查找构造函数参数值
- 属性绑定   将用户输入绑定到setter，将用户输入的键与目标对象结构的属性进行匹配

#### 构造器绑定
要使用构造函数绑定:
- 创建一个以null作为目标对象的DataBinder
- 将targetType设置为目标类。
- 调用构造
目标类应该有一个公共构造函数或一个带参数的非公共构造函数。如果有多个构造函数，则使用默认构造函数(如果存在)。
默认情况下，构造函数参数名用于查找参数值，但您可以配置namesolver。
Spring MVC和WebFlux都依赖于允许通过构造函数参数上的@BindParam注释来定制要绑定的值的名称。

根据需要应用类型转换来转换用户输入。如果构造函数参数是一个对象，则以相同的方式递归地构造它，但通过嵌套的属性路径。这意味着构造函数绑定创建目标对象和它包含的任何对象。

绑定和转换错误反映在DataBinder的BindingResult中。如果目标创建成功，则在调用构造之后将目标设置为创建的实例。

#### 使用BeanWrapper进行属性绑定
org.springframework.beans包遵循JavaBeans标准。JavaBean是一个具有默认无参数构造函数的类，它遵循命名约定，其中(例如)名为bingoMadness的属性将具有setter方法setBingoMadness(..)和getter方法getBingoMadness()。有关JavaBeans和规范的更多信息，请参见JavaBeans。

bean包中一个非常重要的类是BeanWrapper接口及其相应的实现(BeanWrapperImpl)。正如javadoc中引用的那样，BeanWrapper提供了设置和获取属性值(单独或批量)、获取属性描述符和查询属性以确定它们是可读还是可写的功能。此外，BeanWrapper还提供了对嵌套属性的支持，允许在子属性上无限深度地设置属性。
BeanWrapper还支持添加标准JavaBeans propertychangelistener和vetoablechangelistener的功能，而不需要在目标类中支持代码。
最后但并非最不重要的一点是，BeanWrapper提供了设置索引属性的支持。BeanWrapper通常不是由应用程序代码直接使用，而是由DataBinder和BeanFactory使用。

BeanWrapper的工作方式部分由其名称表示:它包装bean以在该bean上执行操作，例如设置和检索属性。

#### 设置和获取基本属性和嵌套属性
设置和获取属性是通过BeanWrapper的setPropertyValue和getPropertyValue重载方法变体完成的。
详细信息请参见Javadoc。下表显示了这些约定的一些例子:
-  name 与getName()或isName()和setName(..)方法对应的属性名。
-  account.name  表示对应于(例如)getAccount(). setname()或getAccount(). getname()方法的属性帐户的嵌套属性名称。
-  account[2]    指示索引属性帐户的第三个元素。索引属性的类型可以是数组、列表或其他自然排序的集合。
-  account[COMPANYNAME]   指示由帐户map属性的COMPANYNAME键索引的映射条目的值。
(如果您不打算直接使用BeanWrapper，那么下一节对您来说并不重要。如果您只使用DataBinder和BeanFactory以及它们的默认实现，那么您应该跳到有关propertyeditor的部分。

下面的两个示例类使用BeanWrapper来获取和设置属性:
```java
public class Company {

	private String name;
	private Employee managingDirector;

	public String getName() {
		return this.name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Employee getManagingDirector() {
		return this.managingDirector;
	}

	public void setManagingDirector(Employee managingDirector) {
		this.managingDirector = managingDirector;
	}
}
```

```java
public class Employee {

	private String name;

	private float salary;

	public String getName() {
		return this.name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public float getSalary() {
		return salary;
	}

	public void setSalary(float salary) {
		this.salary = salary;
	}
}
```
下面的代码片段展示了一些如何检索和操作实例化Companys和Employees的一些属性的示例:
```java
BeanWrapper company = new BeanWrapperImpl(new Company());
// setting the company name..
company.setPropertyValue("name", "Some Company Inc.");
// ... can also be done like this:
PropertyValue value = new PropertyValue("name", "Some Company Inc.");
company.setPropertyValue(value);

// ok, let's create the director and tie it to the company:
BeanWrapper jim = new BeanWrapperImpl(new Employee());
jim.setPropertyValue("name", "Jim Stravinsky");
company.setPropertyValue("managingDirector", jim.getWrappedInstance());

// retrieving the salary of the managingDirector through the company
Float salary = (Float) company.getPropertyValue("managingDirector.salary");
```

#### PropertyEditor's
Spring使用PropertyEditor的概念来实现对象和字符串之间的转换。用不同于对象本身的方式来表示属性是很方便的。例如，Date可以以人类可读的方式表示(如String: '2007-14-09')，而我们仍然可以将人类可读的形式转换回原始日期(或者，更好的是，将以人类可读形式输入的任何日期转换回Date对象)。这种行为可以通过注册java.beans.PropertyEditor类型的自定义编辑器来实现。在BeanWrapper上注册自定义编辑器，或者在特定的IoC容器中注册(如前一章所述)，使其了解如何将属性转换为所需的类型。
有关PropertyEditor的更多信息，请参见java. xml的javadoc。从Oracle获取beans包。

在Spring中使用属性编辑的几个例子:
- 通过使用PropertyEditor实现来设置bean上的属性。当您使用String作为在XML文件中声明的某些bean的属性值时，Spring(如果相应属性的setter具有Class参数)将使用classseditor尝试将参数解析为Class对象。
- 在Spring的MVC框架中解析HTTP请求参数是通过使用各种PropertyEditor实现来完成的，您可以在CommandController的所有子类中手动绑定这些实现。

Spring有许多内置的PropertyEditor实现，使工作变得简单。它们都位于org.springframework.beans.propertyeditor包中。
默认情况下，大多数(但不是全部，如下表所示)都是由BeanWrapperImpl注册的。在属性编辑器以某种方式可配置的地方，您仍然可以注册自己的变体来覆盖默认的变体。下表描述了Spring提供的各种PropertyEditor实现:

1. ByteArrayPropertyEditor  字节数组编辑器。将字符串转换为相应的字节表示形式。BeanWrapperImpl默认注册。
2. ClassEditor  将表示类的字符串解析为实际类，反之亦然。当没有找到类时，抛出一个IllegalArgumentException。默认情况下，由BeanWrapperImpl注册。
3. CustomBooleanEditor 布尔属性的可定制属性编辑器。默认情况下，由BeanWrapperImpl注册，但可以通过将其自定义实例注册为自定义编辑器来覆盖它。
4. CustomCollectionEditor 属性编辑器，将任何源集合转换为给定的目标集合类型。
5. CustomDateEditor  可定制的java.util属性编辑器。日期，支持自定义日期格式。默认情况下未注册。必须根据需要使用适当的格式进行用户注册。
6. CustomNumberEditor 任何数字子类(如Integer、Long、Float或Double)的可自定义属性编辑器。默认情况下，由BeanWrapperImpl注册，但可以通过将其自定义实例注册为自定义编辑器来覆盖它。
7. FileEditor   将字符串解析为java.io.File对象。默认情况下，由BeanWrapperImpl注册。
8. InputStreamEditor  单向属性编辑器，可以接受字符串并产生(通过中间的ResourceEditor和Resource) InputStream，这样InputStream属性可以直接设置为字符串。请注意，默认用法不会为您关闭InputStream。默认情况下，由BeanWrapperImpl注册。
9. LocaleEditor       可以将字符串解析为Locale对象，反之亦然(字符串格式为[language]_[country]_[variant]，与Locale的toString()方法相同)。也接受空格作为分隔符，作为下划线的替代。默认情况下，由BeanWrapperImpl注册。
10. PatternEditor     可以将字符串解析为java.util.regex.Pattern对象，反之亦然。
11. PropertiesEditor   可以将字符串(使用java.util.Properties类的javadoc中定义的格式进行格式化)转换为Properties对象。默认情况下，由BeanWrapperImpl注册。
12. StringTrimmerEditor  属性编辑器，用于编辑字符串。可选地允许将空字符串转换为空值。默认为未注册-必须为用户注册。
13. URLEditor   可以将URL的字符串表示形式解析为实际的URL对象。默认情况下，由BeanWrapperImpl注册。

Spring使用java.beans.PropertyEditorManager为可能需要的属性编辑器设置搜索路径。搜索路径还包括sun.bean.editors，其中包括用于Font, Color等类型和大多数基本类型的PropertyEditor实现。还要注意的是，如果PropertyEditor类与它们处理的类在同一个包中，并且具有与该类相同的名称，则标准JavaBeans基础结构会自动发现PropertyEditor类(无需显式注册它们)，并附加了Editor。
例如，可以有以下的类和包结构，这将足以使SomethingEditor类被识别并用作某事类型属性的PropertyEditor。

```txt
com
  chank
    pop
      Something
      SomethingEditor // the PropertyEditor for the Something class
```

注意，您也可以在这里使用标准的BeanInfo JavaBeans机制(这里在一定程度上进行了描述)。
下面的例子使用BeanInfo机制显式地将一个或多个PropertyEditor实例注册为关联类的属性:
```txt
com
  chank
    pop
      Something
      SomethingBeanInfo // the BeanInfo for the Something class
```

下面引用的SomethingBeanInfo类的Java源代码将一个CustomNumberEditor与SomethingBeanInfo类的age属性关联起来:  
```java
public class SomethingBeanInfo extends SimpleBeanInfo {

	public PropertyDescriptor[] getPropertyDescriptors() {
		try {
			final PropertyEditor numberPE = new CustomNumberEditor(Integer.class, true);
			PropertyDescriptor ageDescriptor = new PropertyDescriptor("age", Something.class) {
				@Override
				public PropertyEditor createPropertyEditor(Object bean) {
					return numberPE;
				}
			};
			return new PropertyDescriptor[] { ageDescriptor };
		}
		catch (IntrospectionException ex) {
			throw new Error(ex.toString());
		}
	}
}
```

#### 自定义PropertyEditor's
当将bean属性设置为字符串值时，Spring IoC容器最终使用标准JavaBeans PropertyEditor实现将这些字符串转换为属性的复杂类型。

Spring预先注册了许多自定义PropertyEditor实现(例如，将表示为字符串的类名转换为class对象)。
此外，Java的标准JavaBeans PropertyEditor查找机制允许适当地命名类的PropertyEditor，并将其放置在与其提供支持的类相同的包中，以便可以自动找到它。

如果需要注册其他自定义propertyeditor，有几种机制可用。
最手动的方法(通常不方便也不推荐)是使用ConfigurableBeanFactory接口的registerCustomEditor()方法，假设您有一个BeanFactory引用。另一种(更方便的)机制是使用名为CustomEditorConfigurer的特殊bean工厂后处理器。虽然您可以将bean工厂后置处理器与BeanFactory实现一起使用，但CustomEditorConfigurer有一个嵌套的属性设置，因此我们强烈建议您将其与ApplicationContext一起使用，您可以在其中以类似于任何其他bean的方式部署它，并且可以自动检测和应用它。

请注意，所有bean工厂和应用程序上下文都自动使用许多内置属性编辑器，通过使用BeanWrapper来处理属性转换。前一节列出了BeanWrapper注册的标准属性编辑器。此外，ApplicationContexts还覆盖或添加额外的编辑器，以适合特定应用程序上下文类型的方式处理资源查找。

标准JavaBeans PropertyEditor实例用于将表示为字符串的属性值转换为属性的实际复杂类型。
您可以使用CustomEditorConfigurer(一个bean工厂后处理器)，方便地为ApplicationContext添加对额外PropertyEditor实例的支持。

考虑下面的例子，它定义了一个名为ExoticType的用户类和另一个名为DependsOnExoticType的类，后者需要将ExoticType设置为属性:
```java
package example;

public class ExoticType {

	private String name;

	public ExoticType(String name) {
		this.name = name;
	}
}

public class DependsOnExoticType {

	private ExoticType type;

	public void setType(ExoticType type) {
		this.type = type;
	}
}
```

当一切设置妥当后，我们希望能够将type属性分配为字符串，PropertyEditor将其转换为实际的ExoticType实例。下面的bean定义展示了如何建立这种关系:
```xml
<bean id="sample" class="example.DependsOnExoticType">
	<property name="type" value="aNameForExoticType"/>
</bean>
```

PropertyEditor的实现如下所示:
```java
package example;

import java.beans.PropertyEditorSupport;

// converts string representation to ExoticType object
public class ExoticTypeEditor extends PropertyEditorSupport {

	public void setAsText(String text) {
		setValue(new ExoticType(text.toUpperCase()));
	}
}
```
最后，下面的例子展示了如何使用CustomEditorConfigurer向ApplicationContext注册新的PropertyEditor，这样ApplicationContext就可以根据需要使用它了:
```java
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
	<property name="customEditors">
		<map>
			<entry key="example.ExoticType" value="example.ExoticTypeEditor"/>
		</map>
	</property>
</bean>
```

#### PropertyEditorRegistrar
在Spring容器中注册属性编辑器的另一种机制是创建并使用PropertyEditorRegistrar。当您需要在几种不同的情况下使用同一组属性编辑器时，此接口特别有用。您可以编写相应的注册器并在每种情况下重用它。
PropertyEditorRegistrar实例与一个名为PropertyEditorRegistry的接口一起工作，该接口由Spring BeanWrapper(和DataBinder)实现。PropertyEditorRegistrar实例在与CustomEditorConfigurer(此处描述)结合使用时特别方便，后者公开了一个名为setPropertyEditorRegistrars(..)的属性。
以这种方式添加到CustomEditorConfigurer中的PropertyEditorRegistrar实例可以很容易地与DataBinder和Spring MVC控制器共享。
此外，它避免了对自定义编辑器进行同步的需要:期望PropertyEditorRegistrar为每个bean创建尝试创建新的PropertyEditor实例。

下面的例子展示了如何创建你自己的PropertyEditorRegistrar实现:
```java
package com.foo.editors.spring;

public final class CustomPropertyEditorRegistrar implements PropertyEditorRegistrar {

	public void registerCustomEditors(PropertyEditorRegistry registry) {

		// it is expected that new PropertyEditor instances are created
		registry.registerCustomEditor(ExoticType.class, new ExoticTypeEditor());

		// you could register as many custom property editors as are required here...
	}
}
```
另请参见org.springframework.beans.support.ResourceEditorRegistrar，以获得PropertyEditorRegistrar实现的示例。
注意，在registerCustomEditors(..)方法的实现中，它如何创建每个属性编辑器的新实例。

下一个例子展示了如何配置一个CustomEditorConfigurer，并注入一个CustomPropertyEditorRegistrar的实例:
```xml
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
	<property name="propertyEditorRegistrars">
		<list>
			<ref bean="customPropertyEditorRegistrar"/>
		</list>
	</property>
</bean>

<bean id="customPropertyEditorRegistrar"
	class="com.foo.editors.spring.CustomPropertyEditorRegistrar"/>
```
最后，对于那些使用Spring MVC web框架的人来说(这与本章的重点有点不同)，使用propertyeditorregistrator与数据绑定web控制器结合使用会非常方便。下面的例子在@InitBinder方法的实现中使用了PropertyEditorRegistrar:
```java
@Controller
public class RegisterUserController {

	private final PropertyEditorRegistrar customPropertyEditorRegistrar;

	RegisterUserController(PropertyEditorRegistrar propertyEditorRegistrar) {
		this.customPropertyEditorRegistrar = propertyEditorRegistrar;
	}

	@InitBinder
	void initBinder(WebDataBinder binder) {
		this.customPropertyEditorRegistrar.registerCustomEditors(binder);
	}

	// other methods related to registering a User
}
```
这种风格的PropertyEditor注册可以导致简洁的代码(@InitBinder方法的实现只有一行长)，并让常见的PropertyEditor注册代码被封装在一个类中，然后在需要的许多控制器之间共享。