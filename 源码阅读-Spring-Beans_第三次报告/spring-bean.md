# Sprint-Beans 源码阅读

## 4 Spring Bean中的设计模式

### 4.1 设计模式定义

在这部分开始之前，我们有必要回顾一下，什么是设计模式呢？

总的来说，设计模式（Design Pattern）是一种设计思维的高度抽象，就像算法可以被理解为解决一些特定计算问题的思路的抽象一样，设计模式也可以被理解为一种解决设计问题的思路的抽象。设计模式的概念非常具有哲学意味，很容易联想到柏拉图哲学理论中被频频提及的“理型”概念，所谓理型，即时现实事物在人的理性思维中投射的完美“模板”，如果把现实事物比作姜饼的话，“理型”即是对应的姜饼模具。尽管现实中的姜饼可能形态各异，但它们身上都可以看到姜饼模具的影子，这就是代码复用性（笑）。

具体来说，设计模式是一套被反复使用、多数人知晓的、经过分类编目的、代码设计经验的总结。使用设计模式是为了可重用代码、让代码更容易被他人理解、保证代码可靠性。 毫无疑问，运用设计模式无论对于项目本身，还是对于项目作者，还是对于阅读项目的读者来说，都是多赢的。可以说，设计模式使代码编制真正“工程化”。从某种程度上说，每种模式在现实中都有相应的原理来与之对应，每一个模式描述了一个在我们周围不断重复发生的问题，以及该问题的核心解决方案。正如上文所说的，设计模式是一种设计问题解决方案的高度抽象，如柏拉图所言的现实事物在人的理性思维中呈现的完美“模板”，正因如此，设计模式才能在众多项目中被反复使用，同时取得良好效果。

### 4.2  Spring Bean中体现的设计模式

回顾我们之前的debug过程，我们一路经过了DefaultSingletonBeanRegistry，AbstractApplicationContext等等重要类，它们都蕴含一些典型的设计模式。

#### 4.2.1 单例模式

单例模式可以确保系统中某个类只有一个实例，该类自行实例化并向整个系统提供这个实例的公共访问点，除了该公共访问点，不能通过其他途径访问该实例。单例模式的优点在于：

- 系统中只存在一个共用的实例对象，无需频繁创建和销毁对象，节约了系统资源，提高系统的性能
- 可以严格控制客户怎么样以及何时访问单例对象。

更具体而言，实现单例的方法主要有两种，饿汉式与懒汉式。饿汉式是在类加载时，就将单例初始化完成，保证获取实例的时候，单例是已经存在的了。所以在第一次调用时速度也会更快，因为其资源已经初始化完成。懒汉式会延迟加载，只有在首次调用时才会实例化单例，如果初始化所需要的工作比较多，那么首次访问性能上会有些延迟，不过之后就和饿汉式一样了。

在Spring Bean中，默认的作用域就是singleton单例的。

这样的好处在于对一些特定的对象，省略了重复创建对象花费的时间，减少了系统的开销，因为有些对象我们只需要一个，或者只能存在一个，比如一些资源管理器；同时这一方面节省下了重复创建对象的开销，另一方面也节省了回收内存的开销。具体到代码上，减少new操作的次数就是实实在在的节约开销，优化代码。

spring又是如何创建单例bean的呢？我们可以看到DefaultSingletonBeanRegistry的getSingleton()方法。

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    /** 保存单例Objects的缓存集合ConcurrentHashMap，key：beanName --> value：bean实例 */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
 
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");
        synchronized (this.singletonObjects) {
            //检查缓存中是否有实例，如果缓存中有实例，直接返回
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //省略...
                try {
                    //通过singletonFactory获取单例
                    singletonObject = singletonFactory.getObject();
                    newSingleton = true;
                }
                //省略...
                if (newSingleton) {
                    addSingleton(beanName, singletonObject);
                }
            }
            //返回实例
            return singletonObject;
        }
    }
    
    protected void addSingleton(String beanName, Object singletonObject) {
      synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
      }
    }
}
```

从中可以看到，它的单例池使用了一个ConcurrentHashMap来实现，如果在Map中存在则直接返回，如果不存在则创建，并且put进Map集合中。这一段逻辑外同时使用了同步代码块，保证了操作的原子性，所以是线程安全的。

#### 4.2.2 工厂模式

这一设计模式我们并不陌生，在上文分析Bean的存取时我们就频频提及工厂设计模式这一概念。在面向对象编程中，当需要创建对象实例时，我们往往不假思索地使用 new 操作符构造一个对象实例，但在某些情况下，new 操作符直接生成对象会存在一些问题。

举例来说，对象的创建需要一系列的步骤：可能需要计算或取得对象的初始位置、选择生成哪个子对象实例、或在生成之前必须先生成一些辅助对象。 在这些情况，新对象的建立就是一个 “过程”，而不仅仅是一个操作，就像一部大机器中的一个齿轮传动。这个问题在复杂系统的编程中尤为明显，如何简化这样一个复杂的创建过程呢？一种方法是采取工厂设计模式，使用一个工厂类来创建对象。

工厂模式将目的将创建对象的具体过程屏蔽隔离起来，从而达到更高的灵活性，工厂模式可以分为三类：

- 简单工厂模式（Simple Factory）
- 工厂方法模式（Factory Method）
- 抽象工厂模式（Abstract Factory）

在Spring Bean中，主要采用的是抽象工厂模式这一思路。在Spring中，可以使用工厂方法或者构造器来实例化Bean，也可以通过静态工厂方法或实例工厂方法来实例化Bean。在使用工厂方法或构造器实例化Bean时，Spring容器会调用相应的工厂方法或构造器来创建Bean实例。在使用静态工厂方法或实例工厂方法实例化Bean时，Spring容器会调用相应的工厂方法来创建Bean实例，工厂方法也可以是静态的或实例的。

Spring容器还可以通过使用工厂Bean来实例化其他的Bean。工厂Bean是一种特殊的Bean，它的实例化过程需要调用一个工厂方法。工厂Bean本身并不会被直接注入到应用程序中使用，而是会被用来创建其他Bean的实例。Spring Bean使得程序员可以使用依赖注入的方式将Bean注入到应用程序中，而无需关心Bean的创建过程。

而体现在代码上，让我们回顾BeanFactory的源码：

```java
public interface BeanFactory {

	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	Object getBean(String name, Object... args) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);

}
```

正如前文所分析的，BeanFactory是Spring中用于创建和管理Bean的工厂接口，它体现了抽象工厂设计模式，因为它提供了一组用于创建Bean的方法，而不需要指定具体的类是什么。这就大大简化了Bean的创建过程，程序员可以使用依赖注入的方式将Bean注入到应用程序中，而无需关心Bean的创建过程。

#### 4.2.3 原型模式

在Spring Bean中，我们可以使用`scope="prototype"`来声明一个原型Bean，这体现原型模式的设计方法模式。

原型模式是一种创建型设计模式，它允许将一个对象作为原型，通过复制该原型来创建新的对象。在Spring中，使用`scope="prototype"`声明的Bean就是原型Bean，每次注入或者通过Spring容器获取该Bean时，都会创建一个新的Bean实例。例如：

```java
<bean id="sheep" class="com.java.springtest.prototype.Sheep" scope="prototype"/>
```

使用原型模式有诸多益处，由于每次获取的都是新的对象，因此可以在运行时动态地改变对象的属性，而不会影响其他对象。同时可以避免使用单例模式带来的资源消耗问题。

但原型模式也有几点需要注意，原型Bean的属性可能是可变的，因此在使用原型Bean时要注意同步问题。如果原型Bean依赖了其他的Bean，那么这些依赖的Bean也会被复制。原型模式可能会影响性能，因为每次获取Bean都会创建新的对象。

总之，在Spring Bean中，使用`scope="prototype"`声明的Bean就是原型Bean，可以通过复制该Bean来创建新的对象，这就是在Spring Bean中体现出的原型模式。

#### 4.2.4 模板方法模式

模板方法是基于继承实现的，在抽象父类中声明一个模板方法，并在模板方法中定义算法的执行步骤（即算法骨架）。在模板方法模式中，可以将子类共性的部分放在父类中实现，而特性的部分延迟到子类中实现，只需将特性部分在父类中声明成抽象方法即可，使得子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤，不同的子类可以以不同的方式来实现这些逻辑。

在之前debug的过程中，我们曾遇到过ApplicationContext与AbstractApplicationContext这两个抽象类，后者就是对前者的抽象实现。而AbstractApplicationContext中的getbean()方法要求先获得一个BeanFactory，再调用BeanFactory中的getBean()方法。如下：

```java
@Override
	public <T> T getBean(Class<T> requiredType) throws BeansException {
		assertBeanFactoryActive();
		return getBeanFactory().getBean(requiredType);
	}
```

这里的getBeanFactory()就是抽象方法，要求子类完成具体实现。这就体现了模板方法设计模式，它能够实现代码复用，将不变的行为转移到父类，去除子类中的重复代码。

## 5 一点总结

在本次的源码阅读中，我们主要探索了以下内容：

- Spring Framework的特性历史
- Spring模块的介绍
- Spring核心技术
- BeanFactory存取Bean的流程与实现
- Spring Bean中体现的设计方法模式

由于时间精力以及篇幅所限，Spring Bean的源码阅读不得不告一段落了。这一路下来说长不长，说短不短，但着实让我这个榆木脑袋多多少少体会到一点顶级Coder设计思路的精妙，让我感慨，真正的面向对象设计思维已经近乎哲学的范畴。

当然，作为一个刚刚接触面向对象编程的菜鸟，我对Spring Bean的阅读也只是浅尝辄止。固然，之前的学习中接触到C/Verilog这些较为底层的语言偏多，但之后的学习工作中无可避免的会接触到大型项目的开发与维护，提前接触到这些面向对象的思维于我益处颇多。感谢老师的引导，能让我有意识地去阅读，去品味这些原汁原味的“名著”，这一次的Spring Bean让我受益良多，接受了顶级Coder的教诲，我也更有信心继续进行面向对象编程的学习。这是这次源码阅读的结束，但是也是我面向对象编程学习的开始。