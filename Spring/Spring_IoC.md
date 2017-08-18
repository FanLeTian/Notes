---
layout: post
title: Spring IoC
category: 技术
comments: true
---
# Spring IoC

### 1. 为什么要用Spring？

Spring是一个解决了许多在J2EE开发中常见的问题的强大框架。

Spring提供了管理业务对象的一致方法并且鼓励了注入对接口编程而不是对类编程的良好习惯。

Spring的架构基础是基于使用JavaBean属性的Inversion of Control容器。然而，这仅仅是完整图景中的一部分：Spring在使用IoC容器作为构建关注所有架构层的完整解决方案方面是独一无二的。 

Spring提供了唯一的数据访问抽象，包括简单和有效率的JDBC框架，极大的改进了效率并且减少了可能的错误。Spring的数据访问架构还集成了[hibernate](http://lib.csdn.net/base/javaee)和其他O/R mapping解决方案。 

Spring还提供了唯一的事务管理抽象，它能够在各种底层事务管理技术，例如JTA或者JDBC之上提供一个一致的编程模型。 

Spring提供了一个用标准[Java](http://lib.csdn.net/base/java)语言编写的AOP框架，它给POJOs提供了声明式的事务管理和其他企业事务--如果你需要--还能实现你自己的aspects。这个框架足够强大，使得应用程序能够抛开EJB的复杂性，同时享受着和传统EJB相关的关键服务。 

Spring还提供了可以和总体的IoC容器集成的强大而灵活的MVC web框架。

***

### 2. IOC

#### 2.1、定义

​	IoC(控制反转：Inverse of Control)是Spring容器的内核，AOP、声明式事务等功能在此基础上开花结果。虽然IoC这个重要概念不容易理解，但它确实包含很多内涵，它涉及代码解耦、设计模式、代码优化等问题。

​	因为IoC概念的不容易理解，Martin Fowler提出了DI(依赖注入：Dependency Injection)的概念用来代替IoC，即让调用类对某一接口实现类的依赖关系由第三方（容器或协作类）注入，以移除调用类对某一接口实现类的依赖。

​	Spring通过一个配置文件描述Bean和Bean之间的依赖关系，利用Java语言的反射功能实例化Bean并建立Bean之间的依赖关系。 

​	Spring的IoC容器在完成这些底层工作的基础上，还提供了Bean实例缓存、生命周期管理、Bean实例代理、时间发布、资源装载等高级服务。

#### 2.2、bean初始化

##### 2.2.1、BeanFactory初始化

​	com.springframework.beans.factory.BeanFactory，是Spring框架最核心的接口，它提供了高级IoC的配置机制；使管理不同类型的Java对象成为可能；是Spring框架的基础设施，面向Spring本身。

* 初始化

  ```java
  //揭示了在IoC容器实现中的关键类，比如：Resource、DefaultListableBeanFactory、BeanDefinitionReader，之间的相互联系，相互协作
  ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
  Resource res = resolver.getResource("classpath:com/smart/beanFactory/benas.xml");
  //ClassPathResource res = new ClassPathResource("com/samrt/beanFactory/benas.xml");
  //在Spring3.2中被废弃掉了，不建议使用
  //BeanFactory bf = new XmlBeanFactory(res);
  DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
  XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
  reader.loadBeanDefinitions(res);
  System.out.println("init BeanFactory");
  Car car = factory.getBean("car",Car.class);
  System.out.println("car bean is ready for use!");
  car.introduce();
  ```

​        XmlBeanFactory通过Resource装载Spring配置信息并启动IoC容器，然后就可以通过getBean方法从IoC容器获取Bean了。通过BeanFactory启动IoC容器时，不会初始化配置文件中定义的Bean，初始化动作发生在第一个调用时 。对于SingleTon的Bean来说，BeanFactory会缓Bean。

* beans.xml内容

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
      xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd"
      >
  	<bean id="car" class="com.smart.Car"></bean>
  </beans>
  ```

  **!**在初始化BeanFactory的时候必须为其提供一种日志框架，如使用Log4j，即在类路径下提供Log4J的配置文件，这样启动Spring容器才不会报错。

##### 2.2.2、APPlicationContext初始化

​	com.springframework.context.ApplicationContext，建立在BeanFactory基础上，提供了更多面向应用的功能，它提供了国际化支持和框架事件体系，更易于创建应用；面向使用Spring的开发者，几乎所有的应用场合我们都直接使用ApplicationContext而非底层的BeanFactory。

* 初始化

  ```java
  ApplicationContext ctx = new ClassPathXmlApplicationContext(new String[]{"conf/beans1.xml","conf/beans2.xml"});
  Car car = ctx.getBean("car",Car.class);
  ```

​       ApplicationContext的初始化和BeanFactory有一个重大的区别：后者在初始化容器时，并未实例化Bean，直到第一次访问某个Bean时才实例目标Bean；而前者则在初始化应用上下文时就实例化所有单实例的Bean。因此ApplicationContext的初始化时间会比BeanFactory稍长一些，不过稍后的调用则没有“第一次惩罚”的问题；另一个最大的区别，前者会利用Java反射机制自动识别出配置文件中定义的BeanPostProcessor、InstantiationAwareBeanPostPrcecssor和BeanFactoryPostProcessor，并自动将他们注册到应用上下文中；而后者需要在代码中通过手工调用addBeanPostProcessor方法进行注册。这也是为什么在应用开发时，我们普遍使用ApplicationContext而很少使用BeanFactory的原因之一。可以在beans的属性中定义default-lazy-init="true"，达到延迟初始化的目的，这不能保证所有的bean都延迟初始化，因为有的bean可能被依赖导致初始化。不推荐延迟初始化。

##### 2.2.3、WebApplicationContext初始化

​	WebApplicationContext是专门为Web应用准备的，它允许从相对于web根目录的路径中加载配置文件完成初始化工作。从WebAppicationContext中获得ServletContext的引用，整个Web应用上下文对象将作为属性放置到ServletContext中，以便Web应用环境可以访问Spring应用上下文。Spring专门提供了一个工具类WebApplicationContextUtils，通过该类的getApplicationContext(ServletContext sc)方法，可以从ServerletContext中获取WebApplicationContext的实例。

* 初始化

​        WebApplicationContext的初始化方式和BeanFactory、ApplicationContext有所区别，因为WebApplicationContext需要ServletContext实例，也即是说它必须在Web容器的前提下才能完成启动的工作。有过Web开发经验的读者都知道可以在web.xml中配置自启动的Servlet或定义Web容器监听器，借助这两者中的任何一个，我们就可以完成启动Spring Web应用上下文的工作。所有版本的Web容器都可以定义自启动的Servlet，但只有Servlet2.3及以上版本的Web容器才支持Web容器监听器。有些即使支持Servlet2.3的Web服务器，但也不能再Servlet初始化之前启动Web监听器，比如Weblogic 8.1,Websphere 5.x,[Oracle](http://lib.csdn.net/base/oracle) OC4J 9.0。Web.xml里面的配置节点如下：

**通过web容器监听器引导**

```xml
<!--指定配置文件-->
<context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>
   /WEB_INF/applicationContext.xml
  </param-value>
</context-param>
<!--声明Web容器监听器-->
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

**通过自启动的Servlet引导**

```xml
<!--指定配置文件-->
<context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>
   /WEB_INF/applicationContext.xml
  </param-value>
</context-param>
<!--声明自启动的Servlet-->
<servlet>
  <servlet-name>springContextLoaderServlet</servlet-name>
  <servlet-class>org.springframwork.web.context.ContextLoaderServlet</servlet-class>
  <load-on-startup>1</load-on-startup>
</servlet>
```

由于WebApplicationContext需要使用日志功能，所以用户可以将Log4J的配置文件放在类路径WEB_INF/classes下，这时Log4J即可顺利启动。如果放置在其他位置，则必须在web.xml中声明其路径。Spring为启动Log4j提供了两个类似于WebApplicationContext的实现类：Log4JConfigSevlet和Log4JConfigListener,不管采用哪种方式，都必须保证在装载Spring配置文件之前装载Log4J配置信息。

```xml
<!--指定Log4J配置文件位置-->  
<context-param>  
  <param-name>log4jConfigLocation</param-name>  
  <param-value>/WEB-INF/classes/log4j.xml</param-value>  
</context-param>  
<!--声明Log4J监听器-->  
<listener>  
  <listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>  
</listener>  
```

**使用标注@Configuration的Java类来提供配置信息的配置**

```xml
<web-app>
  <!--通过指定Context参数，让Spring使用AnnotationConfigWebApplicationCOntext而非XMLApplicationContext来启动容器-->
  <context-param>
    <param-name>contextClass</param-name>
  	<param-value>
  	 org.springframwork.web.context.support.AnnotationConfigWebApplicationContext
 	</param-value>
  </context-param>
  <!--指定标注了Configuration的配置类，可以使用逗号或空格分隔多个-->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
      com.smart.APPConfig1
    </param-value>
  </context-param>
  <!--ContextLoaderListener会根据上面的配置使用AnnotationConfigWebApplicationContext来根据contextConfigLoaction指定的配置类来启动Spring容器-->
  <listener>
  	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
</web-app>
```

**使用Groovy DSL配置的配置**

```xml
<web-app>
  <!--通过指定Context参数，让Spring使用GroovyWebApplicationCOntext而非XMLApplicationContext来启动容器-->
  <context-param>
    <param-name>contextClass</param-name>
  	<param-value>
  	 org.springframwork.web.context.support.GroovyWebApplicationContext
 	</param-value>
  </context-param>
  <!--指定标注了Groovy的配置文件，可以使用逗号或空格分隔多个-->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
      Classpath*:conf/spring_mvc.groovy
    </param-value>
  </context-param>
  <!--ContextLoaderListener会根据上面的配置使用GroovyWebApplicationContext来根据contextConfigLoaction指定的配置文件来启动Spring容器-->
  <listener>
  	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
</web-app>
```

### 2.3、bean的生命周期及作用域

Spring为Bean提供了细致周全的生命周期过程，通过实现特定的接口或通过<bean>属性设置，都可以对Bean的生命周期过程施加影响，Bean的生命周期不但和其实现的接口相关，还与Bean的作用范围有关。为了让Bean绑定在Spring框架上，我们推荐使用配置方式而非接口方式进行Bean生命周期的控制。

在实际的开发过程中，很少需要控制Bean生命周期，而是把这个工作交给Spring，采用默认的方式。

Bean的作用域：singleton,prototype,request,session,globalSession，默认是singleton。

|      类型       |                    说明                    |
| :-----------: | :--------------------------------------: |
|   singleton   | 在Spring IoC容器中仅存在一个bean实例，bean以单实例的方式存在。 |
|   prototype   | 每次从容器中调用bean的时候，都返回一个新的实例，即每次调用getBean()时，相当于new XxxBean()操作 |
|    request    | 每次http强求都会创建一个新的bean，该作用域仅适用于WebApplicationContext环境 |
|    session    | 同一个http session共享一个bean ，不同的http session使用不同的bean，该作用域仅适用于WebApplicationContext环境 |
| globalSession | 同一个全局session共享一个bean，一般应用于portlet应用环境，该作用域仅适用于WebApplicationContext环境 |

### 2.4、bean的配置方式

基于Xml配置方式中，配置文件的3种格式：完整配置格式、简化配置方式、使用p命名空间 。

基于注解配置方式中，使用到的注解符号：@Compoment,@Repository,@Service,@Controller,@Autowired(@Resource,@Inject),@Qualifier,@Scope,@PostConstruct,@PreDestroy  。

基于Java类配置方式中，使用到的注解符号：@Configuration,@Bean 。

Bean不同配置方式比较，总结如下：

|            |                 基于xml配置                  |                  基于注解配置                  |                基于Java类配置                 |              基于Groovy DSL配置              |
| ---------- | :--------------------------------------: | :--------------------------------------: | :--------------------------------------: | :--------------------------------------: |
| bean定义     | 在xml文件中通过<bean>元素定义Bean，如<bean class="com.smart.Car"/> | 在Bean实现类处通过标注@Compoent或衍生类(@Repository、@Service及@Controller)定义Bean | 在标注了@Configuration的Java类中，通过在类方法上标注@Bean来定义一个Bean。方法必须提供Bean的实例化逻辑 | 在Groovy文件中通过DSL定义一个Bean，如：userDao(userDao) |
| bean名称     | 通过<bean>的id或name属性定义，如<bean id="car" class="com.smart.Car"/> | 通过注解的value属性定义，如@Compoent("car"),默认名称为小写字母开头的类名(不带包名)car。 |  通过@Bean的那么属性定义，如@Bean("car"),默认名称为方法名。  | 通过Groovy的DSL定义Bean的名称(Bean的类型，Bean构建函数参数)，如：logonService(logonService, userDao) |
| bean注入     | 通过<property>子元素或通过p命名空间的动态属性，如p：userDao-ref=“userDao”进行注入 | 通过在成员变更或方法入参处标注@Autowired，按类型匹配自动注入，还可以配合使用@Qualifer按名称匹配方式注入 | 比较灵活，可以在方法处通过@Autowired使方法入参绑定Bean，然后在方法中通过代码进行注入；还可以通过调用配置类的@Bean方法进行注入 | 比较灵活，可以在方法处通过ref()方法进行注入，如ref("logDao")  |
| bean生命过程方法 | 通过<bean>的init-method和destory-method属性指定Bean实现类的方法名，最多只能指定一个初始化方法和一个销毁方法 | 通过在目标方法上标注@PostConstruct和@PreDestory注解指定初始化或销毁方法，可以定义任意多个 | 通过@Bean的initMethod或destoryMethod指定一个初始化或销毁方法。对于初始化方法来说，可以直接在方法内部通过代码的方式灵活定义初始化逻辑。 | 通过bean-> bean.initMethod或bean.destoryMethod指定一个初始化或销毁方法 |
| bean作用范围   | 通过<bean>的scope属性指定，如<bean class="com.smart.UserDao" scope="prototype"/> |  通过在类定义处标注@Scope指定，如@Scope(“prototype”)  |          通过在Bean方法定义处标注@Scope指定          |     通过bean->bean.scope="prototype"指定     |
| bean延迟初始化  | 通过<bean>的lazy-init属性指定，默认为default，继承于<beans>的default-lazy-init设置，该值默认为false |      通过在类定义处标注@Lazy指定，如@Lazy(true)       |          通过在Bean方法定义处标注@Lazy指定           |       通过bean->bean.lazyInit=true指定       |

这4中配置文件很难说孰优孰劣，它们都有各自的适用场景。

|      |                 基于XMl配置                  |                基于注解配置                 |                基于Java类配置                 |              基于Groovy DSL配置              |
| :--: | :--------------------------------------: | :-----------------------------------: | :--------------------------------------: | :--------------------------------------: |
| 适用场景 | (1)Bean实现类来源于第三方类库，如DataSource、JdbcTemplate等，因无法再类中标注注解，所以通过XML配置方式较好。(2)命名空间的配置，如aop、context等，只能采用基于Java的配置 | Bean的实现类是当前项目开发的，可以直接在Java类中使用基于注解的配置 | 基于Java类配置的优势在于可以通过代码的方式控制Bean的初始化的繁体逻辑。如果实例化Bean的逻辑比较繁杂，则比较适合基于Java类配置的方式。 | 基于Groovy DSL配置的优势在于可以通过Grooovy脚本灵活的控制Bean初始化的过程，如果实例化Bean的逻辑比较复杂，则较适合基于Groovy DSL配置的方式 |

### 2.5、总结

BeanFactory、ApplicationContext和WebApplicationContext是Spring框架三个最核心的接口，框架中其他大部分的类都围绕它们展开、为它们提供支持和服务。

在这些支持类中，Resource是一个不可忽视的重要接口，框架通过Resource实现了和具体资源的解耦，不论它们位于何种介质中，都可以通过相同的实例返回。

与Resource配合的另一个接口是ResourceLoader，ResourceLoader采用了策略模式，可以通过传入资源的信息，自动选择适合的底层资源实现类，为生产对资源的引用提供了极大的便利。

在一些非Web项目中，在入口函数（main）中，会显示的初始化Spring容器，比如：ApplicationContext instance = new ClassPathXmlApplicationContext(new String[]{"applicationContext.xml"})，这和Web项目中通过Listener（web.xml）初始化Spring容器，效果是一样的。一个是手工加载，一个是通过web容器加载，对于Spring容器的初始化，效果是一样的。