

# Spring Bean生命周期

---

## SpringBean生命周期流程

**实例化Spring容器**

**扫描出符合Spring Bean规则的类，添加到一个集合中**

**遍历集合中的类，将类信息缓存到BeanDefinition对象**(实例化BeanDefinition)

**将BeanDefinition对象put到BeanDefinitionMap中**

**执行BeanFactoryPostProcessor**(包括Spring容器默认的和程序员提供的)

**遍历BDMap，验证BD对象**(验证是否单例、是否原型、是否懒加载、是否DependsOn等)

**实例化对象**(得到类的class对象，推断出一个合适的构造方法，使用反射调用构造方法)

**缓存注解信息，解析合并后的BD对象**

**提前暴露一个工厂对象**(循环依赖)(三级缓存 addSingletonFactory)

**判断是否需要属性注入**(属性注入)

**回调部分Aware接口**(不同的Aware接口的执行时机不同)

**执行生命周期初始化回调方法**(先调注解形式，再调接口形式)

**完成代理**(AOP、事件发布、监听)

**将对象put到单例池中**(singletonObjects)(对象成为真正的Bean)

**销毁Bean**



## 初始化Spring容器

Spring容器：**ApplicationContext**

ApplicationContext是一个接口，包含众多的实现类，常用的实现类有：`ClassPathXmlApplicationContext`、`AnnotationConfigApplicationContext`....等

下面以`AnnotationConfigApplicationContext`为例研究Spring初始化流程：

~~~ java
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class)
~~~

~~~ java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		// 调用构造方法
		this();

		// 注册配置，因为配置需要解析，一般不需要自己扫描
		// beanDefinitionMap.put("appConfig", beanDefinition)
		register(annotatedClasses);

		// Bean的扫描
		// Bean的实例化
		refresh();
	}
~~~

**this**：`AnnotationConfigApplicationContext`类的有参构造方法调用了`this()`方法，进而调用其父类的默认无参构造方法

~~~ java
/**
DefaultResourceLoader --> AbstractApplicationContext --> 
    GenericApplicationContext --> AnnotationConfigApplicationContext
**/

public class AnnotationConfigApplicationContext extends GenericApplicationContext
public GenericApplicationContext() {
    // 实例化了一个BeanFactory --> bean工厂
    this.beanFactory = new DefaultListableBeanFactory();
}

public class GenericApplicationContext extends AbstractApplicationContext
public AbstractApplicationContext() {
    // 实例化一个资源解释器
    this.resourcePatternResolver = getResourcePatternResolver();
}

public abstract class AbstractApplicationContext extends DefaultResourceLoader
public DefaultResourceLoader() {
    // 实例化一个类加载器
    this.classLoader = ClassUtils.getDefaultClassLoader();
}

~~~

**register**：`register`方法会一直调用到`doRegisterBean`方法，将配置类的BeanDefinition对象注册到Spring容器

**regiater --> doRegister**

~~~ java
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, 
                        @Nullable String name, @Nullable Class<? extends Annotation>[] qualifiers,
                        BeanDefinitionCustomizer... definitionCustomizers) {

		// 将class对象转换为一个BeanDefinition对象(包括类的注解元数据)
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
    
		// ......省略与主题无关的代码

		// 将BeanDefinition对象存到BeanDefinitionHolder中(封装)
		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		// 将BD对象放到BDMap中
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}
~~~

- **BeanDefinitionHolder**：封装BD对象，方便传参(没有多高深)

  ~~~ java
  public class BeanDefinitionHolder implements BeanMetadataElement {
  
  	private final BeanDefinition beanDefinition;
  
  	private final String beanName;
  
  	@Nullable
  	private final String[] aliases;
  ~~~

**doRegisterBean --> registerBeanDefinition**

~~~ java
public static void registerBeanDefinition(BeanDefinitionHolder definitionHolder,							BeanDefinitionRegistry registry)throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();
    // 将BeanDefinition对象(编写的注解类/配置类) 放到BeanDefinitionMap中
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
    
    // .....
}
~~~

- **Question**：

  - 为什么需要把配置类放到BeanDefinitionMap中？配置类又不是一个Bean，只有Bean才需要被存到BDMap中？

    为了将配置类实例化出来成为一个Bean
  
  - 为什么需要将配置类手动注册到Spring容器，而不是通过扫描进行注册？
  
    因为只有解析了配置类的信息，Spring才能知道要去扫描哪些包下的类(`ComponentScan`注解)，才能将其他的类扫描到Spring容器



## 扫描类

## 缓存BeanDefinition

## 执行BeanFactoryPostProcessor

**扫描类、缓存BeanDefinition、执行BeanFactoryPostProcessor**都是在`refresh`方法的`invokeBeanFactoryPostProcessors`方法中完成

**refresh --> invokeBeanFactoryPostProcessors**

~~~ java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
            // ......
            
            // Invoke factory processors registered as beans in the context.
            // 完成class文件的扫描，将该class代表的类的信息(是否单例、原型、是否懒加载、类名、类的class等等)封装到一个beanDefinition中
            // 并将该beanDefinition放入到beanDefinitionMap中 -- key: 对象名(例如userService),  value: IndexService的封装beanDefinition对象
            invokeBeanFactoryPostProcessors(beanFactory);
~~~

~~~ java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) { 
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}
~~~



## 遍历BDMap，验证BD对象

此步骤在`refresh`方法的`finishBeanFactoryInitialization`方法中完成

**refresh --> finishBeanFactoryInitialization --> preInstantiateSingletons**

~~~ java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
            // .....
            
            // Instantiate all remaining (non-lazy-init) singletons.
            // spring开始实例化单例的类
            finishBeanFactoryInitialization(beanFactory);
~~~

~~~ java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // .....

    // Instantiate all remaining (non-lazy-init) singletons.
    // 实例化所有的单例，非lazy的类对象
    // 完成Bean的生命周期
    beanFactory.preInstantiateSingletons();
}
~~~

~~~ java
public void preInstantiateSingletons() throws BeansException {
    
    // ......
    
    // 拿到所有封装的beanDefinition的类名(例如userService)
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
        // 使用BeanName作为key 去 BeanDefinitionMap中获取到一个BeanDefinition对象
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 对bd对象进行校验(校验是否非抽象类、是否单例、是否非懒加载)
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 验证当前对象是否是一个FactoryBean
            if (isFactoryBean(beanName)) {
                // .....
            }
            else {
                // 非FactoryBean -> 开始实例化普通的对象
                getBean(beanName);
            }
~~~



## 实例化对象

此步骤在`getBean`方法中完成：

~~~ java
// 空壳方法
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}
~~~

**getBean --> doGetBean**

~~~ java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    // 校验bean的名字是否合法(初步理解)(事实上这个方法还做了很多事情)
    final String beanName = transformedBeanName(name);
    Object bean;

    // Eagerly check singleton cache for manually registered singletons.
    // 重点方法：getSingleton()
    // 从单例池中根据beanName拿Bean -> 验证beanName对应的这个对象是否在Spring容器中存在
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 单例池中存在当前对象的Bean.....
    }

    else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        // 判断beanName对应的这个对象是否在创建过程中(是否在 正在创建的原型集合 当中)
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 执行一系列判断.....

        try {
            // 继续判断(DependsOn信息等)......

            // Create bean instance.
            if (mbd.isSingleton()) {
                // 实例化Bean --> Bean的生命周期在getSingleton()方法中走完
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        // 真正完成Bean的创建  -> 对象创建 + 走Bean的生命周期
                        return createBean(beanName, mbd, args);
                    }
~~~

**doGetBean --> getSingleton**

~~~ java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        // 从单例池中拿Bean -> 判断对象是否已经被创建
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            if (this.singletonsCurrentlyInDestruction) {
                // 判断对象是否正在被销毁(是否在 正在销毁的集合 当中)
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            // 将当前对象put到 正在创建的集合 当中 --> 与循环依赖有关
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                // getObject(): 调用getSingleton方法中lambda表达式中的createBean()方法
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
~~~

**getSingleton --> getObject --> createBean**

在`createBean`方法中真正完成Bean的创建  -> 对象创建 + 走Bean的生命周期

~~~ java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
    // .....
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    // 得到类对象 -> 从beanDefinition对象中获取bean的类型
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    // .....

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        // 第一次调用后置处理器
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch {...}

    try {
        // 创建bean
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
}
~~~

**createBean --> doCreateBean**

使用`BeanWrapper`包装类封装Bean，开始初始化Bean

~~~ java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) throws BeanCreationException {

    // Instantiate the bean.
    // Bean的包装类，开始初始化Bean
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 实例化对象(此步仅完成对象的创建，对象还未成为一个Bean)，并第二次调用后置处理器
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
~~~

**doCreateBean --> createBeanInstance**

通过类的class对象推断出一个合适的构造方法，使用该构造方法实例化对象

~~~ java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // .....

    // Candidate constructors for autowiring?
    // 如果是自动装配，则推断出各种候选的构造方法
    // 第二次调用后置处理器推断构造方法，选择一个合适的构造方法，通过反射实例化对象
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        // 利用推断出来的构造方法实例化对象
        return autowireConstructor(beanName, mbd, ctors, args);
    }
    
    // .....

    // No special handling: simply use no-arg constructor.
    // 如果没有推断出合适的构造方法(或没有提供特殊的构造方法)，则使用默认的构造方法
    // 实例化对象
    return instantiateBean(beanName, mbd);
}
~~~

**createBeanInstance --> instantiateBean**

调用重载的方法`instantiate(mbd, beanName, parent)`实例化对象

~~~ java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        if (System.getSecurityManager() != null) {
            beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
				getInstantiationStrategy().instantiate(mbd, beanName, parent), 									getAccessControlContext());
        }
        else {
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    }
~~~

~~~ java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
    // Don't override the class with CGLIB if no overrides.
    if (!bd.hasMethodOverrides()) {
        Constructor<?> constructorToUse;
        synchronized (bd.constructorArgumentLock) {
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse == null) {
                final Class<?> clazz = bd.getBeanClass();
                try {
                    // .....
                    // 得到默认的构造方法
                    constructorToUse = clazz.getDeclaredConstructor();
                // .....
                    
        // 使用反射实例化对象
        return BeanUtils.instantiateClass(constructorToUse);
~~~

**instantiate --> instantiateClass**

使用反射，调用构造器的`newInstance`方法实例化对象

~~~ java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws 			BeanInstantiationException {
    Assert.notNull(ctor, "Constructor must not be null");
    try {
        ReflectionUtils.makeAccessible(ctor);
        // ctor.newInstance(args): 使用反射来实例化对象
        return (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass()) ?
                KotlinDelegate.instantiateClass(ctor, args) : ctor.newInstance(args));
~~~

## 缓存注解信息，合并解析后的BD对象

略

## 提前暴露一个工厂对象

**doCreateBean --> addSingletonFactory**

~~~ java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(singletonFactory, "Singleton factory must not be null");
    // singletonObjects 一级缓存(单例池)
    synchronized (this.singletonObjects) {
        // 此处主要是为了解决循环依赖
        // 如果单例池中已有了当前beanName对相应的Bean，一个完整的Bean自然已经完成了属性注入，不需要再解决循环依赖问题
        if (!this.singletonObjects.containsKey(beanName)) {
            // 把工厂对象put到singletonFactories(三级缓存)中去
            this.singletonFactories.put(beanName, singletonFactory);
            // 从二级缓存中remove掉这个对象
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
~~~

## 判断属性注入

**doCreateBean --> populateBean**

~~~ java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // .....

    // 判断是否要完成属性注入
    if (!continueWithPropertyPopulation) {
        return;
    }

    // .....

    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        // .....
        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
        // 遍历后置处理器，完成属性注入
        pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(),beanName);
~~~

## 回调部分Aware接口

## 执行生命周期初始化回调方法

**doCreateBean --> initializeBean**

回调Aware接口和执行生命周期初始化回调方法都是在`initializeBean`方法中进行

~~~ java
protected Object initializeBean(final String beanName, final Object bean, 
                                	@Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            // 执行部分Aware接口(BeanNameAware、BeanClassLoaderAware、BeanFactoryAware)
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        // 执行部分Aware接口(BeanNameAware、BeanClassLoaderAware、BeanFactoryAware)
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 执行部分Aware接口
        // 执行spring中的内置处理器(注解版的生命周期初始化回调方法)  -->  xxxxPostProcessor  -->  @PostConstruct
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 执行InitializingBean方法  (其中会执行接口形式和Xml配置形式的生命周期初始化回调方法)
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    
    if (mbd == null || !mbd.isSynthetic()) {
        // 完成AOP --> 生成代理
        // 完成事件发布、监听等
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
~~~

## 完成代理

**initializeBean --> applyBeanPostProcessorsAfterInitialization**

~~~ java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        // 完成代理
        Object current = processor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
~~~

## 将对象put到单例池

**getSingleton --> addSingleton**

~~~ java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        // 从单例池中拿Bean -> 判断对象是否已经被创建
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            
            // .....
            
            try {
                // getObject(): 调用lambda表达式中的createBean()方法实例化对象
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            
            // .....
            
            if (newSingleton) {
                // newSingleton为true --> 对象创建成功 --> 将创建好的、执行完生命周期的对象放到singletonObject(单例池)中，成为一个Bean
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
~~~

~~~ java
// 将对象put到单例池中，完成此步骤后，该单例对象即成为一个Bean
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
    }
}
~~~

