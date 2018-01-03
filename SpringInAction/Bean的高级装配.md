

## 处理歧义

@Qualifier 指定Bean的别名

```java
//在Bean的声明处
@Component
@Qualifier("code")
public class IceCream implements Dessert{...}
```

```java
//注入的地方引用限定符
@Autowired
@Qualifier("code")
public void setDessert(Dessert dessert){
 ...
}
```

在使用javaConfig的时候也可以这么做：

```java
@Bean
@Qualifier("code")
public Dessert iceCream(){
  return enw IceCream();
}
```

或者我们自定义限定符

```java
@Target({ElementType.CONSTRUCTOR, ElementType.FIELD,ElementType.METHOD,ElementType.TYPE})//表示注解可以用在哪里
@Retention(RetentionPolicy.RUNTIME)//表示注解在什么级别上保存，这里时运行时
@Qualifier               
public @interface Creamy{}
```

## bean的作用域

Singleton单例（默认），Prototype每次注入或者通过Context获取都会创建一个新实例，Session在Web应用中为每一个会话创建一个Bean，Request为每个请求创建一个Bean。

@Scope指定注解

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class Notepad{...}
```

在javaConfig中：

```java
@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public Notepad notepad(){
  return new Notepad();
}
```

在XML中：

```java
<bean id="notepad" class="com.myapp.Notepad" scope="prototype"/>
```

在购物车这个场景下，用Session作用域比较合适：

```java
@Component
@Scope(value=WebApplicationContext.SCOPE_SESSION,
      proxyMode=ScopedProxyMode.INTERFACES)
public ShoppingCart cart(){
}
//proxyMode=ScopedProxyMode.INTERFACES这个比较有意思
//ShoppingService注入ShoppingCart,但是我们不希望把一个特定的ShoppingCart注入到ShoppingService中，应该是等到session创建的时候才能初始化一个Cart然后注入，也就是注入时机是用户访问的时候。
//因此这里Spring会注入一个代理，在调用的时候才会委托给会话作用域中真正的ShoppingCart bean
```

![image](http://note.youdao.com/yws/public/resource/1f530a42ccf1941d7b5e6d2e23556c1a/xmlnote/597C388CFC3F4B7FA457FF27B85D7768/1263)

XML中：

```xml
<bean id="cart"
  	class="com.myapp.ShoppingCart"
      scope="session">
      <aop:scoped-proxy proxy-target-class="false"/><!--实现代理,并实现基于接口的代理 -->
<bean>
```

## SpringEL和属性占位符

先看普通的方式：

```properties
//properties
disc.title="test"
disc.artist="aaaaaaaaa"
```

```java
@Component
@PropertySource("classpath:test.properties")
public class Notepad {
  // the details of this class are inconsequential to this example
    @Autowired
    Environment environment;

    public UniqueThing getUniqueThing(){
        return new UniqueThing(
                environment.getProperty("disc.title"),
                environment.getProperty("disc.artist")
        );
    }
}
```

我们用Environment.getProperty方式获取属性。

属性占位符的方式：在Spring的装配中，属性占位符的形式为"${...}"，例如：

```java
//定义一个配置，有该配置才能使用属性占位符
@Configuration
@ComponentScan(excludeFilters={@Filter(type=FilterType.ANNOTATION, value=Configuration.class)})
public class ComponentScannedConfig {
    @Bean
    public
    static PropertySourcesPlaceholderConfigurer placeholderConfigurer(){
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```

or再xml中

```xml
<context:propertyplaceholder/>
```

使用占位符：

```java
@Component
public class UniqueThing {
  // the details of this class are inconsequential to this example
    @Value("${disc.title}")
    private String aa;
    @Value("${disc.artist}")
    private String bb;
    private UniqueThing(String test1, String test2){
        this.aa = test1;
        this.bb  = test2;
    }
    private UniqueThing(){}
    public String toString(){
        return aa+" "+bb;
    }
}
```

**SpEL**

SpEL要放入#{...}中去，例如：

```xml
#{1}
```

引用属性：

```xml
#{systemProperties['disc.title']}
```

在Bean装配的时候使用

```java
  @Value("#{systemProperties['disc.title']}")
```

使用java库

```java
"#{T(java.lang.Math).PI*circle.radius^2}"
```











