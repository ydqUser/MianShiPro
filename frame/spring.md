
<!-- TOC -->

- [Spring](#spring)
    - [IOC](#ioc)
    - [AOP：面向切面编程](#aop面向切面编程)
        - [动态代理](#动态代理)
        - [Spring Aop和AspectJ AOP有啥区别](#spring-aop和aspectj-aop有啥区别)
    - [Bean](#bean)
    - [事务](#事务)
        - [事务隔离级别](#事务隔离级别)
            - [事务产生的问题](#事务产生的问题)
            - [事务隔离级别](#事务隔离级别-1)
        - [事务传播行为](#事务传播行为)
        - [事务底层实现](#事务底层实现)
- [SpringBoot](#springboot)

<!-- /TOC -->

spring 单例 多线程安全
controller的static变量 线程安全问题

## Spring
### IOC
### AOP：面向切面编程

将那些与业务无关，却被多个模块共同调用的东西，封装起来，减少代码的重复

#### 动态代理

[cglib refer](https://blog.csdn.net/yhl_jxy/article/details/80633194)

- jdk动态代理(基于接口)：利用反射为需要增强的方法生成代理类，调用invocationHandle的invoke执行相应方法
- cglib动态代理(基于类)：CGLIB包的底层是通过使用一个小而快的字节码处理框架ASM，来转换字节码并生成新的类。生成一个需要代理类的子类，子类中拦截父类中的方法
    - CGLib不能对声明为final的方法进行代理，因为CGLib原理是动态生成被代理类的子类
    - 实现CGLIB动态代理必须实现MethodInterceptor(方法拦截器)接口
    - spring可以强制使用cglib（在Spring配置中加入<aop:aspectj-autoproxy proxy-target-class=“true”/>）
    
JDK和CGLib的性能对比：jdk对jdk动态代理进行优化，使得jdk代理效率高于CGLib代理  
    
#### Spring Aop和AspectJ AOP有啥区别

Spring AOP是运行时增强，AspectJ AOP是编译时增强

### Bean

- Bean生命周期
	- 实例化bean对象
	- 如果涉及到属性值，则通过set方法对属性进行赋值
	- 判断是否实现Aware接口，实现则调用相关方法，比如BeanNameAware，BeanClassLoaderAware
	- 通过BeanPostProcessor类的postProcessBeforeInitialization进行Bean的前置通知处理
	- 初始化Bean
	- 判断配置文件中是否包含init-method属性，有则执行相关方法
	- 通过BeanPostProcessor类的postProcessAfterInitialization进行Bean的后置通知处理
	- 判断是否实现Disposable接口，有则执行destroy方法销毁bean
	
- Aware：实现了Aware接口的bean在被初始化之后，可以获取相应资源。通过Aware接口，可以对Spring相应资源进行操作
    - ApplicationContextAware会向实现了这个接口的Bean提供ApplicationContext，也就是IOC容器的上下文的信息。当然，实现了这个接口的Bean必须配置到bean的配置文件中去，并且由Spring的Bean容器去加载，这样才能实现这样的效果
    - BeanNameAware类似，它会提供一个关于BeanName定义的这样一些内容
    
- 创建对象的三种方式：
    - 使用无参构造器创建
    - 使用静态工厂方法创建
    - 使用实例化对象工厂方法创建
```
<!-- 使用无参数构造器 -->
<bean id="person" class="com.boe.Person"></bean>
<!-- 使用静态工厂方法 -->
<bean id="cal" class="java.util.Calendar" factory-method="getInstance">
 <!-- 使用实例化对象工厂方法 -->
<bean id="date" factory-bean="cal" factory-method="getTime"></bean>
```
- 作用域
    - singleton
        - 对象只创建了一次
    - prototype
        - 创建了多个对象

延迟加载：默认情况下容器启动之后，会将作用域为singleton的bean创建好，设置延迟加载容器启动之后，对作用域为singleton的bean不再创建，直到调用getBean方法才会创建，设置延迟加载需在配置文件中设置lazy-init属性

```
(1）scope="singleton"，lazy-init="false"：启动容器就创建对象，并且只有一个
(2）scope="singleton"，lazy-init="true"：启动容器不会创建对象，直到调用getBean方法才会创建对象，并且只有一个
(3）scope="prototype"，无论是否设置延迟加载，均只有在调用getBean方法才会创建对象，并且是创建多个不同的对象
```


### 事务

#### 事务隔离级别

##### 事务产生的问题

- 脏读
- 不可重复读：读取了修改前的数据
- 幻读

##### 事务隔离级别

- READ UNCOMMITTED（读未提交数据）：允许事务读取未被其他事务提交的变更数据，会出现脏读、不可重复读和幻读问题。
- READ COMMITTED（读已提交数据）：只允许事务读取已经被其他事务提交的变更数据，可避免脏读，仍会出现不可重复读和幻读问题。
- REPEATABLE READ（可重复读）：确保事务可以多次从一个字段中读取相同的值，在此事务持续期间，禁止其他事务对此字段的更新，可以避免脏读和不可重复读，仍会出现幻读问题。
- SERIALIZABLE（序列化）：确保事务可以从一个表中读取相同的行，在这个事务持续期间，禁止其他事务对该表执行插入、更新和删除操作，可避免所有并发问题，但性能非常低

#### 事务传播行为

- PROPAGATION_REQUIRED–支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。
- PROPAGATION_SUPPORTS–支持当前事务，如果当前没有事务，就以非事务方式执行。
- PROPAGATION_MANDATORY–支持当前事务，如果当前没有事务，就抛出异常。
- PROPAGATION_REQUIRES_NEW–新建事务，如果当前存在事务，把当前事务挂起。
- PROPAGATION_NOT_SUPPORTED–以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
- PROPAGATION_NEVER–以非事务方式执行，如果当前存在事务，则抛出异常
- PROPAGATION_NESTED–以非事务方式执行，如果当前存在事务，则创建一个事务作为当前事务的嵌套事务执行，如果不存在事务则为require

REQUIRED,REQUIRES_NEW,NESTED异同
- NESTED和REQUIRED修饰的内部方法都属于外围方法事务，如果外围方法抛出异常，这两种方法的事务都会被回滚。但是REQUIRED是加入外围方法事务，所以和外围事务同属于一个事务，一旦REQUIRED事务抛出异常被回滚，外围方法事务也将被回滚。而NESTED是外围方法的子事务，有单独的保存点，所以NESTED方法抛出异常被回滚，不会影响到外围方法的事务。
- NESTED和REQUIRES_NEW都可以做到内部方法事务回滚而不影响外围方法事务。但是因为NESTED是嵌套事务，所以外围方法回滚之后，作为外围方法事务的子事务也会被回滚。而REQUIRES_NEW是通过开启新的事务实现的，内部事务和外围事务是两个事务，外围事务回滚不会影响内部事务。

- 外围方法抛异常
    - NESTED和REQUIRED都会回滚
    - REQUIRES_NEW是通过开启新的事务实现的，内部事务和外围事务是两个事务，外围事务回滚不会影响内部事务
- 抛异常是否影响外围方法
    - REQUIRED是加入外围方法事务，外围方法事务也将被回滚
    - NESTED是外围方法的子事务，有单独的保存点，所以NESTED方法抛出异常被回滚，不会影响到外围方法的事务
    - REQUIRES_NEW都可以做到内部方法事务回滚而不影响外围方法事务

事务的传播机制演示
```
// UserSerivce.createUser(name)
@Transactional
public void createUser(String name) {
    // 新增用户基本信息
    jdbcTemplate.update("INSERT INTO `user` (name) VALUES(?)", name);

    //调用accountService添加帐户
    accountService.addAccount(name, 10000);
 ｝
 
 // AccountService.addAccount(name,initMoney) 方法的最后有一个异常
 @Transactional(propagation = Propagation.REQUIRED)
 public void addAccount(String name, int initMoney) {
     String accountid = new SimpleDateFormat("yyyyMMddhhmmss").format(new Date());
     jdbcTemplate.update("insert INTO account (accountName,user,money) VALUES (?,?,?)", accountid, name, initMenoy);
 
     // 出现分母为零的异常
     int i = 1 / 0;
 }
```
场景 | createUser | addAccount(异常) | 预测结果
---|---|---|---
场景1 | 无事务 | required | createUser（成功） addAccount（不成功）
场景2 | required | 无事务 | createUser（不成功） addAccount（不成功）
场景3 | required | not_supported | createUser（不成功） addAccount（成功）
场景4 | required | required_new | createUser（不成功） addAccount（不成功）
场景5 | required(异常移至createUser方法未尾) | required_new | createUser（不成功） addAccount（成功）
场景6 | required(异常移至createUser方法未尾)（addAccount 方法移至createUser方法的同一个类里） | required_new | createUser（不成功） addAccount（不成功）

对于场景二：虽然addAccount是无事务，但是createUser为required，会将addAccount放到一个事务，所以addAccount也不成功
#### 事务底层实现

Spring 支持编程式事务管理和声明式事务管理两种方式
- 编程式事务：编程式事务管理使用 TransactionTemplate
- 声明式事务：声明式事务管理建立在 AOP 之上的
    - 最大优点是不需要在业务逻辑代码中掺杂事务管理的代码，只需要通过@Transactional 注解的方式
    - 比编程式更加非侵入式，不足：最细粒度只能作用到方法级别，无法做到像编程式事务那样作用到代码块级别
    - 通过AOP功能，对方法前后进行拦截，将事务处理的功能编织到拦截的方法中，也就是在目标方法开始之前加入一个事务，在执行完目标方法之后根据执行情况提交或回滚事务


- IOC：控制反转

	- 思想

		- 反转

			- 将自主创建对象变成交给Spring管理

		- 控制什么？

			- 创建管理bean的权利，控制整个bean的生命周期

	- 实践

		- DI依赖注入

			- 配置文件讲外部资源注入到内部，容器加载了外部资源文件，对象，然后把这些资源注入到程序内部对象中，维护了程序内外对象之间的依赖关系



- 容器启动流程
- 用到的设计模式

	- 单例模式

		- 对象默认单例

	- 工厂模式

		- BeanFactory生产Bean过程

	- 代理模式

		- SpringAop基于代理模式

	- 模板方法

		- JdbcTemplate

	- 适配器模式

		- SpringAOP的增加或通知，用到了适配器模式

	- 包装器模式

		- 配置多个数据源时使用



- SpringMvc

	- 用户请求进来，交给dispatchServlet处理
	- dispatchServlet根据请求调用handleMapping解析对应的handle
	- 解析到相应的handle（controller）后，由handleAdapter根据handle处理相应业务逻辑
	- 处理完成之后返回ModelAndView视图
	- 视图解析器会将view封装成jsp返回到dispatchServlet在返回给用户
	
## SpringBoot

SpringBoot的运行机制(SpringBootApplication):
- Configuration：标注一个类为配置类
- ComponentScan：将指定包下需要装配的组件注册到容器中
- EnableAutoConfiguration：自动配置
SpringBoot根据配置文件自动装配所属依赖的类，再用动态代理的方式，注入到容器中

#### jpa

java persistence api，用注解或者xml描述对象和关系表的映射关系，并将运行期的实体对象持久化到数据库中


