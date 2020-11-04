---
title: SSM简易笔记
date: 2018-07-09 14:14:02
tags: 笔记
---

> 2018年上半年接手Portal服务端的开发任务，使命艰巨啊，由此先从简单的入手，走一遍SSM基础教程。内容来自[How2J](http://how2j.cn/k/)。~~惊了！我怎么什么都学过，为什么一点都记不起来了！~~

#### Spring部分

​	Spring的作用是提供一个基本框架用以注入对象（IOC）以及提供非业务服务（AOP），从而达到通过配置文件就能改变对象属性以及尽量少地侵入代码的效果。

#### 一、IOC/DI

IOC即控制反转，想想Dagger就好，这里最简单的实现是在**applicationContext.xml**文件 *(在src目录下)*的bean节点里声明类以及需要注入的属性，其样式如下：

<!--more-->

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

省略数据库映射对象类文件，就是一个简单的Bean（*当然其成员属性名与数据库的列名要一致，或者说，和数据库返回的结果表列名一致*），然后我们可以看看SQL是怎么写在xml里的：

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
      	//如果对数据库的操作有修改的内容，则需要加上session.commit提交修改
      	session.close();//关闭连接
    }
}

```

就形式上的确没什么好说的，和之前的其他框架基本大同小异。

#### 二、CRUD

​	有了上面的铺垫，增删改查代码其实也很简单，无非就是在xml里配置完对应的语句，然后由SqlSession对象调用罢了，下面直接给出所有代码：

```xml
<!--Category.xml-->
<mapper namespace="pojo">
  	<!--通过各自的标签声明语句的功能-->
    <insert id="addCategory" parameterType="Category" >
        insert into category_ ( name ) values (#{name})
    </insert>

    <delete id="deleteCategory" parameterType="Category" >
        delete from category_ where name= #{name}
    </delete>

    <!--这里三个参数分别声明SQL语句代号、参数类型与参数返回值类型-->
    <select id="getCategory" parameterType="string" resultType="Category">
        select * from   category_  where name= #{name}
    </select>

    <update id="updateCategory" parameterType="Category" >
        update category_ set name=#{name} where id=#{id}
    </update>
    <select id="listCategory" resultType="Category">
        select * from   category_
    </select>
</mapper>

```

```java
public static void addCategory(SqlSession session, int id,String name){
    Category c = new Category();
    if(id > 0)
        c.setId(id);
    c.setName(name);
    session.insert("addCategory",c);
}

public static void deleteCategory(SqlSession session, String name){
    Category c = new Category();
    c.setName(name);
    session.delete("deleteCategory",c);
}

public static void getCategory(SqlSession session, String name){
  	//这里得到的值是一个泛型类型，可以被任意强转
    System.out.println(((Category)session.selectOne("getCategory",name)).getId());
}

public static void updateCategory(SqlSession session, int id,String name){
    Category c = new Category();
    c.setId(id);
    c.setName(name);
    session.update("updateCategory",c);
}

public static void listCategory(SqlSession session){
    List<Category> cs=session.selectList("listCategory");
    for (Category c : cs) {
        System.out.println(c.getName());
    }
}
```

这里有一点要声明的是，xml中的parameterType这个属性，对于基本类型和类类型的识别根据版本不同而不同，有时基本数据类型需要在加上前缀下划线，如上面的`_int`，有时候引用类型需要写下完全限定名，如`java.lang.String` 。当然也可以用设置别名的方式进行省略，如：

```xml
<typeAlias type="com.someapp.model.User" alias="User"/>
```

另外，如果数据库语句执行有误，如删除一个不存在的元组，也不会在结果上出现什么异常。

#### 三、一对多关系

​	从这里开始SQl的查询就涉及到了多表连接和查询了，其实现重点是**resultMap**的声明，上代码：

```xml
<!--Category.xml-->
<mapper namespace="pojo">
  	<!--这里的返回类型仍然是Category，但resultMap指示了如何对其Product成员变量进行初始化-->
    <resultMap type="Category" id="categoryBean">
      	<!--这里的id是表的标识，表明其下的属性和该列来自同一张表，以此避免了同名列的混淆，并提升了性能-->
        <id column="cid" property="id" /> 
      	<!--column表示SQl表（或结果的虚表）中的列名，property表示对应的bean属性名-->
        <result column="cname" property="name" />

        <!-- 一对多的关系 -->
        <!-- property: 指的是集合属性的值, ofType：指的是集合中元素的类型 -->
        <collection property="products" ofType="Product">
            <id column="pid" property="id" />
            <result column="pname" property="name" />
            <result column="price" property="price" />
        </collection>
    </resultMap>

    <!-- 关联查询分类和产品表 这里用resultMap替换了resultType，此两者不能同时出现-->
    <select id="listCategory" resultMap="categoryBean">
        select c.*, p.*, c.id 'cid', p.id 'pid', c.name 'cname', p.name 'pname' from category_ c left join product_ p on c.id = p.cid
    </select>
</mapper>
```

简单来说，resultMap就是以xml声明了对复杂查询结果的处理，指示如何将这些字段注入相对应的类对象中去，其也是MyBatis的核心，更详细的属性信息可见[官方文档](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html)。另外附上Java代码：

```java
//pojo/Category.java
public class Category{
	//...
	List<Product> products; //添加对Product类的引用
	//..
	public List<Product> getProducts() {
        return products;
    }
    public void setProducts(List<Product> products) {
        this.products = products;
    }
}

//Test
public static void listCategory(SqlSession session){
    List<Category> cs=session.selectList("listCategory");
    for (Category c : cs) {
        System.out.println(c);
        List<Product> ps = c.getProducts(); //与之前只得到了Category集合不同，这次其持有的Product也得到了初始化
        for (Product p : ps) {
            System.out.println("\t"+p);
        }
    }
}
```

#### 四、多对一关系

​	依照上面的例子，多对一关系实际就是从Category为主体转换成了以Product为主体，其配置和上面类似，只是把collection换成了association：

```xml
<!--Product.xml-->
<mapper namespace="pojo">
    <resultMap type="Product" id="productBean">
        <id column="pid" property="id" />
        <result column="pname" property="name" />
        <result column="price" property="price" />

        <!-- 多对一的关系 -->
        <!-- property: 指的是属性名称, javaType：指的是属性的类型 -->
        <association property="category" javaType="Category">
            <id column="cid" property="id"/>
            <result column="cname" property="name"/>
        </association>
    </resultMap>

    <!-- 根据id查询Product, 关联将Orders查询出来 -->
    <select id="listProduct" resultMap="productBean">
        select c.*, p.*, c.id 'cid', p.id 'pid', c.name 'cname', p.name 'pname' from category_ c left join product_ p on c.id = p.cid
    </select>
</mapper>
```

当然，与此相对的，Product.java中也增加了相应的Category属性，以及其getter、setter方法，不再赘述。

####五、多对多关系

​	这里给出的例子是：*一份订单包含多种产品，同时每种产品对应多份订单*，其实际上可以视作**对应项交叉的一对多情况**，此时需要给出一张额外表作为两个对象的中间关系声明，在代码上的体现即为Order（订单表）通过一对多关系查询OrderItem（订单关系表），然后OrderItem去多对一查询Product（产品表），代码如下：

```xml
<!--Order.xml-->
<mapper namespace="pojo">
    <resultMap type="Order" id="orderBean">
        <id column="oid" property="id" />
        <result column="code" property="code" />

        <!--先获取OrderItem集合-->
        <collection property="orderItems" ofType="OrderItem">
            <id column="oiid" property="id" />
            <result column="number" property="number" />
            <!--再通过集合内的订单信息多对一地获取相应的产品信息-->
            <association property="product" javaType="Product">
                <id column="pid" property="id"/>
                <result column="pname" property="name"/>
                <result column="price" property="price"/>
            </association>
        </collection>
    </resultMap>

    <select id="listOrder" resultMap="orderBean">
        select o.*,p.*,oi.*, o.id 'oid', p.id 'pid', oi.id 'oiid', p.name 'pname'
        from order_ o
        left join order_item_ oi    on o.id =oi.oid
        left join product_ p on p.id = oi.pid
    </select>

    <select id="getOrder" resultMap="orderBean">
        select o.*,p.*,oi.*, o.id 'oid', p.id 'pid', oi.id 'oiid', p.name 'pname'
        from order_ o
        left join order_item_ oi on o.id =oi.oid
        left join product_ p on p.id = oi.pid
        where o.id = #{id}
    </select>
</mapper>
```

由于多表操作中无法避免地需要用到级联操作，而在一个标签内执行多条SQL语句时，需要在mybatis-config.xml文件中配置：

```xml
 <environments default="development">
      <environment id="development">
          <!--...-->
              <property name="url" value="jdbc:mysql://localhost:3300/demo?characterEncoding=UTF-8&amp;allowMultiQueries=true"/>
          <!--...-->
      </environment>
</environments>
```



#### 六、动态SQL

1. **if**，在SQL语句标签内，可以通过\<if\>来声明SQL语句执行的条件，如：

   ```xml
   <select id="findActiveBlogWithTitleLike"
        resultType="Blog">
     SELECT * FROM BLOG 
     WHERE state = ‘ACTIVE’ 
     <if test="title != null"> <!--如果title不为空，则采取近似查询，否则忽略-->
       AND title like #{title}
     </if>
   </select>
   ```

2. **choose、when、otherwise**，类似于switch，表示一个多选项判断：

   ```xml
   <select id="findActiveBlogLike"
        resultType="Blog">
     SELECT * FROM BLOG WHERE state = ‘ACTIVE’
     <choose>
       <when test="title != null">
         AND title like #{title}
       </when>
       <when test="author != null and author.name != null">
         AND author_name like #{author.name}
       </when>
       <otherwise>
         AND featured = 1
       </otherwise>
     </choose>
   </select>
   ```

3. **trim、where、set**，这几个标签的作用是避免上面的判断结果不符合SQL语法，比如多一个“，”或者“AND”等，示例如下：

   ```xml
   <select id="findActiveBlogLike" resultType="Blog">
     SELECT * FROM BLOG 
     <where> 
       <if test="state != null">
            state = #{state}
       </if> 
       <if test="title != null">
           AND title like #{title}
       </if>
       <if test="author != null and author.name != null">
           AND author_name like #{author.name}
       </if>
     </where>
   </select>
   ```

   其等价为：

   ```xml
   <trim prefix="WHERE" prefixOverrides="AND |OR ">
     ... 
   </trim>
   ```

   即将\<where\>标签替换为WHERE，且去除AND与OR前缀（都包括空格），而\<set\>则会去除后缀的`,`，如：

   ```xml
   <update id="updateAuthorIfNecessary">
     update Author
       <set>
         <if test="username != null">username=#{username},</if>
         <if test="password != null">password=#{password},</if>
         <if test="email != null">email=#{email},</if>
         <if test="bio != null">bio=#{bio}</if>
       </set>
     where id=#{id}
   </update>
   ```

4. **bind**，其功能相当于创建一个与入参相关的新变量，完成诸如拼接之类的工作：

   ```xml
   <select id="selectBlogsLike" resultType="Blog">
     <bind name="pattern" value="'%' + _parameter.getTitle() + '%'" />
     SELECT * FROM BLOG
     WHERE title LIKE #{pattern}
   </select>
   ```

#### 七、注解

1. CRUD替换，这里主要是把原来类的同名xml文件以注解的形式体现在新的Java接口类上，如：

   ```java
   public interface CategoryMapper {

       @Insert(" insert into category_ ( name ) values (#{name}) ")
       public int add(Category category);

       @Delete(" delete from category_ where name= #{name} ")
       public void delete(int id);

       @Select(" select * from   category_  where id= #{id} ")
       public Category get(int id);

       @Update(" update category_ set name=#{name} where id=#{id} ")
       public int update(Category category);

       @Select(" select * from category_ ")
       public List<Category> list();
   }
   ```

   然后再把这个mapper注册到配置文件中去：

   ```xml
   <!--mybatis-config.xml-->
   <!--...-->
   <mappers>
       <!--...-->
       <mapper class="mapper.CategoryMapper"/>
   </mappers>
   <!--...-->
   ```

   最后调用：

   ```java
   public static void main(String[] args) throws IOException {
       String resource = "mybatis-config.xml";
       InputStream inputStream = Resources.getResourceAsStream(resource);
       SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
       SqlSession session=sqlSessionFactory.openSession();
       CategoryMapper mapper = session.getMapper(CategoryMapper.class);
       listCategory(mapper);
       //...
   }

   //...

   public static void listCategory(CategoryMapper mapper){
       List<Category> cs=mapper.list(); //直接通过接口方法来进行SQL操作
       for (Category c : cs) {
           System.out.println(c);
       }
   }
           
   ```

   就其声明方法和调用方法来说，它和Retrofit倒是比较相似，也基本保持了这一套框架的一贯特点。

2. 一对多替换，这里用一个@Results注解代替了\<resultType\>标签，声明了自身属性对象以及与其他表连接的外键，外键匹配条件则写在另一个查询语句里，如下：

   ```java
   //CategoryMapper.java
   //...
   @Select(" select * from category_ ")
   @Results({
       //@Result(property = "id", column = "id") 这里已经不再需要用属性区别表了
     	//这里的column = "id" 是外键声明，@Many的注解参数则为外键匹配条件查询语句的路径,javaType其实也可以省略
       @Result(property = "products", javaType = List.class, column = "id",
           many = @Many(select = "mapper.ProductMapper.listByCategory"))})
   	public List<Category> list();

   	//ProductMapper.java
       public interface ProductMapper {
           @Select(" select * from product_ where cid = #{cid}")
           public List<Product> listByCategory(int cid);
       }
   ```

3. 多对一替换，基本大同小异，区别在把上面的@Many换成了@One：

   ```java
   	@Select(" select * from product_ ")
       @Results({
               @Result(property = "category", column = "cid", one = @One(select = "mapper.CategoryMapper.get"))
       })
       public List<Product> listProduct();
   ```

4. 多对多替换，实际上就是一个一对多和一个多对一的组合，没有新的东西，这里代码就不贴了。

5. 动态SQL替换，这里有点像Android的SQLite的用法，用函数+语句片段的方式调用SQL，然后再通过注解把函数引用送到要用的地方，如下：

   ```java
   //CategoryDynaSqlProvider.java
   public class CategoryDynaSqlProvider {
       public String list() {
            return new SQL()
                    .SELECT("*")
                    .FROM("category_")
                    .toString();
            
       }
       public String get() {
           return new SQL()
                   .SELECT("*")
                   .FROM("category_")
                   .WHERE("id=#{id}")
                   .toString();
       }
        
       public String add(){
           return new SQL()
                   .INSERT_INTO("category_")
                   .VALUES("name", "#{name}")
                   .toString();
       }
       public String update(){
           return new SQL()
                   .UPDATE("category_")
                   .SET("name=#{name}")
                   .WHERE("id=#{id}")
                   .toString();
       }
       public String delete(){
           return new SQL()
                   .DELETE_FROM("category_")
                   .WHERE("id=#{id}")
                   .toString();
       }
        
   }

   //CategoryMapper.java
   public interface CategoryMapper {
     
       @InsertProvider(type=CategoryDynaSqlProvider.class,method="add") 
       public int add(Category category); 
           
       @DeleteProvider(type=CategoryDynaSqlProvider.class,method="delete")
       public void delete(int id); 
           
       @SelectProvider(type=CategoryDynaSqlProvider.class,method="get") 
       public Category get(int id); 
         
       @UpdateProvider(type=CategoryDynaSqlProvider.class,method="update") 
       public int update(Category category);  
           
       @SelectProvider(type=CategoryDynaSqlProvider.class,method="list")     
       public List<Category> list(); 
   }
   ```

   由于SQL语句实际上是几个函数片段拼凑而成的，故可以手动在Provider中进行逻辑流判断，然后返回最后的值。

   #### 八、其他特性

   1.  **日志**，MyBatis自身不带有日志功能，需要借助第三方日志功能，如log4j，其配置非常简单：导入log4j的jar包后，在根目录下配置log4j.properties,代码如下：

       ```properties
       #log4j.properties
       # Global logging configuration
       log4j.rootLogger=ERROR, stdout
       # MyBatis logging configuration...
       log4j.logger.mapper=TRACE
       # Console output...
       log4j.appender.stdout=org.apache.log4j.ConsoleAppender
       log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
       log4j.appender.stdout.layout.ConversionPattern=%5p [%t] - %m%n
       ```

       关于配置文件的具体信息，参见[参考教程](http://how2j.cn/k/log4j/log4j-config/1082.html)。

   2.  **延迟加载（懒加载）**，该功能开启后，在一对多或多对一操作中，只有在用到了相关的对象时，才会进行二级查找，以此减少了数据库操作的消耗，其主要是在mybatis-config.xml中进行配置：

       ```xml
       <configuration>
         	<!--....-->
           <settings>
               <!-- 打开延迟加载的开关 -->
               <setting name="lazyLoadingEnabled" value="true" />
               <!-- 将积极加载改为消息加载即按需加载 -->
               <setting name="aggressiveLazyLoading" value="false"/>
           </settings>
         	<!--....-->
       </configuration>
       ```

       需要注意触发懒加载的条件，目前遇到的有直接将对象放入流输出就触发了懒加载的情况。

   3.  **分页**，这里借助了MyBatis的插件**PageHelper**，当然也可以直接写在SQL里，不过用注解的话就会出现分页参数不可配的情况。下面是使用实例：

       ```xml
       <!--mybatis-config.xml-->
       <configuration>
       	<!--...-->
           <plugins>
               <plugin interceptor="com.github.pagehelper.PageInterceptor"/>
           </plugins>
           <!--...-->
       </configuration>

       //Test.java
       //...在main的数据库查询调用之前加一句
       PagerHelper.offsetPage(offset,limit);
       ```

   4.  **一级缓存**，session会对查询过的结果进行缓存，当查询参数一致时，不会执行SQL语句而直接返回值，在数据库被增删改之后缓存会被清除。

   5.  **二级缓存**，在配置了缓存设置的情况下，SessionFactory会缓存数据库结果（试验失败）。
