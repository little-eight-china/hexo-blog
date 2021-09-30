---
title: Spring实战笔记（一）
date: 2018-10-01 16:20:23
categories: 
  - java杂记
tags: 
  - little_eight
---

## 装配bean

它提供了三种主要的装配机制：

 * 隐式的Bean发现机制和自动装配
 * 在Java中进行显式配置
 * 在XML中进行显式配置

### 自动化装配Bean

* 组件扫描（Component Scanning）：Spring会自动发现应用上下文中所创建的Bean
* 自动装配（Autowired）：Spring自动满足Bean之间的依赖

#### @Component 
这个注解表明该类会作为组件类（还有个类似的注解--@Named("bean name")，用法一样。）

#### @ComponentScan
不过，组件扫描默认是不开启的，用@ComponentScan注解，如果没有其他配置，它默认会扫描与配置类相同的包（包括这个包下的所有子包），查找带有@Component注解的类，并在Spring中自动为其创建一个Bean

#### @ComponentScan的属性
* basePackages,值为扫描对象包，如：@ComponentScan(basePackages={"first","second"...})
* basePackageClasses，值为扫描对象类，如：@ComponentScan(basePackageClasses={"first","second"...})

#### @Autowired（@Inject用法大致一样）
自动装配，bean之间的依赖，有构造器、setter方法、其他方法注入。

    在构造器上加此注解，如下，表明当Spring创建BeanA的bean时，会通过这个构造器来进行实例化并且会传入一个可设置给BeanB类型的bean
```java
@Component
public BeanA implement BeanDady{
    private BeanB b；
    @Autowired
    public BeanA（BeanB b）{
        this.b = b
    }
}
<!--more-->
```

    用在属性的setter方法上，比如：
```java
@Component
public BeanA implement BeanDady{
    private BeanB b；
    @Autowired
    public void setBeanB（BeanB b）{
        this.b = b
    }
}
```
    用在方法上,比如：
```java
@Component
public BeanA implement BeanDady{
    private BeanB b；
    @Autowired
    public void test（BeanB b）{
        this.b = b
    }
}
```
不管什么方式，Spring都会尝试满足方法参数上所声明的依赖。假如有且只有一个bean匹配依赖需求的话，那么这个bean将会被装配起来，其他情况的话，在应用上下文创建时，Spring会抛出异常。为了避免异常，你可以将@Autowired的属性required设置为false
```java
@Autowired(required=false)
```
设置false后，如果没有匹配的bean，Spring会设置这个bean为未装配的状态，当然你的代码也有可能会出现空指针异常。

### 通过Java代码装配bean

#### @Configuration
该注解表明这个类是一个配置类
### Bean
声明bean，我们给配置类里的方法加上此注解
```java
@Configuratione
public class BeanConfig{
    @Bean
    public BeanA beanA(){
        return new BeanA();
    }
}
```
@Bean注解会告诉Spring这个方法将会返回一个对象，该对象要注册为Spring应用上下文的bean。默认情况下，bean的id跟此方法名一样，你也可以设置@Bean的属性name来指定一个不同的名字
```java
@Bean(name="xxx")
```

### 通过XML装配bean
不喜欢这种，，，

#### 声明一个简单的bean
```xml
<bean id="beanA" class="com.BeanA"/>
```
### 小结
建议使用自动化配置，减少维护成本。
<!--more-->
## 高技装配

### @Primary
在声明bean时，会有多个匹配的可能，比如BeanDady是一个接口，实现类有BeanA、BeanB
```java
@Autowired
public void setBean(BeanDady beanDady){
    this.beanDady = beanDady;
}

```
在组件扫描时，Spring就无法判断是装配哪个然后出现异常，所以我们将这个@Primary注解加到指定的类或者方法上，就可成为匹配首选,在匹配群里，此注解只能是唯一。
```java
@Component
@Primary
public class BeanA implement BeanDady{
}
```
```java
@Bean
@Primary
public BeanA beanA(){
    return new BeanA();
}
```
当然xml方式也有。。
```xml
<bean id="beanA" class="com.BeanA" primary="true"/>
```

### @Qualifier
这个注解是使用限定符的主要方式，在注入的时候指定想要注入的bean
```java
@Autowried
@Qualifier("beanA")
public void setBean(BeanDady beanDady){
    this.beanDady = beanDady;
}
```
这个注解的值默认是bean的id，当然你也可以自定义限定符,这样匹配的时候就以这个“a”为值，跟@Primary一样，此注解只能唯一
```java
@Component
@Qualifier("a")
public BeanA implement BeanDady{}
```

## bean的作用域

Spring定义了多种作用域，可以基于这些作用域创建bean
* 单例（singleton）：在整个应用中，只创建bean的一个实例
* 原型（prototype）：每次注入或者通过Spring应用上下文获取的时候，都会创建一个新的bean实例
* 会话（session）：在web应用中，为每个会话创建一个bean实例
* 请求（request）：在web应用中，为每个请求创建一个bean实例

### @Scope
单例是默认的作用域，如果想选择其他作用域，在bean的类加上此注解,值的话，为了安全跟不易出错，用ConfigurationBeanFactory.SCOPE_PROTOTYPE常量更好。
```java
@Component
@Scope("prototype")
//这个更好。。 @Scope(ConfigurationBeanFactory.SCOPE_PROTOTYPE)
public class BeanA...
```
```java
@Bean
@Scope(ConfigurationBeanFactory.SCOPE_PROTOTYPE)
public BeanA beanA（）{
    return new BeanA（）；
}
```
当然xml也有。。
```xml
<bean id="beanA" class="com.BeanA" scope="prototype"/>
```


## 面向切面编程

### AOP术语
* 通知（advice）：定义切面是什么以及什么时候使用
* 切点（poincut）：定义切面在哪里
* 连接点（join point）：所有可能的切面
* 切面（aspect）：通知和切点的结合
* 引入（introduction）：向现有的类加入新方法或者属性
* 织入（weaving）：

#### 通知
* 前置通知（before）：在目标方法被调用前通知功能
* 后置通知（after）：在目标方法完成之后通知功能
* 返回通知（after-retrning）：在目标成功执行后调用通知
* 异常通知（after-throwing）：在目标方法抛出异常后通知
* 环绕通知（around）：通知包裹了被通知的方法，在被通知的方法调用之前和之后执行自定义的行为

### 编写切面

#### 编写切点
```java
execution(* com.Test.test(..)) && within(com.*)
```
* execution（），表示使用这种指示器
* 里面的“*”，这里表示方法返回值的类型，星号表示全部
* com.Test.test(..),指定类型和方法名，括号的..代表不在乎这个方法的参数是什么
* &&,和的意思，可选，后面的within（com.*）代表仅匹配com包


#### @Aspect、@Before、@AfterReturning、@AfterThrowing

```java
@Aspect
public class Qiemian{
    //调用之前，其他类型通知类似
    @Before("execution(* com.Test.test(..))")
    public void before(){}
    ...
} 
```
#### @Pointcut
当然每次都这么写切点表达式不麻烦死，所以这里可以用这个注解
```java
@Aspect
public class Qiemian {
    //定义切点的方法,里面的实现最好是空的
    @Pointcut("execution(* com.Test.test(..))")
    public void pointcutMethod()  {}
    //调用之前，其他类型通知类似
    @Before("pointcutMethod()")
    public void before() {}
    ...
} 
```

#### @Around
这个通知注解也是其他注解（@After是一定执行的。）的整合
```java
public class Qiemian {
    //定义切点的方法,里面的实现最好是空的
    @Pointcut("execution(* com.Test.test(..))")
    public void pointcutMethod()  {}
    //环绕通知 
    @Around("pointcutMethod()")
    public void around(ProceedingJoinPoint jp) {
        try{
            //这里是调用之前通知
            System.out.println("调用之前");
            //调用这个方法后，代表调用之后
            jp.proceed();
            // 这里是调用执行成功之后通知
            System.out.println("调用执行成功之后");
        } catch (Exection e){
            System.out.println("抛异常之后");
        }
    }
} 
```

当然xml也有，这里不搞了。
