# Sprint-Beans 源码阅读

## 1 写在最前面

在选修这门课之前，笔者的面向对象编程知识几乎为零，或者说，完全没有“面向对象”的概念。由于学校课程体系的安排，在过去的三年中笔者主要接触的是C这类面向过程的语言。可以说，这次源码阅读是一次从零开始的学习过程。在选择题目时，固然笔者对更复杂的系统有着更多好奇，但还是希望能从基础一点的项目入手，希望能够通过阅读原码管窥顶级Coder们的设计思路，借此学习建立“面向对象”的思想。

## 2 Spring Framework简介

### 2.1 Spring Framework特性与历史

在查阅资料时，笔者看到了一个很有意思的说法：“我得出一个公式：Spring = 春天 = Java程序员的春天 = 简化开发。最后的简化开发正是Spring框架带来的最大好处。”

事实确实如此，Spring确实为Java带来了春天。它是于2003 年兴起的一个轻量级的Java 开发框架。由Rod Johnson创建，其前身为Interface21框架，后改为了Spring并且正式发布。Spring是为了解决企业应用开发的复杂性而创建的。它解决的是业务逻辑层和其他各层的松耦合问题，因此它将面向接口的编程思想贯穿整个系统应用。框架的主要优势之一就是其分层架构，分层架构允许使用者选择使用哪一个组件，同时为 J2EE 应用程序开发提供集成的框架。Spring使用基本的JavaBean来完成以前只可能由EJB完成的事情。然而，Spring的用途不仅限于服务器端的开发。从简单性、可测试性和松耦合的角度而言，任何Java应用都可以从Spring中受益。简单来说，Spring是一个分层的JavaSE/EE full-stack(一站式) 轻量级开源框架。Spring 的理念：不去重新发明轮子。其核心是控制反转（IOC）和面向切面（AOP）。

在2002年10月，由Rod Johnson 编著的书名为《Expert One-to-One J2EE Design and Development》一书中，对Java EE 系统框架臃肿、低效、脱离现实的种种现状提出了质疑，并阐述了 J2EE 使用 EJB 开发设计的优点及解决方案，他提出了一个基于普通 Java 类和依赖注入的更简单的解决方案。然后以此书为指导思想，他编写了interface21框架，这是一个力图冲破J2EE传统开发的困境，从实际需求出发，着眼于轻便、灵巧，易于开发、测试和部署的轻量级开发框架。Spring框架即以interface21框架为基础，经过重新设计，并不断丰富其内涵，于2004年3月24日，发布了1.0正式版。同年他又推出了一部堪称经典的力作《Expert one-on-one J2EE Development without EJB》，该书在Java世界掀起了轩然大波，不断改变着Java开发者程序设计和开发的思考方式。在该书中，作者根据自己多年丰富的实践经验，对EJB的各种笨重臃肿的结构进行了逐一的分析和否定，并分别以简洁实用的方式替换之。至此一战功成，Rod Johnson成为一个改变Java世界的大师级人物。值得注意的是，Rod Johnson是悉尼大学的博士，然而他的专业不是计算机，而是音乐学。

Spring框架自从发布以来就发展迅速，现在已经是最受欢迎的企业级 Java 应用程序开发框架，数以百万的来自世界各地的开发人员使用 Spring 框架来创建性能好、易于测试、可重用的代码。从2004发布的第一个Spring版本，到现在已经更新到第五个Spring版本了。

### 2.2 Spring模块

Spring框架包含的功能大约由20个小模块组成。这些模块按组可分为核心容器(Core Container)、数据访问/集成(Data Access/Integration)、Web、面向切面编程(AOP和Aspects)、设备(Instrumentation)、消息(Messaging)和测试(Test)。Spring官方文档中为这些模块画出了形象的示意图：

![1745215-20200715183321528-138974993](.\1745215-20200715183321528-138974993.png)

### 2.3 Spring 核心技术

Spring最核心的部分是控制反转（IOC）和面向切面编程（AOP）。由于本次源码阅读关注的模块是Spring-Beans，不包含AOP技术，故暂时略过AOP，重点介绍IOC。

在Spring的相关文档中，另一个概念：DI（Dependency Injection依赖注入）也被频频提及。其实，控制反转(IOC)和依赖注入(DI)是从不同的角度的描述的同一件事情，就是指通过引入IOC容器，利用依赖注入的方式，实现对象之间的解耦。其中IOC是个更宽泛的概念，而DI是更具象的概念。

具体而言，IOC技术就是，当用户使用对象调用一个方法或者类时，不是由用户主动去创建这个类的对象，而是将控制权交给Spring框架。说复杂点就是资源（组件）不再由使用资源双方进行管理，而是由不使用资源的第三方统一管理，这样带来的好处。第一，实现了资源的集中管理，增加了资源的可配置性和易管理程度。第二，降低了使用资源双方的依赖程度，也就是常说的耦合度。

而DI技术是指，由Spring框架主动创建被调用类的对象，然后把这个对象注入到用户自己的类中，使得用户可以直接使用它。

### 2.4 Spring-Beans模块简介

首先需要厘清的一点是——什么是bean？

bean这一命名不是Spring开创的，Java中bean的使用由来已久。同样是在查找资料的过程中，笔者找到了一段有意思的叙述：

“*想想 Java 的图标是什么？没错，是一杯咖啡。想想咖啡的原料是什么？没错，是咖啡豆（bean）。*”

在Java中，bean是一种特殊的，可重用的，私有的类。在Spring中也是如此。

在Spring中，Spring-Beans模块主要实现两方面功能，一是实现了一个通用的对象工厂，即BeanFactory，通过它，用户可以获取到所需的对象。二是担任全局的上下文，用户把某个对象丢进这个上下文，然后就可以在应用的任何位置获取到这个对象。