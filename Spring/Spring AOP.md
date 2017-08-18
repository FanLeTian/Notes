# Spring AOP

### 1、概述

AOP，Aspect Oriented Programming，面向切面编程。AOP的出现，是作为OOP的有益补充；AOP的应用场合是受限的，它一般只适合于那些具有横切逻辑的应用场合：如性能监测、访问控制、事务管理、日志记录。OOP是通过纵向继承的机制，达到代码重用的目的；AOP通过横向抽取机制，把分散在各个业务逻辑中的相同代码，抽取到一个独立的模块中，还业务逻辑类一个清新的世界；把这些横切性的逻辑独立出来很容易，但如何将这些横切逻辑融合到业务逻辑中完成和原来一样的业务操作，才是事情的关键，这也正是AOP要解决的主要问题。

AOP术语：连接点（Joinpoint）、切点（Pointcut）、增强（Advice）、目标对象（Target）、引介（Introduction）、织入（Weaving）、代理（Proxy）、切面（Aspect）；

AOP织入方式：编译器织入、类装载期织入、动态代理织入，Spring采用动态代理织入，AspectJ采用前两种织入方式；

Spring采用了两种代理机制：基于JDK的动态代理和基于CGLib的动态代理；前者创建的代理对象性能差，后者创建对象花费时间长，都大约是10倍的差距，所以对于单例的代理对象或者具有实例池的代理对象，比较适合CGLib，反之适合JDK。

四种切面类型：@AspectJ、<aop:aspect>、Advisor、<aop:advisor>，JDK5.0以后全部采用@AspectJ方式吧。这四种切面定义方式，其底层实现实际是相同的，表象不同，本质归一。

1、@AspectJ使用JDK5.0注解和正规的AspectJ的切点表达式语言描述切面，由于Spring只支持方法的连接点，所以Spring仅支持部分AspectJ的切点语言，这种方式是被Spring推荐的，需要在xml中配置的内容最少，演示：<>

2、如果项目只能使用低版本的JDK，则可以考虑使用<aop:aspect>，这是基于Schema的配置方式

3、如果正在升级一个基于低版本Spring AOP开发的项目，则可以考虑使用<aop:advisor>复用已经存在的Advice类

4、基于Advisor类的方式，其实也是蛮自动的，在xml中配置一个Advisor节点，开启自动创建代理就好了。

<!--开启注解 -->
 <context:annotation-config />
 <!-- 开启自动切面代理 -->
 <aop:aspectj-autoproxy />

声明自动为spring容器中那些配置@aspectJ切面的bean创建代理，织入切面。当然，spring在内部依旧采用AnnotationAwareAspectJAutoProxyCreator进行自动代理的创建工作，但具体实现的细节已经被<aop:aspectj-autoproxy />隐藏起来了。<aop:aspectj-autoproxy />有一个proxy-target-class属性，默认为false，表示使用jdk动态代理织入增强，当配为<aop:aspectj-autoproxy  proxy-target-class="true"/>时，表示使用CGLib动态代理技术织入增强。不过即使proxy-target-class设置为false，如果目标类没有声明接口，则spring将自动使用CGLib动态代理；

### 2、JDK动态代理

​	JDK的动态代理主要涉及java.lang.reflect包中的两个类：Proxy和InvocationHandler。其中InvocationHandler是一个接口，可以通过实现该接口定义横切的逻辑，并通过反射机制调用目标类的代码，动态地将横切逻辑和业务逻辑编织在一起。而Proxy利用InvocationHandler动态创建了一个符号某一接口的实例，生成目标类的代理对象。

```java
//接口
public interface ForumService {
	void removeTopic(int topicId);
	void removeForum(int forumId);
}

//接口实现类
public class ForumServiceImpl implements ForumService {

	public void removeTopic(int topicId) {
//		PerformanceMonitor.begin("com.smart.proxy.ForumServiceImpl.removeTopic");
		System.out.println("模拟删除Topic记录:"+topicId);
		try {
			Thread.currentThread().sleep(20);
		} catch (Exception e) {
			throw new RuntimeException(e);
		}		
//		PerformanceMonitor.end();
	}

	public void removeForum(int forumId) {
//		PerformanceMonitor.begin("com.smart.proxy.ForumServiceImpl.removeForum");
		System.out.println("模拟删除Forum记录:"+forumId);
		try {
			Thread.currentThread().sleep(40);
		} catch (Exception e) {
			throw new RuntimeException(e);
		}		
//		PerformanceMonitor.end();
	}
}

//横切代码
public class PerformaceHandler implements InvocationHandler {
    private Object target;
	public PerformaceHandler(Object target){
		this.target = target;
	}
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		PerformanceMonitor.begin(target.getClass().getName()+"."+ method.getName());
		Object obj = method.invoke(target, args);
		PerformanceMonitor.end();
		return obj;
	}
}

//测试
public class ForumServiceTest {
	@Test
	public void proxy() {
		// 使用JDK动态代理
		ForumService target = new ForumServiceImpl();
		PerformaceHandler handler = new PerformaceHandler(target);
		ForumService proxy = (ForumService) Proxy.newProxyInstance(target
				.getClass().getClassLoader(),
				target.getClass().getInterfaces(), handler);
		proxy.removeForum(10);
		proxy.removeTopic(1012);

	} 
}
```

### 3、基于CGLib动态代理

对于没有通过接口定义业务方法的类，如何动态创建代理实例呢？JDK的代理技术显然已经黔驴技穷，CGLib作为一个替代者，填补了这个空缺。CGLib采用非常底层的字节码技术，可以为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类的方法调用，并顺势织入横切逻辑。

```java
import java.lang.reflect.Method;  
import net.sf.cglib.proxy.Enhancer;  
import net.sf.cglib.proxy.MethodInterceptor;  
import net.sf.cglib.proxy.MethodProxy;  
  
public class TestForumService  
{  
    public static void main(String[] args) throws Exception {  
        CglibProxy proxy = new CglibProxy();  
          
        ForumServiceImpl forumService = (ForumServiceImpl)proxy.getProxy(ForumServiceImpl.class);  
        forumService.removeTopic(100);  
    }  
}  
  
class CglibProxy implements MethodInterceptor  
{  
    private Enhancer enhancer = new Enhancer();  
  
    public Object getProxy(Class<?> clazz) {  
        enhancer.setSuperclass(clazz);  
        enhancer.setCallback(this);  
        return enhancer.create();   //通过字节码技术动态创建子类实例  
    }  
    //拦截父类所有方法的调用  
    public Object intercept(Object obj,Method method,Object[] args,MethodProxy proxy)  
        throws Throwable {  
        PerformanceMonitor.begin(obj.getClass().getName() + "." + method.getName());  
        Object result = proxy.invokeSuper(obj, args);  
        PerformanceMonitor.end();  
        return result;  
    }  
}  
  
//将要被注入切面功能的类  
class ForumServiceImpl  
{  
    public void removeTopic(int topicId) throws Exception {  
        System.out.println("模拟删除Topic记录：" + topicId);  
        //Thread.sleep(20);  
    }  
}  
  
//性能监视类（切面类）  
class PerformanceMonitor  
{  
    private static ThreadLocal<MethodPerformance> performanceRecord =   
        new ThreadLocal<MethodPerformance>();  
      
    public static void begin(String method) {  
        System.out.println("begin monitor...");  
        MethodPerformance mp = new MethodPerformance(method);  
        performanceRecord.set(mp);  
    }  
  
    public static void end() {  
        System.out.println("end monitor...");  
        MethodPerformance mp = performanceRecord.get();  
        mp.printPerformance();  
    }  
}  
  
class MethodPerformance  
{  
    private long begin,end;  
    private String serviceMethod;  
      
    public MethodPerformance(String serviceMethod) {  
        this.serviceMethod = serviceMethod;  
        this.begin = System.currentTimeMillis();  
    }  
  
    public void printPerformance() {  
        end = System.currentTimeMillis();  
        long elapse = end - begin;  
        System.out.println(serviceMethod + "花费" + elapse + "毫秒。");  
    }  
}  

//测试
public class ForumServiceTest {
	@Test
	public void proxy() {
		//使用CGLib动态代理
		CglibProxy cglibProxy = new CglibProxy();
		ForumService forumService = (ForumService)cglibProxy.getProxy(ForumServiceImpl.class);
		forumService.removeForum(10);
		forumService.removeTopic(1023);
	} 
}
```

### 4、Spring AOP实现方式，从低级到高级

这个例子通过一个“前置增强”的逐步简化步骤，演示SpringAOP如何从最低级的实现，到最高级（最简化、最自动）的实现的进化过程。

#### 4.1、纯粹代码方式（ProxyFactory）

```java
import java.lang.reflect.Method;  
import org.springframework.aop.BeforeAdvice;  
import org.springframework.aop.MethodBeforeAdvice;  
import org.springframework.aop.framework.ProxyFactory;  
  
public class TestBeforeAdvice {  
    public static void main(String[] args) {  
        Waiter target = new NaiveWaiter();  
        BeforeAdvice advice = new GreetingBeforeAdvice();  
        //Spring提供的代理工厂  
        ProxyFactory pf = new ProxyFactory();  
        //设置代理目标  
        pf.setTarget(target);  
        //为代理目标添加增强  
        pf.addAdvice(advice);  
        //生成代理实例  
        Waiter proxy = (Waiter)pf.getProxy();  
        proxy.greeTo("John");  
        proxy.serveTo("Tom");  
    }  
}  
  
interface Waiter {  
    void greeTo(String name);  
    void serveTo(String name);  
}  
  
class NaiveWaiter implements Waiter {  
    public void greeTo(String name) {  
        System.out.println("greet to " + name + "...");  
    }  
    public void serveTo(String name) {  
        System.out.println("serving " + name + "...");  
    }  
}  
  
class GreetingBeforeAdvice implements MethodBeforeAdvice {  
    @Override  
    public void before(Method method, Object[] args, Object obj)  
            throws Throwable {  
        String clientName = (String)args[0];  
        System.out.println("How are you! Mr." + clientName + ".");  
    }     
}  
```

**ProxyFactory代理工厂将GreetingBeforeAdvice的增强织入到目标类NaiveWaiter中。在ProxyFactory内部，依然使用的是JDK代理或CGLib代理，将增强应用到目标类中。**

**Spring定义了org.springframework.aop.framework.AopProxy接口，并提供了两个实现类，Cglib2AopProxy和JdkDynamicAopProxy。**

#### 4.2、Spring的配置方式（ProxyFactoryBean）

###### 1.beans.xml内容

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
    xmlns:p="http://www.springframework.org/schema/p"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd"  
    >  
    <bean id="greetingAdvice" class="cl.an.advice.GreetingBeforeAdvice"/>  
    <bean id="target" class="cl.an.advice.NaiveWaiter"></bean>  
    <bean id="waiter" class="org.springframework.aop.framework.ProxyFactoryBean"  
        p:proxyInterfaces="com.smart.advice.Waiter"  
        p:interceptorNames="greetingAdvice"  
        p:target-ref="target"></bean>  
</beans>  
```

* target:代理的目标对象
* proxyInterfaces:代理所需要实现的接口,可以是多个接口，该属性还有一个别名属性intefaces
* interceptorNames:需要织入目标对象的Bean列表，采用Bean的名称指定。这些Bean必须是实现了org.aopalloance.intercept.MethodInterceptor或org.springframework.aop.Advisor的Bean，配置中的顺序对应着调用顺序。
* singleton:返回的代理是否是单实例，默认是单实例的。
* optimize:当设置为true时，强制使用CGLib动态代理。对于singleton的代理，我们推荐使用CGLib；对于其他作用域的最好使用JDK动态代理。原因是使用CGLib创建代理时速度比较慢，但其创建出的代理对象运行效率比较高；而使用JDK创建代理的表现则正好是相反的。
* proxyTargetClass:是否对类进行代理(而不是对接口进行代理)，当设置为true时，使用CGLib动态代理。

###### 2.调用代码也变得简单了

```java
public class TestBeforeAdvice {  
    public static void main(String[] args) {  
        String configPath = "com/smart/advice/beans.xml";  
        ApplicationContext ctx = new ClassPathXmlApplicationContext(configPath);  
        Waiter waiter = (Waiter)ctx.getBean("waiter");  
        waiter.greeTo("John");  
        waiter.serveTo("Tom");  
    }  
}  
```

#### 4.3、切面(Advisor)方式

###### 1.beans.xml内容

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
    xmlns:p="http://www.springframework.org/schema/p"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd"  
    >  
    <bean id="regexpAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor"  
        p:advice-ref="greetingAdvice">  
        <property name="patterns">  
            <list>  
                <!-- 用正则表达式，只对greeTo方法进行增强 -->  
                <value>.*greeT.*</value>  
            </list>  
        </property>     
    </bean>  
    <bean id="waiter" class="org.springframework.aop.framework.ProxyFactoryBean"  
        p:interceptorNames="regexpAdvisor"  
        p:target-ref="target"  
        p:proxyTargetClass="true">  
    </bean>  
    <bean id="greetingAdvice" class="cl.an.advice.GreetingBeforeAdvice"/>  
    <bean id="target" class="cl.an.advice.NaiveWaiter"></bean>  
</beans>  

```

#### 4.4、自动创建代理

以上，我们通过ProxyFactoryBean创建织入切面的代理，对于小型系统，可以将就使用，但对拥有众多需要代理Bean的系统系统，需要做的配置工作就太多太多了。

但是，Spring为我们提供了自动代理机制，让容器为我们自动生成代理，把我们从繁琐的配置工作中解放出来。

在内部，Spring使用BeanPostProcessor自动完成这项工作。

我们可以使用BeanNameAutoProxyCreator，只对某些符合通配符的类进行增强，也可以使用DefaultAdvisorAutoProxyCreator，自动扫描容器中的Advisor，并将Advisor自动织入到匹配的目标Bean中，即为匹配的目标Bean自动创建代理。

###### 1.beans.xml内容

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
    xmlns:p="http://www.springframework.org/schema/p"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd"  
    >  
    <bean id="regexpAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor" 
        p:advice-ref="greetingAdvice">  
        <property name="patterns">  
            <list>  
                <!-- 用正则表达式，只对greeTo方法进行增强 -->  
                <value>.*greeT.*</value>  
            </list>  
        </property>     
    </bean>  
    <bean id="waiter" class="cl.an.advice.NaiveWaiter"/>  
    <bean id="greetingAdvice" class="cl.an.advice.GreetingBeforeAdvice"/>  
    <!-- 开启自动切面代理 -->  
    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>  
</beans>  
```

#### 4.5、@AspectJ定义切面

这是一种配置最少的切面定义方式，功能强大，但是需要学习AspectJ切点表达式语言。

###### 1.代码内容

```java
package com.smart.aspectj;  
  
import org.aspectj.lang.annotation.Aspect;  
import org.aspectj.lang.annotation.Before;  
import org.springframework.context.ApplicationContext;  
import org.springframework.context.support.ClassPathXmlApplicationContext;  
  
public class AspectJProxyTest {  
    public static void main(String[] args) {  
        String configPath = "cl/an/aspectj/beans.xml";  
        ApplicationContext ctx = new ClassPathXmlApplicationContext(configPath);  
        Waiter waiter = (Waiter)ctx.getBean("waiter");  
        waiter.greeTo("John");  
        waiter.serveTo("Tom");  
    }  
}  
  
  
interface Waiter {  
    void greeTo(String name);  
    void serveTo(String name);  
}  
  
class NaiveWaiter implements Waiter {  
    public void greeTo(String name) {  
        System.out.println("greet to " + name + "...");  
    }  
    public void serveTo(String name) {  
        System.out.println("serving " + name + "...");  
    }  
}  
  
@Aspect  
class PreGreetingAspect {  
    @Before("execution(* greeTo(..))")  
    public void beforeGreeting() {  
        System.out.println("How are you");  
    }  
}  

```

###### 2.beans.xml内容

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
    xmlns:p="http://www.springframework.org/schema/p"  
    xmlns:aop="http://www.springframework.org/schema/aop"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans  http://www.springframework.org/schema/beans/spring-beans-4.0.xsd  
    http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd"  
    >  
    <!-- 开启自动切面代理:基于AspectJ切面的驱动器 -->  
    <!-- true:代表使用cglib进行动态代理 -->  
    <aop:aspectj-autoproxy proxy-target-class="true" />  
  
    <bean id="waiter" class="cl.an.aspectj.NaiveWaiter"/>  
    <bean id="greetingAdvice" class="com.smart.aspectj.PreGreetingAspect"/>  
</beans>  
```

#### 4.6、基于Schema的AOP

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd">
	<aop:config proxy-target-class="true">
	    <aop:advisor advice-ref="testAdvice"  pointcut="execution(* com..*.Waiter.greetTo(..))"/>  
		<aop:aspect ref="adviceMethods">
			<aop:before method="preGreeting"
				pointcut="target(com.smart.NaiveWaiter) and args(name)"
				arg-names="name" />
			<aop:after-returning method="afterReturning"
				pointcut="target(com.smart.SmartSeller)" returning="retVal" />
			<aop:around method="aroundMethod"
				pointcut="execution(* serveTo(..)) and within(com.smart.Waiter)" />
			<aop:after-throwing method="afterThrowingMethod"
				pointcut="target(com.smart.SmartSeller) and execution(* checkBill(..))"
				throwing="iae" />
			<aop:after method="afterMethod"
				pointcut="execution(* com..*.Waiter.greetTo(..))" />
			<aop:declare-parents
				implement-interface="com.smart.Seller"
				default-impl="com.smart.SmartSeller"
				types-matching="com.smart.Waiter+" />
            <aop:before method="bindParams" 
                   pointcut="target(com.smart.NaiveWaiter) and args(name,num,..)"/>
		</aop:aspect>
	</aop:config>
    <bean id="testAdvice" class="com.smart.schema.TestBeforeAdvice"/>
	<bean id="adviceMethods" class="com.smart.schema.AdviceMethods" />
	<bean id="naiveWaiter" class="com.smart.NaiveWaiter" />
	<bean id="naughtyWaiter" class="com.smart.NaughtyWaiter" />
	<bean id="seller" class="com.smart.SmartSeller" />
</beans>
```

基于@AspectJ注解的切面，本质上是将切点、增强类型的信息使用注解进行描述，现在把这两个信息移到Schema的XML配置文件中，只是配置信息所在的地方不一样，表达的信息完全可以相同。

### 5、切面不同定义方式具体实现比较

![不同的方式比较](https://raw.githubusercontent.com/FanLeTian/Notes/master/picture/aop.png)