# SpringMVC

SpringMVC故名思议，是Spring提供的MVC框架，那么自然有其他MVC框架，例如Struts等等。目前Struts正在陷入漏洞风波，SpringMVC越来越受到大家的欢迎。

## 一个HTTP请求的旅程

要了解SpringMVC的原理首先我们从最基本的着手。一个HTTP请求到底要经历什么才能从请求发起到返回至浏览器中？一个请求被HTTP协议封装之后，从OSI应用层出发，历经传输层，网络层，链路层，物理层，直到与服务端Servlet容器（Tomcat）建立连接。这时候Tomcat管理的Servlet为该请求建立一个新线程或者从线程池中拿出线程做处理。如果请求是首次发出，我们需要为该请求创建一个Session；如果非首次访问，且Session没有过期，那么就通过Cookies访问对应的Session。请求经过一系列的处理，一般包括过滤器、监听器，结果通过response返回。需要注意的是Servlet、过滤器、监听器，在一般情况下只有一个实例，但是如果你在web.xml中声明了多个Servlet，或者一个Servlet实现了SingleThreadMode接口，又或者在分布式的环境中，那么就存在多个实例。

## 服务端程序的进化

**JSP**

起初我们只需要一个web.xml加上一个jsp文件，请求到达时直接返回url指定页面。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <welcome-file-list>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>
</web-app>
```

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
  <head>
    <title>Welcome！I'm a baby web</title>
  </head>
  <body>
  Hello, I'm still young.
  </body>
</html>
```

后来觉得静态页面不够酷炫编程也不方便，我们要把Java库应用到JSP中。

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>

<html>
  <head>
    <title>Welcome！I'm a baby web</title>
  </head>
  <body>
  Hello, I'm still young.<br/>
  现在时间为：<%= (new java.util.Date()).toString()%>
  <!--JSP中声明变量-->
  <%!int i = 0;%>
  <!--声明函数-->
  <%!public String f(int i) {
    if (i < 3)
     return "i小于3";
    else
    return "i大于等于3";
    }%>
  你的值是：<%= f(1)%><<br/>
  <!--使用Bean-->
  <jsp:useBean id="test" class="com.book.web.SimpleBean"/>
  <jsp:setProperty name="test" property="name" value="Baby"/>
  你叫什么：<jsp:getProperty name="test" property="name"/>
  <!--获取Session-->
  <%! int number = 0;
    synchronized void countPeople(){
        number++;
    }
    %>
  <%
    if(session.isNew()){
        countPeople();
        String str=String.valueOf(number);
        session.setAttribute("count", str);
        application.setAttribute("count",str);
    }
  %>
  <p>
    你是第<%=(String) application.getAttribute("count")%>个访问本网站的人。
  </p>
  <!--几个默认对象 application：应用启动的时候存在，应用停掉消失，用户在哥哥页面浏览都是这一个对象。
        pageContext范围局限在本页面内。
        Request从请求到页面之后。
        Session用户持续连接服务器的时间，连接断开则Session无效。
        -->
  </body>
</html>
```

java代码嵌入在样式代码中，虽然可用，但是凌乱。因此有部分Java程序员提出，不用把java嵌入到网页中，而要把网页嵌入到java中，这种方案就是servlet。

**Servlet**

jsp出现以后出现了有两种编程方式，一种程序员啥代码都往JSP里面堆积，为此扩展了各种各样的标签；一派觉得这种程序非常丑陋，坚持用Servlet技术，在纯粹的Java类中接收请求，填充网页样式，并返回。

下面是Servlet的一个例子：

```java
public class MyServlet extends HttpServlet {
    public void doGet(HttpServletRequest request, HttpServletResponse response)throws ServletException,IOException{
        response.setContentType("text/html");
        response.setCharacterEncoding("utf-8");
        PrintWriter out = response.getWriter();
        out.println("<HTML>");
        out.println("<HEAD><TITLE>Servlet实例</TITLE></HEAD>");
        out.println("<body>");
        out.println("Servlet实例");
        out.println(this.getClass());
        out.println("</body>");
        out.println("</HTML");
        out.flush();
        out.close();
    }
}
```

我们将JSP用String的方式传递给response，并在前端展现。这种方式可以在服务端做纯粹的java编程，虽然有点奇怪，但是这个程序没有涉及任何标签（额，除了String里面的）。然而，在构建大型工程的时候，一方面样式代码会占用大量代码且不可以重用，另一方面我们无法对样式做扩展了。

**MVC**

当当当当...MVC在这JSP方案和servlet方案中取得了平衡。MVC将网页样式和java代码分离，样式我们叫“View”，java代码我们叫“Controller”，两者之间交换数据我们叫“Model”。这样样式可以自由扩展，用JSP、HTML/CSS、甚至PHP都没有问题；Controller只需要专心于业务逻辑实现；Model既可以用来做持久化也可以用来做接收表单的对象。

## SpringMVC来了

![image](https://note.youdao.com/yws/public/resource/1f530a42ccf1941d7b5e6d2e23556c1a/xmlnote/CA9040F35D2E496EB319B66A007854BB/1294)

上文说，MVC是将Model、View、Controller三者分离。那么如何做的分离呢？可以看一下上面的图，一个携带URL和表单信息的请求首先会发送到DispatcherServlet。DispatcherServlet是前端控制器，在应用中为一个单例。DispatcherServlet通过处理器映射得知请求的下一站是在哪个控制器。控制器接收请求，并将数据打包进模型中，返回一个视图名，传递给DispatcherServlet。DispatcherServlet通过视图解析器定位到视图实现。请求到达视图，视图用模型数据渲染输出，写入到response对象中。

###DispatcherServlet的配置

有两种方法，第一种是直接按照Servlet的配置方法在web.xml中配置：

```xml
    <servlet>
        <servlet-name>servlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>servlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
```

另一种是基于Servlet3.0提供的机制，Servlet3.0环境中容器会查找javax.servlet.ServletContainerInitialize的实现类，Spring实现了这个接口SpringServletContainerInitializer，这个类查找WebApplicationInitializer类并将配置任务交给它，AbstractAnnotationConfigDispatcherServletInitializer是WebApplicationInitializer的一个基础实现，于是我们可以做出如下的实现：

```java
public class SpitterWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
  //用来配置ContextLoaderListener创建的应用上下文中的bean
  @Override
  protected Class<?>[] getRootConfigClasses() {
    return new Class<?>[] { RootConfig.class };
  }
//定义DispatcherServlet应用上下文中的bean
  @Override
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] { WebConfig.class };
  }
//所有请求都会被拦截
  @Override
  protected String[] getServletMappings() {
    return new String[] { "/" };
  }

}
```

实现webconfig类：

```java
@Configuration
@EnableWebMvc
@ComponentScan("spittr.web")
public class WebConfig extends WebMvcConfigurerAdapter {

  @Bean
  public ViewResolver viewResolver() {
    InternalResourceViewResolver resolver = new InternalResourceViewResolver();
    resolver.setPrefix("/WEB-INF/views/");
    resolver.setSuffix(".jsp");
    return resolver;
  }
  
  @Override
  public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
    configurer.enable();
  }
  
  @Override
  public void addResourceHandlers(ResourceHandlerRegistry registry) {
    // TODO Auto-generated method stub
    super.addResourceHandlers(registry);
  }

}
```

在Spring MVC中我们其实是可以创建出多个DispatcherServlet的（只要创建多个继承自AbstractAnnotationConfigDispatcherServletInitializer的类即可）。而每个DispatcherServlet有自己的应用上下文（WebApplicationContext），这个应用上下文只针对这个DispatcherServlet有用。这也就是`getServletConfigClasses`的作用，获取这个DispatcherServlet的应用上下文的配置类。

而除了每个DispatcherServlet配置类的应用上下文之外，还有一个根应用上下文，这个应用上下文的作用是为了在多个DispatcherServlet之间共享Bean，比如数据源Bean，这就是`getRootConfigClasses`的作用，用于返回根应用上下文的配置类。Spring框架的机制会保证如果在当前DispatcherServlet的应用上下文中没有找到想要的bean时，会去根应用上下文中去找。

这里我们只需要一个DispatcherServlet配置，因此可以不对`RootConfig.class`做复杂配置。

```java
@Configuration
@Import(DataConfig.class)
@ComponentScan(basePackages={"spittr"}, 
    excludeFilters={
        @Filter(type=FilterType.CUSTOM, value=WebPackage.class)
    })
public class RootConfig {
  public static class WebPackage extends RegexPatternTypeFilter {
    public WebPackage() {
      super(Pattern.compile("spittr\\.web"));
    }    
  }
}
```

编写控制器：

```java
@Controller
@RequestMapping("/")
public class HomeController {

  @RequestMapping(method = GET)
  public String home(Model model) {
    return "home";
  }

}
```

home.jsp

```jsp
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ page session="false" %>
<html>
  <head>
    <title>Spitter</title>
    <link rel="stylesheet" 
          type="text/css" 
          href="<c:url value="/resources/style.css" />" >
  </head>
  <body>
    <h1>Welcome to Spitter</h1>

    <a href="<c:url value="/spittles" />">Spittles</a> | 
    <a href="<c:url value="/spitter/register" />">Register</a>
  </body>
</html>
```

### 表单处理

Spring提供了数据验证功能。

```java
public class Spitter {
  private Long id;
  
  @NotNull
  @Size(min=5, max=16)
  private String username;

  @NotNull
  @Size(min=5, max=25)
  private String password;
  
  @NotNull
  @Size(min=2, max=30)
  private String firstName;

  @NotNull
  @Size(min=2, max=30)
  private String lastName;
  
  @NotNull
  @Email
  private String email;
  //get set略
}
```

我们发起一个表单：

```jsp
<html>
  <head>
    <title>Spitter</title>
    <link rel="stylesheet" type="text/css" 
          href="<c:url value="/resources/style.css" />" >
  </head>
  <body>
    <h1>Register</h1>

    <form method="POST">
      First Name: <input type="text" name="firstName" /><br/>
      Last Name: <input type="text" name="lastName" /><br/>
      Email: <input type="email" name="email" /><br/>
      Username: <input type="text" name="username" /><br/>
      Password: <input type="password" name="password" /><br/>
      <input type="submit" value="Register" />
    </form>
  </body>
</html>
```

后台接收这个请求的Controller如下：

```java
@Controller
@RequestMapping("/spitter")
public class SpitterController {

  private SpitterRepository spitterRepository;

  @Autowired
  public SpitterController(SpitterRepository spitterRepository) {
    this.spitterRepository = spitterRepository;
  }
  //接收Get请求的函数映射
  @RequestMapping(value="/register", method=GET)
  public String showRegistrationForm() {
    return "registerForm";
  }
  //接收Post请求的映射
  @RequestMapping(value="/register", method=POST)
  public String processRegistration(
      @Valid Spitter spitter, 
      Errors errors) {
    if (errors.hasErrors()) {
      return "registerForm";
    }
    
    spitterRepository.save(spitter);
    return "redirect:/spitter/" + spitter.getUsername();
  }
  //动态映射地址
  @RequestMapping(value="/{username}", method=GET)
  public String showSpitterProfile(@PathVariable String username, Model model) {
    Spitter spitter = spitterRepository.findByUsername(username);
    model.addAttribute(spitter);
    return "profile";
  }
  
}
```







