title: Spring基础小结
date: 2015-04-15 17:19:00
updated: 2015-12-25 17:19:00
categories:
- tool
tags:
- tool
- spring
---

> Spring IoC 就像串烧，把对象串起来。

## Spring IoC

IoC，控制反转，或称DI(Denpendency Injection，依赖注入)。Spring做到了把App运行时需要的Object在配置文件中就体现出来，并且在App初始化时就把这些Object创建出来。在Spring之前，App需要主动去`new Object()`，将各个Object组织在一起，然后管理所有Object的生命周期。Spring出现后，把这些都通过配置文件来解决了。在App启动时，Spring加载定义好的xml文件，其中描述了各个Object的关系，Spring接管所有Object的创建以及生命周期的管理。

比如，下面一个`taskExecutor`定义，在其内部还有一个`rejectedExecutionHandler`对象参数。Spring会创建一个taskExecutor的单例，rejectedExecutionHandler也会同时new出来。如果传入的对象参数已经定义过，则用`<property name='abc' ref=abc>`即可
``` xml
<bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
	<property name="corePoolSize" value="4"/>
	<property name="maxPoolSize" value="100"/>
	<property name="keepAliveSeconds" value="200"/>
	<property name="rejectedExecutionHandler">
		<bean class="java.util.concurrent.ThreadPoolExecutor$CallerRunsPolicy"/>
	</property>
</bean>
```

Spring手动加载xml文件
``` java
ApplicationContext appContext = new ClassPathXmlApplicationContext(new String[]{"classpath:config/spring/appcontext-*.xml"});
ThreadPoolTaskExecutor taskExecutor = (ThreadPoolTaskExecutor) appContext.getBean("taskExecutor");
```

Tomcat通过Listener集成Spring, 其中`classpath*`的意思是也要加载jar包里面的classpath
``` xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>
	classpath*:config/spring/common/appcontext-*.xml,
	classpath*:config/spring/local/appcontext-*.xml,
	classpath*:/config/spring/pagelet/appcontext-pagelet-core.xml
	classpath*:/config/spring/wedbase/appcontext-*.xml
	</param-value>
</context-param>
<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

## Spring FactoryBean
`FactoryBean`的一个重要应用场景就是RPC(Remote Process Call)。RPC本质上就是用TCP协议传送stub，stub就是一系列参数的request，然后接收response。request之间的区别就是那些参数，对于Java来说就是类名、方法名、参数类型、参数值，所以`FactoryBean`需要将这些参数包装到TCP的一个统一proxy上就可以了。bean上的class是这个proxy类，但是对外又要伪装成传入参数的那个类，因为最终RPC的效果应该是各个被代理的类就像是普通类一样的。

如下定义一个bean，其中`RpcIfaceFactory`就是一个FactoryBean
``` xml
<bean id="hello" class="factory.RpcIfaceFactory" init-method="init">
	<property name="url" value="http://localhost:8081/apis/hello/" />
	<property name="iface" value="iface.Hello" />
	<property name="user" value="client1" />
	<property name="password" value="client1" />
	<property name="overloadEnabled" value="true" />
	<property name="debug" value="true" />
</bean>
```

RpcIfaceFactory的实现，主要是将传入的iface作为对外伪装的类型，而真正做操作时候用一个proxy类来发送tcp请求。
``` java
public class RpcIfaceFactory implements FactoryBean {
	//input
	private String url;
	private String user;
	private String password;
	private boolean overloadEnabled;
	private boolean debug;
	private String iface;
	
	//internal
	private Class<?> objType;
	private Object obj;
	
	public void init() throws ClassNotFoundException, MalformedURLException{
		HessianProxyFactory hessianFactory = new HessianProxyFactory();
		hessianFactory.setUser(user);
		hessianFactory.setPassword(password);
		hessianFactory.setDebug(debug);
		hessianFactory.setOverloadEnabled(overloadEnabled);
		objType = ClassUtils.loadClass(iface);
		obj = hessianFactory.create(objType, url);
	}

	public Object getObject() throws Exception {
		return obj;
	}

	public Class<?> getObjectType() {
		return objType;
	}

	public boolean isSingleton() {
		return false;
	}

	public String getUrl() {
		return url;
	}

	public void setUrl(String url) {
		this.url = url;
	}

	public String getUser() {
		return user;
	}

	public void setUser(String user) {
		this.user = user;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	public boolean isOverloadEnabled() {
		return overloadEnabled;
	}

	public void setOverloadEnabled(boolean overloadEnabled) {
		this.overloadEnabled = overloadEnabled;
	}

	public boolean isDebug() {
		return debug;
	}

	public void setDebug(boolean debug) {
		this.debug = debug;
	}

	public String getIface() {
		return iface;
	}

	public void setIface(String iface) {
		this.iface = iface;
	}

}
```

> Spring AOP就像一个后门，知道了暗号就能走捷径。

## Spring AOP

AOP需要明确几件事情：1）切哪些；2）什么时候切；3）怎么切，前面两个是配置的问题，后面一个是应用的问题。切哪些，通过expression来声明；什么时候切，通过before、after、around之类的关键字声明；怎么切，就看对应的方法里面怎么写了。一般的，AOP是一个难点，首先以读懂经典的代码和模仿为主。在没有把握的时候不要轻易尝试，因为极有可能反而把逻辑搞得复杂了。

如下就定义了“切哪些”和“什么时候切”，在Java类里面又进行了实现，即“怎么切”。
``` xml
<aop:config>
	<aop:aspect id="TestAspect" ref="aspectBean">
		<aop:pointcut id="businessService" expression="execution(* test.spring.aop.*.*(..))" />
		<aop:before pointcut-ref="businessService" method="doBefore"/>
		<aop:after pointcut-ref="businessService" method="doAfter"/>
		<aop:around pointcut-ref="businessService" method="doAround"/>
		<aop:after-throwing pointcut-ref="businessService" method="doThrowing" throwing="ex"/>
	</aop:aspect>
</aop:config>
<bean id="aspectBean" class="test.spring.aop.TestAspect" />
<bean id="aService" class="test.spring.aop.AServiceImpl"></bean>
<bean id="bService" class="test.spring.aop.BServiceImpl"></bean>
```

``` java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
public class TestAspect {

    public void doAfter(JoinPoint jp) {
        System.out.println("log Ending method: " + jp.getTarget().getClass().getName() + "." + jp.getSignature().getName());
    }

    public Object doAround(ProceedingJoinPoint pjp) throws Throwable {
        long time = System.currentTimeMillis();
        Object retVal = pjp.proceed();
        time = System.currentTimeMillis() - time;
        System.out.println("process time: " + time + " ms");
        return retVal;
    }

    public void doBefore(JoinPoint jp) {
        System.out.println("log Begining method: " + jp.getTarget().getClass().getName() + "." + jp.getSignature().getName());
    }

    public void doThrowing(JoinPoint jp, Throwable ex) {
        System.out.println("method " + jp.getTarget().getClass().getName() + "." + jp.getSignature().getName() + " throw exception");
        System.out.println(ex.getMessage());
    }
}
```

## Spring Annotation

Spring注解是用来简化xml配置文件的，但是并不能做到没有配置文件，因为注解的扫描开启开关都是写在配置文件里面的。在遇到Spring注解的时候要知道怎么回事，自己也要会写简单的注解。

### IoC注解
- 前提, `<context:annotation-config/>` `<context:component-scan base-package="xx.xx, yy.yy"/>` 用来扫描`@Autowired`, `@Resource`
- 对应的xml中不需要再写`<property>`
- 在Java文件中不需要`set`和`get`, 只需要在对应的属性上`@Autowired`, `@Resouce`
- 如果连`<bean>`都不想写，可以用注解`@Component`

### AOP注解
首选需要开启`<aop:aspectj-autoproxy/>`

``` java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;

@Aspect
public class TestAspect {

    @Pointcut("execution(* test.spring.aop.*.*(..))")
    private void pointCutMethod() {
    }

    //声明前置通知
    @Before("pointCutMethod()")
    public void doBefore() {
        System.out.println("前置通知");
    }

    //声明后置通知
    @AfterReturning(pointcut = "pointCutMethod()", returning = "result")
    public void doAfterReturning(String result) {
        System.out.println("后置通知");
        System.out.println("---" + result + "---");
    }

    //声明例外通知
    @AfterThrowing(pointcut = "pointCutMethod()", throwing = "e")
    public void doAfterThrowing(Exception e) {
        System.out.println("例外通知");
        System.out.println(e.getMessage());
    }

    //声明最终通知
    @After("pointCutMethod()")
    public void doAfter() {
        System.out.println("最终通知");
    }

    //声明环绕通知
    @Around("pointCutMethod()")
    public Object doAround(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("进入方法---环绕通知");
        Object o = pjp.proceed();
        System.out.println("退出方法---环绕通知");
        return o;
    }
}
```

## Code
[MySampleCode@Github](https://github.com/wtcctw/spring-basic)