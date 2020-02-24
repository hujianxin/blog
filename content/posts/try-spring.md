---
title: 在Spring边缘试探
description: Spring原理初探，关于IOC和AOP
date: 2017-05-03
categories:
  - "开发设计"
tags:
  - "Java"
  - "Spring"
draft: false
---

![spring-by-pivotal.png](https://cnbj2.fds.api.xiaomi.com/v2tool/3780192716361256736?GalaxyAccessKeyId=5411787938902&Expires=202212332102020&Signature=pbzDTCQBrceHtWVeGqKFwGIrJQk=) 
> 本文首发于简书，于2018年11月迁移至本博客
<!--more-->

虽说是Java web，但Spring已经成为了事实标准，Spring原理的深入学习，无论是在工作中，还是在面试中，都尤为重要。

Spring的两个核心概念是IOC（控制反转）和AOP（面向切面编程）。想了解Spring的工作原理，毫无疑问，首先要从这两个概念的Spring实现入手。但是Spring源码浩如烟海，里面掺杂了太多的实现细节，入门可谓极其困难。当我正苦于难以入门时，好友介绍了[tiny-spring](https://github.com/code4craft/tiny-spring)这个开源项目，这个项目用了不到千行的代码，就将Spring的IOC、AOP的核心流程实现完毕，真是居家旅行、吹逼面试之必备呀！

废话少说，我们开始吧！

## 目录结构

在github上clone下项目来之后，我们关注src文件夹，其余的是一些爱好者提的注释PR，恰巧被作者merge了，不必理会。目录结构是这样的：

![tiny-spring.png](https://cnbj2.fds.api.xiaomi.com/v2tool/1512984167763575744?GalaxyAccessKeyId=5411787938902&Expires=202212332102020&Signature=n3S94HNDk2slbdVBUDEtnY2A+xg=) 
1. `aop`包，顾名思义，实现了Spring的AOP功能，可以通过bean的自动AOP切入，文件稍多，暂时先不展开。
2. `bean.factory`包，通过`BeanFactory`、`AbstractBeanFactory`、`AutowireCapableBeanFactory`三个类，实现了`BeanFactory`的核心功能，详情稍后讲解。
3. `bean.io`包定义了资源加载相关的抽象概念，这里的资源包括xml配置文件等。
4. `bean.xml`包中只包含一个类：`XmlBeanDefinitionReader`，主要负责在xml配置文件中读取bean定义。
5. `bean`包其他类，定义了BeanDefinition等核心概念，详情后讲。
6. `context`包定义了`ApplicationContext`的核心概念。
7. `BeanReference`指的是引用类型的Bean，而不是实体类。

## IOC--浮沙筑台之根基

IOC（控制翻转）是一种编程范式，可以在一定程度上解决复杂系统对象耦合度太高的问题，并不是Spring的专利。IOC最常见的方式是DI（依赖注入），可以通过一个容器，将Bean维护起来，方便在其他地方直接使用，而不是重新new。可以说，IOC是Spring最基本的概念，没有IOC就没有Spring。

为什么DI可以起到解耦的作用？

一个软件系统包含了大量的对象，每个对象之间都有依赖关系，在普通的软件编写过程中，对象的定义分布在各个文件之中，对象之间的依赖，只能通过类的构造器传参，方法传参的形式来完成。当工程变大之后，复杂的逻辑会让对象之间的依赖梳理变得异常困难。

在Spring IOC中，一般情况，我们可以在XML文件之中，统一的编写bean的定义，bean与bean之间的依赖关系，从而增加逻辑的清晰度。而且，bean的创建是由Spring来完成的，不需要编程人员关心，编程人员只需要将精力放到业务的逻辑上面，减轻了思维的负担。

在`tiny-spring`里面，整个`beans`和`context`包都是用来实现`IOC`的。

`beans`包实现的核心关注点是`BeanFactory`，`BeanFactory`也叫作Bean容器，顾名思义，是用来盛放、管理bean的。

`context`包实现的核心关注是`ApplicationContext`，`ApplicationContext`也是用来获取Bean的，但是它更高层，它的面向用户是Spring的使用者，而BeanFactory面向的用户更多是Spring开发者。BeanFactory定义了Bean初始化的流程，ApplicationContext定义了从XML读取，到Bean初始化，再到使用的过程。

Bean在哪定义？

刚才有说到，Spring通常通过xml文件，来统一的描述bean，bean与bean的依赖关系。所以说，bean的定义表述，发生在xml配置文件之中。这个XML文件就是我们需要读取的资源文件。

因此，首要任务就是研究与读取XML资源文件相关的类。

`bean.io`中存放的是读取资源文件的抽象概念。其中包含了三个类或者接口：

1. `Resource`接口，这个接口只有一个方法，`InputStream getInputStream() throws IOException;`。实现这个接口的类就是一个抽象的资源，可以获取这个资源的输入流，从而获取其中的内容。
2. `UrlResource`类，这个类实现了`Resource`接口，通过构造器传入一个url地址，代表的是这个url所对应的文件。
3. `ResourceLoader`类，只有一个方法，`public Resource getResource(String location)`。输入url的文件地址（并不是真正的URL格式地址），来获取Resource。

通过分析上面三个类、接口，我们知道，这个包完成了一个任务：通过`ResourceLoader`这个类，获取某一个地址的`Resource`，从而获取这个文件的输入流。因为使用了Resource概念，可以使用网络文件或者本地文件。

Bean如何定义？

1. `BeanDefinition`是Bean定义的核心概念，`BeanDefinition`包含了：bean的对象、bean的类类型、bean的名字，bean的所有属性。这个类对bean的基本信息做好一个包装。
2. `BeanDefinitionReader`接口，只有一个方法：`void loadBeanDefinitions(String location) throws Exception;`，实现这个接口的类，具有将某一个文件中的所有bean定义载入的功能。所以`BeanDefinitionReader`定义了，在哪载入bean定义，至于载入到哪里、如何载入，稍后看具体实现。
3. `AbstractBeanDefinitionReader`抽象类，上面刚说了实现了`BeanDefinitionReader`接口的类，具有将某一个文件中描述的bean定义载入的功能，`AbstractBeanDefinitionReader`就实现了这样一个抽象功能。它的作用就是定义，载入到哪和如何载入的问题。在这个类里面，有两个属性：`Map<String,BeanDefinition> registry;`和`ResourceLoader resourceLoader;`。`registry`是一个注册表，他保存的就是所有的Bean定义，Map结构，key是bean的名字，value就是BeanDefinition。`resourceLoader`描述了如何载入。
4. `XmlBeanDefinitionReader`这是`beans.xml`包里面的唯一一个方法，也是最重要的方法之一。它继承了`AbstractBeanDefinitionReader`，实现了所有方法，解决了bean定义中：在哪载入、如何载入、载入到哪的三个大问题。这个类面向用户的方法有两个，一个是`loadBeanDefinitions`，毫无疑问，这个是必须的。另一个是`getRegistry`用来获取bean注册表，得到所有bean的信息，registry是bean们在内存中实际的家。但是这个`getRegistry`方法并不是面向用户的，而是面向ApplicationContext的。
5. `PropertyValue`和`PropertyValue`代表一种抽象概念，在xml中，bean的属性包括属性名和属性对象，`PropertyValue`就是这么一个实体。
6. `BeanReference`代表的是Bean的属性不是真实对象，而是另一个bean的引用。

Bean的组装全过程?

上面两部分是铺垫，而BeanFactory才是重点对象。`beans.factory`包中有三个类用来定义BeanFactory相关的概念。

1. `BeanFactory`接口，只有一个方法：`Object getBean(String name) throws Exception;`，实现这个接口的类，就具有了得到一个bean的能力。
2. `AbstractBeanFactory`类，较为复杂。详情后讲。
3. `AutowireCapableBeanFactory`继承了`AbstractBeanFactory`，实现了applyPropertyValues方法，通过反射，将bean的所有属性，通过set方法注入进去。

`AbstractBeanFactory`有三大属性：

+ `beanDefinitionMap`，类似于registry，但是他是BeanFactory里面私有的，代表的是这个BeanFactory里面暂时有哪些bean定义。
+ `beanDefinitionNames`代表里面，这个BeanFactory里面有哪些bean（名字）。
+ `beanPostProcessors`，代理处理器，AOP会用到，详情后讲。

`AbstractBeanFactory`实现了几大功能：

+ `getBean`，这是主要功能，可以获取一个Bean对象。
+ `registerBeanDefinition`，面向ApplicationContext，用来将XML配置文件里面的bean定义注册到BeanFactory里面。
+ `preInstantiateSingletons`，面向ApplicationContext，用来在开始的时候，将所有的bean都初始化完毕，避免懒加载。
+ `addBeanPostProcessor`添加代理处理器。
+ `getBeansForType`，在BeanFactory里面，获取某一个类型的所有bean。

经过上面的分析，我们可以知道BeanFactory完成了Bean初始化的整个流程。BeanFactory的工作流程如下：

1. getBean, 在beanDefinitionMap里面得到bean，如果没有的话，先初始化。（为什么会没有，因为ApplicationContext读取xml文件时候，只是给BeanDefinition服了类类型，并没有赋值对象，这个对象还是需要BeanFactory通过反射生成的）。
2. createBeanInstance，通过反射，根据BeanDefinition的类对象新建实体对象->将得到的bean对象赋值给beandefinition，然后将BeanDefinition里面的属性都注入到Bean里面，这就完成了doCreateBean。
3. initializeBean就是调用BeanPostProcessor的postProcessBeforeInitilizztion方法和postProcessAfterIntilizatin方法，获取新的bean，这里会在aop中用到。

好了，到这BeanFactory就讲完了，下面是更重要的ApplicationContext。

ApplicationContext-用户与BeanFactory之间的桥梁

`beans.context`包有三个类、接口，完成了ApplicationContext的基本功能。

1. ApplicationContext接口，没有任何方法，只是继承了BeanFactory接口，暗示ApplicationContext与BeanFactory都是获取Bean的地方。
2. `AbstractApplicationContext`抽象类，首先，它的构造函数接收入参BeanFactory，所以说ApplicationContext内部具有一个BeanFactory。类似于一种装饰器模式，但不是装饰器模式，类似于代理模式，但也不是代理模式。fresh方法分为三个步骤：1.loadBeanDefinitions，这个是一个模板方法，需要子类实现，它的作用就是从某一个地方读取BeanDefinition，然后写入到ApplicationContext自己的BeanFactory里面，这就是ApplicationContext与BeanFactory之间的联系，也就是ApplicationContext还负责了读取定义。2. registerBeanPostProcessors，这个就是在BeanFactory里面找到BeanPostProcessor，然后将他们放到BeanFactory的beanPostProcessors容器里面，方便BeanFactory初始化使用。3. onRefresh初始化一遍所有的bean。
3. `ClassPathXmlApplicationContext`实现了loadBeanDefinitions的方法，将xml文件和BeanFactory结合在一起。

总结-ApplicationContext初始化流程如下图

![1.png](https://cnbj2.fds.api.xiaomi.com/v2tool/-7353628852246507420?GalaxyAccessKeyId=5411787938902&Expires=202212332102020&Signature=ZbdiKffKK58/LGxOLu8nNJEp5IA=) 

总结-ApplicationContext获取bean流程如下图
![2.png](https://cnbj2.fds.api.xiaomi.com/v2tool/3648676362047457744?GalaxyAccessKeyId=5411787938902&Expires=202212332102020&Signature=Imq3TlQ0wAkvcjF60rZoNy6PpKk=) 

## AOP--移花接木之魔法

上一节，讲完了Spring IOC的整个流程，也就是bean从定义获取，到得到bean之间的整个流程。本节，我们接触一下Spring另一个重要概念，AOP。AOP用途十分广泛，其中Spring内部的声明式事务和拦截器都是利用了AOP的强大威力，才得以优雅的实现。

AOP是什么呢，简单来说，它可以让编程人员在不修改对象代码的情况下，为这个对象添加额外的功能或者限制。
很熟悉吧，这就是代理模式！

Java中存在两种代理模式：

一种叫静态代理，就是通过接口继承复用的方式来完成的， 代理类与被代理对象实现相同的接口，然后代理类里面会拥有一个被代理对象，代理类与被代理对象相同的方法，活调用被代理对象的方法，不过中间会加以限制，您翻开任何一本设计模式相关的书，翻到代理模式这一节，讲的就是它了。

另一种叫做动态代理，动态代理就是允许我们在程序运行过程中，为动态生成的对象，动态的生成代理。显然，这比静态代理灵活太多了。
Java默认提供了动态代理的实现方式，但是有限制，它要求被代理对象必须实现某一个接口。为了突破这一限制，为普通类也可以提供代理，CGLib这个库横空出世。

因为AOP涉及的知识较为复杂，所以我先将背景知识介绍一下。

1. Java动态代理，就是Java本身提供的代理实现，要求对象必须实现某一个接口。
2. CGLib库，为Java提供了，为普通类提供代理的功能。
3. aopalliance，aop联盟包，统一类aop编程的一些概念，这个包里没有具体的类实现，而是定义了几个重要的概念接口，具体的aop实现，要遵从这些接口编程，才能达到一定的通用性。
4. aspectj包，实现了，通过一种固定的编程语言，通过这种简单的编程语言，我们可以定位到被代理的类，自动完成代理。

在aopallicance里面，定义了几个核心概念:

1. `Advice`增强，说明这是一个，实现这个接口，说明这个类负责某一种形式的增强。
2. `Joinpoint`连接点，表示切点与目标方法连接处的信息。
3. `MethodInterceptor`继承了Interceptor接口，而Interceptor继承了Advice接口，是一种Advice，但是有一个方法invoke。这个方法需要一个参数MethodInvocation。
4. `MethodInvocation`表示的是连接点的信息以及连接点函数的调用。

结合上面的信息，我们发现，其实MethodInterceptor的invoke方法，调用的就是MethodInvocation的proceed方法，而这个proceed方法呢，应该调用的肯定是Method.invoke方法。所以，这是一种变相调用method.invoke的方式。为什么这样做呢，猜一猜的话，肯定是为了代码的复用，哈哈哈，这是废话。

在Spring中，还定义了几个核心概念：

1. `Pointcut`，切点，可以定位类以及方法
2. `Advisor`，可以获取一个增强。
3. `PointcutAdvisor`，定义了哪些方法，具有什么类型的增强。
4. `MethodMatcher`表示某一个方法是否符合条件
5. `ClassFilter` 定义了某个类是否符合条件。
6. `TargetSource`被代理的对象，包括对象本身，类类型、所有接口
7. `AdvisedSupport`代理相关的元数据，包括被代理的对象，增强等。
8. `ProxyFactory`，代理工厂，可以获得一个代理对象，同时具有AdvisedSupport的所有特性。
9. `Cglib2AopProxy`，使用cglib实现的动态代理类，继承了AbstractAopProxy抽象类，这个类的主要方法就是getProxy，通过什么呢，通过AdvisorSupport。
10. `ReflectiveMethodInvocation`可以获取连接点的信息，代理的对象等。
11. `JdkDynamicAopProxy`，和Cglib2AopProxy类一个作用，通过AdvisorSupport来getProxy，不过是使用Java自带的动态代理实现的。

其中，ProxyFactory是获取一个代理对象的直接工厂，而这个代理对象，可以通过Cglib2AopProxy产生，也可以通过JdkDynamicAopProxy产生。

Spring AOP之所以能够为动态生成的Bean提供代理，得益于PostProcessor接口。我们会议IOC初始化流程中，最后一部，就是得到BeanFactory之中所有继承了PostProcessor接口的bean，调用它们的postProcessBeforeInitilization、postProcessAfterInitilization方法，来代理bean，生成新的bean。

基于这个突破口，我们只需要在xml配置文件中，放入PostProcessor对象，Spring就会自动的用这写对象，来代理真正的对象。

![3.png](https://cnbj2.fds.api.xiaomi.com/v2tool/2687759870624423260?GalaxyAccessKeyId=5411787938902&Expires=202212332102020&Signature=pg1GbFx/f8Gf6BXoZ/ze8fotcu0=) 

在这里，我们的对象是AspectJAwareAdvisorAutoProxyCreator。

在这个对象的方法中，逻辑是这样的，找到xml里面所有切面bean，然后在这些bean里面，找到符合被代理类的切面bean，找到切面bean之后，就可以获得增强，切点等，于是可有构造一个AdvisorSupport，知道了AdvisorSupport，我们就能够通过proxyFactory来获取代理了。

至于如何这个类切面是用来切入代理类的，这个就要交给PointCut来实现了，pointcut有很多实现方式，这里我们用的是aspectj。具体这个类我就不细讲了。

到目前位置，我自己已经将整个AOP的流程搞清楚了，下面通过流程图的形式展示出来：

![4.png](https://cnbj2.fds.api.xiaomi.com/v2tool/-1763378526263523659?GalaxyAccessKeyId=5411787938902&Expires=202212332102020&Signature=FEQVRGpi6w06+CLugw0TVRF0eYo=)

