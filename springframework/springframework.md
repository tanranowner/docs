# 基本概念

## BeanDefinition

​        BeanDefinition就是bean在spring容器中的描述（配置信息），也就是bean的元数据。bean在容器扫描过程中，首先会生成BeanDefinition。常见定义bean的方式：

- <bean></bean>
- @Bean
- @Component

​        也可以通过代码方式，注册一个BeanDefinition到容器。

```java
// 定义了一个BeanDefinition
AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();

// 当前Bean对象的类型
beanDefinition.setBeanClass(User.class);

// 将BeanDefinition注册到BeanFactory中
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
beanFactory.registerBeanDefinition("user", beanDefinition);

// 获取Bean
System.out.println(beanFactory.getBean("user"));
```

 BeanDefinition中Bean的元数据包含：

- scope：作用域（singleton, prototype, request, session）；
- lazy：是否懒加载；
- dependsOn：依赖；
- primary：主要的装配候选标识；
- factoryBeanName：该bean生成的工厂bean名称；
- initMethodName：该bean的初始化方法名；
- destoryMethodName：该bean的销毁方法名；

### Bean解析BeanDefinition

#### BeanDefinitionReader

AnnotatedBeanDefinitionReader可以将Bean对应的类，解析为BeanDefinition。

```java
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
AnnotatedBeanDefinitionReader annotatedBeanDefinitionReader = new AnnotatedBeanDefinitionReader(beanFactory);

// 将User.class解析为BeanDefinition
annotatedBeanDefinitionReader.register(User.class);

System.out.println(beanFactory.getBean("user"));
```

reader可以解析指定类，并在其构造方法中，注册6个internal后置处理器（BeanFactoryPostProcessor）。

#### ClassPathBeanDefinitionScanner

scanner可以在指定目录进行扫描@Component和@ManagedBean组件。

## BeanFactory

BeanFactory是spring的核心类，其中最重要的实现类为DefaultListableBeanFactory。

![image-20201121113456794](springframework.assets\image-20201121113456794.png)

从上图可以看出，它实现和继承了众多的接口和父类。所以它具备多种能力。

1. AliasRegistry：支持别名功能，一个名字可以对应多个别名；

2. BeanDefinitionRegistry：可以注册、保存、移除、获取某个BeanDefinition；

3. BeanFactory：Bean工厂，可以根据某个bean的名字、或类型、或别名获取某个Bean对象；

4. SingletonBeanRegistry：可以直接注册、获取某个单例Bean；

5. SimpleAliasRegistry：它是一个类，实现了AliasRegistry接口中所定义的功能，支持别名功能；

6. ListableBeanFactory：在BeanFactory的基础上，增加了其他功能，可以获取所有BeanDefinition的beanNames，可以根据某个类型获取对应的beanNames，可以根据某个类型获取{类型：对应的Bean}的映射关系；

7. HierarchicalBeanFactory：在BeanFactory的基础上，添加了获取父BeanFactory的功能；

8. DefaultSingletonBeanRegistry：它是一个类，实现了SingletonBeanRegistry接口，拥有了直接注册、获取某个单例Bean的功能；

9. ConfigurableBeanFactory：在HierarchicalBeanFactory和SingletonBeanRegistry的基础上，添加了设置父BeanFactory、类加载器（表示可以指定某个类加载器进行类的加载）、设置Spring EL表达式解析器（表示该BeanFactory可以解析EL表达式）、设置类型转化服务（表示该BeanFactory可以进行类型转化）、可以添加BeanPostProcessor（表示该BeanFactory支持Bean的后置处理器），可以合并BeanDefinition，可以销毁某个Bean等等功能；

10. FactoryBeanRegistrySupport：支持了FactoryBean的功能；

11. AutowireCapableBeanFactory：是直接继承了BeanFactory，在BeanFactory的基础上，支持在创建Bean的过程中能对Bean进行自动装配；

12. AbstractBeanFactory：实现了ConfigurableBeanFactory接口，继承了FactoryBeanRegistrySupport，这个BeanFactory的功能已经很全面了，但是不能自动装配和获取beanNames；

13. ConfigurableListableBeanFactory：继承了ListableBeanFactory、AutowireCapableBeanFactory、ConfigurableBeanFactory；

14. AbstractAutowireCapableBeanFactory：继承了AbstractBeanFactory，实现了AutowireCapableBeanFactory，拥有了自动装配的功能；

15. DefaultListableBeanFactory：继承了AbstractAutowireCapableBeanFactory，实现了ConfigurableListableBeanFactory接口和BeanDefinitionRegistry接口，所以DefaultListableBeanFactory的功能很强大。

## ApplicationContext

ApplicationContext也是一种BeanFactory。

![image-20201121114831788](springframework.assets\image-20201121114831788.png)

从上图可以看出，它除过实现了了BeanFactory（基础容器能力），还实现了其它接口（增强了基础业务能力和易用性）。

1. HierarchicalBeanFactory：拥有获取父BeanFactory的功能；

2. ListableBeanFactory：拥有获取beanNames的功能；

3. ResourcePatternResolver：资源加载器，可以一次性获取多个资源（文件资源等等）；

4. EnvironmentCapable：可以获取运行时环境（没有设置运行时环境功能）；

5. ApplicationEventPublisher：拥有广播事件的功能（没有添加事件监听器的功能）；

6. MessageSource：拥有国际化功能。

其中最重要的实现类为AnnotationConfigApplicationContext。

![image-20201121182832765](springframework.assets/image-20201121182832765.png)

从上图可以看到，AnnotationConfigApplicationContext在ApplicationContext基础上继承和实现了一些接口及类。

1. ConfigurableApplicationContext：继承了ApplicationContext接口，增加了，添加事件监听器、添加BeanFactoryPostProcessor、设置Environment，获取ConfigurableListableBeanFactory等功能；

2. AbstractApplicationContext：实现了ConfigurableApplicationContext接口；

3. GenericApplicationContext：继承了AbstractApplicationContext，实现了BeanDefinitionRegistry接口，拥有了所有ApplicationContext的功能，并且可以注册BeanDefinition，注意这个类中有一个属性(DefaultListableBeanFactory beanFactory)；

4. AnnotationConfigRegistry：可以单独注册某个为类为BeanDefinition（可以处理该类上的@Configuration注解，已经可以处理@Bean注解），同时可以扫描；

5. AnnotationConfigApplicationContext：继承了GenericApplicationContext，实现了AnnotationConfigRegistry接口，拥有了以上所有的功能。

### 国际化

```java
// 定义一个MessageSource
@Bean
public MessageSource messageSource() {
    ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
    messageSource.setBasename("messages");
    return messageSource;
}

// 使用国际化
annotationConfigApplicationContext.getMessage("login.info", null, new Locale("zh_CN"))
```

### 资源加载

```java
// 加载多个文件
Resource[] resources = context.getResources("classpath:application.yml");
for (Resource resource : resources) {
    System.out.println(resource.contentLength());
}
```

### 环境变量

```java
// 获取JVM所允许的操作系统的环境变量
annotationConfigApplicationContext.getEnvironment().getSystemEnvironment();

// 获取JVM本身的一些属性，包括-D所设置的命令行参数（包含classpath）
annotationConfigApplicationContext.getEnvironment().getSystemProperties();

// 增加配置文件
@PropertySource("classpath:application.yml")

// 还可以直接获取系统的环境变量，命令行参数或properties文件中的属性变量对应的属性值
annotationConfigApplicationContext.getEnvironment().getProperty("abc")

// 获取所有环境变量（systemEnvironment，systemProperties，class path resource [application.yml]）
annotationConfigApplicationContext.getEnvironment().getPropertySources()
```

### 事件发布

```java
annotationConfigApplicationContext.publishEvent("触发一个事件");
```

### 类型转换

### 后置处理器

#### BeanFactoryPostProcessor

Bean工厂的后置处理器

#### BeanPostProcessor

Bean的后置处理器

#### FactoryBean

工厂Bean