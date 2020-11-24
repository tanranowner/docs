[toc]

---



# 启动源码分析

## 入口方法
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        // 传入主配置类和参数
        SpringApplication.run(Application.class, args);
    }
}
```
## 创建SpringApplication对象
```JAVA
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        // 先创建SpringApplication对象，在调用run方法
		return new SpringApplication(primarySources).run(args);
}
```
```JAVA
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	this.resourceLoader = resourceLoader;
	Assert.notNull(primarySources, "PrimarySources must not be null");
	// 保存配置类，可以是多个
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
	// 判断应用类型（根据加载类的情况），servlet，webflux，none
	this.webApplicationType = WebApplicationType.deduceFromClasspath();
	// 搜索所有的META-INF/spring.factories中的ApplicationContextInitializer的实现类，并保存
	setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
	// 搜索所有的META-INF/spring.factories中的ApplicationListener的实现类，并保存
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	// 推断主配置类（通过主动抛出异常，获取异常堆栈中是否含有main方法，如果有，择该类为主配置类）
	this.mainApplicationClass = deduceMainApplicationClass();
}
```
## 调用run方法
```JAVA
public ConfigurableApplicationContext run(String... args) {
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
	// 和awt相关
	configureHeadlessProperty();
	// 获取SpringApplicationRunListener的实现类
	SpringApplicationRunListeners listeners = getRunListeners(args);
	// 调用所有的org.springframework.boot.SpringApplicationRunListener#starting
	listeners.starting();
	try {
	    // 封装所有的命令行参数
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
		// 准备环境参数（获取，如果无，则创建）
		// 回调org.springframework.boot.SpringApplicationRunListener#environmentPrepared
		ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
		configureIgnoreBeanInfo(environment);
		// 打印banner
		Banner printedBanner = printBanner(environment);
		// 创建IOC容器（根据applicationType）
		context = createApplicationContext();
		// 异常处理报告
		exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
				new Class[] { ConfigurableApplicationContext.class }, context);
		// 准备context环境，关联IOC环境
		// 回调initializers#initialize方法（即绑定context）
		// 回调listener.contextPrepared（环境配置准备完毕）
		// 回调listener.contextLoaded（context加载beanDefinition完毕）
		prepareContext(context, environment, listeners, applicationArguments, printedBanner);
		// 刷新容器（初始化容器refresh）
		refreshContext(context);
		afterRefresh(context, applicationArguments);
		stopWatch.stop();
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
		}
		// 回调listener.started（容器已经启动）
		listeners.started(context);
		// 从容器获取ApplicationRunner和CommandLineRunner，并回调runner方法
		callRunners(context, applicationArguments);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}

	try {
	// 回调SpringApplicationRunListener#running
		listeners.running(context);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	return context;
}
```


## 事件监听
### 事件类型
- ApplicationContextInitializer(spring.factories)
- ApplicationListener(spring.factories)
- SpringApplicationRunListener(spring.factories)
- ApplicationRunner(IOC)
- CommandLineRunner(IOC)

### 事件触发流程

事件类型 | 开始run方法 | 准备环境参数后 | IOC注册组件后 | 加载beanDefinition后 | refresh后 |
---|---|---|---|---|---
ApplicationContextInitializer |  |  | 1.initialize |  |  |
ApplicationListener |  |  |  |  |  |
SpringApplicationRunListener | starting | environmentPrepared | 2.contextPrepared | contextLoaded | 1.started;4.running |
ApplicationRunner |  |  |  |  | 2.run |
CommandLineRunner |  |  |  |  | 3.run |



# 自动配置原理

## 自动配置原理

springboot中开始自动配置使用注解**@SpringBootApplication**，该注解是个组合注解。

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication{
    ……
}
```

其中**@EnableAutoConfiguration**是用来开始自动配置的注解。

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ……
}
```

 该注解中的**@Import**，通过导入**AutoConfigurationImportSelector**类，间接来实现扫描META-INF/spring.factories中的EnableAutoConfiguration配置类的自动加载。

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
……
```

 该注解中的**@AutoConfigurationPackage**注解，通过导入**AutoConfigurationPackages.Registrar**类，来间接完成当前注解类所在包路径下component扫描。

```java
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
}

static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        register(registry, new PackageImport(metadata).getPackageName());
    }

    @Override
    public Set<Object> determineImports(AnnotationMetadata metadata) {
        return Collections.singleton(new PackageImport(metadata));
    }
}
```

## 其它自动配置的方法

除过上面spring-boot中主流的开启自动配置的方法，也可以采用其它的方式。

### 自定义注解方式

可以自定义注解**@EnableFun**类似注解，配合导入ImportSelector实现类的方式，将自定义的配置类导入到spring-boot中。

```java
// 1. 自定义配置属性
@ConfigurationProperties(prefix = "funProperties")
public class FunProperties {
    private String name;
    private String age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }
}
// 2. 自定义自动配置类，并导入配置属性类
@Configuration
@EnableConfigurationProperties(FunProperties.class)
public class FunAutoConfig {
    @Autowired
    private FunProperties Properties;

    @Bean
    public FunClient getFunClient()
    {
        FunClient funClient = new FunClient();
        funClient.setFunProperties(Properties);
        return service;
    }
}
// 3. 定义工具类
public class FunClient {
    FunProperties Properties;

    public void setFunProperties(FunProperties Properties) {
        this.Properties = Properties;
    }
    public void doSomeThing()
    {
        // todo:
    }
}
// 4. 自定义注解， 通过@Import间接导入自动配置类
@Import(FunImportSelector.class)
public @interface EnableFun {
}
// 5. 导入自动配置类
public class FunImportSelector implements ImportSelector {
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{FunAutoConfig.class.getName()};
    }
}
// 6. 在其它程序中开启
@EnableFun
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
// 7. 在业务类中注入并使用
public class Service {
    @Autowired
    private FunClient client;
    public void do() {
        client.doSomeThing();
    }
}

```



### 条件配置方式

通过配置spring.factories方式，会导致默认配置并注入该组件，这显然在有些场景下是不适合的。可以在通过判断配置属性值的方式，判断是否注入（enable为true才注入，否则不注入）。

```java
@ConditionalOnProperty(prefix = "funProperties", value = "enable")
public class FunAutoConfig  
// 或者
@ConditionalOnExpression("${funProperties.enabled}==1&&${funProperties.enabled:true}")
public class FunAutoConfig  
```

在application.yml配置文件中加入配置即可开启。

```yaml
funProperties:
	enable: true
```

## 配置自动补全

spring-boot官方提供的自动配置，可以在IDE中能够自动补全，这是因为这些组件的配置属性在META-INF下面的文件进行了配置说明。配置文件有additional-spring-configuration-metadata.json和spring-configuration-metadata.json。

有时候需要导入如下依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
</dependency>
```

# 事件原理

## 事件机制

springboot中的事件使用的是springframework中的事件机制，事件机制中，包含无个基本元素。

- 事件：指定的动作或者变化；
- 监听器：特定类型事件的处理者；

- 事件监听器注册表：保存事件的监听者；

- 事件广播器：负责（通过事件监听器注册表）将事件源产生的事件对象通知到事件监听器；

- 事件源：事件的产生者；

  ![springboot-5.1.x源码分析](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/springboot-5.1.x%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.jpg)

事件源发布事件后，事件广播器会根据事件监听器注册表的信息，将事件通知到各自的监听器，最终由监听器完成事件的处理。

## 事件的实现

定义事件

```java
// 集成了ApplicationContextEvent
public class EmailEvent extends ApplicationContextEvent {

	private String sendTo;
	private String subject;
	private String content;

	public EmailEvent(ApplicationContext source, String sendTo, String subject, String content) {
		super(source);

		this.sendTo = sendTo;
		this.subject =subject;
		this.content = content;
	}
}
```

定义事件监听者（事件的处理者）

```java
// 实现了ApplicationListener的泛型接口，对应的泛型为监听类型EmailEvent
@Component
public class SendEmailListener implements ApplicationListener<EmailEvent> {
	@Override
	public void onApplicationEvent(EmailEvent event) {
		System.out.println(event);
        // todo: doAnyThing
	}
}
```

定义事件源触发事件

```java
// 通过事件源触发事件（通过ApplicationContext的事件发布能力）
@Component
public class SendEmail implements ApplicationContextAware {
	private ApplicationContext context;
	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.context = applicationContext;
	}

	public void send(String sendTo, String subject, String content)
	{
		context.publishEvent(new EmailEvent(this.context, sendTo, subject, content));
	}
}
```

测试

```java
public class App {
	public static void main(String[] args) {
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
		SendEmail sendEmail = context.getBean(SendEmail.class);
		sendEmail.send("***@gmail.com", "hello", "hello world");
	}
}
```

结果

```json
EmailEvent{sendTo='***@gmail.com', subject='hello', content='hello world'}
```

## 源码分析

spring在refresh过程中，会初始化事件广播器，注册事件监听器。

```java
public void refresh() throws BeansException, IllegalStateException {
	// 初始化事件广播器
    // Initialize event multicaster for this context.
    initApplicationEventMulticaster();

    // Initialize other special beans in specific context subclasses.
    onRefresh();
	// 扫描并注册容器中的事件监听器
    // Check for listener beans and register them.
    registerListeners();

    // Instantiate all remaining (non-lazy-init) singletons.
    finishBeanFactoryInitialization(beanFactory);

    // Last step: publish corresponding event.
    finishRefresh();
	}
```

注册事件监听器

```java
protected void registerListeners() {
    // 首先注册非bean中的监听器（如@EventListener注解的方法，通过代理对象实现（在bean初始化之后））
    // Register statically specified listeners first.
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }
	// 再从bean中获取ApplicationListener注册（外部自定义的监听器）
    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }

    // 如果是早期事件，则直接广播通知监听者
    // Publish early application events now that we finally have a multicaster...
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (earlyEventsToProcess != null) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```

发布事件

```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
    Assert.notNull(event, "Event must not be null");

    // Decorate event as an ApplicationEvent if necessary
    ApplicationEvent applicationEvent;
    if (event instanceof ApplicationEvent) {
        applicationEvent = (ApplicationEvent) event;
    }
    else {
        applicationEvent = new PayloadApplicationEvent<>(this, event);
        if (eventType == null) {
            eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
        }
    }

    // 立刻广播
    // Multicast right now if possible - or lazily once the multicaster is initialized
    if (this.earlyApplicationEvents != null) {
        this.earlyApplicationEvents.add(applicationEvent);
    }
    else {
        getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
    }

    // Publish event via parent context as well...
    if (this.parent != null) {
        if (this.parent instanceof AbstractApplicationContext) {
            ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
        }
        else {
            this.parent.publishEvent(event);
        }
    }
	}
```

事件调用

```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) 
{
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    Executor executor = getTaskExecutor();
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        // 如果线程池为空（默认），通过线程池方式并发调用
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
        }
        // 在本线程调用
        else {
            invokeListener(listener, event);
        }
    }
}
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    ErrorHandler errorHandler = getErrorHandler();
    // 如果有广播器的ErrorHandler，则把捕获的异常交由它来处理
    if (errorHandler != null) {
        try {
            doInvokeListener(listener, event);
        }
        catch (Throwable err) {
            errorHandler.handleError(err);
        }
    }
    // 如果没有定义ErrorHandler，直接调用，可能会出现异常（可以定义org.springframework.context.event.SimpleApplicationEventMulticaster#setErrorHandler，，也可以在监听器方法onApplicationEvent中自行处理异常）
    else {
        doInvokeListener(listener, event);
    }
}
```

# 嵌入式Servlet容器

## 调用时机

在springboot程序启动过程中（run方法），会调用到refresh方法，该方法会调用onRefresh防范，在onRefresh中会实例化Servlet容器，并启动。

```java
public void refresh() throws BeansException, IllegalStateException {
    // Allows post-processing of the bean factory in context subclasses.
    postProcessBeanFactory(beanFactory);

    // Invoke factory processors registered as beans in the context.
    invokeBeanFactoryPostProcessors(beanFactory);

    // Register bean processors that intercept bean creation.
    registerBeanPostProcessors(beanFactory);

    // Initialize message source for this context.
    initMessageSource();

    // Initialize event multicaster for this context.
    initApplicationEventMulticaster();

    // 其中包含初始化Servlet容器
    // Initialize other special beans in specific context subclasses.
    onRefresh();

    // Check for listener beans and register them.
    registerListeners();

    // Instantiate all remaining (non-lazy-init) singletons.
    finishBeanFactoryInitialization(beanFactory);

    // 其中包含启动Servlet容器
    // Last step: publish corresponding event.
    finishRefresh();
}
// org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#onRefresh
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}
// 开始启动Servlet容器
@Override
protected void finishRefresh() {
    super.finishRefresh();
    WebServer webServer = startWebServer();
    if (webServer != null) {
        publishEvent(new ServletWebServerInitializedEvent(webServer, this));
    }
}
```

## 自动装配容器

springboot会根据jar依赖情况，自动装配Servlet容器。

1. spring.factories

   初始配置文件中spring-boot-autoconfigure\src\main\resources\META-INF\spring.factories，已经配置了自动加载类，**@SpringBootApplication**先根据自动化配置加载配置类**ServletWebServerFactoryAutoConfiguration**

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
```

2. **ServletWebServerFactoryAutoConfiguration**

   该类会根据类依赖和加载情况，选择性的创建容器工厂

```java
// org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
@Configuration
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
	// Servlet容器工厂的定制化器（会将serverProperties（server）属性赋值给Servlet容器工厂）
	@Bean
	public ServletWebServerFactoryCustomizer servletWebServerFactoryCustomizer(ServerProperties serverProperties) {
		return new ServletWebServerFactoryCustomizer(serverProperties);
	}
	// tomcat容器工厂的定制化器（会将serverProperties（server）中的tomcat属性赋值给Servlet容器工厂）
	@Bean
	@ConditionalOnClass(name = "org.apache.catalina.startup.Tomcat")
	public TomcatServletWebServerFactoryCustomizer tomcatServletWebServerFactoryCustomizer(
			ServerProperties serverProperties) {
		return new TomcatServletWebServerFactoryCustomizer(serverProperties);
	}

	public static class BeanPostProcessorsRegistrar implements ImportBeanDefinitionRegistrar {
		@Override
		public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
				BeanDefinitionRegistry registry) {
			registerSyntheticBeanIfMissing(registry, "webServerFactoryCustomizerBeanPostProcessor",
					WebServerFactoryCustomizerBeanPostProcessor.class);
			registerSyntheticBeanIfMissing(registry, "errorPageRegistrarBeanPostProcessor",
					ErrorPageRegistrarBeanPostProcessor.class);
		}
	}
}

public class WebServerFactoryCustomizerBeanPostProcessor implements BeanPostProcessor {
    // 所有注入到IOC中的WebServerFactoryCustomizer.class
	private List<WebServerFactoryCustomizer<?>> customizers;

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (bean instanceof WebServerFactory) {
			postProcessBeforeInitialization((WebServerFactory) bean);
		}
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
	// 遍历所有的WebServerFactoryCustomizer.class，并应用customize方法
	@SuppressWarnings("unchecked")
	private void postProcessBeforeInitialization(WebServerFactory webServerFactory) {
		LambdaSafe.callbacks(WebServerFactoryCustomizer.class, getCustomizers(), webServerFactory)
				.withLogger(WebServerFactoryCustomizerBeanPostProcessor.class)
				.invoke((customizer) -> customizer.customize(webServerFactory));
	}

	private Collection<WebServerFactoryCustomizer<?>> getCustomizers() {
		if (this.customizers == null) {
			// Look up does not include the parent context
			this.customizers = new ArrayList<>(getWebServerFactoryCustomizerBeans());
			this.customizers.sort(AnnotationAwareOrderComparator.INSTANCE);
			this.customizers = Collections.unmodifiableList(this.customizers);
		}
		return this.customizers;
	}

	@SuppressWarnings({ "unchecked", "rawtypes" })
	private Collection<WebServerFactoryCustomizer<?>> getWebServerFactoryCustomizerBeans() {
		return (Collection) this.beanFactory.getBeansOfType(WebServerFactoryCustomizer.class, false, false).values();
	}
}
```

通过以上源码可以看到，如果需要对Servlet容器进行扩展，则需要增加实现**WebServerFactoryCustomizer**定制化器。

```java
// Servlet容器自定义器的扩展，并放入到IOC容器
@Component
public class MyTomcatWebServerCustomizer implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
    // 通过customize可以添加各种自定义器（）
    public void customize(TomcatServletWebServerFactory factory) {
        // 通过添加TomcatConnectorCustomizer自定义器，来扩展Servlet工厂
        factory.addConnectorCustomizers(new TomcatConnectorCustomizer()
        {
            public void customize(Connector connector) {
                connector.setPort(9999);
            }
        });
		// 通过添加TomcatContextCustomizer自定义器，来扩展Servlet工厂
        factory.addContextCustomizers(new TomcatContextCustomizer() {
            public void customize(Context context) {
                // todo: doAnyThing
            }
        });
    }
}
// org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory
// 上面定义的各种自定义器，最终都可以通过获取Servlet容器的实例的方法应用生效
@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    configureEngine(tomcat.getEngine());
    // 应用全部TomcatConnectorCustomizer
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    // 应用全部TomcatContextCustomizer
    prepareContext(tomcat.getHost(), initializers);
    return getTomcatWebServer(tomcat);
}
```

# 注册组件

## Servlet注册

```java
// 将自定义的MyServlet注册到容器中
@Bean
public ServletRegistrationBean myServlet()
{
    return new ServletRegistrationBean<MyServlet>(new MyServlet(), "/hello", "hello-servlet");
}
```

## Filter注册

```java
// 将自定义的的MyFilter注册到容器
@Bean
public FilterRegistrationBean myFilter()
{
    FilterRegistrationBean<MyFilter> filterRegistrationBean = new FilterRegistrationBean<MyFilter>();
    filterRegistrationBean.setFilter(new MyFilter());
    filterRegistrationBean.addUrlPatterns("/hello", "hello-servlet");

    return filterRegistrationBean;
}
```

## Listener添加

```java
// 自定义Listener
public class MyListener implements ServletContextListener {
    public void contextDestroyed(ServletContextEvent sce) {
        // todo:
    }

    public void contextInitialized(ServletContextEvent sce) {
        // todo:
    }
}
// 将自定义的Listener添加到容器
@Bean
public ServletListenerRegistrationBean myListener()
{
    return new ServletListenerRegistrationBean<MyListener>(new MyListener());
}
// 通过上面方式注册的Listener只支持一下类型
static {
    Set<Class<?>> types = new HashSet<>();
    types.add(ServletContextAttributeListener.class);
    types.add(ServletRequestListener.class);
    types.add(ServletRequestAttributeListener.class);
    types.add(HttpSessionAttributeListener.class);
    types.add(HttpSessionListener.class);
    types.add(ServletContextListener.class);
    SUPPORTED_TYPES = Collections.unmodifiableSet(types);
}
```

# Web开发

## 自动配置

在spring.factories配置中，springboot已经添加了servlet的自动配置。

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
```

根据配置，springboot在**@EnableAutoConfiguration**注解中，会自动加载该配置类**DispatcherServletAutoConfiguration**，并初始化springmvc相关的配置。

```java
// 在自动配置完毕Servlet容器之后
@AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
public class DispatcherServletAutoConfiguration {
	/*
	 * The bean name for a DispatcherServlet that will be mapped to the root URL "/"
	 */
	public static final String DEFAULT_DISPATCHER_SERVLET_BEAN_NAME = "dispatcherServlet";

	/*
	 * The bean name for a ServletRegistrationBean for the DispatcherServlet "/"
	 */
	public static final String DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME = "dispatcherServletRegistration";

	@Configuration
	@Conditional(DefaultDispatcherServletCondition.class)
	@ConditionalOnClass(ServletRegistration.class)
    // 导入spring.http和spring.mvc的相关配置
	@EnableConfigurationProperties({ HttpProperties.class, WebMvcProperties.class })
	protected static class DispatcherServletConfiguration {
		private final HttpProperties httpProperties;
		private final WebMvcProperties webMvcProperties;
		// 只有一个构造函数的情况下，spring会自动装配注入
		public DispatcherServletConfiguration(HttpProperties httpProperties, WebMvcProperties webMvcProperties) {
			this.httpProperties = httpProperties;
			this.webMvcProperties = webMvcProperties;
		}
		// 加入DispatcherServlet到IOC容器（配置是否支持trace和options请求）
		@Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServlet dispatcherServlet() {
			DispatcherServlet dispatcherServlet = new DispatcherServlet();
dispatcherServlet.setDispatchOptionsRequest(this.webMvcProperties.isDispatchOptionsRequest());	dispatcherServlet.setDispatchTraceRequest(this.webMvcProperties.isDispatchTraceRequest());
			dispatcherServlet	.setThrowExceptionIfNoHandlerFound(this.webMvcProperties.isThrowExceptionIfNoHandlerFound());			dispatcherServlet.setEnableLoggingRequestDetails(this.httpProperties.isLogRequestDetails());
			return dispatcherServlet;
		}
		// 加入文件处理的MultiPartResolver到IOC容器
		@Bean
		@ConditionalOnBean(MultipartResolver.class)
		@ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
		public MultipartResolver multipartResolver(MultipartResolver resolver) {
			// Detect if the user has created a MultipartResolver but named it incorrectly
			return resolver;
		}

	}

	@Configuration
	@Conditional(DispatcherServletRegistrationCondition.class)
	@ConditionalOnClass(ServletRegistration.class)
	@EnableConfigurationProperties(WebMvcProperties.class)
	@Import(DispatcherServletConfiguration.class)
	protected static class DispatcherServletRegistrationConfiguration {
		private final WebMvcProperties webMvcProperties;
		private final MultipartConfigElement multipartConfig;
		public DispatcherServletRegistrationConfiguration(WebMvcProperties webMvcProperties,
				ObjectProvider<MultipartConfigElement> multipartConfigProvider) {
			this.webMvcProperties = webMvcProperties;
			this.multipartConfig = multipartConfigProvider.getIfAvailable();
		}
		// 构造DispatcherServlet的注册Bean
		@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
		@ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet) {
			DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
					this.webMvcProperties.getServlet().getPath());
			registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
			registration.setLoadOnStartup(this.webMvcProperties.getServlet().getLoadOnStartup());
			if (this.multipartConfig != null) {
				registration.setMultipartConfig(this.multipartConfig);
			}
			return registration;
		}

	}
}
```

## 静态资源

springmvc的自动配置是在spring.factories中。

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```



```java
@Configuration
public class WebMvcAutoConfiguration {
	public static final String DEFAULT_PREFIX = "";
	public static final String DEFAULT_SUFFIX = "";
	private static final String[] SERVLET_LOCATIONS = {"/"};
	// 如果配置中支持hiddenmethod（支持除过GET,POST之外的Method），如果支持或者没有配置，则加入到IOC
	@Bean
	@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
	@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled", matchIfMissing = true)
	public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
		return new OrderedHiddenHttpMethodFilter();
	}

	@Bean
	@ConditionalOnMissingBean(FormContentFilter.class)
	@ConditionalOnProperty(prefix = "spring.mvc.formcontent.filter", name = "enabled", matchIfMissing = true)
	public OrderedFormContentFilter formContentFilter() {
		return new OrderedFormContentFilter();
	}

	static String[] getResourceLocations(String[] staticLocations) {
		String[] locations = new String[staticLocations.length + SERVLET_LOCATIONS.length];
		System.arraycopy(staticLocations, 0, locations, 0, staticLocations.length);
		System.arraycopy(SERVLET_LOCATIONS, 0, locations, staticLocations.length, SERVLET_LOCATIONS.length);
		return locations;
	}
	// 导入资源属性配置文件
	@Configuration
	@Import(EnableWebMvcConfiguration.class)
	@EnableConfigurationProperties({WebMvcProperties.class, ResourceProperties.class})
	@Order(0)
	public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {
		private static final Log logger = LogFactory.getLog(WebMvcConfigurer.class);
		private final ResourceProperties resourceProperties;
		private final WebMvcProperties mvcProperties;
		private final ListableBeanFactory beanFactory;
		private final ObjectProvider<HttpMessageConverters> messageConvertersProvider;
		final ResourceHandlerRegistrationCustomizer resourceHandlerRegistrationCustomizer;

		public WebMvcAutoConfigurationAdapter(ResourceProperties resourceProperties, WebMvcProperties mvcProperties, ListableBeanFactory beanFactory, ObjectProvider<HttpMessageConverters> messageConvertersProvider,
											  ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider) {
			this.resourceProperties = resourceProperties;
			this.mvcProperties = mvcProperties;
			this.beanFactory = beanFactory;
			this.messageConvertersProvider = messageConvertersProvider;
			this.resourceHandlerRegistrationCustomizer = resourceHandlerRegistrationCustomizerProvider.getIfAvailable();
		}
		
		@Override
		public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
			this.messageConvertersProvider
					.ifAvailable((customConverters) -> converters.addAll(customConverters.getConverters()));
		}

		@Override
		public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
			if (this.beanFactory.containsBean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)) {
				Object taskExecutor = this.beanFactory
						.getBean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME);
				if (taskExecutor instanceof AsyncTaskExecutor) {
					configurer.setTaskExecutor(((AsyncTaskExecutor) taskExecutor));
				}
			}
			Duration timeout = this.mvcProperties.getAsync().getRequestTimeout();
			if (timeout != null) {
				configurer.setDefaultTimeout(timeout.toMillis());
			}
		}

		@Override
		public void configurePathMatch(PathMatchConfigurer configurer) {
configurer.setUseSuffixPatternMatch(this.mvcProperties.getPathmatch().isUseSuffixPattern());
			configurer.setUseRegisteredSuffixPatternMatch(
			this.mvcProperties.getPathmatch().isUseRegisteredSuffixPattern());
		}

		@Override
		public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
			WebMvcProperties.Contentnegotiation contentnegotiation = this.mvcProperties.getContentnegotiation();
			configurer.favorPathExtension(contentnegotiation.isFavorPathExtension());
			configurer.favorParameter(contentnegotiation.isFavorParameter());
			if (contentnegotiation.getParameterName() != null) {
				configurer.parameterName(contentnegotiation.getParameterName());
			}
			Map<String, MediaType> mediaTypes = this.mvcProperties.getContentnegotiation().getMediaTypes();
			mediaTypes.forEach(configurer::mediaType);
		}
		// 加入视图解析器
		@Bean
		@ConditionalOnMissingBean
		public InternalResourceViewResolver defaultViewResolver() {
			InternalResourceViewResolver resolver = new InternalResourceViewResolver();
			resolver.setPrefix(this.mvcProperties.getView().getPrefix());
			resolver.setSuffix(this.mvcProperties.getView().getSuffix());
			return resolver;
		}

		@Bean
		@ConditionalOnBean(View.class)
		@ConditionalOnMissingBean
		public BeanNameViewResolver beanNameViewResolver() {
			BeanNameViewResolver resolver = new BeanNameViewResolver();
			resolver.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
			return resolver;
		}

		@Bean
		@ConditionalOnBean(ViewResolver.class)
		@ConditionalOnMissingBean(name = "viewResolver", value = ContentNegotiatingViewResolver.class)
		public ContentNegotiatingViewResolver viewResolver(BeanFactory beanFactory) {
			ContentNegotiatingViewResolver resolver = new ContentNegotiatingViewResolver();
			resolver.setContentNegotiationManager(beanFactory.getBean(ContentNegotiationManager.class));
			// ContentNegotiatingViewResolver uses all the other view resolvers to locate
			// a view so it should have a high precedence
			resolver.setOrder(Ordered.HIGHEST_PRECEDENCE);
			return resolver;
		}

		@Bean
		@ConditionalOnMissingBean
		@ConditionalOnProperty(prefix = "spring.mvc", name = "locale")
		public LocaleResolver localeResolver() {
			if (this.mvcProperties.getLocaleResolver() == WebMvcProperties.LocaleResolver.FIXED) {
				return new FixedLocaleResolver(this.mvcProperties.getLocale());
			}
			AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
			localeResolver.setDefaultLocale(this.mvcProperties.getLocale());
			return localeResolver;
		}

		@Override
		public MessageCodesResolver getMessageCodesResolver() {
			if (this.mvcProperties.getMessageCodesResolverFormat() != null) {
				DefaultMessageCodesResolver resolver = new DefaultMessageCodesResolver();
resolver.setMessageCodeFormatter(this.mvcProperties.getMessageCodesResolverFormat());
				return resolver;
			}
			return null;
		}

		@Override
		public void addFormatters(FormatterRegistry registry) {
			for (Converter<?, ?> converter : getBeansOfType(Converter.class)) {
				registry.addConverter(converter);
			}
			for (GenericConverter converter : getBeansOfType(GenericConverter.class)) {
				registry.addConverter(converter);
			}
			for (Formatter<?> formatter : getBeansOfType(Formatter.class)) {
				registry.addFormatter(formatter);
			}
		}

		private <T> Collection<T> getBeansOfType(Class<T> type) {
			return this.beanFactory.getBeansOfType(type).values();
		}
		// 添加资源映射
		@Override
		public void addResourceHandlers(ResourceHandlerRegistry registry) {
			if (!this.resourceProperties.isAddMappings()) {
				logger.debug("Default resource handling disabled");
				return;
			}
            // 设置静态资源的缓存时间
			Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
			CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
			if (!registry.hasMappingForPattern("/webjars/**")) {
				customizeResourceHandlerRegistration(registry.addResourceHandler("/webjars/**")
						.addResourceLocations("classpath:/META-INF/resources/webjars/")
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
			String staticPathPattern = this.mvcProperties.getStaticPathPattern();
			if (!registry.hasMappingForPattern(staticPathPattern)) {
				customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
						.addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
		}

	}
}
```

资源的属性配置，设置资源有关的配置（静态资源，缓存时间等）。

```java
@ConfigurationProperties(prefix = "spring.resources", ignoreUnknownFields = false)
public class ResourceProperties {

	private static final String[] CLASSPATH_RESOURCE_LOCATIONS = {"classpath:/META-INF/resources/",
			"classpath:/resources/", "classpath:/static/", "classpath:/public/"};
	/**
	 * Locations of static resources. Defaults to classpath:[/META-INF/resources/,
	 * /resources/, /static/, /public/].
	 */
	private String[] staticLocations = CLASSPATH_RESOURCE_LOCATIONS;
}
```

通过以上代码可以看到静态资源的检索路径。

```properties
"/**" 静态资源映射到下面路径 # 来自spring.mvc.staticPathPattern
"classpath:/META-INF/resources/",
"classpath:/resources/", 
"classpath:/static/", 
"classpath:/public/"

"/webjars/**" 静态资源映射到下面路径
"classpath:/META-INF/resources/webjars/"
```

添加webjars依赖，会以jar包形式将静态资源打包提供。

```xml
<!--添加webjars的依赖-->
<dependency>
    <groupId>org.webjars.bower</groupId>
    <artifactId>jquery</artifactId>
    <version>3.3.1</version>
</dependency>
```

- 启动后测试webjars静态资源映射成功。

  ![image-20191207191433181](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/image-20191207191433181.png)

![image-20191207184228754](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/image-20191207184228754.png)

- 测试其它静态资源映射成功。

  ![image-20191207190629918](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/image-20191207190629918.png)

  ![image-20191207190842950](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/image-20191207190842950.png)

  

## Thymeleaf模板

### 引入依赖

thymeleaf模板引擎是springboot推荐的，首先，需要加入pom依赖。

```xml
<!--默认不需要-->
<properties>
    <thymeleaf.version>3.0.11.RELEASE</thymeleaf.version>
    <!--布局支持程序，如果thymeleaf为3，则layout为2以上-->
    <thymeleaf-layout-dialect.version>2.3.0</thymeleaf-layout-dialect.version>
</properties>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

### 自动配置

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration
```

```java
// org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration
@Configuration
// 加载配置
@EnableConfigurationProperties(ThymeleafProperties.class)
@ConditionalOnClass({TemplateMode.class, SpringTemplateEngine.class})
@AutoConfigureAfter({WebMvcAutoConfiguration.class, WebFluxAutoConfiguration.class})
public class ThymeleafAutoConfiguration {
    // thymeleaf的模板解析器
    @Bean
    public SpringResourceTemplateResolver defaultTemplateResolver() {
        SpringResourceTemplateResolver resolver = new SpringResourceTemplateResolver();
        resolver.setApplicationContext(this.applicationContext);
        resolver.setPrefix(this.properties.getPrefix());
        resolver.setSuffix(this.properties.getSuffix());
        resolver.setTemplateMode(this.properties.getMode());
        if (this.properties.getEncoding() != null) {
            resolver.setCharacterEncoding(this.properties.getEncoding().name());
        }
        resolver.setCacheable(this.properties.isCache());
        Integer order = this.properties.getTemplateResolverOrder();
        if (order != null) {
            resolver.setOrder(order);
        }
        resolver.setCheckExistence(this.properties.isCheckTemplate());
        return resolver;
    }
}
// thymeleaf配置属性
@ConfigurationProperties(prefix = "spring.thymeleaf")
public class ThymeleafProperties {
	private static final Charset DEFAULT_ENCODING = StandardCharsets.UTF_8;
	public static final String DEFAULT_PREFIX = "classpath:/templates/";
	public static final String DEFAULT_SUFFIX = ".html";
}
```

由代码可以看出，thymeleaf模板的默认目录是在classpath:/templates/，后缀为.html。

### 编写页面

现在可以编写第一个html页面了（默认已经开启thymeleaf引擎，并且模板路径在templates目录）。

```html
<!--templates/success.html-->
<!DOCTYPE html>
<!--必须导入名称空间-->
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<h1>成功进入thymeleaf页面</h1>
<div th:text="${message}"></div>
</body>
</html>
```

编写controller代码。

```java
@GetMapping("/success")
public ModelAndView success(ModelMap modelMap)
{
    ModelAndView modelAndView = new ModelAndView();
    modelMap.addAttribute("message", "hello world thymeleaf");
    modelAndView.addObject(modelMap);
    modelAndView.setViewName("success");

    return modelAndView;
}
```

通过浏览器访问来验证。

![image-20191207213803353](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/image-20191207213803353.png)

### 缓存

在开发调试阶段需要关闭缓存。

- thymeleaf服务端缓存；

```yaml
spring:
  thymeleaf:
    cache: false
```

- 开发工具的缓存；
  
  1. 开启自动编译；
  
     ![image-20191208171509189](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/image-20191208171509189.png)
  
  2. registry注册表
  
     使用ctrl+alt+shift+/调出
  
     ![image-20191208183650864](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/image-20191208183650864.png)

- 浏览器缓存

  在调试模式下，关闭cache。F12在Network的Disable cache选项上打勾禁用缓存。

  ![img](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/20180803111823740.png)

禁用缓存之后，在修改静态资源之后，就可以用ctrl+F9重新编译实时生效。

### 登录拦截

在用户打开页面时，需要判断用户是否登录，如果已经登录则正常处理，否则重定向到登录页面。

拦截器实现。

```java
// 编写登录拦截器
public class SigninHandlerInterceptor implements HandlerInterceptor {
	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
		HttpSession session = request.getSession(false);
		if(null != session && !ObjectUtils.isEmpty(session.getAttribute("username"))
				&& !ObjectUtils.isEmpty(session.getAttribute("password")))
		{
			return true;// 登录验证通过
		}
		else
		{
			request.setAttribute("message", "请输入密码");
			request.getRequestDispatcher("/signin").forward(request, response);// 验证是否，则转发到登录页面
			return false;
		}
	}
}
// 实现WebMvcConfigurer接口
@Configuration
public class AppConfig implements WebMvcConfigurer {
	// 拦截器需要排除静态资源和登录页面
	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(new SigninHandlerInterceptor()).addPathPatterns("/**")
				.excludePathPatterns(Arrays.asList("/signin", "/static/**", "/webjars/**", "/assets/**", "/docs/**"));
	}
}
```

如果直接登录非登录页面（http://127.0.0.1:8080/users），则会自动跳转到登录页面。

![image-20191208211716248](springboot%E5%90%AF%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.assets/image-20191208211716248.png)



# 热部署

引用devtools监听文件变动，来实现修改后的文件重新classloader

