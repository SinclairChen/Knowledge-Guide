## 1、容器启动入口

```java
ApplicationContext app = new ClassPathXmlApplicationContext("application.xml");
```



调用构造函数：

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    this(new String[] {configLocation}, true, null);
}
```



最终调用的构造函数：

```java
public ClassPathXmlApplicationContext(
    String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
    throws BeansException {
	
    //调用父类构造方法，并设置资源加载器
    super(parent);
    
    //定位资源
    setConfigLocations(configLocations);
    if (refresh) {
        //重启、刷新、重置
        refresh();
    }
}
```

不管是AnnotionConfigApplicationContext、FileSystemXmlApplicationContext和ClassPathXmlApplicationContext等都继承至AbstractApplicationContext，容器的启动最终都是调用这个refresh()方法。



## 2、获得配置文件路径

在refresh()方法调用之前需要设置好资源加载器和定位资源位置，这两个工作都是交给其父类AbstractApplicationContext完成的。



### 2.1 调用父类的构造方法，并且设置好资源加载器：

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    
    //静态初始化块，在整个容器创建过程中只执行一次
	static {
		//为了避免应用程序在Weblogic8.1关闭时出现类加载异常加载问题
		// 加载IoC容器关闭事件(ContextClosedEvent)类
		ContextClosedEvent.class.getName();
	}
    
    /**
	 * 在ApplicationContext创建的时候拿到资源加载器
	 * Create a new AbstractApplicationContext with no parent.
	 */
	public AbstractApplicationContext() {
		this.resourcePatternResolver = getResourcePatternResolver();
	}
    
    /**
	 * 获取一个Spring Source的加载器用于读取Spring Bean定义资源文件
	 */
	protected ResourcePatternResolver getResourcePatternResolver() {
		//AbstractApplicationContext继承DefaultResourceLoader，因此它自己也是一个资源加载器
        //通过调用PathMatchingResourcePatternResolver的构造方法设置资源加载器
		//Spring资源加载器，其getResource(String location)方法用于载入资源
		return new PathMatchingResourcePatternResolver(this);
	}
}
```



### 2.2 定位资源

调用AbstractRefreshableConfigApplicationContext中的setConfigLocations()方法定位资源

```java
/**
* 处理单个资源文件路径为一个字符串的情况
*/
public void setConfigLocation(String location) {
    //String CONFIG_LOCATION_DELIMITERS = ",; \t\n";
    //即多个资源文件路径之间用” ,; \t\n”分隔，解析成数组形式
    setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));
}


/**
* 解析Bean定义资源文件的路径，处理多个资源文件字符串数组
*/
public void setConfigLocations(@Nullable String... locations) {
    if (locations != null) {
        Assert.noNullElements(locations, "Config locations must not be null");
        this.configLocations = new String[locations.length];
        for (int i = 0; i < locations.length; i++) {
            // resolvePath为同一个类中将字符串解析为路径的方法
            this.configLocations[i] = resolvePath(locations[i]).trim();
        }
    }
    else {
        this.configLocations = null;
    }
}
```



## 3、 载入refresh()方法

refresh()方法是一个模板方法，它规定了IOC容器的启动流程，具体则交给子类实现。主要作用就是在创建IOC容器之前，查看是否有容器存在，如果有则销毁和关闭，并建立新的IOC容器。之后完成Bean配置资源的载入。

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {

      //1、调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
      prepareRefresh();

      //2、告诉子类启动refreshBeanFactory()方法，
      //Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      //3、为BeanFactory配置容器特性，例如类加载器、事件处理器等
      prepareBeanFactory(beanFactory);

      try {
         //4、为容器的某些子类指定特殊的BeanPost事件处理器
         postProcessBeanFactory(beanFactory);

         //5、调用所有注册的BeanFactoryPostProcessor的Bean
         invokeBeanFactoryPostProcessors(beanFactory);

         //6、为BeanFactory注册BeanPost事件处理器.
         //BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
         registerBeanPostProcessors(beanFactory);

         //7、初始化信息源，和国际化相关.
         initMessageSource();

         //8、初始化容器事件传播器.
         initApplicationEventMulticaster();

         //9、调用子类的某些特殊Bean初始化方法
         onRefresh();

         //10、为事件传播器注册事件监听器.
         registerListeners();

         //11、初始化所有剩余的单例Bean
         finishBeanFactoryInitialization(beanFactory);

         //12、初始化容器的生命周期事件处理器，并发布容器的生命周期事件
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         //13、销毁已创建的Bean
         destroyBeans();

         //14、取消refresh操作，重置容器的同步标识。
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }
      finally {

         //15、重设公共缓存
         resetCommonCaches();
      }
   }
}
```



## 4、创建容器

obtainFreshBeanFactory()方法是为了调用子类AbstractRefreshableApplicationContext中的refreshBeanFactory()方法：

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    //这里使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法
    // 具体实现调用子类容器的refreshBeanFactory()方法，刷新工厂
    refreshBeanFactory();
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```



调用refreshBeanFactory()方法：

```java
protected final void refreshBeanFactory() throws BeansException {
    //如果已经有容器，销毁容器中的bean，关闭容器
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        //创建保存所有bean定义信息的IOC容器，并设置序列化id
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());

        //对IOC容器进行定制化，如设置启动参数，开启注解的自动装配等
        customizeBeanFactory(beanFactory);

        //调用载入Bean定义的方法，主要这里又使用了一个委派模式
        //在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
        //加载bean定义信息
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```



调用createBeanFactory()方法，创建DefaultListableBeanFactory：

```java
protected DefaultListableBeanFactory createBeanFactory() {
    return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}
```



## 5、初始化Bean定义信息读取器，去读取资源

调用loadBeanDefinitions()方法，在AbstractRefreshableApplicationContext中这个方法只是一个抽象方法，具体是调用它子类AbstractXmlApplicationContext中的loadBeanDefinitions()方法。

```
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		
		// 创建XmlBeanDefinitionReader，即创建Bean读取器
		// 并通过回调设置到容器中去，容器使用该读取器读取Bean定义资源
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		beanDefinitionReader.setEnvironment(this.getEnvironment());

		// 为Bean读取器设置Spring资源加载器
		// AbstractXmlApplicationContext的祖先父类
		// AbstractApplicationContext继承DefaultResourceLoader
		// 因此，容器本身也是一个资源加载器
		beanDefinitionReader.setResourceLoader(this);

		//为Bean读取器设置SAX xml解析器
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		// 初始化beanDefinition读取器，设置xml校验器
		initBeanDefinitionReader(beanDefinitionReader);

		// Bean读取器真正实现加载的方法
		loadBeanDefinitions(beanDefinitionReader);
	}
```



## 6、获取资源配置，调用Bean定义信息读取器读取

调用loadBeanDefinitions()方法，获取配置资源，并使用去读取器读取配置资源

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    //获取Bean定义资源
    Resource[] configResources = getConfigResources();

    //如果子类中获取的Bean定义资源定位不为空
    if (configResources != null) {
        //Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取Bean定义资源
        reader.loadBeanDefinitions(configResources);
    }

    //如果子类中获取的Bean定义资源定位为空
    //则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        //重新使用资源加载器去加载资源然后再使用读取器读取
        reader.loadBeanDefinitions(configLocations);
    }
}

//如果有多个资源文件，逐个加载
//重载方法，调用loadBeanDefinitions(String);
@Override
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
    Assert.notNull(locations, "Location array must not be null");
    int counter = 0;

    for (String location : locations) {
        counter += loadBeanDefinitions(location);
    }
    return counter;
}
```

这里XmlBeanDefinitionReader.loadBeanDefinitions(Resource)方法其实是调用其父类AbstractBeanDefinitionReader中的loadBeanDefinitions()方法。



### 6.1 如果资源为空，需要重新加载资源

调用AbstractBeanDefinitionReader.loadBeanDefinitions()方法：

```java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    //获取在IoC容器初始化过程中设置的资源加载器
    ResourceLoader resourceLoader = getResourceLoader();
    if (resourceLoader == null) {
        throw new BeanDefinitionStoreException(
            "Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
    }

    //加载多个资源的
    if (resourceLoader instanceof ResourcePatternResolver) {
        // Resource pattern matching available.
        try {
            //使用资源加载器将资源解析为Spring IOC容器封装的资源Resource
            //加载多个指定位置的Bean定义资源文件
            Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);

            //最终都是委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
            int loadCount = loadBeanDefinitions(resources);
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
        //加载单个资源的
        // Can only load single resources by absolute URL.
        Resource resource = resourceLoader.getResource(location);
        //委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
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

//如果有多个配置资源，逐个加载每个配置资源中的内容
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int counter = 0;
    
    for (Resource resource : resources) {
        counter += loadBeanDefinitions(resource);
    }
    return counter;
}
```



## 7、开始读取配置资源

调用XmlBeanDefinitionReader.loadBeanDefinitions()方法：

```java
/**
*
* XmlBeanDefinitionReader加载资源的入口方法
*/
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    //将读入的XML资源进行特殊编码处理
    return loadBeanDefinitions(new EncodedResource(resource));
}

/**
* 载入XML形式Bean定义资源文件方法
*
*/
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isInfoEnabled()) {
        logger.info("Loading XML bean definitions from " + encodedResource.getResource());
    }
    
	//获取当前资源
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
        //将资源文件转为InputStream的IO流
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            //从InputStream中得到XML的解析源
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            //这里是具体的读取过程
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            //关闭从Resource中得到的IO流
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
```



调用doLoadBeanDefinitions()方法读取资源

```java
/**
* 从特定XML文件中实际载入Bean定义资源的方法
*
*/
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    throws BeanDefinitionStoreException {
    try {
        //将XML文件转换为DOM对象，解析过程由documentLoader实现
        Document doc = doLoadDocument(inputSource, resource);

        //注册Bean定义信息
        //首先需要解析Document对象，然后完成注册
        return registerBeanDefinitions(doc, resource);
    }
    catch (BeanDefinitionStoreException ex) {
        throw ex;
    }
    catch (SAXParseException ex) {
        throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                                                  "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
    }
    catch (SAXException ex) {
        throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                                                  "XML document from " + resource + " is invalid", ex);
    }
    catch (ParserConfigurationException ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                                               "Parser configuration exception parsing XML from " + resource, ex);
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                                               "IOException parsing XML document from " + resource, ex);
    }
    catch (Throwable ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                                               "Unexpected exception parsing XML document from " + resource, ex);
    }
}
```



## 8、将资源转换为Document对象

调用documentLoader.loadDocument()方法：

```java
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
   return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
         getValidationModeForResource(resource), isNamespaceAware());
}
```



最终调用的是DefaultDocumentLoader.loadDocument()方法将其转换为Document对象：

```java
/**
* 使用标准的JAXP将载入的Bean定义资源转换成document对象
*/
@Override
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
                             ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

    //创建文件解析器工厂
    DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
    if (logger.isDebugEnabled()) {
        logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
    }
    //设置文档解析器
    DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
    //解析Spring的Bean定义资源
    return builder.parse(inputSource);
}

//创建文档解析工厂
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
    throws ParserConfigurationException {

    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    factory.setNamespaceAware(namespaceAware);

    //设置解析XML的校验
    if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
        factory.setValidating(true);
        if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
            // Enforce namespace aware for XSD...
            factory.setNamespaceAware(true);
            try {
                factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
            }
            catch (IllegalArgumentException ex) {
                ParserConfigurationException pcex = new ParserConfigurationException(
                    "Unable to validate using XSD: Your JAXP provider [" + factory +
                    "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
                    "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
                pcex.initCause(ex);
                throw pcex;
            }
        }
    }

    return factory;
}
```

