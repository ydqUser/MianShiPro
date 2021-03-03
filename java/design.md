
<!-- TOC -->

- [设计模式六大原则](#设计模式六大原则)
- [设计模式](#设计模式)
    - [单例模式](#单例模式)
    - [工厂模式](#工厂模式)
    - [java中的使用](#java中的使用)
        - [IO](#io)
        - [Spring](#spring)
    - [设计模式使用](#设计模式使用)
        - [单例模式使用](#单例模式使用)

<!-- /TOC -->

## 设计模式六大原则



## 设计模式

创建型模式（Creational Patterns)
    - 工厂模式、抽象工厂模式、单例模式、建造者模式、原型模式
    - 关注于对象的创建，同时隐藏创建逻辑
结构型模式（Structural Patterns) 
    - 适配器模式、过滤器模式、装饰模式、享元模式、代理模式、外观模式、组合模式、桥接模式
    - 关注类和对象之间的组合
行为型模式（Behavioral Patterns)
    - 责任链模式、命令模式、中介者模式、观察者模式、状态模式、策略模式、模板模式、空对象模式、备忘录模式、迭代器模式、解释器模式、访问者模式 
    - 关注对象之间的通信
    
![设计模式之间的关系](https://gitee.com/lylw/image/raw/master/img/design.jpg)

### 单例模式

- 单例对象的类必须保证只有一个实例存在
- 不会频繁地创建和销毁对象，浪费系统资源
- 使用场景：IO 、数据库连接、Redis 连接等

- 优点：该类只存在一个实例，节省系统资源；对于需要频繁创建销毁的对象，使用单例模式可以提高系统性能。
- 缺点：不能外部实例化（new），调用人员不清楚调用哪个方法获取实例时会感到迷惑，尤其当看不到源代码时。

[具体使用](#单例模式使用)

### 工厂模式

### 抽象工厂模式



### java中的使用

#### IO

- 适配器模式：由于 InputStream 是字节流不能享受到字符流读取字符那么便捷的功能，借助 InputStreamReader 将其转为 Reader 子类，因而可以拥有便捷操作文本文件方法；
- 装饰器模式：将 InputStream 字节流包装为其他流的过程就是装饰器模式，比如，包装为 FileInputStream、ByteArrayInputStream、PipedInputStream 等

#### Spring

- 代理模式：在 AOP 中有使用
- 单例模式：bean 默认是单例模式
- 模板方法模式：jdbcTemplate
- 工厂模式：BeanFactory
- 观察者模式：Spring 事件驱动模型就是观察者模式很经典的一个应用，比如，ContextStartedEvent 就是 ApplicationContext 启动后触发的事件
- 适配器模式：Spring MVC 中也是用到了适配器模式适配 Controller

### 设计模式使用

#### 单例模式使用

- 饿汉模式：在主动请求该类的时候就会在内存中初始化一个单例对象，在调用getInstance()的时候直接获取该对象
```
public class Singleton_Simple {  
    private static final Singleton_Simple simple = new Singleton_Simple();  
      
    private Singleton_Simple(){}  
      
    public static Singleton_Simple getInstance(){  
        return simple;  
    }  
} 
```
声明静态私有类变量，且立即实例化，保证实例化一次；私有构造，防止外部实例化（通过反射是可以实例化的，不考虑此种情况）；提供public的getInstance（）方法供外部获取单例实例

优点：线程安全；获取实例速度快 缺点：类加载即初始化实例，内存浪费

- 懒汉模式：在jvm进程启动并在我们主动使用该类的时候不会在内存中初始化一个单例对象，只有当我们调用getInstance()的时候才去创建该对象，
他的创建是在我们调用getInstance()静态方法之后，为了解决同步问题，我们在getInstance()方法上加了一个锁，这个方法每次只允许一个线程进来，虽然同步问题是解决了，
但是相应的性能问题就出现了

```
// 延迟加载加锁版(懒汉式)
public class Singleton_lazy {  
  
    private static Singleton_lazy lazy = null;  
      
    private Singleton_lazy(){}  
      
    public static synchronized Singleton_lazy getInstance(){  
        if( lazy == null ){  
            lazy = new Singleton_lazy();  
        }  
        return lazy;  
    }  
}
```
懒汉式没有同步锁：
- 优点：在获取实例的方法中，进行实例的初始化，节省系统资源
- 缺点：
    - 如果获取实例时，初始化工作较多，加载速度会变慢，影响系统系能
    - 每次获取实例都要进行非空检查，系统开销大
    - 非线程安全，当多个线程同时访问getInstance()时，可能会产生多个实例
加了同步锁之后，线程安全；但是每次获取实例都要加锁，耗费资源，实例生成之后，以后获取就不需要加锁了

double check双重锁：
```
public class SingletonClass { 

  private static SingletonClass instance = null; 

  public static SingletonClass getInstance() { 
    if (instance == null) { 
      synchronized (SingletonClass.class) { 
        if (instance == null) { 
          instance = new SingletonClass(); 
        } 
      } 
    } 
    return instance; 
  } 
}
```
当线程进行第一次检查的时候，代码读取到instance不为null时，instance引用的对象有可能还没有完成初始化([解析](https://dongguabai.blog.csdn.net/article/details/82828125))
> double check的问题：由于第一个if没有synchronized，所以synchronized带来的有序性对第一个if是不生效的。
于是就会出现一种情况，就是 new SingletonClass()时，由于重排序，先将引用分配给了instance，后创建的实例。这样在其他线程进入到第一个if的时候就会为false，使用未生成实例的引用。
这个时候使用volatile实际上是保证了第一个if读的时候的有序性，对volatile变量的写happen-before读，从而禁止了newSingletonClass();时的重排序
                  
用volatile修复双重检验锁([知乎为什么用volatile](https://www.zhihu.com/question/337265532/answer/794398131))

double check双重锁解决办法：
- SingletonClass instance为 volatile 变量
- 使用静态内部类：极避免了同步带来的性能损耗，又能够延迟加载(支持)
- 枚举

```
public class Singleton {
    private Singleton() {}

    private static class SingletonHolder {
        private static final Singleton singleton = new Singleton();
    }
    
    public static Singleton getInstance() {
        return SingletonHolder.singleton;
    }
}
```

