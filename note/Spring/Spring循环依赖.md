# Spring循环依赖

---

## Spring循环依赖

### Bean的生命周期

- 普通类的实例化：**加载-验证-连接-解析-初始化**

  > User.java --> javac编译 --> User.class文件 --> 运行main方法 --> 调用c++代码启动jvm虚拟机 
  >
  > --> jvm启动后，就到磁盘上将编译好的.class文件加载到jvm内存中(eg: 方法区) 
  >
  > --> 当jvm执行遇到new关键字 --> jvm根据方法区中类的模板去堆上为它分配一块内存用来存储这个对象

- spring Bean的实例化：

  > User.java --> javac编译 --> User.class文件 
  >
  > --> scan parse(spring扫描、转换) --> BeanDefinition(包含User类的所有信息)
  >
  > --> beanDefinitionMap.put(类名(userService), User对应的BeanDeeafinition)  
  >
  > -->  BeanFactoryPostProcessor(处理器扩展，可对BeanDeeafinition进行修改)
  >
  > --> preInstantiateSingletons  new Object(此时单例的类才调用new方法实例化对象，创建出springBean)
  >
  > --> bean放入spring单例池中(一个map结构)

### 三级缓存

spring解决循环依赖的三级缓存涉及到三个Map：**singletonObjects**、**earlySingletonObjects**、**singletonFactories**

#### singletonObjects  -  spring单例池  -  一级缓存

~~~ java
/** Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
~~~

#### earlySingletonObjects  -  二级缓存

~~~ java
/** Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
~~~

#### singletonFactories  -  三级缓存

~~~ java
/** Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
~~~
