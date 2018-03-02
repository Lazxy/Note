### SSM简易笔记

> 2018年上半年接手Portal服务端的开发任务，使命艰巨啊，由此先从简单的入手，走一遍SSM基础教程。内容来自[How2J](http://how2j.cn/k/)。

#### Spring部分

​	Spring的作用是提供一个基本框架用以注入对象（IOC）以及提供非业务服务（AOP），从而达到通过配置文件就能改变对象属性以及尽量少地侵入代码的效果。

#### 一、IOC/DI

IOC即控制反转，想想Dagger就好，这里最简单的实现是在**applicationContext.xml**文件 *(在src目录下)*的bean节点里声明类以及需要注入的属性，其样式如下：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
   http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
   http://www.springframework.org/schema/aop
   http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
   http://www.springframework.org/schema/tx
   http://www.springframework.org/schema/tx/spring-tx-3.0.xsd
   http://www.springframework.org/schema/context
   http://www.springframework.org/schema/context/spring-context-3.0.xsd">
   //上面一堆乱七八糟的spring相关命名空间声明
   //这里开始声明bean的类以及属性名和值
    <bean name="c" class="pojo.Category">//标识符以及类路径
        <property name="name" value="category 1" /> //属性名及值
    </bean>
</beans>
```

在调用时代码大致如下：

```java
ApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"applicationContext.xml"}); //指定上下文的获取方式，这里是通过前面定义的xml文件

Category c = (Category) context.getBean("c");//通过xml中定义的标识符获取bean对象
//...
```

这里需要注意的是，**getBean这个方法通过标识符获取到的bean值是唯一的**，即无论取几次，该bean值都指向同一块内存。

#### 二、对象注入

Spring可以将对象作为另一个对象的成员变量构造出来，其声明也很简单：

```xml
<!--省去上面的一堆xmlns声明-->
<bean name="c" class="pojo.Category"> //仍然是原来那个类
    <property name="name" value="category 1" />
</bean>
<bean name="p" class="pojo.Product">
   <property name="name" value="product1" />
   <property name="category" ref="c" /> //这里通过ref指定需要注入的某个声明过的bean对象
</bean>
```

Java代码也没什么好贴的，与之前的IOC方式相同，在Product对象通过getBean被构造出来时，相应的Category对象也会被构造出来，且该对象仍然是**唯一**的

#### 三、注解注入

再想想Dagger，其实这里才是这两个IOC框架相似度最高的地方，可以用注解来完成对象的注入，其代码如下：

```java
//POJO
public class Product{
  //...
  @Autowired/@Resource(name="c")
  Category category
  //...
  @Autowired/@Resource(name="c")
  Category  getCategory(){
    return category;
  }
}
//applicationContext
<context:annotation-config/>//表示注解可用
<bean name="p" class="pojo.Product">
   <property name="name" value="product1" />
   <!--<property name="category" ref="c" />--> //这个时候就不需要再声明这个依赖了
</bean>
```

这里需要注意的是，**@Autowired**不指定标识符，于是当xml中有两个同类型的bean被声明时，其构造时就会因为无法确定构造对象而抛出异常；而**@Resource**根据标识符进行构造对象的选择，就不会有这个问题。

当然也可以用注解来标识构造整个bean对象，代码如下：

```java
//POJO
@Component("p")
public class Product{
  //...
}
@Component("c")
public class Category{
  //...
}
//applicationContext
<bean 
	...>
	<context:component-scan base-package="pojo"/> //声明POJO类的位置，多个包用空格隔开，不再需要原来的bean节点了
</bean>
```

这样一来就不需要在xml中声明每个bean了，但是其缺点是需要在每个POJO类中声明其变量的默认值（如果需要默认值的话）

####四、AOP

概念参见*面试相关知识点总结*。直接上用法：

```java
//Java
//切点相关类
public class ProductService {
    public void doSomeService(){
        System.out.println("doSomeService");
    }
}
//日志类
public class LoggerAspect {
    public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("start log:" + joinPoint.getSignature().getName());
        Object object = joinPoint.proceed();//执行切点的方法
        System.out.println("end log:" + joinPoint.getSignature().getName());
        return object;
    }
}

//applicationContext
<bean id="s" class="service.ProductService"> //这里必须声明切点类的bean，否则spring没有办法给其设置监听
<bean id="loggerAspect" class="aspect.LoggerAspect"/>
<aop:config>//配置AOP的标签
   <aop:pointcut id="loggerCutpoint" //设置切点，其触发条件是ProductService的任意参数的任意方法调用时
        expression="execution(* service.ProductService.*(..)) "/>
   <aop:aspect id="logAspect" ref="loggerAspect">//设置切面，指定各切点需要编织入的方法
      <aop:around pointcut-ref="loggerCutpoint" method="log"/>
   </aop:aspect>
</aop:config>
```

只要通过getBean获得ProductService对象后调用其任意方法，LoggerAspect的log方法就会生效，其中log方法只要求其参数中有ProceedingJoinPoint，以及选择是否异常捕获，其返回值无限定条件。

#### 五、注解AOP

与IOC的注解方式差不多，AOP的注解实现方式同样是把xml里的声明和配置移到了Java代码中，如下：

```java
//Java
@Aspect
public class LoggerAspect{
  	@Around(value = "execution(* service.ProductService.*(..))") //声明切入点，并通过注解来设置编织方式是在切入点之前、之后还是前后都有。
    public void log(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("start log:" + joinPoint.getSignature().getName());
        Object object = joinPoint.proceed();
        System.out.println("end log:" + joinPoint.getSignature().getName());
    }
}

//applicationContext
<aop:aspectj-autoproxy/> //用自动代理的方式处理注解，代替之前的 <aop:config>标签
```

#### 六、注解测试

这里是采用JUnit来进行测试，通过注解，就不再需要在main方法里解析xml获取上下文来构造相应对象了，其代码如下：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext.xml")//指定配置文件
public class TestSpring {
    @Autowired
    Category c;//从xml获取配置后自动注入
 
    @Test //标识测试方法，有该标识的方法可作为程序运行入口
    public void test(){
        System.out.println(c.getName());
    }
}
```

PS：这里的注解实现需要导入junit与hamcrest两个jar包。

---

#### SpringMVC部分

​	SpringMVC主要是用以作为前端和后端连接的桥梁，其负责响应前端的请求，从而返回数据或由此构造所需的前端页面。

#### 一、基础使用方式

首先构造一个Dynamic Web Application，其默认的两个主要包为`Java Resource`和`WebContent`，其中前者主要放置业务逻辑相关的Java代码(如**Controller类**)，而后者包含了**配置文件**和网页文件（如果需要的话）。下面先上配置文件：

```xml
<!--WebContent/WEB-INF/web.xml-->
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.4" xmlns="http://java.sun.com/xml/ns/j2ee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee
http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd"> <!--命名空间和头，先不管了-->
    <servlet> <!--声明servlet的类型以及其名称-->
        <servlet-name>springmvc</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
       <!--标记容器是否在启动的时候就加载这个servlet。当值为0或者大于0时，表示容器在应用启动时就加载这个servlet；当是一个负数时或者没有指定时，则指示容器在该servlet被选择时才加载。正数的值越小，启动该servlet的优先级越高。-->
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping> <!--匹配servlet与相应的url-->
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern> <!--这里的匹配规则比较复杂，主要有两个点：1.*可以作为通配符，但只能放在最开头（拓展名匹配）或最末尾（子路径匹配）。2.这里的匹配对象是[主域名]/[应用名]/ 之后的内容-->
    </servlet-mapping>
</web-app>

<!--WebContent/WEB-INF/springmvc-servlet.xml-->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
  	<!--定义匹配实体的类型与匹配内容，根据类的不同有不同的匹配方式-->
    <bean id="simpleUrlHandlerMapping" 
        class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
        <property name="mappings">
            <props>
                <prop key="/index">indexController</prop>
                <prop key="/hello">helloController</prop>
            </props>
        </property>
    </bean>
    <!--声明需要与url对应的Controller对象-->
    <bean id="indexController" class="controller.IndexController"></bean>
    <bean id="helloController" class="controller.HelloController"></bean>
</beans>
```

首先明确一点，springmvc-servlet.xml文件的文件名来源于web.xml中声明的servlet名，故这里的逻辑关系是：**web.xml中定义某个Servlet响应某个路径下的请求，而Servlet通过其同名xml文件来声明该路径的下真正工作的Controller并分发事件。**再来补全Java代码：

```java
//Java Resource/src/controller/IndexController
public class IndexController implements Controller {
    //Controller类的接口方法， 需要通过该方法判断请求类型并返回视图
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ModelAndView mav = new ModelAndView("index.jsp"); //通过jsp文件构造相应的视图对象
        mav.addObject("message", "Hello Spring MVC");//给jsp中预留的对象赋值
        return mav;
    }
}
```

显而易见的是Servlet将事件分发下去后调用了Controller的对应方法，SpringMVC中也预置了很多类实现了Controller，其在响应时的表现可能不太一样，这个之后再说。另外，这里的`index.jsp` 文件放在WebContent目录下，其内容很简单，只是预留了一个对象入口：

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" isELIgnored="false"%>
 
<h1>${message}</h1>
```

#### 二、配置页面文件路径

这里的主要目的是将页面文件划分到不同的路径下预备被读取，且使Controller在解析界面时更加灵活，其代码如下：

```xml
<!--WebContent/WEB-INF/springmvc-servlet.xml-->
<!--省略xml的头和类型声明-->
<beans>
	<!--定义静态文件解析类，并制定其解析对象的前缀（路径）及后缀（类型）-->
	<bean id="viewResolver"
		class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="prefix" value="/WEB-INF/page/" />
		<property name="suffix" value=".jsp" />
	</bean>
  	<!--定义匹配实体的类型与匹配内容，根据类的不同有不同的匹配方式-->
    <bean id="simpleUrlHandlerMapping" 
        class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
        <property name="mappings">
            <props>
                <prop key="/index">indexController</prop>
                <prop key="/hello">helloController</prop>
            </props>
        </property>
    </bean>
    <!--声明需要与url对应的Controller对象-->
    <bean id="indexController" class="controller.IndexController"></bean>
    <bean id="helloController" class="controller.HelloController"></bean>
</beans>
```

在Java代码中，即将之前的`ModelAndView mav = new ModelAndView("index.jsp");`改为`ModelAndView mav = new ModelAndView("index");`，这样一来，在需要解析界面时，ViewResolver就会自动为index添加前缀和后缀，从而索引到相应文件。

#### 三、注解实现

由Spring的种种实现基本可以猜出，SpringMVC的配置也是可以由注解实现的，其用法也基本遵循一个逻辑：**涉及到bean对象的类型声明，一般把注解加在类声明之前；涉及到具体属性的声明，把注解加到方法声明之前。xml负责用一行上下文配置开启注解解析开关。**代码如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
    http://www.springframework.org/schema/context        
    http://www.springframework.org/schema/context/spring-context-3.0.xsd">
    
    <!--指定被注解的类所在的位置-->
    <context:component-scan base-package="controller" />
	<bean id="viewResolver"
		class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="prefix" value="/WEB-INF/page/" />
		<property name="suffix" value=".jsp" />
	</bean>
</beans>
```

```java
@Controller //声明这是一个Controller方法载体
public class IndexController{//注意这里就不再需要实现Controller接口了
	@RequestMapping("/index") //声明匹配的url
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
    //...
    }
}
```

#### 四、接收表单数据

这里的接收表单数据只需要修改一下Controller功能方法的参数，SpringMVC即可自动将名称相同的属性值注入到该参数里，代码如下：

```java
//Product类
public class Produc {
	private int id;
	private String name;
	private float price;
	//...getter与setter方法
}

//Controller类
@Controller
public class ProductController {
	@RequestMapping("/addProduct")
	public ModelAndView showProduct(Product p){
		ModelAndView result = new ModelAndView("showProduct");//这里可以称作是一个转发行为
		result.addObject(p);
		return result;
	}
}
```

这样一来，当请求的地址为类似`.../addProduct?name=lazxy&price=100`时，name与price的值就会被注入到p对象中，以此提供数据给静态文件。

#### 五、重定向

重定向指客户端选择的页面跳转，这种情况下其路径会发生改变；而上面的转发行为虽然也发生了页面的变化，但路径不变，其代码很简单：

```java
//之前的代码全部省略...
	@RequestMapping("/jump")
	public ModelAndView jump2Index(){
		return new ModelAndView("redirect:/index"); //声明跳转的目的地
	}
```

#### 六、Session

为了在不同界面间传递信息以及加快数据访问，Web应用采用Session作为信息的载体，其在用户第一次访问服务器时产生，在浏览器关闭或者长时间（Tomcat中是20分钟）不活跃时被销毁。基于它的作用，它的携带参数通过自定义键值对存放（想想Intent），下面是它的简单用法:

```java
	@RequestMapping("/check")
	public ModelAndView checkSession(HttpSession session){//函数参数提供Session的注入入口
		Integer count = (Integer)session.getAttribute("count");//获取自定义的信息
		if(count == null)
			count = 0;
		count++;
		session.setAttribute("count", count); //设置自定义信息
		return new ModelAndView("check");
	}
```

#### 七、拦截器

SpringMVC里的拦截器（Interceptor）感觉上与Spring的AOP框架差不多，但其大概不像AOP框架那样用代理的方式来完成对目标代码的包围，而直接是以函数调用先后来对请求和响应进行一定的过程处理。其使用格式也更固定一些，如下：

```java
//interceptor/IndexInterceptor
public class IndexInterceptor extends HandlerInterceptorAdapter { 
 
     /** 
     * 在业务处理器处理请求之前被调用 
     * 如果返回false 
     *     从当前的拦截器往回执行所有拦截器的afterCompletion(),再退出拦截器链
     * 如果返回true 
     *    执行下一个拦截器,直到所有的拦截器都执行完毕 
     *    再执行被拦截的Controller 
     *    然后进入拦截器链, 
     *    从最后一个拦截器往回执行所有的postHandle() 
     *    接着再从最后一个拦截器往回执行所有的afterCompletion() 
     */   
    public boolean preHandle(HttpServletRequest request,   
            HttpServletResponse response, Object handler) throws Exception {
        System.out.println("preHandle(), 在访问Controller之前被调用"); 
        return true;
    } 
 
    /**
     * 在业务处理器处理请求执行完成后,生成视图之前执行的动作   
     * 可在modelAndView中加入数据，比如当前时间
     */ 
     
    public void postHandle(HttpServletRequest request,   
            HttpServletResponse response, Object handler,   
            ModelAndView modelAndView) throws Exception { 
        System.out.println("postHandle(), 在访问Controller之后，访问视图之前被调用,这里可以注入一个时间到modelAndView中，用于后续视图显示");
        modelAndView.addObject("date","由拦截器生成的时间:" + new Date());
    } 
 
    /** 
     * 在DispatcherServlet完全处理完请求后被调用,可用于清理资源等  
     *  
     * 当有拦截器抛出异常时,会从当前拦截器往回执行所有的拦截器的afterCompletion() 
     */
     
    public void afterCompletion(HttpServletRequest request,   
            HttpServletResponse response, Object handler, Exception ex) 
    throws Exception { 
           
        System.out.println("afterCompletion(), 在访问视图之后被调用"); 
    } 
       
}
```

其配置文件也与前面的各种组件大同小异：

```xml
<mvc:interceptors>   
        <mvc:interceptor>
          	<!--这里表示“所有”的通配符为**-->
            <mvc:mapping path="/index"/> 
            <!-- 定义在mvc:interceptor下面的表示是对特定的请求才进行拦截的 --> 
            <bean class="interceptor.IndexInterceptor"/>     
        </mvc:interceptor> 
        <!-- 当设置多个拦截器时，先按顺序调用preHandle方法，然后逆序调用每个拦截器的postHandle和afterCompletion方法 --> 
</mvc:interceptors>
```

---

#### MyHabit部分

​	极其费力地装完了MySQL，终于可以开始写代码了。怎么说呢，Java的这些叫做框架的东西，其主要的作用其实就是简化和分离代码，而就像Android里用xml表示布局一样，Java框架们实现目的的途径也主要就是两种：xml配置文件和注解，MyBatis终究也不例外。

#### 一、简单使用

​	当我好不容易弄到了IDEA U 的正版使用权之后，我又要开始建普通Java工程了。。尴尬。。首先看看MyBatis的主要配置文件：

```xml
<!--src/mybatis-config.xml-->
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <typeAliases>
        <package name="pojo"/> <!--声明一下数据库映射对象的类所在的包-->
    </typeAliases>
    <environments default="development"><!--这里的命名随意，但要保证匹配下面的id之一-->
        <environment id="development">
            <transactionManager type="JDBC"/><!--如果和Spring一起用，则这个配置可忽略-->
            <dataSource type="POOLED"> <!--保持连接，节省初始化时间-->
                <!--传统JDBC语句中需要的属性配置-->
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3300/demo?characterEncoding=UTF-8"/>
                <property name="username" value="lazxy"/>
                <property name="password" value="secret"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="pojo/Category.xml"/> <!--划重点！！声明SQL语句所在的文件-->
    </mappers>
</configuration>
```

更具体的属性可以看[官方文档](http://www.mybatis.org/mybatis-3/zh/configuration.html)。

省略数据库映射对象类文件，就是一个简单的Bean，然后我们可以看看SQL是怎么写在xml里的：

```xml
<!--pojo/Category.xml-->
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="pojo">
    <select id="listCategory" resultType="Category"> <!--这里声明了这条语句的Id以及数据库映射对象的类别-->
        select * from   category_
    </select>
</mapper>
```

最后，调用数据库：

```java
//src/Test
public class Test{

    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        //获取数据库配置文件
        InputStream inputStream = Resources.getResourceAsStream(resource);
        //从配置文件中构造出一个数据库对话
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        //开启数据库连接
        SqlSession session=sqlSessionFactory.openSession();
		//调用相应Id的SQL语句
        List<Category> cs=session.selectList("listCategory");
        for (Category c : cs) {
            System.out.println(c.getName());
        }
    }
}

```

就形式上的确没什么好说的，和之前的其他框架基本大同小异。