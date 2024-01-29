# 使用泛型作为自动装配限定符

除了@Qualifier注释之外，还可以使用Java泛型类型作为隐式的限定形式。例如，假设您有以下配置:

```java
@Configuration
public class MyConfiguration {

	@Bean
	public StringStore stringStore() {
		return new StringStore();
	}

	@Bean
	public IntegerStore integerStore() {
		return new IntegerStore();
	}
}
```

假设前面的bean实现了一个泛型接口(即Store<String>和Store<Integer>)，您可以@Autowire Store接口，并将泛型用作限定符，如下面的示例所示:

```java
@Autowired
private Store<String> s1; // <String> qualifier, injects the stringStore bean

@Autowired
private Store<Integer> s2; // <Integer> qualifier, injects the integerStore bean
```

泛型限定符也适用于自动装配列表、Map实例和数组。下面的例子自动连接一个泛型列表:

```java
// Inject all Store beans as long as they have an <Integer> generic
// Store<String> beans will not appear in this list
@Autowired
private List<Store<Integer>> s;
```



