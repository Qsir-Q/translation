# 使用Spring的验证器接口进行验证
Spring提供了一个Validator接口，您可以使用它来验证对象。Validator接口通过使用Errors对象来工作，以便在进行验证时，验证器可以向Errors对象报告验证失败。

考虑下面一个对象的例子:
```java
public class Person {

	private String name;
	private int age;

	// the usual getters and setters...
}
```
下一个示例通过实现org.springframework. validate . validator接口的以下两个方法，为Person类提供验证行为:
supports(Class): 这个验证器能验证所提供类的实例吗?validate(Object, org.springframework.validation.Errors): 验证给定的对象，如果出现验证错误，将这些错误注册到给定的errors对象。实现验证器相当简单，特别是当您知道Spring框架也提供了ValidationUtils帮助器类时。下面的例子实现了Person实例的Validator:

```java
public class PersonValidator implements Validator {

	/**
	 * This Validator validates only Person instances
	 */
	public boolean supports(Class clazz) {
		return Person.class.equals(clazz);
	}

	public void validate(Object obj, Errors e) {
		ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
		Person p = (Person) obj;
		if (p.getAge() < 0) {
			e.rejectValue("age", "negativevalue");
		} else if (p.getAge() > 110) {
			e.rejectValue("age", "too.darn.old");
		}
	}
}
```

ValidationUtils类上的静态rejectIfEmpty(..)方法用于在name属性为null或空字符串时拒绝该属性。

虽然实现单个Validator类来验证富对象中的每个嵌套对象当然是可能的，但是最好将每个嵌套对象类的验证逻辑封装在自己的Validator实现中。“富”对象的一个简单示例是由两个String属性(一个名和一个名)和一个复杂的Address对象组成的Customer。地址对象可以独立于客户对象使用，因此已经实现了一个不同的AddressValidator。如果你想让你的CustomerValidator重用AddressValidator类中包含的逻辑，而不需要复制粘贴，你可以在你的CustomerValidator中注入依赖或实例化一个AddressValidator，如下面的例子所示:
```java
public class CustomerValidator implements Validator {

	private final Validator addressValidator;

	public CustomerValidator(Validator addressValidator) {
		if (addressValidator == null) {
			throw new IllegalArgumentException("The supplied [Validator] is " +
				"required and must not be null.");
		}
		if (!addressValidator.supports(Address.class)) {
			throw new IllegalArgumentException("The supplied [Validator] must " +
				"support the validation of [Address] instances.");
		}
		this.addressValidator = addressValidator;
	}

	/**
	 * This Validator validates Customer instances, and any subclasses of Customer too
	 */
	public boolean supports(Class clazz) {
		return Customer.class.isAssignableFrom(clazz);
	}

	public void validate(Object target, Errors errors) {
		ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "field.required");
		ValidationUtils.rejectIfEmptyOrWhitespace(errors, "surname", "field.required");
		Customer customer = (Customer) target;
		try {
			errors.pushNestedPath("address");
			ValidationUtils.invokeValidator(this.addressValidator, customer.getAddress(), errors);
		} finally {
			errors.popNestedPath();
		}
	}
}
```

将验证错误报告给传递给验证器的errors对象。在Spring Web MVC的情况下，您可以使用< Spring:bind/>标签来检查错误消息，但是您也可以自己检查Errors对象。关于它提供的方法的更多信息可以在javadoc中找到。

验证器也可以在本地被调用，以便对给定对象进行即时验证，而不涉及绑定过程。在6.1版本中，这已经通过一个新的Validator.validateObject(Object)方法得到了简化，该方法现在默认可用，返回一个简单的' Errors '表示，可以检查:通常调用hasErrors()或新的failOnError方法将错误摘要消息转换为异常(例如Validator.validateObject(myObject).failOnError(IllegalArgumentException::new))。