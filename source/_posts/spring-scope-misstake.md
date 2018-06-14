---
title: spring boot 默认的单例bean导致的错误
date: 2018-06-14 11:08:44
categories: spring
tags: spring
---

## # 介绍

spring在创建bean的时候，scope有5中类型：

1. singleton
2. prototype
3. request
4. session
5. global session

这个我们不讨论3,4,5.因为这三个取值都作用在web上面，没有1,2通用．

singleton,表示spring在初始化spring上下文的时候，就像标注@Component的类进行创建，并注册到springContext中，程序中每次请求这个实例的时候都会去springcontext中获取已经初始化的实例．进行方法的调用．scope默认的取值是singleton.

prototype,表示程序每次请求时，spring都会帮我们创建一个新的实例返回给我们．我们根据新实例进行方法的调用．

## # 错误

最近在开发公司项目时候，将当前时间声明在类的全局变量中，由于我没有指定scope的值为prototype导致在项目初始化的时候，该全局变量就已经被赋值．导致其他服务模块在应用该时间的时候都是一个时间－系统启动的时间．

![001](http://wx3.sinaimg.cn/large/74b07056ly1fs9r1ald8tj20rh07gt9t.jpg)

解决办法有两个:

1. 将声明的全局变量删除，获取当前时间从具体的方法中获取，这样当前时间就不会从堆内存中获取．
2. 声明该类的scope = prototype．这样在上级程序调用的时候，spring就会帮我们创建一个新的实例返回给我们，这样堆内存中的实例就会是新创建的实例，同时类的成员变量就会是创建实例的时间

最好还是使用第一种方式来解决这个问题，因为不断的创建新实例会导致堆内存的增加，垃圾回收将会回收这些堆内存中存在的垃圾实例．严重的情况下会触发full gc从而导致stop the world.使jvm暂时停止工作．

## # 测试

talk is cheap,show me the code.我们来写个例子来进行测试．

### # 错误
我们创建一个spring boot的工程，创建一个类声明一个全局变量，并赋值系统的当前时间．不声明scope为prototype．看看时间的变化．

启动类：
```java
@SpringBootApplication
public class SpringiocApplication {

	public static void main(String[] args) throws Exception {
		ConfigurableApplicationContext context = SpringApplication
		.run(SpringiocApplication.class, args);
		while (true) {
			context.getBean(TestScope.class).printDateTime();
			Thread.sleep(1000);
		}
	}

}
```

测试类：
```java
@Component
public class TestScope {

    private String nowDateTime = LocalDateTime.now()
    .format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));

    public void printDateTime() {
        System.out.println(nowDateTime);
    }
}
```

程序很简单，我们看一下输出结果．

![001](http://wx2.sinaimg.cn/large/74b07056ly1fs9r1aywx0j20bp07idg3.jpg)

可以看见输出时间是不变的，可以证明springcontext创建bean默认的策略是singleton.

### # 解决方法一

我们将声明的全局变量放入到方法体内，每一次循环都会创建新的时间指向局部变量．这样每次获取的局部变量的时间值都是当前时间．

```java
@Component
public class TestScope {

    public void printDateTime() {
        String nowDateTime = LocalDateTime.now()
        .format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
        System.out.println(nowDateTime);
    }
}
```

我们看一下输出结果.

![001](http://wx2.sinaimg.cn/large/74b07056ly1fs9r1bz7e6j208906t0su.jpg)

可以看见输出的时间是系统当前时间．

### # 解决方法二

我们来看下第二种解决方式．同样在类的成员变量声明当前时间．在类上声明注解@Scope(scopeName = "prototype")明确指出该类每次请求的时候就新创建一个实例．

```java
@Component
@Scope(scopeName = "prototype")
public class TestScope {

    private String nowDateTime = LocalDateTime.now()
    .format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));

    public void printDateTime() {
        System.out.println(nowDateTime);
    }
}
```

我们来看一下输出的结果．

![001](http://wx2.sinaimg.cn/large/74b07056ly1fs9r1cf95mj206h04odft.jpg)

可以看见输出的时间是系统当前时间.

## # 源码

### # singleton

我们先看看在不声明scope的情况下，spring是如何帮我们返回实例的．首先通过getBean方法进入．会进入AbstractApplicationContext的类中的getBean方法.

![001](http://wx4.sinaimg.cn/large/74b07056ly1fs9r1dbjwrj20na0p4q6u.jpg)

在getBean方法中我们看见该类的BeanFactory是DefaultListableBeanFactory,我们在getBean方法中可以看见接着调用了DefaultListableBeanFactory的getBean方法．好，我们接着进去看看．

![001](http://wx2.sinaimg.cn/large/74b07056ly1fs9r1e37zuj20ne0bewfv.jpg)

在DefaultListableBeanFactory的getBean方法中我们主要关注下resolveNamedBean方法.

![001](http://wx1.sinaimg.cn/large/74b07056ly1fs9r1endk0j20ok06haaw.jpg)

继续跟进去看看．在resolveNamedBean方法中我们主要关注一下代码即candidateNames.length == 1这个部分，可以看见获取到beanName之后创建NamedBeanHolder实例的第二个参数中调用了父类的getBean(String,Class.Object...)的方法．接着这个方法又调用了doGetBean的方法创建或者获取bean.

![001](http://wx2.sinaimg.cn/large/74b07056ly1fs9r1fbslxj20tr0dy76s.jpg)

我们可以看见由于没有声明scope为prototype，所以可以从spring上下文中获取该名称的实例.而获得的这个实例是在项目启动的时候创建的．所以我们声明在全局变量的时间是不会被改变的，而时间恰好是项目的启动时间．

### # prototype

下面我们看看声明scope为prototype之后spring帮我们做了什么．首先需要说明的是在项目启动的时候spring会扫描所有的注解注册的类并判断是否是singleton如果不是，在没有显示声明Lazy = false的情况下，spring默认是懒加载的，也就是在程序真正使用该类的时候才会创建．

对于prototype来说，前面的步骤和singleton是一样的，不同的是在获取不到sharedInstance的时候进行bean的创建并返回创建的实例．

![001](http://wx2.sinaimg.cn/large/74b07056ly1fs9r1fvpzrj20vk0q1gp9.jpg)

在createBean方法中，使用了类的反射进行实例的创建，包括一些集成的方法和变量的校验等等．

![001](http://wx2.sinaimg.cn/large/74b07056ly1fs9r1gl6ukj20xc0ckdi5.jpg)

所以我们可以看见scope声明prototype之后，每次获取该实例执行方法的时候，spring都会帮我们创建一个新实例，这样，声明的成员变量的时间都会是系统的当前时间．

## # 最后

最后通过这个bug总结下spring scope的用法，以及了解spring创建获取实例的过程．
