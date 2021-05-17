# BeanFactory

## BeanFactory的类图      

  beanFacotry是用于创建bean的工厂类。

​		其实现子类如下图：

​	![image-20210130170515274](D:\文档\笔记\spring5核心原理书籍\Spring--BeanFactory.assets\image-20210130170515274.png)

​	BeanFactory作为一个顶层的接口类，定义了一个IOC容器的基本功能规范，BeanFactory有三个重要的子类，分别是ListableBeanFactory、HierarchicalBeanFactory和AutowireCapableBeanFactory。其中DefaultListableBeanFactory实现了这三个接口。

​	ListableBeanFactory表示这些Bean可列表化，HierarchicalBeanFactory表示这些Bean之间有父子关系，AutowireCapableBeanFactory表示Bean自动装配的一些规则。

接下来我们先看BeanFactory的源码

```
public interface BeanFactory {

	// 对FactoryBean的转义定义，因为如果使用Bean的名字检索FactoryBean得到的对象是工厂生成的对象
	// 如果需要得到工厂本身，需要转义
	String FACTORY_BEAN_PREFIX = "&";

	// 根据Bean的名字拿到对应的实例
	Object getBean(String name) throws BeansException;

	// 根据Bean的名字和Class类型来获取Bean的实例，增加了类型安全检查机制
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	/**
	 * Return an instance, which may be shared or independent, of the specified bean.
	 * <p>Allows for specifying explicit constructor arguments / factory method arguments,
	 * overriding the specified default arguments (if any) in the bean definition.
	 */
	Object getBean(String name, Object... args) throws BeansException;

	/**
	 * Return the bean instance that uniquely matches the given object type, if any.
	 */
	<T> T getBean(Class<T> requiredType) throws BeansException;

	/**
	 * Return an instance, which may be shared or independent, of the specified bean.
	 */
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	/**
	 * Return a provider for the specified bean, allowing for lazy on-demand retrieval
	 * of instances, including availability and uniqueness options.
	 */
	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

	/**
	 * Return a provider for the specified bean, allowing for lazy on-demand retrieval
	 * of instances, including availability and uniqueness options.
	 */
	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);
	
	// bean容器中是否有该bean
	boolean containsBean(String name);
	// 根据bean的名字获取bean，并判断是否是单例
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	/**
	 * Check whether the bean with the given name matches the specified type.
	 */
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	/**
	 * Check whether the bean with the given name matches the specified type.
	 */
	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	/**
	 * 得到 Bean 实例的 Class 类型
	 * Determine the type of the bean with the given name. More specifically,
	 * determine the type of object that {@link #getBean} would return for the given name.
	 */
	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

	/**
	 * Return the aliases for the given bean name, if any.
	 */
	String[] getAliases(String name);
```

ApplicationContext除了提供IOC容器的基本功能，还为用户提供了以下附加服务。

* 支持信息源，可以实现国际化（实现MessageSource接口）
* 访问资源（实现ResourcePatternResolver接口）
* 支持应用事件（实现ApplicationEventPublisher接口）

## Bean的定义：BeanDefinition

Spring IOC容器管理我们定义的各种Bean对象及其相互关系，Bean对象在Spring实现中以BeanDefinition来描述。其继承体系如下：

![image-20210130202542888](D:\文档\笔记\spring5核心原理书籍\Spring--BeanFactory.assets\image-20210130202542888.png)

## Bean的解析：BeanDefinitionReader

BeanDefinitionReader：Bean的解析主要是对Spring配置文件的解析，这个解析过程主要通过BeanDefinitionReader来完成。结构图如下：

![image-20210130203259752](D:\文档\笔记\spring5核心原理书籍\Spring--BeanFactory.assets\image-20210130203259752.png)

## ApplicationContext 类图



![ApplicationContext](D:\文档\笔记\spring5核心原理书籍\Spring--BeanFactory.assets\ApplicationContext.png)

## Bean容器的启动

关键点：AbstractApplicationContext的refresh()方法

```java
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

			// Prepare this context for refreshing.
			// 1. 调用容器准确刷新方法，该方法用于获取容器当前时间，同时给容器设置同步标识
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			// 2. 告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从子类的refreshBeanFactory()
			// 方法启动
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// 3. 为BeanFactory配置容器特性，例如类加载器、事件处理器等
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 4. 为容器的某些子类指定特殊的post事件处理器等
				postProcessBeanFactory(beanFactory);

				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
				// Invoke factory processors registered as beans in the context.
				// 5.调用所有注册的BeanFactoryPostProcessors
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 6. 为BeanFactory注册Post事件处理器
				// BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
				registerBeanPostProcessors(beanFactory);
				beanPostProcess.end(); // 记录beanPostProcess的状态，并且该状态不能再被改变

				// Initialize message source for this context.
				// 7. 初始化信息源，与国际化相关
				initMessageSource();

				// Initialize event multicaster for this context.
				// 8. 初始化容器事件传播器
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 9. 调用子类的某些特殊Bean的初始化方法
				onRefresh();

				// Check for listener beans and register them.
				// 10. 为事件传播器注册事件监听器
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 11. 初始化所有剩余的单例bean
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 12. 初始化容器的生命周期事件处理器，并发布容器的生命周期事件
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				// 13. 销毁已经创建的bean
				destroyBeans();

				// Reset 'active' flag.
				// 14. 取消刷新操作，重置容器的同步标识
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				// 重置公共缓存
				resetCommonCaches();
				// 上下文刷新结束
				contextRefresh.end();//记录上下文刷新状态，该状态不能再改变
			}
		}
	}
```

   ## 创建容器

​		上文的AbstractApplicationContext定义了bean容器启动的一些过程，其中第二步就是创建容器`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()`，该方法调用了refreshBeanFactory()方法，该方法是一个抽象方法，由子类来实现，然后调用子类的方法来完成容器的创建。先看obtainFreshBeanFactory的代码。

```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}
	protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;
```

我们可以到实现该方法主要有两个类：

![image-20210131230012672](D:\文档\笔记\spring5核心原理书籍\Spring--BeanFactory.assets\image-20210131230012672.png)

接下来来看下AbstractRefreshableApplicationContext类中的refreshBeanFactory()方法代码：

```java
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) { // 如果有容器，销毁容器中的Bean，关闭容器
			destroyBeans(); // 销毁容器中的Bean
			closeBeanFactory(); // 关闭容器
		}
		try {
            // 创建IOC容器，也就是说DefaultListableBeanFactory就是用来存储bean的
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
            // 对IOC容器进行定制，如设置启动参数、开启注解的自动装配
			customizeBeanFactory(beanFactory);
            // 调用载入Bean定义的方法，在当前类中只定义了抽象的loadBeanDefinitions()方法，调用子类容器实现
            // 具体实现了该方法的类如下图loadBeanDefinitions
			loadBeanDefinitions(beanFactory);
			this.beanFactory = beanFactory;
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

![image-20210131231237245](D:\文档\笔记\spring5核心原理书籍\Spring--BeanFactory.assets\image-20210131231237245.png)

​																					图 loadBeanDefinitions

要求：Debug看看

##  装载路径配置

接着看看AbstractXmlApplicationContext类中对于loadBeanDefinitions()方法的实现

```java
//载入bean定义	
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
    	// 创建Bean读取器。通过回调设置到容器中，容器使用该读取器读取Bean配置资源
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment. 为Bean读取器设置Spring资源加载器
		beanDefinitionReader.setEnvironment(this.getEnvironment());
    	// AbstractXmlApplicationContext的父类AbstractApplicationContext继承DefaultResurceLoader
    	// 因此，容器本身也是一个资源加载器
		beanDefinitionReader.setResourceLoader(this);
    	// 为Bean读取器设置SAX xml解析器
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
    	// 当Bean读取器读取到Bean定义的xml资源文件时，启用xml的校验机制
		initBeanDefinitionReader(beanDefinitionReader);
    	// 为Bean读取器真正实现加载的方法
		loadBeanDefinitions(beanDefinitionReader);
	}

	protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
		reader.setValidating(this.validating);
	}

	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        // 获取Bean配置的资源定位
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
            // 调用的XmlBeanDefinitionReader的父类的方法
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
            // 同样调用的其父类的方法
			reader.loadBeanDefinitions(configLocations);
		}
	}
```
## 分配路径处理策略

AbstractBeanDefinitionReader类

```java
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

    	//resourceLoader是 ClassPathXmlApplicationContext，ClassPathXmlApplicationContext是ResourcePatternResolver的子类
		if (resourceLoader instanceof ResourcePatternResolver) { 
			// Resource pattern matching available.
			try {
                // 将指定位置的Bean配置信息解析为Spring IOC容器封装的资源
                // 加载多个指定位置的xml文件中配置的Bean信息
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
                // 调用子类XmlBeanDefinitionReader的方法，实现加载功能
				int loadCount = loadBeanDefinitions(resources);// 一个config的配置返回4，去了解 TODO
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}
```

## 解析配置文件路径

​	XmlBeanDefinitionReader通过调用ClassPathXmlApplicationContext的父类DefaultResourceLoader的getResource()方法获取要加载的资源，来看看DefaultResourceLoader加载资源的方式：

```java
/**
 *  资源路径的表示方式的不同，对应着不同的配置文件加载方式
 *   1. ClassPathContextResource 2. ClassPathResource  3.UrlResource 4.FileSystemResource
 *   它们对应的路径表示是什么样？显示它们之间的差异。
**/
public Resource getResource(String location) {
		Assert.notNull(location, "Location must not be null");

		for (ProtocolResolver protocolResolver : this.protocolResolvers) {
			Resource resource = protocolResolver.resolve(location, this);
			if (resource != null) {
				return resource;
			}
		}

		if (location.startsWith("/")) {
			return getResourceByPath(location); // 通过ClassPathContextResource解析
		}
		else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader()); // 通过ClassPathResource解析，即以" classpath*："开头
		}
		else {
			try {
				// Try to parse the location as a URL...
				URL url = new URL(location);
				return new UrlResource(url); //
			}
			catch (MalformedURLException ex) {
				// No URL -> resolve as resource path.
                // 如果既不是classpath又不是Url，就用默认的ClassPathContextResource
				return getResourceByPath(location);
			}
		}
	}

	protected Resource getResourceByPath(String path) {
		return new ClassPathContextResource(path, getClassLoader());
	}
```

## 读取配置内容

​	XmlBeanDefinitionReader的loadBeanDefinitions()

```java
    // 对读入的XML资源进行特殊编码处理
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
        return loadBeanDefinitions(new EncodedResource(resource));
    }
	
	// 载入XML形式Bean配置信息方法
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
            // 将资源文件转化为I/O流
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
                // 从I/O流中得到XML解析源
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                // 具体读取过程
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}

protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            // 将xml转化为dom对象
			Document doc = doLoadDocument(inputSource, resource);
            // 启动对Bean定义解析的详细过程，该解析过程会用dao
			return registerBeanDefinitions(doc, resource);
		}
		......
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
```

接下来是一堆解析，此处省略....

## 分配注册策略

​	    在DefaultBeanDefinitionDocumentReader对Bean的解析后，将BeanDefinition(Bean的定义)转换为BeanDefinitionHold对象，然后调用BeanDefinitionReaderUtils的registerBeanDefinition()方法向Spring IOC容器注册解析的Bean.

```java
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
        // 获取Bean的名称,一般为 包名+类名
		String beanName = definitionHolder.getBeanName();
        // 向Spring容器中注入BeanDefinition
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
        // 若有别名，则向Spring容器中注册别名
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

​	当调用BeanDefinitionReaderUtils向Spring IOC容器注册解析的BeanDefinition时，真正完成注册功能的DefaultListBeanFactory。实际上，DefaultListBeanFactory继承了BeanDefinitionRegistry。

![DefaultListableBeanFactory](D:\文档\笔记\spring5核心原理书籍\Spring--BeanFactory.assets\DefaultListableBeanFactory.png)

```java
	DefaultListBeanFactory类中实现BeanDefinitionRegistry接口中的registerBeanDefinition方法
        
  	// 来看DefaultListBeanFactory中存储Bean的数据结构
    // 存储注册信息的BeanDefinition
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);

	/** Map of singleton and non-singleton bean names, keyed by dependency type */
	// 结合getBean()来理解
	private final Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<Class<?>, String[]>(64);

	/** Map of singleton-only bean names, keyed by dependency type */
    // 同样结合getBean()来理解
	private final Map<Class<?>, String[]> singletonBeanNamesByType = new ConcurrentHashMap<Class<?>, String[]>(64);
	
    @Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition oldBeanDefinition;

		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + oldBeanDefinition + "] bound.");
			}
			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (this.logger.isWarnEnabled()) {
					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							oldBeanDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(oldBeanDefinition)) {
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
                // 不能修改启动集合中的任何元素
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
                // 启动注册阶段
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
```

# 基于注解的IOC初始化

## 注解处理策略

​	1）类级别注解，如@Component、@Respsitory、@Controller、@Service等。处理策略：根据注解过滤规则扫描读取注解Bean定义类，并将其注册到spring容器中。

​	2）类内部的注解。如@Autowired、@Value、@Resource。都是添加在类内部的字段或者方法上的类内部注解，Spring IOC容器通过Bean的后置处理器(BeanPostProcessor)解析Bean内部注解

## 定位Bean扫描路径

​	主要看AnnotationConfigApplicationContext.

先看继承的AnnotationConfigRegistry接口

```java
public interface AnnotationConfigRegistry {

	/**
	 * Register one or more annotated classes to be processed.
	 * <p>Calls to {@code register} are idempotent; adding the same
	 * annotated class more than once has no additional effect.
	 * @param annotatedClasses one or more annotated classes,
	 * e.g. {@link Configuration @Configuration} classes
	 */
	void register(Class<?>... annotatedClasses);

	/**
	 * Perform a scan within the specified base packages.
	 * @param basePackages the packages to check for annotated classes
	 */
	void scan(String... basePackages);

}
```

```
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {

	// 读取Bean定义读取器,并将其设置到容器中
	private final AnnotatedBeanDefinitionReader reader;
	// 扫描指定类路径中注解Bean定义扫描器,并将其设置到容器中
	private final ClassPathBeanDefinitionScanner scanner;


	/**
	 * Create a new AnnotationConfigApplicationContext that needs to be populated
	 * through {@link #register} calls and then manually {@linkplain #refresh refreshed}.
	 */
	public AnnotationConfigApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}

	/**
	 * Create a new AnnotationConfigApplicationContext with the given DefaultListableBeanFactory.
	 * @param beanFactory the DefaultListableBeanFactory instance to use for this context
	 */
	public AnnotationConfigApplicationContext(DefaultListableBeanFactory beanFactory) {
		super(beanFactory);
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}

	/**
	 * Create a new AnnotationConfigApplicationContext, deriving bean definitions
	 * from the given annotated classes and automatically refreshing the context.
	 * @param annotatedClasses one or more annotated classes,
	 * e.g. {@link Configuration @Configuration} classes
	 * 将涉及到的配置类传递给该构造函数,实现将相应配置类中的Bean自动注册容器中
	 */
	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();
	}

	/**
	 * Create a new AnnotationConfigApplicationContext, scanning for bean definitions
	 * in the given packages and automatically refreshing the context.
	 * @param basePackages the packages to check for annotated classes
	 * 自动扫描已给定的包以及其子包下的所有类,并自动识别所有的spring bean,将其注册到容器中
	 */
	public AnnotationConfigApplicationContext(String... basePackages) {
		this();
		scan(basePackages);
		refresh();
	}


	/**
	 * {@inheritDoc}
	 * <p>Delegates given environment to underlying {@link AnnotatedBeanDefinitionReader}
	 * and {@link ClassPathBeanDefinitionScanner} members.
	 */
	@Override
	public void setEnvironment(ConfigurableEnvironment environment) {
		super.setEnvironment(environment);
		this.reader.setEnvironment(environment);
		this.scanner.setEnvironment(environment);
	}

	/**
	 * Provide a custom {@link BeanNameGenerator} for use with {@link AnnotatedBeanDefinitionReader}
	 * and/or {@link ClassPathBeanDefinitionScanner}, if any.
	 * <p>Default is {@link org.springframework.context.annotation.AnnotationBeanNameGenerator}.
	 * <p>Any call to this method must occur prior to calls to {@link #register(Class...)}
	 * and/or {@link #scan(String...)}.
	 * @see AnnotatedBeanDefinitionReader#setBeanNameGenerator
	 * @see ClassPathBeanDefinitionScanner#setBeanNameGenerator
	 * 为Bean读取器和扫描器设置Bean名字产生器
	 */
	public void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
		this.reader.setBeanNameGenerator(beanNameGenerator);
		this.scanner.setBeanNameGenerator(beanNameGenerator);
		getBeanFactory().registerSingleton(
				AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
	}

	/**
	 * Set the {@link ScopeMetadataResolver} to use for detected bean classes.
	 * <p>The default is an {@link AnnotationScopeMetadataResolver}.
	 * <p>Any call to this method must occur prior to calls to {@link #register(Class...)}
	 * and/or {@link #scan(String...)}.
	 * 为Bean读取器和扫描器设置元信息解析器
	 */
	public void setScopeMetadataResolver(ScopeMetadataResolver scopeMetadataResolver) {
		this.reader.setScopeMetadataResolver(scopeMetadataResolver);
		this.scanner.setScopeMetadataResolver(scopeMetadataResolver);
	}

	@Override
	protected void prepareRefresh() { 
		this.scanner.clearCache(); // Clear the underlying metadata cache, removing all cached class metadata.
		super.prepareRefresh(); // Prepare this context for refreshing, setting its startup date and active flag as well as performing any initialization of property sources.
	}


	//---------------------------------------------------------------------
	// Implementation of AnnotationConfigRegistry
	//---------------------------------------------------------------------

	/**
	 * Register one or more annotated classes to be processed.
	 * <p>Note that {@link #refresh()} must be called in order for the context
	 * to fully process the new classes.
	 * @param annotatedClasses one or more annotated classes,
	 * e.g. {@link Configuration @Configuration} classes
	 * @see #scan(String...)
	 * @see #refresh()
	 * 为容器注册一个要被处理的注解Bean,新注册的Bean,必须手动调用容器的refresh()方法刷新容器,触发容器对新注册Bean的处理
	 */
	public void register(Class<?>... annotatedClasses) {
		Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
		this.reader.register(annotatedClasses);
	}

	/**
	 * Perform a scan within the specified base packages.
	 * <p>Note that {@link #refresh()} must be called in order for the context
	 * to fully process the new classes.
	 * @param basePackages the packages to check for annotated classes
	 * @see #register(Class...)
	 * @see #refresh()
	 * 扫描指定包路径及其子包下的注解类,为了使新添加的类被处理,必须手动调用refresh()方法刷新容器
	 */
	public void scan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		this.scanner.scan(basePackages);
	}

}

```

## 读取注解元数据

AnnotationBeanDefintionReader的register()方法向容器中注册指定注解Bean。源码如下：

```
/**
	 * Register one or more annotated classes to be processed.
	 * <p>Calls to {@code register} are idempotent; adding the same
	 * annotated class more than once has no additional effect.
	 * @param annotatedClasses one or more annotated classes,
	 * e.g. {@link Configuration @Configuration} classes
	 * 注册一个或多个注解bean定义类
	 */
	public void register(Class<?>... annotatedClasses) {
		for (Class<?> annotatedClass : annotatedClasses) {
			registerBean(annotatedClass);
		}
	}

	/**
	 * Register a bean from the given bean class, deriving its metadata from
	 * class-declared annotations.
	 * @param annotatedClass the class of the bean
	 * 注册一个注解bean定义类
	 */
	@SuppressWarnings("unchecked")
	public void registerBean(Class<?> annotatedClass) {
		registerBean(annotatedClass, null, (Class<? extends Annotation>[]) null);
	}

	/**
	 * Register a bean from the given bean class, deriving its metadata from
	 * class-declared annotations.
	 * @param annotatedClass the class of the bean
	 * @param qualifiers specific qualifier annotations to consider,
	 * in addition to qualifiers at the bean class level
	 */
	@SuppressWarnings("unchecked")
	public void registerBean(Class<?> annotatedClass, Class<? extends Annotation>... qualifiers) {
		registerBean(annotatedClass, null, qualifiers);
	}

	/**
	 * Register a bean from the given bean class, deriving its metadata from
	 * class-declared annotations.
	 * @param annotatedClass the class of the bean
	 * @param name an explicit name for the bean
	 * @param qualifiers specific qualifier annotations to consider,
	 * in addition to qualifiers at the bean class level
	 */
	@SuppressWarnings("unchecked")
	public void registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers) {
	   // 创建spring容器中对注解bean的封装的数据结构
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}
		
		// 解析注解Bean定义的作用域，若@Scope("prototype"),则bean为原型类型，若为@Scope("singleton")，
		// 则Bean为单态类型
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		abd.setScope(scopeMetadata.getScopeName()); //设置注解bean的作用域
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
		// 处理注解Bean定义中的通用注解
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		// 如果在向容器中注册Bean定义时，使用了额外的限定符注解，则解析限定符注解
		// 主要配置autowiring自动依赖注入装配的限定条件，即@Qualified注解
		// spring注入的时候默认按照type进行装配，若使用@Qualified则按名称进行装配
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true); // @Primary
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true); // @Lazy
				}
				else { // 如果使用了除@Primary和@Lazy以外的注解，则为该Bean添加一个autowiring自动
					// 自动注入装配限定符，该Bean在autowiring自动依赖注入装配时，根据名称装配限定符指定的Bean
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}

```



