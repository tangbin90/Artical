## 面向切面编程

一些术语：通知（advice）、切点（pointcut）、连接点（join point）。他们的关系如下：

![image](http://note.youdao.com/yws/public/resource/1f530a42ccf1941d7b5e6d2e23556c1a/xmlnote/F2EF6A093F664B9A873CA8ABEDB437F4/1278)

**通知**：描述切面的行为和何时执行这个切面动作，在哪个方法执行之前？之后？

五种类型的通知：

* 前置通知Before
* 后置 After
* 返回 After-returning，此时方法能够成功执行
* 异常通知 After-throwing
* 环绕Around 再调用之前之后都执行

**连接点（Join point）**:

电表抄表员目标时电表，电表为连接点。应用执行的过程中能够插入切面的一个点。切面代码利用这些点插入正常流程，添加新行为。

**切点（Pointcut）**

定义所要织入的一个连接点的范围，一般为类和方法名称。

**切面（Aspect）**

切点和通知的统称

## 实现

AOP时通过动态代理实现的。

![image](http://note.youdao.com/yws/public/resource/1f530a42ccf1941d7b5e6d2e23556c1a/xmlnote/91A21ED2B7B2408B8221E14992BB7BBA/1281)

首先，我们定义一个接口作为切点的方法。

```java
public interface Performance {
    void perform();
}
```

 定义切点：

```java
    @Pointcut("execution(* com.aop.annotation.Performance.perform(..))")
    public void performance(){}
```

切点语法表示返回值任意，入参任意。

此外我们可以通过语法对切点做范围限制。例如这里限制连接点的必须再com.concert这个包下面

```java
@Pointcut("execution(* com..Performance.perform(..))&&within(com.concert.*)")//只在concert package内做织入
public void performance(){}
```

还可以通过切点选择Bean

```java
@Pointcut("execution(* com..Performance.perform(..))&&bean('woodstock')")//只在wookstock这个bean内做织入
public void performance(){}
```

注解定义切面：

```java
@Aspect//定义切面
public class Audience {
    @Pointcut("execution(* com..Performance.perform(String))"+"&& args(name)"+"&& within(com.aop.annotation.*)")//定义切点
    public void performance(String name){}

    @Before("performance(name)")//定义advice通知方法
    public void silenceCellPhones(String name){
        System.out.println("Silencing cell phones"+name);
    }

    @Around("performance(name)")
    public void watchPerformance(ProceedingJoinPoint jp,String name){
        try{
            System.out.println("Silencing cell phones");
            System.out.println("Taking seats");
            jp.proceed();
            System.out.println("CLAP CLAP CLAP!!!");
        }catch (Throwable e){
            System.out.println("Demanding a refund");
        }
    }

}
```

在配置处需要允许AspectJ自动代理

```java
@Configuration
@ComponentScan(basePackages = {"com.aop"})
@EnableAspectJAutoProxy
```

### 注解引入新功能

Groovy和Ruby有开放类的概念，可以不用直接修改类定义就可以向类中增加新的功能。Java则完全不一样，一旦编译完毕，无法增加新功能。在本章中，我们通过AOP以代理的方式变相增加了新的代码，因此同样的道理，既然是动态生成的代理，我们自然可以动态增加新的功能。

![image](http://note.youdao.com/yws/public/resource/1f530a42ccf1941d7b5e6d2e23556c1a/xmlnote/8ABAA02C9917496CB700E09C07FB4B3B/1283 "引入新方法")

实现如下：

```java
//声明一个接口，包含需要引入的新功能
public interface Encoreable {
    void performEncore();
}
```

```java
//接口的实现
public class DefaultEncoreable implements Encoreable{
    @Override
    public void performEncore() {
        System.out.println("this is my Encoreable aaaaaaaaa!!!!!");
    }
}
```

```java
//定义切面
@Aspect
public class EncoreableIntroducer {
  //value代表需要引入方法的Bean的类型，此处是所有performance子类型，defaultImpl指向实现了接口的类
    @DeclareParents(value="com.aop.annotation.Performance+", defaultImpl = DefaultEncoreable.class)//注解标注的静态属性指名要引入的接口
    public static Encoreable encoreable;
}
```

## XML中声明切面

声明bean和切面

```xml
    <bean id ="audience" class="com.aop.xmlaop.Audience"/>
    <bean id="performanceForXML" class="com.aop.xmlaop.PerformanceForXML"/>
    <aop:config>
        <aop:aspect ref="audience"><!--声明切面-->
            <aop:pointcut id="pointcutforaudience" expression="execution(* com.aop.xmlaop.PerformanceForXML.perform(String)) and args(name)"/><!--切点-->
            <aop:before  method="silenceCellPhones" pointcut-ref="pointcutforaudience"/>
            <aop:after method="demandRefund" pointcut-ref="pointcutforaudience"/>
            <aop:around method="roundAdvice" pointcut-ref="pointcutforaudience"/><!--环绕通知-->
        </aop:aspect>
    </aop:config>
```

需要切入performance的程序如下：

```java
public class Audience {
    public void silenceCellPhones(String name){
        System.out.println("Silencing cell phones"+name);
    }

    public void takeSeats(String name){
        System.out.println("Taking seats");
    }

    public void demandRefund(String name){
        System.out.println("Demanding a refund");
    }

    public void roundAdvice(ProceedingJoinPoint jp, String name){
        try {
            System.out.println("Silencing for roundAdvice");
            System.out.println("Taking seats");
            jp.proceed();
            System.out.println("CLAP CLAP CLAP");
        } catch (Throwable e) {
            System.out.println("Demanding a refund for roundAdvice");
        }
    }
}
```

xml通过AOP为Bean引入新功能

```xml
<aop:declare-parents types-matching="com.aop.xmlaop.PerformanceXML+"
                                 implement-interface="com.aop.xmlaop.EncoreableXML" default-impl="com.aop.xmlaop.DefaultEncoreable"/>
```

