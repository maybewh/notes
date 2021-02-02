# BeanFactory

## BeanFactory的类图      

  beanFacotry是用于创建bean的工厂类。

​		其实现子类如下图：

​	![image-20210130170515274](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210130170515274.png)

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

![image-20210130202542888](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210130202542888.png)

## Bean的解析：BeanDefinitionReader

BeanDefinitionReader：Bean的解析主要是对Spring配置文件的解析，这个解析过程主要通过BeanDefinitionReader来完成。结构图如下：

![image-20210130203259752](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210130203259752.png)

## ApplicationContext 类图



![ApplicationContext](D:\文档\笔记\spring5核心原理书籍\ApplicationContext.png)

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

![image-20210131230012672](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210131230012672.png)

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

![image-20210131231237245](C:\Users\abc\AppData\Roaming\Typora\typora-user-images\image-20210131231237245.png)

​																					图 loadBeanDefinitions

要求：Debug看看

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
```

